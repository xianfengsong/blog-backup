---
title: 为什么别人能把java gc讲清楚
date: 2020-05-27 02:13:10
tags: [java基础]
---

你关于JavaGC的知识都是从哪儿学习的呢?是看博客或者看书还是看oracle的文档?今天来推荐 [Plumbr.io][1]上的一个文章(Plumbr是一家提供jvm监控和优化方案的公司),标题叫《Java Garbage Collection handbook》整个文章大概有75页,包括GC概念介绍/算法介绍/算法实现/gc调优等7个小节,这里只引用其中几个段落,来看一下作者是怎么介绍JavaGC知识的.

<!--more-->

### 形象地概括 Garbage Collection
在[费曼学习法][2]中有这样一个逻辑:如果你不能把一个概念简化到让一个小孩子都能理解,那么说明你还没有掌握它. 在plumbr.io的文档第一节,作者形象地把gc和生活中的清理垃圾做对比: 打扫垃圾时我们会把垃圾挑出来然后扔掉, 而gc的过程是把需要保留的内存对象挑出来, 然后清理剩下的部分. **在文章开始先给出通俗又准确的定义,让别人对核心概念有一个认识**.

>At first sight, garbage collection should be dealing with what the name suggests – finding and throwing away the garbage. In reality it is doing exactly the opposite. Garbage Collection is tracking down all the objects that are still used and marks the rest as garbage.

### 先说为什么

我们都知道jvm内存模型中堆内存是分代的,**那么为什么分代呢?又为什么要分成新生代和老年代呢?**

在文章中,作者先介绍分代的原因:
首先每次GC处理的对象越少,那么STW暂停的时间越短,所以把对象分为几个部分(分代也是一种分治的思想)
>As we have mentioned before, doing a garbage collection entails stopping the application completely. It is also quite obvious that the more objects there are the longer it takes to collect all the garbage.But what if we would have a possibility to work with smaller memory regions? 

而分为young/old两部分,是因为根据研究人员通过对Java程序中的对象的观察得出的分代假设(Weak Generational Hypothesis),根据存活时间的不同,对象自然分为两个部分:
{% asset_img 1.png image %}

介绍JVM堆内存组成前,作者先介绍了Weak Generational Hypothesis,**先说为什么再说是什么**,可以让人更容易理解. 比如,在了解对象分代假设之后,因为通常对象只要活过一段时间后(中间的谷底),大多数都能活很久(右边),所以在young generational中添加了survivor区域,并且存活的对象要在两个survivor区域间复制几次才会进入老年代.

>This process of copying the live objects between the two Survivor spaces is repeated several times until some objects are considered to have matured and are ‘old enough’. Remember that, based on the generational hypothesis, objects which have survived for some time are expected to continue to be used for very long time.

{% asset_img 2.png image %}

### 使用图片描述细节
cms回收器的工作流程步骤比较多(有些书简化成了4步,明显和GC日志的输出对应不上),plumbr的在介绍cms回收器时使用一些精致的图片,来描述gc时内存中对象的变化.
在Initial Mark阶段,图片上特别画出了跨代引用的对象,因为这个阶段只标记gc root直接引用的对象,所有有三个绿色obj.
{% asset_img 3.png image %}
而在concurrent mark阶段,又因为cms回收器会遍历对象,图中把间接引用的对象标记绿色.
{% asset_img 4.png image %}
画的最好的是preclean阶段,图中标记出了跨代引用更新之后,因为[card marking机制][7]被标记为dirty的card被clean的过程(红色的card被更新,看了这个两个图,我才理解preclean的过程和card marking的作用)
{% asset_img 5.png image %}
{% asset_img 6.png image %}
### 总结
最容易骗你的就是你自己,所以有时候你感觉自己懂了并不能说明你懂了,如果你能让别人也懂那你大概是真的懂,而让别人懂还需要有讲述问题的技巧才行.用一张图来总结plumbr的文章
![nb](/images/nb.jpeg)

***想看《Java Garbage Collection handbook》完整版的朋友,在公众号后台回复"gc"即可***

  [1]: https://plumbr.io/java-garbage-collection-handbook
  [2]: https://www.zhihu.com/question/20576786/answer/17338936
  [3]: https://plumbr.io/app/uploads/2015/05/object-age-based-on-GC-generation-generational-hypothesis.png
  [4]: https://plumbr.io/app/uploads/2015/05/how-java-garbage-collection-works.png
  [5]: https://plumbr.io/app/uploads/2015/06/g1-06.png
  [6]: https://plumbr.io/app/uploads/2015/06/g1-07.png
  [7]: http://psy-lob-saw.blogspot.com/2014/10/the-jvm-write-barrier-card-marking.html
  [8]: https://plumbr.io/app/uploads/2015/06/g1-08.png
  [9]: https://plumbr.io/app/uploads/2015/06/g1-09.png