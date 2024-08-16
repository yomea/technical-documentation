## 一、类介绍

- NioEventLoopGroup：用于创建Nio socket程序的事件轮询组，每个事件轮询都会一个Selector对象
- ServerBootstrap：构建服务通道，设置tcp参数，绑定端口的辅助类
- ServerBootstrapAcceptor：服务通道使用，用于接收请求，轮询EventLoop，注册通道到对应的selector上，并设置通道参数，如果有ChannelInitializer的话还会设置客户端通道处理器
- NioServerSocketChannel：netty封装的服务socket通道，用于接收请求
- ChannelPromise：继承了JDK的Future，当然netty自己也有一个future，增强了一些功能，比如增加了Future监听器
- NioSocketChannel：netty封装客户端通道
- ChannelPipeline：通道排管，当发生通道的读写等事件时，像流水一样从管道中流过，管道中可以设置管道阀（通道处理器）做些过滤操作
- ChannelHandlerContext：封装ChannelHandler，是责任链条上的一个个链子（也可比喻为管道中管道阀），可以携带额外的属性，SkipFlag等信息，在创建它的时候，会解析ChannelHandler方法上的@Skip注解
- ChannelHandler：如果把ChannelPipeline比喻成排管，那么ChannelHandlerContext就是排管上管道阀，而ChannelHandler就是管道阀中起过滤作用的过滤网
- NioByteUnsafe：用于处理客户端读写事件

## 二、大致流程

### 2.1 服务端流程

创建Boss事件轮询组，通过ServerBootstrap辅助类创建NioServerSocketChannel通道，并为其设置tcp参数，通道处理等

-》初始化NioServerSocketChannel后，再进行注册，从boss事件轮询组中轮询一个事件轮询对象（EventLoop），将当前通道注册到EventLoop持有的Selector对象中，注册成功后触发通道的注册事件，此时会调用通道处理链

-》绑定端口，绑定成功后触发通道激活事件

netty的注册和绑定都是通过线程池来处理的，注册和绑定都是包装成一个任务设置到NioEventLoop的任务队列中，少部分代码如下：

```java
 private void io.netty.channel.nio.NioEventLoop#select() throws IOException {
    Selector selector = this.selector;
    try {
        int selectCnt = 0;
        //获取当前纳秒时间
        long currentTimeNanos = System.nanoTime();
        //从延时队列中获取调度任务，计算时间差，还差多少时间执行
        //如果任务有延时任务到了时间，那么selectDeadLineNanos <= currentTimeNanos，delayNanos(currentTimeNanos)计算的值为零或负值。
        long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);
        for (;;) {
            //计算超时时间，用延时差值加0.5ms（四舍五入），然后取整，换算成ms为单位的数值
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
            //看了下netty4的实现，在netty4中，这里应该还有一段代码逻辑，那段代码逻辑是用于判断select是否有等待  timeoutMillis ，如果等待了selectCnt置为1 
            //那么可能发生了selector死循环
            if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                    selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                // The selector returned prematurely many times in a row.
                // Rebuild the selector to work around the problem.
                logger.warn(
                        "Selector.select() returned prematurely {} times in a row; rebuilding selector.",
                        selectCnt);
                
                //重建Selector
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
```


在没有设置延时任务的情况下，select的timeout时间为1000.5ms，每次select都会计数，如果进入select循环的时间与当前时间的差值为负值时（或者任务队列中设置添加了任务，或者被要求wakeup），会跳出循环，除此之外
netty中设置了一个轮询阈值，如果出现jdk的selector bug，它的循环次数达到了512（默认值，可以通过系统参数配置），就重新创建selector

netty在NioEventLoop#processSelectedKey中处理SelectionKey.OP_ACCEPT事件
对应的unsafe实现为NioMessageUnsafe，创建NioSocketChannel，从工作事件轮询组中轮询一个轮询器，将NioSocketChannel注册到它的Selector中，然后触发通道read事件（通常我们没有给父通道设置通道处理器，只有netty自己设置的一个叫ServerBootstrapAcceptor的通道处理器，这个通道处理器在接收客户端请求后触发子通道的注册事件，通常是ChannelInitializer，用于添加其他的子通道处理器），处理完当前通道能读取到的数据后，再触发readComplete事件，如果发生错误，将触发通道的异常处理事件


通道处理会被包装成ChannelHandlerContext，多个之间通过责任链的模式进行调用，通道处理上的每个方法都可以写上@Skip注释，表示跳过某个方法，不进行处理。


优雅关闭：调用每一个事件轮询的关闭方法，关闭通道，取消延时任务队列中还未处理的任务，尽量处理还可以select到的事件，尽量将任务队列的任务处理掉，关闭线程，如果关闭超时将强行退出。

三、池化

PoolChunk：使用平衡二叉树，不包括起始点，默认11层，通过伙伴内存分配法，将内存在相同层数上进行平均分配

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/5c6e97c21f335383ae151ddb6f5801ac.png)


一直分配到第11层，每个节点大小为8k，当用户申请大于8k的内存时从这颗树上进行分配，如果用户输入的内存大小不是2的倍数的，那么会自动调整为2的倍数去分配，被占用的节点被标记为12（第12层，一个不存在的层）

定位层数，通过计算公式：maxOrder - (log2(用户输入值调整后的值) - 13)，这里的maxOrder是11，所以netty将被分配的节点设置为12。12是不可能被访问到的

PoolSubpage：这个类被划分成了两个类型，一个tiny，另一个是small
- tiny表示申请小于512的内存时使用

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ed3f3e3facd19ef3750d504edaa43b2c.png)


从图中可以看到，这个tiny是一个32个元素PoolSubpage数组，从1开始，相邻元素之间以16字节递增。

每个内存页的大小为8k，而下标为1的内存页被划分为16字节的内存块，那么下标为1的元素为一个可以分配8192 / 16 = 512 个16字节的PoolSubpage元素，分配完之后，PoolSubpage会从链表中移除，如果还需要分配新的16字节的空间，那么会从PoolChunk中再划分一个8k的空间构建成PoolSubpage进行初始化加入到链表中，释放时，会放入ThreadLOCALcahe
- small表示申请大于512小于8k的内存时使用

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/c3bc3ecca9bdd97c40d068f2a34b86b7.png)


从图中可以看到这个数组长度为4，从512字节开始，后面元素的子页大小都是 512 * 2^(下标)字节

