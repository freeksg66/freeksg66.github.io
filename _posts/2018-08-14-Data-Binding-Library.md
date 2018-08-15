---
layout:     post
title:      Data Binding Library
subtitle:   Android开发文档学习
date:       2018-08-14
author:     GL
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - Android
---

Data Binding Library是一个support library，它允许您将布局中的UI组件绑定到应用程序中的数据源，而不是通过编程方式。

布局通常是在activities中调用UI framework的方法进行编码的。例如，通过调用findViewById()方法去找到一个TextView控件，并且将viewModel变量的userName属性与这个控件进行绑定：

{% highlight java %}
TextView textView = findViewById(R.id.sample_text);
textView.setText(viewModel.getUserName());
{% endhighlight %}

下面的例子展示了怎样直接在布局文件中直接使用Data Binding Library去分配text到指定的控件。
这就消除了调用上面所示的任何Java代码的需要。注意在赋值表达式中使用@{}标志：

{% highlight java %}
<TextView
    android:text="@{viewmodel.userName}" />
{% endhighlight %}

在布局文件中绑定组件能让你在activities中少些很多UI framework调用方法，使之更简洁，更简单，更具维护性。这也能提升你app的性能和有助于防止内存泄漏和空指针的异常。

使用下面这个链接可以学习怎样使用Data Binding Library在你的app中。[Android Data Binding Library samples](https://github.com/googlesamples/android-databinding)

下面我们开始正式学习Data Binding Library的使用。

## 开始

首先让你的Android Studio支持Data Binding Library。

Data Binding Library提供了既灵活又广泛的兼容性，因为它是一个support library，你能使用它泡在Android4.0（API level 14）或者更高的版本。

推荐在你的项目中使用最新的Android gradle插件。然而，只要版本大于等于1.5.0就支持data binding。更多细节，请看[怎样更新Gradle的Android插件](https://developer.android.google.cn/studio/releases/gradle-plugin#updating-plugin) 

## 搭建环境

首先从Android SDK的Support Repository中下载Data Binding Library。更多的信息请看[更新IDE和SDK工具](https://developer.android.google.cn/studio/intro/update)。

如果要配置你的app中使用data binding，那么需要在你app module的build.gradle文件中添加dataBinding元素，如下面的代码所示：

	android {
    	...
    	dataBinding {
    	    enabled = true
    	}
	}

> 注意：你必须在app module中配置data binding机器依赖库，即使app module没有直接使用data binding

## 布局文件和绑定表达式

这种表达式语言允许你去写处理由views分发的events表达式。Data Binding Library会自动生成所需的类，以便在布局中与数据对象绑定视图。

Data binding布局文件有些轻微的不同，起始以一个layout为根tag，后面跟着data元素，再后面就是一个普通的view根元素了。这个你想绑定的view元素是一个还未绑定的布局文件。可以看如下代码：

{% highlight java %}
<?xml version="1.0" encoding="utf-8"?>
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
           android:text="@{user.firstName}"/>
       <TextView android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.lastName}"/>
   </LinearLayout>
</layout>
{% endhighlight %}

data中的user变量描述了也许会被用于布局文件的一个属性。

{% highlight java %}
<variable name="user" type="com.example.User" />
{% endhighlight %}

布局文件中的表达式可以通过使用符号“@{}”写进控件属性中。这里TextView的text被设置为user变量的firsrName属性：

{% highlight java %}
<TextView android:layout_width="wrap_content"
          android:layout_height="wrap_content"
          android:text="@{user.firstName}" />
{% endhighlight %}

> 注意：布局文件的表达式应该保持短小且简洁，因为它们不能被单元测试并且受限于IDE的支持。为了简化布局文件表达式，你可以使用自定义的[binding adapter](https://developer.android.com/topic/libraries/data-binding/binding-adapters)

## 数据对象

现在我们假定你有一个“老式（plain-old）”的对象，它被描述为User实体：

{% highlight java %}
public class User {
  public final String firstName;
  public final String lastName;
  public User(String firstName, String lastName) {
      this.firstName = firstName;
      this.lastName = lastName;
  }
}
{% endhighlight %}

上面这种对象类型一旦拥有数据就永远不会改变。一次读取永不改变的这种模式在application中非常常见。它可以通过一种访问方法去获取数据，如下面代码所示：

{% highlight java %}
public class User {
  private final String firstName;
  private final String lastName;
  public User(String firstName, String lastName) {
      this.firstName = firstName;
      this.lastName = lastName;
  }
  public String getFirstName() {
      return this.firstName;
  }
  public String getLastName() {
      return this.lastName;
  }
}
{% endhighlight %}

从数据绑定的观点来看，这两个类是等价的。android:text属性中的“@{user.firstName}”表达式在前一个类中直接从firstName中获取值，而在后一类中通过getFirstName()方法去获取值。或者，如果该方法存在，它也许会被解析为firstName()。

## 绑定数据

每一个布局文件会生成一个绑定类。默认地，新生成的类名是基于布局文件的名字的，将它转化为Pascal case并在后面加上后缀。如果上面的布局文件名是activity_main.xml，那么对应生成的类就是ActivityMainBinding。这个类持有来自布局文件的所有layout属性绑定（例如，user变量），并且知道怎唐根据绑定的表达式分配数值。推荐的创建绑定的方法是在inflaing布局时执行绑定操作，代码如下所示：

{% highlight java %}
@Override
protected void onCreate(Bundle savedInstanceState) {
   super.onCreate(savedInstanceState);
   ActivityMainBinding binding = DataBindingUtil.setContentView(this, R.layout.activity_main);
   User user = new User("Test", "User");
   binding.setUser(user);
}
{% endhighlight %}

在运行阶段，app的UI会显示“Test”用户。或者，你可以通过<b>LayoutInflater</b>获取，如下面代码所示：

{% highlight java %}
ActivityMainBinding binding = ActivityMainBinding.inflate(getLayoutInflater());
{% endhighlight %}

如果你在使用数据绑定Fragment、ListView、或者RecyclerView适配器的items，你也许更喜欢用inflate()方法绑定类，而不是使用DataBindingUtil类，如下代码所示：

{% highlight java %}
ListItemBinding binding = ListItemBinding.inflate(layoutInflater, viewGroup, false);
// or
ListItemBinding binding = DataBindingUtil.inflate(layoutInflater, R.layout.list_item, viewGroup, false);
{% endhighlight %}

## 表达式语法

### 公共特性

表达式语法看起来很像在代码中发现表达式。在表达式语法中你可以使用下面的操作符和关键字：

* 数学计算的：+ - * / %
* 字符串连接的：+
* 逻辑判断的：&& ||
* 二值运算的：& | ^
* 一元运算：+ - ！ ~
* 位移运算：>> >>> <<
* 比较运算：== > < >= <=
* instanceof
* Grouping()
* Literals-character,String,numeric,null
* Cast
* Method calls
* Field access
* Array access []
* 三元运算符 ?:

例子：

{% highlight java %}
android:text="@{String.valueOf(index + 1)}"
android:visibility="@{age < 13 ? View.GONE : View.VISIBLE}"
android:transitionName='@{"image_" + id}'
{% endhighlight %}

### Missing operations

下面的操作符是缺省表达式符号的，你可以使用这些操作符在代码中：

* this
* super
* new
* Expicit generic invocation

## 空聚合操作符

空聚合操作符（??）的含义是，如果左边的对象不为null，就选择左边的，否则就选择右边的对象。

{% highlight java %}
android:text="@{user.displayName != null ? user.displayName : user.lastName}"
{% endhighlight %}

上面这个式子等价于：

{% highlight java %}
android:text="@{user.displayName != null ? user.displayName : user.lastName}"
{% endhighlight %}

### 属性引用

一个表达式能通过使用下面的样式引用一个类中的一个属性。

{% highlight java %}
android:text="@{user.lastName}"
{% endhighlight %}

### 回避空指针异常

生成的数据绑定代码自动地检查null值，并且回避空指针异常。举个例子，在表达式“@{user.name}”中，如果user为空，那么user.name配分配一个它的默认值null。如果你引用user.age，其中age是int类型的，那么数据绑定使用默认值0。

### 集合

普通的集合，比如arrays,lists,sparse lists,和maps，能通过[]操作符方便的访问数据元素。

{% highlight java %}
<data>
    <import type="android.util.SparseArray"/>
    <import type="java.util.Map"/>
    <import type="java.util.List"/>
    <variable name="list" type="List<String>"/>
    <variable name="sparse" type="SparseArray<String>"/>
    <variable name="map" type="Map<String, String>"/>
    <variable name="index" type="int"/>
    <variable name="key" type="String"/>
</data>
…
android:text="@{list[index]}"
…
android:text="@{sparse[index]}"
…
android:text="@{map[key]}"
{% endhighlight %}

> 注意：你也能在map中使用object.key访问一个值。例如，在上面的例子中@{map[key]}能被替换为@{map.key}

### 字符串常量

你可以使用单引号将属性值包括起来，这样你就可以在表达式中使用双引号，如下面代码所示：

{% highlight java %}
android:text='@{map["firstName"]}'
{% endhighlight %}

你也可以使用双引号将属性值包括起来，一旦你这样做，字符串常量就应该使用back quotes:

{% highlight java %}
android:text="@{map[`firstName`]}"
{% endhighlight %}

### 资源

你可以用下面的符号在表达式中访问资源：

{% highlight java %}
android:padding="@{large? @dimen/largePadding : @dimen/smallPadding}"
{% endhighlight %}

格式化的字符串和复数也许会在提供的参数中被计算出来

{% highlight java %}
android:text="@{@string/nameFormat(firstName, lastName)}"
android:text="@{@plurals/banana(bananaCount)}"
{% endhighlight %}

如果一个复数有不同的参数，那么所有的参数都会被传递：

{% highlight java %}
  Have an orange
  Have %d oranges

android:text="@{@plurals/orange(orangeCount, orangeCount)}"
{% endhighlight %}

有些资源需要显示类型评估，如下表所示：

<table>
    <th>Type</th>
    <th>Normal reference</th>
    <th>Expression reference</th>
    <tr>
        <td>String[]</td>
        <td>@array</td>
        <td>@stringArray</td>
    </tr>
    <tr>
        <td>int[]</td>
        <td>@array</td>
        <td>@intArray</td>
    </tr>
    <tr>
        <td>TypedArray</td>
        <td>@array</td>
        <td>@typeArray</td>
    </tr>
    <tr>
        <td>Animator</td>
        <td>@animator</td>
        <td>@animator</td>
    </tr>
    <tr>
        <td>StateListAnimator</td>
        <td>@animator</td>
        <td>@stateListAnimator</td>
    </tr>
    <tr>
        <td>color int</td>
        <td>@colcor</td>
        <td>@color</td>
    </tr>
    <tr>
        <td>ColorStateList</td>
        <td>@color</td>
        <td>@colorStateList</td>
    </tr>
</table>

## 事件处理

数据绑定允许你通过写表达式去处理事件，这个事件是从view上发送的（例如，onClick()方法）。事件属性名由监听方法名决定。例如，View.OnClickListener有一个onClick()方法，那么这个事件属性就是Android:onClick。

这里有一些特殊的点击事件处理需要用其他的属性，以便于和android:onClick造成冲突。详情见下表：

<table>
	<th>Class</th>
	<th>Listener setter</th>
	<th>Attribute</th>
	<tr>
		<td>SearchView</td>
		<td>setOnSearchClickListener(View.OnClickListener)</td>
		<td>android:onSearchClick</td>
	</tr>
	<tr>
		<td>ZoomControls</td>
		<td>setOnZoomInClickListener(View.OnClickListener)</td>
		<td>android:onZoomIn</td>
	</tr>
	<tr>
		<td>ZoomControls</td>
		<td>setOnZoomOutClickListener(View.OnClickListener)</td>
		<td>android:onZoomOut</td>
	</tr>
</table>

你可以使用下面的机制去处理一个事件：

* [方法引用](https://developer.android.com/topic/libraries/data-binding/expressions#method_references):在你的表达式中，你可以引用符合监听器方法签名的方法。当一个表达式评估一个方法引用时，数据绑定封装了监听器方法引用和监听器拥有者对象，并且在目标view上设值这个监听器。如果表达式被计算为null，数据绑定不会创建监听器，并且设置的监听器会被置为null。
* [监听绑定](https://developer.android.com/topic/libraries/data-binding/expressions#listener_bindings)：当事件发生时，lambda表达式会被计算。数据绑定总是创建一个监听器，这个监听器是被设置在view上的。当事件被分发时，监听器会执行lambda表达式。

### 方法引用

事件可以直接绑定到处理程序方法，简单地通过android:onClick能在activity中分配一个方法。相比于View的onClick属性的一个主要优势是这个表达式在编译器会被执行，因此如果这个方法不存在或者它的签名不正确，你将收到一个编译错误。

方法引用和监听绑定主要的不同在于，当数据绑定而不是事件触发时创建实现了实际的监听器。如果你更倾向于在事件发生时执行表达式，你应该使用[监听绑定](https://developer.android.com/topic/libraries/data-binding/expressions#listener_bindings)。

为了分配一个事件到它的handler中，使用一个常规的绑定表达式，这个表达式的值是调用的方法名。例如，思考底下的例子：

{% highlight java %}
public class MyHandlers {
    public void onClickFriend(View view) { ... }
}
{% endhighlight %}

绑定的表达式能通过onClickFriend()方法分配一个click监视器到一个view，如下所示：

{% highlight java %}
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
   <data>
       <variable name="handlers" type="com.example.MyHandlers"/>
       <variable name="user" type="com.example.User"/>
   </data>
   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <TextView android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.firstName}"
           android:onClick="@{handlers::onClickFriend}"/>
   </LinearLayout>
</layout>
{% endhighlight %}

> 注意：表达式中方法的签名必须准确地匹配监听器中方法的签名。

### 监听绑定

绑定监听是在事件发生时绑定表达式。虽然这与方法引用很相似，但是监听绑定可以让你在绑定表达式中运行任意的数据。这个特性可以在Android Gradle插件2.0及以后的版本中获得。

在方法引用中，方法的参数必须匹配事件监听器的参数。在监听绑定中，只需要你的返回值必须匹配监听器中的返回值（除非它被显示的声明为void）。例如，考虑下面的presenter类有一个onSaveClick()方法：

{% highlight java %}
public class Presenter {
    public void onSaveClick(Task task){}
}
{% endhighlight %}

然后，你可以绑定click事件到onSaveClick()方法，如下所示：

{% highlight java %}
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
    <data>
        <variable name="task" type="com.android.example.Task" />
        <variable name="presenter" type="com.android.example.Presenter" />
    </data>
    <LinearLayout android:layout_width="match_parent" android:layout_height="match_parent">
        <Button android:layout_width="wrap_content" android:layout_height="wrap_content"
        android:onClick="@{() -> presenter.onSaveClick(task)}" />
    </LinearLayout>
</layout>
{% endhighlight %}

当在一个表达式中使用一个回调函数时，数据绑定自动地创建必要的监听器，并且为事件自动注册监听器。当视图触发事件时，数据绑定执行给定的表达式。正如常规的绑定表达式所示，当那些监听器表达式执行的时候，你扔会得到空的或者这线程安全的数据绑定。

在上面的例子中，我们还没有定义view的参数（这个view参数会被传递到onClick(View）中）。监听绑定为监听器参数提供了两个选择：您可以忽略该方法的所有参数或名称。如果你更倾向于参数名字，你可以使用它们在你的表达式中。例如，上面的表达式可以写为如下的形式：

{% highlight java %}
android:onClick="@{(view) -> presenter.onSaveClick(task)}"
{% endhighlight %}

或者，如果你想在表达式中使用参数，你可以这样做：

{% highlight java %}
public class Presenter {
    public void onSaveClick(View view, Task task){}
}

android:onClick="@{(theView) -> presenter.onSaveClick(theView, task)}"
{% endhighlight %}

你可以在至少有一个参数时使用lambda表达式：

{% highlight java %}
public class Presenter {
    public void onCompletedChanged(Task task, boolean completed){}
}

<CheckBox android:layout_width="wrap_content" android:layout_height="wrap_content"
      android:onCheckedChanged="@{(cb, isChecked) -> presenter.completeChanged(task, isChecked)}" />
{% endhighlight %}

如果在你正在监听的事件中返回了一个不为void类型的值，那么你的表达式也必须返回相同类型的值。例如，如果你想监听长点击事件，那么你的表达式应该返回一个boolean类型值。

{% highlight java %}
public class Presenter {
    public boolean onLongClick(View view, Task task) { }
}

android:onLongClick="@{(theView) -> presenter.onLongClick(theView, task)}"
{% endhighlight %}

如果表达式因为null而不能二执行，数据绑定会返回一个相同类型的默认值。例如，引用类型的默认值是null，int的默认值是0，boolean的默认值是false，等等。

如果你需要使用一个判断表达式，你可以使用void作为一个符号。

{% highlight java %}
android:onClick="@{(v) -> v.isVisible() ? doSomething() : void}"
{% endhighlight %}


### 避免使用复杂的监听

一方面，监听表达式是功能强大的，并且使你的code非常简洁易读。另一方面，包含复杂表达式的监听器会使你的布局文件难以阅读和维护。这些表达式应该尽可能简洁的调用你的回调函数从你的UI中传递获得的数据。你应该在监听器表达式调用的回调函数中实现业务逻辑。

## imports,variables,和includes

数据绑定库提供了imports，variables和includes几个特性。imports可以简化在你的布局文件中引用外面的classes。variables允许你描述一个使用在绑定表达式中的一个属性。includes让你在app中复用复杂的布局。

### imports

imports允许你在布局文件中容易地引用其他的classes。在data元素中可以使用0个或更多的import元素。下面的代码在布局文件中引入了View类：

{% highlight java %}
<data>
    <import type="android.view.View"/>
</data>
{% endhighlight %}

引入view类允许你从绑定表达式中引用它。下面的例子展示了怎样引用View类中的VISIBLE和GONE常量：

{% highlight java %}
<TextView
   android:text="@{user.lastName}"
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"
   android:visibility="@{user.isAdult ? View.VISIBLE : View.GONE}"/>
{% endhighlight %}

### 类型别名 Type aliases

当有class名冲突时，其中的一个类应该被重命名。下面的例子在com.example.real.estate中重命名View类为Vista:

{% highlight java %}
<import type="android.view.View"/>
<import type="com.example.real.estate.View"
        alias="Vista"/>
{% endhighlight %}

你可以使用Vista去引用com.example.real.estate.View，并且在布局文件中可以使用android.view.View引用View。

### imports other classes

在变量和表达式中，已经引入的类型能作为类型引用被使用。下面的例子展示了User和List作为一个变量类型被使用：

{% highlight java %}
<data>
    <import type="com.example.User"/>
    <import type="java.util.List"/>
    <variable name="user" type="User"/>
    <variable name="userList" type="List<User>"/>
</data>
{% endhighlight %}

> 注意：Android Studio还不能处理imports，所以在你的IDE中可能无法使用已经导入的变量的自动补全功能。你的应用程序仍然可以编译，你可以解决这个IDE问题通过用全名定义变量。

你也可以使用已经导入的类型转化一个表达式。下面的例子将connection属性转化为User类型：

{% highlight java %}
<TextView
   android:text="@{((User)(user.connection)).lastName}"
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"/>
{% endhighlight %}

当引用表达式中的静态字段和方法时，也可以使用已经导入的类型。下面的代码导入了MyStringUtils类并且引用它的capitalize方法：

{% highlight java %}
<data>
    <import type="com.example.MyStringUtils"/>
    <variable name="user" type="com.example.User"/>
</data>
…
<TextView
   android:text="@{MyStringUtils.capitalize(user.lastName)}"
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"/>
{% endhighlight %}

其中java.lang.*是被自动导入的。

### variables

你可以在data元素中使用多个variable元素。每个variable元素都描述了一个属性，该属性可以在布局文件中的绑定表达式中使用。下面的例子声明了user，image，和note variables：

{% highlight java %}
<data>
    <import type="android.graphics.drawable.Drawable"/>
    <variable name="user" type="com.example.User"/>
    <variable name="image" type="Drawable"/>
    <variable name="note" type="String"/>
</data>
{% endhighlight %}

variable类型在编译器被检查，因此如果一个variable实现了[Observable](https://developer.android.com/reference/android/databinding/Observable.html)或者是[observable collection](https://developer.android.com/topic/libraries/data-binding/observability.html#observable_collections),通过反射的方式获取type。如果variable是一个base类或者没有实现Observable的接口，那么这个variable不会被“观察”。

当不同的布局文件有多种配置时（例如，landscape或者portrait），这些variable会被合并。The variables take the default managed code values until the setter is called—null for reference types, 0 for int, false for boolean, etc.

一个特殊的context variable会在需要时通过绑定表达式生成。context值是从root view的[getContext()]()方法中获取的Context对象。这个context variable能通过显示地定义，从而override默认值。

### includes

下面的例子展示了从name.xml和contract.xml布局文件中included user variables。

{% highlight java %}
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:bind="http://schemas.android.com/apk/res-auto">
   <data>
       <variable name="user" type="com.example.User"/>
   </data>
   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <include layout="@layout/name"
           bind:user="@{user}"/>
       <include layout="@layout/contact"
           bind:user="@{user}"/>
   </LinearLayout>
</layout>
{% endhighlight %}

数据绑定不支持merge元素的直接子元素。例如，在面布局文件是不支持的：

{% highlight java %}
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:bind="http://schemas.android.com/apk/res-auto">
   <data>
       <variable name="user" type="com.example.User"/>
   </data>
   <merge><!-- Doesn't work -->
       <include layout="@layout/name"
           bind:user="@{user}"/>
       <include layout="@layout/contact"
           bind:user="@{user}"/>
   </merge>
</layout>
{% endhighlight %}
