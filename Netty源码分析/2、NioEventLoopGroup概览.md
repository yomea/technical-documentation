
下面是一个简单的服务器程序

```
public class TimeServer {
	
	
	public void bind(int port) {
		//NIO线程组，Reactor线程组，一个用于接受客户端的请求，另一个用于进行SocketChannel的操作
		EventLoopGroup bossGroup = new NioEventLoopGroup();
		
		。。。。。。
}
```
在上一节中，我们看了一个小例子，其中有上面的一段代码，我们可以看到它构建了一个NioEventLoopGroup类，现在我们看下这个类的类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/2a4ec9ec9eaa2732d218b312a4a419a7.png)

首先，我们尝试着去了解他们每个类的作用，如果暂时看不懂的跳过，后面肯定会用到，自然而然就知道是用来做什么的了。

- Executor：JDK提供的接口，就一个方法

```
//提交任务执行，相信用过线程池的人都认识这个方法
void execute(Runnable command);
```
- ExecutorService：JDK提供的接口，这个接口最大变化就是提供了Future，用于阻塞等待结果返回

- ScheduledExecutorService：JDK提供的接口，从字面意思上来看就是调度执行服务，增加了定时，延时调度的方法

- EventExecutorGroup：netty提供的接口，从字面上来看，它表示时间执行器组，可以包含多个事件执行器，所以它有一下方法

```
//获取下一个事件执行器
EventExecutor next();

//获取所有事件执行器，而且还是子执行器
<E extends EventExecutor> Set<E> children();
```
另外它还添加了优雅关闭线程的接口方法

- NioEventLoopGroup：最大亮点就是增加了一个rebuildSelectors方法

```
public void rebuildSelectors() {
        for (EventExecutor e: children()) {
            ((NioEventLoop) e).rebuildSelector();
        }
    }
```
rebuildSelectors方法循环调用每个事件执行器的rebuildSelector方法，结合方法名和netty源码中注释可以知道，它就是重新创建selector的方法。

- AbstractEventExecutorGroup：这个是抽象类，一看就知道肯定是一个模板类，提供某部分方法的实现，具体不同的动作由子类去实现，取其中一个方法来看看

```
public Future<?> submit(Runnable task) {
        //获取下一个事件执行器，然后向它提交任务
        return next().submit(task);
}
```
从上面的代码来看，它调用过了next()方法，这个方法最早是在EventExecutorGroup中定义的，顺便说下其他的提交任务，不管是延时，还是立即的都是调用了这个方法去获取事件执行器，然后提交任务。

- MultithreadEventExecutorGroup：多线程事件执行器组。

```
public abstract class MultithreadEventExecutorGroup extends AbstractEventExecutorGroup {
    //事件执行器数组
    private final EventExecutor[] children;
    //只读事件执行器
    private final Set<EventExecutor> readonlyChildren;
    //子线程事件执行器下标
    private final AtomicInteger childIndex = new AtomicInteger();
    //终止的子线程事件执行器下标还是计数？？？这个在后面看到在回答
    private final AtomicInteger terminatedChildren = new AtomicInteger();
    //Promise???是什么，后面在看
    private final Promise<?> terminationFuture = new DefaultPromise(GlobalEventExecutor.INSTANCE);
    
    。。。。。。
}
```

除此之外，这个类与其父类相比，在构造器中做了一些初始化操作，而且还是实现了next()方法，构造器中逻辑比较多，而且依赖外部传递参数，暂时不能完全明白其意图，这个可以押后，但是next方法比较简单，直接就可以看懂

```
public EventExecutor next() {
        //从数组中轮询的方式获取事件执行器
        return children[Math.abs(childIndex.getAndIncrement() % children.length)];
    }
```
此处之外，它新增了一个抽象方法

```
//构建事件执行器方法
protected abstract EventExecutor newChild(Executor executor, Object... args) throws Exception;
```


- MultithreadEventLoopGroup：多线程事件循环组，看名字就是循环执行事件的意思，真正的意图还是要看代码，它有个静态块

```
//默认的事件循环线程
private static final int DEFAULT_EVENT_LOOP_THREADS;

static {
    //如果系统属性中没有指定eventLoopThreads线程数，那么获取系统cpu核心数的两倍
    DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
            "io.netty.eventLoopThreads", Runtime.getRuntime().availableProcessors() * 2));

    if (logger.isDebugEnabled()) {
        logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
    }
}
```
除此之外，它继承了父类的newChild方法，但是没有进行实现

```
protected abstract EventLoop newChild(Executor executor, Object... args) throws Exception;
```

- NioEventLoopGroup：NIO事件循环组，从字面上来将这个执行组是用于NIO的，它实现了newChild方法

```
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        //构建的是NioEventLoop，那么问题来了，这个NioEventLoop又是干啥的呢？
        return new NioEventLoop(this, executor, (SelectorProvider) args[0]);
}
```
我们大致了解了NioEventLoopGroup的作用，那么我们现在开始进入NioEventLoopGroup的构建构成过程

```
public NioEventLoopGroup() {
        this(0);
}
            |
            V
public NioEventLoopGroup(int nThreads) {
        this(nThreads, (Executor) null);
}
            |
            V
public NioEventLoopGroup(int nThreads, Executor executor) {
        //SelectorProvider适用于打开selector多用复用器的提供者
        this(nThreads, executor, SelectorProvider.provider());
}
            |
            V
public NioEventLoopGroup(
            int nThreads, Executor executor, final SelectorProvider selectorProvider) {
        super(nThreads, executor, selectorProvider);
}
            |
            V
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
        //假如此时你的电脑是4核心的，按照MultithreadEventLoopGroup静态块的操作，这里的值就是8，也就是构建8个线程
        super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
}
            |
            V
protected MultithreadEventExecutorGroup(int nThreads, Executor executor, Object... args) {
        if (nThreads <= 0) {
            throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
        }

        if (executor == null) {
            //从上面一路走来，executor是null值，所以这里讲创建一个ThreadPerTaskExecutor，那么这个类是干啥的呢？？？后面再说
            //newDefaultThreadFactory这个方法是用于创建线程工厂，用于获取线程的工厂类
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }
        //构建子事件执行器数组
        //注意children是MultithreadEventExecutorGroup的成员变量
        children = new EventExecutor[nThreads];
        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                //调用NioEventLoopGroup实现的newChild方法，上面我们已经看到过，它构建的EventExecutor是NioEventLoop
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                //如果抛出错误，也就是创建事件执行器不成功，执行以下逻辑
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        //循环调用每个事件执行器的优雅关闭方法，这里迫切需要知道这个执行器是用了干啥的了，字面意思肯定是执行任务的类
                        children[j].shutdownGracefully();
                    }

                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                //如果没有关闭，主线程等待它关闭
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            //发生中断
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }
        //这里又涉及到了一个类，从字面上来看就是Future监听器
        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                //terminatedChildren已经解答了上面的问题，它就是用来记录停止执行器的个数
                //如果终止的执行器数量已经等于children的数量了，那么说明全都终止了，然后设置terminationFuture为null
                //这个啥意思呢？带着问题继续吧
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                }
            }
        };

        for (EventExecutor e: children) {
            //给每个EventExecutor添加终止监听器，也就是每个执行器内部的任务完成都会调用这个监听方法
            e.terminationFuture().addListener(terminationListener);
        }

        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        //添加到只读集合中
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
}
```

好了，目前看到的与NioEventLoopGroup直接相关的未知类有SelectorProvider，DefaultPromise（MultithreadEventExecutorGroup的一个成员对象），FutureListener，NioEventLoop（newChild方法构建的执行器），ThreadPerTaskExecutor（传递给newChild的参数），DefaultThreadFactory（线程工厂）

> DefaultPromise

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/45731d3560e15c33b9a8f68a3c37d8b1.png)

- java.util.concurrent.Future：用于阻塞线程，知道有结果返回

- io.netty.util.concurrent.Future：继承自java.util.concurrent.Future，添加了监听器EventListener，就是上面的FutureListener这类监听器

- io.netty.util.concurrent.Promise：这个类继承io.netty.util.concurrent.Future，它有个方法就是上面我们遇到的那个方法

```
Promise<V> setSuccess(V result);
```
从源代码中的解释来看，它表示某个future是否成功，并通知注册到它的监听器

- AbstractFuture：实现了get方法
- DefaultPromise：实现了添加，移除监听器，get等方法

> SelectorProvider

这是一个selector多路复用器提供者，可以通过属性参数java.nio.channels.spi.SelectorProvider执行自定义selector提供者，其他情况一般都是由JDK的ServiceLoader去加载selector提供者，一般在windows平台上，selector的实例为WindowsSelectorImpl。

> FutureListener

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/f8c6564dfcf798861bcfd1ffadd5f8d8.png)

- EventListener：这是jdk提供的一个标记接口
  注意上面的Futrue是netty的接口，就是那个定义了添加监听器的接口
  FutureListener本身也是个接口，它没有新增任何方法，但是它的父类GenericFutureListener有一个方法

```
//当netty的Future（继承自jdk的Future）完成时触发
void GenericFutureListener.operationComplete(F future) throws Exception;
```
> DefaultThreadFactory

这个默认的线程工厂是netty提供的，它实现了jdk的ThreadFactory，它的构造器

```
protected ThreadFactory MultithreadEventLoopGroup.newDefaultThreadFactory() {
        return new DefaultThreadFactory(getClass(), Thread.MAX_PRIORITY);
}

public DefaultThreadFactory(Class<?> poolType, int priority) {
        this(poolType, false, priority);
}

public DefaultThreadFactory(Class<?> poolType, boolean daemon, int priority) {
        //toPollName方法，用于构建线程池名称
        this(toPoolName(poolType), daemon, priority);
}
//我们构建的是NioEventLoopGroup对象，所以这个池类型就是NioEventLoopGroup
private static String DefaultThreadFactory.toPoolName(Class<?> poolType) {
        if (poolType == null) {
            throw new NullPointerException("poolType");
        }
        String poolName;
        Package pkg = poolType.getPackage();
        if (pkg != null) {
            poolName = poolType.getName().substring(pkg.getName().length() + 1);
        } else {
            poolName = poolType.getName();
        }

        switch (poolName.length()) {
            case 0:
                return "unknown";
            case 1:
                return poolName.toLowerCase(Locale.US);
            default:
                if (Character.isUpperCase(poolName.charAt(0)) && Character.isLowerCase(poolName.charAt(1))) {
                    return Character.toLowerCase(poolName.charAt(0)) + poolName.substring(1);
                } else {
                    return poolName;
                }
        }
}

//从上面的代码来看，它的名字为nioEventLoopGroup
//poolName：线程池名称，daemon：是否为守护线程，priority：定义线程优先级
public DefaultThreadFactory(String poolName, boolean daemon, int priority) {
        if (poolName == null) {
            throw new NullPointerException("poolName");
        }
        if (priority < Thread.MIN_PRIORITY || priority > Thread.MAX_PRIORITY) {
            throw new IllegalArgumentException(
                    "priority: " + priority + " (expected: Thread.MIN_PRIORITY <= priority <= Thread.MAX_PRIORITY)");
        }
        //nioEventLoopGroup-原子递增的数字-
        prefix = poolName + '-' + poolId.incrementAndGet() + '-';
        this.daemon = daemon;
        this.priority = priority;
}
```
创建线程

```
public Thread newThread(Runnable r) {
        //线程名就变成了 nioEventLoopGroup-原子递增的数字-原子递增的数字
        Thread t = new Thread(r, prefix + nextId.incrementAndGet());
        try {
            if (t.isDaemon()) {
                if (!daemon) {
                    t.setDaemon(false);
                }
            } else {
                if (daemon) {
                    t.setDaemon(true);
                }
            }

            if (t.getPriority() != priority) {
                t.setPriority(priority);
            }
        } catch (Exception ignored) {
            // Doesn't matter even if failed to set.
        }
        return t;
}
```

> ThreadPerTaskExecutor

这也是netty实现的执行器，它很简单，就下面几行代码

```
public final class ThreadPerTaskExecutor implements Executor {
    //这个就是我们上面分析的那个线程工程
    private final ThreadFactory threadFactory;

    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
        if (threadFactory == null) {
            throw new NullPointerException("threadFactory");
        }
        this.threadFactory = threadFactory;
    }

    @Override
    public void execute(Runnable command) {
        threadFactory.newThread(command).start();
    }
    
}

```

> NioEventLoop

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/12033fe442aafb99ba36ee163909bfd9.png)

已经分析过的类就不再赘述了，我们来看看那些没有分析过的

- AbstractExecutorService：JDK提供的类，对ExecutorService进行一部分默认的实现
- EventExecutor：这个接口继承自EventExecutorGroup，新增parent方法，获取父事件执行组，还有newPromise，创建Promise，还有判断某个线程是否是当前执行组的线程
- AbstractEventExecutor：主要实现了parent方法，其父执行器组通过构造器注入

```
protected AbstractEventExecutor(EventExecutorGroup parent) {
    this.parent = parent;
}
    
public EventExecutorGroup parent() {
    return parent;
}
```
- SingleThreadEventExecutor：单线程事件执行器，从字面意思上看就是一个单线程的额执行器，它定义好多的字段，有些字段暂时不知道啥意思，留到后面在回头看，我们只要大概知道这个类是干啥的就行了，当然有些字段还是很好理解的，比如taskQueue，一看就是用于存放任务的队列
- SingleThreadEventLoop：加了一个成员变量ChannelHandlerInvoker，这个类是干啥的呢？暂时先不管，我们先理解NioEventLoopGroup和它的父类以及与它直接相关的类的大致意思，这样看起源码来比较好理解些

```
public abstract class SingleThreadEventLoop extends SingleThreadEventExecutor implements EventLoop {

    private final ChannelHandlerInvoker invoker = new DefaultChannelHandlerInvoker(this);
    
    。。。。。。
}
```
- NioEventLoop：这个类也添加了很多的字段，其中有一下比较容易看懂的字段

```
//多路复用器
Selector selector;
//SelectionKey集合
private SelectedSelectionKeySet selectedKeys;
//多路复用器提供者
private final SelectorProvider provider;
```

之所以需要大致了解某个类是为了增加印象，很多时候我们并不一定能够看懂一个类到底是用来干什么的，但是通过类图，大致的浏览，可以让我接下来真正读他们的构建，执行有一定的帮助，边读代码边看类图，还有大致的介绍，然后话序列图，这样就不至于在局部代码中迷失方向。
