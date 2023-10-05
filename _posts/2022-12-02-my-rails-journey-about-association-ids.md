---
title: Rails踩坑记录 - association_ids
date: 2022-12-02 17:09:15
tags: ruby rails
categories: rails
---

# Rails踩坑记录 - association_ids

## 场景
假设有这些数据实体：
```ruby
class Book < ApplicationRecord
  has_many :book_tags
  has_many :tags, through: :book_tags
end

class Tag < ApplicationRecord
  has_many :book_tags
  has_many :books, through: :book_tags
end

class BookTag < ApplicationRecord
  belongs_to :book
  belongs_to :tag
end
```
有这样的需求，用户可以为书本随意添加、删除多个标签。
最直接的做法可以分为两个接口提供给用户，一个负责添加、一个负责删除。
```ruby
# add
book.tags << tag1, tag2, tag3

# remove
book.tags.destroy(tag1, tag2, tag3)
```
这种方式后端操作比较简单， 麻烦的部分是前端需要知道用户这次操作是增加了标签，还是删除了标签，还是都有。
还有一种不论是前后端的都相对简单的操作，[collection_singular_ids=](https://guides.rubyonrails.org/association_basics.html#methods-added-by-has-many-collection-singular-ids)
```ruby
book.tag_ids = params[:tag_ids]
```
前端不管用户如何操作， 只需要将最终标签id数组告诉后端。

## 坑
虽然将设置标签的操作变简单了，但是使用了counter_cache这个属性的话就需要注意了，如:
```ruby
class BookTag < ApplicationRecord
  belongs_to :book, counter_cache: :tags_count
  belongs_to :tag
end

# 增加标签
book.tag_ids = [1,2,3]
book.tags_count # 3

# 移除标签
book.tag_ids = [2,3]
book.tags_count # 3
```
当我们使用tag_ids移除标签的时候，tags_count计数并没有减少。即使给`has_many :book_tags`加上`dependent: :destroy`也是如此。
通过查看官方api文档，发现`collection_singular_ids=ids`实际是将ids的所有对象加载出，然后调用`collection=`方法去更新关联的集合。

> `collection=objects`
> Replaces the collections content by deleting and adding objects as appropriate. If the :through option is true callbacks in the join models are triggered except destroy callbacks, since deletion is direct by default. You can specify dependent: :destroy or dependent: :nullify to override this.

意思大概就是因为`tag_ids=`通过`through`选项关联，destroy回调不会触法，因为删除操作默认直接执行。可以通过指定`dependent: :destroy` 或者 `dependent: :nullify`覆盖这种行为。
但是即使加上了`dependent: :destroy`之后`counter_cache`仍然没有因为删除关联而减少。最后通过另外的手段去手动执行`increment`操作。相关代码如下：
```ruby
  def counter_change(associaton, &block)
    counter = "#{associaton}_count"
    old_count = public_send(counter)
    yield block
    decrease_count = public_send(associaton).size - old_count
    increment!(counter, decrease_count) if decrease_count < 0
  end

book.counter_change(:tags) do
  book.tag_ids = params[:tag_ids]
end
```

## 后记
`increment` 和 `increment!`的区别是什么？
`increment`操作只对修改对象，不执行数据库更新操作。

慎用`collection_singular_ids=`， 虽然方便，但也不是所有场景都是合适的，比如它一次加载出所有的ids对象，如果ids数组长度很大，可能会把内存吃完。