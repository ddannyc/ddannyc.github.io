---
title: "Nginx缓存笔记"
date: "2023-12-06 11:01:01"
tags: 
---

### 缓存
介于浏览器和应用服务器之间的缓存：
* 客户端的浏览器缓存
* 中间缓存
* CDN内容分发网络
* 负载均衡/反向代理（Nginx Page Cache属于这一层）

### 如何开启缓存
通过[proxy_cache_path](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_path)和[proxy_cache](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache)两个指令就可以开启Nginx缓存。

```
proxy_cache_path /path/to/cache levels=1:2 keys_zone=my_cache:10m max_size=10g 
                 inactive=60m use_temp_path=off;

server {
    # ...
    location / {
        proxy_cache my_cache;
        proxy_pass http://my_upstream;
    }
}
```
* 缓存路径
* levels设置缓存存储目录层级
* keys_zone设置一个共享键区，不需要每次读区硬盘就可以知道当前请求缓存是否命中。
* max_size设置缓存在硬盘上最大存储容量， 如果达到存储上限，Nginx会移除least recently used(最近最少使用)的缓存。
* inactive=60m， 告诉Nginx当一个缓存单位(item)60分钟都没有被访问过就可以从缓存里删除；
  Nginx不会自动删除cache control认为过期的缓存，当过期的内容被访问时，Nginx从源服务器或许新的内容并重置缓存的**inactive**时间
* use_temp_path=off, 缓存临时存放路径，推荐设置为关闭，可以避免不必要的文件复制开销。


### 缓存调优
```
proxy_cache_path /path/to/cache levels=1:2 keys_zone=my_cache:10m max_size=10g 
                 inactive=60m use_temp_path=off;

server {
    # ...
    location / {
        proxy_cache my_cache;
        proxy_cache_revalidate on;
        proxy_cache_min_uses 3;
        proxy_cache_use_stale error timeout updating http_500 http_502
                              http_503 http_504;
        proxy_cache_background_update on;
        proxy_cache_lock on;

        proxy_pass http://my_upstream;
    }
}
```
* [proxy_cache_revalidate](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_revalidate)告诉Nginx向源服务器获取新内容的时候使用`condition GET`请求，Nginx向源服务器请求时包含**If-Modified-Since**头，
  服务器检查内容是否变更，如果内容没有变更则服务器不需要重新发送完整的内容以节省带宽。
* [proxy_cache_min_uses](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_min_uses)设置最少被客户请求x次才会生成缓存，默认值是1。
* [proxy_cache_use_stale](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_use_stale)当源服务器宕机的时候，Nginx是否可以使用过期的缓存作为响应。
* [proxy_cache_background_update](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_background_update)与**proxy_cache_use_stale**指令组合使用的时候，若请求过期Nginx直接返回过期的内容，并在后台向源服务器请求新的内容。
* [proxy_cache_lock](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_lock)如果开启，当多个客户端同时请求一个缓存未命中的文件时，只有第一个请求会到达源服务器，其他的请求等待第一个请求的结果生成缓存后，返回缓存的结果。

### Nginx怎样决定生成或不生成缓存？
当源服务器请求结果中包含**Expires**或者**Cache-Control: max-age**这样的头时，Nginx生成缓存。  
当请求结果包含**Cache-Control**头和**Private**, **No-Cache**, **No-Store**这些值时，Nginx不生成缓存。  
当请求结果包含**Set-Cookie**头时Nginx也不生成缓存， 默认情况下Nginx仅生成**GET**和**HEAD**请求的缓存。  

### 参考
* [Nginx Caching Guide](https://www.nginx.com/blog/nginx-caching-guide/)