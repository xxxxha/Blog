# 一起来做chrome扩展《AJAX请求》
chrome在一次更新之后，出于安全考虑，完全的禁止了content_script从https向http发起ajax请求，即使正常情况下也会在console里给出提示。这对于WEB来讲是好事，但对于扩展来讲就是坏事。平时可以很容易的请求数据，现在就没那么容易了。好在chrome还提供了background_script，利用content_script和background_script之前的通信来实现ajax的请求，就跳过了chrome的这一限制。
## content_script
从名字里就知道，content_script是植入型的，它会被植入到符合匹配的网站页面上。在页面加载完成后执行。content_script最有用的地方是操作网站页面上的DOM。一切平时做前端的一些操作它都可以做，像什么添加、修改、删除DOM，获取DOM值，监听事件等等，都可以很容易的做到。所以，如果想获取人家的登录帐户和密码，就是件非常容易的事，只需要添加content_script，监听帐户和密码的文本框，获得值后将数据发送到自己的服务器就可以了。因此，特别说明，别乱装扩展，特别是不从官方扩展库里下载的扩展。

使用content_script

要使用content_script，需要在manifest.json中配置，如下：
```
{
	"manifest_version": 2,
	"name": "My Extension",
	"description": "Extension description",
	"version": "1.0",

	"content_scripts": {
		"js": [
			"content.js"
		]
	}
}
```
这样，在页面加载完成后，就会加载content.js，在content.js里，就可以控制页面元素。如果要在content.js中使用jquery，需要将jquery文件加到content.js前面，如：

content_script使用jquery
```
{
	"content_scripts": {
		"js": [
			"jquery.js",
			"content.js"
		]
	}
}
```
除了可以加载js，content_scripts里还可以加载CSS文件，这样可以让你的扩展漂亮一点，如：

content_script使用CSS
```
{
	"content_scripts": {
		"js": [
			"content.js"
		],
		"css": ["style.css"]
	}
}
```
在content_scripts中，还有一项重要的设置就是matches，它是用来配置，符合扩展使用的网址，如：我只想这个扩展在打开www.jgb.cn时才启用，那么matches就要这样写：

设置匹配网站
```
{
	"content_scripts": {
		"js": [
			"content.js"
		],
		"css": ["style.css"]
	},
	"matches": [
		"http://*.jgb.cn/*"
	]
}
```
如果还要匹配www.amazon.com，那就加上：
```
{
	"matches": [
		"http://*.jgb.cn/*",
		"http://*.amazon.com/*"
	]
}
```
注意，http只适用于http，像amazon.com这样的站即有http也有https，所以得把https也加上，如下：
```
{
	"matches": [
		"http://*.jgb.cn/*",
		"http://*.amazon.com/*",
		"https://*.amazon.com/*"
	]
}
```
## background_script
它在chrome扩展启动的时候就启动了，做着它的事，而且等待着你给他的指令。它没办法控制页面元素，但可以通过content_script告诉它。ajax同理，如果要在页面打开时向别的服务器请求数据，这时就可以告诉background_script，让它去请求，然后把返回的数据发送给content_script。这样就不会受到浏览器的安全限制影响。

使用background_script

要使用background_script，需要在manifest.json中配置，如下：
```
{
	"manifest_version": 2,
	"name": "My Extension",
	"description": "Extension description",
	"version": "1.0",

	"background": {
		"scripts": [
			"background.js"
		]
	}
}
```
使用jquery和content_scripts同理，需要把jquery文件加到background.js前面，如：

在background_script中使用jquery
```
{
	"background": {
		"scripts": [
			"jquery.js",
			"background.js"
		]
	}
}
```
## 跨域
默认情况下Ajax是不允许跨域的，但扩展提供了跨域的配置，在前一篇《基础介绍》中提到过，那就是permissions，它除了可以让扩展使用chrome的一些功能外，还可以允许JS实现对目录网站的跨域访问，如：
```
{
	"permissions": [
		"http://www.jgb.cn/" // 允许跨域访问www.jgb.cn
	]
}
```
有了以上的配置，这时候就可以来看看怎样通过background_scripts来实现Ajax请求了。
## 向background发送请求
在content_script中向background_script发送请求有好几种方式，这里只列出我常的一种，应该来讲，能满足大多数情况的使用，其它方法，请查询文档，方法如下：
```
chrome.extension.sendMessage({}, callBack);
```
sendMessage()方法，它有两个参数，第一个要发送的数据，就像post请求一样，第二个是回调函数。如在content_script中，点击一个按钮，将一个字符串发送到background_script
```
$(function(){
	$("#button").click(function(){
		chrome.extension.sendMessage({'txt': '这里是发送的内容'}, function(d){
			console.log(d); // 将返回信息打印到控制台里
		});
	});
})
```
## 在background中监听content请求
在background中监听content请求，使用chrome.extension.onMessage.addListener()，示例如下：
```
chrome.extension.onMessage.addListener(function(objRequest, _, sendResponse){});
```
objRequest，即为请求的参数，在上一个例子就是{'txt': '这里是发送的内容'}，可以通过objRequest.txt来获取内容。其实就是一个字典。

sendResponse，为返回值方法，可以将数据返回给content_script，那么一个简单的例子就是：
```
chrome.extension.onMessage.addListener(function(objRequest, _, sendResponse){
	var strText = objRequest.txt;
	// 将信息能过Ajax发送到服务器
	$.ajax({
		url: 'http://www.jgb.cn/',
		type: 'POST',
		data: {'txt': strText},
		dataType: 'json',
	}).then(function(){
		// 将正确信息返回content_script
		sendResponse({'status': 200});
	}, function(){
		// 将错误信息返回content_script
		sendResponse({'status': 500});
	});
});
```
这样一去一来，也就实现content_script向background_script发送请求，并使用background_script执行ajax请求的目的，非常的简单好用

在此基础上，增加一些条件和数据，就可以很好的实现接收，发送数据的操作。比如向自己的服务器请求或发送数据。
## 通过修改chrome启动参数，实现可在https页面向http页面发起ajax请求
除了使用background_script来发起Ajax请求外，还可以通过修改chrome的启动参数来达到这个目的。参数为：--allow-running-insecure-content，操作方法：

1. 右键chrome快捷方式，选择属性
2. 在目标的最后，输入--allow-running-insecure-content，中间有个空格

这样chrome就可以允许你在https页面向http发起ajax请求了。这个方法可以达到目的，但不推荐，因为不科学。
