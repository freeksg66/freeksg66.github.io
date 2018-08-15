---
layout:     post
title:      observable data objects学习
subtitle:   Android开发文档翻译
date:       2018-08-15
author:     GL
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - Android
---

Observability是一种能让一个被观察的对象当它内部数据改变时通知其他观察者的能力。这个数据绑定库允许你创建可观察对象，字段或集合。

虽然任何普通的对象能被数据绑定使用，但是更改对象并不能自动地创建或更新UI。数据绑定能赋予你的数据对象一种能力，一种当它数据改变时，像一个监听器一样通知其它被观察者。这里有三个不同类型的classes：[objects](https://developer.android.com/topic/libraries/data-binding/observability#observable_objects)，[field](https://developer.android.com/topic/libraries/data-binding/observability#observable_fields)，和[collections](https://developer.android.com/topic/libraries/data-binding/observability#observable_collections)。

当这些观察者数据对象中的一个被绑定在UI上，并且数据对象的一个属性改变时，UI也自动的改变了。

## Observable fileds

如果你的classes中只有少数几个属性，创建这些classes并实现了Observable接口是不值得的。在这种情况下，你可以使用通用的Observable class并且按照下面的基本类型类去创建observable字段：

* [ObservableBoolean](https://developer.android.com/reference/android/databinding/ObservableBoolean.html)
* [ObservableByte](https://developer.android.com/reference/android/databinding/ObservableByte.html)
* [ObservableChar](https://developer.android.com/reference/android/databinding/ObservableChar.html)
* [ObservableShort](https://developer.android.com/reference/android/databinding/ObservableShort.html)
* [ObservableInt](https://developer.android.com/reference/android/databinding/ObservableInt.html)
* [ObservableLong](https://developer.android.com/reference/android/databinding/ObservableLong.html)
* [ObservableFloat](https://developer.android.com/reference/android/databinding/ObservableFloat.html)
* [ObservableDouble](https://developer.android.com/reference/android/databinding/ObservableDouble.html)
* [ObservableParcelable](https://developer.android.com/reference/android/databinding/ObservableParcelable.html)

Observable字段是一个自包含observable对象，这个对象有单一的字段。这个基本版本回避了在执行操作时的装箱和卸箱。为了使用这个机制，在java程序中可以创建一个public final属性，如下面的例子所示：

{% highlight java %}
private static class User {
    public final ObservableField<String> firstName = new ObservableField<>();
    public final ObservableField<String> lastName = new ObservableField<>();
    public final ObservableInt age = new ObservableInt();
}
{% endhighlight %}

为了获得相应的字段，我们为字段添加set()和get()方法，如下所示:

{% highlight java %}
user.firstName.set("Google");
int age = user.age.get();
{% endhighlight %}

> 注意：Android studio 3.1和更高的版本允许你用LiveData对象取代observable字段，这会给你的app带来一些额外的好处。更多的信息请看[User LiveData to notify the UI about data changes](https://developer.android.com/topic/libraries/data-binding/architecture.html#livedata)。

### Observable collections

一些app使用动态的结构去持有数据。Observable collections允许通过使用一个key访问这些结构。[ObservableArrayMap](https://developer.android.com/reference/android/databinding/ObservableArrayMap.html)类适用于以key作为一种引用类型时，比如String，如下面代码所示：

{% highlight java %}
ObservableArrayMap<String, Object> user = new ObservableArrayMap<>();
user.put("firstName", "Google");
user.put("lastName", "Inc.");
user.put("age", 17);
{% endhighlight %}

在布局文件中，按如下方式使用map：

{% highlight java %}
<data>
    <import type="android.databinding.ObservableMap"/>
    <variable name="user" type="ObservableMap<String, Object>"/>
</data>
…
<TextView
    android:text="@{user.lastName}"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"/>
<TextView
    android:text="@{String.valueOf(1 + (Integer)user.age)}"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"/>
{% endhighlight %}

当key是整数时，可以用[ObservableArrayList](https://developer.android.com/reference/android/databinding/ObservableArrayList.html)，如下所示：

{% highlight java %}
ObservableArrayList<Object> user = new ObservableArrayList<>();
user.add("Google");
user.add("Inc.");
user.add(17);
{% endhighlight %}

在布局文件中，能通过下标的方式访问list的元素，如下所示：

{% highlight java %}
<data>
    <import type="android.databinding.ObservableList"/>
    <import type="com.example.my.app.Fields"/>
    <variable name="user" type="ObservableList<Object>"/>
</data>
…
<TextView
    android:text='@{user[Fields.LAST_NAME]}'
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"/>
<TextView
    android:text='@{String.valueOf(1 + (Integer)user[Fields.AGE])}'
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"/>
{% endhighlight %}

## Observable objects

一个实现了[Observable](https://developer.android.com/reference/android/databinding/Observable.html)接口的类允许注册监听器，当观察对象的属性发生改变时这个监听器会被通知。

虽然这个[Observable](https://developer.android.com/reference/android/databinding/Observable.html)接口有增加和删除监听器的机制，但是你必须决定合适发送这些通知。为了使开发更容易，Data Binding Library提供了[BaseObservable](https://developer.android.com/reference/android/databinding/BaseObservable.html)类，该类实现了监听器注册机制。实现了BaseObservable的数据类会在属性发生改变时负责任的发出通知。你可以通过在getter上使用[Bindable](https://developer.android.com/reference/android/databinding/Bindable.html)注解，在setter中调用[notifyPropertyChanged()](https://developer.android.com/reference/android/databinding/BaseObservable.html#notifyPropertyChanged(int))方法，如下所示：

{% highlight java %}
private static class User extends BaseObservable {
    private String firstName;
    private String lastName;

    @Bindable
    public String getFirstName() {
        return this.firstName;
    }

    @Bindable
    public String getLastName() {
        return this.lastName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
        notifyPropertyChanged(BR.firstName);
    }

    public void setLastName(String lastName) {
        this.lastName = lastName;
        notifyPropertyChanged(BR.lastName);
    }
}
{% endhighlight %}

在包含了使用数据绑定的资源ID的module package中，数据绑定生成了一个叫做BR的类。在编译期，[Bindable](https://developer.android.com/reference/android/databinding/Bindable.html)注解会在BR类文件中生成了一个实体。如果数据类的基类没改变，那么能通过使用一个[PropertyChangeRegistry](https://developer.android.com/reference/android/databinding/PropertyChangeRegistry.html)对象实现Observable接口，去有效的注册和通知监听器。