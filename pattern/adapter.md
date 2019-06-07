# Adapter(适配器模式)

在适配器模式中可以定义一个包装类，包装不兼容接口的对象，这个包装类指的就是适配器(Adapter)，它所包装的对象就是是适配者(Adaptee)，即被适配的类。
适配器提供客户类需要的接口，适配器的实现就是把客户类的请求转化为对适配者的相应接口的调用。也就是说：当客户类调用适配器的方法时，在适配器类的内部将调用适配者类的方法，而这个过程对客户类是透明的，客户类并不直接访问适配者类。因此，适配器可以使由于接口不兼容而不能交互的类可以一起工作。这就是适配器模式的模式动机

适配模式(Adapter Pattern)：将一个接口转换成客户希望的另一个接口，适配器模式使接口不兼容的那些类可以一起工作，其别名为包装(Wrapper)。适配器模式既可以做为类结构型模式，也可以作为对象结构型模式。

根据适配器类与适配者类的关系不同，适配器模式可分为对象适配器和类适配器两种，在对象适配器模式中，适配器与适配者之间是关联关系；在类适配器模式中，适配器与适配者之间是继承关系。

* Target（目标抽象类）：目标抽象类定义客户所需接口，可以是一个抽象类或接口，也可以是具体类

    ```java
    public interface RawingBoat {
        void row();
    }
    ```

* Adaptee（被适配者）：定义了一个已经存在的接口，这个接口需要适配，被适配者一般是一个具体类，包含了客户希望使用的业务方法

    ```java
    public class FishingBoat {

        public void sail() {
            log.i("The fishing boat is sailing")
        }
    }
    ```

* Adapter（适配器类）：适配器类可以调用另一个接口，作为一个转换器，对Adaptee和Targe进行适配，适配器类是适配器模式的核心，在对象适配器中，它通过继承Target并关联一个Adaptee对象使得二者产生关联

    ```java
    public class FinishBoatAdapter implements RowingBoat {

        private FinishBoatAdapter boat;

        public FinishBoatAdapter() {
            boat = new FinishBoat();
        }

        @Override
        public void row() {
            boat.sail();
        }
    }
    ```

## 适配器模式总结

将一个类的接口转换成客户希望的另一个接口。adapter模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。

### 适配器优点

* 将目标类和适配者类解耦，通过引入一个适配类来重用现有的适配者类，而无需修改原有代码
* 增加类的透明性和复用性，将具体的实现封装在适配者类中，对于客户端类来说透明的，而且了提高了适配者的复用性
* 灵活性和扩展性都非常好，通过使用配置文件，可以很方便地更换适配器，也可以在不修改原有代码的基础上增加新的适配器类，符合“开闭原则”

**类适配模式具有如下优点**
由于适配器类适配者类的子类，因此可以在适配器类中置换一些适配者的方法，使得适配的灵活性更强

**对象适配器模式具有如下优点**
一个对象适配器可以把多个不同的适配者适配到同一个目标，也就是说，同一个适配器可以把适配者类和它的子类都适配到目标接口

### 适配器缺点

类适配器对于不支持多重继承的语言，一次最多只能适配一个适配者类，而且目标抽象类只能为抽象，不能为具体类，其使用有一定的局限性，不能讲一个适配者类和它的子类都适配到目标接口

对象适配器模式置换时适配者的方法不容易

### 适配器模式适用场景

* 系统需要使用一些现有的类，这些类的接口不符合系统的需要，甚至没有这些类的源代码
* 想创建一个可以重复使用的类，用于与一些彼此之间没有太大关联的一些类，包括一些可能在将来引进的类一起工作

## 适配器实例

* [java.util.Arrays#asList()](http://docs.oracle.com/javase/8/docs/api/java/util/Arrays.html#asList%28T...%29)
* [java.util.Collections#list()](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#list-java.util.Enumeration-)
* [java.util.Collections#enumeration()](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#enumeration-java.util.Collection-)
* [javax.xml.bind.annotation.adapters.XMLAdapter](http://docs.oracle.com/javase/8/docs/api/javax/xml/bind/annotation/adapters/XmlAdapter.html#marshal-BoundType-)