
# 控制反转（Inversion of Control缩写IoC）

面向对象编程中的一种设计原则，可以用来降低计算机代码之间的耦合度，最常见的方式叫做依赖注入（Dependency injection简称DI），还有一种方式叫“依赖查找”（Dependency Lookup）。通过控制反转，对象在被创建的时候，由一个调控系统内所有对象的外界实体，将由依赖的对象的引用传递给它。也可以说，依赖注入到对象中

## 实现方式

依赖注入：被动的接收对象，在类A实例创建过程中即创建了依赖的对象，通过类型或者名称来判断不同的对象注入到不同的属性中
依赖查找：主动索取相应类型的对象，获得依赖对象的时间也可以在代码中自由控制

# 依赖注入

依赖注入（Dependency Injection）是用于实现控制反转的最常见的方式之一

## 为什么需要依赖注入

控制反转用于解耦，解的究竟是谁和谁的耦

```java
public class MovieLister {
    private MovieFinder finder;
    public MovieLister() {
        finder = new MovieFinderImpl();
    }

    public Movie[] moviesDirectedBy(String arg) {
        List allMovies = finder.findAll();
        for (Iterator it = allMovies.iterator(); it.hasNext();) {
            Movie movie = (Movie) it.next();
            if (!movie.getDirector().equals(arg)) it.remove();
        }
        return (Movie[]) allMovies.toArray(new Movie[allMovies.size()]);
    }
    ...
}

public interface MovieFinder {
    List findAll();
}
```

一切看起来不粗。但是，当我们希望修改finder，将finder替换为一种新的实现时候，（比如为MovieFinder增加一个参数表明Movie数据的来源是哪个数据库），我们不仅需要修改MovieFinderImpl类，还需要修改我们MovieLister中创建MovieFinderImpl的代码。

这就是依赖注入要处理的耦合。这种在MovieLister中创建MovieFinderImpl的方式，使得MovieLister不仅仅依赖于MovieFinder这个接口，它还依赖于MovieListImpl这个实现。
这种在一个类中直接创建另一个类的对象的代码，和硬编码（hard-cored strings）以及硬编码的数字（magic numbers）一样，是一种导致耦合的坏味道，可以把这种坏味道称为硬初始化（hard init）。同时，new对象也是有毒的。

Hard Init带来的主要坏处有两方面的：1）上文所述的修改其实现时，需要修改创建处的代码；2）不便于测试，这种创建类无法单独被测试，其行为和MovieFinderImpl紧紧耦合在一起，同时，也会导致代码的可读性问题

## 依赖注入的实现方式

1. 构造函数注入（Contructor Injection）
    ```java
    public class MovieLister {
        private MovieFinder finder;
        public MovieLister(MovieFinder finder) {
            this.finder = finder;
        }
    }
    ```

2. setter注入
    ```java
    public class MovieLister {
        public void setFinder(MovieFinder finder) {
            this.finder = finder;
        }
    }
    ```

3. 接口注入

    接口注入使用接口来提供setter方法，
    ```java
    public interface InjectFinder {
        void injectFinder(MovieFinder finder);
    }
    ```

    ```java
    class MovieLister implements InjectFinder {
        public void injectFinder(MovieFinder finder) {
            this.finder = finder;
        }
    }
    ```

4. 基于注解
    基于Java的注解的功能，在私有变量前加“@Autowired”等注解，不需要显示的定义以上代码，便可以让外部容器传入对应的对象。该方案相当于定义了public的set方法，但是因为没有真正的set方法，从而不会为了实现依赖注入导致暴露了不该暴露的接口（因为set方法只想让容器访问来注入而并不希望其他依赖此类的对象访问）。

需要根据不同的框架创建被依赖的MovieFinder的实现

## 依赖注入框架

### @Inject

使用@Inject注解告诉Dagger创建类的实例应该用的构造器。当需要一个实例对象时，Dagger将会获得需要的参数并调用这个构造器

#### 构造器注入

@Inject 使用在类的构造器上

```java
public class LoginActivityPresenter {
    private LoginActivity loginActivity;
    private UserDataStore userDataStore;
    private UserManager userManager;

    @Inject
    public LoginActivityPresenter(LoginActivity loginActivity,
                                  UserDataStore userDataStore,
                                  UserManager userManager) {
        this.loginActivity = loginActivity;
        this.userDataStore = userDataStore;
        this.userManager = userManager;
    }
}
```

所有的参数都是通过依赖图表中获取的。

```java
public class LoginActivity extends BaseActivity {
    @Inject
    LoginActivityPresenter presenter;
}
```

#### 属性注入

```java
public class SplashActivity extends AppCompatActivity {
    @Inject
    LoginActivityPresenter presenter;

    @Inject
    AnalyticsManager analyticsManager;

    @Override
    protected void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        getAppComponent().inject(this);
    }
}
```

#### 方法注入

在这个类的public方法中作注解：

```java
public class LoginActivityPresenter {
    private LoginActivity loginActivity;

    @Inject
    public LoginActivityPresenter(LoginActivity loginActivity) {
        this.loginActivity = loginActivity;
    }

    @Inject
    public void enableWatches(Watches watches) {
        watches.register(this);
    }
}

```

### Module注解

这个注解用于标记提供依赖的类-多亏它Dagger才会知道某些地方需要的对象被构建

```java
@Module
public class GithuApiModule {
    @Provides
    @Singleton
    OkHttpClient provideOkHttpClient() {
        OkHttpClient okHttpClient = new OkHttpClient();
        okHttpClient.setConnectTimeout(60 * 1000, TimeUnit.MILLISECONDS);
        return okHttpClient;
    }

    @Privides
    @Singleton
    RestAdapter provideRestAdapter(Application application, OkHttpClient okHttpClient) {
        RestAdapter.Build build = new RestAdapter.Builder();
        build.setClient(new OkClient(okHttpClient))
            .setEndpoint(application.getString(R.string.endpoint));
        return builder.build();
    }
}
```

### Provides注解

这个注解在@Module类，@Provides会标记Module中那些返回依赖的方法

### Component注解

@Component通常是@Module和@Inject之间的桥梁

### Scope注解

在Dagger 2中，`@Scope`被用于标记自定义的scope注解。被作注解的依赖会变成单例，但是这会与component的生命周期关联

### MapKey

这个注解用在定义一些依赖集合

```java
@MapKey(unwrapValue=true)
@interface TestKey {
    String value();
}
```

```java
@Provides(type = Type.MAP)
@TestKey("foo")
String provideFooKey() {
    return "foo value";
}

@Provides(type = Type.MAP)
@TestKey("bar")
String provideBarKey() {
    return "bar value";
}
```

```java
@Inject
Map map;

map.toString() // => „{foo=foo value, bar=bar value}”
```

### Qualifier

Qualifier注解帮助我们去相同接口的依赖创建“tags”

```java
@Provides
@Singleton
@GithubRestAdapter  //Qualifier
RestAdapter provideRestAdapter() {
    return new RestAdapter.Builder()
        .setEndpoint("https://api.github.com")
        .build();
}

@Provides
@Singleton
@FacebookRestAdapter  //Qualifier
RestAdapter provideRestAdapter() {
    return new RestAdapter.Builder()
        .setEndpoint("https://api.facebook.com")
        .build();
}
```

```java
@Inject
@GithubRestAdapter
RestAdapter githubRestAdapter;

@Inject
@FacebookRestAdapter
RestAdapter facebookRestAdapter;
```

# 使用Dagger

## 声明依赖

Dagger构造应用程序类的实例并满足它们的依赖关系。 它使用javax.inject.Inject批注来标识它感兴趣的构造函数和字段。

```java
class Thermosiphon implements Pump {
    private final Heater headter;

    @Inject
    Thermosiphon(Heater heater) {
        this.heater = heater;
    }
}
```

```java
class CoffeeMaker {
  @Inject Heater heater;
  @Inject Pump pump;

  ...
}
```

## 实现依赖

默认情况下，Dagger会通过创建需要类型的实例来提供依赖。当你需要一个CofferMaker，它会通过new CofferMaker()来获取一个实例并赋值给需要注入的字段。

但@Inject不能被创建

- 接口不能被创建（不支持接口注入）
- 第三方的类不能被注解（第三方类没有@Inject，除非可以改源码）
- 可配置的对象必须配置好

使用@Provides注解方法来实现依赖。方法返回的类型与其实现的依赖一致

```java
@Provides static Heater provideHeater() {
    return new ElectricHeater();
}
```

@Provides方法可能具有自己的依赖关系，只要需要Pump就返回一个Thermosiphon

```java
@Provides static Pump providePump(Thermosiphon pump) {
    return pump;
}
```

所有的@Provides方法都必须属于一个module，包含@Module注解的类

```java
@Module
class DripCoffeeModule [
    @Provides static Heater provideHeater() {
        return new ElectricHeater();
    }

    @Provides static Pump providePump(Thermosiphon pump) {
        return pump;
    }
]
```

按照惯例，@Provide方法以提供前缀命名，模块类以Module后缀命名。

## 建立对象图

@Inject和@Provides注解类形成了一个对象图，由它们的依赖关系连接。
在Dagger2中，该集合由一个接口定义，该接口具有没有参数的方法并返回所需的类型。通过将@Component注释应用于此类接口并将模块类型传递给modules参数，Dagger2然后根据这个协议生成所有的实现

```java
@Component(modules = DripCoffeeModule.class)
interface CoffeeShop {
    CoffeeMaker maker();
}
```

实现是前缀Dagger加上接口名字，通过调用实现类的builder()方法可以获得Builder实例，通过这个实例可以设置依赖并build()得到一个新的实例。

```java
CoffeeShop coffeeShop = DaggerCoffeeShop.builder()
    .dripCoffeeModule(new DripCoffeeModule())
    .build();
```

如果您的@Component不是顶级类型，则生成的组件名称将包含其封闭类型的名称，并使用下划线连接

```java
class Foo {
    static class Bar {
        @Component
        interface BazComponent {

        }
    }
}
```

生成的Component名为`DaggerFoo_Bar_BazComponent`

任何有可达的默认构造器的module都可以被省略，如果没有设置builder会自动创建一个实例。而且任何@Provides方法都是静态的module,builder是不需要其实例的。如果不需要用户创建依赖实例就可以创建所有的依赖，那么生成的实现类将会包含一个create()方法，可以使用此方法得到一个实例而不用于builder打交道。

## 对象图中的绑定

上面的例子展现如何构建一个拥有一些典型绑定的component，但还有不同的机制来为图贡献绑定。作为依赖下面这些是可用的而且可以用来生成更好的component.

- 由@Provides方法在@Module中直接由@Component.modules引用或通过@Module.includes引用的方法
- 任何类型的@Inject注解的构造器可以没有作用域也可以有与某个Component作用域一致的@Scope注解
- component依赖的component提供方法)
- component自己任何包含的subcomponent的不合格的builders
- 上面所有绑定的Provider和Lazy 包装器
- 上面绑定的懒加载的供应器（e.g Provider>
- 任何类型的MemberInjector

## 单例和域绑定

用@Singleton注解@Provides方法或可注入的类，对象图将会在应用中使用同一个一个实例

```java
@Provides @Singleton static Heater provideHeater() {
    return new ElectricHeater();
}
```

可注入的类上的@Singleton注解也可以作为文档

```java
@Singleton
class CoffeeMaker {

}
```

因为Dagger2会将图中添加了作用域的实例和component实现类的实例联系起来，所以这些component需要声明作用域。

## 可重用的scope

有时你可能想限制@Inject注解的构造器初始化的次数或者@Provides方法被调用的次数，但并不需要保证单例。这在内存比较吃紧的环境比如Android下会很有用

## 延迟注入

对于任意绑定T，可以创建Lazy，这样就可以延迟对象初始化直到调用Lazy的get方法。如果T是单例，那么对象图中所有的注入都是同一个Lazy实例。否则每个注入拿到都是自己的Lazy实例。对同一个Lazy实例连续调用get()方法放回都是一个T对象

```java
class GridingCoffeeMaker {
    @Inject Lazy<Grinder> lazyGrinder;

    public void brew() {
        while(needsGrinding()) {
            lazyGrinder.get().grind();
        }
    }
}
```

## Provider注入

有时你需要返回多个实例而不是注入单个值。你有多种选择（Factories,Builders,等等）,其中一种选择就是注入一个Provider而不是T。每次调用get()方法时Provider会调用绑定逻辑。如果那个绑定逻辑是@Inject注解的构造器，会创建一个新对象，但一个@Provides方法是无法保证这点的。

```java
class BigCoffeeMaker {
  @Inject Provider<Filter> filterProvider;

  public void brew(int numberOfPots) {
  ...
    for (int p = 0; p < numberOfPots; p++) {
      maker.addFilter(filterProvider.get()); //每次都是新的filter对象
      maker.addCoffee(...);
      maker.percolate();
      ...
    }
  }
}
```

## Qualifiers

有时类型不足以区分依赖。例如：一个复杂的咖啡机想要将睡和盘子的加热器分开。

这种情况，我们添加一个qualifier annotation。这是任何本身有@Qualifier注解的注解。下面是@Named的声明，它是javax.inject中的注解。

```java
@Qualifier
@Documented
@Retention(RUNTIME)
public @interface Named {
    String value() default "";
}
```

你可以创建自定义的qualifier注解，或使用@Named.在关心的字段或参数上使用qualifiers.类型+qualifier将会用来标识一个依赖

```java
class ExpensiveCoffeeMaker {
    @Inject @Named("water") Heater waterHeater;
    @Inject @Named("hot plate") Heater hotPlateHeater;
}
```

注解对应的@Provides方法来提供限定的值。

```java
@Provides @Named("hot plate") static Heater provideHotPlateHeater() {
  return new ElectricHeater(70);
}

@Provides @Named("water") static Heater provideWaterHeater() {
  return new ElectricHeater(93);
}
```