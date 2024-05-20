---
title: Java调用linux命令行基础使用
date: 2019-12-02 21:29:07
tags:
- Java
- LinuxShell
categories:
- Java基础
---
### 在Java中如何实现调用命令行
首先申明，不推荐使用此方式，我了解到的情况如下：执行过程是首先克隆一条和当前虚拟机拥有一样环境变量的进程，再用这个新的进程执行外部命令，最后退出这个进程。因此，频繁的创建对CPU和内存的消耗很大
基础代码实现：
```.java
    /**
     * run command with one command
     *
     * @param command command
     * @return command out
     */
    public static String run(String command) {
        ProcessBuilder processBuilder = new ProcessBuilder();
        processBuilder.command("bash", "-c", command);
        String out = "";
        try {
            log.info(String.join(" ", processBuilder.command()));
            Process process = processBuilder.start();
            out = IOUtils.toString(process.getInputStream());
        } catch (IOException e) {
            log.error("run command error {}", e.getMessage());
            e.printStackTrace();
        }
        return out;
    }
```
### 需要考虑的问题
1. 异常数据的输出，是否监控，以及如何监控，之前的做法是新启动一个线程去监听
2. 使用此方法时，性能会有较大的损耗，如何改善，考虑去除所有监控进行性能对比
3. 在执行命令出现异常时的异常处理如何。
以上几点我会在后续的编码中考虑解决，可以关注我的github的demo项目 [Git地址](https://github.com/beiluonever/demo-collection/tree/master/process)

### 注意事项
1. 命令无权限
考虑文件的可执行权限
2. 等待shell返回
在process.waitFor()前读取缓冲区
3.命令不存在
需要将你的程序在bin下添加软链接
以上参考了https://yq.aliyun.com/articles/2362 感谢
