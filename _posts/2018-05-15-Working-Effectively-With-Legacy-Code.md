---
layout:     post
title:      "Working Effectively With Legacy Code"
subtitle:   "修改代码的艺术"
date:       2018-05-15 23:30:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - 软件工程
    - 软件开发之道
---

## 什么是遗留代码

关于遗留代码，不同的人有不同的理解：1）从其他人那得到的代码；2）无法理解、难以修改的代码。

而作者的定义：**没有编写相应测试的代码**。

`没有编写测试的代码是糟糕的代码。不管我们有多细心去编写它们，不管它们有多漂亮、面向对象或封装良好，只要没有编写测试，我们实际上就不知道修改后的代码是变得更好还是更糟了。反之，有了测试，我们就能迅速、可验证地修改代码的行为。`

## 修改代码的概念和原则

#### 我们要积极地去修改代码

修改软件的四个主要起因：

1. 添加新特性
2. 修正Bug
3. 改善设计，在不改变软件行为的前提下优化代码称为重构。
4. 优化资源使用，通常是时间和内存

|          | 添加特性 | 修正bug | 重构 | 优化 |
| -------- | -------- | ------- | ---- | ---- |
| 结构     | 改变     | 改变    | 改变 | ——   |
| 新功能   | 改变     | ——      | ——   | ——   |
| 功能     | ——       | 改变    | ——   | ——   |
| 资源使用 | ——       | ——      | ——   | 改变 |

从上表看出，特性添加和修改bug非常相似，重构和优化非常相似，但是他们的共同点都是：要保留更多的既有行为不变。程序的依赖是复杂的，并不是避免修改代码就可以保持其他行为不变，这是软件开发中最具挑战性的任务之一。

避免修改是通常被采用的一种降低风险的方式，但它会导致很多后果：

- 当我们避免创建新类和方法时，既有方法和类会变得越来越庞大，越来越难以理解；
- 会造成对代码的生疏；
- 会带来恐惧心理；

所以我们要努力地去修改代码。

#### 我们要用测试来保护修改

回归测试存在的问题：错误定位慢；执行时间过长；覆盖不好评估。

单元测试的品质：运行快；帮助快速定位问题。

`一个需要耗时十分之一秒才能执行完的单元测试就已经算一个慢的单元测试了`

单元测试运行的快，运行的不快的不是单元测试，例如：跟数据库交互；进行了网络间通信；调用了文件系统；需要对环境作特定的准备才能运行。将它们和真正的单元测试区分开还是很有必要的。

**遗留代码的困境**

`我们在修改代码时，应当有测试在周围“护着”。而为了将这些测试安置得当，我们往往又得先去修改代码`

诀窍就是要非常保守地进行最初的重构。

算法步骤：

1. 确定改动点；16，17
2. 找出测试点；11，12
3. 解依赖；9，10，22，23，25，7
4. 编写测试；13
5. 修改、重构；20，21，22

Fake Object与Mock Object的区别：Mock Object是一种更高级的Fake Object，可以在内部进行断言检查等功能.

Mock, Junit

## 修改代码的技术

#### 1. 时间紧迫，但必须修改

原则就是不在原有逻辑上修改，这样新写的逻辑可以被充分测试，而且实现速度快。

- 新方法：优点，新旧代码被清楚地隔离开，不需要将旧代码放在测试之下，开发速度快；缺点，效果等于暂时放弃了原有方法和类，没有改善和测试旧代码。
- 新类：两种情况下需要使用新生类，第一是所要进行的修改迫使你为某个类添加一个全新的职责；第二是无法将现有的类放在测试中。优点，它让你进行侵入性较强的修改时有更大的自信去继续开展自己的工作；缺点，让系统概念进一步复杂化。
- 外覆方法：方式一，创建一个与原方法同名的新方法，并在新方法中调用更名后的原方法；方式二，增加一个没有调用的新方法，在方法中调用旧方法。优点是显式地将新功能独立于既有功能；缺点是可能导致糟糕的命名。
- 外覆类，方式一，装饰模式，继承原有类的接口；方式二，新建另外一个类和新的接口，将旧类放在其中。



#### 2. 添加特性

上面提到的快速修改技术有什么缺点呢？

- 没有修改既有代码，不会在短期得到改善
- 新的代码重复

怎样解决这些问题呢？

**测试驱动开发**：

1. 将想要修改的类置于测试之下
2. 编写一个失败测试用例
3. 让它通过编译
4. 让测试通过（不改动代码）
5. 消除重复
6. 重复上述步骤

**差异式编程**：一开始借助于类的继承，不改动原有类的前提下引入新的特性，然后加入测试，当发现继承不再合适的时候，可以改成其他设计。注意不要违反Liskov置换原则子类对象应当能够替换代码中的父类对象。

规范化的继承体系：任何类都不会包含同一个方法的多个实现，即**不要重写其父类中的具体方法**！！！



#### 3. 无法将类放入测试用具中

1） 恼人的参数

- 接口提取（可以Mock interface对象）。
- 传Null值。注意：不到万不得已，千万不要在产品代码中传递null值。可以考虑使用空对象模式。

2） 隐藏依赖

```java
class A {
    DAO dao = DAO.getInstance();
    Connection con;
    public A(){}
    // 或者
    public A(){
        con = new Connection();
    }
}
```

有些类是具有欺骗性的，这个时候，我们就需要static mock DAO对象，来对A进行测试。

```java
class A{
    DAO dao;
    public A(DAO dao){};
    
    public A() {
        dao = DAO.getInstance();
        this(dao);
    }
}
```

通过这种**参数化构造函数技术**，可以将Mock的对象传进要测试的类。同时，如果需要保留之前的接口，可以新写构造函数。

3）构造块

上面提到的参数化构造函数技术应该是最先想到的一项技术，但是如果一个构造函数创造了大量对象，或者访问了大量的全局变量，则这种技术会导致很长的参数列表。更糟糕的是，可能会在构造函数中先创建一些对象，然后使用它们创建另一些对象，存在构造顺序的问题。

可以采用`替换实例变量`的手法：

```java
void supersedeCursor(FocusWidget newCursor) {
    cursor = newCursor;
}
```

除非万不得已，不要使用替换实例变量的方法，可能会带来一些资源管理方面的问题。



4）恼人的全局变量

对于单例模式，我们怎样去Mock？

方法一：添加set方法替换变量

```java
public static void setInstance(A a) {
    instance = a;
}
```

但是，往往构造函数是私有的，这个时候就不可以创建A对象了。这个时候，我们要重新思考下，为什么要使用单例模式，也许可以改为public或者protected，并不会有太大问题。

方法二：接口提取

```java
public class A implements IA {
    private static IA instance = null;
    private A(){}
    public static IA getInstance(){}
}
```

这种问题的缺点是，对于已有代码，需要改变他们持有的A对象的引用。

注意：如果你发现一个全局变量真的被到处使用，意味着代码没有进行任何层次化设计。



#### 4. 无法在测试用具中运行方法

1）私有方法

方案一：判断能否通过一个公用的方法来进行测试，这样可以避免访问私有方法以及重构工作，好处是可以按照该方法实际使用的方式来测试。

方案二：如果要直接测试这个私有方法，答案就是将它设置为public。如果不方便设为public，则说明类职责太多，需要重构了。

Java反射方法来访问私有变量：

通过这种方式在测试中访问私有变量是不建议的，是一种欺骗的手段，会蒙蔽团队的眼镜，使他们看不到代码变得有多糟糕。所以，不推荐使用。

2）无法探知的副作用

命令和查询分离：一个方法要么是一个命令，要么是一个查询，但不能两者都不是。命令方式指那些会改变对象状态但并不返回值得方法。而查询方法则是指那些有返回值但不改变对象状态的方法。

重构之前，DetailFrame是一个UI组件，不好测试：

```java
public void actionPerformed(ActionEvent event) {
    String source = (String)event.getActionCommand();
    if (source.equals("xxx")) {
        DetailFrame detailDisplay = new DetailFrame();
        detailDisplay.setDescription("asdf");
        detailDisplay.show();
        String accountDesc = detailDisplay.getAccountSymbol();
        display.setText(accountDesc + ":")
    }
}
```

将这个大方法进行提取，将命令和查询分开。同时，将detailDisplay提取成类变量。这样调用UI的地方就可以Mock了。

```java
private DetailFrame detailDisplay;
public void performCommand(String source) {
     if (source.equals("xxx")) {
        setDesciption("asdf");
        String accountDesc = getAccountSymbol();
        setDisplayText(accountDesc + ":");
    }
}
void setDescription(String description) {
    detailDisplay = new DetailFrame();
    display.setDescription(description);
    detailDisplay.show();
}
String getAccountSymbol() {
    return detailDisplay.getAccountSymbol();
}
void setDisplayText(String desc) {
    display.setText(desc + ":")
}
```

#### 5. 修改时应该怎样写测试

特征测试：用于行为保持的测试。

1. 在测试用具中使用目标代码块
2. 编写一个你知道会失败的断言
3. 从断言的失败中得知代码的行为
4. 修改你的测试，让它预期目标代码的实际行为
5. 重复上述步骤

这种测试不再是一种准则，而是描述了系统各部分的实际行为，并不是为了寻找bug，而是设置一个机制便于以后寻找bug，例如对代码的修改导致行为的改变。

如果是新的代码，则根据测试驱动的思想，应该先预期测试结果，然后修改代码让测试通过。

#### 6. 处理大类

对于庞大的类，一个关键的补救手段就是重构，将这个类分解为一小组类。

指导原则：单一职责原则（SRP），每个类应该仅承担一个职责，它在系统中的意图应当是单一的，且修改它的原因应该只有一个。

探索式方法：

1. 方法分组，寻找相似的方法名；
2. 观察隐藏方法，**大量私有或受保护的方法往往意味着一个类内部有赢一个类急迫地想要独立出来**；
3. 寻找可以更改的决定，比如有什么地方（与数据库交互，API调用，等）采用了硬编码等
4. 寻找成员变量和方法之间的关系；
5. 寻找主要职责，仅用一句话来描述该类的职责；接口隔离原则，试着为这个类建立一套接口
6. 当所有方法都行不通时，做一些草稿式重构
7. 关注当前工作。也许可以只是将与正在进行的修改独立出新的职责。



#### 7. 需要修改大量相同的代码

消除重复代码，采用继承或者组合的方式。

开放/封闭原则：代码对于扩展应该是开放的，而对于修改则应该是封闭的。



## 解依赖技术

这些技术都是重构技术，保持了代码的行为，将类进行解依赖而使它能够被置于测试之下，与业界的重构技术不同的是，这些技术是在没有测试的情况下使用的。

#### 参数适配

当我们发现测试一个方法无法创建需要的参数时，首先想到的是`接口提取`方法，将传入的参数使用接口替代。但是，有些时候无法对一个参数的类型使用接口提取，这时可以使用`参数适配`的手法。

例子：

```Java
public void populate(HttpServletRequest request) {
    String[] values = request.getParameterValues(pageStateName);
    ...
}
```

上面方法中，我们很难创建一个HTTPServletRequest实例，并且这个类有大概23个方法，很难使用接口提取手法创建一个窄接口。通过参数适配方法，可以将这个类包裹起来，完全解除依赖。

```java
public void populate(ParameterSource source) {
    String[] values = source.getParameterForName(pageStateName);
    ...
}
class FakeParameterSource implements ParameterSource {
    public String value;
    
    public String getParameterForName(String name) {
        return value;
    }
}
class ServletParamterSource implements ParameterSource {
    private HttpServletRequest request;
    public ServletParameterSource(HttpServletRequest request) {
        this.request = request;
    }
    String getParameterValue(String name) {
        request.getParameterValues(name);
    }
}
```

多了一层抽象，可以使业务代码和API调用解耦。但是，这种参数适配的方法修改了方法的签名，需要格外小心。所以一定要将测试补齐，一旦测试到位，便可以更加侵入性地改动代码了。

步骤：

- 创建一个新的简单明了的接口。注意不要导致大规模的代码修改。
- 为新接口创建一个用于产品的代码实现
- 为新接口创建一个用于测试的Mock实现
- 编写一个简单的测试用例，并将Mock对象传入
- 对该方法修改以使其能使用新的参数
- 运行测试来确保使用Mock对象测试方法



#### 分解出方法对象

分解出方法对象的手法适用于类很难实例化，并且要测试的方法规模大，或该方法使用了实例数据或其他方法的情况，其核心理念是将一个长方法移至一个新类中，后者成为方法对象，他们只含有单个方法的代码，这样我们就可以对这个新的方法对象进行测试了。

```Java
class GDIBrush {
    public void draw(List<Point> rederingRoots, ColorMatrix colors, List<Point> selection) {
        ...
        drawPoint(...);
    }
    private void drawPoint(int x, int y, Color color) {}
}
```

假设我们测试中无法实例化GDIBrush，draw方法又是一个长方法，我们可以创建一个新类Renderer，为它编写一个public的构造函数，签名和draw方法一致，为每个参数创建一个成员变量并初始化。接着添加一个draw方法，从原有的方法中复制过来，并且解决编译错误。

```Java
class Renderer {
	private GDIBrush brush;
	private List<Point> rederingRoots,
	private ColorMatrix colors,
	private List<Point> selection,
    public Renderer(GDIBrush brush, List<Point> rederingRoots, ColorMatrix colors, List<Point> selection) {//初始化}
    public void draw() {//复制过来}
}
```

如果有对GDIBrush的变量或者方法，可以在GDIBrush中添加相应的public方法获取。现在问题解决了吗？显然没有，我们想要创建Renderer对象，仍然需要实例化GDIBrush并且传入，这时候就可以利用接口提取来完全解除Renderer对GDIBrush的依赖。

```java
public interface PointRenderer {
    void drawPoint(int x, int y, Color color);
}
public class GDIBrush implements PointRenderer {
    void drawPoint(int x, int y, Color color) {
        ...
    }
}
class Renderer {
	private PointRenderer pointRenderer;
	private List<Point> rederingRoots,
	private ColorMatrix colors,
	private List<Point> selection,
    public Renderer(PointRenderer pointRenderer, List<Point> rederingRoots, ColorMatrix colors, List<Point> selection) {//初始化}
    public void draw() {//复制过来}
}
```

这只是开始，如果GDIBrush一直不能实例化并放入测试用具，我们就要一步步进行解依赖，一旦有测试覆盖，就可以大胆重构了。



#### 封装全局引用

1. 找出有待封装的全局变量、函数
2. 为它们创建一个类
3. 将全局变量复制过来并初始化
4. 将原来的变量注释掉
5. 声明新类的一个全局对象
6. 依靠编译器找出所有依赖的地方
7. 加上刚声明的前缀
8. 在想要使用Mock对象的地方，利用参数化构造函数、以获取方法替换全局引用等。

#### 暴露静态方法

如果一个方法，不使用实例变量或者其他方法，就可以将它设置成静态的，这样就可以添加测试了。

#### 提取并重写调用

将对某个方法的调用提取到一个新方法中，达到解耦的目的，并用一个测试子类来覆盖。下面的例子中，就可以对formStyles进行任意的Mock了。

```java
protected void rebindStyles() {
    styles = formStyles(template, id);
}
protedted List formStyles(StyleTemplate template, int id) {
    return StyleMaster.formStyles(template, id);
}
```

#### 提取并重写工厂方法

构造函数中固定了的初始化工作可能会给测试带来很大的麻烦。

```Java
public class WorkflowEngine{
    public WorkflowEngine() {
        Reader reader = new ModelReader();
        Persister persister = new XMLStore();
        this.tm = new TransactionManager(reader, persister);
    }
}
```

提取并重写工厂方法（注意，该方法不适用于C++，因为C++不允许在构造函数中的虚函数调用被决议到派生类中）：

```java
public class WorkflowEngine{
    public WorkflowEngine() {
        this.tm = makeTransactionManager();
    }
    
    protected TransactionManager makeTransactionManager() {
        Reader reader = new ModelReader();
        Persister persister = new XMLStore();
        return new TransactionManager(reader, persister);
    }
}
```

有了工厂方法后，便可以将它子类化并重写了，我们可以在重写的工厂方法中返回我们想要的东西。

#### 提取并重写获取方法

上面提到的重写工厂方法并不适用于C++，而如果你只是在构造函数中创建新对象，并不用新对象做其他事情的话，可以采用重写获取方法的手法，为你想要替换的成员变量引入一个获取方法。

```Java
public class WorkflowEngine{
    public WorkflowEngine() {
        Reader reader = new ModelReader();
        Persister persister = new XMLStore();
        this.tm = new TransactionManager(reader, persister);
    }
    
    public TransactionManager getTransactionManager() {
        if (tm == null) {
            Reader reader = new ModelReader();
        	Persister persister = new XMLStore();
        	tm = new TransactionManager(reader, persister);
        }
    }
}
```

#### 实现提取

接口提取经常会遇到一个问题：命名。好的名字有助于人们理解系统，并令系统更易对付。如果一个类的名字恰恰适合用来做你的接口名，而且你手头又没有自动重构的工具的话，可以使用实现提取手法来获得所需的分离。

步骤：

1. 将目标类的声明复制一份，起一个新名字；
2. 将目标类变成一个接口，通过删除所有非公有方法和数据成员实现；
3. 将所有剩下来的公有方法设为抽象方法；
4. 删除不必要的import
5. 让你复制的新类实现该接口，编译并确保所有方法都实现了，找出创建原来类的地方，修改为创建新的产品类对象。

#### 接口提取

接口提取是最安全的解依赖技术之一。接口提取时并不一定要提取类上所有公有方法，你可以依靠编译器来发现那些方法需要。

步骤：

1. 创建一个新接口，起个好名字
2. 令你提取接口的目标类实现该接口
3. 将所有使用的地方从原类改成接口
4. 编译系统。

#### 引入实例委托

如果你的项目中有静态方法，则很可能它们并不会给你带来麻烦，除非其中包含了一些你没法或者不想在测试的时候依赖的东西（术语叫做 静态黏着）。

方案之一是往目标类上引入委托实例方法，将静态方法调用改为实例方法调用。

#### 特性提升与依赖下推

特性提升：将一簇与问题无关的方法提升到一个抽象基类中，然后再对这个抽象基类进行子类化，并在测试中创建这个子类的实例；

依赖下推：把目标类设置成抽象类，然后创建一个它的子类，后者便是你的新的产品类了，将所有问题依赖都下推到这个子类。

#### 以获取方法替换全局引用

例如：我们使用单例模式的时候，可以在方法里写一个get方法获取这个单例。注意方法的修饰符：protected。这样就可以通过子类化并重写get方法的技术类Mock单例。

```java
public class RegisterSale{
    public void addItem() {
        Item newItem = getInventory().itemForBarcode();
        items.add(newItem);
    }
    protected Inventory getInventory() {
        return Inventory.getInstance();
    }
}
```



参考：

《Working Effectively With Legacy Code》

