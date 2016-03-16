# Netty source code analysis
## Class Diagram
![class diagram](https://raw.githubusercontent.com/backslideZhang/pictures/master/netty/EventLoop.png)
From the Class Diagram, we can find that the composition of EventLoop can separate into
2 parts : single-thread model &amp; multi-thread model.
## Single-Thread Model
The most important code in single-thread model is the 'execute' method in class
'SingleThreadEventLoop'
``` java
    public void execute(Runnable task) {
        if(task == null) {
            throw new NullPointerException("task");
        } else {
            boolean inEventLoop = this.inEventLoop();
            if(inEventLoop) {
                // invoke by internal thread
                this.addTask(task);
            } else {
                // invoke by
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
the code is very simple : it receive runnable task,