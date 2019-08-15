title: thrift源码（一）源码结构和客户端介绍
date: 2019-05-22 23:55:19
categories: thrift源码
tags: [thrift,rpc框架]

---

## thrift架构图


![整体视图][1]

如图所示，作为一个通信框架，thrift架构上也分了不同的层，和tcp/ip的分层逻辑类似，最上层是我们的业务逻辑层，这里是我们编写的HelloService接口的实现类。

<!--more-->

第二层是thrift生成的HelloService.Client(或者AsyncClient)和HelloService.Processor类，分别给客户端和服务端使用，对开发者屏蔽了如何发送/处理请求，返回结果，异常检查等通信细节。

第三层是thrift生成的数据序列化和反序列化的write/read方法，read()负责根据协议从字节流中解析调用的函数名，参数等，比如方法HelloService.helloString_args#read,write()则相反。

第四层及以下是thrift框架的源码，TProtocol定义了数据传输协议，thrift提供了binary/json等多种编码实现。
第五层是thrift对i/o层的封装，TTransport包含网络连接的打开关闭，输入输出流的处理等等。
## 类图

{% asset_img overview.png image %}

**我使用的0.5.0版本的thrift**

深色的是thrift生成的HelloService和他的内部类，Client和AsyncClient是客户端使用，Processor是服务端使用。

Processor实现TProcessor接口，Client实现TServiceClient接口

TTransport被TProcessFactory，TProtocolFactory,TProtocol依赖

TProtocal由TTransport组成，几乎被图中所有类依赖。


## 客户端流程

![客户端时序图][2]

### 同步客户端
发起调用的代码：
```java
TTransport transport=new TSocket("localhost",7911);
                transport.open();
                TProtocol protocol=new TBinaryProtocol(transport);
                HelloService.Client client=new HelloService.Client(protocol);
                client.helloVoid();
                transport.close();
```
#### 创建socket连接(TTransport)，选择协议(TProtocol)

TSocket是TIOStreamTransport的子类，TIOStreamTransport封装了输入、输出流的处理，TScocket封装了一个Socket对象。
TBinaryProtocol是TProtocol的子类，定义了基于二进制的通信协议

#### 初始化Client,调用helloVoid函数

helloVoid内部执行发送请求和接收响应两步操作：
```java
 public void helloVoid() throws TException
    {
      send_helloVoid();
      recv_helloVoid();
    }
```
#### 序列化请求并发送

客户端发送请求时必然要告知服务端本次请求的方法名，参数，还有调用序号，版本号等信息，这些都要通过TProtocol序列化，协议相关细节后面介绍。
序列化完成后，TBinaryProtocol把这些内容通过TTransport写入到输出流，通过网络发送到服务端。
```java
        protected TProtocol iprot_;
        protected TProtocol oprot_;
        protected int seqid_;

public void send_helloVoid() throws TException
    {
      //构造协议消息体
      oprot_.writeMessageBegin(new TMessage("helloVoid", TMessageType.CALL, ++seqid_));
      helloVoid_args args = new helloVoid_args();
      //把参数按协议序列化
      args.write(oprot_);
      oprot_.writeMessageEnd();
      //发送数据
      oprot_.getTransport().flush();
    }
```
helloVoid_args虽然这样命名但他是一个静态内部类，HelloSerive的每个函数都会生成这样的类。生成的代码，根据thrift文件中定义的函数参数，告诉thrift程序如何进行序列化操作(比如从TProtocol中读取TField，字节流的哪个位置起对应哪个参数，参数对应多少字节等)。
helloVoid_args的函数write()代码如下：
```java
public static class helloVoid_args implements TBase<helloVoid_args, helloVoid_args._Fields>, java.io.Serializable, Cloneable   {
    public void write(TProtocol oprot) throws TException {
      //对声明为required的字段检查，required字段不能为null
      validate();

      oprot.writeStructBegin(STRUCT_DESC);
      oprot.writeFieldStop();
      oprot.writeStructEnd();
    }
    }
```
#### 接收响应并反序列化结果

当服务端返回数据后，`recv_helloVoid`函数处理服务端返回的数据，反序列化得到`helloVoid_result`，并对异常做处理。注意对seqid的检查，thrift在这里处理服务端响应乱序的情况(后接收的先返回)，处理方式是直接抛出异常。
```java
public void recv_helloVoid() throws TException
    {
      TMessage msg = iprot_.readMessageBegin();
      if (msg.type == TMessageType.EXCEPTION) {
        TApplicationException x = TApplicationException.read(iprot_);
        iprot_.readMessageEnd();
        throw x;
      }
      if (msg.seqid != seqid_) {
        throw new TApplicationException(TApplicationException.BAD_SEQUENCE_ID, "helloVoid failed: out of sequence response");
      }
      helloVoid_result result = new helloVoid_result();
      result.read(iprot_);
      iprot_.readMessageEnd();
      return;
    }
```
`helloVoid_result`和`helloVoid_args`类似，他定义了方法返回值的序列化操作方式,read方法代码如下：
```java
  public static class helloVoid_result implements TBase<helloVoid_result, helloVoid_result._Fields>, java.io.Serializable, Cloneable   {

 public void read(TProtocol iprot) throws TException {
      TField field;
      //找到字节流中类起始位置
      iprot.readStructBegin();
      //循环解析TField，直到遇到TType.STOP
      while (true)
      {
        //找到field起点
        field = iprot.readFieldBegin();
        if (field.type == TType.STOP) { 
          break;
        }
        //根据id找到field具体对应的那个属性，因为helloVoid返回值为空，直接跳过
        switch (field.id) {
          default:
            TProtocolUtil.skip(iprot, field.type);
        }
        iprot.readFieldEnd();
      }
      iprot.readStructEnd();
      //检查require字段
      // check for required fields of primitive type, which can't be checked in the validate method
      validate();
    }
    }
```

## 参考
http://www.cnblogs.com/luxiaoxun/archive/2015/03/11/4331110.html
https://my.oschina.net/xinxingegeya/blog/620576
https://www.ibm.com/developerworks/cn/java/j-lo-apachethrift/index.html


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