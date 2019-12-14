---
title: "记一次尴尬的git reset丢失分支故障"
date: 2018/06/29 21:29:00
---
最近...似乎一直在踩坑...  
也不是什么故障,只是把一个分支的功能弄没了,之后在reflog里找到又恢复了.  

产生原因是有同事错误地把分支B merge到了分支A并push.  
我直接在分支A上reset到了merge前的一个节点(但这个节点其实是B分支的).  
这导致分支A的头跑到了B分支上,A本来那个分支没有引用就丢了.  
先说下解决方法,git reflog找到那个reset,或者直接用`git fsck --no-reflogs --unreachable`找到那个commit之后直接再reset回去即可.  

主要原因是git merge之后的log会显示另一个分支的commit记录(毕竟没有真正的分支...).  
这里简单重现下:  
```
*   dacbc52 (HEAD -> master) Merge branch 'a'
|\
| * 7b83cf6 (a) a commit
* | 9f4488b master commit
|/
* 47a0ab3 init
```   
这样的一个分支情况,master merge了a节点产生2126这个commit.  
这个时候看一下master的log,信息如下:   
```
commit dacbc528153d48f3a210a9673ebc8148cfdb50d3
Merge: 9f4488b 7b83cf6
Date:   Fri Jun 29 21:21:10 2018 +0800

    Merge branch 'a'

commit 7b83cf6664cd5b84c5f66ef99f0fb39fb5ab57bd
Date:   Fri Jun 29 21:19:59 2018 +0800

    a commit

commit 9f4488ba008889b583205fc9405f4760150a2337
Date:   Fri Jun 29 21:18:26 2018 +0800

    master commit

commit 47a0ab3a70e7b4a663f99bc23542a51a90af152c
Date:   Fri Jun 29 21:18:03 2018 +0800

    init
```  
可以看到a节点上的commit也被包含在了里面....(PS:之前我们使用的是hg,分支的概念有所不同,merge之后查看当前分支只是会显示本分支和merge节点,并且也没有git里的fast-forward概念)  

误操作就是因为要撤销master的merge,执行了`git reset --hard 7b83c`(因为刚好是merge下面的第一个节点)  
执行后的结果:  
```
* 7b83cf6 (HEAD -> master, a) a commit
* 47a0ab3 init
```
可以看到default的结果已经不复存在了  

找回的话
```
> git reflog
7b83cf6 HEAD@{0}: reset: moving to 7b83c
dacbc52 HEAD@{1}: merge a: Merge made by the 'recursive' strategy.
```  
可以直接reset回`dacbc52`再进行正常操作  

主要原因还是对于git的理解不深再加上之前使用hg的惯性产生了误判.  

---  
参考资料:  
https://git-scm.com/book/zh/v1/Git-%E5%86%85%E9%83%A8%E5%8E%9F%E7%90%86-%E7%BB%B4%E6%8A%A4%E5%8F%8A%E6%95%B0%E6%8D%AE%E6%81%A2%E5%A4%8D  
https://git-scm.com/book/zh/v1/Git-%E5%88%86%E6%94%AF-%E5%88%86%E6%94%AF%E7%9A%84%E6%96%B0%E5%BB%BA%E4%B8%8E%E5%90%88%E5%B9%B6
