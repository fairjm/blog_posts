---
title: "异步/同步,阻塞/非阻塞,单线程/多线程概念梳理"
date: 2019/02/25 00:35:00
---
最近看了一文说到异步一定是单线程的,顿时就傻眼了,对相关概念和同事进行了一些讨论和总结.  
文中的描述是以我的理解来说的,可能不一定准确甚至正确,有错欢迎指正.  

这三个概念我认为是描述不同的维度的,概念正交.    

# 异步 同步  
异步和同步是不同的流程设计风格.  
但存在依赖关系的操作之间是同步的,也就是如果操作B依赖操作A的返回,那么B必须要在A结束后才能执行.  
比如你要读取文件然后对文件内容进行处理,那么读取内容和处理内容就是同步的.  

而异步这是操作间没有依赖关系,或者先后顺序并不重要.  
比如用户登陆要给登陆奖励,在确认用户可登陆后之后的登陆流程和发放奖励间并无依赖关系,那么他们就可以异步执行.  

这个概念主要描述依赖关系和流程设计.  

# 阻塞 非阻塞
这个概念是描述操作是否会立即返回的.  
这个概念常常和异步/同步混在一起.  
同步就一定是阻塞的吗,我觉得这不一定,主要还是取决于同步执行里的各个操作是否是阻塞的.  

非阻塞操作可以被用在异步API的实现.  
基于单线程的异步API实现里的操作一般都会要求为非阻塞.  

# 单线程 多线程  
这两个概念常和异步 同步混在一起.  
比如认为异步和多线程绑定的,但其实不然,有很多基于Eventloop的单线程实现.  
单线程 多线程主要描述的是运行的环境.  
如图:  
![](/images/1244488-20190225002929991-1118096007.png)  
(来源参考3)  
这张图表述了他们的一种组合.  

# 一些实现中相关的概念  
在实现中,为了高效的执行这些概念会被组合起来使用.    
单线程的异步操作,需要依赖非阻塞操作(不然单个线程就直接阻塞了).而这里异步的目的是为了提高线程的利用率,这在IO密集的应用中比较有效.  
但如果操作本身是计算密集的,那么单线程的异步操作就没有太大的意义了.  

而如果无法满足操作都是非阻塞的,那常常会使用多线程来规避主线程(例如服务器中的请求处理线程和GUI中的UI线程)阻塞.  
这种一般会返回`Future`等类似的占位符,并提供非阻塞查询结果是否返回,或者支持回调函数.  
这在java中比较常见,毕竟java世界中的阻塞操作比较多.  

上述概念和IO的关系.  
一些对应的API.  
![](/images/1244488-20190225003047863-763080176.png)  
要记住的是这些操作和单线程/多线程都没什么关系.  
更多的内容之前有翻译一篇文章: [各个类型的IO - 阻塞, 非阻塞,多路复用和异步](https://www.cnblogs.com/fairjm/p/translate-the-various-kinds-of-io-blocking-non.html)  
对应语言的各种IO要看使用了系统提供的哪些类型的API.  
例如Java,在Linux中的AIO是多线程模拟实现的,而在windows下使用了IOCP使用.  

暂时想到的就是这些,后续有一些想法会陆续补充.  


再次提醒一下..本文中的不少观点都是自己梳理理解的,不一定准确甚至正确,如有问题欢迎指正 ^_^



参考资料:  
1.[https://blog.slaks.net/2014-12-23/parallelism-async-threading-explained/](https://blog.slaks.net/2014-12-23/parallelism-async-threading-explained/)  

2.[https://stackoverflow.com/questions/41770985/i-o-blocking-in-green-threads](https://stackoverflow.com/questions/41770985/i-o-blocking-in-green-threads)    

3.[https://stackoverflow.com/questions/748175/asynchronous-vs-synchronous-execution-what-does-it-really-mean](https://stackoverflow.com/questions/748175/asynchronous-vs-synchronous-execution-what-does-it-really-mean)
