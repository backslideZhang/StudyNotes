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
通过上述代码可以直接发现，返回值就是判断，类中成员变量thread是否和当前线程一致。
也就是调用execute方法的线程是否是这个类执行线程。如果当前执行线程