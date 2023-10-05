---
title: traefik-actioncable-fail
date: 2022-07-15 15:24:35
tags:
---

## 问题
通过traefik配置路由转发到Rails内置ActionCable时，websocket建立不成功

## 排查
查看traefik日志
```
159.75.250.8 - - [15/Jul/2022:07:32:51 +0000] "GET /cable HTTP/1.1" 404 133 "-" "-" 326145 "1024-rails-staging@docker" "http://10.0.2.96:3000" 41ms
```
请求正确转发，但是应用没有输出相关请求的日志，经过调查，最终发现为`action_cable.allowed_request_origins`设置问题， 而应用没有输出请求日志应该是ActionCable需要单独配置日志输出。