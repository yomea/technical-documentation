## 一、类图
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/fd75e34c65e8dda88353bc4dde38dcb2.png)

### 1.1 CyclicBarrier类基础

```java
//珊栏
public class CyclicBarrier {
    
    //内部类，表示代，每次珊栏内线程容量达到指定容量后会进行换代，就简单理解为改朝换代吧
    private static class Generation {
        //表示珊栏是否被破坏，比如抛出错误啥的
        boolean broken = false;
    }

    
    //重入锁，关于重入锁的分析可参考第5节JDK并发锁之AQS与独占锁
    private final ReentrantLock lock = new ReentrantLock();
    
    //条件，关于Condition的分析可参考第5节JDK并发锁之AQS与独占锁
    private final Condition trip = lock.newCondition();
    
    //珊栏可容纳线程个数
    private final int parties;
    
    //表示珊栏达到parties个数时的通知函数
    private final Runnable barrierCommand;
    
    //表示代，每次珊栏内的线程被唤醒都会构建一个新的代
    private Generation generation = new Generation();

    //表示珊栏此刻剩余的可容纳线程数
    private int count;
    
}
```
### 1.2 构造器

```java
//parties：表示珊栏的容量
public CyclicBarrier(int parties) {
    this(parties, null);
}

//parties：表示珊栏的容量，barrierAction：表示珊栏达到容量时进行回调的函数
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    //表示珊栏容量大小
    this.parties = parties;
    //count表示珊栏剩余可用容量大小
    this.count = parties;
    //珊栏达到容量大小时的回调函数
    this.barrierCommand = barrierAction;
}
```


## 二、await

```java
private int java.util.concurrent.CyclicBarrier#dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
    final ReentrantLock lock = this.lock;
    //获取独占锁
    lock.lock();
    try {
        //记录当前代
        final Generation g = generation;
        //当前代的线程是否已经释放，如果已经释放，那么不允许再使用
        if (g.broken)
            throw new BrokenBarrierException();

        //如果有一个线程发生中断，那么唤醒所有被waiting在条件队列的线程
        //然后标志当前珊栏被broken
        //珊栏可用容量count重新赋值 count = parties
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }
        
        //珊栏可用容量递减
        int index = --count;
        //如果可用容量达到了零，那么可以改朝换代了
        if (index == 0) {  // tripped
            //ranAction用于标识唤醒通知是否正常执行
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    //如果接收唤醒通知的观察者执行发生错误，那么ranAction将不能赋值为true
                    command.run();
                ranAction = true;
                //唤醒所有阻塞在条件队列的线程
                //重置珊栏可用个数
                //new出一个新的代
                nextGeneration();
                return 0;
            } finally {
                //如果接收唤醒通知的观察者执行发生错误，那么ranAction将不能赋值为true，此处将调用breakBarrier方法去唤醒
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                //是否需要定时，如果不需要，那么直接阻塞
                if (!timed)
                    trip.await();
                else if (nanos > 0L)
                    //指定阻塞时间
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                //发生中断异常
                //检查是否还是原来的代，如果还是原来的代，也就是珊栏容量没有使用完，发生中断异常
                //并且没有broken，那么主动唤醒其他wait的线程，然后broken
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    //如果是在珊栏可用容量变为零也就是nextGeneration后抛出的中断异常，那么主动中断当前线程，恢复中断标志，交由用户
                    //处理，因为它不属于在珊栏wait期间的中断
                    Thread.currentThread().interrupt();
                }
            }
            //broken = true，抛出珊栏被broken的异常
            if (g.broken)
                throw new BrokenBarrierException();
            //某线程被唤醒后，如果珊栏容量被用完发生的signal行为，那么必定会更新代，如果代没有发生更新，有以下情况
            //1、用户指定了wait超时时间，在指定时间内线程自动被唤醒，然后发现珊栏还没有被signal
            if (g != generation)
                return index;
                
            //错误的定时参数
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        //释放锁
        lock.unlock();
    }
}
```

从dowait方法中可以看到，CyclicBarrier直接使用了重入锁和Condition来实现珊栏，首先会判断当前珊栏是否被broken，被broken的珊栏不允许再次使用，需要调用reset方法
进行修补（为啥有种亡羊补牢的感觉）。修补的方式很简单代码如下：

```java
public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //先broken，然后恢复珊栏容量，最后signal条件队列
        breakBarrier();   // break the current generation
        //先signal条件队列，然后恢复珊栏容量，最后new一个新的generation
        nextGeneration(); // start a new generation
    } finally {
        lock.unlock();
    }
}
```
breakBarrier与nextGeneration方法的代码

```java
private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}

private void nextGeneration() {
    // signal completion of last generation
    trip.signalAll();
    // set up next generation
    count = parties;
    generation = new Generation();
}
```
条件队列的具体signal过程请移步到第5节的《JDK并发锁之AQS与独占锁》博文，此处不再赘述。

一个代被broken，通常是因为抛出了异常导致，对于这种抛出了异常的珊栏，默认是不会再启用的，主要是考虑它很可能在接下的业务处理中仍然会抛出这样错误，所以用户
要么选择去排除问题，要么选择直接reset重新启动。

当珊栏容量达到最大值的时候，会主动唤醒条件队列中的线程，并恢复可用容量，构建下一个代。

## 三、总结

Doug Lea直接使用ReentrantLock与Condition实现了CyclicBarrier，其源码相对来说不是很难理解，各路朋友只要仔细阅读并可理解其实现原理