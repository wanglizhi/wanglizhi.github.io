---
layout:     post
title:      "改善Java程序的151个建议读书笔记（下）"
subtitle:   "8、异常；9、多线程和并发；10、性能和效率；11、开源世界；12、思想为源"
date:       2016-05-20 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Java
---

## 第八章 异常

#### 建议110：提倡异常封装

Java语言的异常处理机制可以确保程序的健壮性，提高系统的可用率，但是Java API提供的异常都是比较低级的，只有开发人员能看懂，异常封装有三方面优点。

**1、提高系统的友好性**

```java
public static void doStuff() throws MyBussinessException{
  try{
    InputStream is = new FileInputStream("无效文件.txt");
  }catch(FileNotFoundException e){
    e.printStackTrace();
    //抛出业务异常
    throw new MyBussinessException(e);
}
}
```

**2、提高系统的可维护性**

```java
try{
  //do something
}catch(FileNotFoundException e){
  log.info("文件未找到");
}catch(SecurityException e){
  log.error("无权访问");
}
```

**3、解决Java异常机制自身的缺陷**

Java的异常一次只能抛出一个，如果在第一个逻辑片段抛出异常，则第二个逻辑片段就不在执行了。MyException异常只是一个异常容器，可以容纳多个异常，但它本身并不代表任何异常含义。

```java
public static void doStuff() throws MyException{
  List<Throwable> list = new ArrayList<Throwable>();
  try{
    //do something
  }catch(Exception e){
    list.add(e);
  }
  try{
    //do something
  }catch(Exception e){
    list.add(e);
  }
  if(list.size() > 0){
    throw new MyException(list);
  }
}
```

#### 建议111：采用异常链传递异常

异常的传递处理应该采用责任链模式。

```java
//IOException
public static IOException extends Exception{
  public IOException(String message){ super(message); }
  public IOException(String message, Throwable cause){
    super(message, cause);
  }
  //保留原始异常信息
  public IOException(Throwable cause){
    super(cause);
  }
}

try{
  // do something
}catch(Exception e){
  throw new IOException(e);
}
```

捕捉到Exception异常，转化为IOException异常并抛出（此方式叫做异常转移），调用者获得该异常后再调用getCause方法即可获得Exception的异常信息

#### 建议112：受检异常尽可能转化为非受检异常

受检异常是正常逻辑的一种补偿手段，特别是对可靠性要求比较高的系统来说，但是受检异常确实有缺陷：

1、受检异常使接口声明脆弱

```java
interface User{
  public void changePassword() throws MySecurityException;
}
```

异常是主逻辑的补充逻辑，修改一个补充逻辑，会导致主逻辑也被修改，接口是稳定的，一旦定义了异常，增加了接口的不稳定性；实现类变更最终会影响到调用者，破坏了封装性

2、受检异常使代码可读性降低

3、受检异常增加了开发工作量

注意：受检异常威胁到系统的安全性、稳定性、可靠性、正确性时，不能转换为非受检异常。

#### 建议113：不要在finally块中处理返回值

在finally代码块中加入return语句会导致两个问题：1、覆盖了try代码块中的return返回值；2、屏蔽try块抛出的异常

注意：不要在finally代码块中出现return语句

#### 建议114：不要在构造函数中抛出异常

Java的异常机制有三种：

- Error类及其子类表示的是错误，它是不需要程序员处理也不能处理的异常，比如VirtualMachineError虚拟机错误，ThreadDeath线程僵死
- RuntimeException类及其子类表示非受检异常，是系统可能会抛出的异常，可以处理也可以不处理，比如NullPointerException空指针和IndexOutOfBoundsException越界异常
- Exception类及其子类（不包含非受检异常）表示受检异常，是程序员必须处理的异常，不处理不能通过编译，如IOException，SQLException

尽量不要在构造函数中抛出异常

1、构造函数抛出错误是程序员无法处理的

2、构造函数不应该抛出非受检异常

```java
class Person{
  public Person(int age){
    if(age < 10){
      throw new RuntimeException("年龄必须大于18");
    }
  }
}
Person p = new Person(17);
//p对象会抛出异常，但非受检
```

3、构造函数尽可能不要抛出受检异常

```java
class Base{
  //父类抛出IOException
  public Base()throws IOException{}
}
//子类
class Sub extends Base(){
  //子类抛出Exception异常，必须抛出IOException或其父类
  public Sub() throws Exception{}
}
```

违背了里氏替换原则，因为抛出的异常类型不同，所以父类不能替换为子类。

#### 建议115：使用Throwable获得栈信息

AOP编程可以很轻松地控制一个方法调用哪些类，也能控制哪些方法允许被调用，下面，使用Throwable获得栈信息，然后鉴别调用者并分别输出。

```java
class Foo{
  public static boolean m(){
    //取得当前栈信息
    StackTraceElement[] sts = new Throwable().getStackTrace();
    //检查是否是m1方法调用
    for(StackTraceElement st : sts){
      if(st.getMethodName().equals("m1")){
        return true;
      }
    }
    return false;
  }
}
//调用者
class Invoker{
  //该方法打印出true
  public static void m1(){
    System.out.println(Foo.m());
  }
  //该方法打印出false
  public static void m2(){
    System.out.println(Foo.m());
  }
}
```

在异常出现时，JVM会通过fillInStackTrace方法记录下栈帧信息，然后生成一个Throwable对象，这样就可以知道类间的调用顺序、方法名以及当前行号了。

#### 建议116：异常只为异常服务

异常只能在非正常的情况下，不能成为正常情况的主逻辑。

#### 建议117：多使用异常，把性能问题放一边

性能问题不是拒绝异常的借口



## 第九章 多线程和并发

#### 建议118：不推荐覆写start方法

多线程比较简单的实现方式是继承Thread类，然后覆写run方法，在客户端程序中通过调用对象的start方法即可启动一个线程。

```java
class MultiThread extends Thread{
  @Override
  public void start(){
    run();
  }
  @Override
  public void run(){}
}
MultiThread multiThread = new MultiThread();
multiThread.start();
```

注意，次数并没有创建子线程，以为对start方法进行了覆写！start()方法中的关键的**start0()**方法，实现了启动线程、申请栈内存、运行run方法、修改线程状态等职责。实际上如果覆写start方法时，记得添加**super.start()**方法即可

```java
public synchronized void start(){
  //判断栈线程状态，必须是未启动状态
  if(threadStarts != 0)
    throw new IllegalThreadStateException();
  //加入到线程组中
  group.add(this);
  //分配栈内存、启动线程、运行run
  if(stopBeforeStart){
    stop0(throwableFromStop);
  }
}
//本地方法
private native void start0();
```

注意：继承自Thread类的多线程类不必覆写start方法

#### 建议119：启动线程前stop方法是不可靠的

线程启动（start方法）的一个缺陷：Thread类的stop方法会根据线程状态来判断是终结线程还是设置线程为不可运行状态，对于未启动的线程（线程状态为NEW）来说，会设置其标志位为不可启动，而其他状态则直接停止

#### 建议120：不使用stop方法停止线程

1、stop方法是过时的

2、stop方法会导致代码逻辑不完整，一旦执行stop方法，即终止当前正运行的线程，不管逻辑是否完整，都是非常危险的

3、stop方法会破坏原子逻辑，stop方法会丢弃所有的**锁**，导致原子逻辑受损

所以，如果期望终止一个正运行的线程，需要自行编码实现，保证原子逻辑不被破坏，代码逻辑不会异常。如果使用的是线程池（ThreadPoolExecutor类），那么可以通过shutdown方法逐步关闭池中的线程，它采用的是比较温和、安全的关闭线程的方法

#### 建议121：线程优先级只使用三个等级

线程的优先级决定了线程获得CPU运行的机会，Java的线程有10个级别（级别为0的线程是JVM，不能设置）。但是并**不严格遵照线程优先级**来执行，优先级差别越大，运行机会差别越明显。Java不能保证优先级高肯定会先执行，只能保证高优先级有更多的机会。

建议使用优先级常量，而不是1到10随机数字

```java
public class Thread implements Runnable{
  //最低优先级
  public final static int MIN_PRIORITY = 1;
  //普通优先级，默认
  public final static int NORM_PRIORITY = 5;
  //最高优先级
  public final static int MAX_PRIORITY = 10;
}
```

#### 建议123：volatile不能保证数据同步

volatile关键字比较少用，一是在Java1.5之前该关键字在不同操作系统上有不同的表现，移植性差；二是比较难设计，误用较多

每个线程运行在**栈内存**中，每个线程都有自己的工作内存，线程运行时，如果是读取，则直接从工作内存中读取，若是写入则先写到工作内存中，之后再刷新到主存中。多线程同步可以使用synchronized同步代码块，或者使用Lock锁来解决。

若是在一个变量前加上volatile关键字，可以确保每个线程对本地变量的访问和修改都是直接与主内存交互的，而不是与本线程的工作内存交互的，保证每个线程都能获得最新的变量值。但是**volatile关键字并不能保证线程安全**，它只是保证当线程需要该变量的值时能够获得最新的值，而不能保证多个线程修改的安全性。

#### 建议124：异步运算考虑使用Callable接口

多线程应用有两种实现方式，一种是实现Runnable接口，另一种是继承Thread类，两个方式都有缺点：**run方法没有返回值，不能抛出异常**。Java1.5开始引入了一个新的接口Callable，类似于Runnable接口，实现它就可以实现多线程任务。

```java
//税款计算器
class TaxCalculator implements Callable<Integer>{
  //本金
  private int seedMoney;
  public TaxCalculator(int _seedMoney){
    seedMoney = _seedMoney;
  }
  @Override
  public Integer call()throws Exception{
    //复杂计算，运行一个需要10秒
    TimeUnit.MILLISECONDS.sleep(10000);
    return seedMoney / 10;
  }
}
public static void main(String[] args)throws Exception{
  //生成一个单线程的异步执行器
  ExecutorService es = Executors.newSingleThreadExecutor();
  //线程执行后的期望值
  Future<Integer> future = es.submit(new TaxCalculator(100));
  while(!future.isDone()){
    //还没有运算完成，等待200毫秒
    TimeUnit.MILLISECONDS.sleep(200);
    System.out.print("#");
  }
  System.out.println("计算完成，税金是："+ future.get() + "元");
  es.shutdown();
}
```

Executors是一个静态工具类，提供了异步执行器的创建能力，如单线程执行器newSingleThreadExecutor、固定线程数量的执行器newFixedThreadPool等。Future关注的是执行后的结果。此类异步计算的好处是：

- 尽可能多地占用系统资源，提供快速运算
- 可以监控线程执行的情况，比如是否执行完成、是否有返回值、是否有异常等
- 可以为用户提供更好的支持，如运算进度

#### 建议125：优先选择线程池

一个线程有五个状态：新建状态（New）、可运行状态（Runnable，也叫运行状态）、阻塞状态（Blocked）、等待状态（Waiting）、结束状态（Terminated），线程的状态只能由新建转变为运行态后才可能被阻塞或等待，最后终结。

线程的运行时间分为三个部分：T1为线程启动时间，T2为线程体的运行时间，T3为线程销毁时间。T1和T3都可以通过线程池（Thread Pool）来缩减时间。

```java
//两个线程的线程池
ExecutorService es = Executors.newFixedThreadPool(2);
//多次执行线程体
for(int i=0; i<4; i++){
  es.submit(new Runnable(){
    public void run(){
      System.out.println(Thread.currentThread().getName());
    }
  });
}
//关闭执行器
es.shutdown();
```

线程池的实现设计三个名词：

- 工作线程（Worker）：线程池中的线程，只有两个状态：可运行状态和等待状态，在没有任务时处于等待状态，运行时可以循环地执行任务
- 任务接口（Task）：每个任务必须实现的接口，以供工作线程调度器调度，它主要规定了任务的入口、任务执行完成的场景处理、任务的执行状态等。
- 任务队列（Work Queue）：也叫工作队列，用于存放等待处理的任务，一般是BlockingQueue的实现类，用来实现任务的排队处理

Executor.newFixedThreadPool(2)表示创建一个具有2个线程的线程池：

```java
public class Executor{
  public static ExecutorService newFixedThreadPool(int nThreads){
    //生成一个最大为nThreads的线程池执行器
    return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());
}
}
```

LinkedBlockingQueue作为任务队列管理器，所有等待处理的任务都会放在该队列中，是一个阻塞式的单端队列。

线程池中的线程是在submit第一次提交任务时建立的：

```java
public Future<?> submit(Runnable task){
  if(task == null) throw new NullPointerException();
  //把Runnable任务包装成具有返回值的任务对象，此时没有执行，只是包装
  RunnableFuture<Object> ftask = newTaskFor(task, null);
  //执行此任务
  execute(ftask);
  //返回任务预期执行结果
  return ftask;
}
```

此代码关键是execute方法，它实现了三个职责：

- 创建足够多的工作线程数，数量不超过最大线程数量，并保持线程处于运行或等待状态
- 把等待处理的任务放到任务队列中
- 从任务队列中取出任务来执行

经过包装的Worker线程代码如下：

```java
private final class Worker implements Runnable{
  //运行一次任务
  private void runTask(Runnable task){
    //这里的task是我们自定义实现Runnable接口的任务
    task.run();
  }
  //工作线程也是线程，必须实现run方法
  public void run(){
    while(task != null || (task = getTask()) != null){
      runTask(task);
      task = null;
    }
  }
  //从任务队列中获得任务
  Runnable getTask(){
    for(;;){
      return workQueue.take();
  }
}
}
```

execute方法是通过Worker类启动的一个工作线程，执行的是第一个任务，然后通过getTask方法从任务队列中获取任务，之后继续执行。

LinkedBlockingQueue的take方法如下：

```java
public E take() throws InterruptedException{
  //如果队列中元素数量为0，则等待
  while(count.get() == 0)
    notEmpty.await();
  //等待状态结束，弹出头元素
  x = extract();
  //如果队列数量还多余1个，唤醒其他线程
  if(c > 1){
    notEmpty.signal();
  }
  return x; //返回头元素
}
```

#### 建议126：适时选择不同的线程池来实现

Java的线程池实现根本上有两个：ThreadPoolExecutor类和ScheduledThreadPoolExecutor类，父子关系，Java为了简化并行计算还提供了一个Executors的静态类，可以直接生成多种不同的线程池执行器。

public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit uinit, BlockingQueue< Runnable > workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler);

ThreadPoolExecutor完整的构造函数，参数分别是：

- corePoolSize：最小线程数
- maximumPoolSize：最大线程数量
- keepAliveTime：线程最大生命期
- unit：时间单位
- workQueue：任务队列
- threadFactory：线程工厂
- handler：拒绝任务处理器

Executorts提供的几个创建线程池的便捷方法（ThreadPoolExecutor的简化版）：

```java
//单线程池，池中只有一个线程在运行，永不超时
Executors.newSingleThreadExecutor();
//缓冲功能的线程池，多个线程，一旦一个线程60秒等待则会终止
Executors.newCachedThreadPool();
//固定线程数量的线程池
Executors.newFixedThreadPool();
```

#### 建议127：Lock与synchronized是不一样的

显式锁（Lock类）和内部锁（synchronized关键字）比较

```java
//任务类
class Task{
  public void doSomething(){
    try{
      //等待2秒，线程从运行状态转变为等待状态
      Thread.sleep(2000);
    }catch (Exception e){}
  }
  StringBuffer sb = new StringBuffer();
  sb.append("线程名称："+Thread.currentThread().getName());
  //运行时间戳
  sb.append("执行时间："+Calendar.getInstance().get(13)+" 秒");
  System.out.println(sb);
}
```

```java
//显式锁任务
class TaskWithLock extends Task implements Runnable{
  //声明显式锁
  private final Lock lock = new ReentrantLock();
  @Override
  public void run(){
    try{
      //开始锁定
      lock.lock();
      doSomething();
    }finally{
      //释放锁，finally确保可以释放锁
      lock.unlock();
    }
  }
}
```

```java
//内部锁任务
class TaskWithSync extends Task implements Runnable{
  @Override
  public void run(){
    //内部锁
    synchronized("A"){
      doSomething();
    }
  }
}
```

```java
//运行
public static void runTasks(Class<? extends Runnable> clz)throws Exception{
  ExecutorService es = Executors.newCacheThreadPool();
  //启动三个线程
  for(int i=0; i<3; i++){
    es.submit(clz.newInstance());
  }
  //等待足够长时间，然后关闭
  TimeUnit.SECONDS.sleep(10);
  es.shutdown();//关闭执行器
}
```



**运行结果**：显式锁是同时运行的，没有互斥。而内部锁的输出则是预期的，一个线程执行时，其他线程处于等待状态，执行完毕后，JVM从等待线程池中**随机**获得一个线程。

**Lock锁为什么不出现互斥情况呢？**因为对于同步资源来说，显式锁是对象级别的锁，而内部锁是类界别的锁，Lock锁是跟随对象的，所以把Lock定义为多线程类的私有属性是起不到互斥作用的，除非把Lock定义为所有线程的共享变量。

```java
public static void main(String[] args){
  //多个线程共享锁
  final Lock lock = new ReetrantLock();
  //启动三个线程
  for(int i=0; i<3; i++){
    new Thread(new Runnable(){
      try{
        lock.lock();
        Thread.sleep(2000);
        System.out.println(Thread.currentThread().getName());
      }catch(InterruptedException e){e.printStackTrace()}
      finally{
        lock.unlock();
      }
    })
  }
}
```

**显式锁和内部锁的不同点：**

1、Lock支持更细粒度的锁控制。例如：读写锁分离，写操作时不允许有读写操作，而读操作时读写可以并发执行

2、Lock是无阻塞锁，synchronized是阻塞锁。即当线程A持有锁时，线程B也期望获得锁，使用Lock时B处于等待状态，使用synchronized时B处于阻塞状态。

3、Lock可实现公平锁，synchronized只能是非公平锁。

当A持有锁，B、C处于等待状态时，若A释放锁，JVM将从B、C中随机挑选一个获得线程，称为**非公平锁**。若JVM选择了等待时间最长的一个线程持有锁，则为公平锁（保证每个线程的等待时间均衡）。Lock默认是非公平锁，在构造函数中加入参数true来声明公平锁，而synchronized不能实现公平锁

4、Lock是代码级别的，synchronized是JVM级的

Lock是通过编码实现的，synchronized是在运行期由JVM解释的，相对来说synchronized的优化可能性更高，Lock的优化则需要用户自行考虑。

**灵活强大则选择Lock，快捷安全则选择synchronized。**

#### 建议128：预防线程死锁

同一个线程持有对象锁，在一个线程内多个synchronized方法重入完全是可行的，不会产生死锁。

产生死锁的代码

```java
static class A{
  public synchronized void a1(B b){
    Thread.sleep(1000);
    b.b2(); //访问b2
  }
  public synchronized void a2(){
   System.out.println("进入a2");  
  }
}
static class B{
  public synchronized void b1(A a){
    Thread.sleep(1000);
    a.a2();//访问a2
  }
  public synchronized void b2(){
    System.out.println("进入b2");
  }
}
final A a = new A();
final B b = new B();
new Thread(new Runnable(){
  public void run(){
    a.a1(b);
  }
}, "线程A").start();
new Thread(new Runnable(){
  public void run(){
    b.b1(a);
  }
}, "线程B").start();
```

线程A和B分别持有a和b的锁，并且互相等待释放资源，产生死锁。

产生死锁的条件：

- 互斥条件：一个资源每次只能被一个线程使用
- 资源独占条件：一个线程因请求资源而阻塞时，对已获得的资源保持不放
- 不剥夺条件：线程已获得的资源在未使用完之前，不能强行剥夺
- 循环等待条件：若干线程之间形成一种循环等待资源条件

解决死锁的方式：

1、避免或减少资源共享

2、使用自旋锁，lock.tryLock(2, TimeUnit.SECONDS)，如果线程在等待2秒后无法获得资源，则自行终结该任务。

#### 建议129：适当设置阻塞队列长度

阻塞队列的容量是笃定的，非阻塞队列则是变长的。紫色队列可以在声明时指定队列的容量，若不指定，队列容量为Integer的最大值。

#### 建议130：使用CountDownLatch协调子线程

CountDownLatch的作用是一个控制计数器，每个线程在运行完毕后会执行countDown，表示自己运行结束

#### 建议131：CyclicBarrier让多线程齐步走

CyclicBarrier可以让所有线程处于等待状态，然后在满足条件的情况下继续执行。

## 第十章 性能和效率

#### 建议132：提升Java性能的基本方法

1、不要在循环条件中计算

2、尽可能把变量、方法声明为final static类型

3、缩小变量的作用范围，例如：能定义在一个try catch块内就放置在该块内，其目的是加快GC的回收

4、频繁字符串操作使用StringBuilder或StringBuffer

5、使用非线性检索

6、不建立冗余对象

#### 建议133：若非必要，不要克隆对象

克隆对象并不比直接生成对象效率高

#### 建议137：调整JVM参数以提升性能

**1、调整堆内存大小**

栈空间是由线程开辟，线程结束，由JVM回收，因此它的大小一般不会对性能有太大影响，但会影响系统的稳定性，超出栈内存时会报StackOverflowError错误。可以通过“java -Xss < size >”设置栈内存大小

堆内存的调整不能太随意，太小会导致Full GC频繁执行，性能下降；太大会浪费资源，产生系统不稳定的情况。

**2、调整堆内存中各分区的比例**

JVM的堆内存中包括三部分：新生区、养老区、永久存储区。其中新生成的对象都在新生区，又分为伊甸区、幸存0区、幸存1区。

当程序使用new关键字时，首先在伊甸区生产该对象，如果伊甸区满了，则用垃圾回收器进行回收，然后把剩余对象移动到幸存区（1或2），如果幸存区也满了，垃圾回收会再回收一个，把剩余的对象移动到养老区，当养老区也满了，就会触发**Full GCC**（非常危险的动作，JVM会停止所有执行，所有系统资源都会让位于垃圾回收器），如果还没有就抛出OutOffMemoryError错误

一般新生区和养老区比例为1：3左右

java -XX:NewSize=32m -XX:MaxNewSize=640m -XX:MaxPermSize=1280m -XX:NewRatio=5

该配置指定新生代初始化为32M，最大不超过640M，养老区最大不超过1280M

**3、变更垃圾回收策略**

java -XX:+UseParallelGC -XX:ParallelGCThread=20

该设置启用并行垃圾回收机制，并定义了20个收集线程。其他回收策略有：UseSerialGC（启用串行GC，默认），ScavengeBeforeFullGC（新生代GC优先于Full GC）、UseConcMarkSweepGC（对老生代采用并发标记交换算法进行GC）

**4、更换JVM**

目前市面上流行的JVM：Java HotSpot VM、Oracle JRockit JVM、IBM JVM



**剩余的策略建议略。。。**



参考：编写高质量代码——改善Java程序的151个建议


