---
title: how-to-write-a-git-commit-message
date: 2022-04-27 14:24:27
tags:
categories: translation
---
原文: [how to write a git commit message](https://cbea.ms/git-commit)

# 关于Git commit message的7条规则(译文)

1. 主题和内容之间用一个空行分隔
2. 主题长度限制在50个字符以内
3. 主题以大写字母开头（针对英文commit message）
4. 主题结束不要写句号
5. 主题使用祈使语气描述
6. 内容长度控制在72个字符以内
7. 使用内容部分来解释What、Why

# 5. 主题使用祈使语气
用命令或指令的方式去说或者写，举例：
* 清洁你的房间
* 把门关上
* 把垃圾带走
  
上面你看到的7条规则都是用祈使语气写的（“将内容长度限制在72个字符以内”，等等）
  
祈使语气听起来可能有一点粗鲁，所以我们不太常用。但是用它来写commit message主题简直完美。
一个理由是，当你在利用Git的时候，Git本身就使用了祈使语气。
  
看个例子，当你使用 `git merge` 的时候，默认创建的commit message是：
``` 
Merge branch 'myfeature'
```
当用`git revert`：
```
Revert "Add the thing with the stuff"

This reverts commit cc87791524aedd593cff5a74532befe7ab69ce9d.
```
或者当你在github的pr点击“Merge”的时候：
```
Merge pull request #123 from someuser/somebranch
```
所以当你使用祈使语气写commit message的时候，你就遵循了Git内置的约定。举例：
* 重构子系统X使其易读
* 更新入门文档
* 删除废弃方法
* 发布1.0版本
  
刚开始用这种方式写的时候可能有点别扭。因为我们最常说话的方式是**陈述语气**，说的大都关于报告事实。
所以为什么通常我们的commit message看起来会是这样：
* ~~Fixed bug with Y~~
* ~~Changing behavior of X~~

和有时候commit message写得像是描述内容本身：

* ~~change file.size to File.size()~~
* ~~wemeet version 2.0 problems~~
  
为了移除这些让人疑惑的commit message，有这样一条规则。  
## 一个合适的Git commit主题应该总是能够组成以下句子：
* 如果生效，这个commit将 <u>你的commit message主题</u>  

举例：
* 如果生效，这个commit将 *重构子系统X使其易读*
* 如果生效，这个commit将 *更新入门文档*
* 如果生效，这个commit将 *删除废弃方法*
* 如果生效，这个commit将 *发布1.0版本*
* 如果生效，这个commit将 *从user分支合并pull request #123*
  
看看使用非祈使语气构成的例子：
* 如果生效，这个commit将 *wemeet version 2.0 problems*
* 如果生效，这个commit将 *update jsticket api for wemeet change*
* 如果生效，这个commit将 *sweet new API methods*
* 如果生效，这个commit将 *more fixes for broken stuff*

> 记住：只有commit message主题才需要祈使语句。
  commit message内容的时候没有这个限制。

# 7. 使用内容部分来解释What、Why
这个[commit from Bitcoin Core](https://github.com/bitcoin/bitcoin/commit/eb0b56b19017ab5c16c745e6da39c53126924ed6)是一个解释了What和Why的特别好的例子：
```
commit eb0b56b19017ab5c16c745e6da39c53126924ed6
Author: Pieter Wuille <pieter.wuille@gmail.com>
Date:   Fri Aug 1 22:57:55 2014 +0200

   Simplify serialize.h's exception handling

   Remove the 'state' and 'exceptmask' from serialize.h's stream
   implementations, as well as related methods.

   As exceptmask always included 'failbit', and setstate was always
   called with bits = failbit, all it did was immediately raise an
   exception. Get rid of those variables, and replace the setstate
   with direct exception throwing (which also removes some dead
   code).

   As a result, good() is never reached after a failure (there are
   only 2 calls, one of which is in tests), and can just be replaced
   by !eof().

   fail(), clear(n) and exceptions() are just never called. Delete
   them.
```

看一眼[full diff](https://github.com/bitcoin/bitcoin/commit/eb0b56b19017ab5c16c745e6da39c53126924ed6)和想一下作者为现在和将来参与的小伙伴理解当前上下文节约了多少时间。如果作者没有这样做的话，可能就永远丢失了。（想象回头看看自己一个月前写的代码，能理解作者的心情吧）

大多数情况，关于代码修改的细节是不用讲述的。通常代码是可以自解释的（然而如果代码十分复杂到需要解释的话，那就写到代码注释里吧）。只要专注于讲清楚修改的原因，修改前和修改后都是怎么工作的，为什么决定要这样的方式去解决。

将来的维护者会感谢你这么做（也许是你自己）！