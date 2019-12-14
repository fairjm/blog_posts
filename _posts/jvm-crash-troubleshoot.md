---
title: "[翻译]Java排错指南 - 5 确定崩溃何地发生"
date: 2018/12/05 16:53:00
---
原文地址: https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/crashes001.html

这几天公司其他组遇到了一个segmentation fault的问题,找到了这个官方文档,基于Java8,感觉不错就翻译了下.  
一些地方翻译比较生硬,如有问题请麻烦指正~^_^ by fairjm

---  


# 5.1 确定崩溃何地发生
这一节提供了一些例子来演示如何使用错误日志来找到崩溃的原因,并且给出一些排查这些问题的建议.  

错误日志的头指出了错误的类型和有问题的帧(frame),thread stack指出了当前的线程和堆栈轨迹.查看[Header Format](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/felog003.html#BABFFJBB)  

Crash in Native Code

Crash in Compiled Code

Crash in HotSpot Compiler Thread

Crash in VM Thread

Crash Due to Stack Overflow  

## 5.1.1 本地代码崩溃
如果致命错误日志(fatal error log)指出的问题帧来自于本地库,那么可能是本地库或者JNI库代码存在bug.这种崩溃当然也有可能是其他原因造成的,但是分析这个库和其他的core file或者crash dump是一个很好的开始.  
以下是一个致命错误日志的头:  
```
# An unexpected error has been detected by HotSpot Virtual Machine:
#
#  SIGSEGV (0xb) at pc=0x417789d7, pid=21139, tid=1024
#
# Java VM: Java HotSpot(TM) Server VM (6-beta2-b63 mixed mode)
# Problematic frame:
# C  [libApplication.so+0x9d7]
```  
在这个例子中,`SIGSEGV`发生在线程执行`libApplication.so`中的代码.  
一些其他例子是Java VM的本地库导致的.在下面的例子中,`JavaThread `在`_thread_in_vm `状态中失败了(表明它正在执行Java VM代码)  
```  
# An unexpected error has been detected by HotSpot Virtual Machine:
#
#  EXCEPTION_ACCESS_VIOLATION (0xc0000005) at pc=0x08083d77, pid=3700, tid=2896
#
# Java VM: Java HotSpot(TM) Client VM (1.5-internal mixed mode)
# Problematic frame:
# V  [jvm.dll+0x83d77]

---------------  T H R E A D  ---------------

Current thread (0x00036960):  JavaThread "main" [_thread_in_vm, id=2896]
 :
Stack: [0x00040000,0x00080000),  sp=0x0007f9f8,  free space=254k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
V  [jvm.dll+0x83d77]
C  [App.dll+0x1047]          <========= C/native frame
j  Test.foo()V+0
j  Test.main([Ljava/lang/String;)V+0
v  ~StubRoutines::call_stub
V  [jvm.dll+0x80f13]
V  [jvm.dll+0xd3842]
V  [jvm.dll+0x80de4]
V  [jvm.dll+0x87cd2]
C  [java.exe+0x14c0]
C  [java.exe+0x64cd]
C  [kernel32.dll+0x214c7]
 :
```  
在这个例子中,虽然出问题的帧是VM的,线程栈显示一个本地例子(native routine)`App.dll`已经被VM调用(可能是通过JNI).  

解决这个本地库崩溃的第一步是调查本地库发生崩溃的那段源代码.  
- 如果本地库是由你的应用程序提供,那么调查本地库的源代码.大量的问题可以通过在运行应用程序时使用`-Xcheck:jni`参数被识别.查看[The -Xcheck:jni Option](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/clopts002.html#CHDDEGBI).  
- 如果这个本地库是由其他供应商提供,被你的程序所使用,那么就向他们提供bug报告和致命错误日志信息.  
- 如果本地库是来自于JRE的(比如awt.dll,net.dll或其他的),那有可能你遇到了一个库或者API的bug.那么就尽可能获取足够的信息提交一个bug并指名库名称.你可以在JRE的发行版中的 jre/lib 或 jre/bin目录中找到JRE的库.  

如果可能的话,你可以通过本地debugger attach到core file或crash dump的方式来排查本地库崩溃.取决于你所使用的系统,本地debugger有`dbx`,`gdb`,或`windbg`.查看[ Native Operating System Tools](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr020.html#BABBHHIE)

## 5.1.2 编译代码崩溃  
如果致命错误日志指出崩溃发生在编译代码(compiled code),那有可能你遇到了一个编译器导致的不正确代码生成的bug.你可以通过问题堆栈的类型是`J`(代表一个compiled java frame)来识别.  
```  
# An unexpected error has been detected by HotSpot Virtual Machine:
#
#  SIGSEGV (0xb) at pc=0x0000002a99eb0c10, pid=6106, tid=278546
#
# Java VM: Java HotSpot(TM) 64-Bit Server VM (1.6.0-beta-b51 mixed mode)
# Problematic frame:
# J  org.foobar.Scanner.body()V
#
:
Stack: [0x0000002aea560000,0x0000002aea660000),  sp=0x0000002aea65ddf0,
  free space=1015k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
J  org.foobar.Scanner.body()V

[error occurred during error reporting, step 120, id 0xb]
```  
注意:该例子无法获得一个完整的线程栈.输出的"error occurred during error reporting" 表示问题发生在试图获取堆栈跟踪(可能是栈损坏了).  

通过更换编译器的方式可能可以临时解决(例如使用HotSpot Client VM取代HotSpot Server VM,或反过来)或者从编译中排除掉导致崩溃的方法.在这个例子中,将64位的Server VM换成32位的Client VM可能会有用.  

更多可能的规避措施见[Working Around Crashes in the HotSpot Compiler Thread or Compiled Code](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/crashes002.html#CIHDIBJA).

## 5.1.3 HotSpot编译器线程崩溃  
如果核心错误日志显示的当前线程是一个叫`CompilerThread0`,`CompilerThread1`或`AdapterCompiler`的`JavaThread`,那么你可能遇上了编译器bug.可能解决方式和上一节一样.(懒得复读了...)  

## 5.1.4 VM线程崩溃  
如果核心错误日志显示的当前线程是`VMThread`,那么查看一下在`THREAD`那节包含`VM_Operation `的那一行.`VMThread `是特殊的的HotSpot VM线程.它执行一些特殊的工作比如GC.如果`VM_Operation`表明操作是GC,那么你可能遇到了诸如堆损坏的问题.  

包括GC问题之外,它同样可能是一些其他问题(例如编译器或者运行时bug)导致的对象引用在堆中处于一个不完整和不正确的状态.这种情况下,收集尽可能多的环境信息和尝试可能的规避方案.如果发现和GC有关,你可能通过修改GC配置的方式来临时保证正常运行.  

对于更多的规避方案,查看[Working Around Crashes during Garbage Collection](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/crashes002.html#CIHEEBDG)

## 5.1.5 爆栈崩溃  
java语言的一个栈溢出通常会导致线程抛出烦人的`java.lang.StackOverflowError`异常.另一方面,C和C++写入超过了栈的结束会引起一个栈溢出.这是一个致命的错误,会导致进程终止.  

在HotSpot实现中,Java方法和C/C++本地代码共享栈帧,即用户本地代码和VM自身.  
Java方法产生的代码会检查栈离栈的结束是否会有固定距离可用的空间,所以本地代码的调用可以不担心是否会超过栈空间.  
到栈结束的距离被称为`Shadow Pages`.这个大小取决于所在平台,shadow pages在3到20页之间.  
这个距离是可以调整的,所以应用使用到了本地代码想要比默认更大的距离时可以增加shadow page的大小.  
增加的参数是`-XX:StackShadowPages=n`,n设定为比当前平台默认值大.  

如果你的应用遇上了segmentation fault但是没有core file或致命错误日志参见[Appendix A](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/felog.html#fatal_error_log_vm),或者在windows上`STACK_OVERFLOW_ERROR `,或者得到一个消息"An irrecoverable stack overflow has occurred",这说明超过了`StackShadowPages`,需要更大的空间.  

如果你增加了`StackShadowPages`,你可能也需要使用`-Xss`参数增加默认的线程栈大小.增加默认的线程栈大小可能会减少可创建的线程数,所以请小心选择这个数值.线程栈的大小于不同平台上在256KB到1024KB.  

```
# An unexpected error has been detected by HotSpot Virtual Machine:
#
#  EXCEPTION_STACK_OVERFLOW (0xc00000fd) at pc=0x10001011, pid=296, tid=2940
#
# Java VM: Java HotSpot(TM) Client VM (1.6-internal mixed mode, sharing)
# Problematic frame:
# C  [App.dll+0x1011]
#

---------------  T H R E A D  ---------------

Current thread (0x000367c0):  JavaThread "main" [_thread_in_native, id=2940]
:
Stack: [0x00040000,0x00080000),  sp=0x00041000,  free space=4k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
C  [App.dll+0x1011]
C  [App.dll+0x1020]
C  [App.dll+0x1020]
:
C  [App.dll+0x1020]
C  [App.dll+0x1020]
...<more frames>...

Java frames: (J=compiled Java code, j=interpreted, Vv=VM code)
j  Test.foo()V+0
j  Test.main([Ljava/lang/String;)V+0
v  ~StubRoutines::call_stub
```  
你可以从例子中获得以下信息:  
- 异常是`EXCEPTION_STACK_OVERFLOW`  
- 线程的状态是`_thread_in_native`,表示线程在执行native或者JNI代码.  
- 线程信息中,可用的空间仅仅只有4KB(windows系统中的单页).另外线程指针(sp)在`0x00041000`,很接近栈结束`0x00040000`.  
- 输出的本地栈显示一个递归的本地方法是这个问题的原因.`...<more frames>...`表明还有更多的帧存在但是没有输出.输出只限于100帧.  

---  

相关资料:  
[Do we need Unsafe in Java?](https://www.slideshare.net/AndreiPangin/do-we-need-unsafe-in-java)  

[Shortest code that raises a SIGSEGV](https://codegolf.stackexchange.com/questions/4399/shortest-code-that-raises-a-sigsegv?page=1&tab=votes#tab-top)  

[Best way on how to solve/debug JVM crash (SIGSEGV)
](https://stackoverflow.com/questions/36109313/best-way-on-how-to-solve-debug-jvm-crash-sigsegv)

openjdk相关bug:  
[JVM Crash in # Problematic frame: # J 569 C2 java.lang.Long.getChars(JI\[C)V (221 bytes) @ 0x00007fb9b618dfd8 [0x00007fb9b618dc40+0x398]](https://bugs.openjdk.java.net/browse/JDK-8171429)  

[JVM crash. Problematic frame: J 4518 C2 java.lang.Long.getChar](https://bugs.openjdk.java.net/browse/JDK-8144565)
