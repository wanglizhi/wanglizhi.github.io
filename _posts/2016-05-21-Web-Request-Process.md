---
layout:     post
title:      "深入Web请求过程"
subtitle:   "涉及浏览器、HTTP协议解析、DNS解析、CDN涉及工作等内容"
date:       2016-05-21 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Java
    - Java Web
    - HTTP
    - DNS
    - CDN
---

## B/S网络架构概述

B/S网络架构基于HTTP交互数据，采用无状态的短连接通信方式，其架构如下，既要满足海量用户的访问请求，又要保持用户请求的快速响应。

当一个用户在浏览器输入www.taobao.com时，发生的操作有：首先请求DNS把**域名解析**成IP地址，根据IP地址找到服务器，向服务器发起一个get请求，由服务器决定返回默认的数据资源给访问的用户。其中，多台服务器需要一个**负载均衡设备**来平均分配所有用户的请求；请求的数据是存储在**分布式缓存**里还是静态文件或者**数据库**里；当数据返回浏览器时，发现还有一些静态资源（css、JS或图片），又会发起另外的HTTP请求，这些请求可能会在**CDN**上。

![](http://img.blog.csdn.net/20140409123513515?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2VuZHlfNjk1NzY3NTA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

B/S架构的原则：

- 互联网上所有资源都要用一个URL来表示
- 必须基于HTTP协议与服务端交互
- 数据展示必须在浏览器中运行

## 如何发起一个请求

**发起一个HTTP请求的过程，就是建立一个Socket通信的过程。**

例如用HttpClient调试；

Linux下curl + URL命令也可以简单地发起HTTP请求，加上-I选项可以查看HTTP协议头信息，通过-HI可以在访问UTL时增加HTTP头

## HTTP协议解析

理解HTTP协议，最重要的要熟悉HTTP Header。

**常见的HTTP请求头：**

- Accept-Charset：用于指定客户端接受的字符集
- Accept-Encoding：用于指定可接受的内容编码，如gzip.deflate
- Accept-Language：用于指定一种自然语言，如zh-cn
- Host：指定被请求资源的Internet主机和端口号，如www.taobao.com
- User-Agent：客户端将它的操作系统、浏览器和其他属性告诉服务器
- Connection：当前连接是否保持，如：Keep-Alive

**常见的HTTP响应头：**

- Server：使用的服务器名称，如Apache/1.3.6(Unix)
- Content-Type：发送给接收者的实体正文的媒体类型，如：text/html;charset=GBK
- Content-Encoding：告诉浏览器服务端采用什么压缩编码
- Content-Language：资源所用的自然语言
- Content-Length：正文的长度，用以字节方式存储的十进制数字表示
- Keep-Alive：保持连接时间，如：timeout=5,max=120

**常见的HTTP状态码：**

- 200：客户端请求成功
- 302：临时跳转，跳转地址通过Location制定
- 400：客户端请求有语法错误，不能被服务器识别
- 403：服务器收到请求，但是拒绝提供服务
- 404：请求的资源不存在
- 500：服务器发生不可预期的错误

#### 浏览器的缓存机制

当使用Ctrl+F5组合键刷新一个页面时，在HTTP的请求头中会增加一些请求Pragma:no-cache和Cache-Control:no-cache，告诉服务器要获取最新的数据而不是缓存。

**1、Cache-Control/Pragma**

这个HTTP Head字段用于指定所有缓存机制在整个请求/响应中必须服从的命令，如果知道该页面是否为缓存，不仅可以控制浏览器，还可以控制和HTTP协议相关的缓存或代理服务器。

Cache-Control字段的可选值：

- Public：所有内容都将被缓存，在响应头中设置
- Private：内容只缓存到私有缓存中，在响应头中设置
- no-cache：所有内容都不会被缓存，在请求和响应头中设置
- no-store：所有内容都不会被缓存到缓存或Internet临时文件，在响应头中设置
- must-revalidation/proxy-revalidation：如果缓存内容失效，请求必须发送到服务器/代理重新验证，请求头中设置
- max-age=xxx：缓存内容在xxx秒后失效，只能在HTTP1.1中使用

Pragma字段作用和Cache-Control类似。Cache-Control请求字段被各个浏览器支持的较好，优先级也比较高

**2、Expires**

Expires:Sta, 25 Feb 2012 12:22:17 GMT，超过这个时间值后，缓存的内容将失效。浏览器在发请求之前坚持页面的该字段，如果过期就重新向服务器发起请求。

**3、Last-Modified/Etag**

一般用于表示一个服务器上的资源的最后修改时间，资源可以是静态（静态内容自动加Last-Modified字段）或动态内容（如Servlet提供了一个getLastModified方法用于检查动态内容是否更新），通过最后修改可以判断当前请求的资源是否是最新的。

一般服务器响应头返回如：Last-Modified: Sat, 25 Feb 2012 12:55:04 GMT，告诉浏览器页面的最后修改时间，浏览器再次请求时在请求头加入If-Modified-Since:Sat, 25 Feb 2012 12:55:04 GMT字段，询问当前缓存页是否罪行，如果是最新的就返回304状态码，不会传输新数据。

Etag字段作用是让服务器给每个页面分配一个唯一的编号，然后通过这个编号来区分当前这个页面是否是最新的。

## DNS域名解析

#### 域名解析过程

![](http://img.blog.csdn.net/20140409143631359?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2VuZHlfNjk1NzY3NTA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

1. 浏览器检测缓存，有没有解析过的IP地址。域名被缓存的时间可以通过TTL属性设置，一般几分钟到几小时不等，太长如果IP变化不能正确解析，太短导致用户每次都要重新解析
2. 如果浏览器缓存没有，浏览器会查找操作系统缓存。etc\host文件，黑客通过修改域名解析到指定的IP地址上，导致这些**域名被劫持**。
3. 网络配置中有**“DNS服务器地址”**，设置的是本地区的域名服务器（LDNS），它们一般会缓存域名解析结果，大约80%的域名解析都到这里完成。
4. 如果LDNS没有命中，直接到**Root Server**域名服务器请求解析
5. 根域名服务器返回给本地域名服务器一个所查询域的主域名服务器（gTLD Server）地址，如.com，.cn，.org等顶级域名服务器，全球只有13台左右
6. LDNS再向gTLD服务器发送请求
7. 接受请求的gTLD服务器查找并返回此域名对应的Name Server域名服务器地址。通常就是你注册的域名服务器，由这个域名提供商的服务器完成。
8. Name Server会查询存储的域名与IP的映射关系，连同**TTL值**返回给DNS Server域名服务器
9. 返回该域名对应的IP和TTL值，LDNS Server会缓存这个域名和IP对应关系，缓存时间由TTL值控制
10. 把解析结果返回给用户，用户根据TTL值缓存在本地系统缓存中，域名解析过程结束。

#### 清除缓存的域名

基本上Local DNS Server的缓存时间是TTL控制的，很难人工介入，但本机缓存可以清除。

Linux下 /etc/init.d/nscd restart来清除缓存

JVM也会缓存DNS的解析结果，在InetAddress类中完成，包括正确解析结果缓存和失败解析结果缓存，缓存时间由两个配置项控制。java.security文件中 networkaddress.cache.ttl和networkaddress.cache.negative.ttl

注意：如果要用InetAddress类解析域名时，一定要是单例模式。

#### 几种域名解析方式

- A记录，Address，指定域名对应的IP地址，可以将多个域名解析到一个IP地址，不能将一个域名解析到多个IP地址
- MX记录，Mail Exchange，将某个域名下的邮件服务器指向自己的Mail Server
- CNAME记录，别名解析，即为一个域名设置一个或者多个别名，如将taobao.com解析到xulingbo.net
- NS记录，为某个域名指定DNS解析服务器
- TXT记录，为某个主机名或域名设置说明

## CDN工作机制

CDN（Content Delivery Network）就是内容分布网络，是构筑在Internet上的一种先进的流量分配网络。用户可以**就近**取的所需要的内容，提高访问网站的速度。

CDN = 镜像 + 缓存 + 整体负载均衡

目前CDN以缓存静态数据为主，如CSS，JS，图片和静态页面。

#### CDN架构

![](http://img.blog.csdn.net/20140409151931984?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2VuZHlfNjk1NzY3NTA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

假设用户访问cdn.taobao.com，首先向Local DNS服务器发起请求，经过迭代解析后会到域名的注册服务器解析，一般每个公司都会有一个DNS解析服务器。这时，DNS解析服务器把它从新CNAME解析到另外一个域名，这个域名最终指向CDN全局中的DNS负载均衡服务器，再由这个GTM来分配是哪个地方的访问用户，返回给离这个访问用户最近的CND节点。

如果节点不存在请求资源，就会再去源站上获取这个文件

#### 负载均衡

Load Balance，提高服务器响应速度和利用效率，避免单点失效，解决网络拥塞，实现地理位置无关性等。

通常有三种负载均衡架构，分别是链路负载均衡、集群负载均衡和操作系统负载均衡。

**链路负载均衡**：通过DNS解析成不同的IP，然后用户根据这个IP来访问不同的目标服务器。某台web server如果挂掉，很难更新用户的域名解析结构，可能会无法访问这个域名。

**集群负载均衡：**一般分为硬件负载均衡和软件负载均衡。

**硬件负载均衡**一般使用一台昂贵的的硬件设备来转发请求，要求性能好，通常一主一备，但是当访问剧增时不能动态扩容。

![](http://img.blog.csdn.net/20140409153606171?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2VuZHlfNjk1NzY3NTA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**软件负载均衡**是最普遍的一种负载方式，直接使用PC搭建，缺点是请求要经过多次代理服务器，增加网络延时。

![](http://img.blog.csdn.net/20140409153646593?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2VuZHlfNjk1NzY3NTA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

上面两台是LVS，在网络层利用IP地址进行地址转发。下面使用HAProxy，根据访问用户的HTTP请求头来进行负载均衡，如可以根据**不同的URL**来将请求转发到特定机器或者根据**用户的cookie**信息来指定访问的机器。

操作系统负载均衡就是利用操作系统级别的软终端或硬中断来达到负载均衡，如可以设置多队列网卡等来实现。



**参考** 深入分析Javac Web技术内幕 第一章

