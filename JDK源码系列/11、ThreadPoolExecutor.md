## 一、基础

### 1.1 类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/24bef3bd0bc465e006662250f0af11df.png)

#### 1.1.1 Executor

```java
public interface Executor {

    //提交一个实现了Runnable的任务，异步执行
    void execute(Runnable command);
}
```

#### 1.1.2 ExecutorService

以下对于方法的描述全是基于ThreadPoolExecutor的实现说明的，因为不同的实现可能产生不同的说法，但是实现的效果肯定是要符合接口定义的。

```java
public interface ExecutorService extends Executor {

    //关闭任务服务，设置线程池状态为SHUTDOWN，将不再接收新的任务，但仍然会执行队列中等待的任务
    //中断空闲的线程（不会中断正在执行任务的线程，未start的线程（即使中断也无效），未将state由-1设置为0的worker）
    //被中断的线程如果没有将中断标志位清除，那么它就不会再阻塞。
    void shutdown();

    //立即关闭任务服务，设置线程池状态为STOP，将不再接收新的任务，正在等待的任务将不再处理
    //中断空闲和正在执行任务的线程
    //返回等待执行的任务列表
    List<Runnable> shutdownNow();

    //检查当前任务服务是否已经被关闭，在ThreadPoolExecutor的实现中，只要大于等于SHUTDOWN状态都视为关闭
    boolean isShutdown();

    //状态为TERMINATED
    boolean isTerminated();

    //阻塞线程直到wait超时或者被唤醒发现发现已经TERMINATED
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    //提交任务异步执行，返回代表task的Future对象
    <T> Future<T> submit(Callable<T> task);

    //提交任务异步执行，返回代表task的Future对象，执行完毕之后，将传入的result值设置到Future中
    //Runnable没有返回值，内部实现是将task包装到一个实现了Callable节点的适配器中
    <T> Future<T> submit(Runnable task, T result);

    //提交任务并返回Future对象，任务执行完，Future的get方法将返回null
    Future<?> submit(Runnable task);

    //传入任务列表，线程池处理完所有的任务后（内部会调用Future的get方法等待），返回Future列表
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    //传入任务列表，在指定时间内等待执行，如果超过时间没有完成，那么将返回任务列表的Future列表
    //由用户处理
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    //传入任务列表，从集合中循环调用任务，如果在循环中有任务已经完成，那么未循环的任务将不会执行
    //如果调用的任务发生超时或者其他异常，继续执行集合中后面的任务直到有一个任务完成返回
    //当然了，这里可能有一些任务白白执行了，如果执行的任务比较耗时，那么所有的任务都会被放进线程池，但最终就取最先执行完成的任务
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    //这个方法和上面这个invokeAny是差不多的，只是加了时间限制，在指定内没有任何一个任务完成，那么将抛出TimeoutException
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

ExecutorService的方法声明中使用到Future这个接口，在ThreadPoolExecutor的实现，其创建的实现类为FutureTask，下面是它的类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/277b51fb20d84f81ea7db6eace635d26.png)

- Runnable：被FunctionalInterface注解标注，表示它是一个函数式接口，这个接口干什么的就不用多说了
- Future：代表某个任务，其定义的方法如下：

以下方法的解释以FutureTask实现为准

```java
//取消某任务，任务还是NEW的状态下可取消，mayInterruptIfRunning如果为true，那么会中断线程并且任务状态变化为  NEW -> INTERRUPTING -> INTERRUPTED
//mayInterruptIfRunning如果为false，那么状态变化为NEW -> CANCEL
//cancel方法很大概率导致任务白白执行，Doug Lea为什么不增加RUNNING状态呢？只取消那些还未执行的任务，比如存在队列中的，INTERRUPTING的话可以理解，可能有些任务
//比较耗时，在等待某些网络IO操作，存在阻塞，超时中断取消很合理，但是我个人觉的CANCEL这种状态应该建立在还未运行的情况下，否则任务真的就白白执行了
//任务取消后，调用get方法将收到CancellationException异常
boolean cancel(boolean mayInterruptIfRunning);

//状态大于等于CANCELLED
boolean isCancelled();

//只要状态不为NEW即为done
boolean isDone();

//获取任务执行结果，如果任务状态小于等于COMPLETING，将会阻塞
//等待线程被中断将抛出中断异常，如果任务执行出错将抛出执行异常
V get() throws InterruptedException, ExecutionException;

//在指定时间内等待任务完成，超过时间将抛出TimeoutException
V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
```
- RunnableFuture：综合Runnable与Future
- FutureTask：具体的任务实现，下面我们来分下它的字段含义

```java

//任务
private Callable<V> callable;

//任务的返回值
private Object outcome;

//执行这个任务的线程对象
private volatile Thread runner;

//等待获取执行结果被挂起的线程节点
private volatile WaitNode waiters;

//表示任务的状态
private volatile int state;

//表示任务是新建的并且还没有执行完成
private static final int NEW          = 0;

//表示任务对应的用户代码已经执行完成，但是还没有将返回值设置到字段outcome中
//由于并不是原子操作，所以有个COMPLETING状态进行过渡
//其操作过程在set和setException方法中，下面会举其中的set方法的例子
private static final int COMPLETING   = 1;

//表示任务正常执行完成
private static final int NORMAL       = 2;

//表示任务抛出了异常
private static final int EXCEPTIONAL  = 3;

//表示任务被取消，但并不代表任务一定没有被执行
private static final int CANCELLED    = 4;

//表示任务正在中断取消中
private static final int INTERRUPTING = 5;

//表示任务已经被取消，对应的工作线程被中断
private static final int INTERRUPTED  = 6;



```
为什么会有COMPLETING与INTERRUPTING这种过渡状态？下面是set方法的代码

```java
//将任务执行的返回值设置到字段outcome
protected void set(V v) {
    //将任务状态设置为COMPLETING，
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        //将状态修改为最终状态NORMAL
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        //唤醒等待任务执行结果被挂起的线程
        finishCompletion();
    }
}
```
其实不难发现，将任务返回值设置到outcome和将状态修改成最终状态NORMAL的操作并非原子的，为了不使用锁这种比较重的操作，Doug Lea采用过渡状态COMPLETING去，可能
有人会问一个任务不就一个线程在跑吗？的确是一个线程在跑，但是状态的改变可能会是多个线程参与的，比如取消任务，这是由其他线程来操作的。INTERRUPTING这种过渡
状态是一样的道理，这里就不再细说了。



```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    
    //记录线程池运行状态和线程数，高三位表示线程池运行状态，低29表示线程池的线程数量
    //为什么不使用long，目前来说int已经足够了，int的操作相对于long来说操作更快
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    //用于记录线程数的位数为29
    private static final int COUNT_BITS = Integer.SIZE - 3;
    //总线程数量2^29 - 1，高三位用于记录线程池运行状态
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
    
    // runState is stored in the high-order bits
    //二进制表示 11100...00，表示线程池状态，线程池正在运行
    private static final int RUNNING    = -1 << COUNT_BITS;
    
    //二进制表示 00000...00，表示线程池正在关闭，此时拒绝接收任务，但是会把还在队列中的任务执行完
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    
    //二进制表示 00100...00，表示线程池已经停止，此时拒绝接收任务，不会处理还在队列中的任务并且还会中断正在执行任务的线程
    private static final int STOP       =  1 << COUNT_BITS;
    
    //二进制表示 01000...00，表示线程池中所有线程都已经停止，线程数量为零，将会调用terminated方法
    private static final int TIDYING    =  2 << COUNT_BITS;
    
    //二进制表示 01100...00，terminated方法被调用
    private static final int TERMINATED =  3 << COUNT_BITS;
    
    //总线程数量2^29 - 1，我们知道一个2的n次幂的值除了符号位之外所有的bit位上就只有一个1，减去1，那么低29位的bit值都是1
    //这里取反，那么高3位都为1，与c相与那么就是获取高三位的runState
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    
    //由于低29位都是1，那么c与CAPACITY位与，那么获取的当然就是线程数
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    
    //用户更换运行状态
    private static int ctlOf(int rs, int wc) { return rs | wc; }
    
    //检查当前状态是否小于某个状态
    private static boolean runStateLessThan(int c, int s) {
        return c < s;
    }
    
    //检查当前运行状态是否大于等于某个状态
    private static boolean runStateAtLeast(int c, int s) {
        return c >= s;
    }
    
    //检查线程池是否还在运行，判断依据小于SHUTDOWN，RUNNING状态对应int值为-1
    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }

    //递增线程数
    private boolean compareAndIncrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect + 1);
    }

    //递减线程数
    private boolean compareAndDecrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect - 1);
    }

    //递减线程数直到成功
    private void decrementWorkerCount() {
        do {} while (! compareAndDecrementWorkerCount(ctl.get()));
    }

    //当前核心线程全部都已经用上了，那么其他的任务可以放在这个队列中
    private final BlockingQueue<Runnable> workQueue;

    //用于同步，比如循环中断每个工作线程，像这种比较粗粒度的操作需要使用到同步锁
    private final ReentrantLock mainLock = new ReentrantLock();

    //记录当前正在工作的线程，可以认为它是个职员表吧
    private final HashSet<Worker> workers = new HashSet<Worker>();

    //用于awaitTermination方法，在指定时间被wait
    private final Condition termination = mainLock.newCondition();

    //记录曾经存在过的最多线程的数量
    private int largestPoolSize;

    //综合所有线程完成的任务
    private long completedTaskCount;

    //用于创建线程的工厂
    private volatile ThreadFactory threadFactory;

    //拒绝执行策略，如果用户没有传递，那么默认为AbortPolicy，AbortPolicy会抛出错误
    private volatile RejectedExecutionHandler handler;

    //主要用于超过核心线程创建的线程，如果这部分线程空闲，那么在指定时间后将被关闭
    //如果allowCoreThreadTimeOut字段被设置为true，那么核心线程也会使用这个keepAliveTime参数用于设置wait超时时间
    private volatile long keepAliveTime;

    //默认为false，如果设置为true，那么将使用keepAliveTime参数作为wait超时时间
    private volatile boolean allowCoreThreadTimeOut;

    //核心线程数，当处理任务的线程数达到核心线程数之后，并且都在处理任务的话，那么其他任务提交的任务将放入队列
    //核心线程数不会有存活时间限制
    private volatile int corePoolSize;

    //最大的线程数，这个值是由用户创建线程池的时候设置的，超过核心线程创建出来的线程在空闲情况下，会在达到指定的存活期限之后关闭
    private volatile int maximumPoolSize;

    //当前队列满了并且线程数已经达到用户设置的最大线程数量，那么将会调用RejectedExecutionHandler处理任务
    //默认是抛出错误
    private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();

    //当开启了jdk的安全管理时，会检查当前程序是否有权限去修改线程的状态，比如中断等
    private static final RuntimePermission shutdownPerm =
        new RuntimePermission("modifyThread");

}
```
#### 1.1.3 AbstractExecutorService

AbstractExecutorService是一个模板方法，实现了一些通用的方法，大部分接口方法都已经被它实现，只有以下方法没有实现

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/307d00de44bb439e1a93bf22e3263ada.png)

#### 1.1.4 Worker

封装任务与工作线程，实现了Runnable接口

```java
//工作线程
final Thread thread;

//任务
Runnable firstTask;

//完成任务计数
volatile long completedTasks;
```
Worker还实现了AbstractQueuedSynchronizer

## 二、任务的提交

前面我们将一些基础的东西梳理了一遍，现在我们深入到具体的逻辑中去分析它的实现，下面任务提交方法的代码

```java
public Future<?> AbstractExecutorService#submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    //创建的对象为FutureTask
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    //提交任务
    execute(ftask);
    return ftask;
}
```
execute方法就是Executor接口定义的异步执行方法

```java
public void ThreadPoolExecutor#execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    //获取记录了线程池运行状态与工作线程数的int值
    //高三位表示线程池运行状态，低29位表示工作线程数
    int c = ctl.get();
    //如果当前工作线程数小于核心线程数，那么尝试创建工作线程去执行任务
    if (workerCountOf(c) < corePoolSize) {
        //创建工作线程处理任务，可能会失败，比如线程池被关闭等等
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    //如果线程池没有被关闭，执行到这，那么就是核心线程数量已经满了，那么先丢到队列中的
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        //再检查一下线程池状态，如果已经关闭，那么将刚才扔进队列的任务吐出来
        //为啥又判断一遍？可能在offer任务到队列的时候线程被关闭了，当然具体是在offer前关闭的还是offer后关闭的就没法判断的，时间非常短，但是很显然作者认为它
        //就是应该被移除
        if (! isRunning(recheck) && remove(command))
            //调用拒绝策略处理
            reject(command);
            //没有工作线程，可能是线程池被关闭，可能是线程设置了超时时间，超时被释放
            //不管哪种，由于刚才往队列中offer了任务，虽然不确定是否已经被执行完了，但还是要尝试去创建工作线程去处理
            //如果线程池是STOP以上的状态，那么不会执行任务，addWorker传入空任务，表示从队列中获取任务执行
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    //如果线程关闭或者队列已满，那么继续尝试创建工作线程处理任务，如果线程池是关闭的，那么将返回false
    //或者线程数已经达到最大，也会返回false
    else if (!addWorker(command, false))
        //调用拒绝策略处理
        reject(command);
}
```
从上面的代码中，我们大致可以知道线程池的运作原理了，当然还有一部分原理还需要继续看代码得出，这里先写出来，我们带着原理继续往下看:
- 当线程还没有达到核心线程数的时候，将创建核心工作线程去处理
- 当线程数达到了核心线程数之后，将任务直接加入到队列中，等待线程空闲去队列中取任务执行
- 当队列满了之后，如果线程数没有超过最大线程数，那么继续创建工作线程去处理任务
- 当队列满了，线程数已达到最大，那么调用拒绝策略处理

我们继续往下看，看线程池是怎么创建工作线程处理任务的

```java
private boolean ThreadPoolExecutor#addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        //获取线程池状态
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        //如果线程池状态为STOP以上，那么直接返回
        //如果线程池状态为SHUTDOWN，那么得看下firstTask是否为null并且队列中还有任务要处理，那么会继续创建工作线程去处理任务
        //firstTask == null表示去队列中取任务执行
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            //获取工作线程数量
            int wc = workerCountOf(c);
            //如果达到容量限制或者用户线程数量限制，那么创建工作线程失败，返回false
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            //cas递增工作线程数，准备创建工作线程
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read 
            //如果线程池状态发生改变，那么重新循环，看下是否是关闭了
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        //创建工人（内部调用线程工厂创建工作线程）
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            //同步，防止在添加任务时，线程发生关闭
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());
                //如果此时线程池为STOP以上的状态，那么任务将不会启动
                //如果线程池状态为SHUTDOWN，那么在firstTask == null时允许启动，用于处理队列中的任务
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    //记录创建的Worker，主要作用用于在线程池关闭时进行线程的中断
                    workers.add(w);
                    int s = workers.size();
                    //记录线程池曾经出现过的最大工作线程是数（工人数）
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            //如果任务添加成功，那么启动任务
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            //对于添加失败的，需要将线程数减1
            addWorkerFailed(w);
    }
    return workerStarted;
}
```
创建工作线程的步骤如下：

- 判断线程池是否已经关闭，如果已经关闭那么将不接受新的任务
- 根据传入的core判断是否达到线程数量的上限，如果达到上限那么将不会创建新的工作线程
- 如果允许创建，那么cas递增线程数量
- 创建工作线程，添加记录，启动，如果在此期间，如果在添加工作线程记录之前发生了线程池的关闭，那么将不会启动工作线程
- 如果添加任务失败，那么cas递减线程数

启动工作线程

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    //创建Worker时，worker的state的值被设置为-1，表示不允许被中断，因为线程池关闭的时候会中断线程
    //中断一个还没有启动的线程是没有任何效果的，但是这里可能又会出现一个问题，那就是刚好启动的时候
    //还没来得及将state由-1变成0，线程池就关闭了，那么就有可能丢失中断信号，所以在下面的代码会判断一下线程池的状态
    //然后中断线程
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        //如果task为空，那么将从队列中获取任务
        while (task != null || (task = getTask()) != null) {
            //工作线程锁主要用于线程池SHUTDOWN时，中断线程，但是中断的是空闲线程，怎么判断线程不空闲呢？通过获取锁即可
            //能获取到锁的那么就是空闲线程
            w.lock();

            //如果线程池状态大于STOP，为了确保线程被中断，此处会调用线程的中断方法
            //如果线程池没有STOP，那么确保这个线程没有被中断（Thread.interrupted()将清除中断状态），即使是SHUTDOWN，也只是中断空闲的工作线程
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                //任务执行前置
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    //执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    //任务执行后置
                    afterExecute(task, thrown);
                }
            } finally {
                //清空当前任务
                task = null;
                //任务数加1
                w.completedTasks++;
                //释放锁
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        //如果抛出异常，那么completedAbruptly不会被赋值为false
        processWorkerExit(w, completedAbruptly);
    }
}
```
启动工作线程的步骤：
- 修改worker的state，由-1变成0，此时工作线程可以被中断并且可以获取锁
- 如果线程池被关闭，那么检查是否需要中断线程
- 执行任务前置
- 执行任务
- 执行任务后置

如果当前任务执行完毕，那么需要从队列中获取任务执行，代码如下：

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        //如果线程池状态大于等于STOP，那么递减线程数量并返回空任务
        //如果线程池状态为SHUTDOWN并且队列中已经没有任务可领，那么递减线程数量并返回空任务
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
        //获取线程数
        int wc = workerCountOf(c);

        // Are workers subject to culling?
        //默认allowCoreThreadTimeOut是false，如果allowCoreThreadTimeOut为true，那么核心线程也会因为等待任务超时而关闭
        //如果allowCoreThreadTimeOut是false，那么只需检查线程数量是否超过了核心线程，超过核心线程数量的线程将会超时关闭
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        
        //wc > maximumPoolSize：线程数量会出现大于最大线程数的情况吗？不会，我想这是一种防御性编程，避免因为程序上的未知缺陷导致问题
        //timed && timedOut：表示定时超时，比如设置了定时时间的最大线程，超时之后需要释放掉
        //wc > 1：在满足以上两个条件后，如果工作线程大于1，那么需要释放
        //workQueue.isEmpty()：如果wc > 1不满足，也就是说工作线程可能是0也可能是1，但是队列如果还有任务的话，就不应该释放掉，要不然没人
        //去处理队列里的任务了
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            //释放工作线程，需要将线程池的线程数减1
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            //是否需要定时，如果需要定时，在指定时间内不能从阻塞队列获取到任务，那么将会被定义为超时等待，线程可能会被释放
            //一般地，我们的核心线程是没有定时的，所以它会一直阻塞直到有任务可领
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            //等待线程发生中断，不认为是wait超时，因为有可能是因为线程池关闭触发的中断信号,对于SHUTDOWN，仍然需要从队列中获取任务，对于STOP以上的
            //状态就会直接返回空任务
            timedOut = false;
        }
    }
}
```
获取任务步骤：
- 先检查线程池是否已经关闭，对于线程池状态大于等于STOP的，将释放线程，返回空任务，如果是SHUTDOWN，那么还要检查队列中是否有任务可领，如果没有的，也将释放
  线程，然后返回空任务
- 检查线程是否超时等待，如果线程在指定时间内没有获取到任务，那么将被释放并返回空任务
- 对于没有超时等待的线程将会一直阻塞下去直到线程池被关闭或者有任务可领才会唤醒


如果执行任务的时候抛出错误或者因为超时没有任务可领时，将会给这个即将释放的线程做收尾工作，代码如下：

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    //如果执行任务时，任务抛出了异常，那么当前工作线程并未正常递减线程数量，所以这里需要做调整
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        //记录当前工作线程处理了多少任务
        completedTaskCount += w.completedTasks;
        //将对应工作对象从工人表中移除，被开除了
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
    //尝试给线程池设置停止状态，只有在线程池被关闭的情况下执行才有可能被彻底关闭
    tryTerminate();

    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        //如果抛出了异常，那么抛出异常的线程被释放到之后，需要尝试是否需要补充线程去处理队列中的任务
        //如果没有抛出异常，那么判断线程是否具有超时时间
        //一般地，我们的核心线程是没有超时时间的，所有对于超过核心线程数量的超生线程将被return
        //如果都是有超时时间的，那么在队列还有任务的情况下，还是要留下一个线程去处理任务的
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```

调用processWorkerExit方法可以分为以下几种情况：
- 工作线程调用用户代码时发生异常，对于发生异常的工作线程，首先会将线程池线程数量减去1，然后尝试创建新的工作线程去工作
- 工作线程被指定了等待任务超时时间，在指定时间内没有获取到任务或者线程池状态被修改为SHUTDOWN，队列中没有任务了，那么会调用processWorkerExit方法
    - 如果核心没有指定可以超时，那么将判断当前线程池的线程数量是否大于核心线程数，如果大于，那么将不再去尝试创建的新的工作线程
    - 如果核心线程指定了超时时间，那么判断队列中是否存在等待的任务，如果存在，那么需要保留一个线程去处理，如果队列没有任务，那么全部线程都可以直接释放掉
- 如果线程池状态大于等于STOP，那么直接释放

## 三、线程池的关闭

### 3.1 shutdown

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        //使用JDK的安全框架检查当前应用是否有修改线程的权限
        checkShutdownAccess();
        //将线程池运行状态修改为SHUTDOWN
        advanceRunState(SHUTDOWN);
        //中断空闲的工作线程
        interruptIdleWorkers();
        //钩子函数，在ScheduledThreadPoolExecutor线程池中有实现，用于取消任务
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    //尝试将线程池状态修改成TEMINATE
    tryTerminate();
}
```
中断空闲工作线程代码：

```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        //循环职工表
        for (Worker w : workers) {
            Thread t = w.thread;
            //如果能够获取到工作线程锁，那么说明工作线程是空闲的
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    //中断，将唤醒阻塞在队列上的线程或者准备阻塞在队列上的工作线程
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```
线程池将由RUNNING -> SHUTDONW -> TIDYING -> TERMINATED

### 3.1 shutdownNow

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        //检查是否有修改线程的权限
        checkShutdownAccess();
        //将线程池状态修改为STOP
        advanceRunState(STOP);
        //中断空闲和正在工作的线程
        interruptWorkers();
        //获取等待执行的任务列表
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```
下面我们再来分析下tryTerminate方法

```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        //线程池还在未关闭，返回
        //已经处于TIDYING以上的关闭状态，返回
        //SHUTDOWN状态并且队列中还有需要处理的任务，返回
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
        //如果还有还在工作的线程，那么寻找到一个空闲线程进行中断
        //因为在设置线程池为SHUTDOWN时，有些线程可能在执行用户任务时，抛出错误，此时可能刚创建了新的线程，并且已经启动了新的线程，
        //只是还没有来得及进行中断，还有一些工作线程正在工作，这种状态下的工作线程也不会被中断
        //对于这样的线程还是会被阻塞在队列上的，
        if (workerCountOf(c) != 0) { // Eligible to terminate
            //中断一个空闲线程，这个线程退出时候也会调用tryTerminate方法，将关闭的信号传播到下一个空闲线程手里
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //将线程池状态变更为TIDYING
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    //调用钩子函数terminated
                    terminated();
                } finally {
                    //将线程池状态修改为最终状态TERMINATED
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```

线程池将由RUNNING -> STOP -> TIDYING -> TERMINATED

> Q：为什么设置线程池状态为SHUTDOWN的时候不能像STOP那么，管它工作线程有没有空闲，一股脑的全部中断不行吗？

> A：这个是取决作者对状态的定义，SHUTDOWN不会去中断正在执行任务的线程，STOP会中断正在执行的任务，假设这个工作线程正在处理与网络io或者其他阻塞型的任务时将
会抛出中断异常

## 四、FutureTask


### 4.1 run

上面我们对FutureTask进行了一些基础性的分析，现在我们深入到它的内容一探究竟，首先是run方法

```java
public void run() {
    //如果该任务已经处理或者被其他线程抢占，那么直接返回。有人可能会问了，我不就一个工作线程吗，会出现多个线程访问的情况吗？
    //注意这个FutureTask是可以单独拿出来运行的，它并不是ThreadPoolExecutor的附属品
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                //调用用户的逻辑
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                //出现异常，设置异常对象，修改任务状态 NEW -> COMPLETING -> EXCEPTIONAL并唤醒正在等待的线程
                setException(ex);
            }
            if (ran)
                //正常执行，设置返回值，修改任务状态 NEW -> COMPLETING -> NORMAL并唤醒正在等待的线程
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        //如果被中断取消
        if (s >= INTERRUPTING)
            //内部自旋等待状态变更为INTERRUPTED
            handlePossibleCancellationInterrupt(s);
    }
}
```

上面的方法在逻辑上并不是非常复杂，首先进行任务的抢占，如果成功，那么执行任务，执行任务如果抛出异常，那么将设置异常对象并修改状态 NEW -> COMPLETING -> EXCEPTIONAL 如果成功，那么设置状态为 NEW -> COMPLETING -> NORMAL，如果被中断，那么会在finally块中自旋式等待，等待状态由 INTERRUPTING -> INTERRUPTED

### 4.2 get

```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    //如果还没有完成，那么wait
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    //处理任务结果
    return report(s);
}
```
如果任务还未完成，那么进行阻塞等待

```java
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    //计算最后期限
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        //等待线程被中断，移除对应自己的节点，并抛出中断异常
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
        //如果任务完成，那么可以返回了，将对应的节点的线程关联关系断绝，避免重复unpark
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        //如果任务正在完成，自旋等待
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
            //自旋一次都还没有完成，那么开始创建对应的线程的等待节点准备挂起，但还会进行一次自旋
        else if (q == null)
            q = new WaitNode();
        else if (!queued)
            //头插法插入，cas替换waiter字段
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        else if (timed) {
            //如果存在wait超时时间，那么计算是否已经超时
            nanos = deadline - System.nanoTime();
            //超时的，将会把自己从等待队列中移除，并返回当前任务状态
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            //挂起指定时间
            LockSupport.parkNanos(this, nanos);
        }
        else
            LockSupport.park(this);
    }
}
```

如果线程被中断，将会抛出中断异常，实现获取不到执行结果的，将会将自己包装成等待节点，加入到等待队列中，直到任务完成被唤醒或者wait超时或者被中断。

### 4.3 cancel

```java
public boolean cancel(boolean mayInterruptIfRunning) {
    //取消的条件：状态必须由NEW -> INTERRUPTING或者CANCELLED
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    // in case call to interrupt throws exception
        if (mayInterruptIfRunning) {
            try {
                //中断
                Thread t = runner;
                if (t != null)
                    t.interrupt();
            } finally { // final state
                //中断取消的最终状态INTERRUPTED
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        //唤醒正在等待任务执行结果的线程们
        finishCompletion();
    }
    return true;
}
```
取消的条件：状态必须由NEW -> INTERRUPTING或者CANCELLED，通过mayInterruptIfRunning判断是中断取消还是直接修改状态为CANCELLED，修改成功后将会唤醒还在等待结果
的线程们

### 4.4 节点的移除

```java
private void removeWaiter(WaitNode node) {
    if (node != null) {
        //将要移除的节点关联的线程置空
        node.thread = null;
        retry:
        for (;;) {          // restart on removeWaiter race
            //从头往后遍历
            for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                s = q.next;
                //如果节点的没有被cancel，那么继续往后遍历
                if (q.thread != null)
                    pred = q;
                else if (pred != null) {
                    //如果q.thread == null，那么当前这个q节点需要被移除，直接将前一个节点连接到下一个节点
                    pred.next = s;
                    if (pred.thread == null) // check for race
                        //如果pred节点的线程在某一刻被置空，想要删除pred节点，那么从头开始
                        continue retry;
                }
                //如果pred == null，也就是说q.thread == null，那么需要将头节点移除
                else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                      q, s))
                    //从头开始遍历
                    continue retry;
            }
            break;
        }
    }
}
```
FutureTask的内容就说这么多，至此呢，我们基本上已经知道线程池是咋回事了。

## 五、参考资料

- JDK安全策略相关资料：https://www.ibm.com/developerworks/cn/java/j-lo-javasecurity/

## 六、总结

### 6.1 线程池的参数

线程池常用的参数有这么几个，核心线程数，最大线程数，线程空闲时存活的时间，阻塞队列，线程工厂，拒绝策略。
- 线程工厂：用于创建线程的工厂
- 核心线程数：用于指定空闲时至少有多少个线程。当然我们可以调用allowCoreThreadTimeOut方法，将核心线程设置为可以超时释放。
- 阻塞队列：当提交的任务的速度超过核心线程处理任务的速度时，将会把任务储存到阻塞队列里面。
- 最大线程数：指定线程池最大可以创建的线程个数，当提交的任务的速度超过核心线程处理任务的速度时，将会把任务储存到阻塞队列里面，如果队列满了，那么将会继续创建线程
  至最大线程数
- 拒绝策略：当队列已满，线程已经达到最大线程数时，任务将会调用拒绝策略处理，默认为AbortPolicy，抛出错误
- 线程空闲时存活的时间：线程处理完任务之后，从阻塞队列拿任务而没有任务阻塞住的时间，如果超过这个时间还是阻塞的，那么这个线程就会退出（如果我们没有设置
  allowCoreThreadTimeOut为true的话，那么只对超过核心线程数的线程起作用）。

### 6.2 线程池的状态

线程池使用一个int类型值ctl的高三位用于表示线程池状态，具体状态描述如下：

- RUNNING：一个负数，表示线程池正在正常运行
- SHUTDOW：线程池被关闭，此时无法提交新的任务（队列中存在的任务还是会执行），中断空闲线程，让空闲线程退出，不会中断正在执行任务的线程（每个Worker内部都持有lock，任务在执行开始后将会获取锁，所以我们只要tryLock一下就能知道这个线程是否正在执行任务），在退出线程时，如果发现队列中还有任务，那么会保留一个线程去处理队列中的任务
  ，直到空闲被中断。当线程池不存在活着的线程之后，线程池进入STOP阶段。
- STOP：线程池停止，此时无法提交新的任务，如果我们直接调用的是shutdowNow方法，那么会将队列中存在的任务进行取消。
- TIDYING：进入TERMINATED的过渡阶段，因为在进入TERMAINATED时需要调用terminated()方法，调用完才会将状态设置为TERMINATED，从某种程度上来讲TIDYING状态就像一把锁，只
  有成功将线程状态设置为TIDYING的才能调用terminated()方法。
- TERMINATED：线程都已经完全退出，任务都已经被执行完毕或者被取消。

### 6.3 线程池工作流程

1. 提交任务，首先判断是否超过核心线程，如果未超过，创建线程去处理这个任务，任务执行完毕之后，这个线程会去队列中取任务，然后判断当前是否已经超过了核心线程数，超过
   的会设置超时时间（如果设置了允许核心线程超时，那么核心线程也会超时）。
2. 继续提交任务，如果线程数未超过核心线程数，那么重复步骤一的动作，如果超过了核心线程数，那么这个任务将存放到阻塞队列中。
3. 继续提交任务，如果核心线程处理不过来，队列的容量也被填满了，那么会判断当前线程池线程数是否超过了最大线程数，如果未超过，创建新的线程去处理任务，处理完之后准备
   到阻塞队列中去取任务，如果没有任务将会被阻塞一段时间（这个时间是我们创建线程池时设置的），获取任务超时后，超过核心线程数的线程将被释放。
4. 继续提交任务，如果线程池的线程数达到了最大线程数，它的处理速度依然无法赶上提交任务的速度，那么提交的任务将交由拒绝策略处理。

### 6.4 FutureTask

FutureTask实现了Future，它代表一个任务，它记录了这个任务的执行状态与执行结果，具体的任务状态描述如下：
- NEW：这个状态表示任务刚被创建或者正在执行中
- COMPLETING：这个状态表示任务已经执行完毕，正在设置执行结果（也许是正常返回结果，也许是抛出的异常）。
- NORMAL：表示任务正常完成。
- EXCEPTIONAL：表示任务执行时抛出了用户未捕获的异常
- CANCELLED：表示任务被取消，任务被取消后，如果调用get方法，将抛出取消异常
- INTERRUPTING：表示正在中断中，将会调用执行任务线程的interrupt()方法，最后将任务状态设置为INTERRUPTED
- INTERRUPTED：表示任务被中断，调用get方法时将抛出取消异常