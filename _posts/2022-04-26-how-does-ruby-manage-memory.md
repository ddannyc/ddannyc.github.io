---
title: Ruby的内存管理
date: 2022-04-26 10:12:22
tags: ruby
layout: post
categories: translation
---

原文地址: [How does Ruby manage memory?](https://blog.saeloun.com/2022/04/12/ruby-variable-width-allocation.rb)

在这个两篇的系列中，我们的目标是搞清楚Ruby内存管理背后的概念和深入得了解 [Variable Width Allocation](https://bugs.ruby-lang.org/issues/18045) 是如何改善Ruby内存性能的。

## RVALUE
对于动态内存分配，Ruby使用堆内存，在堆中最基本的单元是slot。每个slot中可以存储一个叫做**RVALUE**的结构。一个**RVALUE**占用40字节，是存储对象的容器(如Array,String,Class)，
在这40字节当中，起始8个字节作为标记位，接着8个字节是类指针，剩余24字节记录对象特定的字段。  
举个栗子，对于一个类对象，它存储的是指针和扩展对象，对于一个字符串对象，它存储的是内容。  
  
![rvalue](https://d33wubrfki0l68.cloudfront.net/85642274080a571512f90e5b6d35c5003ccdbf11/441c4/images/ruby-memory/r-value.jpg)

## Heap Pages
Heap pages由40字节的slot组织而成，是一段16kb的内存容器。每个Heap pages可以存放408至409个slot，所有存储在同一页的slot之间是连续。

![heap pages](https://d33wubrfki0l68.cloudfront.net/63bcbaa6706173a8a208c92d822420af0e371dcc/9f2a4/images/ruby-memory/heap-page.png)
  
## Freelist
最初，在创建Heap page的时候，所有的slot会填充入一个特殊的RVALUE类型的值**T_NONE**。
它表示一个空的slot，仅包含一个标记和指向下一个slot的指针**next**。
  
![heap page initially](https://d33wubrfki0l68.cloudfront.net/a200b6323a44dadc0596381b932ec18aae9b775e/b776f/images/ruby-memory/freelist1.png)
  
Heap page初始化的时候Ruby会设置一个**freelist**指针指向第一个slot，然后遍历每个slot。遍历的过程中设置一个当前slot的指针和一个指向上一个slot的指针，随着遍历到最后一个slot我们得到了**FreeList**链表。

![Heap page initially](https://d33wubrfki0l68.cloudfront.net/6a466eec2d1ecd56b1ad785ccec04c8d5fc0fa1d/37534/images/ruby-memory/freelist.gif)
  

## 创建对象
当我们需要创建对象的时候，Ruby会向Heap page索要一个空slot的地址。毫无疑问，Heap page总是返回freelist指针指向的空slot，然后更新freelist指针至下一个空slot并删除当前slot的连接。这将允许Ruby存放数据。freelist的作用就是让申请对象空间的操作在常量时间内完成，所以每次Ruby需要一个空slot的时候，Heap page只需要取到freelis指针并返回即可。

![allocating an object](https://d33wubrfki0l68.cloudfront.net/04a02afd788d5778a81d7cd39a12eabc0fbb8827/a2219/images/ruby-memory/alloc1.png)
  
分配一个RClass对象  
![allocating rclass](https://d33wubrfki0l68.cloudfront.net/086e852d3634f5dd961ffe56e6fd3b9535ea0f68/cddfe/images/ruby-memory/alloc2.png)
  
分配一个RString对象
![allocating string](https://d33wubrfki0l68.cloudfront.net/491312f971ea0a25016d3136068630e7db5b221d/c68c4/images/ruby-memory/alloc3.png)
  
分配一个RArray对象
![allocating array](https://d33wubrfki0l68.cloudfront.net/633c2756f93a4dcc45bde09b81b3f92e5871380d/ac5b9/images/ruby-memory/alloc4.png)

  
然后当所有slot都填满之后，**GC**会回收已经死掉的对象。
  
## Garbage collection
Ruby使用**Mark-Sweep-Compact**垃圾回收算法，当GC正在运行的时候，ruby代码会停止执行。让我们深入GC的每一个阶段看看：  
**Marking**: 这个阶段决定哪些对象继续存活哪些可以释放。首先，我们从全局变量，类，等等的根节点开始标记。  
接着标记它们关联的子节点直到标记盏为空。
  
假设我们有2个Heap pagges，每个Heap page包含4个slots，从A到C和从E到G。空白slots为需要释放的slots，黑色slots为需要标记的slots。这里箭头展示了指向的引用，举例，一个箭头从A到G表示对象A声明了一个实例变量G。
  
![gc-marking](https://d33wubrfki0l68.cloudfront.net/624e693d3ff49571b2944f4ed0a79359757fc102/9adeb/images/ruby-memory/mark1.png)
  
我们从根节点开始将子节点A和B放入标记盏。现在从盏中取出一个元素，标记这个元素并将子节点元素放入标记盏。这里提取元素A，并放入A的子元素G到标记盏。接着取出元素B，标记它并将子元素E放入标记盏然后一直循环直至标记盏为空。一旦所有对象被标记完成，就会进入清扫阶段。
  
![gc-sweeping](https://d33wubrfki0l68.cloudfront.net/d7268c15e432a6bb6fc02350d6b08c1346768004/12851/images/ruby-memory/mark.gif)
  
**Sweeping**: 这个阶段中所有未被标记的对象可以被GC回收。所以经过扫描阶段之后我们的heap pages变成这样:  
![after-marking](https://d33wubrfki0l68.cloudfront.net/8dcb6b078b211a16e7cb812657283900f9fbb35e/b327d/images/ruby-memory/sweep1.png)
  
现在，GC扫描所有heap pages，检查未标记对象，释放空间。在我们的例子中，C和F未标记，所以GC回收了它们的空间。
![after-sweeping](https://d33wubrfki0l68.cloudfront.net/34be1971f131404f99dc209da305ac19bf8b7c94/45e3a/images/ruby-memory/sweep2.png)
  
**Compacting**: 压实操作是将对象移动到heap page起始位置，它的好处是减少内存使用，加快GC速度，和提升写入性能。这涉及到了两个步骤:
  
* **Compact step**: 这里用到了两个游标，向前移动的**Free cursors**和向后的**Compact cursor**。所以也可以被称为**2 Finger algorithm**，当两个游标相遇时操作完成。
  
让我们通过一个例子来了解它。这里，白色箭头代表Free cursor黑色则代表Compact cursor。Free cursor从heap起始位置开始移动到第一个空白slot。然后Compact cursor从heap的末尾位置开始移动到第一个已使用的slot。之后将Compact cursor位置的对象移动至Free cursor并记录下对象被移动到的新地址。现在重复之前的步骤直到Free cursor和Compact cursor相遇，这个步骤就完成了。经过这个步骤以后，我们可以看到起始空白的slot已经被填充。
![compact-step](https://d33wubrfki0l68.cloudfront.net/a7e535212fe9d17b142f89b70e1a2f512f8b3aef/7b797/images/ruby-memory/compact.gif)
* **Update reference step**: 这一步，我们需要更新经过压实步骤被移动对象的指针，继续上一个例子：  
![update-reference](https://d33wubrfki0l68.cloudfront.net/e25cb08049b6ea2f25060c015a9082cb2061e5da/d81fb/images/ruby-memory/refer1.png)
现在我们通过一个游标线性扫描对象检查有没有需要更新的引用，所以在我们的例子中，A和B的引用现在说`Moved to Heap Page 1`，然后将引用更新到`G`和`E`正确的地址。
![update-reference-2](https://d33wubrfki0l68.cloudfront.net/c5d04e0d1b35388b47cd977f459672efba5f78da/6bc2d/images/ruby-memory/refer2.png)
  
在这篇博文中，我们介绍了Ruby如何管理内存和GC如何工作，下篇我们将介绍Variable Width Allocation怎么工作。