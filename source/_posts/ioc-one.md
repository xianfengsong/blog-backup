# 动手编写一个IOC容器#

title: 动手编写一个IOC容器
date: 2016-09-28 23:30:55
categories: 
tags: [Spring]
---

### 什么是IOC
 所谓的IOC(Inversion of control)控制反转，实际是引入一个管理资源的第三方来统一管理分配资源，来代替各个资源持有者,使用者相互依赖的情况。

 以打牌为例，之前玩家需要自己摸牌，现在改成由发牌人为所有玩家发牌，这个过程就叫做“控制反转”，发牌人就是IOC容器，发牌人给玩家发牌的过程就被称为“依赖注入(Dependency Injection)”。

``` java
//不使用IOC Player自己负责实例化Card
public Class PlayerA{
    private Card myCard=new Card();
    ...
}
public Class PlayerB{
    private Card myCard=new Card();
    ...
}


//实例化在IOC容器（Dealer）中进行
public Class Dealer{
    private Card card=new Card();
    ...
}
//Player感知不到Dealer存在
public Class Player{
    @Inject
    private Card myCard;
}
```


在实际的j2ee项目中，使用依赖注入有这样几个好处： <!--more-->
1.使用IOC容器可以把加载Bean的工作集中进行，使用者不再负责资源的初始化，在编译期将资源和使用者解耦；
2.在使用单例模式的情况下可以避免重复的创建相同对象，减少资源占用；

### IOC容器工作过程

那么IOC容器怎么来实现依赖注入呢，这里提供一个基于注解的简单的IOC实现方案：
第一步：
我们将所有被IOC容器管理的类称为Bean，在项目启动时需要扫描当前所有的java Bean并放入到一个BeanMap中，并且将他们实例化，BeanMap保存BeanClass和Bean实例的对应关系。
第二步：
遍历BeanMap中的Bean，逐个判断Bean的成员变量中是否有@Inject注解，如果有，就从BeanMap中取出这个成员变量类对应的对象，注入给这个Bean。



### 编写代码
根据上面的流程我们需要定义这几个工具类：
**ClassUtil**：扫描项目package下的所有Class
**ClassHelper**: 调用ClassUtil，并返回需要的Bean
**ReflectionUtil**: 返回对象实例，调用Settter方法实现注入
**BeanHelper**: 初始化BeanMap
>代码参考《架构探险》一书

*ClassUtil:*
需要通过ClassLoader获得项目的所有资源，分别查找文件和jar包中的类
得到这个ClassSet之后，我们可以根据已经定义的注解来过滤出不同的Bean
``` java
public static Set<Class<?>> getClassSet(String packageName) {
        Set<Class<?>> classSet = new HashSet<Class<?>>();
        try {
            Enumeration<URL> urls = getClassLoader().getResources(
                    packageName.replace(".", "/"));
            while (urls.hasMoreElements()) {
                URL url = urls.nextElement();
                if (url != null) {
                    String protocol = url.getProtocol();
                    if (protocol.equals("file")) {
                        String packagePath = url.getPath().replaceAll("%20", " ");
                        addClassFromFile(classSet, packagePath, packageName);
                    } else if (protocol.equals("jar")) {
                        addClassFromJar(classSet, url);
                    }
                }
            }
        } catch (Exception e) {
            log.error("get class set failed", e);
            throw new RuntimeException(e);
        }
        return classSet;
    }
```
*BeanHelper:*
在初始化时立即将所有bean实例化，放入BeanMap中
``` java
private static final Map<Class<?>,Object> BEAN_MAP=new HashMap<Class<?>, Object>();
    static {
        Set<Class<?>> beanClass=ClassHelper.getBeanClassSet();
        for(Class<?> cls:beanClass){
            //通过反射获取对象实例
            Object obj=ReflectionUtil.getInstance(cls);
            BEAN_MAP.put(cls,obj);
        }
    }
```
之后就可以在IOCHelper中实现依赖注入了。
``` java
public class IOCHelper {
    static{
        Map<Class<?>,Object> beanMap=BeanHelper.getBeanMap();

        if(!CollectionsUtil.isEmpty(beanMap)){
            for(Map.Entry<Class<?>,Object> beanEntry:beanMap.entrySet()){
                //实例化对象由容器统一管理
                Class<?> beanClass=beanEntry.getKey();
                Object beanInstance=beanEntry.getValue();
                Field[] fields=beanClass.getDeclaredFields();
                if(fields.length!=0){
                    for(Field beanField:fields){
                    //判断是否需要注入
                        if(beanField.isAnnotationPresent(Inject.class)){
                            Class<?> beanFieldClass=beanField.getType();
                            Object beanFieldInstance=beanMap.get(beanFieldClass);
                            if(beanFieldInstance!=null){
                                ReflectionUtil.setField(beanInstance,beanField,beanFieldInstance);
                            }
                        }
                    }
                }
            }
        }
    }
}
```
**测试：**
定义一个Service
``` java
@Service
public class LoginService {
    public LoginService(){}
    public void login(){
        System.out.println("login...");
    }
}
```
再定义一个Controller，依赖于Service
``` java
@Controller
public class IndexController {
    @Inject
    private LoginService loginService;
    public void login(){
        System.out.println("call login method");
        loginService.login();
    }

}
```
测试IOCHelper,是否注入了Service(@service,@Controller标记的类会被装配成Bean)
``` java

public class Start {

    public static void main(String[] arg){
        ClassUtil.loadClass(IOCHelper.class.getName(), true);
        IndexController controller= BeanHelper.getBean(IndexController.class);
        controller.login();
    }
}
```

### 总结
到这里相信你已经了解了IOC容器的大概工作方式，这里的IOCHelper还存在很多问题：
1. 只支持Bean的setter注入，没有支持构造器注入。
2. BeanMap中所有的Class都是单例的，每次getBean得到的都是同一实例。
3. 不支持集合类型的对象注入
4. 这里BeanMap不支持并发访问


[完整代码地址][1]


  [1]: https://github.com/xianfengsong/concise-framework/tree/master/src/main/java/com/throwsnew/conciseframework