# Http
## http协议
超文本协议。主要用来规范浏览器和服务端的行为的。
### 通信过程
以TCP（传输层）作为底层协议，默认端口80，通信过程主要如下：
1. 服务端在80端口等待客户请求。
2. 浏览器发起服务端的TCP连接。
3. 服务接收来自浏览器的TCP连接。
4. 浏览器和Web服务器交换HTTP消息。
5. 关闭TCP连接。
## HTTPS协议
是HTTP的加强安全版本。基于HTTP，也是用TCP作为底层协议，并额外使用SSL/TLS协议作为加密和安全认证。默认端口443。
HTTPS协议中，SSL通道通常使用基于密钥的加密算法，密钥长度通常是40bit或123bit。
### 核心--SSL/TLS协议
之所以能达到较高的安全性要求，就是结合了SSL/TLS和TCP协议，对通信数据进行加密，解决了HTTP数据透明问题。
#### SSL和TLS没有太大区别
SSL指安全套接字协议（Secure Sockets Layer）。SSL3.0进一步升级，新版本被命名为TSL 1.0。
#### SSL/TLS的工作原理
##### 非对称加密
非对称加密--一个公钥，一个私钥。在通信时，私钥仅由解密者保存，公钥由任何一个想与解密者通信的发送者（加密者）所知。
##### 对称加密
使用SSL/TLS进行通信的双方需要使用非对称加密方案，但是非对称加密设计了较为复杂的数学算法，在实际通信过程中，计算的代价较高，效率太低，因此SSL/TLS实际对消息的加密使用的是对称加密。
##### 数学签名
数学签名要解决的问题，是防止证书被伪造。
数字签名，是CA给服务器颁发证书时，使用散列技术生成一个摘要。CA使用CA私钥对该摘要进行加密，并附在证书下方，发送给服务器。

1. Http与Https有什么区别

https是一种通过计算机网络进行安全通信的传输协议。Https经由http进行通信，但是利用SSL/TLS来加密数据包。https开发的主要目的，是提供对网站服务器的身份验证，保护交换数据的隐私与完整性。

https（全程：Hypertext Transfer Protocol over Secure Socket Layer），是以安全为目标的http通道