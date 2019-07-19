```java
/**
 * A Handler allows you to send and process {@link Message} and Runnable
 * objects associated with a thread's {@link MessageQueue}.  Each Handler
 * instance is associated with a single thread and that thread's message
 * queue.  When you create a new Handler, it is bound to the thread /
 * message queue of the thread that is creating it -- from that point on,
 * it will deliver messages and runnables to that message queue and execute
 * them as they come out of the message queue.
 * 
 * <p>There are two main uses for a Handler: (1) to schedule messages and
 * runnables to be executed as some point in the future; and (2) to enqueue
 * an action to be performed on a different thread than your own.
 * 
 /
```

Handler在开发中的使用场景不算少，直接new一个Handler对象，Handler的构造函数有下面几种，相信也都有一定的了解，后面也会通过源码分析下这几种构造方法的区别。有些时候还会重写handleMessage方法，自己处理message中的一些信息。

![image-20190625163720484](/Users/zhangjie/Library/Application Support/typora-user-images/image-20190625163720484.png)

![reward](images/打赏  @3x.png)

![topic](images/话题@3x.png)

```java
Handler handler = new Handler(){
  @Override
  public void handleMessage(Message msg) {
  }
};
```

​	当我们不指定Looper时，默认生成的是和当前Thread的looper联系在一起的Handler，如果当前Thread没有Looper时则会抛出Exception（这也是在子线程中使用创建Handler时，需要Looper.prepared()方法的原因）。

这里顺便说下在子线程中创建Handler的几个需要注意的点：

1.就是前面所说的，要在一个新的子线程中创建Handler时，需要先调用Looper.prepared()方法，为当前线程创建一个Looper实例。当然如果我们如果使用了带有Looper入参的构造函数时，那就不需要了，该Handler实例会和传入的Looper联系起来。

```java
public static void prepare() {
    prepare(true);
}
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

2.Looper.prepared()方法在同一个Thread中不能被调用两次，因为"Only one Looper may be created per thread"，每个线程只能创建一个Looper对象。

3.要能正常的使用和子线程联系在一起的Handler对象，还必须在prepared方法之后调用Looper.loop()方法。该方法主要是为MessageQueue开启了一个死循环来轮询所有入队的message，进而进行消息分发，后面我们还会重点看一下这个方法的实现。从逻辑也可以看出，这个方法必须要在prepared方法之后才能调用。

```java
/**
 * Run the message queue in this thread. Be sure to call
 * {@link #quit()} to end the loop.
 */
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
  	......
    for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            ......
            final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
            final long dispatchEnd;
            try {
                msg.target.dispatchMessage(msg);
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            ......
            msg.recycleUnchecked();
        }
}
```

4.从上面loop方法的注释，我们也可以看出，我们必须要保证在不需要使用该Handler时要退出其关联的loop方法，否则该方法会一直轮询，那么子线程也会一直运行。退出方法有下面两个

```kotlin
Looper.myLooper()?.quit()
Looper.myLooper()?.quitSafely()
```

这两个方法的却别在于：

当我们调用Looper的quit方法时，实际上执行了MessageQueue中的removeAllMessagesLocked方法，该方法的作用是把MessageQueue消息池中所有的消息全部清空，无论是延迟消息（延迟消息是指通过sendMessageDelayed或通过postDelayed等方法发送的需要延迟执行的消息）还是非延迟消息。

当我们调用Looper的quitSafely方法时，实际上执行了MessageQueue中的removeAllFutureMessagesLocked方法，通过名字就可以看出，该方法只会清空MessageQueue消息池中所有的延迟消息，并将消息池中所有的非延迟消息派发出去让Handler去处理，quitSafely相比于quit方法安全之处在于清空消息之前会派发所有的非延迟消息。

无论是调用了quit方法还是quitSafely方法之后，Looper就不再接收新的消息。即在调用了Looper的quit或quitSafely方法之后，消息循环就终结了，这时候再通过Handler调用sendMessage或post等方法发送消息时均返回false，表示消息没有成功放入消息队列MessageQueue中，因为消息队列已经退出了。

##### 从上面的几个点也就可以看出来，每个Thread只有一个Looper，然后在当前Thread创建Handler时，会把Handler和Looper进行关联，在Looper初始化的时候又会为给该Looper初始化一个MessageQueue，这样我们平时使用Handler时的几个要素就关联起来了。



下面我们就看下Handler是怎么来发送和处理Message的：

![image-20190626140227031](/Users/zhangjie/Library/Application Support/typora-user-images/image-20190626140227031.png)

![image-20190626140310511](/Users/zhangjie/Library/Application Support/typora-user-images/image-20190626140310511.png)

![测试图片](https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=1718395925,3485808025&fm=27&gp=0.jpg)

上面就是我们平常使用Handler发送消息时使用的方法。

```java
public final boolean sendMessage(Message msg){
    return sendMessageDelayed(msg, 0);
}
public final boolean sendMessageDelayed(Message msg, long delayMillis){
  if (delayMillis < 0) {
    delayMillis = 0;
  }
  return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
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
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
  msg.target = this;
  if (mAsynchronous) {
    msg.setAsynchronous(true);
  }
  return queue.enqueueMessage(msg, uptimeMillis);
}
```

```java
public final boolean post(Runnable r){
   return  sendMessageDelayed(getPostMessage(r), 0);
}
private static Message getPostMessage(Runnable r) {
  Message m = Message.obtain();
  m.callback = r;
  return m;
}
```

通过查看源码，我们发现不论是sendMessage相关的方法还是post相关的方法，最终都会调用到enqueueMessage方法（区别在于post方法内部会把我们需要执行的runnable包装成一个Message对象，这里还有一点：我们可以new一个Message对象，也可以使用Message.obtain()方法；两者都可以，但是更建议使用obtain方法，因为Message内部维护了一个Message池用于Message的复用，避免使用new 重新分配内存。），进而调用到MessageQueue的enqueueMessage方法。

并且我们看到在enqueueMessage之前给msg.target = this; 就是把当前的Handler对象赋值给了Message的target属性，这里就和loop方法中msg.target.dispatchMessage(msg);对应了起来。

```java
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

private static void handleCallback(Message message) {
  message.callback.run();
}

public void handleMessage(Message msg) {
}
```

当Message的callback不为空时，会直接run方法执行我们post进来的runnable方法，否则会判断mCallback属性，这个值就是我们在创建Handler对象时如果传入了Callback的入参，则会执行该方法处理消息，否则最后会调用Handler的handleMessage方法，这个方法在Handler类中是一个空方法，所以通常我们会重写该方法来处理message。

到这里我们了解了Handler及其一些相关的类的基本的一些信息。



下面带着问题去更详细的看源码：

Handler是怎么实现Message的准确延迟的？Looper.loop()方法中有一个for(;;)的死循环，获取不到需要处理的Message就会阻塞，那在UI线程中为什么不会导致ANR？

（SystemClock.uptimeMillis()方法用来计算自开机启动到目前的毫秒数。如果系统进入了深度睡眠状态（CPU停止运行、显示器息屏、等待外部输入设备）该时钟会停止计时，但是该方法并不会受时钟刻度、时钟闲置时间亦或其它节能机制的影响。因此SystemClock.uptimeMillis()方法也成为了计算间隔的基本依据，比如Thread.sleep()、Object.wait()、System.nanoTime()以及Handler都是用SystemClock.uptimeMillis()方法。这个时钟是保证单调性,适用于计算不跨越设备的时间间隔）

参考文章：https://www.jianshu.com/p/8c829dc15950

前面我们看到MessageQueue的enqueueMessage方法

```java
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }

    synchronized (this) {
        if (mQuitting) {//表示此消息队列已经被放弃了
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
          //如果此队列中头部元素是null(空的队列，一般是第一次)，或者此消息不是延时的消息，则此消								息需要被立即处理，此时会将这个消息作为新的头部元素，并将此消息的next指向旧的头部元素，									然后判断如果Looper获取消息的线程如果是阻塞状态则唤醒它，让它立刻去拿消息处理
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
          //如果此消息是延时的消息，则将其添加到队列中，原理就是链表的添加新元素，按照when，也就									是延迟的时间来插入的，延迟的时间越长，越靠后，这样就得到一条有序的延时消息链表，取出										消息的时候，延迟时间越小的，就被先获取了。插入延时消息不需要唤醒Looper线程
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
      	//唤醒线程
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

```java
Message next() {
    
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            //阻塞方法，主要是通过native层的epoll监听文件描述符的写入事件来实现的。
           //如果nextPollTimeoutMillis=-1，一直阻塞不会超时。
           //如果nextPollTimeoutMillis=0，不会阻塞，立即返回。
           //如果nextPollTimeoutMillis>0，最长阻塞nextPollTimeoutMillis毫秒(超时)，如果期间有程							序唤醒会立即返回。
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
           
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    //msg.target == null表示此消息为消息屏障（通过postSyncBarrier方法发送来的）
                    //如果发现了一个消息屏障，会循环找出第一个异步消息（如果有异步消息的话），所有同步											消息都将忽略（平常发送的一般都是同步消息）
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // 如果消息此刻还没有到时间，设置一下阻塞时间nextPollTimeoutMillis，进入													下次循环的时候会调用nativePollOnce(ptr, nextPollTimeoutMillis)进行													阻塞；
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        //正常取出消息
                        //设置mBlocked = false代表目前没有阻塞
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    //没有消息，会一直阻塞，直到被唤醒
                    nextPollTimeoutMillis = -1;
                }

                if (mQuitting) {
                    dispose();
                    return null;
                }

            pendingIdleHandlerCount = 0;
            nextPollTimeoutMillis = 0;
        }
    }
```

通过上面的源码我们可以看到，如果取出的消息没有到时间，则会重新设置超时时间并赋值给nextPollTimeoutMillis，然后调用nativePollOnce(ptr, nextPollTimeoutMillis)进行阻塞，这是一个native方法，最终会通过Linux的epoll监听文件描述符的写入事件来实现延迟的。

至于为什么主线程的死循环不会导致ANR的原因，网上很多文章都是说因为epoll机制的原因，这里有个博主从另一个角度解释了一下自己的看法，还挺有意思的http://www.androidchina.net/9544.html，他是认为

1. 耗时操作本身并不会导致主线程卡死, 导致主线程卡死的真正原因是耗时操作之后的触屏操作, 没有在规定的时间内被分发
2. Looper 中的 loop()方法, 他的作用就是从消息队列MessageQueue 中不断地取消息, 然后将事件分发出去

他的验证方式也很简单：想象一下, 我们自己写的那个例子, 造成ANR是因为死循环后面的事件没有在规定的事件内分发出去, 而 loop()中的死循环没有造成ANR, 是因为 loop()中的作用就是用来分发事件的, 那么如果我们让自己写的死循环拥有 loop()方法中同样的功能, 也就是让我们写的死循环也拥有事件分发这个功能, 如果没有造成死循环, 那岂不是就验证了第二点原因?? 接下来我将我们的代码改造一下, 我们首先通过一个 Handler 将我们的死循环发送到主线程的消息队列中, 然后将 loop() 方法中的部分代码 copy 过来, 让我们的死循环拥有分发的功能:

```java
new Handler(Looper.getMainLooper()).post(new Runnable() {
    @Override
    public void run() {
        try {
            Looper mainLooper = Looper.getMainLooper();
            final Looper me = mainLooper;
            final MessageQueue queue;
            Field fieldQueue = me.getClass().getDeclaredField("mQueue");
            fieldQueue.setAccessible(true);
            queue = (MessageQueue) fieldQueue.get(me);
            Method methodNext = queue.getClass().getDeclaredMethod("next");
            methodNext.setAccessible(true);
            Binder.clearCallingIdentity();
            for (; ; ) {
                Message msg = (Message) methodNext.invoke(queue);
                if (msg == null) {
                    return;
                }
                msg.getTarget().dispatchMessage(msg);
                msg.recycle();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

    }
});
```



至于最终为什么loop内部的这个死循环没有引发主线程的ANR的原因，以后有机会再
