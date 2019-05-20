---
layout: post
title:  "NSURLProtocol相关"
date:   2019-02-02
tag: iOS
permalink: /page/NSURLProtocol.html
type: legacy

---


### NSURLProtocol
1.重定向网络请求
2.忽略网络请求，使用本地缓存
3.自定义网络请求的返回结果
4.一些全局的网络请求设置


[NSURLProtocol详解](http://www.oudahe.com/p/41781/)

[webview预加载](https://github.com/ShannonChenCHN/iOSDevLevelingUp/
issues/55)

[NSURLProtocol拦截wkwebview](https://blog.moecoder.com/2016/10/26/support-nsurlprotocol-in-wkwebview/)
  
### 一些关键字段
 
 
 请求头信息 Request cache headers
 
If-Modified-Since:与响应头Last-Modified相对应，其值为最后一次响应头中的Last-Modified。

If-None-Match:与响应头Etag相对应，其值为最后一次响应头中的Etag。

响应头信息 Response cache headers

Last-Modified:资源最近修改时间

Etag:（Entity tag缩写）是请求资源的标识符，主要用于动态生成、没有Last-Modified值的资源。

Cache-Control:缓存控制，只有包含此设置可能使用默认缓存策略。可能包含如下选项：

max-age：缓存时间（单位：秒）。 

public：可以被任何区缓存，包括中间经过的代理服务器也可以缓存。通常不会被使用，因为 

max-age本身已经表示此响应可以缓存。 private:只能被当前客户端缓存，中间代理无法进行缓存。 

no-cache:必须与服务器端确认响应是否发生了变化，如果没有变化则可以使用缓存，否则使用新请求的响应。 

no-store:禁止使用缓存

Vary:决定请求是否可以使用缓存，通常作为缓存key值是否唯的确定因素，同一个资源不同的Vary设置会被作为两个缓存资源（注意：NSURLCache会忽略Vary请求缓存）。

默认缓存策略下当客户端发起一个请求时首先会检查本地是否包含缓存，如果有缓存则检查缓存是否过期（通过Cache-Control:max-age或者Expires判断），如果没有过期则直接使用缓存数据。如果缓存过期了，则发起一个请求给服务器端，此时服务器端对比资源Last-Modified或者Etags(二者都存在的情况下下如果有一个不同则认为缓存已过期)，如果不同则返回新数据，否则返回304 Not Modified继续使用缓存数据（客户端可以继续使用"max-age"秒缓存数据）。这个过程中客户端发送不发送请求主要看max-age是否过期，而过期后是否继续使用缓存则需要重新发起请求，服务器端根据情况通知客户端是否可以继续使用缓存（返回结果可能是200或者304）

缺点：只有在缓存过期的情况下才会去直接请求

1. 跟服务器配合缓存，通过cache-control去判断是否去服务器拉去新的信息（上面那种），
2. 完全本地缓存策略，如果设置了缓存策略，则下一次就不会进行网络请求，只要清空缓存或超过缓存限制



[https://juejin.im/post/5a69f8366fb9a01cb42c90bc](https://juejin.im/post/5a69f8366fb9a01cb42c90bc)
[NSURLCache缓存](https://juejin.im/entry/59e02c6e6fb9a0451b038e3d)


### 思考
  
  网络数据如果是某条数据的为null，发生崩溃，怎么办（该不该存入缓存，存入缓存后又该如何处理呢
  ，数据库插入空会怎么样）
  数据库优化 