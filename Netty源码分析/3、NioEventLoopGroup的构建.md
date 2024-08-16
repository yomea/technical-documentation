在前一小节中我们大致浏览了NioEventLoopGroup的类图和与直接相关的类的类图，现在我们直接进入细节代码，分析NioEventLoopGroup的构建

下面这段代码，在第一小节我们已经看过了，这里再回顾下

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
            //从上面一路走来，executor是null值，这里创建了一个ThreadPerTaskExecutor，从第一小节中可以知道它对于每个提交的任务都会使用
            //DefaultThreadFactory创建一个新的线程去执行
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
对以上代码进行分阶段分析，然后再总结下它做了什么，现在关联了什么，再画个序列图，这样我们才能清楚刚才都做了什么，才不至于忘记

NioEventLoopGroup的构建序列图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/7d6255ae2c777e4341ded54dba11d48b.png)

> 构建EventExecutor数组

前面说到构建EventExecutor数组的元素是NioEventLoop，我们来看看它是怎么构建的

```
//parent：这个值是上面的NioEventLoopGroup对象，executor：这个是ThreadPerTaskExecutor对象，selectorProvider：selector多路复用器提供者
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider) {
        super(parent, executor, false);
        if (selectorProvider == null) {
            throw new NullPointerException("selectorProvider");
        }
        provider = selectorProvider;
        selector = openSelector();
}
                    |
                    V
protected SingleThreadEventLoop(EventLoopGroup parent, Executor executor, boolean addTaskWakesUp) {
        super(parent, executor, addTaskWakesUp);
}
                    |
                    V
protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor, boolean addTaskWakesUp) {
        super(parent);

        if (executor == null) {
            throw new NullPointerException("executor");
        }

        //默认传入的是false
        this.addTaskWakesUp = addTaskWakesUp;
        this.executor = executor;
        //new LinkedBlockingQueue<Runnable>()
        //构建了一个无边界阻塞队列
        taskQueue = newTaskQueue();
}
                    |
                    V
protected AbstractEventExecutor(EventExecutorGroup parent) {
        this.parent = parent;
}
```
下面是它的序列图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d8551fa314d262d583c6f1480fa6665d.png)

下面是NioEventLoopGroup关系类图，省略了部分继承结构，继承结构可查看上一小节

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/37030f60fc0d5c3dcd9b76b71cffa808.png)

好了，这样NioEventLoopGroup就构建好了。