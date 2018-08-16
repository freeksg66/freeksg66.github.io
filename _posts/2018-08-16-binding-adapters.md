---
layout:     post
title:      Binding adapters学习
subtitle:   Android开发文档翻译
date:       2018-08-16
author:     GL
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - Android
---

绑定适配器会负责的调用合适的framework回调方法去设值值。一个例子是调用setText()方法去设值一个属性值。另一个例子是调用setOnClickListener()方法去设值一个事件监听器。

数据绑定库允许你指定调用方法来设置一个值，提供你自己的绑定逻辑，并使用适配器指定的返回对象的类型。

## 设值属性值

无论何时一个绑定数据改变了，生成的绑定类必须通过绑定表达式调用view上的setter方法。你可以允许数据绑定库自动确定方法，显式地声明方法，或者提供自定义逻辑来选择方法。

### 自动选择方法

对于一个名为example的属性，数据绑定库自动地尝试寻找```setExample(arg)```方法，这个方法接受兼容类型的参数。属性的命名空间不被考虑，只有在寻找方法时才使用属性名和类型。

举个例子，假定一个表达式为```android:text="@{user.name}"```，数据绑定库会寻找```setText(arg)```方法，该方法接收的数据类型与```user.getName()```返回的类型一致。如果```user.getName()```返回类型是```String```类型，数据绑定库会寻找一个接受```String```类型参数的```setText()```方法。如果表达式返回一个```int```类型，那么数据绑定库会寻找一个接收```int```类型参数的```setText()```方法。这个表达式必须返回正确的类型，以便于你可以正确的转换返回值。

即使给定的name没有属性存在，数据绑定也可以正常个工作。通过数据绑定，你可以为任何setter创建属性。举个例子，虽然[DrawerLayout](https://developer.android.com/reference/android/support/v4/widget/DrawerLayout.html)类没有任何属性，但是有大量的setters。下面的布局自动地使用```setScrimColor(int)```和```setDrawerListener(DrawerListener)```方法，分别作为该app的```app:scrimColor```和```app:drawerListener```属性。

{% highlight java %}
<android.support.v4.widget.DrawerLayout
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:scrimColor="@{@color/scrim}"
    app:drawerListener="@{fragment.drawerListener}">
{% endhighlight %}

### 指定一个自定义的方法名

一些属性拥有不与名字匹配的setters。在这些情况下，可以使用[BindingMethods](https://developer.android.com/referenceandroid/databinding/BindingMethods.html)注解将setter分配给一个属性。这个注解这一使用在一个类上，也能包含多个[BindingMethods](https://developer.android.com/referenceandroid/databinding/BindingMethods.html)注解。被注解的绑定方法能被用于你的app上的任何类。在下面的例子中，```android:tint```属性是由[setImageTintList(ColorStateList)](https://developer.android.com/reference/android/widget/ImageView.html#setImageTintList(android.content.res.ColorStateList))方法设置的，而不是由```setTint()```方法设置的。

{% highlight java %}
@BindingMethods({
       @BindingMethod(type = "android.widget.ImageView",
                      attribute = "android:tint",
                      method = "setImageTintList"),
})
{% endhighlight %}

大多数时候，你不需要在Android framework类中重命名setters的名字。这些属性已经通过name约定实现了，以便自动找到匹配的方法。

### 提供自定义的逻辑

一些逻辑需要自定义绑定逻辑。例如，对于```android:paddingLeft```属性没有分配的setter。你可以使用```setPadding(left, top, right, bottom)```方法来给属性```android:paddingLeft```设定值。一个使用[BindingAdapter](https://developer.android.com/reference/android/databinding/BindingAdapter.html)注解的静态的绑定适配器方法允许你自定义setter设值属性的方式。

Android framework类的属性已经创建了```BindingAdapter```注解。举个例子，下面的例子展示了为```paddingLeft```属性绑定适配器：

{% highlight java %}
@BindingAdapter("android:paddingLeft")
public static void setPaddingLeft(View view, int padding) {
  view.setPadding(padding,
                  view.getPaddingTop(),
                  view.getPaddingRight(),
                  view.getPaddingBottom());
}
{% endhighlight %}

参数类型是非常重要的。第一个参数表示与属性关联的view的类型。第二个参数表示给定属性的绑定表达式中接受的类型。

绑定适配器是对于其他自定义类型是非常有用的。举个例子，一个自定义的loader可以从工作线程中加载一张图象。

当冲突发生的时候，你定义的绑定表达式override默认的Android framework适配器。

你也可以在adapter中接受多个属性，如下面例子所示：

{% highlight java %}
@BindingAdapter({"imageUrl", "error"})
public static void loadImage(ImageView view, String url, Drawable error) {
  Picasso.get().load(url).error(error).into(view);
}
{% endhighlight %}

你可以像例子那样，在布局文件中使用这个适配器。注意```@drawable/venueError```引用了你app中的一个资源，可在有效的绑定表达式中，资源文件要用```@{}```来包括。

{% highlight java %}
<ImageView app:imageUrl="@{venue.imageUrl}" app:error="@{@drawable/venueError}" />
{% endhighlight %}

> 注意：数据绑定库忽视了自定义的命名空间来进行匹配的目的。

对一个ImageView对象，如果```imageUrl```和```error```都在适配器中调用，并且```imageUrl```是一个字符串，```error```是一个```Drawable```。如果你想让适配器在任何属性被设置时调用，那么你可以将适配器可选的```requireAll```标志设置为false，如下所示：

{% highlight java %}
@BindingAdapter(value={"imageUrl", "placeholder"}, requireAll=false)
public static void setImageUrl(ImageView imageView, String url, Drawable placeHolder) {
  if (url == null) {
    imageView.setImageDrawable(placeholder);
  } else {
    MyImageLoader.loadInto(imageView, url, placeholder);
  }
}
{% endhighlight %}

> 注意：当冲突发生时，你绑定的适配器override了默认的数据绑定适配器。

绑定适配器方法也许可以在它们的handlers中可选地获取旧值。一个方法应该先获取所有的旧值，再将获取新值，如下面代码所示：

{% highlight java %}
@BindingAdapter("android:paddingLeft")
public static void setPaddingLeft(View view, int oldPadding, int newPadding) {
  if (oldPadding != newPadding) {
      view.setPadding(newPadding,
                      view.getPaddingTop(),
                      view.getPaddingRight(),
                      view.getPaddingBottom());
   }
}
{% endhighlight %}

事件处理函数也许被用在接口或抽象类的抽象方法中，如下所示：

{% highlight java %}
@BindingAdapter("android:onLayoutChange")
public static void setOnLayoutChangeListener(View view, View.OnLayoutChangeListener oldValue,
       View.OnLayoutChangeListener newValue) {
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {
    if (oldValue != null) {
      view.removeOnLayoutChangeListener(oldValue);
    }
    if (newValue != null) {
      view.addOnLayoutChangeListener(newValue);
    }
  }
}
{% endhighlight %}

像下面这样使用事件处理函数在你的布局文件中：

{% highlight java %}
<View android:onLayoutChange="@{() -> handler.layoutChanged()}"/>
{% endhighlight %}

当一个监听器由多个方法时，它必须分割这些监听器。例如，```View.OnAttachStateChangeListener```有两个方法：```onViewAttachedToWindow(View)```和```onViewDetachedFromWindow(View)```。数据绑定库提供了两个接口去区分它们的属性处理理函数：

{% highlight java %}
@TargetApi(VERSION_CODES.HONEYCOMB_MR1)
public interface OnViewDetachedFromWindow {
  void onViewDetachedFromWindow(View v);
}

@TargetApi(VERSION_CODES.HONEYCOMB_MR1)
public interface OnViewAttachedToWindow {
  void onViewAttachedToWindow(View v);
}
{% endhighlight %}

因为一个监听器改变了也能影响其他的监听器，你需要一个适用于任何属性的适配器或者两者皆适用。
你可以在注解中将```requireALL```设置为false去指定不是每一个属性必须要与绑定表达式相关联，如下所示：

{% highlight java %}
@BindingAdapter({"android:onViewDetachedFromWindow", "android:onViewAttachedToWindow"}, requireAll=false)
public static void setListener(View view, OnViewDetachedFromWindow detach, OnViewAttachedToWindow attach) {
    if (VERSION.SDK_INT >= VERSION_CODES.HONEYCOMB_MR1) {
        OnAttachStateChangeListener newListener;
        if (detach == null && attach == null) {
            newListener = null;
        } else {
            newListener = new OnAttachStateChangeListener() {
                @Override
                public void onViewAttachedToWindow(View v) {
                    if (attach != null) {
                        attach.onViewAttachedToWindow(v);
                    }
                }
                @Override
                public void onViewDetachedFromWindow(View v) {
                    if (detach != null) {
                        detach.onViewDetachedFromWindow(v);
                    }
                }
            };
        }

        OnAttachStateChangeListener oldListener = ListenerUtil.trackListener(view, newListener,
                R.id.onAttachStateChangeListener);
        if (oldListener != null) {
            view.removeOnAttachStateChangeListener(oldListener);
        }
        if (newListener != null) {
            view.addOnAttachStateChangeListener(newListener);
        }
    }
}
{% endhighlight %}

上面的例子相比于普通的例子有些复杂，因为View类使用了```addOnAttachStateChangeListener()```和```removeOnAttachStateChange()```方法代替了```OnAttachStateChange``` ```OnAttachStateChangeListener```中的一个setter方法。```android.databinding.adapters.ListenerUtil```类帮助与以前的监听器保持联系，因此它们也许应该在绑定适配器中被移除。

通过```@TargetApi(VERSION_CODES.HONEYCOMB_MR1)```注解```OnViewDetachedFromWindow```和```OnViewAttachedToWindow```，只有在Android3.1（API level 12）或者更高的版本中，数据绑定代码生成器认识这个监听器，并且支持```addOnAttachStateChangeListener()```方法。

## 对象转化

### 自动地对象转化

当一个对象从绑定表达式中返回时，数据绑定库会选择设置属性值的方法。这个对象会被转化为选择方法的一个类型参数。在app中可以通过使用```ObservableMap```类储存数据方便的完成对象转化，如下所示：

{% highlight java %}
<TextView
   android:text='@{userMap["lastName"]}'
   android:layout_width="wrap_content"
   android:layout_height="wrap_content" />
{% endhighlight %}

> 注意：你也许更倾向于使用```object.key```标注在map中引用一个值。例如，在上面的例子中```@{userMap["lastName"]}```能被替换为```@{userMap.lastName}```。

在表达式中的```userMap```对象会返回一个值，它被自动转换为用于设置```android:text```属性值的```setText(CharSequence)```方法中的参数类型。如果参数类型不确定，你必须在表达式中转换返回的类型。

### 自定义转换

在一些情况下，需要在指定的类型之间自定义的转换。例如，虽然一个view的```android:background```属性期待一个```Drawable```，但是```color```值是一个整数。下面的例子展示了一个期待```Drawable```的属性，但是这里却提供了一个整数：

{% highlight java %}
<View
   android:background="@{isError ? @color/red : @color/white}"
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"/>
{% endhighlight %}

无论何时一个```Ddrawable```被期待并且一个整数被返回，这个```int```将被转化为```ColorDrawable```。这个转换能使用一个[BindingConversion]()注解的静态的方法完成，如下所示：

{% highlight java %}
@BindingConversion
public static ColorDrawable convertColorToDrawable(int color) {
    return new ColorDrawable(color);
}
{% endhighlight %}

然而，在绑定表达式中提供的值的类型必须是一致的。你不能在同一个表达式中使用不同的类型，如下所示：

{% highlight java %}
<View
   android:background="@{isError ? @drawable/error : @color/white}"
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"/>
{% endhighlight %}