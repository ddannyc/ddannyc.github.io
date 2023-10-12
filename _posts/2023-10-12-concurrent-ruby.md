---
title: "Concurrent Ruby"
date: "2023-10-12 16:33:46"
tags: ruby
---

> Modern concurrency tools for Ruby. Inspired by Erlang, Clojure, Scala, Haskell, F#, C#, Java, and classic concurrency patterns.

根据官网里的一句介绍，可以了解到Concurrent Ruby是一个吸收了各种语言和经典并发模式思想的一个现代的Ruby并发工具（框架）。



### 异步任务
最基本的异步执行操作，通过是`Promises#future`类的工厂方法构造，该方法返回一个`Future`对象。
```ruby
Concurrent::Promises.future { do_stuff }
# => #<Concurrent::Promises::Future:0x000029 pending>
```


### 状态
Future可以分为`pending`和`resolved`两种事件状态， 当Future事件状态为resolved， Future的状态会演进为`fulfilled`和`rejected`。
* pending   待执行
* resolved  已执行
* fulfilled 已解决
* rejected  异常


### 参数传递
错误示例:
```ruby
arg = 1                                  # => 1
Thread.new { do_stuff arg }
# => #<Thread:0x00000c@promises.in.md:204 run>
Concurrent::Promises.future { do_stuff arg }
# => #<Concurrent::Promises::Future:0x00000d pending>
```
使用了非线程安全局部变量

正确示例:
```ruby
arg = 1                                  # => 1
Thread.new(arg) { |arg| do_stuff arg }
# => #<Thread:0x00000e@promises.in.md:212 run>
Concurrent::Promises.future(arg) { |arg| do_stuff arg }
# => #<Concurrent::Promises::Future:0x00000f pending>
```


### 异常处理
```ruby
Concurrent::Promises.
    fulfilled_future(Object.new).
    then(&:succ).
    then(&:succ).
    rescue { |err| 0 }.
    result                               # => [true, 0, nil]
```
可以通过rescue方法捕获异步任务抛出的异常。

### promises是如何执行的？
`Promises`使用全局线程池执行任务。因此，每个任务可能执行在不同的线程上，那就意味着用户需要注意不要依赖线程局部变量。
由于任务可能运行在线程池里不同的线程上，所以最好遵循一下规则：
* 仅使用参数传递或者父级`future`传递的数据, 从而更好得控制`future`可以访问的数据。
* 使用immutable(不可变)数据作为`future`得输入和输出可以让`future`更容易得处理它们，或者至少假设它们是不可变得。
* 任何mutable和mutated对象需要被超过一个线程或者`future`访问必须保证对象是线程安全的，可以参见框架实现的[Concurrent::Array](https://ruby-concurrency.github.io/concurrent-ruby/master/Concurrent/Array.html),[Concurrent::Hash](https://ruby-concurrency.github.io/concurrent-ruby/master/Concurrent/Hash.html)和[Concurrent::Map](https://ruby-concurrency.github.io/concurrent-ruby/master/Concurrent/Map.html)
* Futures可以访问外部的对象，但是它们必须是线程安全的。

### Using executors
工厂方法(future => future_on), chain, callback(then => then_on)这些方法都有另外一个可以指定执行器(executor)参数的版本。
* :fast 对应非阻塞任务
* :io   对应需要长时间执行/阻塞执行的任务

```ruby
Concurrent::Promises.future_on(:fast) { 2 }.
    then_on(:io) { File.read __FILE__ }.
    value.size                           # => 25384
```

### Run
Future有一个`run`方法，它可以用来模拟类似线程一样的处理过程而不需要占用实际的线程。
当调用Future的`run`方法时，会类似递归函数那样一层一层不限得执行下去直到达到结束递归的条件, 而`run`结束的条件是直到future fulfils为止。
```ruby
count = lambda do |v|
  v += 1
  v < 5 ? Concurrent::Promises.future_on(:fast, v, &count) : v
end
# => #<Proc:0x000018@promises.in.md:521 (lambda)>
400.times.
    map { Concurrent::Promises.future_on(:fast, 0, &count).run.value! }.
    all? { |v| v == 5 }                  # => true
```