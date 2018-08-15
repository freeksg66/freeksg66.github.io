---
layout:     post
title:      APP架构指南
subtitle:   Android开发文档翻译
date:       2018-08-13
author:     GL
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - Android
---

## app开发者普遍面临的问题

在大多数情况下，APP开发不同于桌面程序开发，桌面程序通常有单一的入口，并且跑在独立进程中，Android apps有更多更复杂的入口。一个典型的Android app由多样的app组件构造，包括activities，fragments，services，content providers和broadcase receivers。

多数的这些app组件被声明在app的Manifest文件中，Android系统使用这个这个manifest文件去决定怎样去集成你的app到所有的用户体验中在他们的设备上。然而，正如前面的叙述的，一个桌面app时泡在一个独立进程中的，当用户在他们的设备上通过不同的应用程序，不断地切换流和任务时，一个正确编写的Android应用需要更加灵活。

举个例子，思考如下情形，当你分享照片在你最喜爱的社交网络app中。这个app触发了一个camera intent，这个intent从Android系统中启动了一个camera app去处理这个请求。此时，虽然用户离开了社交网络app，但体验却是无缝的。反过来，这个camera app也许会触发其他的intent，比如启动相册，这也许会启动其他的app。最终这个用户又回到了社交网络app并且分享了这个图片。或者，在这个进程的任何一个时间点上，这个用户也许被一个电话打断，并且在打完电话后继续回去分享图片。

在Android中，这个app跳跃行为很常见，因此你的app必须正确的处理这些flows。请记住，移动设备受资源限制，在任何时候，操作系统也许会为了创建一个新的app杀死一些app。

其中的重点是应用程序组件可能会被单独和无序的启动，并且可能会被用户或系统在任何时候销毁。因为应用程序组件是短暂的，并且其生命周期（什么时候被创建和销毁）不受你控制，所以不应该在应用程序组件中存储任何应用数据或状态，同时应用程序组件不应该相互依赖。

## 常见的架构原则

如果你不能使用app组件去存储app数据和状态，那应该怎么构建app呢？

你要注意的最重要的事情就是你的app中[separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns)。将你的所有代码写进activity或者fragment是一个常见的错误。任何不是处理 UI 或 操作系统交互的代码都不应该在这些类中。保持它们尽可能的精简可以避免许多与生命周期有关的问题。不要忘记你不拥有这些类，它们只是体现了 OS 和 应用之间协议的粘合类。Android OS 可能会因为用户交互或其他因素（如低内存）的原因在任何时候销毁它们。最好尽量减少对它们的依赖以提供一个稳固的用户体验。

第二个重要的原则是应该用 Model 驱动 UI，最好是持久化的 Model。持久化是最佳的原因有两个：一是如果 OS 销毁应用释放资源，用户不用担心丢失数据；二是即使网络连接不可靠或者是断开的，应用仍将继续运行。Model 是负责处理应用数据的组件。Modle 独立于应用中的 View 和应用程序组件，因此 Model 和这些组件的生命周期问题隔离开了。保持 UI 代码精简并且摒除应用的逻辑使其更易于管理。基于 Model 类构建的应用程序其管理数据的职责明确，使应用程序可测试并且稳定。

推荐的App架构

在本节中，我们将通过一个例子展示怎样使用架构组件构建一个app。

> 注意：不可能有一种应用程序的编写方式对于每种情况都是最好的。话虽如此，这个推荐的架构应该是大多数用例的良好起点。如果你已经有一种很好的应用程序编写方式则不需要改变。

假设我们正在构建一个展示用户信息的UI。这个用户信息是使用一个REST API从我们私有的后端获取的。

### 构建用户接口

这个UI由一个fragment（UserProfileFragment.java）和它对应的layout文件（user_profile_layout.xml）组成。

为了驱动UI，我们的数据模型需要持有两个数据元素。

* User ID:用户唯一标识。最好使用 fragment 的参数将用户 ID 传到 fragment 中。如果 Android OS 销毁进程，该 ID 将会被保存，以便下次应用重启时该 ID 可用。
* User Object:一个POJO对象持有用户数据。

我们将基于VideModel类创建一个UserProfileViewModel去持有这个信息。

一个ViewModel为一个特定的UI组件提供了数据，比如一个fragment或者activity，并且处理与业务数据处理之间的通信，比如调用其他的组件去加载数据或者转发用户修改。这个ViewModel不必知道View，并且不受配置改变（比如旋转造成的activity重新创建）的影响。

现在我们有三个文件

* user_profile.xml：屏幕上的UI定义
* UserProfileViewMode.java：为了UI准备数据的类
* UserProfileFragment.java：展示ViewModel中数据和响应用户交互的UI controller


下边是我们最初的实现（简单起见省略布局文件）

{% highlight java %}
public class UserProfileViewModel extends ViewModel {
    private String userId;
    private User user;

    public void init(String userId) {
        this.userId = userId;
    }
    public User getUser() {
        return user;
    }
}
{% endhighlight %}

{% highlight java %}
public class UserProfileFragment extends Fragment {
    private static final String UID_KEY = "uid";
    private UserProfileViewModel viewModel;

    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        String userId = getArguments().getString(UID_KEY);
        viewModel = ViewModelProviders.of(this).get(UserProfileViewModel.class);
        viewModel.init(userId);
    }

    @Override
    public View onCreateView(LayoutInflater inflater,
                @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        return inflater.inflate(R.layout.user_profile, container, false);
    }
}
{% endhighlight %}

现在我们有了三个代码块，我们怎样连接它们呢？毕竟，当ViewModel的user field被设定后，我们需要一个方法去通知UI。这里我们就要引入LiveData了。

> [LiveData](https://developer.android.google.cn/jetpack/docs/livedata.html)是一个可观察的数据持有者。它允许应用程序中的组件观察[LiveData](https://developer.android.com/topic/libraries/architecture/livedata.html)进行改变，而不会在组件之间创建显示的，固定的依赖。另外，LiveData 还遵守应用程序组件（如：activity，fragment，service）的生命周期状态，并且防止对象泄漏使应用不会消耗更多的内存。

> 注：如果你已经再使用像[RxJava](https://github.com/ReactiveX/RxJava)或[Agera](https://github.com/google/agera)的库，你可以继续使用它们而不用换成 LiveData。但是当使用它们或其它的方式时，请确保正确处理生命周期，如：当相关 LifecycleOwner 停止时暂停数据流或在 LifecycleOwner 被销毁时销毁数据流。可以添加 android.arch.lifecycle:reactivestreams 工具，和其它的响应流库（如：RxJava2）一起使用 LiveData。

现在我们就可以通过使用LiveData<User>在UserProfileViewModel中取代User filed，这样fragment就能在数据发生更新时被及时通知。这个LiveData的好东西是处于生命周期感知的(lifecycle aware)，并且在它们不再被需要时自动地清除引用。

{% highlight java %}
public class UserProfileViewModel extends ViewModel {
    ...
    <del>private User user;</del>
    <table><tr><td bgcolor=yello>private LiveData<User> user;</td></tr></table>
    public LiveData<User> getUser() {
        return user;
    }
}
{% endhighlight %}

现在我们可以更改UserProfileFragment去观察数据并且更新UI。

{% highlight java %}
@Override
public void onActivityCreated(@Nullable Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);
    viewModel.getUser().observe(this, user -> {
      // update UI
    });
}
{% endhighlight %}

用户数据每次更新时，[onChanged](https://developer.android.google.cn/reference/android/arch/lifecycle/Observer#onChanged(T))回调函数将被调用，并且UI将被更新。

如果你熟悉其它库的可观察回调的使用，你也许已经认识到，我们不必override fragment的onStop()方法去停止数据的观察。这对于LiveData是不必要的，因为它时生命周期可感知的，这意味着它将不会调用这个回调函数，除非fragment在一个active状态（收到了onStart()但是没有接收到onStop()）。当fragment接收到onDestroy()时，LiveData也将自动地移除这个观察者。

我们也不会做任何特殊的事情去处理配置改变（比如，当屏幕旋转时）。当配置发生改变时，这个ViewModel会自动的存储。因此，当新的fragment被创建时，它将接收到相同的ViewModel实例，并且回调将立即被当前数据调用。这就是为什么ViewModel不应该直接持有View引用的原因。ViewModel的生命周期可以比View的生命周期更长。详见[ViewModel的生命周期](https://developer.android.google.cn/topic/libraries/architecture/viewmodel#the_lifecycle_of_a_viewmodel)。

### 获取数据

虽然现在我们已经连接了ViewModel和fragment，但是怎样让ViewModel获取数据呢？在这个例子中，我们假定我们后端提供一个REST API。我们将使用[Retrofit](http://square.github.io/retrofit/)库去访问我们后端。

这里是我们的retrofit WebService与我们的后端通信：

{% highlight java %}
public interface Webservice {
    /**
     * @GET declares an HTTP GET request
     * @Path("user") annotation on the userId parameter marks it as a
     * replacement for the {user} placeholder in the @GET path
     */
    @GET("/users/{user}")
    Call<User> getUser(@Path("user") String userId);
}
{% endhighlight %}

实现ViewModel的第一个想法也许是直接调用WebService去获取数据并且分配给user对象。即使它能够工作，你的app会随着代码的增长越来越难维护。因为你给了ViewModel类太多的责任，这违背了“关注点分离”的原则。另外，一个ViewModel的作用域与Activity或者Fragment一致的生命周期相同，这会造成当生命周期finished时所有的数据都会丢失，这是一个非常差的用户体验。与此相反，我们的ViewModel将委派这个工作在一个新的Repository module中。

> Repository modules负责处理数据的操作。它们为app的其他部分将提供一个干净的API。它们既知道从哪里获得数据也知道调用什么API去使数据更新。你可以将它们想成不同数据源（持久化的model，web service，cache等）之间的中间体。

UserRepository类按如下方式使用WebService去获取用户数据：

{% highlight java %}
public class UserRepository {
    private Webservice webservice;
    // ...
    public LiveData<User> getUser(int userId) {
        // This is not an optimal implementation, we'll fix it below
        final MutableLiveData<User> data = new MutableLiveData<>();
        webservice.getUser(userId).enqueue(new Callback<User>() {
            @Override
            public void onResponse(Call<User> call, Response<User> response) {
                // error case is left out for brevity
                data.setValue(response.body());
            }
        });
        return data;
    }
}
{% endhighlight %}

虽然 Repository 模块看起来是不必要的，但是它起着一个重要的作用；它抽象了应用程序其它部分的数据源。现在 ViewModel 不知道数据是由 Webservice 获取的，这意味着可以根据需求将其切换为其它实现。

> 注意：为了简单起见，我们忽略了网络错误的情况。有关于暴露错误和加载状态的可选实现方式，请参阅[附录：暴露网络状态](https://developer.android.google.cn/jetpack/docs/guide#addendum)。

### 管理组件间的依赖

上面的UserRepository类需要一个WebService实例去做这项工作。虽然可以简化它，但是如果去做的话，需要去知道WebService类的依赖去创建它。
这将会显著的使代码复杂和重复（例如：需要 Webservice 实例的每个类都需要知道如何使用它的依赖来构造它）。另外，UserRepostory 可能不是唯一需要 Webservice 的类。如果每个类都创建一个新的Webservice，这将会造成非常大的资源负担。

有两种模式可以解决这个问题：

* [依赖注入](https://en.wikipedia.org/wiki/Dependency_injection)：依赖注入允许类定义其依赖而不构造它们。在运行时，另一个类负责提供这些依赖。推荐使用Google的[Dagger 2](https://google.github.io/dagger/)库在 Android应用中实现依赖注入。Dagger 2通过遍历依赖关系树自动构建对象并为依赖提供编译时保障。
* [服务定位](https://en.wikipedia.org/wiki/Service_locator_pattern)：服务定位提供了一个注册表，类可以从中获取它们的依赖关系，而不是构造它们。与依赖注入（DI）相比，服务定位实现起来相对容易，所以如果不熟悉 DI，请使用服务定位代替。

这些模式允许你扩展代码，因为它们提供清晰的模式来管理依赖关系，而不会重复代码或增加复杂性。两者都允许替换实现进行测试；这是使用它们的主要好处之一。

在这个例子中，我们将使用 Dagger 2 来管理依赖。

### 连接ViewModel和repository

现在我们使用这个repository更改USerProfileViewModel。

{% highlight java %}
public class UserProfileViewModel extends ViewModel {
    private LiveData<User> user;
    private UserRepository userRepo;

    @Inject // UserRepository parameter is provided by Dagger 2
    public UserProfileViewModel(UserRepository userRepo) {
        this.userRepo = userRepo;
    }

    public void init(String userId) {
        if (this.user != null) {
            // ViewModel is created per Fragment so
            // we know the userId won't change
            return;
        }
        user = userRepo.getUser(userId);
    }

    public LiveData<User> getUser() {
        return this.user;
    }
}
{% endhighlight %}

### 缓存数据

虽然上面实现的repository对网络服务的调用进行了很好的抽象，但是因为它仅依赖一个数据源，所以它不是很实用。

上面实现的UserRepository存在的问题是，在获取数据之后，它不会在任何地方都持有这个对象。如果用户离开UserProfileFragment并且再返回时，我们的app又重新获取了数据。这有两个不好的因素：

1. 浪费了宝贵的网络带宽
2. 迫使用户等待新的查询完成

为了解决这个问题，我们将在 UserRepository 中添加一个新的数据源，用以在内存中缓存 User 对象。

{% highlight java %}
@Singleton  // informs Dagger that this class should be constructed once
public class UserRepository {
    private Webservice webservice;
    // simple in memory cache, details omitted for brevity
    private UserCache userCache;
    public LiveData<User> getUser(String userId) {
        LiveData<User> cached = userCache.get(userId);
        if (cached != null) {
            return cached;
        }

        final MutableLiveData<User> data = new MutableLiveData<>();
        userCache.put(userId, data);
        // this is still suboptimal but better than before.
        // a complete implementation must also handle the error cases.
        webservice.getUser(userId).enqueue(new Callback<User>() {
            @Override
            public void onResponse(Call<User> call, Response<User> response) {
                data.setValue(response.body());
            }
        });
        return data;
    }
}
{% endhighlight %}

数据持久化

在我们现在的实现中，如果用户旋转了屏幕或者离开又返回了app，已经存在的UI将立即可见，因为repository从内存缓存中恢复了数据。但是如果用户离开了app又在几个小时后返回或者Android系统已经杀死了进程，这其中又发生了什么？

根据现在的实现，我们将需要再次从网络中获取数据。这既是一个不好的用户体验，也是一种浪费的行为，因为重新获取了相同的网络数据。虽然你可以通过缓存网络请求来简单地修复这个问题，但是又有新的问题出现了。如果相同的用户数据从另一种类型的请求中展示（例如，获取朋友列表）呢？那么我们的数据可能展示不一样的数据，这就造成了用户体验的困惑。例如，相同的用户数据可能不同的被展示，因为朋友列表请求和用户请求再不同的时间被执行。你的app需要合并他们，为了回避展示不一致的数据。

比较合理的方式去处理这个问题时使用model持久化。这就是持久化库 [Room](https://developer.android.google.cn/training/data-storage/room/)的用武之地了。

> [Room](https://developer.android.google.cn/training/data-storage/room/)是一个以最少的样板代码提供本地数据持久化的对象映射库。在编译时，它会根据模式验证每个查询，所以损坏的 SQL 查询只会导致编译时错误，而不是运行时崩溃。Room 抽象出一些使用原始 SQL 表查询的底层实现细节。它还允许观察数据库数据（包括集合和连接查询）的变化，并通过<b>LiveData</b>对象暴露这些变化。另外，它明确定义了线程约束以解决常见问题（如在主线程访问存储）。

> 注意：如果你熟悉其它的持久化解决方案，如：SQLite ORM 或像 Realm 等其他的数据库，你不需要用更换为 Room，除非 Room 的功能对你的用例更加适用。

使用 Room，需要定义我们的局部模式。首先，用 @Entity 注释 User 类，将其标记为数据库中的一个表。

{% highlight java %}
@Entity
class User {
  @PrimaryKey
  private int id;
  private String name;
  private String lastName;
  // getters and setters for fields
}
{% endhighlight %}

然后，通过继承[RoomDatabase](https://developer.android.google.cn/reference/android/arch/persistence/room/RoomDatabase)为应用创建一个数据库。

{% highlight java %}
@Database(entities = {User.class}, version = 1)
public abstract class MyDatabase extends RoomDatabase {
}
{% endhighlight %}

注意，MyDatabase是一抽象的。Room自动地提供了一个关于它的实现。细节详见[Room](https://developer.android.google.cn/training/data-storage/room/)文档。

现在我们需要一个方法去插入用户数据到数据库中。我们将创建一个[data access object (DAO)](https://en.wikipedia.org/wiki/Data_access_object)。

{% highlight java %}
@Dao
public interface UserDao {
    @Insert(onConflict = REPLACE)
    void save(User user);
    @Query("SELECT * FROM user WHERE id = :userId")
    LiveData<User> load(String userId);
}
{% endhighlight %}

然后从我们的数据库类中引用DAO

{% highlight java %}
@Database(entities = {User.class}, version = 1)
public abstract class MyDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}
{% endhighlight %}

注意，load方法返回了一个LiveData<User>。Room知道数据库合适被改变并且当数据改变时它将自动地通知所有的active观察者。因为它使用LiveData，这是很有效的，因为它只在至少有一个active观察者时才会更新数据。

> 注意：Room基于表修改的检查无效，这意味着它可能会发送错误的通知。

现在修改 UserRepository 来整合 Room 的数据源。

{% highlight java %}
@Singleton
public class UserRepository {
    private final Webservice webservice;
    private final UserDao userDao;
    private final Executor executor;

    @Inject
    public UserRepository(Webservice webservice, UserDao userDao, Executor executor) {
        this.webservice = webservice;
        this.userDao = userDao;
        this.executor = executor;
    }

    public LiveData<User> getUser(String userId) {
        refreshUser(userId);
        // return a LiveData directly from the database.
        return userDao.load(userId);
    }

    private void refreshUser(final String userId) {
        executor.execute(() -> {
            // running in a background thread
            // check if user was fetched recently
            boolean userExists = userDao.hasUser(FRESH_TIMEOUT);
            if (!userExists) {
                // refresh the data
                Response response = webservice.getUser(userId).execute();
                // TODO check for error etc.
                // Update the database.The LiveData will automatically refresh so
                // we don't need to do anything else here besides updating the database
                userDao.save(response.body());
            }
        });
    }
}
{% endhighlight %}

请注意，即使我们更改了 UserRepository 中的数据来源，我们也不需要更改 UserProfileViewModel 或 UserProfileFragment。这是抽象带来的灵活性。这样也非常易于测试，因为在测试 UserProfileViewModel 可以提供一个假的 UserRepository。

现在我们的代码是完整的，如果用户日后再回到相同的 UI，他们会立即看到用户信息，因为我们已经将其持久化了。同时，如果数据过期，Repository 将会在后台更新数据。当然，根据你的用例，如果持久化的数据太旧你可能不希望显示它们。

在一些用例中，如下拉刷新，在进行网络操作时显示用户数据对于 UI 来说非常重要。将 UI 操作从实际数据中分离是一个很好的做法，因为其可能由于各种原因而被更新（例如：如果获取一个朋友列表，可能会再次获取到相同的 user 并触发 LiveData<User> 更新）。从 UI 的角度来看，动态请求实际上只是另一个数据点，类似其它任何数据片段（如：User 对象）

这种用例有两种常见的解决方案：

* 更改getUser以返回包含网络操作状态的 LiveData。在[附录：暴露网络状态](https://developer.android.google.cn/jetpack/docs/guide#addendum)部分提供了一个实现例子。
* 在 Repository 类中提供一个可以返回 User 刷新状态的公共方法。如果只是为了响应明确的用户操作而在 UI 中显示网络状态（如：下拉刷新），则这种方式是更好的选择。

### 单一信任的数据源

不同的 REST API 接口返回相同的数据是很常见的。例如，如果后端有另一个返回朋友列表的接口，相同的用户对象（也许是不同的粒度）可能来自两个不同的 API 接口。如果 UserRepository 把从 Webservice 请求获取到的响应原样返回，码么 UI 可能会显示不一致的数据，因为在这些请求之间数据可能在服务端发生了改变。这就是为什么在 UserRepository 的实现中 Web 服务的回调只是将数据保存到了数据库。然后，对数据库的更改将会触发处于活动状态的 LiveData 对象上的回调。

在这个模型中，数据库服务作为单一数据源，应用程序的其它部分通过 Repository 来访问它。无论是否是否磁盘缓存，建议 Repository 指定一个数据源作为应用程序其它部分的单一数据源。

### 测试

我们已经提到分离的好处之一是可测试性。让我们看看如何测试每个代码模块。

* <b>用户界面或用户交互</b>：这将是唯一一次需要[Android UI Instrumentation test](https://developer.android.google.cn/training/testing/unit-testing/instrumented-unit-tests)。测试 UI 代码的最佳方式是创建一个[Espresso](https://developer.android.google.cn/training/testing/ui-testing/espresso-testing)测试。可以创建一个 fragment 并为其提供一个模拟的 ViewModel。因为 fragment 只和 ViewModel 交互，随意模拟 ViewModel 足以完全测是 UI。
* <b>ViewModel</b>：可以使用[JUnit test](https://developer.android.google.cn/training/testing/unit-testing/local-unit-tests)来测试ViewModel。只需要模拟 UserRepository来测试它。
* <b>UserRepository</b>：也可以使用[JUnit test](https://developer.android.google.cn/training/testing/unit-testing/local-unit-tests)来测试UserRepository。需要模拟 WebService和DAO。可以测试UserRepository是否进行了正确的 Web 服务调用，将结构保存到数据库，如果数据被缓存且是最新的，则不会发起任何不必要的请求。因为 WebService 和 UserDao 都是接口，所以可以模拟它们或者为更复杂的测试用例创建假的实现。
* UserDao：推荐使用 Instrumentation 测试的方式测试 DAO 类。因为 Instrumentation 测试不需要任何 UI，它们会运行的很快。对于每个测试，可以创建一个内存数据库以确保测试没有任何副作用（例如：改变磁盘上的数据库文件）。

Room 还允许指定数据库实现，所以可以通过向其提供[SupportSQLiteOpenHelper](https://developer.android.com/reference/android/arch/persistence/db/SupportSQLiteOpenHelper.html)的JUnit实现来测试它。通常不推荐这种方式，因为设备上运行的SQLite版本可能和主机上的SQLite版本不同。

* WebService：重点是使测试相对于外部独立，所以WebService的测试要避免通过网络调用后端。有许多库可以帮助完成该测试。例如：[MockWebServer](https://github.com/square/okhttp/tree/master/mockwebserver)是一个很好的库，可以帮助为测试创建一个假的本地服务。
* 测试工件架构组件提供了一个maven工件来控制其后台线程。在 android.arch.core:core-testing 工件中，有两个JUnit规则：

1)InstantTaskExecutorRule：该规则可用于强制架构组件立即执行调用线程上的任何后台操作。

2)CountingTaskExecutorRule：该规则可用于 Instrumentation 测试，以等待架构组件的后台操作，或将其连接到 Espresso 作为闲置资源。

### 最终的架构

![](http://pbmurxnd0.bkt.clouddn.com/final-architecture.png)

### 指导原则

编程是一个创作领域，构建 Android 应用也不例外。有许多方法来解决问题，无论是在多个 activity 或 fragment 之间传递数据，是获取远程数据并为了离线模式将其持久化到本地，还是特殊应用遭遇的其它常见情况。

虽然一下建议不是强制性的，但是以我们的经验，从长远来看，遵循这些建议将会使代码库更健壮，易测试和易维护。

* 在 manifest 中定义的入口点，如：acitivy，fragment，broadcast receiver 等，不是数据源。相反，它们应该只是协调与该入口点相关的数据子集。由于每个应用程序组件的存活时间很短，这取决于用户与其设备的交互以及运行时的总体状况，所以任何入口点都不应该成为数据源
* 严格的在应用程序的各个模块之间创建明确的责任界限。例如：不在代码库中的多个类或包中扩散从网络加载数据的代码。同样，不要将无关的责任（如：数据缓存和数据绑定）放到同一个类中。
* 每个模块尽可能少的暴露出来。不要视图创建暴露模块内部实现细节的“只一个”的快捷方式。你可能会在短期内节省一些时间，但是随着代码库的发展，你将会多次偿还更多的基数债务。
* 当定义模块间的交互时，请考虑如何让每个模块可以独立的测试。例如，拥有一个用于从网络获取数据且定义良好的 API 的模块，将会使其更易于测试在本地数据库中持久化数据。相反，如果将两个模块的逻辑放在一个地方，或者将网络代码扩散到整个代码库，测试将会变的非常困难（并非不可能）。
* 应用程序的核心是使其脱颖而出。不要花费时间重复造轮子或一次又一次的编写相同的样板代码。相反，将精力集中在使应用程序独一无二上，让 Android Architecture Components 和其它的优秀的库来处理重复的样板代码。
* 持久化尽可能多的相关最新数据，以便应用程序在设备处于离线模式时还可以使用。即使你可以享用稳定高速的网络连接，但是你的用户可能无法享用。
* Repository 应该指定一个数据源作为单一数据源。每当应用程序需要访问数据时，数据应该始终来源于单一数据源。有关更多信息，请参阅[单一数据源](https://developer.android.google.cn/jetpack/docs/guide#truth)

### 附录：暴露网络状态

在上面推荐的应用程序架构部分，为了保持示例简单我们故意忽略网络错误和加载状态。在本节中，我们演示一种通过Resource类暴露网络状态来封装数据和其状态。

以下是一个实现的例子：

{% highlight java %}
//a generic class that describes a data with a status
public class Resource<T> {
    @NonNull public final Status status;
    @Nullable public final T data;
    @Nullable public final String message;
    private Resource(@NonNull Status status, @Nullable T data, @Nullable String message) {
        this.status = status;
        this.data = data;
        this.message = message;
    }

    public static <T> Resource<T> success(@NonNull T data) {
        return new Resource<>(SUCCESS, data, null);
    }

    public static <T> Resource<T> error(String msg, @Nullable T data) {
        return new Resource<>(ERROR, data, msg);
    }

    public static <T> Resource<T> loading(@Nullable T data) {
        return new Resource<>(LOADING, data, null);
    }
}
{% endhighlight %}

因为从磁盘中获取并显示数据同时再从网络获取数据是一种常见的用例。我们将创建一个可以在多个地方使用的帮助类 NetworkBoundResource。下面是 NetworkBoundResource 的决策树。

![](http://pbmurxnd0.bkt.clouddn.com/network-bound-resource.png)

它通过观察资源的数据库。当首次从数据库加载条目时，NetworkBoundResource 检查返回结果是否足够好可以被发送和（或）应该从网络获取数据。请注意，他们可能同时发生，因为你可能会希望在显示缓存数据的同时从网络更新数据。

如果网络调用成功，则将返回数据保存到数据库中并重新初始化数据流。如果网络请求失败，直接发送一个错误。

> 注：将新的数据保存到磁盘后，要从数据库重新初始化数据流，但是通常不需要这样做，因为数据库将会发送变更。另一方面，依赖数据库发送变更会有一些不好的副作用，因为在数据没有变化时如果数据库会避免发送更改将会使其中断。我们也不希望发送从网络返回的结果，因为这违背的单一数据源原则（即使在数据库中有触发器会改变保存值）。我们也不希望在没有新数据的时候发送 SUCCESS，因为这会给客户端发送错误信息。

以下是 NetworkBoundResource 类为其子类提供的公共 API：

{% highlight java %}
// ResultType: Type for the Resource data
// RequestType: Type for the API response
public abstract class NetworkBoundResource<ResultType, RequestType> {
    // Called to save the result of the API response into the database
    @WorkerThread
    protected abstract void saveCallResult(@NonNull RequestType item);

    // Called with the data in the database to decide whether it should be
    // fetched from the network.
    @MainThread
    protected abstract boolean shouldFetch(@Nullable ResultType data);

    // Called to get the cached data from the database
    @NonNull @MainThread
    protected abstract LiveData<ResultType> loadFromDb();

    // Called to create the API call.
    @NonNull @MainThread
    protected abstract LiveData<ApiResponse<RequestType>> createCall();

    // Called when the fetch fails. The child class may want to reset components
    // like rate limiter.
    @MainThread
    protected void onFetchFailed() {
    }

    // returns a LiveData that represents the resource, implemented
    // in the base class.
    public final LiveData<Resource<ResultType>> getAsLiveData();
}
{% endhighlight %}

请注意，上述类定义了两个类型参数（ResultType，RequestType），因为从 API 返回的数据类型可能和本地使用的数据类型不同。

还要注意，上述代码使用 ApiResponse 作为网络请求，ApiResponse 是对于 Retrofit2.Call 类的简单封装，用以将其响应转换为 LiveData。

以下是 NetworkBoundResource 类的其余实现部分。

{% highlight java %}
public abstract class NetworkBoundResource<ResultType, RequestType> {
    private final MediatorLiveData<Resource<ResultType>> result = new MediatorLiveData<>();

    @MainThread
    NetworkBoundResource() {
        result.setValue(Resource.loading(null));
        LiveData<ResultType> dbSource = loadFromDb();
        result.addSource(dbSource, data -> {
            result.removeSource(dbSource);
            if (shouldFetch(data)) {
                fetchFromNetwork(dbSource);
            } else {
                result.addSource(dbSource,
                        newData -> result.setValue(Resource.success(newData)));
            }
        });
    }

    private void fetchFromNetwork(final LiveData<ResultType> dbSource) {
        LiveData<ApiResponse<RequestType>> apiResponse = createCall();
        // we re-attach dbSource as a new source,
        // it will dispatch its latest value quickly
        result.addSource(dbSource,
                newData -> result.setValue(Resource.loading(newData)));
        result.addSource(apiResponse, response -> {
            result.removeSource(apiResponse);
            result.removeSource(dbSource);
            //noinspection ConstantConditions
            if (response.isSuccessful()) {
                saveResultAndReInit(response);
            } else {
                onFetchFailed();
                result.addSource(dbSource,
                        newData -> result.setValue(
                                Resource.error(response.errorMessage, newData)));
            }
        });
    }

    @MainThread
    private void saveResultAndReInit(ApiResponse<RequestType> response) {
        new AsyncTask<Void, Void, Void>() {

            @Override
            protected Void doInBackground(Void... voids) {
                saveCallResult(response.body);
                return null;
            }

            @Override
            protected void onPostExecute(Void aVoid) {
                // we specially request a new live data,
                // otherwise we will get immediately last cached value,
                // which may not be updated with latest results received from network.
                result.addSource(loadFromDb(),
                        newData -> result.setValue(Resource.success(newData)));
            }
        }.execute();
    }

    public final LiveData<Resource<ResultType>> getAsLiveData() {
        return result;
    }
}
{% endhighlight %}

现在，可以使用 NetworkBoundResource 在 Repository 中编写磁盘和网络绑定 User 的实现。

{% highlight java %}
class UserRepository {
    Webservice webservice;
    UserDao userDao;

    public LiveData<Resource<User>> loadUser(final String userId) {
        return new NetworkBoundResource<User,User>() {
            @Override
            protected void saveCallResult(@NonNull User item) {
                userDao.insert(item);
            }

            @Override
            protected boolean shouldFetch(@Nullable User data) {
                return rateLimiter.canFetch(userId) && (data == null || !isFresh(data));
            }

            @NonNull @Override
            protected LiveData<User> loadFromDb() {
                return userDao.load(userId);
            }

            @NonNull @Override
            protected LiveData<ApiResponse<User>> createCall() {
                return webservice.getUser(userId);
            }
        }.getAsLiveData();
    }
}
{% endhighlight %}
