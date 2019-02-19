---
title: Twemproxy核心原理介绍
date: 2019-02-19 09:46:42
tags:
- redis
categories: 
- redis 
---

本文简要的介绍了Tweproxy的内部实现的几个核心点， 其中包括：tweproxy的消息传递和Zero Copy，不过没有涉及tweproxy的另一个核心内容：基于一致性哈希实现的路由表；

<!--more-->

#### 消息传递

tewmproxy又称nutcracker(下面简称nc);

nc用内部的一个`conn`结构体来表示`client->proxy`和 `proxy->server(redis or memcached)`之间的连接,同时,每个`conn`中维护了两个队列(fifo), 一个是conn的input queue(`conn->imsg_q`)和output queue(`conn->omsg_q`)来组织和缓存当前conn的request和response, 一个简单的请求的路径如下图所示:

1. `client conn`将获取client的reqest, proxy将根据当前request请求的key的来选择存储该key的后端的server(conn);
   . client的request经过解析后将进	入proxy的 `server conn`的input fifo中. 同时也将在`client conn`的output fifo中追踪forward到`server conn`的request;
2. proxy从`server conn`的input queue中取出请求发送到对应的server上, 同时将在`server conn`的output fifo中追踪forward到后端`(redis)server`的请求, 用来等到获取server response后和具体的request 对应;
3. `client conn`的output fifo中记录了forward到`server conn`的请求, 同时`server conn`的output fifo中记录了forward到后端`(redis)server`的请求,  通过这些proxy就能够将`redis server`的response和client的request对应;

​     以上的分析中针对了一个client和一个server, 但是实际的情况是一个proxy会有多个client(conn)和多个server(conn), 每个client都可能访问任意一个server. nc为了提高效率, 将不同client对同一个的server的请求都push到该proxy和该server的`server conn` 的input queue中, 同时client的output queue中也记录了forward到`server conn`中的request; 

​	然后proxy以pipeline的方式向server发送请求, 这样便充分利用了proxy和server之间连接的带宽, 提高了proxy访问server的效率, 同时将server的request push到四`server conn`的output queue中,  当redis server 返回后, 由于`server conn`的output queue中和`client conn`的output queue都记录了所有的request, proxy 能够识别出redis server的response和client 的request之间的对应关系, 然后就能将不同的response 返回给与之的对应的client:

​	下图给出了nc源码中的消息传递的示意图:

![sss](https://i.imgur.com/RiURJv4.png)

图中有两个client和两个server. 其中client1的`req11`和`req12`请求server1, client2的`req21`, `req22`请求server1,  client2的`req23`请求server2.  对server1的请求顺序是 `req11` -> `req21` -> `req12` -> `req22` , 其中前三个请求server1 已经处理faward到redis server, `req22`还缓冲在`server1 conn`的 input queue中等待发往server1, 同样, `req23` 也还缓冲在`server2 conn`的input queue中, 等待forward到server2.  同时可以看到: `client conn`所有`forward server conn`的request都会在`cilent conn`的`out_q`中记录;

> client 的 in_q好像是没有使用的



#### 消息传递的触发机制:

nc的request消息传递的路径是: 

​				client -> client conn -> server conn -> redis serer

nc的response消息的传递路径刚好是相反的;

nc通过epoll来高效的控制消息的传递, client conn和server conn均为 nc的connection类型, 都有一个文件描述符, 在每次nc的`core_loop`中都会通过epoll机制来遍历所有ready的fd,然后进行处理, nc message的传递就是在这些event的处理的过程中进行的;



#### zero copy:

nc性能优异的另一个原因是由于nc内部通过zero copy 避免了message的复制. nc内部通过mbuf存储message, 同时mbuf一旦创建出来是不进行回收的, nc会通过一个queue(`free_mbufq`)来存储所有的可用的mbuf,在需要申请mbuf的时候会先从`free_mbufq`中获取(如果没有可用的,则会先创建mbuf, 等mbuf使用完之后在放入到`free_mbuf`中), 这样就避免了mbuf的频繁的申请和释放,提到了效率;