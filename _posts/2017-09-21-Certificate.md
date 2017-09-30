---
layout:     post
title:      "签名与证书"
subtitle:   "从微信支付、支付宝和PayPal三种支付API对签名和证书的调用中，总结签名和证书的原理"
date:       2017-09-21 23:30:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - 数字证书
    - 支付安全
---

## 前置知识

TCP：通过三次握手，四次挥手，保证连接的可靠性；

SSL（安全套接层）&TLS（传输层安全）：在传输层对网络连接进行加密；通过Client生成的随机数R1,R3 + Server生成的随机数R2 根据算法生成对称秘钥；

SSL双向握手：Server端请求Client发送证书和公钥，Client发送私钥加密后的Hash信息，Server用公钥验证。其他都类似单向握手；

HTTP & HTTPS：HTTP握手=TCP握手，HTTPS握手= TCP握手+SSL握手；

RSA算法：非对称加密算法，生成公钥和私钥。





![](https://upload.wikimedia.org/wikipedia/commons/a/ae/SSL_handshake_with_two_way_authentication_with_certificates.svg)



参考：[SSL/TLS握手详解](http://www.jianshu.com/p/7158568e4867)

[Android HTTPS 双向证书验证](http://frank-zhu.github.io/android/2014/12/26/android-https-ssl/)

[RSA算法原理](http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)

## PayPal

#### SOAP API

证书（Certificate）Java调用代码

```java
// PayPal-Java-SDK/rest-api-sdk/src/main/java/com/paypal/base/SSLUtil.java
/**
	 * Create a SSLContext with provided client certificate
	 *
	 * @param certPath
	 * @param certPassword
	 * @return SSLContext
	 * @throws SSLConfigurationException
	 */
	public static SSLContext setupClientSSL(String certPath, String certPassword)
			throws SSLConfigurationException {
		SSLContext sslContext = null;
		try {
			KeyStore ks = p12ToKeyStore(certPath, certPassword);
			KMF.init(ks, certPassword.toCharArray());
			sslContext = getSSLContext(KMF.getKeyManagers());
		} catch (NoSuchAlgorithmException e) {
			throw new SSLConfigurationException(e.getMessage(), e);
		} catch (KeyStoreException e) {
			throw new SSLConfigurationException(e.getMessage(), e);
		} catch (UnrecoverableKeyException e) {
			throw new SSLConfigurationException(e.getMessage(), e);
		} catch (CertificateException e) {
			throw new SSLConfigurationException(e.getMessage(), e);
		} catch (NoSuchProviderException e) {
			throw new SSLConfigurationException(e.getMessage(), e);
		} catch (IOException e) {
			throw new SSLConfigurationException(e.getMessage(), e);
		}
		return sslContext;
	}
// PayPal-Java-SDK/rest-api-sdk/src/test/java/com/paypal/base/ConfigurationUtil.java
public static Map<String, String> getCertificateConfiguration() {
		Map<String, String> initMap = new HashMap<String, String>();
		initMap.put("acct2.UserName", "certuser_biz_api1.paypal.com");
		initMap.put("acct2.Password", "D6JNKKULHN3G5B8A");
		initMap.put("acct2.CertKey", "password");
		initMap.put("acct2.CertPath", "src/test/resources/sdk-cert.p12");
		initMap.put("acct2.AppId", "APP-80W284485P519543T");
		initMap.put("mode", "sandbox");
		return initMap;
	}
```

签名（Signature）Java调用代码

```java
// PayPal-Java-SDK/rest-api-sdk/src/test/java/com/paypal/base/ConfigurationUtil.java
public static Map<String, String> getSignatureConfiguration() {
		Map<String, String> initMap = new HashMap<String, String>();
		initMap.put("acct1.UserName", "jb-us-seller_api1.paypal.com");
		initMap.put("acct1.Password", "WX4WTU3S8MY44S7F");
		initMap.put("acct1.Signature",
				"AFcWxV21C7fd0v3bYYYRCpSSRl31A7yDhhsPUU2XhtMoZXsWHFxu-RWy");
		initMap.put("acct1.AppId", "APP-80W284485P519543T");
		initMap.put("mode", "sandbox");
		return initMap;
	}
// 存储在 SignatureCredential 中, 最后返回在请求Header中
//sdk-core-java/src/main/java/com/paypal/core/AbstractSignatureHttpHeaderAuthStrategy.java
/**
	 * Returns {@link CertificateCredential} as HTTP headers
	 */
	public Map<String, String> generateHeaderStrategy(SignatureCredential credential)
			throws OAuthException {
		Map<String, String> headers;
		if (credential.getThirdPartyAuthorization() instanceof TokenAuthorization) {
			headers = processTokenAuthorization(credential,
					(TokenAuthorization) credential
							.getThirdPartyAuthorization());

		} else {
			headers = new HashMap<String, String>();
			headers.put(Constants.PAYPAL_SECURITY_USERID_HEADER,
					credential.getUserName());
			headers.put(Constants.PAYPAL_SECURITY_PASSWORD_HEADER,
					credential.getPassword());
			headers.put(Constants.PAYPAL_SECURITY_SIGNATURE_HEADER,
					credential.getSignature());
		}
		return headers;
	}
```

使用SOAP的Java SDK本地调用SearchTransaction API，验证证书，然后进行HTTPS抓包。

```Java
Map<String, String> customConfigurationMap = new HashMap<String, String>();
        customConfigurationMap.put("mode", "sandbox");
        customConfigurationMap.put("acct1.UserName", "-facilitator_api1.qq.com");
        customConfigurationMap.put("acct1.Password", "TYNJTTKGE5XTE3MJ");
        customConfigurationMap.put("acct1.CertKey", "cert key");
        customConfigurationMap.put("acct1.CertPath", "/Users/lizwang/Downloads/paypal_cert.p12");
        customConfigurationMap.put("acct1.AppId", "");

        TransactionSearchReq txnreq = new TransactionSearchReq();
        TransactionSearchRequestType requestType = new TransactionSearchRequestType();

        requestType.setStartDate("2017-09-04T00:00:00.000Z");
        requestType.setEndDate("2017-10-05T23:59:59.000Z");
        requestType.setVersion("95.0");
        requestType.setTransactionID("4JY569287K521160L");
        txnreq.setTransactionSearchRequest(requestType);

        PayPalAPIInterfaceServiceService service = new PayPalAPIInterfaceServiceService(customConfigurationMap);
        TransactionSearchResponseType txnresponse = service.transactionSearch(txnreq);
```



参考：[Manage credentials](https://developer.paypal.com/docs/classic/api/apiCredentials/)

[Java merchant sdk](https://github.com/paypal/merchant-sdk-java)

[Java sdk core](https://github.com/paypal/sdk-core-java)

## 微信支付

#### 签名算法

签名生成步骤：

第一步，设所有发送或者接收到的数据为集合M，将集合M内非空参数值的参数按照参数名ASCII码从小到大排序（字典序），使用URL键值对的格式（即key1=value1&key2=value2…）拼接成字符串stringA。

第二步，在stringA最后拼接上key得到stringSignTemp字符串，并对stringSignTemp进行MD5运算，再将得到的字符串所有字符转换为大写，得到sign值signValue。

```Java
stringA="appid=wxd930ea5d5a258f4f&body=test&device_info=1000&mch_id=10000100&nonce_str=ibuaiVcKdpRxkhJA";
stringSignTemp=stringA+"&key=192006250b4c09247ec02edce69f6a2d" //注：key为商户平台设置的密钥key
sign=MD5(stringSignTemp).toUpperCase()="9A0A8659F005D6984697E2CA0A9CF3B7" //注：MD5签名方式
sign=hash_hmac("sha256",stringSignTemp,key).toUpperCase()="6A9AE1657590FD6257D693A078E1C3E4BB6BA4DC30B23E0EE2496E54170DACD6" //注：HMAC-SHA256签名方式
```



随机数字段nonce_str，主要保证签名不可预测？

#### 证书

微信支付接口中，涉及资金回滚的接口会使用到商户证书，包括退款、撤销接口。

商户下载的文件中内容：

- apiclient_cert.p12：包含了`私钥`信息的证书文件，由微信支付签发给商户用来标识和界定商户的身份。双击导入，证书密码默认为商户ID。
- apiclient_cert.pem：从apiclient_cert.p12中导出的`证书`部分的文件，pem格式为部分开发语言和环境（PHP）。你也可以使用openssl命令自己导出该文件，openssl pkcs12 -clcerts -nokeys -in apiclient_cert.p12 -out apiclient_cert.pem
- apiclient_key.pem：从apiclient_cert.p12中导出的`秘钥`部分的文件，openssl命令 - openssl
  pkcs12 -clcerts -nokeys -in apiclient_cert.p12 -out apiclient_cert.pem
- rootca.pem：微信支付api服务器上也部署了证明微信支付身份的服务器证书，您在使用api进行调用时也需要验证所调用服务器及域名的真实性，该文件为签署微信支付证书的权威机构的根证书，可以用来验证微信支付服务器证书的真实性。

**HTTPS双向认证过程：**

既服务器验证客户端的时候通过客户端证书和签名（既：apiclient_cert.p12 或者 apiclient_cert.pem和apiclient_key.pem），客户端验证服务器通过ca的根证书进行（rootca.pem），根证书有些操作系统上或者开发环境中已经包含，此时不需要导入。

**JAVA只需要使用apiclient_cert.p12即可**

```Java
//指定读取证书格式为PKCS12
KeyStore keyStore = KeyStore.getInstance("PKCS12");
//读取本机存放的PKCS12证书文件
FileInputStream instream = new FileInputStream(new File("D:/apiclient_cert.p12"));
try {
//指定PKCS12的密码(商户ID)
keyStore.load(instream, "10010000".toCharArray());
} finally {
instream.close();
}
SSLContext sslcontext = SSLContexts.custom()
.loadKeyMaterial(keyStore, "10010000".toCharArray()).build();
//指定TLS版本
SSLConnectionSocketFactory sslsf = new SSLConnectionSocketFactory(
sslcontext,new String[] { "TLSv1" },null,
SSLConnectionSocketFactory.BROWSER_COMPATIBLE_HOSTNAME_VERIFIER);
//设置httpclient的SSLSocketFactory
CloseableHttpClient httpclient = HttpClients.custom()
.setSSLSocketFactory(sslsf)
.build();
try {
            HttpGet httpget = new HttpGet("https://api.mch.weixin.qq.com/secapi/pay/refund");
            System.out.println("executing request" + httpget.getRequestLine());
            CloseableHttpResponse response = httpclient.execute(httpget);
            try {
                HttpEntity entity = response.getEntity();
                System.out.println("----------------------------------------");
                System.out.println(response.getStatusLine());
                if (entity != null) {
                    System.out.println("Response content length: " + entity.getContentLength());
                    BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(entity.getContent()));
                    String text;
                    while ((text = bufferedReader.readLine()) != null) {
                        System.out.println(text);
                    }
                }
                EntityUtils.consume(entity);
            } finally {
                response.close();
            }
        } finally {
            httpclient.close();
        }
```

在 Java 平台下，证书常常被存储在 KeyStore 文件中，上面说的 cacerts 文件就是一个 KeyStore 文件，KeyStore 不仅可以存储数字证书，还可以存储密钥，存储在 KeyStore 文件中的对象有三种类型：Certificate、PrivateKey 和 SecretKey 。Certificate 就是证书，PrivateKey 是非对称加密中的私钥，SecretKey 用于对称加密，是对称加密中的密钥。



参考：[微信支付安全规范](https://pay.weixin.qq.com/wiki/doc/api/micropay.php?chapter=4_3)

[Java 和 HTTP的那些事](http://www.aneasystone.com/archives/2016/04/java-and-https.html)

## 支付宝



out_trade_no



参考：[支付宝RSA秘钥](https://docs.open.alipay.com/291/105974/)

[签名与验签](https://docs.open.alipay.com/200/105351)

[MD5签名](https://docs.open.alipay.com/58/103591/)

## PKI系统与数字证书结构

#### X.509标准

X.509是一种非常通用的证书格式。所有的证书都符合ITU-T X.509国际标准；因此(理论上)为一种应用创建的证书可以用于任何其他符合X.509标准的应用。在一份证书中，必须证明公钥及其所有者的姓名是一致的。对X.509证书来说，认证者总是 CA或由CA指定的人，一份X.509证书是一些标准字段的集合，这些字段包含有关用户或设备及其相应公钥的信息。X.509标准定义了证书中应该包含哪些信息，并描述了这些信息是如何编码的(即数据格式)，所有的X.509证书包含以下数据。

- **版本号：**指出该证书使用了哪种版本的X.509标准（版本1、版本2或是版本3），版本号会影响证书中的一些特定信息，目前的版本为3
- **序列号：** 标识证书的唯一整数，由证书颁发者分配的本证书的唯一标识符
- **签名算法标识符：** 用于签证书的算法标识，由对象标识符加上相关的参数组成，用于说明本证书所用的数字签名算法。例如，SHA-1和RSA的对象标识符就用来说明该数字签名是利用RSA对SHA-1杂凑加密
- **认证机构的数字签名：**这是使用发布者私钥生成的签名，以确保这个证书在发放之后没有被撰改过
- **认证机构：** 证书颁发者的可识别名（DN），是签发该证书的实体唯一的CA的X.500名字。使用该证书意味着信任签发证书的实体。(注意：在某些情况下，比如根或顶级CA证书，发布者自己签发证书)
- **有效期限：** 证书起始日期和时间以及终止日期和时间；指明证书在这两个时间内有效
- **主题信息：**证书持有人唯一的标识符(或称DN-distinguished name)这个名字在 Internet上应该是唯一的
- **公钥信息：** 包括证书持有人的公钥、算法(指明密钥属于哪种密码系统)的标识符和其他相关的密钥参数
- **颁发者唯一标识符：**标识符—证书颁发者的唯一标识符，仅在版本2和版本3中有要求，属于可选项

扩展部分包括：

- **发行者密钥标识符：**证书所含密钥的唯一标识符，用来区分同一证书拥有者的多对密钥。
- **密钥使用：**一个比特串，指明（限定）证书的公钥可以完成的功能或服务，如：证书签名、数据加密等。如果某一证书将 KeyUsage 扩展标记为“极重要”，而且设置为“keyCertSign”，则在 SSL 通信期间该证书出现时将被拒绝，因为该证书扩展表示相关私钥应只用于签写证书，而不应该用于 SSL。
- **CRL分布点：**指明CRL的分布地点
- **私钥的使用期：**指明证书中与公钥相联系的私钥的使用期限，它也有Not Before和Not After组成。若此项不存在时，公私钥的使用期是一样的。
- **证书策略：**由对象标识符和限定符组成，这些对象标识符说明证书的颁发和使用策略有关。
- **策略映射：**表明两个CA域之间的一个或多个策略对象标识符的等价关系，仅在CA证书里存在
- **主体别名：**指出证书拥有者的别名，如电子邮件地址、IP地址等，别名是和DN绑定在一起的。
- **颁发者别名：**指出证书颁发者的别名，如电子邮件地址、IP地址等，但颁发者的DN必须出现在证书的颁发者字段。
- **主体目录属性：**指出证书拥有者的一系列属性。可以使用这一项来传递访问控制信息。

#### 数字证书格式

数字证书体现为一个或一系列相关经过加密的数据文件。常见格式有：

- 符合PKI ITU-T X509标准，传统标准（.DER .PEM .CER .CRT）
- 符合PKCS#7 加密消息语法标准(.P7B .P7C .SPC .P7R)
- 符合PKCS#10 证书请求标准(.p10)
- 符合PKCS#12 个人信息交换标准（.pfx *.p12）

当然，这只是常用的几种标准，其中，X509证书还分两种编码形式：

- X.509 DER(Distinguished Encoding Rules)编码，后缀为：.DER .CER .CRT
- X.509 BASE64编码，后缀为：.PEM .CER .CRT

X509是数字证书的基本规范，而P7和P12则是两个实现规范，P7用于数字信封，P12则是带有私钥的证书实现规范。采用的标准不同，生成的数字证书，包含内容也可能不同。下面就证书包含/可能包含的内容做个汇总，一般证书特性有：

- 存储格式：二进制还是ASCII
- 是否包含公钥、私钥
- 包含一个还是多个证书
- 是否支持密码保护（针对当前证书）

其中：

- DER、CER、CRT以二进制形式存放证书，只有公钥，不包含私钥
- CSR证书请求
- PEM以Base64编码形式存放证书，以”—–BEGIN CERTIFICATE—–” 和 “—–END CERTIFICATE—–”封装，只有公钥
- PFX、P12也是以二进制形式存放证书，包含公钥、私钥，包含保护密码。PFX和P12存储格式完全相同只是扩展名不同
- P10证书请求
- P7R是CA对证书请求回复，一般做数字信封
- P7B/P7C证书链，可包含一个或多个证书

理解关键点：凡是包含私钥的，一律必须添加密码保护（加密私钥），因为按照习惯，公钥是可以公开的，私钥必须保护，所以明码证书以及未加保护的证书都不可能包含私钥，只有公钥，不用加密。

上文描述中，DER均表示证书且有签名，实际使用中，还有DER编码的私钥不用签名，实际上只是个“中间件”。另外：证书请求一般采用CSR扩展名，但是其格式有可能是PEM也可能是DER格式，但都代表证书请求，只有经过CA签发后才能得到真正的证书。

## Openssl

从P12文件提取证书：

```
openssl pkcs12 -in test.p12 -clcerts -nokeys -out cert.pem  //pem格式
openssl pkcs12 -in test.p12 -clcerts -nokeys -out cert.crt  //crt格式
```

如果需要携带秘钥,则去掉 -nokeys

```
openssl pkcs12 -in test.p12 -clcerts  -out cert.pem  //pem格式
openssl pkcs12 -in test.p12 -clcerts  -out cert.crt  //crt格式
```

提取私钥:

```
openssl pkcs12 -in test.p12 -nocerts -out key.pem
```





参考：[PKI系统与数字证书结构](http://www.enkichen.com/2016/04/12/certification-and-pki/)

[Java加密技术](http://snowolf.iteye.com/blog/735294)