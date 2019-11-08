---
title: Spring Boot中使用quartz定时任务并进行事务管理
date: 2019-11-05 09:39:47
categories: 
- Java
tags: 
- "Spring Boot"
- quartz
- 事务

---
## 在Spring Boot2.0中集成Quartz
考虑到业务中需要对任务进行实时修改，因此本文的配置方式可能较为复杂
如果是想了解事务为何不生效的，先告诉结论：将Job中的RUN方法，提取成一个Class的方法，并在此方法上加@Transactional，在job.run中调用。具体原因请参考下文
### 以Spring Boot2.19版本为例，添加POM依赖：
```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-quartz</artifactId>
        </dependency>
```
父项目为Spring Boot
```
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.9.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
```
<!-- more -->
### 添加各种ConfigBean
QuartzConfig.java:
```
import org.quartz.Scheduler;
import org.quartz.spi.JobFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Scope;
import org.springframework.scheduling.quartz.SchedulerFactoryBean;

@Configuration
public class QuartzConfig {

	private JobFactory jobFactory;

	public QuartzConfig(JobFactory jobFactory) {
		this.jobFactory = jobFactory;
	}

	/**
	 * 配置SchedulerFactoryBean
	 *
	 * 将一个方法产生为Bean并交给Spring容器管理
	 */
	@Bean
	@Scope("singleton")
	public SchedulerFactoryBean schedulerFactoryBean() {
		// Spring提供SchedulerFactoryBean为Scheduler提供配置信息,并被Spring容器管理其生命周期
		SchedulerFactoryBean factory = new SchedulerFactoryBean();
		// 设置自定义Job Factory，用于Spring管理Job bean
		factory.setJobFactory(jobFactory);
		return factory;
	}

	@Bean(name = "scheduler")
	public Scheduler scheduler() {
		return schedulerFactoryBean().getScheduler();
	}
}
```
JobFactory.java 用于管理Job
```
import org.quartz.spi.TriggerFiredBundle;
import org.springframework.beans.factory.config.AutowireCapableBeanFactory;
import org.springframework.scheduling.quartz.AdaptableJobFactory;
import org.springframework.stereotype.Component;

/**
 * @author beiluo
 * 解决SpringBoot不能再Quartz中注入Bean的问题
 */
@Component
public class JobFactory extends AdaptableJobFactory {
	/**
	 * AutowireCapableBeanFactory接口是BeanFactory的子类
	 * 可以连接和填充那些生命周期不被Spring管理的已存在的bean实例
	 */
	private AutowireCapableBeanFactory factory;

	public JobFactory(AutowireCapableBeanFactory factory) {
		this.factory = factory;
	}

	/**
	 * 创建Job实例
	 */
	@Override
	protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {

		// 实例化对象
		Object job = super.createJobInstance(bundle);
		// 进行注入（Spring管理该Bean）
		factory.autowireBean(job);
		// 返回对象
		return job;
	}
}
```
QuartzConfigration.java  用于管理所有的job，可以根据自己的业务进行修改
```
import org.quartz.*;
import org.quartz.impl.matchers.GroupMatcher;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.List;
import java.util.Map;


@Service
public class QuartzConfigration {

    private final static String JOB_GROUP_NAME = "JOB_GROUP";

    private final static String TRIGGER_GROUP_NAME = "TRIGGER_GROUP";

    private static Scheduler scheduler;

    public QuartzConfigration(Scheduler scheduler) {
        QuartzConfigration.scheduler = scheduler;
    }

    /**
     * @param jobName     任务名
     * @param triggerName 触发器名
     * @param jobClass    任务
     * @param cron        时间设置
     * @Description: 添加定时任务
     */
    public static boolean addJob(String jobName, String triggerName, Class<? extends Job> jobClass, String cron) {
        try {
            // 任务名，任务组，任务执行类
            JobDetail jobDetail = JobBuilder.newJob(jobClass).withIdentity(jobName, JOB_GROUP_NAME).build();
            // 触发器
            TriggerBuilder<Trigger> triggerBuilder = TriggerBuilder.newTrigger();
            // 触发器名,触发器组
            triggerBuilder.withIdentity(triggerName, TRIGGER_GROUP_NAME);
            triggerBuilder.startNow();
            // 触发器时间设定
            triggerBuilder.withSchedule(CronScheduleBuilder.cronSchedule(cron));
            // 创建Trigger对象
            CronTrigger trigger = (CronTrigger) triggerBuilder.build();
            // 调度容器设置JobDetail和Trigger
            scheduler.scheduleJob(jobDetail, trigger);
            // 启动
            if (!scheduler.isShutdown()) {
                scheduler.start();
            }

            // 启动
            if (!scheduler.isShutdown()) {
                scheduler.start();
            }
            return true;
        } catch (Exception e) {
            return false;
        }
    }

    /**
     * @param jobName     任务名
     * @param triggerName 触发器名
     * @param cron        时间设置
     * @Description: 修改任务的触发时间
     */
    public static boolean modifyJobTime(String jobName, String triggerName, String cron) {
        try {
            // Scheduler sched = schedulerFactory.getScheduler();
            TriggerKey triggerKey = TriggerKey.triggerKey(triggerName, TRIGGER_GROUP_NAME);
            CronTrigger trigger = (CronTrigger) scheduler.getTrigger(triggerKey);
            if (trigger == null) {
                return false;
            }

            String oldTime = trigger.getCronExpression();
            if (!oldTime.equalsIgnoreCase(cron)) {
                /* 方式一 ：调用 rescheduleJob 开始 */
                // 触发器
                TriggerBuilder<Trigger> triggerBuilder = TriggerBuilder.newTrigger();
                // 触发器名,触发器组
                triggerBuilder.withIdentity(triggerName, TRIGGER_GROUP_NAME);
                triggerBuilder.startNow();
                // 触发器时间设定
                triggerBuilder.withSchedule(CronScheduleBuilder.cronSchedule(cron));
                // 创建Trigger对象
                trigger = (CronTrigger) triggerBuilder.build();
                // 方式一 ：修改任务的触发时间
                scheduler.rescheduleJob(triggerKey, trigger);
                /* 方式一 ：调用 rescheduleJob 结束 */
                /* 方式二：先删除，然后在创建新的Job */
                // JobDetail jobDetail = sched.getJobDetail(JobKey.jobKey(jobName,
                // JOB_GROUP_NAME));
                // Class<? extends Job> jobClass = jobDetail.getJobClass();
                // removeJob(jobName, JOB_GROUP_NAME, triggerName, TRIGGER_GROUP_NAME);
                // addJob(jobName, JOB_GROUP_NAME, triggerName, TRIGGER_GROUP_NAME, jobClass,
                // cron);
                /* 方式二 ：先删除，然后在创建新的Job */
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    /**
     * @param jobName     任务名
     * @param triggerName 触发器名
     * @Description: 删除任务
     */
    public static boolean removeJob(String jobName, String triggerName) {
        try {
            TriggerKey triggerKey = TriggerKey.triggerKey(triggerName, TRIGGER_GROUP_NAME);
            scheduler.pauseTrigger(triggerKey);// 停止触发器
            scheduler.unscheduleJob(triggerKey);// 移除触发器
            scheduler.deleteJob(JobKey.jobKey(jobName, JOB_GROUP_NAME));// 删除任务
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    /**
     * @Description: 启动所有任务
     */
    public static boolean startAllJobs() {
        try {
            scheduler.start();
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    /**
     * @Description: 关闭所有任务
     */
    public static boolean shutdownAllJobs() {
        try {
            if (!scheduler.isShutdown()) {
                scheduler.shutdown();
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    /**
     * @Description: 暂停当前任务
     */
    public static boolean pauseJob(String jobName) {
        try {
            scheduler.pauseJob(JobKey.jobKey(jobName, JOB_GROUP_NAME));
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    /**
     * @Description: 开启当前任务
     */
    public static boolean resumeJob(String jobName) {
        try {
            scheduler.resumeJob(JobKey.jobKey(jobName, JOB_GROUP_NAME));
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    /**
     * 获取所有任务名称
     *
     * @return 所有任务名
     * @throws SchedulerException
     */
    public static Map<String, String> getAllJobName() {
        Map<String, String> jobNameTrigger = new HashMap<>();
        try {
            scheduler.getJobGroupNames().forEach(group -> {
                try {
                    scheduler.getJobKeys(GroupMatcher.jobGroupEquals(group)).forEach(jobKey -> {
                        String jobName = jobKey.getName();
                        try {
                            List<? extends Trigger> triggers = scheduler.getTriggersOfJob(jobKey);
                            String triggerName = triggers.get(0).getKey().getName();

                            jobNameTrigger.put(jobName,triggerName);
                        } catch (SchedulerException e) {
                            e.printStackTrace();
                        }
                    });
                } catch (SchedulerException e) {
                    e.printStackTrace();
                }
            });
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
        return jobNameTrigger;
    }
}
```

因为需要动态的修改Job，因此项目中所有Job配置放置于数据库中，在系统启动时初始化所有Job，当有Job变化时，进行reload操作
```
import com....entity.QuartzConfigBean;
import com...QuartzConfigration;
import com...service.IQuartzConfigService;
import org.apache.commons.lang3.StringUtils;
import org.quartz.Job;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.atomic.AtomicBoolean;


@Component
@Order(value = 1)
public class QuartzInitRunner implements CommandLineRunner {

    /**
     * 日志
     */
    private Logger logger = LoggerFactory.getLogger(getClass());

    @Autowired
    private IQuartzConfigService quartzConfigService;

    @Override
    public void run(String... arg0) throws Exception {
        load();
    }

    /**
     * 加载数据字典数据到内存.
     */
    private void load() {
        logger.info("启动初始化Quart加载项.....start");

        List<QuartzConfigBean> list = quartzConfigService.getAllList();

        if (list != null)
            list.forEach(quartzConfig -> {
                try {
                    QuartzConfigration.addJob(quartzConfig.getJobName(), quartzConfig.getTriggerName(), (Class<? extends Job>) Class.forName(quartzConfig.getExecuteClassPath()), quartzConfig.getQuartzCron());
                } catch (ClassNotFoundException e) {
                    logger.error("初始化class失败，class不存在！{}", quartzConfig.getExecuteClassPath());
                }
            });
        logger.info("启动初始化Quart加载项.....end");
    }

    /**
     * 重新加载任务的执行时间
     */
    public boolean reload() {
        logger.info("重新加载所有任务.....start");
        List<QuartzConfigBean> jobs = quartzConfigService.getAllList();
        Map<String, String> jobTrigger = QuartzConfigration.getAllJobName();

        List<String> dbJobNames = new ArrayList<>();
        List<String> quartzAllJobs = new ArrayList<>(jobTrigger.keySet());

        AtomicBoolean flag = new AtomicBoolean(true);
        jobs.forEach(configBean -> {
            String jobName = configBean.getJobName();
            String triggerName = configBean.getTriggerName();
            dbJobNames.add(configBean.getJobName());
            boolean result = false;
            if (jobTrigger.values().contains(jobName)) {
                result = QuartzConfigration.modifyJobTime(jobName, triggerName, configBean.getQuartzCron());
            } else {
                try {

                    Class<?> jobClazz = Class.forName(configBean.getExecuteClassPath());
                    result = QuartzConfigration.addJob(jobName, triggerName,
                            (Class<? extends Job>) jobClazz, configBean.getQuartzCron());
                } catch (ClassNotFoundException e) {
                    logger.error("初始化class失败，class不存在！{}", configBean.getExecuteClassPath());
                }
            }
            //更新cron,
            if (!result) {
                flag.set(false);
            }
        });
        //获取当前内存中job和数据库中job差集
        quartzAllJobs.removeAll(dbJobNames);
        quartzAllJobs.forEach(job -> {
            //删除数据库中不存在的jobs
            if (StringUtils.isNotEmpty(jobTrigger.get(job))) {
                QuartzConfigration.removeJob(job, jobTrigger.get(job));
            }
        });

        //获取现在执行的job，重新load
        logger.info("重新加载所有任务.....end");
        return flag.get();
    }
}
```

表结构如下,此处也可根据自己的业务进行修改，仅供参考：
```
CREATE TABLE `quartz_config`  (
  `id` int(10) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键',
  `quartz_name` varchar(100) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '具体任务名',
  `job_name` varchar(100) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '任务名',
  `trigger_name` varchar(100) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '触发器名',
  `quartz_cycle` char(1) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL DEFAULT '1' COMMENT '字典值frequency 1每月 2每周 3每日 ',
  `execute_time` time(6) NOT NULL COMMENT '执行时间 时分秒',
  `quartz_cron` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT 'cron表达式',
  `execute_class_path` varchar(2000) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '执行类路径',
  `is_delete` tinyint(1) NOT NULL DEFAULT 0 COMMENT '是否删除',
  `type` varchar(1) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL COMMENT '任务类型',
  `associated_id` varchar(20) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL COMMENT '数据关联id',
  `status` varchar(1) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL DEFAULT '1' COMMENT '任务状态 1正在执行 2已暂停 ',
  `start_time` datetime(0) DEFAULT NULL COMMENT '任务开始时间',
  `end_time` datetime(0) DEFAULT NULL COMMENT '任务结束时间',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 66 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

```
编写业务代码implements org.quartz.Job
## 事务无效问题出现
在run方法上添加@Transactional注解，发现事务没有生效，伪代码如下：
```.java
@Transactional(rollbackFor = Exception.class)
public void run(){
	userService.remove();
	int a = 1/0;
	userService.saveBatch(saveUsers);
}
```

### 问题排查

1. 检查事务注解是否添加，是否启用
   - ~~Spring Boot的@SpringBootApplication默认添加了@EnableTransactionManagement~~ 添加了@EnableTransactionManagement无效
2. 了解到事务是对~~@Service注解~~ Spring托管的类 下public 方法进行切入，检查了当前类是否被Spring托管
   - 已经使用@Component ，切换为@Service，依旧没有效果

#### 启用日志查看原因

在application.yaml或applicationproperties添加Debug

```.yaml
logging:
  level:
    org.springframework.jdbc.datasource: debug
```

观察日志发现没有开启create transaction，翻看源码发现在Job.run()方法调用方中，使用try catch对异常进行了捕捉。

```.java
public class JobRunShell extends SchedulerListenerSupport implements Runnable {
	……
	public void run() {
			……
			// execute the job
            try {
                log.debug("Calling execute on job " + jobDetail.getKey());
                job.execute(jec);
                endTime = System.currentTimeMillis();
            } catch (JobExecutionException jee) {
                endTime = System.currentTimeMillis();
                jobExEx = jee;
                getLog().info("Job " + jobDetail.getKey() +
                " threw a JobExecutionException: ", jobExEx);
            } catch (Throwable e) {
                endTime = System.currentTimeMillis();
                getLog().error("Job " + jobDetail.getKey() +
                " threw an unhandled Exception: ", e);
                SchedulerException se = new SchedulerException(
                "Job threw an unhandled exception.", e);
                qs.notifySchedulerListenersError("Job ("
                + jec.getJobDetail().getKey()
                + " threw an exception.", se);
                jobExEx = new JobExecutionException(se, false);
            }
	}
}
```

**因此通常的事务处理会被此处拦截掉，所以要把方法从excute中拿出来。**

这里大致进行了如下修改：

```.java

public void execute(){
	userSync()
}

@Transactional(rollbackFor = Exception.class)
public void userSync(){
	userService.remove();
	int a = 1/0;
	userService.saveBatch(saveUsers);
}
```

实际测试依旧无效，这里涉及到另一个知识点：

**Spring的Aop原理：默认使用Cglib方式做切面，只能在不同类中调用才能正常使用到Spring提供的切面服务**

最后修改方案为把具体业务抽取到另一个类中，在Job类中注入这个类调用方法。

大致如下：

```.java
public class UserSyncJob implements Job {
	@Autowired
    UserSyncServiceImpl syncService;
	
    public void execute(){
        syncService.cleanUserAndSync()
    }
}

@Compoment//@Service 没有明显区别
public class UserSyncServiceImpl {

	@Transactional(rollbackFor = Exception.class)
    public void cleanUserAndSync() {
    	userService.remove();
        userService.saveBatch(saveUsers);
    }
}
```



添加中间类后的日志,删减了部分

> *2019-11-06 11:25:00.023 DEBUG 14228 --- [ryBean_Worker-1] o.s.j.d.DataSourceTransactionManager     : **Creating new transaction with name [UserSyncServiceImpl.cleanUserAndSync]:** PROPAGATION_REQUIRED,ISOLATION_DEFAULT,-java.lang.Exception 
> 2019-11-06 11:25:00.058 DEBUG 14228 --- [ryBean_Worker-1] o.s.j.d.DataSourceTransactionManager     : Acquired Connection [HikariProxyConnection@406789872 wrapping com.mysql.cj.jdbc.ConnectionImpl@1c9d21d9] for JDBC transaction
> 2019-11-06 11:25:00.061 DEBUG 14228 --- [ryBean_Worker-1] o.s.j.d.DataSourceTransactionManager     : Switching JDBC Connection [HikariProxyConnection@406789872 wrapping com.mysql.cj.jdbc.ConnectionImpl@1c9d21d9] to manual commit
> 2019-11-06 11:25:01.038 DEBUG 14228 --- [ryBean_Worker-1] o.s.j.d.DataSourceTransactionManager     : **Initiating transaction rollback**
> 2019-11-06 11:25:01.038 DEBUG 14228 --- [ryBean_Worker-1] o.s.j.d.DataSourceTransactionManager     : Rolling back JDBC transaction on Connection [HikariProxyConnection@406789872 wrapping com.mysql.cj.jdbc.ConnectionImpl@1c9d21d9]
> 2019-11-06 11:25:01.226 DEBUG 14228 --- [ryBean_Worker-1] o.s.j.d.DataSourceTransactionManager     : Releasing JDBC Connection [HikariProxyConnection@406789872 wrapping com.mysql.cj.jdbc.ConnectionImpl@1c9d21d9] after transaction*