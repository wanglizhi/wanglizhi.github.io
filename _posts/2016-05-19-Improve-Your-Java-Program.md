---
layout:     post
title:      "改善Java程序的151个建议读书笔记（中）"
subtitle:   "5、数组和集合；6、枚举和注解；7、泛型和反射"
date:       2016-05-19 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Java
---

## 第五章 数组和集合

#### 建议60：性能考虑，数组是首选

List< Integer > 对比int[]：

在初始化List数组时要把int类型**包装**成一个Integer对象，而对象是在堆内存中操作的，堆内存的特点是速度慢、容量大，栈内存的特点是速度快、容量小。并且，在求和计算时要做**拆箱**动作，性能消耗就产生了。

测试发现，数组的效率是集合的10倍。

#### 建议61：若有必要，使用变长数组

```java
public static <T> T[] expandCapacity(T[] datas, int newLen){
  newLen = newLen<0 ? 0 : newLen;
  return Arrays.copyOf(datas, newLen);
}
```

数组扩容，曲折地解决了数组变长问题

#### 建议62：警惕数组的浅拷贝

Arrays.copyOf(); 对于基本类型是直接拷贝值，其他都是拷贝引用地址，此外**数组和集合的clone方法**也都是浅拷贝

#### 建议63：在明确的场景下，为集合指定初始容量

ArrayList底层使用数组存储，当长度达到elementData临界点时，将elementData扩容1.5倍。避免多次为数组重新分配内存，性能消耗严重。而默认的elementData的长度是10，所以如果数据量很大且不设置初值，每次扩容都是一次**数组的拷贝**，效率非常低。

ArrayList的长度扩展方式：

```java
public void ensureCapacity(int minCapacity){
  modCount++;
  int oldCapacity = elementData.length;
  if(minCapacity > oldCapacity){
    Object oldData[] = elementData;
    int newCapacity = (oldCapacity * 3)/2 + 1;
    if(newCapacity < minCapacity)
      newCapacity = minCapacity;
    elementData = Arrays.copyOf(elementData, newCapacity);
  }
}
```

Vector的长度扩展方式：

```java
private void ensureCapacityHalper(int minCapacity){
  int oldCapacity = elementData.length;
  if(minCapacity > oldCapacity){
    Object[] oldData = elementData;
    //若有递增步长，则按照步长增长，否则扩容2倍
    int newCapacity = (capacityIncrement > 0) ? (oldCapacity + capacityIncrement) : (oldCapacity * 2);
    if(newCapacity < minCapacity){
      newCapacity = minCapacity;
    }
    elementData = Arrays.copyOf(elementData, newCpacity);
}
}
```

Vector提供了递增步长变量。不设置则容量翻倍；

HashMap是按照倍数增加的；Stack继承自Vector；

注意：非常必要在集合初始化时声明容量。

#### 建议64：多种最值算法，适时选择

```java
public static int getSecond(Integer[] data){
  List<Integer> dataList = Arrays.asList(data);
  TreeSet<Integer> ts = new TreeSet<Integer>(dataList);
  return ts.lower(ts.last());
}
```

删除重复元素并升序排序，然后再使用lower方法寻找小于最大值的值。利用TreeSet类简化

#### 建议65：避开基本类型数组转换列表陷阱

```java
int[] data = {1, 2, 3, 4, 5};
List list = Arrays.asList(data);
System.out.println("列表中元素数量是： "+list.size());
//结果是 1，我们看下asList的代码
public static <T> List<T> asList(T...a){
  return ArrayList<T>(a);
}
//asList输入的是一个泛型变长参数，而基本类型参数是不能泛化的，所以必须使用包装类型
Integer[] data = {1, 2, 3, 4, 5};
List list = Arrays.asList(data);
```

注意：原始类型数组不能作为asList的输入参数，否则会引起程序逻辑混乱。

#### 建议66：asList方法产生的List对象不可更改

asList返回的ArrayList对象是一个内部类对象，而不是java.util.ArrayList

```java
private static class ArrayList<E> extends AbstractList<E> implements RandomAccess, java.io.Serializable{
  private final E[] a;
  ArrayList(E[] array){
    if(array == null){
      throw new NullPointerException();
    }
    a = array;
  }
}
```

该内部类没有实现List.add和List.remove方法，所以长度不可变。

注意：asList转换的列表不可变长。

#### 建议67：不同的列表选择不同的遍历方法

ArrayList类实现了RandomAccess接口，是随机存取的，采取**下标**方式遍历列表速度会更快。

LinkedList实现了双向链表，两个元素间是有关联的，所以foreach和**迭代器**效率更高。

#### 建议68：频繁插入和删除时使用LinkedList

LinkedList是双向链表，插入效率比ArrayList快50倍以上，删除速度快40倍以上。

ArrayList随机访问，修改元素效率高

#### 建议69：列表相等只需关心元素数据

```java
ArrayList<String> strs = new ArrayList<String>();
strs.add("A");
Vector<String> strs2 = new Vector<String>();
strs2.add("A");
System.out.println(strs.equals(strs2));
```

结果是：true，因为两者都实现了List接口，equals方法在AbstractList中定义，只要**所有元素相等并且长度相等**，就表示两个List相等。与集合类型无关。

注意：判断集合是否相等时只需关注元素是否相等即可。

#### 建议70：子列表只是原列表的一个视图

subList方法返回的SubList类是AbstractList的子类，其所有的方法如get、set、add、remove等都是在原始列表上的操作，它自身并没有生成一个数组或链表，只是原列表的一个视图。

#### 建议71：推荐使用subList处理局部列表

```java
//删除索引位置为20-30的元素
List<Integer> initData = Collections.nCopies(100, 0);
ArrayList<integer> list = new ArrayList<Integer>(initData);
list.subList(20,30).clear();
```

#### 建议72：生成子列表后不要再操作原列表

```java
List<String> list = new ArrayList<String>();
List.add("A");
List.add("B");
list.add("C");
List<String> subList = list.subList(0, 2);
list.add("D");
System.out.println(list.size()); //输出 4
System.out.println(subList.size()); //抛异常，java.util.ConcurrentModificationException
```

子列表提供size方法检测，checkForComodification检测是否发生并发修改，如果变化则抛出异常。所以有效的办法就是通过Collections.unmodifiableList方法设置列表为只读状态。

```java
List<String> subList = list.subList(0, 2);
list = Collections.unmodifiableList(list);
```

注意：subList生成子列表后，保持原列表的只读状态

#### 建议73：使用Comparator进行排序

Java给数据排序有两种方式，一种是实现Comparable接口，一种是实现Comparator接口，两者区别如下：

```java
class Employee implements Comparable<Employee>{
  private int id;
  private String name;
  private Position position;
  @Override
  public int comparaTo(Employee o){
    return new CompareToBuilder().append(id, o.id).toComparison();
}
}
enum Position{Boss, Manager, Staff}
List<Employee> list = new ArrayList<Employee>(5);
Collections.sort(list); //按照id排序
```

Collections.sort(List<T> list, Comparator<? super T> c)，该接口可以接受一个Comparator实现类

```java
class PositionComparator implements Comparator<Employee>{
  @Override
  public int compare(Employee o1, Employee o2){
    return o1.getPosition().compareTo(o2.getPosition());
}
}
Collections.sort(list, new PositionComparator());
```

实现了Comparable接口的类表明自身是可比较的，有了比较才能排序；而Comparator接口是一个**工具接口**，只是实现两个类的比较逻辑，一个类可以有多个比较器，产生N种排序。而一个类只能有一个固定的、由compareTo方法提供的默认排序算法。

#### 建议74：不推荐使用binarySearch对列表进行检索

二分查找必须是排序后的列表。

- indexOf依赖equals方法查找，binarySearch则依赖于compareTo方法查找

注意：实现了compareTo方法，就应该覆写equals方法，确保两者同步。

#### 建议76：集合运算时使用更优雅的方式

```java
//并集
list1.addAll(list2);
//交集
list1.retainAll(list2);
//差集
list1.removerAll(list2);
//无重复的并集
list2.removeAll(list1);
list1.addAll(list2);
```

#### 建议77：使用shuffle打乱列表

Collections.shuffle(list);

#### 建议78：减少HashMap中元素的数量

```java
static class Entry<K,V> implements Map.Entry<K,V>{
  final K key;
  V value;
  Entry<K,V> newxt;
  final int hash;
}
```

HashMap比ArrayList多了一层Entry的底层对象封装，多占用了内存，并且它的扩容策略是**2倍长度的递增**，同时还会依据阈值判断规则进行判断。只要HashMap的size大于数组长度的0.75倍时，就开始扩容。

#### 建议79：集合中的哈希码不要重复

HashMap的put函数：

```java
public V put(K key, V value){
  //处理null键
  if(key==null) return putForNullKey(value);
  //计算hash值,并定位
  int hash = hash(key.hashCode());
  int i = indexFor(hash, table.lenght);
  for(Entry<K, V> e = table[i]; e!=null; e=e.next){
    Object k;
    //哈希码系统，并key相等，则覆盖
    if(e.hash == hash && ( (k=e.key) == key) || key.equals(k))){
      V oldValue = e.value;
      e.value = value;
      e.recordAccess(this);
      return oldValue;
    }
  }
}
//有时间深入研究下
static int hash(int h){
  h ^= (h >> 20) ^ (h >>> 12);
  return h ^ (h >>> 7) ^ (h >>> 4);
}
static int indexFor(int h, int length){
  return h & (length-1);
}
```

HashMap的存储诛仙还是数组，遇到哈希冲突的时候使用链表解决。所以，HashMap中hashCode应避免冲突。

#### 建议80：多线程使用Vector或HashTable

Vector是ArrayList的多线程版本，HashTable是HashMap的多线程版本。

**线程安全**是指同一时间只允许一个线程进入该方法；

**线程同步**是为了保护集合中的数据不被脏读、脏写。vector集合如果多个线程同时进行读、写操作，会产生线程同步问题。

基本所有的集合类都有一个叫做快速失败（Fail-Fast）的校验机制，如果读列表时，modCount发生变化则会抛出ConcurrentModificationException异常。

#### 建议81：非稳定排序推荐使用List

TreeSet类实现了默认排序为升序的Set集合（Set中的元素不可重复），如果插入一个元素，默认按照升序排列。TreeSet实现了SortedSet接口，定义了在给定集合加入元素时将其进行排序，并不能保证元素修改后的排序结果，因此**TreeSet适用于不变量的集合数据排序**。

#### 建议82：集合大家族

1. List：实现LIst接口的集合主要有：ArrayList、LinkedList、Vector、Stack，其中ArrayList是一个动态数组，LinkedList是一个双向列表，Vector是一个线程安全的动态数组，Stack是一个对象栈，后进先出
2. Set：是不包含重复元素的集合，其主要实现类有：EnumSet、HashSet、TreeSet，其中EnumSet是枚举类型的专用Set，所有元素都是枚举类型；HashSet是以哈希码决定其元素位置的Set，原理与HashMap相似，提供快速插入和查找；TreeSet是一个自动排序的Set，它实现了SortedSet接口
3. Map：排序Map主要是TreeMap，根据Key值自动排序；非排序Map，主要包括：HashMap、HashTable、Properties、EnumMap等。其中Properties是HashTable的子类，重要用途是从Property文件中加载数据
4. Queue：分为两类，一类是阻塞式队列，队列满了后再插入会抛出异常，主要包括：ArrayBlockingQueue、PriorityBlockingQueue、LinkedBlockingQueue；另一类是非阻塞队列，无边界，只要内存允许都可以持续追加元素，最常使用的是PriorityQueue。
5. 数组：数组可以容纳基本类型二集合不行，所有的集合底层存储的都是数组
6. 工具类：java.util.Arrays, Java.lang.reflect.Array, Java.util.Collections

## 第六章 枚举和注解

#### 建议83：推荐使用枚举定义常量

枚举的优点

1. 枚举常量更简单
2. 枚举常量属于稳态型，switch判断语句中必须制定该枚举类型，避免校验问题
3. 枚举有内置方法，values获取所有枚举项，获得排序值得ordinal方法，compareTo比较方法
4. 枚举可以自定义方法

```java
enum Season{
  Spring, Summer, Autumn, Winter;
  //最舒服的季节
  public static Season getComfortableSeason(){
    return Spring;
  }
}
```

接口常量的优点：继承，**枚举类型不能继承**

#### 建议84：使用构造函数协助描述枚举项

```java
enum Season{
  Spring("春"), Summer("夏"), Autumn("秋"), Winter("冬");
  private String desc;
  Season(String _desc){
    desc = _desc;
  }
  public String getDesc(){
    return desc;
  }
}
```

#### 建议85：小心switch带来的空值异常

switch(enum)中，如果输入null，会产生空指针。

因为switch语句是先计算enum变量的**排序值** （ordinal方法），然后与枚举常量的排序值进行对比。所以null没有ordinal方法，所以产生空指针异常

#### 建议86：在switch的default代码块中增加AssertionError错误

#### 建议87：使用valueOf前必须进行校验

Enum类中的valueOf方法会把一个String类型的名称转变成枚举项。

Season s = Season.valueOf("Spring");

原理：valueOf方法通过反射从枚举类的常量声明中查找，若找到就直接返回，若找不到则抛出无效参数异常

所以要使用try...catch捕获异常，或者扩展枚举类

#### 建议88：用枚举实现工厂方法模式更简洁

工厂方法模式：创建对象的接口，让子类决定实例化哪一个类，并使一个类的实例化延迟到其子类。

```java
//一般实现
interface Car{};
class FordCar implements Car{};
class BuickCar implements Car{};
//工厂类
class CarFactory{
  public static Car createCar(Class<? extends Car> c){
    try{
      return (Car)c.newInstance();
    }catch(Exception e){
      e.printStackTrack();
    }
  }
}
Car car = CarFactory.createCar(FordCar.class);
```

枚举实现工厂方法模式有两种方法：

1、枚举非静态方法实现工厂方法模式

```java
enum CarFactory{
  FordCar, BuickCar;
  public Car create(){
    switch(this){
    case FordCar:
      return new FordCar();
    case BuickCar:
      return new BuickCar();
    default:
      throw new AssertionError("无效参数");
    }
  }
}
//create是一个非静态方法，只有通过FordCar、BuickCar枚举项才能访问
Car car = CarFactory.BuickCar.create();
```

2、通过抽象方法生成产品

```java
enum CarFactory{
  FordCar{
    public Car create(){
      return new FordCar();
    }
  },
  BuickCar{
    public Car create(){
      return new BuickCar();
    }
  };
  //抽象生产方法
  public abstract Car create();
}
//调用和第一种方法相同
```

枚举类型的工厂方法模式有三个有点：

- 避免错误调用的发生
- 性能好，使用便捷
- 降低类之间的耦合

#### 建议89：枚举项的数量限制在64个以内

EnumSet表示其元素必须是某一枚举的枚举项；EnumMap表示Key值必须是某一枚举的枚举项。

当枚举项数量小于等于64时，创建一个RegularEnumSet实例对象，大于64时则创建一个JumboEnumSet实例对象。两者原理相似，只是JumboEnumSet使用了long数组容纳更多的枚举项。

#### 建议90：小心注解继承

@Inherited注解，表示的意思是只要把注解@Desc加到父类Bird上，它的所有子类都会自动从父类继承@Desc注解，不需要显示声明

#### 建议92：注意@Override不同版本的区别

@Override注解用于方法覆写上，在编译期有效，JVM编译时检查方法是否覆写，如果不是就报错，拒绝编译。该注解可以解决误写问题

Java1.5版本中，需要删除接口方法上的@Override注解，父类必须是一个类。

## 第七章 泛型和反射

#### 建议93：Java的泛型是类型擦除的

Java的泛型在编译期有效，在运行期被删除。编译后所有泛型的类型都会做相应转化，规则如下：

- List< String >, List< Integer>, LIst< T >擦除后为List
- List< String >[] 擦除后为List[]
- List<? extends E>, List<? super E>擦除后为List< E >
- List< T extends Serializable & Cloneable > 擦除后为List< Serializable >

1、泛型的class对象是相同的

```java
List<String> ls = new ArrayList<String>();
List<Integer> li = new ArrayList<Integer>();
System.out.println(li.getClass() == li.getClass());
//结果是true，都是List类型
```

2、泛型数组初始化时不能声明泛型类型

List< String >[] listArray = new List< String >[]; 编译不通过

3、instanceof不允许存在泛型参数

List< String > list = new ArrayList< String >();

System.out.println(list instanceof List< String >); 编译不通过

#### 建议94：不能初始化泛型参数和数组

```java
class Foo<T>{
  private T t = new T();
  private T[] tArray = new T[5];
  private List<T> list = new ArrayList<T>();
}
```

这段代码编译不通过，因为编译器在编译期擦除了泛型，所以new T()和new T[5]都会报错。

如果确实需要泛型数组，应该交给构造函数初始化

```java
class Foo<T>{
  private T t;
  private T[] tArray;
  public Foo(){
    try{
      Class<?> tType = Class.forName("");
      t = (T)tType.newInstance();
      tArray = (T[])Array.newInstance(tType, 5);
    }catch...
  }
}
```

类成员变量是在类初始化前初始化的，所以要求在初始化前它必须具有明确的类型，否则就只能声明，不能初始化。

#### 建议96：不同场景使用不同的泛型通配符

1、泛型结构只参与“读”操作则限定**上界（extends关键字）**

```java
public static <E> void read(List<? extends E> list){
  for(E e:list){ //已经推断出取出的是E类型的元素
  }
}
```

2、泛型结构只参与“写”操作则限定**下界（super关键字）**

```java
public static void write(List<? super Number> list){
  list.add(123);
  list.add(3.14);
}
```

写操作情况下，限定下界，甭管是Integer还是Float类型都可以加入到List中，因为它们都是Number类型

如果既用作“读”，又用作“写”操作，不限定泛型。List< E >

#### 建议97：警惕泛型是不能协变和逆变的

协变是用一个窄类型替换宽类型，逆变是用宽类型覆盖窄类型。

泛型不支持协变也不支持逆变，可以通过**super和extend**关键字模拟实现

List< Number > ln =new ArrayList < Integer >(); 编译不通过

List<? extends Number> ln = new ArrayList< Integer >(); 模拟协变

List<? super Integer> li = new ArrayList< Number > (); 模拟逆变

#### 建议98：采用的顺序是List< T >、List< ? >、List< Object >

三者都可以容纳所有的对象，但顺序应该是List< T >，次之List< ? >，最后List< Object >，原因如下

1. List< T >是确定的某个类型
2. List< T >可以进行读写操作，List< ? >是只读类型，不能进行增加、修改操作；List< Object >也是可以读写操作，但是执行写入操作时需要向上转型，读操作后需要向下转型。

#### 建议99：严格限定泛型类型采用多重界限

Java的泛型中可以使用&符号关联多个上界并实现多个边界限定，而且只有上界有此限定。< T extends Staff & Passenger>

#### 建议101：注意Class类的特殊性

Java把源文件编译成class字节码文件，然后通过ClassLoader把这些类文件加载到内存中，最后生成实例执行。其中，Java使用一个元类（MetaClass）来描述加载到内存中的类数据，即Class类，它是一个描述类的类对象。

- Class类无构造函数
- Class类可以描述基本类型，如int.class
- 其对象是单例模式，一个类只有一个Class实例对象

Class类是Java的反射入口，只有在获得了一个类的描述对象后才能动态地加载、调用，获得Class对象有三种途径：

- 类属性，String.class
- 对象的getClass方法，new String().getClass()
- forName方法，Class.forName("java.lang.String")

#### 建议102：适时选择getDeclaredXXX和getXXX

Java的Class类提供了很多getDeclaredXXX和getXXX方法，如getDeclaredMethod和getMethod成对出现，getDeclaredConstructors和getConstructors成对出现

区别是：getMethod获得所有public基本的方法，包括从父类继承的方法；getDeclaredMethod获得自身类的所有方法，包括public、private，不受访问限制

#### 建议103：反射访问属性或方法时将Accessible设置为true

Java通过反射执行一个方法过程如下：获取一个方法对象，根据isAccessible确定是否能够执行，如果返回false则setAccessible(true)，最后再调用invoke方法。

Accessible属性只是用来判断是否要进行安全监测的，如果不安全检查可以大幅提升系统性能。

#### 建议104：使用forName动态加载类文件

动态加载指程序运行时加载需要的类库文件，因为不知道生成的实例对象是什么类型，而且方法和属性都不可访问，所以使用forName动态加载。

forName只是把一个类加载到内存中，并不保证由此产生一个实例对象，也不会执行任何方法。但是加载类机制决定要初始化该类的static变量、代码块。

```java
//加载驱动
Class.forName("com.mysql.jdbc.Driver");
//Driver源码
public class Driver extends NonRegisteringDriver implements java.sql.Driver{
  //静态代码块
  static{
    try{
      java.sql.DriverManager.registerDriver(new Driver());
    }catch...
  }
}
```

#### 建议105：动态加载不适合数组

通过反射操作数组时使用Array类，不要采用通用的反射处理API

```java
//动态创建数组
String[] strs = (String[]) Array.newInstance(String.class, 8);
```

#### 建议106：动态代理可以使代理模式更加灵活

```java
//静态代理
//主题接口
interface Subject{
  public void request();
}
class RealSubject implements Subject{
  public void request(){//实现
  }
}
//代理主题角色
class Proxy implements Subject{
  private Subject subject = null;
  public Proxy(){
    subject = new RealSubject();
  }
  public Proxy(Subject _subject){
    subject = _subject;
}
  public void request(){
    before();
    subject.request();
    after();
  }
  private void before(){}
  private void after(){}
}
```

通过java.lang.reflect.Proxy实现动态代理：只要提供一个抽象接口和具体主题角色，就可以动态实现其逻辑。

```java
//动态代理
//主题接口
interface Subject{
  public void request();
}
class RealSubject implements Subject{
  public void request(){//实现
  }
}
class SubjectHandler implements InvocationHandler{
  private Subject subject;
  public SubjectHandler(Subject _subject){
    subject = _subject;
  }
  //委托处理方法
  @Override
  public Object invoke(Object proxy, Method, Object[] args)throws Throwable{
    //预处理
    Object obj = method.invoke(subject, args);
    //后处理
    return obj;
}
}
```

动态代理是根据被代理的接口生成所有方法的，而所有方法都是由Handler进行处理的。

使用场景：

```java
Subject subject = new RealSuject();
InvocationHandler handler = new SubjectHandler(subject);
//当前加载器
ClassLoader cl = subject.getClass().getClassLoader();
//动态代理
Subject proxy = (Subject)Proxy.newProxyInstance(cl, subject.getClass().getInterfaces(), handler);
proxy.request();
```

AOP 就是通过动态代理实现的

#### 建议107：使用反射增加装饰模式的普适性

装饰模式：动态地给一个对象添加一些额外的职责

```java
//某种能力
interface Feature{
  public  void load();
}
//飞行能力
class FlyFeature implements Feature{
  public void load(){}
}
//钻地能力
class DigFeature implements Feature{
  public void load(){}
}
//包装动作类
class DecorateAnimal implements Animal{
  //被包装的动物
  private Animal animal;
  //使用哪一个包装器
  private Class<? extends Feature> clz;
  public DecorateAnimal(Animal _animal, Class<? extends Feature> _clz){
    animal = _animal;
    clz = _clz;
  }
  @Override
  public void doStuff(){
    InvocationHandler handler = new InvocationHandler(){
      //具体包装行为
      public Object invoke(Object p, Method m, Object[] args)throws Throwable{
        Object obj = null;
        //设置包装条件
        if(Modifier.isPublic(m.getModifiers())){
          obj = m.invoke(clz.newInstance(), args);
        }
        animal.doStuff();
        return obj;
      }
    };
    //当前加载器
    ClassLoader cl = getClass.getClassLoader();
    //动态代理
    Feature proxy = (Feature)Proxy.newProxyInstance(c1, clz.getInstance(), handler);
    proxy.load();
  }
}
```

调用代码

```java
Animal jerry = new Rat();
jerry = new DecorateAnimal(jerry, FlyFeature.class);
jerry = new DecorateAnimal(jerry, DigFeature.class);
jerry.doStuff();
```

#### 建议108：反射让模板方法模式更强大

模板方法模式：定义一个操作中的算法骨架，将一些步骤延迟到子类中，使得子类不改变算法结构就可以冲定义该算法某些步骤。

```java
public abstract class AbsPopulator{
  //模板方法
  public final void dataInitialing() throws Exception{
    Method[] methods = getClass.getMethods();
    for(Method m:methods){
      if(isInitDataMethod(m)){
        m.invoke(this);
      }
    }
  }
  //判断是否是数据初始化方法
  private boolean isInitDataMethod(Method m){
    return m.getName().startsWith("init") //init开始
      && Modifier.isPublic(m.getModifiers()) //公开方法
      &&m.getReturnType().equals(Void.TYPE) //返回void
      && !m.isVarArgs()  //输入参数为空
      && !Modifier.isAbstract(m.getModifiers()); //不能是抽象方法
}
}
```

JUnit4之前，要求方法名以test开头、无返回值、无参数、public修饰，实现原理相似。

#### 建议109：不需要太多关注反射效率

反射效率相对于正常代码确实低很多，但是它是一个非常有效的运行期工具类。很少有项目是因为反射问题引起系统效率故障。





参考：编写高质量代码——改善Java程序的151个建议



















