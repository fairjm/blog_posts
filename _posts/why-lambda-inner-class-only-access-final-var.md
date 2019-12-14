---
title: "lambda和匿名内部类使用外部变量为什么要语义final?"
date: 2018/10/19 16:00:00
---
今天群里讨论`java`的`lambda`实现.  
后来不断衍生谈到了为什么lambda和匿名内部类只能使用语义`final`的外部变量.  

最开始以为是java的lambda实现问题,编译期魔法会把外部引用作为参数传入所以在内部变化也影响不了下次调用的值,所以就干脆final了,如果用类的属性来保管这个变量就可以了.  
```python
In [64]:  def outer(a:int):
    ...:      def inner():
    ...:          nonlocal a
    ...:          a = a + a
    ...:          return a
    ...:      return inner
    ...:

In [65]: x = outer(1)

In [66]: x
Out[66]: <function __main__.outer.<locals>.inner>

In [67]: x()
Out[67]: 2

In [68]: x()
Out[68]: 4

In [69]: x()
Out[69]: 8

```
举例就是这种情况  
lambda用参数传入外部int,如果在方法里修改了,下次调用这个lambda依旧是以前的值.  

后来又去看了眼匿名内部类的实现
```java
public class InnerTest {

    public static void main(String[] args) {
        String name = "123";
        Runnable r = new Runnable() {
            @Override
            public void run() {
                System.out.println(name);
            }
        };
        r.run();
    }
}
```  
```
 InnerTest$1(java.lang.String val$name);
     0  aload_0 [this]
     1  aload_1 [val$name]
     2  putfield InnerTest$1.val$name : java.lang.String [12]
     5  aload_0 [this]
     6  invokespecial java.lang.Object() [14]
     9  return
      Line numbers:
        [pc: 0, line: 1]
        [pc: 5, line: 11]
      Local variable table:
        [pc: 0, pc: 10] local: this index: 0 type: new InnerTest(){}
      Method Parameters:
        final synthetic val$name
```
构造方法字节码明明都存下来了呀...为什么那时候就要求`final`了  

查阅了一下,发现这样有个很大的问题.  
这个外部变量在匿名内部类初始化之后就被固定了下来,之后他如果被重新赋值(引用类型内部状态修改除外),就会出现内部无法看见外部,外部也无法看见内部的问题... ...  
举个简单的例子:  
```python
In [101]: def outer(x):
     ...:     def inner():
     ...:         nonlocal x
     ...:         x = x+1
     ...:         print("inner"+str(x))
     ...:     inner()
     ...:     print("outer"+str(x))
     ...:     x = x+1
     ...:     inner()
     ...:

In [102]: outer(1)
inner2(inner内部+1)
outer2(外部看到这个变化)
inner4(外部+1 内部+1)
```

这段在py下能正常工作的代码,如果java没有`final`限制的话,就会变成
```
inner2(inner内部+1)
outer1(外部看不到inner变化)
inner3(inner内部+1)
```  
所以这个约定和闭包实现没关系...  
还是考虑在定义变量的作用域下规避掉因为java实现问题导致上面的结果返回...我既然没有`nonlocal`这样的机制(毕竟不支持引用传递....),索性就用`final`限制起来.  

呜呼哀哉  

---  
参考资料:

[https://stackoverflow.com/questions/4732544/why-are-only-final-variables-accessible-in-anonymous-class](https://stackoverflow.com/questions/4732544/why-are-only-final-variables-accessible-in-anonymous-class) 

[Closure_(computer_programming)](https://en.wikipedia.org/wiki/Closure_(computer_programming))
