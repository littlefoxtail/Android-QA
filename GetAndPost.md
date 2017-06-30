## 1.GET和POST是HTTP请求的两种基本方法,最直观的区别就是GET把参数包含在URL中，POST通过request body传递参数。

传说中的标准答案：
* GET在浏览器回退时时无害的，而POST会再次提交请求。
* GET产生的URL地址可以被Bookmark，而POST不可以。
* GET请求会被浏览器主动cache，而POST不会，除非手动设置。
* GET请求只能进行url编码，而POST支持多种编码方式。
* GET请求参数会被完整保留在浏览器历史记录里，而POST中的参数不会被保留
* GET请求在URL中传送的参数是有长度限制的，而POST没有。
* 对参数的数据类型，GET只接受ASCII字符，而POST没有限制。
* GET比POST更不安全，因为参数直接暴露在URL上，所以不能用来传递敏感信息。
* GET参数通过URL传递，POST放在Request body中

如果按本质来说的话：`GET和POST本质上没有区别`
GET和POST是什么？HTTP协议中的两种发送请求的方法。
HTTP是什么？HTTP是基于TCP/IP的关于数据如何在万维网中如何通信的协议。
HTTP的底层是TCP/IP。所以GET和POST的底层也是TCP/IP，也就是说GET/POST都是TCP链接。GET和POST能做的
事情是一样的。你要给GET加上request body，给POST地上url参数，完全可行。

但是由于HTTP的规定和浏览器/服务器的限制，导致他们在应用过程中体现出一些不同。

## 2.GET产生一个TCP数据；POST产生两个TCP数据包

对于GET方式来说，浏览器会把http header和data一并发送出去，服务器响应200（返回数据）；
而对于POST，浏览器先发送header，服务器响应100 continue，浏览器在发送data，服务器响应200 ok

也就是说，GET只需要汽车跑一趟就把货送到了，而POST得跑两趟，第一趟，先去和服务器打个招呼
“嗨，我等下要送一批货来，你们打开门迎接我”，然后再回头把货送过去。

因为POST需要两步，时间上消耗的更多一点，看起来GET比POST更有效。

1. GET与POST都有自己的语义，不能随便混用。
2. 在网络环境好的情况下，发一次包的时间和发两次包的时间差别基本可以无视。而在网络环境差的情况下，两次包的TCP在验证数据包完整性上，有非常大的优点。
3. 并不是所有浏览器都会在POST中发送两次包，Firefox就只发送一次。


