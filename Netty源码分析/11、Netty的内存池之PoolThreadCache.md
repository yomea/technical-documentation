## 一、PoolThreadLocalCache

在申请池化内存时，netty并不会直接从内存池中申请，而是先从PoolThreadLocalCache获取，同样的release一个ByteBuf时并不会直接归还到内存池，而是先缓存在
PoolThreadLocalCache中

PoolThreadLocalCache继承了FastThreadLocal，FastThreadLocal类似JDK的ThreadLocal的作用，将对象和本地线程关联，它内部使用了一个叫
InternalThreadLocalMap的集合，这个集合内部使用数组实现，在初始化FastThreadLocal时会通过一个原子递增类获取一个下标，数据就存储在指定的下标上，这点和
ThreadLocal有比较大的区别，ThreadLocal是以ThreadLocal对象做key的，当产生hash冲突时需要往后顺延找坑位。所以要说FastThreadLocal与ThreadLocal最大区别是什么，
那就是存值的方式不同。

除此之外如果当前线程对象是FastThreadLocalThread类型，那么这个InternalThreadLocalMap直接是FastThreadLocalThread一个字段，就无需使用ThreadLocal去本地线程中获
取InternalThreadLocalMap集合了。

下面我们来看下PoolThreadLocalCache初始值方法

```java
protected synchronized PoolThreadCache io.netty.buffer.PooledByteBufAllocator.PoolThreadLocalCache#initialValue() {
    //遍历heapArenas，找出使用缓存的线程数最少的那个PoolArena
    final PoolArena<byte[]> heapArena = leastUsedArena(heapArenas);
    final PoolArena<ByteBuffer> directArena = leastUsedArena(directArenas);

    final Thread current = Thread.currentThread();
    //useCacheForAllThreads表示是否启用线程缓存
    //FastThreadLocalThread是一个继承了Thread的线程对象，它有一个成员字段叫InternalThreadLocalMap
    //InternalThreadLocalMap类似于JDK的Thread中的ThreadLocalMap
    if (useCacheForAllThreads || current instanceof FastThreadLocalThread) {
        final PoolThreadCache cache = new PoolThreadCache(
                heapArena, directArena, tinyCacheSize, smallCacheSize, normalCacheSize,
                DEFAULT_MAX_CACHED_BUFFER_CAPACITY, DEFAULT_CACHE_TRIM_INTERVAL);
        //如果指定了定时清理缓存的时间，那么启动一个调度线程池去处理
        if (DEFAULT_CACHE_TRIM_INTERVAL_MILLIS > 0) {
            final EventExecutor executor = ThreadExecutorMap.currentExecutor();
            if (executor != null) {
                executor.scheduleAtFixedRate(trimTask, DEFAULT_CACHE_TRIM_INTERVAL_MILLIS,
                        DEFAULT_CACHE_TRIM_INTERVAL_MILLIS, TimeUnit.MILLISECONDS);
            }
        }
        return cache;
    }
    // No caching so just use 0 as sizes.
    //不允许使用缓存，创建一个空的PoolThreadCache
    return new PoolThreadCache(heapArena, directArena, 0, 0, 0, 0, 0);
}
```

从上面的代码中可以看到，PoolThreadLocalCache内部缓存的是一个叫做PoolThreadCache的对象，这个对象才是最终记录释放的内存块的地方

## 二、PoolThreadCache

### 2.1 PoolThreadCache字段说明

```java
//当前 PoolThreadCache 缓存的内存所属 PoolArena
final PoolArena<byte[]> heapArena;
final PoolArena<ByteBuffer> directArena;

//用于缓存tiny类型的堆内存，数组长度为 32
//MemoryRegionCache内部维护了一个队列，长度默认为512
private final MemoryRegionCache<byte[]>[] tinySubPageHeapCaches;

//用于缓存small类型的堆内存，数组长度为 4
//MemoryRegionCache内部队列长度默认为256
private final MemoryRegionCache<byte[]>[] smallSubPageHeapCaches;

//用于缓存tiny类型的直接内存，数组长度为 32
//MemoryRegionCache内部维护了一个队列，长度默认为512
private final MemoryRegionCache<ByteBuffer>[] tinySubPageDirectCaches;

//用于缓存small类型的直接内存，数组长度为 4
//MemoryRegionCache内部队列长度默认为256
private final MemoryRegionCache<ByteBuffer>[] smallSubPageDirectCaches;

//用于缓存normal类型的堆内存，数组长度默认为3
//MemoryRegionCache内部队列长度默认为64
private final MemoryRegionCache<byte[]>[] normalHeapCaches;

//用于缓存normal类型的直接内存，数组长度默认为3
//MemoryRegionCache内部队列长度默认为64
private final MemoryRegionCache<ByteBuffer>[] normalDirectCaches;

// Used for bitshifting when calculate the index of normal caches later
//计算直接内存的页偏移，默认是13 = log2(8k)
private final int numShiftsNormalDirect;
//计算堆内存的页偏移，默认是13 = log2(8k)
private final int numShiftsNormalHeap;
//缓存的内存被申请了 freeSweepAllocationThreshold 次以后将被清理，归还到内存池中
private final int freeSweepAllocationThreshold;
//原子类，用于确保io.netty.buffer.PoolThreadCache#free(boolean)方法只被调用一次
//free方法用于将PoolThreadCache缓存的内存都归还到内存池中
private final AtomicBoolean freed = new AtomicBoolean();
//记录从PoolThreadCache分配内存的次数
private int allocations;

```
### 2.2 PoolThreadCache.MemoryRegionCache

#### 2.2.1 简介

MemoryRegionCache是具体记录内存块信息的地方，按照内存大小分成两个实现类，一个是SubPageMemoryRegionCache，另一个是NormalMemoryRegionCache，其主要原因是因为
初始化PooledByteBuf时，计算偏移的方式不一样，具体的初始化方式可翻看博文[《netty内存池源码分析》](https://blog.csdn.net/qq_27785239/article/details/105893805)


#### 2.2.2 字段

```java
//记录queue的元素个数
private final int size;
//缓存内存块
private final Queue<Entry<T>> queue;
//枚举，用于记录当前内存块所属类型，tiny，small or normal
private final SizeClass sizeClass;
//记录分配次数，每次从MemoryRegionCache分配内存，都会加1
private int allocations;
```
#### 2.2.3 添加缓存对象

```java
public final boolean MemoryRegionCache#add(PoolChunk<T> chunk, ByteBuffer nioBuffer, long handle) {
    //(1)
    Entry<T> entry = newEntry(chunk, nioBuffer, handle);
    //存入队列
    boolean queued = queue.offer(entry);
    if (!queued) {
        // If it was not possible to cache the chunk, immediately recycle the entry
        //如果已经达到队列容量上限，那么就存不进去了，回收这个entry，供下次使用
        entry.recycle();
    }

    return queued;
}

//(1)
private static Entry io.netty.buffer.PoolThreadCache.MemoryRegionCache#newEntry(PoolChunk<?> chunk, ByteBuffer nioBuffer, long handle) {
    //从对象池中获取一个Entry
    Entry entry = RECYCLER.get();
    //记录内存块相关信息
    //PoolChunk，表示当前缓存的内存块来自哪个内存池
    entry.chunk = chunk;
    //缓存当前内存块使用的ByteBuffer对象，用于ByteBuf转ByteBuffer时使用
    entry.nioBuffer = nioBuffer;
    //记录当前内存块在PoolChunk二叉树中的索引，如果是小于8k的，另外还会记录其在bitMap中的bit位索引
    entry.handle = handle;
    return entry;
}
```
可以看到，netty添加缓存的时候，将内存块的信息封装成了一个Entry对象存入到一个队列中，而为了避免创建对象的开销，netty使用了对象池去复用Entry对象。关于对象池
相关的内容可以查看博文[《对象池》](https://blog.csdn.net/qq_27785239/article/details/105827771)


#### 2.2.4 分配内存

```java
public final boolean io.netty.buffer.PoolThreadCache.MemoryRegionCache#allocate(PooledByteBuf<T> buf, int reqCapacity) {
    //从队列中取一个Entry
    Entry<T> entry = queue.poll();
    //entry为null，缓存分配失败
    if (entry == null) {
        return false;
    }
    //成功，初始化PooledByteBuf
    initBuf(entry.chunk, entry.nioBuffer, entry.handle, buf, reqCapacity);
    entry.recycle();

    // allocations is not thread-safe which is fine as this is only called from the same thread all time.
    //分配次数加1
    ++ allocations;
    return true;
}
```
#### 2.2.5 内存的释放

内存的释放很简单，只要调用其持有的PoolChunk的free方法即可

### 2.3 PoolThreadCache的构造器

```java
PoolThreadCache(PoolArena<byte[]> heapArena, PoolArena<ByteBuffer> directArena,
                    int tinyCacheSize, int smallCacheSize, int normalCacheSize,
                    int maxCachedBufferCapacity, int freeSweepAllocationThreshold) {
    //检查maxCachedBufferCapacity参数，maxCachedBufferCapacity > 0
    checkPositiveOrZero(maxCachedBufferCapacity, "maxCachedBufferCapacity");
    this.freeSweepAllocationThreshold = freeSweepAllocationThreshold;
    this.heapArena = heapArena;
    this.directArena = directArena;
    if (directArena != null) {
        //创建用于缓存tiny类型内存块的MemoryRegionCache数组，数组长度为32
        //MemoryRegionCache内部队列长度默认为512
        tinySubPageDirectCaches = createSubPageCaches(
                tinyCacheSize, PoolArena.numTinySubpagePools, SizeClass.Tiny);
        //创建用于缓存small类型内存块的MemoryRegionCache数组，数组长度为4
        //MemoryRegionCache内部队列长度默认为256
        smallSubPageDirectCaches = createSubPageCaches(
                smallCacheSize, directArena.numSmallSubpagePools, SizeClass.Small);
        //页偏移，pageSize默认为8k，所以偏移值为13
        numShiftsNormalDirect = log2(directArena.pageSize);
        //创建用于缓存normal类型内存块的MemoryRegionCache数组，数组长度默认为3
        //MemoryRegionCache内部队列长度默认为64
        normalDirectCaches = createNormalCaches(
                normalCacheSize, maxCachedBufferCapacity, directArena);
        //递增，表示当前PoolArena所管理的内存池已被多少线程缓存
        directArena.numThreadCaches.getAndIncrement();
    }
    
    。。。。。。省略创建堆PoolArean的代码，和创建直接内存的PoolArean是一样的
}
```
从构造器中可以看到，PoolThreadCache按照分配内存的大小分成了三种类型的缓存，tiny，small，normal
- tiny数组长度的为32，分别表示0,16,32...480,496，相邻元素直接相差16字节

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/a9887d2eb99932bc8a95e814e2402783.png)


- small类型的数组长度为4，分别表示512,1024,2048,4096，相邻元素之间相差2倍

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/1c6422b278bbe447be359afa7ae4b67c.png)


- normal数组的长度默认是3，相邻元素之间的能够储存的最小值相差2被，最大值也是相差2倍，每个元素的范围相差两倍，能够存储的值不包含最大值，由于Netty会将内存大小修正为2的n次幂的值，所以其实下面的图也就只能存 8k, 16k, 32k这样的内存值

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/901944136cfc98807f1cb9ea05f48ac7.png)

### 2.4 添加缓存数据

```java
boolean io.netty.buffer.PoolThreadCache#add(PoolArena<?> area, PoolChunk chunk, ByteBuffer nioBuffer,
                long handle, int normCapacity, SizeClass sizeClass) {
    //(1)
    MemoryRegionCache<?> cache = cache(area, normCapacity, sizeClass);
    if (cache == null) {
        return false;
    }
    //添加到MemoryRegionCache的队列中
    return cache.add(chunk, nioBuffer, handle);
}

//(1)
private MemoryRegionCache<?> cache(PoolArena<?> area, int normCapacity, SizeClass sizeClass) {
    switch (sizeClass) {
    case Normal:
        return cacheForNormal(area, normCapacity);
    case Small:
        return cacheForSmall(area, normCapacity);
    case Tiny:
        return cacheForTiny(area, normCapacity);
    default:
        throw new Error();
    }
}
```

上面的逻辑不复杂，通过SizeClass判断当前是属于什么类型的内存缓存，然后找到对应类型的数组，计算下标，获取MemoryRegionCache，将值存入即可

- tiny类型的下标计算

```java
static int tinyIdx(int normCapacity) {
    //除以16即可
    return normCapacity >>> 4;
}
```
- small类型的下标计算

```java
static int smallIdx(int normCapacity) {
    int tableIdx = 0;
    //除以512，得到2的n次方的商
    int i = normCapacity >>> 10;
    //相当于log2(i)
    while (i != 0) {
        i >>>= 1;
        tableIdx ++;
    }
    return tableIdx;
}
```
- normal类型的下标计算

```java
//numShiftsNormalDirect为13，normCapacity >> numShiftsNormalDirect相当于除以pageSize
int idx = log2(normCapacity >> numShiftsNormalDirect);
```

### 2.5 分配缓存数据

看了前面缓存数据的存储和MemoryRegionCache的内存分配，不用看代码都知道，首先确定是哪种类型的内存分配，然后计算下标，获取对应MemoryRegionCache，从其队列中
poll一个Entry，最后对PooledByteBuf初始化即可。

## 三、总结

Netty在释放或者分配内存池内存时，为了性能考虑，并不会直接从内存池中进行分配与释放，而是先从本地线程中获取缓存或者添加缓存。记录缓存的类PoolThreadCache按照
内存的大小，分成了tiny，small，normal三块缓存域，每个域都是使用数组加队列实现的，tiny数组长度默认32，相邻元素之间相差16，每个元素中的队列大小默认为512；
small数组长度默认为4，相邻元素之间相差两倍，每个元素的队列大小默认为256；normal数组的长度默认为3，相邻元素之间能够缓存的最小内存大小之间相差2倍，每个元素
能够储存的范围为 (minMemory ~ 2minMemory（不含最大值，由于Netty会将内存大小修正为2的n次幂的值，所以其实也就只能存 8k, 16k, 32k这样的）)，队列大小默认为64。