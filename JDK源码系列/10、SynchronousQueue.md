## 一、类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/8d6488409c76be9c74014d1ec114e653.png)

SynchronousQueue从字面意思上来将就是一个同步队列，啥是同步队列？你只要往同步队里中添加元素，你的线程就会被阻塞，直到另外一个线程去获取对应的元素并唤醒这个
线程。根据获取元素的顺序，同步队列也分为公平与不公平的，公平的使用FIFO队列实现，对应的内部类为TransferQueue，不公平由栈实现，对应的内部类为TransferStack

以下为SynchronousQueue一些字段说明

```java
//cpu个数，用于自旋控制
static final int NCPUS = Runtime.getRuntime().availableProcessors();

//最大自旋次数，在多个cpu的时候，自旋次数为32，这个值是根据经验所得
static final int maxTimedSpins = (NCPUS < 2) ? 0 : 32;

//非定时自旋，如果用户没有指定需要定时，那么将会使用这个值进行自旋，由于非定时的自旋不需要计算超时时间，从某种情况下来说，
//非定时自旋的时间即使次数比较多，也有可能比定时时间的自旋快
 static final int maxUntimedSpins = maxTimedSpins * 16;
 
 //如果超时时间间隔已经小于这个1000ns了，那么就没必要阻塞了（即使阻塞很快就醒了）
 //将阻塞改为自旋
 static final long spinForTimeoutThreshold = 1000L;

```


### 1.0 Transferer

Transferer为TransferQueue和TransferStack的抽象父类

```java
abstract E transfer(E e, boolean timed, long nanos);
```

transfer这个方法用来定义如何传递传入的对象和如何将值传递出去，后面将会将传递数据的线程称之为生产线程，取值的线程称为消费线程
- e：表示用户需要用于传递的值，后面将称之为产品
- timed：表示是否需要定时，与nanos参数配置使用，一个线程将数据进行transfer时，在指定时间内进行阻塞
- nanos：定时时间，与timed配合使用

### 1.1 TransferQueue

#### 1.1.1 TransferQueue基础

TransferQueue是一个FIFO的队列，SynchronousQueue使用它来保证公平性，TransferQueue使用的数据结构为单向链表，这个链表的每个元素是一个叫做QNode的内部类，以下是
这个内部类字段的说明

```java

static final class java.util.concurrent.SynchronousQueue.TransferQueue.QNode {
    //下一个QNode
    volatile QNode next;          // next node in queue
    
    //节点对应的用户传入的数据（产品）
    volatile Object item;         // CAS'ed to or from null
    
    //用户put 这个item时使用的线程
    volatile Thread waiter;       // to control park/unpark
    //是否为数据，如果用户调用transfer方法时，传入了数据，那么isData为true，如果没有传入数据，那么是false，表示我是来拿数据的
    final boolean isData;
    
    //通常用于获取某对象某字段的内存偏移地址
    private static final sun.misc.Unsafe UNSAFE;
    //QNode的item字段在内存中的偏移地址
    private static final long itemOffset;
    //QNode的next字段在内存中的偏移地址
    private static final long nextOffset;
    
    。。。。。。省略QNode的方法声明

}
```


以下是TransferQueue的字段说明

```java
static final class TransferQueue<E> extends Transferer<E> {

    //。。。。。。省略部分代码
    
    TransferQueue() {
        //初始化一个虚拟的头节点
        QNode h = new QNode(null, false); // initialize to dummy node.
        head = h;
        tail = h;
    }

    //表示头节点
    transient volatile QNode head;
    
    //表示尾节点
    transient volatile QNode tail;
    
    //用于表示延后清理的节点，具体的意思我们在分析清理节点方法的时候会详细说明
    transient volatile QNode cleanMe;
    
    //一个用于底层操作的工具类，可以获取某对象某个字段在内存中的地址
    private static final sun.misc.Unsafe UNSAFE;
    
    //头节点的偏移地址，向这个地址写入数据，那么就意味着给head赋值
    private static final long headOffset;
    
    //TransferQueue的tail字段内存偏移地址
    private static final long tailOffset;
    
    //TransferQueue的cleanMe字段内存偏移地址
    private static final long cleanMeOffset;

    //。。。。。。省略部分代码
}
```
#### 1.1.2 数据传入


```java
E transfer(E e, boolean timed, long nanos) {

    QNode s = null; // constructed/reused as needed
    boolean isData = (e != null);

    for (;;) {
        QNode t = tail;
        QNode h = head;
        //在构建TransferQueue的时候，会在其构造器中创建QNode，然后将设置设置给头节点和尾节点
        //如果还没有值，那么说明发生了多线程的访问，某些线程访问到了还未完全初始化的对象
        if (t == null || h == null)         // saw uninitialized value
            continue;                       // spin
        
        //h == t：表示此时队列中还没有用户存入的数据
        //t.isData == isData：一般是队列中已经存在用户数据了，然后又有用户传入数据
        //也有可能是队列为空的时候，用户进行pull，往外拿东西，往外拿东西的时候，传入的数据为null
        if (h == t || t.isData == isData) { // empty or same-mode
            QNode tn = t.next;
            //如果此时tail发生改变，就是其他线程修改了tail节点，重新再来一次
            //如果我某时刻t==tail，然后执行完t==tail判断为true，然后立马被另外一个线程给修改了,这看起似乎没有必要现在判断吧？
            //提前判断可以在一定程度上减少CAS的开销
            if (t != tail)                  // inconsistent read
                continue;
            //如果尾节点的next已经有值，帮助那个设置next的线程设置尾节点
            //有可能有人会不理解这个地方为什么tn不为空的时候，t == tail
            //如果你继续往下看你就会发现，设置t的next之后，并把next设置为tail不是原子的，这个时候如果有另外一个线程先设置了t的next，但是还没有来得及设置
            //t的next为tail节点，这个时候就会出现tn不为空的现象。
            if (tn != null) {               // lagging tail
                //帮助设置tail节点
                advanceTail(t, tn);
                continue;
            }
            //设置了定时，但是却没有给对应的时间，直接返回null，如果用户在队列没有值的情况调用了SynchronousQueue的pull方法，那么会走到这个语句里面
            //直接返回null，当然即使是设置了值，指定了timed为true却没有给值，那么一样返回null
            if (timed && nanos <= 0)        // can't wait
                return null;
            if (s == null)
                //创建新的QNode节点
                s = new QNode(e, isData);
            //CAS设置t的下一个节点
            if (!t.casNext(null, s))        // failed to link in
                continue;
            //CAS设置尾节点
            advanceTail(t, s);              // swing tail and wait
            //准备挂起线程，等待其他线程取值
            Object x = awaitFulfill(s, e, timed, nanos);
            //x == s：表示节点被cancel，如果线程被中断或者wait超时，节点的值将被设置为其本身或者已经节点被其他线程获取过值了
            if (x == s) {                   // wait was cancelled
                //清理掉这个节点，从节点中移除
                clean(t, s);
                return null;
            }
            //如果节点的next引用的自身，那么表示这个节点从队列中被移除了
            //如果没有被移除，那么尝试帮助获取数据的线程将节点前移
            if (!s.isOffList()) {           // not already unlinked
                advanceHead(t, s);          // unlink if head
                //从目前的实现来说，x的值要么是当前节点s，要么是用户传入的值，要么是null
                //x == s的情况表示要clean，x == 用户之前传入的值的情况下不会跳出awaitFulfill方法
                //如果为null会跳出awaitFulfill方法
                //所以x != null不会为true，但是为什么要加这段代码进行判断呢？
                //不排除以后有其他形式的获取用户值的方式，比如调用SynchronousQueue的pull的方法时，我不在传入null，我可以传入其他的值，以便和原来放入值
                //的线程做交换，类似于我卖包子，你给钱
                if (x != null)              // and forget fields
                    //cancel掉这个节点
                    s.item = s;
                s.waiter = null;
            }
            //在通常情况下，x为空，返回用户之前传入的值，这个x不排除在未来设计成可以返回其他的不为空的值，比如卖家给包子，买家给钱。
            return (x != null) ? (E)x : e;

        } else {                            // complementary-mode
            。。。。。。省略获取数据代码
        }
    }
}
```
数据传入主要分为以下几个步骤：
1. 先判断用户将要做什么操作，如果有传入值，那么认为是存入数据，如果数据为空，那么认为是需要取数据
2. 然后判断尾节点是否发生变更，如果发生变更，那么重新循环，避免CAS带来的开销
3. 创建节点，CAS设置到尾节点的next中，然后变换尾节点
4. 最后进行await
5. 如果被中断，那么会取消节点，取消的方式就是将节点的数据设置成它本身，然后准备移除这个被cancel的节点
6. 如果被正常唤醒，那么将会尝试前移节点到头节点。


下面我们来分析下线程的await

```java
Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
    /* Same idea as TransferStack.awaitFulfill */
    //如果需要定时，那么会计算最后的期限
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    Thread w = Thread.currentThread();
    //如果头节点的下一个节点是自己，那么自旋
    //为什么必须是头节点的下一个节点才会会自旋?
    //因为TransferQueue是FIFO的，如果自己已经不是头节点的第二节点，那么必须等前面的节点被取走，才会轮到自己，所以没有必要自旋了
    int spins = ((head.next == s) ?
                 (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    for (;;) {
        //如果被中断，那么将节点cancel，cancel的方式就是将节点的值设置为当前节点对象
        if (w.isInterrupted())
            s.tryCancel(e);
        Object x = s.item;
        //如果值发生变化，那么返回
        if (x != e)
            return x;
        //定时，如果超时，cancel节点
        if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                s.tryCancel(e);
                continue;
            }
        }
        //自旋
        if (spins > 0)
            --spins;
        else if (s.waiter == null)
            s.waiter = w;
        else if (!timed)
            LockSupport.park(this);
            //如果离await超时时间非常接近了，那么没有必要阻塞了，改为自旋
        else if (nanos > spinForTimeoutThreshold)
            //阻塞指定时间
            LockSupport.parkNanos(this, nanos);
    }
}
```

#### 1.1.3 获取数据

生成线程生产完产品之后，那么消费线程就可以去消费产品了

```java
E transfer(E e, boolean timed, long nanos) {
    
    QNode s = null; // constructed/reused as needed
    boolean isData = (e != null);

    for (;;) {
        QNode t = tail;
        QNode h = head;
        if (t == null || h == null)         // saw uninitialized value
            continue;                       // spin

        if (h == t || t.isData == isData) { // empty or same-mode
           。。。。。。省略数据存储代码
        } else {                            // complementary-mode
            //从头节点开始读取下一个节点
            QNode m = h.next;               // node to fulfill
            //如果头节点，尾节点，没有下一个节点，那么循环
            //m == null：对于那些队列确实已经没有节点的情况，会返回null
            //h != head：头节点发生变化，被其他线程先获取了数据，那么当前线程应当尝试读取下一个节点
            //t != tail：尾节点发生变化
            if (t != tail || m == null || h != head)
                continue;                   // inconsistent read
            
            //获取节点值
            Object x = m.item;
            //走这个分支的时候isData为false，如果x也为空，那么说明当前m节点的值已经被其他线程抢走了
            if (isData == (x != null) ||    // m already fulfilled
                //x == m表示节点被cancel，节点被取消说明创建这个节点的线程已经被中断
                x == m ||                   // m cancelled
                //cas将m节点的旧值改为新值，新值为null
                !m.casItem(x, e)) {         // lost CAS
                //如果确实被其他线程抢走了值或者节点被cancel了，那么直接前移头节点即可
                advanceHead(h, m);          // dequeue and retry
                continue;
            }
            //节点数据竞争成功前移头节点
            advanceHead(h, m);              // successfully fulfilled
            //唤醒对应的生产线程
            LockSupport.unpark(m.waiter);
            return (x != null) ? (E)x : e;
        }
    }
}
```
所有的消费线程都从队列头开始消费，如果对应的节点已被其他线程消费，那么将目标转移到下一个节点，继续和其他消费线程竞争，直至竞争成功或者已没有可消费的对象
为止，然后被消费的节点的值会置为null，其节点对应的生产线程会被唤醒

这里做个假设，如果脱去SynchronousQueue这层外壳，我们直接调用这个transfer这个方法，此时队列中没有用户数据，我们传入的参数为e=null，time=false，那么将会创建
一个含有null值的QNode，然后线程被阻塞，后面只要是e=false并且time=false或者time=true,nanos有值，那么都会加入到队里中并wait。如果此时有线程持有e != null的
参数进来，那么毫无疑问它将走到原本是消费线程执行的分支代码去了，那么此时会出现什么？<br />
条件 isData == (x != null) 将会成立，接下来就会执行advanceHead(h, m) 和 continue，虽然节点被移除了，但是却无法唤醒那些wait时间的线程。


> Q：在构造TransferQueue时为什么要构建一个空的虚拟头节点呢？不能直接将持有用户数据的业务节点作为头节点吗？

> A：为了方便操作，为了提高性能。假设我们直接使用业务节点作为头节点，此时队列中的节点如下：<br />
A -> B -> C <br />
消费线程从头节点A开始取值，取完将B设置为头节点，B被取完将C设置为头节点，C取完呢？那就要将头节点设置为null，要不然下次将重复C取值，那么问题就来了，如果我
将头节点设置为空的时候，某生产线程就有可能在C的后面连接上了新的节点，那么将会丢失节点，如果引入锁，那么性能就变差了。

#### 1.1.4 清理数据

当生产线程发生中断或者wait超时的时候，就会cancel掉一个节点，那么这个节点属于无效节点，需要移除，就像一个产品一样，如果产品坏了，过期了，那么就不能再卖给别
人，否则要被投诉，伤害他人的健康，从法律，从道德上都过不去。

```java
void clean(QNode pred, QNode s) {
    //清理掉对应的线程
    s.waiter = null; // forget thread
    //如果s节点已经被移除或者前驱节点被消费线程消费移除队列（消费线程在消费到cancel节点的时候会把节点直接移除），那么表示已经被清理，无需再次操作
    while (pred.next == s) { // Return early if already unlinked
        QNode h = head;
        QNode hn = h.next;   // Absorb cancelled first node as head
        //如果头节点的第二任节点由取消的节点了，那么直接替换
        if (hn != null && hn.isCancelled()) {
            advanceHead(h, hn);
            continue;
        }
        QNode t = tail;      // Ensure consistent read for tail
        //队列中已经没有节点了，都被消费了，那么直接返回（消费者线程遇到被cancel的节点将直接跳过并头节点移动）
        if (t == h)
            return;
        QNode tn = t.next;
        //如果头节点发生了变动，那么重新再来，因为后面需要比较当前移除的节点是否为尾节点，因为尾节点不会立马清理，会延后清理
        if (t != tail)
            continue;
        //帮助设置尾节点
        if (tn != null) {
            advanceTail(t, tn);
            continue;
        }
        //不是尾节点才进行替换，如果是尾节点的话，不能直接将其移除，因为尾节点随时都有可能被其他线程连接到新的节点
        if (s != t) {        // If not tail, try to unsplice
            QNode sn = s.next;
            //sn == s：当一个节点next是其本身的时候，那么可以收这个节点已经被消费线程移除了队列
            //pred.casNext(s, sn)：移除当前需要清理的s节点
            //那会不会cas失败呢？会，在pred被消费线程进行消费并被移除头节点的时候，那么可能会失败
            if (sn == s || pred.casNext(s, sn))
                return;
        }
        //cleanMe为某次清理未被清理的节点的前驱节点，比如可能上次清理的对象是尾节点或者其前驱节点被消费线程移除队列
        QNode dp = cleanMe;
        if (dp != null) {    // Try unlinking previous cancelled node
            //获取清理节点
            QNode d = dp.next;
            QNode dn;
            //没有清理节点，不会出现这种情况，防御性编程
            if (d == null ||               // d is gone or
                //清理节点的前驱节点被移除队列
                d == dp ||                 // d is off list or
                //清理节点未被cancel，可能在被其他线程给清理掉了，重新连接了下一个未被cancel的节点
                !d.isCancelled() ||        // d not cancelled or
                //清理的节点不能是尾节点，原因就是怕清理尾节点的时候，尾节点又被其他线程连接了新节点
                (d != t &&                 // d not tail and
                //有后继节点，那么d肯定就不是尾节点了
                 (dn = d.next) != null &&  //   has successor
                 //清理节点还没有被移除队列，被消费线程移除队列，那就不用清理了
                 dn != d &&                //   that is on list
                 //cas移除清理节点d
                 dp.casNext(d, dn)))       // d unspliced
                 //cas将清理节点字段重新赋值为null，准备用于存储下一个延迟清理的节点
                casCleanMe(dp, null);
            //如果清理节点的前驱节点就是当前清理节点s的前驱节点，那么很好，清理的节点是一样的，那么可以直接返回了
            if (dp == pred)
                return;      // s is already saved node
                //记录需要延迟清理节点的前驱节点，为什么不直接记录清理节点呢？假设我们记录的是清理节点，我们要清理掉它，那么我们先要找到前驱节点，然后将
                //前驱节点的next设置为清理节点的next，那我为啥不直接记录前驱节点呢？是吧
        } else if (casCleanMe(null, pred))
            return;          // Postpone cleaning s
    }
}
```
清理节点，也就是将节点从队列中移除或者消费的时候碰到被cancel的节点，跳过，将其移除队列。不过在清理节点的时候需要注意一些问题，如果清理节点是尾节点，那么先
不移除，延后移除，主要原始因为你在移除的过程中你不知道其他线程在什么时候在尾节点后面追加了新的节点，如果贸然移除尾节点，将导致节点丢失。

> Q：在清理节点的时候，为什么不把清理节点的next引用置空？

> A：可能导致消费线程在队列有值的情况下，因为刚好执行到这个节点，导致无法找到下一个节点，增加没有必要的再次循环，再者加个置空操作反倒增加了指令长度，增加开
销，只要这个节点是不可达的，那么它就会被gc。

### 1.2 TransferStack

stack，先进后出，下面为TransferStack的一些字段说明

```java
static final class TransferStack<E> extends Transferer<E> {
    
    //代表消费数据
    static final int REQUEST    = 0;
    
    //代表生产数据
    static final int DATA       = 1;
    
    //表示有消费者正在消费，但是还未完成，因为完成后，FULFILLING模式的节点会从栈中移除
    static final int FULFILLING = 2;
}
```

TransferStack内部类SNode字段说明

```java
static final class SNode {
    //栈节点的下一个节点
    volatile SNode next;        // next node in stack
    //如果match值和当前节点相同，那么表示被cancel的节点，如果match是一个FULFILLING节点，那么表示节点被消费
    volatile SNode match;       // the node matched to this
    //对应的生产线程
    volatile Thread waiter;     // to control park/unpark
    //生产线程生产的数据
    Object item;                // data; or null for REQUESTs
    //模式，REQUEST，DATA，FULFILLING
    int mode;
    
}
```
TransferStack由单向链表实现，TransferStack采用了头插法，将新的节点插入到头节点，然后从头节点开始往后尾部读。

#### 1.2.1 生产数据

```java
E transfer(E e, boolean timed, long nanos) {

    SNode s = null; // constructed/reused as needed
    int mode = (e == null) ? REQUEST : DATA;

    for (;;) {
        SNode h = head;
        //h == null：表示栈中还没有数据
        //h.mode == mode：节点模式一样，通常是生产线程会进入此方法，当然消费线程会在栈为空的时候进入这个分支，不过将会被返回null值
        if (h == null || h.mode == mode) {  // empty or same-mode
            //如果指定需要定时，但是却没有传递需要定时的时间，顺便利用这个线程去看看有没有需要移除被cancel的节点
            if (timed && nanos <= 0) {      // can't wait
                if (h != null && h.isCancelled())
                    casHead(h, h.next);     // pop cancelled node
                else
                    return null;
                    //使用头插法将新的节点插入到栈顶，原来的头节点将被连接到新节点的后边
            } else if (casHead(h, s = snode(s, e, h, mode))) {
                //wait，等待消费线程消费产品
                SNode m = awaitFulfill(s, timed, nanos);
                //如果节点的match字段的值是字节本身，那么节点被cancel了
                //当线程被中断或者wait超时时会将节点cancel
                if (m == s) {               // wait was cancelled
                    //清理节点，无非就是将节点从栈中移除，具体的移除方式将在后面讨论
                    clean(s);
                    return null;
                }
                //如果节点对应的线程被唤醒后发现消费线程还没有将被消费节点的下一个节点移到头部，那么帮助它完成任务
                //如果h.next是自己，那么这个h是消费线程构建出来的一个Fulfill模式的虚拟节点
                if ((h = head) != null && h.next == s)
                    casHead(h, s.next);     // help s's fulfiller
                    //返回节点对应的值，那么一个线程走到这里会出现mode == REQUEST的情况吗？如果不使用SynchronousQueue调用，直接调用这个transfer方法
                    //当栈为空，传入e参数为null，timed为false或者time为true，有定时值时，会产生新的SNode节点并加入到栈中，当它被唤醒的时候mode == REQUEST
                return (E) ((mode == REQUEST) ? m.item : s.item);
            }
        } else if (!isFulfilling(h.mode)) { // try to fulfill
           。。。。。。省略消费数据代码
        } else {                            // help a fulfiller
            。。。。。。省略消费数据代码
        }
    }
}
```

生产数据的时候，使用头插法将新的节点push到栈中，如果生产线程被中断或者wait超时，那么节点将被cancel，cancel的形式表现为将SNode的match字段设置为节点本身，被
取消的节点将从栈中移除。如果线程是被消费者线程唤醒的，那么将会判断是否需要帮助消费线程一起移动节点。

TransferStack的await线程的方法与TransferQueue是差不多的，所以不会重复分析，不同点在于取消节点的方式和获取自旋次数的条件。当头节点就是当前线程刚刚插入的节点
或者头节点为空（自己可能是最后的一个节点，然后被消费线程给消费了）或者头节点是一个fulfill模式的节点（表示有消费线程正在消费）那么需要自旋

#### 1.2.2 消费数据

```java
E transfer(E e, boolean timed, long nanos) {
    
    SNode s = null; // constructed/reused as needed
    int mode = (e == null) ? REQUEST : DATA;

    for (;;) {
        SNode h = head;
        if (h == null || h.mode == mode) {  // empty or same-mode
            。。。。。。省略生产代码
            //问，当前头节点是否是Fulfill模式的节点，如果是，说明已有消费线程正在消费，需要去帮助消费
        } else if (!isFulfilling(h.mode)) { // try to fulfill
            //移除被取消的节点
            if (h.isCancelled())            // already cancelled
                casHead(h, h.next);         // pop and retry
                //创建一个FULFILLING模式的虚拟节点
            else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                for (;;) { // loop until matched or waiters disappear
                    //m才是我们需要获取值的节点
                    SNode m = s.next;       // m is s's match
                    //如果没有值可以获取了，那么头节点置空，重新进入主循环
                    //什么时候为null呢？当设置虚拟节点后，有很多的消费线程竞争同一个业务节点，就假设三个吧，其中有一个成功获取到节点B的值，其他线程就会
                    //试着将虚拟节点的next设置为B节点的下一个节点，如果B是尾节点，那么虚拟节点可能就会引用一个空的节点
                    if (m == null) {        // all waiters are gone
                        casHead(s, null);   // pop fulfill node
                        s = null;           // use new node next time
                        break;              // restart main loop
                    }
                    SNode mn = m.next;
                    //将m的match字段用FULFILLING模式的虚拟节点s进行cas替换
                    if (m.tryMatch(s)) {
                        //取值成功，返回m的值，那么这里会出现mode 与 REQUEST不相等的情况吗？如果我们绕过SynchronousQueue来调用
                        //并且栈是空的，此时有个线程传入e=null,time=false|time=true,nanos>0，那么栈中将存入值为空的节点。
                        //当一个入参为e!=null的线程进来之后将会走到这里，那么mode 不会和 REQUEST相等
                        casHead(s, mn);     // pop both s and m
                        return (E) ((mode == REQUEST) ? m.item : s.item);
                    } else                  // lost match
                        //如果m.tryMatch(s)失败，要么节点被取消了，要么就是被其他线程给抢先消费了，帮助一起移除m节点
                        //然后继续循环获取下一个节点数据
                        s.casNext(m, mn);   // help unlink
                }
            }
        } else {                            // help a fulfiller
            //走到这，说明已经有消费线程正在消费了
            //h为FULFILLING模式的节点
            SNode m = h.next;               // m is h's match
            if (m == null)                  // waiter is gone
                casHead(h, null);           // pop fulfilling node
            else {
                SNode mn = m.next
                //帮助消费线程处理match
                if (m.tryMatch(h))          // help match
                    casHead(h, mn);         // pop both h and m
                else                        // lost match
                    //帮助消费线程一起移除m节点
                    h.casNext(m, mn);       // help unlink
            }
        }
    }
}
```

当消费线程去消费栈的数据的时候，首先需要创建一个虚拟的头节点，然后通过cas的方式将节点的match字段设置为这个虚拟节点，表示消费成功。如果消费线程发现有其他的
消费线程在消费（判断依据为头节点为FULFILLING模式的节点），那么帮助那个线程match节点，移除已被消费的节点，自己还不抢节点的值，挺有道德的。

> Q：消费的时候为什么要创建一个FULFILLING类型的虚拟节点插入到头节点呢？不能直接消费吗？

> A：我个人觉的是可以的，但是假设没有FULFILLING类型的虚拟节点，我们直接去消费业务节点，那么此时就没有帮助线程了，难以避免的去竞争同一个节点值。要实现这个逻
辑那么我们需要cas设置一个特殊的值给节点的match字段，每个消费线程去消费的时候都要判断这个节点是否已经被消费过了，避免重复消费，消费成功之后进行节点前移
操作，那么没有消费成功的线程可以帮助前移

#### 1.2.3 清理数据


```java
void clean(SNode s) {
    s.item = null;   // forget item
    s.waiter = null; // forget thread
    
    SNode past = s.next;
    //获取s的后继节点，如果后继节点被cancel，那么获取下一个节点
    //我可不可以把if改成while呢？
    if (past != null && past.isCancelled())
        past = past.next;

    // Absorb cancelled nodes at head
    SNode p;
    
    //如果头节点被cancel，那么将后继节点设置为头节点，然后继续判断是否被cancel
    while ((p = head) != null && p != past && p.isCancelled())
        casHead(p, p.next);

    // Unsplice embedded nodes
    //移除p与past之间被cancel的节点
    while (p != null && p != past) {
        SNode n = p.next;
        if (n != null && n.isCancelled())
            //移除被cancel的后继节点
            p.casNext(n, n.next);
        else
            //如果后继节点没有被移除，那么将后继节点作为新的p节点继续往后，直到p节点等于past
            p = n;
    }
}
```

上面一段清理节点的代码原理不是非常复杂，先确定两个点，然后从某个点开始向另一点靠拢，在靠近的中途将路上被cancel的节点移除。

上面有个问题就是以下代码能不能将if改为while

```java
if (past != null && past.isCancelled())
    past = past.next;
```

其实我个人觉得是可以的，但是为什么作者没有这么做呢？其实被取消的节点都会有其对应线程去清理，每个线程去处理其自身范围内（清理节点时需要从头节点开始，所以可能认为头到清理节点的next为清理范围）的节点即可，没有必要与其他的线程cas去竞争移除节点。


## 二、总结

以上我们分析了SynchronousQueue，SynchronousQueue根据公平性设置了两个Transferer，一个是基于链表实现的TransferStack，表现为不公平，先进的后出，另一个也是基于
链表实现的TransferQueue，表示为公平，先进先出。