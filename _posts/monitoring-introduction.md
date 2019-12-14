---
title: "监控基础"
date: 2018/07/26 20:57:00
---
Monitoring with Prometheus读书笔记  
原书见:  
https://www.safaribooksonline.com/library/view/monitoring-with-prometheus/9780988820289/  

---    
监控需要监控技术环境和监控业务情况  

![](/images/1244488-20180726204211275-746653901.png)
  
# 监控基本原则  
监控需要在开发业务前就进行考虑,结合业务情况和环境,考虑要监控的metric.    

![](/images/1244488-20180726204317681-327402975.png)  

监控设计要从业务逻辑出发 再下沉到应用的监控和操作系统的监控  
但这不是说通用的系统监控不需要 只是他们不能用来汇报业务价值  

监控服务的正确性 例如不是单单看health接口是否返回了200 而要确认里面的值  

不要只使用静态的阈值 阈值应该是根据窗口数据等方式计算获取  

保存足够多的历史监控数据(用于之后的分析)   

监控需要容易被设置/部署 需要自动化支持  

# 监控机制  

Probe :应用外  
Introspection:应用内  

推
拉
普罗米修斯基本上是基于pull的 但也有push的方式

监控数据的类型  
Metrics:时间序列数据 记录应用的度量的状态  
Logs:应用程序发出的文件事件 通常情况更有用 但这本书不怎么讲 https://www.logstashbook.com/  

# Metrics  

metrics提供了动态的,实时的基础设施状态,帮助你管理环境和给做更好的决定.  
Metrics are measures of properties of components of software or hardware  

固定时间间隙收集 -> 颗粒性或者分辨率   

## Metrics类型  
### guage  
![](/images/1244488-20180726204558560-2093206341.png)


### counter   
累计数据
![](/images/1244488-20180726204611839-1525545816.png)  

### Histograms  
统计数据  
![](/images/1244488-20180726204702984-1305783817.png)  

其他统计字段:  
Counter  
Sum  
Average  
Median  
Percentiles  
Standard deviation  
Rates of change  

平均数的陷阱:  
![](/images/1244488-20180726204746075-945944029.png)  

![](/images/1244488-20180726204757068-237046767.png)  


# 监控方法论  
## USE方法  
For every resource, check utilization, saturation, and errors  
对于每个资源 检查利用率 饱和度和错误  
http://www.brendangregg.com/USEmethod/use-linux.html  


## Google Four golden signals  
Latency  
Traffic   
Errors  
Saturation  

# 上下文相关 有用的警报和通知  
Alert和notification是监控的主要输出  
- 通知是需要可以指导行动 明确清晰的.  
- 在通知上增加上下文信息.  
- 只发送有意义的通知.  

# 可视化  
- 清晰地展示数据  
- 引导观察者思考背后的实质 而不是专注于视觉效果  
- 避免歪曲数据  
- 清晰地展示大数据集  
- 允许改变视图的粒度而不影响理解
