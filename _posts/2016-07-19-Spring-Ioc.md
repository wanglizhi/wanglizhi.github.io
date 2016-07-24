---
layout:     post
title:      "Spring IoC实现解析"
subtitle:   "Spring概述，容器、Bean加载及应用原理"
date:       2016-07-19 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Java Web
    - Spring
---

> 将Spring的核心分为IoC容器和AOP模块，IoC容器用来管理POJO对象和它们之间的耦合关系，属于Spring最核心的模块，AOP则以动态和非侵入式的方式来增强服务的功能，分为两篇文章分别详细介绍其原理。

## Spring的设计理念和整体架构

#### Spring发展历史

Spring是于2003年兴起的一个轻量级Java开源框架，为了解决企业应用开发的复杂性而创建的。相比J2EE规范，它降低了应用开发对传统J2EE技术规范的依赖，而EJB组件要遵循相应的接口，依赖EJB容器，查找其他EJB组件通过JNDI等。

- 2006年10月，Spring2.0
- 2007年11月，Spring2.5，点评目前普遍使用的版本
- 2009年12月，Spring3.0，
- 2013年12月，Spring4.0
- 2016年6月，Spring4.3，目前最新

#### Spring整体架构

Spring框架是一个分层架构，它包含一系列的功能要素，并被分为大约20个模块，如下：

![](http://howtodoinjava.com/wp-content/uploads/2015/02/spring-modules.png)

**1、Core Container（核心容器）**

Core和Beans模块是框架的基础部分，提供IOC和依赖注入特性。这里的基础概念是BeanFactory，它提供对Factory模式的经典实现来消除对程序性单例模式的需要，允许你从程序路基中分离出依赖关系和配置。

- Core模块主要包含Spring框架基本的核心工具类（Util）
- Beans模块是所有应用都要用到的，包含访问配置文件、创建和管理bean以及进行Inversion of Control/Dependency Injection(Ioc/DI)操作相关的类
- Context模块构建于Core和Beans模块之上，提供了一种类似于JNDI注册器的框架式的对象访问方法。它集成了Beans的特性，为Spring核心提供了大量的扩展，添加了对国际化、事件传播、资源加载和对Context的透明创建的支持。ApplicationContext接口是Context模块的关键
- Expression Language模块提供了一个强大的表达式语言，用于在运行时查询和操作对象。

**2、Data Access/Integration**

- JDBC模块提供了一个JDBC抽象层，可以消除冗长的JDBC解码和解析数据库厂商特有的错误代码
- ORM模块为流行的对象-关系映射API，如JPA、JDO、Hibernate、IBatis等提供了一个交互层
- OXM模块提供了一个队Object/XML映射实现的抽象层
- JMS（Java Messaging Service）主要包含一些制造和消费消息的特性
- Transaction模块支持编程和声明性的事务管理

**3、Web**

Web上下文模块建立在应用程序上下文模块之上，为基于Web的应用程序提供了上下文。

- Web模块提供了基础的面向Web的集成特性
- Web-Servlet模块包含Spring的MVC实现
- Spring-WebSocket，**WebSocket 是 HTML5 一种新的协议。它实现了浏览器与服务器全双工通信，能更好的节省服务器资源和带宽并达到实时通讯，它建立在 TCP 之上。WebSocket 是一种双向通信协议，在建立连接后，WebSocket 服务器和 Browser/Client Agent 都能主动的向对方发送或接收数据，就像 Socket 一样；**

**4、AOP**

面向切面编程。让你可以定义如方法拦截器和切点，将逻辑代码分开，降低它们之间的耦合性。

- Aspects模块提供了对AspectJ的集成支持
- Instrumentation模块提供了class instrumentation支持和classloader实现

5、Test

Test模块支持使用JUnit和TestNG对Spring组件进行测试

#### Spring优点

- Spring是一个非侵入性框架，目标是使应用程序代码对框架的依赖最小号，应用代码可以在没有Spring或者其他容器的情况下运行；


- Spring提供了一个一致的编程模型，使应用直接使用POJO开发，从而可以与运行环境隔离开来


- Spring推动应用的设计风格向面向对象及面向接口编程转变，提高了代码的重用性和可测试性
- Spring改进了体系结构的选择，可以帮助我们选择不同的技术实现

## IoC 控制反转

#### 控制反转（IoC）的定义

如果合作对象的引用或依赖关系的管理都由具体对象来完成，会导致代码的高度耦合和可测试性的降低。而这种依赖关系可以通过把对象的依赖注入交给框架或IoC容器来完成，解耦代码并提高可测试性，所以IoC（Inversion of Control）就是把控制权交给容器，反转是指管理对象依赖关系的责任交给了IoC容器，也可以称之为DI（Dependency Injection），IoC容器在对象生成或初始化时直接将数据注入到对象中。

#### 核心组件的关系：Bean、Context、Core

这三个核心组件构建起了整个Spring的骨骼框架，Bean就像OOP中的Object，是Spring中的关键，Context组件就是给Bean提供生存环境，发现每个Bean之间的关系，为它们建立这种关系并且要维护好这种关系，所以Context就是一个Bean的关系集合，又称为IoC容器。而Core就是发现、建立和维护每个Bean之间关系锁需要的一系列工具，可以叫它Util。

三个组件的关系：

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image002.gif)

例如：Spring中我们可以看到两个主要的容器系列，一个是实现BeanFactory接口的简单容器系列，只实现了容器最基本的功能，它属于Bean组件；另一个是Context组件中的ApplicationContext应用上下文，它作为容器的高级形态而存在，在BeanFactory的基础上，增加了许多面向框架的特性，同时对应用环境做了许多适配。

### Bean组件

Bean组件在Spring的org.springframework.beans包下，主要解决了三件事：Bean的定义、Bean的创建以及对Bean的解析。对Spring的使用者来说唯一需要关心的就是Bean的创建，其他两个由Spring在内部帮你完成了，对你来说是透明的。

#### Bean的创建，BeanFactory与FactoryBean

Bean工厂的继承关系：

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image003.png)

BeanFactory有三个子类：ListableBeanFactory、HierarchicalBeanFactory和AutowireCapableBeanFactory，上图中默认实现类是DefaultListableBeanFactory，它实现了所有的接口。那为何要定义这么多层次的接口能？查阅接口源码和说明发现，每个接口都有它使用的场合，主要是为了区分Spring内部在操作过程中对象的传递和转化过程中，对对象的数据访问所做的限制。ListableBeanFactory接口表示这些Bean是可列表的，HierarchicalBeanFactory表示这些Bean是有继承关系的，每个Bean可能有父Bean，AutowireCapableBeanFactory接口定义Bean的自动装配规则。

BeanFactory：

```java
public interface BeanFactory {
  // 用来引用一个实例，或把它和工厂生产的Bean区分开，如果一个FactoryBean的名字为a，那么&a会得到那个Factory
    String FACTORY_BEAN_PREFIX = "&";
//四种不同形式的getBean方法获取实例
    Object getBean(String var1) throws BeansException;

    Object getBean(String var1, Class var2) throws BeansException;

    Object getBean(String var1, Object[] var2) throws BeansException;

    boolean containsBean(String var1); //是否存在
// 是否是单例
    boolean isSingleton(String var1) throws NoSuchBeanDefinitionException;
// 是否为原型
    boolean isPrototype(String var1) throws NoSuchBeanDefinitionException;
// 名称、类型是否匹配
    boolean isTypeMatch(String var1, Class var2) throws NoSuchBeanDefinitionException;
// 获取类型
    Class getType(String var1) throws NoSuchBeanDefinitionException;
// 根据实例的名字获取实例的别名
    String[] getAliases(String var1);
}
```

FactoryBean：

```java
public interface FactoryBean {
  // 返回由FactoryBean创建的Bean实例，如果isSingleton()返回true，则该实例会放到Spring容器中单例缓存池中
    Object getObject() throws Exception;
// 类型
    Class getObjectType();
// 作用域是单例还是原型
    boolean isSingleton();
}
```

两者的区别：BeanFactory定义了IOC容器的最基本的形式，并提供了IOC容器应遵守的最基本的接口；而FactoryBean则让用户可以通过实现该接口**定制实例化Bean的逻辑**，如果按照传统Spring反射机制需要在< bean >中提供大量配置信息，灵活性受限。

#### Bean的定义，BeanDefinition

Bean的定义就是完整的描述了在Spring的配置文件中你定义的< bean >节点中所有的信息，包括各种子节点。当 Spring 成功解析你定义的一个 <bean/> 节点后，在 Spring 的内部他就被转化成 BeanDefinition 对象。以后所有的操作都是对这个对象完成的。

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image004.png)

AbstractBeanDefinition：

```java
public abstract class AbstractBeanDefinition extends BeanMetadataAttributeAccessor implements BeanDefinition, Cloneable {
 private volatile Object beanClass;
    private String scope;//默认scope为“”，相当于单例 
    private boolean singleton;//bean默认为单例的
    private boolean prototype;//bean默认不是原型的  
    private boolean abstractFlag;//bean默认不是抽象的 
    private boolean lazyInit;//bean默认不开启延迟初始化
    private int autowireMode;//bean默认自动装配功能是关闭的
    private int dependencyCheck;//bean的默认依赖检查方式 
    private String[] dependsOn;//这个bean要初始化依赖的bean名称数组 
    private boolean autowireCandidate;//自动装配候选者  
    private boolean primary;//默认不是主要候选者
    private final Map<String, AutowireCandidateQualifier> qualifiers;//用于记录Qualifier，对应子元素qualifier
    private boolean nonPublicAccessAllowed;//允许访问非公开的构造器和方法
    private boolean lenientConstructorResolution;//是否以一种宽松的模式解析构造函数，默认true
    private ConstructorArgumentValues constructorArgumentValues;//记录构造函数注入属性，对应bean属性constructor-arg
    private MutablePropertyValues propertyValues;//普通属性集合
    private MethodOverrides methodOverrides;//方法重写持有者
    private String factoryBeanName;// 对应bean属性: factory-bean
    private String factoryMethodName;//对应bean属性: factory-method
    private String initMethodName;//初始化方法，对应bean属性: init-method
    private String destroyMethodName;// 销毁方法，对应destory-method
    private boolean enforceInitMethod;// 是否执行init-method
    private boolean enforceDestroyMethod;//是否执行destory-method
    private boolean synthetic;//是否是用户定义的而不是应用程序本身定义的
    private int role;//定义这个bean的应用
    private String description;// bean的描述
    private Resource resource; //bean定义的资源
}
```

Spring容器中的bean可以分为5个范围。所有范围的名称都是自说明的，但是为了避免混淆，还是让我们来解释一下：

1. singleton：这种bean范围是默认的，这种范围确保不管接受到多少个请求，每个容器中只有一个bean的实例，单例的模式由bean factory自身来维护。
2. prototype：原形范围与单例范围相反，为每一个bean请求提供一个实例。
3. request：在请求bean范围内会每一个来自客户端的网络请求创建一个实例，在请求完成以后，bean会失效并被垃圾回收器回收。
4. Session：与请求范围类似，确保每个session中有一个bean的实例，在session过期后，bean会随之失效。
5. global-session：global-session和Portlet应用相关。当你的应用部署在Portlet容器中工作时，它包含很多portlet。如果你想要声明让所有的portlet共用全局的存储变量的话，那么这全局变量需要存储在global-session中。

#### Bean的解析

Bean 的解析过程非常复杂，功能被分的很细，因为这里需要被扩展的地方很多，必须保证有足够的灵活性，以应对可能的变化。Bean 的解析主要就是对 Spring 配置文件的解析。这个解析过程主要通过下图中的类完成

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image005.png)

```java
public interface BeanDefinitionReader {
    BeanDefinitionRegistry getRegistry();
    ResourceLoader getResourceLoader();
    ClassLoader getBeanClassLoader();
    BeanNameGenerator getBeanNameGenerator();
    int loadBeanDefinitions(Resource var1) throws BeanDefinitionStoreException;
    int loadBeanDefinitions(Resource... var1) throws BeanDefinitionStoreException;
    int loadBeanDefinitions(String var1) throws BeanDefinitionStoreException;
    int loadBeanDefinitions(String... var1) throws BeanDefinitionStoreException;
}
```

### Context组件

Context 在 Spring 的 org.springframework.context 包下，他实际上就是给 Spring 提供一个运行时的环境，用以保存各个对象的状态。ApplicationContext 是 Context 的顶级父类，他除了能标识一个应用环境的基本信息外，他还继承了五个接口，这五个接口主要是扩展了 Context 的功能。

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image006.png)

从上图中可以看出 ApplicationContext 继承了 BeanFactory，这也说明了 Spring 容器中运行的主体对象是 Bean，另外 ApplicationContext 继承了 ResourceLoader 接口，使得 ApplicationContext 可以访问到任何外部资源，这将在 Core 中详细说明。

ApplicationContext 的子类主要包含两个方面：

1. ConfigurableApplicationContext 表示该 Context 是可修改的，也就是在构建 Context 中用户可以动态添加或修改已有的配置信息，它下面又有多个子类，其中最经常使用的是可更新的 Context，即 AbstractRefreshableApplicationContext 类。
2. WebApplicationContext 顾名思义，就是为 web 准备的 Context 他可以直接访问到 ServletContext，通常情况下，这个接口使用的少。

再往下分就是按照构建 Context 的文件类型，接着就是访问 Context 的方式。这样一级一级构成了完整的 Context 等级层次。

总体来说 ApplicationContext 必须要完成以下几件事：

- 标识一个应用环境
- 利用 BeanFactory 创建 Bean 对象
- 保存对象关系表
- 能够捕获各种事件

Context 作为 Spring 的 Ioc 容器，基本上整合了 Spring 的大部分功能，或者说是大部分功能的基础。

### Core组件

Core 组件作为 Spring 的核心组件，他其中包含了很多的关键类，其中一个重要组成部分就是定义了资源的访问方式。这种**把所有资源都抽象成一个接口**的方式很值得在以后的设计中拿来学习。

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image007.png)

从上图可以看出 Resource 接口封装了各种可能的资源类型，也就是对使用者来说屏蔽了文件类型的不同。对资源的提供者来说，如何把资源包装起来交给其他人用这也是一个问题，我们看到 Resource 接口继承了 InputStreamSource 接口，这个接口中有个 getInputStream 方法，返回的是 InputStream 类。这样所有的资源都被可以通过 InputStream 这个类来获取，所以也屏蔽了资源的提供者。另外还有一个问题就是加载资源的问题，也就是资源的加载者要统一，从上图中可以看出这个任务是由 ResourceLoader 接口完成，他屏蔽了所有的资源加载者的差异，只需要实现这个接口就可以加载所有的资源，他的默认实现是 DefaultResourceLoader。

#### Context 和 Resource 的类关系图

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image008.png)

从上图可以看出，Context 是把资源的加载、解析和描述工作委托给了 ResourcePatternResolver 类来完成，他相当于一个接头人，他把资源的加载、解析和资源的定义整合在一起便于其他组件使用。Core 组件中还有很多类似的方式。

### **Ioc 容器如何工作**

前面介绍了 Core 组件、Bean 组件和 Context 组件的结构与相互关系，下面这里从使用者角度看一下他们是如何运行的，以及我们如何让 Spring 完成各种功能，Spring 到底能有那些功能，这些功能是如何得来的，下面介绍。

#### **如何创建 BeanFactory 工厂**

Ioc 容器实际上就是 Context 组件结合其他两个组件共同构建了一个 Bean 关系网，如何构建这个关系网？构建的入口就在 AbstractApplicationContext 类的 refresh 方法中。这个方法的代码如下：

```java
public void refresh() throws BeansException, IllegalStateException { 
    synchronized (this.startupShutdownMonitor) { 
        // Prepare this context for refreshing. 
        prepareRefresh(); 
        // 在子类中启动refreshBeanFactory 
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory(); 
        // Prepare the bean factory for use in this context. 
        prepareBeanFactory(beanFactory); 
        try { 
            // 设置BeanFactory的后置处理 
            postProcessBeanFactory(beanFactory); 
            // 调用BeanFactory的后处理器，这些后处理器是在Bean定义中向容器注册的 
            invokeBeanFactoryPostProcessors(beanFactory); 
            // 注册Bean的后处理器，在Bean创建过程中调用
            registerBeanPostProcessors(beanFactory); 
            // 对上下文的消息源进行初始化
            initMessageSource(); 
            // 初始化上下文中的事件机制
            initApplicationEventMulticaster(); 
            // 初始化其他的特殊Bean 
            onRefresh(); 
            // 检查坚挺Bean并且将这些Bean向容器注册
            registerListeners(); 
            // 实例化所有(non-lazy-init)单件 
            finishBeanFactoryInitialization(beanFactory); 
            // 发布容器事件，结束Refresh过程 
            finishRefresh(); 
        } 
        catch (BeansException ex) { 
            // Destroy already created singletons to avoid dangling resources. 
            destroyBeans(); 
            // Reset 'active' flag. 
            cancelRefresh(ex); 
            // Propagate exception to caller. 
            throw ex; 
        } 
    } 
}
```

这个方法就是构建整个 Ioc 容器过程的完整的代码，了解了里面的每一行代码基本上就了解大部分 Spring 的原理和功能了。

这段代码主要包含这样几个步骤：

- 构建 BeanFactory，以便于产生所需的“演员”
- 注册可能感兴趣的事件
- 创建 Bean 实例对象
- 触发被监听的事件

下面就结合代码分析这几个过程。

第二三句就是在创建和配置 BeanFactory。这里是 refresh 也就是刷新配置，前面介绍了 Context 有可更新的子类，这里正是实现这个功能，当 BeanFactory 已存在是就更新，如果没有就新创建。下面是更新 BeanFactory 的方法代码：

```java
protected final void refreshBeanFactory() throws BeansException { 
    if (hasBeanFactory()) { 
        destroyBeans(); 
        closeBeanFactory(); 
    } 
    try { 
        DefaultListableBeanFactory beanFactory = createBeanFactory(); 
        beanFactory.setSerializationId(getId()); 
        customizeBeanFactory(beanFactory); 
        loadBeanDefinitions(beanFactory); 
        synchronized (this.beanFactoryMonitor) { 
            this.beanFactory = beanFactory; 
        } 
    } 
    catch (IOException ex) { 
        throw new ApplicationContextException(
			"I/O error parsing bean definition source for " 
			+ getDisplayName(), ex); 
    } 
}
```

这个方法实现了 AbstractApplicationContext 的抽象方法 refreshBeanFactory，这段代码清楚的说明了 BeanFactory 的创建过程。注意 BeanFactory 对象的类型的变化，前面介绍了他有很多子类，在什么情况下使用不同的子类这非常关键。BeanFactory 的原始对象是 DefaultListableBeanFactory，这个非常关键，因为他设计到后面对这个对象的多种操作。

DefaultListableBeanFactory与 Bean 的 register 相关。这在 refreshBeanFactory 方法中有一行 loadBeanDefinitions(beanFactory) 将找到答案，这个方法将开始加载、解析 Bean 的定义，也就是把用户定义的数据结构转化为 Ioc 容器中的特定数据结构。

##### 创建 BeanFactory 时序图

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image010.png)

##### 解析和登记 Bean 对象时序图

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image011.png)

创建好 BeanFactory 后，接下去添加一些 Spring 本身需要的一些工具类，这个操作在 AbstractApplicationContext 的 prepareBeanFactory 方法完成。

AbstractApplicationContext 中接下来的三行代码对 Spring 的功能扩展性起了至关重要的作用。前两行主要是让你现在可以对已经构建的 BeanFactory 的配置做修改，后面一行就是让你可以对以后再创建 Bean 的实例对象时添加一些自定义的操作。所以他们都是扩展了 Spring 的功能，所以我们要学习使用 Spring 必须对这一部分搞清楚。

#### 添加一些自定义的操作：BeanFactoryPostProcessor

其中在 invokeBeanFactoryPostProcessors 方法中主要是获取实现 BeanFactoryPostProcessor 接口的子类。并执行它的 postProcessBeanFactory 方法，这个方法的声明如下：

```java
void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) 
    throws BeansException;
```

它的参数是 beanFactory，说明可以对 beanFactory 做修改，这里注意这个 beanFactory 是 ConfigurableListableBeanFactory 类型的，这也印证了前面介绍的不同 BeanFactory 所使用的场合不同，这里只能是可配置的 BeanFactory，防止一些数据被用户随意修改。

registerBeanPostProcessors 方法也是可以获取用户定义的实现了 BeanPostProcessor 接口的子类，并执行把它们注册到 BeanFactory 对象中的 beanPostProcessors 变量中。BeanPostProcessor 中声明了两个方法：postProcessBeforeInitialization、postProcessAfterInitialization 分别用于在 Bean 对象初始化时执行。**可以执行用户自定义的操作**。

```java
public interface BeanPostProcessor {
    Object postProcessBeforeInitialization(Object var1, String var2) throws BeansException;

    Object postProcessAfterInitialization(Object var1, String var2) throws BeansException;
}
```

后面的几行代码是初始化监听事件和对系统的其他监听者的注册，监听者必须是 ApplicationListener 的子类。

#### **如何创建 Bean 实例并构建 Bean 的关系网**

下面就是 Bean 的实例化代码，是从 finishBeanFactoryInitialization 方法开始的。**AbstractApplicationContext.finishBeanFactoryInitialization**

```java
protected void finishBeanFactoryInitialization(
	ConfigurableListableBeanFactory beanFactory) { 

    // Stop using the temporary ClassLoader for type matching. 
    beanFactory.setTempClassLoader(null); 
    // Allow for caching all bean definition metadata, not expecting further changes.
    beanFactory.freezeConfiguration(); 
    // Instantiate all remaining (non-lazy-init) singletons. 
    beanFactory.preInstantiateSingletons(); 
}
```

从上面代码中可以发现 Bean 的实例化是在 BeanFactory 中发生的。preInstantiateSingletons 方法的代码如下：**DefaultListableBeanFactory.preInstantiateSingletons**

```java
public void preInstantiateSingletons() throws BeansException { 
    if (this.logger.isInfoEnabled()) { 
        this.logger.info("Pre-instantiating singletons in " + this); 
    } 
    synchronized (this.beanDefinitionMap) { 
        for (String beanName : this.beanDefinitionNames) { 
            RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName); 
            if (!bd.isAbstract() && bd.isSingleton() 
                && !bd.isLazyInit()) {
                if (isFactoryBean(beanName)) { 
                    final FactoryBean factory = 
                        (FactoryBean) getBean(FACTORY_BEAN_PREFIX+ beanName); 
                    boolean isEagerInit; 
                    if (System.getSecurityManager() != null 
                        && factory instanceof SmartFactoryBean) { 
                        isEagerInit = AccessController.doPrivileged(
                            new PrivilegedAction<Boolean>() { 
                            public Boolean run() { 
                                return ((SmartFactoryBean) factory).isEagerInit(); 
                            } 
                        }, getAccessControlContext()); 
                    } 
                    else { 
                        isEagerInit = factory instanceof SmartFactoryBean 
                            && ((SmartFactoryBean) factory).isEagerInit(); 
                    } 
                    if (isEagerInit) { 
                        getBean(beanName); 
                    } 
                } 
                else { 
                    getBean(beanName); 
                } 
            } 
        } 
    } 
}
```

这里出现了一个非常重要的 Bean —— FactoryBean，可以说 Spring 一大半的扩展的功能都与这个 Bean 有关，这是个特殊的 Bean 他是个工厂 Bean，可以产生 Bean 的 Bean，这里的产生 Bean 是指 Bean 的实例，如果一个类继承 FactoryBean 用户可以自己定义产生实例对象的方法只要实现他的 getObject 方法。然而在 Spring 内部这个 Bean 的实例对象是 FactoryBean，通过调用这个对象的 getObject 方法就能获取用户自定义产生的对象，从而为 Spring 提供了很好的扩展性。Spring 获取 FactoryBean 本身的对象是在前面加上 & 来完成的。

如何创建 Bean 的实例对象以及如何构建 Bean 实例对象之间的关联关系式 Spring 中的一个核心关键，下面是这个过程的流程图。

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image012.gif)

如果是普通的 Bean 就直接创建他的实例，是通过调用 getBean 方法。下面是创建 Bean 实例的时序图：

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image013.png)

还有一个非常重要的部分就是建立 Bean 对象实例之间的关系，这也是 Spring 框架的核心竞争力，何时、如何建立他们之间的关系请看下面的时序图：

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/origin_image014.png)

#### 循环依赖

循环依赖就是循环引用，就是两个或多个bean相互之间的持有对方。在Spring中将循环依赖的处理分成了3种情况。

1、构造器循环依赖

通过构造器注入构成的循环依赖，是无法解决的

2、setter循环依赖

- Spring容器创建单例“circleA” Bean，首先根据无参构造器创建Bean，并暴露一个“ObjectFactory ”用于返回一个提前暴露一个创建中的Bean，并将“circleA” 标识符放到“当前创建Bean池”；然后进行setter注入“circleB”；


- Spring容器创建单例“circleB” Bean，首先根据无参构造器创建Bean，并暴露一个“ObjectFactory”用于返回一个提前暴露一个创建中的Bean，并将“circleB” 标识符放到“当前创建Bean池”，然后进行setter注入“circleC”；


- Spring容器创建单例“circleC” Bean，首先根据无参构造器创建Bean，并暴露一个“ObjectFactory ”用于返回一个提前暴露一个创建中的Bean，并将“circleC” 标识符放到“当前创建Bean池”，然后进行setter注入“circleA”；进行注入“circleA”时由于提前暴露了“ObjectFactory”工厂从而使用它返回提前暴露一个创建中的Bean；


- 最后在依赖注入“circleB”和“circleA”，完成setter注入。

3、Prototype也范围的依赖处理

对于“prototype”作用域Bean，Spring容器无法完成依赖注入，因为“prototype”作用域的Bean，Spring容器不进行缓存，因此无法提前暴露一个创建中的Bean。

#### **Ioc 容器的扩展点**

对 Spring 的 Ioc 容器来说，主要有这么几个。`BeanFactoryPostProcessor`， `BeanPostProcessor`。他们分别是在构建 BeanFactory 和构建 Bean 对象时调用。还有就是 InitializingBean 和 DisposableBean 他们分别是在 Bean 实例创建和销毁时被调用。用户可以实现这些接口中定义的方法，Spring 就会在适当的时候调用他们。还有一个是 FactoryBean 他是个特殊的 Bean，这个 Bean 可以被用户更多的控制。

**BeanFactoryPostProcessor是在spring容器加载了bean的定义文件之后，在bean实例化之前执行的。接口方法的入参是ConfigurrableListableBeanFactory，使用该参数，可以获取到相关bean的定义信息**

SpringConfig（lion）、SqlClientReplacer（读写Master/Slave）

```java
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory var1) throws BeansException;
}
```

**BeanPostProcessor**，可以在**spring容器实例化bean之后**，在执行bean的初始化方法前后，添加一些自己的处理逻辑。这里说的初始化方法，指的是下面两种：1）bean实现了InitializingBean接口，对应的方法为afterPropertiesSet2）在bean定义的时候，通过init-method设置的方法

**注意：BeanPostProcessor是在spring容器加载了bean的定义文件并且实例化bean之后执行的。BeanPostProcessor的执行顺序是在BeanFactoryPostProcessor之后。**

BeanValidationPostProcessor、CommonAnnotationBeanPostProcessor、InitDestroyAnnotationBeanPostProcessor（@PostConstruct）

```java
public interface BeanPostProcessor {
    Object postProcessBeforeInitialization(Object var1, String var2) throws BeansException;

    Object postProcessAfterInitialization(Object var1, String var2) throws BeansException;
}
```



#### BeanFactoryPostProcessor实践：Slave注解修改Bean

通过@Slave注解可以替换DAO类中的SqlClient，实现主从库的轻松切换。实现方式即在Bean加载完时，通过BeanFactory获得所有的Bean的列表，判断是否有Slave注解，如果有，则替换。

```java
@Component
public class SqlClientReplacer implements BeanFactoryPostProcessor, PriorityOrdered{
   
   private static boolean hasProcessd;
   private DefaultListableBeanFactory listableBeanFactory = null;

   @Override
   public int getOrder() {
      return Ordered.LOWEST_PRECEDENCE - 2;
   }

   @Override
   public void postProcessBeanFactory(
         ConfigurableListableBeanFactory beanFactory) throws BeansException {
      if (hasProcessd) {
         return;
      }
      hasProcessd = true;

      listableBeanFactory = (DefaultListableBeanFactory) beanFactory;
      String[] beanDefinitionNames = listableBeanFactory.getBeanNamesForType(BaseDAO.class);
      
      for (String definitionName : beanDefinitionNames) {
         try {
            AbstractBeanDefinition beanDefinition = (AbstractBeanDefinition) listableBeanFactory
                  .getBeanDefinition(definitionName);
            Class<?> beanClazz = beanDefinition.resolveBeanClass(ClassUtils.getDefaultClassLoader());
            if (beanClazz != null && BaseDAO.class.isAssignableFrom(beanClazz)) {
               BeanDefinition definition = listableBeanFactory.getBeanDefinition(definitionName);
               
               Slave slave = beanClazz.getAnnotation(Slave.class);
               if(slave != null){
                  definition.getPropertyValues().addPropertyValue("sqlMapClient", new RuntimeBeanReference("bookingSlaveSqlMapClient"));
               }else{
                  definition.getPropertyValues().addPropertyValue("sqlMapClient", new RuntimeBeanReference("bookingMasterSqlMapClient"));
               }
            }
         } catch (ClassNotFoundException e) {
            throw new RuntimeException("postProcessBeanFactory failed", e);
         }
      }
   }

}
```

#### BeanFactoryPostProcessor实践：lion的SpringConfig

```java
public class SpringConfig extends InitializeConfig ;
<bean name="placeholder" lazy-init="false" class="com.dianping.lion.client.SpringConfig">
		<property name="propertiesPath" value="config/applicationContext.properties" />
	</bean>
  // 具体实现，看父类InitializeConfig，将${}中的字符串替换成配置
  public class InitializeConfig implements BeanFactoryPostProcessor, PriorityOrdered, BeanNameAware, BeanFactoryAware {
    private static Logger logger;
    public static final String DEFAULT_PLACEHOLDER_PREFIX = "${";
    public static final String DEFAULT_PLACEHOLDER_SUFFIX = "}";
    private String placeholderPrefix = "${";
    private String placeholderSuffix = "}";
    private int order = 2147483647;
    private BeanFactory beanFactory;
    private String beanName;
    private String nullValue = "";
    protected String address;
    private String environment;
    private String propertiesPath;
    private boolean includeLocalProps;
    private Properties localProps;
    private Map<String, List<BeanData>> propertyMap = new HashMap();
  }
```



参考：

《Spring技术内幕：深入分析Spring架构与设计原理》

《Spring源码深度解析》

[Spring 框架的设计理念与设计模式分析](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/index.html)