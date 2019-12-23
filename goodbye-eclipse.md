从大一开始自学`java`开始一路用到现在，终于要告别陪伴了快9年的`eclipse`。全部替换为`IDEA`。    

作为一个只索取不付出的码畜说来也有些惭愧，毕竟使用这么久，一行代码也没贡献，一毛钱都没捐献过，最多就是去网上问问对应的问题，提过几个issue。  

因为其他地方都在使用`jetbrains`家的东西，整个切换是比较平滑的，2019.02这个版本已经相当顺滑，依旧使用了`eclipse`的`keymap`，换成`dark plus`的主题之后，使用起来和之前的`eclipse`也没差多少，一天写下来没发现什么问题。  

除了这次较为成功并且不会再退回来（可预见的未来内吧...）之前，还有过三次的迁移，第一次是大二，当时自学`java`是为了安卓开发（结果也就大二学了点...做了几个没人用的app），尝试了下`IDEA`，结果因为小破电脑退了，不过当时的安卓开发在`IDEA`上似乎只是个插件还不是`android studio`，后面两次都是公司里，当时还是`eclipse`主流，现在基本都是`IDEA`了。8G的运存不能支撑同时打开过多项目（一个项目一个窗口，一个项目多module的使用方式），整体使用会比较卡顿。  
但那时候还是很多人迁移过去了，可能信仰的加成比较多，看着同事一个智能提示要卡个几秒才能出来，但还是选择继续用着。  

但之前2，3年电脑的配置就已经足够支撑了，运存也到了16G，硬盘也都是SSD，迁过去已经没有物理环境上的限制了。  

但心里肯定还是对`eclipse`有点感情的，也因为习惯了各种操作快捷键，界面上倒是不怎么考虑，毕竟一个全屏写代码也看不到toolbar和menu之类的东西。  

最终决定迁移的，还是因为文本渲染的问题。  
[https://bugs.eclipse.org/bugs/show_bug.cgi?id=536562](https://bugs.eclipse.org/bugs/show_bug.cgi?id=536562)，首先是之前就有的问题，有一定记录下，注释如果以非英文数字开头，缩进的显示会很奇怪，几个空格会被“压”在一起，但这种情况下，绝大多数连字符是正常的 `->`之类的会正常显示为`→`，还有些情况是缩进正常了，但只有少数连字符能工作了，绝大部分的箭头都阵亡了。  
缩进不正常不打紧不过是注释，我更在意连字符显示，而且打开IDE又是那种十天半个月不会关的，一次打开连字符不正常多试几次就可以了。  
然后上周看到出了新版，也周期性升级到最新，结果发现缩进的问题可能已经fix了（issue也VERIFIED FIXED了），然后这个状态下连字符爆炸。  
在92楼已经有人发现这个问题：  
> It's good, but still doesn't support font ligatures because fMergeNeutralItems is false. so some font such as 'Cascadia Code' or 'Fire Code' has no completely effective. see also [https://bugs.eclipse.org/bugs/show_bug.cgi?id=398656](https://bugs.eclipse.org/bugs/show_bug.cgi?id=398656 "NEW - Text editor should support ligatures")
The following code has been tested, seems ugly but maybe worth considering:
```
    segmentsText.getChars(0, length, chars, 0);
    // enable font ligatures and avoid ligatures between ascii and non-ascii chars
    scriptControl.fMergeNeutralItems = true;
    for (int i = length - 2; i > 0; i--) {
        char i0 = chars[i];
        char i1 = chars[i + 1];
        if (i0 > 255 && i1 < '~' && i1 >= ' ') {
            chars[i + 1] = 'A';
        } else if (i1 > 255 && i0 < '~' && i0 >= ' ') {
            chars[i] = 'A';
            i--;
        }
    }
    OS.ScriptItemize(chars, length, MAX_ITEM, scriptControl, scriptState, pItems, pcItems);
```
这就是所谓的修好了一个bug引入了个新bug。  
去爆栈问也被人说箭头之类的连字符从来没有支持，附上截图后也没了回复，顺带close vote被投了一票。  
也失去了耐心，刚升好级却看到了如此。恰好一些插件还没导入重新配置，索性主力直接切到`IDEA`。
没想到箭头符号成为了压倒骆驼的最后一根稻草/(ㄒoㄒ)/~~

不过不得不说，`eclipse`本身没有网传的那么糟糕，很多卡顿问题并不是来源其本身还是插件，比如装了`spotbugs`又开启了修改后就检查，这样保存文件的时候就会感觉卡卡的，又比如一些优化极差的插件疯狂在`workspace`下的log里刷报错。  
其实很多时候都是有点妖魔化的，一些较为基础的功能都可能被认为缺失，但其实都有，`Ctrl + 3`找一找就可以了（对应`Shift + Shift`）。很多诸如`quick action`可以在`problem`view里批量部分报错和警告的功能也不被人所知。  
但细节上的确有很大差距，已经做了差不多两年的`code mining`依旧是个半成品，以及上面说的这个bug愣是改了一年半（不过受影响的主要是诸如中文之类的字符，本身重视程度也很低吧）；`EJC`有自己的优势但是对`annotation processor`支持比较稀烂，等等。  

也许之后的升级会再回来看看。  
但现在就暂时告别了~
（我也是闲得蛋疼换个IDE也要写篇文... ...

> Written with [StackEdit](https://stackedit.io/).
