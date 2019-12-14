---
title: "linux性能优化参数小节"
date: 2018/05/29 02:05:00
---
总结一些和性能相关的常见参数  
  
# 内核相关参数
位于`/etc/sysctl.conf`文件,向文件中添加  
用`sysctl -a`可以查看默认配置  
修改后可以通过`sysctl -p`执行并看看有没有错误  
例如设置错了参数:  
![](/images/1244488-20180529020156928-1352852257.png)

`net.core.somaxconn=65535`  
一个端口最大的监听TCP连接的队列长度  


`net.core.netdev_max_backlog=65535`  
数据包速率比内核处理块时 允许送到队列的数据包的最大数目  


`net.ipv4.tcp_max_syn_backlog=65535`  
TCP syn队列的最大长度 第一次握手的连接 参数过大可能也会遭受syn flood攻击  


`net.ipv4.tcp_fin_timeout=10`   
fin超时时间 表示如果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2状态的时间 
`net.ipv4.tcp_tw_reuse=1`   
开启重用
`net.ipv4.tcp_tw_recycle=1`   
快速回收  

```
net.core.wmem_default=87380  
net.core.wmem_max=16777216  
net.core.rmem_default=87380  
net.core.rmem_max=16777216  
```
缓冲区的默认值和最大值  


`net.ipv4.tcp_keepalive_time=120`  
keepalive的检测时间间隔 单位为秒  
`net.ipv4.tcp_keepalive_intvl=30`  
检测无效时 重发消息间隔  
`net.ipv4.tcp_keepalive_probes=3`  
检测无效时 最多重复确认次数

`kernel.shmmax=4294967295`  
linux内核参数中最重要的参数之一   
用于定义单个共享内存段的最大值
64位linux 可取最大值为物理内存值-1byte 建议值为物理内存一半  

`vm.swappiness=0`  
free -m Swap中的内容  
![](/images/1244488-20180529015148150-543066928.png)

风险:  
- 降低操作系统性能  
- 在系统资源不足下,容易被OOM kill掉   

设置为0是告诉系统除非虚拟内存完全满了 否则不要使用交换区


# 增加资源限制
位于 /etc/security/limit.conf
```
* soft nofile 65535
* hard nofile 65535
```
- \* 对所有用户有效
- soft 当前系统生效的设置
- hard 系统所能设定的最大值
- nofile 打开文件的最大数目
- 65535 限制的数量
- 需要重启系统生效

设置前:  
![](/images/1244488-20180529015137572-1250617565.png)

之后open files的值会提高为65535

# 磁盘调度策略
`/sys/block/devname/queue/scheduler`
查看可通过`cat /sys/block/sda/queue/scheduler`
![](/images/1244488-20180529014941012-898523088.png)  
现在使用的cfq 可选的是noop和deadline  

![](/images/1244488-20180529015114120-317499833.png)  
用echo写入可以立即生效  

简介:
* noop电梯式调度策略  
实现了一个FIFO队列 倾向饿死读而利于写 对闪存设备 RAM和嵌入式系统是最好的选择

* deadline 截止时间调度策略  
确保了在一个截止时间内服务请求 这个截止时间是可调整的 而默认读期限短于写期限
对于数据库类应用是最好的选择

* anticipatory 预料IO调度策略  
本质上和deadline一样 但在最后一次读操作后 要等待6ms 才能继续进行对其他IO请求进行调度 将一些小写入流合并成一个大写入流 用写入延迟换取最大的写入吞吐量 适合写入较多的环境 比如文件服务器 对数据库环境表现很差

* cfq 绝对公平算法


参考资料:  
[TCP/IP及内核参数优化调优](https://www.cnblogs.com/jking10/p/5472386.html)

[Linux IO Scheduler（Linux IO 调度器）](https://www.cnblogs.com/cobbliu/p/5389556.html)
