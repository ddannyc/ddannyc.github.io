---
title: 用两个盏实现队列
date: 2022-07-05 14:58:07
tags:
---

## 题目
https://leetcode.cn/problems/yong-liang-ge-zhan-shi-xian-dui-lie-lcof

## 思路
定义盏A以队列结构要求的顺序存放数字，定义盏B以正常盏结构存放数字。appendTail操作可直接对盏B做入盏操作，deleteHead操作则直接对盏A做出盏操作。如果盏A为空，则把所有盏B元素依次取出再入盏至盏A。举例：  
```
// appendTail 3，7，5

A: 
B: 3, 7, 5

// deleteHead
// 首先将所有盏B元素出盏并入盏至盏A

A: 5, 7, 3
B:

// 从盏A中移除盏顶元素
A: 5, 7
B:

// 返回盏顶元素
3

```

## 实现
```typescript
class CQueue {
    head: number[]
    tail: number[]
    constructor() {
        this.head = []
        this.tail = []
    }

    appendTail(value: number): void {
        this.tail.push(value)
    }

    deleteHead(): number {
        if (this.head.length) return this.head.pop()
        if (!this.tail.length) return -1
        while(this.tail.length) this.head.push(this.tail.pop())
        return this.head.pop()
    }
}

/**
 * Your CQueue object will be instantiated and called as such:
 * var obj = new CQueue()
 * obj.appendTail(value)
 * var param_2 = obj.deleteHead()
 */
```

## 链表实现一个队列
```typescript
class MyNode {
    val: number
    next?: MyNode
    constructor(val: number) {
        this.val = val
    }
}
class CQueue {
    head: MyNode
    tail: MyNode
    constructor() {
    }

    appendTail(value: number): void {
        if (this.tail) {
            this.tail.next = new MyNode(value)
            this.tail = this.tail.next
        } else {
            this.head = new MyNode(value)
            if (!this.tail) this.tail = this.head
        }
    }

    deleteHead(): number {
        if (!this.head) {
            return -1
        }
        // 当头尾节点为相同节点时需注意，清除尾节点指针
        if (this.head == this.tail) {
            this.tail = null
        }
        const val = this.head.val
        this.head = this.head.next
        return val
    }
}
```