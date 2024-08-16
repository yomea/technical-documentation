## 独占锁获取锁的整体流程图
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/eef1d28835b1d94496a7b1762a634c5b.png#pic_center)

## 一、AbstractQueuedSynchronizer基础

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/5e186c1ea5b244209a8f2893e12c5d56.png)

### 1.1 父类AbstractOwnableSynchronizer

```java
/*
 * @since 1.6
 * @author Doug Lea
 */
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable {

    /**
     * 记录独占当前锁的线程
     */
    private transient Thread exclusiveOwnerThread;

    /**
     * 设置竞争到锁的线程
     */
    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }

    /**
     * 获取独占着这把锁的线程
     */
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
}
```
这个类主要就是定义了当前锁的所属关系，将竞争到锁的线程进行所属关系的关联，表示当前这把锁被某个线程占用了，通常用于锁重入判断。

### 1.2 内部类Node

这个类用于将线程包装成Node，存放到队列中

```java
static final class Node {
    //共享锁模式节点，设置在nextWaiter属性上，用于辨别当前节点正在竞争何种模式的锁
    static final Node SHARED = new Node();
    
    //独占锁模式节点
    static final Node EXCLUSIVE = null;

    //取消锁资格，通常是因为我们设置了获取锁的超时时间，比如调用tryAcquireNanos(int arg, long nanosTimeout)方法
    //当在指定时间内没有获取到锁时，该节点将会被取消锁资格
    //这样的节点将会被移除队列
    static final int CANCELLED =  1;
    
    //下一个节点将被阻塞的时候，将会设置此节点的wait状态为SIGNAL
    //当此节点被释放或者被取消的时候，需要唤醒下一个节点
    static final int SIGNAL    = -1;
   
    //被Condition挂起的条件节点
    static final int CONDITION = -2;
    
    //用于指示共享锁唤醒传播，只能使用在头节点上，这个属性在分析读写锁时会重点分析
    static final int PROPAGATE = -3;

    //标识当前节点的状态，它的值为上面的CANCELLED，SIGNAL，CONDITION，PROPAGATE
    //在没有设置状态的时候，为零，表示被创建，然后添加到同步队列尾部
    volatile int waitStatus;

    //前一个挂起的线程节点，用于同步队列
    volatile Node prev;

    //下一个被挂起的线程节点，用于同步队列
    volatile Node next;

    //被挂起的线程
    volatile Thread thread;

    //引用下一个被Condition await的节点，
    //也会用于读写锁，用于判断是否为读锁
    Node nextWaiter;
    
    //是否共享锁
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    //获取前驱节点
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    // Used to establish initial head or SHARED marker
    }
    
    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

> 约定：注意下面将CANCEL的wait状态称为取消锁资格，从CONDITION变成其他状态称为取消条件等待

### 1.2 内部类ConditionObject

ConditionObject实现了Condition接口，Condition接口定义了阻塞和唤醒线程的方法，而ConditionObject恰是Condition的实现类，以下是它的成员变量

```java
//链表头
private transient Node firstWaiter;
//链表尾
private transient Node lastWaiter;
```
ConditionObject通过以上两个变量来维护一个条件队列，首先我们先来研究一下，当一个线程调用了await方法之后会发生什么？

#### 1.2.1 await方法

```java
public final void await() throws InterruptedException {
    //线程是否被中断，如果被中断，那么抛出中断异常
    if (Thread.interrupted())
        throw new InterruptedException();
    //将当前线程包装成Node，添加到条件队列中
    Node node = addConditionWaiter();
    。。。。。。省略释放锁等代码
}
                   
```
上面一堆代码首先会判断当前将要挂起的线程是否被中断，如果被中断，那么直接抛出中断异常，如果没有被中断，那么尝试将当前线程包装成一个Node，添加到条件队列中
以下便是添加当前线程到条件队列的逻辑

```java
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    //如果尾节点不再是CONDITION状态，被取消条件等待了，那么需要将这个节点从链表中移除
    if (t != null && t.waitStatus != Node.CONDITION) {
        //从头节点开始，将取消条件等待的节点都移除
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    //将当前线程包装成一个Node，wait状态设置为CONDITION，表示当前节点是被Condition挂起的，并存放在条件队列中
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    //添加到链表尾部
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```
在添加被Condition挂起的节点时，首先会判断尾节点是否被取消条件等待，如果被取消条件等待那么会遍历整个链表将被取消条件等待的节点统统移除，为什么不一上来就遍历
移除掉被取消条件等待的节点呢？如果每次添加节点都要遍历一遍会消耗性能，有的时候可能整个链表都没有一个被取消条件等待的节点，这样就白白遍历了一遍，
做了无用功。还不如直接判断一下尾节点是否被取消条件等待，如果取消掉了，那么就确定一定存在被取消条件等待的node（即使只有尾节点被取消掉了）。

在添加线程到条件队列之后，我们需要释放锁，以便其他线程能够获取锁，下面是释放锁的代码

```java
public final void await() throws InterruptedException {
    。。。。。。省略添加线程到条件队列的代码
    //释放锁，内部调用release方法，具体的释放逻辑我们将在后面介绍
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    //检查当前节点是否已经在同步队列里，检查方式可以通过以下方式判段
    //1、wait状态如果为CONDITION或者没有前驱节点（同步队列一定有一个空的头节点，why？后面分析lock方法的时候再说），那么表示不在同步队列中
    //2、当前节点有next节点的，那么一定是在同步队列中
    //3、遍历同步队列进行对比，如果能找到说明在同步队列中，如果找不到，那么就不在同步队列中
    while (!isOnSyncQueue(node)) {
        //挂起当前线程
        LockSupport.park(this);
        //当线程被唤醒，将继续执行下面这行代码，这行代码主要用于检查线程是否被中断，如果被中断，还是会尝试加入到同步队列，如果是当前线程（wait超时自动唤醒
        //）自己加入到了同步队列中，那么中断模式将被设置为抛出中断异常，如果是其他线程（唤醒线程）将自己加入到同步队列的，那么中断模式为重新中断
        //THROW_IE：表示被signal前的中断，REINTERRUPT：表示被signal后的中断
        //为什么要这么做?
        //JSR133修订以后，就要求如果中断发生在signal操作之前，await方法必须在重新获取到锁后，抛出InterruptedException。但是，如果中断发生在signal后，await
        //必须返回且不抛异常，同时设置线程的中断状态
        //Doug Lea的论文：http://gee.cs.oswego.edu/dl/papers/aqs.pdf
        //中文翻译：http://ifeve.com/aqs/
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //进入无限循环锁竞争逻辑，如果前一个节点的wait状态为SIGNAL，那么被挂起，否则需要将前一个节点CAS设置为SIGNAL
    //acquireQueued返回在获取锁时是否被中断
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        //重新中断
        interruptMode = REINTERRUPT;
    
    //如果线程执行到了这，那么获取到了锁
    if (node.nextWaiter != null) // clean up if cancelled
        //清理那些被取消条件等待的节点。当前线程从条件队列中被转换到同步队列，在转换的过程中并没有将当前线程的节点从条件队列中移除，所以
        //此时在条件队列中必定存在被取消条件等待的节点
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        //如果中断模式是THROW_IE，也就是在wait期间被中断，那么将会抛出中断异常，
        //如果中断模式为REINTERRUPT，也就是在signal之后被中断的，那么会调用Thread.currentThread().interrupt()方法，重新设置中断位
        //让用户去处理这样的中断
        reportInterruptAfterWait(interruptMode);
}
```
将当前线程包装成Node添加到条件队列之后，需要释放锁，以便其他线程能有机会去竞争锁。然后将自己挂起，等待被唤醒，当被唤醒时，会判断当前线程是否被中断，如果没
有被中断，循环判断当前线程对应的节点是否已经被加入到同步队列。如果加入到了同步队列，那么跳出循环进入无限循环获取锁的逻辑，如果获取失败将再次阻塞线程。

> Q：为什么要循环判断节点是否已经入同步队列而不是直接往下执行？

> A：对于这种没有wait超时时间的，我个人的理解即使没有isOnSyncQueue判断也是可以的，对于有wait超时时间的，是需要isOnSyncQueue判断的，因为可能存在某个线程调用
signal方法的同时，wait阻塞时间已经超时然后自动唤醒，此时对于同一个节点来说是存在竞争关系的，当唤醒线程将节点的wait状态由CONDITON修改为0后，会将节点进行入
同步队列的逻辑，如果那个因wait超时被自动唤醒的线程在CAS修改节点wait状态失败，那么它需要进行自旋等待唤醒线程将节点添加到同步队列中才能继续往下去竞争锁。

#### 1.2.2 signal方法

```java
public final void signal() {
    //如果当前线程不是拥有锁的对象，将会抛出错误
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        //从头节点开始寻找下一个被唤醒的线程
        doSignal(first);
}
```
doSignal方法代码如下：

```java
private void doSignal(Node first) {
    //从头节点开始遍历，直到找到一个可以被通知的节点
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}

final boolean transferForSignal(Node node) {

    //修改失败表示节点被取消条件等待，为什么可能会被取消条件等待？我们除了会调用await方法之外，也有可能调用的是一个awaitNanos这种限制了超时时间的方法
    //超过指定时间后线程自动被唤醒，然后将wait状态修改为0，加入到同步队列中，准备竞争锁
    //所以对于已经取消条件等待的节点，不会再次加入到同步队列中
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    //将节点加入到同步队列，返回node的前驱节点
    Node p = enq(node);
    int ws = p.waitStatus;
    //如果前驱节点已被取消获取锁资格（比如获取锁超时）或者修改前驱节点的wait状态失败（可能前驱节点自己获取锁超时而被取消获取锁资格，也可能是因为当前Node
    //节点wait超时，竞争锁时率先把前驱节点的wait状态改成过了SIGNAL）
    //在以上条件下，将会唤醒这个节点的线程，为什么不直接唤醒？因为同步队列是FIFO的，你前面的节点如果是SIGNAL的，它都还没有获取到锁，你个后面加入的就得先等
    //着。那为啥前驱节点被取消锁资格之后就需要被唤醒？当前驱节点被取消锁资格后，可能前面所有的节点都已经取消锁资格，唤醒线程，即使一开始获取不到锁，也会
    //将被取消锁资格的节点剔除队列
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}

```

唤醒条件队列中的节点的时候，首先会判断当前节点是否已经取消了条件等待，如果已经取消掉了，那么不会重复加入到同步队列中。

> Q：那么问题来了，我们知道signal方法是在同步块中进行的，这会发生竞争吗？有什么原因导致我们将要通知的节点已经被其他线程给通知了呢？

> A：当我们调用的条件等待方法具有超时条件的时候将可能发生竞争的问题。比如我调用了一个叫做awaitNanos(java.lang.Long)方法，在指定阻塞时间之后，线程被自动唤醒
，它将自己取消了条件等待，加入到了同步块。与此同时，另一个刚因为前一个线程await的时候获取到锁的线程调用了signal方法，那么他们可能就会发生并发。

将需要通知的节点加入到同步队列之后，检查加入节点的前驱节点，如果前驱节点被取消锁资格，那么将会唤醒被通知节点关联的线程

> Q：为什么在前驱节点被取消锁资格时才主动唤醒，难道不能直接唤醒吗？管它前驱节点啥状态

> A：同步队列是一个先进先出的队列，如果被通知的节点的前一个节点是SIGNAL状态的，那么只有它发生锁的释放或者被取消锁资格后，后继节点（被通知节点）
才会被唤醒去竞争锁。如果前一个节点是SIGNAL状态的，还没有获取到锁，那么即使唤醒被通知的节点，它接下来还是会被阻塞，这样就浪费性能了。
如果被通知的节点的前一个节点是被CANCEL的，唤醒被通知的节点，可以剔除掉那些CANCEl状态的节点，甚至前面所有的节点为CANCEL状态时，还可以有机会去竞争锁

## 二、ReentrantLock

### 2.1 类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ddd4494cd852b7f805b860c343bc8d6b.png)

- Lock：定义获取锁，获取Condition（用于阻塞线程和唤醒线程的接口），定义了以下几种获取锁的方法
    - lock：获取锁，如果获取不到那么阻塞
    - lockInterruptibly：首先检查是否被中断，然后获取锁，如果获取不到将被阻塞，当被唤醒时，会继续判断是否被中断，被中断将会抛出InterruptedException异常
    - tryLock：尝试获取锁，如果获取到锁将返回true，如果失败，不会阻塞，返回false，对于可以指定时间的tryLock方法表示在指定时间内获取不到锁将返回false
    - unlock：释放锁
    - newCondition：创建状态队列，用于挂起线程和唤醒线程
- Condition：状态队列，用于挂起线程和唤醒线程
- Sync：ReentrantLock的内部类，模板类，提供了一些基本的时候，比如不公平锁的获取，锁释放
- FairSync：公平锁
- NonfairSync：不公平锁，怎么个不公平法，后面进行分析

### 2.2 公平锁

#### 2.2.1 锁的获取

```java
protected final boolean FairSync#tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    //c大于零，表示锁已经被占用，c的值表示锁被重入c次
    int c = getState();
    if (c == 0) {
        //首先判断队列中是否要正在等待锁的节点，如果没有通过CAS操作竞争锁
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            //将当前线程设置为锁的主人
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //如果当前线程就是锁的主人，那么c的计数递增
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

公平锁在获取锁的时候，首先判断锁是否被占用，如果被占用再判断当前线程是否是锁持有者，如果是，那么进行重入计数即可，否则需要进行锁的竞争，但是在竞争之前它会
先判断同步队列中是否已经有在等待锁的节点，如果有那么就不去竞争锁了，这就是体现了公平锁公平的地方了，想要锁可以，但是要排好队，一个一个来。

hasQueuedPredecessors方法用于判断当前同步队列中是否存在被当前线程等待更久的线程

```java
//判断同步队列中是否有节点
public final boolean hasQueuedPredecessors() {
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```
当h != t时，那么能够肯定队列中一定存在等待着的节点，那么(s = h.next) == null这段代码是什么意思呢？既然肯定存在等待着的节点，那么h.next不应该为null啊？我们
看到插入节点的一小段代码

```java
Node t = tail;
if (t == null) { // Must initialize
    if (compareAndSetHead(new Node()))
        tail = head;
}
```
假设此时执行到了compareAndSetHead(new Node())这段代码，并且已经将头节点设置成功，还没有来得及执行tail = head;这段代码，那么此时head与tail不相等，head的next
也为null，但是已经表明有比当前线程更早的线程在等待了，但是从另一方面来讲，如果已经执行了tail = head;那么head == tail，head的next还没有来得及设置，那么上面
的head != t似乎少了这种情况的判断，head == tail还有另外一种情况，那就是同步队列中的节点已全部经获取过锁了并全部执行完毕，那么此时的head == tail。这样
head == tail这种情况似乎又出现了歧义。


当获取不到锁的时候，需要将竞争锁失败的线程包装成Node，加入到同步队列中

```java
private Node addWaiter(Node mode) {
    //将当前线程包装成一个Node，第二个参数在同步队列中表示锁的模式，这里是独占锁模式，设置的值为
    //java.util.concurrent.locks.AbstractQueuedSynchronizer.Node#EXCLUSIVE
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    //将新new出来的Node加入到队尾
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
                            |
                            V
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        //如果没有初始化，那么先进行初始化，创建一个空的头节点
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            //将新加入的节点设置到链表尾部，然后返回前一个节点
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

```
当竞争锁失败之后，会将当前线程包装成Node添加到同步队列中，在加入的过程中首先会判断是否已经初始化过了
- 如果初始化过了之后，通过cas操作加入到队尾，如果CAS失败，会进入无限循环，直到加入队列成功。
- 如果没有初始化，那么会通过CAS操作创建一个空的头节点，然后再将新节点插入到队尾

> Q：为什么要创建一个空的头节点？

> A：一个线程在被阻塞之前，它的前驱节点必须被设置为SIGNAL，SIGNAL代表当前节点在释放锁的时候需要通知下一个节点。为零的话，那么表示没有需要通知的节点了，在
释放锁时不再去尝试唤醒下一个节点。


下面这段代码是每个锁竞争失败线程的阻塞点和唤醒之后的锁竞争点

```java
final boolean acquireQueued(final Node node, int arg) {
    //是否获取锁失败，在设置了超时获取锁的方法中有用
    boolean failed = true;
    try {
        //线程中断标志
        boolean interrupted = false;
        for (;;) {
            //获取当前节点的前驱节点
            final Node p = node.predecessor();
            //对于刚加入的新节点，如果前驱节点是头节点，那么尝试竞争锁，当线程执行到这里的时候，有可能上一个获取到锁的线程已经释放了锁，所以可以试着再次去
            //竞争锁。除此之外，上个线程的锁释放并不一定能通知到当前节点，因为此时的头节点wait状态可能是零，不是SIGNAL，为什么可能？如果当前线程被包装成
            //Node的时候，同步队列中没有任何节点，此时加入的当前节点还没来得及将头节点设置为SIGNAL，就会导致丢失signal信号
            //对于被唤醒的节点，那么是按照队列的FIFO原则进行锁竞争的
            if (p == head && tryAcquire(arg)) {
                //如果锁竞争成功，那么将当前节点设置为头节点，并把对应持有的线程置空，变成一个空节点
                setHead(node);
                p.next = null; // help GC
                //表示锁竞争成功
                failed = false;
                return interrupted;
            }
            //检查锁竞争失败之后是否需要挂起
            if (shouldParkAfterFailedAcquire(p, node) &&
                //挂起并检查线程是否发生了中断
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            //取消节点的锁资格
            cancelAcquire(node);
    }
}
```
acquireQueued方法的逻辑可以分为以下情况：
- 如果是新加入的节点，那么首先会获取它的前驱节点，判断其是否为头节点，如果是头节点会尝试再次去获取锁，因为在此时上一个获取到锁的线程有可能已经释放了锁。
- 如果是被唤醒的节点，那么按照队列的FIFO原则去获取锁，获取到锁之后会将当前节点设置为头节点，并去掉其关联的线程，变成一个只保留了wait状态的空节点

#### 2.2.2 线程的挂起

下面再看到shouldParkAfterFailedAcquire方法，不过得先等等，在研究这个shouldParkAfterFailedAcquire方法之前我们先来研究下挂起的方法

parkAndCheckInterrupt方法

```java
private final boolean parkAndCheckInterrupt() {
    //调用本地方法挂起线程
    LockSupport.park(this);
    //判断当前线程是否被中断，如果被中断将会返回true，并清除中断标志
    return Thread.interrupted();
}
```

shouldParkAfterFailedAcquire方法

```java
//pred：表示node节点的前驱节点，node：表示当前节点
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    //如果前驱节点的wait状态为SIGNAL，那么返回true，表示可以挂起线程
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
        //如果wait状态大于零，那么说明它的前驱节点被CANCEL（取消锁资格）了，那么需要剔除掉这些节点，知道找到一个不是CANCEL了节点作为新的前驱节点
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
         //寻找没有被CANCEL的前驱节点，并重新建立关系
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
         //如果既没有SIGNAL也没有被CANCEL，那么通过CAS操作设置为SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    //返回false，表示没有达到挂起线程条件，需要再次尝试获取锁
    return false;
}
```

从上面的一段代码可知，一个节点被挂起的前提是前驱节点必须是SIGNAL的。如果前驱节点被取消锁资格，那么会从后往前一直找到那个没有被CANCEL的节点，然后建立新的前后关系。如果前驱节点是其他的状态，比如0和PROPAGATE，那么会通过原子操作设置为SIGNAL，并返回false，返回false表示没有达到挂起的条件，需要重新尝试去竞争锁。

> Q：为什么在讲前驱节点设置为SIGNAL之后不直接挂起呢？

> A：分两种情况<br />
1、前驱节点被CANCEL了，那么会寻找到没有被CANCEL的前驱节点，此时有这么一种可能，那就是当前节点前面的所有节点都是CANCEL的，当前节点为第二个节点<br />
2、前驱节点wait状态为0或者PROPAGATE，当前节点有可能是第二个节点<br />
以上两种情况都可能存在这样一个情况，那就是在设置前驱节点为SINGAL之前，锁可能已经被释放了，那么当前节点将丢失signal信号，所以需要去尝试判断前驱节点是否为头
节点，如果是头节点会进行锁竞争。

#### 2.2.3 节点的CANCEL

当一个线程获取锁被指定了超时时间时，如果在指定时间内没有获取到锁，那么将会被CANCEL

```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    //计算deadline
    final long deadline = System.nanoTime() + nanosTimeout;
    //将线程包装成Node加入到同步队列中
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            //获取前驱节点
            final Node p = node.predecessor();
            //如果前驱节点为头结点，那么尝试竞争锁
            if (p == head && tryAcquire(arg)) {
                //将当前节点设置为头节点，并将光联的线程置空，变成空节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            //计算时间是否超时
            nanosTimeout = deadline - System.nanoTime();
            //如果超时将返回
            if (nanosTimeout <= 0L)
                return false;
            //线程的挂起
            if (shouldParkAfterFailedAcquire(p, node) &&
                //如果超时时间已经非常小了，小于1000ns，那么自旋
                nanosTimeout > spinForTimeoutThreshold)
                //指定挂起时间
                LockSupport.parkNanos(this, nanosTimeout);
            //如果发生中断，将会抛出异常
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        //如果超时获取锁失败，将节点设置为CANCEL
        if (failed)
            cancelAcquire(node);
    }
}
```
如果给一个线程设置了超时时间，这个线程在指定时间内却没有竞争到锁，那么对应的节点将被CANCEL

```java
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;
    //取消线程的光联
    node.thread = null;

    // Skip cancelled predecessors
    //获取当前节点的前驱节点
    Node pred = node.prev;
    //移除被CANCEL的节点
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary.
    //用于后面CAS替换pred.next节点
    Node predNext = pred.next;

    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference from other threads.
    //将当前节点的wait状态设置为CANCELLED
    node.waitStatus = Node.CANCELLED;

    // If we are the tail, remove ourselves.
    //如果当前节点为队列尾部节点，那么移除它
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        //由于上面将一些CANCEL的节点去除掉了，这里需要将断裂的节点重新连接起来
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            //如果pred已经是头节点了，或者pred被取消掉了，那么唤醒下一个节点
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```
当一个线程在指定时间内没有获取到锁（不是说在指定时间一直循环获取锁），那么其对应的Node将被CANCEL，取消锁资格，然后将自己从链表中移除，如果前驱节点也有已经
被CANCEL的，那么也会一并移除，如果前驱节点已经是头节点了，那么会唤醒当前节点的下一个未被CANCNEL的节点

> Q：为什么在前驱节点为头节点的情况下要唤醒下一任节点，而不是将自己移除后直接退出？

> A：假设当前节点是头节点的第二任节点，与此同时，它与其他线程在竞争锁的时候败下阵来而且它还超时了，那么当前节点将被CANCEL，在CANCEL之前，那个与当前节点
竞争的线程有可能已经释放了锁，由于当前节点的存在，它阻挡了singal信号，所以需要当前节点来唤醒下一任节点。

#### 2.2.4 唤醒下一任节点

```java
private void unparkSuccessor(Node node) {
    
    int ws = node.waitStatus;
    if (ws < 0)
        //将当前节点的wait状态重置为0
        compareAndSetWaitStatus(node, ws, 0);
    
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        //如果下一个节点为空或者被CANCEL了，那么从同步队列的队尾开始往前找未被CACNEL的节点
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        //唤醒线程
        LockSupport.unpark(s.thread);
}
```
唤醒下一个节点，如果下一个节点被CANCEL了，或者当前节点已经是队尾节点，那么从队尾节点开始往前找没有被CANCEL的节点进行唤醒。

> Q：为什么当前节点为tail节点时还要从队尾往前找？
> A：因为存在并发，有可能此时已经有新的节点被插入

> Q：为什么要从尾节点开始往前找未被CANCEL的节点？
> A：在插入节点的时候，代码如下：

```java
//首先设置当前节点的前驱节点
node.prev = pred;
//然后通过CAS操作将当前节点设置为tail节点
if (compareAndSetTail(pred, node)) {
    //设置成功的话就将前驱节点的next指向当前节点
    pred.next = node;
    return node;
}
```
从这段代码插入节点的代码来看，插入一个节点它不是原子的，当执行到compareAndSetTail(pred, node)这段代码的时候，如果tail节点已经被替换成功，
但是pred.next = node;这段代码还没有来得及执行的时候，我们从前往后遍历，我们就无法遍历到node这个节点，如果我们从后往前遍历，那么我们是从node节点开始的，那么
就不会遗漏节点。如下图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0b892c7bfe55fbd30d11b86732ccabeb.png)

这就是执行到compareAndSetTail(pred, node)这段代码并且还成功了，节点之间的状态，如果我们从前往后遍历，node节点就不会被遍历到，从后往前就能保证一定能够遍历到

看了以上唤醒下一任节点的方法，发现在唤醒下一任节点的时候会被当前节点的wait状态修改成0，于是有了以下这个问题：

> Q：会不会出现这么一种情况，假设B线程对应的Node节点为同步队列中第二个节点并且头节点的wait状态为SIGNAL，B线程执行到了if(ws == Node.SIGNAL)这段代码，
并且准备返回true了，也就是已经达到阻塞的条件，此时另外一个线程A刚好释放了锁还调用了singal方法，将头节点的wait状态修改成了0，然后去唤醒线程B，此时的B还是
醒着的，还没有调用阻塞方法，知道线程A调用unPark方法后，线程B才调用park方法准备阻塞自己，那是不是意味着signal信号的丢失了呢？

> A：线程A先调用unPark(B线程)，然后B线程再调用park(B线程)，此时B线程不会被阻塞，是直接被唤醒。what？我们就low一点，直接理解为每个线程上有一个标志位，表示自
己被唤醒，当线程A调用unPark(B线程)时，线程B的唤醒标志位被设置为1，当线程B调用park(B线程)时，发现自己的唤醒标志位为1，那么无需阻塞。多次调用unPark会怎样，不
会怎样，就跟调用一次是一样的，因为多次调用也是将标志位设置为1。

### 2.3 不公平锁

不公平锁和公平锁唯一的区别在于下面这段代码

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        //这段代码与公平锁对比，公平锁会判断同步队列中是否有在等待的节点，如果没有才会进行锁的竞争
        //而不公平锁就不一样，我管你有没有线程在等待，我上来就直接竞争锁，插队
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        。。。。。。省略重入逻辑
    }
    return false;
}
```

这节主要分析了AQS并发框架和独占锁的实现，为我们下面继续分析并发包内其他的类打下了基础。下一节我们将分析读写锁的实现。

## 三、总结
### 3.1 AQS

AQS是一个抽象的同步队列，实现了线程被包装成节点，然后加入到同步队列并挂起的基本能力，它使用一个int类型state用于控制是否获取到锁，同步队列是一个FIFO类型的队列，其
节点有这么几种waitStatus，SIGNAL，CANCELLED，CONDITION，PROPAGATE。
- SIGNAL用于表示在当前节点唤醒（共享锁）或者释放锁的时候，有下一个节点需要进行唤醒
- CANCELLED表示某个节点被取消，比如有些线程获取锁时会设置超时时间，如果超过时间还未获取到锁，那么将被取消
- CONDITION是调用Condition.await()之后阻塞在条件队列中的节点
- PROPAGATE这个状态用于共享锁，表示共享锁唤醒传播，从某个共享锁唤醒开始往节点尾部传播，最初引入这个状态主要是为了解决一个信号量导致的bug，详情可查看下篇读写锁博文

当一个线程在没有获取到锁的时候，会将前驱节点的waitStatus设置为SIGNAL，设置为SIGNAL后还会再尝试一遍去获取锁，如果存在其他的等待节点或者没有其他等待节点并且竞锁失
败时，那么就会被阻塞，如果是设置了超时时间的，那么超时后将会取消当前线程对应的节点，设置waitStatus为CANCELLED

释放锁时，将判断头节点是否小于0（SIGNAL是小于零），如果小于零表示有线程正在等待被唤醒。如果第二节点被CANCELLED，那么将从尾部往前寻找到离头节点最近的
第一个未被CANCELLED的节点进行唤醒，为什么这么做，主要存在并发插入的情况，并且插入节点并非原子的，cas替换时的操作为尾部节点的pre指向旧的tail节点，旧的tail节点还
未来得及将next指向新插入的节点，所有从后往前遍历是可以遍历到完整的节点的。从后往前遍历更容易找到未被CANCELLED的节点

### 3.2 ConditionObject

调用Condition.await()会释放锁，然后将自己包装成Node加入到条件队列当中，这个时候存在三种情况，一种是线程中断导致自己被唤醒，第二种是线程设置了wait超时时间，第三种
是被其他线程signal，在wait阶段被中断的线程将再获取到锁之后抛出中断异常，在唤醒之后发生的中断，会重新设置中断标记交由用户处理。一个条件队列中的线程被唤醒之后，
需要将自己的waitStatus由CONDITON修改成0，然后在检查是否有其他等待的节点或者没有其他等待节点并竞争锁失败时加入到同步队列中挂起。

### 3.3 锁的公平性

公平锁的主要特点就是先检查同步队列中是否已经存在正在等待的节点，如果已经存在那么就会加入到同步队列中进行挂起。非公平锁则不会，它一上来就先去竞争锁，如果失败才加
入到同步队列中挂起

## 四、参考资料

- Doug Lea的论文英文版：http://gee.cs.oswego.edu/dl/papers/aqs.pdf
- Doug Lea的论文中文版：http://ifeve.com/aqs/