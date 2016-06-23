---
layout:     post
title:      "Java集合框架"
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

![](https://raw.githubusercontent.com/wanglizhi/wanglizhi.github.io/master/img/2016-06-22/Java collection cheat sheet.png)

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



## HashMap

