[TOC]

#  Taskservice

## 简介

是Task处理的框架，主要有以下特性：

1. 为每一个**batch**自动启动一个线程池。并且启动对应的**Worker**；
2. 自动处理**Task**状态，支持系统重启后自动继续处理任务；
3. 可以根据**Task**的返回列表，继续生成更多的**Task**，并且交由**线程池**去处理；

## 项目中涉及的方法

### init()

1. 在项目初始化的时候进行初始化，调用`revertAllDownladingStatus()`将所有task表中还没开始的任务状态（0）都更改为正在下载（1）；
2. 其中用了`@PostConstruct`注解实现在这个`bean`被注入后，就自动初始化所有的任务，<span style='color:red'>实现系统重启后任务的自动开始</span>

```java
@PostConstruct
public void init() {
    log.info("init...change all task's status from downloading to not start ....");
    revertAllDownladingStatus();
}
private void revertAllDownladingStatus() {
    dao.execute(Sqls.create("update task set status=0 where status=1"));
}
```

### destroy()

在`taskService`这个bean被容器销毁前，因为`@PreDestroy`注解，执行`destroy()`，将所有正在进行的任务停止下来；

```java
@PreDestroy
public void destroy() {
    log.info("destroy...stop all task's executor ....");
    for (TaskBatchExecutor executor : executors.values()) {
        executor.stopTaskBatch();
    }
}
```

### createBatch(String title, int workerCount)

在我们递交任务之前，需要建立一个`batch`任务，以表示我们之后所有递交的任务，都是属于这一个大的`batch`下的任务；

在`createBatch(String title, int workerCount)`中，首先建造了一个`batch`，其次将这个`batch`任务记录在**task_batch**表中；

```java
public TaskBatch createBatch(String title, int workerCount) {
    TaskBatch batch = TaskBatch.builder().title(title)
            .workerCount(workerCount).taskCount(0).build();
    return dao.insert(batch, true, false, false);
}
```

### createBatch(String title, String beanName, int workerCount)

**重点学习**，里面涉及到了事务。

```java
 /** 
 * 创建一个batch，可以加实现了 TaskBatchProcessor 接口的 beanName
 */
public TaskBatch createBatch(String title, String beanName, int workerCount) {
    // 防止有同样 beanName 的batch 已经存在并且尚未完成
    TaskBatch existedRunningBatch = dao.fetch(TaskBatch.class,
            Cnd.where("batch_bean", "=", beanName).and("finished", "=", 0));
    if (existedRunningBatch != null) {
        //在这里如果发现了存在相同bean并且没有完成的batch，就抛出异常，之后的代码都不会执行
        throw Lang.makeThrow("存在同batch bean并且未完成的batch " + existedRunningBatch.getId());
    }
    //否则就是没有发现存在相同bean的batch，于是就创建一个batch，同时还记录了一个TaskProcesser的bean
    TaskBatch batch = TaskBatch.builder().title(title).batchBean(beanName)
            .workerCount(workerCount).taskCount(0).build();
    TaskBatchProcessor processor = (TaskBatchProcessor) ctx.getBean(beanName);
    //TaskBatchProcessor可实现在startDownloader前后做一些操作，有时候逻辑不一样无法都写道startDownloader中去，就可以使用processor
    try {
        //开启了一个事务，执行下面的代码
        Trans.exec(() -> {
            //lamda表达式，表示给exec这个方法传入的是一系列的原子方法
            processor.beforeStart();
            TaskBatch batch1 = dao.insert(batch, true, false, false);
            batch.setId(batch1.getId());
            //这里之所以需要用batch来设置id而不是直接返回batch1，是因为batch1包裹在事务中无法被返回回去
        });
    } catch (Exception e) {
        //如果事务失败了就会报错，并且回滚，不会影响到正常表内原来的数据
        log.error("Err： ", e);
        String message = String.format("batch-%d执行prev的任务异常,%s", batch.getId(), e.getMessage());
        throw new TechmapException(message);
    }
    return batch;
}
```

### addTask2Batch（）

递交一个任务到`Batch`里面；

```java
public Task addTask2Batch(TaskBatch batch, Task task) {
    return addTask2Batch(batch.getId(), task);
}

public Task addTask2Batch(Long batchId, Task task) {
    task.setBatchId(batchId); //把这个task的batchid赋值，记录这个任务是属于哪一个batch
    task.setStatus(NOT_START);
    return dao.insert(task, true, false, false); //记录到db里面
}
```

在递交一个`Task`到`Batch`中的时候，需要定义`task`的`beanName`，其是一个继承了`TaskDownloader`接口的类，并重写了`startDownloader`方法；例如：

```JAVA
taskService.addTask2Batch(batch, Task.builder().batchId(batch.getId())
                    .beanName("wikiCheckService")
                    .params(String.valueOf(project.getId())).build());
```

### checkExecutors（）

在这一部分找到所有没有开始的`task`，并判断是否给这个`task`的`Batch`分配了`executor`，`executor`表继承了`Thread`，分配给一个

```JAVA
@Scheduled(fixedDelayString = "${checkExecutors:10000}")
public void checkExecutors() {
    Sql sql = Sqls.create("select distinct batch_id from task where status=" + NOT_START);
    //首先找到了所有存在没有开始的任务的batchId
    List<Long> batchIds = DaoUtil.fetchList(dao, sql, "batch_id");
    for (Long batchId : batchIds) {
        if (executors.get(batchId) == null) {
            //executors是一个Hashmap，K-V分别对应了btachId和分配的executor，如果是null的话，说明没有给这个Batch分配线程池去进行任务，所以我们首先在表里把对应batchId的这个batch给查出来了
            TaskBatch batch = dao.fetch(TaskBatch.class, batchId);
            if (needSkip(batch)) {
                continue;
            }
            TaskBatchExecutor executor = new TaskBatchExecutor(this, batch);
            //创建一个新的线程池，然后立刻启动这个线程池
            executors.put(batchId, executor);
            executor.start();
        }
    }
}

private boolean needSkip(TaskBatch batch) {
    if (!"local".equals(System.getProperty("tencent.rainbow.group", ""))) {
        //如果七彩石配置里面不是local环境
        if (batch != null && batch.getTitle() != null && batch.getTitle()
            .contains("(local)")) {
            //不是local环境下如果这个batch是local的，就跳过，否则说明不是local，不能跳过，任务必须执行
            return true;
        }
    } else {
        //如果七彩石配置里面是local环境
        if (batch == null || batch.getTitle() == null || !batch.getTitle()
            .contains("(local)")) {
            //为null，或者这个任务根本不包含local，不是local任务，就可以直接跳过了
            return true;
        }
    }
    return false;
}
```

### clearOldTask()

```JAVA
     /**
     * 每天清理一次30天前的task
     */
@Scheduled(cron = "0 0 1 * * *")
public void clearOldTask() {
    Sql sql = Sqls.create(
        "select id from task_batch where updated_at<DATE_SUB(NOW(), INTERVAL 30 DAY)");
    List<Long> batchIds = DaoUtil.fetchList(dao, sql, "id");
    //筛选时隔30天的batch的id
    for (Long id : batchIds) {
        // delete tasks for batch
        int cleared = 0;
        do {
            cleared = dao.clear("task", Cnd.where("batch_id", "=", id).limit(1, 1000));
            //首先把task表里的旧的batchId的项删除了，每一次删除一部分
            log.info("Clear " + cleared + " task for batch " + id);
        } while (cleared > 0);
        dao.delete(TaskBatch.class, id);
        //最后把在task_batch表里的对应id的任务记录删除
    }
}
```

### getWaitingTasks()

```JAVA
List<Task> getWaitingTasks(Long batchId, int limit) { //给定任务的batchId和需要拿出来的任务数量限制limit，返回一个任务列表
    try {
        final List<Task> tasks = new ArrayList<>();
        Trans.exec(() -> {
            tasks.addAll(dao.query(Task.class,
                                   Cnd.where("batchId", "=", batchId).and("status", "=", NOT_START)
                                   .limit(1, limit))); //把所有status是NOT_START的任务都找出来并存在List<Task>里面
            Sql sql = Sqls.create("update task set status=@status where id=@id");
            for (Task t : tasks) { // change status to downloading
                sql.setParam("id", t.getId()).setParam("status", DOWNLOADING);
                sql.addBatch();
                //更新所有的没有执行的正在等待的task的状态为DOWNLOADING，addBatch是将所有的参数保存下载，以便在execute的时候批量处理
            }
            dao.execute(sql);
        });
        return tasks;
    } catch (Exception e) {
        log.error("Can't get waiting tasks", e);
        return new ArrayList<>();
    }
}
```

### startDownload（）

该方法在`TaskBatchExecutor`中的`run`方法中会被调用；

```JAVA
void startDownload(Task t) {
    TaskDownloader downloader = (TaskDownloader) ctx.getBean(t.getBeanName());
    //通过绑定在task上的beanName获得downloader的bean
    List<Task> nextTasks = null;
    t.setTraceId(TraceContext.traceId()); //分配一个跟踪ID以跟踪任务动向
    try {
        nextTasks = downloader.startDownload(t);
        //根据不同下载器对startDownloader的重写，有时候不会返回下一个任务，因此这时候nextTasks就是null
        t.setStatus(FINISHED); //把任务状态置为完成
    } catch (TechmapException te) {
        //如果在startDownload方法中报错了，就把任务状态置为以报错结束
        t.setStatus(FINISHED_TECHMAP_EXCEPTION);
        t.setMessage(Strings.cutStr(500, te.getMessage(), null));
    } catch (Throwable e) {
        if (e != null && e.getMessage() != null && (e.getMessage().contains("404") || e.getMessage()
                                                    .contains("403"))) {
            t.setStatus(FINISHED_TECHMAP_EXCEPTION);
        } else {
            t.setStatus(FINISHED_EXCEPTION);
        }
        t.setMessage(Strings.cutStr(500, e.getMessage(), null));
        log.error("task download Err", e);
    }

    final List<Task> finalNextTasks = nextTasks;
    Trans.exec(() -> {
        t.setUpdatedAt(null);
        dao.updateIgnoreNull(t);
        //这里更新的就是Task的状态status，trace id，message等列的数据
        dao.update(TaskBatch.class, Chain.makeSpecial("taskCount", "+1"),
                   Cnd.where("id", "=", t.getBatchId()));
        //将task_batch表中该task对应的batch执行完毕的任务数量+1
        if (finalNextTasks != null) {
            //如果还有子任务的话，就也添加到这个Batch内，等待执行
            for (Task nextTask : finalNextTasks) {
                addTask2Batch(t.getBatchId(), nextTask);
            }
        }
    });
}
```

### allTaskDone（）and  setBatchFinished()

```JAVA
     /**
     * 是否所有任务都完成了？ 如果 batch 当中，没有未开始的任务，也没有下载中的任务，那么我们认为这个batch已经完成了
     */
boolean allTaskDone(Long batchId) {
    if (dao.fetch(TaskBatch.class, batchId).getFinished() == 1) {
        // 如果在task_batch表中的结束已经被更改，则把task表中的所有任务都视作完成
        dao.update(Task.class, Chain.make("status", FINISHED), Cnd.where("batchId", "=", batchId).and(
            Cnd.exps(Cnd.exp("status", "=", NOT_START)).or(Cnd.exp("status", "=", DOWNLOADING))
        ));
        //这句话的dao重点看一下，在Cnd.exps内传入一个exp，会将这个exp变为一个条件表达式组，因此调用表达式组的or方法，就将这两个条件封装在一起，并且被前面的and连接，相当于我们写成了 where batchid = #{batchid} and (status = #{NOT_STRAT} or status = #{DPWNLOADING})
        return true;
    }
    //如果在task_batch表中没有finished置1，我们就在task表中找当前batchId下的所有还未开始和正在进行的任务，如果没有还未开始或者正在进行的说明就全部任务都完成了，就返回true，否则就返回false
    int count = dao.count(Task.class,
                          Cnd.where("batchId", "=", batchId).and(
                              Cnd.exps(Cnd.exp("status", "=", NOT_START))
                              .or(Cnd.exp("status", "=", DOWNLOADING))));
    return count == 0;
}

//如果发现所有这个Batch下的Task都执行完毕了，就可以吧这个TaskBatch置为Finished
void setBatchFinished(TaskBatch batch) {
    // 有可能会有多个cron在运行，如果batch已经是finished的了，那么就不继续执行 afterFinish了
    AtomicBoolean needFinished = new AtomicBoolean(false);
    Trans.exec(() -> {
        Sql sql = Sqls
            .create("select * from task_batch where id=" + batch.getId() + " for update");
        // for update 给这个sql上锁，别的会话可以选择这些行但是不能更改或者删除这些行
        TaskBatch batchCurrent = DaoUtil.fetchEntity(dao, sql, TaskBatch.class);
        if (batchCurrent.getFinished() == 0) {
            needFinished.set(true);
            batchCurrent.setFinished(1);
            dao.update(batchCurrent);
        }
    });
    executors.remove(batch.getId());
    if (needFinished.get() && !StringUtils.isEmpty(batch.getBatchBean())) {
        TaskBatchProcessor processor = (TaskBatchProcessor) ctx.getBean(batch.getBatchBean());
        processor.afterFinish();
    }
}
```



# TaskBatchExecutor

## 简介

这个类继承了`Thread`，是用来执行`startDownloader`方法的线程类。

## 项目中涉及的方法

### 基本属性变量

```JAVA
private static AtomicInteger poolNumber = new AtomicInteger(1);
//这是一个静态的原子Int量，其可以实现不加synchronized锁的前提下实现线程安全的数字更改。
private TaskService taskService;
private ThreadPoolExecutor executorService;
private TaskBatch batch;
private volatile boolean destroying = false;
```

### TaskBatchExecutor

```JAVA
/**
     * 构造函数
     */
public TaskBatchExecutor(TaskService taskService, TaskBatch batch) {
    this.taskService = taskService;
    this.batch = batch;
    int poolNum = poolNumber.getAndIncrement(); //原子自增，因为这是一个static量，其计算了所有executor内创建出的线程池的总数。
    executorService = (ThreadPoolExecutor) Executors
        .newFixedThreadPool(batch.getWorkerCount(), new ThreadFactory() { //这里get了预先设定的workCount，即线程池的最大核心线程数量
            AtomicInteger threadNumer = new AtomicInteger(1); //这个可能是线程数量
            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r,
                                  String.format("batchTask-pool%d-t%d", poolNum,
                                                threadNumer.getAndIncrement())); //设定当前线程的名字
            }
        });
}
//其中newFixedThreadPool的源代码如下
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
}
//这个线程池只有nThreads个核心线程，最大线程数也是nThreads，其次工作队列用的是基于链表的FIFO工作队列
```

### stopTaskBtach

```java
public void stopTaskBatch() {
    destroying = true;
}
```

### run

```JAVA
     /**
     * 线程执行
     */
public void run() {
    boolean stopping = false;
    while (!stopping && !destroying) {
        if (executorService.getQueue().size() > 100) { //首先判断一下当前线程池内工作队列等待长度，如果大于100个任务就休息1s
            sleep1second();
        } else {
            List<Task> tasks = taskService.getWaitingTasks(batch.getId(), 100); //获得100个正在等待的任务，修改表中得status从“未开始”到“下载”，然后返回Task列表
            if (tasks.size() == 0) {
                sleep1second(); //如果列表是0，有可能是报错了，也有可能是
            }
            for (Task t : tasks) {
                executorService.execute(() -> taskService.startDownload(t));
                // 这里的execute方法需要传入一个Runnable command，利用lamda表达式，其实就是用startDownload方法重写了Runnable接口的run方法
            }
        }
        if (taskService.allTaskDone(batch.getId())) { //判断一下是否这个Batch下的所有Task都执行完毕了
            stopping = true;
        }
    }
    if (stopping) {
        taskService.setBatchFinished(batch);
    }
    executorService.shutdown(); //关闭线程池
}

private void sleep1second() {
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        // Do nothing here.
    }
}
```













































