---
layout:     post
title:      "深入理解JVM之自动内存管理"
subtitle:   "内存管理、内存分配、垃圾收集器、性能与调优"
date:       2016-07-06 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Java
    - JVM
---

## 第二章 Java内存区域与内存溢出异常

#### 运行时数据区域

![](http://dl.iteye.com/upload/picture/pic/115264/4991b17e-a8b4-3d0a-a316-4651bb23da5e.png)

- **程序计数器：**较小的内存空间，指向当前线程所执行的字节码，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器完成。同一时刻，一个处理器（多核处理器的一个内核）只会执行一条线程中的指令，各条线程之间的计数器互不影响，独立存储，这类内存成为`线程私有`的内存。如果线程正在执行Java方法，计数器记录的是正在执行的虚拟机字节码指令的地址；如果正在执行的是Native方法，这个计数器则为空（Undefined），程序计数器是唯一一个在JVM中没有规定OutOfMemoryError情况的区域。
- **Java虚拟机栈：**VM Stack也是**线程私有**的，它的生命周期与线程相同。虚拟机栈描述Java方法执行的内存模型：每个方法在执行的同时都会创建一个栈帧（Stack Frame）用于存储局部变量表、操作数栈、动态链接、方法出口等信息。一个方法从调用到执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。**局部变量表**存放了编译期可知的基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference）和return Address类型。其中64位长度的long和double类型会占用2个局部变量空间（Slot），其余占用1个。局部变量表在编译期间完成分配，运行期间大小不会改变。在JVM规范中规定了两种异常：线程请求栈深度大于允许深度，抛出StackOverflowError异常；如果虚拟机栈扩展时无法申请到足够的内存，抛出OutOfMemoryError异常
- **本地方法栈：**Native Method Stack和虚拟机栈发挥所用相似，本地方法栈为虚拟机使用Native方法服务，对使用的语言、方式、数据结构没有强制规定，由虚拟机自由实现。Sun HotSpot虚拟机直接就把本地方法栈和虚拟机栈合二为一。本地方法栈也会抛出StackOverflowError和OutOfMemoryError异常。
- **Java堆：**Heap是JVM内存中最大的一块，被**所有线程共享**，在虚拟机启动时创建，用于存放对象实例。Java堆是垃圾收集器管理的主要区域，由于现在收集器基本都采用分代收集算法，还可以细分为：`新生代`和`老年代`。再细点有`Eden空间`、`From Survivor空间`、`To Survivor空间`等。根据JVM规范，Java堆可以处在物理不连续的内存空间，只要逻辑连续即可，主流虚拟机的堆都是可扩展的（通过-Xmx和-Xms控制），如果堆无法扩展会抛出OutOfMemoryError异常。
- **方法区：**Method Area与Java堆一样，是各个线程共享的内存区域，用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。JVM虚拟机规范把方法区描述为堆的一个逻辑部分，但它有一个别名叫做Non-Heap（非堆），目的应该是与Java堆区分开。HotSpot虚拟机上方法区为`永久代(Permanent Generation)`，可以使用GC分代垃圾回收。但是使用永久代实现方法区容易遇到内存溢出问题（永久代有-XX：MaxPermSize的上限）。对于其他虚拟机（如IBM J9）不存在永久代的概念，在JDK1.7的HotSpot中，也把原本放在永久代的字符串常量池移出。当方法区无法满足内存分配需求时，将抛出OutOfMemoryError异常。
- **运行时常量池：**Runtime Constant Pool是方法区的一部分，用于存放编译期生成的各种字面量和符号引用，JVM规范对于运行时常量池没有细节要求，除了保存Class文件中描述的符号引用外，还会把翻译出来的直接引用也存储在运行时常量池。运行时常量池相对于Class文件常量池的另一个特征是具备动态性，运行期间也可能将新的常量放入池中，这种特性对多的是String类的intern方法。
- **直接内存：**Direct Memory并不是虚拟机运行时数据区的一部分，但也被频繁使用，而且也可能导致OutOfMemoryError异常出现。在NIO类中，引入了一种基于通道（Channel）与缓冲区（Buffer）的I/O方式，它意义使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作，这样能在一些场景中显著提高性能，因为避免了在Java堆和Native堆中来回复制数据。直接内存不会受到Java堆限制，但是会受到本机总内存（RAM以及SWAP区）大小以及处理器寻址空间的限制。

#### HotSpot虚拟机对象探秘

**对象的创建**

虚拟机遇到一条new指令时，首先将检查这个指令的参数能够在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已经被加载、解析和初始化过。如果没有，那必须先执行相应的类加载过程，后面会介绍。

对象所需内存的大小在类加载完成后便可完全确定，然后就是从Java堆中划分出一块确定大小的内存。如果Java堆内存是绝对规整的，所有用过的内存放在一边，中间指针作为分界点，那么分配内存就是把指针挪动与对象大小相等的距离，这种方式成为`指针碰撞（Bump the Pointer）`。如果Java堆不是规整的，就必须维护一个`空闲列表（Free List）`，记录哪些内存是可用的。选择哪种分配方式有Java堆是否规整决定，又由垃圾收集器是否带有压缩整理功能决定。因此，在使用Serial、ParNew等带压缩过程的收集器时，系统采用指针碰撞，而使用CMS种种基于Mark-Sweep（标记-整理）收集器时，通常采用空闲列表。

对于并发分配内存的情况有两种方案，一种是对分配内存空间的动作进行同步处理——实际上虚拟机采用CAS配上失败重试的方法保证更新操作的原子性；另一种是把内存分配的动作按照线程划分在不同空间进行，成为本地线程分配缓冲（Thread Local Allocation Buffer， TLAB）。线程在自己的TLAB上分配内存，只有TLAB用完并分配新的TLAB时，才需要同步锁定，虚拟机是否使用TLAB，可以通过-XX:+/-UseTLAB参数来设定。内存分配完成后，虚拟机需要初始化零值（不包括对象头），保证对象的实例字段在不赋初值就可以直接使用。

接下来，虚拟机要对对象进行设置，例如对象是哪个类的实例、类的元数据信息、哈希码、GC分代年龄等，存放在对象头（Object Header）中。执行new执行后会接着执行< init >方法，把对象按照程序员的意愿进行初始化，这样一个真正可用的对象才完全产生出来。

**对象的内存布局**

在Hotspot虚拟中，对象在内存可分为3块区域：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）。

- 对象头（Header）：Hotspot虚拟机对象头包括两部分信息，第一部分用于存储对象自身的运行时数据，如哈希码、GC分代年龄、锁状态标识、线程持有的锁、偏向线程ID、偏向时间戳等，这部分数据的长度在32位和64位的虚拟机中分别为32bit和64bit，官方成为“Mark Word”。第二部分是类型指针，即指向它的元数据的指针，通过这个指针来确定这个对象是哪个类的实例。
- **实例数据**部分是对象真正存储的有效信息，也是程序代码中所定义的各种类型的字段内容，无论是从父类继承还是子类定义的，都要记录。这部分存储顺序会受到虚拟机分配策略参数和字段在Java源码中定义顺序的影响，HotSpot默认分配策略为longs/doubles、ints、shots/chars、bytes/booleans、oops（Ordinary Object Pointer）相同宽度的字段总是被分配到一起。在满足这个前提条件下，在父类中定义的变量会出现在子类之前。如果CompactFields参数为true（默认false），那么子类中较窄的变量也可能会插入到父类变量的空隙之中。
- 对齐填充并不是必然存在的，也没有特别的含义，仅仅起到占位符作用。由于HotSpot VM要求对象起始地址必须是8字节的整数倍，当对象实例没有对齐时，就需要通过对齐填充来补全。

**对象的访问定位**

Java程序需要通过栈上的reference数据来操作堆上的具体对象。而对象的访问方式取决于虚拟机实现而定的，目前主流的访问方式有使用句柄和直接指针两种。

![](http://static.open-open.com/lib/uploadImg/20120929/20120929194851_755.jpg)

- 如果使用句柄访问，Java堆中会划分出句柄池，reference存储句柄地址，而句柄中包含了对象实例数据与类型数据各自的具体地址信息。

![](http://static.open-open.com/lib/uploadImg/20120929/20120929194851_292.jpg)

- 如果使用直接指针访问，那么Java堆对象的布局中就必须考虑如何放置访问类型数据的相关信息，而reference中存储的直接就是对象地址。

使用句柄访问方式最大好处就是reference中存储的是稳定的句柄地址，在对象移动时只需要改变句柄中的实例数据指针，而reference不需要改变。使用指针访问方式最大好处就是速度快，它节省了一次指针定位的时间开销，就Sun HotSpot虚拟机而言，它使用的是第二种方式 (直接指针访问)。

#### 实战：OutOfMemoryError异常

**1、Java堆溢出**

只要不断地创建对象，并且保证GC Roots到对象之间有可达路径来避免垃圾回收机制清除这些对象，当达到堆容量限制后就会产生内存溢出异常。

VM的配置：-Xms20m（堆的最小值20MB），-Xmx（堆的最大值20MB），-XX：+HeapDumpOnOutOfMemoryError（让虚拟机在出现内存溢出异常时Dump出当前的内存堆转储快照以便事后分析）

```java
public class HeapOOM{
  static class OOMObject{}
  public static void main(String[] args){
    List<OOMObject> list = new ArrayList<OOMObject>();
    while(true){
      list.add(new OOMObject());
    }
  }
}
//运行结果
java.lang.OutOfMemoryError:Java Heap space
```

要解决一个区域的异常，一般先通过内存映像分析工具对Dump出来的堆转储快照进行分析，确认是内存泄漏（Memory Leak）还是内存溢出（Memory Overflow）。如果是内存泄漏，可通过工具查看泄漏对象到GC Roots的引用链，定位泄漏代码位置。如果是内存溢出，应当检测堆参数（-Xmx和-Xms）从代码上检查是否存在某些对象生命周期过长、持有状态时间过长的情况，尝试减少程序运行期的内存消耗。

**2、虚拟机栈和本地方法栈溢出**

HotSpot不区分虚拟机栈和本地方法栈，因此-Xoss（设置本地方法栈大小）是无效的，栈容量只由-Xss参数设定。

```java
//VM ： -Xss128k
public class JavaVMStackSOF{
  private int stackLength = 1;
  public void stacLeak(){
    stackLength ++;
    stackLeak();
  }
  public static void main(String[] args)throws Throwable{
    JavaVMStackSOF oom = new JavaVMStackSOF();
    try{
      oom.stackLeak();
    }catch(Throwable e){
      System.out.pringln("stck length" + oom.stackLength);
      throw e;
    }
  }
}
//结果
stack length:2402
Exception in thread "main" java.lang.StackOverflowError
```

实验表明在单线程下，无论由于栈帧太大还是虚拟机栈容量太小，当内存无法分配时都会抛出StackOverflowError异常。

如果不限于单线程，通过不断建立线程的方式会产生内存溢出异常，栈深度在大多数情况下达到1000-2000没有问题，发生StackOverflow异常可以于都错误堆栈，但是如果是建立过多线程导致的内存溢出，在不能减少线程数或更换64位虚拟机情况下，只能通过减少最大堆和减少栈容量来换取更多的线程。

```java
public class JavaVMStackOOM{
  private void dontStop(){
    while(true){}
  }
  public void stackLeakBythread(){
    while(true){
      Thread thread = new Thread(new Runnable(){
      public void run(){
        dontStop();
      }
    });
      thread.start();
    }
  }
  public static void main(String[] args)throws Throwable{
    JavaVMStackOOM oom = new JavaVMStackOOM();
    oom.stackLeakByThread();
}
}
//运行结果
Exception in thread "main" java.lang.OutOfMemoryError: unable to create new native thread
```

**3、方法区和运行时常量池溢出**

HotSpot中常量池是方法区的一部分，都分配在永久代内。常量池可以使用String.intern()方法将新的String对象添加到常量池，产生溢出。

VM 配置：-XX：PermSize=10M -XX：MaxPermSize=10M

```java
public static void main(String[] args){
  //使用List保存引用，避免Full GC回收常量池
  List<String> list = new ArrayList<String>();
  // 10MB的PermSize在integer范围内足够产生OOM了
  int i=0;
  while(true){
    list.add(String.valueOf(i++).intern());
}
}
//运行结果
Java.lang.OutOfMemoryError: PermGen space
```

方法区用于存放Class相关信息，基本思路是产生大量的类去填满方法区直到溢出。通过CGLib字节码技术，在运行时生成大量动态类，产生溢出。

```java
static class OOMObject{}
public static void main(String[] args){
  while(true){
    Enhancer enhance = new Enhancer();
    enhancer.setSuperclass(OOMObject.class);
    enhancer.setUseCache(false);
    enhancer.setCallback(new MethodInterceptor(){
      public Object intercept(Object obj, Method method, Object[] args, MehtodProxy proxy)throws Throwable{
        return proxy.invokeSuper(obj, args);
}
});
}
}
//运行结果
Java.lang.OutOfMemoryError: PermGen space
```

方法区溢出也是一种常见的内存溢出异常，一个类要被垃圾收集器回收掉，判定条件是比较苛刻的。

**4、本机直接内存溢出**

DirectMemory容量可通过-XX：MaxDirectMemorySize指定，如果不指定则默认与Java堆最大值一样。

```java
private static final int _1MB = 1024 * 1024;
public static void main(String[] args){
  Filed unsafeField = Unsafe.class.getDeclaredFields()[0];
  unsafeField.setAccessible(true);
  Unsafe unsafe = (Unsafe)unsafeFiled.get(null);
  while(true){
    unsafe.allocateMemory(_1MB);
  }
}
```

由于DirectMemory导致的内存溢出，在Heap Dump文件看不见明显异常，而程序有直接或间接使用了NIO，可以考虑是否发生本地直接内存异常。

## 第三章 垃圾收集器与内存分配策略

垃圾收集（Garbage Collection， GC）第一次使用是1960年诞生于MIT的Lisp语言。GC主要完成3件事情，哪些内存需要回收？什么时候回收？如何回收？

由于程序计数器、虚拟机栈、本地方法栈3个区域随线程而生而灭，Java垃圾收集器主要关注Java堆和方法区，动态共享的内存。

#### 对象已死吗

**1、引用计数法（Reference Counting）**

给对象添加一个引用计数器，每新增一个引用计数加一，当引用失效时计数减一，任何时刻计数器为0的对象就是不可能再被使用的。

该算法实现简单，判定效率高，但是主要问题是很难解决对象之间相互循环引用的问题。

**2、可达性分析算法（Reachability Analysis）**

该算法基本思想是通过一系列成为`GC Roots`的对象作为起始点，向下搜索，所走过的路径成为引用链，当一个对象到GC Roots没有任何引用链相连时，则证明此对象不可用。

在Java语言中，可作为GC Roots的对象包括下面几种：

- 虚拟机栈（栈帧中的本地变量表）中引用的对象
- 方法区中类静态属性引用的对象
- 方法区常量引用的对象
- 本地方法栈中JNI引用的对象

**3、再谈引用**

- 强引用：指在程序代码中普遍存在的，类似“Object obj = new Object()”，只要强引用还存在，垃圾收集器永远不会回收掉被引用的对象
- 软引用：用来描述一些还有用但并非必须的对象，对于软引用关联的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围进行第二次回收。JDK1.2之后提供了SoftReference类来实现软引用
- 弱引用：用来描述非必须对象，强度比软引用更弱，被弱引用关联的对象只能生存到下一次垃圾收集发生前。在JDK1.2后提供了WeakReference类来实现弱引用。
- 虚引用：也成为幽灵引用或幻影引用，是最弱的一种引用关系。为一个对象设置虚引用唯一目的就是能在这个对象被收集器回收时收到一个系统通知。JDK1.2后提供PhantomReference类实现虚引用

**4、生存还是死亡**

对象死亡至少经历两次标记过程：

- 可达性分析发现没有与GC Roots的引用链，**第一次**标记并筛选是否必要执行finalize()方法，筛选的条件是对象是否覆盖了finalize方法或是否已经被调用过
- 如果有必要执行finalize()方法，则将对象放置在`F-Queue`队列中，并在稍后由一个由虚拟机自动建立的、低优先级的Finalizer线程去执行它。finalize()方法中对象可以拯救自己——重新与引用链上的任何一个对象建立关联即可。稍后GC将对F-Queue中的对象进行**第二次**小规模的标记，此时标记的对象基本就真的被回收了。

finalize()方法应尽量避免使用它，运行代价高昂，不确定性大，无法保证各个对象的调用顺序。finalize()所能做的所有工作，使用try-finally或者其他方式都可以做得更好、更及时。

**5、回收方法区**

永久代的垃圾收集主要包括两部分内容：废弃常量和无用的类。判断常量回收比较简单，只要没有引用即可，但是判断一个类是否是`无用的类`条件则苛刻许多：

- 该类所有的实例都已经被回收
- 加载该类的ClassLoader已经被回收
- 该类对应的java.lang.Class对象没有在任何地方引用，无法在任何地方通过反射访问该类的方法

虚拟机可以对上述3个条件的无用类进行回收，但不是必然回收。HotSpot提供了-Xnoclassgc参数空着，还可以使用-verboss:class以及-XX:+TraceClassLoading、-XX:TraceClassUnLoading查看加载和卸载信息。在大量使用反射、动态代理、CGLib等ByteCodeigniter框架、动态JSP、自定义ClassLoader场景中都需要虚拟机提供类卸载的功能，以保证永久代不会溢出。

#### 垃圾收集算法

**1、标记-清除算法**

Mark-Sweep算法，首先标记所有要回收的对象，标记完成后统一回收。有两个不足：一是效率问题，标记和清除过程效率不高；二是空间问题，标记清除之后会产生大量不连续的内存碎片，导致以后在程序运行需要分配较大对象时，无法找到足够的连续内存而不得不提前出发另一次垃圾收集动作。

![](http://static.oschina.net/uploads/img/201303/18092408_nT2F.jpg)

**2、复制算法**

复制收集算法将内存分成两块，每次使用一块，当这块用完了就将存活的对象复制到另一块上面，然后把使用过的内存空间清理掉。但是这种算法太耗费空间，将内存缩小为原来的一半。

![](http://static.oschina.net/uploads/img/201303/18092409_AaxY.jpg)

现在商业虚拟机都采用这种收集算法来**回收新生代**，新生代中的对象98%是“朝生夕死”的，并不需要1：1划分内存。而是分为一块较大的Eden空间和两块较小的Survivor空间。HotSpot默认Eden和Survivor比例为8：1，则每次新生代可用空间为90%，“浪费”了10%的空间。当Survivor空间不够用时，需要依赖老年代进行`分配担保`。

3、标记-整理算法

复制收集算法在对象存活率较高时要进行较多复制操作，效率降低，还要分配额外空间进行分配担保，所以在老年代一般不能直接蒜用这种算法。根据老年代特点，提出标记-整理（Mark-Compact）算法，最后将存活对象都向一端移动，减少碎片。

![](http://static.oschina.net/uploads/img/201303/18092410_aV8b.jpg)

**4、分代收集算法**

当代商业虚拟机都采用`分代收集（Generational Collection）`算法，其思路是根据对象存活周期将内存划分为新生代和老年代，新生代采用`复制算法`，老年代采用`标记清理`或者`标记-整理`算法。

#### HotSpot的算法实现

**1、枚举根节点**

可达性分析中的问题：应用方法区数百兆，遍历引用耗时；分析必须确保一致性，导致GC进行时必须停顿所有Java执行线程。

目前主流Java虚拟机都是准确式GC，当执行系统停顿下来后，不需要一个不漏地检查所有引用，JVM应当知道哪些地方存放对象引用。在HotSpot实现中，使用一组成为`OopMap`的数据结构来达到这个目的。在类加载完成时，HotSpot就把对象内什么偏移量上是什么类型的数据计算出来，在JIT编译过程中，也会在特定位置记录下栈和寄存器中哪些位置是引用，这样GC在扫描时就可以直接得知这些信息了。

**2、安全点**

导致引用关系变化的指令非常多，如果都要记录在OopMap中会需要大量额外空间。HotSpot只是在`安全点（Safepoint）`记录这些信息，即程序只能在安全点时才能暂停并GC。安全点的选定以程序“是否具有让程序长时间执行的特征”为标准，最明显特征就是指令序列复用，例如方法调用、循环跳转、异常跳转等，所以具有这些功能的指令才会产生Safepoint。

GC发生时要让所有线程运行到安全点停顿下来，有两种方案：`抢先式中断`，GC发生时首先全部中断，如果有不在安全点则恢复线程，让它跑到安全点，现在几乎没有JVM采用抢先式中断；`主动式中断`，简单设置一个标志，各个线程主动轮询这个标志，发现中断标志为真时自己中断挂起，轮询标志的地方和安全点是重合的。

**3、安全区域**

安全点解决不了程序“不执行”的时候，线程处于Sleep或者Blocked状态，不会自动挂起，这是就需要`安全区域（Safe Region）`。

安全区域是指在一段代码中，引用关系不会发生变化。当线程执行到Safe Region中代码时，首先标识自己进入Safe Region，当发生GC时，就不用管标识为Safe Region状态的线程了，在线程要离开Safe Region时，要检查系统是否完成根节点枚举，如果完成线程继续执行，否则它就要等待直到收到可以安全离开Safe Region的信号为止。

#### 垃圾收集器

不同厂商、不同版本的虚拟机提供的垃圾收集器可能差别很大，这里讨论的收集器基于JDK1.7Update14之后的HotSpot虚拟机。

![](http://img.my.csdn.net/uploads/201210/03/1349278110_8410.jpg)

**1、Serial收集器**

- 缺点：serial收集器是一个单线程的收集器，但它的“单线程”的意义并不仅仅说明它只会使用一个CPU或一条收集线程去完成来及收集工作，更重要的是在它进行垃圾收集时，必须暂停其他所有的工作线程，直到它收集结束。
- 优点：简单而高效，对于限定单个CPU的环境来说，Serial收集器由于没有线程交互的开销，专心做垃圾收集可以获得最高的单线程收集效率。因此Serial收集器对于运行在Client模式(默认Serial收集器)下的虚拟机来说是一个很好的选择。

![](http://7xkjk9.com1.z0.glb.clouddn.com/jvm-9.jpg)

**2、ParNew收集器**

- 优点：PerNew收集器是Serial收集器的多线程版本。它是除类Serial收集外，目前只能与CMS收集器配合工作。
- 缺点：在单CPU的环境下绝对不会比Serial收集器有更好的效果，甚至由于存在线程交互的开销，该收集器在通过超线程技术实现两个CPU的环境中都不能保证比Serial收集器好。但随着CPU的增加，它的优点就越明显。
- 它默认开启的线程数与CPU的数量相同，可以使用-XX：ParallelGCThreads参数来限制垃圾收集的线程数。

![](http://7xkjk9.com1.z0.glb.clouddn.com/jvm-10.png)

并行（Parallel）：指多条垃圾收集线程并行工作，但此时用户线程仍然处于等待状态；

并发（Concurrent）：指用户线程与垃圾收集线程同时执行，用户程序在继续运行，而垃圾收集程序运行于另一个CPU上。

**3、Parallel Scavenge收集器**

Parallel Scavenge收集器使用复制算法收集器，又是并行的多线程收集器。Parallel Scavenge收集器的目标是达到一个可控制的吞吐量（吞吐量=运行用户代码时间/（运行用户代码时间+垃圾收集时间））。Parallel Scavenge收集器提供了两个参数用于精准控制吞吐量，分别是控制最大垃圾收集停顿时间的-XX:MaxGCPauseMillis参数(x>0的毫秒数)以及直接设置吞吐量大小的-XX:GCTimeRatio参数(设置一个0<x<100的整数，表示垃圾收集器时间占总时间的比率，相当于吞吐量的倒数)。

![](http://7xkjk9.com1.z0.glb.clouddn.com/jvm-11.jpg)

**4、Serial Old收集器**

Serial Old是Serial收集器的老年代版本，统一是一个单线程收集器，使用“标记-整理”算法，主要意义在于给Client模式下的虚拟机使用。

**5、Parallel Old收集器**

Parallel Old是Parallel Scavenge收集器的老年代版本，使用多线程和“标记-整理”算法。

**6、CMS收集器**

CMS（Concurrent Mark Sweep）收集器是一种以获取最短停顿时间为目标的收集器。它是基于“标记-清除”算法实现的，整个过程分为4个步骤：

- 初始标记(CMS initial mark)：需要用户线程停顿。标记一下GC Roots能直接关联到的对象，
- 并发标记(CMS concurrent mark)：进行GC Roots Tracing的过程。
- 重新标记(CMS remark):需要用户线程停顿，是为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段稍长一些，但比并发标记要短。
- 并发清除(CMS  concurrent sweep)：与用户线程一起工作

![](http://7xkjk9.com1.z0.glb.clouddn.com/jvm-12.jpg)

缺点：

- CMS收集器对CPU资源非常敏感.CMS默认启动的回收线程数是（CPU数量+3）/4，在并发阶段，它虽然不会导致用户线程停顿，但是会占用一部分线程而导致应用程序变慢，总吞吐量对降低。
- CMS收集器无法处理浮动垃圾，可能出现“Concurrent Mode Failure”失败而导致另一次Full Gc的产生。也是由于在垃圾收集阶段用户线程还需要运行，那么久还需要预留有足够的内存空间给用户线程使用，因此CMS收集器不能像其他收集器那样等到老年代几乎完全被填满了再收集，需要预留一部分空间提供并发收集时的程序运作使用。可以使用-XX:CMSInitiatingOccupancyFraction的值来设置触发百分比。
- CMS是一款基于“标记-清除”算法实现的收集器，这意味着收集结束时会有大量的空间碎片产生。空间碎片过多将会给大对象带来麻烦，往往会出现老年代还有很大剩余空间，但是无法找到足够大的连续空间来分配当前对象，不得不提前触发一次full Gc。CMS提供了-XX:+UseCMSCompactAtFullCollection开关参数(默认是开的)，用户在CMS收集器顶不要进行Full GC时开启内存碎片的合并整理过程，内存整理的过程是无法并发的，空间碎片的问题没有了，但停顿的时间会变长；提供-XX:CMSFullGCBeforeCompaction,这个参数用户设置多少次不压缩Full GC后，跟着来一次带压缩的碎片整理，默认为0。

**7、G1收集器**

G1收集器是一款面向服务端应用的垃圾收集器，成熟版基于JDK1.7，与其他收集器相比，它以下特点。

- 并行与并发：G1能充分利用多CPU、多核环境下的硬件优势，使用多个CPU来缩短Stop-The-World停顿时间。
- 分代收集：G1收集器可收集新生代与老年代两种，不需要其他收集器配合就可以独立管理整个GC堆。
- 空间整合：G1采用“标记-整理”算法实现收集器，着意味着G1运作期间不会产生内存空间碎片，收集后可提供规整的可用内存。
- 可预测的停顿：建立可预测的停顿时间模型，能让使用者明确指定在一个长度为M毫秒的时间片段内，消耗在垃圾收集器上的时间不得超过N毫秒。

![](http://7xkjk9.com1.z0.glb.clouddn.com/jvm-13.jpg)

8、垃圾收集器参数总结

| 参数                     | 描述                                       |
| ---------------------- | ---------------------------------------- |
| UseSerialGC            | 虚拟机运行在Client模式下的默认值，打开此类开关后，使用Serial+Serial Old的收集器组合进行内存回收 |
| UseParNewGC            | 打开此类开关后，使用ParNew + Serial Old的收集器组合进行内存回收 |
| UseConcMarkSweepGC     | 打开此类开关后，使用ParNew + CMS + Serial Old的收集器组合进行内存回收。Serial Old收集器将作为CMS收集器出现Concurrent Mode Failure失败后的后备收集器使用 |
| UseParallelGC          | 虚拟机运行在Server模式下的默认值，打开此开关后，使用Parallel Scavenge +Serial Old的收集器组合进行内存回收 |
| UseParallelOldGC       | 打开此开关后，使用Parallel Scavenge +Parallel Old的收集器组合进行内存回收 |
| SurvivorRatio          | 新生代中Eden区域与Survivor区域的容量比值，默认为8，代表Eden:Survivor = 8:1 |
| PretenureSizeThreshold | 直接晋升到老年代的对象大小，设置这个参数后，大于这个参数的对象将直接在老年代分配 |
| MaxTenuringThreshold   | 晋升到老年代的对象年龄。每个对象在坚持过一次Minor GC之后，年龄就增加1，当超过这个参数值时就进入老年代 |
| UseAdaptiveSizePolicy  | 动态调整java堆中各个区域的大小以及进入老年代的年龄              |

| HandlePromotionFailure         | 是否允许分配担保失败，即老年代的剩余空间不足以应付新生代整个Eden和Survivor区的所有对象存活的极端情况 |
| ------------------------------ | ---------------------------------------- |
| ParallelGTThreads              | 设置并行的GC进行内存回收的线程数                        |
| GCTimeRatio                    | GC时间占总时间的比率，默认值是99，即允许1%的GC时间，仅在使用Parallel Scavenge收集器时生效 |
| MaxGCPauseMillis               | 设置GC最大停顿时间，仅在使用Parallel Scavenge收集器时生效   |
| CMSInitiatingOccupancyFraction | 设置CMS收集器在老年代空间被使用多少后触发垃圾收集，默认值为68%，仅在使用CMS收集器时生效。 |
| UseCMSConpactAtFullCollection  | 设置CMS收集器在完成垃圾收集器后是否要进行一次内存碎片整理。仅在使用CMS收集器时生效。 |
| CMSFullGCsBeforeCommpaction    | 设置CMS收集器在进行若干次来及收集后再启动一次内存碎片整理。仅在使用CMS收集器时生效。 |

#### 内存分配与回收策略

对象的内存分配，主要在新生代Eden区上，如果启动了本地线程分配缓冲，将按线程优先在TLAB上分配，少数情况也可能会直接分配在老年代，下面讲解几条普遍的内存分配规则

**1、对象优先在Eden分配**

当Eden区没有足够空间进行分配时，虚拟机将发起一次Minor GC（新生代GC）。VM设置Java堆大小20MB，不可扩展，10MB分给新生代，10MB分给老年代。-XX:SurvivorRatio=8决定Eden区域Survivor区比例为8：1，所以新生代可用空间为9MB（Eden区加一个Survivor区）

```java
//VM 参数: -verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
private static final int _1MB = 1024 * 1024;
public static void testAllocation(){
  byte[] a1, a2, a3, a4;
  a1 = new byte[2 * _1MB];
  a2 = new byte[2 * _1MB];
  a3 = new byte[2 * _1MB];
  a4 = new byte[4 * _1MB]; //出现一次Minor GC，但没有回收对象，此时只好通过分配担保机制将3个2MB大小对象提前转移到老年代中。
}
```

**2、大对象直接进入老年代**

大对象指需要大量连续内存空间的Java对象，最典型的就是长字符串以及数组。-XX:PretenureSizeThreshold参数设置大于该值对象直接分配在老年代。

```java
// VM参数： -XX:PretenureSizeThreshold=3145728 (3MB)
private static final int _1MB = 1024 * 1024;
public static void testPretenureSizeThreshold(){
  byte[] allocation = new byte[4 * _1MB]; //直接分配在老年代中
}
```

**3、长期存活的对象将进入老年代**

虚拟机给每个对象定义一个年龄（Age）计数器，如果对象在Eden出生并进入Survivor区，则每“熬过”一次Minor GC，年龄就增加1岁，当增加到一定程度（默认15岁）就会晋升到老年代。该阈值可以通过-XX:MaxTenuringThreshold设置。

**4、动态对象年龄判定**

虚拟机并不必须要求对象年龄到达MaxTenuringThreshold才能晋升老年代，如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间一半，年龄大于或等于该年龄的对象就可以直接进入老年代，不用等到阈值年龄。

**5、空间分配担保**

发生Minor GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总和，如果成立则Minor GC是安全的，否则会查看HandlePromotionFailure是否允许担保失败。如果允许，则继续判断最大连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，则尝试进行Minor GC；如果小于或者不允许冒险，则改为Full GC。

分配担保的冒险就是老年代要担保有容纳存活对象的剩余空间。

## 第四章 虚拟机性能监控与故障处理工具

#### Sun JDK 监控和故障处理工具

- jps：JVM Process Status Tool，显示指定系统内所有的HotSpot虚拟机进程
- jstat：JVM Statistics Monitoring Tool，用于收集HotSpot虚拟机各方面的运行数据
- jinfo：Configuration Info for Java，显示虚拟机配置信息
- jmap：Memory Map for Java，生成虚拟机的内存转储快照（heapdump文件）
- jhat：JVM Heap Dump Browser，用于分析Heapdump文件，它会建立一个HTTP/HTML服务器，让用户可以在浏览器查看分析结果
- jstack：Stack Trace for Java，显示虚拟机的线程快照，Java堆栈跟踪工具

#### JDK的可视化工具

- JConsole：一种基于JMX的可视化监视、管理工具，主界面包括概述、内存、线程、类、VM摘要、MBean6个页签。
- VisualVM：多合一故障处理工具，除了运行监视、故障处理外，还提供了性能分析等功能。功能有：显示虚拟机进程以及进程的配置、环境信息（jps，jinfo）；监视应用程序的CPU、GC、堆、方法区、以及线程信息（jstat，jstack）；dump以及分析堆转储快照（jmap，jhat）；方法级的程序运行性能分析，找出被调用最多、运行时间最长的方法；离线程序快照：收集程序的运行时配置、线程dump、内存dump等信息建立一个快照。

## 第五章 调优案例分析与实战

#### 案例分析

1、高性能硬件上的程序部署策略

一个15万PV/天左右的在线文档类型网站最近更换了硬件系统，新硬件为4个CPU、16GB物理内存，64位操作系统，Rsin为服务器，64位的JDK1.5，通过-Xmx和-Xms将堆固定在12GB。

问题：文档序列化产生大对象，进入老年代，产生Full GC，停顿高达14秒，每隔十几分钟就会出现Full GC。

在高性能硬件上部署程序，目前主要有两种方式：

- 通过64位JDK来使用大内存
- 使用若干个32位虚拟机建立逻辑集群来利用硬件资源

使用64位JDK来管理大内存可能面临的问题：

- 内存回收导致长时间停顿
- 现阶段（？JDK1.8验证），64位JDK的性能测试结果普遍低于32位JDK
- 需要保证程序足够稳定，因为堆溢出产生十几GB的快照无法分析
- 相同程序在64位JDK中消耗的内存一般比32位的JDK大，这是由指针膨胀及数据类型对齐补白等因素导致的


32位解决方案，在一台物理机启动多个应用服务器进程，给每个服务器进程分配不同的端口，然后在前端搭建一个负载均衡器，以反向代理的方式来分配访问请求。可能面临的问题：

- 尽量避免节点竞争全局资源，如磁盘竞争，容易导致IO异常
- 很难最高效地利用某些资源池，如连接池，一般是各个节点建立独立的连接池
- 各个节点仍然不可避免受到32位的内存限制，在Windows平台只能支持2GB内存，Linux可以到3GB，但最高是4GB
- 大量使用本地缓存的应用，在逻辑集群中会造成较大的内存浪费，因为每个节点都有一份缓存，这时可以考虑把本地缓存改为集中式缓存

最终解决方案：

建立5个32位的逻辑集群，每个进程2GB内存，共10GB内存，另外建立一个Apache服务作为前端均衡代理。改为CMS收集器进行垃圾回收，提高响应速度。

2、集群间同步导致的内存溢出：网络传输不稳定，导致重发数据在内存堆积

3、堆外内存导致的溢出错误：除了Java堆和永久代之外，还应该注意其他的内存大小，Direct Memory、线程堆栈、Socket缓存区、JNI代码、虚拟机和GC

4、外部命令导致系统缓慢：fork系统调用时Linux用来产生新进程的，在Java虚拟机中，用户编写的Java代码最多只有线程的概念，不应当有进程的产生

5、服务器JVM进程崩溃：B/S的MIS系统，采用异步的方式使得等待的线程和Socket连接越来越多，超过虚拟机承受能力使得虚拟机进程崩溃。解决：将异步调用该为生产者/消费者模式的消息队列实现。

#### 实战：Eclipse运行速度调优

1、调优前的程序运行状态

2、升级JDK1.6的性能变化及兼容问题

3、编译时间和类加载时间的优化

4、调整内存设置控制垃圾回收频率

5、选择收集器降低延迟






参考：《深入理解Java虚拟机》