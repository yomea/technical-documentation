## 一、什么是对象池？

顾名思义，就是有一个池子装着一群对象。我们需要对象的时候就从池子中取一个出来，用完之后就把对象放回池子中去。这样可以复用对象并且减少创建一个对象的开销，也可以
避免申请对象太多导致的频繁gc。从设计模式上来讲，它是一种享元模式。

## 二、netty的对象池

netty为了避免过多的创建对象和频繁的gc使用了对象池，在需要创建ByteBuf的时候，从对象池中找，如果没有才会去创建一个新的ByteBuf。

### 2.1 类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/85569ab74aa7c26da97832abdfb690b4.png)


下面对图中所涉及到的一些比较重要的类进行一些简要的说明

#### 2.1.1 Recycler

```java
//maxCapacityPerThread为0时，设置 NOOP_HANDLE ，表示不会回收对象
private static final Handle NOOP_HANDLE;
//用于表示线程id
private static final AtomicInteger ID_GENERATOR = new AtomicInteger(Integer.MIN_VALUE);
//谁创建的Recycler，那么就给这个线程记录一个id
private static final int OWN_THREAD_ID = ID_GENERATOR.getAndIncrement();
//默认的设置每个线程的Stack的初始化最大容量大小
private static final int DEFAULT_INITIAL_MAX_CAPACITY_PER_THREAD = 4 * 1024; // Use 4k instances as default.
//每个线程的Stack的最大容量大小，如果没有设置，那么将会将 DEFAULT_INITIAL_MAX_CAPACITY_PER_THREAD 设置给它
private static final int DEFAULT_MAX_CAPACITY_PER_THREAD;
//Stack的初始化容量
private static final int INITIAL_CAPACITY;
//最大共享容量的负载因子，用于限制 WeakOrderQueue 的容量大小
//默认是2
private static final int MAX_SHARED_CAPACITY_FACTOR;
//用于限制每个Recycler中记录中 DELAYED_RECYCLED 的大小
private static final int MAX_DELAYED_QUEUES_PER_THREAD;
//设置每个Link对象的默认容量
private static final int LINK_CAPACITY;
//默认回收率
private static final int RATIO;

//每个线程对应的Stack的容量大小
private final int maxCapacityPerThread;
//用于设置WeakOrderQueue容量的负载因子
private final int maxSharedCapacityFactor;
//回收率掩码
private final int ratioMask;
//WearOrderQueue的最大容量
private final int maxDelayedQueuesPerThread;
//用于记录Stack，是netty自己实现的一个threadLocal，如果线程对象是FastThread，那么可以直接从
//FastThread的字段中找到它
private final FastThreadLocal<Stack<T>> threadLocal

//WeakOrderQueue内部有一个 next 字段，用于指定下一个 WeakOrderQueue
//当回收数据时，操作 Stack 对象的不是 Stack 的线程，那么将会封装一个 WeakOrderQueue put 到 Map<Stack<?>, WeakOrderQueue> 中
//用于并发加锁，导致性能下降
private static final FastThreadLocal<Map<Stack<?>, WeakOrderQueue>> DELAYED_RECYCLED；

```

#### 2.1.2 Stack

io.netty.util.Recycler.Stack

```java
//当前Stack所属Recycler
final Recycler<T> parent;
//当前Stack所属线程，使用弱引用引用，如果这个线程不再使用，那么
//在gc的时候将被回收
final WeakReference<Thread> threadRef;
//用于限制Stack的WorkOrderQueue总共所能使用的大小
final AtomicInteger availableSharedCapacity;
//限制 Recycler#DELAYED_RECYCLED中 Map<Stack<?>, WeakOrderQueue> 容量的大小
final int maxDelayedQueues;
//Stack的最大容量
private final int maxCapacity;
//回收率掩码
private final int ratioMask;
//用于存储回收对象的数组，这个Stack是用过这个数组来实现的
private DefaultHandle<?>[] elements;
//回收对象个数
private int size;
//回收次数，不为零就表示已经被回收
private int handleRecycleCount = -1; // Start with -1 so the first one will be recycled.
//cursor用于指示当前进行需要transfer的WeakOrderQueue
//prev用于表示cursor的前驱节点，用于剔除被gc回收了线程的WeakOrderQueue
private WeakOrderQueue cursor, prev;
//WeakOrderQueue链表的头节点
private volatile WeakOrderQueue head;
```

#### 2.1.3 WeakOrderQueue

io.netty.util.Recycler.WeakOrderQueue

```java
//WeakOrderQueue中用于引用Link节点的头节点
//用于控制Link链表的容量
private final Head head;
//Link的tail节点
private Link tail;
// pointer to another queue of delayed items for the same stack
// WeakOrderQueue 链表的下一个节点
private WeakOrderQueue next;
//当前WeakOrderQueue所属线程
private final WeakReference<Thread> owner;
//给每个创建的 WeakOrderQueue 的一个 id
private final int id = ID_GENERATOR.getAndIncrement();
```

#### 2.1.4 Link

io.netty.util.Recycler.WeakOrderQueue.Link

```java
// LINK_CAPACITY 默认为16，elements用于存储回收对象
private final DefaultHandle<?>[] elements = new DefaultHandle[LINK_CAPACITY];

//当前可读元素起始下标
private int readIndex;
//Link链表的下一个节点
Link next;

```
#### 2.1.5 Head

io.netty.util.Recycler.WeakOrderQueue.Head

```java
//用于限制Stack的WorkOrderQueue总共所能使用的大小
private final AtomicInteger availableSharedCapacity;
//记录下一个Link节点
Link link;
```

### 2.2 回收


```java
@Deprecated
public final boolean io.netty.util.Recycler#recycle(T o, Handle<T> handle) {
    //NOOP_HANDLE 无需回收，通常是 maxDelayedQueuesPerThread 被设置为0
    if (handle == NOOP_HANDLE) {
        return false;
    }
    
    DefaultHandle<T> h = (DefaultHandle<T>) handle;
    //只回收由当前Recycler创建的Stack
    if (h.stack.parent != this) {
        return false;
    }
    //回收
    //(1)
    h.recycle(o);
    return true;
}

//(1)
public void io.netty.util.Recycler.DefaultHandle#recycle(Object object) {
    //如果回收对象和上次回收的对象不一致，抛出错误
    if (object != value) {
        throw new IllegalArgumentException("object does not belong to handle");
    }
    //获取当初回收自己的Stack
    Stack<?> stack = this.stack;
    //每次回收的时候 lastRecycledId 都会被设置为一个id（如果是直接存入Stack的，那么这个id是创建Recycler时通过AtomicInteger自增而来的
    //如果是存入WeakOrderQueue，那么就是创建WeakOrderQueue时通过AtomicInteger自增而来）
    //被使用时重置为零
    if (lastRecycledId != recycleId || stack == null) {
        throw new IllegalStateException("recycled already");
    }
    //压栈
    //(2)
    stack.push(this);
}

//(2)
void io.netty.util.Recycler.Stack#push(DefaultHandle<?> item) {
    Thread currentThread = Thread.currentThread();
    //检查当前Stack所属线程是否为当前线程，如果是当前线程，那么很简单，直接往Stack中的数组中添加即可
    //如果不是当前线程，那么为了避免多线程带来的加锁操作，这里会创建专属于当前线程的 WeakOrderQueue
    if (threadRef.get() == currentThread) {
        // The current Thread is the thread that belongs to the Stack, we can try to push the object now.
        pushNow(item);
    } else {
        // The current Thread is not the one that belongs to the Stack
        // (or the Thread that belonged to the Stack was collected already), we need to signal that the push
        // happens later.
        pushLater(item, currentThread);
    }
}
```
以上方法首先判断是否需要回收对象，如果需要回收对象，那么会判断回收这个对象的线程是否就是当前Stack所属线程，如果是，那么直接push的栈中即可，如果不是，那么为
了避免多线程带来的同步问题，这里创建了专属于某个线程的 WeakOrderQueue

push到Stack数组中的代码比较简单，就是往数组后面添加元素即可，当然如果容量不够，那么会扩容2倍，如果超过最大容量，那么取最大容量

下面我们看到pushLater方法

```java

private void io.netty.util.Recycler.Stack#pushLater(DefaultHandle<?> item, Thread thread) {
    //从每个线程上获取其Map<Stack<?>, WeakOrderQueue>，key表示当初当前线程回收对象所属的Stack，
    //value表示当前线程回收对象时为当前线程专属创建的WeakOrderQueue。
    Map<Stack<?>, WeakOrderQueue> delayedRecycled = DELAYED_RECYCLED.get();
    WeakOrderQueue queue = delayedRecycled.get(this);
    if (queue == null) {
        //如果超过了最大容量，那么将不再回收对象，这里直接put一个空队列标记
        if (delayedRecycled.size() >= maxDelayedQueues) {
            // Add a dummy queue so we know we should drop the object
            delayedRecycled.put(this, WeakOrderQueue.DUMMY);
            return;
        }
        // Check if we already reached the maximum number of delayed queues and if we can allocate at all.
        //cas stack.availableSharedCapacity 的值，如果还能分配容量，那么创建WeakOrderQueue
        //容量不足，直接返回，如果容量充足将新创建的WeakOrderQueue设置到Stack的头节点（这里使用了同步的方式去设置头节点）
        if ((queue = WeakOrderQueue.allocate(this, thread)) == null) {
            // drop object
            return;
        }
        delayedRecycled.put(this, queue);
    } else if (queue == WeakOrderQueue.DUMMY) {
        // drop object
        return;
    }
    //入队列
    //(1)
    queue.add(item);
}

//(1)
void io.netty.util.Recycler.WeakOrderQueue#add(DefaultHandle<?> handle) {
    //设置回收id，用于检查是否被回收
    handle.lastRecycledId = id;
    Link tail = this.tail;
    int writeIndex;
    //检查尾部节点可写索引，如果已经达到最大容量，那么检查是否还有剩余容量可分配
    //如果没有那么抛弃
    if ((writeIndex = tail.get()) == LINK_CAPACITY) {
        if (!head.reserveSpace(LINK_CAPACITY)) {
            // Drop it.
            return;
        }
        // We allocate a Link so reserve the space
        //还有足够容量进行分配，那么创建一个新的Link
        this.tail = tail = tail.next = new Link();

        writeIndex = tail.get();
    }
    tail.elements[writeIndex] = handle;
    //将stack置空，表示当前对象已被回收
    //并且并不是由Stack所属线程所回收
    handle.stack = null;
    // we lazy set to ensure that setting stack to null appears before we unnull it in the owning thread;
    // this also means we guarantee visibility of an element in the queue if we see the index updated
    //更新到下一个索引，这个方法名虽然叫 lazySet，但是我没看出来怎么个lazy法了
    //内部直接使用Unsafe设置值
    tail.lazySet(writeIndex + 1);
}
```

为了减少并发带来的锁消耗，netty通过创建WeakOrderQueue来优化性能，首先每个线程最多可以维护默认CPU个数的2倍个数的Stack，再者Stack需要有足够默认16的容量用于创
建Link，如果当前线程还未为Stack创建过WeakOrderQueue，那么会进行创建，并将创建的WeakOrderQueue同步插入到Stack的头节点。

由此可画出其结构图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/95e78a0ae95f50b616228085692868ed.png)


### 2.3 复用

#### 2.3.1 FastThreadLocal

FastThreadLocal从名字上看就觉得不明觉厉，那它到底有多fast呢？我们接下来分析下它

```java
//对应InternalThreadLocalMap中数组的下标，用于快速通过这个index找到对应的数据
private final int index;

//构造器
public FastThreadLocal() {
    //内部使用一个 AtomicInteger 递增
    index = InternalThreadLocalMap.nextVariableIndex();
}

//get方法
public final V get() {
    //根据当前线程对象获取InternalThreadLocalMap
    InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
    //通过构建FastThreadLocal时设置下标获取数据
    Object v = threadLocalMap.indexedVariable(index);
    if (v != InternalThreadLocalMap.UNSET) {
        return (V) v;
    }
    //初始化
    return initialize(threadLocalMap);
}
```

可以看到在创建FastThreadLocal对象时通过一个原子递增类获取一个下标，这个下标用于快速从InternalThreadLocalMap中定位数据（这个 InternalThreadLocalMap 内部是
由一个数组实现的，容量为2的n次幂的值）。


#### 2.3.2 从Stack中获取复用对象

```java
public final T io.netty.util.Recycler#get() {
    //如果禁止了对象池的使用，那么返回一个 NOOP_HANDLE（内部回收逻辑为空）
    if (maxCapacityPerThread == 0) {
        return newObject((Handle<T>) NOOP_HANDLE);
    }
    //这个threadLocal是一个FastThreadLocal，如果当前线程是一个FastThreadLocalThread对象
    //那么会直接从FastThreadLocalThread的一个InternalThreadLocalMap类型的字段中获取
    //如果当前线程不是FastThreadLocalThread类型的，那么将会从当前线程的ThreadLocal中获取InternalThreadLocalMap对象
    //由于创建一个FastThreadLocal对象时，会通过原子递增类获取一个下标，所以可以直接通过下标从InternalThreadLocalMap获取Stack对象
    Stack<T> stack = threadLocal.get();
    //从Stack中弹出一个对象
    DefaultHandle<T> handle = stack.pop();
    //如果为空，创建一个新的对象
    if (handle == null) {
        handle = stack.newHandle();
        handle.value = newObject(handle);
    }
    return (T) handle.value;
}
```

上面这个方法首先判断是否允许回收对象，如果允许，那么会从Stack中pop对象，下面是Stack的pop方法

```java
DefaultHandle<T> io.netty.util.Recycler.Stack#pop() {
    int size = this.size;
    //如果当前栈中没有值，那么尝试从WeakOrderQueue中查找
    if (size == 0) {
        //(1)
        if (!scavenge()) {
            return null;
        }
        size = this.size;
    }
    //弹栈
    size --;
    DefaultHandle ret = elements[size];
    elements[size] = null;
    //不允许多次回收，每个对象只能被一个线程回收一次
    if (ret.lastRecycledId != ret.recycleId) {
        throw new IllegalStateException("recycled multiple times");
    }
    //复位，表示未被回收
    ret.recycleId = 0;
    ret.lastRecycledId = 0;
    this.size = size;
    return ret;
}

//(1)
boolean scavenge() {
    // continue an existing scavenge, if any
    //扫描WeakOrderQueue
    if (scavengeSome()) {
        return true;
    }

    // reset our scavenge cursor
    //没有扫描到，复位
    prev = null;
    cursor = head;
    return false;
}
```

上面的逻辑比较简单，首先判断当前Stack中是否有回收对象，如果没有，那么将从WeakOrderQueue中寻找，如果有，那么直接从当前Stack中弹出一个即可


```java
boolean io.netty.util.Recycler.Stack#scavengeSome() {
    //用于记录当前扫描的节点的前一节点，用于剔除被gc回收掉的节点（prev.setNext(回收节点的next)）
    WeakOrderQueue prev;
    //正被扫描的节点
    WeakOrderQueue cursor = this.cursor;
    if (cursor == null) {
        prev = null;
        //头节点开始
        cursor = head;
        if (cursor == null) {
            return false;
        }
    } else {
        prev = this.prev;
    }

    boolean success = false;
    do {
        //尝试将当前扫描到的节点的回收对象转移到Stack的数组中
        if (cursor.transfer(this)) {
            success = true;
            break;
        }
        WeakOrderQueue next = cursor.next;
        //当前节点线程被gc回收后，cursor.owner.get()将返回null
        if (cursor.owner.get() == null) {
            // If the thread associated with the queue is gone, unlink it, after
            // performing a volatile read to confirm there is no data left to collect.
            // We never unlink the first queue, as we don't want to synchronize on updating the head.
            //检查这个节点是否还有回收对象
            if (cursor.hasFinalData()) {
                for (;;) {
                    //将当前WeakOrderQueue的数组转移到Stack中
                    if (cursor.transfer(this)) {
                        success = true;
                    } else {
                        break;
                    }
                }
            }

            if (prev != null) {
                //剔除当前被回收掉的节点
                prev.setNext(next);
            }
        } else {
            prev = cursor;
        }

        cursor = next;

    } while (cursor != null && !success);

    this.prev = prev;
    this.cursor = cursor;
    return success;
}
```
从Stack的头节点开始，寻找能够转移数据到Stack的WeakOrderQueue，对于那些被gc回收掉线程的WeakOrderQueue，首先会将他们回收的对象全部转移到Stack中，然后再将它们
从WeakOrderQueue链表中剔除


```java
boolean io.netty.util.Recycler.WeakOrderQueue#transfer(Stack<?> dst) {
    Link head = this.head.link;
    //当前WeakOrderQueue没有回收对象了
    if (head == null) {
        return false;
    }
    //检查读下标是否已经达到最大索引
    if (head.readIndex == LINK_CAPACITY) {
        if (head.next == null) {
            return false;
        }
        //移除当前使用完的节点
        this.head.link = head = head.next;
        //释放容量
        this.head.reclaimSpace(LINK_CAPACITY);
    }
    //获取当前遍历到的节点的可读开始索引
    final int srcStart = head.readIndex;
    //当前节点的end下标
    int srcEnd = head.get();
    //当前Link节点可用对象个数
    final int srcSize = srcEnd - srcStart;
    //为零表示没有回收对象可用
    if (srcSize == 0) {
        return false;
    }
    //计算当前Stack将要达到的容量
    final int dstSize = dst.size;
    final int expectedCapacity = dstSize + srcSize;
    //如果超过了Stack数组长度，那么需要进行扩容
    if (expectedCapacity > dst.elements.length) {
        final int actualCapacity = dst.increaseCapacity(expectedCapacity);
        //由于Stack有最大容量限制，默认是4096，所以这里需要进行判断
        srcEnd = min(srcStart + actualCapacity - dstSize, srcEnd);
    }

    //将Link中的元素迁移到Stack中
    if (srcStart != srcEnd) {
        final DefaultHandle[] srcElems = head.elements;
        final DefaultHandle[] dstElems = dst.elements;
        int newDstSize = dstSize;
        for (int i = srcStart; i < srcEnd; i++) {
            DefaultHandle element = srcElems[i];
            if (element.recycleId == 0) {
                element.recycleId = element.lastRecycledId;
            } else if (element.recycleId != element.lastRecycledId) {
                throw new IllegalStateException("recycled already");
            }
            srcElems[i] = null;

            if (dst.dropHandle(element)) {
                // Drop the object.
                continue;
            }
            element.stack = dst;
            dstElems[newDstSize ++] = element;
        }
        //srcEnd == LINK_CAPACITY表示当前遍历到的Link已经被充分使用了，在此次迁移
        //数据之后将被释放容量
        if (srcEnd == LINK_CAPACITY && head.next != null) {
            // Add capacity back as the Link is GCed.
            this.head.reclaimSpace(LINK_CAPACITY);
            this.head.link = head.next;
        }

        head.readIndex = srcEnd;
        if (dst.size == newDstSize) {
            return false;
        }
        dst.size = newDstSize;
        return true;
    } else {
        // The destination stack is full already.
        return false;
    }
}
```
上面这个方法将遍历WeakOrderQueue中的Link，将他们回收的对象迁移到Stack中，如果发现被使用完的Link，那么将从链表中剔除并回收容量。


## 三、总结

获取过程：

使用 FastThreadLocal#get方法从线程对象（FastThreadLocalThread或者普通的Thread对象，FastThreadLocalThread对象是直接通过一个属性引用InternalThreadLocalMap，
而Thread就得从ThreadLocal中查找，性能较差）中获取 InternalThreadLocalMap（通过一个Object[] indexedVariables数组记录数据，下标是创建FastThreadLocal时在
构造器中调用原子递增类得到的），然后从InternalThreadLocalMap获取Stack对象，从Stack中pop一个DefaultHandler对象，如果Stack中有值，那么直接从栈中弹出一个元素
，如果Stack中没有值，那么从WeakOrderQueue获取, 遍历WeakOrderQueue的Link，将Link中的元素迁移到Stack，如果发现用完了的Link，那么断开链接，在Head中记录释放了
多少多少空间（Link的容量默认是16），如果在遍历WeakOrderQueue的过程中，发现其对应的线程已经被垃圾回收了，那么将这个WeakOrderQueue中的所有元素移到Stack中。

回收过程：

如果当前DefaultHandler对应的Stack的线程对象就是当前线程对象，那么直接存到Stack的数组中即可，如果不是，创建一个WeakOrderQueue，存放到Recycler对象的

FastThreadLocal<Map<Stack<?>, WeakOrderQueue>> DELAYED_RECYCLED属性中，这个属性的容量默认是CPU个数乘以2。调用WeakOrderQueue的add方法，检查最后一个Link容量
是否已满，如果没有满，将值存入，如果满了，检查是否超过Stack允许的WeakOrderQueue的容量（默认是2048），没有超过就创建一个新的Link存入。

这样可以有效提高并发量，除了构建WeakOrderQueue需要线程安全之外，其他的操作就不需要锁啦。

类图：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/711267985903e107e709e0b5e524e71c.png)


结构图：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/c7552e1daa18d18840d27edeadc5bc9e.png)






