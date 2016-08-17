---
layout:     post
title:      "深入浅出Java Concurrency——并发容器"
subtitle:   "java.util.concurrent，并发容器"
date:       2016-08-07 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Java
    - Java并发编程
---

## ConcurrentMap

在JDK 1.4以下只有Vector和Hashtable是线程安全的集合（也称并发容器，Collections.synchronized*系列也可以看作是线程安全的实现）。从JDK 5开始增加了线程安全的Map接口ConcurrentMap和线程安全的队列BlockingQueue（尽管Queue也是同时期引入的新的集合，但是规范并没有规定一定是线程安全的，事实上一些实现也不是线程安全的，比如PriorityQueue、ArrayDeque、LinkedList等，在Queue章节中会具体讨论这些队列的结构图和实现）。

在介绍ConcurrencyMap之前先来回顾下Map的体系结构。下图描述了Map的体系结构，其中蓝色字体的是JDK 5以后新增的并发容器。

[![image](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency16part1ConcurrentMap1_10A52/image_thumb.png)](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency16part1ConcurrentMap1_10A52/image_2.png)

1. Hashtable是JDK 5之前Map唯一线程安全的内置实现（Collections.synchronizedMap不算）。特别说明的是Hashtable的t是小写的（不知道为啥），Hashtable继承的是Dictionary（Hashtable是其唯一公开的子类），并**不继承AbstractMap或者HashMap**。尽管Hashtable和HashMap的结构非常类似，但是他们之间并没有多大联系。
2. ConcurrentHashMap是HashMap的线程安全版本，ConcurrentSkipListMap是TreeMap的线程安全版本。
3. 最终可用的线程安全版本Map实现是ConcurrentHashMap/ConcurrentSkipListMap/Hashtable/Properties四个，但是Hashtable是过时的类库，因此如果可以的应该尽可能的使用ConcurrentHashMap和ConcurrentSkipListMap。

 

回到正题来，这个小节主要介绍ConcurrentHashMap的API以及应用，下一节才开始将原理和分析。

除了实现Map接口里面对象的方法外，ConcurrentHashMap还实现了ConcurrentMap里面的四个方法。

**V putIfAbsent(K key,V value)**

如果不存在key对应的值，则将value以key加入Map，否则返回key对应的旧值。这个等价于清单1 的操作：

**清单1 putIfAbsent的等价操作**

```java
if (!map.containsKey(key)) 
   return map.put(key, value);
else
   return map.get(key);
```

在前面的章节中提到过，连续两个或多个原子操作的序列并不一定是原子操作。比如上面的操作即使在Hashtable中也不是原子操作。而putIfAbsent就是一个线程安全版本的操作的。

有些人喜欢用这种功能来实现[**单例模式**](http://www.blogjava.net/xylz/archive/2009/12/18/306622.html)，例如清单2。

**清单2 一种单例模式的实现**

```java
public class ConcurrentDemo1 {

    private static final ConcurrentMap<String, ConcurrentDemo1> map = new ConcurrentHashMap<String, ConcurrentDemo1>();
    private static ConcurrentDemo1 instance;
    public static ConcurrentDemo1 getInstance() {
        if (instance == null) {

            map.putIfAbsent("INSTANCE", new ConcurrentDemo1());

            instance = map.get("INSTANCE");
        }
        return instance;
    }
    private ConcurrentDemo1() {
    }
}
```

当然这里只是一个操作的例子，实际上在[**单例模式**](http://www.blogjava.net/xylz/archive/2009/12/18/306622.html)文章中有很多的实现和比较。清单2 在存在大量单例的情况下可能有用，实际情况下很少用于单例模式。但是这个方法避免了向Map中的同一个Key提交多个结果的可能，有时候在去掉重复记录上很有用（如果记录的格式比较固定的话）。

**boolean remove(Object key,Object value)**

只有目前将键的条目映射到给定值时，才移除该键的条目。这等价于清单3 的操作。

**清单3 remove(Object,Object)的等价操作**

```java
f (map.containsKey(key) && map.get(key).equals(value)) {
   map.remove(key);
   return true;
}
return false;
```

由于集合类通常比较的hashCode和equals方法，而这两个方法是在Object对象里面，因此两个对象如果hashCode一致，并且覆盖了equals方法后也一致，那么这两个对象在集合类里面就是“相同”的，不管是否是同一个对象或者同一类型的对象。也就是说只要key1.hashCode()==key2.hashCode() && key1.equals(key2)，那么key1和key2在集合类里面就认为是一致，哪怕他们的Class类型不一致也没关系，所以在很多集合类里面允许通过Object来类型来比较（或者定位）。比如说Map尽管添加的时候只能通过制定的类型<K,V>，但是删除的时候却允许通过一个Object来操作，而不必是K类型。

既然Map里面有一个remove(Object)方法，为什么ConcurrentMap还需要remove(Object,Object)方法呢？这是因为尽管Map里面的key没有变化，但是value可能已经被其他线程修改了，如果修改后的值是我们期望的，那么我们就不能拿一个key来删除此值，尽管我们的期望值是删除此key对于的旧值。

这种特性在原子操作章节的[AtomicMarkableReference](http://www.blogjava.net/xylz/archive/2010/07/02/325079.html)和[AtomicStampedReference](http://www.blogjava.net/xylz/archive/2010/07/02/325079.html)里面介绍过。

**boolean replace(K key,V oldValue,V newValue)**

只有目前将键的条目映射到给定值时，才替换该键的条目。这等价于清单4 的操作。

**清单4 replace(K,V,V)的等价操作**

```java
f (map.containsKey(key) && map.get(key).equals(oldValue)) {
   map.put(key, newValue);
   return true;
}
return false;
```

**V replace(K key,V value)**

只有当前键存在的时候更新此键对于的值。这等价于清单5 的操作。

**清单5 replace(K,V)的等价操作**

```java
if (map.containsKey(key)) {
   return map.put(key, value);
}
return null;
```

replace(K,V,V)相比replace(K,V)而言，就是增加了匹配oldValue的操作。

 

其实这4个扩展方法，是ConcurrentMap附送的四个操作，其实我们更关心的是Map本身的操作。当然如果没有这4个方法，要完成类似的功能我们可能需要额外的锁，所以有总比没有要好。比如清单6，如果没有putIfAbsent内置的方法，我们如果要完成此操作就需要完全锁住整个Map，这样就大大降低了ConcurrentMap的并发性。这在下一节中有详细的分析和讨论。

**清单6 putIfAbsent的外部实现**

```java
public V putIfAbsent(K key, V value) {
    synchronized (map) {
        if (!map.containsKey(key)) return map.put(key, value);
        return map.get(key);
    }
}
```

### HashMap原理

我们从头开始设想。要将对象存放在一起，如何设计这个容器。目前只有两条路可以走，一种是采用分格技术，每一个对象存放于一个格子中，这样通过对格子的编号就能取到或者遍历对象；另一种技术就是采用串联的方式，将各个对象串联起来，这需要各个对象至少带有下一个对象的索引（或者指针）。显然第一种就是数组的概念，第二种就是链表的概念。所有的容器的实现其实都是基于这两种方式的，不管是数组还是链表，或者二者俱有。HashMap采用的就是数组的方式。

有了存取对象的容器后还需要以下两个条件才能完成Map所需要的条件。

- 能够快速定位元素：Map的需求就是能够根据一个查询条件快速得到需要的结果，所以这个过程需要的就是尽可能的快。
- 能够自动扩充容量：显然对于容器而然，不需要人工的去控制容器的容量是最好的，这样对于外部使用者来说越少知道底部细节越好，不仅使用方便，也越安全。

首先条件1，快速定位元素。快速定位元素属于算法和数据结构的范畴，通常情况下哈希（Hash）算法是一种简单可行的算法。所谓**哈希算法**，是将任意长度的二进制值映射为固定长度的较小二进制值。常见的MD2,MD4,MD5，SHA-1等都属于Hash算法的范畴。具体的算法原理和介绍可以参考相应的算法和数据结构的书籍，但是这里特别提醒一句，由于将一个较大的集合映射到一个较小的集合上，所以必然就存在多个元素映射到同一个元素上的结果，这个叫“碰撞”，后面会用到此知识，暂且不表。

条件2，如果满足了条件1，一个元素映射到了某个位置，现在一旦扩充了容量，也就意味着元素映射的位置需要变化。因为对于Hash算法来说，调整了映射的小集合，那么原来映射的路径肯定就不复存在，那么就需要对现有重新计算映射路径，也就是所谓的rehash过程。

好了有了上面的理论知识后来看HashMap是如何实现的。

在HashMap中首先由一个对象数组table是不可避免的，修饰符transient只是表示序列号的时候不被存储而已。size描述的是Map中元素的大小，threshold描述的是达到指定元素个数后需要扩容，loadFactor是扩容因子(loadFactor>0)，也就是计算threshold的。那么元素的容量就是table.length，也就是数组的大小。换句话说，如果存取的元素大小达到了整个容量(table.length)的loadFactor倍（也就是table.length*loadFactor个），那么就需要扩充容量了。在HashMap中每次扩容就是将扩大数组的一倍，使数组大小为原来的两倍。

 [![HashMap数据结构](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency17part2ConcurrentMap2_FF15/image_thumb.png)](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency17part2ConcurrentMap2_FF15/image_2.png)

然后接下来看如何将一个元素映射到数组table中。显然要映射的key是一个无尽的超大集合，而table是一个较小的有限集合，那么一种方式就是将key编码后的hashCode值取模映射到table上，这样看起来不错。但是在Java中采用了一种更高效的办法。由于与(&)是比取模(%)更高效的操作，因此Java中采用hash值与数组大小-1后取与来确定数组索引的。为什么这样做是更有效的？[参考资料7](http://www.javaeye.com/topic/539465)对这一块进行非常详细的分析，这篇文章的作者非常认真，也非常仔细的分析了里面包含的思想。

**清单1 indexFor片段**

```java
static int indexFor(int h, int length) {
    return h & (length-1);
}
```

前面说明，既然是大集合映射到小集合上，那么就必然存在“碰撞”，也就是不同的key映射到了相同的元素上。那么HashMap是怎么解决这个问题的？

在HashMap中采用了下面方式，解决了此问题。

1. 同一个索引的数组元素组成一个链表，查找允许时循环链表找到需要的元素。
2. 尽可能的将元素均匀的分布在数组上。

[![Map.Entry结构](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency17part2ConcurrentMap2_FF15/image_thumb_1.png)](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency17part2ConcurrentMap2_FF15/image_4.png)对于问题1，HashMap采用了上图的一种数据结构。table中每一个元素是一个Map.Entry，其中Entry包含了四个数据，key,value,hash,next。key和value是存储的数据；hash是元素key的Hash后的表现形式（最终要映射到数组上），这里链表上所有元素的hash经过清单1 的indexFor后将得到相同的数组索引；next是指向下一个元素的索引，同一个链表上的元素就是通过next串联起来的。

再来看问题2 尽可能的将元素均匀的分布在数组上这个问题是怎么解决的。首先清单2 是将key的hashCode经过一系列的变换，使之更符合小数据集合的散列模型。

**清单2 hashCode的二次散列**

```java
static int hash(int h) {
    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

至于清单2 为什么这样散列我没有找到依据，也没有什么好的参考资料。[参考资料1](http://www.javaeye.com/topic/709945)分析了此过程，认为是一种比较有效的方式，有兴趣的可以研究下。

第二点就是在清单1 的描述中，尽可能的与数组的长度减1的数与操作，使之分布均匀。这在[参考资料7](http://www.javaeye.com/topic/539465) 中有介绍。

第三点就是构造数组时数组的长度是2的倍数。清单3 反映了这个过程。为什么要是2的倍数？在[参考资料7](http://www.javaeye.com/topic/539465) 中分析说是使元素尽可能的分布均匀。

**清单3 HashMap 构造数组**

```java
// Find a power of 2 >= initialCapacity
int capacity = 1;
while (capacity < initialCapacity)
    capacity <<= 1;

this.loadFactor = loadFactor;
threshold = (int)(capacity * loadFactor);
table = new Entry[capacity];
```

另外loadFactor的默认值0.75和capacity的默认值16是经过大量的统计分析得出的，很久以前我见过相关的数据分析，现在找不到了，有兴趣的可以查询相关资料。这里不再叙述了。

有了上述原理后再来分析HashMap的各种方法就不是什么问题的。

**清单4 HashMap的get操作**

```java
public V get(Object key) {
    if (key == null)
        return getForNullKey();
    int hash = hash(key.hashCode());
    for (Entry<K,V> e = table[indexFor(hash, table.length)];
         e != null;
         e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
            return e.value;
    }
    return null;
}
```

清单4 描述的是HashMap的get操作，在这个操作中首先判断key是否为空，因为为空的话总是映射到table的第0个元素上（可以看上面的清单2和清单1）。然后就需要查找table的索引。一旦找到对应的Map.Entry元素后就开始遍历此链表。由于不同的hash可能映射到同一个table[index]上，而相同的key却同时映射到相同的hash上，所以一个key和Entry对应的条件就是hash(key)==e.hash 并且key.equals(e.key)。从这里我们看到，Object.hashCode()只是为了将相同的元素映射到相同的链表上（Map.Entry)，而Object.equals()才是比较两个元素是否相同的关键！这就是为什么总是成对覆盖hashCode()和equals()的原因。

**清单5 HashMap的put操作**

```java
public V put(K key, V value) {
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key.hashCode());
    int i = indexFor(hash, table.length);
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    addEntry(hash, key, value, i);
    return null;
}
void addEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new Entry<K,V>(hash, key, value, e);
        if (size++ >= threshold)
            resize(2 * table.length);
}
```

清单5 描述的是HashMap的put操作。对比get操作，可以发现，put实际上是先查找，一旦找到key对应的Entry就直接修改Entry的value值，否则就增加一个元素。增加的元素是在链表的头部，也就是占据table中的元素，如果table中对应索引原来有元素的话就将整个链表添加到新增加的元素的后面。也就是说新增加的元素再次查找的话是优于在它之前添加的同一个链表上的元素。这里涉及到就是扩容，也就是一旦元素的个数达到了扩容因子规定的数量(threhold=table.length*loadFactor)，就将数组扩大一倍。

**清单6 HashMap扩容过程**

```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable);
    table = newTable;
    threshold = (int)(newCapacity * loadFactor);
}

void transfer(Entry[] newTable) {
    Entry[] src = table;
    int newCapacity = newTable.length;
    for (int j = 0; j < src.length; j++) {
        Entry<K,V> e = src[j];
        if (e != null) {
            src[j] = null;
            do {
                Entry<K,V> next = e.next;
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            } while (e != null);
        }
    }
}
```

清单6 描述的是HashMap扩容的过程。可以看到扩充过程会导致元素数据的所有元素进行重新hash计算，这个过程也叫rehash。显然这是一个非常耗时的过程，否则扩容都会导致所有元素重新计算hash。因此尽可能的选择合适的初始化大小是有效提高HashMap效率的关键。太大了会导致过多的浪费空间，太小了就可能会导致繁重的rehash过程。在这个过程中loadFactor也可以考虑。

举个例子来说，如果要存储1000个元素，采用默认扩容因子0.75，那么1024显然是不够的，因为1000>0.75*1024了，所以选择2048是必须的，显然浪费了1048个空间。如果确定最多只有1000个元素，那么扩容因子为1，那么1024是不错的选择。另外需要强调的一点是扩容因此越大，从统计学角度讲意味着链表的长度就也大，也就是在查找元素的时候就需要更多次的循环。所以凡事必然是一个平衡的过程。

这里可能有人要问题，一旦我将Map的容量扩大后（也就是数组的大小），这个容量还能减小么？比如说刚开始Map中可能有10000个元素，运行一旦时间以后Map的大小永远不会超过10个，那么Map的容量能减小到10个或者16个么？答案就是不能，这个capacity一旦扩大后就不能减小了，只能通过构造一个新的Map来控制capacity了。

 

HashMap的几个内部迭代器也是非常重要的，这里限于篇幅就不再展开了，有兴趣的可以自己研究下。

Hashtable的原理和HashMap的原理几乎一样，所以就不讨论了。另外LinkedHashMap是在Map.Entry的基础上增加了before/after两个双向索引，用来将所有Map.Entry串联起来，这样就可以遍历或者做LRU Cache等。这里也不再展开讨论了。

 

[memcached](http://memcached.org/) 内部数据结构就是采用了HashMap类似的思想来实现的，有兴趣的可以参考资料8,9，10。

参考资料：

1. [HashMap hash方法分析](http://www.javaeye.com/topic/709945)
2. [通过分析 JDK 源代码研究 Hash 存储机制](http://www.ibm.com/developerworks/cn/java/j-lo-hash/)
3. [Java 理论与实践: 哈希](http://www.ibm.com/developerworks/cn/java/j-jtp05273/)
4. [Java 理论与实践: 构建一个更好的 HashMap](http://www.ibm.com/developerworks/cn/java/j-jtp08223/)
5. [jdk1.6 ConcurrentHashMap](http://yk94wo.blog.sohu.com/155835132.html)
6. [ConcurrentHashMap之实现细节](http://www.javaeye.com/topic/344876)
7. [深入理解HashMap](http://www.javaeye.com/topic/539465)
8. [memcached-数据结构](http://www.lampchina.net/article/htmls/201005/Mjg1MTYy.html)
9. [memcached存储管理 数据结构](http://www.cublog.cn/u/20146/showart_1820089.html)
10. [memcached](http://memcached.org/)

### ConcurrentHashMap原理

在[读写锁章节部分](http://www.blogjava.net/xylz/archive/2010/07/14/326080.html)介绍过一种是用读写锁实现Map的方法。此种方法看起来可以实现Map响应的功能，而且吞吐量也应该不错。但是通过前面对[读写锁原理](http://www.blogjava.net/xylz/archive/2010/07/15/326152.html)的分析后知道，读写锁的适合场景是读操作>>写操作，也就是读操作应该占据大部分操作，另外读写锁存在一个很严重的问题是读写操作不能同时发生。要想解决读写同时进行问题（至少不同元素的读写分离），那么就只能将锁拆分，不同的元素拥有不同的锁，这种技术就是“锁分离”技术。

默认情况下ConcurrentHashMap是用了16个类似HashMap 的结构，其中每一个HashMap拥有一个独占锁。也就是说最终的效果就是通过某种Hash算法，将任何一个元素均匀的映射到某个HashMap的Map.Entry上面，而对某个一个元素的操作就集中在其分布的HashMap上，与其它HashMap无关。这样就支持最多16个并发的写操作。 [![image](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency18part3ConcurrentMap3_693/image_thumb_3.png)](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency18part3ConcurrentMap3_693/image_8.png)

上图就是ConcurrentHashMap的类图。参考上面的说明和HashMap的原理分析，可以看到ConcurrentHashMap将整个对象列表分为segmentMask+1个片段（Segment）。其中每一个片段是一个类似于HashMap的结构，它有一个HashEntry的数组，数组的每一项又是一个链表，通过HashEntry的next引用串联起来。

这个类图上面的数据结构的定义非常有学问，接下来会一个个有针对性的分析。

首先如何从ConcurrentHashMap定位到HashEntry。在HashMap的原理分析部分说过，对于一个Hash的数据结构来说，为了减少浪费的空间和快速定位数据，那么就需要数据在Hash上的分布比较均匀。对于一次Map的查找来说，首先就需要定位到Segment，然后从过Segment定位到HashEntry链表，最后才是通过遍历链表得到需要的元素。

在不讨论并发的前提下先来讨论如何定位到HashEntry的。在ConcurrentHashMap中是通过hash(key.hashCode())和segmentFor(hash)来得到Segment的。清单1 描述了如何定位Segment的过程。其中hash(int)是将key的hashCode进行二次编码，使之能够在segmentMask+1个Segment上均匀分布（默认是16个）。可以看到的是这里和HashMap还是有点不同的，这里采用的算法叫Wang/Jenkins hash，有兴趣的可以[参考资料1](http://tech.puredanger.com/2007/07/25/hash/)和[参考资料2](http://www.goworkday.com/2010/03/19/single-word-wangjenkins-hash-concurrenthashmap/)。总之它的目的就是使元素能够均匀的分布在不同的Segment上，这样才能够支持最多segmentMask+1个并发，这里segmentMask+1是segments的大小。

**清单1 定位Segment**

```java
private static int hash(int h) {
    // Spread bits to regularize both segment and index locations,
    // using variant of single-word Wang/Jenkins hash.
    h += (h <<  15) ^ 0xffffcd7d;
    h ^= (h >>> 10);
    h += (h <<   3);
    h ^= (h >>>  6);
    h += (h <<   2) + (h << 14);
    return h ^ (h >>> 16);
}
final Segment<K,V> segmentFor(int hash) {
    return segments[(hash >>> segmentShift) & segmentMask];
}
```

显然在不能够对Segment扩容的情况下，segments的大小就应该是固定的。所以在ConcurrentHashMap中segments/segmentMask/segmentShift都是常量，一旦初始化后就不能被再次修改，其中segmentShift是查找Segment的一个常量偏移量。

有了Segment以后再定位HashEntry就和HashMap中定位HashEntry一样了，先将hash值与Segment中HashEntry的大小减1进行与操作定位到HashEntry链表，然后遍历链表就可以完成相应的操作了。

 

能够定位元素以后ConcurrentHashMap就已经具有了HashMap的功能了，现在要解决的就是如何并发的问题。要解决并发问题，加锁是必不可免的。再回头看Segment的类图，可以看到Segment除了有一个volatile类型的元素大小count外，Segment还是集成自ReentrantLock的。另外在前面的原子操作和锁机制中介绍过，要想最大限度的支持并发，那么能够利用的思路就是尽量读操作不加锁，写操作不加锁。如果是读操作不加锁，写操作加锁，对于竞争资源来说就需要定义为[volatile](http://www.blogjava.net/xylz/archive/2010/07/03/325168.html)类型的。[volatile](http://www.blogjava.net/xylz/archive/2010/07/03/325168.html)类型能够保证[happens-before法则](http://www.blogjava.net/xylz/archive/2010/07/03/325168.html)，所以volatile能够近似保证正确性的情况下最大程度的降低加锁带来的影响，同时还与写操作的锁不产生冲突。

同时为了防止在遍历HashEntry的时候被破坏，那么对于HashEntry的数据结构来说，除了value之外其他属性就应该是常量，否则不可避免的会得到ConcurrentModificationException。这就是为什么HashEntry数据结构中key,hash,next是常量的原因(final类型）。

有了上面的分析和条件后再来看Segment的get/put/remove就容易多了。

**get操作**

**清单2 Segment定位元素**

```java
V get(Object key, int hash) {
    if (count != 0) { // read-volatile
        HashEntry<K,V> e = getFirst(hash);
        while (e != null) {
            if (e.hash == hash && key.equals(e.key)) {
                V v = e.value;
                if (v != null)
                    return v;
                return readValueUnderLock(e); // recheck
            }
            e = e.next;
        }
    }
    return null;
}
HashEntry<K,V> getFirst(int hash) {
    HashEntry<K,V>[] tab = table;
    return tab[hash & (tab.length - 1)];
}

V readValueUnderLock(HashEntry<K,V> e) {
    lock();
    try {
        return e.value;
    } finally {
        unlock();
    }
}
```

清单2 描述的是Segment如何定位元素。首先判断Segment的大小count>0，Segment的大小描述的是HashEntry不为空(key不为空)的个数。如果Segment中存在元素那么就通过getFirst定位到指定的HashEntry链表的头节点上，然后遍历此节点，一旦找到key对应的元素后就返回其对应的值。但是在清单2 中可以看到拿到HashEntry的value后还进行了一次判断操作，如果为空还需要加锁再读取一次（readValueUnderLock）。为什么会有这样的操作？尽管ConcurrentHashMap不允许将value为null的值加入，但现在仍然能够读到一个为空的value就意味着此值对当前线程还不可见（这是因为HashEntry还没有完全构造完成就赋值导致的，后面还会谈到此机制）。

**put操作**

清单3 描述的是Segment的put操作。首先就需要加锁了，修改一个竞争资源肯定是要加锁的，这个毫无疑问。需要说明的是Segment集成的是ReentrantLock，所以这里加的锁也就是独占锁，也就是说同一个Segment在同一时刻只有能一个put操作。

接下来来就是检查是否需要扩容，这和HashMap一样，如果需要的话就扩大一倍，同时进行rehash操作。

查找元素就和get操作是一样的，得到元素就直接修改其值就好了。这里onlyIfAbsent只是为了实现ConcurrentMap的putIfAbsent操作而已。需要说明以下几点：

- 如果找到key对于的HashEntry后直接修改就好了，如果找不到那么就需要构造一个新的HashEntry出来加到hash对于的HashEntry的头部，同时就的头部就加到新的头部后面。这是因为HashEntry的next是final类型的，所以只能修改头节点才能加元素加入链表中。
- 如果增加了新的操作后，就需要将count+1写回去。前面说过count是volatile类型，而读取操作没有加锁，所以只能把元素真正写回Segment中的时候才能修改count值，这个要放到整个操作的最后。
- 在将新的HashEntry写入table中时是通过构造函数来设置value值的，这意味对table的赋值可能在设置value之前，也就是说得到了一个半构造完的HashEntry。这就是重排序可能引起的问题。所以在读取操作中，一旦读到了一个value为空的value是就需要加锁重新读取一次。为什么要加锁？加锁意味着前一个写操作的锁释放，也就是前一个锁的数据已经完成写完了了，根据happens-before法则，前一个写操作的结果对当前读线程就可见了。当然在JDK 6.0以后不一定存在此问题。
- 在Segment中table变量是volatile类型，多次读取volatile类型的开销要不非volatile开销要大，而且编译器也无法优化，所以在put操作中首先建立一个临时变量tab指向table，多次读写tab的效率要比volatile类型的table要高，JVM也能够对此进行优化。

**清单3 Segment的put操作**

```java
V put(K key, int hash, V value, boolean onlyIfAbsent) {
    lock();
    try {
        int c = count;
        if (c++ > threshold) // ensure capacity
            rehash();
        HashEntry<K,V>[] tab = table;
        int index = hash & (tab.length - 1);
        HashEntry<K,V> first = tab[index];
        HashEntry<K,V> e = first;
        while (e != null && (e.hash != hash || !key.equals(e.key)))
            e = e.next;

        V oldValue;
        if (e != null) {
            oldValue = e.value;
            if (!onlyIfAbsent)
                e.value = value;
        }
        else {
            oldValue = null;
            ++modCount;
            tab[index] = new HashEntry<K,V>(key, hash, first, value);
            count = c; // write-volatile
        }
        return oldValue;
    } finally {
        unlock();
    }
}
```

**remove 操作**

 

清单4 描述了Segment删除一个元素的过程。同put一样，remove也需要加锁，这是因为对table可能会有变更。由于HashEntry的next节点是final类型的，所以一旦删除链表中间一个元素，就需要将删除之前或者之后的元素重新加入新的链表。而Segment采用的是将删除元素之前的元素一个个重新加入删除之后的元素之前（也就是链表头结点）来完成新链表的构造。

**清单4 Segment的remove操作**

```java
V remove(Object key, int hash, Object value) {
    lock();
    try {
        int c = count - 1;
        HashEntry<K,V>[] tab = table;
        int index = hash & (tab.length - 1);
        HashEntry<K,V> first = tab[index];
        HashEntry<K,V> e = first;
        while (e != null && (e.hash != hash || !key.equals(e.key)))
            e = e.next;

        V oldValue = null;
        if (e != null) {
            V v = e.value;
            if (value == null || value.equals(v)) {
                oldValue = v;
                // All entries following removed node can stay
                // in list, but all preceding ones need to be
                // cloned.
                ++modCount;
                HashEntry<K,V> newFirst = e.next;
                for (HashEntry<K,V> p = first; p != e; p = p.next)
                    newFirst = new HashEntry<K,V>(p.key, p.hash,
                                                  newFirst, p.value);
                tab[index] = newFirst;
                count = c; // write-volatile
            }
        }
        return oldValue;
    } finally {
        unlock();
    }
}
```

下面的示意图描述了如何删除一个已经存在的元素的。假设我们要删除B3元素。首先定位到B3所在的Segment，然后再定位到Segment的table中的B1元素，也就是Bx所在的链表。然后遍历链表找到B3，找到之后就从头结点B1开始构建新的节点B1（蓝色）加到B4的前面，继续B1后面的节点B2构造B2（蓝色），加到由蓝色的B1和B4构成的新的链表。继续下去，直到遇到B3后终止，这样就构造出来一个新的链表B2（蓝色）->B1（蓝色）->B4->B5，然后将此链表的头结点B2（蓝色）设置到Segment的table中。这样就完成了元素B3的删除操作。需要说明的是，尽管就的链表仍然存在(B1->B2->B3->B4->B5)，但是由于没有引用指向此链表，所以此链表中无引用的（B1->B2->B3）最终会被GC回收掉。这样做的一个好处是，如果某个读操作在删除时已经定位到了旧的链表上，那么此操作仍然将能读到数据，只不过读取到的是旧数据而已，这在多线程里面是没有问题的。

[![image](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency18part3ConcurrentMap3_693/image_thumb_4.png)](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency18part3ConcurrentMap3_693/image_10.png)[![image](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency18part3ConcurrentMap3_693/image_thumb_5.png)](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency18part3ConcurrentMap3_693/image_12.png)

除了对单个元素操作外，还有对全部的Segment的操作，比如size()操作等。

**size操作**

size操作涉及到统计所有Segment的大小，这样就会遍历所有的Segment，如果每次加锁就会导致整个Map都被锁住了，任何需要锁的操作都将无法进行。这里用到了一个比较巧妙的方案解决此问题。

在Segment中有一个变量modCount，用来记录Segment结构变更的次数，结构变更包括增加元素和删除元素，每增加一个元素操作就+1，每进行一次删除操作+1，每进行一次清空操作(clear)就+1。也就是说每次涉及到元素个数变更的操作modCount都会+1，而且一直是增大的，不会减小。

遍历两次ConcurrentHashMap中的segments，每次遍历是记录每一个Segment的modCount，比较两次遍历的modCount值的和是否相同，如果相同就返回在遍历过程中获取的Segment的count的和，也就是所有元素的个数。如果不相同就重复再做一次。重复一次还不相同就将所有Segment锁住，一个一个的获取其大小(count)，最后将这些count加起来得到总的大小。当然了最后需要将锁一一释放。清单5 描述了这个过程。

这里有一个比较高级的话题是为什么在读取modCount的时候总是先要读取count一下。为什么不是先读取modCount然后再读取count的呢？也就是说下面的两条语句能否交换下顺序？

> sum += segments[i].count;
>
> mcsum += mc[i] = segments[i].modCount;

答案是不能！为什么？这是因为modCount总是在加锁的情况下才发生变化，所以不会发生多线程同时修改的情况，也就是没必要时volatile类型。另外总是在count修改的情况下修改modCount，而count是一个volatile变量。于是这里就充分利用了volatile的特性。

根据[happens-before法则](http://www.blogjava.net/xylz/archive/2010/07/03/325168.html)，第（3）条：对volatile字段的写入操作happens-before于每一个后续的同一个字段的读操作。也就是说一个操作C在volatile字段的写操作之后，那么volatile写操作之前的所有操作都对此操作C可见。所以修改modCount总是在修改count之前，也就是说如果读取到了一个count的值，那么在count变化之前的modCount也就能够读取到，换句话说就是如果看到了count值的变化，那么就一定看到了modCount值的变化。而如果上面两条语句交换下顺序就无法保证这个结果一定存在了。

在ConcurrentHashMap.containsValue中，可以看到每次遍历segments时都会执行int c = segments[i].count;，但是接下来的语句中又不用此变量c，尽管如此JVM仍然不能将此语句优化掉，因为这是一个volatile字段的读取操作，它保证了一些列操作的happens-before顺序，所以是至关重要的。在这里可以看到：

> ConcurrentHashMap将volatile发挥到了极致！

另外isEmpty操作于size操作类似，不再累述。

**清单5 ConcurrentHashMap的size操作**

```java
public int size() {
    final Segment<K,V>[] segments = this.segments;
    long sum = 0;
    long check = 0;
    int[] mc = new int[segments.length];
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    for (int k = 0; k < RETRIES_BEFORE_LOCK; ++k) {
        check = 0;
        sum = 0;
        int mcsum = 0;
        for (int i = 0; i < segments.length; ++i) {
            sum += segments[i].count;
            mcsum += mc[i] = segments[i].modCount;
        }
        if (mcsum != 0) {
            for (int i = 0; i < segments.length; ++i) {
                check += segments[i].count;
                if (mc[i] != segments[i].modCount) {
                    check = -1; // force retry
                    break;
                }
            }
        }
        if (check == sum)
            break;
    }
    if (check != sum) { // Resort to locking all segments
        sum = 0;
        for (int i = 0; i < segments.length; ++i)
            segments[i].lock();
        for (int i = 0; i < segments.length; ++i)
            sum += segments[i].count;
        for (int i = 0; i < segments.length; ++i)
            segments[i].unlock();
    }
    if (sum > Integer.MAX_VALUE)
        return Integer.MAX_VALUE;
    else
        return (int)sum;
}
```

参考资料：

1. [Hash this](http://tech.puredanger.com/2007/07/25/hash/)
2. [Single-word Wang/Jenkins Hash in ConcurrentHashMap](http://www.goworkday.com/2010/03/19/single-word-wangjenkins-hash-concurrenthashmap/)
3. [指令重排序与happens-before法则](http://www.blogjava.net/xylz/archive/2010/07/03/325168.html)
4. [通过分析 JDK 源代码研究 TreeMap 红黑树算法实现](http://www.ibm.com/developerworks/cn/java/j-lo-tree/index.html?ca=drs-)

## 并发队列与Queue

Queue是JDK 5以后引入的新的集合类，它属于Java Collections Framework的成员，在Collection集合中和List/Set是同一级别的接口。通常来讲Queue描述的是一种FIFO的队列，当然不全都是，比如PriorityQueue是按照优先级的顺序（或者说是自然顺序，借助于Comparator接口）。

下图描述了Java Collections Framework中Queue的整个家族体系。

对于Queue而言是在Collection的基础上增加了offer/remove/poll/element/peek方法，另外重新定义了add方法。对于这六个方法，有不同的定义。

| **** | **抛出异常**  | **返回特殊值** | **操作描述**         |
| ---- | --------- | --------- | ---------------- |
| 插入   | add(e)    | offer(e)  | 将元素加入到队列尾部       |
| 移除   | remove()  | poll()    | 移除队列头部的元素        |
| 检查   | element() | peek()    | 返回队列头部的元素而不移除此元素 |

特别说明的是对于Queue而言，规范并没有规定是线程安全的，为了解决这个问题，引入了可阻塞的队列BlockingQueue。对于BlockingQueue而言所有操作的是线程安全的，并且队列的操作可以被阻塞，直到满足某种条件。Queue的另一个子接口Deque描述的是一个双向的队列。与Queue不同的是，Deque允许在队列的头部增加元素和在队列的尾部删除元素。也就是说Deque是一个双向队列。二者功能都有的队列就是BlockingDeque，这种阻塞队列允许在队列的头和尾部分别操作元素，应该说是Queue中功能最强大的实现。

 

[![image](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/ead4e8800e0c_FD45/image_thumb_1.png)](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/ead4e8800e0c_FD45/image_4.png)

在JDK 5之前LinkedList就已经存在，而且本身实现都是一种双向队列。所以到了JDK 5以后就将LinkedList同时实现Deque接口，这样LinkedList就又属于Queue的一部分了。

通常情况下Queue都是靠链表结构实现的，但是链表意味着有一些而外的引用开销，如果是双向链表开销就更大了。所以为了节省内存，一种方式就是使用固定大小的数组来实现队列。在这种情况下队列的大小是固定，元素的遍历通过数组的索引进行，很显然这是一种双向链表的模型。ArrayDeque就是这样一种实现。

另外ArrayBlockingQueue也是一种数组实现的队列，但是却没有改造成双向，仅仅实现了BlockingQueue的模型。理论上和ArrayDeque一样也应该容易改造成双向的实现。

PriorityQueue和PriorityBlockingQueue实现了一种排序的队列模型。这很类似与SortedSet，通过队列的Comparator接口或者Comparable元素来排序元素。这种情况下元素在队列中的出入就不是按照FIFO的形式，而是根据比较后的自然顺序来进行。

CocurrentLinkedQueue是一种线程安全却非阻塞的FIFO队列，这种队列通常实现起来比较简单，但是却很有效。在接下来的章节会详细的描述它。

SynchronousQueue是一种特别的BlockingQueue，它只是把一个add/offer操作的元素直接移交给remove/take操作。也就是说它本身不会缓存任何元素，所以严格意义上说来讲并不是一种真正的队列。此队列维护一个线程列表，这些线程等待从队列中加入元素或者移除元素。简单的说，至少有一个remove/take操作时add/offer操作才能成功，同样至少有一个add/offer操作时remove/take操作才能成功。这是一种双向等待的队列模型，出队列等待加入等列，而入队列又等待出队列。这种队列的好处在于能够最大线程的保持吞吐量却又是线程安全的。所以对于一个需要快速处理的任务队列，SynchronousQueue是一个不错的选择。

BlockingQueue还有一种实现DelayQueue，这种实现允许每一个元素(Delayed)带有一个延时时间，当调用take/poll的时候会检测队列头元素这个时间是否<=0，如果满足就是说已经超时了，那么此元素就可以被移除了，否则就会等待。特别说明的是这个头元素应该是最先被超时的元素（这个时间是绝对时间）。这个类设计很巧妙，被用于ScheduledFutureTask来进行定时操作。希望后面会开辟一个章节讲讲这里面的想法。实在不行在讲线程池部分肯定会提到这个。

## ConcurrentLinkedQueue

ConcurrentLinkedQueue是Queue的一个线程安全实现。先来看一段文档说明。

> 一个基于链接节点的无界线程安全队列。此队列按照 FIFO（先进先出）原则对元素进行排序。队列的头部 是队列中时间最长的元素。队列的尾部 是队列中时间最短的元素。新的元素插入到队列的尾部，队列获取操作从队列头部获得元素。当多个线程共享访问一个公共 collection 时，ConcurrentLinkedQueue 是一个恰当的选择。此队列不允许使用 null 元素。

 

由于ConcurrentLinkedQueue只是简单的实现了一个队列Queue，因此从API的角度讲，没有多少值的介绍，使用起来也很简单，和前面遇到的所有FIFO队列都类似。出队列只能操作头节点，入队列只能操作尾节点，任意节点操作就需要遍历完整的队列。

重点放在解释ConcurrentLinkedQueue的原理和实现上。

在继续探讨之前，结合前面线程安全的相关知识，我来分析设计一个线程安全的队列哪几种方法。

第一种：使用synchronized同步队列，就像Vector或者Collections.synchronizedList/Collection那样。显然这不是一个好的并发队列，这会导致吞吐量急剧下降。

第二种：使用Lock。一种好的实现方式是使用ReentrantReadWriteLock来代替ReentrantLock提高读取的吞吐量。但是显然ReentrantReadWriteLock的实现更为复杂，而且更容易导致出现问题，另外也不是一种通用的实现方式，因为ReentrantReadWriteLock适合哪种读取量远远大于写入量的场合。当然了ReentrantLock是一种很好的实现，结合Condition能够很方便的实现阻塞功能，这在后面介绍BlockingQueue的时候会具体分析。

第三种：使用CAS操作。尽管Lock的实现也用到了CAS操作，但是毕竟是间接操作，而且会导致线程挂起。一个好的并发队列就是采用某种非阻塞算法来取得最大的吞吐量。

ConcurrentLinkedQueue采用的就是第三种策略。它采用了[参考资料1](http://www.cs.rochester.edu/u/scott/papers/1996_PODC_queues.pdf) 中的算法。

在锁机制中谈到过，要使用非阻塞算法来完成队列操作，那么就需要一种“循环尝试”的动作，就是循环操作队列，直到成功为止，失败就会再次尝试。这在前面的章节中多次介绍过。

针对各种功能深入分析。

在开始之前先介绍下ConcurrentLinkedQueue的数据结构。

[![image](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency20part5ConcurrentLinkedQu_C9AC/image_thumb.png)](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency20part5ConcurrentLinkedQu_C9AC/image_2.png)在上面的数据结构中，ConcurrentLinkedQueue只有头结点、尾节点两个元素，而对于一个节点Node而言除了保存队列元素item外，还有一个指向下一个节点的引用next。 看起来整个数据结构还是比较简单的。但是也有几点是需要说明：

1. 所有结构（head/tail/item/next）都是volatile类型。 这是因为ConcurrentLinkedQueue是非阻塞的，所以只有volatile才能使变量的写操作对后续读操作是可见的（这个是有happens-before法则保证的）。同样也不会导致指令的重排序。
2. 所有结构的操作都带有原子操作，这是由AtomicReferenceFieldUpdater保证的，这在原子操作中介绍过。它能保证需要的时候对变量的修改操作是原子的。
3. 由于队列中任何一个节点（Node）只有下一个节点的引用，所以这个队列是单向的，根据FIFO特性，也就是说出队列在头部(head)，入队列在尾部(tail)。头部保存有进入队列最长时间的元素，尾部是最近进入的元素。
4. 没有对队列长度进行计数，所以队列的长度是无限的，同时获取队列的长度的时间不是固定的，这需要遍历整个队列，并且这个计数也可能是不精确的。
5. 初始情况下队列头和队列尾都指向一个空节点，但是非null，这是为了方便操作，不需要每次去判断head/tail是否为空。但是head却不作为存取元素的节点，tail在不等于head情况下保存一个节点元素。也就是说head.item这个应该一直是空，但是tail.item却不一定是空（如果head!=tail，那么tail.item!=null）。

对于第5点，可以从ConcurrentLinkedQueue的初始化中看到。这种头结点也叫“伪节点”，也就是说它不是真正的节点，只是一标识，就像c中的字符数组后面的\0以后，只是用来标识结束，并不是真正字符数组的一部分。

**清单1 入队列操作**

```java
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    Node<E> n = new Node<E>(e, null);
    for (;;) {
        Node<E> t = tail;
        Node<E> s = t.getNext();
        if (t == tail) {
            if (s == null) {
                if (t.casNext(s, n)) {
                    casTail(t, n);
                    return true;
                }
            } else {
                casTail(t, s);
            }
        }
    }
}
```

清单1 描述的是入队列的过程。整个过程是这样的。

1. 1. 获取尾节点t，以及尾节点的下一个节点s。如果尾节点没有被别人修改，也就是t==tail，进行2，否则进行1。
   2. 如果s不为空，也就是说此时尾节点后面还有元素，那么就需要把尾节点往后移，进行1。否则进行3。
   3. 修改尾节点的下一个节点为新节点，如果成功就修改尾节点，返回true。否则进行1。

从操作3中可以看到是先修改尾节点的下一个节点，然后才修改尾节点位置的，所以这才有操作2中为什么获取到的尾节点的下一个节点不为空的原因。

特别需要说明的是，对尾节点的tail的操作需要换成临时变量t和s，一方面是为了去掉volatile变量的可变性，另一方面是为了减少volatile的性能影响。

清单2 描述的出队列的过程，这个过程和入队列相似，有点意思。

头结点是为了标识队列起始，也为了减少空指针的比较，所以头结点总是一个item为null的非null节点。也就是说head!=null并且head.item==null总是成立。所以实际上获取的是head.next，一旦将头结点head设置为head.next成功就将新head的item设置为null。至于以前就的头结点h，h.item=null并且h.next为新的head，但是由于没有对h的引用，所以最终会被GC回收。这就是整个出队列的过程。

**清单2 出队列操作**

```java
public E poll() {
    for (;;) {
        Node<E> h = head;
        Node<E> t = tail;
        Node<E> first = h.getNext();
        if (h == head) {
            if (h == t) {
                if (first == null)
                    return null;
                else
                    casTail(t, first);
            } else if (casHead(h, first)) {
                E item = first.getItem();
                if (item != null) {
                    first.setItem(null);
                    return item;
                }
                // else skip over deleted item, continue loop,
            }
        }
    }
}
```

另外对于清单3 描述的获取队列大小的过程，由于没有一个计数器来对队列大小计数，所以获取队列的大小只能通过从头到尾完整的遍历队列，显然这个代价是很大的。所以通常情况下ConcurrentLinkedQueue需要和一个AtomicInteger搭配才能获取队列大小。后面介绍的BlockingQueue正是使用了这种思想。

**清单3 遍历队列大小**

```java
public int size() {
    int count = 0;
    for (Node<E> p = first(); p != null; p = p.getNext()) {
        if (p.getItem() != null) {
            // Collections.size() spec says to max out
            if (++count == Integer.MAX_VALUE)
                break;
        }
    }
    return count;
}
```

参考资料：

1. [Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms](http://www.cs.rochester.edu/u/scott/papers/1996_PODC_queues.pdf)
2. [多线程基础总结十一—ConcurrentLinkedQueue](http://yanxuxin.javaeye.com/blog/586943)
3. [对ConcurrentLinkedQueue进行的并发测试](http://www.javaeye.com/topic/68279) 

## BlockingQueue

BlockingQueue相对于Queue而言增加了两个操作：put/take。下面是一张整理的表格。

[![image](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency21part5ConcurrentLinkedQu_E370/image_thumb_1.png)](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency21part5ConcurrentLinkedQu_E370/image_4.png)看似简单的API，非常有用。这在控制队列的并发上非常有好处。既然加入队列和移除队列能够被阻塞，这在实现生产者-消费者模型上就简单多了。

清单1 是生产者-消费者模型的一个例子。这个例子是一个真实的场景。服务端（ICE服务）接受客户端的请求(accept)，请求计算此人的好友生日，然后将计算的结果存取缓存中（Memcache）中。在这个例子中采用了ExecutorService实现多线程的功能，尽可能的提高吞吐量，这个在后面线程池的部分会详细说明。目前就可以理解为new Thread(r).start()就可以了。另外这里阻塞队列使用的是LinkedBlockingQueue。

**清单1 一个生产者-消费者例子**

```java
public class BirthdayService {

    final int workerNumber;

    final Worker[] workers;

    final ExecutorService threadPool;

    static volatile boolean running = true;

    public BirthdayService(int workerNumber, int capacity) {
        if (workerNumber <= 0) throw new IllegalArgumentException();
        this.workerNumber = workerNumber;
        workers = new Worker[workerNumber];
        for (int i = 0; i < workerNumber; i++) {
            workers[i] = new Worker(capacity);
        }
        //
        boolean b = running;// kill the resorting
        threadPool = Executors.newFixedThreadPool(workerNumber);
        for (Worker w : workers) {
            threadPool.submit(w);
        }
    }

    Worker getWorker(int id) {
        return workers[id % workerNumber];

    }

    class Worker implements Runnable {

        final BlockingQueue<Integer> queue;

        public Worker(int capacity) {
            queue = new LinkedBlockingQueue<Integer>(capacity);
        }

        public void run() {
            while (true) {
                try {
                    consume(queue.take());
                } catch (InterruptedException e) {
                    return;
                }
            }
        }

        void put(int id) {
            try {
                queue.put(id);
            } catch (InterruptedException e) {
                return;
            }
        }
    }

    public void accept(int id) {
        //accept client request
        getWorker(id).put(id);
    }

    protected void consume(int id) {
        //do the work
        //get the list of friends and save the birthday to cache
    }
}
```



 后续略……

参考：

1. [并发容器 part 1 ConcurrentMap (1)](http://www.blogjava.net/xylz/archive/2010/07/19/326527.html)
2. [并发容器 part 2 ConcurrentMap (2)](http://www.blogjava.net/xylz/archive/2010/07/20/326584.html)
3. [并发容器 part 3 ConcurrentMap (3)](http://www.blogjava.net/xylz/archive/2010/07/20/326661.html)
4. [并发容器 part 4 并发队列与Queue简介](http://www.blogjava.net/xylz/archive/2010/07/21/326723.html)
5. [并发容器 part 5 ConcurrentLinkedQueue](http://www.blogjava.net/xylz/archive/2010/07/23/326934.html)
6. [并发容器 part 6 可阻塞的BlockingQueue (1)](http://www.blogjava.net/xylz/archive/2010/07/24/326988.html)
7. [并发容器 part 7 可阻塞的BlockingQueue (2)](http://www.blogjava.net/xylz/archive/2010/07/27/327265.html)
8. [并发容器 part 8 可阻塞的BlockingQueue (3)](http://www.blogjava.net/xylz/archive/2010/07/30/327582.html)
9. [并发容器 part 9 双向队列集合 Deque](http://www.blogjava.net/xylz/archive/2010/08/12/328587.html)
10. [并发容器 part 10 双向并发阻塞队列 BlockingDeque](http://www.blogjava.net/xylz/archive/2010/08/18/329227.html)
11. [并发容器 part 11 Exchanger](http://www.blogjava.net/xylz/archive/2010/11/22/338733.html)
12. [并发容器 part 12 线程安全的List/Set CopyOnWriteArrayList/CopyOnWriteArraySet](http://www.blogjava.net/xylz/archive/2010/11/23/338853.html)



