
title: 为什么Redis使用ziplist能节省内存？
date: 2017-09-12 21:15:46
categories: redis进阶
tags: [redis,源码,nosql]

---


**环境准备:**
redis 2.8.19 单机模式
保存500万条用户信息数据，key是用户名 ，value是json格式字符串，包含邮箱等信息，内容如下:
`userid:  "tomcat125"
info:  "[null,\"tomcat125@gmail.com\",188XXXXXXXX,\"\",null,null,\"\xe9\x9c\x8d\xe6\xb3\x13\xx4\xb8\xs0\",\"1\",null,\"tom123321\"]"`


redis和ziplist配置有关的有这几个，默认值如下：
<!--more-->

`config get *ziplist*`
 >1) "hash-max-ziplist-entries"
 2) "512"
 3) "hash-max-ziplist-value"
 4) "64"
 5) "list-max-ziplist-entries"
 6) "512"
 7) "list-max-ziplist-value"
 8) "64"
 9) "zset-max-ziplist-entries"
10) "128"
11) "zset-max-ziplist-value"
12) "64"


-entries表示ziplsit对多保存数据项的个数，超出之后不能再使用ziplist
-value表示每个数据项的最大字节数，超出之后不能再使用ziplist

## 一.使用ziplist真的能节省内存吗？

在这我用hash对象用户信息，对比一下使用ziplist前后的内存差异：
为了使用hash保存，对用户名做hash然后取模得到一个hashkey
然后保存时执行命令：`hset hashkey username info`

### 使用hashtable保存：
ziplist设置：
>config get hash*
1) "hash-max-ziplist-entries"
2) "512"
3) "hash-max-ziplist-value"
4) "64"

hash对象信息(hashtable保存)：
>“debug object nozip:3134”
Value at:0x7f4957621d40 refcount:1 
**encoding**:**hashtable** 
**serializedlength:39927** 
lru:3356615 lru_seconds_idle:24



内存占用：
>“dbsize”
(integer) 12000
info Memory
\# Memory
used_memory:1102326968
**used_memory_human:1.03G**
used_memory_rss:1150930944
used_memory_peak:1102463200
used_memory_peak_human:1.03G
used_memory_lua:35840
mem_fragmentation_ratio:1.04



清理内存，重启redis
```bash
redis>flushall
systemctl stop redis.service
systemctl start redis.service
```


### 使用ziplist保存:
修改ziplist设置(json串平均长度150)：
>"config get hash*"
1) "hash-max-ziplist-entries"
2) "1000"
3) "hash-max-ziplist-value"
4) "250"


hash对象信息（ziplist保存）：
>"debug object nozip:3134"
Value at:0x7f490d21c150 refcount:1
**encoding:ziplist** 
**serializedlength:18713** 
lru:3360532 lru_seconds_idle:93



内存占用：

>"dbsize"
(integer) 12000
info Memory
\# Memory
used_memory:573333824
**used_memory_human:546.77M**
used_memory_rss:633442304
used_memory_peak:1102178520
used_memory_peak_human:1.03G
used_memory_lua:35840
mem_fragmentation_ratio:1.10




在上面的例子中可以看到利用ziplist在保存12000个hash对象时，**used_memory节省了近50%**
同一个key的serializedlength也差不多是原来的一半(39927-18731)，但是serializedLength表示的不是key的实际内存占用，而是保存到rdb文件之后key的占用字节数，小于key的实际内存占用。
（[参考serializedLength源][1]）

## 二.ziplist为什么能节省内存？

从debug object的输出信息来看，两次分别是使用hashtable和ziplist保存的，为什么ziplist会比hashtable节省内存呢，这要从二者的数据结构说起。
### 1.hashtable结构：
hashtable使用字典保存，字典包含两个hashtable,hashtable由entry数组构成，数组中的节点包含一个next指针，形成一个单向链表。
redis字典定义：
```c
//dict.h
//字典
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int iterators; /* number of iterators currently running */
} dict;
//字典内部hashtable
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;
//hashtable节点
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    struct dictEntry *next;
} dictEntry;
```
下图展示了一个普通状态下的字典,数据保存在ht[0]，ht[1]在扩容时使用：
![ht][2]
### 2.ziplist结构：
**zlbytes**：表示ziplist占用字节数，在执行resize操作时使用
**zltail**：表示最后节点的偏移量，也是避免了整体遍历list
**zllen**：表示ziplist节点个数（节点数超过65535,zllen字段值无效,需要遍历才能得到真实数量）
**zlend**：表示ziplist结束的标识符

**ziplist节点数据结构（抽象）：**
每个压缩列表节点都由previous_entry_length、encoding、content三个部分组成（不是指实际结构体的字段）
**previous_entry_length**：前一个节点的长度，用来由后向前遍历，根据前一个节点的长度，可能需要一个或五个字节。
**encoding**：记录节点保存的数据类型和数据长度。
**content**：节点保存的数据内容。
ziplist节点结构体（和上面的有点不一样）:
```c
//ziplist.c
typedef struct zlentry {
    unsigned int prevrawlensize, prevrawlen;
    unsigned int lensize, len;
    unsigned int headersize;
    unsigned char encoding;
    unsigned char *p;
} zlentry;
```
下图展示了一个ziplist的基本结构：
![ziplist][3]

可以看到，ziplist采用了一段连续的内存来存储数据，相比hashtable减少了内存碎片，和指针的内存占用。而且当节点较少时，ziplist更容易被加载到CPU缓存中。

## 三，调试源码
眼见为实，现在通过vs2012启动redis server，调试redis源码对比hashtable和ziplist的处理过程。

### 调试hashtable保存hash对象
server启动后，首先在t_hash.c 的hsetCommand函数打断点
```c
//t_hash.c
void hsetCommand(redisClient *c) {
    int update;
    robj *o;

    if ((o = hashTypeLookupWriteOrCreate(c,c->argv[1])) == NULL) return;
    hashTypeTryConversion(o,c->argv,2,3);
    hashTypeTryObjectEncoding(o,&c->argv[2], &c->argv[3]);
    update = hashTypeSet(o,c->argv[2],c->argv[3]);
    addReply(c, update ? shared.czero : shared.cone);
    signalModifiedKey(c->db,c->argv[1]);
    server.dirty++;
}
```
在客户端执行命令,修改ziplist配置最大长度改成10，然后添加一个value长度超过10的hash对象：
```
127.0.0.1:6379> config set hash-max-ziplist-value 10
127.0.0.1:6379> hset hash_key fieldkey abcdefghijklmn
```

进入t_hash.c文件的函数hashTypeSet,在这里redis判断对象编码为hashtable，使用字典保存:
```c
//t_hash.c
else if (o->encoding == REDIS_ENCODING_HT) {
        if (dictReplace(o->ptr, field, value)) { /* Insert */
            incrRefCount(field);
        } else { /* Update */
            update = 1;
        }
        incrRefCount(value);
```
在dict.c文件dictAdd标记断点，开始分别保存key&value到dict：
```c
//dict.c
int dictAdd(dict *d, void *key, void *val)
{
    //往字典中添加一个只有key的dictEntry
    dictEntry *entry = dictAddRaw(d,key);

    if (!entry) return DICT_ERR;
    //保存entry值的内容
    dictSetVal(d, entry, val);
    return DICT_OK;
}
```
首先redis调用函数dictAddRaw()保存hash key
```c
//dict.c
dictEntry *dictAddRaw(dict *d, void *key)
{
    int index;
    dictEntry *entry;
    dictht *ht;
    //因为redis采用渐进式rehash，首先要做rehash检查
    if (dictIsRehashing(d)) _dictRehashStep(d);

    /* Get the index of the new element, or -1 if
     * the element already exists. */
    if ((index = _dictKeyIndex(d, key)) == -1)
        return NULL;

    // 如果字典正在rehash，则插入到新表ht[1]，否则到旧表ht[0]
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    /* 申请内存 保存新的entry节点*/
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    /* 保存entry key的内容 */
    dictSetKey(d, entry, key);
    return entry;
}
```
**估算内存**
因为内存对齐的原因，sizeof(dictEntry)=24，为entry申请了24字节内存。估算一个节点的字节数 **47byte**(key 9bytes,val 14bytes,entry 24bytes)
### 调试ziplist保存hash对象
首先清理数据，修改ziplist配置放宽value长度限制，保存同样内容：
```
127.0.0.1:6379> flushall
127.0.0.1:6379> config set hash-max-ziplist-value 15
127.0.0.1:6379> hset hash_key fieldkey abcdefghijklmn
```
此时还是在t_hash.c文件的函数hashTypeSet打断点,redis选择用ziplist编码保存hash对象
```c
//t_hash.c hashTypeSet()
if (o->encoding == REDIS_ENCODING_ZIPLIST) {
        unsigned char *zl, *fptr, *vptr;

        field = getDecodedObject(field);
        value = getDecodedObject(value);

        zl = o->ptr;
        fptr = ziplistIndex(zl, ZIPLIST_HEAD);
        if (fptr != NULL) {
            //略
        }

        if (!update) {
            /* Push new field/value pair onto the tail of the ziplist */
            zl = ziplistPush(zl, field->ptr, (unsigned int)sdslen(field->ptr), ZIPLIST_TAIL);
            zl = ziplistPush(zl, value->ptr, (unsigned int)sdslen(value->ptr), ZIPLIST_TAIL);
        }
        o->ptr = zl;
        //引用计数
        decrRefCount(field);
        decrRefCount(value);

        /*检查是否要换成hashtable编码*/
        if (hashTypeLength(o) > server.hash_max_ziplist_entries)
            hashTypeConvert(o, REDIS_ENCODING_HT);
    }
```
在ziplistPush内部调用了 ziplistInsert()函数，打开ziplist.c文件，在__ziplistInsert()函数打断点：

```c   
//ziplist.c
static unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    //当前ziplist的长度
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen;
    unsigned int prevlensize, prevlen = 0;
    size_t offset;
    int nextdiff = 0;
    unsigned char encoding = 0;
    long long value = 123456789; 
    zlentry tail;

    if (p[0] != ZIP_END) {
        // 如果p节点后面还有节点，取出p节点前一个节点的长度信息和存储该长度值所需要的字节数信息
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
    } else {
        // 如果p节点为ziplist结束标识，则取出尾节点，即最后一个节点
        unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
        if (ptail[0] != ZIP_END) {
            prevlen = zipRawEntryLength(ptail);
        }
    }

    // 尝试看能否将s保存为整数，如果可以则返回1，且value和encoding分别保存新值和编码信息
    if (zipTryEncoding(s,slen,&value,&encoding)) {
        // 如果s可以保存为整数，则进一步计算保存该数值所需要的字节数
        reqlen = zipIntSize(encoding);
    } else {
        // 如果s不能保存为整数，则直接使用其字符串长度
        reqlen = slen;
    }
    
    // 计算编码prevlen所需要的字节数，prevlen用于保存前一个节点的长度
    reqlen += zipPrevEncodeLength(NULL,prevlen);
    // 计算编码slen所需要的长度
    reqlen += zipEncodeLength(NULL,encoding,slen);

    
    // 当插入的位置不是ziplist尾部时，需要确保下一个节点（即p节点）的prevlen能够用来保存即将插入节点的长度
    // 这里计算两者差值
    nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;

    // ziplistResize操作会重新分配空间，需要事前记录p节点偏移量
    offset = p-zl;
    zl = ziplistResize(zl,curlen+reqlen+nextdiff);
    // 重新取得p节点
    p = zl+offset;

    /* Apply memory move when necessary and update tail offset. */
    if (p[0] != ZIP_END) {
        /* 将原来 p-nextdiff 开始的数据全部后移，中间出现reqlen个字节保存即将插入的数据 
            主要需要考虑一下几种情况：
            nextdiff == 0：p节点中用来存储原先前一个节点长度信息的数据区域正好保存待插入节点的长度
            nextdiff == 4：原先p节点只需要1个字节来存储上一个节点的长度，现在需要5个字节。那就将p-4后面的数据偏移到p+reqlen
            nextdiff == -4：原先p节点需要5个字节来存储上一个节点的长度，现在只需要1个字节。那就将p+4后面的数据偏移到p+reqlen
        */
        memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

        // 为p节点的prevlen设置新值，即待插入节点的长度
        zipPrevEncodeLength(p+reqlen,reqlen);

        // 更新尾节点偏移量
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

        tail = zipEntry(p+reqlen);
        // 同样，如果p节点不是尾节点，尾节点的偏移量还需要加上nextdiff值
        if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
        }
    } else {
        // 如果p节点指向zlend，更新zltail值，待添加节点为尾部节点
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
    }
    
    // 同样，如果nextdiff的值不为0，说明原节点（此时的首地址为p+reqlen）的长度发生改变，需要执行级联更新操作
    if (nextdiff != 0) {
        offset = p-zl;
        //这里可能造成连锁更新
        zl = __ziplistCascadeUpdate(zl,p+reqlen);
        p = zl+offset;
    }

    // 下面才是真正执行插入操作
    /* Write the entry */
    // 填写上一节点的长度
    p += zipPrevEncodeLength(p,prevlen);
    // 填写当前节点的长度
    p += zipEncodeLength(p,encoding,slen);
    // 根据编码方式执行相应的插入操作
    if (ZIP_IS_STR(encoding)) {
        memcpy(p,s,slen);
    } else {
        zipSaveInteger(p,value,encoding);
    }
    // 长度加1
    ZIPLIST_INCR_LENGTH(zl,1);
    return zl;
}

```
**估算内存**
curlen表示当前ziplist的占用字节数,curlen初始值11，保存"fieldkey"后变为21，保存"abcdefghijklmn"变成38，估计一个节点占用**27字节**。

**注意：**当在ziplist头部执行增加或删除操作时，如果节点长度过长（超过255），previous_entry_length要使用5个字节保存，这时会执行__ziplistCascadeUpdate()函数，可能触发连锁更新，最坏情况要更新每一个后继节点（previous_entry_length从一个字节变成5个字节后，所有节点长度都超过了255）。

## 总结
**ziplist的优点**
内存占用少 容易被加载到CPU缓存
结构紧凑 减少内存碎片

**ziplist的缺点**
连锁更新（头部插入 或 删除，对头部的操作主要是在用ziplist保存list类型对象时发送，保存hash对象不需要在头部更新。）
查询复杂度从O（1）变成O（N）（保存hash对象时）

	

参考：
[Redis内置数据结构之压缩列表ziplist][4]
[vs2012调试redis][5]
[《redis设计与实现》][6]


  [1]: https://github.com/antirez/redis/blob/4082c38a60eedd524c78ef48c1b241105f4ddc50/src/debug.c#L337-L343
  [2]: http://7xl4v5.com1.z0.glb.clouddn.com/ht.png
  [3]: http://7xl4v5.com1.z0.glb.clouddn.com/ziplist.png
  [4]: http://blog.csdn.net/xiejingfa/article/details/51072326
  [5]: http://blog.csdn.net/Rongbo_J/article/details/45288223
  [6]: http://www.duokan.com/book/53962