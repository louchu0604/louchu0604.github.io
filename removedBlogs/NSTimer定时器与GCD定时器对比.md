---
title: NSTimer定时器与GCD定时器对比
date: 2017-03-05 14:56:33
tags:
---

最近使用的项目需要用到定时器，对时间的精度要求较高。
NSTimer是常用的定时器，先来复习一下。
###初始化一个timer 每个一秒钟执行timerAction,supTimer：给定时间后停止timer

```objc

self.timer = [NSTimer scheduledTimerWithTimeInterval:1 target:self selector:@selector(timerAction) userInfo:nil repeats:YES];
self.supTimer = [NSTimer scheduledTimerWithTimeInterval:10 target:self selector:@selector(invalidateTimer) userInfo:nil repeats:false];


```
```objc

- (void)timerAction
{
NSLog(@"timerAction");
}
- (void)invalidateTimer
{
if (self.timer) {
[self.timer invalidate];
}
}

```
看一下控制台输出结果

2017-03-05 15:16:38.108 eg0305[1235:130688] timerAction
2017-03-05 15:16:38.663 eg0305[1235:130688] timerAction
2017-03-05 15:16:39.117 eg0305[1235:130688] timerAction
2017-03-05 15:16:39.647 eg0305[1235:130688] timerAction
2017-03-05 15:16:40.128 eg0305[1235:130688] timerAction
2017-03-05 15:16:40.627 eg0305[1235:130688] timerAction
2017-03-05 15:16:41.167 eg0305[1235:130688] timerAction
2017-03-05 15:16:41.602 eg0305[1235:130688] timerAction
2017-03-05 15:16:42.095 eg0305[1235:130688] timerAction
2017-03-05 15:16:42.670 eg0305[1235:130688] timerAction

误差很明显


接下来尝试一下GCDTimer
```objc
@property (nonatomic, strong) dispatch_source_t GCDTimer;


```

```objc

dispatch_queue_t queue = dispatch_get_main_queue();
//1.创建GCD中的定时器
/*
第一个参数:创建source的类型 DISPATCH_SOURCE_TYPE_TIMER:定时器
第二个参数:0
第三个参数:0
第四个参数:队列
*/
dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);

//2.设置时间等
/*
第一个参数:定时器对象
第二个参数:DISPATCH_TIME_NOW 表示从现在开始计时
第三个参数:间隔时间 GCD里面的时间最小单位为 纳秒
第四个参数:精准度(表示允许的误差,0表示绝对精准)
*/
dispatch_source_set_timer(timer, DISPATCH_TIME_NOW, 0.5 * NSEC_PER_SEC, 0 * NSEC_PER_SEC);

//3.要调用的任务
dispatch_source_set_event_handler(timer, ^{
NSLog(@"GCDTimerAction");
//        NSLog(@"GCD-----%@",[NSThread currentThread]);
});

//4.开始执行
dispatch_resume(timer);

//
self.GCDTimer = timer;


```
查看控制台输出
2017-03-05 15:25:23.538 eg0305[1393:177332] GCDTimerAction
2017-03-05 15:25:24.095 eg0305[1393:177332] GCDTimerAction
2017-03-05 15:25:24.584 eg0305[1393:177332] GCDTimerAction
2017-03-05 15:25:25.065 eg0305[1393:177332] GCDTimerAction
2017-03-05 15:25:25.585 eg0305[1393:177332] GCDTimerAction
2017-03-05 15:25:26.108 eg0305[1393:177332] GCDTimerAction
2017-03-05 15:25:26.586 eg0305[1393:177332] GCDTimerAction
2017-03-05 15:25:27.036 eg0305[1393:177332] GCDTimerAction
2017-03-05 15:25:27.588 eg0305[1393:177332] GCDTimerAction
2017-03-05 15:25:28.051 eg0305[1393:177332] GCDTimerAction
2017-03-05 15:25:28.533 eg0305[1393:177332] GCDTimerAction
2017-03-05 15:25:29.033 eg0305[1393:177332] GCDTimerAction
2017-03-05 15:25:29.594 eg0305[1393:177332] GCDTimerAction
2017-03-05 15:25:30.106 eg0305[1393:177332] GCDTimerAction
2017-03-05 15:25:30.591 eg0305[1393:177332] GCDTimerAction
2017-03-05 15:25:31.033 eg0305[1393:177332] GCDTimerAction


