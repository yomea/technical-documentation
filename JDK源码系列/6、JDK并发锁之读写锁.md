## 一、前言
&nbsp;&nbsp;&nbsp;&nbsp;前一节我们分析了AQS并发框架和独占锁以及condition的实现，为我们继续往下分析并发包中其他的类打下了基础，现在我们趁热打铁，分析一下读写锁。

## 二、基础

### 2.1 类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/85f1a631f5b34be4bd72b348fab94e54.png)

- ReadWriteLock：定义了两个接口方法，获取读锁和写锁
- Sync：ReentrantReadWriteLock的内部类，实现了AQS框架的基本方法，定义了一些读写锁所需的成员变量，后面会具体分析
- ReadLock：读锁，实现Lock接口，内部持有Sync，使用于共享锁场景
- WriteLock：写锁，实现Lock接口，内部持有Sync，使用于独占锁场景
- HoldCounter：用于记录某线程重入读锁的次数
- ThreadLocalHoldCounter：继承ThreadLocal，用于记录HoldCounter

### 2.2 基本类分析

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
        
    //读写锁位偏移量
    static final int SHARED_SHIFT   = 16;
    
    //共享锁将一个int的值分为高16位和低16位，高16位表示读锁被获取的次数，低16位表示写锁被重入的次数
    //当一个线程获取到读锁时，需要将高位的数值加1，那么只需将state加上以下这个SHARED_UNIT值即可
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    
    //读锁或者写锁的最大获取次数
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    
    //独占锁掩码，共享锁将一个int的值分为高16位和低16位，高16位表示读锁被获取的次数，低16位表示写锁被重入的次数
    //将state值与这个独占锁掩码进行位与，可以得到低16的值，即独占锁重入次数
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

    //将state无符号右移，获取到高16的值，即读锁的获取次数
    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
    
    //将state与独占锁掩码进行位与获取低16位，即独占锁重入次数
    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
    
    //用于记录和读取线程锁持有的读锁次数
    private transient ThreadLocalHoldCounter readHolds;
    
    //用于缓存某线程的获取读锁记录，避免每次都从ThreadLocalHoldCounter获取对应线程的读锁记录，当然从某种程度上讲，那么一瞬间可以肯定对象是处于共享锁模式的
    private transient HoldCounter cachedHoldCounter;
    
    //记录第一个获取到读锁的线程
    //这个属性有值，那么那一瞬间一定是处于读锁状态，由于并发的缘故，有可能下一瞬间就变成独占锁模式
    //所以实际上他主要是用于检测当前线程是否在在重入，另一方面也可减少从ThreadLocal中获取记录的开销
    private transient Thread firstReader = null
    
    //记录第一个获取读锁线程的重入次数
    private transient int firstReaderHoldCount;

```

从上面的Sync可知，读写锁将一个int值划分为高16位和低16位，高16位表示读锁获取次数（含线程的重入获取锁），低16表示写锁的重入次数

## 三、读锁

### 3.1 读锁的获取

```java
//这个方法定义在父类AbstractQueuedSynchronizer中
public final void java.util.concurrent.locks.AbstractQueuedSynchronizer#acquireShared(int arg) {
    //首先尝试获取读锁，如果失败，进入无线循环竞争锁逻辑，在符合前节点为SIGNAL状态下将被挂起
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```
下面是尝试获取读锁的具体逻辑

```java
protected final int java.util.concurrent.locks.ReentrantReadWriteLock.Sync#tryAcquireShared(int unused) {
  
    Thread current = Thread.currentThread();
    int c = getState();
    //获取独占锁的个数，这个逻辑在分析Sync时讲过，就是将低16提取出来
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        //如果存在独占锁，并且当前线程不是独占锁的持有者，那么对不起，获取失败
        return -1;
    //获取读锁的个数，右移16位，获取高16位
    int r = sharedCount(c);
    //检查同步队列中是否存在等待唤醒的节点，如果是公平锁，那么和第5节我们分析独占锁时一样，调用hasQueuedPredecessors方法，队列有值返回true
    //如果不是公平锁，会检查第一个等待的节点（头节点的后继节点）是否为写锁节点，为写锁节点返回true，为什么第一个等待节点为写锁节点就返回true？
    //避免写线程饥饿
    if (!readerShouldBlock() &&
        //检查是否超过最大值
        r < MAX_COUNT &&
        //竞争锁，成功读锁加1
        compareAndSetState(c, c + SHARED_UNIT)) {
        //如果当前线程是第一个获取到读锁的线程，那么将直接由成员变量firstReader和firstReaderHoldCount来记录
        //可以减少从ThreadLocal中获取数据的次数
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            //是第一个获取到读锁的线程，直接加1
            firstReaderHoldCount++;
        } else {
            //从缓存中获取读锁计数记录，可用于表示当前读写锁瞬时处于什么类型锁状态，另外一方面可减少从本地线程中获取的次数，最主要用于判断是否重入
            HoldCounter rh = cachedHoldCounter;
            //如果当前没有缓存计数记录或者有缓存计数记录，但是不是当前线程的，那么需要重新构建一个HoldCounter
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            //读锁计数加一
            rh.count++;
        }
        return 1;
    }
    //进一步判断是否可以获取到锁，与上面这段代码不同之处在于readerShouldBlock()方法返回true之后，需要进行锁模式的进一步确认，另外会进行无线循环
    //要么获取到锁，要么没获取到锁
    return fullTryAcquireShared(current);
}
```
想要获取一个读锁的步骤：
- 首先判断当前所处锁模式，如果当前被独占模式持有锁，那么只需判断当前持锁人是否为当前线程，如果不是，直接返回-1
- 如果是公平锁，如果同步队列中有其他的等待者，那么先放弃，如果是非公平锁，那么判断当前同步队列中的第一个等待节点是否为写锁节点，如果是写锁节点，那么很大
  概率当前处于写锁模式，暂时放弃，需要到fullTryAcquireShared方法中进一步确认
- 如果获取到了锁，那么需要判断当前锁是否是第一个获取到读锁的线程，如果是，缓存到成员变量中，如果不是第一个获取到线程锁的线程，
  需要线程单独维护锁记录。

> Q：当执行了上面的exclusiveCount(c)判断之后会不会因为并发导致当前被写锁线程持有锁？
> A：会，因为上面的判断并不是原子的，很有可能在判断当前不处于独占锁模式后，被写锁线程持有锁，但是此时c肯定是被改动过的，后面的CAS操作不会成功。

> Q：非公平锁判断同步队列中第一个等待的节点为独占锁模式时就返回true，为什么？
> A：对于读写锁使用的场景大概率适用于读的，而写比较少，写线程需要和这么多读线程进行PK，很容易发生饥饿，导致一直被阻塞

下面是获取读锁的完整版：

```java
final int fullTryAcquireShared(Thread current) {
    
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        //和tryAcquireShared方法一样，判断对象是否处于独占锁模式
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
            //公平锁：检查同步队列是否存在其他正在等待的线程，如果存在，那么需要判断是否是重入，如果是重入，那么可以继续获取锁，不影响公平
            //非公平锁：检查同步队列中的第一个节点是否为写线程，如果是写线程，跟公平锁一样，允许线程重入，不是重入的直接拒绝获取读锁，只是目的不一样
            //非公平锁是为了避免写线程发生饥饿
        } else if (readerShouldBlock()) {
            // Make sure we're not acquiring read lock reentrantly
            //检查当前线程是否第一次获取到读锁的线程
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } else {
                //判断当前线程是否是重入（曾经获取了锁），如果不是重入，那么获取读锁失败
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        //
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    return -1;
            }
        }
        //读锁太多了，抛出错误
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            。。。。。。省略获取到读锁之后的一些计数操作，这部分代码在tryAcquireShared方法讲过，不再重复赘述
            return 1;
        }
    }
}
```
readerShouldBlock()方法在公平锁与非公平锁的实现不同
- 公平锁：当同步队列存在其他的等待节点时，返回true，需要判断线程是否重入，如果是重入可以继续去获取读锁，不影响公平性
- 非公平锁：当同步队列中第一个等待节点为写线程时，返回true。这是为了避免写线程发生饥饿，除非当前线程是重入，否则都将直接认为获取读锁失败

当确实无法获取到读锁的时候，需要将线程挂起，将线程包装成节点加入到同步队列中

```java
 private void java.util.concurrent.locks.AbstractQueuedSynchronizer#doAcquireShared(int arg) {
    //将当前线程包装成Node对象加入到同步队列中，具体的加入过程在第5小节已经分析过，此处不再赘述
    //不同的是，它的锁模式为共享模式，这是有什么作用呢？
    //读写锁可以让读锁与写锁共同存一个同步队列中，所以需要有字段区分它们是什么锁节点，以便于共享锁的唤醒传播
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //前驱节点
            final Node p = node.predecessor();
            if (p == head) {
                //由于同步队列是FIFO的，所以如果当前线程所处的前驱节点是头节点，那么会再次尝试获取锁
                //另一方面还可以避免singal信号丢失信号的情况（还没来得及将前驱节点设置为SIGNAL，就已经发生了锁的释放）
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    //如果获取到了锁，那么更新头节点，然后再判断是否需要唤醒下一个读线程
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            //是否需要挂起，判断方式和独占锁一样，都是前驱节点必须为SIGNAL，才能被挂起
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            //取消锁资格
            cancelAcquire(node);
    }
}
```
上面方法的逻辑和前一节分析的独占锁的逻辑没有很大的区别，唯一的区别就是多了一个setHeadAndPropagate方法，这个方法是做什么的呢？

```java
//node：刚刚获取到了读锁的节点，propagate：调用tryAcquireShared的返回值，这个值需要根据不同的锁实现决定它的意义，我们现在研究是读写锁，那么获取到锁，那就是1
private void java.util.concurrent.locks.AbstractQueuedSynchronizer#setHeadAndPropagate(Node node, int propagate) {
    //记录老的头节点
    Node h = head; // Record old head for check below
    //替换头节点
    setHead(node);
    //对于我们现在分析的读写锁，调用到这里这个propagate肯定是1，大于零
    //但是tryAcquireShared方法有不同的实现，除了读写锁，还有Semaphore，CountDownLatch
    //CountDownLatch跟读写锁一样只会返回1或者-1，但是Semaphore不同，它可能返回任何值，-1，-2,0,1,2...，返回什么值通常是由用户设置的许可数量
    //如果setHeadAndPropagate方法放在一个无线循环竞争锁阻塞方法里，那么这个head节点在目前的实现来看应该不会为空。但是AQS谁都可以去实现，就有出现为null的
    //情况
    //旧头节点状态小于零，通常是PROPAGATE状态(这个状态更详细的解读将在后面分析)，主要出现在propagate为零时，什么时候会是零，使用Semaphore时可能会返回零，此//时这个PROPAGATE将派上用场
    //新的头节点（就是当前节点）状态小于零，通常是SIGNAL状态，它的唤醒必然需要唤醒下一个节点
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        //如果下个节点为空，依然会去尝试唤醒下一个节点，因为存在并发，此刻没有，不代表下一刻会没有
        //如果下一个节点是共享模式节点，尝试唤醒，如果是写锁就不唤醒了吗？是的，暂时不唤醒，写锁将通过其他线程的读锁释放唤醒
        //why？如果唤醒它，很大概率会被重新阻塞住，因为前面的线程都是获取读锁的，让读锁释放的时候唤醒更有可能获取到锁
        if (s == null || s.isShared())
            //尝试唤醒下一个节点
            doReleaseShared();
    }
}
```
setHeadAndPropagate方法的逻辑定义在AQS上，它是一个段通用方法，用于共享锁，对于上面的判断逻辑我们拆开来分析下：
- propagate > 0：这个propagate值由tryAcquireShared方法返回，不同的实现可能返回意义不同，但如果大于零进入分支，其他值不确定，不过满足以下任意条件依然认为唤
  醒行为需要继续传播
- 头节点 == null：从目前的源代码中，除了刚初始化头节点会是空的之外，其他情况，即使队列已经没有等待节点了，这个头节点都不会为null，所以我个人认为这是一种
  防御性编程，谁都可以基于AQS框架进行实现，你无法保证别人会认为同步队列已经没有等待的节点了，为什么不能释放掉头节点（Doug Lea的实现是不会释放的，避免下次
  又要创建头节点的开销）。从另一方面来说，头节点为null就表明这一瞬间同步队列中没有等待的节点了，但很有可能在下一个不确定的时刻出现了等待的节点，所以还是可以
  尝试去传播唤醒行为的
- 旧头节点.waitStatus < 0：旧头节点状态小于零，通常是PROPAGATE状态(这个状态更详细的解读将在后面分析)，什么时候会使用到这个条件判断？当出现propagate为零时，
  什么时候会是零？使用Semaphore时可能会返回零，此时这个PROPAGATE将派上用场
- 新头节点.waitStatus < 0：新的头节点（就是当前节点）状态小于零，通常是SIGNAL状态，它的唤醒必然需要唤醒下一个节点，也有可能是PROPAGATE状态，因为它刚刚将
  自己晋升为头节点，对于不公平锁，其他的读线程的释放将会唤醒当前节点的后继节点。
- s == null：如果下个节点为空，依然会去尝试唤醒下一个节点，因为存在并发，此刻没有，不代表下一刻会没有
- s.isShared()：如果下一个节点是共享模式节点，尝试唤醒，如果是写锁就不唤醒了吗？是的，暂时不唤醒，写锁将通过其他线程的读锁释放唤醒，why？如果唤醒它，很大概
  率会被重新阻塞住，因为前面的线程都是获取读锁的，让读锁释放的时候唤醒更有可能获取到锁

> Q：为什么要判断旧的头节点的waitStatus，又为什么要或上当前节点的waitStatus？
> A：这个问题将留到后面的读锁的释放里面与PROPAGATE状态一起分析。


我们学习独占锁的时候，我们可以看到，在线程被唤醒并获取到锁时并不会立即去唤醒下一个线程，因为即使唤醒了线程很大概率是获取不到锁的，所以没有必要。如果是读锁
那么可以在每个线程唤醒之后，如果获取到了锁，就像传染了瘟疫一样，一个一个往下传播，而不是在执行完用户逻辑之后，调用unlock时进行通知，这样反到影响了读锁的吞吐量，特别是用户逻辑比较费时的情况，将严重影响性能。

> Q：那为啥我获取到读锁之后不能直接循环，把同步队列里面是获取共享锁的节点统统唤醒嘞？
> A：首先同步队列被设计为一个FIFO的队列，你不能一下子全部唤醒，如果中间多个节点获取到锁，那么将导致节点需要从中间断裂，这需要进行加锁同步，反到影响性能，
Doug Lea有更好的办法，那就是每节点被唤醒之后，如果获取到锁，那么就向后传播，像接力一样，如果顺利都能获取到锁的话，那么看起来就和循环没有多大差别了。

doReleaseShared方法释放逻辑将与读锁的释放一起分析

### 3.2 读锁的释放

```java
public final boolean java.util.concurrent.locks.AbstractQueuedSynchronizer#releaseShared(int arg) {
    //释放锁
    if (tryReleaseShared(arg)) {
        //唤醒下一个节点
        doReleaseShared();
        return true;
    }
    return false;
}
```
释放锁

```java
protected final boolean java.util.concurrent.locks.ReentrantReadWriteLock.Sync#tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    //第一个获取读锁的线程
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        //如果是等于1，直接将firstReader置空，为什么不减到0呢？
        //其实大部分情况，读线程只会获取读锁一次，这样就没有必要进行firstReaderHoldCount--计算操作，直接将firstReader赋null即可，可以节省性能
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            //递减
            firstReaderHoldCount--;
    } else {
        //缓存计数记录
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            //读锁全被释放，移除
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count;
    }
    for (;;) {
        int c = getState();
        //高16位减去1
        int nextc = c - SHARED_UNIT;
        //CAS修改state
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
```
读锁的释放，首先会判断当前线程是否是第一个获取读锁的线程（准确的来说，firstReader就一坑位，谁先抢到就是谁的，等这个人走了，下个人可以继续抢），如果是，那么
只需将firstReaderHoldCount递减即可，如果firstReaderHoldCount本来就是1了，那么无需进行递减计算，直接将firstReader置空即可，为什么呢？大部分场景下，在某次操
作下只会获取一次读锁，重入的现象比较少，所以为了避免没有必要的递减，直接就将firstReader置空。

唤醒节点逻辑：

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                //如果节点由Node.SIGNAL修改成0成功，那么唤醒下一个节点
                unparkSuccessor(h);
            }
            //如果下一个节点已经被唤醒，那么将当前头节点设置为PROPAGATE，表示唤醒行为需要被传播
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        //在此过程，头节点可能被其他唤醒并获取到读锁的线程给替换，那么继续循环，帮助唤醒下一个节点
        if (h == head)                   // loop if head changed
            break;
    }
}
```
> Q：这里的逻辑除了Node.PROPAGATE，其他的逻辑都很好理解，那么为啥这里要将一个状态为0的头节点设置为PROPAGATE状态呢？

> A：从线程的挂起条件来看，0和PROPAGATE最终都可以转换成SIGNAL，看起似乎没有什么区别。唯一比较可能有区别的地方就是使用waitStatus和0进行比较运算的地方。

有两个场景可能使用到这个PROPAGATE状态：

- 写锁的释放：

```java
public final boolean java.util.concurrent.locks.AbstractQueuedSynchronizer#release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        //h.waitStatus < 0的话将会唤醒下一个节点
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
可能使用到的场景：<br />
如果有2个以上的读线程释放了锁，那么头节点被设置为了PROPAGATE，等待着读锁的节点被唤醒，此时有一个写线程瞄准时机，立马抢了锁，变成了写锁，那么被唤醒的这个
线程竞争锁失败，准备将头节点设置为SIGNAL，但还没有设置（即使设置了，它依然是小于零的），写锁被释放，又重新unpark了一次当前唤醒的读线程，读线程照常将头节点
设置为SIGNAL，然后继续竞争锁，如果依然失败，那么下次在调用park时将不会被阻塞，直接继续去获取锁，这是因为前面有线程调用了unpark方法。先unpark一个线程，再
park相同的线程，将不会被阻塞。

- 还有就是在setHeadAndPropagate方法被使用：

```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```
要想使用到h.waitStatus这个判断条件，那么需要propagate不大于0，并且头节点不为空，那什么时候propagate不大于0呢？使用信号量Semaphore，如有以下程序，这个程序
来自提交到JDK的bug：https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6801020 <br />

> Q：我是怎么找到这段程序的？
> A：之前本人为了搞清楚这个Node.PROPAGATE的含义，本人想破脑袋也只想到了写锁释放这种场景，但是我个人认为这不是引入Node.PROPAGATE的目的，所以本人选择在互联网
上搜索，我搜索了很久，包括StackOverflow，CSDN等等网站，都没有找到我想要的答案，基本上都是直接略过，有个别人发表了自己的看法，说只是做指示所用，但我对这种看
法持有怀疑态度。无奈继续找，最后我无意中在博客园的一篇博客中找到了线索，这篇博客的地址为：https://www.cnblogs.com/micrari/p/6937995.html，这位博主提到了自
己查找问题的思路，他曾也在网上搜索答案，但是跟本人（我相信不止是我，想要明白这个状态意义的人都一样）一样找不到合理的答案，于是他通过浏览Doug Lea的个人网站
发现了以前是没有Node.PROPAGATE这个状态的，引入这个Node.PROPAGATE状态是为了解决程序可能出现的hang住的问题，他给了我极大的启发，我突然明白了一个道理，有的时
候仅仅通过源码去倒退实现的意图可能不够，如果能在分析源码之前浏览一下相关的文档，论文可能会使自己在分析源码的时候更加的顺利。在此再次对这位博主发出由衷的感谢。

```java
import java.util.concurrent.Semaphore;

public class TestSemaphore {

    private static Semaphore sem = new Semaphore(0);

    private static class Thread1 extends Thread {
        @Override
        public void run() {
            sem.acquireUninterruptibly();
        }
    }

    private static class Thread2 extends Thread {
        @Override
        public void run() {
            sem.release();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 10000000; i++) {
            Thread t1 = new Thread1();
            Thread t2 = new Thread1();
            Thread t3 = new Thread2();
            Thread t4 = new Thread2();
            t1.start();
            t2.start();
            t3.start();
            t4.start();
            t1.join();
            t2.join();
            t3.join();
            t4.join();
            System.out.println(i);
        }
    }
}
```
上面这段程序起了四个线程，两个线程去获取许可，另外两个线程释放许可，信号量的构造函数传入的许可数为0，也就是说在t3，t4进行释放之前t1，t2是不能获取到许可的。

bug提交者提交的问题是：这段程序进行多次执行时，偶尔会出现一个问题，那就是程序会被阻塞住。<br />

> Q：本人一开始看到的这个程序的时候我就觉得有点刁钻，为什么呢？

> A：程序的本意是通过t3，t4两个线程释放许可，然后t1，t2获取许可从而使得许可最终和我们在构造器传入的值一样是0，所以这段程序的t1，t2没有调用release方法

> Q：那么，上面这段程序为什么会发生阻塞呢？

以前释放共享锁的代码与现在的对比如下：

```java
-------------->以前的共享锁释放代码
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

-------------->JDK7与JDK8的共享锁释放代码
private void doReleaseShared() {
        
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}

```
可以看到这段代码跟释放独占锁的代码是差不多的，判断waitStatus不为零才会唤醒下一个节点

下面这一段是以前的setHeadAndPropagate方法和现在的对比

```java
-------------->以前的setHeadAndPropagate代码
private void setHeadAndPropagate(Node node, int propagate) {
    setHead(node);

    if (propagate > 0 && node.waitStatus != 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            unparkSuccessor(node);
    }
}

-------------->JDK7的setHeadAndPropagate代码
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    
    if (propagate > 0 || h == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}

-------------->JDK8的setHeadAndPropagate代码
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    setHead(node);
   
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

加入Node.PROPAGATE前的代码地址：http://gee.cs.oswego.edu/cgi-bin/viewcvs.cgi/jsr166/src/main/java/util/concurrent/locks/AbstractQueuedSynchronizer.java?revision=1.73&view=markup

（1）假设t1，t2在获取许可失败后，同步队列的节点关系如下图：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/dde1c0e3e3c6eb2db5e4dc885f544e98.png)

（2）t3释放许可，调用了上面的releaseShared方法，许可值由零变成了1，然后唤醒了t1，t1获取到了许可，许可返回值为0，此时还没有调用setHead方法，其节点状态如下：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/e51c34382584f4a96010e3a4bb2396f4.png)

（3）t4释放许可，t4调用上面的releaseShared方法，许可值又由0变成了1，然后准备去唤醒同步队列中的节点，但是此时的头节点的waitStatus是0，不会去唤醒下一个节点

（4）t1调用setHead方法将自己的节点顶替之前的head节点，然后判断propagate > 0是否大于零，很明显propagate是等于零的，所以他们么有唤醒它的后继节点，由于t1节点
不会主动释放许可，这就导致了阻塞在同步队列中的t2将永远的沉睡下去。

如果引入Node.PROPAGATE，重走上面的步骤

（1）假设t1，t2在获取许可失败后，同步队列的节点关系如下图：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d47e0ed9fb45d2d1c006821361bccd4d.png)

（2）t3释放许可，调用了上面的releaseShared方法，许可值由零变成了1，然后唤醒了t1，t1获取到了许可，许可返回值为0，此时还没有调用setHead方法，其节点状态如下：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/a723423a9d6b75d86cfa9e71839f35b3.png)

（3）t4释放许可，t4调用上面的releaseShared方法，许可值又由0变成了1，然后准备去唤醒同步队列中的节点，但是此时的头节点的waitStatus是0，不会去唤醒下一个节点，
但是会把waitStatus为0的头节点设置为Node.PROPAGATE，其节点状态如下：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/afe8861043aaed3166ee7fe9bb362d08.png)

（4）t1调用setHead方法将自己的节点顶替之前的head节点，然后判断propagate > 0是否大于零，很明显propagate是等于零的，此时不满足propagate > 0这个条件，转而继续
判断h.waitStatus < 0这个条件，由于旧的头节点被设置为了Node.PROPAGATE，所以h.waitStatus < 0将返回true，t1会唤醒t2

似乎一切都走通了，但是你有没有注意到本人在进行setHeadAndPropagate方法的对比的时候放了JDK7的实现，JDK8比JDK7多了(h = head) == null || h.waitStatus < 0这个判断，这段含义是什么？那会不会是这样的

我们继续以JDK7实现的setHeadAndPropagate方法再以不同的方式走一遍，步骤如下：

（1）假设t1，t2在获取许可失败后，同步队列的节点关系如下图：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/23a0afa9321c734638b8c0d171dce3a5.png)

（2）t3释放许可，调用了上面的releaseShared方法，许可值由0变成了1，然后唤醒了t1，t1获取到了许可，许可返回值为0，还未调用setHead方法，其节点状态如下：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/737205990535eda6b5b9e87d3c64a61c.png)

（3）t4释放许可，t4调用上面的releaseShared方法，许可值又由0变成了1，然后准备去唤醒同步队列中的节点，但是此时的头节点的waitStatus是0，不会去唤醒下一个节点，
正准备将waitStatus为0的头节点设置为Node.PROPAGATE，可是第4步提前发生了

（4）t1调用setHead方法将自己的节点顶替之前的head节点，然后判断propagate > 0是否大于零，很明显propagate是等于零的，此时不满足propagate > 0这个条件，转而继续
判断h.waitStatus < 0这个条件，由于第三步还没有来得及将旧的头节点设置为Node.PROPAGATE，所以此时的整个条件都为false，依然没有唤醒t2节点

（5）t4将老的头节点修改为Node.PROPAGATE，继续往下执行doReleaseShared方法中if (h == head) break;代码发现头节点发生了变化，然后循环，然后唤醒了t2。？？？似乎
没啥问题啊！

下面我们将用JDK8的实现重走上面的步骤：

（1）假设t1，t2在获取许可失败后，同步队列的节点关系如下图：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/e0b0ab35bab68dcfa40df7a819d07090.png)

（2）t3释放许可，调用了上面的releaseShared方法，许可值由0变成了1，然后唤醒了t1，t1获取到了许可，许可返回值为0，还未调用setHead方法，其节点状态如下：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d7afb731e38e5ed7d0ece1812212275f.png)

（3）t4释放许可，t4调用上面的releaseShared方法，许可值又由0变成了1，然后准备去唤醒同步队列中的节点，但是此时的头节点的waitStatus是0，不会去唤醒下一个节点，
正准备将waitStatus为0的头节点设置为Node.PROPAGATE，可是第4步提前发生了

（4）t1调用setHead方法将自己的节点顶替之前的head节点，然后判断propagate > 0是否大于零，很明显propagate是等于零的，此时不满足propagate > 0这个条件，转而继续
判断h.waitStatus < 0这个条件，由于第三步还没有来得及将旧的头节点设置为Node.PROPAGATE，所以此时的条件propagate > 0 || h == null || h.waitStatus < 0返回false
，由于JDK8中增加了(h = head) == null || h.waitStatus < 0的判断，t1后面又挂着t2，所以此时t1的的waitStatus是SINGAL（-1），所以会唤醒t2。

我们分别用JDK7与JDK8走了相同的步骤，但是最终都能够唤醒t2，那么JDK8比JDK7多加的几个判断有啥意义呢？我们还是到Doug Lea代码提交历史中去找找答案
1. 首先进入Doug Lea的代码提交网站，地址我直接定位到了并发包下：http://gee.cs.oswego.edu/cgi-bin/viewcvs.cgi/jsr166/src/main/java/util/concurrent/locks/

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/3f41cf6f392ed662b3726988afbaebd0.png)

2. 点击上图中的AbstractQueuedSynchronizer.java，进入以下页面

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/6378dc2730a75c8e10339b35c66c5b66.png)

上面用蓝色画笔圈起来的就是Doug Lea引入Node.PROPAGATE的提交，点击它，可以看到与上一个版本的对比

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/75f19b90754d7a6f1b88cbcfca02a03b.png)

好了，回到正题，我们找到JDK8比JDK7新增的一些条件判断的提交历史，如下图：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/8b900e4f1fe78968ff8ba8d0a2513037.png)

看注释：Recheck need for signal in setHeadAndPropagate --》 重新检查setHeadAndPropagate中是否需要信号

似乎不是为了解决什么bug添加的代码，只是为了让信号能够及时的传递，并且像接力一样的传播，由t1传播给t2，t2在传播给其他的节点，否则就会像上面走JDK7的步骤那样，
需要释放锁的t4线程去唤醒t2节点。抛开传播的特性来讲，按道理一个节点为SINGAL，它就应该拥有继续往下传播的权利，而不是因为老的head节点缘故，阻止自己的传播。

## 四、总结

读写锁与独占锁最大的区别就是增加了共享锁，共享锁的特点就是当前处于共享状态时（被共享锁持有锁），独占锁被挂起，处于独占锁时，获取共享锁和其他要获取独占锁的线程都
会被挂起，检查当前处于什么类型锁的一个重要的依据就是state变量，state被换分成了高16位和低16位，高16代表共享锁获取的个数（包括重入共享锁），低16表示独占锁被持有的
个数。
对于公平读写锁来说，在共享锁定情况下队列中存在等待中的节点（无论是独占还是读写锁）并且不是重入的时候，将视为获取锁失败，很公平，但是对于非公平的读写锁来说，在共
享锁定情况下，仅仅在队列中第一个节点（头节点除外）为写锁节点并且不是重入时才会直接判定为获取锁失败（避免写锁饥饿），其他情况直接抢锁，一点都不公平。