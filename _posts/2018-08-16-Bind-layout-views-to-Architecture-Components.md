---
layout:     post
title:      绑定布局视图与架构组件
subtitle:   Android开发文档翻译
date:       2018-08-16
author:     GL
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - Android
---

AndroidX库包含了[架构组件](https://developer.android.com/topic/libraries/architecture/index.html)，你可以用它设计出鲁棒的，可测试的和可维护的app。数据绑定库与架构组件无缝协作，以进一步简化UI的开发。在你的app布局文件中能绑定架构组件的数据，它已经帮助你管理UI controllers的生命周期和在数据改变时通知UI controllers。

这篇文章展示了如何将架构组件合并到你的应用程序中，以进一步增强使用数据绑定库的好处。

## 使用LiveData在数据改变时去通知UI

你能使用```LiveData```对象作为数据绑源，在数据发生改变时去自动地通知UI。关于这个架构组件的更多信息请看，[Live Data Overview](https://developer.android.com/topic/libraries/architecture/livedata)。

与实现了```Observable```接口的对象（比如[observable fields](https://developer.android.com/topic/libraries/data-binding/observability.html#observable_fields)）不同，```LiveData```对象知道订阅数据观察者的生命周期。这一点就体现了LiveData的优越性[The advantages of using LiveData](https://developer.android.com/topic/libraries/architecture/livedata.html#the_advantages_of_using_livedata)。在Android Studio3.1或者更高的版本中，你可以在你的数据绑定代码中用```LiveData```去替换```observable fields```。

为了使用LiveData对象在你的绑定类中，你需要指定一个生命周期所有者来定义LiveData对象的范围。

下面的例子指定了当绑定类被实例化后的生命周期所有者：

{% highlight java %}
class ViewModelActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // Inflate view and obtain an instance of the binding class.
        UserBinding binding = DataBindingUtil.setContentView(this, R.layout.user);

        // Specify the current activity as the lifecycle owner.
        binding.setLifecycleOwner(this);
    }
}
{% endhighlight %}

你能使用ViewModel组件（在这里可以看[使用ViewModel管理你的与UI相关的数据](https://developer.android.com/topic/libraries/data-binding/architecture#viewmodel)）去绑定数据到布局文件中。在```ViewModel```组件中，你可以使用```LiveData```对象转化数据或者合并多样的数据源。下面的例子展示了在ViewModel中如何转化数据：

{% highlight java %}
class ScheduleViewModel extends ViewModel {
    LiveData username;

    public ScheduleViewModel() {
        String result = Repository.userName;
        userName = Transformations.map(result, result -> result.value);
}
{% endhighlight %}

### 使用ViewModel管理与UI相关的数据

数据绑定库与ViewModel组件可以无缝的工作，它暴露了布局所观察到的数据并可以对其变化作出反应。使用数据绑定库的```ViewModel```组件允许你将UI逻辑移除布局文件，并且移入控件中，这将有助于你进行测试。数据绑定库确保了在需要时将view与data进行绑定与解绑。剩下的大部分工作都是确保你暴露了正确的数据。更过的关于架构组件的信息，请看[ViewModel Overview](https://developer.android.com/topic/libraries/architecture/viewmodel.html)。

在数据绑定库中使用```ViewModel```组件，你必须初始化你的组件（它继承与```ViewModel```类）获取一个你的绑定类的实例，并且给你的ViewModel组件分配一个绑定类中的属性。下面的代码展示了怎样使用数据绑定库里的这个组件：

{% highlight java %}
class ViewModelActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // Obtain the ViewModel component.
        UserModel userModel = ViewModelProviders.of(getActivity())
                                                  .get(UserModel.class);

        // Inflate view and obtain an instance of the binding class.
        UserBinding binding = DataBindingUtil.setContentView(this, R.layout.user);

        // Assign the component to a property in the binding class.
        binding.viewmodel = userModel;
    }
}
{% endhighlight %}

在你的布局文件中，使用绑定表达式给你ViewModel组件的属性和方法分配对应的视图，如下所示：

{% highlight java %}
<CheckBox
    android:id="@+id/rememberMeCheckBox"
    android:checked="@{viewmodel.rememberMe}"
    android:onCheckedChanged="@{() -> viewmodel.rememberMeChanged()}" />
{% endhighlight %}

## 使用可观察的ViewModel对绑定适配器进行更多控制

在数据发生改变时，你可以使用实现了```Observable```接口的ViewModel组件去通知另一个app组件，当然你也可以使用```LiveData```对象。

有几种情况也许可以让你更倾向于使用实现了Observable接口的ViewModel组件，而不是使用```LiveData```对象，即使你会失去LiveData的管理生命周期的能力。在你的app中，使用实现了Observable接口的ViewModel组件可以让你控制更多的绑定适配器。例如，当数据发生变化时，这种模式使你能够更好地控制通知，它还允许你指定一个定制方法来在双向数据绑定中设置属性的值。

为了实现可观察的ViewModel组件，你必须创建一个继承于ViewModel的类，并且实现Observable接口。当一个观察者订阅或者解订阅时，你可以使用```addOnPropertyChangedCallback()```和```removeOnPropertyChangedCallback()```方法提供你自定义的逻辑去通知。你也可以提供自定义的逻辑，并且在```notifyPropertyChanged()```方法中属性改变时运行。下面的代码展示了怎样实现一个可观察的ViewModel：

{% highlight java %}
/**
 * A ViewModel that is also an Observable,
 * to be used with the Data Binding Library.
 */
class ObservableViewModel extends ViewModel implements Observable {
    private PropertyChangeRegistry callbacks = new PropertyChangeRegistry();

    @Override
    protected void addOnPropertyChangedCallback(
            Observable.OnPropertyChangedCallback callback) {
        callbacks.add(callback);
    }

    @Override
    protected void removeOnPropertyChangedCallback(
            Observable.OnPropertyChangedCallback callback) {
        callbacks.remove(callback);
    }

    /**
     * Notifies observers that all properties of this instance have changed.
     */
    void notifyChange() {
        callbacks.notifyCallbacks(this, 0, null);
    }

    /**
     * Notifies observers that a specific property has changed. The getter for the
     * property that changes should be marked with the @Bindable annotation to
     * generate a field in the BR class to be used as the fieldId parameter.
     *
     * @param fieldId The generated BR id for the Bindable field.
     */
    void notifyPropertyChanged(int fieldId) {
        callbacks.notifyCallbacks(this, fieldId, null);
    }
}
{% endhighlight %}

