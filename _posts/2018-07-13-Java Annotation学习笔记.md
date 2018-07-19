---
layout:     post
title:      Java Annotation学习笔记
subtitle:   在本地轻松调试自己的Blog
date:       2018-07-13
author:     BY
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Java
---

## 概念

>An annotation is a form of metadata, that can be added to Java source code. Classes, methods, variables, parameters and packages may be annotated. Annotations have no direct effect on the operation of the code they annotate.

注解Annotation是java 1.5的新特性，是一种能够添加到 Java 源代码的语法元数据。类、方法、变量、参数、包都可以被注解，可用来将信息元数据与程序元素进行关联。Annotation 中文常译为“注解”。

## 分类

标准Annotation
> @Override、@Deprecated、@SuppressWarnings

元Annotation
> @Retention、@Target、@Inherited、@Documented

自定义Annotation
> 自定义 Annotation 表示自己根据需要定义的 Annotation，定义时需要用到上面的元 Annotation。

## 自定义Annotation

调用

{% highlight ruby %}
@DBTableY(name = "YYD__DB")
public class DB {
    @DBColumnY(name = "column11xx")
    public String column1;
    @DBColumnY(name = "column22xx")
    public String column2;
}
{% endhighlight %}

定义

{% highlight ruby %}
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface DBTableY {
    String name() default "YUYIDONG11";
}

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD})
public @interface DBColumnY {
    String name() default "YUYIDONG";
}
{% endhighlight %}

1.通过 @interface 定义，注解名即为自定义注解名

2.注解配置参数名为注解类的方法名，且：

* 所有方法没有方法体，没有参数没有修饰符，实际只允许 public & abstract 修饰符，默认为 public ，不允许抛异常
* 方法返回值只能是基本类型，String, Class, annotation, enumeration 或者是他们的一维数组
* 若只有一个默认属性，可直接用 value() 函数。一个属性都没有表示该 Annotation 为 Mark Annotation

3.可以加 default 表示默认值

### 元Annotation

@Documented 是否会保存到 Javadoc 文档中

@Retention 保留时间，可选值 SOURCE（源码时），CLASS（编译时），RUNTIME（运行时），默认为 CLASS，SOURCE 大都为 Mark Annotation，这类 Annotation 大都用来校验，比如 Override, SuppressWarnings

作用：表示需要在什么级别保存该注释信息，用于描述注解的生命周期（即：被描述的注解在什么范围内有效）

取值（RetentionPoicy）有：

* SOURCE:在源文件中有效（即源文件保留）

* CLASS:在class文件中有效（即class保留）

* RUNTIME:在运行时有效（即运行时保留）

​

@Target 可以用来修饰哪些程序元素，如 TYPE, METHOD, CONSTRUCTOR, FIELD, PARAMETER 等，未标注则表示可修饰所有。

作用：用于描述注解的使用范围（即：被描述的注解可以用在什么地方）

取值(ElementType)有：

* CONSTRUCTOR:用于描述构造器
* FIELD:用于描述域
* LOCAL_VARIABLE:用于描述局部变量
* METHOD:用于描述方法
* PACKAGE:用于描述包
* PARAMETER:用于描述参数
* TYPE:用于描述类、接口(包括注解类型) 或enum声明

@Inherited 是否可以被继承，默认为 false

当 @Inherited annotation类型标注的annotation的Retention是RetentionPolicy.RUNTIME，则反射API增强了这种继承性。如果我们使用java.lang.reflect去查询一个

@Inherited annotation类型的annotation时，反射代码检查将展开工作：检查class和其父类，直到发现指定的annotation类型被发现，或者到达类继承结构的顶层。

## 注解

使用 @interface自定义注解时，自动继承了java.lang.annotation.Annotation接口，由编译程序自动完成其他细节。在定义注解时，不能继承其他的注解或接口。 @interface用来声明一个注解，其中的每一个方法实际上是声明了一个配置参数。方法的名称就是参数的名称，返回值类型就是参数的类型（返回值类型只能是基本类型、Class、String、enum）。可以通过default来声明参数的默认值。

{% highlight ruby %}
public @interface 注解名 {定义体}
{% endhighlight %}

注解参数的可支持数据类型：

* 所有基本数据类型（int,float,boolean,byte,double,char,long,short)
* String类型
* Class类型
* enum类型
* Annotation类型
* 以上所有类型的数组

## 解析

<b>运行时 Annotation 解析</b>

运行时 Annotation 指 @Retention 为 RUNTIME 的 Annotation

{% highlight ruby %}
method.getAnnotation(AnnotationName.class);
method.getAnnotations();
method.isAnnotationPresent(AnnotationName.class);
{% endhighlight %}

>getAnnotation(AnnotationName.class) 表示得到该 Target 某个 Annotation 的信息，因为一个 Target 可以被多个Annotation 修饰
>
>getAnnotations() 则表示得到该 Target 所有 Annotation
>
>isAnnotationPresent(AnnotationName.class) 表示该 Target 是否被某个 Annotation 修饰

{% highlight ruby %}
Class clazz = DB.class;
if (clazz.isAnnotationPresent(DBTableY.class)) {
    DBTableY dbTableY = (DBTableY) clazz.getAnnotation(DBTableY.class);
    Log.i("yyd", "dbTableY.name()--->" + dbTableY.name());
    Field[] fields = clazz.getFields();
    for (Field field : fields) {
        DBColumnY dbColumnY = field.getAnnotation(DBColumnY.class);
        Log.i("yyd", "dbColumnY.name()--->" + dbColumnY.name());
        Log.i("yyd", "field.getName()--->" + field.getName());
    }
}
{% endhighlight %}

<b>编译时 Annotation 解析</b>

编译时 Annotation 指 @Retention 为 CLASS 的 Annotation，甴编译器自动解析。需要做的

* 自定义类集成自 AbstractProcessor
* 重写其中的 process 函数

编译器在编译时自动查找所有继承自 AbstractProcessor 的类，然后调用他们的 process 方法去处理。

{% highlight ruby %}
SupportedAnnotationTypes({ "com.yydcdut.test.annotation.DBColumnY" })
public class DBColumnProcessor extends AbstractProcessor {    
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) {
        for (TypeElement te : annotations) {
            for (Element element : env.getElementsAnnotatedWith(te)) {
                DBColumnY dbColumnY = element.getAnnotation(DBColumnY.class);
                Log.i("yyd",element.getEnclosingElement().toString());
            }
        }
        return false;
    }
}
{% endhighlight %}

SupportedAnnotationTypes 表示这个 Processor 要处理的 Annotation 名字。

process 函数中参数 annotations 表示待处理的 Annotations，参数 env 表示当前或是之前的运行环境。

process 函数返回值表示这组 annotations 是否被这个 Processor 接受，如果接受后续子的 rocessor 不会再对这个 Annotations 进行处理。

