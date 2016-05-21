---
layout:     post
title:      "改善Java程序的151个建议读书笔记（上）"
subtitle:   "1、Java开发中通用的方法和准则；2、基本类型；3、类、对象及方法；4、字符串"
date:       2016-05-18 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Java
---

> 放弃了Effective Java，太过于抽象，转战这本书，最起码知识点更精细，读起来也更快。这里将书中的内容整理下来，为了以后参考，长期翻阅~

## 第一章 Java开发中通用的方法和准则

#### 建议1：不要在常量和变量中出现易混淆的字母

Java编码规范：包名全小写、类名首字母全大写、常量全大写并用下划线分隔、变量采用驼峰法命名等

```java
long i = 1l; //容易和数字混淆
long i = 1L; // 正确
```

**注意：**字母“l”作为长整型标志时务必大写

#### 建议2：莫让常量蜕变成变量

```java
interface Const{
  // 这还是常量吗？
  public static final int RAND_CONST = new Random().nextInt();
}
```

**注意：**务必让常量的值在运行期保持不变

#### 建议3：三元操作符的类型务必一致

```java
int i = 80;
String s = String.valueOf(i < 100 ? 90 : 100);
String s1 = String.valueOf(i < 100 ? 90 : 100.0);
System.out.pringln(" 两者是否相等 ："+s.equals(s1));
```

三元操作符类型的转换规则：

- 若两个操作符不可转换，则不做转换，返回值为Object类型
- 若两个操作符是明确类型的表达式，则按照正常的二进制数字来转换，int类型转换为long类型，long转换为float类型等
- 若两个操作数中有一个是数字S，另一个是表达式，其类型为T，那么若S在T的范围内，则转换为T类型；若S超出了T类型的范围，则T转换为S类型
- 过两个操作数都是直接量数字，则返回值类型为**范围较大者**

#### 建议4：避免带有变长参数的方法重载

```java
public void calPrice(int price, int discount){}
public void calPrice(int price,int... discounts){}
```

一个方法不能定义多个变长参数。

变长参数方法重载，调用calPrice(49900, 75)时两者都符合，实际结果是调用了第一个方法（编译器偷懒），但是要慎用变长参数的方法重载

#### 建议5：别让null值和空值威胁到变长方法

```java
public void methodA(String str, Integer... is){}
public void methodA(String str, String... strs){}
public static void main(String[] args){
  client.methodA("China");
  client.methodA("China", null);
}
```

两个methodA都进行重载，两处调用都编译不通过，方法模糊不清，编译器不知道调用哪一个方法。

methodA("China")方法，两个方法都符合形参，变长参数可以是0~N个

methodA("China", null)方法，null没有类型，不知道调用哪一个

```java
// 明确实参类型
String[] strs = null;
clinet.methodA("China", strs);
```

#### 建议6：覆写变长方法也循规蹈矩

子类覆写父类方法，可以修正Bug和扩展业务功能，符合开闭原则。覆写必须满足条件：

- 重写方法不能缩小访问权限
- **参数列表必须与被重写方法相同**
- 返回类型必须与被重写方法相同或是其子类
- 重写方法不能抛出新的异常，或超出父类范围的异常

```java
class Base{
  void fun(int price, int... discounts){}
}
class Sub extends Base{
  @Override
  void fun(int price, int[] discounts){}
}
public static void main(String[] args){
  Base base = new Sub();
  base.fun(100, 50);
  Sub sub = new Sub();
  sub.fun(100, 500); //此处编译不通过，子类覆写了父类方法，参数必须是int + int[]
}
```

父类变长方法编译后的形参也是int+int[]，所以覆写是正确的，编译器调用变长方法时会智能转换

但是子类覆写后就要强制类型匹配！

注意：覆写的方法参数与父类相同，不仅仅是类型、数量，还包括显示形式。

#### 建议7：警惕自增的陷阱

```java
public static void main(String[] args){
  int count = 0;
  for(int i=0; i<10; i++){
  	count = count ++;
  }
  System.out.println("count="+count);
}
//运行结果：0
```

count = count ++;的详细执行步骤：

1. JVM把count值（值是0）拷贝到临时变量区
2. count值加1，这是count值是1
3. 返回临时变量区的值，注意这个值是0
4. 返回值赋值给count，此时count被重置成0

```java
public static int mockAdd(int count){
  int temp = count;
  count = count + 1;
  return temp;
}
```

该问题在不同语言环境中有不同的实现：C++中“count=count++”与“count++”是等效的

**注意：i = i ++;**

#### 建议8：不要让旧语法困扰你

Java抛弃了goto语法，但是保留了该关键字，同时扩展了break和continue关键字，它们后面都可以加标号做跳转。但是，根本不要使用该goto功能，甚至continue和break都很少使用，提高代码可读性。

#### 建议9：少用静态导入

```java
//不使用静态导入
public static double calCircleArea(double r){
  return Math.PI * r * r;
}
//使用静态导入
import static java.lang.Math.PI;
public static double calCircleArea(double r){
  return PI * r * r;
}
```

静态导入的作用是把Math类中的PI常量引入到本类中，会使程序更简单，但是滥用静态导入会使程序更难读，更难维护。若是还使用了*（星号）通配符，把一个类的所有静态元素都导入进来，简直是噩梦。

对于静态导入，一定要遵循两个规则：

- 不实用*（通配符），除非是导入静态常量类（只包含常量的类或接口）
- 方法名是具有明确、清晰表象意义的工具类

```java
import static org.junit.Assert.*;
public class DaoTest{
  @Test
  public void testInsert(){
    //断言
    assertEquals("foo", "foo");
    assertFalse(Boolean.FALSE);
 }
}
//减少了代码量，可读性也提高了
```

#### 建议10：不要在本类中覆盖静态导入的变量和方法

如果要变更一个被静态导入的方法，最好的办法是在原始类中重构，而不是在本类中覆盖

#### 建议11：养成良好习惯，显式声明UID

```java
public class Person implements Serializable{
  //显式声明SerialVersionUID
  private static final long serialVersionUID = 55799L;
  private String name;
}
```

对象序列化，把对象从内存块转化为可传输的数据流；反序列化，将对象数据流转换为一个实例对象。

如果序列化和反序列化得类不一致情形下，反序列化会报一个**InvalidClassException**异常。JVM是就是通过SerialVersionUID（叫做流标识符）来判断类的版本定义，它可以显式声明也可以隐式声明。

当然，如果类改变不大，可以显式声明向JVM“撒谎”，反序列化也是可以运行的，只是消费端不能读取新增的业务属性（age）而已

```java
public class Person implements Serializable{
  private static final long serialVersionUID = 55799L;
  private String name;
  private String age;
}
```

注意：显式声明serialVersionUID可以避免对象不一致，但尽量不要以这种方式向JVM“撒谎”

#### 建议12：避免用序列化类在构造函数中为不变量赋值

```java
public class Person implements Serializable{
  private static final long serialVersionUID = 98979L;
  // 不变量初始不赋值
  public final String name;
  public Person(){
    name = "德天使";
  }
}
```

反序列化的执行过程是这样的：JVM从数据流中获取一个Object对象，然后根据数据流中的类文件描述信息（在序列化时，保存到磁盘的对象文件中包含了类描述信息）查看，发现是final变量，需要重新计算，于是引用Person类中的name值，而此时JVM又发现name没有赋值，于是它就“聪明”地不再初始化，保持原值。

注意：在序列化类中，不使用构造函数为final变量赋值。

#### 建议13：避免为final变量复杂赋值

```java
public class Person implements Serializable{
  private static final long serialVersionUID = 98979L;
  // 通过方法返回值为final变量赋值
  public final String name = initName();
  public String initName(){
    return "德天使";
  }
}
```

序列化保存到磁盘上的对象文件包括两部分：

1. 类描述信息。包括路径、继承关系、访问权限、变量描述、变量访问权限、方法签名、返回值，以及变量的关联类信息。注意它不记录方法、构造函数、static变量等具体实现
2. 非瞬态（transient关键字）和非静态（static关键字）的实例变量值

反序列化时final变量在以下情况下不会被重新赋值：

- 通过构造函数为final变量赋值
- 通过方法返回值为final变量赋值
- final修饰的属性不是基本类型

#### 建议14：使用序列化类的私有方法巧妙解决部分属性持久化问题

部分属性持久化问题看似简单，只要把不需要持久化的属性加上瞬态关键字（transient关键字）即可，但有时候是行不通的。

```java
public class Person implements Serializable{
  private static final long serialVersionUID = 60407L;
  private String name;
  private transient Salary salary;
  //序列化委托方法
  private void writeObject(ObjectOutputStream out)throws IOException{
    //告知JVM按照默认规则写入对象，一般放在第一句话
    out.defaultWriteObject();
    //写入
    out.writeInt(salary.getBasePay());
}
  //反序列化委托方法
  private void readObject(ObjectInputStream in)throws IOException,ClassNotFountException{
    //告知JVM按照默认规则读入对象
    in.defaultReadObject();
    //读出相应的值，队列，先进先出
    salary = new Salary(in.readInt(), 0);
  }
}
```

**序列化回调机制：**Java调用ObjectOutputStream类把一个对象转换成流数据时，会通过**反射**检查被序列化的类是否有writeObject方法，并检查其是否符合私有、无返回值的特性。若有，则委托该方法进行对象序列化，若没有，则自己按照默认规则序列化。同样，从流数据恢复对象时，也会检查一个readObject方法。

#### 建议15：break万万不可忘

记住在case语句后面随手写上break，养成良好的习惯。

可以修改IDE的警告级别，在Eclipse中，修改Preference->Java->Compiler->Errors/Warnings->Potential Programming problems，修改'switch' case fall-through 为Error级别。哈哈

#### 建议16：易变业务使用脚本语言编写

脚本语言，在运行期解释执行

灵活、便捷、简单

#### 建议17：慎用动态编译

从Java6开始，可以在运行期直接编译.java文件，执行.class，并且能够获得相关的输入输出，甚至还能监听相关的事件。

```java
public static void main(String[] args) throws Exception{
  //Java源代码
  String sourceStr = "pubilc class Hello{public String sayHello(String name){return \"Hello,\" + name + \"!\";}}";
  //类名及文件名
  String clsName = "Hello";
  //方法名
  String methodName = "sayHello";
  //当前编译器
  JavaCompiler cmp = ToolProvider.getSystemJavaCompiler();
  //Java标准文件管理器
  StandardJavaFileManager fm = cmp.getStandardFileManager(null,null,null);
  //Java对象文件
  JavaFileObject jfo = new StringJavaObject(clsName, sourceStr);
  //编译参数，类似javac <options>中的options
  List<String> optionsList = new ArrayList<String>();
  //编译文件的存放地方，此处为Eclipse工具特设
  optionsList.addAll(Arrays.asList("-d","./bin"));
  //要编译的单元
  List<JavaFileObject> jfos = Arrays.asList(jfo);
  //设置编译环境
  JavaCompiler.comilationTask task = cmp.getTask(null, fm, null, optionsList, null, jfos);
  //编译成功
  if(task.call()){
    //生成对象
    Object obj = Class.forName(clsName).newInstance();
    Class<? extends Object> cls = obj.getClass();
    //调用sayHello方法
    Method m = cls.getMethod(methodName, String.class);
    String str = (String)m.invoke(obj, "Dynamic Compilation");
    System.out.println(str);
  }
}
//文本中的Java对象
class StringJavaObject extends SimpleJavaFileObject{
  //源代码
  private String content = "";
  //遵循Java规范的类名及文件
  public StringJavaObject(String _javaFileName, String _content){
    super(_createStringJavaObjectUri(_javaFileName), Kind.SOUREC);
    content = _content;
}
  //产生一个URL资源路径
  private static URI _createStringJavaObjectUri(String name){
    //注意此处没有设置包名
    return URI.create("String:///" + name + Kind.SOURCE.extension);
 }
  //文本文件代码
  @Override
  public CharSequence getCharContent(boolean ignoreEncodingErrors) throws IOException{
  return content;
}
}
```

只要符合Java规范就可以在运行期动态加载，其实现方式是实现JavaFileObject接口，重写getCharContent、openInputStream、openOutputStream，或者实现SimpleJavaFileObject、ForwardingJavaFileObject

使用动态编译时，注意点：

- 在框架中谨慎使用，如Structs、spring
- 不要在要求高性能的项目使用，毕竟多了编译过程。
- 动态编译要考虑安全问题，防止注入漏洞（比如上传一个Java文件运行）
- 记录动态编译过程，如源文件、目标文件、编译过程、执行过程等日志

#### 建议18：避免instanceof非预期结果

```java
boolean b1 = "String" instanceof Object; // true
boolean b2 = new String() instanceof String; // true
boolean b3 = new Object() instanceof String; //false
boolean b4 = 'A' instanceof Charaacter; //编译不通过，基本类型不能用于对象判断
boolean b5 = null instanceof String; //false，左操作数是null，直接返回false
boolean b6 = (String)null instanceof String; //false，null没有类型，即时强制转换还是null
boolean b7 = new Date() instanceof String; //编译报错，没有继承实现关系
boolean b8 = new GenericClass<String>().isDateInstance(""); //false，范型T表面类型是Object(实际是String)，所以编译通过，Object显然不一定是Date
```

#### 建议19：断言绝对不是鸡肋

assert的语法有两个特性：

- assert默认是不启用的


- assert抛出的异常AssertionError是继承自Error的

assert虽然是做断言的，但不能将其等价于if...else...这样的条件判断，在以下两种情况不可使用：

1. 在对外公开的方法中，不能用断言做外部输入校验
2. 在执行逻辑代码的情况下。assert的支持是可选的，在开发时可以让它运行，但在生产系统中则不需要其运行了（以便提高性能）

按照正常逻辑**不可能到达的代码区域**可以防止assert

1. 在私有方法中放置assert作为输入参数校验
2. 流程控制中不可能达到的区域
3. 建立程序探针

#### 建议20：不要只替换一个类

对于final修饰的**基本类型和String类型**，编译器会认为它是稳定态，所以在编译时就直接把**值编译到字节码**中了，避免了在运行期引用；

对于final修饰的**类**，编译器认为它是不稳定态，在编译时建立引用关系（该类型叫做soft final），即使不重新编译也会输出最新值。

所以，如果直接替换class类文件来发布，类似`final static int MAX_AGE = 180 改成 150;`这种更改不会更新到调用代码中

注意：发布应用系统时禁止使用类文件替换方式，整体WAR包发布才是万全之策。

## 第二章 基本类型

Java中的基本数据类型有8个：byte、char、short、int、long、float、double、boolean。

#### 建议21：用偶判断，不用奇判断

```java
//模拟取余计算dividend被除数 divisor除数
public static int remainder(int dividend, int divisor){
  return dividend - dividend / divisor * divisor ;
}
```

所以：**-1 % 2 = -1**

用偶数判断替代奇数判断：i % 2 == 0 ? "偶数" : "奇数"

注意：对于基础知识，我们应该知其然，并知其所以然

#### 建议22：用整数类型处理货币

```java
System.out.println(10.00 - 9.60);
// 结果是 0.4000000000036
```

十进制0.4不能使用二进制准确表示，在二进制的世界里它是一个无限循环小数。（十进制转换二进制，使用“乘2取整，顺序排列”法）

解决方法：

1. 使用BigDecimal：专门为弥补浮点数无法精确计算而设计的类，并且本身提供了加减乘除的常用数学算法。特别是与数据库Decimal类型的字段映射时首选
2. 使用整型：把参与运算的值扩大100倍，并转换为整型，展现时再缩小100倍。

#### 建议23：不要让类型默默转换

```java
final int LIGHT_SPEED = 30 * 10000 * 1000;
long dis2 = LIGHT_SPEED * 60 * 8;
// dis2的结果是 ： -2028888064
```

计算的过程中三者都是int类型，结果已经超出int最大值，虽然最后转换成了long，结果还是负值

解决方案：`long dis2 = 1L * LIGHT_SPEED * 60 * 8;`

注意：基本类型转换时，使用主动声明方式减少不必要的Bug

#### 建议24：边界，边界，还是边界

```java
int order = 2147483647;
int cur = 1000;
int LIMIT = 2000;
if(order>0 && order+cur <= LIMIT){
  System.out.println("你已经成功预订"+order+"个产品！");
}
```

**int的取值范围：-2147483648 ~ 2147483647**

因为order + cur越界，所以结果为负值，肯定小于LIMIT

注意：对输入的边界测试很重要，不要过度依赖web前端的js校验，因为高手可以用HTTP模拟调用

#### 建议25：不要让四舍五入亏了一方

四舍五入是有误差的：其误差值是舍入位的一半。

银行家舍入：四舍六入五考虑，五后非零就进一，五后为零看奇偶，五前为偶应舍去，五前为奇要进一。

注意：根据不同的场景，慎重选择不同的舍入模式，以提高项目的精准度，减少算法损失。

#### 建议26：提防包装类型的null值

Java引入包装类型是为了解决基本类型的实例化问题，以便让一个基本类型也能参与到面向对象的编程世界中。

但是对null值进行包装自动装箱和自动拆箱会产生NullPointerException异常，所以包装类型参与运算时，要做null值校验

#### 建议27：谨慎包装类型的大小比较

```java
Integer i = new Integer(100);
Integer j = new Integer(100);
System.out.println(i == j); //false，相等比较，对象比较引用
System.out.println(i > j); //false，比较大小，去intValue()方法的值，显然是相等的
System.out.println(i.compareTo(j)) //返回0，表示相等
```

对象的比较应该采用相应的方法，而不是通过Java的默认机制来处理。

#### 建议28：优先使用整型池

```java
int ii = input.nextInt();
Integer i = new Integer(ii);
Integer j = new Integer(ii);
System.out.println("new 产生的对象"+ (i == j));
i = ii;
j = ii;
System.out.println("基本类型转换的对象"+ (i == j));
i = Integer.valueOf(ii);
j = Integer.valueOf(ii);
System.out.println("valueOf 产生的对象"+ (i == j));
```

输入：127，false，true，true

输入：128，false，false，false

输入：555，false，false，false

1. new产生的Integer对象：两个对象，地址不同，肯定false
2. 装箱生成的对象：首先说明装箱动作是通过valueOf()方法实现的，所以后两个算法相同。下面看下Integer.valueOf的实现代码：

```java
public static Integer valueOf(int i){
  final int offset = 128;
  if(i >= -128 && i<-127){
  return IntegerCache.cache[i + offset];
  }
  return new Integer(i);
}
```

所以-128到127之间的int类型转Integer对象，则直接从cache数组中获得。cache是IntegerCache内部类的一个静态数组，容纳的是-128到127之间的Integer对象。

整型池提高了性能，节约了内存空间。

在判断对象相等时，最好用equals方法！

注意：通过包装类的valueOf生成包装实例可以显著提高空间和时间性能

#### 建议29：优先选择基本类型

基本类型可以先加宽，再转变成宽类型的包装类型，但不能直接转变成宽类型的包装类型。

无论是从安全性、性能方面来说，还是稳定性来说，基本类型都是首选方案。

#### 建议30：不要随便设置随机种子

```java
Random r = new Random(1000);
r.nextInt();
```

记住：同一台机器上，甭管运行多少次，所打印的随机数都是相同的。也就是每次运行都会打印出相同的随机数。

在Java中，随机数产生取决于种子。

- 种子不同，产生不同的随机数；


- 种子相同，即使实例不同也产生相同的随机数；

Random的默认种子是System.nanoTime()返回值，不同操作系统和硬件有不同的固定时间点。

new Random(1000)显式地设置了随机种子为1000，运行多次获得随机数相同。

Math.random()方法也是生成一个Random类实例，没有差别。

注意：若非必要，不要设置随机数种子。

## 第三章 类、对象及方法

#### 建议31：在接口中不要存在实现代码

```java
public class Client{
  public static void main(String[] args){
  B.s.doSomething();
}
}
//在接口中存在实现代码
interface B{
  public static final S s = new S(){
    public viod doSomething(){
      System.out.println("我在接口中实现了");
    }
  }
}
//被实现的接口
interface S{
  public void doSomething();
}
```

在B接口中声明了一个静态常量s,其值是一个匿名内部类的实例对象，就是该匿名内部类实现了S的接口。

注意：接口中不能存在实现代码

#### 建议32：静态变量一定要先声明后赋值

```java
public class Client{
  static{
    i = 100;
  }
  public static int i=1;
  public static void main(String[] args){
    System.out.println(i);
  }
}
```

可以编译，结果是1.

静态变量在类初始化时首先被加载，JVM先查找静态声明，然后分配空间，还没有赋值。之后JVM会根据静态赋值（包括静态类赋值和静态块赋值）的先后顺序来执行。所以如果有多个静态块对i赋值，结果是最后一个赋值决定的。

所以变量一定要**先声明后使用**

#### 建议33：不要覆写静态方法

一个实例对象有两个类型：表面类型和实际类型，表面类型是声明时的类型，实际类型是对象产生时的类型。

静态方法不依赖实例对象，它是通过类名访问的；如果通过对象调用静态方法，JVM会通过对象的表面类型查找到静态方法的入口。

在子类中构建与父类同名、输入、输出、访问权限相同，并且都是静态方法，此种行为叫做**隐藏**，与覆写有两点不同：

- 表现形式不同
- 职责不同。隐藏是为了抛弃父类静态方法；而覆写是将父类的行为增强或减弱，延续父类的职责。

注意：通过实例对象访问静态方法或静态属性是代码的“坏味道”

#### 建议34：构造函数尽量简化

子类实例化时，会首先初始化父类，也就是初始化**父类的变量**，调用**父类的构造函数**，然后才会初始化**子类的变量**，调用**子类自己的构造函数**，最后生成一个实例对象。

所以，构造函数应当简化，再简化。

#### 建议35：避免在构造函数中初始化其他类

```java
public static void main(String[] args){
  Son s = new Son();
  s.doSomething();
}
class Father{
  Father(){
    new Other();
  }
}
class Son extends Father{
  public void doSomething(){
    System.out.println("Hi, something");
  }
}
class Other{
  public Other(){
    new Son();
}
}
```

这段代码报StackOverflowError，栈内存移除。死循环调用构造函数。

所以：不要在构造函数中声明初始化其他类，养成良好的习惯。

#### 建议36：使用构造代码块精炼程序

Java中一共有四种类型的代码块：

1. 普通代码块，方法后使用的{}的代码片段，必须通过方法调用。
2. 静态代码块，使用static修饰并用{}括起来的代码片段，用于静态变量的初始化或对象创建前的环境初始化
3. 同步代码块，使用synchronized关键字修饰，并使用{}括起来，表示同一时间只能有一个线程进入到该方法块中
4. 构造代码块，在类中没有任何前缀或后缀，使用{}括起来的代码片段。

```java
public class Client{
  {
    System.out.println("执行构造代码块");
  }
  public Client(){
    System.out.println("执行无参构造");
  }
  public Client(){
    System.out.println("执行有参构造");
}
}
```

执行结果是：每个**构造函数的最前端都插入**了构造代码块。

所以可以把构造代码块用到以下场景：初始化实例变量；初始化实例环境。每个代码块完成不同的业务逻辑，JVM按照顺序依次执行，实现复杂对象的**模块化创建**。

#### 建议37：构造代码块会想你所想

```java
class Base{
private static int numOfObject = 0;
  {
    numOfObject ++;
  }
  pupblic Base(){
  }
  public Base(String _str){
  this();
  }
}
```

JVM会把代码块插入到没有this方法的构造函数中，而调用其他构造函数的则不插入，确保每个构造函数只执行一个构造代码块。

#### 建议38：使用静态内部类提高封装性

Java中的嵌套类有两种，静态内部类和内部类。

静态内部类有两个优点：加强了类的封装性和提高了代码的可读性。

```java
public class Person{
  private String name;
  private Home home;
  public Person(String _name){
    name = _name;
  }
  // 静态内部类
  public static class Home{
    private String address;
    private String tel;
    public Home(String _address, String _tel){
      address = _address;
      tel = _tel;
    }
  }
}
Person p = new Person("张三");
p.setHome(new Person.Home("上海", "021"));
```

静态内部类和不同内部类的区别：

1. 静态内部类不持有外部类的引用。在普通内部类中，可以访问外部类的private的属性和方法，因为它持有一个外部类的引用。而静态内部类只可以访问外部类的静态方法和静态属性（private权限的也能访问，这是由代码位置决定的）。
2. 静态内部类不依赖外部类。普通内部类实例不能脱离外部类，会一起被垃圾回收器回收。而静态内部类可以独立存在。
3. 普通内部类不能声明static的方法和变量。注意是变量，常量（final static）是可以的，而静态内部类形似外部类，没有任何限制。

#### 建议39：使用匿名类的构造函数

```java
List l1 = new ArrayList();
List l2 = new ArrayList(){};
List l3 = new ArrayList(){ {} };
boolean b1 = l1.getClass() == l2.getClass(); //false
boolean b2 = l2.getClass() == l3.getClass(); //false
boolean b3 = l1.getClass() == l3.getClass(); //false
//三个类不同
```

List l2 = new ArrayList(){}，代表一个匿名类的声明和赋值，它定义了一个继承与ArrayList的匿名类，只是没有任何的覆写方法而已，其代码类似于：

```java
class Sub extends ArrayList{  
}
List l2 = new Sub();
```

`List l3 = new ArrayList(){ {} }`，也是一个匿名类的定义，只是多了初始化块而已，起到构造函数的功能，其代码类似于：

```java
class Sub extends ArrayList{
  {
  //初始化块
  }
}
List l3 = new Sub();
```

当然，一个类的初始化块可以有多个，所以`List l4 = new ArrayList(){ {} {} {} {} {} };` 代码是没有问题的。

匿名类虽然没有名字，但也是可以有构造函数的，虽然父类相同，但类还是不同的。

#### 建议40：匿名类的构造函数很特殊

匿名类的构造函数只能由构造代码块代替，它在初始化时直接调用父类的同参构造，然后再调用自己的构造代码块。

#### 建议41：让多重继承成为现实

```java
class Daughter extends MotherImpl implements Father{
  public int strong(){
    return new FatherImpl(){
      @Override
      public int strong(){
        return super.strong() - 2;
      }
    }.strong();
  }
}
```

多重继承指的是一个类可以同时从多于一个父类那里继承行为与特征。

#### 建议42：让工具类不可实例化

将工具类构造函数设置成private访问权限，还抛异常。Java的反射机制可以修改构造函数的访问权限。

```java
public class UtilsClass{
  private UtilClass(){
    throw new Error("不要实例化我！");
  }
}
```

#### 建议43：避免对象的浅拷贝

Object的clone方法拷贝规则如下：

1. 基本类型，拷贝其值，如int、float等
2. 对象，拷贝其地址引用
3. String字符串，拷贝地址引用，但是在修改时，会从字符串池中重新生成新的字符串，原有的字符串对象保持不变

浅拷贝只是Java提供的一种简单拷贝机制，不便于直接使用。所以在实现Cloneable接口，覆写clone方法的时候，注意不要简单调用super.clone()，解决浅拷贝问题

#### 建议44：推荐使用序列化实现对象拷贝

```java
public class CloneUtils{
  public static <T extends Serializable> T clone(T obj){
    //拷贝产生的对象
    T clonedObj = null;
    try{
      ByteArrayOutputStream baos = new ByteArrayOutputStream();
      ObjectOutputSream oss = new ObjectOutputSream(baos);
      oss.writeObject(obj);
      oss.close();
      //分配内存空间，写入原始对象，生成新对象
      ByteArrayInputSream bais = new ByteArrayInputSream(baos.toByteArray());
      ObjectInputSream ois = new ObjectInputStream(bais);
      clonedObj = (T)ois.readObject();
      ois.close();
    }catch(Exception e){
      e.printStackTrace();
    }
    return clonedObj.
  }
}
```

此方法对象拷贝时注意两点：

- 对象的内部属性都是可序列化的，否则会抛出序列化异常


- 注意方法和属性的特殊修饰符，比如final、static变量的序列化问题，transient变量等等

#### 建议45：覆写equals方法时不要识别不出自己

**保证自反性**

#### 建议46：equals应该考虑null情景

#### 建议47：在equals中使用getClass进行类型判断

getClass有自反性，而instanceof没有，所以要用getClass进行类型判断

#### 建议48：覆写equals方法必须覆写hashCode方法

对象元素的hashCode方法返回对象的哈希码，由Object类本地方法生成，确保每个对象有一个哈希码。**相同的对象的哈希码必须相同，不同的对象，哈希码可以相同。**

```java
public boolean equals(Object obj){
  //null和getClass使用
  if(obj != null && obj.getClass() == this.getClass()){
    Person p = (Person) obj;
    if(p.getName()==null || name == null){
      return false;
    }else{
      return name.equalsIgnoreCase(p.getName());
    }
  }
  return false;
}
public int hashCode(){
  return new HashCodeBuilder().append(name).toHashCode();
}
```

#### 建议49：推荐覆写toString方法

pringln实现机制：如果是一个原始类型就直接打印，如果是一个类类型，则打印出其toString方法的返回值

#### 建议50：使用package-iinfo类为包服务

- 声明友好类和包内访问常量
- 为在包上标注注解提供便利
- 提供包的整体注释说明

#### 建议51：不要主动进行垃圾回收

System.gc要停止所有响应，才能检查内存中是否有可回收的对象，这对一个应用系统来说风险极大。

## 第四章 字符串

#### 建议52：推荐使用String直接量赋值

```java
String str1 = "中国";
String str2 = "中国"；
String str3 = new String("中国");
String str4 = str3.intern();
boolean b1 = (str1 == str2);
boolean b2 = (str1 == str3);
boolean b3 = (str1 == str4);
//引用地址比较，运行结果是true, false, true
```

**字符串池：**创建字符串时，首先检查池中是否有字面值相等的字符串，如果有则直接返回池中该对象的引用；没有则创建之，然后放到池中，并返回新建对象的引用。

new String("中国")；不会检查字符串池，也不会把对象放到池中。

intern方法会检查当前的对象在对象池中是否有字面值相同的引用对象，如果有则返回池中对象，如果没有则放置到对象池中，并返回当前对象。

**对象池会不会产生线程安全问题**：String类是不可变的，一是String类是final类，不可继承，不能产生String子类；二是在String提供的方法中，如果有返回值，会新建一个String对象，不对原来对象进行修改。保证了原对象是不可变的。

字符串池存在JVM的常量池中，垃圾回收不会对它进行回收。

#### 建议53：注意方法中传递的参数要求

replaceAll传递的第一个参数是正则表达式

#### 建议54：正确使用String、StringBuffer、StringBuilder

三者都是**CharSequence**接口的实现类

- String类是不可改变的量，创建后不能修改。String s = "a"; s = a + "b";  s初始化时是“a”对象的引用，加号操作之后指向“ab”对象的新引用地址


- StringBuffer是一个可变字符序列。StringBuffer sb = new StringBuffer("a"); sb.append("b");


- StringBuilder和StringBuffer基本相同，不同的是StringBuffer是线程安全的，方法前都有synchronized关键字，所以StringBuffer 在性能上低于StringBuilder。

使用String类的场景：字符串不经常变化的场景，如常量的声明、少量的变量运算；

使用StringBuffer的场景：频繁进行字符串的运算，且在多线程环境中，如XML解析，HTTP参数解析封装

使用StringBuilder的场景：频繁进行字符串的运算，且在单线程环境中，如SQL语句的拼接、JSON封装等

#### 建议55：注意字符串的位置

1 + 2 + “apples”与“apples ” + 1+ 2

Java对加号的处理机制：只要遇到String字符串，则 所有的数据都会转换为String类型拼接，如果是原始数据则直接拼接，如果是对象，则调用toString方法的返回值拼接

#### 建议56：自由选择字符串拼接方法

字符串拼接的三种方法：加号、concat方法和StringBuilder的append方法。其中，append方法最快，concat方法次之，加号最慢

- 加号原理类似于：str = new StringBuilder(str).append("c").toString();，所以每次执行都要调用toString方法
- concat方法，整体看上去是一个数组拷贝，但每次操作都会创建一个String对象
- append方法，字符数组处理，加长，然后数组拷贝

注意：适当的场景使用适当的字符串拼接方式

#### 建议57：推荐在复杂字符串操作中使用正则表达式

正则表达式在字符串查找、替换、剪切、赋值、删除等方面有着非凡的作用，特别是面对大量文本字符（如log日志）需要处理，使用正则表达式可以大幅提高开发效率和系统性能

#### 建议58：强烈建议使用UTF编码

Java程序设计的编码包括两部分：

- Java文件编码，.java文件的编码格式，一般是操作系统默认格式
- Class文件编码，通过javac命令生成的.class文件是UTF-8编码的UNICODE文件，在任何操作系统上都一样

#### 建议59：对字符串排序持一种宽容的心态

注意：如果排序对象是经常使用的汉子，使用Collator类即可。GB2312包含7000多个字符集，超出的部分，如果需要严格排序，则需要使用一些开源项目来实现了

```java
String[] strs = {"张三(z)", "李四(L)", "王五(w)"};
Comparator c = Collator.getInstance(Locale.CHINA);
Arrays.sort(strs, c);
```



参考：编写高质量代码——改善Java程序的151个建议



















