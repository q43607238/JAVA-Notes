[TOC]

# Excutors (ThreadPool)

## 简介

在项目中，会通过创建线程池来实现一些具体的数据拉取任务，相比于for循环方式的顺序拉取，能够有效保证中途拉取数据出现异常时，不会影响到后续的绝大多数数据。

## 创建一个线程池

在项目中的`TaskBatchExecutor`类（其继承了`Thread`类）里面有如下的一个私有变量，表示为在当前线程下的一个线程池对象。

```java
private ThreadPoolExecutor executorService;
```

其专门为这一个`TaskBatchExecutor`负责，而一个`TaskBatchExecutor`会为一个单独的`TaskBatch`负责。因此，**在`TaskBatchExecutor`中重写的run方法内，其主要任务就是检测是否还有任务没有完成，并不断往线程池里面推送任务。**

如下，为`TaskBatchExecutor`的构造函数：

```java
    public TaskBatchExecutor(TaskService taskService, TaskBatch batch) {
        this.taskService = taskService;
        this.batch = batch;
        int poolNum = poolNumber.getAndIncrement();
        executorService = (ThreadPoolExecutor) Executors
            //在这里，调用了newFixedThreadPool来创建一个新的线程池
                .newFixedThreadPool(batch.getWorkerCount(), new ThreadFactory() {
                    AtomicInteger threadNumer = new AtomicInteger(1);

                    @Override
                    public Thread newThread(Runnable r) {
                        return new Thread(r,
                                String.format("batchTask-pool%d-t%d", poolNum,
                                        threadNumer.getAndIncrement()));
                    }
                });
    }
```



### newFixedThreadPool()

可以看到，`newFixedThreadPool`实际上调用的是`ThreadPoolExecutor`方法。

```java
    public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
```

通过更改`ThreadPoolExecutor`方法里的参数，我们可以实现自定义线程池。



### ThreadPoolExecutor()

上一部分调用了`ThreadPoolExecutor`的构造函数之一，如下：

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             threadFactory, defaultHandler);
    }
```

其一共有几个参数：

* `corePoolSize`核心线程数量，这些核心线程一旦创建，在没有任务的时候，**线程也会保证一直存在**
* `maximumPoolSize`允许在线程池内同时存在的线程最大数量，即**非核心线程+核心线程**
* `keepAliveTime`这是**非核心线程**在空闲时候的存活时间
* `unit`是存活时间参数的单位
* `workQueue`是任务队列，用来递交过来但还没来得及执行的任务
* `threadFactory`线程工厂

可以看到在方法内又重新调用了另一个构造函数，并且加入了一个参数`defaultHandler`。

* `defaultHandler`参数用来**决定任务由于线程限制或者任务队列容量不够时的处理方式**

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

## 四种handler

### AbortPolicy

这是一个**默认**的`handler`，其在出现任务被拒绝的时候，抛出异常

```java
    /**
     * A handler for rejected tasks that throws a
     * {@code RejectedExecutionException}.
     */
    public static class AbortPolicy implements RejectedExecutionHandler {
        /**
         * Creates an {@code AbortPolicy}.
         */
        public AbortPolicy() { }

        /**
         * Always throws RejectedExecutionException.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         * @throws RejectedExecutionException always
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }
```

### CallerRunsPolicy

对于到来的任务，一直尝试执行；

```java
    /**
     * A handler for rejected tasks that runs the rejected task
     * directly in the calling thread of the {@code execute} method,
     * unless the executor has been shut down, in which case the task
     * is discarded.
     */
    public static class CallerRunsPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code CallerRunsPolicy}.
         */
        public CallerRunsPolicy() { }

        /**
         * Executes task r in the caller's thread, unless the executor
         * has been shut down, in which case the task is discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }
```

### DiscardPolicy

对于任何被拒绝的任务，**直接丢弃。**

```java
    /**
     * A handler for rejected tasks that silently discards the
     * rejected task.
     */
    public static class DiscardPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardPolicy}.
         */
        public DiscardPolicy() { }

        /**
         * Does nothing, which has the effect of discarding task r.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }
```

### DiscardOldestPolicy

当有任务被拒绝的时候，**丢弃等待队列中最老的任务。**

```java
    /**
     * A handler for rejected tasks that discards the oldest unhandled
     * request and then retries {@code execute}, unless the executor
     * is shut down, in which case the task is discarded.
     */
    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardOldestPolicy} for the given executor.
         */
        public DiscardOldestPolicy() { }

        /**
         * Obtains and ignores the next task that the executor
         * would otherwise execute, if one is immediately available,
         * and then retries execution of task r, unless the executor
         * is shut down, in which case task r is instead discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```

## 线程池执行

### execute()

结合源码注释，可以很清楚的明白`execute`的功能：

* 将command任务交付给线程池
* 任务会被一个新的线程或者已经在线程池中存在的空闲线程执行
* 如果说这个任务无法被执行，则会按照事先定义好的`handler`处理



一个任务递交到线程池的流程：

1. 当一个任务被递交到一个全新的线程池，**首先会创建核心线程来执行任务，直到核心线程到达数量限制。**
2. 当核心线程数到限制数量后，继续进来的任务就会被放到`workQueue`中，等待执行。
3. 如果`workQueue`被放满了，就尝试创建新的**非核心线程**来执行任务。

```java
	public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        
        //ctl是一个AtomicInteger，c可以表征这个线程池的状态，也可以通过一系列运算获得workCount数量等信息
        int c = ctl.get();
        //通过workerCountOf()方法，获得了核心线程数，如果发现小于我们设定的核心线程数量，则增加一个worker
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //如果在线程数大于核心线程数，先判断线程池是不是RNNING状态，其次，看是否可以往工作队列插入一个task
        if (isRunning(c) && workQueue.offer(command)) {
            //如果可以，我们还需要进行二次确认，避免线程池其余线程的任务变化导致的线程池变化
            int recheck = ctl.get();
            //如果线程池不在是RUNNING状态并且我们可以从WorkQueue中移除task，就直接reject
            if (! isRunning(recheck) && remove(command))
                reject(command);
            //否则说明线程池可能是RUNNING状态，并且无法移除task，就判断线程数是否为0，若是的话则创建一个非核心线程让他去拿WorkQueue中的任务
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //如果线程池不是RUNNING状态或者无法插入task，就尝试直接用一个非核心线程直接执行该task
        else if (!addWorker(command, false))
            reject(command);
    }
```

### ctl

`ctl`的主要作用是记录线程池的生命周期状态和当前工作的线程数。作者通过巧妙的设计，**将一个整型变量按二进制位分成两部分，分别表示两个信息。**

```java
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
	//两个工具常量，其中Integer.SIZE为32
    private static final int COUNT_BITS = Integer.SIZE - 3; //实际值为29
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
	//CAPACITY的值为1左移了29位并减1，从二进制的角度来看，其可以理解为：
	//0000 0000 0000 0001
	//1 << 29 - 1
	//0001 1111 1111 1111 即32，31，30的高位为0，其余都为1

	//在接下来的代码中，COUNT_BITS用来分隔runState和workCount的位数；而CAPACITY作为取这两个变量的工具。

    //线程池的状态有5个，所以COUNT_BITS需要减3
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    //这样的话就非常好理解了，整个二进制数字的高位三位是runState，其通过与CAPACITY的反码做与计算得来
	//与之对应的，后29位就是wokerCount的数量了
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```

### 线程池状态切换

1. RUNNING：能够接收新任务，以及对已添加的任务进行处理。线程池被一旦被创建，就处于RUNNING状态，并且线程池中的任务数为0。
2.  SHUTDOWN：线程池处在SHUTDOWN状态时，**不接收新任务，但能处理已添加的任务。**
3. STOP：线程池处在STOP状态时，**不接收新任务，不处理已添加的任务，并且会中断正在处理的任务。**
4. TIDYING：当所有的任务已终止，ctl记录的”任务数量”为0，线程池会变为TIDYING状态。当线程池变为TIDYING状态时，会执行钩子函数terminated()。terminated()在ThreadPoolExecutor类中是空的，若用户想在线程池变为TIDYING时，进行相应的处理；可以通过重载terminated()函数来实现。
5. TERMINATED：**线程池彻底终止**，就变成TERMINATED状态。



这里可以顺带复习一下**线程的生命周期**：

1. 新建状态：使用 new 关键字和 Thread 类或其子类建立一个线程对象后，该线程对象就处于新建状态。它保持这个状态直到程序 start() 这个线程。
2. 就绪状态：当线程对象调用了start()方法之后，该线程就进入就绪状态。就绪状态的线程处于就绪队列中，要等待JVM里线程调度器的调度。
3. 运行状态：如果就绪状态的线程获取 CPU 资源，就可以执行 run()，此时线程便处于运行状态。处于运行状态的线程最为复杂，它可以变为阻塞状态、就绪状态和死亡状态。
4. 阻塞状态：如果一个线程执行了sleep（睡眠）、suspend（挂起）等方法，失去所占用资源之后，该线程就从运行状态进入阻塞状态。在睡眠时间已到或获得设备资源后可以重新进入就绪状态。可以分为三种：
   * 等待阻塞：运行状态中的线程执行 wait() 方法，使线程进入到等待阻塞状态。
   * 同步阻塞：线程在获取 synchronized同步锁失败(因为同步锁被其他线程占用)。 
   * 其他阻塞：通过调用线程的 sleep() 或 join() 发出了 I/O请求时，线程就会进入到阻塞状态。当sleep() 状态超时，join() 等待线程终止或超时，或者 I/O 处理完毕，线程重新转入就绪状态。
5. 死亡状态：一个运行状态的线程完成任务或者其他终止条件发生时，该线程就切换到终止状态。

### addWorker()

当增加一个worker成功的时候返回true，参数中的`firstTask`是在`execute()`中传入的`Runnable command`参数。

```java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 一个最基本的判断条件就是，如果rs是RUNNING状态即rs >= SHUTDOWN(false)时，会直接跳过这个if。其次，若rs >= SHUTDOWN(true)，则需要保证rs == SHUTDOWN，firstTask == null，!workQueue.isEmpty()三个条件均满足，否则就return false；
            // 
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    //如果当前worker线程的数量大于等于核心线程数量（core=true），或者最大数量（core=false），返回true
                    return false;
                	//采用CAS操作让workerCount增加1
                if (compareAndIncrementWorkerCount(c))
                    //如果返回了true，说明增加成功了，直接跳出循环
                    break retry;
                
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    //如果状态变化了，调到外循环重新判断当前runState
                    continue retry;
                //否则就是CAS操作失败了，重新进行内循环即可
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            //构建一个Worker对象
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        //workers是一个HashSet，存储了这个线程池内的所有worker
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            //更新线程池的size
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    //启动和Worker绑定的线程，调用了Worker的run方法，进而调用了runWorker(this)，进而调用了我们task的run方法(其中就是我们真正要执行的任务)；
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                //如果线程启动失败
                addWorkerFailed(w);
        }
        return workerStarted;
    }
------------------------------------------------------------------------------------------------------------
		//Worker对象的构造函数，其实就是把task绑定并且用ThreadFactory创建了一个新的线程，并且把自己绑到了这个线程上，为啥呢么可以这么做？因为Worker也实现了Runnable接口并重写了run方法
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
        public void run() {
            //在这个方法中，会执行task的run()方法
            runWorker(this);
        }
```

​	





