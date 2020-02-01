---
title: thrift源码（四）非阻塞服务端
date: 2019-10-25 00:12:07
categories: thrift源码
tags: [thrift,rpc框架]
---

### 非阻塞服务端TNonblockingServer
thrift的非阻塞服务端，采用还是类似Reactor的IO复用模型

![reactor uml][4]
对于TNonblockingServer，使用的是单线程的reactor模式，
客户端的请求会被公平的处理(没有优先级，看哪个请求先触发selector)，使用TNonblockingServer时必须使用TFramedTransport，不然无法确定数据包何时读取完成(数据边界)。
<!--more-->

{% asset_img reactor-single.png image %}

#### SelectThread

{% asset_img SelectThread.png image %}

selector工作线程，作用reactor模式中的dispatcher,分离IO事件，触发对应的handler。
SelectThread的run方法：
执行完select()，就会检查&更新FrameBuffers上关注的事件：
```java
public void run() {
      try {
        while (!stopped_) {
          select();
          processInterestChanges();
        }
      } catch (Throwable t) {
        LOGGER.error("run() exiting due to uncaught error", t);
      } finally {
        stopped_ = true;
      }
    }
```
select方法，根据不同的IO事件，执行对应调用：
和上面Reactor的图片有点不同，这里没有单独的acceptor，也没有handler,都由SelectThread处理。
```java
private void select() {
      try {
        // wait for io events.
        selector.select();

        // process the io events we received
        Iterator<SelectionKey> selectedKeys = selector.selectedKeys().iterator();
        while (!stopped_ && selectedKeys.hasNext()) {
          SelectionKey key = selectedKeys.next();
          selectedKeys.remove();

          // skip if not valid
          if (!key.isValid()) {
            cleanupSelectionkey(key);
            continue;
          }

          // if the key is marked Accept, then it has to be the server
          // transport.
          if (key.isAcceptable()) {
            handleAccept();
          } else if (key.isReadable()) {
            // deal with reads
            handleRead(key);
          } else if (key.isWritable()) {
            // deal with writes
            handleWrite(key);
          } else {
            LOGGER.warn("Unexpected state in select! " + key.interestOps());
          }
        }
      } catch (IOException e) {
        LOGGER.warn("Got an IOException while selecting!", e);
      }
    }
```
SelectThread中除了serverTransport和selector还有一个属性selectInterestChanges，用来保存需要改变关注事件的FrameBuffer
```java
private final TNonblockingServerTransport serverTransport;
private final Selector selector;

// List of FrameBuffers that want to change their selection interests.
private final Set<FrameBuffer> selectInterestChanges =
      new HashSet<FrameBuffer>();
```
为了管理这些FrameBuffer，SelectThread提供两个方法，**processInterestChanges()**函数遍历FrameBuffer触发他们更新关注的IO事件并注册，然后清空集合。
**requestSelectInterestChange()**函数，允许向集合添加新的需要改变关注事件的FrameBuffer.
**这俩函数不是同步执行的**，主要是为了多线程版本的服务端使用，同时对selectInterestChanges集合加锁，也是因为TNonblockingServer的子类(THsHaServer)，使用了多线程的reactor模型，在这里不会起作用。
```java

/**
    * Check to see if there are any FrameBuffers that have switched their
    * interest type from read to write or vice versa.
*/
private void processInterestChanges() {
      synchronized (selectInterestChanges) {
        for (FrameBuffer fb : selectInterestChanges) {
          fb.changeSelectInterests();
        }
        selectInterestChanges.clear();
      }
    }
    
/**
     * Add FrameBuffer to the list of select interest changes and wake up the
     * selector if it's blocked. When the select() call exits, it'll give the
     * FrameBuffer a chance to change its interests.
     */
public void requestSelectInterestChange(FrameBuffer frameBuffer) {
      synchronized (selectInterestChanges) {
        selectInterestChanges.add(frameBuffer);
      }
      // wakeup the selector, if it's currently blocked.
      selector.wakeup();
    }
```
#### FrameBuffer

**看名字你会以为这是一个数据缓冲区（类似ByteBuffer），但实际上FrameBuffer做的更多，这就是thrift的神奇命名。。**
{% asset_img FrameBuffer.png image %}

FrameBuffer内部是一个类似状态机的设计，ta负责数据的读写，修改selector关注事件，发起接口调用，返回结果等操作。服务端IO的具体细节都由它来完成。
FrameBuffer定义的状态:
```java
    // in the midst of reading the frame size off the wire
    private static final int READING_FRAME_SIZE = 1;
    // reading the actual frame data now, but not all the way done yet
    private static final int READING_FRAME = 2;
    // completely read the frame, so an invocation can now happen
    private static final int READ_FRAME_COMPLETE = 3;
    // waiting to get switched to listening for write events
    private static final int AWAITING_REGISTER_WRITE = 4;
    // started writing response data, not fully complete yet
    private static final int WRITING = 6;
    // another thread wants this framebuffer to go back to reading
    private static final int AWAITING_REGISTER_READ = 7;
    // we want our transport and selection key invalidated in the selector thread
    private static final int AWAITING_CLOSE = 8;
```
FrameBuffer状态图：
{% asset_img FrameBufferState.png image %}
状态由**READING_FRAME_SIZE**开始(此时bufferSize大小只有4字节)，如果buffer读取完成，变**成READING_FRAME**（此时buffer被初始化为frameSize大小），如果buffer又读取完成，变成
**READ_FRAME_COMPLETE**状态，这时selectThread会执行requestInvoke()方法:
```java
private void handleRead(SelectionKey key) {
...
// if the buffer's frame read is complete, invoke the method.
      if (buffer.isFrameFullyRead()) {
        if (!requestInvoke(buffer)) {
          cleanupSelectionkey(key);
        }
      }
...
}
```
然后调用FrameBuffer的invoke方法，frameBuffer会调用服务端本地的接口实现类，执行完成后调用responseReady()准备返回调用结果，如果接口实现执行失败，状态由**READ_FRAME_COMPLETE**变成**AWAITING_CLOSE**，随后socket会被关闭。
```java
public void invoke() {
      TTransport inTrans = getInputTransport();
      TProtocol inProt = inputProtocolFactory_.getProtocol(inTrans);
      TProtocol outProt = outputProtocolFactory_.getProtocol(getOutputTransport());

      try {
        outProt.setServerSide(true);
        processorFactory_.getProcessor(inTrans).process(inProt, outProt);
        responseReady();
        return;
      } catch (TException te) {
        LOGGER.warn("Exception while invoking!", te);
      } catch (Exception e) {
        LOGGER.error("Unexpected exception while invoking!", e);

      } catch (Throwable t) {
        LOGGER.error("Unexpected throwable while invoking!", t);
      }
      // This will only be reached when there is an exception.
      state_ = AWAITING_CLOSE;
      requestSelectInterestChange();
    }
```
responseReady方法中，状态可能由**READ_FRAME_COMPLETE**变成**AWAITING_REGISTER_READ**(不需要向客户端返回结果)，或者变成**AWAITING_REGISTER_WRITE**（准备发送结果到客户端）：
```java
public void responseReady() {
      // the read buffer is definitely no longer in use, so we will decrement
      // our read buffer count. we do this here as well as in close because
      // we'd like to free this read memory up as quickly as possible for other
      // clients.
      readBufferBytesAllocated.addAndGet(-buffer_.array().length);

      if (response_.len() == 0) {
        // go straight to reading again. this was probably an oneway method
        state_ = AWAITING_REGISTER_READ;
        buffer_ = null;
      } else {
        buffer_ = ByteBuffer.wrap(response_.get(), 0, response_.len());

        // set state that we're waiting to be switched to write. we do this
        // asynchronously through requestSelectInterestChange() because there is a
        // possibility that we're not in the main thread, and thus currently
        // blocked in select(). (this functionality is in place for the sake of
        // the HsHa server.)
        state_ = AWAITING_REGISTER_WRITE;
      }
      requestSelectInterestChange();
    }
```
如同注释所说，函数requestSelectInterestChange()告知selectThread_我要改变关注的IO事件然后就返回，是一种异步的方式，然后在函数requestSelectInterestChange中，会检查当前线程，如果就是selectThread_那就直接改变关注的IO事件，否则通过TNonblockingServer调用selectThread_执行：
```java
private void requestSelectInterestChange() {
      if (Thread.currentThread() == selectThread_) {
        changeSelectInterests();
      } else {
        TNonblockingServer.this.requestSelectInterestChange(this);
      }
    }
```
在changeSelectInterests方法中，状态可能由**AWAITING_REGISTER_WRITE**变成**WRITING**，或者由**AWAITING_REGISTER_READ**变成**READING_FRAME_SIZE**，如果是**AWAITING_CLOSE**那么直接关闭连接，取消关注的事件：
```java
/**
     * Give this FrameBuffer a chance to set its interest to write, once data
     * has come in.
     */
    public void changeSelectInterests() {
      if (state_ == AWAITING_REGISTER_WRITE) {
        // set the OP_WRITE interest
        selectionKey_.interestOps(SelectionKey.OP_WRITE);
        state_ = WRITING;
      } else if (state_ == AWAITING_REGISTER_READ) {
        prepareRead();
      } else if (state_ == AWAITING_CLOSE){
        close();
        selectionKey_.cancel();
      } else {
        LOGGER.error(
          "changeSelectInterest was called, but state is invalid ("
          + state_ + ")");
      }
    }
```

#### 时序图
非阻塞服务端的初始化代码：
```java
TNonblockingServerTransport serverTransport = new TNonblockingServerSocket(7911);
                HelloService.Processor processor = new HelloService.Processor(new HelloServiceImpl());
                TServer server = new TNonblockingServer(processor, serverTransport);
                server.serve();
```
时序图包含了异步服务端启动和响应请求的过程(不包括异常)：
**1,2,3,4** 
执行Server的初始化，和上面的代码一致
**5,5.1,5.2.1** 
开始准备响应客户端请求，监听端口，创建SelectThread线程并启动
**5.2.1.1 , 5.2.1.2** SelectThread在构造函数中已经打开selector并注册ACCEPT事件
**5.2.2.1和灰色部分**
selector执行select()方法，灰色部分是select()返回的IO事件集合的遍历循环。
**2.x** 处理accept事件，获得客户端的连接(TNonblockTransport) ，注册READ事件，创建FrameBuffer,准备读取客户端发送的数据。
**3.x** 处理READ事件，调用服务端的接口实现类
**4.x** 处理WRITE事件，向客户端发送调用结果
**5.2.2.2**
遍历 selectInterestChanges，注册FrameBuffer现在要关注的IO事件。
**5.3 5.4**
等待SelectThread退出，停止监听端口，服务结束。


{% asset_img serverflow.png image %}


  [1]: https://www.ibm.com/developerworks/cn/java/j-lo-apachethrift/image003.jpg
  [2]: https://www.ibm.com/developerworks/cn/java/j-lo-apachethrift/image006.png
  [3]: https://www.ibm.com/developerworks/cn/java/j-lo-apachethrift/image004.png
  [4]: https://www.2cto.com/uploadfile/2012/0816/20120816083811838.jpg
  [5]: https://upload-images.jianshu.io/upload_images/4235178-4047d3c78bb467c9.png
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
  [19]: https://jin-yang.github.io/post/network-tcpip-timewait.html