## 一、简介

PoolArena从功能上来讲综合了PoolThreadCache与PoolChunk，就像一个门面一样。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4d10fb1964b134f8fd3664f29187fc34.png)


PoolArenaMetric

```java
//当前PoolArean所管理的内存池已被多少个线程缓存
int numThreadCaches();

//返回tiny类型的数组的长度，默认就是32
int numTinySubpages();

//返回small类型数组的长度，默认是4
int numSmallSubpages();

//返回PoolChunkList的个数，默认6
int numChunkLists();

//返回tiny类型数组中除了头节点的所有PoolSubPage对象
List<PoolSubpageMetric> tinySubpages();

//返回small类型数组中除了头节点的所有PoolSubPage对象
List<PoolSubpageMetric> smallSubpages();

//返回所有使用占比的PoolChunkList，具体是做什么的，下面字段说明的时候会看到
List<PoolChunkListMetric> chunkLists();

//返回总分配内存的次数
long numAllocations();

//返回tiny类型内存分配的次数
long numTinyAllocations();

//返回small类型内存分配次数
long numSmallAllocations();

//返回normal类型内存分配次数
long numNormalAllocations();

//返回分配大内存的次数
long numHugeAllocations();

//返回释放内存的总次数
long numDeallocations();

//返回tiny释放内存的次数
long numTinyDeallocations();

//返回small类型内存释放次数
long numSmallDeallocations();

//返回normal类型内存释放的次数
long numNormalDeallocations();

//返回分配大内存释放的次数
long numHugeDeallocations();

//返回内存分配后还未释放的数量
long numActiveAllocations();

//返回tiny类型内存分配后还未释放的数量
long numActiveTinyAllocations();

//返回small类型内存分配后还未释放的数量
long numActiveSmallAllocations();

//返回normal类型内存分配后还未释放的数量
long numActiveNormalAllocations();

//返回分配大内存后还未释放的数量
long numActiveHugeAllocations();

//返回所有PoolChunk的总内存大小
long numActiveBytes();
```
## 二、字段说明

```java
//枚举，用于标识内存大小所属类型
enum SizeClass {
    //分配小于512字节的内存
    Tiny,
    //分配大于等于512小于8k的内存
    Small,
    //分配大于等于8k的内存
    Normal
}

//tiny类型的PoolSubPage数组的长度，32
static final int numTinySubpagePools = 512 >>> 4;

//PooledByteBuf 分配器，这个分配器用于指定一些基本参数，比如pageSize，chunkSize，缓存大小，二叉树深度，是否启用线程内存池缓存等
final PooledByteBufAllocator parent;
//二叉树深度，默认为11
private final int maxOrder;
//页大小，默认为8k
final int pageSize;
//页偏移，log2(8k) = 13
final int pageShifts;
//一个PoolChunk的大小，默认为 1 << 24 = 16M
final int chunkSize;
// ~(pageSize - 1) 用于判断申请内存是否大于8k
final int subpageOverflowMask;
//用于指定small类型PoolSubPage数组的长度，默认是4
final int numSmallSubpagePools;
//直接内存校准值，默认是0，如果指定了大于零的值，通常在用户申请的内存大于等于chunkSize或者小于512时会使用
//原理就是将这类申请的内存修正成能够被 directMemoryCacheAlignment 整除的值
final int directMemoryCacheAlignment;
//校准值掩码，校准值是一个2的n次幂的值
final int directMemoryCacheAlignmentMask;
//tiny类型的PoolSubPage数组，用于分配小于512字节的内存
private final PoolSubpage<T>[] tinySubpagePools;
//small类型的PoolSubPage数组，用于分配大于等于512小于8k的内存
private final PoolSubpage<T>[] smallSubpagePools;

//PoolChunk集合，内部使用链表实现
//集合内存使用率为 50% ~ 100% 的PoolChunk
private final PoolChunkList<T> q050;
//集合内存使用率为 25% ~ 75% 的PoolChunk
private final PoolChunkList<T> q025;
//集合内存使用率为 1% ~ 50% 的PoolChunk
private final PoolChunkList<T> q000;
//集合内存使用率为 0% ~ 25% 的PoolChunk
private final PoolChunkList<T> qInit;
//集合内存使用率为 75% ~ 100% 的PoolChunk
private final PoolChunkList<T> q075;
//集合内存使用率为 100% 的PoolChunk
private final PoolChunkList<T> q100;

//记录 以上 q050，q025，q000，qInit，q075，q100 PoolChunkList
private final List<PoolChunkListMetric> chunkListMetrics;

// Metrics for allocations and deallocations
//记录总分配次数
private long allocationsNormal;
// We need to use the LongCounter here as this is not guarded via synchronized block.
//tiny类型内存分配次数
private final LongCounter allocationsTiny = PlatformDependent.newLongCounter();
//small类型内存分配次数
private final LongCounter allocationsSmall = PlatformDependent.newLongCounter();
//分配大内存次数
private final LongCounter allocationsHuge = PlatformDependent.newLongCounter();
//分配过单还未释放的大内存字节数
private final LongCounter activeBytesHuge = PlatformDependent.newLongCounter();

//tiny类型内存释放次数
private long deallocationsTiny;
//small类型内存释放次数
private long deallocationsSmall;
//normal类型内存释放次数
private long deallocationsNormal;

// We need to use the LongCounter here as this is not guarded via synchronized block.
//大内存释放次数
private final LongCounter deallocationsHuge = PlatformDependent.newLongCounter();

// Number of thread caches backed by this arena.
//当前PoolArean管理的内存池被多少线程缓存
final AtomicInteger numThreadCaches = new AtomicInteger();

```

## 三、内存的分配

```java
//cache：用户缓存内存块
//buf：一个还未初始化的PooledByteBuf（未指定内存块）
//reqCapacity：用户指定内存
private void io.netty.buffer.PoolArena#allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
    //将用户输入的内存大小进行规范化，如果是大于等于512字节的，规范最近的一个2的n次幂的值
    //小于512的，规范成tiny类型的内存值
    final int normCapacity = normalizeCapacity(reqCapacity);
    //判断当前申请的内存是否小于pageSize
    if (isTinyOrSmall(normCapacity)) { // capacity < pageSize
        int tableIdx;
        PoolSubpage<T>[] table;
        //是否小于512字节
        boolean tiny = isTiny(normCapacity);
        if (tiny) { // < 512
            //先从PoolThreadCache 缓存中申请内存，如果没有再从tiny类型的PoolSubPage[]中申请
            if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            //计算下标
            tableIdx = tinyIdx(normCapacity);
            table = tinySubpagePools;
        } else {
            //从缓存中申请small类型的内存
            if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            //计算下标
            tableIdx = smallIdx(normCapacity);
            table = smallSubpagePools;
        }
        //从tiny类型或者small类型的PoolSubPage数组中获取对应内存大小的头节点
        final PoolSubpage<T> head = table[tableIdx];

        synchronized (head) {
            final PoolSubpage<T> s = head.next;
            //如果s = head，需要从二叉树中申请一个8k的内存
            if (s != head) {
                assert s.doNotDestroy && s.elemSize == normCapacity;
                long handle = s.allocate();
                assert handle >= 0;
                //初始化PooledByteBuf
                s.chunk.initBufWithSubpage(buf, null, handle, reqCapacity);
                incTinySmallAllocation(tiny);
                return;
            }
        }
        synchronized (this) {
            //申请8k的内存
            allocateNormal(buf, reqCapacity, normCapacity);
        }
        //记录tiny或者small类型内存的分配次数
        incTinySmallAllocation(tiny);
        return;
    }
    //分配8k以上的内存
    if (normCapacity <= chunkSize) {
        //先从缓存中获取
        if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
            // was able to allocate out of the cache so move on
            return;
        }
        synchronized (this) {
            //(1)
            allocateNormal(buf, reqCapacity, normCapacity);
            //计数
            ++allocationsNormal;
        }
    } else {
        // Huge allocations are never served via the cache so just call allocateHuge
        //分配大内存
        allocateHuge(buf, reqCapacity);
    }
}

//(1)
private void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
    //从占用率50%左右开始分配，这样可以提高内存分配成率，又能避免分配失衡的情况
    //如果分配了内存之后，其占用率已经不符合当前PoolChunkList的占用率，那么会添加到下一个PoolChunk中
    if (q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) ||
        q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) ||
        q075.allocate(buf, reqCapacity, normCapacity)) {
        return;
    }

    // Add a new chunk.
    //如果上面没有分配成功，那么创建一个新的PoolChunk分配内存
    PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
    boolean success = c.allocate(buf, reqCapacity, normCapacity);
    assert success;
    //刚创建的先尝试添加到qInit（0~25%）中，如果这个c在刚才的分配中使用率已经超过25%了，那么会继续往下，添加到使用率为25%以上的PoolChunkList中
    qInit.add(c);
}
```
上面的方法的流程如下：

1. 对用户输入的内存req大小进行修正，如果是是大于512小于16M的，将其修正为最接近req的2的n次幂的值，如果是大于16M的或者小于512字节的都会判断是否指定了内存
   校准值，如果指定了那么使用校准值校准，将req设置为能被小准直整除的数，如果没有指定校准值，对于小于512字节的内存会修正成一个16的倍数的值。
2. 判断申请的内存大小的类型，如果是小于8k的，走PoolSubPage的方式申请
3. 对于大于8k小于16M的，直接在二叉树中分配
4. 大于16M的直接创建

对于具体的分配过程，请查看博文[《netty内存池源码分析》](https://blog.csdn.net/qq_27785239/article/details/105893805)


上面我们提到修正用户输入的内存大小，那么它是怎么修正的呢？

```java
int io.netty.buffer.PoolArena#normalizeCapacity(int reqCapacity) {
    //检查用户输入的内存大小，必须是大于0的数值
    checkPositiveOrZero(reqCapacity, "reqCapacity");
    //大于16M的，如果指定了校准值，那么使用校准值进行校准，如果没有指定，直接返回
    if (reqCapacity >= chunkSize) {
        return directMemoryCacheAlignment == 0 ? reqCapacity : alignCapacity(reqCapacity);
    }
    //处理大于等于512字节的内存
    if (!isTiny(reqCapacity)) { // >= 512
        // Doubled
        //修正为最接近reqCapacity的2的n次幂的值，原理就是将reqCapacity中最高bit位为1以下的bit位统统变成1
        //最后在加1就得到了最近的2的n次幂的值，由于最高为1的bit位不确定是第几位，所以这里迁移了31次
        int normalizedCapacity = reqCapacity;
        normalizedCapacity --;//减一是因为有可能当前reqCapacity就是一个2的n次幂的值
        normalizedCapacity |= normalizedCapacity >>>  1;//对于一个第32个bit为1的值来说，向右移动一位，至少可以得到2个1
        normalizedCapacity |= normalizedCapacity >>>  2;//因为上一步至少得到了2个1，那么此时我可以移动2位，得到了4个1
        normalizedCapacity |= normalizedCapacity >>>  4;//因为上一步至少得到了4个1，那么我可以移4位，这样就可以至少得到8个1
        normalizedCapacity |= normalizedCapacity >>>  8;//因为上一步至少得到了8个1，那么我可以移8位，这样就可以至少得到16个1
        normalizedCapacity |= normalizedCapacity >>> 16;//因为上一步至少得到了16个1，那么我可以移16位，这样就可以至少得到32个1
        normalizedCapacity ++;
        //如果溢出，无符号右移，变成整数
        if (normalizedCapacity < 0) {
            normalizedCapacity >>>= 1;
        }
        assert directMemoryCacheAlignment == 0 || (normalizedCapacity & directMemoryCacheAlignmentMask) == 0;

        return normalizedCapacity;
    }
    //是否需要校准
    if (directMemoryCacheAlignment > 0) {
        return alignCapacity(reqCapacity);
    }

    // Quantum-spaced
    //对16求余，如果没有余数，那么说明可以整除16，它是16的倍数，可以直接返回
    if ((reqCapacity & 15) == 0) {
        return reqCapacity;
    }
    //如果reqCapacity比16小，那么reqCapacity & ~15将会返回0
    //~15 低4位全是0，高28位全是1，所以如果一个值小于16，那么 reqCapacity & ~15 等于0
    //对于大于16的，进行 reqCapacity & ~15 之后低4全部置零，而高28位，不管哪个bit为是1，都是16的倍数
    return (reqCapacity & ~15) + 16;
}
```
上面一段代码对用户输入的内存req大小进行修正，如果是是大于512小于16M的，将其修正为最接近req的2的n次幂的值，如果是大于16M的或者小于512字节的都会判断是否指定
了内存校准值，如果指定了那么使用校准值校准，将req设置为能被小准直整除的数，如果没有指定校准值，对于小于512字节的内存会修正成一个16的倍数的值。

## 四、内存的释放

```java
void io.netty.buffer.PoolArena#free(PoolChunk<T> chunk, ByteBuffer nioBuffer, long handle, int normCapacity, PoolThreadCache cache) {
    //如果是非池化的，比如申请了大内存
    if (chunk.unpooled) {
        int size = chunk.chunkSize();
        //回收内存
        destroyChunk(chunk);
        //剩下还未释放的内存大小
        activeBytesHuge.add(-size);
        //释放大内存计数
        deallocationsHuge.increment();
    } else {
        //判断是什么类型的内存释放
        SizeClass sizeClass = sizeClass(normCapacity);
        //先缓存
        if (cache != null && cache.add(this, chunk, nioBuffer, handle, normCapacity, sizeClass)) {
            // cached so not free it.
            return;
        }
        //缓存失败或者无需缓存，归还到内存池中
        //(1)
        freeChunk(chunk, handle, sizeClass, nioBuffer, false);
    }
}

 //(1)
void io.netty.buffer.PoolArena#freeChunk(PoolChunk<T> chunk, long handle, SizeClass sizeClass, ByteBuffer nioBuffer, boolean finalizer) {
    final boolean destroyChunk;
    synchronized (this) {
        // We only call this if freeChunk is not called because of the PoolThreadCache finalizer as otherwise this
        // may fail due lazy class-loading in for example tomcat.
        if (!finalizer) {
            switch (sizeClass) {
                case Normal:
                    //normal释放计数
                    ++deallocationsNormal;
                    break;
                case Small:
                    //small释放计数
                    ++deallocationsSmall;
                    break;
                case Tiny:
                    //tiny释放计数
                    ++deallocationsTiny;
                    break;
                default:
                    throw new Error();
            }
        }
        //调用PoolChunkList释放，在释放完成后，其使用率会下降，如果已经下降到一定程度了，那么会进行迁移到其他PoolChunkList中
        destroyChunk = !chunk.parent.free(chunk, handle, nioBuffer);
    }
    if (destroyChunk) {
        // destroyChunk not need to be called while holding the synchronized lock.
        destroyChunk(chunk);
    }
}
```

先判断当前要释放的内存是否来自内存池，如果不是直接销毁，如果是那么归还到内存池中，其归还过程可以查看[《netty内存池源码分析》](https://blog.csdn.net/qq_27785239/article/details/105893805)
。

归还成功后，PoolChunk的使用率将下降，如果下降到原来的PoolChunkList的阈值，那么会进行前移，将其挪到使用率更小的PoolChunkList中。


## 五、总结

PoolArena 在分配内存时首先会到PoolThreadCache缓存中申请，如果申请失败，再到内存池中申请，申请时先从使用率为50%左右的PoolChunkList中申请，如果申请失败，再往
使用率低的PoolChunkList中申请。释放时首先会判断是否是非池化的内存释放，如果是那么直接销毁，如果不是，先尝试缓存到PoolThreadCache中，如果缓存失败，再归还到
内存池，归还内存池后，其对应的PoolChunk的使用率降低，降到一定程度后会进行迁移，放到其他代表使用率更低的PoolChunkList中。