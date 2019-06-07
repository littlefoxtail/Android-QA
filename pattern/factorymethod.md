# 工厂方法模式(Factory Method)

工厂方法模式又称为工厂模式，也叫虚拟构造器(Virtual Constructor)模式或者多态工厂，它属于类创建型。

在工厂方法模式中，不再提供一个统一的工厂类来创建所有的产品对象，而是针对不同的产品提供不同的工厂
系统提供一个与产品等级结构对应的工厂等级结构

定义一个用于创建对象的接口，让子类决定将哪一个类实例化。工厂方法模式让一个类的实例化延迟到其子类

工厂方法模式提供了一个抽象工厂接口来声明抽象工厂方法，而由其子类来具体实现工厂方法，创建具体的产品对象

* Product（抽象产品）：它是定义产品的接口，是工厂方法模式所创建对象的超类型，也就是产品对象的公共父类

    ```java
    public interface Weapon {
        WeaponType getWeaponType();
    }
    ```

* ConcreteProduct（具体产品）：它实现了抽象产品接口，某些类型的具体产品由专门的具体工厂创建，具体工厂和具体产品之间一一对应

    ```java
    public class ElfWeapon implements Weapon {
        private WeaponType weaponType;

        public ElfWeapon(WeaponType weaponType) {
            this.weaponType = weaponType;
        }

        @Override
        public WeaponType getWeaponType() {
            return weaponType;
        }
    }
    ```

    ```java
    public class OrcWeapon implements Weapon {
        private WeaponType weaponType;

        public OrcWeapon(WeaponType weaponTye) {
            this.weaponType = weaponType;
        }

        @Override
        public WeaponType getWeaponType() {
            return weaponType;
        }
    }
    ```

* Factory（抽象工厂）：抽象工厂类中，声明了工厂方法(Factory Method)，用于返回一个产品。抽象工厂是工厂方法模块的核心，所有创建对象的工厂方法都必须实现该接口

    ```java
    public interface Blacksmith {
        Weapon manufactureWeapon(WeaponType weaponType);
    }
    ```

* ConcreteFactory（具体工厂）：它是抽象工厂类的子类，实现了抽象工厂中定义的工厂方法，并可由客户端调用，返回一个具体产品类的实例

    ```java
    public class ElfBlacksmith implements Blacksmith {
        public Weapon manufactureWeapon(WeaponType weaponType) {
            return new ElfWeapon(weaponType);
        }
    }
    ```

    ```java
    public class OrcBlacksmith implements Blacksmith {
        public Weapon manufactureWeapon(WeaponType weaponType) {
            return new OrcWeapon(weaponType);
        }
    }
    ```

* Client（客户端）：

    ```java
    public class App {
        private final Blacksmith blacksmith;

        public App(Blacksmith blacksmith) {
            this.blacksmith = blacksmith;
        }

        public static void main(String[] args) {
            App app = new App(new OrcBlacksmith());
            app.manufactureWeapons();

            app = new App(new ElfBlacksmith());
            app.manufactureWeapons();
        }

        private void manufactureWeapons() {
            Weapon weapon;
            weapon = blacksmith.manufactureWeapon(WeaponType.SPEAR);

            weapon = blacksmith.manufactureWeapon(WeaponType.AXE);
        }
    }
    ```

## 工厂方法总结

工厂方法模式是简单工厂模式的进一步抽象和推广。由于使用了面向对象的多态性，工厂方法模式保持了简单工厂模式的有点，而且克服了它的缺点。在工厂方法模式中，核心的工厂类不在负责所有产品的创建，而是将具体创建工作交给子类去做。这个核心类仅仅负责给出具体工厂必须实现的接口，而不负责哪一个产品类被实例化这种细节，这使得工厂方法模式可以允许系统在不修改工厂角色的情况下引入新的产品。

### 工厂方法优点

* 工厂方法隐藏了具体产品类将被实例化这一细节，用户只需要关心所需产品的工厂
* 基于工厂角色和产品角色的多态性设计是工厂方法模式的关键。它能够让工厂可以自主确定创建哪种产品对象
* 加入一个新产品时，无须修改抽象工厂和抽象产品提供的接口，只要添加一个具体工厂类和具体产品就可以，符合“开闭原则”

### 工厂方法缺点

* 在添加新产品时，需要编写新的具体产品类，而且还要提供与之对应的具体工厂类，系统中类的个数将成对增加，在一定程度上增加了系统的复杂度，有更多的类需要编译和运行，会给系统带来一些额外的开销
* 由于考虑到系统的可扩展性，需要引入抽象层，在客户端代码中均使用抽象层进行定义，增加了系统的抽象性和理解难度

### 工厂方法适用场景

* 客户端不知道它所需要的对象的类。在工厂方法模式中，客户端不需要知道具体产品类的类名。
* 抽象工厂类通过其子类来指定创建哪个对象

## 工厂方法实例

* [java.util.Calendar](http://docs.oracle.com/javase/8/docs/api/java/util/Calendar.html#getInstance--)
* [java.util.ResourceBundle](http://docs.oracle.com/javase/8/docs/api/java/util/ResourceBundle.html#getBundle-java.lang.String-)
* [java.text.NumberFormat](http://docs.oracle.com/javase/8/docs/api/java/text/NumberFormat.html#getInstance--)
* [java.nio.charset.Charset](http://docs.oracle.com/javase/8/docs/api/java/nio/charset/Charset.html#forName-java.lang.String-)
* [java.net.URLStreamHandlerFactory](http://docs.oracle.com/javase/8/docs/api/java/net/URLStreamHandlerFactory.html#createURLStreamHandler-java.lang.String-)
* [java.util.EnumSet](https://docs.oracle.com/javase/8/docs/api/java/util/EnumSet.html#of-E-)
* [javax.xml.bind.JAXBContext](https://docs.oracle.com/javase/8/docs/api/javax/xml/bind/JAXBContext.html#createMarshaller--)