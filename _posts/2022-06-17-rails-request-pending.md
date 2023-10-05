---
title: 1024-Rails服务请求超时问题
date: 2022-06-17 17:00:28
tags:
---

# 问题
staging环境rails服务突然所有请求pending
# 分析
* portainer查看服务状态(running)，正常运行
* 查看服务log，没有输出新的请求log
* 查看traefik日志
```
[16/Jun/2022:10:27:24 +0000] "GET /api/v1/codecubes/recommends HTTP/2.0" 499 21 "-" "-" 82351 "1024-rails-staging@docker" "http://10.0.2.66:3000" 532555ms
```
请求返回状态码499，处理时间532555ms，通过这两个异常信息可见是rails容器服务问题
* 进入容器bash，查看进程 ps -ef
  
<img width="853" alt="wecom-temp-addb00410776ed055ed142bce84a9190" src="https://user-images.githubusercontent.com/5228544/174217643-dcf385d4-05bd-487d-88bc-43f98b76af28.png">
  
没有发现puma worker进程，怀疑puma异常。  
  
![wecom-temp-2fcb9b820a57e5b0d26731f156e0b45b](https://user-images.githubusercontent.com/5228544/174219030-2e7b72a2-1f22-428e-85d9-53d56280fa82.png)
看到**worker_timeout**设置，以为是worker闲置超时自动终止，后发现此配置仅在develop环境生效。遂在本地验证puma worker配置，发现puma运行在单进程模式，所以没有worker进程是正常的。
  
* 尝试查看容器进程netstat信息
```bash
docker inspect -f '{{.State.Pid}}' 容器id/名称
sudo nsenter -t `进程id` -n netstat
```
![wecom-temp-e2a26de8511e3d336c98dd5a32125042](https://user-images.githubusercontent.com/5228544/174219578-6f3e53cd-94a2-4f53-9af5-f5909891e277.png)
大量的3000端口CLOSE_WAIT状态，加上前两次出现这种情况也是由于Paas接口挂了之后，可以确定由于Paas服务终止导致puma使用的3000端口被大量的CLOSE_WAIT连接占用。  
```bash
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp     2048      0 10.0.13.32:56204        10.0.13.23:6379         ESTABLISHED 113376/puma 5.6.4
```
这个netstat示例意思是，进程113376有一个打开(ESTABLISHED)的连接，本地Kernal在端口56204接收到来自外部(10.0.13.23:6379)的2048字节数据还没被进程收到.正常情况下Recv-Q和Send-Q都应该为0。
  
为何Paas Api终止会导致puma使用的3000端口出现大量的CLOSE_WAIT，可以通过TCP连接关闭的一张图来分析。
![image](https://user-images.githubusercontent.com/5228544/174241352-1e6c4b88-8935-43f4-9617-4b604c458895.png)
Paas服务终止前主动(Client)向正在连接Rails(Server)端发送FIN信号，Rails端接收到FIN信号之后连接进入CLOSE_WAIT状态并发送ACK给Paas服务，这步应该没有执行成功所以连接一直处于CLOSE_WAIT。

# 处理
默认情况下HTTP GEM不强制请求超时，可通过timeout配置支持请求超时，
```ruby
# ---
HTTP.request 
# +++
HTTP.timeout(connect: 5, write: 2, read: 10).request
```
# 参考
[list open sockkets inside container](https://stackoverflow.com/questions/40350456/docker-any-way-to-list-open-sockets-inside-a-running-docker-container)
[浅谈CLOSE_WAIT](https://blog.huoding.com/2016/01/19/488)
[httprb TIMEOUT](https://github.com/httprb/http/wiki/Timeouts)