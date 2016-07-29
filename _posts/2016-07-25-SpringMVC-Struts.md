---
layout:     post
title:      "Spring MVC VS Struts"
subtitle:   ""
date:       2016-07-25 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Java Web
    - Spring MVC
    - Structs
---

# SpringMVC框架介绍

    1) [spring](http://lib.csdn.net/base/17) MVC属于SpringFrameWork的后续产品，已经融合在Spring Web Flow里面。

> Spring 框架提供了构建 Web 应用程序的全功能 MVC 模块。使用 Spring 可插入的 MVC [架构](http://lib.csdn.net/base/16)，可以选择是使用内置的 Spring Web 框架还是 Struts 这样的 Web 框架。通过策略接口，Spring 框架是高度可配置的，而且包含多种视图技术，例如 JavaServer Pages（JSP）技术、Velocity、Tiles、iText 和 POI。Spring MVC 框架并不知道使用的视图，所以不会强迫您只使用 JSP 技术。

        Spring MVC 分离了控制器、模型对象、分派器以及处理程序对象的角色，这种分离让它们更容易进行定制。

    2) Spring的MVC框架主要由DispatcherServlet、处理器映射、处理器(控制器)、视图解析器、视图组成。

# SpringMVC原理图

![](http://img.my.csdn.net/uploads/201211/16/1353059506_5137.jpg)

# SpringMVC接口解释

> 

DispatcherServlet接口：

> 



> Spring提供的前端控制器，所有的请求都有经过它来统一分发。在DispatcherServlet将请求分发给Spring Controller之前，需要借助于Spring提供的HandlerMapping定位到具体的Controller。

HandlerMapping接口：

> 能够完成客户请求到Controller映射。

Controller接口：

> 需要为并发用户处理上述请求，因此实现Controller接口时，必须保证线程安全并且可重用。
>
> Controller将处理用户请求，这和Struts Action扮演的角色是一致的。一旦Controller处理完用户请求，则返回ModelAndView对象给DispatcherServlet前端控制器，ModelAndView中包含了模型（Model）和视图（View）。
>
> 从宏观角度考虑，DispatcherServlet是整个Web应用的控制器；从微观考虑，Controller是单个Http请求处理过程中的控制器，而ModelAndView是Http请求过程中返回的模型（Model）和视图（View）。

ViewResolver接口：

Spring提供的视图解析器（ViewResolver）在Web应用中查找View对象，从而将相应结果渲染给客户。

# SpringMVC运行原理

> \1. 客户端请求提交到DispatcherServlet
>
> \2. 由DispatcherServlet控制器查询一个或多个HandlerMapping，找到处理请求的Controller
>
> \3. DispatcherServlet将请求提交到Controller
>
> \4. Controller调用业务逻辑处理后，返回ModelAndView
>
> \5. DispatcherServlet查询一个或多个ViewResoler视图解析器，找到ModelAndView指定的视图
>
> \6. 视图负责将结果显示到客户端

DispatcherServlet是整个Spring MVC的核心。它负责接收HTTP请求组织协调Spring MVC的各个组成部分。其主要工作有以下三项：

       1. 截获符合特定格式的URL请求。

       2. 截获符合特定格式的URL请求。
       3. 初始化DispatcherServlet上下文对应的WebApplicationContext，并将其与业务层、持久化层的WebApplicationContext建立关联。

       4. 截获符合特定格式的URL请求。
       5. 初始化DispatcherServlet上下文对应的WebApplicationContext，并将其与业务层、持久化层的WebApplicationContext建立关联。

       6. 初始化Spring MVC的各个组成组件，并装配到DispatcherServlet中。



## Struts2请求响应流程：

 

在struts2的应用中，从用户请求到服务器返回相应响应给用户端的过程中，包含了许多组件如：Controller、ActionProxy、ActionMapping、Configuration Manager、ActionInvocation、Inerceptor、Action、Result等。下面我们来具体看看这些组件有什么联系，它们之间是怎样在一起工作的。

![](http://img.blog.csdn.net/20130904161742156?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3V3ZW54aWFuZzkxMzIy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

1）  客户端（Client）向Action发用一个请求（Request）

（2）  [Container](http://lib.csdn.net/base/4)通过web.xml映射请求，并获得控制器（Controller）的名字

（3）  容器（Container）调用控制器（StrutsPrepareAndExecuteFilter或FilterDispatcher）。在Struts2.1以前调用FilterDispatcher，Struts2.1以后调用StrutsPrepareAndExecuteFilter

（4）  控制器（Controller）通过ActionMapper获得Action的信息

（5）  控制器（Controller）调用ActionProxy

（6）  ActionProxy读取struts.xml文件获取action和interceptor stack的信息。

（7）  ActionProxy把request请求传递给ActionInvocation

（8）  ActionInvocation依次调用action和interceptor

（9）  根据action的配置信息，产生result

（10） Result信息返回给ActionInvocation

（11） 产生一个HttpServletResponse响应

（12） 产生的响应行为发送给客服端。

## Struts 和 Spring MVC对比

**1、**Struts2是类级别的拦截， 一个类对应一个request上下文，SpringMVC是方法级别的拦截，一个方法对应一个request上下文，而方法同时又跟一个url对应,所以说从[架构](http://lib.csdn.net/base/16)本身上SpringMVC就容易实现restful url,而struts2的架构实现起来要费劲，因为Struts2中Action的一个方法可以对应一个url，而其类属性却被所有方法共享，这也就无法用注解或其他方式标识其所属方法了。

**2、**由上边原因，SpringMVC的方法之间基本上独立的，独享request response数据，请求数据通过参数获取，处理结果通过ModelMap交回给框架，方法之间不共享变量，而Struts2搞的就比较乱，虽然方法之间也是独立的，但其所有Action变量是共享的，这不会影响程序运行，却给我们编码 读程序时带来麻烦，每次来了请求就创建一个Action，一个Action对象对应一个request上下文。
**3、**由于Struts2需要针对每个request进行封装，把request，session等servlet生命周期的变量封装成一个一个Map，供给每个Action使用，并保证线程安全，所以在原则上，是比较耗费内存的。

**4、** 拦截器实现机制上，Struts2有以自己的interceptor机制，SpringMVC用的是独立的AOP方式，这样导致Struts2的配置文件量还是比SpringMVC大。

**5、**SpringMVC的入口是servlet，而Struts2是filter（这里要指出，filter和servlet是不同的。以前认为filter是servlet的一种特殊），这就导致了二者的机制不同，这里就牵涉到servlet和filter的区别了。

**6、**SpringMVC集成了Ajax，使用非常方便，只需一个注解@ResponseBody就可以实现，然后直接返回响应文本即可，而Struts2拦截器集成了Ajax，在Action中处理时一般必须安装插件或者自己写代码集成进去，使用起来也相对不方便。

**7、**SpringMVC验证支持JSR303，处理起来相对更加灵活方便，而Struts2验证比较繁琐，感觉太烦乱。

**8、**[spring](http://lib.csdn.net/base/17) MVC和Spring是无缝的。从这个项目的管理和安全上也比Struts2高（当然Struts2也可以通过不同的目录结构和相关配置做到SpringMVC一样的效果，但是需要xml配置的地方不少）。

**9、** 设计思想上，Struts2更加符合OOP的编程思想， SpringMVC就比较谨慎，在servlet上扩展。

**10、**SpringMVC开发效率和性能高于Struts2。

**11、**SpringMVC可以认为已经100%零配置。

## 对比

![](http://img.my.csdn.net/uploads/201211/29/1354171519_5139.png)

把这张图放在这里，我是想说SpringMVC和Struts2真的是不一样的，虽然在都有着**核心分发器**等相同的功能组件（这些由MVC模式本身决定的）。

 

为什么SpringMVC会赢得最后的胜利呢？谈几点我自己的看法：

 

第一、MVC框架的出现是为了将URL从HTTP的世界中映射到[Java](http://lib.csdn.net/base/17)世界中，这是MVC框架的核心功能。而在URL这一点SpringMVC无疑更加优雅。

 

第二、从设计实现角度来说，我觉得SpringMVC更加清晰。即使我们去对比Struts2的原理图和SpringMVC的类图，它依然很让人困惑，远没有SpringMVC更加直观：

SpringMVC设计思路：将整个处理流程规范化，并把每一个处理步骤分派到不同的组件中进行处理。

这个方案实际上涉及到两个方面：

l 处理流程规范化 —— 将处理流程划分为若干个步骤（任务），并使用一条明确的逻辑主线将所有的步骤串联起来

l 处理流程组件化 —— 将处理流程中的每一个步骤（任务）都定义为接口，并为每个接口赋予不同的实现模式

处理流程规范化是目的，对于处理过程的步骤划分和流程定义则是手段。因而处理流程规范化的首要内容就是考虑一个通用的Servlet响应程序大致应该包含的逻辑步骤：

l 步骤1—— 对Http请求进行初步处理，查找与之对应的Controller处理类（方法）   ——HandlerMapping

l 步骤2—— 调用相应的Controller处理类（方法）完成业务逻辑                    ——HandlerAdapter

l 步骤3—— 对Controller处理类（方法）调用时可能发生的异常进行处理            ——HandlerExceptionResolver

l 步骤4—— 根据Controller处理类（方法）的调用结果，进行Http响应处理       ——ViewResolver

 

正是这基于组件、接口的设计，支持了SpringMVC的另一个特性：行为的可扩展性。

 

第三、设计原则更加明朗。

    **【Open for extension /closed for modification】**

这条重要的设计原则被写在了[spring](http://lib.csdn.net/base/17)官方的reference中SpringMVC章节的起始段： A key design principle in SpringWeb MVC and in Spring in general is the “Open for extension, closed for modification” principle.

并且重点很好地体现在SpringMVC的实现当中，可以扩展，但却不能改变。我曾经扩展过Spring的IOC、AOP功能，这一点SpringMVC应该和Spring一脉相承。

 

第四、组件化的设计方案和特定的设计原则让SpringMVC形散神聚。

- **神 —— SpringMVC总是沿着一条固定的逻辑主线运行**
- **形 —— SpringMVC却拥有多种不同的行为模式**

SpringMVC是一个基于组件的开发框架，组件的不同实现体系构成了“形”；组件的逻辑串联构成了“神”。因此，“形散神不散”： SpringMVC的逻辑主线始终不变，而行为模式却可以多种多样。

第五、更加贴合Web发展的趋势，这个更加虚了，不再展开说这个 问题了。

 

第六、技术上的放缓导致了程序员对Struts2失去了热情，导致SpringMVC依靠自身的努力和Spring的口碑，逐渐显露了自身的优势和特点。

 

为什么SpringMVC会赢得最后的胜利呢？最后，我们不妨想一想Struts2是怎样流行起来的！

我自己是从Struts1用过来的，后来Struts1的问题很明显了，开源社区出现了很多的MVC框架，最为突出的是**Webwork2。**

Webwork2探索了一条与传统Servlet模型不同的解决方案，逐渐被大家熟识和理解，不断发展并得到了广大程序员的认可。它以优秀的设计思想和灵活的实现，吸引了大批的Web层开发人员投入它的 怀抱。

Apache社区与Opensymphony宣布未来的Struts项目将与Webwork2项目合并，并联合推出Struts2。

Struts2能够在一个相当长的时间段内占据开发市场主导地位的重要原因在于其技术上的领先优势。而这一技术上的领先优势，突出表现为对Controller的彻底改造：

public class UserController {

    private User user

    public String execute() {

        // 这里加入业务逻辑代码

       return "success";

    }

}

 

从上面的代码中，我们可以看到Webwork2 /Struts2对于Controller最大的改造有两点：

- 在Controller中彻底杜绝引入HttpServletRequest或者HttpServletResponse这样的原生Servlet对象。
- 将请求参数和响应数据都从响应方法中剥离到了Controller中的属性变量。

这两大改造被看作是框架的神来之笔。因为通过这一改造，整个Controller类彻底与Web容器解耦，可以方便地进行单元测试。而摆脱了Servlet束缚的Controller，也为整个编程模型赋予了全新的定义。从引入新的编程元素的角度来说，Webwork2 / Struts2无疑也是成功的。因为在传统Servlet模式中的禁地Controller中的属性变量被合理利用了起来作为请求处理过程中的数据部分。这样的改造不仅使得表达式引擎能够得到最大限度的发挥，同时使得整个Controller看起来更像是一个POJO。因而，这种表现形态被笔者冠以的名称 是：POJO实现模式。POJO实现模式是一种具有革命性意义的模式，因为它能够把解耦合这样一个观点发挥到极致。从面向对象的角度来看，POJO模式无疑也是所有程序员所追求的一个目标。这也就是Webwork2 /Struts2那么多年来经久不衰的一个重要原因。

所以，我们看到第一条原因是Struts2依靠技术上的革新赢得了程序员的青睐。但是，这些年来Struts2在技术革新上的作为似乎步子就迈得比较小。我们可以看到，在JDK1.5普及之后，Annotation作为一种新兴的Java语法，逐渐 被大家熟知和应用。这一点上SpringMVC紧跟了时代的潮流，直接用于请求-响应的映射。而Struts2却迟迟无法在单一配置源的问题上形成突破。 当然，这只是技术革新上的一个简单的例子，其他的例子还有很多。

至少给人的感觉是这样的。在这一点上Struts并不是很沾光，因为Spring的口碑和影响力也客观程度上加深了大家对SpirngMVC是技术领导者的印象。





参考：[SpringMVC与Struts2的对比](http://blog.csdn.net/gstormspire/article/details/8239182)