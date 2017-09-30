---
layout:     post
title:      "日志框架分析"
subtitle:   "Java的日志框架，logging,Log4j,Slf4j等功能对比及源码分析"
date:       2017-07-25 23:30:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Slf4j
    - Log4j
---

> Java中打印日志是最基本的操作，也是非常重要的一项功能，找Bug、监控应用都离不开log
>
> Java.util.logging 和 log4j都是实现层
>
> slf4j 和 commons-logging是接口层，底层可以依赖不同的实现

# Tips on logging in Java

1. 为什么我们要在Java中打印日志？每个人都从`System.out.println()`开始向控制台打印信息，但是当应用运行在Server上时，我们需要查看Server上的日志文件来判断应用的运行状态，这时使用log4j或者logging等打印日志就是必须的了。
2. Java有什么日志级别？DEBUG：最低限制的日志级别，只能用于开发和测试，不能用于线上环境；INFO：应该打印信息如服务器启动时间、访问信息、输出信息等；WARN：应打印例如client和server之间连接中断、数据库中断，可以在该日志级别上设置报警阈值，监控程序；ERROR：用来记录Errors和Exception
3. Log4j 比java.util.logging更好用，log4j更灵活，可以通过log4j.properties或者Log4j.xml来配置，log4j是线程安全的，设计用于多线程系统
4. 日志如何影响Java的性能？很明显，过多的日志记录由于文件IO访问而导致应用程序变慢，所以选择正确的日志级别尤为重要，使用DEBUG级别的日志时要放在if块里面`if(logger.isDebugEnabled()){ logger.debug("java logging level is DEBUG Enabled") }`
5. 在Properties文件中规定日志排版，包含Thread name和Java的全类名
6. 记录日志时使用前缀标识来源，如client、DB、session，方便使用grep或者find查询
7. 如果一个logger没有指定级别，它将继承最近祖先的级别
8. 没有日志和过度记录日志都是糟糕的
9. 日志的可读性
10. 如果是使用SLFJ的话，使用参数比字符串连接更快，`logger.debug("No of Executions {} for clients : {}", noOfOrder, client)//faster `
11. 不要打印敏感信息，如密码、银行卡号等

参考：[Top 10 Tips on Logging in Java](http://javarevisited.blogspot.com/2011/05/top-10-tips-on-logging-in-java.html)

# 为什么要使用SLF4J而不是Log4J

SLF4J（Simple logging Facade for Java）不是一个真正的日志实现，而是一个抽象层，允许你在后台使用任意一个日志类库。比如：如果一个项目使用了Log4j，而你加载了一个类库如Apache Active MQ——它依赖于另外一个日志类库logback，那么你就需要把它也加载进来。但如果Apache Active MQ使用了SLF4J，你就可以继续使用你的日志类库而无需加载维护一个新的日志框架。

在开源或者内部类库中使用SLF4J会使得它独立于任何一个特定的日志实现，SLF4J提供了基于占位符的日志方法，通过去除检查isDebugEnabled()来提高代码可读性。

```Java
// Log4j
if (logger.isDebugEnabled) {
  logger.debug("id : " + id + " and symbol " + symbol);
}
//SLF4J
logger.debug("id : {} and symbol : {}", id, symbol);
// 该方法在打印日志之前会检查一个特定的日志级别是不是打开了，降低内存消耗，降低字符串连接命令时间
```



![](http://ifeve.com/wp-content/uploads/2016/04/chart.png)



切换日志框架，只需要替换class path中的slf4j绑定。比如说，从  java.util.logging切换到 log4j，仅仅把  slf4j-jdk14-1.7.19.jar替换成 slf4j-log4j12-1.7.19.jar。

SLF4J不依赖于特定的类加载机制。事实上，每个SLF4J绑定在编译时硬连接来使用一个指定的日志框架。比如说， slf4j-log4j12-1.7.19.jar在编译时绑定使用log4j。在你的代码中，除  *slf4j-api-1.7.19.jar*之外，只能有一个你选择的绑定 到正确的class path 路径上。不要在class path 放置多个绑定。下面是一个图表来说明一般的想法。

![](http://www.slf4j.org/images/concrete-bindings.png)



参考：[为什么要使用SLF4J](http://www.importnew.com/7450.html)

[简化SLF4J和通用日志工具的区别](http://www.importnew.com/19448.html)




