# Netty 源码分析：EventLoopGroup
## 类图
![class diagram](https://raw.githubusercontent.com/backslideZhang/pictures/master/netty/netty-event-loop-group.png)
从类图中可以看出， 整个继承链可以分为两块，一块是EventLoop所在的部分，对应single-thread模型。
一块是EventLoopGroup所在的部分，对应multi-thread模型。
## Single-Thread 模型
### SingleThreadEventLoop  
这个类主要实现了EventLoop接口和EventLoopGroup接口中规定的一系列方法，还实现了一个wakesUpForTask
方法，暂时没什么要说的。
### SingleThreadEventExecutor
这个类主要维护一个task queue。最最关键的是，这个类实现了Executor接口中的execute方法。实现代码如下：
``` java
    public void execute(Runnable task) {
        if(task == null) {
            throw new NullPointerException("task");
        } else {
            boolean inEventLoop = this.inEventLoop();
            if(inEventLoop) {
                this.addTask(task);
            } else {
                this.startThread();
                this.addTask(task);
                if(this.isShutdown() && this.removeTask(task)) {
                    reject();
                }
            }
            if(!this.addTaskWakesUp && this.wakesUpForTask(task)) {
                this.wakeup(inEventLoop);
            }
        }
    }
```
先看看inEventLoop这个方法究竟干了什么。
``` java
    //类AbstractEventExecutor
    public boolean inEventLoop() {
        return this.inEventLoop(Thread.currentThread());
    }
    //类SingleThreadEventExecutor
    private volatile Thread thread;
    public boolean inEventLoop(Thread thread) {
        return thread == this.thread;
    }
```
通过上述代码可以直接发现，这个方法就是判断类中成员变量thread是否和当前线程一致。也就是调用execute
方法的线程是否是这个类执行线程。如果当前执行线程是那个single-thread，则将添加任务，否则先启动线程，
然后在执行添加任务操作。当executor的状态处于shutdown时，需要reject任务。对于wakeup操作，实现代码如
下：
``` java
    protected void wakeup(boolean inEventLoop) {
        if(!inEventLoop || STATE_UPDATER.get(this) == 3) {
            this.taskQueue.add(WAKEUP_TASK);
        }
    }
```
其中，WAKEUP_TASK是一个空Runnable对象。对于添加任务的操作，下面是具体代码：
``` java
    protected void addTask(Runnable task) {
        if(task == null) {
            throw new NullPointerException("task");
        } else {
            if(this.isShutdown()) {
                reject();
            }
            this.taskQueue.add(task);
        }
    }
```
其中，taskQueue的初始化过程如下：
``` java
    protected Queue<Runnable> newTaskQueue() {
        return new LinkedBlockingQueue();
    }
```
所以，addTask过程就是直接向任务队列中添加一个task。下面来看看最最复杂的startThread方法。从字面上意思，就
是启动那个single-thread。直接上代码：
``` java
    private void startThread() {
        if(STATE_UPDATER.get(this) == 1 && STATE_UPDATER.compareAndSet(this, 1, 2)) {
            this.doStartThread();
        }
    }
```
首先是一个if判断，目的是保证当前的executor处于未开始状态。然后进入startThread的主体：doStartThread方法：
``` java
private void doStartThread() {
        assert this.thread == null;
        this.executor.execute(new Runnable() {
            public void run() {...}
        });
    }
```
整个方法调用成员变量executor来执行，executor变量是在类构造函数中传入的变量，用于初始化该类实例。下面是简化
的Runnable实现：
``` java
    public void run() {
        //记录当前thread作为single-thread
        SingleThreadEventExecutor.this.thread = Thread.currentThread();
        label1686: {
            try {
                SingleThreadEventExecutor.this.run();
                //一切顺利，则退出整个label
                break label1686;
            } catch (Throwable ...) {
            } finally {
                //进入catch，start失败，恢复现场
                if(...) {
                    //首先尝试shutdown，标记状态为正在shutdown
                    int oldState1;
                    do {
                        oldState1 = SingleThreadEventExecutor.STATE_UPDATER.get(SingleThreadEventExecutor.this);
                    } while(oldState1 < 3 && !SingleThreadEventExecutor.STATE_UPDATER.compareAndSet(SingleThreadEventExecutor.this, oldState1, 3));
                    try {
                        //等待shutdown完成，confirmShutdown方法完成shutdown过程中，SingleThreadEventExecutor的善后工作
                        while(!SingleThreadEventExecutor.this.confirmShutdown()) {
                            ;
                        }
                    } finally {
                        try {
                            //shutdown完成，对应run方法，做一些清理工作，由实现run方法的子类实现
                            SingleThreadEventExecutor.this.cleanup();
                        } finally {
                            //标记为terminate，threadLock用于awaitTerminate中，此处release用于表示terminate结束。
                            SingleThreadEventExecutor.STATE_UPDATER.set(SingleThreadEventExecutor.this, 5);
                            SingleThreadEventExecutor.this.threadLock.release();
                            //terminate完成
                            SingleThreadEventExecutor.this.terminationFuture.setSuccess((Object)null);
                        }
                    }
                }
            }
            //源码又再次进行了一次恢复现场的工作不知道为什么。。。
        }
        //源码又又又再次进行了一次恢复现场的工作不知道为什么。。。
    }
```
说实话，上面代码看着很不舒服……总的来说，就是得到那个single-thread，然后调用子类run方法。不论run方法如何结
束，都开始善后工作。所以，整个SingleThreadEventExecutor主要工作在于，提供了一个用于维护tasks的队列。（阿西吧！
折腾了半天，就是一个队列而已，擦擦擦！）
### AbstractScheduledEventExecutor
