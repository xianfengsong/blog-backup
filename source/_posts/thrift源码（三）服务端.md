---
title: thrift源码（三）服务端
date: 2019-10-24 23:55:42
categories: thrift源码
tags: [thrift,rpc框架]
---
### 简单的服务端实现 TThreadPoolServer
#### TProcessor
和TServiceClient接口类似，TProcessor是thrift为服务端生成代码时要实现的接口，定义了服务端根据客户端请求调用本地接口实现的过程。

HelloService.Processor代码如下：
<!--more-->

**iface_**是服务启动时，传入的HelloServiceImpl对象实例
**processMap_**用来保存HelloService定义的方法。
**process()**函数会根据方法名，找到对应的ProcessFunction实现类，调用ProcessFunction.process()方法。如果没有则返回异常。
至于方法参数和结果，以helloString()方法为例，helloString_result和helloString_args和thrift客户端使用的类是同一个。
```java
        protected final HashMap<String, ProcessFunction> processMap_ = new HashMap<String, ProcessFunction>();
        private Iface iface_;

        public Processor(Iface iface) {
            iface_ = iface;
            processMap_.put("helloString", new helloString());
            processMap_.put("helloInt", new helloInt());
            processMap_.put("helloBoolean", new helloBoolean());
            processMap_.put("helloVoid", new helloVoid());
            processMap_.put("helloNull", new helloNull());
        }
        public boolean process(TProtocol iprot, TProtocol oprot) throws TException {
            TMessage msg = iprot.readMessageBegin();
            ProcessFunction fn = processMap_.get(msg.name);
            if (fn == null) {
                TProtocolUtil.skip(iprot, TType.STRUCT);
                iprot.readMessageEnd();
                TApplicationException x = new TApplicationException(TApplicationException.UNKNOWN_METHOD,
                        "Invalid method name: '" + msg.name + "'");
                oprot.writeMessageBegin(new TMessage(msg.name, TMessageType.EXCEPTION, msg.seqid));
                x.write(oprot);
                oprot.writeMessageEnd();
                oprot.getTransport().flush();
                return true;
            }
            fn.process(msg.seqid, iprot, oprot);
            return true;
        }

```

#### TThreadPoolServer调用时序图
启动服务端的代码代码如下：
```java
                // 设置服务端口为 7911
                TServerSocket serverTransport = new TServerSocket(7911);
                // 设置协议工厂为 TBinaryProtocol.Factory
                TBinaryProtocol.Factory proFactory = new TBinaryProtocol.Factory();
                // 关联处理器与 Hello 服务的实现
                TProcessor processor = new HelloService.Processor(new HelloServiceImpl());
                TServer server = new TThreadPoolServer(processor, serverTransport,
                        proFactory);
                System.out.println("Start server on port 7911...");
                server.serve();
```
TThreadPoolServer工作的时序图很简单，和所有服务端套接字程序一样，执行listen(),accept(),read(),write()的流程，使用的I/O模型是每处理一个客户端请求就从线程池中取出一个线程。

![时序图1][3]



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
  [19]: https://jin-yang.github.io/post/network-tcpip-timewait.html