[toc]

## 20 | RocketMQ Producer 源码分析：消息生产的实现过程

1.  下载源码

    -   ```bash
        
        git clone git@github.com:apache/rocketmq.git
        cd rocketmq
        git checkout release-4.5.1
        ```

2.  客户端是一个单独的 Module，在 rocketmq/client 目录中。

3.  分析源代码

    -   搞清楚实现原理
    -   发现代码中优秀设计和巧妙构思
    -   学习、总结方法

### 从单元测试看 Producer API 的使用

1.  先看下**单元测试用例**。看看 Producer 提供哪些 API，更重要的是了解这些 API 应该如何使用。

2.  Producer 所有的测试用例都在同一个测试类 “org.apache.rocketmq.client.producer.DefaultMQProducerTest”，这个测试类的主要测试方法如下：

    >   init	# 初始化
    >   terminate	#  结束销毁
    >   testSendMessage_ZeroMessage
    >   testSendMessage_NoNameSrv
    >   testSendMessage_NoRoute
    >   testSendMessageSync_Success
    >   testSendMessageSync_WithBodyCompressed
    >   testSendMessageAsync_Success
    >   testSendMessageAsync
    >   testSendMessageAsync_BodyCompressed
    >   testSendMessageSync_SuccessWithHook

3.  Producer 入口类为 “org.apache.rocketmq.client.producer.DefaultMQProducer”，几个核心类和接口如下

    -   ![img](imgs/ee719ca65c6fb1d43c10c60512913209.png)
    -   这里 RocketMQ 使用一个设计模式：门面模式
        -   MQProducer 就是模式中的门面。
        -   DefaultMQProducer 实现了接口 MQProducer。
        -   DefaultMQProducerImpl 实现了 Producer 大部分业务逻辑
    -   MQAdmin 定义了一些元数据管理的方法，在消息发送过程中会用到。

### 启动过程

1.  从测试用例的 init()：

    -   ```java
        
          @Before
          public void init() throws Exception {
              String producerGroupTemp = producerGroupPrefix + System.currentTimeMillis();
              producer = new DefaultMQProducer(producerGroupTemp);
              producer.setNamesrvAddr("127.0.0.1:9876");
              producer.setCompressMsgBodyOverHowmuch(16);
        
              //省略构造测试消息的代码
        
              producer.start();
        
              //省略用于测试构造mock的代码
          }
        ```

2.  DefaultMQProducer#start() 直接调用 DefaultMQProducerImpl#start() 方法：

    -   ```java
        
        public void start(final boolean startFactory) throws MQClientException {
            switch (this.serviceState) {
                case CREATE_JUST:
                    this.serviceState = ServiceState.START_FAILED;
        
                    // 省略参数检查和异常情况处理的代码
        
                    // 获取MQClientInstance的实例mQClientFactory，没有则自动创建新的实例
                    this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQProducer, rpcHook);
                    // 在mQClientFactory中注册自己
                    boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
                    // 省略异常处理代码
        
                    // 启动mQClientFactory
                    if (startFactory) {
                        mQClientFactory.start();
                    }
                    this.serviceState = ServiceState.RUNNING;
                    break;
                case RUNNING:
                case START_FAILED:
                case SHUTDOWN_ALREADY:
                    // 省略异常处理代码
                default:
                    break;
            }
            // 给所有Broker发送心跳
            this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
        }
        ```

    -   这里，RocketMQ 使用 serviceState 来记录和管理自身的服务状态，这实际上是**状态模式**的变种实现。

    -   使用分支流程（switch-case）来实现不同状态下的不同行为。

    -   在设计状态时，需**注意**

        -   一是，不仅要设计正常状态，还要设计中间状态和异常状态。
        -   二是，将这些状态间的转换路径考虑清楚，并在进行状态转换的时候，检查上一个状态是否能转换到下一个状态。

3.  启动过程：

    -   通过一个单例模式的 MQClientManager 获取 MQClientInstance 的实例 mQClientFactory，没有则自动创建新的实例。
    -   在 mQClientFactory 中注册自己
    -   启动 mQClientFactory
    -   给所有 Broker 发送心跳

4.  每个客户端对应类 MQClientInstance 的一个实例。这个实例维护着客户端的大部分状态信息。MQClientInstance#start() 代码

    -   ```java
        
        // 启动请求响应通道
        this.mQClientAPIImpl.start();
        // 启动各种定时任务
        this.startScheduledTask();
        // 启动拉消息服务
        this.pullMessageService.start();
        // 启动Rebalance服务
        this.rebalanceService.start();
        // 启动Producer服务
        this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
        ```

5.  几个重要的类

    -   **DefaultMQProducerImpl**：Producer 的内部实现类，大部分 Producer 的业务逻辑，也就是发消息的逻辑，都在这个类中。
    -   **MQClientInstance**：这个类中封装了客户端一些通用的业务逻辑，无论是 Producer 还是 Consumer，最终需要与服务端交互时，都需要调用这个类中的方法；
    -   **MQClientAPIImpl**：这个类中封装了客户端服务端的 RPC，对调用者隐藏了真正网络通信部分的具体实现；
    -   **NettyRemotingClient**：RocketMQ 各进程之间网络通信的底层实现类。

### 消息发送过程

1.  Producer 接口 MQProducer 中方法可以分成三类：

    -   单向发送（Oneway）：发送返回，不处理响应，
    -   同步发送（Sync）：发送后，等待响应
    -   异步发送（Async）：发送立即返回，在回调中处理响应

2.  异步发送的实现方法"DefaultMQProducerImpl#send()"（对应源码中的 1132 行）

    -   ```java
        
        @Deprecated
        public void send(final Message msg, final MessageQueueSelector selector, final Object arg, final SendCallback sendCallback, final long timeout)
            throws MQClientException, RemotingException, InterruptedException {
            final long beginStartTime = System.currentTimeMillis();
            ExecutorService executor = this.getAsyncSenderExecutor();
            try {
                executor.submit(new Runnable() {
                    @Override
                    public void run() {
                        long costTime = System.currentTimeMillis() - beginStartTime;
                        if (timeout > costTime) {
                            try {
                                try {
                                    sendSelectImpl(msg, selector, arg, CommunicationMode.ASYNC, sendCallback,
                                        timeout - costTime);
                                } catch (MQBrokerException e) {
                                    throw new MQClientException("unknownn exception", e);
                                }
                            } catch (Exception e) {
                                sendCallback.onException(e);
                            }
                        } else {
                            sendCallback.onException(new RemotingTooMuchRequestException("call timeout"));
                        }
                    }
        
                });
            } catch (RejectedExecutionException e) {
                throw new MQClientException("exector rejected ", e);
            }
        }
        ```

3.  方法 sendSelectImpl() 的实现

    -   ```java
        
        // 省略部分代码
        MessageQueue mq = null;
        
        // 选择将消息发送到哪个队列（Queue）中
        try {
            List<MessageQueue> messageQueueList =
                mQClientFactory.getMQAdminImpl().parsePublishMessageQueues(topicPublishInfo.getMessageQueueList());
            Message userMessage = MessageAccessor.cloneMessage(msg);
            String userTopic = NamespaceUtil.withoutNamespace(userMessage.getTopic(), mQClientFactory.getClientConfig().getNamespace());
            userMessage.setTopic(userTopic);
        
            mq = mQClientFactory.getClientConfig().queueWithNamespace(selector.select(messageQueueList, userMessage, arg));
        } catch (Throwable e) {
            throw new MQClientException("select message queue throwed exception.", e);
        }
        
        // 省略部分代码
        
        // 发送消息
        if (mq != null) {
            return this.sendKernelImpl(msg, mq, communicationMode, sendCallback, null, timeout - costTime);
        } else {
            throw new MQClientException("select message queue return null.", null);
        }
        // 省略部分代码
        ```

    -   方法 sendSelectImpl() 中主要的功能就是选定要发送的队列，然后调用方法 sendKernelImpl() 发送消息。

    -   选择哪个队列发送由 MessageQueueSelector#select 方法决定。这里 RocketMQ 使用**策略模式**，来解决不同场景下需要使用不同的队列选择算法问题。

4.  sendKernelImpl()

    -   主要功能就是构建发送消息的头 RequestHeader 和上下文 SendMessageContext。
    -   调用 MQClientAPIImpl#sendMessage() ，将消息发送给队列所在的 Broder。

5.  MQClientAPIImpl

    -   远程调用封装类，完成后续序列化和网络传输等。

### 小结

1.  本节分析 RocketMQ 客户端消息生产的实现过程。
    -   包括 Producer 初始化和发送消息的主流程。
2.  Producer 启动，在 MQClientInstance 这个类中统一启动。
3.  RocketMQ 分了三种发送方式：
    -   单向
    -   同步
    -   异步
4.  几个重要的业务逻辑实现类
    -   DefaultMQProducerImpl 封装了大部分 Producer 的业务逻辑
    -   MQClientInstance 封装了客户端一些通用的业务逻辑
    -   MQClientAPIImpl 封装了客户端与服务端的 RPC
    -   NettyRemotingClient 实现了底层网络通信

