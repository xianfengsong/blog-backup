---
title: thrift源码（五）非阻塞服务端其他实现
date: 2019-11-18 23:35:19
categories: thrift源码
tags: [thrift,rpc框架]
---

### THsHaServer

HsHa=HalfSync HalfAsync 半同步，半异步
在处理IO事件时是同步的，在执行invoke方法时是在线程池异步执行的。
本质上是一个添加了工作线程池的Reactor模型：
{% asset_img reactor-pool.png image %}

内部使用了一个自定义的线程池ExecutorService，用线程池中的线程执行FrameBuffer的Invoke方法，非常简单。上面已经说明invoke()方法只执行了服务端本地的接口实现类，并没有做IO操作，所以在THsHaServer中IO操作还是在SelectThread线程内完成的。
<!--more-->

```java
  //线程池 invoker
  private ExecutorService invoker;

/**
   * 重写了requestInvoke方法
   */
  @Override
  protected boolean requestInvoke(FrameBuffer frameBuffer) {
    try {
      invoker.execute(new Invocation(frameBuffer));
      return true;
    } catch (RejectedExecutionException rx) {
      LOGGER.warn("ExecutorService rejected execution!", rx);
      return false;
    }
  }

  /**
   * final修饰的FrameBuffer
   */
  private class Invocation implements Runnable {

    private final FrameBuffer frameBuffer;

    public Invocation(final FrameBuffer frameBuffer) {
      this.frameBuffer = frameBuffer;
    }

    public void run() {
      frameBuffer.invoke();
    }
  }
```

下面是THsHaServer对线程池的创建和销毁代码：
Options的默认配置是5个线程，60s空闲时间。

```java
 //创建线程池
 protected static ExecutorService createInvokerPool(Options options) {
    int workerThreads = options.workerThreads;
    int stopTimeoutVal = options.stopTimeoutVal;
    TimeUnit stopTimeoutUnit = options.stopTimeoutUnit;

    LinkedBlockingQueue<Runnable> queue = new LinkedBlockingQueue<Runnable>();
    ExecutorService invoker = new ThreadPoolExecutor(workerThreads, workerThreads,
      stopTimeoutVal, stopTimeoutUnit, queue);

    return invoker;
  }
 //等待线程池退出,只要线程池没彻底关闭,即使被中断也要等待足够的时间再退出
 protected void gracefullyShutdownInvokerPool() {
    // try to gracefully shut down the executor service
    invoker.shutdown();

    // Loop until awaitTermination finally does return without a interrupted
    // exception. If we don't do this, then we'll shut down prematurely. We want
    // to let the executorService clear it's task queue, closing client sockets
    // appropriately.
    long timeoutMS = 10000;
    long now = System.currentTimeMillis();
    while (timeoutMS >= 0) {
      try {
        invoker.awaitTermination(timeoutMS, TimeUnit.MILLISECONDS);
        break;
      } catch (InterruptedException ix) {
        long newnow = System.currentTimeMillis();
        timeoutMS -= (newnow - now);
        now = newnow;
      }
    }
  }
```

### TThreadSelectorServer

我使用的0.5.x版本没有这个server实现，在0.8.x版本thrift添加了TThreadedSelectorServer

据thrift描述：
在多核环境中，如果瓶颈是单线程的selector获得的CPU计算能力不足, 那么它的性能要优于TNonblockingServer/THsHaServer。

而且because the accept handling is decoupled from
  reads/writes and invocation, the server has better ability to handle back-pressure from new connections
  (backpress背压，是一个很有意思的名词，在很多地方有用到 [比如RxJava][7] 或者[背压阀][8])

[TThreadedSelectorServer源码地址][9]

TThreadSelectorServer是**Multiple Reactors**模式的实现，
mainReactor只负责完成accept操作，子reactor处理读、写事件。图中只画出来一个subReactor线程的情况，实际可能配置多个线程运行subReactor。

{% asset_img reactor-pool.png image %}

在实现**Multiple Reactors**时，TThreadSelectorServer的AcceptThread相当于MainReactor,SelectorThread相当于subReactor,下面只选择部分代码做介绍：

#### TThreadedSelectorServer.java

```java
//startThreads()方法,初始化AcceptThread和SelectorThread(集合)：
for (int i = 0; i < args.selectorThreads; ++i) {
        selectorThreads.add(new SelectorThread(args.acceptQueueSizePerThread));
      }
      acceptThread = new AcceptThread((TNonblockingServerTransport) serverTransport_,
        createSelectorThreadLoadBalancer(selectorThreads));
      stopped_ = false;
      for (SelectorThread thread : selectorThreads) {
        thread.start();
      }
      acceptThread.start();
return true;
```

#### AcceptThread.java

**run()**方法里只执行select()操作：

```java
public void run() {
      try {
        while (!stopped_) {
          select();
        }
      } catch (Throwable t) {
        LOGGER.error("run() exiting due to uncaught error", t);
      } finally {
        // This will wake up the selector threads
        TThreadedSelectorServer.this.stop();
      }
}
```

在**handleAccept()**方法中，处理客户端的建立连接的请求：
AcceptThread有两种工作模式，FAST_ACCEPT是收到连接请求就接受，FAIR_ACCEPT是把连接请求丢到工作线程池invoker中，这样会等之前的连接请求被工作线程执行后，才会处理后来的连接请求：

```java
private void handleAccept() {
      final TNonblockingTransport client = doAccept();
      if (client != null) {
        // Pass this connection to a selector thread
        final SelectorThread targetThread = threadChooser.nextThread();

        if (args.acceptPolicy == Args.AcceptPolicy.FAST_ACCEPT || invoker == null) {
          doAddAccept(targetThread, client);
        } else {
          // FAIR_ACCEPT
          try {
            invoker.submit(new Runnable() {
              public void run() {
                doAddAccept(targetThread, client);
              }
            });
          } catch (RejectedExecutionException rx) {
            ...
          }
        }
      }
}
```

**doAddAccept()**方法，把client连接交给SelectorThread处理，后续的其他操作就和AcceptThread无关了：

```java
private void doAddAccept(SelectorThread thread, TNonblockingTransport client) {
      if (!thread.addAcceptedConnection(client)) {
        client.close();
      }
}
```

#### SelectorThread.java

**addAcceptedConnection()**方法

AcceptThread调用它把任务交给SelectorThread处理，客户端连接会被放入SelectorThread的acceptedQueue等待处理：

```java
public boolean addAcceptedConnection(TNonblockingTransport accepted) {
      try {
        acceptedQueue.put(accepted);
      } catch (InterruptedException e) {
        LOGGER.warn("Interrupted while adding accepted connection!", e);
        return false;
      }
      selector.wakeup();
      return true;
}
```

在SelectorThread的**run()**方法

完成对读/写IO事件的select(),处理acceptQueue中的任务，执行processInterestChanges（之前介绍过，触发FrameBuffer的关注事件更新）

```java
while (!stopped_) {
          select();
          processAcceptedConnections();
          processInterestChanges();
}
```

**processAcceptedConnections()**方法

从acceptQueue队列取出客户端连接，向selector注册OP_READ事件

```java
private void processAcceptedConnections() {
      // Register accepted connections
      while (!stopped_) {
        TNonblockingTransport accepted = acceptedQueue.poll();
        if (accepted == null) {
          break;
        }
        registerAccepted(accepted);
      }
}
private void registerAccepted(TNonblockingTransport accepted) {
      SelectionKey clientKey = null;
      try {
        clientKey = accepted.registerSelector(selector, SelectionKey.OP_READ);

        FrameBuffer frameBuffer = new FrameBuffer(accepted, clientKey, SelectorThread.this);
        clientKey.attach(frameBuffer);
      } catch (IOException e) {
        ...
      }
}
```

[1]: https://www.ibm.com/developerworks/cn/java/j-lo-apachethrift/image003.jpg
[2]: https://www.ibm.com/developerworks/cn/java/j-lo-apachethrift/image006.png
  [3]: https://www.ibm.com/developerworks/cn/java/j-lo-apachethrift/image004.png
  [4]: http://upload-images.jianshu.io/upload_images/1452123-35a5505c0d9928f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
  [5]: http://upload-images.jianshu.io/upload_images/1452123-bfb7ef28b21ba29e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
  [6]: https://upload-images.jianshu.io/upload_images/3169646-6eddb6e230677349.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/692
  [7]: http://zhangtielei.com/posts/blog-rxjava-backpressure.html
  [8]: https://baike.baidu.com/item/%E8%83%8C%E5%8E%8B%E9%98%80
  [9]: https://github.com/apache/thrift/blob/0.8.x/lib/java/src/org/apache/thrift/server/TThreadedSelectorServer.java
  [10]: https://upload-images.jianshu.io/upload_images/3169646-eedd2295dcc12725.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/688
  [11]: https://static.oschina.net/uploads/space/2016/0224/192255_iHyl_1469576.png
  [12]: https://static.oschina.net/uploads/space/2016/0224/192347_Vfen_1469576.png
  [13]: https://static.oschina.net/uploads/space/2016/0224/192513_tzOg_1469576.png
  [14]: https://static.oschina.net/uploads/space/2016/0224/192544_xSFh_1469576.png
  [15]: https://static.oschina.net/uploads/space/2016/0224/192757_XK9n_1469576.png
  [16]: https://static.oschina.net/uploads/space/2016/0224/192757_toBx_1469576.png
  [17]: https://upload.wikimedia.org/wikipedia/commons/thumb/5/54/Big-Endian.svg/200px-Big-Endian.svg.png
  [18]: https://upload.wikimedia.org/wikipedia/commons/thumb/e/ed/Little-Endian.svg/200px-Little-Endian.svg.png
  [19]: https://en.wikipedia.org/wiki/Variable-length_quantity
  [20]: https://upload.wikimedia.org/wikipedia/commons/thumb/c/c6/Uintvar_coding.svg/1920px-Uintvar_coding.svg.png
  [21]: https://www.cnblogs.com/en-heng/p/5570609.html
  [22]: http://jnb.ociweb.com/jnb/jnbJun2009_files/jnbJun2009_size_comparison.png
  [23]: https://jin-yang.github.io/post/network-tcpip-timewait.html
