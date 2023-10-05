---
title: ruby-variable-width-allocation-part-2
date: 2022-05-05 16:33:46
tags: ruby
categories: translation
---

原文地址: [introduces Variable Width Allocation for Strings](https://blog.saeloun.com/2022/04/19/ruby-variable-width-allocation-part-2)

延续[上篇](https://blog.saeloun.com/2022/04/12/ruby-variable-width-allocation.rb)，在本篇中我们将深入了解[Variable Width Allocation](https://bugs.ruby-lang.org/issues/18045)是如何工作和改善Ruby内存性能的。了解**VWA**之前让我们先看下大型对象是如何分配的。

## Large objects on the Heap
如我们所知slot大小为40字节，其中只有24字节可以用来存放内容。其余16字节用于存放标记及RVALUE指针。现在来看一个例子，我们需要申请一个12字节和37字节的字符串。  
  
12字节的例子里，由于它少于24字节，Ruby将string的全部内容存储在对应slot中。
![12bytes](https://d33wubrfki0l68.cloudfront.net/12b38b88e92e95556627f29a07959f03810d003f/f518e/images/ruby-memory/large-object1.gif)
  
而37字节的例子里，由于超过了24字节，Ruby调用`malloc`从系统内存堆中申请37字节空间。然后保存对应的内存地址并设置`NO_EMBED`标记，表明当前对象内存未保存在当前slot中。
![37bytes](https://d33wubrfki0l68.cloudfront.net/c0e261482029fa852afff97fd205fe01769f8c34/4182f/images/ruby-memory/lballoc3.png)
  
### Ruby Heap的瓶颈
* 内容存放在heap slot以外的地方导致了较差的内存命中。
* `malloc`系统调用导致性能开销

## Caching
那么我们看看内存命中是如何被影响的。CPU有L1，L2和L3三级缓存。其中L1在内核本身当中，速度最快，但大小只有32Kb。L2快于L3容量是512Kb，L3速度最慢但有较大的容量可以达到32Mb。  
![cpu cache](https://d33wubrfki0l68.cloudfront.net/0efb37703dc31200b092b3fb394d1a3a2187ae29/4d212/images/ruby-memory/iballoc7.png)
  
从主内存获取数据的同时会将数据写入这些缓存之中。如果继续上面37字节字符串例子的话，为了缓存这个字符串，我们需要获取两次，首先从主内存获取内容地址为一次再从主内存到系统内存获取内容为一次。总共获取内容的大小为40(RVALUE) + 37(Content) = 77字节。
  
## Malloc
使用`Malloc`获取系统内存是一项昂贵的操作，会对系统造成性能开销。因此需要尽量减少`malloc`调用。同时`malloc`调用要求额外的空间存储headers元数据，进而增加内存消耗。
  
因此，为了克服上述瓶颈，**Variable Width Allocation**被引入。
  
## Variable Width Allocation
这个项目的主要目标就是改善Ruby的整体性能。因此，将内容直接放在RVALUE之后可以有效改善缓存命中，同时动态分配slot大小可以避免昂贵的`malloc`系统调用。  
  
让我们来了解**VWA**如何工作。  
  
我们知道Ruby heap被切分为页，每一页又被切分为固定40字节的slot。不同于40字节构成的heap pages，**VWA**引入了一种新的结构**Size Pool**。**Size Pool**是一种包含特定大小slot的Heap pages。这些slot的大小是[RVALUE乘2的倍数](https://github.com/ruby/ruby/pull/4933/files#diff-d1cee85c3b0e24a64519c11150abe26fd0b5d8628a23d356dd0b535ac4595d49R2339)，如40，80，160，320等等。  
  
下图是一个由不同大小slot构成heap pages的**Size Pool**。  
![size pool](https://d33wubrfki0l68.cloudfront.net/f2e9b672082f5195d6d36d5ff18bc01f39139574/5f783/images/ruby-memory/size-pool.png)
当需要申请一个同样37字节字符串对象的时，40 (RVALUE) + 37 (Content) = 77 bytes。  
根据[源码](https://github.com/ruby/ruby/pull/4933/files#diff-d1cee85c3b0e24a64519c11150abe26fd0b5d8628a23d356dd0b535ac4595d49R2435)，size pool索引计算公式如下：  
```c
slot_count = ceil ( total_size / sizeOf(R_VALUE) ) = ceil ( 77 / 40)
slot_count = 2

pool_index = ceil (log slot_count ) = ceil (log 2) = 1  // log with base 2
```
下一步，根据得到索引为1，slot大小为80的heap pages分配空间。由于在80字节的slot中申请了77字节的空间，所以会有3字节空间的浪费。但是不管如何，基于测试显示对整体内存及运行时性能仅有细微的影响。  
  
那么关于variable-width allocation如何工作就是这些。目前**VWA**仅支持Class和String类型。对于已知大小的字符串在分配时会将内容嵌入在对象里，对于未知大小和超大内容则会回退到到malloc heap的分配方式。  
  
同时，如果一个嵌入分配的字符串在运行时扩展到slot无法存放的大小会被转移到malloc heap。这意味着slot内的一些空间会被浪费。举例，根据**VWA**一个字符串初始被分配到160字节slot之中，运行时字符串扩展到了200字节，那么字符串内容会转移到malloc heap中存放，在原来的slot里， 40字节以外的120字节空间则会被浪费。  
  
关于**VWA**对于Ruby内存性能改善，[这里](https://bugs.ruby-lang.org/issues/18045#Benchmark-results)可以看到一些测试结果。  
  
想了解更多细节可以查看[这个PR](https://github.com/ruby/ruby/pull/4933)