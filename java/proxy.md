# 动态代理

## 动态代理的示例代码：

```java
public interface Subject {
    String operation(){
        return "operation by subject";
    }
}
```

```java
public class RealSubject implements Subject {
    @Override
    public String operation() {
        return "operation by subject";
    }
}
```

```java
public class ProxySubject implements InvocationHandler {
    protected Subject subject;
    public ProxySubject(Subject subject) {
        this.subject = subject;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwwable {
        // 做点啥逻辑玩玩
        return method.invoke(subject, args);
    }
}
```

调用代码：

```java
Subject subject = new RealSubject();
ProxySubject proxy = new ProxySubject(subject);
Subject sub = (Subject) Proxy.newProxyInstance(subject.getClass().getClassLoader(), subject.getClass().getInterface(), proxy);
sub.operation();
```

## 源码分析

```java
public class Proxy {
    public static Object newProxyInstance(ClassLoader loader, Class<?>[] interface, InvocationHandler h) {
        // 检查h不为空
        Objects.requireNonNull(h);

        Class<?> cl = getProxyClass0(loader, intfs);
    }

    public static Class<?> getProxyClass0(ClassLoader loader, Class<?>... interfaces) {
        return proxyClassCache.get(loader, interfaces);
    }
}
```

```java
public class WeakCache {
    public V get(K key, P parameter) {
        ...
        Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parmeter));
        Supplier<V> supplier = valuesMap.get(subKey);
        Factory factory = null;
        while(true) {
            if (supplier != null) {
                V value = supplier.get();
                if (value != null) {
                    return value;
                }
            }
        }
    }
}
```

```java
public class WeakCache {
    private final class Factory implements Supplier<V> {
        @Override
        public synchronized get() {
            value = Objects.requireNonNull(valueFactory.apply(key, parameter));
        }
    }

    private static final class ProxyClassFactory implements BiFunction {
        @Override
        public Class<?> apply(...) {
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(proxyName, interfaces, accessFlags);
        }
    }
}
```

### 动态生成代理类

```java
public class ProxyGenerator {
    public static byte[] generateProxyClass(...) {
        ProxyGenerator gen = new ProxyGenerator(...);
        final byte[] classFile = gen.generateClassFile();
        if (saveGeneratedFiles) {
                java.security.AccessController.doPrivileged(
                new java.security.PrivilegedAction<Void>() {
                    public Void run() {
                        try {
                            int i = name.lastIndexOf('.');
                            Path path;
                            if (i > 0) {
                                Path dir = Paths.get(name.substring(0, i).replace('.', File.separatorChar));
                                Files.createDirectories(dir);
                                path = dir.resolve(name.substring(i+1, name.length()) + ".class");
                            } else {
                                path = Paths.get(name + ".class");
                            }
                            Files.write(path, classFile);
                            return null;
                        } catch (IOException e) {
                            throw new InternalError(
                                "I/O exception saving generated file: " + e);
                        }
                    }
                });
        }
        return classFile;

        }

}
```

将在指定目录生成一个ProxySubject.class的文件

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class ProxySubject
  extends Proxy
  implements Subject
{
  private static Method m1;
  private static Method m3;
  private static Method m2;
  private static Method m0;

  public ProxySubject(InvocationHandler paramInvocationHandler)
  {
    super(paramInvocationHandler);
  }

  public final boolean equals(Object paramObject)
  {
    try
    {
      return ((Boolean)this.h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final String operation()
  {
    try
    {
      return (String)this.h.invoke(this, m3, null);
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final String toString()
  {
    try
    {
      return (String)this.h.invoke(this, m2, null);
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final int hashCode()
  {
    try
    {
      return ((Integer)this.h.invoke(this, m0, null)).intValue();
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  static
  {
    try
    {
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m3 = Class.forName("Subject").getMethod("operation", new Class[0]);
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      return;
    }
    catch (NoSuchMethodException localNoSuchMethodException)
    {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    }
    catch (ClassNotFoundException localClassNotFoundException)
    {
      throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
    }
  }
}
```


