# Replugin

## Replugin的ClassLoader

主要ClassLoader：

1. `RePluginClassLoader`：宿主App中的Loader，继承PathClassLoader，也是唯一Hook住系统的。
2. `PluginDexClassLoader`：加载插件的Loader，继承DexClassLoader

### 宿主的ClassLoader

重载了`loadClass`

```java
@Override
protected Class<?> loadClass(..) {
    Class<?> = null;
    c = PMF.loadClass(className, resolve);
    if (c != null) {
        return c;
    }
    try {
        c = mOrig.loadClass(className);
        return c;
    } catch (Throwable e) {
        //
    }
    return super.loadClass(className, resolve);
}

```

### 插件的ClassLoader

```java
@Override
protected Class<?> loadClass(..) {
    try {
        pc = super.loadClass(className, resolve);
        if (pc != null) {
            return pc;
        }
    } catch (ClassNotFoundException e) {
        cnfException = e;
    }
    //从宿主中读取
    if (RePlugin.getConfig().isUseHostClassIfNotFound()) {
        try {
            return loadClassFromHost(..);
        } catch (ClassNotFoundException e) {

        }
    }
    return null;
}
```

### 创建和Hook

创建：

```java
public class RePluginCallbacks {
    /**
     * 创建宿主用的ClassLoader
     */
    public RepluginClassLoader createClassLoader(ClassLoader parent, ClassLoader original) {
        return new RePluginClassLoader(parent, original);
    }

    public PluginDexClassLoader createPluginClassLoader(..) {
        return new PluginDexClassLoader(..);
    }
}
```

Hook：

初始化，`PatchClassLoaderUtils`会在Application的`attachBaseContext()`中，通过`patch(application)`Hook住宿主的ClassLoader

## RePlugin的初始化

### attachBaseContext

```java
public class RePluginApplication {
    @Override
    protected void attachBaseContext(..) {
        super.attachBaseContext(base);
        RepluginConfig  c = createConfig();
        if (c == null) {
            c = new RepluginConfig();
        }
        RepluginCallbacks cb = createCallbacks();
        if (cb != null) {
            c.setCallbacks(cb);
        }
        RePlugin.App.attachBaseContext(this, c);
    }
}
```

## 插件内组件

使用ContentProvider

```java
Uri uri = Uri.parse("content://com.qihoo360.replugin.sample.demo1.provider2/test");

ContentValues cv = new ContentValues();
cv.put("address", "beijing");

Uri urii = context.getContentResolver().insert(uri, cv);
```
