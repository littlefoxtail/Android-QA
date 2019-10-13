# 需求背景

## 原生方案的不足

1. 显示intent
   - 由于需要直接持有对应class，从而导致强依赖关系，提高了耦合度
2. 隐式intent
   - action等属性定义在Manifest，导致了扩展性较差
   - 规则集中式管理，导致协作变得非常困难
3. 原生的路由方案会出现跳转过程无法控制的问题，因为一旦使用了`startActivity`就无法插手其中任何环节了，只能交给系统管理，这就导致了在跳转失败的情况下无法降级，而是会直接抛出运营级的异常

## 自定义路由框架的适用场景

- 动态跳转：在一些复杂业务场景下，期望根据下发的数据自动的选择页面并进行跳转
- 组件化：将app按照一定的功能和业务拆分成多个组件module，不同的组件独立开发，组件化不仅能够提供团队工作效率，还能够提高应用性能。而组件化的前提就是解耦
- Native和H5的问题

期望：

- 通过URL索引可以解决类依赖的问题
- 通过分布式管理页面配置可以解决隐式Intent中集中式管理Path的问题
- 自己实现整个路由过程也可以拥有良好的扩展性
- 通过AOP的方式解决跳转过程无法控制的问题
- 能够提供非常灵活的降级方式

1. 分发：把一个URL或者请求按照一定的规则分配给一个服务或者页面来处理，这个流程就是分发，分发是路由框架的最基本的功能，也可以理解为简单的跳转。
2. 管理：将组件和页面按照一定的规则挂你起来，在分发的时候提供搜索、加载、修改等操作，这部分就是管理，也是路由框架的基础，上层功能都是建立在管理之上。
3. 控制：对路由操作做一些定制性的扩展

## 源码分析

### Compiler

Route Processor：处理路径路由。
Interceptor Processor：拦截器
Autowire Processor：自动装配

### API

用户在运行期使用的：

- 最上层是Launcher层，开发者所用到的，所有的API都是在这一层
- facade层
  - Service：这里的Service是ARouter抽象出来的概念，从本质上来讲，这里的Service是接口，从意义上讲是将一定的功能和组件封装成接口，并对外提供能力。
  - Template：模板，主要用于在编译器执行的SDK，这个SDK会在编译器生成一些映射文件也方便Route在运行的时候进行读取
  - Callback

### 自动注册

在编译期处理被注解的类，可以做到在运行中尽可能不使用反射。注解处理器其实是作用在JVM上的，可以通过插入一部分代码来处理被注解标注的类。

流程：

1. 通过注解处理器扫出被标注的类文件
2. 按照不同种类的源文件分类
3. 按照固定的命名格式生成映射文件
4. 初始化的时候通过固定包名加载映射文件

加载：分组管理，按需加载

### 开始

- arouter-annotation：ARouter路由框架所使用的全部注解，及其相关类
- arouter-compiler：注解编译处理器，引入“arouter-annotation”，在编译器把注解标注的相关目标类生成映射文件
- arouter-api：实现路由控制

#### Arouter init

Aouter的初始化过程最重要的一步一定是把前面编译产生的路由清单文件加载到内存，形成一个路由表，以供后面路由查找之用。

Class ARouter是一个代理类，它的所有函数实现都交给Class_ARouter去实现，两个都是单例模式

前文说了，我们断定初始化操作是要把 映射信息加载到内存，那么在哪里存储呢？这里就引入了内存仓库Warehouse的概念，实际上它就是持有了多个statice 的Map对象。

```java
public synchronized static void init(..) {
    // 通过指定包名com.alibaba.android.arouter.routes，找到所有 编译期产生的routes目录下的类名(不包含装载类)
    routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PACKAGE);

    for (String className : routerMap) {
        //【组别的清单列表】com.alibaba.android.arouter.routes.ARouter\$\$Root
        if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
                        // This one of root elements, load root.
            ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
        } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
            // Load interceptorMeta
            ((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);
        } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
            // Load providerIndex
            ((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);
        }
    }

}
```

反射实例化的类

- com.alibaba.android.arouter.routes.ARouter$$Group$$分组名
    包含了对应分组下的，路由URL与目标对象Class的映射关系；
    注意Router注解中无分组的话，默认以“/xx/xx”的第一个xx为分组名
- com.alibaba.android.arouter.routes.ARouter$$Root开头
    `组别的清单文件`，`Map<String, Class<? extends IRouteGroup>> groupsIndex`，包含了组名与对应组内的路由清单列表Class的映射关系，是ARouter在初始化的时候只会一次性地加载所有的root结点，而不会加载任何一个Group结点，这样就极大地降低初始化时加载结点的数量。
- com.alibaba.android.arouter.routes.ARouter$$Interceptors开头
    `Map< Integer, Class< ? extends IInterceptor>> interceptors`，包含了某个模块下的拦截器与优先级的映射关系，一个模块下的所有拦截器都在该类中包含，无分组特性，所以直接以模块名命名类文件
- com.alibaba.android.arouter.routes.ARouter$$Providers开头
    `ioc的动作路由清单列表`，`Map<String, RouteMete> providers`，PROVIDER类型的路由节点的清单列表，包含了使用依赖注入方式的某class（实现了IProvide接口的直接子类）的路由URL与class映射关系。

```java
// Cache route and metas
    //【组别的清单列表】 包含了组名与对应组内的路由清单列表Class的映射关系(这里只存储了未导入到 routes在键盘每个的组)
    static Map<String, Class<? extends IRouteGroup>> groupsIndex = new HashMap<>();
    //【组内的路由清单列表】包含了对应分组下的，路由URL与目标对象Class的映射关系；
    static Map<String, RouteMeta> routes = new HashMap<>();

    // Cache provider
    //缓存 IOC  目标class与已经创建了的对象 TODO ?全局应用共享一个IOc依赖注入对象？
    static Map<Class, IProvider> providers = new HashMap<>();
    //【Ioc的动作路由清单列表】包含了使用依赖注入方式的某class的  路由URL 与class映射关系
    static Map<String, RouteMeta> providersIndex = new HashMap<>();

    // Cache interceptor
    //【模块内的拦截器清单列表】包含了某个模块下的拦截器 与 优先级的映射关系
    static Map<Integer, Class<? extends IInterceptor>> interceptorsIndex = new UniqueKeyTreeMap<>("More than one interceptors use same priority [%s]");
    //已排序的拦截器实例对象
    static List<IInterceptor> interceptors = new ArrayList<>();
```

_ARouter.afterInit()，根据 Ioc.ByName()方式获取 拦截器界面，注意这个拦截器并不是我们定义的拦截器，而是Arouter实现的拦截器逻辑，它持有我们定义的拦截器，可以理解为“拦截器截面控制器”。

#### ARouter运行API调用过程分析

```java
public class _ARouter {
    protected Postcard build(String path) {
        if (TextUtils.isEmpty(path)) {
            throw new HandlerException(Consts.TAG + "Parameter is invalid!");
        } else {
            //1. 通过ARouter的Ioc方式(IProvider的ByType())方式找到  动态修改路由类
            PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
            if (null != pService) {
                //如果全局应用有实现 PathReplaceService.class接口，则执行 “运行期动态修改路由”逻辑。生成转换后的路由
                path = pService.forString(path);
            }
            return build(path, extractGroup(path));
        }
    }
}

protected <T> T navigation(Class<? extends T> service) {
    try {
        //PathReplaceService的实现，把里面东西给弄出来
        Postcard postcard = LogisticsCenter.buildProvider(service.getName());
        //兼容老版本而已 不用管
        // Compatible 1.0.5 compiler sdk.
        // Earlier versions did not use the fully qualified name to get the service
        if (null == postcard) {
            // No service, or this service in old version.
            postcard = LogisticsCenter.buildProvider(service.getSimpleName());
        }

        if (null == postcard) {
            return null;
        }

        LogisticsCenter.completion(postcard);
        return (T) postcard.getProvider();
    } catch (NoReuteFoundException ex) {
        return null;
    }
}
```

#### 一次简单的路由导航

入口：`_ARouter.build(path, extractGoup(path)`，默认分组例如`/11/22`，分组为`11`

```java
public class Postcard {
    public Object navigation(Context context, NavigationCallback callback) {
        return ARouter.getInstance().navigation(context, this, -1, callback);
    }
}
```

```java
public class _ARouter {
    //一次路由跳转的最终调用函数，包含 查找回调的调用、拦截器、绿色通道校验、和具体路由操作
    protected Object navigation(final Context context, final Postcard postcard, fina int requestCode, final NavigationCallback callback) {
        //完善postcard，当前只有path和group
        LogisticsCenter.completion(postcard);

        if (!postcard.isGreenChannel()) {   // It must be run in async thread, maybe interceptor cost too mush time made ANR.
            interceptorService.doInterceptions(postcard, new InterceptorCallback() {
                /**
                 * Continue process
                 *
                 * @param postcard route meta
                 */
                @Override
                public void onContinue(Postcard postcard) {
                    //根据 路由类型执行具体路由操作
                    _navigation(context, postcard, requestCode, callback);
                }

                /**
                 * Interrupt process, pipeline will be destory when this method called.
                 *
                 * @param exception Reson of interrupt.
                 */
                @Override
                public void onInterrupt(Throwable exception) {
                    if (null != callback) {
                        callback.onInterrupt(postcard);
                    }

                    logger.info(Consts.TAG, "Navigation failed, termination by interceptor : " + exception.getMessage());
                }
            });
        } else {
            //根据 路由类型执行具体路由操作
            return _navigation(context, postcard, requestCode, callback);
        }
    }

    //根据路由类型执行具体路由操作
    private Object _navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        switch (postcard.getType()) {
            //如果是Activity，则实现Intent跳转
            case ACTIVITY:
                final Intent intent = new Intent(currentContext, postcard.getDesination());
                intent.putExtras(postcard.getExtras());

                int flags = postcard.getFlags();
                if (-1 != flags) {
                    intent.setFlags(flags);
                } else if (!(currentContext instanceof Activity)) {
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                }

                // Set Actions
                String action = postcard.getAction();
                if (!TextUtils.isEmpty(action)) {
                    intent.setAction(action);
                }
                // Navigation in main looper.
                runInMainThread(new Runnable() {
                    @Override
                    public void run() {
                        startActivity(requestCode, currentContext, intent, postcard, callback);
                    }
                });
                break;

        }
    }
}
```

#### PostCard对象及LogisticsCenter.completion()

```java
public class LogisticsCenter {
    public synchronized static void completion(Postcard postcard) {
        //根据路径URL获取到路径元信息
        RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
        if (null == routeMeta) {
            Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());
            if (null == groupMeta) {

            } else {
                //实例化【组内清单创建逻辑】
                IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
                //将该组的【组内清单列表】加入到内存仓库中
                iGroupInstance.loadInto(Warehouse.routes);
                //从【组别的清单列表】移除当前组
                Warehouse.groupsIndex.remove(postcard.getGroup());
                //再次触发完善逻辑
                completion(postcard);

            }
        } else {
            // 目标class
            postcard.setDestination(routeMeta.getDestination());
            // 路由类型
            postcard.setType(routeMeta.getType());
            // 路由优先级
            postcard.setPriority(routeMeta.getPriority());
            // 额外的配置开关信息
            postcard.setExtra(routeMeta.getExtra());
            Uri rawUri = postcard.getUri();
            if (null != rawUri) {
                //拆分uri中的字段到postcard中
            }
        }

        switch (routeMeta.getType()) {
            case PROVIDER:
                Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                IProvider instance = Warehouse.providers.get(providerMeta);
                if (null == instance) { // There's no instance of this provider
                    IProvider provider;
                    try {
                        provider = providerMeta.getConstructor().newInstance();
                        provider.init(mContext);
                        Warehouse.providers.put(providerMeta, provider);
                        instance = provider;
                    } catch (Exception e) {
                        throw new HandlerException("Init provider failed! " + e.getMessage());
                    }
                }
                //实例化并持有目标类
                postcard.setProvider(instance);
                //
                postcard.greenChannel();
        }
    }
}
```
