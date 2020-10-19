---
title: 如何设计布隆过滤器(1)
date: 2020-08-19 11:53:39
tags: [中间件]
---

## what.布隆过滤器是什么

![示意图](https://upload.wikimedia.org/wikipedia/commons/a/ac/Bloom_filter.svg "示意图")

<!--more-->

和java的hashmap类似，布隆过滤器(后面用BF代替)是一个使用hash算法+bit数组实现的数据结构，一般只提供**add()/contains()** 两个接口.
BF比hashmap使用更少的存储空间，但是BF.contains(x)在返回true时，x可能并不存在BF中，这个情况发生的概率这里用**false positive**（fp）表示。不过bloomfilter.contains(x)返回false时，可以**100%**保证x一定不在BF中。

fp存在的原因是，在hash冲突时，hashmap可以通过元素的原始值来查找元素，但是**为了追求更小的空间占用**，BF不可能保存元素的原始值，所以hash冲突发生时，contains()会直接返回true，虽然牺牲了准确率，不过BF可以保证fp不超过期望值。
因为fp和hash冲突的概率正相关，为了减少hash冲突，BF会使用多个独立的hash函数来定位一个元素，如果一个hash函数冲突的概率是1/m,那么k个hash函数都发生冲突的概率是(1/m)^k。

BF还有其他类型，比如提供resize()方法的可以扩展的BF,提供count()/delete()方法的计数BF。
| 方法    | hashmap   |  bloomfilter  |
| --------   | -----:  | :----:  |
|hash|一次hash计算|多次hash计算|
|add|Y|Y|
|contains|Y|Y(有错误概率)|
|get|Y|N|
|delete|Y|N（可选）|
|resize|Y|N（可选）|

## why.为什么要用布隆过滤器
BF有着出色的空间复杂度，在保存100w元素，fp=0.01条件下，只需要9585058bit,约1.1mb存储空间。
BF的时间复杂度只有常数级别，插入和查询操作复杂度为O(k),k为hash计算次数。
基础的BF实现起来也十分简单。

## when.什么时候使用 
BF一般用来实现一个黑名单的功能，比如浏览器恶意地址检查，爬虫网页地址过滤，推荐系统内容去重等。
因为hash冲突无法避免，false positive不可能为0, 所以过滤内容时必须处理“误杀”的情况。一般有**接受或二次检查**两种方式，比如在推荐系统中，少量内容的“误杀”是可以接受的。而在恶意地址检查时，浏览器不能拒绝用户访问一个正常地址，所以BF判断网址在黑名单中后，还需要用完整的网址在数据库的黑名单表中执行一次查询。

**什么时候不用**
1.要单独靠BF实现100%正确的contains()方法的场景
2.实际元素数量并不多的场景（hashmap就可以搞定）
3.如果你的元素值本身就是唯一且均匀地分布在一个**有限区域的整数**，比如ipv4地址，一定范围内的时间戳，数据库自增主键等，这时直接使用**bitmap**即可，bitmap使用更少的内存，保证100%正确并且能提供count()方法。
比如要统计用户在过去一周中哪几天有登陆行为，且用户id是int类型的数据库主键，只需要每天生成一个可以保存全部用户大小的bitmap，在用户登陆时将制定bit设置为1即可，一亿用户大约需要11mb内存，BF在相同条件下需要约100mb内存(fp=0.01)。
4.如果涉及到删除元素或者很难预估总体元素数量（需要动态扩容）的情况，基础的BF无法满足要求，需要修改内部设计才能实现。

## how.怎么实现布隆过滤器 
### 如何选择hash函数

**安全性考虑**
hash函数一般分为**加密hash和非加密hash**两种，主要区别是加密hash比非加密hash有更“好”的安全性，一般体现在hash函数找到碰撞/根据hash值反推出原始内容的成本是否足够高。常见加密hash函数:md5,sha-2，非加密hash函数:crc32,murmurhash等。

在guava和hbase的BF中使用的是**murmurhash**函数，作为非加密hash,murmurhash的速度比较快，并且已经被广泛使用。
在redission(`github.com/redisson`)的BF实现中使用的是google的**highwayhash**,是一种带token的加密hash函数,需要指定hashseed来计算hash值，我通过简单的测试发现murmurhash和highwayhash的执行速度没有明显区别(字符串长度小于500时)，更准确的性能测试可以参考`github.com/rurban/smhasher`。

选择加密的hash函数可以减少BF受到**hashflood攻击**的可能(恶意制造大量hash冲突发起dos攻击，一般数据结构在hash冲突是查询复杂度会下降，比如java从O(1)下降到O(logn))，因此在使用数据库查询做二次检查的场景，建议使用加密hash。

**速度考虑**
和BF的介绍不同，在实际使用中只需要调用一次hash函数，然后再用hash值计算出多个子hash值，这样可以减少hash运算的时间，前提是hash函数产生的hash值足够长也分布随机。

hash值的计算代码如下：
```java
//128位的hash结果通过长度为2的long数组范围
long[] hashes = HighwayHash.hash128(data, 0, data.length, HASH_KEYS);
long[] indexes = hash(hashes[0], hashes[1], hashIterations, size);
//使用hash值，进行iterations次计算
private long[] hash(long hash1, long hash2, int iterations, long size) {
    long[] indexes = new long[iterations];                                
    long hash = hash1;                                                  
    for (int i = 0; i < iterations; i++) {                              
        indexes[i] = (hash & Long.MAX_VALUE) % size;      
        //最好不要使用乘法运算
        if (i % 2 == 0) {                                               
            hash += hash2;                                              
        } else {                                                        
            hash += hash1;                                              
        }                                                               
    }                                                                   
    return indexes;                                                     
}                                                                       
```

### 如何保证fp不超过期望值 
布隆过滤器是通过调整bit数组大小和hash计算次数来控制fp在一定范围以下的。
**假设bit数组大小为m,要保存的元素数量为n,hash计算次数为k**
k的[计算过程](https://blog.csdn.net/quiet_girl/article/details/88523974 "计算过程")如下：

首先假设每个bit被设置的概率是独立的，那么某个bit被设置为1的概率是 $1/m$

在经过k次hash计算后，这个bit还没有被设置为1的概率是： $(1-1/m)^{k}$，可以近似为： $e^{-k/m}$ (自然对数e的近似公式)

>在已经保存了n个元素后，这个bit还是0的概率：$e^{-nk/m}$, 反之这个bit是1的概率： $1-e^{-nk/m}$

>判断一个元素是否存在时，如果发生hash冲突的概率：$FP=(1-e^{-nk/m})^{k}$

>这时假设函数f(k)，使$FP=f(k)=(1-e^{-nk/m})^{k}$,**让FP取最小值,则函数f(k)的导数等于0**, 经过求导和化简得到:$k=\frac{m}{n}ln2$
所以$k$是fp为最小值的情况下，即最佳条件下的hash计算次数。

m的计算过程如下：
> 前面已经得到$FP=(1-e^{-nk/m})^{k}$, 而$k=\frac{m}{n}ln2$
带入k,然后再化简得到 $m=-\frac{nlnFP}{(ln2)^2}$

在实际使用中，需要先根据fp的期望值和n计算m的大小，然后计算k的值，用代码表示：
```java
//计算m                                                                  
private long optimalNumOfBits(long n, double p) {                      
    if (p == 0) {                                                      
        p = Double.MIN_VALUE;                                          
    }                                                                  
    return (long) (-n * Math.log(p) / (Math.log(2) * Math.log(2)));    
}     
//计算k
private int optimalNumOfHashFunctions(long n, long m) {                
    return Math.max(1, (int) Math.round((double) m / n * Math.log(2)));
}                                                                      
                                                                 
```

## 总结
布隆过滤器是一个常用的数据结构，通过牺牲准确率获得更高的空间利用率，在实现上会调整bit数组大小和hash计算次数提高准确率。布隆过滤器应用场景有限，一般适用于大量数据重复过滤的情况，在使用时应该根据数据特点与hashmap,bitmap等数据结构做对比。

![关注公众号,第一时间更新](http://throwsnew.com/images/qrcode.jpg "关注公众号,第一时间更新")


***下期预告：在分布式场景，怎么用布隆过滤器实现全局过滤？***

