---
title: "java并发编程知识点备忘"
date: 2018/04/26 02:40:00
---
最近有在回顾这方面的知识,稍微进行一些整理和归纳防止看了就忘记.  
会随着进度不断更新内容,比较零散但尽量做的覆盖广一点.  
如有错误烦请指正~

---
# java线程状态图
![](/images/1244488-20180426021937095-779874420.png)

---
# 线程活跃性问题
- 死锁
- 饥饿
- 活锁  

饥饿原因:  
- 高优先级造成低优先级无法运行(概率吧)
- 无法进入同步块(比如进入的线程陷入死循环)
- 无法被唤醒(没有notify)

---
线程安全性问题的条件:
- 多线程环境下
- 多线程共享同个资源
- 存在非原子性操作  
破坏掉其中一条即可 

---  
# synchronized  
内置锁  
涉及字节码:monitorenter monitorexit  
锁的信息存在对象头中

偏向锁 轻量级锁 重量级锁相关:  
![](/images/1244488-20180426022827527-266095883.png)
![](/images/1244488-20180426022837940-1764731172.png)

参考资料:  
[Synchronization](https://wiki.openjdk.java.net/display/HotSpot/Synchronization)  
[java 中的锁 -- 偏向锁、轻量级锁、自旋锁、重量级锁](https://blog.csdn.net/zqz_zqz/article/details/70233767)

---  
# 单例模式实现
积极加载  
懒加载:double check,静态内部类  
枚举实现  

参考资料:
[Java单例模式——并非看起来那么简单](https://blog.csdn.net/goodlixueyong/article/details/51935526)  

---  
# volatile  
硬盘 - 内存 - CPU缓存  
lock指令: 当前处理器缓存行写回内存,其他处理器该内存地址数据失效  

参考资料:  
[Java并发编程：volatile关键字解析](http://www.importnew.com/18126.html)  

---  
# 原子更新类  
JDK5开始提供,J.U.C.atomic包  
底层多是使用了Unsafe类的CAS操作.  
基本分类:  
- 原子更新基本类型: AtomicInteger
- 原子更新数组: AtomicIntegerArray  
- 原子更新引用类型: AtomicReference
- 原子更新字段: AtomicIntegerFieldUpdater  

参考资料:  
[Java中的Atomic包使用指南](http://ifeve.com/java-atomic/)  

---  
# AQS  
可以作为锁 同步器的基础.  
本身是一个模板类,只需要重写所需的抽象方法:  
![](/images/1244488-20180609201705069-2063211362.png)  

获取和释放(独占)的分析:  
![](/images/1244488-20180609201936193-1665209808.png)  
获取成功下直接返回.  
如果获取失败,并且被中断下就直接返回.  

![](/images/1244488-20180609202057443-79184992.png)  
如果没有抢占成功,则判断是否需要park,park后线程将不再被调度,park解除后返回中断状态,(但这个实现中即使被中断也没关系,在acquireInterruptibly中则会直接抛错)  

![](/images/1244488-20180609203155633-1825835499.png)  
是否需要park是由上一个节点的状态来决定的,在父节点为SIGNAL的情况下需要park,如果大于0(表示cancel,这个节点以及前面连续的cancel节点需要从队列里出去了)  
这里可以看到不是SIGNAL的情况下父节点要么被移出要么被设置为了SIGNAL,注意这段代码外层是一个死循环,所以最终如果一直没抢占到,这个线程肯定会被park的.  
这个设计在源码里就有说明,SIGNAL不是用来控制当前节点而是控制他的子节点,同时也说明这个节点的子节点正在等待.    
对应release的时候也是,释放的节点(头节点)需要判断自己的状态是否小于0(为0的话说明没有子节点被添加要唤醒).  

自己看源码的简单分析,只看了这一个流程.    

公平和非公平主要就是对于队列的操作,公平的实现直接获取下一个节点即可.  

更为详细的请参考:  
[【JUC】JDK1.8源码分析之AbstractQueuedSynchronizer（二）]("https://www.cnblogs.com/leesf456/p/5350186.html")  
[AbstractQueuedSynchronizer的介绍和原理分析]("http://ifeve.com/introduce-abstractqueuedsynchronizer/")  


2018年6月09日20点42分

---  

# 读写锁  
要注意用了一个state存了两个锁的状态 共享锁(读锁)高16位 互斥锁(写锁)低16位   
![](/images/1244488-20180719022157877-172838333.png)

读锁获取的前置条件:
![](/images/1244488-20180719022316487-1653952882.png)
读锁要获取前提是没有写锁或者有写锁但是写锁是自己持有的  
这也是锁能降级的实现原理 一个线程持有写锁后可以自己持有读锁  

![](/images/1244488-20180719022519821-1742436118.png)  
要要获取读锁的话,得保证没有读锁,这也就是`ReentrantReadWriteLock`无法实现升级的原因.  
所以千万不要写出这样的代码
```java
readLock.lock();
....
writeLock.lock();
....
readLock.unlock();
....
writeLock.lock();
```  

获取写锁时就会发生死锁.   
 
2018年7月19日02点37分





Update:  
2018年4月26日02点38分 init
2018年5月5日16点17分 单例-原子更新类
