---
title: rust学习感想
date: 2019-08-16 03:43:15
tags: ["rust"]
index_img: ["/images/rust-logo.png"]  
---
最近刚把[The book](https://doc.rust-lang.org/book/title-page.html)看完了,手有些生,断断续续写点小东西,这边写一下这个歌阶段的一些学习感想,感受.  
文章很水,没有语言上的指导只有自己的一些见解,部分内容还可能是完全错误的,欢迎指正~~.

学习目的主要是想开阔一下视野.  
之前就偶尔会听到别人说rust在设计上有一些特别的地方,并且又由于是一门可以进行系统编程的语言,学习的收获也会比学其他的来得多,就这样开始了.  
  
# 低估的ownership概念
最开始看不管是`ownership`还是`borrowing rules`,感觉设计上很精妙,自己仿佛变成了肉体垃圾收集器,看了书中几个例子就飘了,天真地觉得并不是很困难.这种错觉就像我能手动管理好内存的申请和释放一样,稍微复杂一点点的场景就立马吃瘪.  
之前做练习写了个脚本,用`HashMap`做缓存传入一个函数中,并会在传入的函数中放入值.  
先放出最后的成品:  
```rust
fn handle(client:&Client, map: &mut HashMap<String, String>, email: &str, node_name :&str) {
    let node_id = if let Some(id) = map.get(node_name) {
        if id == "" {
            None
        } else {
            Some(String::from(id))
        }
    } else {
        let r = find_id_by_node_name(&client, node_name);
        if let Some(id) = r {
            map.insert(node_name.to_string(), id.clone());
            Some(id)
        } else {
            map.insert(node_name.to_string(), "".to_string());
            None
        }
    };
    if node_id == None { return };
    let user_id = find_id_by_email(&client, email);
    if user_id == None { return };
    move_to_node(&client, node_id.unwrap().as_str(), user_id.unwrap().as_str());
}
```
最开始key和value都用了`&str`,出现了各种报错比如borrow超过了生命周期,lifetime不一致等,中间为了避免报错一些地方用`String`,用`&String`,随后`ownership moved`之类的错也跟着出现了.  
恶化成了编译器报错,改掉这里,其他地方又爆了原因不一样的错,而且越修越多,报的错越来越看不懂.  
感觉就像被人(编译器)打了一顿,打的人还不断告诉你你这样错了要那样做,那样做了之后又有了其他错.  
这实质上还是自己对于所有权的概念理解太浅薄了,我需要将在函数内部产生的`String`增加到由参数传入的`HashMap`中,`HashMap`的生命周期在定义中就比函数中产生的变量长,而我最开始的泛型类型居然是`&str`,于是就产生了一个borrow活得比他实际的数据更长的问题.  
我要将他带出去,数据的ownership必须到`HashMap`中.(不过由于数据除了放到`HashMap`中还要参与之后的运算,于是通过`clone`等方法增加一些)  

对于`ownership`和`borrow`这两个概念,还是要从引入他们的原因来分析,是为了取代运行时GC的.变量会在离开作用域后自动回收掉指向的数据(这里单单讨论堆,毕竟栈上的数据都是Copy),`ownership`可以想象成一根粗粗的箭头,只能有这一根并且支撑数据的存活(unsafe在此不讨论),而`borrow`则是细细的绳,如果箭头消失了,数据回收,绳就空荡荡(悬空)了.  

# 大小可知 大小不可知  
这个是最为困惑的一个点,主要体现在ch17上的内容.  
看到了"为了编译过"而写的代码形式:
```rust
impl State for Draft {
    fn request_review(self: Box<Self>) -> Box<dyn State> {
        Box::new(PendingReview {})
    }
}
```  
使用`Box`的原因之一是为了解决编译期类型大小不可知的问题,而如果被`Box`持有的数据发生了move,那又把这个编译期大小不可知的问题给暴露出来.  
>  In general, Rust doesn’t allow us to move a value out of a Box<T> because Rust doesn’t know how big the value inside the Box<T> will be: recall in Chapter 15 that we used Box<T> precisely because we had something of an unknown size that we wanted to store in a Box<T> to get a value of a known size.

(ch20的内容)

move Box本身.  
在使用`trait object`中会常用到的一点.  
可能需要多写一些代码才能更深入理解了.  

# 提供充足的声明式支持  
这个是比较惊喜的一点,rust的库中提供了很多声明式风格的api,语言本身也有模式匹配的支持,写起来和其他有运行时的语言一样便利(但要时刻记着`ownership`等概念).  
这是语言设计和API设计上的选择,感觉相比某些语言还是相当友好的.  
此外,使用了一些额外的类库也感觉到了API上的轻便,使用舒适,比如`serde_json`的`Value`设计.  
相比java的类库`jackson`里的`JsonNode`,其子类的`get`实现会返回`null`,导致get之后必须检查一下(虽然可以用`Optional`一路串下去),`Value`有`Value::Null`(在使用[]的情况下),如果`jackson`设计不存在返回子类假设有个`MissingNode`的实例可能就会好很多.  
不过这个具体类库具体分析了.  

# 语言本身的扩展性  
这个其实支持macro的语言都有.  
感觉这是一个很棒的方向,语言本身提供足够使用的语法即可,其他靠macro进行扩展,一个小内核+插件扩展的方式.  
这个之后还要具体学习.  

---  

以上只是当前学习过程中的一点感想,随着不断学习可能会有不同的思考.  
rust的学习过程还是比较有意思的,特别对于之前学的所有语言都是带运行时,不用太过在意gc,带来的思想上的冲击还是比较大的,特别是被编译器教做人... ... ...  
此外补充一些收集的学习资料:  

pingCAP的课程   
https://university.pingcap.com/talentplan.shtml  
对应的github:  
https://github.com/pingcap/talent-plan/tree/master/rust/building-blocks


200行的绿色线程:  
https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/

GB模拟器:  
https://blog.ryanlevick.com/DMG-01/public/book/  
nes模拟器:  
http://www.michaelburge.us/2019/03/18/nes-design.html

cheat sheet:  
https://cheats.rs/

exercism:  
https://exercism.io/tracks/rust  

async book:   
https://rust-lang.github.io/async-book/index.html

http://arewegameyet.com/



