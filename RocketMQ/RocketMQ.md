

# 分布式事务解决方案之可靠消息最终一致性

## 前提概要

RocketMQ事务消息方案也只能保证最终一致性，无法保证强一致性，也就是可能，消费者消费失败了，但是生产者的事务是无法回滚的

最终一致性意味着系统中的各个组件在一段时间内可能会出现数据不一致的情况，但最终会达到一致的状态，但他会有一些机制来保证消息最终被成功消费

1. **消息重试机制**：消息系统通常会具备消息重试机制，即当消费者消费失败时，消息会被重新投递给消费者。在一定的重试次数后，如果消息依然无法被消费成功，系统可能会将消息发送到死信队列，待后续处理。
2. **消息幂等性**：生产者和消费者的业务逻辑应该具备幂等性，即多次执行同一个操作，产生的效果应该是一致的。这样即使消息被重复消费，也不会对系统产生不一致的影响。
3. **补偿机制**：如果消费者失败导致了数据不一致或其他问题，系统可能需要一些补偿机制来修复这些问题。例如，可以通过定期的数据校验和修复任务来保证系统的数据一致性。

总之，在最终一致性的场景下，系统中可能会存在一段时间的数据不一致，但随着时间的推移，系统会自动或通过一些机制逐步达到一致的状态。因此，尽管消费者失败但生产者未能回滚的情况下可能会短暂出现数据不一致，但仍然可以认为系统满足最终一致性的要求。

## 本地消息表方案

1. 通过本地事务保证数据业务操作和消息的一致性
2. 然后定时任务将消息发送至消息中间件
3. 待确认消息发送给消费方成功再将消息删除

### 案例

注册送积分为例来说明：下例共有两个微服务交互，用户服务和积分服务，用户服务负责添加用户，积分服务负责增加积分。

![image-20240223110934187](./assets/image-20240223110934187.png)

交互流程如下：
**1、用户注册**
用户服务在本地事务新增用户和增加 ”积分消息日志“。（用户表和消息表通过本地事务保证一致）
下边是伪代码

```mysql
begin transaction；
//1.新增用户
//2.存储积分消息日志
commit transation;
```

这种情况下，本地数据库操作与存储积分消息日志处于同一个事务中，本地数据库操作与记录消息日志操作具备原
子性。

**2、定时任务扫描日志**
如何保证将消息发送给消息队列呢？
经过第一步消息已经写到消息日志表中，可以启动独立的线程，定时对消息日志表中的消息进行扫描并发送至消息
中间件，在消息中间件反馈发送成功后删除该消息日志，否则等待定时任务下一周期重试。

**3、消费消息**
如何保证消费者一定能消费到消息呢？
这里可以使用MQ的ack（即消息确认）机制，消费者监听MQ，如果消费者接收到消息并且业务处理完成后向MQ
发送ack（即消息确认），此时说明消费者正常消费消息完成，MQ将不再向消费者推送消息，否则消费者会不断重
试向消费者来发送消息。
积分服务接收到”增加积分“消息，开始增加积分，积分增加成功后向消息中间件回应ack，否则消息中间件将重复
投递此消息。
由于消息会重复投递，积分服务的”增加积分“功能需要实现幂等性。

## RocketMQ事务消息方案

在RocketMQ 4.3后实现了完整的事务消息，实际上其实是对本地消息表的一个封装，将本地消息表移动到了MQ
内部，解决 Producer 端的消息发送与本地事务执行的原子性问题。

![](./assets/image-20240223140650993.png)

### 名词解释

- 发送方就是用户服务，本地事务就是执行新增用户操作

- MQ订阅方就是积分服务，监听发送方的消息，执行增加积分逻辑
- **半消息**是指在事务消息发送阶段发送的一种特殊类型的消息，它与普通消息不同之处在于，半消息在发送后处于不可消费状态，直到与本地事务执行结果相匹配后才能被消费。

### 执行顺序

1.**生产者发送事务消息**
Producer （MQ发送方）发送事务消息(半消息)至MQ Server，MQ Server将消息状态标记为Prepared（预备状态），注意此时这条消息消费者（MQ订阅方）是无法消费到的。 
本例中，Producer 发送 ”增加积分消息“ 到MQ Server。

2.**MQ Server回应消息发送成功**
MQ Server接收到Producer 发送给的消息则回应发送成功表示MQ已接收到消息。

3.**Producer 执行本地事务**
Producer 端执行本地事务,里面写了业务代码逻辑，通过本地数据库事务控制。
本例中，Producer 执行本地事务方法就是添加用户操作。

4.**消息投递**

- 若Producer **本地事务执行成功**则自动向MQServer发送**commit**消息，MQ Server接收到commit消息后将"增加积分消息" 状态标记为可消费，此时MQ订阅方（积分服务）即正常消费消息
- 若Producer **本地事务执行失败**则自动向MQServer发送**rollback**消息，MQ Server接收到rollback消息后将删除"增加积分消息"
- MQ订阅方（积分服务）消费消息，消费成功则向MQ回应ack，否则将重复接收消息。
- 这里ack默认自动回应，即程序执行正常则自动回应ack

5.**事务回查**
如果执行Producer端本地事务过程中，执行端**挂掉**，或者**超时**，MQ Server将会不停的询问同组的其他 Producer
来获取事务执行状态，这个过程叫**事务回查**。

MQ Server会根据事务回查结果来决定是否投递消息。

以上主干流程已由RocketMQ实现，对用户侧来说，用户需要分别实现本地事务执行以及本地事务回查方法，因此
只需关注本地事务的执行状态即可。

```java
public interface RocketMQLocalTransactionListener {
/**
‐ 发送半消息成功此方法被回调，该方法用于执行本地事务，即本地业务逻辑
‐ @param msg 回传的消息，利用transactionId即可获取到该消息的唯一Id
‐ @param arg 调用send方法时传递的参数，当send时候若有额外的参数可以传递到send方法中，这里能获取到
‐ @return 返回事务状态，COMMIT：提交 ROLLBACK：回滚 UNKNOW：回调
*/
RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg);
/**
如果 RocketMQ 在一定时间内（例如超过了预设的事务消息过期时间）未收到executeLocalTransaction事务消息的提交或回滚确认，RocketMQ 就会认为该事务消息可能存在异常，进而触发消息回查机制

‐ @param msg 通过获取transactionId来判断这条消息的本地事务执行状态
‐ @return 返回事务状态，COMMIT：提交 ROLLBACK：回滚 UNKNOW：回调
*/
RocketMQLocalTransactionState checkLocalTransaction(Message msg);
}
```

**事务消息超过预设的过期时间**可能发生在以下情况下：

1. **本地事务执行时间过长**：生产者执行本地事务的时间超出了预期，导致事务消息在执行完本地事务后，提交或回滚确认的时间延迟。
2. **网络故障或消息延迟**：在消息传输过程中，由于网络故障或消息队列服务器负载较高等原因，导致事务消息的发送或确认过程出现延迟。
3. **生产者异常退出**：生产者在发送事务消息后发生异常退出，导致事务消息无法得到及时的提交或回滚确认，从而超过了预设的过期时间。
4. 生产者未能及时处理回查请求：当 RocketMQ 执行事务消息回查时，生产者未能及时响应回查请求或处理回查请求失败，导致事务消息未能及时得到最终的提交或回滚确认。
5. **业务流程异常**：在复杂的业务场景中，可能存在各种意外情况或异常场景，导致事务消息未能按照预期的方式执行和确认，从而超过了预设的过期时间。

总之，事务消息超过预设的过期时间可能发生在生产者执行本地事务的时间过长、消息传输过程中出现延迟、生产者异常退出、回查请求未能及时处理等情况下。为了避免这种情况的发生，需要合理设置事务消息的过期时间，并确保在生产者端和消息队列服务器端的处理过程中保持高效和稳定。

### 总结执行顺序

1. 在生产者这边发送一条半消息（不可消费）

2. MQ回应发送成功，就去执行本地事务方法
3. 本地事务执行
   3.1. 本地事务执行成功（返回cmomit），就会让半消息可消费，消费者那边执行完逻辑返回成功，则整个事务完成
   3.2. 本地事务执行失败，就回滚，消息肯定也发不出去了
4. 本地事务方法超过预设时间没返回状态，调用事务回查方法看是否执行本地事务成功（通过幂等，就是消息表是否插入成功保证），然后进行提交或回滚

### 案例

#### 业务说明
本实例通过RocketMQ中间件实现可靠消息最终一致性分布式事务，模拟两个账户的转账交易过程

两个账户分别在不同的银行(张三在bank1、李四在bank2)，bank1、bank2是两个微服务和库

交易过程是，张三给李四转账指定金额。

上述交易步骤，张三扣减金额与给bank2发转账消息，两个操作必须是一个整体性的事务。

![image-20240223151510702](./assets/image-20240223151510702.png)

#### 程序组成部分

包括bank1和bank2两个数据库。

rocketmq 服务端：RocketMQ-4.5.0

**微服务及数据库的关系**

dtx/dtx-txmsg-demo/dtx-txmsg-demo-bank1 银行1，操作张三账户， 连接数据库bank1

dtx/dtx-txmsg-demo/dtx-txmsg-demo-bank2 银行2，操作李四账户，连接数据库bank2

**交互流程如下**
1、Bank1向MQ Server发送转账消息
2、Bank1执行本地事务，扣减金额
3、Bank2接收消息，执行本地事务，添加金额

#### 创建数据库

创建bank1库，并导入以下表结构和数据(包含张三账户)

```sql
CREATE DATABASE `bank1` CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';
CREATE TABLE `account_info` (
`id` bigint(20) NOT NULL AUTO_INCREMENT,
`account_name` varchar(100) CHARACTER SET utf8 COLLATE utf8_bin NULL DEFAULT NULL COMMENT '户
主姓名',
`account_no` varchar(100) CHARACTER SET utf8 COLLATE utf8_bin NULL DEFAULT NULL COMMENT '银行
卡号',
`account_password` varchar(100) CHARACTER SET utf8 COLLATE utf8_bin NULL DEFAULT NULL COMMENT
'帐户密码',
`account_balance` double NULL DEFAULT NULL COMMENT '帐户余额',
PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 5 CHARACTER SET = utf8 COLLATE = utf8_bin ROW_FORMAT =
Dynamic;
INSERT INTO `account_info` VALUES (2, '张三的账户', '1', '', 10);
#交易记录表(去重表)，用于交易幂等控制。
CREATE TABLE `de_duplication` (
`tx_no` varchar(64) COLLATE utf8_bin NOT NULL,
`create_time` datetime(0) NULL DEFAULT NULL,
PRIMARY KEY (`tx_no`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_bin ROW_FORMAT = Dynamic;
```

创建bank2库，并导入以下表结构和数据(包含李四账户)

```sql
CREATE DATABASE `bank2` CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';
CREATE TABLE `account_info` (
`id` bigint(20) NOT NULL AUTO_INCREMENT,
`account_name` varchar(100) CHARACTER SET utf8 COLLATE utf8_bin NULL DEFAULT NULL COMMENT '户
主姓名',
`account_no` varchar(100) CHARACTER SET utf8 COLLATE utf8_bin NULL DEFAULT NULL COMMENT '银行
卡号',
`account_password` varchar(100) CHARACTER SET utf8 COLLATE utf8_bin NULL DEFAULT NULL COMMENT
'帐户密码',
`account_balance` double NULL DEFAULT NULL COMMENT '帐户余额',
PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 5 CHARACTER SET = utf8 COLLATE = utf8_bin ROW_FORMAT =
Dynamic;
INSERT INTO `account_info` VALUES (3, '李四的账户', '2', NULL, 0);
#交易记录表(去重表)，用于交易幂等控制。
CREATE TABLE `de_duplication` (
`tx_no` varchar(64) COLLATE utf8_bin NOT NULL,
`create_time` datetime(0) NULL DEFAULT NULL,
PRIMARY KEY (`tx_no`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_bin ROW_FORMAT = Dynamic;
```

#### 启动RocketMQ
1.**下载RocketMQ服务器**
下载地址：http://mirrors.tuna.tsinghua.edu.cn/apache/rocketmq/4.5.0/rocketmq-all-4.5.0-binrelease.zip

**2.解压并启动**

启动nameserver

> 设置环境变量，windows直接去电脑右键属性设置
>
> ROCKETMQ_HOME=[rocketmq服务端解压路径]
>
> 进入bin目录双击`mqnamesrv.cmd`

启动broker

在bin目录下执行

> start mqbroker.cmd ‐n 127.0.0.1:9876 autoCreateTopicEnable=true

代码在https://github.com/Aerozb/learn-project/tree/main/rocketmq/transactional_message

正常转账，初始金额是10，转账后张三-1,李四+1

![image-20240223171856113](./assets/image-20240223171856113.png)
