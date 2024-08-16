## 一、类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/cbe1e5bc3ce0d90899bc2e5bcd98b474.png)

很简单，内部持有实现了AbstractQueuedSynchronizer的Sync，Sync为Semaphore的内部类，它没有什么特殊的成员变量，实现了基本的获取许可与释放的逻辑

## 二、许可的获取与释放

### 2.1 获取许可

```java
//非公平锁
final int java.util.concurrent.Semaphore.Sync#nonfairTryAcquireShared(int acquires) {
    for (;;) {
        //获取可用的许可个数
        int available = getState();
        //减去需要请求的许可数
        int remaining = available - acquires;
        //如果许可小于零，那么认为获取许可失败，如果是正数，那么进行CAS改值
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}

//公平锁
protected int java.util.concurrent.Semaphore.FairSync#tryAcquireShared(int acquires) {
    for (;;) {
        //先判断队列中是否存在其他waiting的节点，如果有，判定线程获取许可失败，保证公平性
        if (hasQueuedPredecessors())
            return -1;
        
        //获取可用的许可个数
        int available = getState();
        //减去需要请求的许可数
        int remaining = available - acquires;
        //如果许可小于零，那么认为获取许可失败，如果是正数，那么进行CAS改值
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}

```
当我们创建信号量的时候会设置一定数量的许可，线程获取到一个许可便会将许可总数减去1，然后返回剩余的许可数量，如果剩余的许可数量非负数，那么表示获取许可成功。
如果获取许可失败，需要加入到同步队列中等待其他线程释放许可唤醒自己

```java
private void java.util.concurrent.locks.AbstractQueuedSynchronizer#doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    //CAS添加节点到同步队列，具体的添加过程，请移步到第5节，此处不再赘述
    //Node.SHARED用于表示当前线程所处模式为共享模式，是来竞争共享锁的
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            //如果前驱节点是空的头节点，那么尝试去获取许可
            if (p == head) {
                int r = tryAcquireShared(arg);
                //如果获取许可成功，那么进入分支，准备替换头节点，并根据需要传播唤醒行为
                if (r >= 0) {
                    //替换头节点为当前线程之前所在的节点，然后判断是需要传播唤醒行为
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            //前驱节点必须为SIGNAL才允许挂起线程，如果不是SIGNAL，是CANCEL，那么先去掉当前节点连续的前驱CANCEL节点
            //如果是其他的状态，比如0或者传播状态，那么先将前驱节点设置为SIGNAL，然后再尝试竞争许可，具体逻辑与这么做的原因
            //请移步到第5节JDK并发锁之AQS与独占锁
            if (shouldParkAfterFailedAcquire(p, node) &&
                //挂起
                parkAndCheckInterrupt())
                //如果发生中断，将抛出中断异常
                throw new InterruptedException();
        }
    } finally {
        //如果许可获取失败（通常在设置有超时时间获取许可的方法中可能会导致failed = true），CANCEL节点，具体逻辑请移步第5节JDK并发锁之AQS与独占锁
        if (failed)
            cancelAcquire(node);
    }
}
```

### 2.1 释放许可

```java
protected final boolean java.util.concurrent.Semaphore.Sync#tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        //将许可还回去
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
```
许可的释放仅仅是将原来线程占用的许可还回去即可

## 三、总结

只要学习了第6节点读写锁，理解Semaphore是非常简单，所以对于Semaphore不会花太多的重复篇幅去分析。

