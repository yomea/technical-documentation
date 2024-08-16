## 一、基础

### 1.1 原理

#### 1.1.1 ForkJoinPool工作队列原理

在分析ForkJoinPool之前，我们先来说说它的实现原理<br />
下面是ForkJoinPool工作队列组的示意图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/14bcde3f07659345c14fb25d5236f0bd.png)

重点说明：

- 工作队列组的容量为2次幂的值，索引为偶数的，用于存放外部线程提交过来的任务，索引下标为奇数的，存放的是ForkJoinPool内部创建的线程fork出来的任务，工作队列组是一个数组，其元素的类型为WorkQueue，WorkQueue内部又维护了一个数组，这个数组元素的类型FutureJoinTask

- 偶数索引位的任务属于共享任务，由工作线程去竞争获取，获取方式为FIFO

- 奇数索引位的任务从属于某个工作线程，其内部的任务通常由fork方法添加

- 工作线程可以去偷取其他工作队列中任务，偷取方式为FIFO

- 工作线程执行自身任务的取值方式默认为LIFO（可以修改成FIFO，但是偷取任务是FIFO的，为了减少竞争，我们最好使用默认的LIFO）

#### 1.1.2 FutureJoinTask任务划分

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4f2fedfbfd7b5f1ab3923f0e7187ace9.png)

当一个ForkJoinPool的线程获取到一个任务的时候，如果这个任务是可以拆分任务的，也就是调用了fork方法，那么它将会把这些子任务push到自己的工作队列中

#### 1.1.3 实现原理参考资料
- Doug Lea论文英文版：http://gee.cs.oswego.edu/dl/papers/fj.pdf
- Doug Lea论文中文版：http://ifeve.com/java-fork-join-framework/

### 1.2 类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/aaf9cabd12b50c04d35d91093babe369.png)

#### 1.2.1 Executor

```java
public interface Executor {

    //提交一个实现了Runnable的任务，异步执行
    void execute(Runnable command);
}
```

#### 1.2.2 ExecutorService

以下对于方法的描述全是基于ForkJoinPool的实现说明。

```java
public interface ExecutorService extends Executor {

    //关闭任务服务，将线程池标记为SHUTDOWN，不再接收新的任务，内部已经提交的任务依然会继续执行，甚至在线程数未满的情况下，还可以
    //创建新的工作线程
    void shutdown();

    //立即关闭任务服务，先将线程池标记为SHUTDOWN，然后再标记为STOP，此时不再接收新的任务，不能创建新的工作线程
    //返回值List<Runnable>表示返回不需要执行的任务，但是ForkJoinPool变为STOP状态后,队列中的任务会被取消
    //最后进入TERMINATED状态，没有工作线程，如果存在任务，那么也是已经被取消的
    List<Runnable> shutdownNow();

    //检查当前任务服务是否已经被关闭，只需判断当前线程池的状态是否已经被标记为SHUTDOWN即可
    boolean isShutdown();

    //线程池状态为TERMINATED
    boolean isTerminated();

    //阻塞线程直到wait超时或者被唤醒发现发现已经TERMINATED
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    //提交任务异步执行，返回代表task的Future对象
    <T> Future<T> submit(Callable<T> task);

    //提交任务异步执行，返回代表task的Future对象，执行完毕之后，将传入的result值设置到Future中
    //Runnable没有返回值，内部实现是将task包装成一个返回result的Future
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

#### 1.2.3 ForkJoinTask

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/49b8bca1be70f219e5bc1a16082b864a.png)

我们提交的任务最终都会封装成一个ForkJoinTask扔到工作队列中去等待执行，ForkJoinTask是一个抽象类，它有三个抽象方法

```java
//用于获取任务执行后的结果
public abstract V getRawResult();

//设置结果
protected abstract void setRawResult(V value);

//执行任务
protected abstract boolean exec();
```
exec是执行用户逻辑的地方，至于怎么去执行用户的逻辑，由子类去实现，执行成功的时候返回true，否则返回false，setRawResult方法用于设置运行用户代码后的返回值，
setRawResult方法用于获取任务执行后的结果。

ForkJoinTask内部定义了几个内部实现
- AdaptedRunnableAction：用于封装Runable对象，getRawResult方法返回null，setRawResult是空实现，exec方法直接调用Runable的run方法
- AdaptedCallable：用于封装Callable对象，setRawResult方法用于设置Callable执行后的返回值，getRawResult返回Callable执行后的返回值，exec调用Callable的call方法
- AdaptedRunnable：用于封装Runable对象，可以通过构造器设置Result结果集
- ExceptionNode：用于记录异常信息，继承自WeakReference，弱引用对象为ForkJoinTask
- RunnableExecuteAction：与AdaptedRunnableAction的实现很像，不同的是它实现了抛出异常的方法

ForkJoinTask的重点成员变量说明

```java
//任务的状态，完成 or 取消 or 执行出错
volatile int status; // accessed directly by pool and workers

//获取任务状态的掩码
static final int DONE_MASK   = 0xf0000000;  // mask out non-completion bits

//表示任务正常执行完成，一个负数
static final int NORMAL      = 0xf0000000;  // must be negative

//表示任务被取消了，负数
static final int CANCELLED   = 0xc0000000;  // must be < NORMAL

//表示任务执行发生了异常，负数
static final int EXCEPTIONAL = 0x80000000;  // must be < CANCELLED

//在获取任务结果时，如果任务还没有执行完成并且需要阻塞的话，那么需要先将status设置为SIGNAl，然后在阻塞
//SIGNAL用于告诉其他线程，这里有线程被阻塞了，需要传递signal信号
//比如在执行完任务后，那个执行完任务的线程会查看是否设置了SIGNAL状态，如果设置了，才会去唤醒其他被阻塞的任务
static final int SIGNAL      = 0x00010000;  // must be >= 1 << 16

//低16位掩码，SMASK（short mask）
//用于给任务打上标签，status的低16用于记录任务的标签
//使用到这个变量的方法为setForkJoinTaskTag方法，但这个方法目前未被使用
static final int SMASK       = 0x0000ffff;  // short bits for tags

//异常列表，在任务执行抛出错误的时候，用于记录异常，ExceptionNode继承了弱引用，弱引用对象为ForkJoinTask
private static final ExceptionNode[] exceptionTable;

//操作异常列表时的同步锁
private static final ReentrantLock exceptionTableLock;

//弱引用队列，将作为参数传入到ExceptionNode中，当ExceptionNode中的task可达性发生变化时，JVM将会将这个准备回收的task
//入队列
private static final ReferenceQueue<Object> exceptionTableRefQueue;

//异常列表容量使用2次幂的值
private static final int EXCEPTION_MAP_CAPACITY = 32;
```

ForkJoinTask的重点方法说明

```java
//执行任务方法
final int doExec() {
    int s; boolean completed;
    //从上面的字段说明来看，只要任务执行完成，不管是出错还是正常完成，它的状态都是负数
    //所以当前status大于等于零的时候，这个任务就还没有执行完成
    if ((s = status) >= 0) {
        try {
            //exec是抽象方法，由子类实现
            completed = exec();
        } catch (Throwable rex) {
            //发生异常，记录异常信息并且将任务状态修改为EXCEPTIONAL
            return setExceptionalCompletion(rex);
        }
        if (completed)
            //执行成功，将任务状态修改为NORMAL
            s = setCompletion(NORMAL);
    }
    return s;
}


//修改任务状态方法
private int setCompletion(int completion) {
    for (int s;;) {
        //任务完成，并且状态已经被修改成完成状态了，就不再操作了
        if ((s = status) < 0)
            return s;
        //修改状态
        if (U.compareAndSwapInt(this, STATUS, s, s | completion)) {
            //无符号右移16位，注意这个s记录的是执行U.compareAndSwapInt前的数据，也就是status >= 0的数据
            //高16位可能是零，也可能是1，当前为1的时候，那么表示有线程将status修改成了SIGNAL
            //需要唤醒这个被阻塞的线程
            if ((s >>> 16) != 0)
                synchronized (this) { notifyAll(); }
            return completion;
        }
    }
}



//fork方法
public final ForkJoinTask<V> fork() {
    Thread t;
    //检查当前fork子任务的线程是否为ForJoinPool的线程
    //如果是，那么将这个子任务push到当前线程的工作队列中
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
        ((ForkJoinWorkerThread)t).workQueue.push(this);
    else
        //如果是外部线程调用的，那么直接push到通用ForJoinPool线程池中去执行
        ForkJoinPool.common.externalPush(this);
    return this;
}



//join方法
public final V join() {
    int s;
    if ((s = doJoin() & DONE_MASK) != NORMAL)
        //如果状态s为CANCELLED，那么将会抛出取消异常
        //如果状态为EXCEPTIONAL，那么会从异常列表中找到对应任务的异常信息，如果获取异常信息的线程是当初那个抛出异常的线程，那么直接返回原始的异常对象
        //否则将按照原来的异常类型new出一个新的异常，将原来的异常对象作为参数传入，为什么要这么做？
        //如果抛出一个其他线程产生的异常对象，那么这个异常对象记录的线程栈轨迹是原来产生这个异常的线程栈轨迹，此处重新包装是为了将当前线程的线程轨迹记录下来
        reportException(s);
    //获取任务的执行结果
    return getRawResult();
}

//doJoin方法
private int doJoin() {
    int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
    //首先判断当前任务是否已经完成了，如果已经完成了，那么直接返回即可
    //如果当前调用这个任务的线程是ForkJoinPool的线程，首先判断这个task是否就是在栈顶，如果在栈顶，那么尝试去pop这个任务，然后执行
    //如果不是栈顶任务或者被其他工作线程给偷走了，那么需要进入寻找偷取者，礼尚往来，你偷我的任务，我也偷你的任务，如果偷不着进行阻塞的逻辑，这部分逻辑后面会分析
    //不是ForkJoinPool的线程交由common ForkJoinPool处理
    return (s = status) < 0 ? s :
        ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
        (w = (wt = (ForkJoinWorkerThread)t).workQueue).
        tryUnpush(this) && (s = doExec()) < 0 ? s :
        wt.pool.awaitJoin(w, this, 0L) :
        externalAwaitDone();
}


//取消
public boolean cancel(boolean mayInterruptIfRunning) {
    //将任务状态修改成CANCELLED，如果有必要的话，还会唤醒阻塞中的线程
    return (setCompletion(CANCELLED) & DONE_MASK) == CANCELLED;
}

//获取结果集
public final V get() throws InterruptedException, ExecutionException {
    //如果调用这个方法的线程是ForkJoinPool中的线程，那么通过doJoin执行或者等待任务
    //如果是外部线程，那么在任务完成之前进行阻塞
    int s = (Thread.currentThread() instanceof ForkJoinWorkerThread) ?
        doJoin() : externalInterruptibleAwaitDone();
    Throwable ex;
    //任务被取消，抛出取消异常，DONE_MASK是获取状态的掩码
    if ((s &= DONE_MASK) == CANCELLED)
        throw new CancellationException();
    //如果执行时发生了异常，那么从异常列表中获取异常，与上面reportException中处理异常的方式一样
    if (s == EXCEPTIONAL && (ex = getThrowableException()) != null)
        throw new ExecutionException(ex);
    //获取任务执行结果
    return getRawResult();
}

```
#### 1.2.4 AbstractExecutorService

AbstractExecutorService是一个模板方法，实现了一些通用的方法，大部分接口方法都已经被它实现，只有以下方法没有实现

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d980af7331f1dbfc2758cf1846327170.png)

#### 1.2.5 WorkQueue

WorkQueue是ForkJoinPool的工作队列，用于维护任务，记录下标，如果有工作线程，还会记录工作线程的状态等等，以下为WorkQueue的字段说明

```java
//初始任务列表大小，2次幂的值，8192
//一开始分这么大，主要是为了避免过多hash冲突导致的扩容
//如果我们提交的任务拆分的小任务比较多，初始容量不够大的话，将会导致每次创建一个
//新的工作线程，等它发现容量不够时就要发生扩容，容易造成性能问题
static final int INITIAL_QUEUE_CAPACITY = 1 << 13;

//任务列表最大容量
static final int MAXIMUM_QUEUE_CAPACITY = 1 << 26;

//扫描状态：高16位用于版本计数，低16位用于表示workQueue的扫描状态
//1.如果是偶数位的workQueue，那么它的扫描状态为负数，表示不活跃的
//2.如果是奇数位置上的workQueue，这个scanState的值通常就是workQueue在工作队列组中的下标，此时表示工作线程正在寻找任务执行
//如果scanState为负数，那么表示这个工作线程没有找到任务，即将被挂起
//如果scanState由计数变成了偶数，那么表示工作线程正在执行任务
volatile int scanState;    // versioned, <0: inactive; odd:scanning

//当WorkQueue对应的工作线程扫描不到任务时，它会将自己的scanState设置到ctl的低32位中
//在设置之前，得将前一个挂起的工作线程的scanState记录下来
int stackPred;             // pool stack (ctl) predecessor

//用于记录当前workQueue的工作线程偷取了多少任务
int nsteals;               // number of steals

//用于保存随机值或者其他偷取了自己任务的WorkQueue的下标
int hint;                  // randomization and stealer index hint

//高16位用于记录队列模式（FIFO或者LIFO或者共享队列）
//低16位用于记录当前WorkQueue在工作队列组中的下标
int config;                // pool index and mode

//队列锁，用于同步，为1表示当前队列被锁定
//小于0，通常是设置成-1，表示当前队列已经被取消注册或者线程池被关闭
//为0表示未被锁定
volatile int qlock;        // 1: locked, < 0: terminate; else 0

//栈底位置，作者没有直接使用数组下标获取值的方式，而是通过计算内存位置的方式去操作数组
//主要为了使用原子操作，如果使用数组下标的方式，那不得不加粗粒度的锁，为了性能考虑
//作者直接使用了内存地址操作
volatile int base;         // index of next slot for poll

//栈顶位置
int top;                   // index of next slot for push

//用于存放任务的数组
ForkJoinTask<?>[] array;   // the elements (initially unallocated)

//ForkJoinPool线程池，表示当前WorkQueue是属于哪个线程池的
final ForkJoinPool pool;   // the containing pool (may be null)

//表示当前WorkQueue从属于哪个工作线程，对于外部提交的任务而创建的WorkQueue，它这个owner为空，它是共享的
final ForkJoinWorkerThread owner; // owning thread or null if shared

//当工作线程扫描不到任务时，将被挂起，此时这个值就是那个被挂起的线程
volatile Thread parker;    // == owner during call to park; else null

//记录当前需要join的任务，我们在执行可拆分任务的时候会fork一个子任务，如果我们的当前的大任务依赖这个子任务，那么通常是调用
//这个子任务的join方法等待子任务的执行返回结果
volatile ForkJoinTask<?> currentJoin;  // task being joined in awaitJoin

//记录偷取到的任务
volatile ForkJoinTask<?> currentSteal; // mainly used by helpStealer

//ForkJoinTask<?>[] array;的基准位置
private static final int  ABASE;

//表示任务数组中两个相邻元素的内存间距的log（间距），底数是2
//比如array[1]与array[0]的内存间距为4，那么这个ASHIFT的值为2
//那么  数组下标1的相对 ABASE的距离 = ABASE + 1 * 4 == ABASE的距离 = ABASE + 1 << 2;
private static final int  ASHIFT;

//top栈顶的内存偏移地址
private static final long QTOP;

//qlock字段的偏移地址
private static final long QLOCK;

//currentSteal字段的偏移地址
private static final long QCURRENTSTEAL;

```
重点方法分析

```java
WorkQueue(ForkJoinPool pool, ForkJoinWorkerThread owner) {
    this.pool = pool;
    this.owner = owner;
    // Place indices in the center of array (that is not yet allocated)
    //默认栈底和栈顶为容量的一半，将数据存放在数组的中间，为什么要存放在中间呢？？？不能从0开始吗
    base = top = INITIAL_QUEUE_CAPACITY >>> 1;
}
```

添加元素

```java
final void push(ForkJoinTask<?> task) {
    ForkJoinTask<?>[] a; ForkJoinPool p;
    int b = base, s = top, n;
    if ((a = array) != null) {    // ignore if queue removed
        int m = a.length - 1;     // fenced write for task visibility
        //对于一个2次幂容量的数组进行求余公式为 s % a.length = s & (a.length - 1) = s % m
        //((m & s) << ASHIFT) + ABASE 计算索引，为什么这么计算，我们在分析WorkQueue的ASHIFT时进行详细的说明
        U.putOrderedObject(a, ((m & s) << ASHIFT) + ABASE, task); //将任务添加到对应索引上
        //更新top字段的值
        U.putOrderedInt(this, QTOP, s + 1);
        //尝试添加或者唤醒线程去处理任务
        //为什么要判断队列任务为1以下时才唤醒或者添加工作线程？
        //首先在没有任务的情况下，很可能这个队列是新建的，那么我们有理由去尝试添加更多的线程去处理任务
        //再者，如果队列以前就有，但是后面任务都消费完了，那么之前存在的线程在获取不到任务的情况下是会被阻塞的，这里添加任务，那么
        //就得告诉那些被阻塞的线程，你可以来消费了，那我能不能每次添加都去尝试唤醒线程？可以，但是没有必要，一次就够了，只要有任务
        //线程就不会阻塞
        if ((n = s - b) <= 1) {
            if ((p = pool) != null)
                //唤醒或者添加工作线程
                p.signalWork(p.workQueues, this);
        }
        //如果元素个数已经达到了最大容量，那么需要扩容
        else if (n >= m)
            growArray();
    }
}
```
push方法在push一个任务的时候，不是通过数组下标的方式去push值，而是通过计算内存偏移地址的方式去定位这个任务该放在何处，计算的方式为

```java
((m & s) << ASHIFT) + ABASE

s为栈顶位置，我们对一个2次幂的值进行求余的公式为 s % 数组大小 == s & (数组大小 - 1)，
此处 m = (数组大小 - 1)，故计算出的数组索引下标为 m & s

ABASE是通过Unsafe计算出的任务数组基准偏移地址，以下是计算ForkJoinTask数组的基准偏移地址的代码

Class<?> ak = ForkJoinTask[].class;
ABASE = U.arrayBaseOffset(ak);

ASHIFT也是通过Unsafe计算出来的，具体操作如下：

//获取ForkJoinTask数组两个相邻元素之间的间距
int scale = U.arrayIndexScale(ak);
//如果不是2次幂的值，抛出错误
if ((scale & (scale - 1)) != 0)
    throw new Error("data type scale not a power of two");
    
//这段代码相当于 log(scale)，如果scale是4，那么这个ASHIFT就是2
// 比如4的二进制为 000...0100 前导零的个数为29，31减去29就是2
ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);

```

如果添加任务后，任务个数为1，那么需要尝试去添加或者唤醒工作线程，为什么要这么做呢？
- 在没有任务的情况下，很可能这个队列是新建的，那么我们有理由去尝试添加更多的线程去处理任务
- 如果队列以前就有，但是后面任务都消费完了，那么之前存在的线程在获取不到任务的情况下是会被阻塞的，这里添加任务，那么就得告诉那些被阻塞的线程，你可以来消费
  了，那我能不能每次添加都去尝试唤醒线程？可以，但是没有必要，一次就够了，只要有任务线程就不会阻塞

如果在添加任务的时候，发现任务个数已经达到了队列容量的上限，那么就得扩容了

```java
final ForkJoinTask<?>[] growArray() {
    //记录老数组
    ForkJoinTask<?>[] oldA = array;
    //两倍扩容
    int size = oldA != null ? oldA.length << 1 : INITIAL_QUEUE_CAPACITY;
    //太大了，抛错
    if (size > MAXIMUM_QUEUE_CAPACITY)
        throw new RejectedExecutionException("Queue capacity exceeded");
    int oldMask, t, b;
    ForkJoinTask<?>[] a = array = new ForkJoinTask<?>[size];
    //老数组里面有任务
    if (oldA != null && (oldMask = oldA.length - 1) >= 0 &&
        (t = top) - (b = base) > 0) {
        int mask = size - 1;
        do { // emulate poll from old array, push to new array
            ForkJoinTask<?> x;
            //从栈底开始，把老数组中的任务搬到新数组中去
            int oldj = ((b & oldMask) << ASHIFT) + ABASE;
            int j    = ((b &    mask) << ASHIFT) + ABASE;
            x = (ForkJoinTask<?>)U.getObjectVolatile(oldA, oldj);
            if (x != null &&
                U.compareAndSwapObject(oldA, oldj, x, null))
                U.putObjectVolatile(a, j, x);
        } while (++b != t);
    }
    return a;
}
```
将任务数组扩容为原来的两倍，然后从栈底开始将老数组中的任务迁移到新数组中去

下面我们继续看到pop操作

```java
final ForkJoinTask<?> pop() {
    ForkJoinTask<?>[] a; ForkJoinTask<?> t; int m;
    //队列已经初始化
    if ((a = array) != null && (m = a.length - 1) >= 0) {
        //队列中有值
        for (int s; (s = top - 1) - base >= 0;) {
            //从栈顶取值 LIFO
            long j = ((m & s) << ASHIFT) + ABASE;
            //没有获取到任务，在ForkJoinPool中，只有工作线程在获取其所属工作队列的任务时，使用的是pop操作，看起来似乎没有竞态问题
            //实际上是有的，在工作线程其所属工作队列的任务只剩一个的时候，可能会与偷取者发生竞争。
            //由于一个线程获取任务，然后置空，再更新栈顶位置的操作不是原子的，此处获取的值为空也就不足为奇了
            //由于是最后一个任务对象，然后还被其他工作线程抢走了，那么就直接break退出
            if ((t = (ForkJoinTask<?>)U.getObject(a, j)) == null)
                break;
            //获取栈顶任务，然后将栈顶置空，更新栈顶位置
            if (U.compareAndSwapObject(a, j, t, null)) {
                U.putOrderedInt(this, QTOP, s);
                return t;
            }
        }
    }
    return null;
}
```
首先判断队列是否已经初始化，如果已经初始化再判断队列中是否有任务，如果有任务尝试去获取任务，获取任务可能存在失败的情况，主要是因为多线程操作的缘故。
在ForkJoinPool中，只有工作线程在获取其所属工作队列的任务时，使用的会是pop操作，这么一看似乎没有竞态问题，那到底有没有呢？ <br />
答案是有的，在工作线程其所属工作队列的任务只剩一个的时候，可能会与偷取者发生竞争关系，如果偷取者先抢到了任务并将对应偏移位置的内容置为了空，还没来得及将top
更新，此时做pop操作的线程判断 (s = top - 1) - base >= 0 成立，那么就会获取到一个null值，获取到null很显然队列里已经没有任务了，直接break退出即可。


poll操作

```java
final ForkJoinTask<?> poll() {
    ForkJoinTask<?>[] a; int b; ForkJoinTask<?> t;
    //任务队列已经初始化并且有任务
    while ((b = base) - top < 0 && (a = array) != null) {
        //从栈底开始获取值 FIFO
        int j = (((a.length - 1) & b) << ASHIFT) + ABASE;
        //获取对应偏移位置的任务
        t = (ForkJoinTask<?>)U.getObjectVolatile(a, j);
        //由于poll存在竞争，所以此处会判断当前base是否已经发生了变化
        if (base == b) {
            //获取任务，然后更新对应偏移位置为null，最后更新base值的操作不是原子的
            //所以此处可能出现获取到的task为null的情况
            if (t != null) {
                //CAS更新
                if (U.compareAndSwapObject(a, j, t, null)) {
                    base = b + 1;
                    return t;
                }
            }
            //任务队列已经空了，直接退出
            else if (b + 1 == top) // now empty
                break;
        }
    }
    return null;
}
```
首先判断当前任务队列是否已经初始化并且队列中有任务可取，如果领取到了任务还需要原子操作去更新对应偏移位置的值为null，更新成功后将base加1，经过这些步骤之后，
这个任务才真正属于你

#### 1.2.6 EmptyTask

```java
static final class EmptyTask extends ForkJoinTask<Void> {
    private static final long serialVersionUID = -7721805057305804111L;
    //直接将任务状态标记为正常完成状态
    EmptyTask() { status = ForkJoinTask.NORMAL; } // force done
    public final Void getRawResult() { return null; }
    public final void setRawResult(Void x) {}
    public final boolean exec() { return true; }
}
```
EmptyTask用于占位，在处理join的时候在 java.util.concurrent.ForkJoinPool.WorkQueue#tryRemoveAndExec 方法中被使用，这个方法tryRemoveAndExec获取的任务不一定在
栈顶，可能在中间某个地方，取走这个任务后，为了不移动其他的任务，使用了EmptyTask进行替换。那为啥不替换成null啊？ForkJoinPool从队列中获取任务基本都是以null进
行判断取值的成功与否的，如果替换成null，工作线程会认为自己抢任务失败，不会去修改栈基或者栈顶位置，这就导致一直获取不到任务。

#### 1.2.7 ForkJoinWorkerThreadFactory

```java
public static interface ForkJoinWorkerThreadFactory {
    
    public ForkJoinWorkerThread newThread(ForkJoinPool pool);
}
```
ForkJoinWorkerThreadFactory用于创建专属于ForkJoinPoll的线程，目前有两个实现 DefaultForkJoinWorkerThreadFactory 与 InnocuousForkJoinWorkerThreadFactory
DefaultForkJoinWorkerThreadFactory没做啥高级操作，直接new ForkJoinWorkerThread，而InnocuousForkJoinWorkerThreadFactory主要是做了一些安全权限的处理，将父
线程的访问上下文设置到ForkJoinWorkerThread，以便能够有权限执行某些操作，此处不会展开描述。

#### 1.2.8 ManagedBlocker

阻塞管理器，通常用于FutureTask的get方法，用户可以自定义是否需要阻塞线程，它不是本章重点，在分析 Phaser 的时候会比较详细的探讨。


## 二、总结
### 2.1 原理

- 工作队列组的容量为2次幂的值，索引为偶数的，用于存放外部线程提交过来的任务，索引下标为奇数的，存放的是ForkJoinPool内部创建的线程fork出来的任务，工作队列组是一个数组，其元素的类型为WorkQueue，WorkQueue内部又维护了一个数组，这个数组元素的类型FutureJoinTask

- 偶数索引位的任务属于共享任务，由工作线程去竞争获取，获取方式为FIFO

- 奇数索引位的任务从属于某个工作线程，其内部的任务通常由fork方法添加

- 工作线程可以去偷取其他工作队列中任务，偷取方式为FIFO

- 工作线程执行自身任务的取值方式默认为LIFO（可以修改成FIFO，但是偷取任务是FIFO的，为了减少竞争，我们最好使用默认的LIFO）

### 2.2 FutureJoinTask

FutureJoinTask具有以下状态，这些状态用一个int类型的state表示，高16位表示状态，低16位表示标签：

- NORMAL：表示任务正常完成
- CANCELLED：表示任务被取消
- EXCEPTIONAL：表示任务执行时抛出了未捕获的异常，这样的异常会记录到异常列表中
- SIGNAL：表示任务执行完毕后需要唤醒因等待任务执行而阻塞的线程

### 2.3 WorkQueue


```java
//初始任务列表大小，2次幂的值，8192
//一开始分这么大，主要是为了避免过多hash冲突导致的扩容
//如果我们提交的任务拆分的小任务比较多，初始容量不够大的话，将会导致每次创建一个
//新的工作线程，等它发现容量不够时就要发生扩容，容易造成性能问题
static final int INITIAL_QUEUE_CAPACITY = 1 << 13;

//任务列表最大容量
static final int MAXIMUM_QUEUE_CAPACITY = 1 << 26;

//扫描状态：高16位用于版本计数（可防止ABA问题），低16位用于表示workQueue的扫描状态
//1.如果是偶数位的workQueue，那么它的扫描状态为负数，表示不活跃的
//2.如果是奇数位置上的workQueue，这个scanState的值通常就是workQueue在工作队列组中的下标，此时表示工作线程正在寻找任务执行
//如果scanState为负数，那么表示这个工作线程没有找到任务，即将被挂起
//如果scanState由奇数变成了偶数，那么表示工作线程正在执行任务，busy
volatile int scanState;    // versioned, <0: inactive; odd:scanning

//当WorkQueue对应的工作线程扫描不到任务时，它会将自己的scanState设置到ctl的低32位中
//在设置之前，得将前一个挂起的工作线程的scanState记录下来
//用这个属性和ctl配合就形成一个栈
int stackPred;             // pool stack (ctl) predecessor

//用于记录当前workQueue的工作线程偷取过多少任务执行
int nsteals;               // number of steals

//用于保存随机值（当初通过随机数确定下标用的随机数）或者其他偷取了自己任务的WorkQueue的下标
int hint;                  // randomization and stealer index hint

//高16位用于记录队列模式（FIFO或者LIFO或者共享队列）
//低16位用于记录当前WorkQueue在工作队列组中的下标
int config;                // pool index and mode

//队列锁，用于同步，为1表示当前队列被锁定
//小于0，通常是设置成-1，表示当前队列已经被取消注册或者线程池被关闭
//为0表示未被锁定
volatile int qlock;        // 1: locked, < 0: terminate; else 0

//栈底位置，作者没有直接使用数组下标获取值的方式，而是通过计算内存位置的方式去操作数组
//主要为了使用原子操作，如果使用数组下标的方式，那不得不加粗粒度的锁，为了性能考虑
//作者直接使用了内存地址操作
volatile int base;         // index of next slot for poll

//栈顶位置
int top;                   // index of next slot for push

//用于存放任务的数组
ForkJoinTask<?>[] array;   // the elements (initially unallocated)

//ForkJoinPool线程池，表示当前WorkQueue是属于哪个线程池的
final ForkJoinPool pool;   // the containing pool (may be null)

//表示当前WorkQueue从属于哪个工作线程，对于外部提交的任务而创建的WorkQueue，它这个owner为空，它是共享的
final ForkJoinWorkerThread owner; // owning thread or null if shared

//当工作线程扫描不到任务时，将被挂起，此时这个值就是那个被挂起的线程
volatile Thread parker;    // == owner during call to park; else null

//记录当前需要join的任务，我们在执行可拆分任务的时候会fork一个子任务，如果我们的当前的大任务依赖这个子任务，那么通常是调用
//这个子任务的join方法等待子任务的执行返回结果
volatile ForkJoinTask<?> currentJoin;  // task being joined in awaitJoin

//记录偷取到的任务
volatile ForkJoinTask<?> currentSteal; // mainly used by helpStealer

//ForkJoinTask<?>[] array;的基准位置
private static final int  ABASE;

//表示任务数组中两个相邻元素的内存间距的log（间距），底数是2
//比如array[1]与array[0]的内存间距为4，那么这个ASHIFT的值为2
//那么  数组下标1的相对 ABASE的距离 = ABASE + 1 * 4 == ABASE的距离 = ABASE + 1 << 2;
private static final int  ASHIFT;

//top栈顶的内存偏移地址
private static final long QTOP;

//qlock字段的偏移地址
private static final long QLOCK;

//currentSteal字段的偏移地址
private static final long QCURRENTSTEAL;


```

这章我们主要讲解了ForkJoinPool的原理和与之相关的类，为我们学习下一章ForkJoinPool的实现打下了基础。