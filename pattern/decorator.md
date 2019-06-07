# Decorator(装饰模式)

装饰模式以对客户透明的方式动态地给一个对象附件上更多的责任，换言之，客户端并不会觉得对象在装饰前和装饰后有什么不同。装饰模式可以在不需要创造更多子类的情况下，将对象的功能加以扩展。这就是装饰模式的动机。

装饰模式是一种用来替代你继承的技术，它通过无须定义子类的方式来给对象动态增加职责，使用对象之间的关联关系取代类之间的继承关系。在装饰模式中引入装饰类，在装饰类中既可以调用带装饰的原有类的方法，还可以增加新的方法，以扩充原有类的功能

* Component（抽象构件）:它是具体构件和抽象装饰类的共同父类，声明了在具体构件展总实现的业务方法，它的引入可以使客户端以一直的方式处理未被装饰的对象以及装饰之后的对象，实现客户端的透明操作

    ```java
    public interface Troll {
        void attach();

        int getAttackPower();

        void fleeBattle();
    }
    ```

* ConcreteComponent (具体构件类)：它是抽象构件类的子类，用于定义具体的构建对象，实现了在抽象构件中声明的方法，装饰器可以给它增加额外的责任

    ```java
    public class SimpleTroll implements Troll {
        @Override
        public void attach() {
            log.i("the troll tries to grab you!");
        }
        @Override
        public int getAttackPower() {
            return 10;
        }
        @Override
        public void fleeBattle() {
            log.i("The troll shrieks in horror and runs away!");
        }
    }
    ```

* Decorator（抽象装饰类）：它是抽象构件类的子类，用于给具体构件增加职责，但是具体职责在其子类中实现。它维护一个指向抽象构件对象的引用，通过该引用可以调用装饰之前构件对象的方法，并通过其子类扩展该方法，以达到装饰的目的
* ConcreteDecorator(具体装饰类)：抽象装饰类的子类，负责给具体构件增加职责。每一个具体装饰类都定义了一些新的行为，它可以调用在抽象装饰类中定义的方法，并可以增加新的方法用以扩充对象的行为

    ```java
    public ClubbedTroll implements Troll {
        private Troll decorated;

        public ClubbedTroll (Troll decorated) {
            this.decorated = decorated;
        }

        @Override
        public void attack() {
            decorated.attach();
        }

        @Override
        public void getAttackPower() {
            decorated.getAttackPower() + 10;
        }

        @Override
        public void fleeBattle() {
            decorated.fleeBattle();
        }
    }
    ```

* Client（客户端）：

    ```java
    Troll troll = new SimpleTroll();
    troll.attack();
    troll.fleeBattle();

    troll = new ClubbedTroll(troll);
    troll.attack();
    troll.fleeBattle();
    ```

## 装饰模式总结

装饰模式降低了系统的耦合度，可以动态增加或删除对象的职责，并使得需要装饰的具体构件类和具体装饰类可以独立变化，以便增加新的具体构件类和具体装饰类

### 装饰模式优点

* 装饰模式与继承关系的目的都是要扩展对象的功能，但是装饰模式可以提供比继承更多的灵活性。
* 可以通过一种动态的方式来扩展一个对象的功能，通过配置文件可以在运行时选择不同的装饰器，从而实现不同的行为。
* 通过使用不同的具体装饰器以及这些装饰类的排列组合，可以创造出很多不同行为的组合。可以使用多个具体装饰类来装饰同一对象，得到功能更为强大的对象。
* 具体构建类与具体修饰类可以独立变化，用户可以根据需要增加新的具体构件类和具体修饰类，在使用时再对其进行组合，原有代码无须改变，符合"开闭原则"

### 装饰模式缺点

* 使用装饰模式进行系统设计时将产生很多小对象，这些对象的区别在于它们之间相互连接的方式有所不同
* 装饰模式比继承有更加的灵活性，但同时也意味着比继承更加容易出错，排错也很困难

### 装饰模式适用场景

* 在不影响其他对象的情况下，以动态、透明的方式给单个对象添加职责
* 当不能采用继承的方式对系统进行扩展或者采用继承不利于系统扩展和维护时可以使用装饰模式

## 装饰模式实例

* [java.io.InputStream](http://docs.oracle.com/javase/8/docs/api/java/io/InputStream.html),
    [java.io.OutputStream](http://docs.oracle.com/javase/8/docs/api/java/io/OutputStream.html),
    [java.io.Reader](http://docs.oracle.com/javase/8/docs/api/java/io/Reader.html) and [java.io.Writer](http://docs.oracle.com/javase/8/docs/api/java/io/Writer.html)
* [java.util.Collections#synchronizedXXX()](http://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#synchronizedCollection-java.util.Collection-)
* [java.util.Collections#unmodifiableXXX()](http://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#unmodifiableCollection-java.util.Collection-)
* [java.util.Collections#checkedXXX()](http://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#checkedCollection-java.util.Collection-java.lang.Class-)
