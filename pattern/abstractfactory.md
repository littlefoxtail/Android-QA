# 抽象工厂模式(Abstract Factory)

工厂方法模式通过引入工厂等级结构，解决了简单工厂类职责太重的问题，但由于工厂方法模式中的每一个工厂只生产一类产品，可能会导致系统中存在大量的工厂类。

抽象工厂模式提供一个创建一系列相关或相互依赖对象的接口，而无须指定它们具体的类。抽象工厂模式又称为Kit模式，属于对象创建型模式。
可以将一组具有同一主题的单独的工厂封装起来，正常使用中，客户端程序需要创建抽象工厂的具体实现，然后使用抽象工厂作为接口来创建这一主题的具体对象。

* AbstractFactory（抽象工厂）：它声明了一组用于创建一族产品的方法，生成一组具体产品，这些产品构成了一个产品族，每一个产品都位于某个产品等级结构中

    ```java
    public interface KingdomFactory {
        Castle createCastle();

        King createKing();

        Army createArmy();
    }
    ```

* ConcreteFactory(具体工厂)：它实现了在抽象工厂中声明的创建产品的方法，生成一组具体产品，这些产品构成了一个产品族，每一个产品都位于某个产品等级结构中

    ```java
    public class ElfKingdomFactory implements KingdomFactory {
        public Castle createCastle() {
            return new ElfCastle();
        }

        public King createKing() {
            return new ElfKing();
        }

        public Army createArmy() {
            return new ElfArmy();
        }
    }
    ```

    ```java
    public class OrcKingdomFactory implements KingdomFactory {
        public Castle createCastle() {
            return new OrcCastle();
        }

        public King createKing() {
            return new OrcKing();
        }

        public Army createArmy() {
            return new OrcArmy();
        }
    }
    ```

* AbstractProduct(抽象产品)：它为每种产品声明接口，在抽象产品中声明了产品所具有的业务方法

    ```java
    public interface Army {
        String getDescription();
    }
    ```

    ```java
    public interface Castle {
        String getDescription();
    }
    ```

    ```java
    public interface King {
        String getDescription();
    }
    ```

* ConcreteProduct(具体产品)：它定义具体工厂生产的具体产品对象，实现抽象产品接口中声明的业务方法

    ```java
    public class ElfKing implements King {
        @Override
        public String getDescription() {
            return "Elfing";
        }
    }
    ```

    ```java
    public class OrcKing implements King {
        @Override
        public String getDescription() {
            return "OrcKing"；
        }
    }
    ```

    ```java
    public class ElfArmy implements Army {
        @Override
        public String getDescription() {
            return "ElfArmy";
        }
    }
    ```

    ```java
    public class OrcArmy implements Army {
        @Override
        public String getDescription() {
            return "OrcArmy";
        }
    }
    ```

## 抽象工厂总结

抽象工厂模式是工厂方法模式的进一步延伸，由于它提供了功能更为强大的工厂类并且具有较好的可扩展性。

### 抽象工厂优点

* 抽象工厂模式隔离了具体类的生成，使得客户端不需要知道什么被创建。由于这种隔离，更换一个具体工厂就变得相对容易，所有的具体工厂都实现了抽象工厂中定义的那些公共接口
* 当一个产品族中的多个对象被设计成一起工作时，它能够保证客户端始终只使用同一个产品族中的对象
* 增加新的产品族很方便，无须修改已有系统，符合“开闭原则”

### 抽象工厂缺点

* 增加新的产品等级结构麻烦，需要对原有系统进行较大的修改，甚至需要修改抽象层代码，违背了“开闭原则”

### 抽象工厂适用场景

* 一个系统不应当依赖于产品类实例如何被创建、组合和表达的细节，这对于所有类型的工厂模式都是很重要的，用户无须关心对象的创建过程，将对象的创建和使用解耦
* 系统中有多于一个的产品族，而每次只使用其中某一产品族
* 属于同一个产品族的产品将在一起使用，这一约束必须在系统的设计中体现出来。同一个产品族中的产品可以是没有任何关系的对象，但是她们都具有一些共同的约束
* 产品等级结构稳定，设计完成之后，不会向系统中增加新的产品等级结构或者删除已有的产品等级结构

## 抽象工厂实例

* [javax.xml.parsers.DocumentBuilderFactory](http://docs.oracle.com/javase/8/docs/api/javax/xml/parsers/DocumentBuilderFactory.html)
* [javax.xml.transform.TransformerFactory](http://docs.oracle.com/javase/8/docs/api/javax/xml/transform/TransformerFactory.html#newInstance--)
* [javax.xml.xpath.XPathFactory](http://docs.oracle.com/javase/8/docs/api/javax/xml/xpath/XPathFactory.html#newInstance--)
