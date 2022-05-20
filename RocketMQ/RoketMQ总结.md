# RocketMQ小结

**简介:**

RocketMQ是一款使用java编写的消息中间件，主要提供了消息发布、消息订阅、消费消息的使用功能，方便对业务进行解耦

本文将从以下几个方面介绍RocketMq



- 名词解释
- 架构与业务流程
- 角色功能说明
- 消息类型
- 负载均衡
- 关于RocketMQ的一些使用经验



## 名词解释

### 1.主题（Topic）

主题，代表某一类消息的集合，一个Topic下会有若干个消息，一个消息只属于一个Topic，是消息订阅的基本单位

Topic是逻辑上的概念，每个Topic会有多个消息队列 messageQueue，而这些消息队列又可以分布在多个broker中

### 2.消息模型（Message Model）

RocketMQ主要由 Producer、Broker、Consumer 三部分组成，其中Producer 负责生产消息，Consumer  负责消费消息，Broker 负责存储消息。Broker 在实际部署过程中对应一台服务器，每个 Broker  可以存储多个Topic的消息，每个Topic的消息也可以分片存储于不同的 Broker。Message Queue  用于存储消息的物理地址，每个Topic中的消息地址存储于多个 Message Queue 中。ConsumerGroup 由多个Consumer 实例构成。

### 3.消息（Message）

消息系统所传输信息的物理载体，生产和消费数据的最小单位，每条消息必须属于一个主题。RocketMQ中每个消息拥有唯一的Message ID，且可以携带具有业务标识的Key。系统提供了通过Message ID和Key查询消息的功能。

### 4.标签（Tag）

为消息设置的标志，用于同一主题下区分不同类型的消息。来自同一业务单元的消息，可以根据不同业务目的在同一主题下设置不同标签。标签能够有效地保持代码的清晰度和连贯性。一个消息可以配置多个Tag



### 5.消费者位点（consumerOffset）

这个位点表示消费者消费MessageQueue的消费位置



### 6.代理者位点（brokerOffset）

这个位点表示当前的broker的MessageQueue中已经有的消息数量

用brokerOffset - consumerOffset 可以知道当前消息消费的情况，是否有消息堆积等

## 架构

![](D:\documentary\mdImgs\rocketMQ\1.jpg)

上图表明了RocketMQ的架构图，以及业务流程

大致的业务流程：

- 启动NameServer，NameServer起来后监听端口，等待Broker、Producer、Consumer连上来，相当于一个路由控制中心。
- Broker启动，跟所有的NameServer保持长连接，定时发送心跳包。心跳包中包含当前Broker信息(IP+端口等)以及存储所有Topic信息。注册成功后，NameServer集群中就有Topic跟Broker的映射关系。
- 收发消息前，先创建Topic，创建Topic时需要指定该Topic要存储在哪些Broker上，也可以在发送消息时自动创建Topic。
- Producer发送消息，启动时先跟NameServer集群中的其中一台建立长连接，并从NameServer中获取当前发送的Topic存在哪些Broker上，轮询从队列列表中选择一个队列，然后与队列所在的Broker建立长连接从而向Broker发消息。
- Consumer跟Producer类似，跟其中一台NameServer建立长连接，获取当前订阅Topic存在哪些Broker上，然后直接跟Broker建立连接通道，开始消费消息。



## 角色功能说明

### 1.NameServer:

NameServer的职责是对Broker进行管理以及提供Broker的路由查询功能，类似于Dubbo中的zookeeper

- Broker管理

  NameServer会接收并保存Broker的注册信息。并且每10秒检测注册的Broker的心跳信息，如果一个Broker两分钟内都没有向NameServer发送心跳信息，那么Nameserver就认为这个Broker已经不可用，将其从路由信息中剔除。

- Broker路由信息查询

  NameServer中会保存Broker集群里的每一个Broker的信息，Broker也会向每一个NameServer发送心跳信息时，会将自身的信息以及Topic信息一起发送过去。所以Consumer以及Produce可以通过NameServer获取到整个Broker集群的信息，进行消息的发送与消费。

  NameServer通常也使用集群部署，但是每个NameServer之间不互相通信，当一个NameServer宕机，Broker仍会向其他NameServer发送信息，Produce和Counsumer仍可以动态的感知到整个Broker集群的信息

### 2.Broker

Broker是RocketMQ中的核心角色，负责存储消息、转发消息，可以将Broker理解为RocketMQ本身。

- **消息的存储**

  在Produce发送消息后，Broker在内存中接收完消息，就会对消息进行持久化，此时可以配置Broker的两种持久化策略：

  - 同步刷盘：Broker在消息写入磁盘完成后再返回给Produce确认信息，这种策略不会让消息在Broker中丢失，因为如果在刷盘的过程中Broker挂掉了，Produce没收到确认信息会进行消息重发。
- 异步刷盘：Broker在消息写入完内存后，启动另一个线程将消息持久化到硬盘中，线程启动后立即返回给Produce确认信息。这种策略有可能导致消息丢失。
  
  大多数情况下都是采用异步刷盘的策略
  
  
  
  RockectMQ中消息持久化到本地的硬盘中，采用顺序写的方式进行持久化
  
  *顺序写*：本地硬盘如果采用随机写的方式，会消耗大量时间在磁盘寻址上（找到硬盘哪个位置没被写入，可以写入数据），而 RockectMQ在每次写入文件时，会事先在磁盘申请好一大块空间，当要写入数据时，就不用再去寻址，在最后一个写入数据的位置接着写入即可
  
  
  
  所有Produce发送的消息都存于名为CommitLog的文件中，单个 CommitLog 默认 1G，并且文件名以起始偏移量命名，固定 20 位，不足则前面补 0，比如 00000000000000000000 代表了第一个文件，第二个文件名就是 00000000001073741824（1G的字节大小），表明起始偏移量为 1073741824，以这样的方式命名用偏移量就能找到对应的文件。
  
  
  
- **消息的读取**

  CommitLog中保存了所有的消息，所有的Topic都混杂在一起，如果要每次读取某个Topic的消息都要从头到尾读一遍CommitLog，那么效率是非常差的，所以我们需要一个高效的方式来从CommitLog中读取到需要的消息。

  

  在RocketMQ中，采用的是类似于Mysql的索引的方式来对读取消息，而这个索引的文件就是**Consumerqueue**

  

  在Broker下，有这样的一个目录文件$HOME/store/consumequeue/{topic}/{queueId}/{fileName}

  这里的{topic} 就是我们声明的topic名字，{queueId}就是topic下某个messageQueue的Id，{fileName}就是Consumerqueue的文件名

  

  当Broker接收到一个消息，存入到CommitLog中后，会异步在这个根据这个Topic下的messageId下创建一个consumerqueue索引文件，然后将这个消息在commitLog中的偏移量以及消息的Tag Hash存储到consumerqueue中。

  

  这样当要消费者读取某个Topic下的消息时，向消费者展现的MessageQueue实际上只存储消息在Commit Log的位置信息，Broker会将consumerqueue内记录了数据传入MessageQueue中，然后消费者读取offset去commitLog中读取真正的消息信息。

  ![](D:\documentary\mdImgs\rocketMQ\2.jpg)

  

- **消息过滤**

  RocketMQ的消息过滤是在Broker端进行的。

  首先，消息在被发送时必须要指定一个Topic，发送给有这个Topic的Broker，然后Broker在接收到这条消息后会根据消息的Tag的hashcode进行过滤，发给对应的消费者队列，消费者在消费时会再比较一次Tag，这次比较的是Tag的真实字符，防止因为消息Tag hash冲突引起的错误消费

  - RockMQ还提供SQL92（类似于SQL的表达式）的过滤方式

### 3.Consumer

消息消费的角色，提供了Pull拉消费和Push推送消费，Push的本质还是Pull，Push模式只是对pull模式的一种封装，其本质实现为消息拉取线程在从服务器拉取到一批消息后，然后提交到消息消费线程池后，又“马不停蹄”的继续向服务器再次尝试拉取消息。如果未拉取到消息，则延迟一下又继续拉取。

与Name Server集群中的其中一个节点（随机）建立长连接，定期从Name Server拉取Topic路由信息，并向提供Topic服务的Master Broker、Slave  Broker建立长连接，且定时向Master Broker、Slave Broker发送心跳。

Consumer既可以从Master Broker订阅消息，也可以从Slave Broker订阅消息，订阅规则由Broker配置决定。



### 4.Produce

消息发布的角色，负责生产消息，一般由业务系统负责生产消息。

与Name Server集群中的其中一个节点（随机）建立长链接，定期从Name Server读取Topic路由信息，并向提供Topic服务的Master Broker建立长链接，且定时向Master Broker发送心跳。                     

一个消息生产者会把业务应用系统里产生的消息发送到broker。RocketMQ提供多种发送方式，同步发送、异步发送、顺序发送、单向发送。同步和异步方式均需要Broker返回确认信息，单向发送不需要。

支持分布式集群方式部署，Producer通过MQ的负载均衡模块选择相应的Broker集群队列进行消息投递。



## 各种类型消息的使用

### 1.顺序消息

RocketMQ提供顺序消息，利用队列先进先出的特性，保证消费者可以按照某种顺序消费消息



使用顺序消息需要指定Produce将消息发送至同一个指定的MessageQueue，然后消费者也要指定顺序消费同一个MessageQueue，这样才能保证消息能够顺序消费



- 如果没有指定Produce发送至同一个MessageQueue，那么后到的消息有可能到其他的MessageQueue里，而有可能被这个MessageQueue对应的消费者先消费了

- 如果没有指定Counsumer顺序消费，那么有可能订单的下单到Consumer消费，但是由于并发问题，支付消息没有抢到CPU，而发货优先被Consumer消费了



### 2.广播消息

如果消息是广播消息，那么Broker在将消息发送给消费者时，会同时向所有订阅了这个Topic的消费者都发送一份同样的消息



### 3.延迟消息

RocketMQ支持将一个消息延迟一段时间后发送，这个消息在Broker中将会放入到一个延时队列中，当时间到了以后再讲这条消息从队列中取出放到所属Topic的消息队列中

RoketMQ支持16个级别的延迟消息，最大可以延迟两个小时



### 4.事务消息

RocketMQ提供事务消息来确保和消息发送相关的功能，可以正确的按照流程执行

![](D:\documentary\mdImgs\rocketMQ\3.jpg)

事务消息的执行流程：

1. 消息发送方先发送一个半事务消息给服务端

2. 服务端将消息持久化后，返回给发送方ACK确认消息，确认消息已经持久化成功，此时消息的状态为UN_KONOW，不会推送给订阅方，这两个对RokectMQ的使用者是不感知的

3. 消息发送方开始执行本地事务

4. 执行完成(成功或者失败)后向服务端提交半事务消息的状态，将消息推送给订阅方。或者提交回滚状态，这条消息将不会发送

   1. 如果在4中服务端长时间没有收到发送方的commit或者rollback，那么服务端会主动去向发送方回查事务状态（调用RocketMQ提供的check方法）

   2. 消息发送方检查本地事务状态，返回相应的事务状态给服务端

      服务端不会一直回查消息，当回查次数超过15（可配置）次时，会默认回滚这条消息

以一个场景举例：

​	当用户支付完订单后，需要让该用户得到奖励积分5，那么，当我们的系统收到用户支付完成这个请求后，在内存中构建好当前的数据，就向服务端发送一个用户加5积分的事务消息，然后执行用户支付完订单后的发货等步骤，执行完成后向Broker发送事务消息提交状态，Broker再将这条消息推送给积分系统，增加用户积分





## 负载均衡

### 1.Produce的负载均衡

Produce在发送消息时，会将根据Topic来获取到Top的路由信息，然后采用轮训取模的方式向Topic下的每一个messagequeue发送消息，因为message会分布在不同的broker上，达到了负债均衡的效果。发送消息有一个sendLatencyFaultEnable开关变量，如果这个变量为true的话，如果Produce在发送给Broker消息失败时，会记录下这个Broker，并在一段时间内不往这个Broker发送消息

![](D:\documentary\mdImgs\rocketMQ\4.jpg)





### 2.Consumer的负载均衡

在消费者集群模式下，每条消息只需要投递到订阅这个topic的Consumer Group下的一个实例即可

每当消费者数量有变更就会触发一次所有消费者的负载均衡，这时候会按照消息队列的数量和消费者的数量平均分配给每个消费者消息队列，达到最佳的预期效果是：每个消息队列都有消费者消费，并且保证每个消息队列不会有多个消费者一起消费



正常的负载均衡应该是由处于中间位置的角色去进行负载均衡，但是RocketMQ的负载均衡是在消费者端去进行的，只要能每个消费者能知道自己是谁，以及每个消息者获取到要进行均衡的数据是一样的，那么消费者端是可以进行负载均衡的

大致的负载均衡过程：

1. 每个消费者询问broker，最后都拿到一样的消费者列表、消息队列列表。

2. 每个消费者都知道自己的名称（如ClientA），这个名称肯定在#1中列表里面的消费者在线列表中

3. 针对#1的视图，对消费者列表和队列列表做一个排序，保证每个消费者看到的列表顺序都是一样的。

4. 每个消费者都调用同样的算法分配函数，函数输出的是本消费者应该分配的队列列表。



RocketMQ主要提供两种分配策略：

- AllocateMessageQueueAveragely：平均分配策略，这个策略是RocketMQ默认的分配策略

  ![](D:\documentary\mdImgs\rocketMQ\5.jpg)

- AllocateMessageQueueAveragely：环形分配策略，也是给每个Consumer平均分配消息队列，不过是按照环形的顺序一个取一个平均分配

  ![](D:\documentary\mdImgs\rocketMQ\6.jpg)

- AllocateMessageQueueConsistentHash：一致性哈希策略

  将Consumer和消息队列的hash 以环的形式进行排序，每个消费者分配到自己hash后面的消息队列

  ![](D:\documentary\mdImgs\rocketMQ\7.jpg)