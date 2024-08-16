## 一、前言

ScheduledThreadPoolExecutor是基于ThreadPoolExecutor实现的，在学习ScheduledThreadPoolExecutor时，请先移步到第11节学习ThreadPoolExecutor相关的知识

## 二、基础

### 2.1 类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/1b6edf999be5292805f21d91c751a24c.png)

ThreadPoolExecutor：线程池实现，关于ThreadPoolExecutor的内容，请移步到第11节

#### 2.1.1 ScheduledExecutorService

定义调度相关的方法

```java
//延时对应时间调度这个任务，ScheduledFuture继承了FutureTask，对于Runnable类型的任务，其内部使用了一个RunnableAdapter适配器
//对应的任务执行完成后调用ScheduledFuture#get方法将返回null
public ScheduledFuture<?> schedule(Runnable command,
                                       long delay, TimeUnit unit);

//延时对应时间调度这个任务，与上面的调度方法不同，它的 ScheduledFuture在执行完任务后将返回callable的返回值
public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay, TimeUnit unit);

//每隔period时间调度一次
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);

//每隔period时间调度一次                                            
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);
```
#### 2.1.2 ScheduledFutureTask

ScheduledThreadPoolExecutor的内部类，以下是它的类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/8e754e32cefa4e15c27d56fbab98e3fe.png)

可以看到这个ScheduledFutureTask继承了FutureTask，FutureTask的相关内容可以移步到第11节ThreadPoolExecutor中学习。
- Delayed：延时节点，定义了一个方法

```java
//获取到期时间间隔，会根据传入的单位转换成对应单位的值
long getDelay(TimeUnit unit);
```
- ScheduledFuture：综合Delayed与Future接口
- RunnableScheduledFuture：定义了一个判断当前任务是否是定期执行任务

```java
boolean isPeriodic();
```

现在我们来分下ScheduledFutureTask类的一些字段的含义

```java
//AtomicLong，为每个延时任务生成一个唯一的序列号
private final long sequenceNumber;

//当前任务到期时间，单位ns
private long time;

//如果是定期执行的任务，那么这个值不为空
//为正数时，period = 下次执行时间点 - 上次执行点，这种有副作用，如果任务执行比period长，那么任务看起来就像是一个无限循环一直在调用，没有时间间隔
//为负数时，period = 下次执行时间点 - 上次任务执行完毕时重新计算时间点的当前时间（System.nanoTime()）
private final long period;

//通常用于引用装饰当前任务的那个装饰对象
RunnableScheduledFuture<V> outerTask = this;

//堆下标，用于快速定位
int heapIndex;

//获取剩余时间，单位ns
public long getDelay(TimeUnit unit) {
    return unit.convert(time - now(), NANOSECONDS);
}

//比较优先级
public int compareTo(Delayed other) {
    if (other == this) // compare zero if same object
        return 0;
    if (other instanceof ScheduledFutureTask) {
        ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
        long diff = time - x.time;
        if (diff < 0)
            return -1;
        else if (diff > 0)
            return 1;
        else if (sequenceNumber < x.sequenceNumber)
            return -1;
        else
            return 1;
    }
    //不是ScheduledFutureTask的延时任务，使用到期时间间隔进行对比
    long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
    return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
}

//是否是定期执行任务
public boolean isPeriodic() {
    return period != 0;
}

//对于定期执行的任务，会计算下一次执行的时间
private void setNextRunTime() {
    long p = period;
    if (p > 0)
        time += p;
    else
        //p为负数表示任务执行完将以当前时间开始延后p时间再次执行，与p为正数的不同，正数的是以上一次执行时间为准，以p时间间隔执行
        //p为正数可能会出现这样的一个问题，那就是执行的任务耗时超过period，这将导致这样的任务没有停顿时间，感觉就像一个循环一样，一直在执行
        time = triggerTime(-p);
}

//这是ScheduledThreadPoolExecutor的方法，用于计算触发时间
long java.util.concurrent.ScheduledThreadPoolExecutor#triggerTime(long delay) {
    //如果延时时间大于了Long.MAX_VALUE的一半，那么将可能有溢出的风险，需要进行overflowFree的处理
    return now() +
        ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
}

//这是ScheduledThreadPoolExecutor的方法，用于屏蔽溢出风险
private long java.util.concurrent.ScheduledThreadPoolExecutor#overflowFree(long delay) {
    //获取延时队列中最近要执行的任务
    Delayed head = (Delayed) super.getQueue().peek();
    if (head != null) {
        //计算到期时间间隔
        long headDelay = head.getDelay(NANOSECONDS);
        //如果到期间隔时间已经是负数了并且delay - headDelay发生了溢出
        if (headDelay < 0 && (delay - headDelay < 0))
            //用Long的最大值加上这个负数，这样在两个任务进行对比的时候，也就是调用compareTo方法的时候就不会导致溢出
            delay = Long.MAX_VALUE + headDelay;
    }
    return delay;
}

//这是ScheduledThreadPoolExecutor的方法，用于判断在线程池状态变为SHUTDOWN时是否还需要去执行定时任务
boolean java.util.concurrent.ScheduledThreadPoolExecutor#canRunInCurrentRunState(boolean periodic) {
    return isRunningOrShutdown(periodic ?
                               continueExistingPeriodicTasksAfterShutdown :
                               executeExistingDelayedTasksAfterShutdown);
}

public void run() {
    boolean periodic = isPeriodic();
    //检查线程池在SHUTDOWN时是否需要继续定时任务的执行，否则取消任务
    if (!canRunInCurrentRunState(periodic))
        cancel(false);
    else if (!periodic)
        //如果不是定期执行任务，那么直接执行，执行完这次之后就结束了
        ScheduledFutureTask.super.run();
        //定期执行的任务，除了出错或者被中断，被取消之外，定期任务的状态一直是NEW
    else if (ScheduledFutureTask.super.runAndReset()) {
        //设置下一个被调度的时间
        setNextRunTime();
        //将任务加入到延时队列中，等待下一次调度
        reExecutePeriodic(outerTask);
    }
}
```
#### 2.1.3 DelayedWorkQueue

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d7846febb2b249e7e8cf5a6cdeda9108.png)

DelayedWorkQueue和其他队列的继承结构一样，这里就不过的说明了，下面我们来分析下DelayedWorkQueue的字段说明

```java
//初始化容量
private static final int INITIAL_CAPACITY = 16;

//使用数组实现延时工作队列
private RunnableScheduledFuture<?>[] queue =
    new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
    
//同步
private final ReentrantLock lock = new ReentrantLock();

//元素个数
private int size = 0;

//与delayedQueue一样有个leader线程，用于避免大量线程等待同一个任务，避免同一时间被唤醒。
private Thread leader = null;

//条件队列
private final Condition available = lock.newCondition();

```

DelayedWorkQueue其实就是和DelayQueue（内部使用了PriorityQueue）一样，都是使用堆实现的排序，DelayedWorkQueue比DelayQueue多了一个给RunnableScheduledFuture记录堆下标的方法，更多DelayQueue相关的内容请移步到第13节 《DelayQueue》

## 三、任务调度

ScheduledThreadPoolExecutor的execute方法，submit方法都是直接调用ScheduledExecutorService的调度方法

```java
public ScheduledFuture<?> ScheduledThreadPoolExecutor#schedule(Runnable command,
                                       long delay,
                                       TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    //装饰任务，在ScheduledThreadPoolExecutor还没有做任务操作，直接返回ScheduledFutureTask对象
    RunnableScheduledFuture<?> t = decorateTask(command,
        //创建一个调度任务
        new ScheduledFutureTask<Void>(command, null,
                                        //计算触发的时间点
                                      triggerTime(delay, unit)));
    //创建工作线程，准备执行
    delayedExecute(t);
    return t;
}
```
下面我们来看下是如何调度延时任务的

```java
private void delayedExecute(RunnableScheduledFuture<?> task) {
    //检查线程池是否已经被关闭
    if (isShutdown())
        //执行拒绝策略
        reject(task);
    else {
        //将延时任务加入到队列中
        super.getQueue().add(task);
        //检查线程池是否被关闭
        if (isShutdown() &&
            //任务为定期执行任务并且线程池状态为SHUTDOWN时，是否需要继续执行这个刚加入进来的任务
            !canRunInCurrentRunState(task.isPeriodic()) &&
            //不需要执行将会被移除
            remove(task))
            //任务被取消
            task.cancel(false);
        else
            //尝试启动工作线程去执行队列任务
            ensurePrestart();
    }
}
```

首先先检查线程池是否已经被关闭，如果已经被关闭，那么将交由拒绝策略去处理，如果没有被关闭，那么加入到延时队列中，如果刚加入线程池又被关闭了，那么检查是否需
要取消刚才加入的任务，如果线程池一直是正常运行的，那么尝试启动新的工作线程去执行队列任务。

```java
void ensurePrestart() {
    //获取当前线程池已经启动的工作线程数
    int wc = workerCountOf(ctl.get());
    //如果线程池数还没有超过核心线程数，那么尝试启动新的工作线程
    //addWorker方法我们在分析ThreadPoolExecutor时进行详细的讲解
    //任务传入null表示去队列中取任务执行
    if (wc < corePoolSize)
        addWorker(null, true);
        //如果走到这个分支，那么说明没有设置核心线程数量
    else if (wc == 0)
        addWorker(null, false);
}
```
先检查核心线程数是否已满，没满，那么尝试创建新的工作线程去处理任务，如果用户没有设置核心线程数，那么最大线程数是2的29次方，int的高三位表示线程池状态

下面我们再来看看定期执行任务的调度

```java
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    if (period <= 0)
        throw new IllegalArgumentException();
    ScheduledFutureTask<Void> sft =
        new ScheduledFutureTask<Void>(command,
                                      null,
                                      triggerTime(initialDelay, unit),
                                      unit.toNanos(period));
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);
    //记录是谁装饰了自己，以便
    sft.outerTask = t;
    delayedExecute(t);
    return t;
}
```
和schedule方法差不多，创建ScheduledFutureTask时多了一个定期时间period，记录了是谁装饰了自己，为什么要记录它呢？在上面我们分析ScheduledFutureTask的时候，
它的run方法执行完任务后对于定期执行的任务需要重新加入到队列中

## 四、总结

调度线程池结合了延时队列和ThreadPoolExecutor的能力，所以学习ScheduledThreadPoolExecutor只要学会了延时队里和ThreadPoolExecutor基本上就可以看懂ScheduledThreadPoolExecutor了
