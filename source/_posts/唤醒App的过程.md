---
title: 唤醒App的过程
date: 2017-12-05 18:13:49
tags: javascript
---
 
#### 当通过一条链接，打开App或者下载App，我们称这个过程为唤醒App的过程
#### 唤醒App的方法：  
1. 方案一：URL schemeURL
scheme格式为:` <scheme>://<host>:<port>/<path>?<query>`<br />

无论在iOS还是android都支持，只要在原生App中定义scheme，我们就可以通过链接点击，唤醒app，并且还能跳转到指定页面。
<br />

例如taobao://就可以唤醒淘宝app<br />
代码使用一般为：`<a href="taobao://">打开手机App</a>`<br />
js控制代码：      

	const ifr = document.createElement('iframe');
	ifr.src = 'taobao://';
	ifr.style.display = 'none'; 
	document.body.appendChild(ifr);
	//记录唤醒时间
	const openTime = +new Date();
	window.setTimeout(function(){
	  document.body.removeChild(ifr);
	  //如果setTimeout 回调超过2500ms，则弹出下载 
	  if( (+new Date()) - openTime > 2500 ){ 
		window.location = '指定的下载页面'; 
	  } 
	},2000)
	
支持情况：代码简单，都支持，但是在IOS中腾讯的微信和QQ都是

2. 方案二：app intent(未测试)
android平台独有的使用app intent<br />
在chrome android18及其以前版本，第一种方案可以使用，但是之后的版本就不再支持了。而intent可以向后兼容，在android都可以使用。

	
		intents用法：
		intent:
		   HOST/URI-path // Optional host 
		   #Intent; 
		      package=[string]; 
		      action=[string]; 
		      category=[string]; 
		      component=[string]; 
		      scheme=[string]; 
		   end; 
	   
在跳转链接的后面添加
> S.browser_fallback_url=[encoded_full_url]是在当没有下载App时跳转的地址。
 
具体实例：

	intent:
	   //scan/
	   #Intent; 
	      package=com.google.zxing.client.android; 
	      scheme=zxing; 
	   end; 
	 <a href="intent://scan/#Intent;scheme=zxing;package=com.google.zxing.client.android;end"> 扫描手机二维码 </a>
	 
增加当没有下载App时的跳转地址

>  `<a href="intent://scan/#Intent;scheme=zxing;package=com.google.zxing.client.android;S.browser_fallback_url=http%3A%2F%2Fzxing.org;end"> Take a QR code </a> `


3. 方案三：Universal Link(未测试)
Universal Link在IOS中使用，当用户点击Universal Link的时候，iOS会唤醒你的App，并且将一个 NSUserActivity对象发送给你，通过NSUserActivity 对象你可以查询IOS是如何唤醒你的App的。<br />

使用步骤：<br />

1.   添加一个entitlement，它是你的app支持的特定的域名(Entitlement（权限），可以想象成App里用于描述该App可以调用哪些服务的字符串。苹果的操作系统（mac os或者iOS)会通过检查这个串，决定这个应用是否可以调用相关功能。比如iCloud权限，推送服务，健康服务等。)

2.  更新你的app，当它接受到一个NSUserActivity 对象的时候，能够有正确的反应。

&nbsp;&nbsp;&nbsp;&nbsp;具体实例：

	创建一个json格式的apple-app-site-association 文件，没有文件后缀。
	{
	    "applinks":
	        {
	           "apps":  [ ],
	          "details":   [
	                             {
	                               "appID": "9JA89QQLNQ.com.apple.wwdc",
	                               "paths": [ "/wwdc/news/", "/videos/wwdc/2015/*"]
	                            },
	                           { 
	                              "appID":  "ABCD1234.com.apple.wwdc",
	                              "paths": [ "*" ]
	                          }
	                        ]
	       }
	}
	
appID是 bundle id。你可以从你的 [苹果开发账号页面]获取你的团队标识<br />
path作为链接的路径，大小写敏感，并且按照最长字符串匹配。<br />
>  链接:
> Android Intents with Chrome: <br />
> https://developer.chrome.com/multidevice/android/intents<br />
> 
> 
> Universal Link:  https://developer.apple.com/library/content/documentation/General/Conceptual/AppSearch/UniversalLinks.html#//apple_ref/doc/uid/TP40016308-CH12-SW2<br />
> 
> NSUserActivity： <br />
> https://developer.apple.com/documentation/foundation/nsuseractivity

 