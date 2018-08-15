---
layout:     post
title:      ViewModel概述
subtitle:   翻译Android开发文档
date:       2018-08-13
author:     GL
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - Android
---

<b>ViewModel</b>类是用来以一种生命周期方式存储和管理在UI相关数据。<b>ViewModel</b>类允许在发生改变时（比如屏幕旋转）保留数据。

> 注意：如果你想引入ViewModel在你的Android Project中，请参见[adding components to your project](https://developer.android.google.cn/topic/libraries/architecture/adding-components#lifecycle)

Android framework管理UI controllers的生命周期，比如activities和fragments，这个framework响应用户动作或者超出你控制的设备事件也许决定去destroy或者recreate一个UI controller。

如果系统detroy或者recreate一个UIcontroller，任何短暂的你存储的与UI相关的数据也许会失去。例如，你的app也许在一个activity中包含了一个用户列表。当activity的状态发生改变recreate时，新的activity不得不re-fetch原用户列表。对于简单的数据，activity能调用onSaveInstanceState()方法并且从Bundle中恢复数据在onCreate()方法中。但是这种方式只适用于恢复能被序列化和反序列化的小规模数据，不能用于可能的大规模数据（比如，用户列表或者bitmaps）。

另一个问题是UI controller经常需要异步调用，这个调用也许会经过长时间才能返回。这个UI controller需要管理这些调用并且确保系统能够在这些调用已经destroy后清除它们，为了回避可能的内存泄漏。这种管理需要大量的维护，并且在配置发生改变时object被recreate的情况下，这个object可能不得不重新发出已经发出的调用，这就造成了大量资源的浪费。

UI controller（比如activities和fragments）主要用于显示UI数据，响应用户动作，或者处理系统通信的操作（比如权限请求）。要求UI controller也要负责从数据库或网络中加载数据，从而增加了类的臃肿。将过多的责任分配给UI controller可能会导致单个以这种方式给UI controller分配过多的责任也会使测试变得更加困难。

将视图数据的所有权从UI控制器逻辑中分离出来是一种更容易也更高效的方式。

### 实现一个ViewModel

Android架构组件提供了一个ViewModel类负责帮助UI controller类为UI准备数据。由于<b>ViewModel</b>对象在配置改变时会自动保存，因此它们持有的这些数据在下一个activity或者fragment实例中会立即获得。例如，如果你需要展示一个用户列表在你的app中，请确保将这些数据分配给一个ViewModel，而不是给一个activity或者fragment。可以参考下面的代码：


{% highlight java %}
public class MyViewModel extends ViewModel {
    private MutableLiveData<List<User>> users;
    public LiveData<List<User>> getUsers() {
        if (users == null) {
            users = new MutableLiveData<List<User>>();
            loadUsers();
        }
        return users;
    }

    private void loadUsers() {
        // Do an asynchronous operation to fetch users.
    }
}
{% endhighlight %}

你可以在activity中通过如下方式获取数据列表


{% highlight java %}
public class MyActivity extends AppCompatActivity {
    public void onCreate(Bundle savedInstanceState) {
        // Create a ViewModel the first time the system calls an activity's onCreate() method.
        // Re-created activities receive the same MyViewModel instance created by the first activity.

        MyViewModel model = ViewModelProviders.of(this).get(MyViewModel.class);
        model.getUsers().observe(this, users -> {
            // update UI
        });
    }
}
{% endhighlight %}

如果activity被recreated，它会收到与在第一个activity中创建的相同的MyViewModel实例。当拥有ViewModel的activity finish的时候，这个framework会调用ViewModel对象的onCleared()方法清除相关的资源。

> 注意：一个[ViewModel](https://developer.android.google.cn/reference/android/arch/lifecycle/ViewModel)一定不能有任何view的引用，任何[生命周相关](https://developer.android.google.cn/reference/android/arch/lifecycle/Lifecycle)的引用，任何持有activity context引用类的引用

ViewModel对象中不包括任何特定的view或者[LifecycleOwner](https://developer.android.google.cn/reference/android/arch/lifecycle/LifecycleOwner)的实例，这种设计的好处是你能写一个很容易的测试viewModel，因为它不包括任何View或者LifecycleOwner的对象。

ViewModel能包含[LifecycleObserver](https://developer.android.google.cn/reference/android/arch/lifecycle/LifecycleObserver)(比如，[LiveData](https://developer.android.google.cn/reference/android/arch/lifecycle/LiveData)对象)。然而ViewModel对象必须从未关注lifecycle-aware observables（比如LiveData对象）。如果ViewModel对象需要Application context，比如为了找一个系统服务，这时可以继承AndroidViewModel类并且拥有一个能狗接受[Application](https://developer.android.google.cn/reference/android/app/Application.html)的构造函数，因为Application类是继承Context的。

### ViewModel的生命周期

ViewModel对象的作用域范围是在获得ViewModel时传递给ViewModelProvider的生命周期。ViewModel保留在内存中，直到它的作用域永久消失：在activity finish的情况下,或者在fragment detached的情况下。

下图展示了当一个activity经历了一次旋转并在它finished时的不同生命周期的阶段。这个图也展示了ViewModel的生命时间与其相关的activity的生命周期。虽然这个图是讲activity的生命周期阶段的，但也是同于fragment的的生命周期阶段。

![](http://pbmurxnd0.bkt.clouddn.com/viewmodel-lifecycle.png)

通常，你可以在系统调用一个activity对象的onCreate()方法时首次请求一个ViewModel。系统也许会在一个activity的生命周期中调用onCreate()方法若干次（比如当设备发生旋转时）。这个ViewModel对象会在你第一次请求ViewModel后一直存在，直到这个activity finished或者destroyed。

### 在fragments之间共享数据

在一个activity中有两个或更多的fragment需要进行通信的情景是很普遍的。想象一个普遍的场景，当你在一个fragment中选择列表中的某一项，然后需要在另一个fragment中展示被选择项时。以前的做法是，在两个fragment最终定义相同的接口，并且必须在包含它们的activity中将它们俩绑定。另外，绑定动作必须在另一个fragment没创建或者可见时处理。

我们现在可以通过使用ViewModel来处理这种情况。这些fragment可以共享一个ViewModel，并且在它们的activity作用域中使用这个ViewModel处理相互之间的通信，可以参考下面这个例子：

{% highlight java %}
public class SharedViewModel extends ViewModel {
    private final MutableLiveData<Item> selected = new MutableLiveData<Item>();

    public void select(Item item) {
        selected.setValue(item);
    }

    public LiveData<Item> getSelected() {
        return selected;
    }
}


public class MasterFragment extends Fragment {
    private SharedViewModel model;
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);
        itemSelector.setOnClickListener(item -> {
            model.select(item);
        });
    }
}

public class DetailFragment extends Fragment {
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        SharedViewModel model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);
        model.getSelected().observe(this, item -> {
           // Update the UI.
        });
    }
}
{% endhighlight %}

注意：当在获取ViewModelProvider时，两个fragment都使用getActivity()方法。结果，两个fragment都得到了同一个在activity作用域中的SharedViewModel实例。

上述方法有如下好处：

* 这个activity不需要做任何事情，或者不需要知道任何与通信有关的细节。
* fragment之间不需要知道除了SharedViewModel contract以外的任何事情。如果有一个fragment disappear了，另一个fragment仍可正常工作。
* 每一个fragment有它们自己的生命周期，并且不会被另一个fragment的生命周期所影响，UI也可以继续正常工作没有任何问题。

### 用ViewModel取代Loaders

加载类（比如[CursorLoader](https://developer.android.google.cn/reference/android/content/CursorLoader)）通常被用于在一个app UI中持有与数据库同步的数据。你可以使用ViewModel和几个其他的类去替代这个loader。使用一个ViewModel可以从数据加载操作中分离出你的UI controller，这意味着你降低了类之间的耦合度。

在一个通用的方法中使用loader时，一个app也许使用一个CursorLoader去观察数据库的内容。当数据库的一个value发生改变时，这个loader会自动的出发重新加载数据操作，并且更新UI。过程如下图所示：

![Loading data with loaders](http://pbmurxnd0.bkt.clouddn.com/viewmodel-loader.png)

ViewModel通过与[Room](https://developer.android.google.cn/topic/libraries/architecture/room)和[LiveData](https://developer.android.google.cn/topic/libraries/architecture/livedata)一起工作，来取代loader。这个ViewModel却道数据在设配状态发生改变时不会丢失。当数据库发生改变时，Room会通知你的LiveData，并且LiveData反过来会更新你的UI和改变你的数据。过程如下图所示：

![Loading data with ViewModel](http://pbmurxnd0.bkt.clouddn.com/viewmodel-replace-loader.png)

### 更多阅读

[这篇博客](https://medium.com/google-developers/lifecycle-aware-data-loading-with-android-architecture-components-f95484159de4)介绍了如何使用一个ViewModel去替换一个AsyncTaskLoader。

当你的数据变得更复杂时，你可以选择从一个单独的类中加载数据。ViewModel的目的是封装UI控制器的数据，让数据在配置更改时不会丢失。
有关如何在配置更改中加载、持久化和管理数据的信息，请参阅[保存UI状态](https://developer.android.google.cn/topic/libraries/architecture/saving-states)。

[Android开发指南](https://developer.android.google.cn/jetpack/docs/guide#fetching_data)建议构建一个repository类去处理这些功能。

ViewModel是一个[Android Jetpack](https://developer.android.google.cn/jetpack/)架构组件。在[Sunflower](https://github.com/googlesamples/android-sunflower)demo app中看关于它的使用。
