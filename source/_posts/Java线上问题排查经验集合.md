---
title: Java线上问题排查经验集合
date: 2023-2-13 14:41:45
tags:
- Java
- JVM
categories:
- Java基础
---

# Java线上问题排查经验集合
## high cpu
1. 使用Top -H 查看CPU占用较高的线程ID。如：
![top-H](https://blog.ohmyssh.top/images/top-h.png)
```
jps -l # find the right process ,if pid is 1
top -H -p 1 # find the top thread CPU usage in this process. different linux commond may different. 
jstack 1 > stack.log # get current java thread info ,save to stack.log file 
#if the CPU usage top 1 is 7
printf %x 7 # get the 16 hexadecimal value of thread id
#find the thread info from stack.log, can use vim or other commond to find.
grep "0x7" -A 10 stack.log
```
2. 获得了thread info后，当然这里也可以通过其他方式获取线程栈信息，比如 [show-busy-java-threads](https://github.com/yikebocai/shell/blob/master/show-busy-java-threads.sh) 和 [arthas thread](https://arthas.aliyun.com/doc/thread.html#%E5%8F%82%E6%95%B0%E8%AF%B4%E6%98%8E)，可以分析堆栈信息，比如：
```
"http-nio-8080-exec-2" #20 daemon prio=5 os_prio=0 cpu=0.90ms elapsed=155.86s tid=0x00001465ecfb8800 nid=0x2b waiting on condition  [0x00001465ac2c4000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(java.base@11.0.18/Native Method)
	at com.dockerforjavadevelopers.hello.HelloController.index(HelloController.java:12)
	at jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(java.base@11.0.18/Native Method)
	at jdk.internal.reflect.NativeMethodAccessorImpl.invoke(java.base@11.0.18/NativeMethodAccessorImpl.java:62)
	at jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(java.base@11.0.18/DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(java.base@11.0.18/Method.java:566)
    ....
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:889)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1743)
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
	- locked <0x000000062aa88668> (a org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper)
	at org.apache.tomcat.util.threads.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1191)
	at org.apache.tomcat.util.threads.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:659)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	at java.lang.Thread.run(java.base@11.0.18/Thread.java:829)
```
线程停留在了，java.lang.Thread.sleep(java.base@11.0.18/Native Method) 我们需要在线程栈中找到自己业务的信息，这个示例为
com.dockerforjavadevelopers.hello.HelloController.index(HelloController.java:12)，同时通过这个信息我们知道当前线程状态为：TIMED_WAITING，其实并不会消耗cpu（为示例使用的Thread.sleep, 通常这一步会找到占用cpu时间超长的thread）
<!-- more -->

## high memory
1. 对线上服务要进行足够的监控，比如使用prometheus
```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>
```
为项目添加依赖后，可以集成Prometheus和Grafana查看Java相关的监控信息
```
$ curl localhost:8080/actuator/prometheus

# HELP executor_queued_tasks The approximate number of tasks that are queued for execution
# TYPE executor_queued_tasks gauge
executor_queued_tasks{name="applicationTaskExecutor",} 0.0
# HELP executor_pool_size_threads The current number of threads in the pool
# TYPE executor_pool_size_threads gauge
executor_pool_size_threads{name="applicationTaskExecutor",} 0.0
# HELP process_start_time_seconds Start time of the process since unix epoch.
# TYPE process_start_time_seconds gauge
process_start_time_seconds 1.676360742796E9
# HELP executor_queue_remaining_tasks The number of additional elements that this queue can ideally accept without blocking
# TYPE executor_queue_remaining_tasks gauge
executor_queue_remaining_tasks{name="applicationTaskExecutor",} 2.147483647E9
# HELP tomcat_sessions_rejected_sessions_total
.....
```
这里不赘述heap thread stack知识。后续会有另外的文章介绍。

2. 对存在high memory的服务进行dump 分析
使用jmap对所有对象惑live对象进行分析（live参数会强行让jvm做一次full gc），详细使用可以参考[jmap](https://docs.oracle.com/javase/7/docs/technotes/tools/share/jmap.html)
```
 jmap -dump:format=b,file=dump.hprof javaPid
 jmap -dump:live,format=b,file=/tmp/dump.hprof javaPid
```
如果是k8s环境，使用kubectl cp <file-spec-src> <file-spec-dest>将文件从pod中cp出来，example：
```kubectl cp  namespace/pod-id:/tmp/dump.hprof  /path/localfile```  
之后可以使用[MAT](https://www.eclipse.org/mat/)或者[jifa](https://github.com/eclipse/jifa)进行dump文件分析

![MAT](https://blog.ohmyssh.top/images/heap-mat.png)
MAT常用的方法有
+ Histogram: 列出所有类的实例数量
+ Dominator Tree: 列出占用最大的对象，并提供path to GC roots来确定为何没有被释放
+ Leak Suspects: 系统总览，并给出memory leak 最大的嫌疑

一般通过上述方法能够找到Heap中占用较大的objects，如果在分析监控后发现并非在heap中，就需要进行对堆外内存进行排查。通常可以在heap中搜索DirectByteBuffer查看是否有分配，这部分内存会直接被jni管理，不会被GC处理。更多的信息可以参考[JVM 内存模型](https://zhuanlan.zhihu.com/p/38348646)简单来说，堆外内存就是除了堆之外的笼统称呼

## 内存快速增长之后回落，如何排查、
1. 添加GC log信息
-Xloggc:gc.demo.log
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
通常使用如下工作进行GC log收集，后续再进行分析，可以参考[GC log分析](https://juejin.cn/post/6844903861556084744)
2. 使用更强的JFR监控
什么是JFR： Java flight Record，简单来说就是JVM运行期的所有事件都会被记录
```
#采集 5min的 JFR任务(采集固定时长JFR任务)
jcmd 18137 JFR.start name=demo settings=profile delay=5s duration=5m filename="/tmp/jfr/jfligh.jfr" compress=true  
#检查任务是否在进行
jcmd 18137 JFR.check
#另一种方式是一直采集数据
jcmd 18137 JFR.start name=demo settings=profile delay=5s duration=0 compress=true
#转存 jfr 文件
jcmd 18137 JFR.dump name=demo filename="/tmp/jfr/jfight2.jfr" compress=true
#停止持续采集 JFR 任务
jcmd 18137 JFR.stop name=demon  
```
获取到jfr文件后可以使用JMC或者[ZMC](https://www.azul.com/products/components/azul-mission-control/)进行分析
信息会包括Thread/memory/GC/JVM config等各种信息，并且可以对我们关心的时间点进行详细分析。
![ZMC](https://blog.ohmyssh.top/images/jflight-zmc.png)
