---
layout:     post
title:      "Spring AOP实现解析"
subtitle:   "Java动态代理，CGLib，Spring AOP用法等"
date:       2016-07-21 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Java Web
    - Spring
---

> 将Spring的核心分为IoC容器和AOP模块，IoC容器用来管理POJO对象和它们之间的耦合关系，属于Spring最核心的模块，AOP则以动态和非侵入式的方式来增强服务的功能，分为两篇文章分别详细介绍其原理。

## 代理模式

代理模式就是给某一个对象创建一个代理对象，而由这个代理对象控制对原对象的引用，而创建这个代理对象就是可以在调用原对象是可以增加一些额外的操作。下面是代理模式的结构：

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image020.png)

- Subject：抽象主题，它是代理对象的真实对象要实现的接口，当然这可以是多个接口组成。
- ProxySubject：代理类除了实现抽象主题定义的接口外，还必须持有所代理对象的引用
- RealSubject：被代理的类，是目标对象。

```java
// 静态代理
public class GreetingProxy implements Greeting {
    private GreetingImpl greetingImpl;
    public GreetingProxy(GreetingImpl greetingImpl) {
        this.greetingImpl = greetingImpl;
    }
    @Override
    public void sayHello(String name) {
        before();
        greetingImpl.sayHello(name);
        after();
    }
    private void before() {
        System.out.println("Before");
    }
    private void after() {
        System.out.println("After");
    }
}
public static void main(String[] args) {
        Greeting greetingProxy = new GreetingProxy(new GreetingImpl());
        greetingProxy.sayHello("Jack");
    }
```

## Java动态代理

```java
// Subject
public interface OnlineShop{
  void sellSomething(double money);
}
//RealSubjcet
public class TeaOnlineShop implements OnlineShop{
  public void sellSomething(double money){
    System.out.println("shop say : give me " + money + " and I sell you some tea");
  }
}
// ProxySubject
public class TaobaoProxy implements InvocationHandler{
  // 被代理的对象
  private Object proxied;
  public Object invoke(Object proxy, Method method, Object[] args){
    System.out.println("taobao say: "+ args[0]+" money save to taobao to increase my gmv");
    before();
    Object result = method.invoke(proxid, args);
    after();
    return result;
  }
}
@Test
public void test(){
  TeaOnlineShop teaShop = new TeaOnlineShop();
  //注意这里的三个参数
  OnlineShop shop = (OnlineShop)Proxy.newProxyInstance(OnlineShop.class.getClassLoader(), new Class[]{OnlineShop.class}, new TaobaoProxy(teaShop));
  shop.sellSomething(20d);
}
// 嫌弃那三个参数的话，可以在TaobaoProxy里加上一个方法封装起来
@SuppressWarnings("unchecked")
public < T > T getProxy(){
  return (T)Proxy.newProxyInstance(
    proxied.getClass().getClassLoader(),
    proxied.getClass().getInterfaces(),
    this
  );
}
//使用起来就变成了
TaobaoProxy taobaoProxy = new TaobaoProxy(new OnlineShop());
OnlineShop onlineShop = taobaoProxy.getProxy();
onlineShop.sellSomething(20d);
```

在 Jdk 的 java.lang.reflect 包下有个 Proxy 类，它正是构造代理类的入口:

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image015.png)

从上图发现最后面四个是公有方法。而最后一个方法 newProxyInstance 就是创建代理对象的方法。这个方法的源码如下：

```java
public static Object newProxyInstance(ClassLoader loader, 
    Class<?>[] interfaces, 
    InvocationHandler h) 
    throws IllegalArgumentException { 
    
	if (h == null) { 
        throw new NullPointerException(); 
    } 
    Class cl = getProxyClass(loader, interfaces); 
    try { 
        Constructor cons = cl.getConstructor(constructorParams);
        return (Object) cons.newInstance(new Object[] { h }); 
    } catch (NoSuchMethodException e) { 
        throw new InternalError(e.toString()); 
    } catch (IllegalAccessException e) { 
        throw new InternalError(e.toString()); 
    } catch (InstantiationException e) { 
        throw new InternalError(e.toString()); 
    } catch (InvocationTargetException e) { 
        throw new InternalError(e.toString()); 
    } 
}
```

这个方法需要三个参数：ClassLoader，用于加载代理类的 Loader 类，通常这个 Loader 和被代理的类是同一个 Loader 类。Interfaces，是要被代理的那些那些接口。InvocationHandler，就是用于执行除了被代理接口中方法之外的用户自定义的操作，他也是用户需要代理的最终目的。用户调用目标方法都被代理到 InvocationHandler 类中定义的唯一方法 invoke 中。这在后面再详解。

##### 创建代理对象时序图

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image016.png)

假如有这样一个接口，如下：

```java
public interface SimpleProxy { 

    public void simpleMethod1(); 
	
    public void simpleMethod2(); 

}
```

代理来生成的类结构如下：

```java
public class $Proxy2 extends java.lang.reflect.Proxy implements SimpleProxy{ 
    java.lang.reflect.Method m0; 
    java.lang.reflect.Method m1; 
    java.lang.reflect.Method m2; 
    java.lang.reflect.Method m3; 
    java.lang.reflect.Method m4; 

    int hashCode(); 
    boolean equals(java.lang.Object); 
    java.lang.String toString(); 
    void simpleMethod1(); 
    void simpleMethod2(); 
}
```

这个类中的方法里面将会是调用 InvocationHandler 的 invoke 方法，而每个方法也将对应一个属性变量，这个属性变量 m 也将传给 invoke 方法中的 Method 参数。整个代理就是这样实现的。

## CGLib代理

JDK动态代理只能代理有接口的类，如果没有接口呢？答案是采用CGLib类库，一个在运行期间动态生成字节码的工具，原理是对指定的目标类生成一个子类，并覆盖其中方法实现增强，**但因为采用的是继承，所以不能对final修饰的类进行代理**。CGLIB是高效的代码生成包，底层依靠ASM操作字节码实现，性能比JDK强。上述Java动态代理方法用CGLib实现如下

```java
// cglib不需要定义接口，直接给出具体实现
public class TeaOnlineShop{
  public void sellSomething(double money){
    System.out.println("shop say : give me " + money + " and I sell you some tea");
  }
}
// cglib需要实现MethodInterceptor接口
public class TaobaoProxy implements MethodInterceptor{
  public < T > T getProxy(Class clazz){
    return (T)Enhancer.create(clazz, this);
  }
  //拦截器
  public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy){
    System.out.println("taobao say: "+ args[0]+" money save to taobao to increase my gmv");
    before();
    Object result = proxy.invokeSuper(obj, args);
    after();
    return result;
  }
}
@Test
public void test(){
  TaobaoProxy proxy = new TaobaoProxy();
  // 通过生成子类的方式创建代理类
  TeaOnlineShop shopProxy = proxy.getProxy(TeaOnlineShop.class);
  shopProxy.sellSomething(21d);
}
```

## Spring动态AOP使用示例

Spring 同时支持JDK动态代理和 CGLIB 类代理，使用示例如下：

```java
//创建用于拦截的bean
public class TestBean{
  private String testStr = "testStr";
  public String getTestStr(){
    return testStr;
  }
  public void setTestStr(String testStr){
    this.testStr = testStr;
  }
  public void test(){
    System.out.println("test");
  }
}
//创建Advisor
@Aspect
public class AspectJTest{
  @Pointcut("execution(* *.test(...))")
  public void test(){}
  @Before("test()")
  public void beforeTest(){
    System.out.println("beforeTest");
  }
  @After("test()")
  public void afterTest(){
    System.out.println("afterTest");
  }
  @Around("test()")
  public Object arountTest(ProceedingJoinPoint p){
    System.out.println("before1");
    Object o = null;
    try{
      o = p.proceed();
    }catch;
    
    System.out.println("after1");
    return o;
}
}
// 创建配置文件
<aop:aspectj-autoproxy/>
<bean id = "test" class = "test.TestBean"/>
<bean class = "test.AspectJTest"/>
//测试
public static void main(String[] args){
  ApplicationContext bf = new ClassPathXmlApplicationContext("apspectTest.xml");
  TestBean bean = (TestBean)bf.getBean("test");
  bean.test();
}
```

不出意外打印结果为：

beforeTest

before1

test

afterTest

after1

## Spring创建代理及调用拦截器

前面提到 Spring Aop 也是实现其自身的扩展点来完成这个特性的，从这个代理类可以看出它正是继承了 FactoryBean 的 ProxyFactoryBean，FactoryBean 之所以特别就在它可以让你自定义对象的创建方法。当然代理对象要通过 Proxy 类来动态生成。

下面是 Spring 创建的代理对象的时序图：

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image018.png)

Spring 创建了代理对象后，当你调用目标对象上的方法时，将都会被代理到 InvocationHandler 类的 invoke 方法中执行，这在前面已经解释。在这里 JdkDynamicAopProxy 类实现了 InvocationHandler 接口。

下面再看看 Spring 是如何调用拦截器的，下面是这个过程的时序图

![](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/image019.png)

## Spring AOP

#### Spring AOP：前置增强、后置增强、环绕增强（编程式）

代理调用前的before()方法，在Spring里就叫做**Before Advice（前置增强）**，after()方法叫做**After Advice（后置增强）**，如果把两个方法合并在一起，那就叫做**Around Advice（环绕增强）**。在这里，把Advice直译为“通知”不太合适，以为它没有通知的含义而是增强，再说CGLib中也有一个Enhancer类，它就是一个增强类。

```java
// 定义一个增强类
public class GreetingBeforeAndAfterAdvice implements MethodBeforeAdvice, AfterReturningAdvice {
    @Override
    public void before(Method method, Object[] args, Object target) throws Throwable {
        System.out.println("Before");
    }
    @Override
    public void afterReturning(Object result, Method method, Object[] args, Object target) throws Throwable {
        System.out.println("After");
    }
}
public static void main(String[] args) {
        ProxyFactory proxyFactory = new ProxyFactory();     // 创建代理工厂
        proxyFactory.setTarget(new GreetingImpl());         // 射入目标类对象
        proxyFactory.addAdvice(new GreetingBeforeAndAfterAdvice());  // 添加增强类 
        Greeting greeting = (Greeting) proxyFactory.getProxy(); // 从代理工厂中获取代理
        greeting.sayHello("Jack");                              // 调用代理的方法
    }
```

好了，这就是 Spring AOP 的基本用法，但这只是“编程式”而已。Spring AOP 如果只是这样，那就太傻逼了，它曾经也是一度宣传用 Spring 配置文件的方式来定义 Bean 对象，把代码中的 new 操作全部解脱出来。

#### Spring AOP：前置增强、后置增强、环绕增强（声明式）

Spring配置文件：

```xml
<!-- 扫描指定包（将 @Component 注解的类自动定义为 Spring Bean） -->
    <context:component-scan base-package="aop.demo"/>
    <!-- 配置一个代理 -->
    <bean id="greetingProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="interfaces" value="aop.Greeting"/> <!-- 需要代理的接口 -->
        <property name="target" ref="greetingImpl"/>       <!-- 接口实现类 -->
        <property name="interceptorNames">                 <!-- 拦截器名称（也就是增强类名称，Spring Bean 的 id） -->
            <list>
                <value>greetingAroundAdvice</value>
            </list>
        </property>
    </bean>
```

其实使用 ProxyFactoryBean 就可以取代前面的 ProxyFactory，其实它们俩就一回事儿。

```java
@Component
public class GreetingImpl implements Greeting {
    ...
}
@Component
public class GreetingAroundAdvice implements MethodInterceptor {
    ...
}
public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("aop/demo/spring.xml"); // 获取 Spring Context
        Greeting greeting = (Greeting) context.getBean("greetingProxy");                        // 从 Context 中根据 id 获取 Bean 对象（其实就是一个代理）
        greeting.sayHello("Jack");                                                              // 调用代理的方法
    }
```

代码量确实少了，我们将配置性的代码放入配置文件，这样也有助于后期维护。更重要的是，代码只关注于业务逻辑，而将配置放入文件中。这是一条最佳实践！

除了上面提到的那三类增强以外，其实还有两类增强也需要了解一下.

#### Spring AOP：抛出增强

程序报错，抛出异常了，一般做法就是打印到控制台或者日志文件中，这样很多地方都得去处理，有没有一劳永逸的方法呢？那就是**Throws Advice（抛出增强）**

```java
@Component
public class GreetingImpl implements Greeting {
    @Override
    public void sayHello(String name) {
        System.out.println("Hello! " + name);

        throw new RuntimeException("Error"); // 故意抛出一个异常，看看异常信息能否被拦截到
    }
}
@Component
public class GreetingThrowAdvice implements ThrowsAdvice {
    public void afterThrowing(Method method, Object[] args, Object target, Exception e) {
        System.out.println("---------- Throw Exception ----------");
        System.out.println("Target Class: " + target.getClass().getName());
        System.out.println("Method Name: " + method.getName());
        System.out.println("Exception Message: " + e.getMessage());
        System.out.println("-------------------------------------");
    }
}
```

抛出增强类需要实现 org.springframework.aop.ThrowsAdvice 接口，在接口方法中可获取方法、参数、目标对象、异常对象等信息。我们可以把这些信息统一写入到日志中，当然也可以持久化到数据库中。

这个功能确实太棒了！但还有一个更厉害的增强。如果某个类实现了 A 接口，但没有实现 B 接口，那么该类可以调用 B 接口的方法吗？如果您没有看到下面的内容，一定不敢相信原来这是可行的！

#### Spring AOP：引入增强

以上提到的都是对方法的增强，那能否对类进行增强呢？用 AOP 的行话来讲，对方法的增强叫做**Weaving（织入）**，而对类的增强叫做**Introduction（引入）**，Introduction Advice（引入增强）就是对类功能的增强。

定义一个新接口Apology：

```java
public interface Apology{
  void saySorry(String name);
}
```

但我不想在代码中让 GreetingImpl 直接去实现这个接口，我想在程序运行的时候动态地实现它。因为假如我实现了这个接口，那么我就一定要改写 GreetingImpl 这个类，关键是我不想改它，于是，我需要借助 Spring 的引入增强。

```java
@Component
public class GreetingIntroAdvice extends DelegatingIntroductionInterceptor implements Apology {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        return super.invoke(invocation);
    }

    @Override
    public void saySorry(String name) {
        System.out.println("Sorry! " + name);
    }
}
```

以上定义了一个引入增强类，扩展了 org.springframework.aop.support.DelegatingIntroductionInterceptor 类，同时也实现了新定义的 Apology 接口。在类中首先覆盖了父类的 invoke() 方法，然后实现了 Apology 接口的方法。我就是想用这个增强类去丰富 GreetingImpl 类的功能，那么这个 GreetingImpl 类无需直接实现 Apology 接口，就可以在程序运行的时候调用 Apology 接口的方法了。这简直是太神奇的！

```xml
<context:component-scan base-package="aop.demo"/>

    <bean id="greetingProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="interfaces" value="aop.demo.Apology"/>          <!-- 需要动态实现的接口 -->
        <property name="target" ref="greetingImpl"/>                    <!-- 目标类 -->
        <property name="interceptorNames" value="greetingIntroAdvice"/> <!-- 引入增强 -->
        <property name="proxyTargetClass" value="true"/>                <!-- 代理目标类（默认为 false，代理接口） -->
    </bean>
```

需要注意 proxyTargetClass 属性，它表明是否代理目标类，默认为 false，也就是代理接口了，此时 Spring 就用 JDK 动态代理。如果为 true，那么 Spring 就用 CGLib 动态代理。

当您看完下面的客户端代码，一定会完全明白以上的这一切：

```java
public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("aop/demo/spring.xml");
        GreetingImpl greetingImpl = (GreetingImpl) context.getBean("greetingProxy"); // 注意：转型为目标类，而并非它的 Greeting 接口
        greetingImpl.sayHello("Jack");

        Apology apology = (Apology) greetingImpl; // 将目标类强制向上转型为 Apology 接口（这是引入增强给我们带来的特性，也就是“接口动态实现”功能）
        apology.saySorry("Jack");
    }
```

没想到 saySorry() 方法原来是可以被 greetingImpl 对象来直接调用的，只需将其强制转换为该接口即可。

#### Spring AOP：切面

之前谈到的 AOP 框架其实可以将它理解为一个拦截器框架，但这个拦截器似乎非常武断。比如说，如果它拦截了一个类，那么它就拦截了这个**类中所有的方法**。类似地，当我们在使用动态代理的时候，其实也遇到了这个问题。需要在代码中对所拦截的方法名加以判断，才能过滤出我们需要拦截的方法，想想这种做法确实不太优雅。在大量的真实项目中，似乎我们只需要拦截特定的方法就行了，没必要拦截所有的方法。于是，老罗同志借助了 AOP 的一个很重要的工具，**Advisor（切面）**，来解决这个问题。它也是 AOP 中的核心！是我们关注的重点！

也就是说，我们可以通过切面，将增强类与拦截匹配条件组合在一起，然后将这个切面配置到 ProxyFactory 中，从而生成代理。这里提到这个“拦截匹配条件”在 AOP 中就叫做 **Pointcut（切点）**，其实说白了就是一个基于表达式的拦截条件罢了。

**归纳一下，Advisor（切面）封装了 Advice（增强）与 Pointcut（切点 ）**

```java
@Component
public class GreetingImpl implements Greeting {
    @Override
    public void sayHello(String name) {
        System.out.println("Hello! " + name);
    }
    public void goodMorning(String name) {
        System.out.println("Good Morning! " + name);
    }
    public void goodNight(String name) {
        System.out.println("Good Night! " + name);
    }
}
```

GreetingImpl 类中故意增加了两个方法，都以“good”开头。下面要做的就是拦截这两个新增的方法，而对 sayHello() 方法不作拦截。在 Spring AOP 中，老罗已经给我们提供了许多切面类了，这些切面类我个人感觉最好用的就是基于正则表达式的切面类。

```xml
<context:component-scan base-package="aop.demo"/>

    <!-- 配置一个切面 -->
    <bean id="greetingAdvisor" class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
        <property name="advice" ref="greetingAroundAdvice"/>            <!-- 增强 -->
        <property name="pattern" value="aop.demo.GreetingImpl.good.*"/> <!-- 切点（正则表达式） -->
    </bean>

    <!-- 配置一个代理 -->
    <bean id="greetingProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="target" ref="greetingImpl"/>                <!-- 目标类 -->
        <property name="interceptorNames" value="greetingAdvisor"/> <!-- 切面 -->
        <property name="proxyTargetClass" value="true"/>            <!-- 代理目标类 -->
    </bean>
```

*注意以上代理对象的配置中的 interceptorNames，它不再是一个增强，而是一个切面，因为已经将增强封装到该切面中了。此外，切面还定义了一个切点（正则表达式），其目的是为了只将满足切点匹配条件的方法进行拦截。*

除了 RegexpMethodPointcutAdvisor 以外，在 Spring AOP 中还提供了几个切面类，比如：

- DefaultPointcutAdvisor：默认切面（可扩展它来自定义切面）
- NameMatchMethodPointcutAdvisor：根据方法名称进行匹配的切面
- StaticMethodMatcherPointcutAdvisor：用于匹配静态方法的切面

总的来说，让用户去配置一个或少数几个代理，似乎还可以接受，但随着项目的扩大，代理配置就会越来越多，配置的重复劳动就多了，麻烦不说，还很容易出错。能否让 Spring 框架为我们自动生成代理呢？

#### Spring AOP：自动代理（扫描Bean名称）

Spring AOP 提供了一个可根据 Bean 名称来自动生成代理的工具，它就是 BeanNameAutoProxyCreator。是这样配置的：

```xml
<bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
        <property name="beanNames" value="*Impl"/>                       <!-- 只为后缀是“Impl”的 Bean 生成代理 -->
        <property name="interceptorNames" value="greetingAroundAdvice"/> <!-- 增强 -->
        <property name="optimize" value="true"/>                         <!-- 是否对代理生成策略进行优化 -->
    </bean>
```

*以上使用 BeanNameAutoProxyCreator 只为后缀为“Impl”的 Bean 生成代理。需要注意的是，这个地方我们不能定义代理接口，也就是 interfaces 属性，因为我们根本就不知道这些 Bean 到底实现了多少接口。此时不能代理接口，而只能代理类。所以这里提供了一个新的配置项，它就是 optimize。若为 true 时，则可对代理生成策略进行优化（默认是 false 的）。也就是说，如果该类有接口，就代理接口（使用 JDK 动态代理）；如果没有接口，就代理类（使用 CGLib 动态代理）。而并非像之前使用的 proxyTargetClass 属性那样，强制代理类，而不考虑代理接口的方式。可见 Spring AOP 确实为我们提供了很多很好地服务！*

根据多年来实际项目经验得知：CGLib 创建代理的速度比较慢，但创建代理后运行的速度却非常快，而 JDK 动态代理正好相反。如果在运行的时候不断地用 CGLib 去创建代理，系统的性能会大打折扣，所以建议一般在系统初始化的时候用 CGLib 去创建代理，并放入 Spring 的 ApplicationContext 中以备后用。

以上这个例子只能匹配目标类，而不能进一步匹配其中指定的方法，要匹配方法，就要考虑使用切面与切点了。Spring AOP 基于切面也提供了一个自动代理生成器：DefaultAdvisorAutoProxyCreator。

#### Spring AOP：自动代理（扫描切面配置）

为了匹配目标类中的指定方法，我们仍然需要在 Spring 中配置切面与切点：

```xml
<bean id="greetingAdvisor" class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
        <property name="pattern" value="aop.demo.GreetingImpl.good.*"/>
        <property name="advice" ref="greetingAroundAdvice"/>
    </bean>

    <bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator">
        <property name="optimize" value="true"/>
    </bean>
```

*这里无需再配置代理了，因为代理将会由 DefaultAdvisorAutoProxyCreator 自动生成。也就是说，这个类可以扫描所有的切面类，并为其自动生成代理。*

在 Spring 配置文件中，仍然会存在大量的切面配置。然而在有很多情况下 Spring AOP 所提供的切面类真的不太够用了，比如：想拦截指定注解的方法，我们就必须扩展 DefaultPointcutAdvisor 类，自定义一个切面类，然后在 Spring 配置文件中进行切面配置。不做不知道，做了您就知道相当麻烦了。神一样的老罗总算认识到了这一点，接受了网友们的建议，集成了 AspectJ，同时也保留了以上提到的切面与代理配置方式

#### Spring + AspectJ（基于注解：通过AspectJ execution表达式拦截方法）

```java
@Aspect
@Component
public class GreetingAspect {
    @Around("execution(* aop.demo.GreetingImpl.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        before();
        Object result = pjp.proceed();
        after();
        return result;
    }
    private void before() {
        System.out.println("Before");
    }
    private void after() {
        System.out.println("After");
    }
}
```

*注意：类上面标注的 @Aspect 注解，这表明该类是一个 Aspect（其实就是 Advisor）。该类无需实现任何的接口，只需定义一个方法（方法叫什么名字都无所谓），只需在方法上标注 @Around 注解，在注解中使用了 AspectJ 切点表达式。方法的参数中包括一个 ProceedingJoinPoint 对象，它在 AOP 中称为Joinpoint（连接点），可以通过该对象获取方法的任何信息，例如：方法名、参数等。*

下面重点来分析一下这个切点表达式：

execution(* aop.demo.GreetingImpl.*(..))

- execution()：表示拦截方法，括号中可定义需要匹配的规则。
- 第一个“*”：表示方法的返回值是任意的。
- 第二个“*”：表示匹配该类中所有的方法。
- (..)：表示方法的参数是任意的。

是不是比正则表达式的可读性更强呢？如果想匹配指定的方法，只需将第二个“*”改为指定的方法名称即可。配置如下：

```xml
<context:component-scan base-package="aop.demo"/>

<aop:aspectj-autoproxy proxy-target-class="true"/>
```

两行配置就行了，不需要配置大量的代理，更不需要配置大量的切面，真是太棒了！需要注意的是 proxy-target-class="true" 属性，它的默认值是 false，默认只能代理接口（使用 JDK 动态代理），当为 true 时，才能代理目标类（使用 CGLib 动态代理）。

#### Spring + AspectJ（基于注解：通过AspectJ@annotation表达式拦截方法） 

Spring 与 AspectJ 结合的威力远远不止这些，我们来点时尚的吧，拦截指定注解的方法怎么样？我们首先需要来自定义一个注解：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Tag {
}
```

只需将前面的 Aspect 类的切点表达式稍作改动：

```java
@Aspect
@Component
public class GreetingAspect {
    @Around("@annotation(aop.demo.Tag)")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        ...
    }
    ...
}
```

这次使用了 @annotation() 表达式，只需在括号内定义需要拦截的注解名称即可。

直接将 @Tag 注解定义在您想要拦截的方法上，就这么简单：

```java
@Component
public class GreetingImpl implements Greeting {
    @Tag
    @Override
    public void sayHello(String name) {
        System.out.println("Hello! " + name);
    }
}
```

除了 @Around 注解外，其实还有几个相关的注解，稍微归纳一下吧：

- @Before：前置增强
- @After：后置增强
- @Around：环绕增强
- @AfterThrowing：抛出增强
- @DeclareParents：引入增强

此外还有一个 @AfterReturning（返回后增强），也可理解为 Finally 增强，相当于 finally 语句，它是在方法结束后执行的，也就说说，它比 @After 还要晚一些。最后一个 @DeclareParents 竟然就是引入增强！为什么不叫做 @Introduction 呢？我也不知道为什么，但它干的活就是引入增强。

#### Spring + AspectJ（引入增强）

为了实现基于 AspectJ 的引入增强，我们同样需要定义一个 Aspect 类：

```java
@Aspect
@Component
public class GreetingAspect {

    @DeclareParents(value = "aop.demo.GreetingImpl", defaultImpl = ApologyImpl.class)
    private Apology apology;
}
```

只需要在 Aspect 类中定义一个需要引入增强的接口，它也就是运行时需要动态实现的接口。在这个接口上标注了 @DeclareParents 注解，该注解有两个属性：

- value：目标类
- defaultImpl：引入接口的默认实现类

我们只需要对引入的接口提供一个默认实现类即可完成引入增强：

```java
public class ApologyImpl implements Apology {
    @Override
    public void saySorry(String name) {
        System.out.println("Sorry! " + name);
    }
}
```

以上这个实现会在运行时自动增强到 GreetingImpl 类中，也就是说，无需修改 GreetingImpl 类的代码，让它去实现 Apology 接口，我们单独为该接口提供一个实现类（ApologyImpl），来做 GreetingImpl 想做的事情。

```java
public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("aop/demo/spring.xml");
        Greeting greeting = (Greeting) context.getBean("greetingImpl");
        greeting.sayHello("Jack");

        Apology apology = (Apology) greeting; // 强制转型为 Apology 接口
        apology.saySorry("Jack");
    }
```

*从 Spring ApplicationContext 中获取 greetingImpl 对象（其实是个代理对象），可转型为自己静态实现的接口 Greeting，也可转型为自己动态实现的接口 Apology，切换起来非常方便。*

使用 AspectJ 的引入增强比原来的 Spring AOP 的引入增强更加方便了，而且还可面向接口编程（以前只能面向实现类），这也算一个非常巨大的突破。

#### Spring + AspectJ（基于配置）

除了使用 @Aspect 注解来定义切面类以外，Spring AOP 也提供了基于配置的方式来定义切面类：

```xml
<beans ...">
    <bean id="greetingImpl" class="aop.demo.GreetingImpl"/>
    <bean id="greetingAspect" class="aop.demo.GreetingAspect"/>

    <aop:config>
        <aop:aspect ref="greetingAspect">
            <aop:around method="around" pointcut="execution(* aop.demo.GreetingImpl.*(..))"/>
        </aop:aspect>
    </aop:config>
</beans>
```

*使用 <aop:config> 元素来进行 AOP 配置，在其子元素中配置切面，包括增强类型、目标方法、切点等信息。*

#### 思维导图

![](http://static.oschina.net/uploads/space/2013/0915/111332_5YL3_223750.png)

再来一张表格，总结一下各类增强类型所对应的解决方案：

| 增强类型                     | 基于 AOP 接口                         | 基于 @Aspect      | 基于 <aop:config>       |
| ------------------------ | --------------------------------- | --------------- | --------------------- |
| Before Advice（前置增强）      | MethodBeforeAdvice                | @Before         | <aop:before>          |
| AfterAdvice（后置增强）        | AfterReturningAdvice              | @After          | <aop:after>           |
| AroundAdvice（环绕增强）       | MethodInterceptor                 | @Around         | <aop:around>          |
| ThrowsAdvice（抛出增强        | ThrowsAdvice                      | @AfterThrowing  | <aop:after-throwing>  |
| IntroductionAdvice（引入增强） | DelegatingIntroductionInterceptor | @DeclareParents | <aop:declare-parents> |

最后给一张 UML 类图描述一下 Spring AOP 的整体架构：

![](http://static.oschina.net/uploads/space/2013/0914/235319_GQUH_223750.png)





参考：

《Spring技术内幕：深入分析Spring架构与设计原理》

《Spring源码深度解析》

[Spring 框架的设计理念与设计模式分析](http://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/index.html)

[关于Spring的69个面试问答——终极列表](http://www.importnew.com/11657.html)

[ AOP 那点事儿](http://my.oschina.net/huangyong/blog/161338)