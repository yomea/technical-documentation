## 一、前言
在第15节中我们讲解了ForkJoinPool的实现原理和与之相关的类，为我们学习ForkJoinPool
的实现打下了基础。

## 二、原理实现

### 2.1 ForkJoinPool字段说明

```java
//低16位掩码，这也是WorkQueue在工作队列组中的最大索引值
static final int SMASK        = 0xffff;        // short bits == max index

//最大工作线程数
static final int MAX_CAP      = 0x7fff;        // max #workers - 1

//偶数掩码，第一个bit为0，所以任何值与它位与都会是一个偶数
static final int EVENMASK     = 0xfffe;        // even short bits

//用于计算偶数下标，SQMASK值为126,0~126之间有64个偶数
//所以偶数位的槽数只有64个
static final int SQMASK       = 0x007e;        // max 64 (even) slots

// Masks and units for WorkQueue.scanState and ctl sp subfield
//用于检测工作线程是否正在执行任务的掩码
static final int SCANNING     = 1;             // false when running tasks

//负数，用于WorkQueue.scanState，与队列的scanState进行位或可以将scanState变成负数，表示
//工作线程扫描不到任务，进入不活跃状态，将可能被阻塞
static final int INACTIVE     = 1 << 31;       // must be negative

//版本计数，用于WorkQueue.scanState，避免ABA问题
static final int SS_SEQ       = 1 << 16;       // version count

// Mode bits for ForkJoinPool.config and WorkQueue.config
//用于ForkJoinPool.config 和 WorkQueue.config的掩码
//获取队列模式
static final int MODE_MASK    = 0xffff << 16;  // top half of int

//表示后进先出队列模式
static final int LIFO_QUEUE   = 0;

//表示先进先出模式
static final int FIFO_QUEUE   = 1 << 16;

//表示共享模式
static final int SHARED_QUEUE = 1 << 31;       // must be negative


//默认创建ForkJoinWorkerThread的工厂类
public static final ForkJoinWorkerThreadFactory
        defaultForkJoinWorkerThreadFactory;
        
//线程修改许可，用于检测代码是否具有修改线程状态的权限
private static final RuntimePermission modifyThreadPermission;

//通用ForkJoinPool池，一旦ForkJoinPool发生类构造器的初始化，这个值就不会为空
//因为它是在静态块中创建的
static final ForkJoinPool common;

//通用ForkJoinPool池的并发数
static final int commonParallelism;

//通用ForkJoinPool池最大线程数
private static int commonMaxSpares;

//用于创建ForkJoinPool池的个数计数，在ForkJoinPool的构造器中使用
private static int poolNumberSequence;

//在工作线程扫描不到任务并且活跃线程为零时，用于计算阻塞时间
private static final long IDLE_TIMEOUT = 2000L * 1000L * 1000L; // 2sec

//通过 IDLE_TIMEOUT 计算出来的deadline会减去 TIMEOUT_SLOP
//主要为了屏蔽系统定时器唤醒的延时时间
private static final long TIMEOUT_SLOP = 20L * 1000L * 1000L;  // 20ms

//通用ForkJoinPool池的默认最大线程数
private static final int DEFAULT_COMMON_MAX_SPARES = 256;

//自旋次数，有两个地方使用了，一个是工作线程扫描任务未扫到准备要阻塞时的最后挣扎
//另一个是在awaitRunStateLock方法中使用，用于给runState加锁
private static final int SPINS  = 0;

//随机数种子生成器的增量
private static final int SEED_INCREMENT = 0x9e3779b9;

//低32位掩码
private static final long SP_MASK    = 0xffffffffL;

//高32位掩码
private static final long UC_MASK    = ~SP_MASK;

// Active counts
//正在运行中的激活线程数偏移位数，也就是高16表示活跃线程数
//所谓活跃就是WorkQueue.scanState为正数（正在扫描任务或者正在执行任务）
private static final int  AC_SHIFT   = 48;

//高16活跃线程数计数单元，高16加1
private static final long AC_UNIT    = 0x0001L << AC_SHIFT;

//高16活跃线程掩码
private static final long AC_MASK    = 0xffffL << AC_SHIFT;

// Total counts
//总线程数计数偏移位，高32的低16记录的是总线程数
private static final int  TC_SHIFT   = 32;

//总线程数计数单元
private static final long TC_UNIT    = 0x0001L << TC_SHIFT;

//总线程数掩码
private static final long TC_MASK    = 0xffffL << TC_SHIFT;

//总线程数的最高位掩码，用于判断线程是否已经达到上限
private static final long ADD_WORKER = 0x0001L << (TC_SHIFT + 15); // sign

// runState bits: SHUTDOWN must be negative, others arbitrary powers of two
//用于runState，表示锁定状态
private static final int  RSLOCK     = 1;

//用于runState，表示signal信号，一个线程在阻塞前，必须设置RSIGNAL，告诉其他线程在释放锁时
//你要叫醒我
private static final int  RSIGNAL    = 1 << 1;

//用于runState，表示ForkJoinPool的启动
private static final int  STARTED    = 1 << 2;

//用于runState，表示ForkJoinPool已经停止，此时不接受新的任务，不能创建新的工作线程去处理任务，剩下还未执行的任务将被取消
private static final int  STOP       = 1 << 29;

//用于runState，表示ForkJoinPool彻底停止，没有任务工作线程，所有任务已经被取消
private static final int  TERMINATED = 1 << 30;

//用于runState，表示停止ForkJoinPool，此时不再接收新的任务，已提交的任务不会被取消，依然会执行
//提交的任务执行完成后会恶化成STOP，直到变成TERMINATED，这是一个负数
private static final int  SHUTDOWN   = 1 << 31;

// Instance fields
//高32用于表示线程数，前16表示活跃线程数量，后16表示总线程数
//低32用于表示失活（空闲）线程的WorkQueue.scanState，前16用于表示版本（每次被挂起到唤醒时版本计数加1）,后16位表示队列在队列组中的索引
volatile long ctl;                   // main pool control

//ForJoinPool的运行状态
volatile int runState;               // lockable status

//ForJoinPool的配置，高16位表示队列模式（FIFO，LIFO，共享），低16表示并行线程数，这里与WorkQueue有所不同
//WorkQueue的config的低16表示WorkQueue在队列组的下标
final int config;                    // parallelism, mode

//创建工作线程时，用于计算对应WorkQueue在工作队列组中的下标
//与SEED_INCREMENT配合使用
int indexSeed;                       // to generate worker index

//工作队列组，容量为2的幂
volatile WorkQueue[] workQueues;     // main registry

//创建ForkJoinWorkerThread的工厂类
final ForkJoinWorkerThreadFactory factory;

//用于处理任务执行时发生的错误
final UncaughtExceptionHandler ueh;  // per-worker UEH

//ForkJoinWorkerThread的线程名
final String workerNamePrefix;       // to create worker name string

//统计所有工作线程偷取任务计数
volatile AtomicLong stealCounter;    // also used as sync monitor

//Unsafe可以直接进行内存操作
private static final sun.misc.Unsafe U;

//ForkJoinTask[]数组的基准偏移位置
//通过这个这个值再加上元素直接的间距就可以定位一个位置
private static final int  ABASE;

//ForkJoinTask[]数组两个元素之间的间距的幂--》log(间距) 底数为2
private static final int  ASHIFT;

//ctl字段的偏移地址
private static final long CTL;

//runState的内存偏移地址
private static final long RUNSTATE;

//stealCounter的内存偏移地址
private static final long STEALCOUNTER;

//parkBlocker的内存偏移地址
private static final long PARKBLOCKER;
//WorkQueue#top的内存偏移地址
private static final long QTOP;
//WorkQueue#qlock的内存偏移地址
private static final long QLOCK;
//WorkQueue#scanState的内存偏移地址
private static final long QSCANSTATE;
//WorkQueue#parker的内存偏移地址
private static final long QPARKER;
//WorkQueue#currentSteal的内存偏移地址
private static final long QCURRENTSTEAL;
//WorkQueue#tcurrentJoin的内存偏移地址
private static final long QCURRENTJOIN;

```

### 2.2 ForkJoinPool构造器和一些基础方法说明

#### 2.2.1 构造器

```java
private ForkJoinPool(int parallelism,
                         ForkJoinWorkerThreadFactory factory,
                         UncaughtExceptionHandler handler,
                         int mode,
                         String workerNamePrefix) {
    //工作线程名前缀
    this.workerNamePrefix = workerNamePrefix;
    //ForkJoinWorkerThread线程创建工厂
    this.factory = factory;
    //异常处理器
    this.ueh = handler;
    //队列模式与线程并行数
    this.config = (parallelism & SMASK) | mode;
    //将线程数变成负数
    long np = (long)(-parallelism); // offset ctl counts
    //设置活跃线程数与总线程数，为啥要设置成负数？为了减少通过config判断当前线程数是否已满的操作，作者采用了负数的形式，每次增加一个线程的时候加1，加满之后
    //对应的值等于零。比如并行数为4，负数就是-4，如果我增加了4个线程，那么-4 + 4 = 0，我只要判断它是否等于零就知道线程数是否已满
    this.ctl = ((np << AC_SHIFT) & AC_MASK) | ((np << TC_SHIFT) & TC_MASK);
}
```
#### 2.2.2 基础方法说明

##### 2.2.2.1 makeCommonPool

创建common ForkJoinPool的方法

```java
private static ForkJoinPool makeCommonPool() {
    int parallelism = -1;
    ForkJoinWorkerThreadFactory factory = null;
    UncaughtExceptionHandler handler = null;
    try {  // ignore exceptions in accessing/parsing properties
        //从系统属性中获取
        String pp = System.getProperty
            ("java.util.concurrent.ForkJoinPool.common.parallelism");
        String fp = System.getProperty
            ("java.util.concurrent.ForkJoinPool.common.threadFactory");
        String hp = System.getProperty
            ("java.util.concurrent.ForkJoinPool.common.exceptionHandler");
        if (pp != null)
            parallelism = Integer.parseInt(pp);
        if (fp != null)
            factory = ((ForkJoinWorkerThreadFactory)ClassLoader.
                       getSystemClassLoader().loadClass(fp).newInstance());
        if (hp != null)
            handler = ((UncaughtExceptionHandler)ClassLoader.
                       getSystemClassLoader().loadClass(hp).newInstance());
    } catch (Exception ignore) {
    }
    if (factory == null) {
        if (System.getSecurityManager() == null)
            factory = defaultForkJoinWorkerThreadFactory;
        else // use security-managed default
            factory = new InnocuousForkJoinWorkerThreadFactory();
    }
    if (parallelism < 0 && // default 1 less than #cores
        (parallelism = Runtime.getRuntime().availableProcessors() - 1) <= 0)
        parallelism = 1;
    if (parallelism > MAX_CAP)
        parallelism = MAX_CAP;
    return new ForkJoinPool(parallelism, factory, handler, LIFO_QUEUE,
                            "ForkJoinPool.commonPool-worker-");
}
```
makeCommonPool方法在静态块中被调用，用于创建通用ForkJoinPool，这个通用ForkJoinPool不能被关闭，它主要用在JDK8的并行流中。

##### 2.2.2.2 lockRunState

这个方法用于抢占runState锁

```java
private int lockRunState() {
    int rs;
    //将runState标记上RSLOCK标志即为抢锁成功，抢锁失败的线程将进入awaitRunStateLock方法 --》 自旋 | 阻塞
    return ((((rs = runState) & RSLOCK) != 0 ||
             !U.compareAndSwapInt(this, RUNSTATE, rs, rs |= RSLOCK)) ?
            awaitRunStateLock() : rs);
}
```
通过CAS的方式给runState打上RSLOCK标记，打上的即为抢到了锁，没有打上的会进入awaitRunStateLock方法

##### 2.2.2.2 awaitRunStateLock

在抢占runState锁失败时，进入此方法，自旋 或者 阻塞

```java
private int awaitRunStateLock() {
    Object lock;
    boolean wasInterrupted = false;
    for (int spins = SPINS, r = 0, rs, ns;;) {
        //如果锁被释放
        if (((rs = runState) & RSLOCK) == 0) {
            //快抢啊
            if (U.compareAndSwapInt(this, RUNSTATE, rs, ns = rs | RSLOCK)) {
                //如果线程在wait阶段被中断过，那么重新把中断标记打上，因为wait方法会抛出中断异常，同时会将中断标志清除
                //所以此处需要重新打上标记
                if (wasInterrupted) {
                    try {
                        Thread.currentThread().interrupt();
                    } catch (SecurityException ignore) {
                    }
                }
                return ns;
            }
        }
        else if (r == 0)
            //获取随机数
            r = ThreadLocalRandom.nextSecondarySeed();
            //自旋
        else if (spins > 0) {
            //将随机数异或偏移，为啥这么搞？不知道，首先随机数怎么弄出来的没有研究过
            r ^= r << 6; r ^= r >>> 21; r ^= r << 7; // xorshift
            if (r >= 0)
                --spins;
        }
        //自旋，等待ForkJoinPool初始化完成
        else if ((rs & STARTED) == 0 || (lock = stealCounter) == null)
            Thread.yield();   // initialization race
        //将runStatue打上RSIGNAL标记，表示自己要阻塞了
        else if (U.compareAndSwapInt(this, RUNSTATE, rs, rs | RSIGNAL)) {
            synchronized (lock) {
                if ((runState & RSIGNAL) != 0) {
                    try {
                        lock.wait();
                    } catch (InterruptedException ie) {
                        if (!(Thread.currentThread() instanceof
                              ForkJoinWorkerThread))
                            wasInterrupted = true;
                    }
                }
                else
                    lock.notifyAll();
            }
        }
    }
}
```
如果设置了自旋次数，那么线程在未获取到锁的情况下会进行自旋，自旋时还有一个随机值，用于减少自旋次数，实在获取不到的，最终会被阻塞

##### 2.2.2.3 transferStealCount

```java
final void transferStealCount(ForkJoinPool p) {
    AtomicLong sc;
    if (p != null && (sc = p.stealCounter) != null) {
        int s = nsteals;
        nsteals = 0;            // if negative, correct for overflow
        //如果s小于0，表明溢出，偷的任务太多了
        sc.getAndAdd((long)(s < 0 ? Integer.MAX_VALUE : s));
    }
}
```

上面这个方法是用于将工作线程偷取的任务数加到ForkJoinPool的stealCounter中

##### 2.2.2.4 tryRelease

这个方法用于唤醒应为扫描不到任务而挂起的线程

```java
private boolean tryRelease(long c, WorkQueue v, long inc) {
    //从ctl中获取挂起线程的scanState
    //(sp + SS_SEQ)表示将sp的高16位进行版本计数
    //(sp + SS_SEQ) & ~INACTIVE ==》 工作线程在未失活的情况下，它的scanState就是队列在队列中的奇数下标，当它因为扫描不到任务失活之后
    //会与INACTIVE进行位或，INACTIVE的值为 1 << 31是int的最小值，一个负数
    //所以此处要恢复活跃状态就要将位与 ~INACTIVE
    int sp = (int)c, vs = (sp + SS_SEQ) & ~INACTIVE; Thread p;
    //检查当前要唤醒线程的scanState就是栈顶的scanState，所谓栈顶，就是直接从ctl的低32位获取的scanState
    //在它之前阻塞的线程信息将由这个栈顶工作线程对应的队列的stackPred字段维护，这样就组成了一个栈
    //因为存在多线程的关系，其他线程可能先唤醒了它
    if (v != null && v.scanState == sp) {          // v is at top of stack
        //线程递增，低32位恢复到上一个被挂起的线程scanState
        long nc = (UC_MASK & (c + inc)) | (SP_MASK & v.stackPred);
        //原子更新
        if (U.compareAndSwapLong(this, CTL, c, nc)) {
            //恢复递增了版本的scanState
            v.scanState = vs;
            //唤醒线程，有人可能会问，为啥要判空呢？其实在工作线程的时候，是先更新ctl，后面才去阻塞线程，并且被标记了失活也不一定就被阻塞
            //根据情况它还有1次机会自旋，此时可能变回活跃线程，所以这里要判null
            if ((p = v.parker) != null)
                U.unpark(p);
            return true;
        }
    }
    return false;
}
```

从ctl的低32位获取挂起线程的scanState，scanState标记了队列在队列组的下标，可以找到对应的工作队列
然后把低32位恢复到上一个阻塞线程的scanState，成功之后就会唤醒栈顶挂起的线程

### 2.3 外部线程任务提交

在前面一节中我们分析过，对于外部线程提交的任务，ForkJoinPool是将这个任务分配在了一个偶数位下标的工作队列里面，那么具体是怎么实现的呢？

```java
private void externalSubmit(ForkJoinTask<?> task) {
    int r;                                    // initialize caller's probe
    //初始化当前提交任务的线程的探针值
    if ((r = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();
        //内部使用一个AtomicLong进行自增操作
        r = ThreadLocalRandom.getProbe();
    }
    for (;;) {
        WorkQueue[] ws; WorkQueue q; int rs, m, k;
        boolean move = false;
        //ForkJoinPool被关闭，不再接收新的任务
        if ((rs = runState) < 0) {
            //尝试去关闭线程池
            tryTerminate(false, false);     // help terminate
            throw new RejectedExecutionException();
        }
        //如果还没有初始化，先将线程池初始化
        else if ((rs & STARTED) == 0 ||     // initialize
                 ((ws = workQueues) == null || (m = ws.length - 1) < 0)) {
            int ns = 0;
            //锁定runState，没有获取到锁的线程将在会自旋或者阻塞
            rs = lockRunState();
            try {
                if ((rs & STARTED) == 0) {
                    U.compareAndSwapObject(this, STEALCOUNTER, null,
                                           new AtomicLong());
                    // create workQueues array with size a power of two
                    //获取并行线程数
                    int p = config & SMASK; // ensure at least 2 slots
                    //通过以下操作找到与p最接近的二次幂的值
                    //找到之后将这个值扩大两倍
                    //它的原理就是将p中最高位的那个1以下的bit为都设置为1，最后加1等到最接近的二次幂的值
                    //它的实现原理可以到参看第1节的ConcurrentHashMap分析
                    int n = (p > 1) ? p - 1 : 1;
                    n |= n >>> 1; n |= n >>> 2;  n |= n >>> 4;
                    n |= n >>> 8; n |= n >>> 16; n = (n + 1) << 1;
                    workQueues = new WorkQueue[n];
                    ns = STARTED;
                }
            } finally {
                //锁释放，并打上STARTED标记
                unlockRunState(rs, (rs & ~RSLOCK) | ns);
            }
        }
        // r & m & SQMASK =》r：随机值，m：工作队列组的容量-1，SQMASK：偶数位最大64个的掩码
        // r & m 求余计算出了下标，与上SQMASK之后变成了一个小于或等于126的偶数
        // ForkJoinPool把外部线程提交的任务存放在偶数位上的任务队列中
        else if ((q = ws[k = r & m & SQMASK]) != null) {
            //锁住队列，外部提交的任务的是共享的，所以必须加锁
            if (q.qlock == 0 && U.compareAndSwapInt(q, QLOCK, 0, 1)) {
                ForkJoinTask<?>[] a = q.array;
                int s = q.top;
                boolean submitted = false; // initial submission or resizing
                try {                      // locked version of push
                    //a.length > s + 1 - q.base 如果不成立，那么说明任务队列容量不够了，需要扩容
                    //工作队列的扩容我们在上一节分析WorkQueue时已经分析过了，此处不再赘述
                    if ((a != null && a.length > s + 1 - q.base) ||
                        (a = q.growArray()) != null) {
                        //计算栈顶位置
                        int j = (((a.length - 1) & s) << ASHIFT) + ABASE;
                        //将任务放入栈顶
                        U.putOrderedObject(a, j, task);
                        //更新栈顶位置
                        U.putOrderedInt(q, QTOP, s + 1);
                        submitted = true;
                    }
                } finally {
                    //解锁
                    U.compareAndSwapInt(q, QLOCK, 1, 0);
                }
                if (submitted) {
                    //提交成功，尝试创建新的工作线程，如果线程数已满，那么尝试唤醒被挂起的线程去处理
                    signalWork(ws, q);
                    return;
                }
            }
            move = true;                   // move on failure
        }
        //为提交的任务创建工作队列
        else if (((rs = runState) & RSLOCK) == 0) { // create new queue
            //创建工作队列，第二个参数表示这个工作队列所属线程，对于外部线程提交的任务
            //它所在的队列是共享的
            q = new WorkQueue(this, null);
            //记录这个随机值
            q.hint = r;
            //低16表示工作队列在队列组中的索引，高16表示队列模式，对于外部线程提交的任务所在的队列是共享的
            q.config = k | SHARED_QUEUE;
            //扫描状态为失活状态，这是一个负数，因为共享队列不属于任何一个工作线程，它不需要标记工作线程状态
            q.scanState = INACTIVE;
            //加锁，将新创建的工作队列存入队列组
            rs = lockRunState();           // publish index
            if (rs > 0 &&  (ws = workQueues) != null &&
                k < ws.length && ws[k] == null)
                ws[k] = q;                 // else terminated
            unlockRunState(rs, rs & ~RSLOCK);
        }
        else
            move = true;                   // move if busy
        if (move)
            //如果队列有其他线程在操作，为了减少竞争，获取下一个随机值
            r = ThreadLocalRandom.advanceProbe(r);
    }
}
```
外部线程提交任务主要有以下几个步骤：
- 检查线程池是否被关闭，被关闭后，任务不允许执行
- 检查线程池是否已经启动，如果未启动，需要加锁，初始化线程池
- 计算偶数位，从对应队列组中获取队列，如果还未创建，那么先要创建工作队列，然后将任务push到这个队列中
- 如果其他线程发现队列比较忙，为了减少竞争，重新随机值

externalSubmit还有一个精简版方法externalPush，这个方法与externalSubmit中向已存在的WorkQueue push值的操作差不多，不再展开叙述，
它主要用于线程池已经启动并且一个线程多次提交任务的情况。


### 2.4 工作线程的创建

```java
private void tryAddWorker(long c) {
    boolean add = false;
    do {
        //添加活跃线程数和总线程数
        long nc = ((AC_MASK & (c + AC_UNIT)) |
                   (TC_MASK & (c + TC_UNIT)));
        //ctl没有被其他线程修改
        if (ctl == c) {
            int rs, stop;                 // check if terminating
            //抢锁，并检查线程池是否已经STOP，没有停止才允许创建新的工作线程
            if ((stop = (rs = lockRunState()) & STOP) == 0)
                add = U.compareAndSwapLong(this, CTL, c, nc);
            //解锁
            unlockRunState(rs, rs & ~RSLOCK);
            //线程池已经STOP，退出
            if (stop != 0)
                break;
            //允许创建新的工作线程
            if (add) {
                createWorker();
                break;
            }
        }
        //ADD_WORKER的第48位是1，与ctl位与就是为了检查总线程是否已满
        //前面说过，总线程使用的并行数的负数，如果线程数加到零了，也就是从负数变成了整数
        //总线程数的第48位为0，与ADD_WORKER位与就是零
        //(int)c == 0 表示截取ctl的低32位，低32位表示某个失活线程的scanState，如果不等于零，说明有线程没有扫描到任务而失活
        //此处在创建工作线程就没有必要了，反正也获取不到任务，等下次有任务再创建也不迟
    } while (((c = ctl) & ADD_WORKER) != 0L && (int)c == 0);
}
```
创建线程首先需要将活跃线程数和总线程数加1，然后通过原子操作更新ctl，只要更新成功才有资格去创建工作线程

```java
private boolean createWorker() {
    ForkJoinWorkerThreadFactory fac = factory;
    Throwable ex = null;
    ForkJoinWorkerThread wt = null;
    try {
        //使用线程工厂常见工作线程
        if (fac != null && (wt = fac.newThread(this)) != null) {
            //启动工作线程
            wt.start();
            return true;
        }
    } catch (Throwable rex) {
        ex = rex;
    }
    //如果创建工作线程时发生错误，需要注销创建的工作队列和更新ctl
    deregisterWorker(wt, ex);
    return false;
}
```
通常我们没什么特殊需要，是用的都是默认的工作线程工厂DefaultForkJoinWorkerThreadFactory，内部直接调用了ForkJoinWorkerThread的构造器

```java
protected ForkJoinWorkerThread(ForkJoinPool pool) {
    // Use a placeholder until a useful name can be set in registerWorker
    super("aForkJoinWorkerThread");
    this.pool = pool;
    //给当前创建的线程注册工作队列
    this.workQueue = pool.registerWorker(this);
}
```
下面我们继续看到工作队列的注册

```java
final WorkQueue registerWorker(ForkJoinWorkerThread wt) {
    UncaughtExceptionHandler handler;
    //将线程设置为守护线程，各位同学在调试的时候记得不要让主线程退出
    //调用System.in.read()阻塞中，以便因为主线程的退出导致无法debug
    wt.setDaemon(true);                           // configure thread
    if ((handler = ueh) != null)
        wt.setUncaughtExceptionHandler(handler);
    //创建与工作线程相关的工作队列
    WorkQueue w = new WorkQueue(this, wt);
    int i = 0;                                    // assign a pool index
    //从ForkJoinPool#config中获取队列模式
    int mode = config & MODE_MASK;
    //同步
    int rs = lockRunState();
    try {
        WorkQueue[] ws; int n;                    // skip if no array
        if ((ws = workQueues) != null && (n = ws.length) > 0) {
            //获取用于计算数组下标的索引种子
            int s = indexSeed += SEED_INCREMENT;  // unlikely to collide
            int m = n - 1;
            //与1位或，也就是将第1个bit位设置为1，那么此时这个数一定是奇数
            //与m位与，是为了得到在m以内的值的奇数下标值
            i = ((s << 1) | 1) & m;               // odd-numbered indices
            //如果对应下标已经存在工作队列了，那么说明发生了碰撞，需要换位置
            if (ws[i] != null) {                  // collision
                int probes = 0;                   // step by approx half n
                //计算步长，步长是一个偶数并且除了2之外，其他步长不能是2次幂的值所以额外会加上2（下面会证明），奇数加上偶数依然是奇数
                //为什么步子要根据容量来计算，步子迈这么大这是为了啥？步子迈的大比较容易找到没有被占用的坑位
                //如果扩容后大家都按照2的这个步子走，就需要循环更多次才能找到未被占用的坑位，所以需要按照容量来计算步长
                int step = (n <= 4) ? 2 : ((n >>> 1) & EVENMASK) + 2;
                
                //迈大步子会不会导致漏掉一些奇数的坑位呢？如果是寻找n/2次(不过这里作者却寻找了n次，莫非就是为了检查两遍吗
                //但是正常情况下，已经占坑的工作线程是不会释放坑位的，除非报错)
                //就可以遍历掉当前数组中所有的奇数坑位，why？下面会给出证明
                while (ws[i = (i + step) & m] != null) {
                    //如果找了一圈都没有找到能用的坑位，那么2倍扩容
                    if (++probes >= n) {
                        workQueues = ws = Arrays.copyOf(ws, n <<= 1);
                        m = n - 1;
                        probes = 0;
                    }
                }
            }
            //记录为随机值，用于下次扫描任务时使用
            w.hint = s;                           // use as random seed
            //高16表示队列模式，低16表示工作队列在工作队列组中的下标
            w.config = i | mode;
            //扫描状态记录工作队列在工作队列组中的下标，为正数表示正在扫描任务状态
            w.scanState = i;                      // publication fence
            ws[i] = w;
        }
    } finally {
        //解锁
        unlockRunState(rs, rs & ~RSLOCK);
    }
    //设置线程名
    wt.setName(workerNamePrefix.concat(Integer.toString(i >>> 1)));
    return w;
}
```

为工作线程创建的队列存放在队列组的奇数位置，如果对应奇数位置已经被其他工作线程占用，那么通过容量计算步长寻找不为空的坑位，如果坑位不够了，那么2倍扩容

> Q：为什么除了2之外的步长不能是2次幂的值？为什么要加2？为什么循环了n次后就表示全部的奇数位置都已经遍历过了？

> A：

```java
假设有某数组容量为2次幂的值n，m = n - 1

m的二进制表示为 000..011..11

我们现在从奇数i开始，步子为2次幂的值step(step < m)
step的二进制表示为 000..000100..0 =》 1 << (31 - Integer.numberOfLeadingZeros(step))
任何一个数x与step相加，它也只是将高 Integer.numberOfLeadingZeros(step) + 1进行加1而已，低 32 - Integer.numberOfLeadingZeros(step)
不会发生改变，所以不管循环加多少次2次幂的步长的，(i + step) & m 的值都不能涵盖m以内的所有奇数，并且最多
2 ^ (Integer.numberOfLeadingZeros(step) - Integer.numberOfLeadingZeros(n)) 个奇数值 与正常的 2 ^ (30 - Integer.numberOfLeadingZeros(n)) 相差
2 ^ (30 - Integer.numberOfLeadingZeros(step))个


举个比较直观的例子：

假设奇数7与step为4的值相加
4的二进制为 00..0100
7的二进制为 00..0111
这两个值相加，改变的bit位为高30位，低2位不会发生改变，如果让它与一个m为7（二进制为00..0111）的进行位与，你会发现得到的值只有两种 00..0111和00..0011

所以除了2之外，不能用其他2次幂的值做步长，那么怎么避免出现2次幂的步长呢？将一个大于2的2次幂的值加2，得到的值即保证了是偶数又保证了它不是2次幂的值
那为啥2可以啊？2的二进制为 00..0010，一个奇数除了第一位必须是1之外，高31位都是可以变化的，任何一个奇数与2相加，改变的就是高31位，所以不会导致m以内的奇数值
丢失


一个大于二的二次幂的值加上2，其二进制表现形式为  00..0..1..10，与步长为2的区别就是多了一个1，如果我们的步长是2，将一个奇数1以步长为2加到m需要加多少次？
2 ^ (30 - Integer.numberOfLeadingZeros(n)) - 1 ==> n >>> 1 - 1

那我不从奇数1开始呢？我就要7开始，我要遍历到所有的奇数应该多少次？同样的那么多次，达到m之后，高位溢出，从奇数1开始

假设我们n是16，m就是15，二进制表示为 00..01111，步长就是 n >>> 1 + 2 = 10，二进制为 00..01010，我们从奇数1开始计数，我们将这个10拆开，变成
00..01010 == 00..01000 | 00..00010，由于n >>> 1，我们可以看到00..01000的那个1就是m的最高位1，所以第4位要么是0，要么就是1，低位在没有加到00..00111之前，
有一半的值第4位是1，一半是第四位是0，当低三位达到00..00111值继续往上加时，向前进了一位，低三位又变成了0，我们知道在低三位加到00..00111的值一半的值第4位是1
，一半是第四位是0，现在向前进了一个1，那么也就是说原来那一半为第四位为1变成了第4位为0，原来那一半第四位为零的，现在变成了低四位，这样一看要从一个奇数遍历到
小于等于m的奇数需要累加 n >>> 1 - 1 ，对于本例来说也就是7次即可

```

下面我们继续看到工作队列的注销

```java
final void deregisterWorker(ForkJoinWorkerThread wt, Throwable ex) {
    WorkQueue w = null;
    if (wt != null && (w = wt.workQueue) != null) {
        WorkQueue[] ws;                           // remove index from array
        //获取对应工作队列的下标，WorkQueue.scanState的低16位表示工作队列的下标
        int idx = w.config & SMASK;
        //加锁
        int rs = lockRunState();
        //释放对应工作队列占用的坑位
        if ((ws = workQueues) != null && ws.length > idx && ws[idx] == w)
            ws[idx] = null;
        unlockRunState(rs, rs & ~RSLOCK);
    }
    long c;                                       // decrement counts
    //减去线程数
    do {} while (!U.compareAndSwapLong
                 (this, CTL, c = ctl, ((AC_MASK & (c - AC_UNIT)) |
                                       (TC_MASK & (c - TC_UNIT)) |
                                       //SP_MASK是低32位掩码，保存低32的bit位
                                       (SP_MASK & c))));
    if (w != null) {
        //标识这个队列已经停止
        w.qlock = -1;                             // ensure set
        //将当前工作队列的偷取任务数加到ForkJoinPool#stealCounter中
        w.transferStealCount(this);
        //取消队里剩余的任务
        w.cancelAll();                            // cancel remaining tasks
    }
    for (;;) {                                    // possibly replace
        WorkQueue[] ws; int m, sp;
        //尝试帮助线程池
        if (tryTerminate(false, false) || w == null || w.array == null ||
            (runState & STOP) != 0 || (ws = workQueues) == null ||
            (m = ws.length - 1) < 0)              // already terminating
            break;
        //如果ctl中的低32位（记录某个线程的scanState，因为没有扫描到任务而失活，甚至被阻塞）
        //尝试唤醒这个线程，用来替换自己
        if ((sp = (int)(c = ctl)) != 0) {         // wake up replacement
            if (tryRelease(c, ws[sp & m], AC_UNIT))
                break;
        }
        //如果不存在因扫描不到任务而挂起的线程，那么判断是否已经达到了总线程上限，没达到可以尝试增加工作线程
        else if (ex != null && (c & ADD_WORKER) != 0L) {
            tryAddWorker(c);                      // create replacement
            break;
        }
        else                                      // don't need replacement
            break;
    }
    if (ex == null)                               // help clean on way out
        //清理异常列表中任务被gc回收后的异常信息
        ForkJoinTask.helpExpungeStaleExceptions();
    else                                          // rethrow
        //抛出异常
        ForkJoinTask.rethrow(ex);
}
```

这个方法首先将注册的工作队列注销，然后看情况是否需要唤醒或者添加线程来补偿自己

## 三、总结

第一部分我们主要分析了外部线程提交任务和创建工作线程的实现，下一节我们继续分析工作线程的启动，任务的fork与join