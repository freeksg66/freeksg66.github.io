---
layout:     post
title:      用能感知生命周期的组件处理生命周期
subtitle:   Android开发文档翻译
date:       2018-08-17
author:     GL
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - Android
---

生命周期感知的组件会响应另一个组件的生命周期状态的变化而执行动作，比如activity和fragment。
这些组件帮助你生产了更好管理的和轻量级代码，并且更易于维护。

一个常见的模式是在activity和fragment的生命周期方法中实现依赖组件的相关操作。然而，这个模式导致了到吗不易管理，并且容易产生错误。通过使用生命周期感知的组件，你可以将依赖组建代码移出生命周期方法，并且移入到它们各自的组件中。

```android.arch.lifecycle```包提供了能让你构建可感知生命周期组件的类和接口，这些组件基于activity或者fragment现在的生命周期阶段可以自动地调整它们的行为。

> 注意：为了引入```android.arch.lifecycle```到你的Android项目中，可以参考[add components to your project](https://developer.android.com/topic/libraries/architecture/adding-components.html#lifecycle)。

大多数有Android Framework定义的app组件有依附于它们的生命周期。在程序运行时，生命周期是通过操作系统或者framework代码来管理的。生命周期是Android工作的核心，你的应用程序必须尊重它们。不这样做可能会触发内存泄漏甚至应用程序崩溃。

假定我们屏幕上有一个展示设备位置的activity。一个普通的实现，也许就是下面的方式：

{% highlight java %}
class MyLocationListener {
    public MyLocationListener(Context context, Callback callback) {
        // ...
    }

    void start() {
        // connect to system location service
    }

    void stop() {
        // disconnect from system location service
    }
}


class MyActivity extends AppCompatActivity {
    private MyLocationListener myLocationListener;

    @Override
    public void onCreate(...) {
        myLocationListener = new MyLocationListener(this, (location) -> {
            // update UI
        });
    }

    @Override
    public void onStart() {
        super.onStart();
        myLocationListener.start();
        // manage other components that need to respond
        // to the activity lifecycle
    }

    @Override
    public void onStop() {
        super.onStop();
        myLocationListener.stop();
        // manage other components that need to respond
        // to the activity lifecycle
    }
}
{% endhighlight %}

即使这个例子在你的app上跑的还不错，但为了响应生命周期现在的状态，你最终会有太多的用来管理UI和其他组件调用。为了管理不同的组件将大量的代码放到生命周期方法之中（比如```onStart```和```onStop()```），这将会使你的代码非常难维护。

另外，你不能保证组件会在activity或者fragment stop之前start。如果我们需要一个长时操作，比如在```onStart()```中检查一些配置，这点会非常正确。在```onStart()```方法之前```onStop()```方法结束了的地方会造成了竞争条件，这个控件会比它需要的时间活的更久。

{% highlight java %}
class MyActivity extends AppCompatActivity {
    private MyLocationListener myLocationListener;

    public void onCreate(...) {
        myLocationListener = new MyLocationListener(this, location -> {
            // update UI
        });
    }

    @Override
    public void onStart() {
        super.onStart();
        Util.checkUserStatus(result -> {
            // what if this callback is invoked AFTER activity is stopped?
            if (result) {
                myLocationListener.start();
            }
        });
    }

    @Override
    public void onStop() {
        super.onStop();
        myLocationListener.stop();
    }
}
{% endhighlight %}

```android.arch.lifecycle```包提供了帮助你以一种坚韧和独立的方式解决这些问题的类和接口。

## 生命周期

```lifecycle```是一个持有关于组件（比如activity或者fragment）生命周期信息的类，并且允许其它对象去观察这个阶段。

```lifecycle```使用两个主要的枚举器来跟踪相关组件的生命周期状态：

<b>Event</b>

来自framework和生命周期的类会分发生命周期事件。在activities或fragments中，这些事件将与回调函数相关联。

<b>State</b>

Lifecycle对象或跟踪组件现在的状态。

![](http://pbmurxnd0.bkt.clouddn.com/lifecycle-states.png)

把状态看作是图和事件的节点，作为这些节点之间的边界。

一个类能通过在它的方法上增加注解的方式监控组件的生命周期状态。然后你能通过调用```lifecycle```类的```addObserver()```方法增加一个观察者，并且传递一个你的观察者实例，如下面代码所示：

```java
public class MyObserver implements LifecycleObserver {
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    public void connectListener() {
        ...
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    public void disconnectListener() {
        ...
    }
}

myLifecycleOwner.getLifecycle().addObserver(new MyObserver());
```

在这个例子中，```myLifecycleOwner```对象实现了LifecycleOwner接口，这个接口将在下节进行介绍。

## LifecycleOwner接口

```LifecycleOwner```是一个单方法接口，它表示一个类拥有```Lifecycle```。它只有一个方法```getLifecycle()```，它必须通过一个类去实现。如果你试图管理整个app的生命周期，请看```ProcessLifecycleOwner```。

这个接口抽象了单个类的生命周期的所有权（比如Fragment和AppCompatActivity），并且允许编写与它们一起工作的组件。任何自定义的application类都能实现```LifecycleOwner```接口。

实现了```LifecycleObserver```接口的组件可以无缝地与实现了```LifecycleOwner```接口的组件一起工作，因为一个拥有着能提供一个生命周期，这样一个观察者就能注册并观察生命周期。

对于位置跟踪的例子，我们可以让```MyLocationListener```类实现```LifecycleOwner```接口，并且然后在onCreate（）方法中使用activity的```Lifecycle```初始化它。这使得```MyLocationListener```类能够自给自足，这意味着对生命周期状态变化作出反应的逻辑是在```MyLocationListener```中声明的，而不是在activity中声明的。让各个组件存储它们自己的逻辑，使得activity和fragment逻辑更容易管理。

{% highlight java %}
class MyActivity extends AppCompatActivity {
    private MyLocationListener myLocationListener;

    public void onCreate(...) {
        myLocationListener = new MyLocationListener(this, getLifecycle(), location -> {
            // update UI
        });
        Util.checkUserStatus(result -> {
            if (result) {
                myLocationListener.enable();
            }
        });
  }
}
{% endhighlight %}

一个常见的用例是，如果```Lifecycle```现在不处于良好状态，则避免调用某些回调。例如，如果回调函数在activity state已经存储之后，运行了一个fragment事务，那么它将出发一个一个Crash,所以我们永远不要调用那个回调。

为了使这个用例容易，```Lifecycle```类允许其他对象查询当前状态。

{% highlight java %}
class MyLocationListener implements LifecycleObserver {
    private boolean enabled = false;
    public MyLocationListener(Context context, Lifecycle lifecycle, Callback callback) {
       ...
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    void start() {
        if (enabled) {
           // connect
        }
    }

    public void enable() {
        enabled = true;
        if (lifecycle.getCurrentState().isAtLeast(STARTED)) {
            // connect if not connected
        }
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    void stop() {
        // disconnect if connected
    }
}
{% endhighlight %}

因为实现了LifecycleObserver接口，我们```locationListener```类是完全感知生命周期的。如果我们需要在另一个activity和fragment中使用我们的LocationListener，那么我们只需要初始化它。
所有的setup和teardown操作都是由类本身来管理的。

如果一个库提供了需要与Android生命周期一起工作的类，那么我们建议你使用生命周期可感知的组件。你的库客户端可以轻松地集成这些组件，而无需在客户端进行手动生命周期管理。

### 实现一个自定义的LifecycleOwner

在Support Libaray 26.1.0和最近的版本已经中，Fragments和Activities已经实现了```LifecycleOwner```接口。

如果你有一个自定义的类，你想要创建一个LifecycleOwner，你可以使用```LifecycleRegistry```类，但是你需要将事件转发到该类中，如下面的代码示例所示：

{% highlight java %}
public class MyActivity extends Activity implements LifecycleOwner {
    private LifecycleRegistry mLifecycleRegistry;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        mLifecycleRegistry = new LifecycleRegistry(this);
        mLifecycleRegistry.markState(Lifecycle.State.CREATED);
    }

    @Override
    public void onStart() {
        super.onStart();
        mLifecycleRegistry.markState(Lifecycle.State.STARTED);
    }

    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
}
{% endhighlight %}

## 使用生命周期可感知组件的最佳实践

* 保持你的UI controllers(activities和fragments)尽可能扁平化。它们应该不会尝试去获得它们自己的数据；使用ViewModel来做到这一点，并观察LiveData对象，以反映视图的变化。
* 试着去写数据驱动的UI。当数据发生变化时，你的UI controllers会得到更新；并且它们也会反向地响应用户动作去通知```ViewModel```。
* 将你的数据逻辑卸载你的```ViewModel```类中。```ViewModel```应该作为一个在UI controller和你app剩余部分之间的连接器。但是，要注意，获取数据（例如，从网络）不是ViewModel的责任。
* 使用```Data Binding```去维护一个在你的视图和UI controller之间干净的接口。这允许你使视图更具声明性，并最小化你需要在activities和fragments中编写的更新代码。如果你喜欢用Java编程语言来做这件事，那么使用像```Butter Knife```这样的库来避免样板代码，并有一个更好的抽象。
* 如果你的UI是复杂的，可以考虑创建一个presenter类去处理UI更改。即使这可能是一项费力的任务，但是它可以使您的UI组件更容易测试。
* 避免在你的```ViewModel```中引用View或者Activity context。如果```ViewModel```超过了activity（在配置更改的情况下），你的activity就会泄漏，并且不会被垃圾收集器正确处理。

## 使用可感知生命周期组件的例子

可感知生命周期组件能让你在多种情形下更容易地管理生命周期。几个例子如下：
* 在粗粒度和细粒度的位置更新之间切换。使用生命周期感知的组件来支持细粒度的位置更新，而你的位置应用是可见的，当应用在后台时切换到粗粒度更新。LiveData是一个生命周期感知的组件，它允许你的应用在用户改变位置时自动更新UI。
* 停止和启动视频缓冲。使用生命周期感知的组件尽快启动视频缓冲，但在应用程序完全启动之前，推迟播放。当应用程序被破坏时，你还可以使用生命周期感知组件来终止缓冲。
* 启动和停止网络连接。使用生命周期感知组件来支持网络数据的实时更新（流），而应用程序处于前台，当应用进入后台时，它也会自动暂停。
* 暂停并恢复动画画。当应用处于后台时，使用生命周期感知的组件来处理暂停的动画绘图，并在应用程序处于前台后恢复绘图。

## Handling on stop events

当一个```Lifecycle```属于```AppCompatActivity```或者```Fragment```时，且AppCompatActivity或者Fragment的```onSaveInstanceState()```被调用时，```Lifecycle```的阶段由```CREATED```变换到```ON_STOP```的事件被分发。

当通过```onSaveInstanceState()```方法存储一个Fragment或者AppCompatActivity的阶段，直到ON_START被调用时，UI被认为是不可变的。试图在状态被保存后修改UI可能会导致应用程序的导航状态不一致，这就是为什么当应用程序在保存状态后运行fragment事务时，FragmentManager会抛出异常。有关详细信息,请参阅[commit()](https://developer.android.com/reference/android/support/v4/app/FragmentTransaction.html)。

```LiveData```通过避免调用它的观察者，如果观察者的相关```Lifecycle```至少STARTED的话，就可以避免这个边缘情况的出现。在幕后，决定调用它的观察者之前，调用isAtLeast()。

不幸的是，在```onSaveInstanceState()```方法之后调用```AppcompatActivity```的```onStop()```方法，这就留下了一个空白，UI状态的改变是不允许的，但是```Lifecycle```还没有被移动到```CREATED```的状态。

为了避免这个问题，```Lifecycle```类的<b>beta2</b>或者更低的版本没有分发event但标记了CREATED阶段，因此任何检查现在阶段的代码都能得到真实的值，即使直到系统调用onStop()时event都没被分发。

不幸的是，这个解决方法有两个主要的问题：

* 在API level 23或者更低的Android系统真实地保存一个activity的状态，即使这个activity被另一个activity局部覆盖。换句话说，虽然Android系统调用了```onSaveInstanceState()```，但是调用onStop()是不必要的。这就产生了一个潜在的长时间间隔，观察者仍然认为生命周期是活动的，即使它的UI状态不能被修改。
* 任何想要向LiveData类公开类似行为的类都必须实现由```Lifecycle```版本beta 2和更低版本提供的解决方案。

> 注意：为了使这个流程更简单，并提供更好的与旧版本的兼容性，起始版本是<b>1.0.0.-rc1</b>，```Lifecycle```对象被标记为```CREATED```和```ON_STOP```时会被分发，当没有等到onStop()方法被调用就调用```onSaveInstanceState()```时。虽然这不太可能影响你的代码，但是你需要意识到它会在API level 26或者更低版本的Activity类中时不匹配的调用。

## 更多阅读

尝试lifecycle组件 [codelab](https://codelabs.developers.google.com/codelabs/android-lifecycles/#0)。

Lifecycle-aware 组件是[Android Jetpack](https://developer.android.com/jetpack/)的一部分。看有关他们更多的使用在[Sunflower](https://github.com/googlesamples/android-sunflower)demo app中。

{% highlight java %}
{% endhighlight %}

{% highlight java %}
{% endhighlight %}

{% highlight java %}
{% endhighlight %}

{% highlight java %}
{% endhighlight %}

{% highlight java %}
{% endhighlight %}

{% highlight java %}
{% endhighlight %}

{% highlight java %}
{% endhighlight %}

{% highlight java %}
{% endhighlight %}
