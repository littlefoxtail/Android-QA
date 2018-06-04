Android Architecture Components
是一个由官方推出的新库，它能够帮助你去构建一个健壮，易测，可维护的应用

# 架构原则
1. 在应用中的关注点分离。比如在一个Activity或者Fragment中写所有的代码明显是错误的。任何与
UI或者交互无关的代码都不应该存在这些类中。保证他们尽可能的职责单一化将会使我们避免很多生命周期相关的问题。
Android系统可能会随时由于用户的行为或者系统状态(比如剩余内存过低)而销毁你的应用组件
2. 应该采用数据模型驱动UI的方式，最好是一个可持久化的模型。
持久化被建议的原因有两个：
1. 用户不会因为系统销毁我们的应用而导致丢失数据
2. 我们的应用可以在网络状况不好甚至断网的情况下继续工作

![app_architecture](../img/app_architecture.png)

* *UI Controllers* - UI Controllers 是activities或者是fragments，只做如何显示数据和传递事件UI，UI Controllers既不包含UI数据也不直接操作数据。
* *ViewModel 和LiveData*-这些类代表UI所显示的所有的数据
* *Repository*-这个类是所有应用数据的真实来源并且充当UI进行通信的干净API。ViewModels只需从存储库中请求数据，他们不需要担心repository从数据库或者网络，或者如何或何时保存数据。存储是不同数据源之间的中介。
* *Remote Network Data Source*-管理远端数据源，比如网络
* *Model*-管理存储在数据库的本地数据

## LifecycleRegistry的addObserver流程

# ViewModel
ViewModel将视图的数据和逻辑从具有生命周期特性的实体(如Activity和Fragment)中剥离开来。
直到关联Activity或Fragment完全销毁时，ViewModel才会随之消失，也就是说，即使在旋转屏幕导致Fragment被重新创建等事件中，
视图数据依旧会被保留。ViewModel不仅消除了常见的生命周期问题，而且可以帮助构建更为模块、更方便测试的用户界面

> 设置ViewModel的字段后，我们需要一种方式来通知用户界面。这是LiveData类的用途


# LiveData
LiveData是一款基于观察者模式的可感知生命周期的核心组件。它允许应用程序中的组件观察LiveData对象的更改，而不会在它们之间创建明确的和严格的依赖关系路径。
LiveData为界面代码(Observer)的监视对象(Observable)。当LiveData所持有的数据改变时，它会通知相应的界面代码进行更新。
同时，LiveData持有界面代码Lifecycle的引用，这意味着它会在界面代码(LifecycleOwner)的生命周期处于stated或resumed时作出相应更新，而在LifecycleOwner被销毁时停止更新。防止对象泄露，以便应用程序消耗更多内存。
通过LiveData，开发者可以方便地构建安全性更高、性能更好的高响应度用户界面。
> 每次更新用户数据时，都会调用onChanged回调，并刷新UI.
如果你熟悉可使用可观察回调的其他库，则可能意识到我们不必重写`Fragment`的`onStop()`方法以停止观察数据。这对LiveData来说不是必需的，因为它可以感知生命周期，这意味着它不会回调，除非`Fragment`处于活动状态(收到onStart()但未收到onStop())。当`Fragment`收到`onDestory`时，LiveData也会自动删除观察者。

## 获取数据
已将ViewModel连接到Fragment，但ViewModel如何获取用户数据。例子中将使用Retrofit库来访问后端。
```java
public interface WebServise {
    @Get("/users/{user}")
    Call<User> getUser(@Path('user') String userId);
}
```
ViewModel一个简单的实现可以直接使用WebService来获取数据并将其分配给User对象。但它给ViewModel类带来了太多的责任，违背了关注点分离。此外VieModel的范围与Activity/Fragment生命周期相关联，生命周期完成时丢失所有数据是不好的用户体验。所以数据获取将委托给Repository模块

# Repository 
存储库类负责处理数据操作，它们为程序提供了一个干净的API，它们知道从何处获取数据以及更新数据时调用API。它们是不同数据源之间的中介。
与Room，LiveData或ViewModels，Repository不会扩展或实现架构组件库，这只是在“应用程序指南”体系结构中描述的应用程序中组织数据的一种方法

```java
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
```
## 管理组件的依赖关系
Repository类需要WebService的一个实例来完成它的工作，可以简单的创建，但还需要知道WebService类的依赖关系来构造它。这会使代码复杂化并重复，UserREpository可能不是唯一需要Web服务的类，如果每个类创建一个新的WebService，它将会非常耗费资源。
有两种模式来解决个问题
- 依赖注入
- Service Locator

## Caching data(缓存数据)
Repository在获取数据之后，它不会将它保留在任何地方。如果用户离开UserProfileFragment并回到它，应用程序会重新获取数据，这是糟糕的。
1. 浪费了宝贵的网络宽带
2. 强制用户等待新的查询完成
为解决此问题，将添加一个新的数据源到Repository，它将把用户对象缓存在内存中。

```java
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
```

## Persisting data(持久化数据)
如果用户离开应用程序一段时间，在目前的实现中，需要再次从网络中获取数据，体验槽糕且浪费。缓存Web请求也不行，会出现不一致的数据，因为请求时间不一致。Room就是解决这问题的

# Room
和 SQLite 有一样强大的功能，但是节省了很多重复编码的麻烦事儿。它的一些功能，如编译时的数据查询验证、内置迁移支持等，让开发者能够更简单地构建健壮的持久层。而且 Room 可以和 LiveData 集成在一起，提供可观测数据库并感知生命周期的对象。Room 集简洁、强大和可靠性为一身，在管理本地储存上表现卓越.
- 模板代码更少。它将数据库对象映射到Java对象。这意味着不需要使用ContentValues或者Cursors
- 编译时验证SQL语句，而不是在运行时
- 允许通过LiveData和RxJava进行数据观察
组成部分：

* @Entity 这个组件定义了数据库中的表。模型对象可以很容易地转换为实体对象
```java
@Entiry
class User {
    @PrimaryKey
    private int id;
    private String name;
    private String lastName;
}
```
* @Database 这个组件代表一个数据库持有者。在这个类定义数据的实体列表和数据的数据访问对象(data access object (DAO))。然后你使用这个类来创建一个新的数据库或者运行时获得一个到数据库的连接
```java
@Database(entities = {User.class}, version = 1) 
public abstract class MyDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}
```

* @Dao 该组件将类或接口表示为数据访问对象(data access object (DAO))，Dao负责定义访问数据库的方法。他们提供了一个读取和写入数据库数据的API
```java
@Dao
public interface UserDao {
    @Inset(onConflict = REPLACT)
    void save(User user);
    @Query("SELECT * FROM user WHERE id = :userId")
    LiveData<User> load(String userId);
}
```
load返回的是LiveData，Room知道数据库何时被修改，当数据改变时它会自动通知所有活动的观察者。

添加完Room数据源的Repository

```java
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
```

为了维护数据的一致性(数据在服务器之间可能出现更改)，数据库应该充当Single source of truth(单一数据来源)
## 附录：揭示网络装填
需求有可能出现，需要显示网络错误和加载状态的问题。通过使用Resource来封装数据及其状态
```java
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
```

从网络加载数据的同时从磁盘显示数据是常见的用例，
助手类：NetworkBoundResource，决策树
1. 条目第一次从数据加载，判断本地数据是否可用
2. 可用从本地磁盘加载
3. 不可用从网络加载，网络更新的时候可能显示缓存数据

# Lifecycle
Lifecycle是一个保存关于组件声明周期状态（activity或者fragment）信息的类，并允许其他对象观察此状态。
Lifecycle使用两个主要枚举来跟踪有关联组件的生命周期状态
## Event
从framework和Lifecycle类派发的生命周期事件。 这些事件映射到activity和fragment的回调事件。

## State
由Lifecycle对象跟踪的组件的当前状态
![lifycycle](../img/lifecycle-states.png)



生命周期管理(Lifecycle)帮助开发者创建“可感知生命周期的”组件，让其自己管理自己的生命周期，从而减少内存泄露和崩溃的可能性。
生命周期库是其他架构组件的基础
- `ProcessLifecycleOwnerInitializer`初始化Lifecycle框架
- `LifecycleDispatcher` 通过ActivityLifecycleCallbacks与LifecycleDispatcher配合监听当前进程中Activitys的生命周期变化并在必要时产生相应lifecycle事件
- `ProcessLifecycleOwner`通过ActivityLifecycleCallbacks与LifecycleDispatcher配合监听当前进程中Activitys的生命周期变化并在必要时产生相应的lifecycle事件
- `LifecycleService`，通过重写Service的生命周期方法并在相应方法内产生相应的lifecycle事件




# PagedList
解决RecyclerView处理大数据集的困难 