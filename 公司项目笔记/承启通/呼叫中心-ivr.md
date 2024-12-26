# 前提概念

### IVR（交互式语音应答）

- **IVR 是一种技术规范和设计理念**：它定义了如何通过自动化系统与用户进行语音交互。IVR 系统通常会提供一系列预设的菜单选项，用户可以通过按键或语音输入进行选择。这个设计旨在提升用户体验，提供自助服务。
- **IVR 的功能**：IVR 系统可以实现多种功能，如呼叫路由、信息查询、表单填写、客户服务等。它是一种广泛应用于客户服务和通信行业的标准技术。

### FreeSWITCH

- **FreeSWITCH 是一种实现**：FreeSWITCH 是一个开源的通信平台，能够实现多种通信协议和功能，包括 IVR。它为开发者提供了构建和管理 IVR 系统所需的工具和功能。
- **如何实现 IVR**：通过 FreeSWITCH，开发人员可以使用**LUA**脚本和配置来创建复杂的 IVR 应用程序。这些应用程序可以根据用户的输入动态地做出反应，处理不同的交互场景。

### 结论

- **IVR 是一种设计规范和设计理念**，描述了如何与用户进行交互。
- **FreeSWITCH 是一个实现平台**，使得开发者能够构建和管理 IVR 系统。

这个关系可以类比于“建筑设计”与“建筑施工”之间的关系。IVR 设计了建筑的蓝图，而 FreeSWITCH 则是负责根据这个蓝图来实际建造和维护建筑的工具。

### LUA

**Lua 是一种编程语言**，在 FreeSWITCH 中用于编写和扩展通信逻辑

Lua 提供了灵活的编程能力，FreeSWITCH 则是实现这些能力的基础平台，而 IVR 是最终面向用户的交互界面

> **FreeSWITCH** 提供了电话呼叫处理的基础功能。
>
> **IVR** 是在 FreeSWITCH 上实现的具体应用，用于自动化处理用户的电话请求。
>
> **Lua** 作为脚本语言，被用来编写和控制 IVR 的具体逻辑和流程。

# 负责的任务

## IVR画布数据保存

1.前端拖动画布，我把画布数据存储下来，以供前端回显

2.解析画布数据，生成节点关系，后续生成lua脚本使用

## 通过 FreeSWITCH 平台结合 Lua 脚本语言实现 IVR 功能

根据节点关系，生成一个个lua函数，最后组成lua脚本，调用fs，实现ivr

### 开始节点

自定义和初始化一些参数，供后续节点使用

### 放音节点

根据填写的播放内容，生成播放函数

### 收号节点

放音并接收用户按键数字，并存储到指定参数

### 工作时间判断

判断当前时间是否是配置好的工作时间

### 转人工

**根据配置的技能去排队，等待空闲坐席接入**

1. 播放转接人工过程中语音

​		2.1 无坐席则播放转接人工失败放音 

​		2.2 还有坐席在线则播放转接人工过程排队等待音 

3. 排队超时则播放排队超时提示音 

4. 其他情况则播放转接人工失败放音

**直接转到指定坐席**

1. 播放转接人工过程中语音 

   2.1 无坐席则播放转接人工失败放音 

   2.2 还有坐席在线则播放转接人工过程排队等待音 

2. 其他情况则播放转接人工失败放音

## 新建IVR服务

建立一个与ivr流程关联的记录，用于企业号码关联ivr画布，企业号码->IVR服务->IVR流程

# 在本地测试打电话接听步骤

## 后端

打开`cloudcc-parent`项目，本地启动下面4个服务，开发环境停止同一命名空间的这4个服务

> cloudcc-sdk-interface
>
> cloudcc-queue-control
>
> cloudcc-client-server
>
> cloudcc-call-control

## 前端

前端项目，`public\config.js`的`baseConfig`切成开发环境，`public\sdk\sdk.js`的`registerRTC`的ws改成wss

`public\sdk\config.js`里面这几个值改成下面这些，如果是就不用改了

```js
const baseUrl = 'ws://localhost:25001/msg/'
const httpUrl = 'http://localhost:5001/'
const standbyUrl = 'http://localhost:20005'
```

启动项目，登录坐席，切成在线

这个是需要转坐席或者用到坐席才需要启动前端，如果走ivr流程且没用到转人工就不需要启动

## 底层数据库

如果打不通，则进入底层库`basecom`,删除一些数据

### **number_managemen**

查找`number`字段，同一号码是否有两个不同`service_code`,删掉和`HJZXAIDEV`不同的数据

### **service_management**

`service_code`字段查找`HJZXAIDEV`是否有不同号码

### **service_info**

是底层用来存储lua执行到转人工节点，调用转技能/坐席的接口信息用的

修改`HJZXAIDEV`地址，，改成开发环境的ng地址，ng里转发天填内网穿透的地址，用于转发到本地的queue-control的ivr接口，因为fs服务器访问不了内网穿透的地址，所以要用开发环境的ng来转发

如果改了service_info的地址，那得重启basecomapi，重新加载转人工接口

**底层basecomapi重启**：

```sh
vim src/aiapi.js
ps -ef|grep node
 cd /home/lrj/lcc-baseComApi
./bin/daemon.sh stop baseComApi
./bin/daemon.sh start baseComApi
```

## 底层redis

打不通时删除或修改`base:number:*`里面电话对应的service_code，看看对不对

## 呼叫中心redis

搜索`*netty*`，看这个登录的坐席的注册ip对不对，不对就删掉，才能打通

# 用户拨打企业号码流程

1. 13888888拨号给95550

2. 底层fs收到后，发送到mq主题`HJZXAIDEV`的消息

3. 我们在`EventMessageListenerOrderly`消费

4. 进入到呼入事件`CallInEventStrategyImpl`

5. 在判断是走ivr类`CallInTransferIvrStrategyImpl`或者转人工`CallInTransferSkillStrategyImpl`类，去处理

6. 如果是ivr且走到转人工节点，则会请求库里面设置的请求转人工地址

7. `cloudcc-queue-control`模块的`CallInIvrActionController#callInIvr`就是转人工方法，请求地址有俩，底层那边要用`call-in-ivr`，实际`distributeAgent`更准确

   ```JSON
   {"call-in-ivr", "distributeAgent"}
   ```

# 类作用

## EventMessageConsumerConfig

**eventMessageConsumer方法**

配置一个 RocketMQ 消费者实例，专门用于从指定的主题（`topic`）和标签（`tag`）中接收消息，`topic`配置在`nacos` -> `cloudcc-parent-common.yaml` -> `base.rocketmq.topic`,开发环境是`HJZXAIDEV`,

注册消息监听器，这里使用的是 `eventMessageListenerOrderly`，它实现了顺序消费的监听逻辑

## EventMessageListenerOrderly

监听电话打进来的主题，如果是`call_in`类型，则进入客户呼入事件`CallInEventStrategyImpl`,在判断是转IVR还是转技能(人工服务)，最后进入到`CallInTransferIvrStrategyImpl`转IVR或者`CallInTransferSkillStrategyImpl`转人工

## CallInEventStrategyImpl

处理客户呼入的类，判断是要转人工还是转ivr



## ServiceListCache

启动的时候，遍历所有公司去给排队的用户分配坐席，看这个公司在人工服务管理页面配的排队策略是什么，企业号码使用这个配的策略

initCompanyRunState()就是去执行这个逻辑

# 呼入和接听建立过程

**Originate**：

- `originate` 是 FreeSWITCH 中的一个命令或操作，用于创建一个新的呼叫会话。通过这个命令，系统可以主动发起一个呼叫到某个目标（接听者）。
- 例如，可以使用 `originate` 命令将呼入者和接听者连接在一起。

**Bridge**：

- `bridge` 是用于连接两个通道（呼入者和接听者）的操作。一旦呼叫被建立，`bridge` 操作会将呼入者和接听者的音频流连接在一起，使他们能够相互通话。
- 在 FreeSWITCH 中，当呼叫成功建立后，通常会调用 `bridge` 来将通道连接。

**呼入者通道创建**：

- 当呼入者发起呼叫时，FreeSWITCH 接收到这个呼叫请求，并为呼入者创建一个通道。

**接听者通道创建**：

- 在接听者接听呼叫之前，FreeSWITCH 会使用 `originate` 命令或相关逻辑，向接听者发起呼叫。这将为接听者创建一个通道。

**桥接通道**：

- 一旦接听者接听电话，FreeSWITCH 会执行 `bridge` 操作，将呼入者的通道和接听者的通道连接在一起，使双方能够进行通话。



每个通道在 FreeSWITCH 中都是独立的，呼入者和接听者分别拥有自己的通道。

当接听者接听电话时，FreeSWITCH 会将这两个通道桥接起来，实现音频流的传输。

这种设计允许 FreeSWITCH 同时处理多个呼叫，每个呼叫都有独立的通道和处理逻辑。

# AI外呼机器人画布

创建一个机器人，点击设计进入画布，一个画布下有多个场景，等于是一个机器人有多个场景，每个场景由多个节点组成

## 实体管理

**概念**

实体识别（[NER](https://so.csdn.net/so/search?q=NER&spm=1001.2101.3001.7020)）是自然语言处理中的一项重要任务，旨在从文本中识别出具有特定意义的实体（如人名、地名、组织名、时间、金额等）。这些实体在许多应用中起着关键作用，尤其是在信息提取、问答系统、文本分类等任务中。

**页面配置**

配置实体名称、实体值

- **人名**：如“Albert Einstein”、“李白”。
- **地名**：如“Paris”、“北京”。

**应用场景**

根据某个节点配置的实体名称，然后用这个实体名称根据用户说的话识别出数据，这个数据赋值给变量

## 变量管理

**概念**

用于保存用户输入的动态数据，可以在整个外呼流程中供系统使用

**页面配置**

配置变量名称，变量描述

date，通话的日期

**应用场景**

在各个节点中，根据用户说的话转成内容，在提取需要的数据赋值给这个变量，之后在java代码中就可以获取并使用这个变量；分析用户的话是底层做，这个变量存在哪都行，只要能拿到，比如redis，或者线程变量中

 ## 普通节点

当走到这个节点，可以分析用户说的话，提取想要的数据到变量，给当前流程使用；

根据用户的回答看是否是肯定、拒绝、否定、无应答、默认、自定义分支五个类型中的一个再接着往下走下个节点

走到当前节点可以设置当前用户的意向等级，意向等级是从意向标签配置页面获取的，比如A(意向较强）B(意向一般）C(带筛选）D(意向较弱）E(需在跟进）F(无需跟进）其中一个

## 跳转节点

比如从上面的肯定分支走到这里，直接执行下面选项，可选项 挂机、跳转到下一场景、跳转到指定场景、转人工

## 判断节点

有多个分支，每个分支是变量和变量间的判断或者变量和自己填的数据判断，符合条件就往下一个节点

## 按键节点

可以添加1-9的数字，根据1-9生生成不同分支

# 外呼流程（了解，没负责）

`com.linkcircle.report.outbound.service.impl.IvrOutboundTaskServiceImpl#addOrUpdateOne`创建任务类型为ivr的外呼定时任务

`com.cqt.queue.calltask.service.impl.IvrOutboundCallTaskStrategyImpl#ivrOutboundCallTaskJob`这个是执行定时任务的方法，到时间等待执行

`com.cqt.queue.calltask.service.impl.IvrOutboundCallTaskStrategyImpl#originate`发起外呼

`com.cqt.call.service.rpc.CallControlRemoteServiceImpl#request`向fs底层发起呼叫请求

`com.cqt.call.strategy.client.impl.OutboundCallTaskClientRequestStrategyImpl#deal`最后调用到这里，里面就是去请求底层了
