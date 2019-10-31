# Annotation Processor Tool

Annotation Processor是javac的一个工具，用来在编译时扫描和处理注解（Annotation）

注解处理器运行在它自己的JVM中。javac启动了一个完整的java虚拟机来运行这些注解处理器

## 示例

顾客要在pizza店购买食物的话，就得输入食物的名称；

```java
public class PizzaStore {
    public Meal create(String id) {
        if (id == null) {
            throw new IllegalArgumentException("id is null!");
        }
        if ("Calzone".equals(id)) {
            return new CalzonePizza();
        }

        if ("Tiramisu".equals(id)) {
            return new Tiramisu();
        }

        if ("Margherita".equals(id)) {
            return new MargheritaPizza();
            }

        throw new IllegalArgumentException("Unknown id = " + id);
        }
}
```

打算使用注解处理器生成MealFactory，现在想要提供一个注解和一个处理器用来生成工厂类

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface Factory {
    Class<?> type();

    String id();
}
```

## 注解处理器

```java
public class FactoryProcessor {
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> annotations = new LinkedHashSet<String>();
        annotations.add(Factory.class.getCanonicalName());
        return annotations;
    }

    public synchronized void init(ProcessingEnvironment processingEnv) {
        //用来处理TypeMirror的工具类
        typeUtils = processingEnv.getTypeUtils();
        // 一个用来处理Element的工具类
        elementUtils = processingEnv.getElementUtils();
        // 可以用这个类来创建文件
        filer = processingEnv.getFiler();
    }


}
```

扫描java源文件，源代码的每一部分都是Element的一个特定类型。Element代表程序中的元素

```java
public class FactoryProcessor {
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        for (Element annotatedElement : roundEnv.getElementsAnnotatedWith(Factory.class)) {
            
        }
    }
}
```