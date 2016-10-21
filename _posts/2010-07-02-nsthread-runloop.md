---
layout: post
title: NSThread runloop | iOS上如何实现长线任务
tags:
- NSThread runloop
- iOS 长线任务
categories: iOS
description: 
---
##背景
手机导航过程中，需要一直不停的建模以及绘制到内存，需要能在iPhone系统中开一个长线任务，去执行这个数据地图建模。但是目前ios本身不支持长线任务的接口。所以需要自己想办法实现。
iphone为我们提供了很多简单易于操作的线程方法。iPhone多线程编程提议用NSOperation和NSOperationQueue，这个确实很好用, 但是每次任务没有连续性，不能作为长线任务使用。
我们知道ios、android、plam这类内核都是基于类linux的，很多原理相同性比较高, 比如eventloop、runloop什么的，只不过不对开发者开放，一般也接触不到。UI主线程一定程度上可以粗暴的理解thread+eventloop。
实现方案基本为NSThread创建一个run loop，使得能够Hold住这个线程不被系统回收，简单的实现就是在这个创建的线程里面跑一个长线的任务，还得是比较单一和弱的任务。程序界的共识就是Timer。

## 代码

下面是用timer hold住线程的代码。 

```objective-c
NSThread *thread1 = [[NSThread alloc] initWithTarget:self selector:@selector(playerThread:) object:nil];
[thread start];

如果要利用NSOperation，原理类似。只需要加入到queue里面去就好了。。queue会在合适的时机调用方法，下面代码作为参考。
- (void) **playerThread:** (void*)unused 
{ 
  audioRunLoop = CFRunLoopGetCurrent();   子线程的runloop引用。       	 NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
  //子线程的run loop 
  [**self initPlayer** ]; 
  CFRunLoopRun(); //运行子线程的run loop,这里就会停住了。
  [pool release]; 
} 
实现一个timer,用于检查子线程的工作状态，并在合适的时候做任务切换。或者是合适的时候停掉自己的runloop.runloop退出以后，系统就会回收这个线程
-(void) initPlayer
{ 
  // 在这里你可以初始化一个工作类。比如声音或者视频播放。 
  NSTimer *stateChange = [NSTimer scheduledTimerWithTimeInterval:0.5 target:self selector:@selector(checkState:) userInfo:nil repeats:YES]; 
} 
-(void)checkState:(NSTimer*) timer 
{ 
  if(需要退出自线程了) 
  { 
    释放子线程里面的资源。 
    CFRunLoopStop( CFRunLoopGetCurrent());
    结束子线程任务 
  } 
} 
其实实现方式还有几种，这个只是其中一种。。。关于NSInvocationOperation。网上例子很多。不多做介绍了。。
```

