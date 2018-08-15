---
layout:     post
title:      Generated binding classes学习
subtitle:   Android开发文档翻译
date:       2018-08-15
author:     GL
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - Android
---

Data Binding Library生成用于访问布局文件中的variable和views的绑定类。这篇文字主要讲怎样创建和自定义生成绑定类。

生成的绑定类连接了布局文件中的layout variables和views。绑定类的名字和包名可以[自定义](https://developer.android.com/topic/libraries/data-binding/generated-binding#custom_binding_class_names)。所有的绑定类都继承于[ViewDataBinding](https://developer.android.com/reference/android/databinding/ViewDataBinding.html)类。

每个布局文件会生成一个绑定类。默认情况下，绑定类的名字是基于布局文件的名字的。如果布局文件是activity_main.xml，那么对应生成的类就是ActivityMainBinding。这个类持有来自layout properties到layout views的所有绑定，并且知道怎样通过表达式分配值。

## 创建绑定类

在inflating布局之后，应该立即创建绑定对象，以确保视图层次结构在与布局内的表达式绑定之前不会被修改。最常见的绑定对象到布局文件的方法是在绑定类上使用静态方法。你可以inflate视图结构并且通过inflate()方法绑定对象到视图，正如下面代码所示：

{% highlight java %}
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    MyLayoutBinding binding = MyLayoutBinding.inflate(getLayoutInflater());
}
{% endhighlight %}

这里有一个inflate()方法的可替代版本，就是除了LayoutInflater对象外使用ViewGroup对象，如下所示：

{% highlight java %}
MyLayoutBinding binding = MyLayoutBinding.inflate(getLayoutInflater(), viewGroup, false);
{% endhighlight %}

如果布局文件通过一个不同的机制inflate，那么它要单独绑定，如下所示：

{% highlight java %}
MyLayoutBinding binding = MyLayoutBinding.bind(viewRoot);
{% endhighlight %}

有时，绑定类型是提前不可预知的。在这种情况下，可以使用[DataBindingUtil](https://developer.android.com/reference/android/databinding/DataBindingUtil.html)类来创建，如下面代码所示：

{% highlight java %}
View rootView = LayoutInflater.from(this).inflate(layoutId, parent, attachToParent);
ViewDataBinding binding = DataBindingUtil.bind(viewRoot);
{% endhighlight %}

如果你使用数据绑定在Fragment、ListView、或者RecyclerView适配器的item中，你也许更倾向于使用绑定类的inflate()方法，或者DataBindingUtil类，如下所示：

{% highlight java %}
ListItemBinding binding = ListItemBinding.inflate(layoutInflater, viewGroup, false);
// or
ListItemBinding binding = DataBindingUtil.inflate(layoutInflater, R.layout.list_item, viewGroup, false);
{% endhighlight %}

## Views with IDs

Data Binding Library在绑定类中为每一个在布局中有ID的视图创建一个不可变字段。例如，Data Binding Library根据下面布局文件中的TextView类型，创建了firstName和lastName字段。

{% highlight java %}
<layout xmlns:android="http://schemas.android.com/apk/res/android">
   <data>
       <variable name="user" type="com.example.User"/>
   </data>
   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <TextView android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.firstName}"
   android:id="@+id/firstName"/>
       <TextView android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.lastName}"
  android:id="@+id/lastName"/>
   </LinearLayout>
</layout>
{% endhighlight %}

数据绑定库从视图层级中提取视图，包括来自视图层次结构中的id。这种机制比在布局文件中每次调用findViewById()方法更快。

虽然ID在不用数据绑定是是没必要存在的，但是仍有一些情况下是必要的，那就是从代码访问视图。

## Variables

数据绑定库为在布局文件中每个声明的variable生成了访问方法。例如，下面的布局为user、image、和note这三个variable，在绑定类中生成了setter和getter方法

{% highlight java %}
<data>
   <import type="android.graphics.drawable.Drawable"/>
   <variable name="user" type="com.example.User"/>
   <variable name="image" type="Drawable"/>
   <variable name="note" type="String"/>
</data>
{% endhighlight %}

## ViewStubs

不像常规的视图，[ViewStub](https://developer.android.com/reference/android/view/ViewStub.html)对象一开始是一个不可见的view。当它们变得可见或者显示的inflate时，它们通过inflate另一个布局来替换自己的布局。

因为ViewStub本质上是从View的层级结构中消失了，绑定对象的的视图也必须消失，以便于在垃圾回收器中声明。因为视图是final的，所以ViewStubProxy对象在生成的绑定类中代替ViewStub，当ViewStub存在时，它允许你访问ViewStub，并且当ViewStub被inflate时，它也可以访问inflate的视图层次结构。

当inflate另一个布局时，必须为新的布局建立一个绑定。因此，ViewStubProxy必须监听ViewStub的OnInflateListener，并在需要时建立绑定。因为只有一个监听器可以在给定的时间存在，所以ViewStubProxy允许你设置一个onInflateListener，它在建立绑定之后调用它。

## 直接绑定

当variable或observable对象发生变化时，绑定将在下一帧之前更改。然而，有时必须立即执行绑定。要强制执行，请使用executePendingBindings()方法。

## 提前绑定

### 动态的variable

有时，具体的绑定类是未知的。例如,一个RecyclerView.Adapter针对任意布局不知道具体的绑定类。它仍然必须在调用onBindViewHolder()方法时分配绑定值。

在下面的例子中，RecyclerView绑定的所有布局都有一个item variable。BindingHolder对象有一个getBinding()方法返回了ViewDataBinding基类。

{% highlight java %}
public void onBindViewHolder(BindingHolder holder, int position) {
    final T item = mItems.get(position);
    holder.getBinding().setVariable(BR.item, item);
    holder.getBinding().executePendingBindings();
}
{% endhighlight %}

> 注意：数据绑定库在模块包中生成一个名为BR的类，其中包含用于数据绑定的资源的id。在上面的例子中，数据绑定库自动生成BR.item variable。

## 后台线程

您可以在后台线程中更改数据模型，只要它不是集合。数据绑定在评估期间将每个variable/字段本地化，以避免任何并发性问题。

## 自定义绑定类的名字

默认地，自己与布局文件的名字生成了一个绑定类，以大写字母开头，移除下划线，复制后面的字符，并且加上了<b>Binding</b>单词作为后缀。这个类被放置在模块包下的一个databinding包中。例如，布局文件名为contract_item.xml生成了ContractItemBinding类。如果module包是com.example.my.app，那么绑定类被放置在com.example.my.app.databinding包中。

绑定类也许通过调整data元素的class属性被重命名或者被放置在不同的包下。例如，下面的布局在现在module下databinding包中生成了ContractItem绑定类：

{% highlight java %}
<data class="ContactItem">
    …
</data>
{% endhighlight %}

你可以通过在一个period内预先固定类名来在另一个包中生成绑定类。下面的例子在module包中生成绑定类：

{% highlight java %}
<data class=".ContactItem">
    …
</data>
{% endhighlight %}

你也可以在你想绑定类的地方使用全包名来生成。下面的例子在com.example包中创建了ContractItem绑定类：

{% highlight java %}
<data class="com.example.ContactItem">
    …
</data>
{% endhighlight %}
