---
layout:     post
title:      "深入浅出Java Concurrency——原子操作"
subtitle:   "java.util.concurrent，原子操作"
date:       2016-08-05 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Java
    - Java并发编程
---

## J.U.C整体认识

java.util.concurrent（J.U.C），Java并发体系，内容大致包括：

- JUC的API：包括完整的类库结构和样例分析
- JUC的硬件原理以及软件思想
- JUC的误区和常见陷阱

JUC的完整API：

![](http://images.blogjava.net/blogjava_net/xylz/Windows-Live-Writer/-Java-Concurrency-1--J.U.C_9314/J.U.C_2.png)

## 原子操作

#### AtomicInteger

从相对简单的Atomic入手（java.util.concurrent是基于Queue的并发包，而Queue很多情况下使用到了Atomic）。通常情况下，在Java里面，++i或者--i不是线程安全的，这里面有三个独立的操作：或者变量当前值，为该值+1/-1，然后写回新的值。在没有额外资源可以利用的情况下，只能使用加锁才能保证读-改-写这三个操作时“原子性”的。

Doug Lea在未将[backport-util-concurrent](http://backport-jsr166.sourceforge.net/)合并到[JSR 166](http://jcp.org/en/jsr/detail?id=166)里面来之前，是采用纯Java实现的，于是不可避免的采用了synchronized关键字。

```java
public final synchronized void set(int newValue);

public final synchronized int getAndSet(int newValue);

public final synchronized int incrementAndGet();
```

同时在变量上使用了volatile （后面会具体来讲volatile到底是个什么东东）来保证get()的时候不用加锁。尽管synchronized的代价还是很高的，但是在没有JNI的手段下纯Java语言还是不能实现此操作的。

JSR 166提上日程后，backport-util-concurrent就合并到JDK 5.0里面了，在这里面重复使用了现代CPU的特性来降低锁的消耗。后本章的最后小结中会谈到这些原理和特性。在此之前先看看API的使用。

一切从java.util.concurrent.atomic.AtomicInteger开始。

```java
int addAndGet(int delta)
          //以原子方式将给定值与当前值相加。 实际上就是等于线程安全版本的i =i+delta操作。

boolean compareAndSet(int expect, int update)
         //如果当前值 == 预期值，则以原子方式将该值设置为给定的更新值。 如果成功就返回true，否则返回false，并且不修改原值。

int decrementAndGet()
          //以原子方式将当前值减 1。 相当于线程安全版本的--i操作。

int get()
          //获取当前值。

int getAndAdd(int delta)
          //以原子方式将给定值与当前值相加。 相当于线程安全版本的t=i;i+=delta;return t;操作。

int getAndDecrement()
          //以原子方式将当前值减 1。 相当于线程安全版本的i--操作。

int getAndIncrement()
         // 以原子方式将当前值加 1。 相当于线程安全版本的i++操作。

int getAndSet(int newValue)
          //以原子方式设置为给定值，并返回旧值。 相当于线程安全版本的t=i;i=newValue;return t;操作。

int incrementAndGet()
          //以原子方式将当前值加 1。 相当于线程安全版本的++i操作。 

void lazySet(int newValue)
          //最后设置为给定值。 延时设置变量值，这个等价于set()方法，但是由于字段是volatile类型的，因此次字段的修改会比普通字段（非volatile字段）有稍微的性能延时（尽管可以忽略），所以如果不是想立即读取设置的新值，允许在“后台”修改值，那么此方法就很有用。如果还是难以理解，这里就类似于启动一个后台线程如执行修改新值的任务，原线程就不等待修改结果立即返回（这种解释其实是不正确的，但是可以这么理解）。

void set(int newValue)
          //设置为给定值。 直接修改原始值，也就是i=newValue操作。

boolean weakCompareAndSet(int expect, int update)
          //如果当前值 == 预期值，则以原子方式将该设置为给定的更新值。JSR规范中说：以原子方式读取和有条件地写入变量但不 创建任何 happen-before 排序，因此不提供与除 weakCompareAndSet 目标外任何变量以前或后续读取或写入操作有关的任何保证。大意就是说调用weakCompareAndSet时并不能保证不存在happen-before的发生（也就是可能存在指令重排序导致此操作失败）。但是从Java源码来看，其实此方法并没有实现JSR规范的要求，最后效果和compareAndSet是等效的，都调用了unsafe.compareAndSwapInt()完成操作。
```

下面的代码是一个测试样例，为了省事就写在一个方法里面来了

```java
public class AtomicIntegerTest {

    @Test
    public void testAll() throws InterruptedException{
        final AtomicInteger value = new AtomicInteger(10);
        assertEquals(value.compareAndSet(1, 2), false);
        assertEquals(value.get(), 10);
        assertTrue(value.compareAndSet(10, 3));
        assertEquals(value.get(), 3);
        value.set(0);
        //
        assertEquals(value.incrementAndGet(), 1);
        assertEquals(value.getAndAdd(2),1);
        assertEquals(value.getAndSet(5),3);
        assertEquals(value.get(),5);
        //
        final int threadSize = 10;
        Thread[] ts = new Thread[threadSize];
        for (int i = 0; i < threadSize; i++) {
            ts[i] = new Thread() {
                public void run() {
                    value.incrementAndGet();
                }
            };
        }
        //
        for(Thread t:ts) {
            t.start();
        }
        for(Thread t:ts) {
            t.join();
        }
        //
        assertEquals(value.get(), 5+threadSize);
    }
}
```

#### 数组、引用的原子操作

**AtomicIntegerArray/AtomicLongArray/AtomicReferenceArray**的API类似，选择有代表性的AtomicIntegerArray来描述这些问题。

**int get(int i)**

获取位置 `i` 的当前值。很显然，由于这个是数组操作，就有索引越界的问题（IndexOutOfBoundsException异常）

对于下面的API起始和AtomicInteger是类似的，这种通过方法、参数的名称就能够得到函数意义的写法是非常值得称赞的。

```java
void set(int i, int newValue)
void lazySet(int i, int newValue)
int getAndSet(int i, int newValue)
boolean compareAndSet(int i, int expect, int update)
boolean weakCompareAndSet(int i, int expect, int update)
int getAndIncrement(int i)
int getAndDecrement(int i)
int getAndAdd(int i, int delta)
int incrementAndGet(int i)
int decrementAndGet(int i)
int addAndGet(int i, int delta)
```

现在关注字段的原子更新。

**AtomicIntegerFieldUpdater/AtomicLongFieldUpdater/AtomicReferenceFieldUpdater**是基于反射的原子更新字段的值。

相应的API也是非常简单的，但是也是有一些约束的。

（1）字段必须是volatile类型的！在后面的章节中会详细说明为什么必须是volatile，volatile到底是个什么东西。

（2）字段的描述类型（修饰符public/protected/default/private）是与调用者与操作对象字段的关系一致。也就是说调用者能够直接操作对象字段，那么就可以反射进行原子操作。但是对于父类的字段，子类是不能直接操作的，尽管子类可以访问父类的字段。

（3）只能是实例变量，不能是类变量，也就是说不能加static关键字。

（4）只能是可修改变量，不能使final变量，因为final的语义就是不可修改。实际上final的语义和volatile是有冲突的，这两个关键字不能同时存在。

（5）对于**AtomicIntegerFieldUpdater**和**AtomicLongFieldUpdater**只能修改int/long类型的字段，不能修改其包装类型（Integer/Long）。如果要修改包装类型就需要使用**AtomicReferenceFieldUpdater**。

在下面的例子中描述了操作的方法。

```java
public class AtomicIntegerFieldUpdaterDemo { 

   class DemoData{
       public volatile int value1 = 1;
       volatile int value2 = 2;
       protected volatile int value3 = 3;
       private volatile int value4 = 4;
   }
    AtomicIntegerFieldUpdater<DemoData> getUpdater(String fieldName) {
        return AtomicIntegerFieldUpdater.newUpdater(DemoData.class, fieldName);
    }
    void doit() {
        DemoData data = new DemoData();
        System.out.println("1 ==> "+getUpdater("value1").getAndSet(data, 10));
        System.out.println("3 ==> "+getUpdater("value2").incrementAndGet(data));
        System.out.println("2 ==> "+getUpdater("value3").decrementAndGet(data));
        System.out.println("true ==> "+getUpdater("value4").compareAndSet(data, 4, 5));
    }
    public static void main(String[] args) {
        AtomicIntegerFieldUpdaterDemo demo = new AtomicIntegerFieldUpdaterDemo();
        demo.doit();
    }
} 
```

在上面的例子中DemoData的字段value3/value4对于AtomicIntegerFieldUpdaterDemo类是不可见的，因此通过反射是不能直接修改其值的

**AtomicMarkableReference**类描述的一个< Object,Boolean >的对，可以原子的修改Object或者Boolean的值，这种数据结构在一些缓存或者状态描述中比较有用。这种结构在单个或者同时修改Object/Boolean的时候能够有效的提高吞吐量。

**AtomicStampedReference**类维护带有整数“标志”的对象引用，可以用原子方式对其进行更新。对比**AtomicMarkableReference**类的< Object,Boolean >，**AtomicStampedReference**维护的是一种类似< Object,int >的数据结构，其实就是对对象（引用）的一个并发计数。但是与**AtomicInteger**不同的是，此数据结构可以携带一个对象引用（Object），并且能够对此对象和计数同时进行原子操作。

在后面的章节中会提到“ABA问题”，而**AtomicMarkableReference/** **AtomicStampedReference**在解决“ABA问题”上很有用**。**

#### 指令重排与happens-before法则

Java并发编程实战中对线程安全的定义：

> **当多个线程访问一个类时，如果不用考虑这些线程在运行时环境下的调度和交替运行，并且不需要额外的同步及在调用方代码不必做其他的协调，这个类的行为仍然是正确的，那么这个类就是线程安全的**

显然只有资源竞争时才会导致线程不安全，因此**无状态对象永远是线程安全的**。

原子操作的描述是： 多个线程执行一个操作时，其中**任何一个线程要么完全执行完此操作，要么没有执行此操作的任何步骤**，那么这个操作就是原子的。

**指令重排序**

Java语言规范规定了JVM线程内部维持顺序化语义，也就是说只要程序的最终结果等同于它在严格的顺序化环境下的结果，那么指令的执行顺序就可能与代码的顺序不一致。这个过程通过叫做指令的重排序。指令重排序存在的意义在于：JVM能够根据处理器的特性（CPU的多级缓存系统、多核处理器等）适当的重新排序机器指令，使机器指令更符合CPU的执行特点，最大限度的发挥机器的性能。

程序执行最简单的模型是按照指令出现的顺序执行，这样就与执行指令的CPU无关，最大限度的保证了指令的可移植性。这个模型的专业术语叫做顺序化一致性模型。但是现代计算机体系和处理器架构都不保证这一点（因为人为的指定并不能总是保证符合CPU处理的特性）。

我们来看最经典的一个案例。

```java
public class ReorderingDemo { 

    static int x = 0, y = 0, a = 0, b = 0; 

    public static void main(String[] args) throws Exception { 

        for (int i = 0; i < 100; i++) {
            x=y=a=b=0;
            Thread one = new Thread() {
                public void run() {
                    a = 1;
                    x = b;
                }
            };
            Thread two = new Thread() {
                public void run() {
                    b = 1;
                    y = a;
                }
            };
            one.start();
            two.start();
            one.join();
            two.join();
            System.out.println(x + " " + y);
        }
    } 

}
```

在这个例子中one/two两个线程修改区x,y,a,b四个变量，在执行100次的情况下，可能得到(0 1)或者（1 0）或者（1 1）。事实上按照JVM的规范以及CPU的特性有很可能得到（0 0）。当然上面的代码大家不一定能得到（0 0），因为run()里面的操作过于简单，可能比启动一个线程花费的时间还少，因此上面的例子难以出现（0,0）。但是在现代CPU和JVM上确实是存在的。由于run()里面的动作对于结果是无关的，因此里面的指令可能发生指令重排序，即使是按照程序的顺序执行，数据变化刷新到主存也是需要时间的。假定是按照a=1;x=b;b=1;y=a;执行的，x=0是比较正常的，虽然a=1在y=a之前执行的，但是由于线程one执行a=1完成后还没有来得及将数据1写回主存（这时候数据是在线程one的堆栈里面的），线程two从主存中拿到的数据a可能仍然是0（显然是一个过期数据，但是是有可能的），这样就发生了数据错误。在两个线程交替执行的情况下数据的结果就不确定了，在机器压力大，多核CPU并发执行的情况下，数据的结果就更加不确定了。

**Happens-before法则**

Java存储模型有一个happens-before原则，就是如果动作B要看到动作A的执行结果（无论A/B是否在同一个线程里面执行），那么A/B就需要满足happens-before关系。

在介绍happens-before法则之前介绍一个概念：JMM动作（Java Memeory Model Action），Java存储模型动作。一个动作（Action）包括：变量的读写、监视器加锁和释放锁、线程的start()和join()。后面还会提到锁的的。

happens-before完整规则：

> （1）同一个线程中的每个Action都happens-before于出现在其后的任何一个Action。
>
> （2）对一个监视器的解锁happens-before于每一个后续对同一个监视器的加锁。
>
> （3）对volatile字段的写入操作happens-before于每一个后续的同一个字段的读操作。
>
> （4）Thread.start()的调用会happens-before于启动线程里面的动作。
>
> （5）Thread中的所有动作都happens-before于其他线程检查到此线程结束或者Thread.join（）中返回或者Thread.isAlive()==false。
>
> （6）一个线程A调用另一个另一个线程B的interrupt（）都happens-before于线程A发现B被A中断（B抛出异常或者A检测到B的isInterrupted（）或者interrupted()）。
>
> （7）一个对象构造函数的结束happens-before与该对象的finalizer的开始
>
> （8）如果A动作happens-before于B动作，而B动作happens-before与C动作，那么A动作happens-before于C动作

**volatile语义**

volatile相当于synchronized的弱实现，也就是说volatile实现了类似synchronized的语义，却又没有锁机制。它确保对volatile字段的更新以可预见的方式告知其他的线程。

volatile包含以下语义：

（1）Java 存储模型不会对volatile指令的操作进行重排序：这个保证对volatile变量的操作时按照指令的出现顺序执行的。

（2）volatile变量不会被缓存在寄存器中（只有拥有线程可见）或者其他对CPU不可见的地方，每次总是从主存中读取volatile变量的结果。也就是说对于volatile变量的修改，其它线程总是可见的，并且不是使用自己线程栈内部的变量。也就是在happens-before法则中，对一个valatile变量的写操作后，其后的任何读操作理解可见此写操作的结果。

尽管volatile变量的特性不错，但是volatile并不能保证线程安全的，也就是说volatile字段的操作不是原子性的，volatile变量只能保证可见性（一个线程修改后其它线程能够理解看到此变化后的结果），要想保证原子性，目前为止只能加锁！

volatile通常在下面的场景：

```java
volatile boolean done = false;
…
    while( ! done ){
        dosomething();
    }
```

应用volatile变量的三个原则：

> （1）写入变量不依赖此变量的值，或者只有一个线程修改此变量
>
> （2）变量的状态不需要与其它变量共同参与不变约束
>
> （3）访问变量不需要加锁

参考：[正确使用 Volatile 变量](http://www.ibm.com/developerworks/cn/java/j-jtp06197.html)

#### CAS操作

在JDK 5之前Java语言是靠synchronized关键字保证同步的，这会导致有锁（后面的章节还会谈到锁）。锁机制存在以下问题：

（1）在多线程竞争下，加锁、释放锁会导致比较多的上下文切换和调度延时，引起性能问题。

（2）一个线程持有锁会导致其它所有需要此锁的线程挂起。

（3）如果一个优先级高的线程等待一个优先级低的线程释放锁会导致优先级倒置，引起性能风险。

volatile是不错的机制，但是volatile不能保证原子性。因此对于同步最终还是要回到锁机制上来。

独占锁是一种悲观锁，synchronized就是一种独占锁，会导致其它所有需要锁的线程挂起，等待持有锁的线程释放锁。而另一个更加有效的锁就是乐观锁。所谓乐观锁就是，每次不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止。

**CAS操作**

上面的乐观锁用到的机制就是CAS，Compare and Swap。

CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。

**非阻塞算法 （nonblocking algorithms）**

> 一个线程的失败或者挂起不应该影响其他线程的失败或挂起的算法。

现代的CPU提供了特殊的指令，可以自动更新共享数据，而且能够检测到其他线程的干扰，而 compareAndSet() 就用这些代替了锁定。

拿出AtomicInteger来研究在没有锁的情况下是如何做到数据正确性的。

> private volatile int value;

首先毫无以为，在没有锁的机制下可能需要借助volatile原语，保证线程间的数据是可见的（共享的）。

这样才获取变量的值的时候才能直接读取。

> public final int get() {
>         return value;
>     }

然后来看看++i是怎么做到的。

> public final int incrementAndGet() {
>     for (;;) {
>         int current = get();
>         int next = current + 1;
>         if (compareAndSet(current, next))
>             return next;
>     }
> }

在这里采用了CAS操作，每次从内存中读取数据然后将此数据和+1后的结果进行CAS操作，如果成功就返回结果，否则重试直到成功为止。

而compareAndSet利用JNI来完成CPU指令的操作。

> public final boolean compareAndSet(int expect, int update) {   
>     return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
>     }

整体的过程就是这样子的，利用CPU的CAS指令，同时借助JNI来完成Java的非阻塞算法。其它原子操作都是利用类似的特性完成的。

而整个J.U.C都是建立在CAS之上的，因此对于synchronized阻塞算法，J.U.C在性能上有了很大的提升。参考资料的文章中介绍了如果利用CAS构建非阻塞计数器、队列等数据结构。

CAS看起来很爽，但是会导致`ABA问题`。

CAS算法实现一个重要前提需要取出内存中某时刻的数据，而在下时刻比较并替换，那么在这个时间差类会导致数据的变化。

比如说一个线程one从内存位置V中取出A，这时候另一个线程two也从内存中取出A，并且two进行了一些操作变成了B，然后two又将V位置的数据变成A，这时候线程one进行CAS操作发现内存中仍然是A，然后one操作成功。尽管线程one的CAS操作成功，但是不代表这个过程就是没有问题的。如果链表的头在变化了两次后恢复了原值，但是不代表链表就没有变化。因此前面提到的原子操作AtomicStampedReference/AtomicMarkableReference就很有用了。这允许一对变化的元素进行原子操作。

参考资料：

（1）[非阻塞算法简介](http://www.ibm.com/developerworks/cn/java/j-jtp04186/)

（2）[流行的原子](https://www.ibm.com/developerworks/cn/java/j-jtp11234/)
