## 一、PooledByteBufAllocator的创建

PooledByteBufAllocator顾名思义，就是一个用于分配PooledByteBuf的分配器

现在有以下例子程序

```java
public class Test {

    public static void main(String[] args) {

        PooledByteBufAllocator allocator = PooledByteBufAllocator.DEFAULT;

        ByteBuf buf = allocator.buffer(1024);

        buf.writeInt(32);

        System.out.println(buf.readInt());

        buf.release();

    }

}
```

上面一段代码主要操作就是向池化的ByteBuf分配器申请1024字节的ByteBuf

### 1.1 静态初始化

PooledByteBufAllocator的静态块代码

```java
static {
    //获取默认页大小，大小为8k
    int defaultPageSize = SystemPropertyUtil.getInt("io.netty.allocator.pageSize", 8192);
    Throwable pageSizeFallbackCause = null;
    try {
        //校验和计算页偏移，pageSize的大小不能小于4096，通过默认的pageSize计算后的页偏移是13，计算方式就是通过pageSize循环除以2，直到为1
        validateAndCalculatePageShifts(defaultPageSize);
    } catch (Throwable t) {
        pageSizeFallbackCause = t;
        defaultPageSize = 8192;
    }
    DEFAULT_PAGE_SIZE = defaultPageSize;
    //默认PageChunk的最大深度为11
    int defaultMaxOrder = SystemPropertyUtil.getInt("io.netty.allocator.maxOrder", 11);
    Throwable maxOrderFallbackCause = null;
    try {
        //计算PageChunk的大小，因为每个pageSize是8k，PageChunk是一颗平衡二叉树，那么其大小就是8192 * 2^11
        validateAndCalculateChunkSize(DEFAULT_PAGE_SIZE, defaultMaxOrder);
    } catch (Throwable t) {
        maxOrderFallbackCause = t;
        defaultMaxOrder = 11;
    }
    DEFAULT_MAX_ORDER = defaultMaxOrder;

    // Determine reasonable default for nHeapArena and nDirectArena.
    // Assuming each arena has 3 chunks, the pool should not consume more than 50% of max memory.
    final Runtime runtime = Runtime.getRuntime();
    final int defaultChunkSize = DEFAULT_PAGE_SIZE << DEFAULT_MAX_ORDER;
    //决定堆内存的PageArena数，通常是cpu个数
    DEFAULT_NUM_HEAP_ARENA = Math.max(0,
            SystemPropertyUtil.getInt(
                    "io.netty.allocator.numHeapArenas",
                    (int) Math.min(
                            runtime.availableProcessors(),
                            Runtime.getRuntime().maxMemory() / defaultChunkSize / 2 / 3)));
    //决定直接内存的PageArena数，通常是cpu个数
    DEFAULT_NUM_DIRECT_ARENA = Math.max(0,
            SystemPropertyUtil.getInt(
                    "io.netty.allocator.numDirectArenas",
                    (int) Math.min(
                            runtime.availableProcessors(),
                            PlatformDependent.maxDirectMemory() / defaultChunkSize / 2 / 3)));
    //默认tiny类型的缓存大小为512
    DEFAULT_TINY_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.tinyCacheSize", 512);
    //默认的small类型的缓存大小为256
    DEFAULT_SMALL_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.smallCacheSize", 256);
    //默认的normal类型的缓存大小为64
    DEFAULT_NORMAL_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.normalCacheSize", 64);

    // 32 kb，用于限制normal类型的缓存数组长度
    DEFAULT_MAX_CACHED_BUFFER_CAPACITY = SystemPropertyUtil.getInt(
            "io.netty.allocator.maxCachedBufferCapacity", 32 * 1024);

    // 指定缓存的ByteBuf在分配次数超过 DEFAULT_CACHE_TRIM_INTERVAL 时释放到内存池中
    DEFAULT_CACHE_TRIM_INTERVAL = SystemPropertyUtil.getInt(
            "io.netty.allocator.cacheTrimInterval", 8192);
    
    //定时释放
    DEFAULT_CACHE_TRIM_INTERVAL_MILLIS = SystemPropertyUtil.getLong(
            "io.netty.allocation.cacheTrimIntervalMillis", 0);
    
    //是否对所有线程进行对象缓存，如果为false，那么每次释放内存时都会直接归还到内存池中
    //为true那么就不会直接归还到内存池，会使用对象池进行缓存
    DEFAULT_USE_CACHE_FOR_ALL_THREADS = SystemPropertyUtil.getBoolean(
            "io.netty.allocator.useCacheForAllThreads", true);
    //设置直接内存的校准值，必须是2的n次幂的值
    DEFAULT_DIRECT_MEMORY_CACHE_ALIGNMENT = SystemPropertyUtil.getInt(
            "io.netty.allocator.directMemoryCacheAlignment", 0);

    // 指定PoolChunk缓存ByteBuffer对象的最大数量
    DEFAULT_MAX_CACHED_BYTEBUFFERS_PER_CHUNK = SystemPropertyUtil.getInt(
            "io.netty.allocator.maxCachedByteBuffersPerChunk", 1023);

   。。。。。。省略日志代码
}
```

上面的静态块设置了一些必要的参数，比如pageSize，页偏移，缓存数等等。

### 1.2 实例化构造器

```java
public PooledByteBufAllocator(boolean preferDirect, int nHeapArena, int nDirectArena, int pageSize, int maxOrder,
                                  int tinyCacheSize, int smallCacheSize, int normalCacheSize,
                                  boolean useCacheForAllThreads, int directMemoryCacheAlignment) {
    //用于指定是否需要使用直接内存
    super(preferDirect);
    //创建 PoolThreadLocalCache ，这个类继承了FastThreadLocal，useCacheForAllThreads表示是否允许线程使用对象池
    threadCache = new PoolThreadLocalCache(useCacheForAllThreads);
    //tiny类型缓存的大小
    this.tinyCacheSize = tinyCacheSize;
    //small类型缓存的大小
    this.smallCacheSize = smallCacheSize;
    //normal类型缓存的大小
    this.normalCacheSize = normalCacheSize;
    //校验并返回一个PoolChunk的大小，默认是16M
    chunkSize = validateAndCalculateChunkSize(pageSize, maxOrder);

    。。。。。。省略部分代码
    //计算页偏移，每个pageSize默认是8k，所以页偏移为log2(8k) = 13
    int pageShifts = validateAndCalculatePageShifts(pageSize);
    
    //初始化创建PoolArean，有两种实现，一种是DirectArena（直接内存管理），另一种是HeapArena（堆内存管理）
    //它们在创建PoolArean时，内部的memory一个使用的是ByteBuffer，另一个用的是byte数组
    。。。。。。省略创建PoolArean的代码
}
```

这个构造器创建了一个PoolThreadLocalCache对象，PoolThreadLocalCache用于缓存内存块，关于PoolThreadLocalCache更多的细节请移步到[《Netty的内存池之PoolThreadCache》](https://blog.csdn.net/qq_27785239/article/details/105904601)


## 二、创建ByteBuf

```java
protected ByteBuf io.netty.buffer.PooledByteBufAllocator#newDirectBuffer(int initialCapacity, int maxCapacity) {
    //从当前线程中获取PoolThreadCache
    PoolThreadCache cache = threadCache.get();
    PoolArena<ByteBuffer> directArena = cache.directArena;

    final ByteBuf buf;
    if (directArena != null) {
        //分配内存
        buf = directArena.allocate(cache, initialCapacity, maxCapacity);
    } else {
        //如果Unsafe不可用，那么直接new一个UnpooledDirectByteBuf对象
        //如果可用，通过Unsafe创建对象
        buf = PlatformDependent.hasUnsafe() ?
                UnsafeByteBufUtil.newUnsafeDirectByteBuf(this, initialCapacity, maxCapacity) :
                new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
    }
    //包装buf，用于检测内存泄露
    return toLeakAwareBuffer(buf);
}
```

首先从本地线程中获取PoolThreadCache，PoolArean将会先尝试从PoolThreadCache获取内存块，如果获取不到才会尝试从内存池中分配内存。分配好内存之后，会对
PooledByteBuf进行装饰，用于检测内存泄露。

## 三、总结

PooledByteBufAllocator通过系统属性获取一些创建内存池时必要的参数，所以我们可以通过系统启动参数去定制我们的内存池，除此之外我们可以指定内存泄露检测级别用于
监控我们使用ByteBuf之后是否有调用release方法进行释放。