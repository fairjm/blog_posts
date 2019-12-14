---
title: "LockSupport浅析"
date: 2017/11/30 01:43:00
---
最初想有没有必要写这类文章，网上相关的文章很多，有些更为透彻，自己再写一篇不免有重复造轮子的感觉。  
但想想写文除了分享知识外也可以帮助自己总结归纳，也稍稍可以提高点自我满足感。

---  

基本的线程阻塞原语，被用于创建锁和其他同步类上。  

这个类的作用有点类似于`Semaphore`，通过许可证(`permit`)来联系使用它的线程。如果许可证可用，调用`park`方法会立即返回并在这个过程中消费这个许可，不然线程会阻塞。调用`unpark`会使许可证可用。(和`Semaphores`有些许区别,许可证不会累加，最多只有一张） 

因为有了许可证，所以调用`park`和`unpark`的先后关系就不重要了，这里可以对比一下Object的`wait`和`notify`,如果先调用同一个对象的`notify`再`wait`，那么调用`wait`的线程依旧会被阻塞，依赖方法的调用顺序。在一些场景下，`LockSupport`的方法都可以取代`notify`和`wait`。举个简单的例子:  

    final Object lock = new Object();
    Thread t1 = new Thread() {
        public void run() {
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (lock) {
                lock.notifyAll();
            }
        };
    };

    Thread t2 = new Thread() {
        public void run() {
            synchronized (lock) {
                try {
                    lock.wait();
                    System.out.println("I am alive");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
    };
    t1.start();
    t2.start();
这段代码，只有确保t2先进入临界区并执行wait后才能正常打印内容。  
换成`LockSupport`就没有这种担心了。  

    Thread t2 = new Thread() {
        public void run() {
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("t2 end sleep");
            LockSupport.park(this);
            System.out.println("I am alive");
        };
    };
    
    Thread t1 = new Thread() {
        public void run() {
            try {
                Thread.sleep(50);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            LockSupport.unpark(t2);
            System.out.println("I am done");
        };
    };
    t1.start();
    t2.start();
可以先`unpark`再`park`，但如API文档所说，`unpark`需要传入的线程已经启动。在上述`t1.start()`后用`t1.join()`等方法在`t2`启动前调用了`unpark`就无效了。  

但也不意味着`wait`,`notify`就没用了。`wait`是调用对象的方法，和这个对象资源绑定，线程间不需要彼此感知到对方（能获取对方引用），只需要共有这个资源即可，另一方面`LockSupport`可以指定`unpark`的线程，这点也是前者无法做到的，两者间不能完全取代彼此。

`park`方法也有可能在没有"任何理由"的情况下返回，所以通常要在一个循环中使用它并重新判断可以返回的条件。这一点和`Object.wait`方法一样，可以参考[wiki](https://en.wikipedia.org/wiki/Spurious_wakeup)，大致可以理解为底层系统因为优化需要允许出现这样的情况，JVM的线程就是系统线程这种情况也无法避免。可以作为"忙等待"的优化,但是必须要注意要有配对的unpark方法。 
使用模式如下：  

    while (!canProceed()) { ... LockSupport.park(this); }}  


在提供的方法中可以看到有一组传入名为blocker对象作为参数的方法，记录这个对象可以方便许可证监视和诊断工具分析。  

取上述代码，不启动t1的情况直接运行t2进入阻塞，jstack可以看到:  

    "Thread-0" #10 prio=5 os_prio=0 tid=0x000000001b586800 nid=0x37f0 waiting on condition [0x000000001c0ae000]
    java.lang.Thread.State: WAITING (parking)
            at sun.misc.Unsafe.park(Native Method)
            - parking to wait for  <0x00000000d603a6e8> (a test.TestLockSupport$1)
            at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
            at test.TestLockSupport$1.run(TestLockSupport.java:21)  

会将传入的对象打出。  

---  
更多资料：  
https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/LockSupport.html  

http://agapple.iteye.com/blog/970055  

http://blog.csdn.net/hengyunabc/article/details/28126139
