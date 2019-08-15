title: restlet 示例
date: 2019-05-05 00:34:58
categories: 
tags:

---

### 背景介绍
#### [restlet][1]
是一个REST设计规范下的框架，让我们简单快速的开发RESTful应用。同类型的框架还有[Jersey][2]等，restlet在同类框架中是诞生比较早的一个（2005年），本文使用的是2.3.10版本是2017年发布的。
<!--more-->

#### [REST][3]
全称 Representational State Transfer（表现层状态转换 ），由Roy Thomas Fielding博士于2000年在他的博士论文中提出，是一种软件架构风格，它基于HTTP定义了一组约束和属性来表述一个WebService。同类的WebService规范还有SOAP等。
主要特征：
1.以资源为基础，每个资源都由URI定义
2.对资源的操作包括获取、创建、修改和删除资源，这些操作正好对应HTTP协议提供的GET、POST、PUT和DELETE方法。
3.Client-Server客户服务器分离模式，任何一个客户端与服务器都是可替换的
4.Stateless 服务端不会为客户端的请求保存context,好处是“通讯本身的无状态性可以让不同的服务器的处理一系列请求中的不同请求，提高服务器的扩展性”
5.Cacheability Responses被设计为可缓存的，并负责缓存的一致性

### 整体视图
1. restlet框架结构 
框架分为api和engine两部分，都被设计为可拓展的，API暴露给应用程序实现交互。
![framework][4]
2. 一个基于restlet的服务架构图
每个大方块代表一个组件Component(代表一类服务、应用的集合)，每个Component既可以是客户端也可以是服务端；
每个小方块是一个connector,方块之间的连线表示一个client-server的连接；restlet不仅支持http协议，也支持SMTP,FTP等类似协议。
![app][5]
3.一个简化的restfule服务结构图
除了上面那种比较标准的架构，restlet还支持使用一个component代理多个应用（用单一jvm运行）,使用virtual host组件同时表示多个主机，并且同时启动多个connector
![component][6]

### 核心概念和类图
#### 概念解释对比mvc
**controller ==** Restlet(包含Filters, Routers, Finders等)
**model ==** Resource 和 Domain Objects（DO可以在resource初始化时加载）
**view ==** Representation可以是json文档，或velocity模板等等

**Uniform** 
一个通用接口，只有一个方法`    void handle(Request request, Response response);`
>以下几个类实例化后都会被并发访问，因此属性大多是final、violate修饰的

**Restlet**
抽象类，除了实现Uniform.handle()方法，还添加了对自身生命周期的管理和对context的维护。restlet实例不是线程安全的。
{% asset_img restlet.png image %}如图，万物都是restlet
**Connector**
连接器，提供component之间的通信接口，对上层隐藏了资源和通信机制的具体实现。restlet框架提供了jetty,nio,servlet,jdbc,javaMail,solr等多种connector。
**Client&Server**
分别对应服务端和客户端的connector，他们内部使用RestletEngine提供的Connection Helper处理请求，当然Server多了地址和端口属性(还有个next属性，啥用？)。
**Router**
Router是一个Restlet，它有助于将URI与Restlet或Resource相关联，以便处理对此URI发出的所有请求;
请注意，Router附加了Restlet子类的实例，而Resource通过它们的类来附加。 原因是Resources的设计是为了处理单个请求，因此实例是通过它们的类在运行时生成的。
在匹配URI地址时，有Template.MODE_EQUALS  和 Template.MODE_STARTS_WITH匹配策略；在选择处理URI的对象时，有几种算法：
>Best match
First match (default)
Last match
Random match
Round robin
Custom

**Application**
管理一套连贯的Resource和Service，可以作为应用的根restlet也可以作为某一个restlet绑定到一个VirtualHost上。
application使用的service是框架来初始化的，我们可以主动禁用某个服务或者设置启用自定义的Service子类。
可以重写createInboundRoot()/createOutboundRoot()方法处理所有请求。
**Service**
Service是restlet框架提供的给Component和Application使用多种服务（比如connectorService可以控制connector的传输协议），他的生命周期和关联的Application或component紧紧绑定,service内部依赖context
{% asset_img service.png image %}


----------


**Message**
component之间传输的内容，负责维护一些http header之类的消息属性、还有对缓存内容的控制（flush/commit等）。
{% asset_img message.png image %}

----------
**Resource**
相当于MVC模型中的model,最为统一资源封装了与特定目标resource相对应的Context，Request和Response。


如图所示，ClientResource和ServerResource分别实现handle()方法（*和restlet不重复？*），Resource与Context,Request,Response是组合关系。
{% asset_img resource.png image %}


- 生命周期:
`init() - handle() - release() ` *（由子类具体定义）*
- 提供的在生命周期中插入自定义行为的函数：
`doInit() doRelease() doCatch()`
- 线程安全：
和restlet不同，resource不会被并发访问，每次调用被handle创建一个resource实例，并且只由一个线程访问*（一个请求一个单线程？）*
- resource使用的方法注解：
>@Get
@Put     类似replace操作
@Delete 
@Post
@Patch  类似update操作，不是幂等的
@Options 测试服务器性能，获得服务端支持的Method等信息
@Status 用于异常处理方法

```
//注解的value定义了返回结果的格式，或者请求的参数，以@Post为例：

   @Post
   public MyOutputBean accept(MyInputBean input);
   
   @Post("json")
   public String acceptJson(String value);
   
   @Post("xml|json:xml|json")
   public Representation accept(Representation xmlValue);
   
   @Post("json?param=val")
   public Representation acceptWithParam(String value);
   
   @Post("json?param")
   public Representation acceptWithParam(String value);
   
   @Post("?param")
   public Representation acceptWithParam(String value);
```



### 代码示例



  [1]: https://restlet.com/open-source/
  [2]: https://jersey.github.io/
  [3]: https://zh.wikipedia.org/wiki/%E8%A1%A8%E7%8E%B0%E5%B1%82%E7%8A%B6%E6%80%81%E8%BD%AC%E6%8D%A2
  [4]: https://restlet.com/static/tech-doc/restlet-framework/tutorial/2.3/images/tutorial01.png
  [5]: https://restlet.com/static/tech-doc/restlet-framework/tutorial/2.3/images/tutorial04.png
  [6]: https://restlet.com/static/tech-doc/restlet-framework/tutorial/2.3/images/tutorial05.png
  [8]: http://7xl4v5.com1.z0.glb.clouddn.com/restlet/service.png
  [9]: http://7xl4v5.com1.z0.glb.clouddn.com/restlet/message.png
  [10]: http://7xl4v5.com1.z0.glb.clouddn.com/restlet/resource.png
