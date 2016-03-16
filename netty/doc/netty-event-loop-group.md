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
ͨ�������������ֱ�ӷ��֣�������������ж����г�Ա����thread�Ƿ�͵�ǰ�߳�һ�¡�Ҳ���ǵ���execute
�������߳��Ƿ��������ִ���̡߳������ǰִ���߳����Ǹ�single-thread����������񣬷����������̣߳�
Ȼ����ִ����������������executor��״̬����shutdownʱ����Ҫreject���񡣶���wakeup������ʵ�ִ�����
�£�
``` java
    protected void wakeup(boolean inEventLoop) {
        if(!inEventLoop || STATE_UPDATER.get(this) == 3) {
            this.taskQueue.add(WAKEUP_TASK);
        }
    }
```
���У�WAKEUP_TASK��һ����Runnable���󡣶����������Ĳ����������Ǿ�����룺
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
���У�taskQueue�ĳ�ʼ���������£�
``` java
    protected Queue<Runnable> newTaskQueue() {
        return new LinkedBlockingQueue();
    }
```
���ԣ�addTask���̾���ֱ����������������һ��task����������������ӵ�startThread����������������˼����
�������Ǹ�single-thread��ֱ���ϴ��룺
``` java
    private void startThread() {
        if(STATE_UPDATER.get(this) == 1 && STATE_UPDATER.compareAndSet(this, 1, 2)) {
            this.doStartThread();
        }
    }
```
������һ��if�жϣ�Ŀ���Ǳ�֤��ǰ��executor����δ��ʼ״̬��Ȼ�����startThread�����壺doStartThread������
``` java
private void doStartThread() {
        assert this.thread == null;
        this.executor.execute(new Runnable() {
            public void run() {...}
        });
    }
```
�����������ó�Ա����executor��ִ�У�executor���������๹�캯���д���ı��������ڳ�ʼ������ʵ���������Ǽ�
��Runnableʵ�֣�
``` java
    public void run() {
        //��¼��ǰthread��Ϊsingle-thread
        SingleThreadEventExecutor.this.thread = Thread.currentThread();
        label1686: {
            try {
                SingleThreadEventExecutor.this.run();
                //һ��˳�������˳�����label
                break label1686;
            } catch (Throwable ...) {
            } finally {
                //����catch��startʧ�ܣ��ָ��ֳ�
                if(...) {
                    //���ȳ���shutdown�����״̬Ϊ����shutdown
                    int oldState1;
                    do {
                        oldState1 = SingleThreadEventExecutor.STATE_UPDATER.get(SingleThreadEventExecutor.this);
                    } while(oldState1 < 3 && !SingleThreadEventExecutor.STATE_UPDATER.compareAndSet(SingleThreadEventExecutor.this, oldState1, 3));
                    try {
                        //�ȴ�shutdown��ɣ�confirmShutdown�������shutdown�����У�SingleThreadEventExecutor���ƺ���
                        while(!SingleThreadEventExecutor.this.confirmShutdown()) {
                            ;
                        }
                    } finally {
                        try {
                            //shutdown��ɣ���Ӧrun��������һЩ����������ʵ��run����������ʵ��
                            SingleThreadEventExecutor.this.cleanup();
                        } finally {
                            //���Ϊterminate��threadLock����awaitTerminate�У��˴�release���ڱ�ʾterminate������
                            SingleThreadEventExecutor.STATE_UPDATER.set(SingleThreadEventExecutor.this, 5);
                            SingleThreadEventExecutor.this.threadLock.release();
                            //terminate���
                            SingleThreadEventExecutor.this.terminationFuture.setSuccess((Object)null);
                        }
                    }
                }
            }
            //Դ�����ٴν�����һ�λָ��ֳ��Ĺ�����֪��Ϊʲô������
        }
        //Դ���������ٴν�����һ�λָ��ֳ��Ĺ�����֪��Ϊʲô������
    }
```
˵ʵ����������뿴�źܲ���������ܵ���˵�����ǵõ��Ǹ�single-thread��Ȼ���������run����������run������ν�
��������ʼ�ƺ��������ԣ�����SingleThreadEventExecutor��Ҫ�������ڣ��ṩ��һ������ά��tasks�Ķ��С��������ɣ�
�����˰��죬����һ�����ж��ѣ�����������
### AbstractScheduledEventExecutor
