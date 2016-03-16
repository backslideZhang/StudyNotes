# Netty Դ�������EventLoopGroup
## ��ͼ
![class diagram](https://raw.githubusercontent.com/backslideZhang/pictures/master/netty/netty-event-loop-group.png)
����ͼ�п��Կ����� �����̳������Է�Ϊ���飬һ����EventLoop���ڵĲ��֣���Ӧsingle-threadģ�͡�
һ����EventLoopGroup���ڵĲ��֣���Ӧmulti-threadģ�͡�
## Single-Thread ģ��
### SingleThreadEventLoop  
�������Ҫʵ����EventLoop�ӿں�EventLoopGroup�ӿ��й涨��һϵ�з�������ʵ����һ��wakesUpForTask
��������ʱûʲôҪ˵�ġ�
### SingleThreadEventExecutor
�������Ҫά��һ��task queue������ؼ����ǣ������ʵ����Executor�ӿ��е�execute������ʵ�ִ������£�
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
�ȿ���inEventLoop���������������ʲô��
``` java
    //��AbstractEventExecutor
    public boolean inEventLoop() {
        return this.inEventLoop(Thread.currentThread());
    }
    //��SingleThreadEventExecutor
    private volatile Thread thread;
    public boolean inEventLoop(Thread thread) {
        return thread == this.thread;
    }
```
ͨ�������������ֱ�ӷ��֣�����ֵ�����жϣ����г�Ա����thread�Ƿ�͵�ǰ�߳�һ�¡�
Ҳ���ǵ���execute�������߳��Ƿ��������ִ���̡߳������ǰִ���߳�