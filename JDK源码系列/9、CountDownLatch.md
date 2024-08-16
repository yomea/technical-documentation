## 一、类图
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4229f271a3c836705226d1016804b36c.png)

从类图中可以看到CountDownLatch有一个内部类Sync，那么可以肯定CountDownLatch是基于AQS来实现的

以下为CountDownLatch的构造函数

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```
## 二、await

CountDownLatch#await方法代码如下：

```java
public void java.util.concurrent.CountDownLatch#await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
                                    |
                                    V
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    //获取共享锁
    if (tryAcquireShared(arg) < 0)
        //自旋挂起，这段代码在第6节分析读写锁的时候进行了深入的分析，此处不再重入赘述
        //大致步骤讲下：首先将线程包装成Node添加到同步队列中，然后会判断前驱节点是否为头节点，如果是头节点那么会再次进行一次获取锁的动作，因为在设置前驱节点
        //为SIGNAL之前有可能有线程释放了锁，然后在判断前驱节点是否为SIGNAL，如果不是将前驱节点设置为SIGNAL，然后在自旋一次，如果还是获取不到锁，前驱节点为
        //SIGNAL的waitStatus的情况下将会被挂起
        doAcquireSharedInterruptibly(arg);
}
```
doAcquireSharedInterruptibly是定义在AQS抽象类中的通用代码，这段代码在第6节分析读写锁的时候进行了深入的分析，此处不再重入赘述，但是
大致步骤还是讲下：首先将线程包装成Node添加到同步队列中，然后会判断前驱节点是否为头节点，如果是头节点那么会再次进行一次获取锁的动作，因为在设置前驱节点
为SIGNAL之前有可能有线程释放了锁，然后在判断前驱节点是否为SIGNAL，如果不是将前驱节点设置为SIGNAL，然后在自旋一次，如果还是获取不到锁，前驱节点为
SIGNAL的waitStatus的情况下将会被挂起

获取共享锁的代码如下：

```java
protected int java.util.concurrent.CountDownLatch.Sync#tryAcquireShared(int acquires) {
    //如果计数等于零，那么返回1表示获取共享锁成功，否则失败
    return (getState() == 0) ? 1 : -1;
}
```
我们在构建CountDownLatch的时候会传入一个计数值，当这个计数值被递减为零后，将会唤醒条件队列中的线程，下面是递减计数器的代码

```java
public void java.util.concurrent.CountDownLatch#countDown() {
    sync.releaseShared(1);
}
                                   |
                                   V
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        //这段代码是AQS抽象类中的通用代码，我们在第6节分析读写锁的时候重点进行了分析
        //大致逻辑是：检查头节点是否为SIGNAL的，如果是，那么会唤醒下一个节点，如果是0，那么会被设置为PROPAGATE，以便唤醒线程能够继续传播唤醒行为
        doReleaseShared();
        return true;
    }
    return false;
}        
```
上面的doReleaseShared方法是AQS抽象类中的通用代码，我们在第6节分析读写锁的时候重点进行了分析，其
大致逻辑是：检查头节点是否为SIGNAL的，如果是，那么会唤醒下一个节点，如果是0，那么会被设置为PROPAGATE，以便唤醒线程能够继续传播唤醒行为

下面是释放共享锁的逻辑：

```java
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState
        //如果已经是零了，那么没必要重复返回true，这样会导致线程重复的去检查是否需要唤醒下一个节点
        if (c == 0)
            return false;
        //递减
        int nextc = c-1;
        //CAS改值，等于零的时候就可以唤醒同步队列中的线程了
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

CountDownLatch除了构造器之外，没有暴露重置计数器的方法，所以CountDownLatch是一次性用品，它的计数器被设置成零之后，其他线程再次调用它的await方法，不会被阻塞

## 三、总结

- CountDownLatch与CyclicBarrier从功能上来讲都是阻塞一批线程，然后通过达到某个条件之后以传播的方式唤醒所有的线程。
- 从实现上来看，CountDownLatch是直接实现的AQS框架，在计数没有被置零的情况下，可以有任意多的线程去获取共享锁，然后被阻塞，直到计数值被置零，然后以传播的方式
  唤醒同步队列中被阻塞的线程，CountDownLatch是一次性的，它不像CyclicBarrier有reset方法，在CountDownLatch被置零之后，任意线程调用其await方法将不会再被阻塞
- CyclicBarrier是通过重入锁和Condition配置实现的，CyclicBarrier的容量是有限制的，其线程阻塞使用的队列为条件队列，在容量达到限制时，将会将条件队列中的线程
  转移到同步队列中，最后通过某线程的unlock进行唤醒，由于从线程await的地方到unlock这段代码间没有做比较费时的操作，看起来依然像瘟疫一样快速传播，但是如果有额外
  的线程介入的话，将会发生锁的竞争。不过额外介入的线程很快会调用await方法将自己挂起，如果运气不好那个因竞争失败的线程可能会被挂起，需要再次唤醒。