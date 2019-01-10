---
title: Redis Blocking 机制
date: 2019-01-10 22:19:31
tags:
- redis
categories: 
- redis
---

​	Redis针对List操作提供了blocking command， 其中主要包括`BRPOP`，` BLPOP`，`BRPOPLPUSH`三个命令，这些命令针对List来完成阻塞式的pop操作，阻塞的逻辑是如果db中pop的key存在，则能够顺利完成pop操作，如果pop的key不存在，可client将会阻塞，直到有其他客户端通过`*PUSH`命令想db中push指定的key（不讨论阻塞超时的问题），下面我们主要来了解一下redis是如何实现Blocking POP操作的，redis版本为`4.0.12`;

<!--more-->

## 阻塞Client

​	Redis `redisServer`结构体中记录了有两个list，

​	`BRPOP`和`BLPOP`命令主要是有函数`blockingPopGenericCommand`来实现的，具体的函数如下：

```
/* Blocking RPOP/LPOP */
void blockingPopGenericCommand(client *c, int where) {
    robj *o;
    mstime_t timeout;
    int j;

    if (getTimeoutFromObjectOrReply(c,c->argv[c->argc-1],&timeout,UNIT_SECONDS)
        != C_OK) return;

    for (j = 1; j < c->argc-1; j++) {
        o = lookupKeyWrite(c->db,c->argv[j]);
        if (o != NULL) {
            if (o->type != OBJ_LIST) {
                addReply(c,shared.wrongtypeerr);
                return;
            } else {
                if (listTypeLength(o) != 0) {
                    /* Non empty list, this is like a non normal [LR]POP. */
                    char *event = (where == LIST_HEAD) ? "lpop" : "rpop";
                    robj *value = listTypePop(o,where);
                    serverAssert(value != NULL);

                    addReplyMultiBulkLen(c,2);
                    addReplyBulk(c,c->argv[j]);
                    addReplyBulk(c,value);
                    decrRefCount(value);
                    notifyKeyspaceEvent(NOTIFY_LIST,event,
                                        c->argv[j],c->db->id);
                    if (listTypeLength(o) == 0) {
                        dbDelete(c->db,c->argv[j]);
                        notifyKeyspaceEvent(NOTIFY_GENERIC,"del",
                                            c->argv[j],c->db->id);
                    }
                    signalModifiedKey(c->db,c->argv[j]);
                    server.dirty++;

                    /* Replicate it as an [LR]POP instead of B[LR]POP. */
                    rewriteClientCommandVector(c,2,
                        (where == LIST_HEAD) ? shared.lpop : shared.rpop,
                        c->argv[j]);
                    return;
                }
            }
        }
    }

    /* If we are inside a MULTI/EXEC and the list is empty the only thing
     * we can do is treating it as a timeout (even with timeout 0). */
    if (c->flags & CLIENT_MULTI) {
        addReply(c,shared.nullmultibulk);
        return;
    }

    /* If the list is empty or the key does not exists we must block */
    blockForKeys(c, c->argv + 1, c->argc - 2, timeout, NULL);
}
```

1. 首先，server会在dict中查看目标key是否存在，如果key存在则该次pop操作不会被阻塞，由于pop操作只支持list类型，所有需要对key的value类型进行检查，如果不是list类型，则会向client报错，如果是list类型，则进行后续的处理;
2. 在key存在的情况下，server的对`B*POP`的操作和非阻塞的POP操作是完全相同的，server的做法也只是将client原始的请求命令中的`B*POP`命令替换成`非阻塞POP`命令，同时保持命令的参数不变，重请求命令通过函数`rewriteClientCommandVector` 重写到client的请求中;

以上是key在存在的情况下server的处理，下面我们主要看下在key不存在或list为空的情况下，server是如何处理的;

​	在server中client请求的key不存在时后者key对应的list为空，server将阻塞请求的client，核心的处理思想是本次命令处理不会回复client，同时将该client的`flags`标志置位`BLOCKED_LIST`,这样后续的main loop循环中也将之际跳过（阻塞）对该client的请求命令的处理;

1. 在上面`blockingPopGenericCommand`函数中，如果key对应的list不存在或为空，将会调用`blockForKeys`函数：

   ```
   /* 为每一个blocking key在client->bpop.keys dict,记录了该client等待的所有key;
    * 同时,client->db.blocking_keys(dict)记录了所有该db中等待该key的client(list);
    */
   void blockForKeys(client *c, robj **keys, int numkeys, mstime_t timeout, robj *target) {
       dictEntry *de;
       list *l;
       int j;
   
       c->bpop.timeout = timeout;
       c->bpop.target = target;
   
       if (target != NULL) incrRefCount(target);
   
       for (j = 0; j < numkeys; j++) {
           /* If the key already exists in the dict ignore it. */
           if (dictAdd(c->bpop.keys,keys[j],NULL) != DICT_OK) continue;
           incrRefCount(keys[j]);
   
           /* And in the other "side", to map keys -> clients */
           de = dictFind(c->db->blocking_keys,keys[j]);
           if (de == NULL) {
               int retval;
   
               /* For every key we take a list of clients blocked for it */
               /* 为每一个blocking_key都创建一个list来记录有哪些clients在等待该key */
               l = listCreate();
               retval = dictAdd(c->db->blocking_keys,keys[j],l);
               incrRefCount(keys[j]);
               serverAssertWithInfo(c,keys[j],retval == DICT_OK);
           } else {
               l = dictGetVal(de);
           }
           //将当前client加入到该key的blocking client list的尾部,这样在指定key到来时,也是先阻塞的client先被serve
           listAddNodeTail(l,c);
       }
       blockClient(c,BLOCKED_LIST);
   }
   ```

   server端的client结构体中维护了一个`blockingState`类型的struct`client->bpop.keys`，该结构其记录了该client所阻塞等待的所有的list，在将要阻塞该client时，需要在该结构中增加client阻塞等待的list;

   ```
   typedef struct blockingState {
   	...
       /* BLOCKED_LIST */
       dict *keys;             /* The keys we are waiting to terminate a blocking
                                * operation such as BLPOP. Otherwise NULL. */
       robj *target;           /* The key that should receive the element,
   	...
   } blockingState;
   ```

   同时在server中表征db的结构体`redisDb`中也记录了该db中所有存在client在阻塞等待的key，在dict类型的`redsiDb->blocking_keys`中记录，其字典的key是client请求的key，value是一个list，该list中记录了该key阻塞的所有client，最新被阻塞的client将被加入到list的尾部。 这样做的目的是在后续某个时刻有其它client对该key进行了push操作，那么server将会知道有哪些client在阻塞等待该key，并选出list的头元素，也就是最先被阻塞的client对其unblock;

   ```
   typedef struct redisDb {	
   	...
       dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
       dict *ready_keys;           /* Blocked keys that received a PUSH */
    	...
   } redisDb;
   ```

   在函数`blockForKeys`最后将通过`blockClient`函数对client的`flag`置位`CLIENT_BLOCKED`，表示当前client被阻塞，这样，后续的事件循环中都将不会处理该命令的请求缓冲区;

至此，server就完成了对该client的阻塞操作;

## 解除Client的阻塞

前面提到过，如果一个client在调用`B*POP`名利时被阻塞，此时需要有其他客户端对相应的list进行`＊PUSH`操作才能解除阻塞;



未完待续...