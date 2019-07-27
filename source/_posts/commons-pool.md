title: 如何设计一个连接池：commons-pool2源码分析
date: 2017-06-12 00:10:50
categories: 开源框架
tags: 
 - 连接池
 - 源码
 - commons-pool
---

## 前言
大家对连接池的概念应该都很熟悉了，[apache commons-pool](https://commons.apache.org/proper/commons-pool/)是apache基金会的一个开源对象池组件，我们常用的数据库连接池dpcp和redis的java客户端jedis都使用commons-pool来管理连接。现在我们通过阅读commons-pool的部分代码来看一看他们在编写连接池组件时采用了什么样的设计思路和技术。

## 准备工作
首先我们思考一下，连接池除应该具备哪些功能？

 1. 连接池要能够管理不同类型的对象，同时作为一个服务组件不需要关心每种连接的创建细节。
 2. 连接池中的连接当长时间空闲时可能会被服务端主动关闭，或者受网络影响断开，连接池要能够定时检查它的连接是否可用。
 3. 支持并发操作,每个线程各种持有连接互不影响。

带着这三个问题，下面我们一起看一下commons-pool的代码,
它主要提供这个几种类型的对象池：
<!--more-->
![](/images/commonspool.gif)
从左至右看，SoftReferenceObjectPool是不用指定容量的对象池，ProxiedObject是使用代理模式来创建对象池的对象，在这篇文章主要介绍GenericObjectPool的实现。

## 封装对象
commons-pool定义了一个接口PooledObject来封装池中的对象，添加了空闲时间，使用时间等信息和对象状态流转的行为。
Pooledbject的状态有这几种：
|    状态     |    说明  |
| --------   | -----  | 
| IDLE     | 空闲 |
|ALLOCATED  |   使用中
|INVALID |不可用 即将/已经被销毁
|ABANDONED |被遗弃
|RETURNING |返回对象池
|EVICTION|  在队列中 正在被检查
|EVICTION_RETURN_TO_HEAD| 检查结束要放回队列头部
|VALIDATION| 在队列中 正在被检查
|VALIDATION_PREALLOCATED |验证结束要被分配
|VALIDATION_RETURN_TO_HEAD| 检查结束要放回队列头部

表格中以RETURN_TO_HEAD结尾的状态是在当前的检查操作和对象借出操作同时进行时出现的，在检查结束后为了尽量保证借出成功这个对象要放回队头。
在我们不需要自定义对象行为的情况下可以直接使用PooledObject的默认实现DefaultPooledObject，主要代码如下：
``` java
private final long createTime = System.currentTimeMillis();
private volatile long lastBorrowTime = createTime;
private volatile long lastUseTime = createTime;
private volatile long lastReturnTime = createTime;
private volatile long borrowedCount = 0;
@Override
    public synchronized boolean allocate() {
        if (state == PooledObjectState.IDLE) {
            state = PooledObjectState.ALLOCATED;
            lastBorrowTime = System.currentTimeMillis();
            lastUseTime = lastBorrowTime;
            borrowedCount++;
            if (logAbandoned) {
                borrowedBy = new AbandonedObjectCreatedException();
            }
            return true;
        } else if (state == PooledObjectState.EVICTION) {
            state = PooledObjectState.EVICTION_RETURN_TO_HEAD;
            return false;
        }
        return false;
    }
```
allocate() 定义了对象分配时的状态变化，当对象在空闲状态时才能被分配；如果对象正在做驱逐检查时,会把这个对象标记为EVICTION_RETURN_TO_HEAD.
```java
@Override
//返回值：对象是否当前未被使用
public synchronized boolean endEvictionTest(
            Deque<PooledObject<T>> idleQueue) {
        if (state == PooledObjectState.EVICTION) {
            state = PooledObjectState.IDLE;
            return true;
        } else if (state == PooledObjectState.EVICTION_RETURN_TO_HEAD) {
            //如果状态是EVICTION_RETURN_TO_HEAD表示当前这个对象试图被借出，返回false.
            state = PooledObjectState.IDLE;
            if (!idleQueue.offerFirst(this)) {
                // TODO - Should never happen
            }
        }
        return false;
    }
```

endEvictionTest()定义了驱逐检查之后对象状态的变化，可以看到EVICTION_RETURN_TO_HEAD状态的对象被放到空闲队列队头。

allocate()和borrowedCount等使用synchronized，violatile都做了线程安全的处理。

 
## 定义对象工厂
commons-pool为了管理多种对象使用工厂模式来创建要管理的对象，使用工厂方法后使用者可以通过重写PooledObjectFactory定义对象的实现，销毁，验证的具体实现，而commons-pool不需要关心这些细节它只管理PooledObject对象即可。

工厂接口：PooledObjectFactory
``` java
public interface PooledObjectFactory<T> {
  PooledObject<T> makeObject() throws Exception;
  void destroyObject(PooledObject<T> p) throws Exception;
  boolean validateObject(PooledObject<T> p);
  void activateObject(PooledObject<T> p) throws Exception;
  void passivateObject(PooledObject<T> p) throws Exception;
}
```
工厂抽象类：BasePooledObjectFactory
``` java
public abstract class BasePooledObjectFactory<T> implements PooledObjectFactory<T> {
...
    public abstract T create() throws Exception;
    public abstract PooledObject<T> wrap(T obj);
    @Override
    public PooledObject<T> makeObject() throws Exception {
        return wrap(create());
    }
    @Override
    public boolean validateObject(PooledObject<T> p) {
        return true;
    }

...
}
```
BasePooledObjectFactory添加了create,wrap两个抽象方法，create可以返回我们要放在连接池的对象，然后使用wrap封装为PooledObject即可，注意validateObject默认返回true;

## 对象池定义
对象池是我们保存对象的地方，对象的获取，归还和定时检查都通过对象池来实现。以GenericObjectPool为例，对象池使用了线程安全的集合类来保存对象，LinkedBlockingDeque用于保存空闲的对象，ConcurrentHashMap保存全部对象（包括各种状态）。
``` java
private final LinkedBlockingDeque<PooledObject<T>> idleObjects = new LinkedBlockingDeque<PooledObject<T>>();
private final Map<T, PooledObject<T>> allObjects = new ConcurrentHashMap<T, PooledObject<T>>();
```
borrowObject()从连接池获取对象：
连接池借出对象时，经过Abandoned和validate两种检查，在连接池满时根据配置执行对应的等待策略，当没有可用对象时会抛出异常。

Abandoned检查目标是连接池所有被借出的对象，主要防止对象借出之后长时间被占用，不能退还（或者使用者忘记return）到连接池导致连接被耗尽。超时时间由AbandonedConfig定义。

validate检查目标是当前即将被借出的对象，目的是保证提供的对象是可用的，检查方式由对象工厂的validateObject方法定义。对象工厂还有activateObject方法来验证对象，不过这个方法是强制执行的。

``` java
public T borrowObject(long borrowMaxWaitMillis) throws Exception {
// getNumIdle() 表示当前空空闲对象数量，getNumActive() 表示当前非空闲的对象数量，getMaxTotal()表示连接池容量
// 那么假设最大容量10个，非空闲8个 > 7 ，空闲对象只要少于2个，就需要开始Abandoned检查

        AbandonedConfig ac = this.abandonedConfig;
        if (ac != null && ac.getRemoveAbandonedOnBorrow() &&
                (getNumIdle() < 2) &&
                (getNumActive() > getMaxTotal() - 3) ) {
            removeAbandoned(ac);
        }

        PooledObject<T> p = null;
        
        //拷贝了变量 blockWhenExhausted
        //因为blockWhenExhausted的修改方法是public的，这样保证在并发情况下，方法执行周期内变量也不会变化
        boolean blockWhenExhausted = getBlockWhenExhausted();

        boolean create;
        long waitTime = 0;

        while (p == null) {
            create = false;
            // 设置了对象池耗尽时等待
            if (blockWhenExhausted) {
                //从空闲队列取对象
                p = idleObjects.pollFirst();
                if (p == null) {
                    //没有空闲对象 新建一个
                    create = true;
                    p = create();
                }
                if (p == null) {
                    if (borrowMaxWaitMillis < 0) {
                        //注意：没配置等待时间，会一直阻塞
                        p = idleObjects.takeFirst();
                    } else {
                        //按照配置的时间等待
.....
                        p = idleObjects.pollFirst(borrowMaxWaitMillis,
                                TimeUnit.MILLISECONDS);
.....
                    }
                }
                 //等待之后还是没有空闲对象
                if (p == null) {
                    throw new NoSuchElementException(
                            "Timeout waiting for idle object");
                }
                 //等待之后获得对象 尝试分配对象
                 //这个方法由pooledobject实现
                if (!p.allocate()) {
                    p = null;
                }
            } else {
                //没有配置blockWhenExhausted 不等待
                p = idleObjects.pollFirst();
                if (p == null) {
                    create = true;
                    p = create();
                }
                if (p == null) {
                    throw new NoSuchElementException("Pool exhausted");
                }
                if (!p.allocate()) {
                    p = null;
                }
            }
            //对象分配成功
            if (p != null) {

                try {
                   //激活对象
                    factory.activateObject(p);
                } catch (Exception e) {
                    try {
                        destroy(p);
                    } catch (Exception e1) {
                    }
                    p = null;
                    .....
                }
                //如果配置了对象检查
                if (p != null && (getTestOnBorrow() || create && getTestOnCreate())) {
                    boolean validate = false;
                    Throwable validationThrowable = null;
                    try {
                        validate = factory.validateObject(p);
                    } catch (Throwable t) {
                        PoolUtils.checkRethrow(t);
                        validationThrowable = t;
                    }
                    if (!validate) {
                    //验证失败 销毁对象
                        try {
                            destroy(p);
                         destroyedByBorrowValidationCount.incrementAndGet();
                        } catch (Exception e) {
                        }
                        p = null;
                        .....
                    }
                }
            }
        }
        //更新借出时间等信息
        updateStatsBorrow(p, waitTime);

        return p.getObject();
    }
```
evict()驱逐对象：

evict驱逐检查：上面介绍了两种对象的检查方式，evictor不同之处在于它由后台线程独立来完成，检查对象主要是连接池中的空闲连接，超时时间等可通过EvictionConfig配置。
``` java
public void evict() throws Exception {
        //确保连接池打开
        assertOpen();

        if (idleObjects.size() > 0) {

            PooledObject<T> underTest = null;
            
            EvictionPolicy<T> evictionPolicy = getEvictionPolicy();

            synchronized (evictionLock) {
                ...
                // 复制变量保证函数内值不改变，同上
                boolean testWhileIdle = getTestWhileIdle();
                //每次驱逐对象数可配置
                for (int i = 0, m = getNumTests(); i < m; i++) {
                    if (evictionIterator == null || !evictionIterator.hasNext()) {
                        //驱逐检查的顺序和空闲队列出入顺序保持一致 
                        if (getLifo()) {
                            //后进先出(逆序遍历)
                            evictionIterator = idleObjects.descendingIterator();
                        } else {
                            //先进先出
                            evictionIterator = idleObjects.iterator();
                        }
                    }
                    if (!evictionIterator.hasNext()) {
                        // Pool exhausted, nothing to do here
                        return;
                    }

                    try {
                        underTest = evictionIterator.next();
                    } catch (NoSuchElementException nsee) {
                        //因为idleObjects.iterator()方法并没有做同步控制，可能被检查的空闲对象再检查期间已经被借出不在队列中了，在并发条件下要考虑这种情况。。
                        // Object was borrowed in another thread
                        // Don't count this as an eviction test so reduce i;
                        i--;
                        evictionIterator = null;
                        continue;
                    }
                    //再次检查，对象在队列中但是要保证状态是空闲。。。
                    if (!underTest.startEvictionTest()) {
                        // Object was borrowed in another thread
                        // Don't count this as an eviction test so reduce i;
                        i--;
                        continue;
                    }
                    //EvictionPolicy定义了驱逐对象的策略，默认实现是DefaultEvictionPolicy
                    //调用ObjectPool的setEvictionPolicyClassName方法指定自定义策略
                    if (evictionPolicy.evict(evictionConfig, underTest,
                            idleObjects.size())) {
                        destroy(underTest);
                        destroyedByEvictorCount.incrementAndGet();
                    } else {
                        if (testWhileIdle) {
                            ...
                            //执行activateObject&validateObject的检查
                            ...
                        }
                        ...
                    }
                }
            }
        }
        //无处不在的Abandoned检查，可以配置连接池是否在evictor线程中执行removeAbandoned
        AbandonedConfig ac = this.abandonedConfig;
        if (ac != null && ac.getRemoveAbandonedOnMaintenance()) {
            removeAbandoned(ac);
        }
    }
```
驱逐线程的调用：
了解驱逐对象时要做的操作之后，我们来看一下后台的驱逐者线程是怎么定义&启动的
``` java
//GenericObjectPool.java
//构造方法中启动驱逐者线程
public GenericObjectPool(PooledObjectFactory<T> factory,
            GenericObjectPoolConfig config) {
        ...
        startEvictor(getTimeBetweenEvictionRunsMillis());
        ...
    }
```
驱逐线程的定义：
驱逐者Evictor,在BaseGenericObjectPool中定义，本质是由java.util.TimerTask定义的定时任务
```java
//父类BaseGenericObjectPool.java
final void startEvictor(long delay) {
        //BaseGenericObjectPool的私有成员，final Object evictionLock = new Object();做对象锁
        synchronized (evictionLock) {
            if (null != evictor) {
                //已存在evictor，取消驱逐者线程，所以也可以二次调用startEvictor来停止驱逐检查
                EvictionTimer.cancel(evictor);
                evictor = null;
                evictionIterator = null;
            }
            if (delay > 0) {
                evictor = new Evictor();
                EvictionTimer.schedule(evictor, delay, delay);
            }
        }
    }
```
Evitor定义：这里有一个比较有意思的问题，驱逐者线程Evictor被多个连接池共享,但是这些连接池可能属于不同的classloader,Evictor必须要保证它的所有行为在**当前这个连接池的classloader**下执行（这是开发者给commons-pool提交的bug,[详细描述请看这里](https://issues.apache.org/jira/browse/POOL-161)）
```java
//内部类Evictor
class Evictor extends TimerTask {
    
        @Override
        public void run() {
            ClassLoader savedClassLoader =
                    Thread.currentThread().getContextClassLoader();
            try {
                // 切换到当前连接池的classLoader
                Thread.currentThread().setContextClassLoader(
                        factoryClassLoader);

                // 执行上面的evict()方法
                try {
                    evict();
                } catch(Exception e) {
                    swallowException(e);
                } catch(OutOfMemoryError oome) {
                    // Log problem but give evictor thread a chance to continue
                    // in case error is recoverable
                    oome.printStackTrace(System.err);
                }
                // 驱逐之后还要保证空闲连接数量不能小于配置
                try {
                    ensureMinIdle();
                } catch (Exception e) {
                    swallowException(e);
                }
            } finally {
                // 切换回之前的classLoader
                Thread.currentThread().setContextClassLoader(savedClassLoader);
            }
        }
    }
```
驱逐策略：
在evict()方法中最后对象是否要被驱逐是调用了evictionPolicy.evict()的方法来判断的，commons-pool提供的驱逐策略如下：
```java
//DefaultEvictionPolicy.java

public class DefaultEvictionPolicy<T> implements EvictionPolicy<T> {
    @Override
    public boolean evict(EvictionConfig config, PooledObject<T> underTest,
            int idleCount) {
        //getIdleTimeMillis()空闲时间
        //config.getIdleSoftEvictTime()空闲连接大于配置的最小值时的超时时间
        //config.getIdleEvictTime()空闲连接超时时间与数量无关
        if ((config.getIdleSoftEvictTime() < underTest.getIdleTimeMillis() &&
                config.getMinIdle() < idleCount) ||
                config.getIdleEvictTime() < underTest.getIdleTimeMillis()) {
            return true;
        }
        return false;
    }
}

```
驱逐策略是支持自定义的，这里使用的是设计模式中的策略模式，我们只要实现EvictionPolicy接口，然后调用setEvictionPolicyClassName()方法既可以更换驱逐策略（实现类要尽可能简单，只描述一种算法即可）：
```java
//父类BaseGenericObjectPool.java
 public final void setEvictionPolicyClassName(
            String evictionPolicyClassName) {
        try {
            //使用接口+反射实现的策略模式
            Class<?> clazz = Class.forName(evictionPolicyClassName);
            Object policy = clazz.newInstance();
            if (policy instanceof EvictionPolicy<?>) {
                @SuppressWarnings("unchecked") 
                EvictionPolicy<T> evicPolicy = (EvictionPolicy<T>) policy;
                this.evictionPolicy = evicPolicy;
            }
        } catch (ClassNotFoundException e) {
            ...
        } catch (InstantiationException e) {
            ...
        } catch (IllegalAccessException e) {
            ...
        }
    }

```
通过阅读GenericObjectPool的部分代码，可以看出来并没有在每个获取&退还对象的方法都做同步控制，线程安全主要是由LinkedBlockingDeque，ConcurrentHashMap这两个并发集合保证的，因此开发者在编写非线程安全方法时也使用了局部变量复制可能被修改的值，多次检查对象状态之类的方法保证并发条件下程序正常的执行（没全部加锁能够提升性能，不过也会有这样的麻烦）。
总结一下连接池中对象的声明周期大概如下图：
![](/images/commonspoolflow.png)
蓝色线是evictor执行时对象状态的变化，红线是abandon执行的过程，绿色线是正常使用中对象的变化。



## 其他功能
附上一份objectpool的配置选项
```java
GenericObjectPoolConfig poolConfig = new GenericObjectPoolConfig();

基本配置：

        //最大连接
        poolConfig.setMaxTotal(100);
        //最大空闲连接
        poolConfig.setMaxIdle(5);
        //最小空闲连接 
        poolConfig.setMinIdle(5);
        //连接满时最多等待时间
        poolConfig.setMaxWaitMillis(5000L);

高级功能：
         //使用时检查对象（默认不检查）
        poolConfig.setTestWhileIdle(true);
        poolConfig.setTestOnCreate(true);
        poolConfig.setTestOnBorrow(true);
        poolConfig.setTestOnReturn(true);

        //jmx启用 之后可以实时的查看线程池对象的状态
        poolConfig.setJmxEnabled(false);
        poolConfig.setJmxNameBase("namebase");
        poolConfig.setJmxNamePrefix("nameprefix");

         //驱逐线程每次检查对象个数
        poolConfig.setNumTestsPerEvictionRun(2);
        //空闲连接被驱逐前能够保留的时间
        poolConfig.setMinEvictableIdleTimeMillis(10000L);
        //当空闲线程大于minIdle 空闲连接能够保留时间，同时指定会被上面的覆盖
        poolConfig.setSoftMinEvictableIdleTimeMillis(10000L);
        //驱逐线程执行间隔时间
        poolConfig.setTimeBetweenEvictionRunsMillis(200000L);

        //放弃长时间占用连接的对象
       AbandonedConfig abandonedConfig=new AbandonedConfig();
       abandonedConfig.setLogAbandoned(true);
       abandonedConfig.setUseUsageTracking(false);
       abandonedConfig.setRemoveAbandonedOnBorrow(true);
       abandonedConfig.setRemoveAbandonedOnMaintenance(true);
       abandonedConfig.setRemoveAbandonedTimeout(20);//second
```


  [1]: https://www.throwsnew.com/img/GenericObjectPool.PNG
  [2]: https://www.throwsnew.com/img/common-pool2.png
