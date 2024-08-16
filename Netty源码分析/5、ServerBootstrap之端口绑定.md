上一节讲到通道的绑定和排管，我们来回顾下绑定方法

```
private ChannelFuture io.netty.bootstrap.AbstractBootstrap#doBind(final SocketAddress localAddress) {
        //初始化和注册通道
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }

        final ChannelPromise promise;
        //是否已经注册完成
        if (regFuture.isDone()) {
            //构建一个新的Promise，用于记录结果
            promise = channel.newPromise();
            //绑定
            doBind0(regFuture, channel, localAddress, promise);
        } else {
            // Registration future is almost always fulfilled already, but just in case it's not.
            //如果异步注册还没有完成，那么创建一个带有全局事件执行器的Promise
            promise = new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE);
            //给注册future注册通知，如果注册成功后将会调用这个ChannelFutureListener的operationComplete方法
            regFuture.addLioperationComplete方法stener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    doBind0(regFuture, channel, localAddress, promise);
                }
            });
        }

        return promise;
}
```
> AbstractBootstrap#doBind0

```
//regFuture：注册future，channel：NioServerSocketChannal，localAddress：你懂的，promise：可能是带有全局事件执行器的ChannelPromise，
//也可能不带任何执行器的ChannelPromise
private static void io.netty.bootstrap.AbstractBootstrap#doBind0(
            final ChannelFuture regFuture, final Channel channel,
            final SocketAddress localAddress, final ChannelPromise promise) {

        // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
        // the pipeline in its channelRegistered() implementation.
        //异步执行
        channel.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                if (regFuture.isSuccess()) {
                    channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
                } else {
                    promise.setFailure(regFuture.cause());
                }
            }
        });
}
```
看到上面，任务绑定任务交给了事件执行器，我们先来分析下事件执行器的是怎么提交任务的

```
public void io.netty.util.concurrent.SingleThreadEventExecutor#execute(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }
        //判断当前线程是否已经是EventExecutor中的线程
        boolean inEventLoop = inEventLoop();
        //如果是，那么直接添加到taskQueue任务队列中，我们在前面分析过这个任务队列是一个LinkedBlockingQueue，无界队列
        if (inEventLoop) {
            //会做些校验，比如添加的任务不能为空，线程池没有被关闭，否则拒绝添加
            //然后直接add到队列中
            addTask(task);
        } else {
            //开始线程
            //(*1*)
            startThread();
            //会做些校验，比如添加的任务不能为空，线程池没有被关闭，否则拒绝添加
            //然后直接add到队列中
            addTask(task);
            //如果执行器已经被关闭，并且移除任务成功，抛出拒绝执行任务的异常
            if (isShutdown() && removeTask(task)) {
                reject();
            }
        }
        //是否在添加任务的时候会主动唤醒线程
        if (!addTaskWakesUp) {
            //如果当前线程不是内部执行器线程或者执行器已经停止，那么添加WAKEUP_TASK，一个空任务，主要是为了让循环的NioEventLoop#select的线程得到释放
            wakeup(inEventLoop);
        }
}

//(*1*)
private void io.netty.util.concurrent.SingleThreadEventExecutor#startThread() {
        synchronized (stateLock) {
            //如果执行器还未启动，那么启动，其他情况忽略
            if (state == ST_NOT_STARTED) {
                state = ST_STARTED;
                //添加延时任务，ScheduledFutureTask是一个调度任务，在指定延时后才会执行
                //第一个参数表示执行器对象，第二参数是延时任务队列，第三个参数将返回一个RunnableAdapter，这个类实现了Runnable接口
                //第四个参数是延时时间，最后一个参数表示周期定时，这里为负数，只会执行一次
                delayedTaskQueue.add(new ScheduledFutureTask<Void>(
                        this, delayedTaskQueue, Executors.<Void>callable(new PurgeTask(), null),
                        ScheduledFutureTask.deadlineNanos(SCHEDULE_PURGE_INTERVAL), -SCHEDULE_PURGE_INTERVAL));
                doStartThread();
            }
        }
    }

    private void io.netty.util.concurrent.SingleThreadEventExecutor#doStartThread() {
        assert thread == null;
        //executor就是前面我们分析过的ThreadPerTaskExecutor，也就会为每个任务创建一个新的线程去执行
        executor.execute(new Runnable() {
            @Override
            public void run() {
                //记录执行的线程对象到执行器成员变量中
                //可能有人要问了，执行器只有一个，但是每次提交任务那么这个thread字段的线程不是一直在变吗？
                //如果同时提交多个任务的话，哪个线程最后执行，这个thread字段指向的就是谁，但是从netty的代码来看
                //外面的的线程提交过一次之后，后面不会再提交任务到执行器中，会提交也是EventLoop自己的线程
                thread = Thread.currentThread();
                if (interrupted) {
                    thread.interrupt();
                }

                boolean success = false;
                //更新上次执行的时间
                updateLastExecutionTime();
                try {
                    //调用执行器的run方法，这个方法被NioEventLoop重写
                    SingleThreadEventExecutor.this.run();
                    success = true;
                } catch (Throwable t) {
                    logger.warn("Unexpected exception from an event executor: ", t);
                } finally {
                    //如果关闭了通道，退出了服务，执行器状态还是小于正在关闭状态，那么就设置为正在关闭
                    if (state < ST_SHUTTING_DOWN) {
                        state = ST_SHUTTING_DOWN;
                    }

                    // Check if confirmShutdown() was called at the end of the loop.
                    //调用执行器的run方法成功，并且没有发生优雅关闭
                    if (success && gracefulShutdownStartTime == 0) {
                        logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                                SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must be called " +
                                "before run() implementation terminates.");
                    }

                    try {
                        // Run all remaining tasks and shutdown hooks.
                        for (;;) {
                            //确认都已经关闭，也就是里面任务都已经执行完毕了
                            if (confirmShutdown()) {
                                break;
                            }
                        }
                    } finally {
                        try {
                            cleanup();
                        } finally {
                            synchronized (stateLock) {
                                //设置状态为完全停止
                                state = ST_TERMINATED;
                            }
                            threadLock.release();
                            if (!taskQueue.isEmpty()) {
                                logger.warn(
                                        "An event executor terminated with " +
                                                "non-empty task queue (" + taskQueue.size() + ')');
                            }
                            //设置terminationFuture，异步任务已经完成
                            //唤醒阻塞在terminationFuture上面的线程
                            terminationFuture.setSuccess(null);
                        }
                    }
                }
            }
        });
 }
```

执行器的run方法

```
protected void io.netty.channel.nio.NioEventLoop#run() {
        for (;;) {
            //获取老的wakenUp标志，并设置新的标志为false
            oldWakenUp = wakenUp.getAndSet(false);
            try {
                //判断是否存在任务
                //!taskQueue.isEmpty()
                if (hasTasks()) {
                    //调用多路复用器的selectNow方法
                    selectNow();
                } else {
                    //(*1*)
                    select();
                    //如果为true，那么唤醒selector
                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                }
                。。。。。。
        }
    }
    
    //(*1*)
    private void io.netty.channel.nio.NioEventLoop#select() throws IOException {
        Selector selector = this.selector;
        try {
            int selectCnt = 0;
            //获取当前纳秒时间
            long currentTimeNanos = System.nanoTime();
            //从延时队列中获取调度任务，计算时间差，还差多少时间执行
            //(*2*delayNanos)，如果任务有延时任务到了时间，那么selectDeadLineNanos <= currentTimeNanos，delayNanos(currentTimeNanos)计算的值为零或负值。
            long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);
            for (;;) {
                //计算超时时间，用延时差值加0.5ms（四舍五入），然后换算成ms为单位的数值
                long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
                //延时时间小于0，该跳出循环去尝试处理任务了
                if (timeoutMillis <= 0) {
                    //selectCnt表示select了多少次
                    if (selectCnt == 0) {
                        selector.selectNow();
                        selectCnt = 1;
                    }
                    //跳出循环
                    break;
                }
                //等待任务调度延时时间
                int selectedKeys = selector.select(timeoutMillis);
                //自增
                selectCnt ++;
                //发现感兴趣的事件或者外部线程添加了任务，或者有线程要求wakenup，跳出循环
                if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks()) {
                    // Selected something,
                    // waken up by user, or
                    // the task queue has a pending task.
                    break;
                }
                
                //selector重新创建的阀值，默认是512，如果大于零，并且选择的次数已经大于重建阀值
                //那么可能发生了selector死循环
                if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                        selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                    // The selector returned prematurely many times in a row.
                    // Rebuild the selector to work around the problem.
                    logger.warn(
                            "Selector.select() returned prematurely {} times in a row; rebuilding selector.",
                            selectCnt);
                    
                    //重建Selector
                    //(*4*)
                    rebuildSelector();
                    selector = this.selector;

                    // Select again to populate selectedKeys.
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }

                currentTimeNanos = System.nanoTime();
            }
			 //select超过MIN_PREMATURE_SELECTOR_RETURNS（默认三次），打印消息提示，通常是因为延时队列任务延时太久导致的
       		 //一般在1000.5ms之后便会跳出循环
            if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely {} times in a row.", selectCnt - 1);
                }
            }
        } catch (CancelledKeyException e) {
            if (logger.isDebugEnabled()) {
                logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector - JDK bug?", e);
            }
            // Harmless exception - log anyway
        }
    }
    
    //(*2*delayNanos)
    protected long io.netty.util.concurrent.SingleThreadEventExecutor#delayNanos(long currentTimeNanos) {
        //从延时队列中获取一个调用任务
        ScheduledFutureTask<?> delayedTask = delayedTaskQueue.peek();
        //如果为空，那么返回1000000000ns，也就是1s
        if (delayedTask == null) {
            return SCHEDULE_PURGE_INTERVAL;
        }
        //计算时间差
        //(*3*)
        return delayedTask.delayNanos(currentTimeNanos);
    }
    
     //(*3*)
     public long io.netty.util.concurrent.ScheduledFutureTask#delayNanos(long currentTimeNanos) {
        //START_TIME表示当前调度任务创建的时间，deadlineNanos()方法获取的是延时的时间
        //延时时间 - (当前时间 - 任务创建时间)，然后取最大值
        return Math.max(0, deadlineNanos() - (currentTimeNanos - START_TIME));
    }
    
    //(*4*)
    public void io.netty.channel.nio.NioEventLoop#rebuildSelector() {
        //使用执行器的线程执行
        if (!inEventLoop()) {
            execute(new Runnable() {
                @Override
                public void run() {
                    rebuildSelector();
                }
            });
            return;
        }

        final Selector oldSelector = selector;
        final Selector newSelector;

        if (oldSelector == null) {
            return;
        }

        try {
            //开启新的Selector
            //(*5*)
            newSelector = openSelector();
        } catch (Exception e) {
            logger.warn("Failed to create a new Selector.", e);
            return;
        }

        // Register all channels to the new Selector.
        int nChannels = 0;
        for (;;) {
            try {
                //将注册在老的selector的selectionkey注册到新的selector中
                for (SelectionKey key: oldSelector.keys()) {
                    Object a = key.attachment();
                    try {
                        if (key.channel().keyFor(newSelector) != null) {
                            continue;
                        }

                        int interestOps = key.interestOps();
                        key.cancel();
                        key.channel().register(newSelector, interestOps, a);
                        nChannels ++;
                    } catch (Exception e) {
                        logger.warn("Failed to re-register a Channel to the new Selector.", e);
                        if (a instanceof AbstractNioChannel) {
                            AbstractNioChannel ch = (AbstractNioChannel) a;
                            ch.unsafe().close(ch.unsafe().voidPromise());
                        } else {
                            @SuppressWarnings("unchecked")
                            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                            invokeChannelUnregistered(task, key, e);
                        }
                    }
                }
            } catch (ConcurrentModificationException e) {
                // Probably due to concurrent modification of the key set.
                continue;
            }

            break;
        }

        selector = newSelector;

        try {
            // time to close the old selector as everything else is registered to the new one
            //关闭老的
            oldSelector.close();
        } catch (Throwable t) {
            if (logger.isWarnEnabled()) {
                logger.warn("Failed to close the old Selector.", t);
            }
        }

        logger.info("Migrated " + nChannels + " channel(s) to the new Selector.");
    }
    
     //(*5*)
    private Selector io.netty.channel.nio.NioEventLoop#openSelector() {
        final Selector selector;
        try {
            //获取一个多路复用器
            selector = provider.openSelector();
        } catch (IOException e) {
            throw new ChannelException("failed to open a new selector", e);
        }
        //如果禁止keyset后去方式的优化，那么直接返回
        if (DISABLE_KEYSET_OPTIMIZATION) {
            return selector;
        }

        try {
            SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();
            //sun公司提供的Selector实现
            Class<?> selectorImplClass =
                    Class.forName("sun.nio.ch.SelectorImpl", false, ClassLoader.getSystemClassLoader());

            // Ensure the current selector implementation is what we can instrument.
            //如果这个selector不是sun提供的，那么没法确保通过反射获取selectedKeySet
            if (!selectorImplClass.isAssignableFrom(selector.getClass())) {
                return selector;
            }
            //反射获取selector的成员变量
            Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
            Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");

            selectedKeysField.setAccessible(true);
            publicSelectedKeysField.setAccessible(true);
            //然后把netty自己new出的集合设置到这个selector中
            selectedKeysField.set(selector, selectedKeySet);
            publicSelectedKeysField.set(selector, selectedKeySet);
            //执行器的成员变量引用，那么以后select有值就可以直接从执行器的这个selectedKeys集合中获取
            selectedKeys = selectedKeySet;
            logger.trace("Instrumented an optimized java.util.Set into: {}", selector);
        } catch (Throwable t) {
            selectedKeys = null;
            logger.trace("Failed to instrument an optimized java.util.Set into: {}", selector, t);
        }

        return selector;
    }
```

下面继续NioEventLoop#run

```
protected void io.netty.channel.nio.NioEventLoop#run() {
        for (;;) {
            //获取老的wakenUp标志，并设置新的标志为false
            oldWakenUp = wakenUp.getAndSet(false);
            try {
                //判断是否存在任务
                //!taskQueue.isEmpty()
                if (hasTasks()) {
                    //调用多路复用器的selectNow方法
                    selectNow();
                } else {
                    //(*1*)
                    select();
                    //如果为true，那么唤醒selector
                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                }
                //取消的key计数
                cancelledKeys = 0;
                //开始selectedKey的处理时间
                final long ioStartTime = System.nanoTime();
                //是否需要再次selector.selectNow()
                needsToSelectAgain = false;
                //如果执行器的selectedKeys不为空，那么说明使用的优化过后的获取方式
                //selector的选择的key将会存储到这个selectedKeys，这个值在获取selector和重建selector会通过一些判断进行设置
                //但是一般只要是JDK提供的selector实现，都会启动这种优化的策略
                if (selectedKeys != null) {
                    //处理selectedKey
                    processSelectedKeysOptimized(selectedKeys.flip());
                } else {
                    //通用的方式，直接通过selector.selectedKeys()获取
                    processSelectedKeysPlain(selector.selectedKeys());
                }
                //计算处理时间
                final long ioTime = System.nanoTime() - ioStartTime;
                //io比率，默认为50
                final int ioRatio = this.ioRatio;
                
                // (100/ioRatio-1) * ioTime 执行队列中任务超时时间
                runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                //检查执行器状态，是否需要被关闭
                if (isShuttingDown()) {
                    //关闭
                    closeAll();
                    //确认都已经关闭
                    if (confirmShutdown()) {
                        break;
                    }
                }
            } catch (Throwable t) {
                logger.warn("Unexpected exception in the selector loop.", t);

                // Prevent possible consecutive immediate failures that lead to
                // excessive CPU consumption.
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    // Ignore.
                }
            }
        }
    }
```

处理selectedKey，processSelectedKeysOptimized与processSelectedKeysPlain内部处理selectedKey是差不多的，所以只那其中一个进行分析就可以了

```
private void io.netty.channel.nio.NioEventLoop#processSelectedKeysOptimized(SelectionKey[] selectedKeys) {
        for (int i = 0;; i ++) {
            final SelectionKey k = selectedKeys[i];
            //如果没有了SelectionKey，返回null
            if (k == null) {
                break;
            }
            //获取附件，还记得在父通道注册到selector的时候设置的附件是NioServerSocketChannal
            final Object a = k.attachment();

            if (a instanceof AbstractNioChannel) {
                //处理
                //(*1*)
                processSelectedKey(k, (AbstractNioChannel) a);
            } else {
                @SuppressWarnings("unchecked")
                //NioTask是个接口，只是多包了那么一层，可以用于在处理前做些操作，他们最终调用的方法和processSelectedKey是一样的
                NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                processSelectedKey(k, task);
            }
            //是否需要再次select，很显然，前面netty默认设置的就是false
            if (needsToSelectAgain) {
                //重新调用selector的selectNow方法
                selectAgain();
                // Need to flip the optimized selectedKeys to get the right reference to the array
                // and reset the index to -1 which will then set to 0 on the for loop
                // to start over again.
                //
                // See https://github.com/netty/netty/issues/1523
                //如果没有值的话，netty会设置一个null进去
                selectedKeys = this.selectedKeys.flip();
                //重置索引
                i = -1;
            }
        }
}

//(*1*)
private static void io.netty.channel.nio.NioEventLoop#processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        //对于我们设置的Nio通道来说，这里创建的NioUnsafe是NioMessageUnsafe的实例
        final NioUnsafe unsafe = ch.unsafe();
        //如果这个key已经失效，关闭通道
        if (!k.isValid()) {
            // close the channel if the key is not valid anymore
            unsafe.close(unsafe.voidPromise());
            return;
        }

        try {
            //获取感兴趣的事件
            int readyOps = k.readyOps();
            // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
            // to a spin loop
            //netty也会检查readyOps为0的情况（ JDK bug），可以看到netty对接收事件被认为是读取事件
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                //读取
                unsafe.read();
                if (!ch.isOpen()) {
                    // Connection already closed - no need to handle write.
                    //通道关闭，啥也不做
                    return;
                }
            }
            if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
                //写，强制刷新缓冲数据到通道中
                ch.unsafe().forceFlush();
            }
            //连接事件
            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
                // See https://github.com/netty/netty/issues/924
                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT;
                k.interestOps(ops);
                //通知注册在promise上的监听器，针对于NioSocketChannal，触发通道的激活事件
                unsafe.finishConnect();
            }
        } catch (CancelledKeyException e) {
            unsafe.close(unsafe.voidPromise());
        }
}
```
> unsafe.read()

```
public void io.netty.channel.nio.AbstractNioMessageChannel.NioMessageUnsafe#read() {
            assert eventLoop().inEventLoop();
            //检查当前通道是否是自动读取的，如果不是将会取消掉对应key的读事件
            if (!config().isAutoRead()) {
                removeReadOp();
            }

            final ChannelConfig config = config();
            //获取每次读取消息的最大数据
            final int maxMessagesPerRead = config.getMaxMessagesPerRead();
            final boolean autoRead = config.isAutoRead();
            final ChannelPipeline pipeline = pipeline();
            boolean closed = false;
            Throwable exception = null;
            try {
                for (;;) {
                    //读取数据，并存储到readBuf中
                    //(*1*)
                    int localRead = doReadMessages(readBuf);
                    //没有读取到任何信息，break
                    if (localRead == 0) {
                        break;
                    }
                    //如果读取为负数，将会关闭通道
                    if (localRead < 0) {
                        closed = true;
                        break;
                    }
                    //如果超过每次最大读取数据，break，等待下次读取
                    if (readBuf.size() >= maxMessagesPerRead | !autoRead) {
                        break;
                    }
                }
            } catch (Throwable t) {
                exception = t;
            }

            int size = readBuf.size();
            for (int i = 0; i < size; i ++) {
                //触发通道的读事件
                pipeline.fireChannelRead(readBuf.get(i));
            }
            //清理掉readBuf的数据
            readBuf.clear();
            //触发读取完成事件
            pipeline.fireChannelReadComplete();
            //触发通道异常事件
            if (exception != null) {
                if (exception instanceof IOException) {
                    // ServerChannel should not be closed even on IOException because it can often continue
                    // accepting incoming connections. (e.g. too many open files)
                    closed = !(AbstractNioMessageChannel.this instanceof ServerChannel);
                }

                pipeline.fireExceptionCaught(exception);
            }
            //关闭
            if (closed) {
                if (isOpen()) {
                    close(voidPromise());
                }
            }
        }
    }
    
    //(*1*)
    protected int io.netty.channel.socket.nio.NioServerSocketChannel#doReadMessages(List<Object> buf) throws Exception {
        //使用jdk通道接收新的请求
        SocketChannel ch = javaChannel().accept();

        try {
            if (ch != null) {
                //new NioSocketChannel(父通道, 子事件执行器组中轮询一个执行器, ch)
                buf.add(new NioSocketChannel(this, childEventLoopGroup().next(), ch));
                return 1;
            }
        } catch (Throwable t) {
            logger.warn("Failed to create a new channel from an accepted socket.", t);

            try {
                ch.close();
            } catch (Throwable t2) {
                logger.warn("Failed to close a socket.", t2);
            }
        }

        return 0;
    }
```

下面，我们来分析排管，下面是它的类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/b4a9cacf890ad3f25dec27479df29b84.png)

DefaultChannelPipeline的创建

```
public DefaultChannelPipeline(AbstractChannel channel) {
        if (channel == null) {
            throw new NullPointerException("channel");
        }
        this.channel = channel;
        //创建tail处理器
        TailHandler tailHandler = new TailHandler();
        //构建DefaultChannelHandlerContext
        tail = new DefaultChannelHandlerContext(this, null, generateName(tailHandler), tailHandler);
        //头部处理器
        HeadHandler headHandler = new HeadHandler(channel.unsafe());
        ////构建DefaultChannelHandlerContext
        head = new DefaultChannelHandlerContext(this, null, generateName(headHandler), headHandler);

        head.next = tail;
        tail.prev = head;
}
```
然后在看下其中的addLast方法

```
public ChannelPipeline addLast(String name, ChannelHandler handler) {
        return addLast((ChannelHandlerInvoker) null, name, handler);
}
                                |
                                V
public ChannelPipeline addLast(ChannelHandlerInvoker invoker, final String name, ChannelHandler handler) {
        synchronized (this) {
            //检查是否重复添加处理器
            checkDuplicateName(name);
            //构建DefaultChannelHandlerContext
            DefaultChannelHandlerContext newCtx =
                    new DefaultChannelHandlerContext(this, invoker, name, handler);

            addLast0(name, newCtx);
        }

        return this;
}
                                |
                                V
private void addLast0(final String name, DefaultChannelHandlerContext newCtx) {
    //检查是否允许单例的处理器被添加到多个排管，或者是被多次添加到同一排管
    //如果想要将单例的处理器添加到多个排管，需要在类上标注@Sharable
    checkMultiplicity(newCtx);

    DefaultChannelHandlerContext prev = tail.prev;
    newCtx.prev = prev;
    newCtx.next = tail;
    prev.next = newCtx;
    tail.prev = newCtx;

    name2ctx.put(name, newCtx);
    //最终的callHandlerAdded0，为空方法
    callHandlerAdded(newCtx);
}

```

构建DefaultChannelHandlerContext

```
DefaultChannelHandlerContext(
            DefaultChannelPipeline pipeline, ChannelHandlerInvoker invoker, String name, ChannelHandler handler) {
        
        //这个名字如果没有设置的话，将会在调用此构造方法的时候进行生成默认的name
        if (name == null) {
            throw new NullPointerException("name");
        }
        if (handler == null) {
            throw new NullPointerException("handler");
        }

        channel = pipeline.channel;
        this.pipeline = pipeline;
        this.name = name;
        this.handler = handler;
        //计算跳过标记，解析我们handler那些注册，读取，写等事件方法上的@Skip注解
        //感兴趣自个去看，比较简单，不过多的叙述
        skipFlags = skipFlags(handler);

        if (invoker == null) {
            //默认是DefaultChannelHandlerInvoker实例，它关联了执行器
            this.invoker = channel.unsafe().invoker();
        } else {
            this.invoker = invoker;
        }
        
}
```
下面我们看到绑定方法

```
//这个promise就是ServerBootstrap中传递过来的，主线程将阻塞在promise的sync的方法，直到绑定成功或失败
public ChannelFuture io.netty.channel.DefaultChannelPipeline#bind(SocketAddress localAddress, ChannelPromise promise) {
        //调用tail的bind方法
        //(*1*)
        return tail.bind(localAddress, promise);
}

//(*1*)
public ChannelFuture io.netty.channel.DefaultChannelHandlerContext#bind(final SocketAddress localAddress, final ChannelPromise promise) {
        //寻找能够处理MASK_BIND（也就是绑定事件的处理器上下文）
        //(*2*)
        DefaultChannelHandlerContext next = findContextOutbound(MASK_BIND);
        //找到后就开始执行
        next.invoker.invokeBind(next, localAddress, promise);
        return promise;
}

//(*2*) Outbound表示从输出端，对于handler来说就会从后往前找
private DefaultChannelHandlerContext findContextOutbound(int mask) {
        DefaultChannelHandlerContext ctx = this;
        do {
            //从后往前找
            ctx = ctx.prev;
            //找到第一个支持就停止扫描，直接返回
        } while ((ctx.skipFlags & mask) != 0);
        return ctx;
}
```
很显然，目前我们没有给父通道设置自定义的handler，它上面的处理器全是netty自己设置的，其中只有HeadHandler感兴趣

```
public void io.netty.channel.DefaultChannelPipeline.HeadHandler#bind(
                ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise)
                throws Exception {
                //(*1*)
            unsafe.bind(localAddress, promise);
}

//(*1*)
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
            //确保通道是开着的
            if (!ensureOpen(promise)) {
                return;
            }

            // See: https://github.com/netty/netty/issues/576
            //用于检测是否支持广播的地址
            if (!PlatformDependent.isWindows() && !PlatformDependent.isRoot() &&
                Boolean.TRUE.equals(config().getOption(ChannelOption.SO_BROADCAST)) &&
                localAddress instanceof InetSocketAddress &&
                !((InetSocketAddress) localAddress).getAddress().isAnyLocalAddress()) {
                // Warn a user about the fact that a non-root user can't receive a
                // broadcast packet on *nix if the socket is bound on non-wildcard address.
                logger.warn(
                        "A non-root user can't receive a broadcast packet if the socket " +
                        "is not bound to a wildcard address; binding to a non-wildcard " +
                        "address (" + localAddress + ") anyway as requested.");
            }
            //判断是否已经激活
            boolean wasActive = isActive();
            try {
                //(*2*)
                doBind(localAddress);
            } catch (Throwable t) {
                promise.setFailure(t);
                closeIfClosed();
                return;
            }
            //触发通道激活事件
            if (!wasActive && isActive()) {
                invokeLater(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.fireChannelActive();
                    }
                });
            }
            //设置成功值
            promise.setSuccess();
}

//(*2*)
protected void io.netty.channel.socket.nio.NioServerSocketChannel#doBind(SocketAddress localAddress) throws Exception {
        //绑定地址，config.getBacklog()在我们例子中设置的是1024
        javaChannel().socket().bind(localAddress, config.getBacklog());
}
```
执行队列任务

```
protected boolean io.netty.util.concurrent.SingleThreadEventExecutor#runAllTasks(long timeoutNanos) {
        //从延时队列中获取已到时的任务到taskQueue队列中
        fetchFromDelayedQueue();
        //从taskQueue中获取一个任务
        Runnable task = pollTask();
        if (task == null) {
            return false;
        }

        final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
        long runTasks = 0;
        long lastExecutionTime;
        for (;;) {
            try {
                //执行队列任务
                task.run();
            } catch (Throwable t) {
                logger.warn("A task raised an exception.", t);
            }

            runTasks ++;

            // Check timeout every 64 tasks because nanoTime() is relatively expensive.
            // XXX: Hard-coded value - will make it configurable if it is really a problem.
            if ((runTasks & 0x3F) == 0) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                //执行一小段时间，避免过多的时间花在执行taskQueue任务中
                if (lastExecutionTime >= deadline) {
                    break;
                }
            }

            task = pollTask();
            if (task == null) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                break;
            }
        }

        this.lastExecutionTime = lastExecutionTime;
        return true;
    }
```

下面是ServerBootstrap辅助NioServerSocketChannal绑定的时序图

- part1
  ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/c2353f613dcb2031ad0dfebe84011fdd.png)

- part2
  ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/2dd43dd53a96e147ef0348705c644b6f.png)

- part3
  ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/aab060122c957021e3ee7160b9ef56db.png)
