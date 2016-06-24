---
layout:     post
title:      "Java集合类分析之Queue"
subtitle:   "Queue集合框架，ArrayDeque，LinkedList，PriorityQueue，ConcurrentLinkedQueue等"
date:       2016-06-18 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Java
    - Java Collections
---

## Queue概览

![](https://raw.githubusercontent.com/wanglizhi/wanglizhi.github.io/master/img/2016-06-22/Java-Collections_API-Queue.jpg)

Queue是一种很常见的数据结构类型，在java里面Queue是一个接口，它只是定义了一个基本的Queue应该有哪些功能规约。实际上有多个Queue的实现，有的是采用线性表实现，有的基于链表实现。还有的适用于多线程的环境。java中具有Queue功能的类主要有如下几个：AbstractQueue, ArrayBlockingQueue, ConcurrentLinkedQueue, LinkedBlockingQueue, DelayQueue, LinkedList, PriorityBlockingQueue, PriorityQueue和ArrayDqueue。

![](http://dl.iteye.com/upload/attachment/0081/3338/30179b73-4b08-34cb-9caf-02c058bf2b6b.jpg)

![](http://dl.iteye.com/upload/attachment/0081/3347/77ce0496-36fb-3b8a-b11a-57184fd44e60.jpg)

#### Queue接口

Queue作为一个接口，它声明的几个基本操作无非就是入队和出队的操作，具体定义如下：

```java
public interface Queue<E> extends Collection<E> {  
    boolean add(E e); // 添加元素到队列中，相当于进入队尾排队。 
    boolean offer(E e);  //添加元素到队列中，相当于进入队尾排队.  
    E remove(); //移除队头元素  
    E poll();  //移除队头元素  
    E element(); //获取但不移除队列头的元素  
    E peek();  //获取但不移除队列头的元素  
} 
```

#### Deque接口

Deque是一个双向队列，这将意味着它不过是对Queue接口的增强。如果仔细分析Deque接口代码的话，我们会发现它里面主要包含有4个部分的功能定义。1. 双向队列特定方法定义。 2. Queue方法定义。 3. Stack方法定义。 4. Collection方法定义。

第3，4部分的方法相当于告诉我们，具体实现Deque的类我们也可以把他们当成Stack和普通的Collection来使用。

```java
public interface Deque<E> extends Queue<E> {
boolean add(E e);  
boolean offer(E e);  
void addFirst(E e);  
void addLast(E e);  
boolean offerFirst(E e);  
boolean offerLast(E e);
E removeFirst();   
E removeLast();  
E pollFirst();  
E pollLast();  
E remove();  
E poll();
}
```

## ArrayDeque

它的内部使用一个数组来保存具体的元素，然后分别使用head, tail来指示队列的头和尾。

```java
transient Object[] elements;
transient int head;
transient int tail;
private static final int MIN_INITIAL_CAPACITY = 8;
public ArrayDeque() {
        elements = new Object[16];
}
public ArrayDeque(int numElements) {
        allocateElements(numElements);
}
public ArrayDeque(Collection<? extends E> c) {
        allocateElements(c.size());
        addAll(c);
}
```

ArrayDeque的默认长度为8，这么定义成2的指数值也是有一定好处的。在后面调整数组长度的时候我们会看到。关于tail需要注意的一点是tail所在的索引位置是null值，在它前面的元素才是队列中排在最后的元素。

#### 调整元素长度

在调整元素长度部分，ArrayDeque采用了两个方法来分配。一个是allocateElements，还有一个是doubleCapacity。allocateElements方法用于构造函数中根据指定的参数设置初始数组的大小。而doubleCapacity则用于当数组元素不够用了扩展数组长度。

```java
private void allocateElements(int numElements) {
        int initialCapacity = MIN_INITIAL_CAPACITY;
        // Find the best power of two to hold elements.
        // Tests "<=" because arrays aren't kept full.
        if (numElements >= initialCapacity) {
            initialCapacity = numElements;
            initialCapacity |= (initialCapacity >>>  1);
            initialCapacity |= (initialCapacity >>>  2);
            initialCapacity |= (initialCapacity >>>  4);
            initialCapacity |= (initialCapacity >>>  8);
            initialCapacity |= (initialCapacity >>> 16);
            initialCapacity++;

            if (initialCapacity < 0)   // Too many elements, must back off
                initialCapacity >>>= 1;// Good luck allocating 2 ^ 30 elements
        }
        elements = new Object[initialCapacity];
}
```

这部分代码里最让人困惑的地方就是对initialCapacity做的这一大堆移位和或运算。首先通过无符号右移1位，与原来的数字做或运算，然后在右移2、4、8、16位。这么做的目的是使得最后生成的数字尽可能每一位都是1。而且很显然，如果这个数字是每一位都为1，后面再对这个数字加1的话，则生成的数字肯定为2的若干次方。而且这个数字也肯定是大于我们的numElements值的最小2的指数值。这么说来有点绕。我们前面折腾了大半天，就为了求一个2的若干次方的数字，使得它要大于我们指定的数字，而且是最接近这个数字的数。这样子到底是为什么呢？因为我们后面要扩展数组长度的话，有了它这个基础我们就可以判断这个数字是不是到了2的多少多少次方，它增长下去最大的极限也不过是2的31次方。这样他每次的增长刚好可以把数组可以允许的长度给覆盖了，不会出现空间的浪费。比如说，我正好有一个数组，它的长度比Integer.MAX_VALUE的一半要大几个元素，如果我们这个时候设置的值不是让它为2的整数次方，那么直接对它空间翻倍就导致空间不够了，但是我们完全可以设置足够空间来容纳的。

```java
private void doubleCapacity() {
        assert head == tail;
        int p = head;
        int n = elements.length;
        int r = n - p; // number of elements to the right of p
        int newCapacity = n << 1;
        if (newCapacity < 0)
            throw new IllegalStateException("Sorry, deque too big");
        Object[] a = new Object[newCapacity];
        System.arraycopy(elements, p, a, 0, r);
        System.arraycopy(elements, 0, a, r, p);
        elements = a;
        head = 0;
        tail = n;
}
```

它只要扩展空间容量的时候左移一位，这就相当于空间翻倍了。如果长度超出了允许的范围，就会发生溢出，返回的结果就会成为一个负数。这就是为什么有 if (newCapacity < 0)这一句来抛异常。

#### 添加元素

```java
public boolean add(E e) {  
    addLast(e);  
    return true;  
}  
  
public void addLast(E e) {  
    if (e == null)  
        throw new NullPointerException();  
    elements[tail] = e;  
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)  
        doubleCapacity();  
}  
  
public boolean offer(E e) {  
    return offerLast(e);  
}  
  
public boolean offerLast(E e) {  
    addLast(e);  
    return true;  
}  
```

这里要注意的一个地方就是我们由于不断的入队和出队，可能head和tail都会移动到超过数组的末尾。这个时候如果有空闲的空间，我们会把头或者尾跳到数组的头开始继续移动。所以添加元素并确定元素的下标是一个将元素下标值和数组长度进行求模运算的过程。addLast方法通过和当前数组长度减1求与运算来得到最新的下标值。它的效果相当于tail = (tail + 1) % elements.length;

#### 取元素

```java
public E element() {  
    return getFirst();  
}  
  
public E getFirst() {  
    E x = elements[head];  
    if (x == null)  
        throw new NoSuchElementException();  
    return x;  
}  
  
public E peek() {  
    return peekFirst();  
}  
  
public E peekFirst() {  
    return elements[head]; // elements[head] is null if deque empty  
}  
  
public E getLast() {  
    E x = elements[(tail - 1) & (elements.length - 1)];  
    if (x == null)  
        throw new NoSuchElementException();  
    return x;  
}  
  
public E peekLast() {  
    return elements[(tail - 1) & (elements.length - 1)];  
}
```

这部分代码算是最简单的，无非就是取tail元素或者head元素的值。

## LinkedList

现在，我们在来看看LinkedList对应Queue的实现部分。在前面一篇文章中，已经讨论过LinkedList里面Node的结构。它本身包含元素值，prev、next两个引用。对链表的增加和删除元素的操作不像数组，不存在要考虑下标的问题，也不需要扩展数组空间，因此就简单了很多。先看查找元素部分：

```java
public E getFirst() {  
    final Node<E> f = first;  
    if (f == null)  
        throw new NoSuchElementException();  
    return f.item;  
}  
  
public E getLast() {  
    final Node<E> l = last;  
    if (l == null)  
        throw new NoSuchElementException();  
    return l.item;  
}
```

这里唯一值得注意的一点就是last引用是指向队列最末尾和元素，和前面ArrayDeque的情况不一样。

这里唯一值得注意的一点就是last引用是指向队列最末尾和元素，和前面ArrayDeque的情况不一样。

添加元素的方法如下：

```java
public boolean add(E e) {  
    linkLast(e);  
    return true;  
}  
  
public boolean offer(E e) {  
    return add(e);  
}  
  
void linkLast(E e) {  
    final Node<E> l = last;  
    final Node<E> newNode = new Node<>(l, e, null);  
    last = newNode;  
    if (l == null)  
        first = newNode;  
    else  
        l.next = newNode;  
    size++;  
    modCount++;  
} 
```

## PriorityQueue

在平时的编程工作中似乎很少碰到PriorityQueue（优先队列） ，故很多人一开始看到优先队列的时候还会有点迷惑。优先队列本质上就是一个最小堆。堆实质上是满足下面条件的树：

树中任一非叶结点的关键字均不大于（或不小于）其左右孩子（若存在）结点的关键字。

典型的优先队列的结构图：

![](http://dl.iteye.com/upload/attachment/0079/8621/bf7fc550-cf95-39b0-9015-ddbd8dbe4180.jpg)

![](http://dl.iteye.com/upload/attachment/0079/8629/d2f4c4bc-4fbf-3dff-92db-7ee06dec7bbf.jpg)

> 左子结点的数组索引号= 父结点索引号 * 2
>
> 右子结点的数组索引号=父结点索引号 * 2 + 1

#### 建堆

PriorityQueue内部的数组声明如下：

```java
transient Object[] queue;
private static final int DEFAULT_INITIAL_CAPACITY = 11;
private void initFromPriorityQueue(PriorityQueue<? extends E> c) {
        if (c.getClass() == PriorityQueue.class) {
            this.queue = c.toArray();
            this.size = c.size();
        } else {
            initFromCollection(c);
        }
}
```

它默认的长度为11。建堆的过程中需要添加新的元素。

下面以元素｛6，5，3，1，4，8，7｝为例，说明建堆的具体过程：

![](http://7xkjk9.com1.z0.glb.clouddn.com/%E5%A0%86%E6%8E%92%E5%BA%8F%E4%B9%8B%E5%BB%BA%E5%A0%86.jpg)

![](https://upload.wikimedia.org/wikipedia/commons/4/4d/Heapsort-example.gif)

PriorityQueue的建堆过程和最大堆的建堆过程基本上一样的，从有子节点的最靠后元素开始往前，每次都调用siftDown方法来调整。这个过程也叫做heapify。

```java
private void initFromCollection(Collection<? extends E> c) {
        initElementsFromCollection(c);
        heapify();
}
private void initElementsFromCollection(Collection<? extends E> c) {
        Object[] a = c.toArray();
        // If c.toArray incorrectly doesn't return Object[], copy it.
        if (a.getClass() != Object[].class)
            a = Arrays.copyOf(a, a.length, Object[].class);
        int len = a.length;
        if (len == 1 || this.comparator != null)
            for (int i = 0; i < len; i++)
                if (a[i] == null)
                    throw new NullPointerException();
        this.queue = a;
        this.size = a.length;
}
//从插入最后一个元素的父节点位置开始建堆
private void heapify() {
    for (int i = (size >>> 1) - 1; i >= 0; i--)
        siftDown(i, (E) queue[i]);
}
//在位置k插入元素x，为了保持最小堆的性质会不断调整节点位置
private void siftDown(int k, E x) {
    if (comparator != null)
        //使用插入元素的实现的比较器调整节点位置
        siftDownUsingComparator(k, x);
    else
        //使用默认的比较器（按照自然排序规则）调整节点的位置
        siftDownComparable(k, x);
}
//具体实现调整节点位置的函数
private void siftDownComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>)x;
    // 计算非叶子节点元素的最大位置
    int half = size >>> 1;        // loop while a non-leaf
    //如果不是叶子节点则一直循环
    while (k < half) {
        //得到k位置节点左孩子节点，假设左孩子比右孩子更小
        int child = (k << 1) + 1; // assume left child is least
        //保存左孩子节点值
        Object c = queue[child];
        //右孩子节点的位置
        int right = child + 1;
        //把左右孩子中的较小值保存在变量c中
        if (right < size &&
            ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
            c = queue[child = right];
        //如果要插入的节点值比其父节点更小，则交换两个节点的值
        if (key.compareTo((E) c) <= 0)
            break;
        queue[k] = c;
        k = child;
    }
    //循环结束，k是叶子节点
    queue[k] = key;
}
```

siftDown的过程是将一个节点和它的子节点进行比较调整，保证它比它所有的子节点都要小。这个调整的顺序是从当前节点向下，一直调整到叶节点。

#### 添加

ok，下面看看如何在一个最小堆中添加一个元素：

```java
public boolean add(E e) {
    //调用offer函数
    return offer(e);
}
//siftUp之前的代码主要确认队列的容量不发生溢出，并保存队列中的元素个数以及发生结构//性修改的次数
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    modCount++;
    int i = size;
    if (i >= queue.length)
        grow(i + 1);
    size = i + 1;
    if (i == 0)
        queue[0] = e;
    else
        //具体执行添加元素的函数
        siftUp(i, e);
    return true;
}
//调用不同的比较器调整元素的位置
private void siftUp(int k, E x) {
    if (comparator != null)
        siftUpUsingComparator(k, x);
    else
        siftUpComparable(k, x);
}
//使用默认的比较器调整元素的位置
private void siftUpComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>) x;
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        //保存父节点的值
        Object e = queue[parent];
        //使用compareTo方法，如果要插入的元素小于父节点的位置则交换两个节点的位置
        if (key.compareTo((E) e) >= 0)
            break;
        queue[k] = e;
        k = parent;
    }
    queue[k] = key;
}
//调用实现的比较器进行元素位置的调整，总的过程和上面一致，就是比较的方法不同
private void siftUpUsingComparator(int k, E x) {
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = queue[parent];
        //这里是compare方法
        if (comparator.compare(x, (E) e) >= 0)
            break;
        queue[k] = e;
        k = parent;
    }
    queue[k] = x;
}
```

每次添加新的元素进来实际上是在数组的最后面增加。在增加这个元素的时候就有了判断数组长度和调整的一系列动作。等这些动作调整完之后就要进行siftUp方法调整。这样做是为了保证堆原来的性质。

siftUp的过程可以用如下图来描述：

![](http://dl.iteye.com/upload/attachment/0081/4119/3aa16a0c-be32-32a9-aa78-32fcd6c1fb1d.jpg)

假设我们在后面添加了元素3，那么我们就要将这个元素和它的父节点比较，如果它比它的父节点小，则要交换他们的顺序。这样一直重复到它大于等于它的父节点或者到根节点。

经过调整之后，则变成符合条件的最小堆：

![](http://dl.iteye.com/upload/attachment/0081/4121/67b3f792-9686-3327-ac3f-8f9470de1433.jpg)

我们看前面的调整方法不管是siftUp还是siftDown都用了两个方法，一个是用的comparator，还有一个是用的默认比较结果。这样做的目的是考虑到我们要比较的元素不仅仅是数字等类型，也有可能是被定义了可比较的数据类型。对于自定义的数据类型，他们的大小比较定义需要实现comparator接口。

#### 出队

```java
private E removeAt(int i) {
    assert i >= 0 && i < size;
    modCount++;
    //s是队列的队头，对应到数组中就是最后一个元素
    int s = --size;
    //如果要移除的位置是最后一个位置，则把最后一个元素设为null
    if (s == i) // removed last element
        queue[i] = null;
    else {
        //保存待删除的节点元素
        E moved = (E) queue[s];
        queue[s] = null;
        //先把最后一个元素和i位置的元素交换，之后执行下调方法
        siftDown(i, moved);
        //如果执行下调方法后位置没变，说明该元素是该子树的最小元素，需要执行上调方//法,保持最小堆的性质
        if (queue[i] == moved) {//位置没变
            siftUp(i, moved);   //执行上调方法
            if (queue[i] != moved)//如果上调后i位置发生改变则返回该元素
                return moved;
        }
    }
    return null;
}
```

![](http://7xkjk9.com1.z0.glb.clouddn.com/%E5%A0%86%E6%8E%92%E5%BA%8F%E4%B9%8B%E5%88%A0%E9%99%A4%E5%85%83%E7%B4%A0%E4%B9%8B%E4%B8%8B%E8%B0%83%E6%96%B9%E6%B3%95.jpg)

#### PriorityQueue小结

经过上面的源码的分析，对PriorityQueue的总结如下：

- 时间复杂度：remove()方法和add()方法时间复杂度为O(logn)，remove(Object obj)和contains()方法需要O(n)时间复杂度，取队头则需要O(1)时间
- 在初始化阶段会执行建堆函数，最终建立的是最小堆，每次出队和入队操作不能保证队列元素的有序性，只能保证队头元素和新插入元素的有序性，如果需要有序输出队列中的元素，则只要调用Arrays.sort()方法即可
- 可以使用Iterator的迭代器方法输出队列中元素
- PriorityQueue是非同步的，要实现同步需要调用java.util.concurrent包下的PriorityBlockingQueue类来实现同步
- 在队列中不允许使用null元素

## BlockingQueue

 在Java多线程应用中，队列的使用率很高，多数生产消费模型的首选数据结构就是队列。Java提供的线程安全的Queue可以分为**阻塞队列和非阻塞队列**，其中阻塞队列的典型例子是BlockingQueue，非阻塞队列的典型例子是ConcurrentLinkedQueue，在实际应用中要根据实际需要选用阻塞队列或者非阻塞队列。 

注：什么叫**线程安全**？这个首先要明确。线程安全的类 ，指的是类内共享的全局变量的访问必须保证是不受多线程形式影响的。如果由于多线程的访问（比如修改、遍历、查看）而使这些变量结构被破坏或者针对这些变量操作的原子性被破坏，则这个类就不是线程安全的。 

BlockingQueue，顾名思义，“阻塞队列”：可以提供阻塞功能的队列。

首先，看看BlockingQueue提供的常用方法：

|      | 可能报异常     | 返回布尔值    | 可能阻塞   | 设定等待时间                  |
| ---- | --------- | -------- | ------ | ----------------------- |
| 入队   | add(e)    | offer(e) | put(e) | offer(e, timeout, unit) |
| 出队   | remove()  | poll()   | take() | poll(timeout, unit)     |
| 查看   | element() | peek()   | 无      | 无                       |

- add(e) remove() element() 方法不会阻塞线程。当不满足约束条件时，会抛出IllegalStateException 异常。例如：当队列被元素填满后，再调用add(e)，则会抛出异常。
- offer(e) poll() peek() 方法即不会阻塞线程，也不会抛出异常。例如：当队列被元素填满后，再调用offer(e)，则不会插入元素，函数返回false。
- 要想要实现阻塞功能，需要调用put(e) take() 方法。当不满足约束条件时，会阻塞线程。

**下面介绍ArrayBlockingQueue类的源码：**

## ArrayBlockingQueue

ArrayBlcokingQueue底层采用数组存储，并且使用ReentranLock保证线程安全。

```java
/** The queued items */
    final Object[] items;
    /** items index for next take, poll, peek or remove */
    int takeIndex;
    /** items index for next put, offer, or add */
    int putIndex;
    /** Number of elements in the queue */
    int count;

    /** Main lock guarding all access */
    final ReentrantLock lock;
    /** Condition for waiting takes */
    private final Condition notEmpty;
    /** Condition for waiting puts */
    private final Condition notFull;
public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
}
```

#### 非阻塞方法

```java
public boolean add(E e) {
        if (offer(e))
            return true;
        else
            throw new IllegalStateException("Queue full");
}
public E remove() {
        E x = poll();
        if (x != null)
            return x;
        else
            throw new NoSuchElementException();
}
public boolean offer(E e) {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count == items.length)
                return false;
            else {
                enqueue(e);
                return true;
            }
        } finally {
            lock.unlock();
        }
}
public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return (count == 0) ? null : dequeue();
        } finally {
            lock.unlock();
        }
}
```

很标准的ReentrantLock使用方式（ReentranLock[http://hellosure.iteye.com/blog/1121157](http://hellosure.iteye.com/blog/1121157)），另外对于insert和extract的实现没啥好说的。 
注：**先不看阻塞与否，这ReentrantLock的使用方式就能说明这个类是线程安全类**。 

#### 阻塞方法

```java
public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await(); // 满了等待
            enqueue(e);
        } finally {
            lock.unlock();
        }
}
private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        items[putIndex] = x;
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        notEmpty.signal(); //入队，唤醒一个空等待线程，类似生产者消费者
}
public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await(); //空了等待
            return dequeue();
        } finally {
            lock.unlock();
        }
}
// dequeue省略，类似enqueue
```

 对于第三类方法，这里面涉及到Condition类，简要提一下， 

- await方法指：造成当前线程在接到信号或被中断之前一直处于等待状态。 


- signal方法指：唤醒一个等待线程。

## LinkedBlockingQueue

Node节点，静态内部类，可以看出和LinkedList一样是双向链表。

```java
static final class Node<E> {
        E item;
        Node<E> prev;
        Node<E> next;
        Node(E x) {
            item = x;
        }
}
```

#### 构造

```java
/** The capacity bound, or Integer.MAX_VALUE if none */
    private final int capacity;
    /** Current number of elements */
    private final AtomicInteger count = new AtomicInteger();
    //两个变量来标示头和尾
    transient Node<E> head;
    private transient Node<E> last;
    /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();
    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();
    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();
    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();
```

 LinkedBlockingQueue相对于ArrayBlockingQueue还有不同是，有两个ReentrantLock，且队列现有元素的大小由一个AtomicInteger对象标示。 

注：AtomicInteger类是以原子的方式操作整型变量。

#### offer和poll方法

```java
// 插入，不扩容，否则失败
public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        final AtomicInteger count = this.count;
        if (count.get() == capacity)
            return false;
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;//入队当然用putLock  
        putLock.lock();
        try {
            if (count.get() < capacity) {
                enqueue(node);
                c = count.getAndIncrement();//队长度+1 
                if (c + 1 < capacity)
                    notFull.signal();//队列没满，当然可以解锁了
            }
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();//这个方法里发出了notEmpty.signal(); 
        return c >= 0;
}
public E poll() {
        final AtomicInteger count = this.count;
        if (count.get() == 0)
            return null;
        E x = null;
        int c = -1;
        final ReentrantLock takeLock = this.takeLock;//出队当然用takeLock 
        takeLock.lock();
        try {
            if (count.get() > 0) {
                x = dequeue();
                c = count.getAndDecrement();
                if (c > 1)
                    notEmpty.signal();//队列没空，解锁
            }
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();//这个方法里发出了notFull.signal(); 
        return x;
}
```

#### put和take方法

```java
public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
}
public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                notEmpty.await();
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
}
```

看看源代码发现和上面ArrayBlockingQueue的很类似，关键的问题在于：**为什么要用两个ReentrantLock**：putLock和takeLock？  

我们仔细想一下，入队操作其实操作的只有队尾引用last，并且没有牵涉到head。而出队操作其实只针对head，和last没有关系。那么就是说入队和出队的操作完全不需要公用一把锁，所以就设计了两个锁，这样就实现了多个不同任务的线程入队的同时可以进行出队的操作，另一方面由于两个操作所共同使用的count是AtomicInteger类型的，所以完全不用考虑计数器递增递减的问题。 

另外，还有一点需要说明一下：await()和singal()这两个方法执行时都会检查当前线程是否是独占锁的当前线程，如果不是则抛出java.lang.IllegalMonitorStateException异常。所以可以看到在源码中这两个方法都出现在Lock的保护块中。 

## LinkedBlockingQueue应用——生产者消费者模型

```java
public class ModelSample {
    /** 线程池提交的任务数*/
    private final int taskNum = Runtime.getRuntime().availableProcessors() + 1;
    /** 用于多线程间存取产品的队列*/
    private final LinkedBlockingQueue<String> queue = new LinkedBlockingQueue<String>(16);
    /** 记录产量*/
    private final AtomicLong output = new AtomicLong(0);
    /** 记录销量*/
    private final AtomicLong sales = new AtomicLong(0);
    /** 简单的线程起步开关*/
    private final CountDownLatch latch = new CountDownLatch(1);
    /** 停产后是否售完队列内的产品的选项*/
    private final boolean clear;
    /** 用于提交任务的线程池*/
    private final ExecutorService pool;
    /** 简陋的命令发送器*/
    private Scanner scanner;

    public ModelSample(boolean clear) {
        this.pool = Executors.newCachedThreadPool();
        this.clear = clear;
    }

    /**
     * 提交生产和消费任务给线程池,并在准备完毕后等待终止命令
     */
    public void service() {
        doService();
        waitCommand();
    }
    /**
     * 提交生产和消费任务给线程池,并在准备完毕后同时执行
     */
    private void doService() {
        for (int i = 0; i < taskNum; i++) {
            if (i == 0) {
                pool.submit(new Worker(queue, output, latch));
            }
            else {
                pool.submit(new Seller(queue, sales, latch, clear));
            }
        }
        latch.countDown();//开闸放狗,线程池内的线程正式开始工作
    }

    /**
     * 接收来自终端输入的终止命令
     */
    private void waitCommand() {
        scanner = new Scanner(System.in);
        while (!scanner.nextLine().equals("q")) {
            try {
                Thread.sleep(500);
            }
            catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        scanner.close();
        destory();
    }
    /**
     * 停止一切生产和销售的线程
     */
    private void destory() {
        pool.shutdownNow(); //不再接受新任务,同时试图中断池内正在执行的任务
        while (clear && queue.size() > 0) {
            try {
                Thread.sleep(500);
            }
            catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.println("Products:" + output.get() + "; Sales:" + sales.get());
    }

    public static void main(String[] args) {
        ModelSample model = new ModelSample(false);
        model.service();
    }
}
/**
 * 生产者
 */
class Worker implements Runnable {

    /** 假想的产品*/
    private static final String PRODUCT = "Thinkpad";
    private final LinkedBlockingQueue<String> queue;
    private final CountDownLatch latch;
    private final AtomicLong output;

    public Worker(LinkedBlockingQueue<String> queue, AtomicLong output,CountDownLatch latch) {
        this.output = output;
        this.queue = queue;
        this.latch = latch;
    }

    public void run() {
        try {
            latch.await(); // 放闸之前老实的等待着
            for (;;) {
                doWork();
                Thread.sleep(100);
            }
        }
        catch (InterruptedException e) {
            System.out.println("Worker thread will be interrupted...");
        }
    }

    private void doWork() throws InterruptedException {
        boolean success = queue.offer(PRODUCT, 100, TimeUnit.MILLISECONDS);
        if (success) {
            output.incrementAndGet(); // 可以声明long型的参数获得返回值,作为日志的参数
            // 可以在此处生成记录日志
        }
    }
}
/**
 * 消费者
 */
class Seller implements Runnable {

    private final LinkedBlockingQueue<String> queue;
    private final AtomicLong sales;
    private final CountDownLatch latch;
    private final boolean clear;

    public Seller(LinkedBlockingQueue<String> queue, AtomicLong sales, CountDownLatch latch, boolean clear) {
        this.queue = queue;
        this.sales = sales;
        this.latch = latch;
        this.clear = clear;
    }

    public void run() {
        try {
            latch.await(); // 放闸之前老实的等待着
            for (;;) {
                sale();
                Thread.sleep(500);
            }
        }
        catch (InterruptedException e) {
            if(clear) { // 响应中断请求后,如果有要求则销售完队列的产品后再终止线程
                cleanWarehouse();
            }
            else {
                System.out.println("Seller Thread will be interrupted...");
            }
        }
    }
    private void sale() throws InterruptedException {
        String item = queue.poll(50, TimeUnit.MILLISECONDS);
        if (item != null) {
            sales.incrementAndGet(); // 可以声明long型的参数获得返回值,作为日志的参数
            // 可以在此处生成记录日志
        }
    }

    /**
     * 销售完队列剩余的产品
     */
    private void cleanWarehouse() {
        try {
            while (queue.size() > 0) {
                sale();
            }
        }
        catch (InterruptedException ex) {
            System.out.println("Seller Thread will be interrupted...");
        }
    }
}
```

ModelSample是主控，负责生产消费的调度，Worker和Seller分别作为生产者和消费者，并均实现Runnable接口。ModelSample构造器内的参数clear主要是用于：当主控调用线程池的shutdownNow()方法时，会给池内的所有线程发送中断信号，使得线程的中断标志置位。这时候对应的Runnable的run()方法使用响应中断的LinkedBlockingQueue的方法(入队，出队)时就会抛出InterruptedException异常，生产者线程对这个异常的处理是记录信息后终止任务。而消费者线程是记录信息后终止任务，还是消费完队列内的产品再终止任务，则取决于这个选项值。

多线程的一个难点在于适当得销毁线程，这里得益于LinkedBlockingQueue的入队和出队的操作均提供响应中断的API，使得控制起来相对的简单一点。在Worker和Seller中共享LinkedBlockingQueue的实例queue时，我没有使用put或者take在queue满和空状态时无限制的阻塞线程，而是使用offer(E e, long timeout, TimeUnit unit)和poll(long timeout, TimeUnit unit)在指定的timeout时间内满足条件时阻塞线程。主要因为在于：先中断生产线程的情况下，如果所有的消费线程之前均被扔到等待集，那么无法它们将被唤醒。而后两者在超时后将自行恢复可运行状态。 

再者看看queue的size()方法，这也是选择LinkedBlockingQueue而不选ArrayBlockingQueue作为阻塞队列的原因。因为前者使用的AtomicInteger的count.get()返回最新值，完全无锁；而后者则需要获取唯一的锁，在此期间无法进行任何出队，入队操作。而这个例子中clear==true时，主线程和所有的消费线程均需要使用size()方法检查queue的元素个数。这类的非业务操作本就不该影响别的操作，所以这里LinkedBlockingQueue使用AtomicInteger计数无疑是个优秀的设计。 

另外编写这个例子时有点玩票的用了CountDownLatch，它的作用很简单。countDown()方法内部计数不为0时，执行了其await()方法的线程将会阻塞等待；一旦计数为0，这些线程将恢复可运行状态继续执行。这里用它就像一个发令枪，线程池submit任务的新线程在run内被阻塞，主线程一声令下countDown！这些生产消费线程均恢复执行状态。最后就是命令的实现过于简陋了，如果要响应其他的命令的话可以改造成响应事件处理的观察者模式，不过它不是演示的重点就从简了。



## ConcurrentLinkedQueue

ConcurrentLinkedQueue充分使用了atomic包的实现打造了一个无锁得并发线程安全的队列。对比锁机制的实现，个人认为使用无锁机制的难点在于要充分考虑线程间的协调。简单的说就是多个线程对内部数据结构进行访问时，如果其中一个线程执行的中途因为一些原因出现故障，其他的线程能够检测并帮助完成剩下的操作。这就需要把对数据结构的操作过程精细的划分成多个状态或阶段，考虑每个阶段或状态多线程访问会出现的情况。 

#### Node节点

```java
public class ConcurrentLinkedQueue<E> extends AbstractQueue<E>  
        implements Queue<E>, java.io.Serializable {  
    private static final long serialVersionUID = 196745693267521676L;  
  
    private static class Node<E> {  
        private volatile E item;  
        private volatile Node<E> next;  
  
        private static final  
            AtomicReferenceFieldUpdater<Node, Node>  
            nextUpdater =  
            AtomicReferenceFieldUpdater.newUpdater  
            (Node.class, Node.class, "next");  
        private static final  
            AtomicReferenceFieldUpdater<Node, Object>  
            itemUpdater =  
            AtomicReferenceFieldUpdater.newUpdater  
            (Node.class, Object.class, "item");  
  
        Node(E x) { item = x; }  
  
        Node(E x, Node<E> n) { item = x; next = n; }  
  
        E getItem() {  
            return item;  
        }  
  
        boolean casItem(E cmp, E val) {  
            return itemUpdater.compareAndSet(this, cmp, val);  
        }  
  
        void setItem(E val) {  
            itemUpdater.set(this, val);  
        }  
  
        Node<E> getNext() {  
            return next;  
        }  
  
        boolean casNext(Node<E> cmp, Node<E> val) {  
            return nextUpdater.compareAndSet(this, cmp, val);  
        }  
  
        void setNext(Node<E> val) {  
            nextUpdater.set(this, val);  
        }  
  
    }  
  
    private static final  
        AtomicReferenceFieldUpdater<ConcurrentLinkedQueue, Node>  
        tailUpdater =  
        AtomicReferenceFieldUpdater.newUpdater  
        (ConcurrentLinkedQueue.class, Node.class, "tail");  
    private static final  
        AtomicReferenceFieldUpdater<ConcurrentLinkedQueue, Node>  
        headUpdater =  
        AtomicReferenceFieldUpdater.newUpdater  
        (ConcurrentLinkedQueue.class,  Node.class, "head");  
  
    private boolean casTail(Node<E> cmp, Node<E> val) {  
        return tailUpdater.compareAndSet(this, cmp, val);  
    }  
  
    private boolean casHead(Node<E> cmp, Node<E> val) {  
        return headUpdater.compareAndSet(this, cmp, val);  
    }  
  
    private transient volatile Node<E> head = new Node<E>(null, null);  
  
    private transient volatile Node<E> tail = head;  
    ...  
}
```

先看看其内部数据结构Node的实现。由于使用了原子字段更新器AtomicReferenceFieldUpdater< T,V >（其中T表示持有字段的类的类型，V表示字段的类型），所以其对应的需要更新的字段要使用volatile进行声明。其newUpdater(Class< U > tclass, Class< W > vclass, String fieldName)方法实例化一个指定字段的更新器，参数分别表示：持有需要更新字段的类，字段的类，要更新的字段的名称。Node的内部变量item，next分别有对应自己的字段更新器，并且包含了对其原子性操作的方法compareAndSet(T obj, V expect, V update)，其中T是持有被设置字段的对象，后两者分别是期望值和新值。

#### offer方法

对于ConcurrentLinkedQueue自身也有两个volatile的线程共享变量：head，tail分别对应队列的头指针和尾指针。要保证这个队列的线程安全就是保证对这两个Node的引用的访问（更新，查看）的原子性和可见性，由于volatile本身能够保证可见性，所以就是对其修改的原子性要被保证。下面看看其对应的方法是如何完成的。

```java
public boolean offer(E e) {  
    if (e == null) throw new NullPointerException();  
    Node<E> n = new Node<E>(e, null);  
    for (;;) {  
        Node<E> t = tail;  
        Node<E> s = t.getNext();  
        if (t == tail) { //------------------------------a  
            if (s == null) { //---------------------------b  
                if (t.casNext(s, n)) { //-------------------c  
                    casTail(t, n); //------------------------d  
                    return true;  
                }  
            } else {  
                casTail(t, s); //----------------------------e  
            }  
        }  
    }  
} 
```

offer()方法都很熟悉了，就是入队的操作。涉及到改变尾指针的操作，所以要看这个方法实现是否保证了原子性。CAS操作配合循环是原子性操作的保证，这里也不例外。此方法的循环内首先获得尾指针和其next指向的对象，由于tail和Node的next均是volatile的，所以保证了获得的分别都是最新的值。 

- 代码a：t==tail是最上层的协调，如果其他线程改变了tail的引用，则说明现在获得不是最新的尾指针需要重新循环获得最新的值。 


- 代码b：s==null的判断。静止状态下tail的next一定是指向null的，但是多线程下的另一个状态就是中间态：tail的指向没有改变，但是其next已经指向新的结点，即完成tail引用改变前的状态，这时候s!=null。这里就是协调的典型应用，直接进入代码e去协调参与中间态的线程去完成最后的更新，然后重新循环获得新的tail开始自己的新一次的入队尝试。另外值得注意的是a,b之间，其他的线程可能会改变tail的指向，使得协调的操作失败。从这个步骤可以看到无锁实现的复杂性。 
- 代码c：t.casNext(s, n)是入队的第一步，因为入队需要两步：更新Node的next，改变tail的指向。代码c之前可能发生tail引用指向的改变或者进入更新的中间态，这两种情况均会使得t指向的元素的next属性被原子的改变，不再指向null。这时代码c操作失败，重新进入循环。 
- 代码d：这是完成更新的最后一步了，就是更新tail的指向，最有意思的协调在这儿又有了体现。从代码看casTail(t, n)不管是否成功都会接着返回true标志着更新的成功。首先如果成功则表明本线程完成了两步的更新，返回true是理所当然的；如果 casTail(t, n)不成功呢？要清楚的是完成代码c则代表着更新进入了中间态，代码d不成功则是tail的指向被其他线程改变。意味着对于其他的线程而言：它们得到的是中间态的更新，s!=null，进入代码e帮助本线程执行最后一步并且先于本线程成功。这样本线程虽然代码d失败了，但是是由于别的线程的协助先完成了，所以返回true也就理所当然了。 

通过分析这个入队的操作，可以清晰的看到无锁实现的每个步骤和状态下多线程之间的协调和工作。理解了入队的整个过程，出队的操作poll()的实现也就变得简单了。基本上是大同小异的，无非就是同时牵涉到了head和tail的状态，在改变head的同时照顾到tail的协调，在此不多赘述。

## 非阻塞算法

阻塞算法其实很好理解，简单点理解就是加锁，比如在BlockingQueue中看到的那样，再往前推点，那就是synchronized。相比而言，非阻塞算法的设计和实现都很困难，要通过低级的原子性来支持并发。下面就简要的介绍一下**非阻塞算法**

#### 使用同步的线程安全的计数器

```java
public final class Counter {  
    private long value = 0;  
    public synchronized long getValue() {  
        return value;  
    }  
    public synchronized long increment() {  
        return ++value;  
    }  
} 
```

#### 基于CAS方法的非阻塞算法

使用 AtomicInteger的compareAndSet()（**CAS方法**）的计数器。compareAndSet()方法规定“将这个变量更新为新值，但是如果从我上次看到这个变量之后其他线程修改了它的值，那么更新就失败” 

```java
public class NonblockingCounter {  
    private AtomicInteger value;//前面提到过，AtomicInteger类是以原子的方式操作整型变量。  
    public int getValue() {  
        return value.get();  
    }  
    public int increment() {  
        int v;  
        do {  
            v = value.get();  
        while (!value.compareAndSet(v, v + 1));  
        return v + 1;  
    }  
} 
```

非阻塞版本相对于基于锁的版本有几个性能优势。首先，它用硬件的原生形态代替 JVM 的锁定代码路径，从而在更细的粒度层次上（独立的内存位置）进行同步，失败的线程也可以立即重试，而不会被挂起后重新调度。更细的粒度降低了争用的机会，不用重新调度就能重试的能力也降低了争用的成本。即使有少量失败的 CAS 操作，这种方法仍然会比由于锁争用造成的重新调度快得多。

NonblockingCounter 这个示例可能简单了些，但是它演示了所有非阻塞算法的一个基本特征——有些算法步骤的执行是要冒险的，因为知道如果 CAS 不成功可能不得不重做。非阻塞算法通常叫作**乐观算法**，因为它们继续操作的假设是不会有干扰。如果发现干扰，就会回退并重试。在计数器的示例中，冒险的步骤是递增——它检索旧值并在旧值上加一，希望在计算更新期间值不会变化。如果它的希望落空，就会再次检索值，并重做递增计算。

#### ConcurrentLinkedQueue算法原理

队列总是处于两种状态之一：正常状态（或称静止状态，图 1 和 图 3）或中间状态（图 2）。在插入操作之前和第二个 CAS（D）成功之后，队列处在静止状态；在第一个 CAS（C）成功之后，队列处在中间状态。在静止状态时，尾指针指向的链接节点的 next 字段总为 null，而在中间状态时，这个字段为非 null。任何线程通过比较 tail.next 是否为 null，就可以判断出队列的状态，这是让线程可以帮助其他线程 “完成” 操作的关键。

![](http://dl.iteye.com/upload/attachment/519329/a79c19ec-1994-3bc6-b452-41c20b9b2127.gif)

*上图显示的是：有两个元素，处在静止状态的队列*  

插入操作在插入新元素（A）之前，先检查队列是否处在中间状态。如果是在中间状态，那么肯定有其他线程已经处在元素插入的中途，在步骤（C）和（D）之间。不必等候其他线程完成，当前线程就可以 “帮助” 它完成操作，把尾指针向前移动（B）。如果有必要，它还会继续检查尾指针并向前移动指针，直到队列处于静止状态，这时它就可以开始自己的插入了。 
第一个 CAS（C）可能因为两个线程竞争访问队列当前的最后一个元素而失败；在这种情况下，没有发生修改，失去 CAS 的线程会重新装入尾指针并再次尝试。如果第二个 CAS（D）失败，插入线程不需要重试 —— 因为其他线程已经在步骤（B）中替它完成了这个操作！ 

![](http://dl.iteye.com/upload/attachment/519331/41902084-1dde-3c9f-b064-a5dd4a47ff8d.gif)

*上图显示的是：处在插入中间状态的队列，在新元素插入之后，尾指针更新之前* 

![](http://dl.iteye.com/upload/attachment/519333/49660d1c-0528-3c7c-b3c8-53fbdea9c1d2.gif)

*上图显示的是：在尾指针更新后，队列重新处在静止状态*



参考：[java集合类深入分析之Queue篇](http://shmilyaw-hotmail-com.iteye.com/blog/1700599)

[java集合类深入分析之PriorityQueue](http://shmilyaw-hotmail-com.iteye.com/blog/1827136)

[深入Java集合系列之五：PriorityQueue](http://blog.csdn.net/u011116672/article/details/50997622)

[Java多线程总结之聊一聊Queue](http://hellosure.iteye.com/blog/1126541)

[多线程基础总结十--LinkedBlockingQueue](http://yanxuxin.iteye.com/blog/582162)

[多线程基础总结十一--ConcurrentLinkedQueue](http://yanxuxin.iteye.com/blog/586943)

[LinkedBlockingQueue应用--生产消费模型简单实现](http://yanxuxin.iteye.com/blog/583645)

