---
title: thrift源码（二）异步客户端
date: 2019-08-05 23:28:22
categories: thrift源码
tags: [thrift,rpc框架]
---
## 异步客户端简介
异步客户端使用java nio实现，和许多NIO的例子相同，启动一个线程执行select()操作，然后把获得的SocketChannel交给统一的handler处理。
异步客户端初始化并发起请求的调用栈如下：
```java
//初始化
HelloServiceClient.main()
= TAsyncClientManager.TAsyncClientManager()

= = TAsyncClientManager.SelectThread.run()
= = TAsyncClientManager.SelectThread.transitionMethods()
= = = TAsyncMethodCall.transition()
= = TAsyncClientManager.SelectThread.timeoutIdleMethods()
= = TAsyncClientManager.SelectThread.startPendingMethods()
= = = TAsyncMethodCall.start()
//发起调用
= HelloService.AsyncClient.helloVoid()
= = TAsyncClientManager.call()
= = = TAsyncMethodCall.prepareMethodCall()
= = = = HelloService.AsyncClient.helloVoid_call.write_args()
= = = TAsyncClientManager.SelectThread.getSelector().wakeup()
= = = TAsyncMethodCall.start()
= = = TAsyncMethodCall#registerForFirstWrite
= = = TAsyncMethodCall#doWritingRequestSize
= = = TAsyncMethodCall#doWritingRequestBody
= = = TAsyncMethodCall#doReadingResponseSize
= = = TAsyncMethodCall#doReadingResponseBody
```
<!--more-->

## 主要组件
### TAsyncClientManager
管理方法调用，请求的IO状态过渡,超时管理等，持有一个Selector线程
```java
  //selector线程
  private final SelectThread selectThread;
  //保存请求的队列
  private final ConcurrentLinkedQueue<TAsyncMethodCall> pendingCalls = new ConcurrentLinkedQueue<TAsyncMethodCall>();

  public TAsyncClientManager() throws IOException {
    this.selectThread = new SelectThread();
    selectThread.start();
  }

```

 ### SelectThread

 SelectThread是TAsyncClientManager的内部类，继承了Thread,使用了java提供的nio包的selector

```java
private class SelectThread extends Thread {
    // Selector waits at most SELECT_TIME milliseconds before waking
    private static final long SELECT_TIME = 5;
    
    private final Selector selector;
    // 线程共享的变量
    private volatile boolean running;
    // 用来管理超时
    private final Set<TAsyncMethodCall> timeoutWatchSet = new HashSet<TAsyncMethodCall>();
    //核心方法，调用select监听IO事件，然后依次执行处理方法
    public void run() {
      while (running) {
        try {
          try {
            selector.select(SELECT_TIME);
          } catch (IOException e) {
            LOGGER.error("Caught IOException in TAsyncClientManager!", e);
          }
          //触发TAsyncMethodCall状态机的状态变化，状态机会按照状态执行操作
          transitionMethods();
          //遍历timeoutWatchSet中的请求，让超时调用返回异常
          timeoutIdleMethods();
          //从pendingCalls队列中取出全部的新请求并处理
          startPendingMethods();
        } catch (Throwable throwable) {
          LOGGER.error("Ignoring uncaught exception in SelectThread", throwable);
        }
      }
    }
```
#### startPendingMethods() 
从pendingCalls队列中取出全部的新请求并处理：
```java
// Start any new calls
    private void startPendingMethods() {
      TAsyncMethodCall methodCall;
      while ((methodCall = pendingCalls.poll()) != null) {
        
        try {
          methodCall.start(selector);

          // If timeout specified and first transition went smoothly, add to timeout watch set
          TAsyncClient client = methodCall.getClient();
          if (client.hasTimeout() && !client.hasError()) {
            //放入超时检查set
            timeoutWatchSet.add(methodCall);
          }
        } catch (Throwable e) {
          LOGGER.warn("Caught throwable in TAsyncClientManager!", e);
          methodCall.onError(e);
        }
      }
    }
```
#### timeoutIdleMethods() 
遍历timeoutWatchSet，检查超时的调用：
```java
private final Set<TAsyncMethodCall> timeoutWatchSet = new HashSet<TAsyncMethodCall>();
// Timeout any existing method calls                                                
private void timeoutIdleMethods() {                                                     
  Iterator<TAsyncMethodCall> iterator = timeoutWatchSet.iterator();                     
  while (iterator.hasNext()) {                                                          
    TAsyncMethodCall methodCall = iterator.next();                                      
    long clientTimeout = methodCall.getClient().getTimeout();                           
    long timeElapsed = System.currentTimeMillis() - methodCall.getLastTransitionTime(); 
                                                                                        
    if (timeElapsed > clientTimeout) {                                                  
      iterator.remove();                                                                
      methodCall.onError(new TimeoutException("Operation " +                            
          methodCall.getClass() + " timed out after " + timeElapsed +                   
          " milliseconds."));                                                           
    }                                                                                   
  }                                                                                     
}                                                                                       
```



### TAsyncMethodCall
抽象类，代表远程方法的异步调用，thrift生成的helloString_call的父类，持有callback调用，负责建立socket连接，写入TNonblockingTransport、构造ByteBuffer等，**对IO操作的切换通过状态机来完成**,重点关注transition()方法。
```java
public abstract class TAsyncMethodCall<T extends TAsyncMethodCall> {

  private static final int INITIAL_MEMORY_BUFFER_SIZE = 128;
  //内部的状态机，一共7个状态
  public static enum State {
    CONNECTING,
    WRITING_REQUEST_SIZE,
    WRITING_REQUEST_BODY,
    READING_RESPONSE_SIZE,
    READING_RESPONSE_BODY,
    RESPONSE_READ,
    ERROR;
  }
  //Transport socket连接的封装
  protected final TNonblockingTransport transport;
  //负责序列化
  private final TProtocolFactory protocolFactory;
  //rpc client
  protected final TAsyncClient client;
  //回调方法
  private final AsyncMethodCallback<T> callback;
  //isOneway==true则不用回调
  private final boolean isOneway;
  //方法调用到达这个状态的时间，用来判断超时
  private long lastTransitionTime;
  //保存数据包的内容size
  private ByteBuffer sizeBuffer;
  private final byte[] sizeBufferArray = new byte[4];
  //保存数据包内容
  private ByteBuffer frameBuffer;
 
  protected TAsyncMethodCall(TAsyncClient client, TProtocolFactory protocolFactory, TNonblockingTransport transport, AsyncMethodCallback<T> callback, boolean isOneway) {
    this.transport = transport;
    this.callback = callback;
    this.protocolFactory = protocolFactory;
    this.client = client;
    this.isOneway = isOneway;
    this.lastTransitionTime = System.currentTimeMillis();
  }
  //Transition方法，根据状态机状态执行对应的操作。
  //这个方法是线程安全的，因为只在SelectThread内部执行
  protected void transition(SelectionKey key) {
    // Ensure key is valid
    if (!key.isValid()) {
      key.cancel();
      Exception e = new TTransportException("Selection key not valid!");
      onError(e);
      return;
    }

    // Transition function
    try {
      switch (state) {
        case CONNECTING:
          doConnecting(key);
          break;
        case WRITING_REQUEST_SIZE:
          doWritingRequestSize();
          break;
        case WRITING_REQUEST_BODY:
          doWritingRequestBody(key);
          break;
        case READING_RESPONSE_SIZE:
          doReadingResponseSize();
          break;
        case READING_RESPONSE_BODY:
          doReadingResponseBody(key);
          break;
        default: // RESPONSE_READ, ERROR, or bug
          throw new IllegalStateException("Method call in state " + state
              + " but selector called transition method. Seems like a bug...");
      }
      lastTransitionTime = System.currentTimeMillis();
    } catch (Throwable e) {
      key.cancel();
      key.attach(null);
      onError(e);
    }
  }
```
#### start()
在TAsyncMethodCall的start()方法中**有一个优化处理**：一般当socket连接未建立时，会向selector注册连接事件的监听。但是**因为非阻塞socket的CONNECT操作可以立刻完成(不会一直阻塞，当连接不能立即完成时，connect返回EINPROGRESS，之后select会再判断描述符是否可写)**，所以添加了对startConnect()返回值的检查，如果return true那么就去注册读事件。
```java
void start(Selector sel) throws IOException {
    SelectionKey key;
    if (transport.isOpen()) {
      state = State.WRITING_REQUEST_SIZE;
      key = transport.registerSelector(sel, SelectionKey.OP_WRITE);
    } else {
      state = State.CONNECTING;
      key = transport.registerSelector(sel, SelectionKey.OP_CONNECT);

      // non-blocking connect can complete immediately,
      // in which case we should not expect the OP_CONNECT
      if (transport.startConnect()) {
        registerForFirstWrite(key);
      }
    }

    key.attach(this);
  }
```

## 初始化的时序图
{% asset_img client-init.png image %}

如图所示，首先HelloServiceClient创建TAsyncClientManger实例，然后SelectThread对象初始化，创建守护线程并启动。
线程启动之后，执行SelectThread.run()方法的循环，selector开始监听I/O事件。
select()执行后，依次执行
transitionMethods();
timeoutIdleMethods();
startPendingMethods();
这些方法分别会调用TAsyncMethodCall的transition(),onError(),start()方法。
最右侧是TAsyncMethodCall中State状态机的状态图。当过渡到RESPONSE_READ状态后，本次调用的I/O操作完成。


## 执行异步调用的时序图
下面是异步客户端初始化并发起异步调用的代码，为了不让程序立即退出最后增加了sleep()方法。
```java
TAsyncClientManager clientManager = new TAsyncClientManager();
TNonblockingTransport transport = new TNonblockingSocket("localhost", 7911);
TProtocolFactory protocol = new TBinaryProtocol.Factory();
HelloService.AsyncClient asyncClient = new HelloService.AsyncClient(protocol, clientManager, transport);
asyncClient.setTimeout(1500L);
System.out.println("client async calls");
HelloStringCallback callback = new HelloStringCallback();
asyncClient.helloString("baba", callback);
//等待触发回调
Thread.sleep(10000L);
```
处理异步调用结果的回调类HelloStringCallback代码如下：
```java
public class HelloStringCallback implements AsyncMethodCallback<helloString_call> {

    private String response = null;

    HelloStringCallback() {
        super();
        System.out.println("init Thread ID=" + Thread.currentThread().getId());
    }

    @Override
    public void onComplete(helloString_call response) {
        try {
            System.out.println("onComplete Thread ID=" + Thread.currentThread().getId());
            //调用helloString_call的getResult方法
            System.out.println("msg:" + response.getResult());
        } catch (TException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void onError(Throwable throwable) {
        System.out.println("onError:");
        throwable.printStackTrace();
    }
}

```
下面的时序图展示了客户端调用异步方法时，thrift内部组件的工作流程。


**步骤1～3** 首先使用thrift为HelloService生成的AsyncClient对象，**checkReady()** 检查当前是否正在执行HelloService的方法，**一个时刻同一个Service只能执行一个方法**，然后创建thrift生成的helloString_call对象(TAsyncMethodCall的子类)，最后调用TAsyncClientManager.call(TAsyncMethodCall method)。

**步骤3.1~3.1.4** TAsyncClientManager创建一个临时TProtocol对象，调用helloString_call的write_args方法，得到这次调用的方法名，参数等信息的序列化内容填充到frameBuffer中，然后根据序列化内容的长度初始化对应大小的sizeBuffer。

**步骤3.2~3.3.2** SelectThread重新执行select(),并调用TAsyncMethodCall.start()方法，将这次调用的socket连接注册到selector上，图中灰色框是SelectThread线程的内部循环，橙色框是对套接字transport是否建立连接的判断(**实际上SelectThread和TAsyncClientManager不在一个线程内工作**，这里只是描述大概的执行顺序，SelectThread的内部细节在上面的初始化时序图中)

**步骤3.4~3.4.6.2** SelectThread调用TAsyncMethodCall.transition()方法，TAsyncMethodCall根据状态机的状态执行操作，这里展示一次调用的正常顺序(没有包含异常的情况)。

3.4.1~3.4.5是建立连接，发送请求包size,请求包内容，接收响应包size，响应包内容 **(注意除了size相关的方法，都需要SelectionKey做参数，因为这次操作需要修改selector上注册的事件)**。

3.4.6 当响应接收完成，执行cleanUpAndFireCallback()方法，先是调用HelloService.AsyncClient的onComplete()方法，让它能去处理下一个HelloService的异步调用，然后调用我们定义的回调函数HelloStringCallback.onComplete(helloString_call response)方法(这也是在SelectThread内完成的)。

{% asset_img thrift-client-call.png image %}

**在调用中selector上注册事件的变化**

向selector注册事件的顺序全部由TAsyncMethodCall来完成，左侧是注册的事件，右侧会注册这个事件的函数：
当通信完成注册0清空事件
|事件| 状态机 | 函数 |
| --- |---|--- |
|OP_CONNECT |State.CONNECTING  |  start() |
|OP_WRITE|State.WRITING_REQUEST_SIZE或State.CONNECTING|start()或doConnecting()|
|OP_READ|State.READING_RESPONSE_SIZE|doWritingRequestBody()|
|0|State.RESPONSE_READ|cleanUpAndFireCallback()|
#### 对比同步客户端
**同步客户端的使用方式**

对于同一个Service,同一时间只能执行一个Service中的方法。需要开发者确保Client不会被多个线程调用，因为同步客户端的**Client对象不是线程安全的。一般都会创建Client的对象池，每次调用从对象池中获得一个Client**。
每次发起调用seqid都会+1，然后在调用完成后检查收到的seqid和发起时一致。

**异步客户端内的使用方式**

**必须使用非阻塞服务端**,在我的测试中，使用同步的服务端&异步客户端会遇到readMessageBegin()操作阻塞的问题。
在执行`int size = readI32();`时，同步服务端获得size是**整个请求包的size**,并不是TMessage第一个构造参数name的size,所以readStringBody会读取整个请求包，之后的readByte()和readI32()已经没有数据可读，就会阻塞（**到底为什么呢**）。
```java
public TMessage readMessageBegin() throws TException {
        int size = readI32();
        if (size < 0) {
            int version = size & VERSION_MASK;
            if (version != VERSION_1) {
                throw new TProtocolException(TProtocolException.BAD_VERSION, "Bad version in readMessageBegin");
            }
            return new TMessage(readString(), (byte) (size & 0x000000ff), readI32());
        } else {
        //使用同步服务端时，执行这部分阻塞
            if (strictRead_) {
                throw new TProtocolException(TProtocolException.BAD_VERSION, "Missing version in readMessageBegin, old client?");
            }
            return new TMessage(readStringBody(size), readByte(), readI32());
        }
    }
```

AsyncClient也不是线程安全的，甚至**即使同一个线程在一个循环中多次调用一个方法**也会出现异常，比如：
```
                for(int i=3;i>0;i--) {
                    HelloStringCallback callback = new HelloStringCallback();
                    asyncClient.helloString("baba", callback);
                }
                //等待触发回调
                Thread.sleep(1000L);
```
会触发异常：
java.nio.channels.ConnectionPendingException//连接正在进行
java.nio.channels.ClosedChannelException//连接已关闭

在上面的例子中，因为没有等上次调用返回就发起了新的调用，每次调用使用的socket连接相同，因此对socket连接的操作产生了冲突，实际上**checkReady()并没有起到作用**，在这个版本的源码中TAsyncClient.**currentMethod字段一直是null**。所以使用异步客户端时**发起一次调用就需要创建一个AsyncClient**，使用新的socket，一般是使用Client的对象池。
没卵用的checkReady():
```java
protected void checkReady() {
    // Ensure we are not currently executing a method
    if (currentMethod != null) {
      throw new IllegalStateException("Client is currently executing another method: " + currentMethod.getClass().getName());
    }

    // Ensure we're not in an error state
    if (error != null) {
      throw new IllegalStateException("Client has an error!", error);
    }
```
**seqid在异步调用中没有变化**，一直是0：
```java
        prot.writeMessageBegin(new TMessage("helloString", TMessageType.CALL, 0));
```
在同步客户端中，seqid会自增：
```java
        oprot_.writeMessageBegin(new TMessage("helloString", TMessageType.CALL, ++seqid_));
```

**在接收调用结果时和同步客户端一样:**
在helloString_call(extends TAsyncMethodCall)中可以看到getResult方法使用了同步客户端(创新了一个同步客户端实例，new Client(prot))的recv_helloString方法。
```java
public String getResult() throws TException {
                if (getState() != State.RESPONSE_READ) {
                    throw new IllegalStateException("Method call not finished!");
                }
                TMemoryInputTransport memoryTransport = new TMemoryInputTransport(getFrameBuffer().array());
                TProtocol prot = client.getProtocolFactory().getProtocol(memoryTransport);
                return (new Client(prot)).recv_helloString();
            }
```
