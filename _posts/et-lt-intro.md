---
title: 理一理Edge-triggered和Level-triggered
date: 2019-10-03 22:46:52
tags: ["OS","rust"]
---
最近群里又在讨论java的NIO,提到了NIO使用的lt而netty使用JNI在linux和MacOS/BSD中封装了et. 
之前对这两个概念笼统了解了下,并没有去查阅额外资料,仅限知道lt在缓冲区还有数据的情况下就会被poll出来,而et则需要有新的请求/事件发生.  
这次查阅了点资料,汇总一些数据来简单(毕竟也没有那么深入..)谈谈这两个概念.  

# 简介
这两个名词应该来源于电气,用于激活电路.摘取一段爆栈网的回答:
> Level Triggering: In level triggering the circuit will become active when the gating or clock pulse is on a particular level. This level is decided by the designer. We can have a negative level triggering in which the circuit is active when the clock signal is low or a positive level triggering in which the circuit is active when the clock signal is high.  
Edge Triggering: In edge triggering the circuit becomes active at negative or positive edge of the clock signal. For example if the circuit is positive edge triggered, it will take input at exactly the time in which the clock signal goes from low to high. Similarly input is taken at exactly the time in which the clock signal goes from high to low in negative edge triggering. But keep in mind after the the input, it can be processed in all the time till the next input is taken.  

[来源.](https://electronics.stackexchange.com/questions/21886/what-does-edge-triggered-and-level-triggered-mean#21891)  
翻译就是LT是根据设定的阈值来控制是否激发而ET是根据信号的高到低或低到高这个变化来控制.  

字面意思上level就是水平指的是某值,而edge是边,是值和值之间的变迁.  
用来控制操作是否进行的一种机制.

对应于epoll中的概念也类似.  
拿文档中的例子:
> 1. The file descriptor that represents the read side of a pipe (rfd) is registered on the epoll instance.
> 2. A pipe writer writes 2 kB of data on the write side of the pipe.
> 3. A call to epoll_wait(2) is done that will return rfd as a ready file descriptor.
> 4. The pipe reader reads 1 kB of data from rfd.
> 5. A call to epoll_wait(2) is done.

使用LT的话,因为依旧有1KB残留所以wait会立即返回开始下一次操作,而ET这次变迁已经结束了,但因为没有处理完所以后续变化也不会再返回了.(Since the read operation done in 4 does not consume the whole buffer data, the call to epoll_wait(2) done in step 5 might block indefinitely.)  

ET在使用上建议遵循以下的规则:  
> i   with nonblocking file descriptors; and  
> ii  by waiting for an event only after read(2) or write(2)
                  return EAGAIN.  

非阻塞的文件描述符(FD或SD)以及要read或write在返回EAGAIN的情况下才开始这个描述符的下一次事件监听.(对于网络IO来说也可能是EWOULDBLOCK, 表示被标记为非阻塞的操作会发生阻塞的异常)  

相对来说使用ET的要求和操作会需要更严谨一些,当然LT写得有问题,比如漏了一个事件的处理,可能会导致出现跑满CPU的死循环.

# 简单使用
这里使用mio来演示这两种的使用方式.代码直接改的官网TcpServer的示例.  
完整代码见:[main.rs](https://github.com/fairjm/feel_boring/blob/master/edge_level_test/src/main.rs).  

首先最外层是一样的,选择感兴趣的事件注册到poll中,poll进行wait等待可用事件.事件的处理上就有些许差异.  

首先是LT:  
```rust
fn read_level(stream: &mut TcpStream, poll: &Poll) -> Result<()> {
    let mut connection_closed = false;
    loop {
        let mut buf = vec![0u8; 8];
        match stream.read(&mut buf) {
            Ok(0) => {
                connection_closed = true;
                break;
            }
            Ok(_n) => {
                println!("======come new read======");
                println!("read:{:?}", String::from_utf8(buf));
                // just return is ok for level
                return Ok(());
            }
            Err(ref err) if would_block(err) => {
                println!("would_block happened");
                break;
            }
            Err(ref err) if interrupted(err) => {
                println!("interrupted happened");
                continue;
            }
            Err(err) => return Err(err),
        }
    }
    if connection_closed {
        // must have this one
        poll.deregister(stream)?;
        println!("{:?} Connection closed", stream.peer_addr());
    }
    Ok(())
}
```
在`OK(_n)`中,当前只读取部分byte并返回是可以的,并且如果未读完会立即发起下一个事件.但要注意如果连接关闭需要移除,不然可能会有奇怪的问题.(在本机上如果不移除可能会导致无限的read事件)  

而ET的话就需要读到出现WouldBlock
```rust
fn read_edge(stream: &mut TcpStream) -> Result<()> {
    let mut connection_closed = false;
    loop {
        let mut buf = vec![0u8; 8];
        match stream.read(&mut buf) {
            Ok(0) => {
                connection_closed = true;
                break;
            }
            Ok(_n) => {
                println!("======come new read======");
                println!("read:{:?}", String::from_utf8(buf));
            }
            Err(ref err) if would_block(err) => {
                println!("would_block happened");
                // edge rely this to return, without this or just return after read(like level)
                // the connection will not be read anymore
                break;
            }
            Err(ref err) if interrupted(err) => {
                println!("interrupted happened");
                continue;
            }
            // Other errors we'll consider fatal.
            Err(err) => return Err(err),
        }
    }
    if connection_closed {
        println!("{:?} Connection closed", stream.peer_addr());
    }
    Ok(())
}
```
这里无法像LT一样直接读完部分返回,需要在出现阻塞的情况下才能继续操作.  
因为mio相对底层,所以描述起来和epoll文档也类似,java的NIO也类似,但因为只提供了LT所以就没有ET什么事了.  

# 选择
ET的问题主要是需要更严谨的操作,而LT是更频繁的wait唤醒.  
在某种程度上,ET更加'惰性'而LT更加积极,你如果不处理或者还不想处理就会反复收到事件,除非你取消注册他.  
所以在不少java的NIO代码中,常常会有channel在读后取消读注册写,在写后取消写注册读的代码,当可以写,但是写的内容还没准备好时,使用LT会遇到不少问题(所以一般是准备写的内容已经好了,再给channel注册上写的兴趣).  
ET在读上会更加麻烦,不读完等到返回EAGAIN,该描述符可能之后的事件触发就会有问题.  
具体还需要看自己的需求和场景.
此外多线程场景下,在同一个描述符上等待ET是保证只会唤醒一个线程,但要注意多个不同的数据块请求可能会导致在一个FD/SD上返回多个事件,需要使用`EPOLLONESHOT`来确保只返回一个(这个flag是接受到一个事件后就解绑和FD/SD关系的).  

java中只支持LT,netty通过JNI实现了ET,选择的原因在一个邮件里提到了一些[Any reason why select() uses only level-triggered notification mode?](http://mail.openjdk.java.net/pipermail/nio-dev/2008-December/000329.html).  
大致的意思是ET和I/O提供的方法更加耦合,可能是为了更高的兼容性放弃了这个机制的提供吧.  

这边就进行了简单的一些资料整合,没有涉及到更深的内容,如果文中有什么错误欢迎评论.


参考资料:  
https://netty.io/wiki/native-transports.html  

http://man7.org/linux/man-pages/man7/epoll.7.html

https://github.com/tokio-rs/mio