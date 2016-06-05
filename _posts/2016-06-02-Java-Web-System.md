---
layout:     post
title:      "Java大型网站架构"
subtitle:   "Java应用架构，京东响应式亿级商品详情页，淘宝大秒系统"
date:       2016-06-02 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Java Web
    - 系统架构
---

## 深入理解Session与Cookie

HTTP协议对Cookie数量和大小的限制问题？Session在多台服务器之间共享问题？Cookie被盗、Cookie伪造等安全问题？

![](http://7xnovl.com1.z0.glb.clouddn.com/image/f/32/8ed4c512b2cd4802f0dae45b7443f.jpg)

![](http://img.blog.csdn.net/20160309181454083)

#### 理解Cookie

Cookie设计的目的是为了记录用户在一段时间内访问Web应用的行为路径。

###### Cookie属性项

Cookie有两个版本：Version0和Version1。通过"Set-Cookie"和"Set-Cookie2"可以设置响应头标识。

- NAME=VALUE
- Expires：过期时间
- Domain：生成该Cookie的域名，如domain=“tabao.com”
- Path：该Cookie是在当前哪个路径下生成的
- Secure：如果设置了这个属性，只会在SSH连接时才会回传该Cookie

###### Cookie如何工作

```java
//创建Cookie
String getCookie(Cookie[] cookies, String key){
  if(cookies != null){
    for(Cookie cookie : cookies){
      if(cookie.getName().equals(key)){
        return cookie.getValue();
      }
    }
  }
  return null;
}
@Override
public void doGet(HttpServletRequest request, HttpServletResponse response)throws IOException, ServletException{
  Cookie[] cookies = request.getCookies();
  String userName = getCookie(cookies, "userName");
  String userAge = getCookie(cookies, "userAge");
  if(userName == null){
    response.addCookie(new Cookie("userName", "junshan"));
  }
  if(userAge == null){
    response.addCookie(new Cookie("userAge", "28"));
  }
  response.getHeaders("Set-Cookie");
}
```

![](https://raw.githubusercontent.com/wanglizhi/wanglizhi.github.io/master/img/2016-06-01/set-cookie.png)

在构建HTTP返回字节流时是将Header中所有项顺序地写出，而没有进行任何修改。所以浏览器在接受HTTP协议返回的数据时是分别解析每一个Header项的

###### 使用Cookie的限制

Chrome，Cookie数限制50个每个域名，Cookie大小限制4097个字节

#### 理解Session

客户端每次和服务端交互时，只要传回一个ID，这个ID是客户端第一次访问时生成的唯一的。通常是NAME为JSESIONID的Cookie

有三种方式可以让Session正常工作：

- 基于URL Path Parameter 默认支持
- 基于Cookie，如果没有修改Context容器的cookies标识，默认也是支持的
- 基于SSL，默认不支持，只有connector.getAttribute("SSLEnabled")为TRUE时才支持

Session工作时序图

![](https://raw.githubusercontent.com/wanglizhi/wanglizhi.github.io/master/img/2016-06-01/session.png)

#### Cookie安全问题

Cookie值在浏览器端可以被查看、添加、修改，所以安全性存在很大问题

而Session将数据保存在服务端，只是通过Cookie传递一个SessionID而已，所以Session更适合存储用户隐私和重要的数据。

#### 分布式Session框架

分布式Session框架可以解决的问题：

- Session配置的统一管理
- Cookie使用的监控和统一规范管理
- Session配置的动态修改
- Session加密key的定期修改
- 充分的容灾机制，保持框架的使用稳定性
- Session各种存储的监控和报警支持
- Session框架的可扩展性
- 跨域名Session与Cookie如何共享

分布式Session框架架构图

![](https://raw.githubusercontent.com/wanglizhi/wanglizhi.github.io/master/img/2016-06-01/distribute-session.png)

统一通过订阅服务器推送配置可以有效地集中管理资源，省去每个应用都来配置Cookie，简化Cookie管理。

由于应用是一个集群，所以要共享这些Session必须将它们存储在一个分布式缓存中，可以随着写入和读取，而且性能要很好。如MEMCache或者淘宝的开源分布式缓存系统Tair。

#### 表单重复提交问题

要防止表单重复提交，就要标识用户的每一次访问请求，使得每一次访问对服务端来说都是唯一确定的。为了标识用户的每一次访问请求，可以在用户请求一个表单域时增加一个隐藏表单项，每次都是唯一的token

![](https://raw.githubusercontent.com/wanglizhi/wanglizhi.github.io/master/img/2016-06-01/token.png)





**参考** 深入分析Java Web技术内幕 第十章

[深入理解Cookie和Session](http://my.oschina.net/kevinair/blog/192829)