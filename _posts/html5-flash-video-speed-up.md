---
title: "几种方式加速网页视频播放速度"
date: 2017/10/21 18:56:00
---
现在有不少视频网站,自带了播放加速功能,例如油管,bilibili,慕课等等.节省了很多看视频的时间,特别是看一些技术教程类的视频,不管是念ppt还是手把手演示.  

在自己付费的一些网站中,一些是自带播放器不支持视频加速的.因为已经被加速惯坏,变得很不习惯,今天特意研究了一下,对于几种形式给出一些解决方法.  

# html5播放器  
主要标志是`<video>`,这种是最方便实现加速的,因为原生支持.  
主要依靠这两个属性:  
>defaultPlaybackRate* (float): The playback speed at which the video should be played  
playbackRate* (float): The current playback speed: 1.0 is normal, 2.0 is two times faster forward, -3.0 is three times faster backwards and so on   

可以自己在console中修改实现([来源](https://stackoverflow.com/questions/3027707/how-to-change-the-playing-speed-of-videos-in-html5)):  

    /* play video twice as fast */
    document.querySelector('video').defaultPlaybackRate = 2.0;
    document.querySelector('video').play();

    /* now play three times as fast just for the heck of it */
    document.querySelector('video').playbackRate = 3.0;  

对应的插件chrome市场也有,用的比较多的有`video speed controller`.  
[市场地址](https://chrome.google.com/webstore/detail/video-speed-controller/nffaoalbilbmmfgbnbgppjihopabppdk)  

[github](https://github.com/igrigorik/videospeed)

# flash播放器  
比较头疼的一个,chrome市场中也没有找到对应的插件.  
比较早之前傲游浏览器做过类似功能,实现的具体讨论见 [新版傲游主打的「马上看」是如何实现对视频广告的快进的？](https://www.zhihu.com/question/22797950?sort=created).  
不过这个功能已经凉了无法使用.  
网上其他比较多的方案是[myspeed](http://www.enounce.com/myspeed).不过是收费软件,并且一切反馈里表示看一会儿就挂了.  
那只能用下载的方式拿到源文件,然后再用本地播放器比如potplayer等加速了.  
但一些网站采用付费课程的形式,做了相应的举措防止用户下载,这样就感觉无计可施了.  

不过有一些迂回的方式,这类网站还在使用flash没有升级到html5,但在mac上使用flash并不方便,所以他们可能为了mac或者iOS的用户采用了其他的播放方式.  

今天遇到了一个这样的例子:  

	if(isMobile() && isSafari() )
	{
		
	 $("#player").html('<video src="'+vodadd_m3+'" class="videoimg"  controls="controls"></video>');	
	}
	else
	{
		var fls = flashChecker();
		 
		if (fls!=null && !fls.f)
		{
			alert("请安装flash播放控件,并使用QQ浏览器观看！");
			 
		}
	 
	 $("#player").html(' <embed src="'+vodadd+'" quality="high"  class="videoimg"   allowScriptAccess="always" allownetworking="all" '+
	'allowfullscreen="true" type="application/x-shockwave-flash" wmode="window" pluginspage="http://www.macromedia.com/go/getflashplayer"></embed>');	
 
	}

js里有增加`video`标签的代码! 这样就转换到了上面的html5播放器的解决方法了.  
`isMobile`和`isSafari`的判断用UA就可以糊弄过去了.(chrome F12设备选择ipad即可, 或者用`Mozilla/5.0(iPad; U; CPU iPhone OS 3_2 like Mac OS X; en-us) AppleWebKit/531.21.10 (KHTML, like Gecko) Version/4.0.4 Mobile/7B314 Safari/531.21.10`)  
然额并打不开,因为他又使用了`m3u8`这种格式,所以才在js里判断safari.  
但有对应chrome插件支持,在应用市场安装`Native HLS Playback`在页面点击`play embedded`即可.  
这样因为之前又装了`video speed controller`,就出现了调整速度的选项.  
至此在这种特殊情况下的flash播放问题也解决了.  

不过针对只有flash播放器的环境还是没找到合适的解决方法.  
但可以通过查看判断的代码,改变UA等方式再找找有没有可以迂回的方式.
