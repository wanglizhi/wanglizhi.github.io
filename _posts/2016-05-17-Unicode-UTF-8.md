---
layout:     post
title:      "字符编码常识及问题解析"
subtitle:   "什么都略懂一点，生活更多彩一些~"
date:       2016-05-17 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - 字符编码
    - 数据库
---

> 看到博客作者在面试中遇到的题目：简述Unicode与UTF-8直接的关系。想起了我之前面试中被问到的问题：使用Mysql中遇到字符编码问题，怎么解决的。

![](http://image.beekka.com/blog/2014/bg2014121103.jpg)

## 基本常识

#### 位和字节

位（bit）是指计算机里存放的二进制（0/1），而8个位组成的“位串”成为字节，有256（2的8次方）个组合方式

字符编码，就是指定义一套规则，将真实世界里的字母/字符与计算机的二进制序列进行相互转化。

例如：01000001（0x41）<-> 65 <-> 'A'

#### 拉丁字符

基础拉丁字母,即指常见的”ABCD“等26个英文字母，这些字母与英语中一些常见的符号（如数字，标点符号）称为**基础拉丁字符**，在基础拉丁字符的基础上，加上一些连字符，变音字符(如’Á’)，形成了**派生拉丁字母**。完整意义上的拉丁字符是指这些变体字符与基础拉丁字符的全集

## 编码标准

#### 拉丁编码

**ASCII**的全称是American Standard Code for Information Interchange（美国信息交换标准代码）设定的ASCII编码也只支持**基础拉丁字符**。ASCII的设计也很简单，用一个字节（8个位）来表示一个字符，并保证最高位的取值永远为’0’。即表示字符含义的位数为7位，不难算出其可表达字符数为27 =128个。这128个字符包括95个可打印的字符（涵盖了26个英文字母的大小写以及英文标点符号能）与33个控制字符（不可打印字符）。

为了用更大的派生拉丁字符集，最简单的办法就是将美国人没有用到的第8位也用上，这个编码规则也常被称为EASCII。EASCII基本解决了整个西欧的字符编码问题。但是对于欧洲其它地方如北欧，东欧地区，256个字符还是不够用，如是出现了ISO 8859。**ISO 8859**采取的不再是单个独立的编码规则，而是由一系列的字符集（共15个）所组成，如ISO 8859-1对应西欧语言，ISO 8859-2对应中欧语言等。其中大家所熟悉的**Latin-1**就是ISO 8859-1的别名,它表示整个西欧的字符集范围。 需要注意的一点的是，ISO 8859-n与ASCII是**兼容**的，即其0000000(0x00)-01111111(0x7f)范围段与ASCII保持一致，而10000000（0x80）-11111111(0xFF)范围段被扩展用到不同的字符集。

#### 中文编码

以上我们接触到的拉丁编码，都是单字节编码，即用一个字节来对应一个字符。但这一规则对于其它字符集更大的语言来说，并不适应，比如中文，而是出现了用多个字节表示一个字符的编码规则。常见的中文**GB2312**（国家简体中文字符集）就是用**两个字节来表示一个汉字**（注意是表示一个汉字，对于拉丁字母，GB2312还是是用一个字节来表示以**兼容ASCII**）

![](https://raw.githubusercontent.com/wanglizhi/wanglizhi.github.io/master/img/2016-05-16/gbk-gb2312.jpg)

#### Unicode

Unicode就是一种。为了能独立表示世界上所有的字符，Unicode采用**4个字节**表示一个字符,这样理论上Unicode能表示的字符数就达到了2的31次方 = 2147483648 = 21 亿左右个字符，完全可以涵盖世界上一切语言所用的符号。我们以汉字”微信“两字举例说明：

```
微  00000000 00000000 01011111 10101110
信  00000000 00000000 01001111 11100001
```

Unicode对所有的字符编码均需要四个字节，而这对于拉丁字母或汉字来说是**浪费**的，其前面三个或两个字节均是0,这对信息存储来说是极大的浪费。另外一个问题就是，如何区分Unicode与其它编码这也是一个问题，比如计算机怎么知道四个字节表示一个Unicode中的字符，还是分别表示四个ASCII的字符呢？

直至UTF-8作为Unicode的一种实现后，部分问题得到解决。UTF-8是Unicode的一种实现方式，而Unicode是一个统一标准规范，Unicode的实现方式除了UTF-8还有其它的，比如UTF-16等。

UTF-8的基本规则：

- 规则1：对于单字节字符，字节的第一位为0，后7位为这个符号的Unicode码，所以对于拉丁字母，UTF-8和ASCII码是一致的


- 规则2：对于n字节的字符，第一个字节前n位都设为1，第n+1位为0，后面字节的前两位一律设为10，剩下没有提及的位，全部为这个符号的Unicode编码。

根据以上规则，可以建立一个Unicode取值范围与UTF-8字节序表示的对应关系，如下表

![](https://raw.githubusercontent.com/wanglizhi/wanglizhi.github.io/master/img/2016-05-16/unicode-utf8.jpg)

举例来说，’微’的Unicode是’\u5fae’，二进制表示是”00000000 00000000 01011111 10101110“，其取值就位于’0000 0800-0000 FFFF’之间，所以其UTF-8编码为’**111**00101 **10**111110 **10**101110’ （加粗部分为固定编码内容）。

作为中文使用者需要注意的一点是Unicode(UTF-8)与GBK，GB2312这些汉字编码规则是完全不兼容的，也就是说这两者之间不能通过任何算法来进行转换,如需转换，一般通过GBK查表的方式来进行。

## 常见问题及解答

#### 1、windows Notepad中的编码ANSI保存选项，代表什么含义？

ANSI是windows的默认的编码方式，对于英文文件是ASCII编码，对于简体中文文件是GB2312编码（只针对Windows简体中文版，如果是繁体中文版会采用Big5码）。所以，如果将一个UTF-8编码的文件，另存为ANSI的方式，对于中文部分会产生乱码。

#### 2、 什么是UTF-8的BOM？

BOM的全称是Byte Order Mark，BOM是微软给UTF-8编码加上的，用于标识文件使用的是UTF-8编码，即在UTF-8编码的文件起始位置，加入三个字节“EE BB BF”。这是微软特有的，标准并不推荐包含BOM的方式。采用加BOM的UTF-8编码文件，对于一些只支持标准UTF-8编码的环境，可能导致问题。比如，在Go语言编程中，对于包含BOM的代码文件，会导致编译出错。

#### 3、为什么数据库Latin1字符集（单字节）可以存储中文呢？

只要能保证传输和存储的字节顺序不会乱即可。作为数据库，只是作为存储的使用的话，只要能保证存储的顺序与写入的顺序一致，然后再按相同的字节顺序读出即可，翻译成语义字符的任务交给应用程序。

#### 4、Mysql数据库中多个字符集变量（其它数据库其实也类似），它们之间分别是什么关系？

![](https://raw.githubusercontent.com/wanglizhi/wanglizhi.github.io/master/img/2016-05-16/mysql-encode.jpg)

- character_set_client：**客户端**来源的数据使用的字符集，用于客户端显式告诉客户端所发送的语句中的的字符编码


- character_set_connection：**连接层**的字符编码，mysql一般用character_set_connection将客户端的字符转换为连接层表示的字符。


- character_set_results:查询**结果**从数据库读出后，将转换为character_set_results返回给前端。

而我们常见的解决乱码问题的操作：mysql_query('SET NAMES GBK')，其相当于将以上三个字符集统一全部设置为GBK，这三者一致时，一般就解决了乱码问题。

character_set_database:当前选中数据库的默认字符集，如当create table时没有指定字符集，将默认选择该字符集。

## 5、什么情况下，表示信息丢失？

对于mysql数据库，我们可以通过hex(colname)函数（其它数据库也有类似的函数，一些文本文件编辑器也具有这个功能），查看实际存储的字节内容，如：

![](https://raw.githubusercontent.com/wanglizhi/wanglizhi.github.io/master/img/2016-05-16/hex.jpg)

通过查看存储的字节序，我们可以从根本上了解存储的内容是什么编码了。而当发现存储的内容全部是’3F’时，就表明存储的内容由于编码问题，信息已经丢失了，无法再找回。

之所以出现这种信息丢失的情况，一般是将不能相互转换的字符集之间做了转换，比如我们在前文说到，UTF-8只能一个个字节地变成Latin-1，但是根本不能转换的，因为两者之间没有转换规则，Unicode的字符对应范围也根本不在Latin-1范围内，所以只能用’?(0x3F)’代替了。



**参考** [字符编码常识及问题解析](http://blog.jobbole.com/76376/)





