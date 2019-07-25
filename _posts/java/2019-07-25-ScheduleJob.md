---
title: schedule job 特定时间区间执行
author: ninuxGithub
layout: post
date: 2019-7-25 14:01:13
description: "schedule job"
tag: java
---

### 场景
    当需要在一天的特定的时间段内推送不同类型的内容到远程的服务器；
    我们一版的schedule可以通过cron表达是使用特定的频率来实现我们的任务的执行调度，但是如何控制我们的任务只在特定的
    时间区间里面执行呢？
    
    
    思路： 还是通过schdule进行一个特定频率的执行， 在schdule内部加入一个schduleJob, 我们可以控制job的
    开始时间和job的结束时间来控制该job执行的时间控制到某个具体的范围[startTime, endTime]之间， 
    并且以适当的频率执行；
    
## 步骤
    spring boot + quartz
    spring boot 2.x以后， 为我们提供了quartz的starter依赖
    首先需要了解spring boot 整合quartz可以有2中方式， 一种job是基于内存的（RAMJobStore）；
    另外一种是基于数据库的（LocalDataSourceJobStore） 需要依赖数据源，不能保证调度的时间的准确性，因为需要读取数据库
    可能会消耗一些时间；
    
    第一步：加入maven 依赖
    
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```      

    第二部： spring 需要配置对schdule进行自动配置所以需要在application.yml 进行配置
    当前的yaml的是开发环境， 配置上quartz的一些自定义的配置
    
```yaml
spring:
  profiles:
    active: dev

  quartz:
    properties:
      org:
        quartz:
          jobStore:
            class: org.quartz.simpl.RAMJobStore
          scheduler:
            instanceName: quartzScheduler
            instanceId: AUTO
          threadPool:
            class: org.quartz.simpl.SimpleThreadPool
            threadCount: 8
            threadPriority: 5
            threadsInheritContextClassLoaderOfInitializingThread: true
```    

    第三部： 开始我们的业务代码
    我们的逻辑是将Schedule注入到我们的service里面去
    然后创建一个Job， 通过DataMap来绑定我们要执行任务的bean , 和要执行的目标方法 
    然后创建一个Triiger , 设置开始时间， 结束时间，调度的频率
    最后就是将我们的job,trigger交给schdule进行调度执行了；
    
    那么schedule 什么时候被执行就通过另外的一个schedule cron表达是来触发了；
    
```java
/**
 * @author shenzm
 * @date 2019-7-24
 * @description 作用
 */

@Service
@EnableScheduling
public class ScheduleService {

    private static final Logger logger = LoggerFactory.getLogger(ScheduleService.class);

    @Autowired
    private Scheduler scheduler;

    @Autowired
    private TargetJob targetJob;

//    1. Seconds （秒）
//    2. Minutes （分）
//    3. Hours （时）
//    4. Day-of-Month （天）
//    5. Month （月）
//    6. Day-of-Week （周）
//    7. Year (年 可选字段)
    @Scheduled(cron = "0/10 * * * *  ?")
    public void runJob(){
        startJob("7:17","15:38",5, new QuartzJob(targetJob, "testSchedule"));

        startJob("13:37","15:38",5, new QuartzJob(targetJob, "testSchedule2"));
    }


    public void startJob(String start, String end,int intervalSecond,QuartzJob quartzJob) {
        try {
            String jobPreffix = quartzJob.getTargetMethod();
            String jobName = jobPreffix+"_jobName";
            String jobGroup = jobPreffix+"_jobGroup";
            String triggerName = jobPreffix+"_triggerName";
            String triggerGroup = jobPreffix+"_triggerGroup";
            JobDataMap jobDataMap = new JobDataMap();
            jobDataMap.put("target",quartzJob.getTarget());
            jobDataMap.put("method",quartzJob.getTargetMethod());
            JobDetail job = JobBuilder.newJob(quartzJob.getClass()).withIdentity(jobName, jobGroup).setJobData(jobDataMap).build();
            Date date = new Date();
            SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm");
            String format = dateFormat.format(date);
            String preffix = format.substring(0, 10);
            Trigger trigger = null;
            JobKey jobKey = JobKey.jobKey(jobName, jobGroup);
            TriggerKey triggerKey = TriggerKey.triggerKey(triggerName, triggerGroup);
            if(scheduler.checkExists(jobKey)){
                // 停止触发器
                scheduler.pauseTrigger(triggerKey);
                // 移除触发器
                scheduler.unscheduleJob(triggerKey);
                //删除job
                scheduler.deleteJob(jobKey);
            }
            try {
                Date startTime = dateFormat.parse(preffix+" "+ start);
                Date endTime = dateFormat.parse(preffix+" "+ end);
                trigger = TriggerBuilder.newTrigger()
                        .withIdentity(triggerName, triggerGroup)
                        .startAt(startTime)
                        .endAt(endTime)
                        .withSchedule(simpleSchedule().withIntervalInSeconds(intervalSecond).repeatForever())
                        .build();
            } catch (ParseException e) {
                e.printStackTrace();
            }
            try {
                scheduler.scheduleJob(job, trigger);
            } catch (SchedulerException e) {
                logger.error("注册任务和触发器失败", e);
            }
        } catch (SchedulerException e) {
            e.printStackTrace();
        } finally {

        }
    }

}


@Component
@EnableScheduling
public class TargetJob {
    private Logger logger = LoggerFactory.getLogger(this.getClass());

    public void testSchedule() {
        this.logger.info("哇被触发了哈哈哈哈哈");
    }
    public void testSchedule2() {
        this.logger.info("哇被触发了哈哈哈哈哈222");
    }

}

```    
    
    
    