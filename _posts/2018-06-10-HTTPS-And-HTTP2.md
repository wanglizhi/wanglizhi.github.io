---
layout:     post
title:      "安全的HTTP协议和HTTP2.0"
subtitle:   "Coumputer network skills"
date:       2018-06-10 23:30:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - HTTPS
    - HTTP2.0
    - 计算机网络
---

> 安全的HTTP主要关于与对HTTP的客户端和服务端两端分别进行识别和认证，以及在传输过程加密防盗的问题。而HTTP2.0则将更多的现代网络技术融入的HTTP协议当中，从性能、功能等角度满足当代Web开发的需求。

## 用户识别与Cookie

HTTP最初是一个匿名、无状态的请求响应协议。但是现代的Web站点希望提供个性化服务：个性化问候、推荐、管理信息存档、购物车等会话，所以需要各种针对HTTP协议的用户识别技术，它们分别是：

- HTTP首部
- 客户端IP地址跟踪
- 用户登录
- 胖URL
- Cookie

#### HTTP首部

下面是七种最常见的承载用户相关信息的HTTP请求首部：

- From，用户的E-mail地址。很少使用
- User-Agent，用户的浏览器软件，还会包括操作系统的信息
- Referer，用户来源的URL
- Authorization， 用户名和密码
- Client-IP，客户端IP
- X-Forwarded-For，客户端IP
- Cookie，服务器产生的ID标签

#### 客户端IP、用户登录以及胖URL

用IP来标识用户是不可行的：IP地址是有限，多个用户使用同一个IP则无法区分；同时一般互联网提供商会动态分配IP，不存在唯一性；如果使用代理，则也会产生问题。

HTTP提供了基本登录认证的方式，服务端返回401 Login required，然后用户输入用户名密码，浏览器将其加密添加在Authorization首部，在整个会话期间就可以不用再登录了。

另外一种方式是服务端生成一段包含用户信息的URL用来跟踪用户状态，我们称作胖URL。但这种技术有以下问题：丑陋的URL；无法共享URL，可能会泄露个人信息；破坏缓存，无法使用公共缓存；用户访问其他站点可能会无意逃离胖URL会话；在会话间是非持久的，用户退出登录可能会丢失信息。

#### Cookie

Cookie是当前识别用户，实现持久会话的最好方式。其基本思想就是让浏览器积累一组服务器特有的信息，每次访问服务器时都将这些信息提供给它。

![Cookie](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1530544038&di=7158150c3529f3d7169a74ea9463abcb&imgtype=jpg&er=1&src=http%3A%2F%2Fs8.sinaimg.cn%2Fmw690%2F003O9lNmzy79leBPff197%26amp%3B690)

不同的浏览器会以不同的方式来存储Cookie，通常包含不同的字段：

![](https://img-blog.csdn.net/20170609115917112?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSWRlYWxpdHlfaHVudGVy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- NAME=VALUE：必须的。“$”是保留字符
- Domain（域）：告诉浏览器哪些站点可以看到这个Cookie
- Path（路径属性）：服务器可以通过Path属性将Cookie与部分Web站点关联起来，有的是共享的，有的是独立的。
- Discard：可选，如果提供了则在客户端终止时，会放弃这个Cookie
- Max-Age：可选，用以秒为单位设置Cookie的生存期
- Port：可选，表示可以应用Cookie的端口号，如果提供则必须为匹配的端口提供Cookie
- Secure：可选，如果包含这个属性，就只有在使用SSL连接时才能发送Cookie；

很多Web站点会与第三方厂商达成协议，由其来管理广告。这些广告内嵌在该Web站点，它们会发送持久的Cookie来同步广告内容。

#### Cookie的安全性和隐私

Cookie是可以禁止的，而且可以通过日志分析或其他方式来实现大部分跟踪记录，所以Cookie自身并不是很大的安全隐患，实际上可以将用户信息保存在远程数据库，使用匿名Cookie作为键值来降低敏感数据的传输。切记不要滥用Cookie用来持久存储用户隐私信息。

Cookie的读取操作。Cookie并不提供修改、删除操作。如果要修改某个Cookie，只需要新建一个同名的Cookie，添加到response中覆盖原来的Cookie。

```javascript
function getCookie(name){
    var cookieName=encodeURIComponent(name)+"=",
    cookieStart=document.cookie.indexOf(cookieName),
    cookieValue=null;
    if(cookieStart>-1){
        var cookieEnd=document.cookie.indexOf(";",cookieStart);
        if(cookieEnd==-1){
            cookieEnd=document.cookie.Length;
        }
        cookieValue=decodeURIComponent(document.cookie.substring(cookieStart+document.cookie.length,cookieEnd));
    }
    return cookieValue;
}
```

所以我们需要防止Cookie的篡改：

**HttpOnly &secure属性**

如果在Cookie中设置了"HttpOnly"属性，那么通过程序(JS脚本、Applet等)将无法读取到Cookie信息，这样能有效的防止XSS攻击。

当设置为true时，表示创建的 Cookie 会被以安全的形式向服务器传输，也就是只能在 HTTPS 连接中被浏览器传递到服务器端进行会话验证，如果是 HTTP 连接则不会传递该信息，所以不会被盗取到Cookie 的具体内容。

**防篡改签名**

- 服务端提供一个签名生成算法`secret`
- 根据方法生成签名`secret(wall)=34Yult8i`
- 将生成的签名放入对应的Cookie项`username=wall|34Yult8i`。其中，内容和签名用`|`隔开。
- 服务端根据接收到的内容和签名，校验内容是否被篡改。

**登录时候用cookie的话，安全性问题怎么解决？**

将用户的认证信息保存在一个cookie中，具体如下：  1.cookie名：uid。推荐进行加密，比如MD5(‘站点名称’)等。  2.cookie值：登录名|有效时间Expires|hash值。hash值可以由”登录名+有效时间Expires+用户密码（加密后的）的前几位 +salt” (**salt是保证在服务器端站点配置文件中的随机数)**

这样子设计有以下几个优点：  

1. 即使数据库被盗了，盗用者还是无法登录到系统，因为组成cookie值的salt是保证在服务器站点配置文件中而非数据 库。  
2. 如果账户被盗了，用户修改密码，可以使盗用者的cookie值无效。  
3. 如果服务器端的数据库被盗了，通过修改salt值可以使所有用户的cookie值无效，迫使用户重新登录系统。  
4. 有效时间Expires可以设置为当前时间+过去时间（比如2天），这样可以保证每次登录的cookie值都不一样，防止盗用者 窥探到自己的cookie值后作为后门，长期登录。

#### Session机制

session机制是一种服务器端的机制，服务器使用一种类似于散列表的结构（也可能就是使用散列表）来保存信息。当程序需要为某个客户端的请求创建一个session时，服务器首先检查这个客户端的请求里是否已包含了一个session标识（称为session id），如果已包含则说明以前已经为此客户端创建过session，服务器就按照session id把这个session检索出来使用（检索不到，会新建一个），如果客户端请求不包含session id，则为此客户端创建一个session并且生成一个与此session相关联的session id，session id的值应该是一个既不会重复，又不容易被找到规律以仿造的字符串，这个session id将被在本次响应中返回给客户端保存。

保存这个session id的方式可以采用cookie，这样在交互过程中浏览器可以自动的按照规则把这个标识发挥给服务器。一般这个cookie的名字都是类似于SEEESIONID。但cookie可以被人为的禁止，则必须有其他机制以便在cookie被禁止时仍然能够把session id传递回服务器。经常被使用的一种技术叫做URL重写，就是把session id直接附加在URL路径的后面。还有一种技术叫做表单隐藏字段。就是服务器会自动修改表单，添加一个隐藏字段，以便在表单提交时能够把session id传递回服务器。Cookie已经被证明是三种方法中最方便最安全的.

**会话劫持（Session Hijacking）**

![](https://img-blog.csdn.net/20160119140203442?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

攻击者获取SessionID的方法： 暴力破解：尝试各种Session ID，直到破解为止；预测：如果Session ID使用非随机的方式产生，那么就有可能计算出来；窃取：使用网络嗅探，XSS攻击等方法获得。 最基本的Cookie窃取方式：XSS漏洞。

**防御方法**：

1. 更改Session名称。PHP中Session的默认名称是PHPSESSID，此变量会保存在Cookie中，如果攻击者不分析站点，就不能猜到Session名称，阻挡部分攻击。
2. 关闭透明化Session ID。透明化Session ID指当浏览器中的Http请求没有使用Cookie来存放Session ID时，Session ID则使用URL来传递。
3. 设置HttpOnly。通过设置Cookie的HttpOnly为true，可以防止客户端脚本访问这个Cookie，从而有效的防止XSS攻击。
4. 关闭所有phpinfo类dump request信息的页面。
5.  使用User-Agent检测请求的一致性。但有专家警告不要依赖于检查User-Agent的一致性。这是因为服务器群集中的HTTP代理服务器会对User-Agent进行编辑，而本群集中的多个代理服务器在编辑该值时可能会不一致。

**会话固定（Session Fixation）**

会话固定是一种诱骗受害者使用攻击者指定的会话标识（SessionID）的攻击手段。这是攻击者获取合法会话标识的最简单的方法。会话固定也可以看成是会话劫持的一种类型，原因是会话固定的攻击的主要目的同样是获得目标用户的合法会话，不过会话固定还可以是强迫受害者使用攻击者设定的一个有效会话，以此来获得用户的敏感信息。

![](https://img-blog.csdn.net/20160119140812530?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 重置Session ID的方法同样也有多种，可以是跨站脚本攻击，如果是URL传递Session ID，还可以通过诱导的方式重置该参数，比如可以通过邮件的方式诱导用户去点击重置Session ID的URL，使用Cookie传递可以避免这种攻击。

 使用Cookie来存放Session ID，攻击者可以在以下三种可用的方法中选择一种来重置Session ID。

​       1、 使用客户端脚本来设置Cookie到浏览器。大多数浏览器都支持用客户端脚本来设置Cookie的，例如document.cookie=”sessionid=123”，这种方式可以采用跨站脚本攻击来达到目的。防御方式可以是设置HttpOnly属性，但有少数低版本浏览器存在漏洞，即使设置了HttpOnly，也可以重写Cookie。所以还需要加其他方式的校验，如User-Agent验证，Token校验等同样有效。

​       2、 使用HTML的<META>标签加Set-Cookie属性。服务器可以靠在返回的HTML文档中增加<META>标签来设置Cookie。例如<meta http-equiv=Set-Cookiecontent=”sessionid=123”>，与客户端脚本相比，对<META>标签的处理目前还不能被浏览器禁止。

​       3、 使用Set-Cookie的HTTP响应头部设置Cookie。攻击者可以使用一些方法在Web服务器的响应中加入Set-Cookie的HTTP响应头部。如会话收养，闯入目标服务器所在域的任一主机，或者是攻击用户的DNS服务器。

防御方法同上。

## 基本认证机制

#### HTTP的认证机制

服务器收到一条HTTP请求时，返回一个“认证质询”响应，要求用户提供一些保密信息来表明身份。

- 服务器返回401 Unauthorized，www-Authenticate首部，其中指定了认证算法
- 客户端重新发送请求，附带一个Authorization首部，用来说明认证算法、用户名和密码
- 如果授权正确，则服务器会返回200 OK，有些授权算法会在Authentication-Info首部返回一些附加信息

安全域：WWW-Authenticate： Basic realm:"Corporate Financials"

**realm**指令表示该授权指定的安全域，每个安全域都可以有不同的授权用户集。

#### 基本认证实例

基本认证是最流行的HTTP认证协议，浏览器收到质询时，会打开一个对话框，请求用户输入这个域的用户名和密码，然后将用户名和密码稍加扰码（通常是Base64编码），再用Authorization请求首部返回给服务器。

![](https://dotnetthoughts.net/assets/images/2013/11/basic_auth.png)

基本认证的缺陷：

- 基本认证会通过网络发送用户名和密码，而且很容易解码；
- 即使很难解码，第三方仍然可以捕获用户名密码并且reply给服务器，以获取授权；
- 基本认证没有提供任何针对代理和中间人节点的防护措施；
- 假冒服务器很容易骗过基本认证。

将基本认证与加密数据传输(比如 SSL)配合使用，向恶意用户隐藏用户名和密码，会使基本认证变得更加安全。这是一种常用的技巧 。

#### 摘要认证

对于基本认证的缺陷，摘要认证进行了如下改进：

- 永远不会以明文方式在网络上发送密码
- 可以防止恶意用户捕获并且重放认证的握手过程
- 可以有选择地防止对报文内容的篡改
- 防范常见的攻击方式

![](https://images2015.cnblogs.com/blog/603834/201611/603834-20161127202412393-673697816.png)

单向摘要：摘要是一种单向函数，主要用于将无限的输入值转换为有限的浓缩输出值，常见的摘要函数MD5会将任意长度的字节序列转换为一个128位的摘要。安全散列算法（Secure Hash Algorithm SHA）是另一种常见的摘要函数。

使用随机数防止reply攻击：服务器可以向客户端发送一个nonce随机数的特殊令牌，每次都会变化，客户端在计算摘要时要将这个随机数附加到密码上去。RFC 2617建议的随机数公式：BASE64(time-stamp H(time-stamp ":" ETag ":" private-key))  

## 安全的HTTP

HTTP 的安全版本要高效、可移植且易于管理，不但能够适应不断变化的情况而且还应 该能满足社会和政府的各项要求。我们需要一种能够提供下列功能的 HTTP 安全技术。 

- 服务器认证(客户端知道它们是在与真正的而不是伪造的服务器通话)。 
- 客户端认证(服务器知道它们是在与真正的而不是伪造的客户端通话)。 
- 完整性(客户端和服务器的数据不会被修改)。
- 加密(客户端和服务器的对话是私密的，无需担心被窃听)。 
- 效率(一个运行的足够快的算法，以便低端的客户端和服务器使用)。 
- 普适性(基本上所有的客户端和服务器都支持这些协议)。
- 管理的可扩展性(在任何地方的任何人都可以立即进行安全通信)。
- 适应性(能够支持当前最知名的安全方法)。 
- 在社会上的可行性(满足社会的政治文化需要)。 

#### 加密，签名和证书

关于此处的算法和技术细节不在本文讨论，之前有一篇文章讲[SSL协议](http://wanglizhi.github.io/2016/04/21/SSH-SSL-SSO-Oauth2.0/)，如有需要另外整理。

秘钥：改变密码行为的数字化参数

对称秘钥加密系统：编码和解密使用相同秘钥的算法；常见的对称秘钥（symmetric-key）算法包括：DES、Triple-DES

不对称秘钥加密系统：编码和解密使用不同的秘钥的算法。如RSA算法

公开秘钥加密系统：一种能够使数百万计算机便捷地发送机密报文的系统

数字签名：用来验证报文未被伪造或篡改的校验和

数字证书：由一个可信的组织验证和签发的识别信息

#### HTTPS

HTTPS是最流行的HTTP安全形势，由网景公司首创，以https开头。其原理是使用SSL的输入输出取代TCP的调用，再增加几个调用来配置和管理安全信息。

SSL是一个二进制协议，其流量一般承载在443端口，客户端首先打开一条到Web服务器的TCP连接，然后初始化SSL层，对加密参数进行沟通，并交换秘钥，握手完成后客户端就可以将请求报文发送给安全层了。

![](http://img1.51cto.com/attachment/201304/222345193.gif)

SSL握手主要完成的工作：

- 交换协议版本号
- 选择一个两端都了解的密码
- 对两端的身份进行认证
- 生成临时的会话秘钥，以便加密信道，同时保证性能

SSL支持双向证书，将服务器的证书承载会客户端，再讲客户端的证书会送给服务器。

![](https://upload.wikimedia.org/wikipedia/commons/a/ae/SSL_handshake_with_two_way_authentication_with_certificates.svg)

#### 通过代理以隧道形式传输安全流量

通常情况下，只要客户端开始对数据进行加密，代理就无法读取HTTP的首部了。常用的技术是HTTPS SSL隧道协议，客户端首先明文告知代理，它想要连接的安全主机和端口，连接建立后就可以在该TCP通道上传输SSL数据了。

请求：

CONNECT home.netscape.com:443 HTTP/1.0 

User-agent: Mozilla/1.1N 

响应：

HTTP/1.0 200 Connection established 

Proxy-agent: Netscape-Proxy/1.1 

代码实现 via URLConnection：

```java
public class URLTunnelReader {
11   private final static String proxyHost = "proxy.sg.ibm.com";
12   private final static String proxyPort = "80";
13   
14   public static void main(String[] args) throws Exception {
15      System.setProperty("java.protocol.handler.pkgs",
16                                  "com.sun.net.ssl.internal.www.protocol");
17      //System.setProperty("https.proxyHost",proxyHost);
18      //System.setProperty("https.proxyPort",proxyPort);
19      
20      URL verisign = new URL("https://www.verisign.com");
21      URLConnection urlc = verisign.openConnection(); //from secure site
22      if(urlc instanceof com.sun.net.ssl.HttpsURLConnection){
23                    ((com.sun.net.ssl.HttpsURLConnection)urlc).setSSLSocketFactory
24                         (new SSLTunnelSocketFactory(proxyHost,proxyPort));
25      }
26      
27      BufferedReader in = new BufferedReader(
28                                    new InputStreamReader(
29                                              urlc.getInputStream()));
30      
31      String inputLine;
32      
33      while ((inputLine = in.readLine()) != null)
34         System.out.println(inputLine);
35      
36      in.close();
37   }
38 }
```

参考：[Implement HTTPS tunneling with JSSE](https://www.javaworld.com/article/2077475/core-java/java-tip-111--implement-https-tunneling-with-jsse.html)

#### TLS1.1 VS TLS1.2



## HTTP 2.0



参考：

《HTTP权威指南》

[Cookie小结](https://juejin.im/entry/58c4ccc7128fe1006b37facb)

[Session的攻击手段](https://blog.csdn.net/h_mxc/article/details/50542038)

[深入理解Session和Cookie](http://wanglizhi.github.io/2016/06/01/Session-Cookie/)

[数据包结构分析](https://blog.csdn.net/cjf1002361126/article/details/70232721)