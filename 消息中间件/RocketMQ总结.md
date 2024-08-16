## 一、架构设计
RocketMQ 中文官方文档：https://github.com/apache/rocketmq/blob/master/docs/cn/architecture.md

## 二、总结
### 2.1 发送消息
1、先从本地缓存中获取对应topic发布信息（消息队列列表），如果本地没有那么从nameserver中获取对应topic的发布信息，如果这个topic是不存在的并且配置了 autoCreateTopicEnable=true 那么会使用一个默认的 topic =》TBW102 获取队列列表。

```java
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
		//从本地缓存中获取对应topic的发布信息
        TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
        if (null == topicPublishInfo || !topicPublishInfo.ok()) {
            this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
            //从nameserver获取最新的topic信息
            this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
            topicPublishInfo = this.topicPublishInfoTable.get(topic);
        }
		//存在对应topic的路由信息，直接返回即可
        if (topicPublishInfo.isHaveTopicRouterInfo() || (topicPublishInfo != null && topicPublishInfo.ok())) {
            return topicPublishInfo;
        }
        else {
            
            //第二参数为true，表示使用默认的topic拉去路由信息，默认的topic名为 TBW102 ，
            //这个topic是每个broker都默认存在的topic，它们在与nameserver取得联系之后就会上报自己的路由信息
            this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
            topicPublishInfo = this.topicPublishInfoTable.get(topic);
            return topicPublishInfo;
        }
    }
```

2、获取到topic消息队列列表之后，如果用户没有指定要投递的消息队列，那么默认通过轮询的方式获取队列，投递失败，默认再重试2次，会跳过上次投递失败的brokerName。

```java
public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
		// lastBrokerName 为上次投递失败的broker名
        if (lastBrokerName != null) {
            int index = this.sendWhichQueue.getAndIncrement();
            for (int i = 0; i < this.messageQueueList.size(); i++) {
                int pos = Math.abs(index++) % this.messageQueueList.size();
                MessageQueue mq = this.messageQueueList.get(pos);
                //跳过上次投递失败的broker
                if (!mq.getBrokerName().equals(lastBrokerName)) {
                    return mq;
                }
            }

            return null;
        }
        else {
        	//轮询
            int index = this.sendWhichQueue.getAndIncrement();
            int pos = Math.abs(index) % this.messageQueueList.size();
            return this.messageQueueList.get(pos);
        }
    }
```

3、通过选出的消息队列，获取其所在的master broker（brokerId为0）信息，先从本地缓存获取，如果获取不到再到nameserver中获取，获取到之后构建发送请求信息，然后获取netty通道，如果之前已经建立过，那么直接从本地缓存中获取使用，如果没有建立过，那么建立长连接，发送消息。

```java
public String findBrokerAddressInPublish(final String brokerName) {
		//获取brokername相同的主从broken，这里只获取master
        HashMap<Long/* brokerId */, String/* address */> map = this.brokerAddrTable.get(brokerName);
        if (map != null && !map.isEmpty()) {
            return map.get(MixAll.MASTER_ID);
        }

        return null;
    }
```

4、broker端收到消息之后，如果发现对应的topic是不存在的，那么会自动创建

### 2.2 消费消息
#### 2.2.1 分配队列
1、广播模式将消费对应topic的所有队列，集群模式会根据消费组从对应topic所在的broker获取所有订阅了该topic的消费者，然后根据用户配置的分配策略分配消费队列，默认是平均分配，目前3.4.6这个版本的实现有以下几种：
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/8dcd4314462aa25c7f3c509fabb01f50.png#pic_center)
- AllocateMessageQueueAveragely: 平均分配

```java
public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
                                       List<String> cidAll) {
        //。。。。。。省略部分代码

        List<MessageQueue> result = new ArrayList<MessageQueue>();
        if (!cidAll.contains(currentCID)) {
            log.info("[BUG] ConsumerGroup: {} The consumerId: {} not in cidAll: {}", //
                    consumerGroup, //
                    currentCID,//
                    cidAll);
            return result;
        }
		//获取当前消费者在统一消费组中的排名
        int index = cidAll.indexOf(currentCID);
        //按照消费者数量进行分配时，并不一定能够完全的平均分配，有可能出现除不尽的问题
        //那么会把多余的消息队列挨个分配给排名小于mod的消费者
        int mod = mqAll.size() % cidAll.size();
        int averageSize =
                mqAll.size() <= cidAll.size() ? 1 : (mod > 0 && index < mod ? mqAll.size() / cidAll.size()
                        + 1 : mqAll.size() / cidAll.size());
        int startIndex = (mod > 0 && index < mod) ? index * averageSize : index * averageSize + mod;
        int range = Math.min(averageSize, mqAll.size() - startIndex);
        for (int i = 0; i < range; i++) {
            result.add(mqAll.get((startIndex + i) % mqAll.size()));
        }
        return result;
    }
```
核心思想就是根据消费者数量进行平均分配，如果遇到除不尽的，那么将多余出来的消息队列挨个分配给排名小于余数的消费者。

- AllocateMessageQueueByMachineRoom：根据机房进行消费

```java
public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
                                       List<String> cidAll) {
        List<MessageQueue> result = new ArrayList<MessageQueue>();
        int currentIndex = cidAll.indexOf(currentCID);
        if (currentIndex < 0) {
            return result;
        }
        //先筛选出符合条件的消息队列
        List<MessageQueue> premqAll = new ArrayList<MessageQueue>();
        for (MessageQueue mq : mqAll) {
            String[] temp = mq.getBrokerName().split("@");
            if (temp.length == 2 && consumeridcs.contains(temp[0])) {
                premqAll.add(mq);
            }
        }
        // Todo cid
        //然后进行平局分配
        int mod = premqAll.size() / cidAll.size();
        int rem = premqAll.size() % cidAll.size();
        //开始分配的下标
        int startindex = mod * currentIndex;
        //结束下标
        int endindex = startindex + mod;
        for (int i = startindex; i < endindex; i++) {
            result.add(mqAll.get(i));
        }
        //将余下的消息队列分配给排名大于余数的消费者
        if (rem > currentIndex) {
            result.add(premqAll.get(currentIndex + mod * cidAll.size()));
        }
        return result;
    }
```

首先根据当前消费者配置的消费id筛选出符合条件的消息队列，然后根据符合条件的消息队列个数做平均分配（分配的是mqAll里的），其实这个我没太明白啥操作，知道的可以告知一下，可能高点的版本是不一样的。

- AllocateMessageQueueByConfig：根据配置分配，默认的实现是空实现，用户可以自定义
- AllocateMessageQueueAveragelyByCircle：循环分配，实际上就是轮询分配，你一个我一个这样

```java
public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
                                       List<String> cidAll) {
        //。。。。。。省略部分代码

        List<MessageQueue> result = new ArrayList<MessageQueue>();
        if (!cidAll.contains(currentCID)) {
            log.info("[BUG] ConsumerGroup: {} The consumerId: {} not in cidAll: {}", //
                    consumerGroup, //
                    currentCID,//
                    cidAll);
            return result;
        }

        int index = cidAll.indexOf(currentCID);
        for (int i = index; i < mqAll.size(); i++) {
            if (i % cidAll.size() == index) {
                result.add(mqAll.get(i));
            }
        }
        return result;
    }
```

轮询分配

#### 2.2.2 消费
##### 2.2.2.1 消费方式
RocketMQ 有两种获取消息的方式，分别为拉取和推送，其实在 RocketMQ 中，push 方式其实就是 pull 方式（基于长轮询）。push消费者在启动的时候，会启动一个线程，这个线程专门从队列里获取拉取请求，代码如下：

```java
PullMessageService.run方法
public void run() {
        log.info(this.getServiceName() + " service started");

        while (!this.isStoped()) {
            try {
            	//从 pullRequestQueue 队列中获取请求任务
                PullRequest pullRequest = this.pullRequestQueue.take();
                if (pullRequest != null) {
                    this.pullMessage(pullRequest);
                }
            }
            catch (InterruptedException e) {
            }
            catch (Exception e) {
                log.error("Pull Message Service Run Method exception", e);
            }
        }

        log.info(this.getServiceName() + " service end");
    }
```
当拉取到消息或者没有拉取到消息时，立马往队列里放置拉取请求，用于模拟成实时 push 的效果，为什么要这么做呢？难道不能直接让broker推过来吗？如果要让broker推送过来，消费速度与生产速度可能不匹配，导致消费者消息堆积，另外broker要对订阅过来的消费者在broker端做消费队列的分配与消费者下线重分配的逻辑或者在客户端分配好，链接到broker时定时上报分配信息，然后某个topic的某个队列有消息进来，那么就要主动找到订阅了这个topic并且消息队列一致的那些消费者，最后把消息发给他们。我觉得让broker主动推送的方式有以下缺点：
1. 在broker端做分配这种方式限制了分配策略的自由切换，当然也可以采用客户端上报的方式，但是需要额外的网络消耗，增加维护成本，增加代码复杂度
2. 在客户端做好分配，然后定时上报分配方案，依然需要额外的网络开销与维护成本，编程复杂度的增加
3. 一次性批量推送多少条消息不能自由切换，offset推送的消息是否消费失败之后，消费者依然需要发出pull请求重新消费
4. 消费者消费能力要求高，可能无法跟上生产者生产的速度

##### 2.2.2.2 消费失败重试
假设我们现在使用的消费方式是push，使用的是并发消费 MessageListenerConcurrently。消费者拉取到一批消息后投递到线程池中消费，结果消费失败了，那么就需要消息重试，代码如下：

```java
public void com.alibaba.rocketmq.client.impl.consumer.ConsumeMessageConcurrentlyService#processConsumeResult(//
            final ConsumeConcurrentlyStatus status, //
            final ConsumeConcurrentlyContext context, //
            final ConsumeRequest consumeRequest//
    ) {
    	//记录最后一个消费成功的消息下标
        int ackIndex = context.getAckIndex();

        if (consumeRequest.getMsgs().isEmpty())
            return;

        switch (status) {
        case CONSUME_SUCCESS:
            if (ackIndex >= consumeRequest.getMsgs().size()) {
                ackIndex = consumeRequest.getMsgs().size() - 1;
            }
            int ok = ackIndex + 1;
            //消费失败的消息个数
            int failed = consumeRequest.getMsgs().size() - ok;
            //记录消费成功tps，用于性能指标的监控，可通过Broker查询Consumer内存数据
            this.getConsumerStatsManager().incConsumeOKTPS(consumerGroup,
                consumeRequest.getMessageQueue().getTopic(), ok);
            //记录消费失败tps，用于性能指标的监控，可通过Broker查询Consumer内存数据
            this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup,
                consumeRequest.getMessageQueue().getTopic(), failed);
            break;
        case RECONSUME_LATER:
        	//全部消费失败
            ackIndex = -1;
            this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup,
                consumeRequest.getMessageQueue().getTopic(), consumeRequest.getMsgs().size());
            break;
        default:
            break;
        }

        switch (this.defaultMQPushConsumer.getMessageModel()) {
        case BROADCASTING:
            for (int i = ackIndex + 1; i < consumeRequest.getMsgs().size(); i++) {
                MessageExt msg = consumeRequest.getMsgs().get(i);
                log.warn("BROADCASTING, the message consume failed, drop it, {}", msg.toString());
            }
            break;
        case CLUSTERING:
            List<MessageExt> msgBackFailed = new ArrayList<MessageExt>(consumeRequest.getMsgs().size());
            for (int i = ackIndex + 1; i < consumeRequest.getMsgs().size(); i++) {
                MessageExt msg = consumeRequest.getMsgs().get(i);
                //将处理失败的消息回发给broker，如果回发失败，那么会直接发给重试topic，重试topic的名字为 “%RETRY%” + 消费组名
                //如果还是失败，那么会直接扔进到消费者的调度线程池中，5s之后再重新消费
                boolean result = this.sendMessageBack(msg, context);
                if (!result) {
                    msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
                    msgBackFailed.add(msg);
                }
            }
			//如果回发消息失败，将消费失败的消息扔到消费者的调度线程池中，5s之后再重新消费
            if (!msgBackFailed.isEmpty()) {
                consumeRequest.getMsgs().removeAll(msgBackFailed);

                this.submitConsumeRequestLater(msgBackFailed, consumeRequest.getProcessQueue(),
                    consumeRequest.getMessageQueue());
            }
            break;
        default:
            break;
        }
		//更新偏移
        long offset = consumeRequest.getProcessQueue().removeMessage(consumeRequest.getMsgs());
        if (offset >= 0) {
            this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(),
                offset, true);
        }
    }
```
1、消费者拉取到消息之后，会将消息提交给线程池消费，线程池线程调用用户的消费代码，用户根据需要设置消费成功的ackIndex，没有消费成功的消息将被回发给broker，回发的请求码为 RequestCode.CONSUMER_SEND_MSG_BACK，broker端收到这个请求码的请求之后会会检查重试的次数，如果超过最大的重试次数，那么将被转移到死信队列中，死信队列的 topic 名为 “%DLQ%消费组名”，如果没有超过最大重试次数，那么将消息根据延时级别暂存到 topic 为 “SCHEDULE_TOPIC_XXXX“  的队列中，延时一段时间之后再转移到重试队列中。（消费者启动时，自动订阅了重试topic）

```java
private void copySubscription() throws MQClientException {
        try {
            Map<String, String> sub = this.defaultMQPushConsumer.getSubscription();
            if (sub != null) {
                for (final Map.Entry<String, String> entry : sub.entrySet()) {
                    final String topic = entry.getKey();
                    final String subString = entry.getValue();
                    SubscriptionData subscriptionData =
                            FilterAPI.buildSubscriptionData(this.defaultMQPushConsumer.getConsumerGroup(),//
                                topic, subString);
                    this.rebalanceImpl.getSubscriptionInner().put(topic, subscriptionData);
                }
            }

            if (null == this.messageListenerInner) {
                this.messageListenerInner = this.defaultMQPushConsumer.getMessageListener();
            }

            switch (this.defaultMQPushConsumer.getMessageModel()) {
            case BROADCASTING:
                break;
            case CLUSTERING:
            	//订阅重试topic
                final String retryTopic = MixAll.getRetryTopic(this.defaultMQPushConsumer.getConsumerGroup());
                SubscriptionData subscriptionData =
                        FilterAPI.buildSubscriptionData(this.defaultMQPushConsumer.getConsumerGroup(),//
                            retryTopic, SubscriptionData.SUB_ALL);
                this.rebalanceImpl.getSubscriptionInner().put(retryTopic, subscriptionData);
                break;
            default:
                break;
            }
        }
        catch (Exception e) {
            throw new MQClientException("subscription exception", e);
        }
    }
```

2、如果上一步失败，那么会尝试直接将消息投递到重试topic队列。
3、如果回发消息失败，那么将会被消费失败的消息直接放到消费者的延时调度线程池中，5s后重试。

#### 2.2.3 记录消费进度
目前记录消费进度的实现有两种，LocalFileOffsetStore与RemoteBrokerOffsetStore，广播模式使用LocalFileOffsetStore，持久化时将消费进度持久化到磁盘，集群模式使用RemoteBrokerOffsetStore，持久化时将偏移量存储到broker端。
