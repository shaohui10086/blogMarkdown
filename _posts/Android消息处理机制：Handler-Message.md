---
title: Android消息处理机制：Handler|Message
tags:
  - Android
  - Handler
  - Message
comment: true
categories: Android
date: 2016-07-15 23:23:44
---


在日常开发中，不管出于什么目的，我们可能都会用到Handler来异步更新UI，有时是为了将一些费时的操作放到异步线程去处理，然后通过Handler将数据更新到UI线程，有时是为了在子线程里更新UI，种种原因，反正我们最后都是选择了直接的Handler+Message组合或者AsyncTask，而了解AsyncTask的同学都知道，AsyncTask内部就是通过Handler和Message实现的线程间通信，所以我们还是要好好熟悉一下这位老朋友

<!-- more -->

#### 为什么要使用Handler
我们使用Handler更多是为了异步操作，那么为什么非得要异步操作呢，直接在子线程更新UI不行吗？答案当然是不行的，不然Handler存在的意义是什么，这一切都是因为UI的控件都不是线程安全的，如果允许并发访问，那控件的状态就是未知的了，可能你刚获取完一个控件的状态，准备进行一些操作，这时候另一个线程改变了这个控件的状态，那就很麻烦了，解决这种问题的方法一般有两种：1.加锁，在对控件进行操作的时候先加锁，不允许其他人访问，这样的问题是会导致UI更新的效率会很差，而且容易堵塞某些线程，因为他需要等上一个访问这个控件的线程释放锁；所以Android选择的是第二种方法，只允许在一个线程内对UI控件进行更新，这个线程就是主线程，这也就是为什么我们在对UI组件进行更新的时候，必须回到主线程去操作。

> 经極[ws丶琪](http://weibo.com/2kpurplerose) 指正，更新View的线程不一定就是主线程，而是创建该View的线程，换言之，更新一个View只能由创建它的线程操作。

#### Handler、MessageQueue和Looper
我用了Handler，不过总是对它一知半解，一直停留在会用的程度，以致于别人在提到MessageQueue和Looper的时候，我竟是一脸懵逼，然后就是无尽的嘲笑，连这些都不知道，你还用什么Handler，回去搬砖吧！我相信也有很多的Android初学者跟我有同样的问题，所以在这里，我想先跟大家详细介绍一下这几个概念

##### Looper
Looper是一个循环类，Handler初始化的时候必须依靠Looper，换言之，Handler初始化的那个线程，必须有Looper，否则就会报异常信息，Looper初始化的过程用代码表示就是如下所示:

``` java
new Thread(new Runnable() {
            @Override
            public void run() {
                Looper.prepare();
                Handler handler = new Handler();
                Looper.loop();
            }
        }).start();
```
我们之所以之前没有感受到它的存在，是因为在主线程，ActivityThread默认会把Looper初始化好，prepare以后，当前线程就会变成一个Looper线程，增加一个Looper对象，而Looper会维护着一个MessageQueue对象，用来存放Handler发送的Message。

> 其实Looper对象的本质是ThreadLocal，是一个线程内部的数据存储类，有兴趣的同学可以去搜索详细了解下

Looper的作用就是从消息队列里取出消息然后进行处理，没有消息处理的时候，它就堵塞在那里，一有新的消息进来，它就从消息队列中取出，处理。它就是一个任劳任怨的搬运工，它的特点在于它是跟它的线程是绑定的，处理消息的过程也是在Looper所在的线程去处理的，这就是为什么Handler在其他线程发的消息，最后也是在主线程处理，因为它只跟Handler初始化的线程有关，确切的说是Handler初始化的时候绑定的Looper所在的线程有关。这样异步的目的就达到了。

如上面代码显示的，Looper主要有两个方法：prepare()用来初始化当前线程的Looper，loop()用来开始Looper的循环，其实还有两个不是很常用的方法：quit()和quitSafely()，这两个方法的不同在于，quit会立即停止循环，而quitSafely会在MessageQueue为空以后才跳出循环。

##### MessageQueue

MessageQueue是一个消息队列，用来存放Handler发送的消息，主要有两个操作:添加和读取，读取的同时伴随着消息从消息队列的移除，分别对应的方法就是：enqueueMessage(Message msg, long when)和next()，enqueueMessage方法就是简单地将一条消息插入MessageQueue，next方法相对会复杂一点，它是一个死循环，返回值是一个Message，它的返回值就是用作Looper处理用的，所以延迟发送消息的主要处理步骤就是在next()方法里，因为next()之前的操作都是记录message延迟到什么时间，然后设置给Message.when，next()之后的Looper是不处理延迟时间的，会直接调用Handler里的处理逻辑，在next()内部，当没有消息返回时，next()就会堵塞，直到有新的消息过来再返回，然后又进入堵塞状态，等待新消息进来。

Message算是一个比较简单的类，它本质上是一辆马车，只是用来装载信息，然后在Handler，MessageQueue和Looper之前传递，主要有这么几个属性：

- what：int型，最主要的属性，用于指定消息的类型，这算是Message唯一一个必须指定的值
- arg0，arg1：两个int型值，一般用来传递一些progress，或者一些简单的整型
- obj：可以传递一些复杂一些的object
- data：Bundle型，这个就不用多解释了，传递较多种数据的时候肯定会用到它
- callback：Runable型，Handler.post(Runnable)内部就是设置的这个属性，这个一般不会手动设置，但是也会用到，只是我们感觉不到，下面会详细解释，用于覆盖Handler的默认处理逻辑

大致就这些属性了，详细了解Message的基本结构还是有必要的，传递什么数据，用什么方法，对以后的实际开发会有一些帮助。

##### Handler

说了这么多铺垫，准备工作算是差不多了，Handler也就要上场了，其实Looper，MessageQueue和Message，除了Message有些接触以外，其他两个在实际开发中其实是见不到的，而Handler则是开发者接触最多的，因为几乎所有的处理逻辑都是写在Handler里的，发送消息和处理消息也都算是Handler接手的，但是我们平常是怎么创建一个Handler的呢? 无外乎两种：

- 创建一个自定义的Handler继承Handler，并重写handlerMessage方法。
- 直接使用默认的Handler类，但是在新建Handler对象时，传入一个Callback对象。

上面说的这两种方法，我用的比较多的是第二种方法，因为用起来更加灵活一点。

#### 获得一个Message：obtain()
在发送一个Message之前 ，我们肯定要先得到一个Message对象，可以直接new一个Message对象出来，但是这种方法并不推荐，更推荐用Message的obtain方法，它类似一个线程池，创建了一个Message池，如果有闲置的Message就直接返回，不然就新建一个，用完以后，返回消息池，这种方法大大减少了当有大量Message对象而产生的垃圾回收问题，而且有obtain方法有多种形式，基本能满足我们的一些需求：

- obtain() 返回一个空的Message
- obtain(Message msg) copy一个message并返回
- obtain(Handler h) 返回一个设置了target的message，设置了这个属性以后message就可以使用sendToTarget()方法，其实内部调用的就是Handler.sendMessage(this)
- obtain(Handler h, Callback callback) 同上，并设置message的callback属性
- obtain(Handler h, int what) 同obtain(Handler h)，并设置message的what属性
- obtain(Handler h, int what, Object obj)
- obtain(Handler h ,int what, int arg0, int arg1)
- obtain(Handler h, int what, int arg0, int arg1, Object obj)

这些方法都大同小异，无非就是预设几个属性，不过，以后尽量还是要用obtain方法，算不上方便快捷吧，但至少算的上优雅。

##### 发送一个Message：sendMessage()
一个Message已经准备好了，蓄势待发，接下来的工作就是把它发射出去，这时候就要交给Handler，调用它的sendMessage(Message msg)，其实我们也可以不去费心得到一个Message对象直接用Handler.sendEmptyMessage(int what)，发送一个空message，而且Handler还有sendMessageDelay和sendMessageAtTime方法，这些有什么不同点，可以直接看源码

``` java
public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
```

``` java
public final boolean sendEmptyMessage(int what)
   {
       return sendEmptyMessageDelayed(what, 0);
   }
```

``` java
public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```

从这些代码中，我们可以看出来，sendEmptyMessage和sendMessage都是调用的sendMessageDelay方法，只不过sendEmptyMessage是在方法内部用Message.obtain()方法得到了一个空message，然后sendMessageDelay方法内部又是调用的sendMessageAtTime，所以殊途同归，最后只要看sendMessageAtTime就好了：

``` java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```

没错，又回来了，看到了enqueueMessage方法，就是这样，一条message被插入MessageQueue队列，接下来的视情节就交给Looper，它会循环调用MessageQueue的next方法，而next会在合适的时机返回要处理的Message，交给Looper处理。

#### 消息处理的过程
具体消息处理的过程又是怎样的呢？Handler的post方法和sendMessage有什么不同？都将在这一节揭晓。其实post的Runnable对象，有没有觉得很熟悉，刚才提到的Message也有一个属性是Runnable，就是callback，正如我们所料想的那样，post其实也是发送的一个message，和sendMessage一样，只不过它给message设置了属性CallBack，还是贴最诚实的源码：

``` java
public void dispatchMessage(Message msg) {
       if (msg.callback != null) {
           handleCallback(msg);
       } else {
           if (mCallback != null) {
               if (mCallback.handleMessage(msg)) {
                   return;
               }
           }
           handleMessage(msg);
       }
   }
```

从源码中我们可以看到，当message的callback属性不为空的话，就会调用handlerCallback(msg)方法，而这个方法内部则是直接运行的message的callback，消息处理结束；当message的callback为空时，就是这个message是一个普通的message，会先查看mCallback是否存在，如果存在的话，尝试调用mCallback的handlerMessage方法，根据返回值决定是否再调用Handler的handlerMessage方法，而这个mCallback变量就是我们在新建handler时，传入的那个Runable对象：

``` java
Handler handler = new Handler(new Handler.Callback() {
            @Override
            public boolean handleMessage(Message msg) {
                return false;
            }
        });
 ```
 
 现在终于理解为什么在这个地方传入一个Runable对象也能拦截下消息处理了，至此，消息处理的优先级就出来了：

> Message的callback>handler的Callback>Handler的handlerMessage

#### 可能存在的问题：内存泄露
在使用Handler的时候，之前一直没有关心过，直到有一次发生了内存泄露才注意到这一点，在Activity销毁的时候要记得把handler的消息队列清空，或者在使用handler时，把handler设成静态变量或者弱引用。

#### Message上限
之前我记得有个同事问我Message是否有上限，我当时答不上来，后来去搜索也没搜索到，只查到Message.obtain()的消息池上限是50个，但是MessageQueue得上限还是不知道，或许是他问错了，或者我理解错了，还是这个问题是存在的，只是我还没找到答案，有知道的看官可以留言指导一下。

#### The End
虽然现在RxJava和Agera用的比较火，但是关于Handler的一些东西，作为开发者，我们还是有必要知道的，通过看任玉刚大大的书，还有网上的各种资料，最后算是对Handler和Message有了个一知半解，有不对的地方，希望大家批评指正。

熬夜写博客，真是一种挑战，困得要死，想着都要放弃了，明天再写，但是我知道以我性格一旦推下去，基本上就废掉了，于是冰箱拿了瓶啤酒，夜里3点，终于写完了。
