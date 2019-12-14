---
title: "JVM内存知识备忘"
date: 2018/09/15 23:35:00
---
又是一篇备忘...  
主要记录一些知识,进行一些资源的汇总.  

先来群里`liufor`大大提供的两张图,清晰易懂:  
![](/images/1244488-20180915225940516-535609245.png)  
Dockerized Java  
https://www.youtube.com/watch?v=NQ5hTEp-GTM    


![](/images/1244488-20180915230059584-477234180.png)   
Java on Linux for devs and ops  
https://www.slideshare.net/aragozin/java-on-linux-for-devs-and-ops  

一些线上排查的注意事项:  
1.greys不能用于增强系统类,即便他给了个开关,但不要用;  
2.btrace增强后无法撤销,要撤销需要重启服务;  
3.NMT不要开启detail,影响性能;  
4.配置jemalloc和tamalloc的采样的时候注意采样率,千万别配0(全部采样);  
5.jmap的-heap,jstack和jmap -F参数会导致JVM暂停.(SA.attch)


# 常用配置&命令  

## 常用命令&工具
### JVM启动用的命令行
```
jcmd process_id VM.command_line
```  

### 手工触发gc  
```
jcmd process_id GC.run
```

### 显示调优标志  
```
jcmd process_id VM.flags [-all]
```  
all比较有用 可以看到全部的 包括默认值 在排查一些问题的时候能看的信息比较多  

### 内存dump 使用情况查看:  
```
jmap -dump:format=b,file=test.bin process_id  
jmap -heap process_id  
```

### heap dump分析  
可以使用`MAT`,`Heap Analyzer`和`VisualVM`这些工具.  

一些提取操作(比如要获得一个大字符串/char[]的值)可以通过OQL表达式.  
例子:  
获取到了address 拿取数据(是个char[])
```javascript
chars = heap.findObject("0x666666666")
var writer = new java.io.FileWriter("D:/temp/oql/666.txt");
for (var i = 0; i < chars.length; i++) {
    writer.write(chars[i]);
}
writer.close()
```  

visualVM中拿到index(这个index是sharp后面的 不是address)  
```javascript
filter(heap.objects('char[]'),
 function(it, index) {
  if (index == 66) {
      var writer = new java.io.FileWriter("D:/temp/oql/666.txt");
      for (var i = 0; i < it.length; i++) {
          writer.write(it[i]);
      }
      writer.close();
      return true;
  }
  return false;
})
```



### 更多内存信息  
在linux上使用,使用/proc/pid/map内的信息,以及pmap.  
使用gdp dump出内存查看信息  
详见: http://lysu.github.io/blog/2015/02/02/how-to-deal-with-non-heap-or-native-memory-leak/  



## GC log相关
```  
//日志数量 每个log大小 存放位置
-XX:NumberOfGCLogFiles=7
-XX:GCLogFileSize=64M
-Xloggc:/opt/jetty/logs/gc.log
//绝对时间戳 相对的用-XX:+PrintGCTimeStamps
-XX:+PrintGCDateStamps
//详细的信息 平均时间等 精简的用-XX:+PrintGC
-XX:+PrintGCDetails

```

## NMT
```
-XX:NativeMemoryTracking=[off | summary | detail]
```
```
jcmd <pid> VM.native_memory [summary | detail | baseline | summary.diff | detail.diff | shutdown] [scale= KB | MB | GB]  
```
可以比较有效看到几个部分的内存使用情况 以及设置baseline后 在看具体的变化量  
使用时要先设置JVM参数  生产环节慎用detail  
参考文档:  
https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr007.html  
直接内存在internal中 但不包含Mapped ByteBuffer  
https://stackoverflow.com/questions/47309859/what-is-contained-in-code-internal-sections-of-jcmd  

# 博客 & 文章   
一些文章和不错的博客.  
随着时间的迁移和JVM的升级,这里(包括这文)的内容可能会过时,自己甄别一下~  

你假笨博客  
http://lovestblog.cn/


Alexey Ragozin的博客(也是第二图ppt的作者)  
http://blog.ragozin.info/  

JVM内存调优相关的一些笔记（杂）  
https://zhanjindong.com/2016/03/02/jvm-memory-tunning-notes  


REDUCE LONG GC PAUSES  
https://blog.gceasy.io/2016/11/22/reduce-long-gc-pauses/  


Oh the Places Your Java Memory Goes  
https://jkutner.github.io/2017/04/28/oh-the-places-your-java-memory-goes.html  


入门科普，围绕JVM的各种外挂技术  
http://calvin1978.blogcn.com/articles/vjtools-tools4.html
