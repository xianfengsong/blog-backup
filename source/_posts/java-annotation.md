# JAVA基础————两分钟学会注解Annotation

title: JAVA基础————两分钟看懂注解Annotation
date: 2016-01-26 00:05:46
categories: 
tags: [java基础,注解,annotation]

---
## 什么是注解

**注解是一种描述程序的[元数据][q1]，我们可以把他当做一种特殊的注释**　  
> [官方文档][q2] : *Annotations, a form of metadata, provide data about a program that is not part of the program itself.*　

+ 注解可以为编译器提供信息，如@Override，@SuppressWarnings
+ 可以代替xml等文件，为程序保存所需的配置
+ 可以在程序运行时根据注解执行一些操作，如Spring中的@Autowired

<!--more-->

## 注解的语法
### 声明注解
注解的定义与接口类似，也可以为注解添加属性。在实现注解时，方法名是对象属性名，返回值是属性值的类型。如 @Override的定义：
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override{}
```
自定义并实例化注解：
```java
@interface Developer{
  String [] name();
  int level() default 0;
}
//一个注解对象
@Developer(name={"j","p"},level=2)
```
### 内置注解
java提供几个内置注解来描述自定义的注解 
 **@Target**：应用到什么对象

| 名称        | 定义   |
| -------- | :-----  |
| ElementType.ANNOTATION_TYPE| 应用于注解 |
| ElementType.FIELD        |   属性    |
| ElementType.METHOD        |     方法   |
| ElementType.TYPE       |    类的任何元素   |
|[...更多内容][更多]|[...更多内容][更多]|


 **@Retention**：注解存储的位置

| 名称|定义|
|--|--|
|RetentionPolicy.SOURCE|保留在源码阶段，被编译器忽略|
|RetentionPolicy.CLASS|保留至编译阶段，被JVM忽略|
|RetentionPolicy.RUNTIME|保留至JVM可以在运行时使用|
**@Inherited**：
类的注解默认不会被继承给子类，如果需要应该在定义时指定@Inherited（只适用于一个类的注解）

 **@Document** : 

如果定义注解时指定@Document那么注解会被添加到用用对象的javadoc文档中

## 使用注解

我们已经了解了注解的基本语法，那么应该怎么在程序中处理注解呢？

![举个例子](/images/example.jpg)


&#160; &#160; &#160;假设公司办公楼里有不同类型的一些房间，而每天都会有各种身份的人来公司，如何让程序来控制这些人对不同房间的访问权限呢？

----------
&#160; &#160; &#160;
通常我们会将权限关系保存在xml等文件中，但同时也可以使用注解来保存各类房间允许进入的人员信息，并在程序中根据注解作出处理。

1.定义房间：

```java
public class Room {
    public void open(String name){
        System.out.println(name+" came in");}
    }
}
```

2.创建注解时选择RETENTION.RUNTIME，这样我们可以通过反射获取注解信息：
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Accessible {
    //保存允许进入房间的人员
    String [] value ();
}
```
3.创建WashRoom,MeetingRoom,BossRoom，并在open方法上添加注解：
```java
//例：洗手间 任何人都可以进入
public class WashRoom extends Room {
    @Accessible({"Boss","Staff","Visitor"})
    @Override
    public void open(String name){
        System.out.println("this is washroom");
        super.open(name);
    }
}
```
4.处理注解：
```java
//初始化房间和人员
Room[] rooms = {new WashRoom(), new MeetingRoom(), new BossRoom()};
String[] persons = {"Boss", "Staff", "Visitor"};

for (String person : persons) {
    for (Room room : rooms) {

        Method[] methods = room.getClass().getMethods();

            for (Method method : methods) {
                //用反射获取带有注解的方法
                if (method.isAnnotationPresent(Accessible.class)) {
                    Accessible accessAnnotation = method.getAnnotation(Accessible.class);
                    //获得注解内容
                    List<String> nameList = Arrays.asList(accessAnnotation.value());
                    if (nameList.contains(person)) {
                        try {
                            method.invoke(room, person);
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    } else {
                            System.out.println("forbidden！");
                        }
                    }
                }
            }
        }
```
# 全文完～
### 参考
oracle java文档：https://docs.oracle.com/javase/tutorial/java/annotations/index.html
Java深度历险（六）——Java注解 http://www.infoq.com/cn/articles/cf-java-annotation


  [img]:  http://7xl4v5.com1.z0.glb.clouddn.com/example.jpg
  [更多]: https://docs.oracle.com/javase/tutorial/java/annotations/predefined.html
  
[q1]: http://www.ruanyifeng.com/blog/2007/03/metadata.html

[q2]: https://docs.oracle.com/javase/tutorial/java/annotations/index.html


