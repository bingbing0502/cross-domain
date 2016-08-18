# cross-domain
随便在一个网页的控制台下，向自己的node的服务器发送请求，就能发现跨域现象。

如果浏览器直接发送ajax请求，在不同域会发生跨域。
而直接调用js文件是不存在跨域的。因此可以使用`<script>、<iframe>`等标签加载js文件解决跨域问题。

解决方案：

##jsonp


jsonp就是利用`<script>`标签，传递一个callback参数给服务端，然后服务端返回数据时会将这个callback参数作为函数名来包裹住JSON数据，这样客户端就可以随意定制自己的函数来自动处理返回数据了。

例子:  

客户端

```

function jsonp(url,succalbck,para,failcalbck){
	var scr = document.createElement('script');
	scr.src = url + '?' + para  + '=' + succalbck;
	ducoment.head.appendChild(scr);
}

```

服务端 （Node的controller层）

```

exports.index = function*() {
  let callback  = this.query.callback ;
  this.body = callback + '(json参数)'
}

```
注意：这里的callback指的是para变量


##iframe
###主域不相同

####document.domain

优点：简单实现

缺点：只能用于同一个主域下不同子域之间的跨域请求。

方法实现：父页面通过`ifr.contentWindow`就可以访问子页面的window，子页面通过`parent.window`或`parent`访问父页面的window，接下来可以进一步获取dom和js。

例子：
在dev.data.com/a.html

```
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8">
    <title>document.domain</title>
</head>
<body>
<iframe id="ifr" src="http://test.data.com:9080/b.html"></iframe>
<script>
	window.domain = 'data.com' ;
	function domain(){console.log('domain')}
		window.onload = function(){
			document.querySelector('#ifr').contentWindow.crossdo();
 }
</script>
</body>
</html>
```
在test.data.com/b.html

```
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8">
    <title>document.domain</title>
</head>
<body>
<script>
	window.domain = 'data.com' ;
	function crossdo(){console.log('crossdo')}
		window.onload = function(){
			parent.domain()
 }
</script>
</body>
</html>
```
###主域不相同

####window.name


方法实现：如果在浏览器控制台设置了`window.name`的值再跳转其他页面时，发现`window.name`的值还是存在。   
同时发现，大部分浏览器不允许修改不同域的父窗体的值，也就是说父窗体虽能修改子窗体的值，但反过来由子窗体不能修改父窗体值却不成立。
为了父子窗体建立联系，需要在子窗体中增加与父窗体的同域的代理。

例子：
在dev.data1.com/a.html

```
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8">
    <title>document.domain</title>
</head>
<body>
<iframe id="ifr" src="http://test.data2.com:9080/b.html"></iframe>
<script>
	function domain(ifm){
		var loc	= ifm.location ,
			 data 	= '' ; 
		//当同域才能获取到它的host值
		if (loc.host){
			data = ifm.name;
			ifm.location.href = 'http://test.data2.com/b.html'//跳转回去继续接受数据
		}else{
			return false;
		}
		
	}
		window.onload = function(){
			var ifm = window.frames[0];
			setInterval(domain,2000,ifm); 
 }
</script>
</body>
</html>
```
在test.data2.com/b.html

```
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8">
    <title>document.domain</title>
</head>
<body>
<script>
	window.name = 'something';
    location = "http://dev.data1.com/c.html"; //跳转到和a同域的c页面
</script>
</body>
</html>
```

####window.hash

解决方案：因为改变hash值并不会刷新页面，因此也可以利用特性进行跨域。而且需要与父窗口同域的代理去修改父窗口的hash值
在dev.data1.com/a.html

```
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8">
    <title>document.domain</title>
</head>
<body>
<iframe id="ifr" src="http://test.data2.com:9080/b.html"></iframe>
<script>
	function domain(){
		var hash	= '' ,
			 data 	= location.hash ? location.hash.substring(1) : hash; ; 
		if (hash !== data){
			hash = data ;
		}
		
	}
		window.onload = function(){
			setInterval(domain,2000); 
 }
</script>
</body>
</html>
```
在test.data2.com/b.html

```
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8">
    <title>document.domain</title>
</head>
<body>
<script>
	var ifr = document.createElement('iframe');
        ifr.style.display = 'none';
        ifr.src = 'dev.data1.com/c.html#a=1&b=2'; //必须跟a.html同域
        document.body.appendChild(ifr);
</script>
</body>
</html>
```
在dev.data1.com/c.html

```
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8">
    <title>document.domain</title>
</head>
<body>
<script>
		 parent.parent.location.hash = self.location.hash.substring(1);
</script>
</body>
</html>
```

####postMessage

这是html5引入的message的api,更方便、有效、安全解决这些安替

postMessage(data,origin)接受两个参数

data：要传递的数据。部分浏览器只能接受字符串，因此需要用`JSON.stringify()`对参数进行序列化

origin:String类型，指定目标窗口的源，协议+主机+端口号[+URL]

例子：
在dev.data1.com/a.html

```
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8">
    <title>document.domain</title>
</head>
<body>
<iframe id="ifr" src="http://test.data.com/b.html"></iframe>
<script>
window.onload = function () {
    var ifr = document.querySelector('#ifr');
    ifr.contentWindow.postMessage({a: 1}, '*');
}
window.addEventListener('message', function(e) {
    console.log('bar say: '+e.data);
}, false);
</script>
</body>
</html>
```
在test.data2.com/b.html

```
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8">
    <title>document.domain</title>
</head>
<body>
<script>
window.addEventListener('message', function(e){
    console.log('foo say: ' + e.data.a);
    e.source.postMessage('get', '*');
}, false)

</script>
</body>
</html>
```

有几个重要属性

data：顾名思义，是传递来的message  
source：发送消息的窗口对象   
origin：发送消息窗口的源（协议+主机+端口号）


###CORS

前后端跨域交互终于迎来了官方的支持，只需在后端设置允许前端请求域，即可轻松实现跨域。
CORS适用与IE8+及现代浏览器。现代浏览器通过XMLHttpRequest，IE通过XDomainRequest实现。
事件句柄（标*的事件XDomainRequest不支持）

后端设置Access-Control-Allow-Origin，标识允许请求的来源域比如http://foo.com，*表示允许来自任意域的请求

####XMLHttpRequest

function domain(url){
	var xhr = new XMLHttpRequest();
	xhr.onload = function () {
	    console.log(xhr.responseText);
	}
	xhr.onerror = function() {
	    console.log('Request Error!');
	}
	xhr.open('get', url);
	xhr.send(null);
}
####XDomainRequest

function domain(url){
	var xhr = new XDomainRequest();
	xhr.onload = function () {
	    console.log(xhr.responseText);
	}
	xhr.onerror = function() {
	    console.log('Request Error!');
	}
	xhr.open('get', url);
	xhr.send(null);
}
