## 一、Phaser

### 1.1 字段说明

```java

//用于记录parties，phase
//高32用于记录当前正在等待释放线程执行的阶段
//低32的高16位表示每一阶段可arrived的parties数量
//低32的低16位表示当前还剩多少parties未arrived
private volatile long state;

//表示最大可arrived的parties数
private static final int  MAX_PARTIES     = 0xffff;

//表示最大可arrived的phase
private static final int  MAX_PHASE       = Integer.MAX_VALUE;

//parties数的偏移bit位
private static final int  PARTIES_SHIFT   = 16;

//获取phase的偏移bit位
private static final int  PHASE_SHIFT     = 32;

//用于获取未arrived parties数的掩码
private static final int  UNARRIVED_MASK  = 0xffff;      // to mask ints

//用于获取每个阶段最多可arrived的parties数
private static final long PARTIES_MASK    = 0xffff0000L; // to mask longs

//获取低32位的掩码
private static final long COUNTS_MASK     = 0xffffffffL;
private static final long TERMINATION_BIT = 1L << 63;

// some special values
//用于减去unarrive parties数的减数
private static final int  ONE_ARRIVAL     = 1;
//用于增加或者减少parties数的加数或者减数
private static final int  ONE_PARTY       = 1 << PARTIES_SHIFT;
//用于同时减去已arrive parties数并减去总parties数的减数
private static final int  ONE_DEREGISTER  = ONE_ARRIVAL|ONE_PARTY;
//在未指定Phaser的parties时使用
private static final int  EMPTY           = 1;
```
### 1.2 构造器


```java
//parent：父Phaser对象，parties：表示每个阶段需要arrive的线程
public Phaser(Phaser parent, int parties) {
    //需要arrive的线程数不能超过（2的16次方-1）个
    if (parties >>> PARTIES_SHIFT != 0)
        throw new IllegalArgumentException("Illegal number of parties");
    int phase = 0;
    this.parent = parent;
    if (parent != null) {
        //将root的值设置为父Phaser
        final Phaser root = parent.root;
        this.root = root;
        this.evenQ = root.evenQ;
        this.oddQ = root.oddQ;
        //如果注册的arrive子Phaser parties数有效，那么需要给父Phaser注册一个parties，但子Phaser arrive的parties数齐了后，也就是表示父Phaser arrive了一个parties
        if (parties != 0)
            //阶段跟随父阶段
            phase = parent.doRegister(1);
    }
    else {
        this.root = this;
        this.evenQ = new AtomicReference<QNode>();
        this.oddQ = new AtomicReference<QNode>();
    }
    this.state = (parties == 0) ? (long)EMPTY :
        ((long)phase << PHASE_SHIFT) |
        ((long)parties << PARTIES_SHIFT) |
        ((long)parties);
}
```
evenQ和oddQ两个变量分别用于记录阶段为偶数和奇数时被挂起的线程。为什么要分开成两个队列呢？在唤醒一个阶段被挂起的线程时，这个阶段要么是偶数，要么是奇数，假设当前阶
段为3，刚好为3这个阶段的parties已经满了，进入了下一个阶段4，如果此时不区分奇数队列和偶数队列，用一个公用队列的话，那么阶段4挂起的线程将添加到同一个队列中，这样
就增大了3阶段唤醒线程的开销。

### 1.3 注册

```java
private int doRegister(int registrations) {
    // adjustment to state
    //用于构建arrive parties数的加数
    long adjust = ((long)registrations << PARTIES_SHIFT) | registrations;
    final Phaser parent = this.parent;
    int phase;
    for (;;) {
        //如果存在父Phaser并且他们的phase不一样的话，会将父Phaser的phase与子Phase的parties进行合并
        //phase跟随父Phaser
        long s = (parent == null) ? state : reconcileState();
        int counts = (int)s;
        //获取总共可以arrive的parties数
        int parties = counts >>> PARTIES_SHIFT;
        //获取还未arrive的parties数
        int unarrived = counts & UNARRIVED_MASK;
        //如果要注册的parties数量超过了最大限制，抛出错误
        if (registrations > MAX_PARTIES - parties)
            throw new IllegalStateException(badRegister(s));
        //获取当前注册所处的阶段
        phase = (int)(s >>> PHASE_SHIFT);
        if (phase < 0)
            break;
        if (counts != EMPTY) {                  // not 1st registration
            if (parent == null || reconcileState() == s) {
                //如果当前还需arrive的parties数量等于零，说明目前正在进入下一个阶段，需求等待最后一个arrive parties的线程更新完阶段和unarrive parties数量
                //然后再注册parties数
                if (unarrived == 0)             // wait out advance
                    root.internalAwaitAdvance(phase, null);
                    //更新parties数量
                else if (UNSAFE.compareAndSwapLong(this, stateOffset,
                                                   s, s + adjust))
                    break;
            }
        }
        else if (parent == null) {              // 1st root registration
            long next = ((long)phase << PHASE_SHIFT) | adjust;
            if (UNSAFE.compareAndSwapLong(this, stateOffset, s, next))
                break;
        }
        else {
            synchronized (this) {               // 1st sub registration
                if (state == s) {               // recheck under lock
                    //如果子Phaser注册了向父Phaser注册parties
                    phase = parent.doRegister(1);
                    if (phase < 0)
                        break;
                    // finish registration whenever parent registration
                    // succeeded, even when racing with termination,
                    // since these are part of the same "transaction".
                    //一直注册子Phaser的state知道成功
                    while (!UNSAFE.compareAndSwapLong
                           (this, stateOffset, s,
                            ((long)phase << PHASE_SHIFT) | adjust)) {
                        s = state;
                        phase = (int)(root.state >>> PHASE_SHIFT);
                        // assert (int)s == EMPTY;
                    }
                    break;
                }
            }
        }
    }
    return phase;
}
```

注册时判断是否存在父Phaser，如果存在父Phaser，那么需要整合父Phaser的phase与当前子Phaser的parties，然后增加需要arrive的parties数和unarrive的parties，如果刚好要
注册的phase的unarrive为零，那么说明当前有一个线程正在修改state（准备进入下一个阶段），那么当前注册操作需要等待其完成后方能继续。


### 1.4 arrive

```java
private int doArrive(int adjust) {
    final Phaser root = this.root;
    for (;;) {
        // reconcileState 整合父Phaser的phase，子Phaser的parties
        long s = (root == this) ? state : reconcileState();
        int phase = (int)(s >>> PHASE_SHIFT);
        if (phase < 0)
            return phase;
        int counts = (int)s;
        //获取还未arrive的parties数
        int unarrived = (counts == EMPTY) ? 0 : (counts & UNARRIVED_MASK);
        if (unarrived <= 0)
            throw new IllegalStateException(badArrive(s));
        //递减unarrive的parties数
        if (UNSAFE.compareAndSwapLong(this, stateOffset, s, s-=adjust)) {
            //当unarrived的parties数等于1
            if (unarrived == 1) {
                long n = s & PARTIES_MASK;  // base of next state
                //用于构建下一个需要arrive的parties数
                int nextUnarrived = (int)n >>> PARTIES_SHIFT;
                if (root == this) {
                    //如果parties数已经全部注销，那么state将变成一个负数
                    if (onAdvance(phase, nextUnarrived))
                        n |= TERMINATION_BIT;
                    else if (nextUnarrived == 0)
                        n |= EMPTY;
                    else
                        //准备下一阶段需要arrive的parties数
                        n |= nextUnarrived;
                    //阶段递增
                    int nextPhase = (phase + 1) & MAX_PHASE;
                    n |= (long)nextPhase << PHASE_SHIFT;
                    //修改成下一阶段state
                    UNSAFE.compareAndSwapLong(this, stateOffset, s, n);
                    //释放当前阶段被阻塞的线程
                    releaseWaiters(phase);
                }
                //如果子Phaser被注销，那么注册在父Phaser上的parties也要去掉一个
                else if (nextUnarrived == 0) { // propagate deregistration
                    phase = parent.doArrive(ONE_DEREGISTER);
                    UNSAFE.compareAndSwapLong(this, stateOffset,
                                              s, s | EMPTY);
                }
                else
                    //子Phaser完成一个阶段后，父Phaser完成一个parties
                    phase = parent.doArrive(ONE_ARRIVAL);
            }
            return phase;
            }
        }
    }
```

arrive一个party，就是将unarrive的parties数递减，如果不存在unarrive的party后，需要更新state，进入下一个阶段，并唤醒正在队列中挂起的线程

### 1.5 wait

arriveAndAwaitAdvance 方法

```java
public int arriveAndAwaitAdvance() {
    // Specialization of doArrive+awaitAdvance eliminating some reads/paths
    final Phaser root = this.root;
    for (;;) {
        long s = (root == this) ? state : reconcileState();
        int phase = (int)(s >>> PHASE_SHIFT);
        if (phase < 0)
            return phase;
        int counts = (int)s;
        int unarrived = (counts == EMPTY) ? 0 : (counts & UNARRIVED_MASK);
        if (unarrived <= 0)
            throw new IllegalStateException(badArrive(s));
        //arrive 一个party，减去unarrive parties数量
        if (UNSAFE.compareAndSwapLong(this, stateOffset, s,
                                      s -= ONE_ARRIVAL)) {
            if (unarrived > 1)
                //阻塞
                return root.internalAwaitAdvance(phase, null);
            if (root != this)
                //能走到这表明unarrived已经小于等于1，子Phaser完成一个phase，那么就是完成了父Phaser一个party
                return parent.arriveAndAwaitAdvance();
            
        
            long n = s & PARTIES_MASK;  // base of next state
            int nextUnarrived = (int)n >>> PARTIES_SHIFT;
            //如果parties数已经全部注销，那么state将变成一个负数
            if (onAdvance(phase, nextUnarrived))
                n |= TERMINATION_BIT;
            else if (nextUnarrived == 0)
                n |= EMPTY;
            else
                //准备下一阶段需要arrive的parties数
                n |= nextUnarrived;
            //阶段递增
            int nextPhase = (phase + 1) & MAX_PHASE;
            n |= (long)nextPhase << PHASE_SHIFT;
            //修改成下一阶段state
            if (!UNSAFE.compareAndSwapLong(this, stateOffset, s, n))
                return (int)(state >>> PHASE_SHIFT); // terminated
            //释放当前阶段被阻塞的线程
            releaseWaiters(phase);
            return nextPhase;
        }
    }
}
```

internalAwaitAdvance方法

```java
private int internalAwaitAdvance(int phase, QNode node) {
        // assert root == this;
        //唤醒旧phase的线程，确保旧链表中的线程都已经释放
        releaseWaiters(phase-1);          // ensure old queue clean
        boolean queued = false;           // true when node is enqueued
        int lastUnarrived = 0;            // to increase spins upon change
        int spins = SPINS_PER_ARRIVAL;
        long s;
        int p;
        //自旋
        while ((p = (int)((s = state) >>> PHASE_SHIFT)) == phase) {
            if (node == null) {           // spinning in noninterruptible mode
                int unarrived = (int)s & UNARRIVED_MASK;
                if (unarrived != lastUnarrived &&
                    (lastUnarrived = unarrived) < NCPU)
                    spins += SPINS_PER_ARRIVAL;
                boolean interrupted = Thread.interrupted();
                if (interrupted || --spins < 0) { // need node to record intr
                    node = new QNode(this, phase, false, false, 0L);
                    node.wasInterrupted = interrupted;
                }
            }
            //阶段被其他唤醒或者可以被中断或者超时阻塞或者阶段发生改变
            else if (node.isReleasable()) // done or aborted
                break;
            else if (!queued) {           // push onto queue
                //入队列
                AtomicReference<QNode> head = (phase & 1) == 0 ? evenQ : oddQ;
                QNode q = node.next = head.get();
                if ((q == null || q.phase == phase) &&
                    (int)(state >>> PHASE_SHIFT) == phase) // avoid stale enq
                    queued = head.compareAndSet(q, node);
            }
            else {
                try {
                    //使用阻塞管理器，阻塞管理器可以定义当前线程在什么情况下可以释放，什么情况下阻塞
                    ForkJoinPool.managedBlock(node);
                } catch (InterruptedException ie) {
                    node.wasInterrupted = true;
                }
            }
        }

        if (node != null) {
            if (node.thread != null)
                node.thread = null;       // avoid need for unpark()
            if (node.wasInterrupted && !node.interruptible)
                Thread.currentThread().interrupt();
            //如果不是因为阶段发生变化导致线程的释放，那么可能就是超时或者中断
            if (p == phase && (p = (int)(state >>> PHASE_SHIFT)) == phase)
                //在当前线程释放之前，依然会检查阶段是否发生改变，如果改变会去唤醒阻塞在链表中的线程
                return abortWait(phase); // possibly clean up on abort
        }
        //释放phase阶段阻塞的线程
        releaseWaiters(phase);
        return p;
    }
```
## 二、总结

Phaser通过一个long类型的state值用于记录所处的阶段以及总共需要arrive的parties数和unarrive的parties数，其中高32位用于记录phase，低32的高16位用于记录总parties数，低
16位用于记录unarrive的parties数。每叠满一个phase的parties数将重置unarrive的parties数，并唤醒阻塞在此阶段的线程，然后进入下一阶段。