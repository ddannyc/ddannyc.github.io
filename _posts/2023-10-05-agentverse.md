---
layout: post
title:  "多LLM环境模拟框架 - AgentVerse"
date:   2023-10-05 15:33:38 +0800
tag: agent llm ai
categories: agent
---
[**AgentVerse**](https://github.com/OpenBMB/AgentVerse)是由[OpenBNB](https://github.com/OpenBMB)开源的一个可以简化构建多Agent交互环境的框架，
通过它可以帮助我们在探索/开发多角色Agent交互相关的应用时，可以专注在研究的业务上，而不是被实现细节所困扰。


### AgentVerse工作流
![](https://user-images.githubusercontent.com/11704492/264917097-6db1c907-b7fc-42f9-946c-89853a28f386.png)

AgentVerse通过多轮迭代的方式去完成用户输入的目标，其中一轮的流程大概是这样：
* 根据配置好的AgentVerse环境里招募合适的专家，构成适合当前目标的专家组
* 专家组协作完成决策
* 专家根据决策执行任务
* 评估当前任务完成状态是否和目标一致

### 专家(Agent)协作决策的方式
目前AgentVerse内置了两种协作决策的方式，
* Horizontal Communication
"横向沟通"方式里，收集本轮专家组每个Agent的想法形成决策。
在需要创造性想法或需要大量协调（例如头脑风暴）的场景中，
咨询，或者合作游戏，横向沟通可能是更实际的选择。
* Vertical Communication
"纵向沟通"的方式里，有一个Agent作为"Solver"，Solver提供一个初始决定，其它的Agent逐个提供优化的建议，Solver根据反馈迭代优化初始决定。
纵向沟通方式适合需要通过迭代逐渐完善特定目标的任务，如软件开发。

### 软件开发场景示例
官方文档提供了多个应用示例，其中一个示例演示了AgentVerse如何通过三个角色开发者(code writer)、测试(code tester)、审查(code reviewer)互相协作解决一个软件设计的问题。
首先开发者根据问题编写代码，然后测试执行单元测试并给出测试结果，审查给出意见，最后开发者根据测试和审核给的反馈迭代优化程序。
[运行示例](https://1024code.com/codecubes/c0ampp7)


### 思考
* 多LLM协作产生的回答质量更高
* 通过迭代的方式可以让Agent完成相对复杂一些的任务

