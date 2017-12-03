# Android的消息机制

**前言：**

> 提到Android消息机制大家应该都不陌生，在日常开发中都不可避免地要涉及到这方面的内容。从开发的角度来说，Handler 是 Android 消息机制的上层接口，使得我们在开发过程中只需要和 Handler 交互即可。而 Handler 的使用过程也很简单，通过它可以轻松地将一个任务切换到 Handler 所在的线程中去执行。

> 但是很多人认为 Handler 的作用就是更新 UI ，这的确没有错，但是更新 UI 仅仅是 Handler 的一个特殊的使用场景。**具体来说应该是这样的：有时候需要在子线程中进行耗时的 I/O 操作，可能会是读取文件亦或许访问网络等，当耗时操作完成以后可能需要在 UI 上做一些改变，由于 Android 开发规范的限制，我们并不能在子线程中访问 UI 控件，否则就会触发程序异常，这个时候通过 Handler 就可以将更新 UI 的操作切换到主线程中执行。因此，本质上来说， Handler 并不是专门用于更新 UI 的，只是常被开发者用来更新 UI 。**

### Android的消息机制概述

* Android 的消息机制主要是指 Handler 的运行机制以及 Handler 所附带的 MessageQueue 和 Looper 的工作过程，这三者实际是一个整体，只不过我们在开发过程中比较多地接触角到 Handler 而已。

* Handler 的主要作用就是将一个任务切换到某个指定的线程中去执行。可能你会问： **Android 为什么要提供这个功能 ？ 或者说 Android 为什么需要提供在某个具体的线程中执行任务 这个功能？** 主要是因为 Android 规定访问 UI 只能在主线程中进行，如果在子线程中访问 UI ，程序就会抛出异常。

* ViewRootImpl 对 UI 操作做了验证，这个验证是由 ViewRootImpl 的checkThread 方法来完成的。如下：


```
void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
```

* 由于 ViewRootImpl 的 checkThread 的这一点限制，导致了必须要在主线程中访问 UI ,但是 Android 又建议不要在主线程中进行耗时操作，否则的话会导致程序无法响应，即：ANR。所以，考虑到这种情况，假如我们需要从服务端获取到一些数据、信息并将其显示在 UI 上，这个时候必须要在子线程中 做耗时的联网操作来获取数据，待获取到数据后又不能在子线程中直接访问 UI ，而此时如果没有 Handler ，那么我们的确没有办法将访问 UI 的工作切换到主线程中去执行。因此，系统之所以提供了 Handler ，主要原因就是为了解决在子线程中无法访问主线程 UI 的矛盾。

* 在延伸一点，系统为什么不允许在子线程中访问 UI 呢？

```
这主要是因为 Android 的 UI 控件不是线程安全的，如果在多线程中 并发访问 可能会导致 UI 控件处于不可预期的状态。

可能你会说：那为什么系统不对 UI 控件的访问加上锁机制呢？

缺点有两个：

首先加上锁机制会让 UI 访问的逻辑变的复杂；

其次，锁机制会降低 UI 访问的效率，因为锁机制会阻塞某些线程的执行。

鉴于这两点，最简单且高效的方法就是采用单线程模型来处理 UI 操作，对于开发者来说也很简单，只是需要通过 Handler 切换一下 UI 访问的执行线程即可。
```

***

**好吧，啰里啰嗦说了这么多，别耐烦，原谅程序猿的我嘴笨不会说。下面简单的说一下 Handler 的工作原理。敲小黑板了！！！**



```
Handler 创建时会采用当前线程的 Looper 来构建内部的消息循环系统，如果当前系统没有 Looper 那么就会报错。

解决这个问题其实也很简单，只需要在当前线程创建 Looper 即可，或者在一个有 Looper 的线程中创建 Handler 也行。具体的接下来会进行介绍说明。

Handler 创建完毕后，此时其内部的 Looper 以及 MessagerQueue 就可以和 Handler 一起协同工作了，然后会通过 Handler 的 post 方法将一个 Runnable
投递到 Handler 内部的 Looper 中去处理。当然，也可以通过 Handler 的 send 方法发送一个消息，这个消息同样会在 Looper 中去处理。其实， post 方法最终也是通过 send 方法来完成的。

当 Handler 的 send 方法被调用时，会调用 MessagerQueue 的 enqueueMessag 方法将这个消息放入消息队列中，然后 Looper 发现有新的消息到来时，就会处理这个消息，最终消息中的 Runnable 或者 Handler 的 handleMessage 方法就会被调用。 

注意： Looper 是运行在创建 Handler 所在的线程中的，这样一来，Handler 中的业务逻辑就被切换到创建 Handler 所在的线程中去执行了。
```

### Android的消息机制分析

> 下面将对 Android 消息机制的实现原理做一个较为全面的分析。由于 Android 的消息机制实际就是 Handler 的运行机制，所以，将主要围绕 Handler 的工作工程来分析 Android 的消息机制，主要包括: Handler、MessageQueue 和 Looper 。同时为了更好的理解 Looper 的工作原理，还会介绍一下 ThreadLocal。


#### ThreadLocal

* **先简单的说一下 ThreadLocal。**


```
Handler 的运行需要底层的 MessageQueue 和 Looper 的支撑。而 Looper 中有一个特殊的概念，那就是 ThreadLocal。

ThreadLocal 不是线程，它的作用是可以在每个线程中存储数据。我们知道，Handler 创建的时候会采用当前线程的 Looper 来构造消息循环系统。那么 Handler 内部是如何获取到当前线程的 Looper 呢？ 
这就要使用 ThreadLocal 了，ThreadLocal 可以在不同的线程中互不干扰地存储并提供数据，通过 ThreadLocal 可以轻松获取每个线程的 Looper 。需要注意的是，线程是默认没有 Looper 的，如果需要使用 Handler 就必须为线程创建 Looper。

而我们经常提到的主线程，也就是 UI 线程，它就是 ActivityThread ，ActivityThread 被创建时就会初始化 Looper ，这也就是在主线程中默认可以使用 Handler 的原因。
```

* **ThreadLocal的工作原理。**

> 接下来又是理论知识，恐怕会让看文章的你比较蒙圈或是感觉抽象，可以简单看看理解一下。


```
ThreadLocal 是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，存储以后，只有在指定线程中可以获取到存储的数据，而对于其他线程来说则无法获取到数据。

而平时的开发中，ThreadLocal使用的地方少之又少，但是在某些特殊的场景下，通过 ThreadLocal 可以轻松地实现一些看起来很复杂的功能，这点在 Android 源码中也有体现，比如：
Looper、ActrivityThread以及 AMS 中都用到了 ThreadLocal。

具体到 ThreadLocal 的使用场景，就不好统一描述。一般来说，当某些数据是以线程为作用域并且不同线程具有不同的数据副本的时候，就可以采用 ThreadLocal 。

比如 Handler ，它需要获取当前线程的 Looper ，而这个 Looper 的作用域就是线程并且不同线程具有不同的 Looper 。这个时候通过 ThreadLocal 就可以轻松实现 Looper 在线程中存取。

而如果不采用 ThreadLocal，那么系统就必须提供一个全局的哈希表共 Handler 查找指定线程的 Looper 。这样的话还必须提供一个类似于 LooperManager 的类了。

但是系统并没有这么做而是选择了 ThreadLocal ，这就是 ThreadLocal 的好处。

```

**ThreadLocal 还有一个使用场景是在复杂逻辑下的对象传递。此处就不在说明了。**

下面通过实际的例子来演示 ThreadLocal 的真正的含义。首先定义一个 ThreadLocal 对象，此处选择的一个 Boolean 类型的。如下：

```
 private ThreadLocal<Boolean> booleanThreadLocal = new ThreadLocal<Boolean>();
```

然后分别在主线程、子线程1和子线程2中设置和访问它的值，代码如下：


```

    mThreadLocal.set(true);
        Log.e(TAG, "[Thread:main]mThreadLocal = " + mThreadLocal.get());

        new Thread("Thread#1") {
            @Override
            public void run() {
                super.run();
                mThreadLocal.set(false);
                Log.e(TAG ,"[Thread#1]mThreadLocal = " + mThreadLocal.get());
            }
        }.start();

        new Thread("Thread#2") {
            @Override
            public void run() {
                super.run();
                Log.e(TAG ,"[Thread#2]mThreadLocal = " + mThreadLocal.get());
            }
        }.start();

```

在上面的代码中，我们在主线程中设置 mThreadLocal 的值为 true ，在子线程1中设置 mThreadLocal 的值为 false ，在子线程2中不设置 mThreadLocal 的值。然后分别在3个线程中通过 get() 方法获取 mThreadLocal 的值，根据前面对 ThreadLocal 的描述，这个时候，主线程中应该是 true 子线程1中应该是 false ，而子线程2中由于没有设置值，应该是 null ，运行代码，打印日志如下：


```
E/JiaYang: [Thread:main]mThreadLocal = true
E/JiaYang: [Thread#1]mThreadLocal = false
E/JiaYang: [Thread#2]mThreadLocal = null
```

从上面的日志可以看出，虽然是在不同的线程中访问的是同一个 ThreadLocal 对象，但是它们 通过 ThreadLocal 获取到的值却是不一样的，这也就是 ThreadLocal 的奇妙之处。

ThreadLocal 之所以有这么奇妙的效果，是因为不同线程访问同一个 ThreadLocal 的 get 方法，ThreadLocal 内部会从各自的线程中取出一个数组，然后再从数组中根据当前 ThreadLocal 的索引去查找出对应的 value 的值。很显然，不同线程中的数组是不同的，这就是 ThreadLocal 可以在不同的线程中维护一套数据的副本并且彼此互不干扰。


* **简单的说了 ThreadLocal 的使用方法和工作过程后，接下里简单分析 ThreadLocal 内部实现。**

ThreadLocal 是一个泛型类，其定义为 public class ThreadLocal<T> ，只要弄清楚 ThreadLocal 的 get 和 set 方法就可以明白它的工作原理。

首先看 ThreadLocal 的 set 方法，如下(源码：Android -21)：

```
public void set(T value) {
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values == null) {
            values = initializeValues(currentThread);
        }
        values.put(this, value);
    }
```

在上面的 set 方法中，首先会通过 values 方法来获取当前线程中的 ThreadLocal 数据，其获取的方式也是很简单的，在 Thread 类的内部有一个成员专门用于存储线程的 ThreadLocal 的数据： ThreadLocal.Values localValues ，因此获取当前线程的 ThreadLocal 数据就显得异常简单了。

如果 localValues 的值为 null ，那么就需要对其进行初始化，初始化后再将 ThreadLocal 的值进行存储。

接下来看一下 ThreadLocal 的值是如何在 localValues 中进行存储的。

```
在localValues 内部有一个数组：private Object[] table，ThreadLocal 的值就存储在这个 table 的数组中。
```
下面看一下 localValues 是如何使用 put 方法将 ThreadLocal 的值存储到 table 数组中的，如下：


```
        /**
         * Sets entry for given ThreadLocal to given value, creating an
         * entry if necessary.
         */
        void put(ThreadLocal<?> key, Object value) {
            cleanUp();

            // Keep track of first tombstone. That's where we want to go back
            // and add an entry if necessary.
            int firstTombstone = -1;

            for (int index = key.hash & mask;; index = next(index)) {
                Object k = table[index];

                if (k == key.reference) {
                    // Replace existing entry.
                    table[index + 1] = value;
                    return;
                }

                if (k == null) {
                    if (firstTombstone == -1) {
                        // Fill in null slot.
                        table[index] = key.reference;
                        table[index + 1] = value;
                        size++;
                        return;
                    }

                    // Go back and replace first tombstone.
                    table[firstTombstone] = key.reference;
                    table[firstTombstone + 1] = value;
                    tombstones--;
                    size++;
                    return;
                }

                // Remember first tombstone.
                if (firstTombstone == -1 && k == TOMBSTONE) {
                    firstTombstone = index;
                }
            }
        }

```

上面的 put 方法实现了数据的存储过程。此处不去分析其具体算法，但是我们可以得出一个存储规则。那就是：**ThreadLocal 的值在 table 数组中的存储位置总是 ThreadLocal 的 reference 字段所标识的对象的下一个位置**，比如 ThreadLocal 的 reference 对象在 table 数组中的索引为 index ，那么 ThreadLocal 的值在 table 数组中的索引就是 index + 1。**最终 ThreadLocal 的值将会被存储到 table 数组中：table[index + 1]= value 。**


***

上面简单的分析了 ThreadLocal 的 set 方法，下面说一下 get 方法，如下：


```
/**
     * Returns the value of this variable for the current thread. If an entry
     * doesn't yet exist for this variable on this thread, this method will
     * create an entry, populating the value with the result of
     * {@link #initialValue()}.
     *
     * @return the current value of the variable for the calling thread.
     */
    @SuppressWarnings("unchecked")
    public T get() {
        // Optimized for the fast path.
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values != null) {
            Object[] table = values.table;
            int index = hash & values.mask;
            if (this.reference == table[index]) {
                return (T) table[index + 1];
            }
        } else {
            values = initializeValues(currentThread);
        }

        return (T) values.getAfterMiss(this);
    }
```

可以发现， ThreadLocal 的 get 方法的逻辑比较清晰，它同样是取出当前线程的 local-Values 对象，如果这个对象为 null 那么就返回初始值，初始值由 ThreadLocal 的 initialValue 方法来描述，默认情况下为 null ，当然也可以重写这个方法，它的默认实现如下所示：


```
    /**
     * Provides the initial value of this variable for the current thread.
     * The default implementation returns {@code null}.
     *
     * @return the initial value of the variable.
     */
    protected T initialValue() {
        return null;
    }
```
如果 localValues 对象不为 null ，那就取出它的 table 数组并找出 ThreadLocal 的 reference 对象在 table 数组中的位置，然后 table 数组中的下一个位置所存储的数据就是 ThreadLocal 的值。

* 总结：从 ThreadLocal 的 set 和 get 方法可以看出，它们所操作的对象都是当前线程的 localValues 对象的 table 数组，因此在不同线程中访问同一个 ThreadLocal 的 set 和 get 方法，它们对 ThreadLocal 所做的读、写操作仅限于各自线程的内部，这就是为什么 ThreadLocal 可以在多个线程中互不干扰地存储和修改数据，理解 ThreadLocal 的实现方式有助于理解 Looper 的工作原理。


#### MessageQueue消息队列的工作原理

> Android 中的消息队列指的是 MessageQueue 。MessageQueue 主要包含两个操作：插入和读取。读取操作本身会伴随着删除操作，插入和读取对应的方法分别为：enqueueMessage 和 next 。

> enqueueMessage 的作用是往消息队列中插入一条消息，而 next 的作用是从消息队列中取出一条消息并将其从消息队列中移除。

> MessageQueue 尽管被称之为消息队列，但是它的内部实现并不是用的队列，实际上它是通过一个 单链表的数据结构来维护消息列表，单链表在插入和删除上比较有优势。下面看一下 enqueueMessage 和 next 方法的实现。

* **enqueueMessage 源码如下：**


```
boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
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
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
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
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
从 enqueueMessage 的实现可以看出，它的主要操作其实就是单链表的插入操作。此处不过多的解释。

接下来看一下 next 方法的实现， next 的主要逻辑如下：


```
Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
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

                ...
            }

            ...
        }
    }
```

可以发现 next 方法是一个无限循环的方法，如果消息队列中没有消息，那么 next 方法会一直阻塞在这里，只有当有新消息到来时，next 方法会返回这条消息并将其从单链表中移除。



#### Looper的工作原理

> Looper 在 Android 的消息机制中扮演着消息循环的角色，具体来说就是它会不停地从 MessageQueue 中查看是否有新消息，如果有新消息就会立刻处理，否则就会一直阻塞在那里。

首先，来看一下 Looper 的构造方法：


```
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

在其构造方法中会创建一个 MessageQueue 消息队列，然后将当前线程的对象保存起来。

* 我们知道 Handler 的工作需要 Looper，但是没有 Looper 的线程就会报错，那么如何创建呢？其实也很简单，通过 Looper.prepare() 方法即可为当前线程创建一个 Looper ，接着通过 Looper.loop()来开启消息循环。如下：



```
new Thread("Thread#1"){
            @Override
            public void run() {
                Looper.prepare();
                Handler mHandler = new Handler();
                
                Looper.loop();
            }
        }.start();
```

估计会疑惑 此时 Handler 最后在 handleMessage 或者 Runnable run 方法下是在哪个线程？我们以post 方法为例：


```
new Thread("Thread#1"){
            @Override
            public void run() {
                Looper.prepare();
                Handler mHandler = new Handler();
                mHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        Log.e("JiaYang", "HandlerThread:" + Thread.currentThread());
                    }
                });
                Looper.loop();
            }
        }.start();
```

运行代码，打印日志：


```
E/JiaYang: HandlerThread:Thread[Thread#1,5,main]
```
看到这 估计会说：你这是在Activity 下写的，默认已经有Looper ，那好，我在一个外部的 Adapter 下写一个 子线程：


```
public class MainAdapter extends BaseAdapter {

    private final String[] mStrings;
    private final Activity mActivity;

    public MainAdapter(String[] strings, Activity context) {
        mActivity = context;
        mStrings = strings;

        new Thread("Thread#1"){
            @Override
            public void run() {
                Looper.prepare();
                Handler mHandler = new Handler();
                mHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        Log.e("JiaYang", "HandlerThread:" + Thread.currentThread());
                    }
                });
                Looper.loop();
            }
        }.start();
    }
    ...
}
```

此 Adapter 是MainActivity 下的一个 Adapter ，运行代码打印日志：

```
E/JiaYang: HandlerThread:Thread[Thread#1,5,main]
```

* Looper 除了 prepare 方法外，还提供了 prepareMainLooper方法，此方法主要是给主线程也就是 ActivityThread 创建 Looper 使用的，其本质也是通过 prepare 方法来实现的。由于主线程的 Looper 比较特殊，所以 Looper 提供了一个 getMainLoopter 方法，通过这个方法可以在任何地方获取到主线程的 Looper 。

Looper 也是可以退出的， Looper 提供了 quit 和 quitSafely 方法来退出一个 Looper 。二者的区别是：

* quit：会直接退出 Looper 。

* quitSafely：只是设定一个退出标记，然后把消息队列中的已有消息处理完毕后才完全地退出。

Looper 退出后，通过 Handler 发送的消息会失败，此时，Handler 的 send 方法会返回 false。在子线程中，如果手动为其创建了 Looper ,那在所有的事情完成以后应该会调用  quit 方法来终止消息循环。否则，这个子线程就会一直处于等待状态，而如果退出 Looper 后，这个线程就会立刻终止，因此，建议不需要的时候终止 Looper 。


* **Looper 中最重要的一个方法就是 loop 方法，只有调用了 loop 以后，消息循环系统才会真正的起作用，源码如下：**


```
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

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
```

Looper 的 loop 方法的工作过程也很好理解。loop 方法就是一个死循环，唯一跳出死循环的方式是：MessageQueue 的 next  方法返回了 null 。

当 Looper 的 quit方法被调用时，Looper就会调用 MessageQueue 的 quit 或者 quitSafely 方法来通知消息队列退出，当消息队列被标记为退出状态时，它的 next 方法就会返回 null。也就是说：**Looper必须退出，否则 loop方法会一直无线循环下去。loop方法会调用 MessageQueue的next 方法来获取新消息，而next是一个阻塞操作，当没有消息的时候，next 方法会一直阻塞在那里，这也会导致 loop 方法一直阻塞在那里。**

如果 MessageQueue 的 next 方法返回了新消息，Looper就会处理这条消息：msg.target.dispatchMessage(msg) ，这里的 msg.target 是发送这条消息的 Handler 对象，这样，Handler 发送的消息最终又交给它的 dispatchMessage方法来处理了。

但是不同的是：Handler 的 dispatchMessage 方法是在创建 Handler 时所使用的 Looper 中执行的，这也就成功的将代码逻辑切换到指定的线程中去执行了。



#### Handler 的工作原理

> Handler 的工作主要包含消息的发送和接收过程。消息的发送可以通过 post 的一系列方法 以及 send 方法来实现，post 的一系列方法最终是通过 send 的一系列方法来实现的。

发送一条消息的典型过程源码如下：


```
    public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
    
    
    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
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

可以发现， Hanlder 发送消息的过程 仅仅是向消息队列中插入了一条消息， MessagerQueue 的 next 方法就会返回这个消息给 Looper， Looper 接收到消息后就开始处理，最终消息由 Looper 交由 Handler 处理，即 Handler 的 dispatchMessage 方法会被调用，这时， Handler 就进入了处理消息的阶段。

* dispatchMessager 的实现源码如下：


```
    /**
     * Handle system messages here.
     */
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
* Handler 处理消息的过程如下：
    * 首先，检查 Message 的 callback 是否为 null ，不为null 就通过 handleCallback 来处理消息。Message 的 callback 是一个 Runnable 对象，实际上就是 Handler 的 post 方法所传递的  Runnable 参数。handleCallback 的逻辑也很简单，源码如下：
```
    private static void handleCallback(Message message) {
        message.callback.run();
    }
```

*
    * 其次，检查 mCallback 是否为 null，不为 null 就调用 mCallback 的 handleMessage 方法来处理消息， Callback 是一个接口，定义如下：
    
```
    /**
     * Callback interface you can use when instantiating a Handler to avoid
     * having to implement your own subclass of Handler.
     *
     * @param msg A {@link android.os.Message Message} object
     * @return True if no further handling is desired
     */
    public interface Callback {
        public boolean handleMessage(Message msg);
    }
```

通过  Callback 可以采用如下方法来创建 Handler 对象： Handler handler = new Handler(callback) 。那么 Callback 的意义又是什么呢？

源码中的注释已经作了说明：可以用来创建一个 Handler 的实例但并不需要派生 Handeler 的子类。

而我们在平时开发的过程中最常见的方式就是派生一个 Handler 的子类并重写其 handleMessage 方法来处理具体的消息。Callback 只是给我们提供了另外一种使用  Handler 的方式。当我们不想派生子类的时候，就可以通过 Callback 来实现。

*
    * 最后，调用 Handler 的 handleMessage 方法来处理消息。Handler 处理消息的过程可以归纳为一个流程图(未用思维导图作图，凑合看吧)：
    
![Handler](http://m.qpic.cn/psb?/V14YlNrL2eQEkW/v2kr4AE9TOjw.C1ZqGeCBF.2NbDLMW*nFes*l7uDrtU!/b/dPMAAAAAAAAA&bo=8AM5A*ADOQMDByI!&rf=viewer_4)

***
Handler 还有一个特殊的构造方法，就是通过一个特定的 Looper 来构造 Handler ，通过这个构造方法可以实现一些特殊的功能。具体实现如下：

```
    /**
     * Use the provided {@link Looper} instead of the default one.
     *
     * @param looper The looper, must not be null.
     */
    public Handler(Looper looper) {
        this(looper, null, false);
    }
```

***
我们来看一下 Handler 的默认构造方法 public Handler()，这个构造方法会调用下面这个构造方法：


```
public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

通过这个构造可以看出，如果线程没有 Looper 的话，就会抛出***Can't create handler inside thread that has not called Looper.prepare()***这个异常，这也就说明了为什么在没有 Looper 子线程中创建 Handler 会引发程序异常的原因。



***

**写在最后：**

* 说了这么多，估计还是晕头转向的，最后，用自己的话总结一下吧：


* 使用 sendMessage() 发送消息后，通过 Hanlder 将消息发送给 消息队列 MessageQueue 。

* 在 Handler 发送消息的时候，使用 message.target = this 为 Handelr 发送的消息 Message 贴上当前 handler 一个标签。

* MessageQueue 调用 enqueueMessage 方法，将消息插入到消息队列中。

* 消息队列中发现有新消息的到来时，MessageQueue 的内部走 next 方法无限循环。而Looper 发现有新消息到来，就会无限循环消息队列，也就是 Looper.loop 方法。

* 在MessageQueue next 方法返回消息并将消息从消息队列中删除后，Looper.loop 此时获得 MessageQueue 对象，从中取出 next 返回的消息。当 MessageQueue next 方法返回 null也就是没有消息 的时候，next 会一直阻塞在那里，而Looper.loop 方法就会停止，并也阻塞在那里。

* Loop.loop 从消息队列中取出消息后，会调用 msg.target.dispatchMessage(msg) 进行消息分发。

* 此时 msg.target 就是刚才发送消息的 handler 对象，最后等于又调用了 handler 的 dispatchMessage(msg) 方法。(而 dispatchMessage(msg) 上面最后也说了实现原理。)

* 在创建 handler 的时候 覆写的方法 handleMessage ，对分发的消息进行了处理。

* 最后，在消息使用完毕后，在 Looper.loop 方法中调用了 msg.recyclerUnchecked() 方法，将消息回收。即将消息的所有字段恢复为初始状态。
