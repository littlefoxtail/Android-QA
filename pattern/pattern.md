[设计模式百科](http://www.baike.com/wiki/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F)
在面向对象软件系统的设计而言，在支持可维护性的同时，提高系统的可复用性是一个至关重要的问题

设计模式原则：
- 单一职责原则(Single Responsibility Principle,SRP) 一个类只负责一个功能领域的相应职责
它用于控制类的粒度大小。一个类承担的职责越多它被复用的可能就越小，而且一个类承担的职责过多，就相当于将这个职责耦合在一起，一个职责变化会影响另一个职责。
单一职责原则实现高内聚、低耦合的指导方针，它是最简单但又最难运用的原则
- 开闭原则(Open Close Principle,OCP)：软件实体应对扩展开放，而对修改关闭
为了满足开闭原则，需要对系统进行抽象化设计，抽象化是开闭原则的关键
通过定义系统的抽象层，再通过具体类来进行扩展。如果需要修改系统的行为，无须对抽象层进行任何改动，只需要增加新的具体类来实现新的业务功能
- 里氏代换原则（Liskov Subsititution Principle,LSP）：父类出现的地方，子类也可出现
- 依赖倒转原则（Dependence Inversion Principle,DIP）：抽象不应该依赖于细节，细节应该依赖于抽象
- 接口隔离原则（Interface Segregation Principle,ISP）：多个隔离的接口，比使用单个接口要好
- 合成复用原则(Composite Reuse Principle,CRP)：尽量使用合成/聚合的方式，而不是使用继承
- 迪米特法则(最少知道原则)(Demeter Principle,LoD)：最少知道原则。一个实体应当尽量少的与其他实体之间发生相互作用

## 创建型模式（六种）
在面向对象程序设计中，工厂通常是一个用来创建其他对象的对象。工厂是构造方法的抽象，用来实现不同的分配方案。
有时，特定类型对象的控制过程比简单地创建一个对象更复杂。在这种情况下，工厂对象就派上用场了。工厂对象可能会动态地创建产品对象的类，或者从对象池中返回一个对象

### 简单工厂(Simple Factory)
简单工厂并不属于GoF23个经典设计模式，但通常将它作为学习其他工厂模式的基础
一个工厂类，它可以根据参数的不同返回不同类的实例，被创建的实例通常都具有共同的父类。因为在简单工厂模式中用于创建实例的方法是静态方法
因此简单工厂模式又被称为静态工厂方法(Static Factory Method)模式，属于类创建型模式

普通的工厂方法模式通常伴随着对象的具体类型与工厂具体的类型的一一对应，客户端代码根据需要选择合适具体类型工厂使用。然而，这种选择可能包含复杂的逻辑。这时，可以创建一个单一的工厂类，
以包含这种逻辑选择，根据参数的不同选择实现不同的具体对象

```java
public class ImageReaderFactory {
    public static ImageReader imageReaderFactoryMethod(InputStream is) {
        ImageReader product = null;

        int imageType = determineImageType(is);
        switch (imageType) {
            case ImageReaderFactory.GIF:
                product = new GifReader(is);
            case ImageReaderFactory.JPEG:
                product = new JpegReader(is);
            //...
        }
        return product;
    }
}
```

#### 优点
* 工厂类抱哈必要的判断逻辑，可以决定在什么时候创建哪一个产品类的实例，客户端可以免除直接创建产品对象的职责，实现创建和使用的分离
* 客户端无须知道所创建的具体产品类的类名，只需要知道具体产品类所对应的参数即可

#### 缺点
* 由于工厂类集中了所有产品的创建逻辑，职责过重
* 使用简单工厂模式势必增加类的个数，增加系统复杂度和理解难度
* 系统扩展困难，一旦添加新产品就得改工厂逻辑，不利于系统扩展和维护
* 简单工厂由于使用静态结构，造成工厂角色无法形成与继承的等级结构
#### 使用场景
* 工厂类负责创建的对象比较少
* 客户端只知道工厂类的参数，对于创建对象不关心

### 名称：工厂方法模式(Factory Method)
工厂方法模式又称为工厂模式，也叫虚拟构造器(Virtual Constructor)模式或者多态工厂，它属于类创建型。

在工厂方法模式中，不再提供一个统一的工厂类来创建所有的产品对象，而是针对不同的产品提供不同的工厂
系统提供一个域产品等级结构对应的工厂等级结构

在工厂方法模式中，工厂父类负责定义创建产品对象的公共接口，而工厂子类则负责生产具体的产品对象，这样做的目的是将产品类的实例化操作延迟到工厂子中完成。

工厂方法模式提供了一个抽象工厂接口来声明抽象工厂方法，而由其子类来具体实现工厂方法，创建具体的产品对象
工厂父类:
```java
public interface Blacksmith {
    Weapon manufactureWeapon(WeaponType weaponType);
}
```
工厂子类：
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
客户端：
```java
OrcBlacksmith ob = new OrcBlacksmith();
ob.manufactureWeapon(WeaponType.SPEAR)；
ob.manufactureWeapon(WeaponType.AXE);
```

* 意图：定义一个用于创建对象的接口，让子类决定实例化哪一个类。Factory Method使一个类的实例化延迟到其子类。
* 适用性：
    1. 客户端不知道它所需要的对象的类。在工厂方法模式中，客户端不需要知道具体产品类的类名。
    2. 当一个类希望由它的子类来指定它所创建的对象的时候。
    3. 当类将创建对象的职责委托给多个帮助子类中的某一个，并且你希望将哪一个帮助子类是代理者这一信息局部化的时候。

#### 模式分析
工厂方法模式是简单工厂模式的进一步抽象和推广。由于使用了面向对象的多态性，工厂方法模式保持了简单工厂模式的有点，而且克服了它的缺点。在工厂方法模式中，核心的工厂类不在负责所有产品的创建，而是将具体创建工作交给子类去做。这个核心类仅仅负责给出具体工厂必须实现的接口，而不负责哪一个产品类被实例化这种细节，这使得工厂方法模式可以允许系统在不修改工厂角色的情况下引入新的产品。

#### 优点
* 工厂方法隐藏了具体产品类将被实例化这一细节，用户只需要关心所需产品的工厂
* 基于工厂角色和产品角色的多态性设计是工厂方法模式的关键。它能够让工厂可以自主确定创建哪种产品对象
* 加入一个新产品时，无须修改抽象工厂和抽象产品提供的接口，只要添加一个具体工厂类和具体产品就可以，符合“开闭原则”

#### 缺点
* 在添加新产品时，需要编写新的具体产品类，而且还要提供与之对应的具体工厂类，系统中类的个数将成对增加，在一定程度上增加了系统的复杂度，有更多的类需要编译和运行，会给系统带来一些额外的开销
* 由于考虑到系统的可扩展性，需要引入抽象层，在客户端代码中均使用抽象层进行定义，增加了系统的抽象性和理解难度

#### Known uses

* [java.util.Calendar](http://docs.oracle.com/javase/8/docs/api/java/util/Calendar.html#getInstance--)
* [java.util.ResourceBundle](http://docs.oracle.com/javase/8/docs/api/java/util/ResourceBundle.html#getBundle-java.lang.String-)
* [java.text.NumberFormat](http://docs.oracle.com/javase/8/docs/api/java/text/NumberFormat.html#getInstance--)
* [java.nio.charset.Charset](http://docs.oracle.com/javase/8/docs/api/java/nio/charset/Charset.html#forName-java.lang.String-)
* [java.net.URLStreamHandlerFactory](http://docs.oracle.com/javase/8/docs/api/java/net/URLStreamHandlerFactory.html#createURLStreamHandler-java.lang.String-)
* [java.util.EnumSet](https://docs.oracle.com/javase/8/docs/api/java/util/EnumSet.html#of-E-)
* [javax.xml.bind.JAXBContext](https://docs.oracle.com/javase/8/docs/api/javax/xml/bind/JAXBContext.html#createMarshaller--)

### 名称：抽象工厂模式(Abstract Factory)
工厂方法模式通过引入工厂等级结构，解决了简单工厂类职责太重的问题，但由于工厂方法模式中的每一个工厂只生产一类产品，可能会导致系统中存在大量的工厂类。

抽象工厂模式提供一个创建一系列相关或相互依赖对象的接口，而无须指定它们具体的类。抽象工厂模式又称为Kit模式，属于对象创建型模式。
可以将一组具有同一主题的单独的工厂封装起来，正常使用中，客户端程序需要创建抽象工厂的具体实现，然后使用抽象工厂作为接口来创建这一主题的具体对象。

```java
public interface KingdomFactory {
    Castle createCastle();

    King createKing();

    Army createArmy();
}
```

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
* 意图：提供一个创建一系列相关或相互依赖对象的接口，而无需指定它们具体的类。
* 适用性：
    1. 一个系统要独立于它的产品的创建、组合和表示时。
    2. 一个系统要由多个产品系列中的一个来配置时。
    3. 当你要强调一系列相关的产品对象的设计，这一约束必须在系统的设计中体现出来，同一个产品族的产品可以每月任何关系，但它们都具有一些共同的约束
    4. 产品等级结构稳定，设计完成之后，不会向系统中增加新的产品等级结构或者删除已有的产品等级结构

#### 缺点
- 在添加新的产品对象时，难以扩展抽象工厂来生产新种类的产品，这是因为在抽象工厂角色中规定了所有可能被创建的产品集合，要支持新种类的产品就意味着要对接口进行扩展，而这将涉及到对抽象工厂角色以及其所有子类的修改。
- 开闭原则的倾斜性(增加新的工厂和产品组容易，增加新的产品等级结构麻烦)。
#### Real world examples

* [javax.xml.parsers.DocumentBuilderFactory](http://docs.oracle.com/javase/8/docs/api/javax/xml/parsers/DocumentBuilderFactory.html)
* [javax.xml.transform.TransformerFactory](http://docs.oracle.com/javase/8/docs/api/javax/xml/transform/TransformerFactory.html#newInstance--)
* [javax.xml.xpath.XPathFactory](http://docs.oracle.com/javase/8/docs/api/javax/xml/xpath/XPathFactory.html#newInstance--)

### 名称：建造者模式（Builder）
*通俗来讲： 允许您创建不同的对象风格，同时避免构造器污染。 当可能有几种风格的对象时很有用。 或者当创建对象时涉及很多步骤*

将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。
建造者模式是一步一步创建一个复杂的对象，它允许用户只通过指定复杂对象的类型和内容就可以构建它们，用户不需要知道内部的具体构建细节。

复杂对象：指那些包含多个成员属性的对象，这些成员属性也称为部件或零件

```java
public final class Hero {
    private final Profession profession;
    private final String name;
    private final HairType hairType;
    private final HairColor hairColor;
    private final Armor armor;
    private final Weapon weapon;

    
    private Hero(Builder builder) {
        this.profession = builder.profession;
        this.name = builder.name;
        this.hairColor = builder.hairColor;
        this.hairType = builder.hairType;
        this.weapon = builder.weapon;
        this.armor = builder.armor;
    }
}
```

builder:
```java
public static class Builder {
    private final Profession profession;
    private final String name;
    private HairType hairType;
    private HairColor hairColor;
    private Armor armor;
    private Weapon weapon;
    public Builder(Profession profession, String name) {
        if (profression == null || name == null) {
            throw new IllegalArgumentException("not be null")
        }
        this.profession = profession;
        this.name = name;
    }

    public Builder withHairType(HairType hairType) {
        this.hairType = hairType
        return this;
    }

    public Builder withHairColor(HairColor hairColor) {
        this.hairColor = hairColor;
        return this;
    }

    public Builder withArmor(Armor armor) {
        this.armor = armor;
        return this;
    }

    public Builder withWeapon(Weapon weapon) {
        this.weapon = weapon;
        return this;
    }

    public Hero build() {
        return new Hero(this);
    }

}
```

used:
```java
Hero mage = new Hero.Builder(Profession.Name, "Riobard").withHairColor(HairColor.BLACK).withWeapon(WeaponD.AGGER).build();
```

* 意图：将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。
* 适用性：
    1. 当创建复杂对象的算法应该独立于该对象的组成部分以及它们的装配方式时。
    2. 当构造过程必须运行被构造的对象有不同的表示时。


#### 优点
* 在建造者模式中，客户端不必知道产品内部组成的细节，将产品本身与产品的创建过程解耦，使得相同的创建过程可以创建不同的产品对象
* 每一个具体建造者都相对独立，而与其他的具体建造者无关，因此可以很方便地替换具体建造者或增加新的具体建造者，用户使用不同的具体建造者即可得到不同的产品对象。
* 可以更加精细地控制产品的创建过程。将复杂产品的创建步骤分解在不同的方法中，使得创建过程更加清晰，也更方便使用程序来控制创建过程。
* 增加新的具体建造者无需修改原有类库的代码，指挥者类针对抽象建造者类编程，系统扩展方便，符合“开闭原则”

#### 缺点
* 建造者模式所创建的产品一般具有较多的共同点，其组成部分相似，如果产品之间的差异性很大，则不适合使用建造者模式，因此其使用范围受到一定的限制。
* 如果产品的内部变化复杂，可能会导致需要定义很多具体建造者类来实现这种变化，导致系统变得很庞大。


#### Real world examples

* [java.lang.StringBuilder](http://docs.oracle.com/javase/8/docs/api/java/lang/StringBuilder.html)
* [java.nio.ByteBuffer](http://docs.oracle.com/javase/8/docs/api/java/nio/ByteBuffer.html#put-byte-) as well as similar buffers such as FloatBuffer, IntBuffer and so on.
* [java.lang.StringBuffer](http://docs.oracle.com/javase/8/docs/api/java/lang/StringBuffer.html#append-boolean-)
* All implementations of [java.lang.Appendable](http://docs.oracle.com/javase/8/docs/api/java/lang/Appendable.html)
* [Apache Camel builders](https://github.com/apache/camel/tree/0e195428ee04531be27a0b659005e3aa8d159d23/camel-core/src/main/java/org/apache/camel/builder)

### 名称：原型模式（Prototype）
* 意图：用原型实例指定创建对象的种类，并且通过拷贝这些原型创建新的对象。
* 适用性：①当要实例化的类是在运行时刻指定时，例如，通过动态装载；或者为了避免创建一个与产品类层次平行的工厂类层次时；或者当一个类的实例只能有几个不同状态组合中的一种时。建立相应数目的原型并克隆它们可能比每次用合适的状态手工实例化该类更方便一些。

### 名称：单例模式（Singleton）
* 意图：保证一个类仅有一个实例，并提供一个访问它的全局访问点。
* 适用性：
    1. 当类只能有一个实例而且客户可以从一个众所周知的访问点访问它时。
    2. 当这个唯一的实例应该是通过子类化可扩展的，并且客户应该无需更改代码就能使用一个扩展的实例时。

#### 优点
* 单例模式提供了对唯一实例的受控访问。因为单例模式封装了它的唯一实例，所以它可以严格控制客户这样以及何时访问它
* 节省系统资源，对一些频繁创建和销毁的对象单例模式无疑可以提高系统的性能
* 允许可变数目的实例。基于单例模式可以进行扩展，使用与单例控制相似的方法来获得指定个数的对象实例

#### 缺点
* 由于单例模式中没有抽象层，因此单例类的扩展有很大的困难
* 单例类的职责过重，在一定程度上违背了单一职责。因为单例类既充当了工厂角色，提供了工厂方法，同时又充当了产品角色，包含一些业务方法，将产品的创建和产品的本身的功能融合到一起
## 结构型模式（七种）
### 名称：Adapter(适配器模式)
在适配器模式中可以定义一个包装类，包装不兼容接口的对象，这个包装类指的就是适配器(Adapter)，它所包装的对象就是是适配者(Adaptee)，即被适配的类。
适配器提供客户类需要的接口，适配器的实现就是把客户类的请求转化为对适配者的相应接口的调用。也就是说：当客户类调用适配器的方法时，在适配器类的内部将调用适配者类的方法，而这个过程对客户类是透明的，客户类并不直接访问适配者类。因此，适配器可以使由于接口不兼容而不能交互的类可以一起工作。这就是适配器模式的模式动机

适配模式(Adapter Pattern)：将一个接口转换成客户希望的另一个接口，适配器模式使接口不兼容的那些类可以一起工作，其别名为包装(Wrapper)。适配器模式既可以做为类结构型模式，也可以作为对象结构型模式。

Client
```java
public class Captain implements RowingBoat {
    private RowingBoat rowingBoat;

    public Captain() {

    }

    public Captain(RowingBoat rowingBoat) {
        this.rowingBoat = rowingBoat;
    }

    public void setRowingBoat(RowingBoat rowingBoat) {
        this.rowingBoat = rowingBoat;
    }

    @Override
    public void row() {
        rowingBoat.row();
    }
}
```

Adaptee
```java
public class FishingBoat {

    public void sail() {
        log.i("The fishing boat is sailing")
    }
}
```

Adapter
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


* 意图：将一个类的接口转换成客户希望的另一个接口。adapter模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。
* 适用性：
    1. 你想使用一个已经存在的类，而它的接口不符合你的需求。
    2. 你想创建一个可以复用的类，该类可以与其他不想关或不可预见的类（即那些接口可能不一定兼容的类）协同工作。
    3. (仅使用对象adapter)你想使用一些已经存在的子类，但是不可能对每一个都进行子类化以匹配它们的接口。对象适配器可以适配它的父类接口。

### 优点
* 将目标类和适配者类解耦，通过引入一个适配类来重用现有的适配者类，而无需修改原有代码
* 增加类的透明性和复用性，将具体的实现封装在适配者类中，对于客户端类来说透明的，而且了提高了适配者的复用性
* 灵活性和扩展性都非常好，通过使用配置文件，可以很方便地更换适配器，也可以在不修改原有代码的基础上增加新的适配器类，符合“开闭原则”

**类适配模式具有如下优点**
由于适配器类适配者类的子类，因此可以在适配器类中置换一些适配者的方法，使得适配的灵活性更强

**对象适配器模式具有如下优点**
一个对象适配器可以把多个不同的适配者适配到同一个目标，也就是说，同一个适配器可以把适配者类和它的子类都适配到目标接口

#### 缺点
类适配器对于不支持多重继承的语言，一次最多只能适配一个适配者类，而且目标抽象类只能为抽象，不能为具体类，其使用有一定的局限性，不能讲一个适配者类和它的子类都适配到目标接口

对象适配器模式置换时适配者的方法不容易

#### Real world examples
* [java.util.Arrays#asList()](http://docs.oracle.com/javase/8/docs/api/java/util/Arrays.html#asList%28T...%29)
* [java.util.Collections#list()](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#list-java.util.Enumeration-)
* [java.util.Collections#enumeration()](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#enumeration-java.util.Collection-)
* [javax.xml.bind.annotation.adapters.XMLAdapter](http://docs.oracle.com/javase/8/docs/api/javax/xml/bind/annotation/adapters/XmlAdapter.html#marshal-BoundType-)


### 名称：Bridge(桥接模式)
桥接模式将继承关系转换为关联关系，从而降低类与类之间的耦合，减少了代码编写量。


* 意图：将抽象部分与它的实现部分分离，使它们都可以独立地变化。
 (其实就是，子类有两个维度的排列组合 用桥接模式)
  脱耦：脱耦就是将抽象化和实现化之间的耦合解脱开，或者说是将它们之间的强关联换成弱关联，将两个角色之间的继承关系
  改为关联关系。桥接模式中所谓的脱耦，就是指一个软件系统的抽象化和实现化之间使用关联关系而不是继承关系。
* 适用性：
    1. 你不希望在抽象和它的实现部分之间有一个固定的绑定关系。例如这种情况可能因为，在程序运行时刻实现部分应可以被选择或者切换。
    2. 类的抽象以及它的实现都可以通过生成子类的方法加以扩充。这时Bridge模式使你可以对不同的抽象接口和实现部分进行组合，并分别对它们进行扩充。
    3. 对一个抽象的实现部分的修改应对客户不产生影响，即客户的代码不必重新编译。
    4. 有许多类要生成。这样一种类层次结构说明你必须将一个对象分解成两个部分。称这种类层次结构为“嵌套的普化”（nested generalizations ）
    5. 你想在多个对象间共享实现，但同时要求客户并不知道这一点。

```java
public interface Weapon {
    void wield();

    void swing();

    void unwield();

    Enchantment getEnchantment();
}
```

```java
public class Sword implements Weapon {
    private final Enchantment enchantment;

    public Sword(Enchantment enchantment) {
        this.enchantment = enchantment;
    }

    @Override
    public void wield() {
        log.i("咖喱棒");
        enchantment.onActivate();
    }

    @Override
    public void swing() {
        log.i("那么多路人")
        enchantment.apply();
    }

    @Override
    public void unwield() {
        log.i("甲");
        enchantment.onDeactivate();
    }

    @Override
    public Enchantment getEnchantment() {
        return enchantment;
    }
}
```

```java
public class Hammer implements Weapon {

    private static Enchantment enchantment;

    public Hammer(Enchantment enchaantment) {
        this.enchantment = enchantment;
    }

    @Override
    public void wield() {
        log.i("星星")
        enchantment.onActivate();
    }

    @Override
    public void swing() {
        enchantment.apply();
    }

    @Override
    public void unwield() {
        enchantment.onDeactivate();
    }

    @Override
    public void unwield() {
        enchantment.onDeactivate();
    }

    @Override
    public Enchantment getEnchantment() {
        return enchantment;
    }
}
```

```java
public interface Enchantment {
    void onActivate();

    void apply();

    void onDeactivate();
}
```

```java
public class FlyingEnchantment implements Enchantment {
    public void onActiviate() {
        log.i("🐶");
    }

    public void apply() {
        log.i("fly apply");
    }

    public void onDeactivate() {
        log.i("fly ondeactivate");
    }
}
```

```java
public class SoulEatingEnchantment implements Enchantment {
    public void onActivate() {
        log.i("soul activate");
    }

    public void apply() {
        log.i("soul apply");
    }

    public void onDeactivate() {
        log.i("soul deactivate")
    }
}
```

#### 优点
- 分离抽象接口及其实现部分
- 桥接模式有时类似于多继承方案，但是多继承方案违背了类的单一职责原则，复用性较差，而且多继承结构中类的个数非常庞大，桥接模式是比多继承方案更好的解决方法。
- 桥接模式提高了系统的可扩充性，在两个变化唯独中任意扩展一个维度，都不需要修改原有系统。
- 实现细节对客户透明，可以对用户隐藏实现细节

#### 缺点
- 桥接模式引入会增加系统的理解和设计难度，由于聚合关联关系建立在抽象层，要求开发者针对抽象进行设计和编程
- 桥接模式要求正确识别出系统中两个独立变化的维度，因此其使用范围有一定的局限

### 名称：Composite(组合模式)
* 意图：将对象组合成树形结构以表示"部分-整体"的层次结构。composite使得用户对单个对象和组合对象的使用具有一致性。
* 适用性：
    1. 你想表达对象的部分-整体层次结构。
    2. 你希望用户忽略组合对象与单个对象的不同，用户将统一地使用组合机构中的所有对象。

组合模式让你可以优化处理递归或分级数据结构。计算机文件系统就是以递归结构来组织的。如果你想要描述这样的数据结构，可以使用组合模式。

组合模式解耦了客户程序与复杂元素内部结构，从而使客户端程序可以像处理简单元素一样处理复杂元素。

#### Real world examples

* [java.awt.Container](http://docs.oracle.com/javase/8/docs/api/java/awt/Container.html) and [java.awt.Component](http://docs.oracle.com/javase/8/docs/api/java/awt/Component.html)
* [Apache Wicket](https://github.com/apache/wicket) component tree, see [Component](https://github.com/apache/wicket/blob/91e154702ab1ff3481ef6cbb04c6044814b7e130/wicket-core/src/main/java/org/apache/wicket/Component.java) and [MarkupContainer](https://github.com/apache/wicket/blob/b60ec64d0b50a611a9549809c9ab216f0ffa3ae3/wicket-core/src/main/java/org/apache/wicket/MarkupContainer.java)


### 名称：Facade(外观模式)
* 意图：为子系统中的一组接口提供一个一致的界面，Facad模式定义了一个高层接口，这个接口使得这一子系统更加容易使用。
* 适用性：①当你要为一个复杂子系统提供一个简单接口时。子系统往往因为不断演化而变得越来越复杂。大多数模式使用时都会产生更多更小的类。
这使得子系统更具可重用性，也更容易对子系统进行定制，但这也给那些不需要定制的子系统的用户带来一些使用上的困难。facad可以提供一个简单的缺省视图，
这一视图对大多数用户来说已经足够，而那些需要更多的可定制性的用户可以越过facade层。
②客户程序与抽象类的实现部分之间存在着很大的依赖性。引入facade将这个子系统与客户以及其他子系统分离，可以提高子系统的独立性和可移植性。
③当你需要构建一个层次结构的子系统时，使用facade模式定义子系统中每层的入口点。如果子系统之间是相互依赖的，你可以让它们仅通过facade进行通讯，从而简化了它们之间的依赖关系。

### 名称：Flyweight(享元模式)
* 意图：运用共享技术有效地支持大量细粒度的对象。
* 适用性：①一个应用程序使用了大量的对象。
②完全由于使用大量的对象，造成很大的存储开销。
③对象的大多数状态都可变为外部状态。
④如果删除对象的外部状体，那么可以用相对较少的共享对象取代很多组对象。
⑤应用程序不依赖于对象标识。由于Flyweight对象可以被共享，对于概念上明显有别的对象，标识测试将返回真值。


### 名称：Decorator(装饰器模式)
装饰模式以对客户透明的方式动态地给一个对象附件上更多的责任，换言之，客户端并不会觉得对象在装饰前和装饰后有什么不同。装饰模式可以在不需要创造更多子类的情况下，将对象的功能加以扩展。这就是装饰模式的动机。

* 意图：动态地给一个对象添加一些额外的职责。就增加功能来说，decorator模式相比生成子类更为灵活。其别名也可以称为包装器(Wrapper)，与适配器模式的别名相同。
* 适用性：
    1. 在不影响其他对象的情况下，以动态、透明的方式给单个对象添加职责
    2. 处理那些不可撤销的职责。
    3. 当不能生成子类的方法进行扩充时。一种情况是，可能有大量的扩展，为支持每一种组合将产生大量的子类，使得子类数据呈爆炸性增长。
另一种情况可能是因为类定义被隐藏，或类定义不能用于生成子类。
Interface:
```java
public interface Troll {
    void attach();

    int getAttackPower();

    void fleeBattle();
}
```
Decorator:
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

```java
Troll troll = new SimpleTroll();
troll.attack();
troll.fleeBattle();

troll = new ClubbedTroll(troll);
troll.attack();
troll.fleeBattle();
```
#### 优点
- 装饰模式与继承关系的目的都是要扩展对象的功能，但是装饰模式可以提供比继承更多的灵活性。
- 可以通过一种动态的方式来扩展一个对象的功能，通过配置文件可以在运行时选择不同的装饰器，从而实现不同的行为。
- 通过使用不同的具体装饰器以及这些装饰类的排列组合，可以创造出很多不同行为的组合。可以使用多个具体装饰类来装饰同一对象，得到功能更为强大的对象。
- 具体构建类与具体修饰类可以独立变化，用户可以根据需要增加新的具体构件类和具体修饰类，在使用时再对其进行组合，原有代码无须改变，符合"开闭原则"

#### 缺点
* 客户端必须知道所有的策略类，并自行决定使用哪一个策略类
* 策略模式将造成系统产生很多具体具体策略类
* 无法同时在客户端使用多个策略类，不支持使用一个策略类完成部分功能后再使用另一个策略类来完成剩余功能的情况

#### 使用场景
- 一个系统需要动态地在几种算法中选择一种
- 一个对象有很多行为，如果不恰当的模式，这些行为就只要使用多重条件选择语句来实现。此时，使用策略模式，把这些行为转移到相应的具体策略类里面，就可以避免使用难以维护的多重条件选择语句
- 不希望客户端知道复杂、与算法的数据结构

#### Real world examples
 * [java.io.InputStream](http://docs.oracle.com/javase/8/docs/api/java/io/InputStream.html), [java.io.OutputStream](http://docs.oracle.com/javase/8/docs/api/java/io/OutputStream.html),
 [java.io.Reader](http://docs.oracle.com/javase/8/docs/api/java/io/Reader.html) and [java.io.Writer](http://docs.oracle.com/javase/8/docs/api/java/io/Writer.html)
 * [java.util.Collections#synchronizedXXX()](http://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#synchronizedCollection-java.util.Collection-)
 * [java.util.Collections#unmodifiableXXX()](http://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#unmodifiableCollection-java.util.Collection-)
 * [java.util.Collections#checkedXXX()](http://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#checkedCollection-java.util.Collection-java.lang.Class-)

### 名称：Proxy(代理模式)
* 意图：为其他对象提供一种代理以控制对这个对象的访问。
* 适用性：在需要比较通用和复杂的对象指针代替简单的指针的时候，使用Proxy模式，下面是一些可以使用Proxy模式常用情况：
1. 远程代理(Remote Proxy)为一个对象在不同的地址空间提供局部代理。
2. 虚代理(Virtual Proxy)根据需要创建开销很大的对象
3. 保护代理（Protection Proxy）控制对原始对象的访问。保护代理用于对象应该有不同的访问权限的时候。
4. 智能指引（Smart Reference）取代了简单的指针，它在访问对象时只需一些附加操作，典型用途：
1.对指向实际对象的引用计数，这样当该对象没有引用时，可以自动释放它
2.当第一次引用一个持久对象时，将它装入内存。
3.在访问一个实际对象前，检查是否已经锁定了它，以确保其他对象不能改变它。

```java
public interface WizardTowser {
    void enter(Wizard wizard);
}

public class IvoryTowser implements WizardTowser {
    public void enter(Wizard wizard) {
        log.i("看不见");
    }
}
```

```java
public class Wizard {
    private final String name;
    public Wizard(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return name;
    }
}
```
```java
public class WizardTowerProxy implements WizardTower {
    private static final int NUM_WIZARDS_ALLOWED = 3;

    private int numWizards;

    private final WizardTower tower;

    public WizardTowerProxy(WizardTower tower) {
        this.tower = tower;
    }

    public void enter(Wizard wizard) {
        if (numWizards < NUM_WIZARDS_ALLOWED) {
            tower.enter(wizard);
            numWizards++;
        } else {
            Log.i("allowed to enter");
        }
    }
}
```

#### Real world examples

* [java.lang.reflect.Proxy](http://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Proxy.html)
* [Apache Commons Proxy](https://commons.apache.org/proper/commons-proxy/)
* Mocking frameworks Mockito, Powermock, EasyMock

## 行为型模式(十一种)
### 名称：Chain of Responsibility(责任链模式)
* 意图：使多个对象都有机会处理请求，从而避免请求的发送者和接受者之间的耦合关系。将这些对象连成一条链，并沿着这条链传递该请求，直到有一个对象处理它为止。
* 适用性：①有多个的对象可以处理一个请求，哪个对象处理该请求运行时刻自动确定。
②你想在不明确指定接收者的情况下，向多个对象中的一个提交一个请求。
③可处理一个请求的对象集合应被动态指定。

```java
public class Requst {
    private final RquestType requestType;
    private final String requestDescription;
    private boolean handled;

    public Request(final RquestType requestType, final String requestDecription) {
        this.requestType = Objects.requestNonNull(requestType);
        this.requestDescription = Object.requestNonNull(requestDescription);
    }

    public String getRequestDescription() { return requestDescription; }

    public RequestType getRequestType() { return requestType; }

    public void markHandled() { this.handled = true; }

    public boolean isHandled() { return this.handled; }

    @Override
    public String toString() { return getRequestDescription(); }
    }

    public enum RequestType {
    DEFEND_CASTLE, TORTURE_PRISONER, COLLECT_TAX
    }
```

```java
public abstract class RequestHandler {
    private static final LOGGER = LoggerFactory.getLogger(RequestHandler.class);
    private RequestHandler next;

    public RequestHandler(RequestHandler next) {
        this.next = next;
    }
    public void handleRequest(Request req) {
        if (next != null) {
            next.handleRequest(req);
        }
    }

    protected void printHandling(Request req) {
        LOGGER.info("{} handling rquest\"{}\"", this, req);
    }

    @Override
    public abstract String toString();
}


public class OrcCommand extends RequestHandler {
    public OrcCommand(RequestHandler handler) {
        super(handler);
    }

    @Override
    public void handleRequest(Request req) {
        if (req.getRequestType().equals(RequestType.DEFEND_CASTLE)) {
            printHanling(req);
            req.markHandled();
        } else {
            super.handleRequest(req);
        }
    }

    @Override
    public String toString() {
        return "Orc commander";
    }
}
```

```java
public class OrcKing {
    RequestHandler chain;

    public OrcKing() {
        buildChain();
    }

    private void buildChain() {
        chain = new OrcCommander(new OrcOffice(new OrcSoldier(null)));
    }

    public void makeRequest(Request req) {
        chain.handleRequest(req);
    }
}
```

### 名称：Command(命令模式)
需要向某些对象发送请求（调用其中的某个或某些方法），但并不知道请求的接收者是谁，也不知道请求的操作是哪个，此时希望能够以一种松耦合的方式来设计软件

命令模式(Command Pattern)：将一个请求封装为一个对象，从而让我们可用不同的请求对客户进行参数化，对请求排队或者记录请求日志，以及支持可撤销的操作，命令模式是一种对象行为型模式，其别名为动作(Action)或事务(Transaction)模式
命令模式包含的角色：
- Command(抽象命令类)
```java
public abstract class Command {
    public abstract void execute(Target target);

    public abstract void undo();

    public abstract void redo();

    @Override
    publci abstarct String toString();
}
```
- ConcreteCommand(具体命令类)
```java
public class InvisibilitySpell extends Command {
    private Target target;

    @Override
    public void execute(Target target) {
        target.setVisibility(Visibility.INVISIBLE);
        this.target = target;
    }
    
    @Override
    public void undo() {
        if (target != null) {
            target.setVisibility(Visibility.INVISIBLE);
        }
    }

    @Override
    public void redo() {
        if (target != null) {
            target.setVisibility(Visibility.INVISIBLE);
        }
    }

    @Override
    public String toString() {
        return "Invisibility spell";
    }  
}
```

```java
public class ShrinkSpell extends Command {
    private Size oldSize;
    private Target target;

    @Override
    public void execute(Target target) {
        oldSize = target.getSize();
        target.setSize(Size.SMALL);
        this.target = target;
    }

    @Override
    public void undo() {
        if (oldSize != null && target != null) {
            Size temp = target.getSize();
            target.setSize(oldSize);
            oldSize = temp;
        }
    }

    @Override
    public void redo() {
        undo();
    }

    @Override
    public String toString() {
        return "Shrink spell";
    }

}
```

- Invoker(调用者)
```java
public class Wizard {
    private Deque<Command> undoStack =  new LinkedList<>();
    private Deque<Command> redoStack =  new LinkedList<>();

    public Wizard() {

    }

    public void castSpell(Command command, Target target) {
        command.execute(target);
        undoStack.offerLast(command);
    }

    public void undoLastSpell() {
        if (!unsoStack.isEmpty()) {
            Command previousSpell = undoStack.pollLast();
            redoStack.offerLast(previousSpell);
            previousSpell.undo();
        }
    }

    public void redoLastSpell() {
        if (!redoStack.isEmpty()) {
            Command previousSpell = redoStack.pollLast();
            undoStack.offerLast(previousSpell);
            previousSpell.redo();
        }
    }

    public void redoLastSpell() {
        if (!redoStack.isEmpty()) {
            Command previousSpell = redoStack.pollLast();
            previousSpell.redo();
        }
    }

    @Override
    public String toString() {
        return "Wizard";
    }
}
```

- Receiver(接收者)
```java
public abstract class Target {
    private Size size;
    private Visibility visibility;

    public Size getSize() {
        return size;
    }

    public void setSize(Size size) {
        this.size = size;
    }

    public Visibility getVisibility() {
        return visibility;
    }

    public void setVisibility(Visibility visibility) {
        this.visibility = visibility;
    }

    @Override
    public abstract String toString();

    public void printStatus() {

    }
}
```

```java
public class Goblin extends Target {
    public Goblin() {
        setSize(Size.NORMAL);
        setVisibility(Visibility.VISIBLE);
    }

    @Override
    public String toString() {
        return "Goblin";
    }
}
```

Client
```java
Wizard wizard = new Wizard();
Goblin goblin = new Goblin();

goblin.printStatus();

wizard.castSpell(new ShrinkSpell(), goblin);
goblin.printStatus();
```

命令队列可以将请求发送者和接受者解耦，请求发送者通过命令对象来间接引用请求接收者，使得系统具有更好的灵活性和可扩展性


* 意图：将一个请求封装为一个对象，从而使你可用不同的请求对客户进行参数化；对请求排队或记录请求日志，以及支持可撤销的操作。
* 适用性：
    - 抽象出待执行的动作以参数化某对象，你可用过程语言中的回调函数表达这种参数化机制。回调函数指函数在某处注册，而它将在稍后某个需要的时候被调用。command模式是回调机制的一个面向对象的替代品。
    - 在不同的时刻指定、排列和执行请求。一个Command对象可以有一个初始请求无关的生存期。如果一个请求的接受者可用一种与地址空间无关的方式表达，那么就将负责该请求的命令对象传送给另一个不同的进程并在那儿实现该请求。
    - 支持取消操作。command的Excute操作可在实施操作前将状态存储起来，在取消操作时这个状态用来消除该操作的影响，执行的命令将被存储在一个历史列表中。可通过向后和向前遍历这一列表并分别调用unExcute和Excute来实现重数不限的“取消”和“重做”。
    - 支持修改日志，这样当系统崩溃时，这些修改可以被重做一遍。在Command接口中添加装载操作和存储操作，可以用来保持变动的一个一致的修改日志。在崩溃中恢复的过程包括从磁盘中重新读入记录下来的命令并用excute操作重新执行它们。
    - 在构建的原语操作上的高层操作构造一组变动。command模式提供了对事务进行建模的方法。command有一个公共接口，使得你可以用同一种方式调用所有的事务。同时使用该模式也易于添加新事务以扩展系统。

#### 优点
- 降低耦合度。由于请求者与接收者之间不存在直接引用，因此请求者与接收者完全解耦
- 新的命令可以很容易地加入到系统中，符合开闭原则
- 可以比较容易设计一个命令队列或组合命令
- 为请求的撤销和恢复操作提供了一种设计和实现方案

#### 缺点
导致某些系统有过多的具体命令类

### 名称：Interpreter(解释器模式)
* 意图：给定一个语言，定义它的文法的一种表示，并定义一个解释器，这个解释器使用该表示来解释语言中的句子。
* 适用性：当有一个语言需要解释执行，并且你可将该语言中的句子表示一个抽象语法树时，可使用解释器模式。而当存在以下情况时该模式效果最好。
该文法简单对于复杂的文法，文法的类层次变得庞大而无法管理。此时语法分析程序生成器这样的工具是更好的选择。它们无需构建语言树即可解释表达式，这样可以节省空间而且可能节省时间。
效率不是一个关键问题最高效的解释器通常不是通过直接解释语法分析树实现的，而是首先将它们转换成另一种形式。

### 名称：Iterator(迭代子模式)
* 意图 提供一种方法顺序访问一个聚合对象中各个元素，而又不需要暴露该对象的内部表示。
* 适用性 访问一个聚合对象的内容而无需暴露它的内部表示
支持对聚合对象的多种遍历。
为遍历不同的聚合结构提供一个统一的接口(即支持多态迭代)

### 名称 Mediator(中介者模式)
* 意图: 用一个中介对象来封装一系列的对象交互。中介者使各对象不需要显示地相互引用，从而使耦合松散，而且可以独立地改变它们之间的交互。
* 适用性：①一组对象以定义良好但是复杂的方式进行通信。产生的相互依赖关系结构混乱且难以理解。
②一个对象引用其他很多对象并且直接与这些对象通信，导致难以复用该对象。
③想定制一个分布在对个类中的行为，而又不想生成太多的子类。

### 名称 Memento(备忘录模式)
* 意图 在不破坏封装性的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态，这样以后就可将该对象恢复到原先保存的状态。
* 适用性 必须保存一个对象在某一个时刻状态，这样以后需要时它才能恢复到先前的状态。
如果一个用接口来让其它对象直接得到这些状态，将会暴露对象的实现细节并破坏对象的封装性。

### 名称 Observer(观察者模式)
#### 模式动机
 定义对象间的一宗一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。发生改变的对象称为观察目标，而被通知的对象称为观察者，一个观察目标可以对应多个观察者，而且这些观察者之间没有相互联系，可以根据需要增加和删除观察者。
#### 模式定义
观察者模式(Observer Pattern):观察者模式又叫做(Publish/Subscribe)模式、模型-视图(Model/View)模式、源-监听器(Source/Listener)模式
Observed(被观察者):
```java
public class Weather {
    private WeatherType currentWeather;
    private List<WeatherObserver> observers;

    public Weather() {
        observers = new ArrayList<>();
        currentWeather = WeatherType.SUNNY;
    }

    public void addObserver(WeatherObserver obs) {
        observers.add(obs);
    }

    public void removeObserver(WeatherObserver obs) {
        observers.remove(obs);
    }

    public void timePasses() {
        WeatherType[] enumValues = WeatherType.values();
        currentWeather = enumValues[(currentWeather.ordinal() + 1) % enumValues.length];
        notifyObservers();
    }

    private void notifyObservers() {
        for (WeatherObserver obs : observers) {
            obs.update(currentWeather);
        }
    }
}
```

Observer:
```java
public interface WeatherObserver {
    void update(WeatherType currentWeather);
}
```

#### 优点
观察者模式的优点
- 观察者模式可以实现表示层和数据逻辑层的分离，并定义了稳定的消息更新传递机制，抽象了更新接口，使得可以有各种各样不同的表示层作为具体观察者角色。
- 观察者模式在观察目标和观察者之间建立了一个抽象的耦合。
- 观察者模式支持广播通信。
- 观察者模式符合"开闭原则"的要求

#### 缺点
- 如果一个观察目标对象有很多直接和间接的观察者的话，将所有的观察者都通知会花费很多时间
- 如果观察者和观察目标之间有循环依赖的话，观察目标会触发它们之间进行循环调用，可能导致系统崩溃
- 观察者模式没有相应的机制让观察者知道所观察的目标对象是怎么发生变化的，而仅仅只是知道观察目标发生了变化

* 适用性 
    1. 当一个抽象模型有两个方面，其中一个方面依赖于另一个方面。将两者封装在独立的对象中以使它们可以各自独立地改变和复用
    2. 当对一个对象的改变需要同时改变其它对象，而不知道具体有多少对象有待改变。
    3. 当一个对象必须通知其它对象，而它又不能假定其它对象是谁。换言之，你不希望这些对象是紧密耦合的。

### 名称 State(状态模式)
* 意图 允许一个对象在其内部状态改变
* 适用性 一个对象的行为取决于它的状态，并且它必须在运行时刻根据状态改变它的行为。
一个操作中含有庞大的多分支的条件语句，且这些分支依赖于该对象的状态。这状态通常用一个或多个枚举常量表示。通常，有多个操作包含这一相同的条件结构。
State模式将每一个条件分支放入一个独立的类中。这使得你可以根据对象自身的情况将对象的状态作为一个对象，这一对象可以不依赖于其他对象独立变化。

### 名称 Strategy(策略模式)
策略模式的主要目的是将算法的定义和使用分开，也就是将算法的行为和环境分开，将算法的定义放在专门的策略类中。
每一个策略类封装了一种实现算法，使用算法的环境类针对抽象策略类进行编程，符合“依赖倒转原则”

抽象策略类：
```java
public interface DragonSlayingStrategy {
    void execute();
}
```
具体策略类：
```java
public class MeleeStrategy implements DragonSlayingStrategy {
    public void execute() {

    }
}
```

```java
public class ProjectileStrategy implements DragonSlayingStrategy {
    public void execute() {

    }
}
```

```java
public class SpellStrategy implements DragonSlayingStragegy {
    public void execute() {

    }
}
```
环境类：
```java
public class DragonSlayer {
    private DragonSlayingStrategy strategy;

    public DragonSlayer(DragonSlayingStrategy strategy) {
        this.strategy = strategy;
    }

    public void changeStrategy(DragonSlayingStrategy strategy) {
        this.strategy = strategy;
    }

    public void goToBattle() {
        strategy.execute();
    }
}
```

#### 优点
* 策略模式提供了对“开闭原则”的完美支持，用户可以在不修改原有系统的基础选择算法或行为，也可以灵活地增加新的算法或行为
* 策略模式提供了管理相关的算法族的办法。策略类的结构等级结定义了一个算法或行为族，恰当使用继承可以把公共的代码移到抽象策略类中
* 策略模式提供了一种可以替换继承关系的办法
* 使用策略模式可以避免多重条件选择语句。多重条件语句不易维护，它把采取哪一种算法或行为的逻辑与算法或者行为本身的实现逻辑混合在一起，将它们全部硬编码(Hard Coding)在一个庞大的多重条件选择语句中，比直接继承环境类的办法还要原始和落后
* 策略模式提供了一种算法的复用机制，由于将算法单独提取出来封装在策略中，因此不同的环境类可以方便地复用这些策略类



* 意图 定义一系列的算法，把它们一个个封装起来，并且使它们可相互替换。本模式使得算法可独立于使用它的客户而变化。
* 适用性 许多相关的类仅仅是行为有异。“策略”提供了一种用多个行为中的一个行为来配置一个类的方法。需要使用一个算法的不同变体。
算法使用客户不应该知道的数据。可使用策略模式以避免暴露复杂的、与算法相关的数据结构。
一个类定义了多种行为，并且这些行为在这个类的操作中以多个条件语句的形式出现。将相关条件分支一如它们各自的Strategy类中以替代这些条件语句。

### 名称 Template Method(模板方法模式)
* 意图 定义一个操作中的算法的骨架，而将一些步骤延迟到子类中，Template Method使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。

* 适用性 一次性实现一个算法的不便部分，而将可变的行为留给子类来实现.各子类中的公共的行为应被提取出来并集中到一个公共父类中以避免代码重复。



### 名称 Visitor(访问者模式)

* 意图 表示一个作用于某对象结构中的各元素的操作。它使你可以在不改变各元素的前提下定义作用于这些元素的新操作。

* 适用性 一个对象结构包含很多类对象，它们由不同的接口，而你想对这些对象实施一些依赖于某具体类的操作。

需要对一个对象结构中的对象进行很多不同的并且不想关的操作，而你想避免让这些操作“污染”这些对象的类。

Visitor使得你可以将相关的操作集中起来定义在一个类中。当该对象结构被很多应用共享时，用Visitor模式让每个应用仅包含需要用到的操作。

定义的对象结构的类很少改变，但经常需要在此结构上定义新的操作。改变对象结构类需要重定义对所有访问者的接口，这可能需要很大的代价。如果对象结构类经常改变，那么可能还是在这些类中定义这些操作较好。

### Delegation委托模式
在委托模式中，有两个对象参与处理同一个请求，接受请求的对象将请求委托给另一个对象来处理，许多其他的模式，如状态模式、策略模式、访问者模式本质上是在更特殊的场合采用了委托模式。委托模式使得我们可以用聚合来替代继承

```java
public interface Printer {
    void print(final String message);
}
```

```java
public class HpPrinter implements Printer {
    @Override
    public void print(String message) {
        Log.i("谁都不行");
    }
}
```

```java
public class CanonPrinter implements Printer {
    @Override
    public void print(String message) {
        Log.i("鬼也不行");
    }
}
```

```java
public class EpsonPrinter implements Printer {
    @Override
    public void print(String messgae) {
        Log.i("神也不行");
    }
}
```

```java
public class PrinterController implements Printer {
    private final Printer printer;

    public PrinterController(Printer printer) {
        this.printer = printer;
    }

    @Override
    public void print(String message) {
        printer.print(message);
    }
}
```

```java
PrinterController hpPrinterController = new PrinterController(new HpPrinter());
PrinterController canonPrinterController = new PrinterController(new CanonPrinter());
PrinterController epsonPrinterController = new PrinterController(new EpsonPrinter());

hpPrinterController.printer("hello world!");
canonPrinterController.printer("hello world!");
epsonPrinterController.printer("hello world!");

```

