# ClassLoader

ClassLoader的基本职责就是根据一个指定的类的名称，找到或者生成对应的字节码，然后从这些字节代码中定义出一个Java类，即java.lang.Class类的一个实例。除此之外，ClassLoader还负责加载Java应用所需的资源，如图像文件和配置文件等

|方法|说明|
|:--:|:--:|
|getParent()|返回该类加载器的父类加载器|
|loadClass(String name)|加载名称为name的类，返回的结果是Class类的实例|
|findClass(String name)|查找名称为name的类，返回的结果是Class的实例|
|defineClass(String name, byte[] b, int off, int len)|把字节数组b中的内容转换成Java类，返回的结果是Class类的实例。这个方法被声明为final|
|resolveClass|链接指定的Java类|

Java语言自带三个类加载器：

- Bootstrap ClassLoader（引导类加载器）：最顶层的加载类，主要加载核心类库
- Extension ClassLoader（扩展类加载器）：负责加载Java的扩展类库，默认加载JAVA_HOME/jre/lib/ext/目下的所有jar。
- App CladdLoader（称为系统类加载器），负责加载应用程序classpath目录下的所有的jar和class文件。

> 除了Java默认提供的三个ClassLoader之外，用户还可以根据需要定义自己的ClassLoader，而这些自定义的ClassLoader都必须继承自java.lang.ClassLoader类，也包括Java提供的另外两个ClassLoader（Extension ClassLoader和App ClassLoader）,但是Bootstrap ClassLoader不继承自ClassLoader，因为它不是一个普通的Java类，底层由于C++编写，已嵌入到了JVM内核当中，当JVM启动后，Bootstrap ClassLoader也随着启动，负责加载完核心类库后，并构造Extension ClassLoader和App ClassLoader类加载器。

## ClassLoader加载类的原理

每一个ClassLoader实例都有一个父类加载器的引用（非继承关系，关联关系），虚拟机内置的类加载器（Bootstrap ClassLoader）本身没有父类加载器，但可以用作其他ClassLoader实例的父类加载。当一个ClassLoader实例需要加载某个类，它会视图亲自搜索某个类之前，先把这个任务委托给它的父类加载器，这个过程是由上至下依次检查的

![parents_delegation_model](/img/parents_delegation_model.png)

双亲委托模型主要避免重复加载

```java
public class ClassLoader {
    protected Class<?> loadClass(String name, boolean resolve) {
        synchronized (getClassLoadingLock(name)) {
            // 检查是否已经加载类
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        // 在父类加载器上调用loadClass
                        c = parent.loadClass(name, false);
                    } else {
                        // 使用虚拟机内置类加载器
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {

                }
                if (c == null) {
                    // 还找不到就调用findClass
                    long t1 = System.nanoTime();
                    c = findClass(name);
                }
            }
            if (resolve) {
                resolveClass(c);
            }
        }
    }
}
```

## JVM搜索类

JVM判定两个class是否相同：

- 判断两个类名是否相同
- 判断是否由同一个类加载器实例加载。

## 自定义ClassLoader

动态加载一个class文件，默认的ClassLoader不能满足需求

步骤：

- 继承java.lang.ClassLoader
- 重写父类的findClass方法

## 加载类的过程

类加载器会首先代理其他类加载器来尝试加载这个类。这就意味着真正完成类的加载工作的类加载器和启动这个加载过程的类加载器，有可能不是同一个。

```java
public class FileSystemClassLoader extends ClassLoader {
    private String rootDir;

    public FileSystemClassLoader(String rootDir) {
        this.rootDir = rootDir;
    }

    @override
    protected Class<?> findClass(String name) throws ClasNotFoundException {
        byte[] classData = getClassData(name);

        if (classData == null) {
            throw new ClassNotFoundException();
        } else {
            // 由defineClass方法来把这些字节代码转换成java.lang.Class类的实例
            return defineClass(name, classData, 0)
        }
    }

    private byte[] getClassData(String className) {
       String path = classNameToPath(className);
       try {
           InputStream ins = new FileInputStream(path);
           ByteArrayOutputStream baos = new ByteArrayOutputStream();
           int bufferSize = 4096;
           byte[] buffer = new byte[bufferSize];
           int bytesNumRead = 0;
           while ((bytesNumRead = ins.read(buffer)) != -1) {
               baos.write(buffer, 0, bytesNumRead);
           }
           return baos.toByteArray();
       } catch (IOException e) {
           e.printStackTrace();
       }
       return null;
   }

    private String classNameToPath(String className) {
        return rootDir + File.separatorChar
               + className.replace('.', File.separatorChar) + ".class";
    }
}
```

# Android中的ClassLoader

[androidClassLoader](androidclassloader.md)
