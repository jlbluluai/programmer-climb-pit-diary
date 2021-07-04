## @Scheduled引起的一次坑


### 发现

![](http://106.15.233.185:8983/1ac42f33-9f2c-42ca-8ead-a45eb72a49ae.jpg)

"S级用户月初自动跟进任务"跑了两次？

### 设计

该任务"S级用户月初自动跟进任务"设定每月初1点跑任务，由于线上环境是2台机器点集群，因此加了一个Redis分布式锁，防止重复跑任务

任务配置
```
@Scheduled(cron = "0 0 1 1 * ?")
```

### 思考过程

1. 第一反应，分布式锁没起作用（基于本公司服务器用的某集团内部自搭，总有点不稳定，带来的怀疑），但是直接否定，重复任务执行时间点是1:04（任务持续20ms，粗略时间）而不是1:00，
   而1点整那个任务已经结束了，所以已经释放了锁，1:04这个任务能够开始是情理之中。
    
2. 疑问？为啥第二台机器点任务会延迟到1:04才开始执行，仔细比对时间，发现猫腻，第二次S任务结束时间01:04:45，持续时间26ms，而它上头一个日志，B、C任务结束时间01:04:18，
   一减发现第二次S任务开始时间是B、C任务结束的时间，也就是有一台服务器的两个1点的任务变成串行了。
   
3. 带着怀疑搜了下，于是知道答案了，@Scheduled标注的定时任务只会给一个线程处理，也就是处理任务的线程只要在工作，就算满足开启条件的任务也只能先进等待队列。


### 解决方案

- 给启动类增加 @EnableAsync 注解（可以理解开启下一步提到的 @Async ，否则不会起作用）

- 给定时任务方法增加 @Async 注解

- 自定义线程池异步执行定时任务
```
    @Bean(name = "scheduledTaskExecutor")
    public ThreadPoolTaskExecutor scheduledTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(8);
        executor.setKeepAliveSeconds(60);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("scheduledTask-");
        // 拒绝后任务交由主线程
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return executor;
    }
```

- 若要异步执行定时任务完善注解 @Async("scheduledTaskExecutor") （如果不自定义线程池，也有默认的异步方案，感兴趣的可以去了解下）


### 总结

对于自己还不太了解的用法，千万不要盲目抄袭公司老代码，别人用的没遇到你的坑的场景，所以也没啥问题，可能自己的场景就会暴露问题，
还是要多了解后再写代码，这次坑带来的影响还好不是太大，该任务重跑产生的多余数据已经删除。
