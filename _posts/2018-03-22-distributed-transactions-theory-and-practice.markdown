---
layout: post
title:  "分布式事物"
categories: distributed-system
---

## 基本事物ACID

 - 原子性：all or nothing。一个原子性的事物系统必须保证无论在系统崩溃，出错甚至断电的情况下，都要保证其原子性，即事务中包含的各项操作必须全部成功执行或者全部不执行。

 - 一致性：state validation。每次事物的成功执行，都将数据库从一个合法状态转变为另一个合法状态，这些合法的状态检查包括主键约束，外键约束,触发器等。例如，如果银行始终遵循着"银行账号必须保持正态平衡"的原则，那么银行系统的状态就是一致的。考虑一个转账例子，在取钱的过程中，账户会出现负态平衡，在事务结束之后，系统又回到一致的状态。这样，系统的状态对于客户来说，始终是一致的。

 - 持久性：durable。事物的提交会让数据持久化，并且不会受到系统崩溃，出错甚至断电的影响。

 - 隔离性：isolation。使得并发执行的事务，彼此无法看到对方的中间状态。保证了并发执行的事务顺序执行，而不会导致系统状态不一致。


**一致性原理**

我们使用SQL Server来举例，我们知道我们在使用 SQL Server 数据库是由两个文件组成的，一个数据库文件和一个日志文件，通常情况下，日志文件都要比数据库文件大很多。数据库进行任何写入操作的时候都是要先写日志的，同样的道理，我们在执行事务的时候数据库首先会记录下这个事务的redo操作日志，然后才开始真正操作数据库，在操作之前首先会把日志文件写入磁盘，那么当突然断电的时候，即使操作没有完成，在重新启动数据库时候，数据库会根据当前数据的情况进行undo回滚或者是redo前滚，这样就保证了数据的强一致性。

当我们的单个数据库的性能产生瓶颈的时候，我们可能会对数据库进行分区，这里所说的分区指的是物理分区，分区之后可能不同的库就处于不同的服务器上了，这个时候单个数据库的ACID已经不能适应这种情况了，而在这种ACID的集群环境下，再想保证集群的ACID几乎是很难达到，或者即使能达到那么效率和性能会大幅下降，最为关键的是再很难扩展新的分区了，这个时候如果再追求集群的ACID会导致我们的系统变得很差，这时我们就需要引入一个新的理论原则来适应这种集群的情况，就是**CAP原则**或者叫**CAP定理**

## CAP定理

 - 一致性(Consistency) ： 客户端知道一系列的操作都会同时发生(生效)
 - 可用性(Availability) ： 每个操作都必须以可预期的响应结束
 - 分区容错性(Partition tolerance) ： 即使出现单个组件无法可用,操作依然可以完成

具体地讲在分布式系统中，在任何数据库设计中，一个Web应用至多只能同时支持上面的两个属性。显然，任何横向扩展策略都要依赖于数据分区。因此，设计人员必须在一致性与可用性之间做出选择。
在分布式系统中，我们往往追求的是可用性，它的重要程序比一致性要高，那么如何实现高可用性呢？ 前人已经给我们提出来了另外一个理论，就是BASE理论，它是用来对CAP定理进行进一步扩充的。BASE理论指的是：

 - Basically Available（基本可用）
 - Soft state（软状态）
 - Eventually consistent（最终一致性）

BASE理论是对CAP中的一致性和可用性进行一个权衡的结果，理论的核心思想就是：我们无法做到强一致，但每个应用都可以根据自身的业务特点，采用适当的方式来使系统达到最终一致性（Eventual consistency）。

## 分布式事物解决方案

### 2PC（Two-phase Commit）

两段提交，又叫`XA Transactions`,是一个两段提交的协议，将数据库事物的提交分成两个阶段：

 - 第一阶段：事务协调器要求每个涉及到事务的数据库预提交(precommit)此操作，并反映是否可以提交.
 - 第二阶段：事务协调器要求每个数据库提交数据

具体实现如下图所示：

![](/assets/images/2018/1.1pc-simple.png)

二阶段提交算法的成立基于以下假设：

 - 所有节点不会永久性损坏，即使损坏后仍然可以恢复。
 - 所有节点都采用WAL（Write Ahead Logging），日志写入后就被保存在可靠的存储设备上。
 - 所有节点上的本地事物即使机器crash也可以从WAL日志上恢复。

二阶段提交算法回滚流程：

![](/assets/images/2018/1.2pc-rollback-flow.png)

二阶段提交算法提交流程

![](/assets/images/2018/1.3pc-commit-flow.png)

二阶段提交缺点：

 - **同步阻塞**。事物执行过程中，所有参与节点都是事物阻塞的，当参与者占有资源时，其他访问相关资源的进程也处于阻塞状态，参与者对资源的释放必须等到事物结束，因此二阶段提交会耗费更多的时间，资源冲突的概率增加，不利于并发场景。

 - **单点故障**。协调者单点一旦出现故障会导致事物操作无法运行。

 - **数据不一致** 在提交阶段，如果出现网络故障，只有部分参与者收到commit消息，则整个分布式系统就出现了数据不一致性。


### TCC（Try/Confirm/Cancel）

TCC型事务（Try/Confirm/Cancel）可以归为补偿型。
补偿型的例子，在一个长事务（ long-running ）中 ，一个由两台服务器一起参与的事务，服务器A发起事务，服务器B参与事务，B的事务需要人工参与，所以处理时间可能很长。如果按照ACID的原则，要保持事务的隔离性、一致性，服务器A中发起的事务中使用到的事务资源将会被锁定，不允许其他应用访问到事务过程中的中间结果，直到整个事务被提交或者回滚。这就造成事务A中的资源被长时间锁定，系统的可用性将不可接受。

在TCC中，还是上面的例子，服务器A的事务如果执行顺利，那么事务A就先行提交，如果事务B也执行顺利，则事务B也提交，整个事务就算完成。但是如果事务B执行失败，事务B本身回滚，这时事务A已经被提交，所以需要执行一个补偿操作，将已经提交的事务A执行的操作作反操作，恢复到未执行前事务A的状态。这样的SAGA事务模型，是牺牲了一定的隔离性和一致性的，但是提高了long-running事务的可用性。

因此，TCC的第一阶段的预提交其实都已经是数据库层面真正的提交，而第二阶段的提交操作根据各个参与者而定，甚至可以什么都不做，也就是说只要有数据库的操作，都是实际的提交。回滚操作也只是一个反向的操作，即insert的delete回去，delete的insert回去，update的重新update回去。TCC不存在XA规范中的数据库层的“预提交”操作，这种操作是在应用层上的模拟。

TCC实现方案有如下特点：

 - 最终一致性。事物过程中可能会出现不一致的情况，但经过系统恢复可以让事物达到最终一致性。
 - 协议简单。定义类似2PC的两段提交接口，业务系统只需要实现对应的接口就可以使用TCC的事物功能。
 - 协议无关。在SOA(Service-oriented Architecture)架构中，多个数据库操作往往封装成服务，服务之间通过rpc通信。TCC事物构建在SOA架构上，于底层协议无关。
 - 底层事物无关。无论是关系型数据库还是非关系型数据库，只要将操作封装成TCC的参与者即可接入到TCC的事物范围。


### 事物消息

2个账号，分布处于2个不同的DB，或者说2个不同的子系统里面，A要扣钱，B要加钱，如何保证原子性？

一般的思路都是通过消息中间件来实现“最终一致性”：A系统扣钱，然后发条消息给中间件，B系统接收此消息，进行加钱。

但这里面有个问题：A是先update DB，后发送消息呢？ 还是先发送消息，后update DB？

假设先update DB成功，发送消息网络失败，重发又失败，怎么办？ 
假设先发送消息成功，update DB失败。消息已经发出去了，又不能撤回，怎么办？

所以，这里下个结论： 只要发送消息和update DB这2个操作不是原子的，无论谁先谁后，都是有问题的。

**错误的方案**

我可以把“发送消息”这个网络调用和update DB放在同1个事务里面，如果发送消息失败，update DB自动回滚。这样不就保证2个操作的原子性了吗？

这个方案看似正确，其实是错误的，原因有2：

（1）网络的2将军问题：发送消息失败，发送方并不知道是消息中间件真的没有收到消息呢？还是消息已经收到了，只是返回response的时候失败了？

如果是已经收到消息了，而发送端认为没有收到，执行update db的回滚操作。则会导致A账号的钱没有扣，B账号的钱却加了。

（2）把网络调用放在DB事务里面，可能会因为网络的延时，导致DB长事务。严重的，会block整个DB。这个风险很大。

**方案1——业务逻辑解决**

假设消息中间件没有提供“事务消息”功能，比如你用的是Kafka。那如何解决这个问题呢？

解决方案如下： 
 - Producer端准备1张消息表，把update DB和insert message这2个操作，放在一个DB事务里面。
 - 准备一个后台程序，源源不断的把消息表中的message传送给消息中间件。失败了，不断重试重传。允许消息重复，但消息不会丢，顺序也不会打乱。
 - Consumer端准备一个判重表。处理过的消息，记在判重表里面。实现业务的幂等。但这里又涉及一个原子性问题：如果保证消息消费 + insert message到判重表这2个操作的原子性？消费成功，但insert判重表失败，怎么办？

 **方案2 – RocketMQ 事务消息**

 事物消息将消息发送分为两个阶段：Prepare阶段和确认阶段

  - 发送prepare消息
  - update db
  - 根据update db返回的结果成功还是失败，confirm或者cancel prepare消息。

可能有人会问了，前2步执行成功了，最后1步失败了怎么办？这里就涉及到了RocketMQ的关键点：RocketMQ会定期（默认是1分钟）扫描所有的Prepared消息，询问发送方，到底是要确认这条消息发出去？还是取消此条消息？所以生产方需要实现一个check接口，RocketMQ会根据发送端设置的策略来决定是回滚还是继续发送确认消息。这样就保证了消息发送与本地事务同时成功或同时失败。

{% highlight java %}
// 也就是上文所说的，当RocketMQ发现`Prepared消息`时，会根据这个Listener实现的策略来决断事务
TransactionCheckListener transactionCheckListener = new TransactionCheckListenerImpl();
// 构造事务消息的生产者
TransactionMQProducer producer = new TransactionMQProducer("groupName");
// 设置事务决断处理类
producer.setTransactionCheckListener(transactionCheckListener);
// 本地事务的处理逻辑，相当于示例中检查Bob账户并扣钱的逻辑
TransactionExecuterImpl tranExecuter = new TransactionExecuterImpl();
producer.start()
// 构造MSG，省略构造参数
Message msg = new Message(......);
// 发送消息
SendResult sendResult = producer.sendMessageInTransaction(msg, tranExecuter, null);
producer.shutdown();


public TransactionSendResult sendMessageInTransaction(.....)  {
    // 逻辑代码，非实际代码
    // 1.发送消息
    sendResult = this.send(msg);
    // sendResult.getSendStatus() == SEND_OK
    // 2.如果消息发送成功，处理与消息关联的本地事务单元
    LocalTransactionState localTransactionState = tranExecuter.executeLocalTransactionBranch(msg, arg);
    // 3.结束事务
    this.endTransaction(sendResult, localTransactionState, localException);
}
{% endhighlight %}

对比方案2和方案1，RocketMQ最大的改变，其实就是把“扫描消息表”这个事情，不让业务方做，而是消息中间件帮着做了。

至于消息表，其实还是没有省掉。因为消息中间件要询问发送方，事物是否执行成功，还是需要一个“变相的本地消息表”，记录事物执行状态。

