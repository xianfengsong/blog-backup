---
title: 理解CMS回收器的preclean阶段
date: 2020-06-21 13:11:23
tags:
---


在《深入理解Java虚拟机：JVM高级特性与最佳实践（第二版）》里这样介绍CMS回收器的工作过程:
>CMS收集器是基于“标记—清除”算法实现的，它的运作过程相对于前面几种收集器来说更复杂一些，整个过程分为4个步骤，包括：   •初始标记（CMS initial mark）   •并发标记（CMS concurrent mark）   •重新标记（CMS remark）   •并发清除（CMS concurrent sweep）

很多人可能只看了这本书的介绍（实际这应该只是作者的概括），就认为CMS回收器就只有这4个阶段，看一下这里的gc log:
<!--more-->
```java
0.245: [GC (CMS Initial Mark) [1 CMS-initial-mark: 32776K(53248K)] 41701K(99328K), 0.0061676 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
0.251: [CMS-concurrent-mark-start]
0.270: [CMS-concurrent-mark: 0.004/0.020 secs] [Times: user=0.08 sys=0.01, real=0.02 secs] 
0.270: [CMS-concurrent-preclean-start]
0.272: [CMS-concurrent-preclean: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
0.272: [CMS-concurrent-abortable-preclean-start]
0.291: [CMS-concurrent-abortable-preclean: 0.004/0.019 secs] [Times: user=0.09 sys=0.00, real=0.02 secs] 
0.291: [GC (CMS Final Remark) [YG occupancy: 17928 K (46080 K)]0.291: [Rescan (parallel) , 0.0082702 secs]0.299: [weak refs processing, 0.0000475 secs]0.299: [class unloading, 0.0002451 secs]0.299: [scrub symbol table, 0.0003183 secs]0.300: [scrub string table, 0.0001611 secs][1 CMS-remark: 49164K(53248K)] 67093K(99328K), 0.0091462 secs] [Times: user=0.04 sys=0.00, real=0.01 secs] 
0.300: [CMS-concurrent-sweep-start]
0.300: [CMS-concurrent-sweep: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
0.300: [CMS-concurrent-reset-start]
0.300: [CMS-concurrent-reset: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

注意日志中“增加”了concurrent-preclean，concurrent-abortable-preclean，concurrent-reset三个阶段，其中concurrent-reset主要是CMS为了下次gc做准备，而重置内部数据结构的过程，这里不做过多介绍。本文主要分享一下我对**concurrent-preclean** 和 **concurrent-abortable-preclean**两个阶段的理解。

## concurrent-preclean

### 分代的理想状态

我们知道jvm利用**分治**的思想，把内存分为老年代和年轻代，在理想情况下，不同分代中的对象彼此不会有引用关系，属于老死不相往来的状态。

{% asset_img gen1.png image %}

在gc开始时，为了标记存活对象，gc线程需要沿着对象引用路径从gc root遍历（可达性分析），这时为了得到正确的引用关系，需要暂停应用线程，所以遍历对象的代价很高。因此，每个分代的垃圾回收器都希望**各扫门前雪**，只标记自己分代的存活对象，比如young gc时，当gc线程遇到引用指向老年代时就会停止遍历，因为它只负责回收年轻代内存空间，不需要去访问老年代对象。

{% asset_img gc.jpg image %}

但是实际上，年轻代和老年代的对象并不会100%的彼此隔离，会有一些对象引用其他分代对象，这被称为**跨代引用**。如果young gc线程只遍历年轻代内的对象引用，那么老年代到年轻代的跨代引用就会被忽略，被老年代存活对象跨代引用的年轻代对象会被回收，这样就破坏了应用程序的运行。下图展示了实际的对象引用情况，红色箭头代表跨代引用，在young gc时需要标记出来：

{% asset_img gen2.png image %}

### Card Marking

在young gc时，为了**找到跨代引用**，通常有这几个方法：

1. 当对象引用路径指向老年代时继续遍历老年代对象找到跨代引用
2. 线性地扫描老年代对象，标记跨代引用，用顺序读代替离散读
3. 从程序开始运行，就使用一个集合记录所有跨代引用的创建，在young gc时扫描这个集合里指向年轻代的跨代引用


前两种方式都需要在young gc时去遍历老年代对象，因为老年代存活对象多，工作量太大，jvm使用的是第三种方式。

首先分析**跨代引用如何产生的**：对于老年代到年轻代的跨代引用（a->b），产生条件有两种，一是gc线程把对象a从年轻代移动到了老年代，二是a本身是老年代对象，应用线程修改了a的引用指向了年轻代的b（
对于年轻代到老年代的跨代引用就只有第二种情况）。

对于gc线程本身创建的跨代引用，可以直接由gc线程在创建时记录，所以问题就变成了：**如何记录应用线程修改对象引用时创建的跨代引用？**。

在jvm中再次使用分治法，将老年代划分成多个card（和linux内存page类似），只要card内对象引用被应用线程修改，就把card标记为dirty。然后young gc时会扫描老年代中dirty card对应的内存区域，记录其中的跨代引用，这种方式被称为**Card Marking**。

jvm通过写屏障（write barrier）来实现监控程序线程对引用的修改，并且标记对应card，写屏障工作方式和**代理模式**类似，具体来说是通过在引用赋值指令执行时，添加对了card table的修改指令。以最简单的`setFoo(Object bar)` 方法为例，[jvm编译的汇编指令如下](http://psy-lob-saw.blogspot.com/2014/10/the-jvm-write-barrier-card-marking.html),第一行是赋值指令，后面几行标记被修改的引用所在的card为脏页，即`CARD_TABLE[this address >> 9] = 0`：


```c
; rsi is 'this' address
; rdx is setter param, reference to bar
; JDK6:
mov    QWORD PTR [rsi+0x20],rdx  ; this.foo = bar
mov    r10,rsi                   ; r10 = rsi = this
shr    r10,0x9                   ; r10 = r10 >> 9;
mov    r11,0x7ebdfcff7f00        ; r11 is base of card table, imagine byte[] CARD_TABLE
mov    BYTE PTR [r11+r10*1],0x0  ; Mark 'this' card as dirty, CARD_TABLE[this address >> 9] = 0
```


**小结**：jvm使用card marking的方式，避免了young gc时扫描整个老年代存活对象，付出的代价是在每次修改引用时添加额外的汇编指令实现写屏障，和额外的内存来保存card table。


### preclean做了什么

现在回到cms回收器，在老年代gc时，同样使用到了card marking，目的不是找到跨代引用（年轻代到老年代的跨代引用是通过从gc root遍历对象标记的），而是找到前面concurrent-marking阶段被应用线程并发修改的对象引用。

preclean阶段是**对这些card marking产生的dirty card进行clean**，cms gc线程会扫描dirty card对应的内存区域，更新之前记录的过时的引用信息，并且去掉dirty card的标记，如下图所示：

![引用自https://plumbr.io/](https://plumbr.io/app/uploads/2015/06/g1-08.png)

在preclean执行后，dirty card被清理，被修改的引用信息也被更新。

![引用自https://plumbr.io/](https://plumbr.io/app/uploads/2015/06/g1-09.png)

## concurrent-abortable-preclean

concurrent-abortable-preclean阶段目的是减轻final remark阶段（会暂停应用线程）的负担，这个阶段同样会对dirty card的扫描/清理，和concurrent-preclean的区别在于，concurrent-abortable-preclean会重复地以迭代的方式执行，直到满足退出条件。**但是concurrent-preclean已经处理过dirty card,为什么jvm还需要再执行一个类似的阶段呢？**

### 连续STW

首先我们考虑下这个情况：如果final-remark阶段开始时刚好进行了young gc（比如ParNew）,应用程序刚因为young gc暂停，然后又会因为final-remark暂停，造成**连续的长暂停**。除此之外，因为young gc线程修改了存活对象的引用地址，会产生很多需要重新扫描的对象，增加了final-remark的工作量。
所以concurrent-abortable-preclean除了clean card的作用，还有**调度final-remark开始时机**的作用[参考](https://blogs.oracle.com/poonam/understanding-cms-gc-logs)。cms回收器认为，final-remark最理想的执行时机就是年轻代占用在50%时，这时刚好处于上次young gc完成（0%）和下次young gc开始（100%）的中间节点，如图所示：

{% asset_img abpreclean.png image %}

### 配置参数

abortable-preclean的**中断条件**，配置参数是`-XX:CMSScheduleRemarkEdenPenetration=50`（默认值），表示当eden区内存占用到达50%时，中断abortable-preclean，开始执行final-remark，对应[jvm源码](https://github.com/JetBrains/jdk8u_hotspot/blob/12d02c91b0b4c63d04daff4e36a79bb6045b2c7f/src/share/vm/gc_implementation/concurrentMarkSweep/concurrentMarkSweepGeneration.cpp)片段如下：

```c
//当eden占用比例超过配置，将_abort_preclean标记赋值为true
if ((_collectorState == AbortablePreclean) && !_abort_preclean) {
    size_t used = get_eden_used();
    size_t capacity = get_eden_capacity();
    assert(used <= capacity, "Unexpected state of Eden");
    if (used >  (capacity/100 * CMSScheduleRemarkEdenPenetration)) {
      _abort_preclean = true;
    }
  }
```

abortable-preclean的**触发条件**配置， `-XX:CMSScheduleRemarkEdenSizeThreshold=2m`（默认值），表示当eden内存占用超过2mb时才会执行abortable-preclean，否则没有执行的必要。


abortable-preclean的**主动退出条件**配置，`-XX:CMSMaxAbortablePrecleanTime=5000`和CMSMaxAbortablePrecleanLoops，主要因为如果年轻代内存占用增长缓慢，那么abortable-preclean要长时间执行，可能因为preclean赶不上应用线程创造dirty card的速度导致dirty card越来越多，此时还不如执行一个final-remark,对应jvm源码片段如下：

```c
// Try and schedule the remark such that young gen
// occupancy is CMSScheduleRemarkEdenPenetration %.
// 保留原始注释，看下abortable_preclean的定位
void CMSCollector::abortable_preclean() {
  //校验触发条件
  if (get_eden_used() > CMSScheduleRemarkEdenSizeThreshold) {

    // 感受一下作者的纠结，他认为目前的主动退出条件有点蠢，FIX ME!!! 哈哈
    // One, admittedly dumb, strategy is to give up
    // after a certain number of abortable precleaning loops
    // or after a certain maximum time. We want to make
    // this smarter in the next iteration.
    // XXX FIX ME!!! YSR
    size_t loops = 0, workdone = 0, cumworkdone = 0, waited = 0;
    //should_abort_preclean会检查上面说的_abort_preclean是否为true
    while (!(should_abort_preclean() ||
             ConcurrentMarkSweepThread::should_terminate())) {
      workdone = preclean_work(CMSPrecleanRefLists2, CMSPrecleanSurvivors2);
      cumworkdone += workdone;
      loops++;
      // 主动停止执行
      if ((CMSMaxAbortablePrecleanLoops != 0) &&
          loops >= CMSMaxAbortablePrecleanLoops) {
        if (PrintGCDetails) {
          gclog_or_tty->print(" CMS: abort preclean due to loops ");
        }
        break;
      }
      if (pa.wallclock_millis() > CMSMaxAbortablePrecleanTime) {
        if (PrintGCDetails) {
          gclog_or_tty->print(" CMS: abort preclean due to time ");
        }
        break;
      }
      // 如果工作效率不高，主动暂停一会儿
      if (workdone < CMSAbortablePrecleanMinWorkPerIteration) {
        // Sleep for some time, waiting for work to accumulate
        stopTimer();
        cmsThread()->wait_on_cms_lock(CMSAbortablePrecleanWaitMillis);
        startTimer();
        waited++;
      }
    }
    //打印工作情况
    if (PrintCMSStatistics > 0) {
      gclog_or_tty->print(" [%d iterations, %d waits, %d cards)] ",
                          loops, waited, cumworkdone);
    }
  }
  return;
}
```

## 实践验证

java代码如下：

```java

 public class DumbObj {
    public DumbObj(int sizeM,DumbObj next) {
        this.data = getM(sizeM);
        this.next = next;
    }
    private Byte[] getM(int m) {
        return new Byte[1024 * 1024 * m];
    }
    private DumbObj next;
    private Byte [] data;
 }
 //存活对象
 private static List<DumbObj> liveObjs = new ArrayList<>(5);
 public static void main(String[] args) throws InterruptedException {
        //创建新对象触发gc
        for(int i=0;i<25;i++){
            DumbObj dumb = new DumbObj(1, null);
            if(liveObjs.size()<5){
                liveObjs.add(new DumbObj(1, dumb));
            }else{
                dumb.setNext(liveObjs.get(i%5));
            }
        }
        //等待gc线程工作
        TimeUnit.SECONDS.sleep(20);
    }

```
在jvm参数里添加-XX:PrintCMSStatistics=1，通过gc日志可以看到cms回收器在preclean阶段执行的操作：
>-Xms101m
-Xmn50m
-Xmx101m
-verbose:gc
-XX:MetaspaceSize=1m
-XX:+UseConcMarkSweepGC
-Xloggc:/tmp/gc.log
-XX:+PrintGCCause
-XX:+PrintGCTimeStamps
-XX:+PrintGCDetails
-XX:PrintCMSStatistics=1
-XX:CMSScheduleRemarkEdenPenetration=50
-XX:CMSScheduleRemarkEdenSizeThreshold=2m
-XX:CMSMaxAbortablePrecleanTime=5000
-XX:+UseCMSInitiatingOccupancyOnly
-XX:CMSInitiatingOccupancyFraction=50

运行程序，查看gc日志:在时间0.303，concurrent-preclean开始，重新扫描了5个card,在0.304时，开始abortable-preclean-start，多个线程又进行了一次迭代，扫描dirty card,在5秒后，5.324时，因为达到最大运行时间主动退出，开始remark阶段。

```java
0.303: [CMS-concurrent-mark: 0.010/0.010 secs] (CMS-concurrent-mark yielded 0 times)
 [Times: user=0.02 sys=0.00, real=0.01 secs] 
0.303: [CMS-concurrent-preclean-start]
 (cardTable: 5 cards, re-scanned 5 cards, 1 iterations)
0.304: [CMS-concurrent-preclean: 0.000/0.000 secs] (CMS-concurrent-preclean yielded 0 times)
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
0.304: [CMS-concurrent-abortable-preclean-start]
 (cardTable: 0 cards, re-scanned 0 cards, 1 iterations)
 (cardTable: 0 cards, re-scanned 0 cards, 1 iterations)
 (cardTable: 0 cards, re-scanned 0 cards, 1 iterations)
 (cardTable: 0 cards, re-scanned 0 cards, 1 iterations)
 (cardTable: 0 cards, re-scanned 0 cards, 1 iterations)
 (cardTable: 0 cards, re-scanned 0 cards, 1 iterations)
 (cardTable: 0 cards, re-scanned 0 cards, 1 iterations)
 //有省略
 CMS: abort preclean due to time  [50 iterations, 49 waits, 0 cards)] 5.324: [CMS-concurrent-abortable-preclean: 0.012/5.020 secs] (CMS-concurrent-abortable-preclean yielded 0 times)
 [Times: user=0.02 sys=0.00, real=5.02 secs] 
5.324: [GC (CMS Final Remark) [YG occupancy: 17157 K (46080 K)]5.324: [Rescan (parallel)  (Survivor:0chunks) Finished young gen rescan work in 4th thread: 0.000 sec
```
修改-XX:CMSScheduleRemarkEdenSizeThreshold=50m,和年轻代大小相等，再观察gc日志，不会出现concurrent-abortable-preclean阶段：

```java
2.296: [CMS-concurrent-mark: 0.010/0.010 secs] (CMS-concurrent-mark yielded 0 times)
 [Times: user=0.02 sys=0.00, real=0.01 secs] 
2.296: [CMS-concurrent-preclean-start]
 (cardTable: 1 cards, re-scanned 1 cards, 1 iterations)
2.296: [CMS-concurrent-preclean: 0.000/0.000 secs] (CMS-concurrent-preclean yielded 0 times)
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
2.296: [GC (CMS Final Remark) [YG occupancy: 17157 K (46080 K)]2.296: [Rescan (parallel)  (Survivor:0chunks) Finished young gen rescan work in 4th thread: 0.000 sec
```

## 总结
跨代引用和card marking:
{% asset_img summary.png image %}


preclean: 清理card marking标记的dirty card，更新引用记录

abortable-preclean: 调节final-remark阶段的运行时机

参考文章：

Brian Goetz的文章: https://www.ibm.com/developerworks/library/j-jtp11253/index.html
card mark介绍: http://psy-lob-saw.blogspot.com/2014/10/the-jvm-write-barrier-card-marking.html
理解preclean: https://stackoverflow.com/questions/44182733/can-someone-explain-what-happens-in-the-concurrent-abortable-preclean-phase-of
jvm源码： https://github.com/JetBrains/jdk8u_hotspot

[1]: https://github.com/JetBrains/jdk8u_hotspot/blob/12d02c91b0b4c63d04daff4e36a79bb6045b2c7f/src/share/vm/gc_implementation/concurrentMarkSweep/concurrentMarkSweepGeneration.cpp 'git'




