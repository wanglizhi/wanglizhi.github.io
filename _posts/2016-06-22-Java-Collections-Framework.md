---
layout:     post
title:      "Java集合框架总结"
subtitle:   "Java Collections Framework"
date:       2016-06-22 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Java
    - Java Collections
---

> Java集合类在开发过程中到处都在使用，理解其内部实现原理和使用方法可以避免误用，同时对于Java语言、数据结构和算法的理解也会更加深入。有时间可以整理下C++的STL，两者做下对比。

## Java集合框架概览

#### Java集合类图

![](https://raw.githubusercontent.com/wanglizhi/wanglizhi.github.io/master/img/2016-06-22/Java-collection-cheat-sheet.png)

#### 常见集合类的基本信息

| **Collection**       | **Ordering** | **Random Access** | **Key-Value** | **Duplicate Elements** | **Null Element** | **Thread Safety** |
| -------------------- | ------------ | ----------------- | ------------- | ---------------------- | ---------------- | ----------------- |
| ArrayList            | Yes          | Yes               | No            | Yes                    | Yes              | No                |
| LinkedList           | Yes          | No                | No            | Yes                    | Yes              | No                |
| HashSet              | No           | No                | No            | No                     | Yes              | No                |
| TreeSet              | Yes          | No                | No            | No                     | No               | No                |
| HashMap              | No           | Yes               | Yes           | No                     | Yes              | No                |
| TreeMap              | Yes          | Yes               | Yes           | No                     | No               | No                |
| Vector               | Yes          | Yes               | No            | Yes                    | Yes              | Yes               |
| Hashtable            | No           | Yes               | Yes           | No                     | No               | Yes               |
| Properties           | No           | Yes               | Yes           | No                     | No               | Yes               |
| Stack                | Yes          | No                | No            | Yes                    | Yes              | Yes               |
| CopyOnWriteArrayList | Yes          | Yes               | No            | Yes                    | Yes              | Yes               |
| ConcurrentHashMap    | No           | Yes               | Yes           | No                     | No               | Yes               |
| CopyOnWriteArraySet  | No           | No                | No            | No                     | Yes              | Yes               |

在java的包java.util和java.util.concurrent里面定义了java的集合类框架。

从总体来看，将集合类划分为两大部分。一种是可以按照一定顺序进行迭代访问的集合类，另一种是通过键值对的映射关系进行访问的集合类。对于迭代访问的集合类来说，Collection接口继承自Iterable接口，定义了用于迭代访问的协议，还定义了一些作为集合类型比较通用的方法，比如说size, isEmpty, add, remove等，常见的Set、List、Queue等都继承了Collection接口。而键值对结构继承自Mapreduce接口，常见的如HashMap、TreeMap、HashTable、LinkedHashMap都继承自该接口

## List

在List中最常用的两个类就数ArrayList和LinkedList。他们两个的实现代表着数据结构中的两种种典型：线性表和链表。在这里，这个线性表是可以根据需要自动增长的。Java的库里面默认没有实现单链表，LinkedList实际上是一个双链表。

#### ArrayList



#### LinkedList



#### CopyOnWriteArrayList



#### Vector



#### Stack





## Queue





## Set



## Map







Collections



iterator

[Java Comparable & Comparator](http://shmilyaw-hotmail-com.iteye.com/blog/1439450)

