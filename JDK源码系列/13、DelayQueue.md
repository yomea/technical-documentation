## 一、类图
在学习延时队列之前，先移步到第12节学习 [PriorityQueue](https://blog.csdn.net/qq_27785239/article/details/103218391)
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/655f77b6c6a764faecdc186ff193a979.png)

- Iterable：可迭代的，声明实现此接口的类具有迭代元素的能力，在JDK8之后增加了默认方法forEach与spliterator，forEach用于遍历元素，spliterator用于分割迭代器，通常 用于并行流
- Collection：定义一些集合操作的基本方法，比如add元素，删除元素，包含，交集等等，还有JDK8的流
- AbstractCollection：模板类，实现了基本的方法，比如remove，contains等方法
- Queue：定义基本的队列操作方法，add，offer，poll，peek等
- AbstractQueue：模板类，实现了Queue接口的一些基本操作方法
- Itr：DelayQueue的内部类，实现了Iterator接口
- BlockingQueue：定义阻塞方法，表示对应的一些方法应该使用阻塞的方式实现

## 二、DelayQueue字段说明

```java
//用于同步
private final transient ReentrantLock lock = new ReentrantLock();

//DelayQueue的实现依赖于PriorityQueue，优先队列请看第12节的PriorityQueue源码分析
private final PriorityQueue<E> q = new PriorityQueue<E>();

//第一个等待某个延时对象的线程，在延时对象还没有到期时，其他线程看到这个leader有值，那么就直接无时间限制wait
//主要是为了避免大量线程在同一时间点唤醒，导致大量的竞争，反而影响性能
//另一方面，leader = null表示有线程获取到了延时对象，具体往后看
private Thread leader = null;

//条件队列，用于wait线程
private final Condition available = lock.newCondition();

```

## 三、添加元素

```java
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //调用优先队列
        q.offer(e);
        //如果加入元素就是第一个元素，那么说明之前队列中没有值，现在有值了，那么需要唤醒之前被wait住的线程
        if (q.peek() == e) {
            leader = null;
            available.signal();
        }
        return true;
    } finally {
        lock.unlock();
    }
}
```

## 四、获取元素

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            //从优先队列中获取第一个元素，peek方法不会删除元素
            E first = q.peek();
            //如果队列中还没有值，那么等待
            if (first == null)
                available.await();
            else {
                //获取当前延时对象是否到期
                long delay = first.getDelay(NANOSECONDS);
                //到期那么返回这个延时对象
                if (delay <= 0)
                    return q.poll();
                first = null; // 
                //leader不为空，表明已经有其他线程在等待这个延时对象了
                //为什么不available.awaitNanos(delay)呢？这将会导致大量的线程在同一时间点被唤醒，然后去竞争
                //这个到期的延时任务，影响性能，还不如直接将他们无时间限制的wait，leader线程或者其他新进来的线程获取到延时对象后，去唤醒
                //让他们去竞争下一个延时对象
                if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        //wait对应延时对象的过期时间，是不是就说，这个阻塞的线程醒了之后就一定能获取到之前为其等待的延时对象呢？
                        //并不是，当前wait住的线程被唤醒后有可能与其他线程竞争失败，就会进入了同步队列阻塞，那个抢到锁的线程就会取走这个延时对象
                        //多么悲伤的故事啊！苦苦等待的人，最后却还是别人的。
                        available.awaitNanos(delay);
                    } finally {
                        //leader线程被唤醒并获取到锁之后会将leader设置为空
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        //leader为空并且队列不为空，那么唤醒正在等待的线程
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock();
    }
}
```

从优先队列中取值，如果取到的延时节点已经已经到期，那么直接返回，如果还没有到期并且已经有其他线程在执行delay时间等待了（也就是leader线程），那么挂起自己（避免延时
相同时间造成大量线程同时唤醒），
leader线程在指定delay时间后主动唤醒，然后取竞争锁，如果竞争成功，那么很大概率可以获取到延时节点，如果竞争失败，将被阻塞。

## 五、删除元素

```java
public boolean remove(Object o) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //调用优先队列的移除方法
        return q.remove(o);
    } finally {
        lock.unlock();
    }
}
```

## 六、总结

DelayQueue队列使用了装饰器模式，在PriorityQueue队列的基础上增加了延时时间获取元素的功能