---
title: Redis Blocking Pop 机制
date: 2019-01-10 22:19:31
tags:
- redis
categories: 
- redis
---

​	Redis针对List操作提供了blocking command， 其中主要包括`BRPOP`，` BLPOP`，`BRPOPLPUSH`三个命令，这些命令针对List来完成阻塞式的pop操作，阻塞的逻辑是如果db中pop的key存在，则能够顺利完成pop操作，如果pop的key不存在, client将会阻塞，直到有其他客户端通过`*PUSH`命令想db中push指定的key（不讨论阻塞超时的问题），下面我们主要来了解一下redis是如何实现Blocking POP操作的，redis版本为`4.0.12`;

<!--more-->

## Block Client

​	Redis `redisServer`结构体中记录了有两个list，

​        `BRPOP` 和`BLPOP`命令均由调用函数`blockingPopGenericCommand`来实现，具体的函数如下：

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
       ...
       
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

## Unblock Client

前面提到过，如果一个client在调用`B*POP`命令时被阻塞，此时需要有其他客户端对相应的list进行`PUSH`操作使的list不为空才能解除阻塞;

​	在开始之前先说说一下unblock client操作涉及到的几个重要结构: `redisDb->blocking_keys`, ``redisDb->ready_keys`和`redisServer->ready_keys`

```
typedef struct redisDb {
    ...
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    ...
} redisDb;
```

`redisDb->blocking_keys` 是一个dict类型, 该dict中记录了当前db中所有client在阻塞等待的key; `redisDb->ready_keys` 也是一个dict类型, 该dict中记录了当前db中所有client在阻塞等待的key中有那些key是已经ready了的, 最阻塞的key进行push操作将会使该key变为ready状态;

```  
struct redisServer {
	...
	list *unblocked_clients; /* list of clients to unblock before next loop */
    list *ready_keys;        /* List of readyList structures for BLPOP & co */
    ...
} redisServer;
```

`redisServer->ready_keys`中记录了所有的已经ready的key, 值为`readyList`结构:

```
typedef struct readyList {
    redisDb *db;
    robj *key;
} readyList
```

通过该结构就可以同时知道该key所属的db;

​	unblock client的过程也就是将push操作请求的key的加入到`redisServer->ready_keys` 中, 最终server每次处理完一条命令请求后都会去根据`redisServer->ready_keys` 中的内容来unblock相应的client;

​	redis代码中`popGenericCommand`函数是redis `RPUSH和`和`LPUSH`的具体实现，

```
void pushGenericCommand(client *c, int where) {
    int j, pushed = 0;
    robj *lobj = lookupKeyWrite(c->db,c->argv[1]);
	...
    for (j = 2; j < c->argc; j++) {
        if (!lobj) {
            lobj = createQuicklistObject();
            quicklistSetOptions(lobj->ptr, server.list_max_ziplist_size,
                                server.list_compress_depth);
            /* 将新创建的key加入到db中,由于是push操作,在该函数中同时也会调用
             * signalListAsRead()函数来将新创建的key的加入到server->ready_keys
             * 中,进而将在server每次处理一条命令后调用handleClientsBlockedOnLists
             * 函数来unblock被BRPOP和BLPOP阻塞的client
             */
            dbAdd(c->db,c->argv[1],lobj);
        }
        listTypePush(lobj,c->argv[j],where);
        pushed++;
    }
    ...
}
```

如果在push时请求的key不存在,server将会创建该key对应的list,并表用`dbAdd`将该kv将入到对应的db中, 并同时调用`signalListAsReady`函数来生成该key对应的`readyList`结构, 并插入到`redisServer->ready_keys` dict中; 在加入到`redisServer->ready_keys`之前需要先检查`redisDb->blocking_keys`中是否存在该key,只有在存在的情况下才表明有client在阻塞等待该key, 此时才将key加入到`redisServer->ready_keys`中, 同时如果`db->ready_keys`中已经存在该key, 表名该key已经加入到了`redisServer->ready_keys`中, 不会重复加入

```
void dbAdd(redisDb *db, robj *key, robj *val) {
    sds copy = sdsdup(key->ptr);
    int retval = dictAdd(db->dict, copy, val);
    ...
    if (val->type == OBJ_LIST) signalListAsReady(db, key);
    ...
}
```

```
void signalListAsReady(redisDb *db, robj *key) {
    readyList *rl;

    /* No clients blocking for this key? No need to queue it. */
    if (dictFind(db->blocking_keys,key) == NULL) return;

    /* Key was already signaled? No need to queue it again. */
    if (dictFind(db->ready_keys,key) != NULL) return;

    /* Ok, we need to queue this key into server.ready_keys. */
    rl = zmalloc(sizeof(*rl));
    rl->key = key;
    rl->db = db;
    incrRefCount(key);
    //将新push的key加入到server.ready_keys,等待handleClientsBlockedOnLists处理
    listAddNodeTail(server.ready_keys,rl);
	...
}
```

在将ready的key加入到`server->ready_key`之后，后续就是在server每次处理命令请求后都会去检查`server->ready_key`, 如果不为空, 则为调用`handleClientsBlockedOnLists`函数来处理被阻塞的client;

```
void handleClientsBlockedOnLists(void) {
    while(listLength(server.ready_keys) != 0) {
        list *l;

        /* Point server.ready_keys to a fresh list and save the current one
         * locally. This way as we run the old list we are free to call
         * signalListAsReady() that may push new elements in server.ready_keys
         * when handling clients blocked into BRPOPLPUSH. */
        l = server.ready_keys;
        server.ready_keys = listCreate();

        while(listLength(l) != 0) {
            listNode *ln = listFirst(l);
            readyList *rl = ln->value;

            /* First of all remove this key from db->ready_keys so that
             * we can safely call signalListAsReady() against this key. */
            dictDelete(rl->db->ready_keys,rl->key);

            /* If the key exists and it's a list, serve blocked clients
             * with data. */
            robj *o = lookupKeyWrite(rl->db,rl->key);
            if (o != NULL && o->type == OBJ_LIST) {
                dictEntry *de;

                /* We serve clients in the same order they blocked for
                 * this key, from the first blocked to the last. */
                de = dictFind(rl->db->blocking_keys,rl->key);
                if (de) {
                    //获取所有该key block的client list
                    list *clients = dictGetVal(de);
                    int numclients = listLength(clients);

                    while(numclients--) {
                        //取出最先被阻塞的client
                        listNode *clientnode = listFirst(clients);
                        client *receiver = clientnode->value;
                        robj *dstkey = receiver->bpop.target;
                        int where = (receiver->lastcmd &&
                                     receiver->lastcmd->proc == blpopCommand) ?
                                    LIST_HEAD : LIST_TAIL;
                        //将目标list中pop出一个元素
                        robj *value = listTypePop(o,where);

                        if (value) {
                            /* Protect receiver->bpop.target, that will be
                             * freed by the next unblockClient()
                             * call. */
                            if (dstkey) incrRefCount(dstkey);
                            //unblock client
                            unblockClient(receiver);

                            //响应unblock的client
                            if (serveClientBlockedOnList(receiver,
                                rl->key,dstkey,rl->db,value,
                                where) == C_ERR)
                            {
                                /* If we failed serving the client we need
                                 * to also undo the POP operation. */
                                 	//unblock client失败,重新将元素加入到list中
                                    listTypePush(o,value,where);
                            }
                        } else {
                            //如果从list pop出的元素为NULL,则将跳出循环,client将继续阻塞
                            break;
                        }
                    }
                }

                if (listTypeLength(o) == 0) {
                	//如果list的长度为0,则从db中delete掉该list
                    dbDelete(rl->db,rl->key);
                }
                /* We don't call signalModifiedKey() as it was already called
                 * when an element was pushed on the list. */
            }

            /* Free this item. */
            decrRefCount(rl->key);
            zfree(rl);
            listDelNode(l,ln);
        }
        listRelease(l); /* We have the new list on place at this point. */
    }
}
```

1. server会遍历`server.ready_keys`中所有ready的key,获取该key阻塞的所有的client, 然后按照最先最先阻塞最先处理的原则, 来尝试unblock client;

2. 清楚阻塞标记: 调用`unblockClient`函数来从`redisDb->blocking_keys`中移除该client,  清除`client->bpop.keys`记录的该client阻塞等待的所有key, 清除`client->flags`中的`CLIENT_BLOCKED`标志位, 并是将client加入到`redisServer->unblocked_clients`中;

3. 会按照被阻塞client请求的命令(`BLPOP`, `BRPOP`或者`BRPOPLPUSH)`,从list合适的位置pop一个元素(此时list中可能不止一个元素), 来将该元素作为client的回复数据在函数`serveClientBlockedOnList`中回复给client(如果是`BRPOPLPUSH命令`则会执行响应的POP PUSH操作). 如果取出元素后回复client失败, 则会重新将该元素push到list中, 继续处理剩余被block的client;

   如果pop的数据为NULL, 表示当前list中已无元素,则会跳出循环, 结束对该ready key的处理, client将继续阻塞;

4.  最终检查目标list是否已经被pop为空, 如果为空,则会从dict中删除该记录;

   

redisServer还存在其他机制引起的client的阻塞, 这里不对其他的block及unblock机制做进一步的阐述了;

































未完待续...