---
title: Android时间传递三部曲：综述
tags:
  - Android
  - Handler
  - EventBus
  - LocalBroadcast
comment: true
categories: Android
date: 2016-07-12 23:08:24
---

简单分析完Handler、EventBus以及LocalBroadCastManager原理之后，我们再好好回忆一下他们之间的异同点以及他们各自的优劣，Handler是利用Looper的无限循环不断将消息队列中的消息取出然后进行分发处理，而EventBus和LocalBroadcastManager类似，主要是基于订阅/发布模式，都是保存了订阅者列表，每次post消息都是从订阅者列表中找到合适的对象，然后通过订阅者的订阅方法将事件传递给它。

为了对后面这种方式有更深入的了解，从EventBus和LocalBroadcastManager中抽象出来一个简单的`SimpleBus`，因为主要是出于深入了解的目的，所以写的比较简陋，主要还是基于观察者模式：注册以后添加到订阅者列表，每次发送事件就从订阅者列表中找出合适的订阅者，然后调用订阅者的订阅方法通知订阅者。

至此，我们的线程间


`Handler`既然不用多说，它实现了在茫茫线程池中，他是主线程的灯塔，不让工作线程迷失，不论它进行到哪一步，都可以通过`Handler`找到主线程，并向其汇报工作进度，最典型的使用场景就是`AsyncTask`，让工作线程找到回到主线程的灯塔
难点在于`EventBus`的订阅方法的查找方法，以及`EventBus`实现了线程的调度，可以选择在那个，但是因为用到了反射，有效率问题
`LocalBroadCast`相比`Handler`，因为有封装，所以使用起来是有相对优势的，无脑走`Handler`逻辑

## 内存泄露
三者如果处理不当都有可能引发内存泄露，具体来说：`Handler`因为发送的每个`Message`对象有持有一个对发送者`Handler`的引用，方便到处理消息的时候调用`handler`的`handleMessage()`方法，所以在把`Handler`当做内部类使用时，一定要使用`static`，不然因为内部类会持有外部类的引用，会导致外部类泄露，直到消息被处理并回收；`EventBus`和`LocalBroadCast`就类似了，因为它们都会在注册的时候将订阅者保存在订阅者列表中，所以在不使用的时候一定要调用取消订阅方法将对象移除