# Android消息机制

### 消息的载体

我们在传输消息对象时，首先要将对象序列化，此时我们可以选则实现Java提供的Serializable接口，也可以实现Android独有的Parcelable 接口。

实现Serializable接口相对简单，只需制定一个唯一标识便可以

```java
private static final long serialVersionUID=1;
```

而实现Parcelable 相对复杂，但我们可以利用IDE自动生成

**Serializable**在序列化时会伴随着大量的I/O操作，且会产生大量的临时变量，引起频繁的GC，所以在内存中传输时，应使用Parcelable。

**Parcelable**是Android平台独有的接口，Android对它进行了优化，使它能够更好地实现对象在IPC之间的传递，效率更高。但是持久化存储用Parcelable实现困难，应使用Serializable。

### Handler机制详解

单独使用Handler是无法执行任何任务的，它还需要MessageQueue和Looper的配合。

![](http://www.theaze.cn/wp-content/uploads/2019/05/批注-2019-05-27-154001.png)

- **MessageQueue**

顾名思义，MessageQueue是一个消息队列，采用的是单链表的存储结构，对外提供消息的插入和删除操作。但其本身并不具有消息的处理能力，它在Looper的构造方法中创建。

MessageQueue的next()是一个无限循环的方法，如果没有消息会一直阻塞，如果有了消息则会返回该消息并删除。

```java
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }
        }

```



- **Looper**

Looper会无限循环的形式查询MessageQueue是否有新消息，如果有就处理，没有就等待。Looper中有一个ThreadLocal，通过ThreadLocal可以轻松地获取各个线程的Looper，但是线程默认没有Looper，在使用时需要先创建Looper，但为什么主线程不需要创建呢？那是因为在Activity启动时，在ActivityThread的main方法中已经创建了Looper

Looper的loop方法是一个无限循环的方法，它会一直调用MessageQueue的next()方法，因为next()方法是无限循环方法，所以没有消息时它也会阻塞。当有消息时就会处理这条消息`msg.target.dispatchMessage(msg);`

```java
    public static void loop() {
        for (;;) {
            Message msg = queue.next(); // 在这里阻塞
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
			//省略若干代码
            try {
                msg.target.dispatchMessage(msg);  //处理消息
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            //省略若干代码
        }
    }

```

在子线程创建Handler时需要创建Looper才能正常使用

```java
	new Thread("Thread1"){
            @Override
            public void run() {
                Looper.prepare();
                Handler handler = new Handler();
                Looper.loop();
            }
        }.start();
```



- **Handler？**

我们都知道只有UI线程才能更新UI，这是因为Android的UI控件不是线程安全的，而如果在UI控件上加上锁机制会使UI间访问变得复杂，且锁机制会降低UI访问的效率。而单线程更新UI就没有这些问题。在子线程想要更新UI时，只需使用Handler切换至UI线程执行就可以了。

Handler发送消息的所有post、send方法都会调用enqueueMessage，它的原理是向MessageQueue里插入一条消息。

```java
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

- **ThreadLocal**

前面说过Looper是通过ThreadLocal来实现线程中的存取的，那么ThreadLocal究竟是什么呢？

ThreadLocal是线程局部变量，每个不同的线程都有不同的副本，且其他线程不能访问该副本。

ThreadLocal内部维护了一个叫ThreadLocalMap的Map，虽然增删的原理和HashMap相似，但实际上它并没有实现Map接口，Map中的Entry继承了一个ThreadLocal的WeakReference的引用，使用的原因是如果一个对象只有WeakReference(弱引用)，那么它就会在下一次垃圾回收时清理掉。 

```java
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```



**ThreadLocal有内存泄漏问题：**

所以在相关线程结束后，没有了强引用的key就会被回收掉。但是如果value是强引用，那么就会出现key为null的value，出现内存泄漏，所以在设计ThreadLocalMap加入了在调用get() 、set() 、remove()的时候会清理key为null的记录。但是如果我们没有调用过上述方法，依旧会发生内存泄漏，但是由于线程池的容量有限，泄漏的内存也不会无限增长，而只要调用了上述方法，泄漏的内存依旧可以回收，因此我们通常任务这种泄漏时可以忍受的。