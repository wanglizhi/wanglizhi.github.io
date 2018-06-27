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

SSL支持双向证书，将服务器的证书承载会客户端，再将客户端的证书会送给服务器。

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

#### TLS协议的发展

SSL协议1.0版本发布于1995年，主要由网景公司开发，存在很多漏洞。后来在SSL3.0的基础上规范了TLS标准，1999年TLS1.0标准颁布，2006年TLS1.1颁布 改动不大是一个过渡版本；2008年发布TLS1.2，在互联网中长期占据主流标准得到广泛使用，TLS是保障网络传输安全最重要的安全标准之一。

然而广泛的应用也使得TLS成为了攻击的“众矢之的”，这些攻击或利用TLS设计本身存在的不足（如幸运十三攻击[1]、三次握手攻击[2]、跨协议攻击 [3]等），或利用TLS所用密码原语本身的缺陷（如RC4加密 [4]、RSA-PKCS#1 v1.5加密 [5]等），或利用TLS实现库中的漏洞（如心脏出血攻击[6]等）。面对这一系列的攻击，一直以来我们采取的措施是“打补丁”，即针对新的攻击做新的修补。然而，由于TLS的应用规模过于庞大，不断地打补丁在如此大规模的实际应用中并不容易全面实施。除此之外，交互双方必须运行复杂的TLS握手协议才能开始传输信息，很多情况下我们希望在握手轮数和握手延迟方面可以有更多的选择。出于以上以及其他种种因素的考虑，IETF从2014年开始着手制定一个“clean”的TLS1.3。

**TLS1.2的优化**

密码学的发展导致MD5/SHA-1等哈希函数的安全性降低了，MD5甚至可以称为不安全的哈希函数必须被剔除掉。

- PRF伪随机数生成函数中的MD5/SHA-1被替换为加密套件指定的RPF函数。
- 需要做数字签名的元素（Element），签名由MD5/SHA-1组合方式替换为单个用户指定的Hash算法。
- TLS_RSA_WITH_AES_128_CBC_SHA必须实现
- 增加了HMAC-SHA256套件。
- 删除了IDEA和DES加密套件

增加了新的密码学操作类型 Authenticated Encryption，TLS1.1及以前包括四种密码学操作类型：

A）数字签名  digital signing

B）流加密      stream cipher encryption

C）块加密      block cipher encryption

D）公钥加密  public key encryption

【TLS1.2】增补了一种

E）authenticated encryption with additional data (AEAD) encryption。authenticated encryption是相对较新的概念，这种操作除了可以对数据加密，提供机密性之外还可以提供完整性和真实性（authenticity）检测。

其他改动

- 许多场景强制要发送Alert
- 如果没有证书回应certificate_request，则Client必须回应一个空的证书列表

#### TLS1.3

TLS 1.3设计的第一个重要目标就是避免之前版本存在的缺陷，为此，一部分相关的改动如下：

（1）禁止使用RSA密钥传输算法。

（2）禁止一些安全性较弱的密码原语如MD5的使用。

（3）不再支持重协商握手模式。

（4）握手消息采取了加密操作。

（5）实现了握手协议和记录协议的密钥分离。

（6）实现了会话密钥与整个握手消息的绑定。

（7）记录层只能使用AEAD（Authenticated Encryption with Additional Data）认证加密模式。

TLS1.3最大的改动是握手模式的改动，可以极大地降低传输延迟。Round-Trip Time (RTT)表示一次握手的延迟。

**TLS1.2的建立连接握手**：

![](https://ws1.sinaimg.cn/large/44cd29dagy1fp6q9dlbabj20r20hq3z9.jpg)

从图中可以看出，在发送应用数据之前，TLS安全传输需要2-RTT才能完成握手。如果是恢复会话，TLS1.2可以采用Session ID或Session Ticket进行快速握手：

![](https://ws1.sinaimg.cn/large/44cd29dagy1fp6ox4fzlrj20r20de74s.jpg)

从上图可以看到，在使用Session Ticket的情况下，需要1-RTT

那么TLS1.3是如何进一步优化的呢？**TLS 1.3完全握手**：

![](https://ws1.sinaimg.cn/large/44cd29dagy1fp6phygku8j20r20eiaak.jpg)

在完全握手情况下，TLS 1.3需要1-RTT建立连接。与TLS1.2有两点不同：握手过程中移除了`ServerKeyExchange`和`ClientKeyExchange`, DH (Diffie-Hellman) 参数通过 `key_share` 传输。

**TLS1.3恢复会话握手**

![](https://ws1.sinaimg.cn/large/44cd29dagy1fp6pi76ohnj20r20km3zg.jpg)

TLS1.3恢复会话可以直接发送加密后的应用数据，不需要额外的TLS握手。因此，我们常说的TLS1.3 0-RTT握手其实是指恢复加密传输层会话不需要额外的RTT。但是，在第一次进行完全握手时，是需要1-RTT的。与TLS1.2比较，无论是完全握手还是恢复会话，TLS1.3均比TLS1.2少1-RTT。因此，TLS1.3从被提上draft草案开始获得了各方面的好评。

使用TLS最多的场景是HTTPS和HTTP/2. 以HTTP/2为例，一个完整的 HTTP Request需要经历的RTT如下：

|              | HTTP/2 OVER TLS1.2首次连接 | HTTP/2 OVER TLS1.2连接复用 | HTTP/2 OVER TLS1.3首次连接 | HTTP/2 OVER TLS1.3连接复用 |
| ------------ | -------------------------- | -------------------------- | -------------------------- | -------------------------- |
| DNS解析      | 1-RTT                      | 0-RTT                      | 1-RTT                      | 0-RTT                      |
| TCP握手      | 1-RTT                      | 0-RTT                      | 1-RTT                      | 0-RTT                      |
| TLS握手      | 2-RTT                      | 1-RTT                      | 1-RTT                      | 0-RTT                      |
| HTTP Request | 1-RTT                      | 1-RTT                      | 1-RTT                      | 1-RTT                      |
| **总计**     | 5RTT                       | 2-RTT                      | 4-RTT                      | 1-RTT                      |

1. 首次连接发起HTTP请求是非常昂贵的。因此，如果你是用HTTPS作一些不可告人的代理应用的话，一定尽量保持长连接，避免频繁建立新连接。
2. 连接的multlplexing(多路复用)可以有效减少RTT次数，降低延迟。在连接复用的情况下，TLS1.3将整个http request的RTT降低到了1个，达到理论的最小值。

## HTTP 2.0

2015 年 5 月：RFC 7540 (HTTP/2) 和 RFC 7541 (HPACK) 发布。HTTP/2 通过支持标头字段压缩和在同一连接上进行多个并发交换，让应用更有效地利用网络资源，减少感知的延迟时间。具体来说，它可以对同一连接上的请求和响应消息进行交错发送并为 HTTP 标头字段使用有效编码。 HTTP/2 还允许为请求设置优先级，让更重要的请求更快速地完成，从而进一步提升性能。出台的协议对网络更加友好，因为与 HTTP/1.x 相比，可以使用更少的 TCP 连接。

#### 二进制分帧层

HTTP/2 所有性能增强的核心在于新的二进制分帧层，它定义了如何封装 HTTP 消息并在客户端与服务器之间传输。

![](http://www.alloyteam.com/wp-content/uploads/2015/03/http2.0.jpg)

这里所谓的“层”，指的是位于套接字接口与应用可见的高级 HTTP API 之间一个经过优化的新编码机制：HTTP 的语义（包括各种动词、方法、标头）都不受影响，不同的是传输期间对它们的编码方式变了。HTTP/1.x 协议以换行符作为纯文本的分隔符，而 HTTP/2 将所有传输的信息分割为更小的消息和帧，并采用二进制格式对它们编码。这样一来，客户端和服务器为了相互理解，都必须使用新的二进制编码机制：HTTP/1.x 客户端无法理解只支持 HTTP/2 的服务器，反之亦然。

HTTP 2.0 通信都在一个连接上完成，这个连接可以承载任意数量的双向数据流。相应地，每个数据流以消息的形式发送，而消息由一或多个帧组成，这些帧可以乱序发送，然后再根据每个帧首部的流标识符重新组装。

新的二进制分帧机制改变了客户端与服务器之间交换数据的方式。 为了说明这个过程，我们需要了解 HTTP/2 的三个概念：

- *数据流*：已建立的连接内的双向字节流，可以承载一条或多条消息。
- *消息*：与逻辑请求或响应消息对应的完整的一系列帧。
- *帧*：HTTP/2 通信的最小单位，每个帧都包含帧头，至少也会标识出当前帧所属的数据流。

这些概念的关系总结如下：

- 所有通信都在一个 TCP 连接上完成，此连接可以承载任意数量的双向数据流。
- 每个数据流都有一个唯一的标识符和可选的优先级信息，用于承载双向消息。
- 每条消息都是一条逻辑 HTTP 消息（例如请求或响应），包含一个或多个帧。
- 帧是最小的通信单位，承载着特定类型的数据，例如 HTTP 标头、消息负载，等等。 来自不同数据流的帧可以交错发送，然后再根据每个帧头的数据流标识符重新组装。

![](https://raw.githubusercontent.com/zqjflash/http2-protocol/master/http2-connect-stream.png)

#### 请求与响应复用

在 HTTP/1.x 中，如果客户端要想发起多个并行请求以提升性能，则必须使用多个 TCP 连接。这是 HTTP/1.x 交付模型的直接结果，该模型可以保证每个连接每次只交付一个响应（响应排队）。更糟糕的是，这种模型也会导致队首阻塞，从而造成底层 TCP 连接的效率低下。

HTTP/2 中新的二进制分帧层突破了这些限制，实现了完整的请求和响应复用：客户端和服务器可以将 HTTP 消息分解为互不依赖的帧，然后交错发送，最后再在另一端把它们重新组装起来。

![](https://raw.githubusercontent.com/zqjflash/http2-protocol/master/http2-request-more.png)

快照捕捉了同一个连接内并行的多个数据流。客户端正在向服务器传输一个 `DATA` 帧（数据流 5），与此同时，服务器正向客户端交错发送数据流 1 和数据流 3 的一系列帧。因此，一个连接上同时有三个并行数据流。

将 HTTP 消息分解为独立的帧，交错发送，然后在另一端重新组装是 HTTP 2 最重要的一项增强。

- 并行交错地发送多个请求，请求之间互不影响。
- 并行交错地发送多个响应，响应之间互不干扰。
- 使用一个连接并行发送多个请求和响应。
- 不必再为绕过 HTTP/1.x 限制而做很多工作（请参阅[针对 HTTP/1.x 进行优化](https://hpbn.co/optimizing-application-delivery/#optimizing-for-http1x)，例如级联文件、image sprites 和域名分片。
- 消除不必要的延迟和提高现有网络容量的利用率，从而减少页面加载时间。
- *等等…*

#### 数据流优先级

HTTP/2 标准允许每个数据流都有一个关联的权重和依赖关系：

- 可以向每个数据流分配一个介于 1 至 256 之间的整数。
- 每个数据流与其他数据流之间可以存在显式依赖关系。

数据流依赖关系和权重的组合让客户端可以构建和传递“优先级树”，表明它倾向于如何接收响应。反过来，服务器可以使用此信息通过控制 CPU、内存和其他资源的分配设定数据流处理的优先级，在资源数据可用之后，带宽分配可以确保将高优先级响应以最优方式传输至客户端。

#### 流控制

**所有 HTTP/2 连接都是永久的，而且仅需要每个来源一个连接**，随之带来诸多性能优势。

流控制是一种阻止发送方向接收方发送大量数据的机制，以免超出后者的需求或处理能力。由于 HTTP/2 数据流在一个 TCP 连接内复用，TCP 流控制既不够精细，也无法提供必要的应用级 API 来调节各个数据流的传输。为了解决这一问题，HTTP/2 提供了一组简单的构建块，这些构建块允许客户端和服务器实现其自己的数据流和连接级流控制：

- 流控制具有方向性。每个接收方都可以根据自身需要选择为每个数据流和整个连接设置任意的窗口大小。
- 流控制基于信用。每个接收方都可以公布其初始连接和数据流流控制窗口（以字节为单位），每当发送方发出 `DATA`帧时都会减小，在接收方发出 `WINDOW_UPDATE` 帧时增大。
- 流控制无法停用。建立 HTTP/2 连接后，客户端将与服务器交换 `SETTINGS` 帧，这会在两个方向上设置流控制窗口。流控制窗口的默认值设为 65,535 字节，但是接收方可以设置一个较大的最大窗口大小（`2^31-1` 字节），并在接收到任意数据时通过发送 `WINDOW_UPDATE` 帧来维持这一大小。
- 流控制为逐跃点控制，而非端到端控制。即，可信中介可以使用它来控制资源使用，以及基于自身条件和启发式算法实现资源分配机制。

#### 服务器推送

HTTP/2 打破了严格的请求-响应语义，支持一对多和服务器发起的推送工作流，在浏览器内外开启了全新的互动可能性。这是一项使能功能，对我们思考协议、协议用途和使用方式具有重要的长期影响。

![](https://raw.githubusercontent.com/zqjflash/http2-protocol/master/http2-server-push.png)

所有服务器推送数据流都由 `PUSH_PROMISE` 帧发起，表明了服务器向客户端推送所述资源的意图，并且需要先于请求推送资源的响应数据传输。这种传输顺序非常重要：客户端需要了解服务器打算推送哪些资源，以免为这些资源创建重复请求。满足此要求的最简单策略是先于父响应（即，`DATA` 帧）发送所有 `PUSH_PROMISE` 帧，其中包含所承诺资源的 HTTP 标头。

在客户端接收到 `PUSH_PROMISE` 帧后，它可以根据自身情况选择拒绝数据流（通过 `RST_STREAM` 帧）。

推送的每个资源都是一个数据流，与内嵌资源不同，客户端可以对推送的资源逐一复用、设定优先级和处理。 浏览器强制执行的唯一安全限制是，推送的资源必须符合原点相同这一政策：服务器对所提供内容必须具有权威性。

#### 首部压缩

每个 HTTP 传输都承载一组标头，这些标头说明了传输的资源及其属性。 在 HTTP/1.x 中，此元数据始终以纯文本形式，通常会给每个传输增加 500–800 字节的开销。如果使用 HTTP Cookie，增加的开销有时会达到上千字节。为了减少此开销和提升性能，HTTP/2 使用 HPACK 压缩格式压缩请求和响应标头元数据，这种格式采用两种简单但是强大的技术：

1. 这种格式支持通过静态 Huffman 代码对传输的标头字段进行编码，从而减小了各个传输的大小。
2. 这种格式要求客户端和服务器同时维护和更新一个包含之前见过的标头字段的索引列表（换句话说，它可以建立一个共享的压缩上下文），此列表随后会用作参考，对之前传输的值进行有效编码。

利用 Huffman 编码，可以在传输时对各个值进行压缩，而利用之前传输值的索引列表，我们可以通过传输索引值的方式对重复值进行编码，索引值可用于有效查询和重构完整的标头键值对。

![](https://raw.githubusercontent.com/zqjflash/http2-protocol/master/http2-header-diff.png)

作为一种进一步优化方式，HPACK 压缩上下文包含一个静态表和一个动态表：静态表在规范中定义，并提供了一个包含所有连接都可能使用的常用 HTTP 标头字段（例如，有效标头名称）的列表；动态表最初为空，将根据在特定连接内交换的值进行更新。因此，为之前未见过的值采用静态 Huffman 编码，并替换每一侧静态表或动态表中已存在值的索引，可以减小每个请求的大小。

注：在 HTTP/2 中，请求和响应标头字段的定义保持不变，仅有一些微小的差异：所有标头字段名称均为小写，请求行现在拆分成各个 `:method`、`:scheme`、`:authority` 和 `:path` 伪标头字段。



参考：

《HTTP权威指南》

[Cookie小结](https://juejin.im/entry/58c4ccc7128fe1006b37facb)

[Session的攻击手段](https://blog.csdn.net/h_mxc/article/details/50542038)

[深入理解Session和Cookie](http://wanglizhi.github.io/2016/06/01/Session-Cookie/)

[数据包结构分析](https://blog.csdn.net/cjf1002361126/article/details/70232721)

[**tls1-3-quic-是怎样做到-0-rtt-的**](https://liudanking.com/network/tls1-3-quic-%E6%98%AF%E6%80%8E%E6%A0%B7%E5%81%9A%E5%88%B0-0-rtt-%E7%9A%84/) 

[HTTP2.0 详解](https://github.com/zqjflash/http2-protocol/blob/master/README.md)

[HTTP2 简介 Google](https://developers.google.com/web/fundamentals/performance/http2/?hl=zh-cn)

[1]AlFardan N J, Paterson K G. Lucky thirteen: Breaking the TLS and DTLS record protocols [C]. Proceedings of 2013 IEEE Symposium on Security and Privacy (S&P), 2013. 526–540.

[2]Bhargavan K, Lavaud A D, Fournet C, et al. Triple handshakes and cookie cutters: Breaking and fixing authentication over TLS [C]. Proceedings of 2014 IEEE Symposium on Security and Privacy (S&P), 2014. 98–113.

[3]Mavrogiannopoulos N, Vercauteren F, Velichkov V, et al. A cross-protocol attack on the TLS protocol[C]//Proceedings of the 2012 ACM conference on Computer and communications security. ACM, 2012: 62-72.

[4]AlFardan N J, Bernstein D J, Paterson K G, et al. On the Security of RC4 in TLS [C]. Proceedings of 22nd USENIX Security Symposium, 2013. 305–320.

[5]Bleichenbacher D. Chosen ciphertext attacks against protocols based on the RSA encryption standard PKCS #1 [C]. Advances in Cryptology—CRYPTO’98, 1998. 1–12.

