---
layout:     post
title:      "HTTP协议概览"
subtitle:   "Introduction to HTTP"
date:       2018-05-20 23:30:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - 计算机网络
    - HTTP
---

## HTTP协议的历史

HTTP（Hypertext Transfer Protocol 超级文本传输协议）是访问万维网的载体，也是与我们接触最密切的网络协议，REST API的概念就强依赖于HTTP协议。从之前的文章[互联网协议概览](http://wanglizhi.github.io/2016/07/01/Internet-Protocol-Suite/)可以看到，它位于应用层，基于TCP/IP协议做可靠的传输和路由，本身设计是为了文本以及多媒体内容的共享，后来随着Web技术的发展，HTTP也提供了诸如缓存、持久连接、代理、隧道、Cookie、安全认证等功能，并且随着富客户端化的发展，计算力从服务器迁移到了客户端，从而对HTTP有了更高的要求，HTTP2.0应运而生。



![](https://ss3.bdstatic.com/70cFv8Sh_Q1YnxGkpoWK1HF6hhy/it/u=3134925206,3072947099&fm=15&gp=0.jpg)

Internet以及HTTP的诞生

1） 1967年ACM提出构建连接计算机的小型网络——高级研究计划署网络（ARPANET），之后其成员开始研究各种连接、传输、可靠等需求，提于1973年提出了传输控制协议（TCP）的概念。

2）1977年，TCP分成两个协议：传输控制协议（TCP）和因特网协议（IP）,IP负责处理数据报的路由选择，TCP负责高层次的功能，如分段、重组、检错。

3）之后美国出现各种网络出现，1981年CSNET-计算机科学网；1986年NSFNET-国家科学基金网络，连接美国5个超级计算机中心；1991年，IBM、Merit和Verizon搭建了高级网络服务网（ANSNET）。

4）Web是由欧洲原子核研究委员会（CERN）的Tim Berners-Lee发明的，他最初希望借助多文档之间关联形成的超文本，连接成可互相参阅的万维网（WWW）。借助HTML、HTTP和URL三项构建技术，1990年11月，CERN成功研发了世界上第一台Web服务器和Web浏览器。

HTTP协议版本

- HTTP/0.9，HTTP1991原型版本，只支持GET方法，不支持多媒体内容、各种HTTP首部，很快就被1.0取代了
- HTTP/1.0，1.0是第一个得到广泛使用的HTTP版本，添加了版本号、多媒体、各种首部等。使得Web页面包含生动图片和交互式表格。
- HTTP/1.0+，90年代中叶，很多流行的Web客户端和服务器向HTTP添加了各种特性，包括持久的keep-alive连接、虚拟主机支持，代理支持。
- HTTP/1.1，1999年HTTP/1.1发布，矫正HTTP设计中的结构性缺陷，性能优化，是目前正在使用的版本。
- HTTP/2.0，作为下一代HTTP协议，目前还在开发中，可见HTTP1.1在20年的发展中依然稳定。2010年Google发布SPDY，旨在解决HTTP的性能瓶颈，缩短Web页面加载时间。2011年WebSocket成为独立的协议标准，实现Web浏览器和Web服务器之间的全双工通信。HTTP2.0主要在压缩、多路复用、TLS义务化、协商、Pull/Push、流量控制、WebSocket等方面研究。

## HTTP报文

HTTP报文由三部分组成：描述报文的起始行、包含属性的首部（header）、可选的数据主体（body）。而报文又可以分为两类：请求报文和响应报文。

![](https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/u=1745884650,2770517668&fm=15&gp=0.jpg)

### 起始行

所有HTTP报文都以一个起始行开始，请求报文起始行说明了**要做些什么**；响应报文起始行说明**发生了什么**。

请求行：方法 URL HTTP版本

响应行：HTTP版本 状态码 状态描述

常用的HTTP方法：

- GET，从服务器获取一份文档
- HEAD，只从服务器获取文档的首部
- POST，向服务器发送需要处理的数据
- PUT，将请求的主体部分存储在服务器上
- TRACE，对可能经过代理服务器传送的报文进行追踪
- OPTIONS，决定可以在服务器上执行哪些方法
- DELETE，从服务器删除一份文档

常见的状态码：

- 1XX：信息提示。100 Continue，客户端应用程序只有在避免向服务器发送一个服务器无法处理或使用的大实体时才应该使用100；101 Switching Protocols；
- 2XX：成功。200 OK；201 Created；202 Accepted，已接收但还未执行；
- 3XX：重定向。300 Multiple Choices；301 Moved Permanently；302 Found；303 See Other；307 Temporary Redirect


- 4XX：客户端错误。401 Unauthorized；404 Not Found；403 Forbidden；405 Method Not Allowed
- 5XX：服务端错误。500 Internal Server Error；501 Not Implemented；502 Bad Gateway；503 Service Unavailable；504 Gateway Timeout；505 HTTP Version Not Supported

### 首部

首部是报文中的附加信息，是一个键值对列表。通常根据规范分为几类：

- 通用首部：与报文相关的最基本的信息；如Connection，Date，MIME-Version，Update， Via。通用缓存首部 Cache-Control
- 请求首部：Client-IP，From，Host，Referer，User-Agent，Accept首部，Expect条件首部，Authorization，Cookie，Max-Forward，Proxy-Connection
- 响应首部：Age，Server，Proxy-Authenticate，Set-Cookie
- 实体首部：描述主体长度和内容；Allow，Location
- 扩展首部：规范中没有的首部

关于首部，需要和具体的如缓存、代理等功能结合理解。

## 连接管理

几乎所有的HTTP通信都是由TCP/IP承载，TCP协议保证数据的可靠性。

### TCP连接

对于HTTP开发者来说，TCP/IP是透明的，但是理解连接的具体行为还是必要的，下图描述了Web浏览器通过TCP连接与Web服务器进行交互。

![](http://img.my.csdn.net/uploads/201302/21/1361411658_1543.png)

**TCP协议段**

![](http://img.my.csdn.net/uploads/201302/21/1361411666_7052.png)

操作系统提供了TCP套接字编程接口，这些API隐藏了所有底层网络协议的握手细节，以及TCP数据流和IP分组之间的分段和重装细节。

![](https://ss3.bdstatic.com/70cFv8Sh_Q1YnxGkpoWK1HF6hhy/it/u=1826830040,2697404243&fm=27&gp=0.jpg)

### TCP性能考虑

HTTP事务的性能在很大程度上取决于底层TCP通道的性能，基于TCP的性能特点，可以更好地理解HTTP的连接优化特性。HTTP事务的时延主要有几个原因：

- 客户端根据URI确定IP地址和端口，如果没有本地缓存，需要通过DNS解析IP地址，可能需要数十秒。
- 建立TCP连接，可能1-2秒
- 发送HTTP请求
- Web服务器返回HTTP响应

TCP相关的时延主要包括：

1）TCP连接建立握手，小的HTTP事务可能会在TCP建立上花费超过50%时间，通过重用连接可以减小这种影响。

2）延迟确认，每个TCP段都有一个序列号和数据完整性校验和，通过确认机制确保数据没有损坏。延迟确认算法会在一个特定的窗口时间将输出确认放在缓冲区，以寻找能够捎带它的输出数据分组，如果那段时间没有其他输出分组，则单独发送确认信息。

2）TCP慢启动拥塞控制，TCP连接会随着时间进行自我协调，起初限制速度，当数据成功传输后，会提高传输速度，所以新连接的传输速度回慢一些。

3）数据聚集的Nagle算法，试图在发送一个分组前将大量TCP数据绑定在一起，以提高网络效率。该算法会导致小的HTTP报文等待额外数据产生时延；同时Nagle算法会阻止数据发送，直到有确认分组到达。可以通过设置TCP_NODELAY来禁用Nagle算法，但要确保会向TCP写入大块数据。

4）TIMS_WAIT时延和端口耗尽。当某个TCP端点关闭TCP连接时，会在内存维护一个小的控制块，记录最近关闭的IP地址和端口号，通常保存2分组左右，确保这段时间内不会创建具有相同地址和端口号的新连接。在进行性能测试的机器上，客户端每次连接到服务器上时都会使用新的端口连接到Server的80端口，以此来确保连接的唯一性，但是由于可用的端口有限（比如60000个），在2min内连接无法重用，所以连接率就在500次/秒，超过的话就会导致TIME_WAIT端口耗尽问题。

### HTTP连接的处理

1. **串行事务**，每个请求都需要新建TCP连接，连接时延和慢启动时延会叠加起来；

![](https://images2017.cnblogs.com/blog/983980/201712/983980-20171206224045441-1160571597.png)

2. **并行连接**不一定快，需要打开大量连接消耗内存资源，消耗客户端网络带宽，浏览器会对并行连接总数限制（通常是4个）。同时TCP慢启动的延迟依然存在。

![](https://images2017.cnblogs.com/blog/983980/201712/983980-20171206224455081-2043211810.png)

3. **持久连接**允许HTTP事务处理结束后保持TCP打开，支持连接重用，可以避免建立连接的延迟和慢启动的拥塞延迟，从而更快地传输数据。

![](https://images2017.cnblogs.com/blog/983980/201712/983980-20171206231649503-800727744.png)

对比持久连接的两种类型：HTTP/1.0+ "keep-alive"连接和HTTP/1.1 "persistent"连接

**Keep-alive**：该首部只是请求将连接保持在活跃状态，客户端和服务端可以随时关闭空闲的Keep-alive连接。

**限制和规则**：

①必须客户端发送一个Connection：Keep-alive请求首部来激活Keep-alive连接；

②该首部必须随请求的报文一起发送；

③只有在确定实体主体部分大小的情况下，连接才能保持在打开状态；

④代理和网关必须执行Connection首部的规则；对于不认识Connection首部的**哑代理**来说，它只是进行转发但并不保持连接，所以第二次再请求的时候会被挂起。Netscape提出加入Proxy-Connection新首部，效果如下图所示，可以解决单个盲中继问题。

![](https://img-blog.csdn.net/20171019171841127?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ29nemY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

HTTP/1.1停止了对keep-alive连接的支持，而使用了一种持久连接的模型取代了它。

**Persistent**：该首部默认情况下是激活的，除非特别指明，否则HTTP/1.1假定所有连接都是持久的。

**限制和规则**：

① 如果需要在事务处理结束后将连接关闭，则应用程序必须向报文中显式的添加一个Connection-close首部；

② 只有当连接上所有报文都有正确的、自动以报文长度时，连接才能持久保持，Body长度与Content-Length一致；

③ 每个持久连接都只适用于一跳传输；

④ 应用程序可以在任意时刻关闭连接，但应该能够从异步关闭中恢复，重试这条请求；

⑤ 一个客户端对任何服务器或代理最多只能维护2条持久连接以防止服务器过载；

4. **管道化连接**

通过共享TCP连接发起并发的HTTP请求，这也是在持久连接的基础上对性能的一种优化。

原理：在响应到达前，将多条请求放入队列，在高延时网络条件下，可以降低网络环回时间，提高性能。

**限制和规则**：

① 如果HTTP客户端无法确认连接是持久的，就不应使用管道连接；

② 必须按照与请求相同的顺序回送HTTP响应；

③ 客户端必须做好连接会在任何时刻关闭的准备，以及重发所有未完成的管道化请求；

④ HTTP客户端不应用管道化的方式发送非幂等性请求（比如POST）；

![](https://images2017.cnblogs.com/blog/983980/201712/983980-20171209135135136-1039612245.png)



##Web服务器



##代理



##缓存



##集成点：网关、隧道及中继



## HTTPS



参考：

《图解HTTP》

《HTTP权威指南》

《计算机网络教程》