---
title: "微服务模式笔记:服务分解策略"
date: 2019/02/13 01:15:00
---
Microservice Patterns第二章的读书笔记  
原章节链接: [https://learning.oreilly.com/library/view/microservices-patterns/9781617294549/kindle_split_010.html](https://learning.oreilly.com/library/view/microservices-patterns/9781617294549/kindle_split_010.html)  

Decomposition strategies 

微服务最关键的挑战 也就是微服务的本质是如何将应用的功能分解到服务中去
服务是业务相关而不是技术相关


# 2.1. What is the microservice architecture exactly?

## 2.1.1. What is software architecture and why does it matter?
应用架构是将应用分解成各个部分(元素)和这些部分间的关系

软件架构的4+1 view model  
简单理解就是从几个角度来描述架构  
![](/images/1244488-20190213011614249-992071651.png)


[www.cs.ubc.ca/~gregor/teaching/papers/4+1view-architecture.pdf](www.cs.ubc.ca/~gregor/teaching/papers/4+1view-architecture.pdf)

- Logical view : 开发者创建的软件元素。就像类和包 类之间的继承 关联和依赖
- Implementataion view: 构建系统的产出。比如java是jar或者war
- Process view:运行时组件.每个元素都是一个进程，和各个进程间的进程间通信。
- Deployment：进程是如何映射到机器的。

架构对于功能需求基本没有什么帮助    
但架构又是重要的 因为他让应用满足第二种需求: 服务质量(quality of service,QoS)需求

## 2.1.2. Overview of architectural styles
一种特定的架构提供有限的元素(组件)调色盘和关系(connectors)   
你从中定义出应用架构的视图

### 分层架构  
定义了三层:
- Presentation layer: 包含实现了用户界面和额外的API的代码
- Business logic layer： 包含业务逻辑
- Persistence layer：包含和数据库交互的逻辑

但是也有明显的缺点：
- 单一的展示层
- 单一的持久层
- 定义的业务逻辑依赖于持久层 离开数据库无法测试  
会存在依赖倒置，业务层定义的数据访问接口最后需要持久层定义DAO类去实现(低层依赖高层所提供的接口)

### 六边形架构
将业务放在了最中心  
`inbound adapter`  
`outbound adapter`  
业务逻辑不依赖于adapter 相反是adapter依赖于业务逻辑  
![](/images/1244488-20190213010815889-2062967823.png)  

一个业务逻辑有一个或多个`ports`  
一个`port`定义了一系列业务逻辑和外部交互的操作

六边形架构最大的好处是将展示层和数据访问逻辑从业务逻辑中解耦


## 2.1.3. The microservice architecture is an architectural style
单体架构在implement view就是一个单独可执行的jar或war
http://microservices.io/patterns/monolithic.html

http://microservices.io/patterns/microservices.html  
![](/images/1244488-20190213010839549-1161206549.png)


### 什么是服务  
服务是独立的 无依赖的可部署软件组件 实现了一些有用的功能  
![](/images/1244488-20190213010850162-1923323128.png)

不同服务间只能通过API调用 不像单体还可以通过代码

### 松耦合
https://en.wikipedia.org/wiki/Loose_coupling
服务间的交互只通过API 封装细节

### 共享库
只在不太可能会发生变化的地方使用共享库
和具体业务无关 通用的功能上

### 服务的大小不重要
和组织结构相关


# 2.2. Defining an application’s microservice architecture
3步定义应用的微服务架构  
这里只是展示一个例子 现实中会有更多的问题  
需要迭代等  
![](/images/1244488-20190213010902827-949019283.png)  



第一步找到服务的具体操作 这里定义为了`system operation`  
其实也就是找到各个请求和回应(或者找到各个事件)  
服务拆分的一些阻碍:  
- 网络延迟
- 服务间的同步调用降低了可用性
- 在不同服务间维持数据一致性

## 2.2.1. Identifying the system operations
从应用的需求出发 包括user story和对应的用户场景

有两种类型的`system operation`  
commands:创建 更新和删除数据
queries: 查询数据

分析user stories和场景中的动词
一个例子:
![](/images/1244488-20190213011117033-940471440.png)


## 2.2.2. Defining services by applying the Decompose by business capability pattern
通过业务能力分解  
http://microservices.io/patterns/decomposition/decompose-by-business-capability.html  

![](/images/1244488-20190213011136593-1822789472.png)

业务能力 子能力以及与服务的对应

但业务变迁   
频繁的RPC带来的性能损耗  
会带来一定的阻碍  

## 2.2.3. Defining services by applying the Decompose by sub-domain pattern
DDD  
subdmain   
bounded context  
http://microservices.io/patterns/decomposition/decompose-by-subdomain.html

![](/images/1244488-20190213011153564-1888824784.png)


## 2.2.4. Decomposition guidelines
一些分解的原则  
可以沿用一些OO里的原则

### 单一职责原则

### 共同封闭原则(Common Closure Principle)
因为一个相同的原因导致两个类需要修改 这两个类应该在一个包下  
对应于微服务 把要因为同一个原因修改的组件放到同一个服务里去

通过业务能力和子域 结合SRP和CCP是一个很好的分解一个应用到服务的技术  
为了应用他们成功地开发微服务架构 你必须解决一些事务管理和跨进程通信问题

## 2.2.5. Obstacles to decomposing an application into services
一些阻碍:
- 网络延迟
- 同步通信导致的可用性降低
- 跨服务的数据一致性维护
- 获取数据的一致性视图
- 上帝类阻止分解

### 网络延迟
网络延迟是分布式系统长期困扰的问题.
服务拆解之后可能会发现有大量的服务间的往返通信
可以提供批处理API在一次往返中
但一些其他场景 可能就需要整合服务了

### 同步通信导致的可用性降低
改进方法是异步消息传输

### 跨服务的数据一致性维护
涉及到分布式事务 saga 最终一致性

### 获取数据的一致性视图
实践中一般不会是太大的问题

### 上帝类阻止分解
上帝类(god classes)是一个庞大的类 在整个项目中被贯穿使用
一般都是为了实现了一个应用不同方面的业务逻辑 有大量的field  
![](/images/1244488-20190213011219565-1974762110.png)


方案1:
提供一个库 并创建一个中央的order数据库  
问题是紧耦合…所有的服务都要连到这个库

方案2:
封装order数据库到order服务  
这样就把他变为了类似数据库API的服务 只是为了提供order数据库的访问而存在

方案3:  
使用DDD 将order的概念分解 各个子域有自己的order  
![](/images/1244488-20190213011237612-887560591.png)

但这样会影响用户体验 在用户体验和多个实际模型间要有一种翻译

## 2.2.6. Defining service APIs
定义服务API  
他的操作和事件

基于两点:
1.`System operation`需要
这个API可能是被客户端直接调用 也可能是服务端直接调用 用来完成这个服务里的某个具体操作  

2.支持服务间协作  
为其他服务的实现提供一些帮助
