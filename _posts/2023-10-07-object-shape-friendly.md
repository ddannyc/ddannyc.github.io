---
title: "Ruby Object Shape friendly code"
date: "2023-10-07 17:33:46"
tags: ruby
---

Ruby 3.2版本引入了叫做[`Object Shapes`](https://poddarayush.com/posts/object-shapes-improve-ruby-code-performance/)的性能优化，它的作用是优化Ruby存储、查找和缓存实例变量的实现，并且YJIT以及将来的3.3版本都可以利用到Object Shapes提升性能。


### 优化原则
```ruby
# Bad: Object Shape unfriendly
class Foo
  def bar 
    @bar = 0
  end

  def baz
    @baz = 1
  end
end

f1 = Foo.new # Shape ID: 0
f1.bar       # Shape ID: 1
f1.baz       # Shape ID: 2

f2 = Foo.new # Shape ID: 0
f2.baz       # Shape ID: 1
f2.bar       # Shape ID: 2
```
假设有这样一个`Foo`类，由于f1与f2实例调用方法的顺序不同，导致它们被认为是"形状"不相同的两个对象，所以是Object Shapes不友好的实践。

```ruby
# Good: Object Shape friendly
class Foo
  def initialize
    @bar = nil
    @baz = nil
  end

  def bar 
    @bar = 0
  end

  def baz
    @baz = 1
  end
end
```
通过类构造方法将实例变量定义顺序固定住， 不管实例变量调用顺序如何，都可以获得Object Shape带来的性能优化。

### Object Shapes
Object Shape顾名思义，就是对象的形状。如果两个对象的形状一样，那么这两个对象就可以共享这个Object Shape， 共享Object Shape的好处之一就是当我们查找实例变量（iv）的时候，如果Object Shape已经形成，就可以直接利用缓存得到对应iv的索引位置，通过内连缓存的方式直接得到iv的地址。

Object Shape会根据iv的创建被构造成一个树的结构， Shape ID 0则就是树的根节点。