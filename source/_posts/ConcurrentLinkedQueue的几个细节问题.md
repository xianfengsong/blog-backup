title: ConcurrentLinkedQueue的几个细节问题
date: 2018-02-22 23:05:46
categories: jdk源码
tags: [java基础,并发]

---
所有的注释和源码都引用自jdk1.7版本的ConcurrentLinkedQueue

### ABA问题
源码注释中对这个问题的说明
> Note that like most non-blocking algorithms in this package,this implementation relies on the fact that in garbage collected systems, there is no possibility of ABA problems due to recycled nodes, so there is no need to use "counted pointers" or related techniques seen in versions used in non-GC'ed settings.

(ConcurrentLinkedQueue的实现依赖GC操作，因为GC会收节点所以不会出现ABA问题，因此不需要使用非GC(non-GC'ed)中使用的“计数指针”或相关技术）

GC能避免ABA问题，主要是因为GC能保证引用值相同时他们指向的内容一定是一样的，具体说明看这里[这个文章讲的不错][1]
<!--more-->

### Node的初始化

为了保证可见性，Node的两个属性都是volatile的
```java
private static class Node<E> {
        volatile E item;
        volatile Node<E> next;
```
但是初始化是没有直接对volatile变量赋值，初始化Node使用的是unsafe的putObject方法
```java
Node(E item) {
        /**
        * Constructs a new node.  Uses relaxed write because item can
         * only be seen after publication via casNext.
         */
            UNSAFE.putObject(this, itemOffset, item);
        }
```
源码中对此的说明
> When constructing a Node (before enqueuing it) we avoid paying for a volatile write to item by using Unsafe.putObject instead of a normal write.  This allows the cost of enqueue to be "one-and-a-half" CASes.（避免写volatile操作的开销，使得入队操作的cost相当与1.5个CAS操作）

UNSAFE的putObject相当于对普通变量的写操作（作者称之为relaxed write），因为item是volatile的所以直接对他赋值会触发volatile写（引发缓存一致性操作），使用putObject方法就是为了避免了这一操作。参考[对putObject()方法的说明][2]

我理解的是初始化的Node在执行构造函数后还需要执行casNext()方法才能被其他线程看到，因此使用"relaxed write"是安全的。既然node入队操作由初始化和casNext组成（参考offer()源码）,而执行casNext()之后保证了node的可见性，那么减少初始化操作的性能消耗就可以优化入队操作。
casNext方法：
```java
class Node{
    boolean casNext(Node<E> cmp, Node<E> val) {
            return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
        }
}
```
### 引申
对UNSAFE的一点介绍,UNSAFE的赋值操作：

1. **putXXX(long address, XXX value)**: Will place the specified value of type XXX directly at the specified address.(指定地址保存变量)
2. **putXXXVolatile(Object target, long offset, XXX value)**
Will place value at target's address at the specified offset and not hit any thread local caches(不会被加载到线程本地缓存，只保存在主存，保证可见性)
3. **putOrderedXXX(Object target, long offset, XXX value)**: Will place value at target's address at the specified offet and might not hit all thread local caches.(不保证可见性)

### offer()方法

jdk1.7中更改了offer的写法，变得更简洁
```java
public boolean offer(E e) {
        checkNotNull(e);
        final Node<E> newNode = new Node<E>(e);

        for (Node<E> t = tail, p = t;;) {
            Node<E> q = p.next;
            if (q == null) {
                // p is last node
                //Node的入队操作
                if (p.casNext(null, newNode)) {
                    // Successful CAS is the linearization point
                    // for e to become an element of this queue,
                    // and for newNode to become "live".
                    if (p != t) // hop two nodes at a time
                        casTail(t, newNode);  // Failure is OK.
                    return true;
                }
                // Lost CAS race to another thread; re-read next
            }
            else if (p == q)
                // We have fallen off list.  If tail is unchanged, it
                // will also be off-list, in which case we need to
                // jump to head, from which all live nodes are always
                // reachable.  Else the new tail is a better bet.
                p = (t != (t = tail)) ? t : head;
            else
                // Check for tail updates after two hops.
                p = (p != t && t != (t = tail)) ? t : q;
        }
    }
```
对于第24行，(t!=(t=tail))的执行顺序是这样的
1. t_old=t
2. t=tail
3. t_old!=t
在执行比较之前tail被赋值给t(new)

### 弱一致迭代器导致的GC问题

在LinkedBlockingQueue也有类似描述
>That would cause two problems:
      - allow a rogue Iterator to cause unbounded memory retention
      - cause cross-generational linking of old Nodes to new Nodes if a Node was tenured while live, which generational GCs have a
 hard time dealing with, causing repeated major collections.
However, only non-deleted Nodes need to be reachable from dequeued Nodes, and reachability does not necessarily have to be of the kind understood by the GC.  We use the trick of linking a Node that has just been dequeued to itself.  Such a self-link implicitly means to advance to head.

### 常见的使用错误

#### 谨慎使用addAll()方法

批量操作不保证原子性，执行addlAll方法是并发的迭代操作可能看不到全部添加到队列的元素。
Additionally, the bulk operations addAll, removeAll, retainAll, containsAll, equals, and toArray are not guaranteed to be performed atomically. For example, an iterator operating concurrently with an addAll operation might view only some of the added elements.

#### 谨慎使用size()方法
int size()方法返回的值不一定是准确的，currentlinkedqueue只能通过从头遍历来统计数量，在此期间发生的添加和删除操作不会反应到size()的返回结果中。

#### head和tail的位置
head和tail都支持延迟更新（延迟等于或超过两个节点才更新），这样可以减少cas操作，所以head/tail不保证一定指向头结点和尾节点，甚至不保证head和tail之间的相互顺序。
>Both head and tail are permitted to lag.  In fact, failing to update them every time one could is a significant optimization(fewer CASes). As with LinkedTransferQueue (see the internal documentation for that class), we use a slack threshold of two;
      that is, we update head/tail when the current pointer appears to be two or more steps away from the first/last node.
     Since head and tail are updated concurrently and independently, it is possible for tail to lag behind head (why not)?
     
     
  [1]: http://www.cnblogs.com/devos/p/4396773.html
  [2]: https://stackoverflow.com/questions/46280313/relaxed-write-what-does-it-mean