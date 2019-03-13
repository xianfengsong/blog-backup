title: mongodb:使用spring-data处理嵌套文档的实践
date: 2018-06-12 23:10:50
categories: mongodb
tags: [mongodb,springboot]
---
### 场景介绍
在使用mongoDB保存数据时，不需要再严格遵守数据库范式，比如在设计用户订单存储时，我们可以把用户和订单的信息保存在同一个文档当中，用嵌套的方式代替用户表引用订单表的方式。本文以一个用户-订单的实例来介绍使用mongoDB嵌套保存数据时如何结合spring mongo data 实现基本功能，并对不同方案做了性能对比。

在用户-订单的例子中，订单作为数嵌套保存在用户信息中，对外提供了更新和分页查询订单的功能，查询返回的订单以时间倒序，数据格式如下：
<!--more-->
```
{
	"_id" : ObjectId("5c209db62a14a9019787860b"),
	"userId" : "u1",
	"userType" : "TYPE",
	"name" : "user1545641398911",
	"orderList" : [
		{
			"_id" : "order1",
			"info" : "Info:order01545641398901",
			"createTime" : NumberLong("1545641398901")
		},
		{
			"_id" : "order0",
			"info" : "Info:order11545641398900",
			"createTime" : NumberLong("1545641398900")
		}
    ]
}
```
### 更新用户订单接口
**接口描述：**首先根据userId查找到用户，如果订单_id已存在则替换，否则插入新订单,如果用户不存在则插入新文档；

我们习惯直接用新数据替换整个文档的内容，不过当用户订单很多时就需要订单数组做局部更新（减少发送数据的大小，
可以用pull/push操作符来完成，假设要添加id为1,2,3的订单，mongo命令如下：
```
//删除_id相同的订单
db.user.update({"userId":"user9"},
            {$pull:
                {orderList:{
                    _id:{$in:["1","2","3"]}
                }}
            })
//保存新数据
db.user.update(
            {"userId":"user9"},
            {
                $push:{orderList:{$each:[{"_id":"1"},{"_id":"2"},{"_id":"3"}]}},
                "$set":{
                    "userId":"user9",
                    "userType":"C",
                    "userName":"name"}
            },
            {"upsert":true}
        )
```

**注意问题**
mongoDB不允许在更新时对某个属性执行多个操作，所以对orderList的pull和push操作只能分开执行

### 订单查询和分页接口
**接口描述：**查询指定用户，某个时间前的订单列表，支持分页，以时间倒序，返回结果包含订单和用户信息

实现方式有两种，一个是用聚合查询，支持灵活的查询方式，不过耗时长，或者用slice对订单数组分页，不过不能按时间过滤订单且订单要先倒序保存好，耗时短；
用聚合查询，mongo命令：
```
 db.user.aggregate([
       {"$match" : { "userId" : "u1", "userType" : "TYPE" }},
       //把orderList打散为独立文档，方便后续查询，排序
       { "$unwind" : "$orderList" },
       //按时间过滤
       { "$match" : { "orderList.createTime" : { "$lte" :  1545030349990}}},
       //时间倒序
       { "$sort" : { "orderList.createTime" : -1 } },
       //分页大小
       {"$limit" : 100},
       //用group还原，如果只返回订单列表则不用这步
       //$last操作符选择返回的用户属性，$push操作符恢复打散的orderList
       { "$group" : {
       "_id" : "$_id",
       "userType" : { "$last" : "$userType" },
       "userId" : { "$last" : "$userId" },
       "name" : { "$last" : "$name" },
       "orderList" : { "$push" : "$orderList" }
       }}
       ])
```
用slice命令：
```
//跳过20条，返回10条
db.user.find({userId:"u1"},{orderList:{$slice:[20,10]}}).pretty()
```
**注意问题**
聚合查询最后的为什么group：
这是执行unwind后的用户文档,原本一个文档安装orderList数组的数量被分成多个文档，用户的信息也被分散，所以最后需要group重新组合：
```
{ "_id" : ObjectId("5c209db62a14a9019787860b"), "userId" : "u1", "userType" : "TYPE", "name" : "user1545641398911", "orderList" : { "_id" : "order18", "info" : "Info:order181545641398918", "createTime" : NumberLong("1545641398918") }, "_class" : "com.throwsnew.springbootstudy.accessdata.mongo.model.User" }
{ "_id" : ObjectId("5c209db62a14a9019787860b"), "userId" : "u1", "userType" : "TYPE", "name" : "user1545641398911", "orderList" : { "_id" : "order19", "info" : "Info:order191545641398919", "createTime" : NumberLong("1545641398919") }, "_class" : "com.throwsnew.springbootstudy.accessdata.mongo.model.User" }
```
### 使用spring-data实现接口
个人感觉mongo这种json风格的命令并不是很好用，当嵌套多层时简直要晃瞎眼，而且在spring-data-mongodb的java驱动实现时使用的又是命令式的风格，所以即使熟悉了mongo命令也不能很快写出对应的java代码：
#### 更新接口实现
借助spring-data-mongodb 中的Update、PushOperatorBuilder类实现
```
        List<String> ids = orderList.stream().map(Order::getId).collect(Collectors.toList());
        Query query = new Query();
        query.addCriteria(Criteria.where("userId").is(userId));
        query.addCriteria(Criteria.where("userType").is(userType));
        Update updateOld = new Update();
        updateOld.pull("orderList", Query.query(Criteria.where("_id").in(
                ids)));
        UpdateResult pullResult = mongoTemplate.updateFirst(query, updateOld, User.class);
        
        Update updateNew = new Update();
        PushOperatorBuilder push = updateNew.push("orderList");
        push.each(orderList);
        //push使用sort会让执行时间加倍
//        push.sort(new Sort(Direction.DESC, "createTime"));
        updateNew.set("userId", userId);
        updateNew.set("userType", userType);
        updateNew.set("userName", "name");
        UpdateResult pushResult = mongoTemplate
                .upsert(query, updateNew, User.class);
                
       Assert.isTrue(pullResult.wasAcknowledged() && pushResult.wasAcknowledged());

```
#### 查询接口实现
```
        MatchOperation matchUser = match(Criteria.where("userId").is(userId)
                .and("userType").is(userType));

        GroupOperation groupOperation = group("_id")
                .last("userType").as("userType")
                .last("userId").as("userId")
                .last("name").as("name")
                .push("orderList").as("orderList");
        MatchOperation matchCreateTime;
        if (maxCreateTime == 0L) {
            matchCreateTime = match(Criteria.where("orderList.createTime").gt(0L));
        } else {
            matchCreateTime = match(Criteria.where("orderList.createTime").lte(maxCreateTime));
        }

        Aggregation aggregation = Aggregation.newAggregation(
                matchUser,
                unwind("orderList"),
                matchCreateTime,
                sort(Direction.DESC, "orderList.createTime"),
                limit(size),
                groupOperation
        );
        AggregationResults<User> users = mongoTemplate
                .aggregate(aggregation, User.class, User.class);
        if (CollectionUtils.isEmpty(users.getMappedResults())) {
            return null;
        } else {
            return users.getMappedResults().get(0);
        }

```
### 性能测试
用jmh框架对查询和更新两个接口做性能测试，mongoDB实例安装在本地，以副本集方式运行，测试数据量为5000用户 每个用户10000订单，本机配置16GB内存，8核CPU。[测试用例源码地址][1]

这里的测试数据只能说明在指定数据量下接口的表现，随着数据量变化接口表现也不同。
`关于jmh输出，以updateByPush为例，说明99.9%的请求耗时分布在50.013-6.460到50.013+6.460之间，cnt表示执行次数`
#### 更新接口测试
测试结果表明，在这个实例中通过push/pull更新（不排序）比替换整个文档更新耗时减少25ms（平均值），如果更新时排序则和替换文档耗时相近。

通过push/pull更新(push时排序)
```
Result "updateByPush":
  50.013 ±(99.9%) 6.460 ms/op [Average]
  (min, avg, max) = (45.299, 50.013, 59.592), stdev = 4.273
  CI (99.9%): [43.553, 56.473] (assumes normal distribution)


# Run complete. Total time: 00:00:14

Benchmark                  Mode  Cnt   Score   Error  Units
JmhBenchmark.updateByPush  avgt   10  50.013 ± 6.460  ms/op

```
通过pull/push更新（push时不排序）
```
Result "updateByPush":
  25.034 ±(99.9%) 22.153 ms/op [Average]
  (min, avg, max) = (16.419, 25.034, 61.959), stdev = 14.653
  CI (99.9%): [2.880, 47.187] (assumes normal distribution)


# Run complete. Total time: 00:00:15

Benchmark                  Mode  Cnt   Score    Error  Units
JmhBenchmark.updateByPush  avgt   10  25.034 ± 22.153  ms/op
```
通过替换整个文档更新
```
Result "updateByReplace":
  45.985 ±(99.9%) 17.885 ms/op [Average]
  (min, avg, max) = (41.466, 45.985, 79.604), stdev = 11.830
  CI (99.9%): [28.100, 63.869] (assumes normal distribution)

Benchmark                     Mode  Cnt   Score    Error  Units
JmhBenchmark.updateByReplace  avgt   10  45.985 ± 17.885  ms/op
```

#### 查询接口测试
分别测试了slice查询，和聚合查询排序，聚合查询不排序三种。其中slice查询不代表实际的耗时，因为它不能实现按时间查询的功能。对于聚合查询，如果更新时排序则不需要再次排序。

slice查询分页(不能设置分页查询条件，不能排序)
```
Result "findBySlice":
  1.531 ±(99.9%) 0.128 ms/op [Average]
  (min, avg, max) = (1.437, 1.531, 1.669), stdev = 0.085
  CI (99.9%): [1.404, 1.659] (assumes normal distribution)

Benchmark                 Mode  Cnt  Score   Error  Units
JmhBenchmark.findBySlice  avgt   10  1.531 ± 0.128  ms/op
```
aggregation查询分页(按时间查询并分页)
```
Result "findByAggregation":
  7.612 ±(99.9%) 0.747 ms/op [Average]
  (min, avg, max) = (6.986, 7.612, 8.697), stdev = 0.494
  CI (99.9%): [6.865, 8.359] (assumes normal distribution)

Benchmark                       Mode  Cnt  Score   Error  Units
JmhBenchmark.findByAggregation  avgt   10  7.612 ± 0.747  ms/op
```
aggregation查询分页(按时间查询并分页，添加了sort操作)
```
Result "findByAggregation":
  15.609 ±(99.9%) 0.795 ms/op [Average]
  (min, avg, max) = (14.731, 15.609, 16.237), stdev = 0.526
  CI (99.9%): [14.814, 16.404] (assumes normal distribution)


Benchmark                       Mode  Cnt   Score   Error  Units
JmhBenchmark.findByAggregation  avgt   10  15.609 ± 0.795  ms/op
```

### 创建索引
创建索引可以优化查询速度，在创建索引后再次对查询和更新接口做性能测试，在本例中创建索引后并没有明显的提升，
而且**在嵌套文档上创建的索引在聚合查询中并没有用到，在聚合查询中只包含match和sort时才能使用索引，在unwind之前match或sort(orderList.createTime)会命中索引，但unwind执行后无法使用**

>来自mongo权威指南的描述：管道如果不是直接从原先的集合中使用数据，那就无法在筛选和排序中使用索引。

#### 对父文档创建索引
```
db.user.createIndex({userId:1},{unique:true})
```

创建索引会加速对用户的查询，但是也会增加写数据时的操作；在测试中添加索引后查询时间略有减少，
更新操作耗时也减少了(因为没有更新被索引字段)。

加索引后聚合查询时间(执行sort)，pull/push时间（不sort)和通过替换更新的时间：


```
Benchmark                       Mode  Cnt   Score   Error  Units
JmhBenchmark.findByAggregation  avgt   10  12.829 ± 0.592  ms/op
JmhBenchmark.updateByPush       avgt   10  17.962 ± 4.572  ms/op
JmhBenchmark.updateByReplace    avgt   10  41.691 ± 2.140  ms/op
```

#### 对子文档创建索引
去掉父文档上的索引，在嵌套的文档上创建一个createTime字段的索引

```
db.user.createIndex({"orderList.createTime":1})
db.user.getIndexes()
```

再执行测试结果如下：
```
Benchmark                       Mode  Cnt    Score     Error  Units
JmhBenchmark.findByAggregation  avgt   10  15.866 ± 0.863  ms/op
JmhBenchmark.updateByPush       avgt   10  172.905 ± 121.534  ms/op
JmhBenchmark.updateByReplace    avgt   10  127.146 ± 84.696  ms/op
```
实际上在查询时索引并没有被使用(COLLSCAN)，反而更新子文档的耗时因为要更新索引大大增加；
对聚合查询执行explain的输出也说明没有用到createTime索引：
```
			"$cursor" : {
				"query" : {
					"userId" : "u666",
					"userType" : "B"
				},
				"queryPlanner" : {
					"plannerVersion" : 1,
					"namespace" : "test.user",
					"indexFilterSet" : false,
					"parsedQuery" : {
						"$and" : [
							{
								"userId" : {
									"$eq" : "u666"
								}
							},
							{
								"userType" : {
									"$eq" : "B"
								}
							}
						]
					},
					"winningPlan" : {
						"stage" : "COLLSCAN",
```

#### ttl索引不支持嵌套文档
通常让mongo文档自动过期可以通过ttl索引实现，但是ttl索引在嵌套文档上无效，只能让父文档过期。
在createTime上设置ttl索引，不会执行过期操作
```
db.user.createIndex( { "orderList.createTime": 1 }, { expireAfterSeconds: 0 }
```
### 总结
使用mongoDB保存嵌套文档时，在被嵌套文档上的CRUD操作会变得相对更复杂，查询一般要通过聚合来实现，带来的好处是mongodb可以保证对一个文档的操作是原子性的，如果分散到多个文档保存需要mongoDB的事务支持(升级到v4.0)。
如果嵌套文档数组元素数量较多，还要注意最后单个文档大小不能超过16MB的限制。
TTL索引和嵌套文档上创建的索引不能保证一定有效，需要手动验证。
当嵌套文档较小时可以直接使用替换的方式更新，查询操作也可以直接返回整个文档，设计上避免了文档相互引用带来的复杂度。


  [1]: https://github.com/xianfengsong/someDemo/tree/master/spring-boot-study