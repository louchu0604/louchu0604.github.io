---
title: iOS事件响应链
date: 2018-11-26 00:51:41
tags: iOS 
---

触摸事件的响应机制：
1. 系统检测到触摸等事件后，会将事件打包成UIEvent对象发给当前活动的APP的事件队列。
2. UIApplication会从事件队列中取出事件并传递给UIWindow
3. UIWindow会使用hitTest:withEvent:方法寻找此次Touch操作初始点所在的视图(View) 
   *  hitTest:withEvent:会在当前视图上调用pointInside:withEvent:方法判断触摸点是否在当前视图内。若返回NO则hitTest:withEvent:返回nil，若返回YES则向当前视图的所有subviews发送hitTest:withEvent:直到有子视图返回非空对象或者全部子视图遍历完毕。

针对以上内容，我写了一个demo：
处理的case有两个：
1. 直接让父视图响应
2. 子视图超出父视图的部分也能响应



