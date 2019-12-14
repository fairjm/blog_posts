---
title: "短信服务构建总结"
date: 2019/03/31 19:23:00
---
又来水文了...  
但感觉没什么其他内容可写,就将之前做的一个消息平台稍微做点总结.  
简单说说短信模块的实现,做的东西不复杂,权当总结了~.  

# 需求  

## 功能需求
功能需求比较直接.  
需要提供一些短信和统计功能即可:  
- 通知短信
- 验证码短信
- 语音短信
- 发送量统计  

## 非功能需求 
主要是以下几点  
- 稳定的发送保证:  
分为两部分,一个是服务本身的稳定性,另一个是短信服务提供商的稳定性.  
服务本身稳定是自己需要实现的,可以通过微服务平台已经有的一些工具保证,这里不进行展开.而另一个是需要外部保证的,外部需要保证的东西永远是脆弱的,所以不能仅仅使用一家服务商,需要使用多家.  
- 方便扩展:  
接着第一点,服务商可能会有更换的需求,需要保证能够方便接入下架服务商.
- 资产保护:  
短信相比邮件,微信通知等的一个不同是他花费比较高,一条的计价在3分左右,较长的短信会被拆为多条,如果不加限制进行保护很容易造成大量的资损.


# 技术选型  
有了上面的需求之后就是选用什么技术了.  
基础的web服务就使用公司现有的即可.  
对于存储,因为涉及到发送信息和报告的记录,量可能会很大(其实上线之后也还好,3个月不到千万记录),统计时可能会按照某些发送内容统计,使用MySQL之类的可能会不太靠谱,选择使用ES.  
对于安全,一定需要做手机号每日发送量和发送频率等的限制,这个用redis可以较为轻松解决.  

最后一个简化版的架构图如下:  
![](/images/1244488-20190331180608613-425422922.png)

# 实现细节  
上面都是这个服务战略上的一些东西,这里说一些战术上的.  

## 接入多家服务商  
需要有多家服务商的接入,一家服务商可能同时供应多个短信功能.  
同时同一个短信功能需要轮流调用所有提供的服务商,并在一家失败后进行重试.  
也需要支持灵活的配置,可以禁用掉一些服务商,也可以方便加入新的.  
在实现上先定义了各个发送功能接口,例如:  
```
interface NotiSmsSender {
    SmsSubmitResult sendNoti(NotiRequest request);
}  


interface VerifyCodeSmsSender {
    SmsSubmitResult sendVerifyCode(VerifyCodeRequest request);
}
```  
对应的服务商实现不同的接口,提供不同的服务,每个对应的实现都有一个id字段,用于之后的配置.  

现在有了一坨sender,接下来要定义router,用来进行请求的分发.  
router内含有sender列表,配置router内的sender就可以实现服务商的下线功能了.  
大致流程如下:    
![](/images/1244488-20190401021438477-1987189345.png)  


实际实现中使用了Akka,各个sender是一个actor,router负责监督各个sender的健康情况,此外天然获得了超时时间的能力(底层使用了异步HttpClient,actor内部不会发生阻塞).


## 发送限制设置  
关于手机号和ip的发送数量计数使用`reids`的`incr`配合时间戳相关的key就可以很简单完成,这里不累述这个.  

发送限制的话首先要针对不同功能,因为不同功能的使用量是截然不同的,并且也有隔离的作用,一个功能超限不能影响其他功能.  

其次,发送的限制需要分层级,当达到了全局最大限制后,就算没有超过单手机限制也不应该发出去了.  
使用的层次是:   
功能全局次数(次数) -> 单接入应用限制(次数/白名单/开关) -> 单手机/ip限制(次数/白名单/开关) -> 风控服务(开关)  
最后次数要根据统计情况和业务更改做适当调整.  

## 性能  
这个服务很明显是IO密集的,就没什么计算,主要是调用各种外部服务,等IO返回.  
所以对于这些各个用来做请求的线程池就可以配置稍微大一点,实际最主要的就是HTTP的连接池.但连接超时,读取超时也不适合设置过大.  
实际测试也没多大问题,平日一天短信请求量也在几w左右,并没有太大的瓶颈.    

其他的一些点和普通的web服务也大同小异了,没有什么特别的技巧.  
其实之前一些文章也是在构建这个服务的时候写的,这段时间没写什么就当划水了,比如:  
[Akka实践一些总结](https://www.cnblogs.com/fairjm/p/10035181.html)  
[ES踩坑笔记](https://www.cnblogs.com/fairjm/p/elasticsearch_in_trouble.html)  
这个服务构建的时候根据需要用了些之前没有使用的组件,也算额外获得了一些回报吧~