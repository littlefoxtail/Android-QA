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

# 创建型模式（六种）
在面向对象程序设计中，工厂通常是一个用来创建其他对象的对象。工厂是构造方法的抽象，用来实现不同的分配方案。
有时，特定类型对象的控制过程比简单地创建一个对象更复杂。在这种情况下，工厂对象就派上用场了。工厂对象可能会动态地创建产品对象的类，或者从对象池中返回一个对象

## 简单工厂(Simple Factory)
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
### 总结
简单工厂模式提供了专门的工厂类用于创建对象，将对象的创建和对象的使用分离开
#### 优点
* 工厂类抱哈必要的判断逻辑，可以决定在什么时候创建哪一个产品类的实例，客户端可以免除直接创建产品对象的职责，实现创建和使用的分离
* 客户端无须知道所创建的具体产品类的类名，只需要知道具体产品类所对应的参数即可
* 通过引入配置文件，可以在不修改任何客户端的情况下更换和增加新的具体产品类，在一定成效上提高了系统的灵活性

#### 缺点
* 由于工厂类集中了所有产品的创建逻辑，职责过重
* 使用简单工厂模式势必增加类的个数（引入新的工厂类），增加系统复杂度和理解难度
* 系统扩展困难，一旦添加新产品就得改工厂逻辑，不利于系统扩展和维护
* 简单工厂由于使用静态结构，造成工厂角色无法形成与继承的等级结构
#### 使用场景 
* 工厂类负责创建的对象比较少，不会造成工厂方法中的业务逻辑太过复杂
* 客户端只知道工厂类的参数，对于创建对象不关心

## 工厂方法模式(Factory Method)
工厂方法模式又称为工厂模式，也叫虚拟构造器(Virtual Constructor)模式或者多态工厂，它属于类创建型。

在工厂方法模式中，不再提供一个统一的工厂类来创建所有的产品对象，而是针对不同的产品提供不同的工厂
系统提供一个与产品等级结构对应的工厂等级结构

定义一个用于创建对象的接口，让子类决定将哪一个类实例化。工厂方法模式让一个类的实例化延迟到其子类

工厂方法模式提供了一个抽象工厂接口来声明抽象工厂方法，而由其子类来具体实现工厂方法，创建具体的产品对象

- Product（抽象产品）：它是定义产品的接口，是工厂方法模式所创建对象的超类型，也就是产品对象的公共父类
```java
public interface Weapon {
    WeaponType getWeaponType();
}
```

- ConcreteProduct（具体产品）：它实现了抽象产品接口，某些类型的具体产品由专门的具体工厂创建，具体工厂和具体产品之间一一对应
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

- Factory（抽象工厂）：抽象工厂类中，声明了工厂方法(Factory Method)，用于返回一个产品。抽象工厂是工厂方法模块的核心，所有创建对象的工厂方法都必须实现该接口
```java
public interface Blacksmith {
    Weapon manufactureWeapon(WeaponType weaponType);
}
```
ConcreteFactory（具体工厂）：它是抽象工厂类的子类，实现了抽象工厂中定义的工厂方法，并可由客户端调用，返回一个具体产品类的实例

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
- Client（客户端）：
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
### 总结
工厂方法模式是简单工厂模式的进一步抽象和推广。由于使用了面向对象的多态性，工厂方法模式保持了简单工厂模式的有点，而且克服了它的缺点。在工厂方法模式中，核心的工厂类不在负责所有产品的创建，而是将具体创建工作交给子类去做。这个核心类仅仅负责给出具体工厂必须实现的接口，而不负责哪一个产品类被实例化这种细节，这使得工厂方法模式可以允许系统在不修改工厂角色的情况下引入新的产品。

#### 优点
* 工厂方法隐藏了具体产品类将被实例化这一细节，用户只需要关心所需产品的工厂
* 基于工厂角色和产品角色的多态性设计是工厂方法模式的关键。它能够让工厂可以自主确定创建哪种产品对象
* 加入一个新产品时，无须修改抽象工厂和抽象产品提供的接口，只要添加一个具体工厂类和具体产品就可以，符合“开闭原则”

#### 缺点
* 在添加新产品时，需要编写新的具体产品类，而且还要提供与之对应的具体工厂类，系统中类的个数将成对增加，在一定程度上增加了系统的复杂度，有更多的类需要编译和运行，会给系统带来一些额外的开销
* 由于考虑到系统的可扩展性，需要引入抽象层，在客户端代码中均使用抽象层进行定义，增加了系统的抽象性和理解难度

#### 适用场景
- 客户端不知道它所需要的对象的类。在工厂方法模式中，客户端不需要知道具体产品类的类名。
- 抽象工厂类通过其子类来指定创建哪个对象
### Known uses
* [java.util.Calendar](http://docs.oracle.com/javase/8/docs/api/java/util/Calendar.html#getInstance--)
* [java.util.ResourceBundle](http://docs.oracle.com/javase/8/docs/api/java/util/ResourceBundle.html#getBundle-java.lang.String-)
* [java.text.NumberFormat](http://docs.oracle.com/javase/8/docs/api/java/text/NumberFormat.html#getInstance--)
* [java.nio.charset.Charset](http://docs.oracle.com/javase/8/docs/api/java/nio/charset/Charset.html#forName-java.lang.String-)
* [java.net.URLStreamHandlerFactory](http://docs.oracle.com/javase/8/docs/api/java/net/URLStreamHandlerFactory.html#createURLStreamHandler-java.lang.String-)
* [java.util.EnumSet](https://docs.oracle.com/javase/8/docs/api/java/util/EnumSet.html#of-E-)
* [javax.xml.bind.JAXBContext](https://docs.oracle.com/javase/8/docs/api/javax/xml/bind/JAXBContext.html#createMarshaller--)

## 抽象工厂模式(Abstract Factory)
工厂方法模式通过引入工厂等级结构，解决了简单工厂类职责太重的问题，但由于工厂方法模式中的每一个工厂只生产一类产品，可能会导致系统中存在大量的工厂类。

抽象工厂模式提供一个创建一系列相关或相互依赖对象的接口，而无须指定它们具体的类。抽象工厂模式又称为Kit模式，属于对象创建型模式。
可以将一组具有同一主题的单独的工厂封装起来，正常使用中，客户端程序需要创建抽象工厂的具体实现，然后使用抽象工厂作为接口来创建这一主题的具体对象。
- AbstractFactory（抽象工厂）：它声明了一组用于创建一族产品的方法，生成一组具体产品，这些产品构成了一个产品族，每一个产品都位于某个产品等级结构中
```java
public interface KingdomFactory {
    Castle createCastle();

    King createKing();

    Army createArmy();
}
```
- ConcreteFactory(具体工厂)：它实现了在抽象工厂中声明的创建产品的方法，生成一组具体产品，这些产品构成了一个产品族，每一个产品都位于某个产品等级结构中
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

- AbstractProduct(抽象产品)：它为每种产品声明接口，在抽象产品中声明了产品所具有的业务方法
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

- ConcreteProduct(具体产品)：它定义具体工厂生产的具体产品对象，实现抽象产品接口中声明的业务方法
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

### 总结
抽象工厂模式是工厂方法模式的进一步延伸，由于它提供了功能更为强大的工厂类并且具有较好的可扩展性。

#### 优点
- 抽象工厂模式隔离了具体类的生成，使得客户端不需要知道什么被创建。由于这种隔离，更换一个具体工厂就变得相对容易，所有的具体工厂都实现了抽象工厂中定义的那些公共接口
- 当一个产品族中的多个对象被设计成一起工作时，它能够保证客户端始终只使用同一个产品族中的对象
- 增加新的产品族很方便，无须修改已有系统，符合“开闭原则”

#### 缺点
- 增加新的产品等级结构麻烦，需要对原有系统进行较大的修改，甚至需要修改抽象层代码，违背了“开闭原则”

#### 适用场景
- 一个系统不应当依赖于产品类实例如何被创建、组合和表达的细节，这对于所有类型的工厂模式都是很重要的，用户无须关心对象的创建过程，将对象的创建和使用解耦
- 系统中有多于一个的产品族，而每次只使用其中某一产品族
- 属于同一个产品族的产品将在一起使用，这一约束必须在系统的设计中体现出来。同一个产品族中的产品可以是没有任何关系的对象，但是她们都具有一些共同的约束
- 产品等级结构稳定，设计完成之后，不会向系统中增加新的产品等级结构或者删除已有的产品等级结构
### Real world examples

* [javax.xml.parsers.DocumentBuilderFactory](http://docs.oracle.com/javase/8/docs/api/javax/xml/parsers/DocumentBuilderFactory.html)
* [javax.xml.transform.TransformerFactory](http://docs.oracle.com/javase/8/docs/api/javax/xml/transform/TransformerFactory.html#newInstance--)
* [javax.xml.xpath.XPathFactory](http://docs.oracle.com/javase/8/docs/api/javax/xml/xpath/XPathFactory.html#newInstance--)

## 建造者模式（Builder）
*通俗来讲： 允许您创建不同的对象风格，同时避免构造器污染。 当可能有几种风格的对象时很有用。 或者当创建对象时涉及很多步骤*

将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。
建造者模式是一步一步创建一个复杂的对象，它允许用户只通过指定复杂对象的类型和内容就可以构建它们，用户不需要知道内部的具体构建细节。

复杂对象：指那些包含多个成员属性的对象，这些成员属性也称为部件或零件。

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
- Builder（抽象建造者）：为创建一个产品对象的各个部件指定抽象接口，在该接口中一般声明两类方法，一类方法是buildPartX()，它们用于创建复杂对象的各个部件；另一类方法是getResult()，它们用于返回复杂对象，Builder既可以是抽象类，也可以是接口。

- ConcreteBuilder（具体建造者）:实现了Builder接口，实现各个部件的具体构造和装配方法，定义并明确它所创建的复杂对象，也可以提供一个方法创建返回好的复杂产品对象。
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

- Product（产品角色）：它是被构建的复杂对象，包含多个组成组件，具体建造者创建该产品的内部表示定义它的装配过程
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

  public Profession getProfession() {
    return profession;
  }

  public String getName() {
    return name;
  }

  public HairType getHairType() {
    return hairType;
  }

  public HairColor getHairColor() {
    return hairColor;
  }

  public Armor getArmor() {
    return armor;
  }

  public Weapon getWeapon() {
    return weapon;
  }
}
```

- Director（指挥者）:它负责安排复杂对象的建造次序，指挥者与抽象建造者之间存在关联关系，可以调用建造者对象部件构造与装配方法，完成复杂对象的建造，客户端一般只需要与指挥者交互，确定具体的建造者的类型

```java
public class App {
    public static void main(String[] args) {
        Hero mage = new Hero.Builder(Profession.Name, "Riobard").withHairColor(HairColor.BLACK).withWeapon(WeaponD.AGGER).build();
    }
}
```
### 总结
建造者的核心在于如何一步步构件一个包含多个组成部分的完整对象，使用相同的构建不同的产品。

#### 优点
* 在建造者模式中，客户端不必知道产品内部组成的细节，将产品本身与产品的创建过程解耦，使得相同的创建过程可以创建不同的产品对象
* 每一个具体建造者都相对独立，而与其他的具体建造者无关，因此可以很方便地替换具体建造者或增加新的具体建造者，用户使用不同的具体建造者即可得到不同的产品对象。
* 可以更加精细地控制产品的创建过程。将复杂产品的创建步骤分解在不同的方法中，使得创建过程更加清晰，也更方便使用程序来控制创建过程。
* 增加新的具体建造者无需修改原有类库的代码，指挥者类针对抽象建造者类编程，系统扩展方便，符合“开闭原则”

#### 缺点
* 建造者模式所创建的产品一般具有较多的共同点，其组成部分相似，如果产品之间的差异性很大，则不适合使用建造者模式，因此其使用范围受到一定的限制。
* 如果产品的内部变化复杂，可能会导致需要定义很多具体建造者类来实现这种变化，导致系统变得很庞大。

### 适用场景

- 需要生成的对产品对象有复杂的内部结构，这些产品对象包含多个成员属性
- 需要生成的产品对象的属性相互依赖，需要执行其生成顺序
- 对象的场景过程独立于创建该对象的类。在建造者模式中引入了指挥者类，将创建过程封装在指挥者类中，而不再建造者和客户类中
- 隔离复杂对象的创建和使用，并使得相同的创建过程可以创建不同的产品

### Real world examples

* [java.lang.StringBuilder](http://docs.oracle.com/javase/8/docs/api/java/lang/StringBuilder.html)
* [java.nio.ByteBuffer](http://docs.oracle.com/javase/8/docs/api/java/nio/ByteBuffer.html#put-byte-) as well as similar buffers such as FloatBuffer, IntBuffer and so on.
* [java.lang.StringBuffer](http://docs.oracle.com/javase/8/docs/api/java/lang/StringBuffer.html#append-boolean-)
* All implementations of [java.lang.Appendable](http://docs.oracle.com/javase/8/docs/api/java/lang/Appendable.html)
* [Apache Camel builders](https://github.com/apache/camel/tree/0e195428ee04531be27a0b659005e3aa8d159d23/camel-core/src/main/java/org/apache/camel/builder)

## 原型模式（Prototype）
使用原型实例指定创建对象的种类，并且通过拷贝这些原型创建新的对象。原型模式是一种对象创建型模式

原型模式的工作原理：将一个原型对象传给那个要发动创建的对象，这个要发动创建的对象通过请求原型对象拷贝自己来实现创建过程。
- Prototype（抽象原型类）：它是声明克隆方法的接口，是所有具体原型类的公共父类，可以是抽象类也可是接口，甚至还可以是具体实现类
```java
public abstract class Prototype implements Cloneable {
    public abstract Object copy() throws CloneNotSupportedException;
}
```

- ConcretePrototype（具体原型类）：它实现在抽象原型类中声明的克隆方法，在克隆方法中返回自己的一个克隆对象

```java
public abstract class Beast extends Prototype {

  @Override
  public abstract Beast copy() throws CloneNotSupportedException;

}
```
```java
public class ElfBeast extends Beast {
  
  private String helpType;

  public ElfBeast(String helpType) {
    this.helpType = helpType;
  }

  public ElfBeast(ElfBeast elfBeast) {
    this.helpType = elfBeast.helpType;
  }

  @Override
  public Beast copy() throws CloneNotSupportedException {
    return new ElfBeast(this);
  }

  @Override
  public String toString() {
    return "Elven eagle helps in " + helpType;
  }

}
```

```java
public class OrcBeast extends Beast {
  
  private String weapon;

  public OrcBeast(String weapon) {
    this.weapon = weapon;
  }
  
  public OrcBeast(OrcBeast orcBeast) {
    this.weapon = orcBeast.weapon;
  }

  @Override
  public Beast copy() throws CloneNotSupportedException {
    return new OrcBeast(this);
  }

  @Override
  public String toString() {
    return "Orcish wolf attacks with " + weapon;
  }
}

```

- Client（客户端）：让一个原型对象克隆自身从而创造一个新的对象，在客户类中只需要直接实例化或通过工厂方法等方式创建一个原型对象，再通过调用该对象的克隆方法即可得到多个相同的对象

```java
Prototype obj1  = new OrcBeast("laser");
Prototype obj2 = obj.copy();
```

## 总结
原型模式作为一种快速创建大量相同或者相似对象的方式，在软件开发中应用较为广泛，很多软件提供的复制(Ctrl + C)和粘贴(Ctrl + V)操作就是原型模式的典型应用，下面对该模式的使用效果和适用情况进行简单的总结。

### 优点
- 当创建新的对象实例较为复杂时，使用原型模式可以简化的创建过程，通过复制一个已有实例可以提高新实例的创建效率
- 扩展性较好，由于在原型模式中提供了抽象原型类，在客户端可以针对抽象原型类进行编程，而将具体原型类写在配置文件中，增加或减少产品类对原有系统都没有任何影响
- 原型模式提供了简化的创建结构，工厂方法模式常常需要有一个与产品类等级结构相同的工厂等级结构，而原型模式就不需要这样，原型模式中产品的复制是通过封装在原型类中的克隆方法实现的，无须专门的工厂类来创建产品
- 可以使用深克隆的方式保存对象的状态，使用原型模式将对象复制一份并将其状态保存起来，以便在需要的时候使用，可辅助实现撤销操作

### 缺点
- 需要为每一个类配备一个克隆方法，而且该克隆方法位于一个类的内部，当对已有的类进行改造时，需要修改源代码，违背了“开闭原则”
- 在实现深克隆时需要编写较为复杂的代码，而且当对象之间存在多重的嵌套引用时，为了实现深克隆，每一层对象对应的类都必须支持深克隆，实现起来可能会比较麻烦

### 适用场景
- 创建新对象成本较大，新的对象可以通过原型模式对已有对象进行复制来获得，如果是相似对象，则可以对其成员变量稍作修改
- 如果系统要保存对象的状态，而对象的状态变化很小，或者对象本身占用内存较少时，可以使用原型模式配合备忘录模式来实现。
-  需要避免使用分层次的工厂类来创建分层次的对象，并且类的实例对象只有一个或很少的几个组合状态，通过复制原型对象得到新实例可能比使用构造函数创建一个新实例更加方便

## 单例模式（Singleton）


* 意图：保证一个类仅有一个实例，并提供一个访问它的全局访问点。
* 适用性：
    1. 当类只能有一个实例而且客户可以从一个众所周知的访问点访问它时。
    2. 当这个唯一的实例应该是通过子类化可扩展的，并且客户应该无需更改代码就能使用一个扩展的实例时。

### 优点
* 单例模式提供了对唯一实例的受控访问。因为单例模式封装了它的唯一实例，所以它可以严格控制客户这样以及何时访问它
* 节省系统资源，对一些频繁创建和销毁的对象单例模式无疑可以提高系统的性能
* 允许可变数目的实例。基于单例模式可以进行扩展，使用与单例控制相似的方法来获得指定个数的对象实例

### 缺点
* 由于单例模式中没有抽象层，因此单例类的扩展有很大的困难
* 单例类的职责过重，在一定程度上违背了单一职责。因为单例类既充当了工厂角色，提供了工厂方法，同时又充当了产品角色，包含一些业务方法，将产品的创建和产品的本身的功能融合到一起
# 结构型模式（七种）
## Adapter(适配器模式)
在适配器模式中可以定义一个包装类，包装不兼容接口的对象，这个包装类指的就是适配器(Adapter)，它所包装的对象就是是适配者(Adaptee)，即被适配的类。
适配器提供客户类需要的接口，适配器的实现就是把客户类的请求转化为对适配者的相应接口的调用。也就是说：当客户类调用适配器的方法时，在适配器类的内部将调用适配者类的方法，而这个过程对客户类是透明的，客户类并不直接访问适配者类。因此，适配器可以使由于接口不兼容而不能交互的类可以一起工作。这就是适配器模式的模式动机

适配模式(Adapter Pattern)：将一个接口转换成客户希望的另一个接口，适配器模式使接口不兼容的那些类可以一起工作，其别名为包装(Wrapper)。适配器模式既可以做为类结构型模式，也可以作为对象结构型模式。

根据适配器类与适配者类的关系不同，适配器模式可分为对象适配器和类适配器两种，在对象适配器模式中，适配器与适配者之间是关联关系；在类适配器模式中，适配器与适配者之间是继承关系。
- Target（目标抽象类）：目标抽象类定义客户所需接口，可以是一个抽象类或接口，也可以是具体类
```java
public interface RawingBoat {
    void row();
}
```

- Adaptee（被适配者）：定义了一个已经存在的接口，这个接口需要适配，被适配者一般是一个具体类，包含了客户希望使用的业务方法
```java
public class FishingBoat {

    public void sail() {
        log.i("The fishing boat is sailing")
    }
}
```

- Adapter（适配器类）：适配器类可以调用另一个接口，作为一个转换器，对Adaptee和Targe进行适配，适配器类是适配器模式的核心，在对象适配器中，它通过继承Target并关联一个Adaptee对象使得二者产生关联
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

## 优点
* 将目标类和适配者类解耦，通过引入一个适配类来重用现有的适配者类，而无需修改原有代码
* 增加类的透明性和复用性，将具体的实现封装在适配者类中，对于客户端类来说透明的，而且了提高了适配者的复用性
* 灵活性和扩展性都非常好，通过使用配置文件，可以很方便地更换适配器，也可以在不修改原有代码的基础上增加新的适配器类，符合“开闭原则”

**类适配模式具有如下优点**
由于适配器类适配者类的子类，因此可以在适配器类中置换一些适配者的方法，使得适配的灵活性更强

**对象适配器模式具有如下优点**
一个对象适配器可以把多个不同的适配者适配到同一个目标，也就是说，同一个适配器可以把适配者类和它的子类都适配到目标接口

### 缺点
类适配器对于不支持多重继承的语言，一次最多只能适配一个适配者类，而且目标抽象类只能为抽象，不能为具体类，其使用有一定的局限性，不能讲一个适配者类和它的子类都适配到目标接口

对象适配器模式置换时适配者的方法不容易

### Real world examples
* [java.util.Arrays#asList()](http://docs.oracle.com/javase/8/docs/api/java/util/Arrays.html#asList%28T...%29)
* [java.util.Collections#list()](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#list-java.util.Enumeration-)
* [java.util.Collections#enumeration()](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#enumeration-java.util.Collection-)
* [javax.xml.bind.annotation.adapters.XMLAdapter](http://docs.oracle.com/javase/8/docs/api/javax/xml/bind/annotation/adapters/XmlAdapter.html#marshal-BoundType-)


## Bridge(桥接模式)
桥接模式是一种很实用的结构型设计模式，如果软件系统中某个类存在两个独立变化的维度，通过该模式可以将两个维度分离出来，使两者可以独立扩展，让系统更加符合“单一职责原则”。与多层继承方案不同，它将两个独立变化的维度设计为两个独立的继承结构，并且在抽象层建立一个抽象关联，该关联关系类似一条连接两个独立继承结构的桥，故名桥接模式。

桥接模式用一种巧妙的方式处理多层继承存在的问题，用抽象关联取代了传统的多层继承，将类之间的静态继承关系转换为动态的对象组合关系

Abstraction（抽象类）：用于定义抽象类的接口，定义了一个Implementor(实现类接口)类型的对象并可以维护该对象，它与Implementor之间具有关联关系，它即可以包含抽象业务方法，也可以包含具体业务方法
```java
public interface Weapon {
    void wield();

    void swing();

    void unwield();

    Enchantment getEnchantment();
}
```

RefinedAbstraction(扩充抽象类)：扩充由于Abstraction定义的接口，通常情况下它不再是抽象类而是具体类，它实现了抽象类声明的抽象业务方法
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
Implementor（实现类接口）：这个接口不一定要与Abstraction的接口完全一致，实际上可以完全不同，一般而言，Implementor接口仅提供基本操作，而Abstraction定义的接口可能做更多复杂的事。Abstraction中不仅拥有自己的方法，还可以调用到Implementor中定义的方法，使用关联关系来替代继承关系
```java
public interface Enchantment {
    void onActivate();

    void apply();

    void onDeactivate();
}
```

ConcreteImplementor(具体实现类)：具体实现Implementor接口，在不同的ConcreteImplementor中提供基本操作的不同实现，在程序运行时，ConcreteImplementor将替换其父类对象，提供给抽象类具体的业务操作方法
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

Client:
```java
public static void main(String[] args) {
    Sword enchantedSword = new Sword(new SoulEatingEnchantment());
    enchantedSword.wield();
    enchantedSword.swing();
    enchantedSword.unwield();

    Hammer hammer = new Hammer(new FlyingEnchantment());
    hammer.wield();
    hammer.swing();
    hammer.unwield();
}
```

### 总结
在软件开发中如果一个类或一个系统有多个变化维度时，都可以尝试使用桥接模式对其进行设计。桥接模式为多维度变化的系统提供了一套完整的解决方案，并且降低了系统的复杂度

使用桥接模式时，首先应该识别出一个类所具有的两个独立变化的维度，将它们设计为两个独立的继承等级结构，为两个维度都提供抽象层，为两个维度都提供抽象层，并建立抽象耦合
* 意图：将抽象部分与它的实现部分分离，使它们都可以独立地变化。
 (其实就是，子类有两个维度的排列组合 用桥接模式)
  脱耦：脱耦就是将抽象化和实现化之间的耦合解脱开，或者说是将它们之间的强关联换成弱关联，将两个角色之间的继承关系
  改为关联关系。桥接模式中所谓的脱耦，就是指一个软件系统的抽象化和实现化之间使用关联关系而不是继承关系。

#### 优点
- 分离抽象接口及其实现部分
- 桥接模式有时类似于多继承方案，但是多继承方案违背了类的单一职责原则，复用性较差，而且多继承结构中类的个数非常庞大，桥接模式是比多继承方案更好的解决方法。
- 桥接模式提高了系统的可扩充性，在两个变化唯独中任意扩展一个维度，都不需要修改原有系统。
- 实现细节对客户透明，可以对用户隐藏实现细节

#### 缺点
- 桥接模式引入会增加系统的理解和设计难度，由于聚合关联关系建立在抽象层，要求开发者针对抽象进行设计和编程
- 桥接模式要求正确识别出系统中两个独立变化的维度，因此其使用范围有一定的局限

#### 适用场景
- 如果一个系统需要在抽象化和具体化之间增加更多的灵活性，避免在两个层次之间建立静态的继承关系，通过桥接模式可以使它们在抽象层建立一个关联关系。
- 抽象部分和实现部分可以以继承方式独立扩展而互不影响，在程序运行时可以动态将一个抽象化子类的对象和一个实现化子类的对象进行组合。
- 一个类存在两个（或者多个）不同变化的维度，且这两个（或多个）维度都需要独立进行扩展。
-  对那些不希望使用继承或因为多层继承导致系统类的个数急剧增加的系统，桥接模式尤为适用

## Composite(组合模式)
组合多个对象形成树形结构以表示具有“整体-部分”关系的层次结构。组合模式对单个对象和组合对象的使用具有一致性，组合模式又可以称为“整体-部分”模式，
它是一种对象结构性模式
* 意图：将对象组合成树形结构以表示"部分-整体"的层次结构。composite使得用户对单个对象和组合对象的使用具有一致性。
* 适用性：
    1. 你想表达对象的部分-整体层次结构。
    2. 你希望用户忽略组合对象与单个对象的不同，用户将统一地使用组合机构中的所有对象。
Component（抽象构件）：可以是接口或抽象类，为叶子构件和容器构件对象声明接口，在该角色中可以包含所有子类共有行为的声明和实现。
在抽象构件中定义了访问及管理它的子构件的方法
```java
public abstract class LetterComposite {
    private List<LetterComposite> children = new ArrayList<>();

    public void add(LetterComposite letter) {
        childred.add(letter);
    }

    public int count() {
        return children.size();
    }

    protected void printThisBefore();

    protected void printThisAfter();

    public void print() {
        printThisBefore();
        for(LetterComposite letter : childred) {
            letter.print();
        }
        printThisAfter();
    }
}
```
Leaf（叶子构件）：它在组合结构中表示叶子节点对象，容器节点包含子节点，其子节点可以是叶子节点，也可以容器节点，它提供一个集合用于存储子节点，
实现了在抽象构件中定义的行为，包括哪些访问及管理子构件的方法，在其业务方法中可以递归调用其子节点的业务方法
```java
public class Letter extends LetterComposite {
    private char c;
    public Letter(char c) {
        this.c = c;
    }
    @Override
    protected void printThisBefore() {
        System.out.print(c);
    }
}
```

Composite（容器构件）：它在组合结构中表示容器节点对象，容器节点包含子节点，其子节点可以是叶子节点，也可以是容器节点，它提供一个集合用于存储子节点，
实现了在抽象构件中定义的行为
```java
public class Word extends LetterComposite {
    public Word(List<Letter> letters) {
        for (Letter l : letters) {
            this.add(l);
        }
    }

    @Override
    protected void printThisBefore() {
        System.out.print(" ");
    }
}
```

```java
public class Sentence extends LetterComposite {
    public Sentence(List<Word> words) {
        for(Word w : words) {
            this.add(w);
        }
    }

    @Override
    protected void printThisAfter() {
        System.out.print(" ");
    }
}
```

组合模式的关键是定义了一个抽象构件类，它既可以代表叶子，又可以代表容器，而客户端对该抽象构件类进行编程

组合模式使用面向对象的思想来实现树形结构的构建与处理，描述了如何将容器对象和叶子对象进行递归组合，实现简单，灵活性好

### 优点
1. 组着模式可以清楚地定义分层次的复杂对象，表示对象的全部或部分层次，它让客户端忽略了层次的差异，方便对整个层次结构进行控制
2. 客户端可以一致的使用一个组合结构或其中单个对象，不必关心处理的是单个对象还是整个组合结构，简化了客户端代码
3. 在组合模式增加新的容器构件和叶子构件都很方面，符合“开闭原则”
4. 组合模式为树形结构的面向对象实现提供了一种灵活的解决方案，通过叶子对象和容器对象的递归组合，可以形成复杂的树形结构，但对树形结构的控制却非常简单

### 缺点
在增加新的构件时难对容器中的构建类型进行限制。

### 场景
1. 在具有整体和部分的层次结构中，希望通过一种方式忽略整体与部分的差异，客户端可以一致地对待它们
2. 需要处理一种树形结构
3. 在一个系统中能够分离出叶子对象和容器对象，而且它们的类型不固定，需要增加一些新的类型


### Real world examples

* [java.awt.Container](http://docs.oracle.com/javase/8/docs/api/java/awt/Container.html) and [java.awt.Component](http://docs.oracle.com/javase/8/docs/api/java/awt/Component.html)
* [Apache Wicket](https://github.com/apache/wicket) component tree, see [Component](https://github.com/apache/wicket/blob/91e154702ab1ff3481ef6cbb04c6044814b7e130/wicket-core/src/main/java/org/apache/wicket/Component.java) and [MarkupContainer](https://github.com/apache/wicket/blob/b60ec64d0b50a611a9549809c9ab216f0ffa3ae3/wicket-core/src/main/java/org/apache/wicket/MarkupContainer.java)


## Facade(外观模式)
* 意图：为子系统中的一组接口提供一个一致的界面，Facad模式定义了一个高层接口，这个接口使得这一子系统更加容易使用。
* 适用性：①当你要为一个复杂子系统提供一个简单接口时。子系统往往因为不断演化而变得越来越复杂。大多数模式使用时都会产生更多更小的类。
这使得子系统更具可重用性，也更容易对子系统进行定制，但这也给那些不需要定制的子系统的用户带来一些使用上的困难。facad可以提供一个简单的缺省视图，
这一视图对大多数用户来说已经足够，而那些需要更多的可定制性的用户可以越过facade层。
②客户程序与抽象类的实现部分之间存在着很大的依赖性。引入facade将这个子系统与客户以及其他子系统分离，可以提高子系统的独立性和可移植性。
③当你需要构建一个层次结构的子系统时，使用facade模式定义子系统中每层的入口点。如果子系统之间是相互依赖的，你可以让它们仅通过facade进行通讯，从而简化了它们之间的依赖关系。

### Flyweight(享元模式)
* 意图：运用共享技术有效地支持大量细粒度的对象。
* 适用性：①一个应用程序使用了大量的对象。
②完全由于使用大量的对象，造成很大的存储开销。
③对象的大多数状态都可变为外部状态。
④如果删除对象的外部状体，那么可以用相对较少的共享对象取代很多组对象。
⑤应用程序不依赖于对象标识。由于Flyweight对象可以被共享，对于概念上明显有别的对象，标识测试将返回真值。


### Decorator(装饰模式)
装饰模式以对客户透明的方式动态地给一个对象附件上更多的责任，换言之，客户端并不会觉得对象在装饰前和装饰后有什么不同。装饰模式可以在不需要创造更多子类的情况下，将对象的功能加以扩展。这就是装饰模式的动机。

装饰模式是一种用来替代你继承的技术，它通过无须定义子类的方式来给对象动态增加职责，使用对象之间的关联关系取代类之间的继承关系。在装饰模式中引入装饰类，在装饰类中既可以调用带装饰的原有类的方法，还可以增加新的方法，以扩充原有类的功能
* 意图：动态地给一个对象添加一些额外的职责。就增加功能来说，decorator模式相比生成子类更为灵活。其别名也可以称为包装器(Wrapper)，与适配器模式的别名相同。
* 适用性：
    1. 在不影响其他对象的情况下，以动态、透明的方式给单个对象添加职责
    2. 处理那些不可撤销的职责。
    3. 当不能生成子类的方法进行扩充时。一种情况是，可能有大量的扩展，为支持每一种组合将产生大量的子类，使得子类数据呈爆炸性增长。
另一种情况可能是因为类定义被隐藏，或类定义不能用于生成子类。

Component（抽象构件）:它是具体构件和抽象装饰类的共同父类，声明了在具体构件展总实现的业务方法，它的引入可以使客户端以一直的方式处理未被装饰的对象以及装饰之后的对象，实现客户端的透明操作
```java
public interface Troll {
    void attach();

    int getAttackPower();

    void fleeBattle();
}
```
ConcreteComponent (具体构件类)：它是抽象构件类的子类，用于定义具体的构建对象，实现了在抽象构件中声明的方法，装饰器可以给它增加额外的责任
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

Decorator（抽象装饰类）：它是抽象构件类的子类，用于给具体构件增加职责，但是具体职责在其子类中实现。它维护一个指向抽象构件对象的引用，通过该引用可以调用装饰之前构件对象的方法，并通过其子类扩展该方法，以达到装饰的目的
ConcreteDecorator(具体装饰类)：抽象装饰类的子类，负责给具体构件增加职责。每一个具体装饰类都定义了一些新的行为，它可以调用在抽象装饰类中定义的方法，并可以增加新的方法用以扩充对象的行为
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
Troll troll = new SimpleTroll();
troll.attack();
troll.fleeBattle();

troll = new ClubbedTroll(troll);
troll.attack();
troll.fleeBattle();
```
装饰模式降低了系统的耦合度，可以动态增加或删除对象的职责，并使得需要装饰的具体构件类和具体装饰类可以独立变化，以便增加新的具体装饰类
#### 优点
- 装饰模式与继承关系的目的都是要扩展对象的功能，但是装饰模式可以提供比继承更多的灵活性。
- 可以通过一种动态的方式来扩展一个对象的功能，通过配置文件可以在运行时选择不同的装饰器，从而实现不同的行为。
- 通过使用不同的具体装饰器以及这些装饰类的排列组合，可以创造出很多不同行为的组合。可以使用多个具体装饰类来装饰同一对象，得到功能更为强大的对象。
- 具体构建类与具体修饰类可以独立变化，用户可以根据需要增加新的具体构件类和具体修饰类，在使用时再对其进行组合，原有代码无须改变，符合"开闭原则"

#### 缺点
* 使用装饰模式进行系统设计时将产生很多小对象，这些对象的区别在于它们之间相互连接的方式有所不同
* 装饰模式比继承有更加的灵活性，但同时也意味着比继承更加容易出错，排错也很困难


#### 使用场景
- 在不影响其他对象的情况下，以动态、透明的方式给单个对象添加职责
- 当不能采用继承的方式对系统进行扩展或者采用继承不利于系统扩展和维护时可以使用装饰模式

#### Real world examples
 * [java.io.InputStream](http://docs.oracle.com/javase/8/docs/api/java/io/InputStream.html), [java.io.OutputStream](http://docs.oracle.com/javase/8/docs/api/java/io/OutputStream.html),
 [java.io.Reader](http://docs.oracle.com/javase/8/docs/api/java/io/Reader.html) and [java.io.Writer](http://docs.oracle.com/javase/8/docs/api/java/io/Writer.html)
 * [java.util.Collections#synchronizedXXX()](http://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#synchronizedCollection-java.util.Collection-)
 * [java.util.Collections#unmodifiableXXX()](http://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#unmodifiableCollection-java.util.Collection-)
 * [java.util.Collections#checkedXXX()](http://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#checkedCollection-java.util.Collection-java.lang.Class-)

### Proxy(代理模式)
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

# 行为型模式(十一种)
## Chain of Responsibility(责任链模式)
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

## Command(命令模式)
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

### 优点
- 降低耦合度。由于请求者与接收者之间不存在直接引用，因此请求者与接收者完全解耦
- 新的命令可以很容易地加入到系统中，符合开闭原则
- 可以比较容易设计一个命令队列或组合命令
- 为请求的撤销和恢复操作提供了一种设计和实现方案

### 缺点
导致某些系统有过多的具体命令类

## Interpreter(解释器模式)
* 意图：给定一个语言，定义它的文法的一种表示，并定义一个解释器，这个解释器使用该表示来解释语言中的句子。
* 适用性：当有一个语言需要解释执行，并且你可将该语言中的句子表示一个抽象语法树时，可使用解释器模式。而当存在以下情况时该模式效果最好。
该文法简单对于复杂的文法，文法的类层次变得庞大而无法管理。此时语法分析程序生成器这样的工具是更好的选择。它们无需构建语言树即可解释表达式，这样可以节省空间而且可能节省时间。
效率不是一个关键问题最高效的解释器通常不是通过直接解释语法分析树实现的，而是首先将它们转换成另一种形式。

## Iterator(迭代器模式)
提供一种方法顺序访问一个聚合对象中各个元素，而又不需要暴露该对象的内部表示。

迭代器模式的角色：
- Iterator（抽象迭代器）：它定义了访问和遍历元素的接口，声明了用于遍历数据元素的方法
```java
public interface ItemIterator {
    boolean hasNext();

    Item next();
}
```

- ConcreteIterator（具体迭代器）：它实现了抽象迭代器接口，完成了对聚合对象的遍历。同时在具体迭代器中通过游标来记录在聚合对象中的位置，在具体实现时，游标通常是一个表示位置的非负整数
```java
public class TreasureChestItemIterator implements ItemIterator {
    private TreasureChest chest;
    private int idx;
    private ItemType type;

    public TreasureChestItemIterator(TreasureChest chest, ItemType type) {
        this.chest = chest;
        this.type = type;
        this.ldx = -1;
    }

    @Override
    public boolean hasNext() {
        return findNextIdx() != 1;
    }

    @Override
    public Item next() {
        ldx = findNextIdx();
        if (ldx != -1) {
            return chest.getItems().get(ldx);
        }
        return null;
    }

    private int findNextIdx() {
        List<Item> items = chest.getItems();
        boolean found = false;
        int tempIdx = idx;
        while(!found) {
            tempIdx++;
            if (tempIdx >= item.size()) {
                tempIdx = -1;
                break;
            }
            if (type.equals(ItemType.ANY) || items.get(tempIdx).getType.equals(type)) {
                break;
            }
        }
        return tempIdx;
    }
}
```
Aggregate(抽象聚合类)：它用于存储和管理元素对象，声明一个creteIterator()方法用来创建一个迭代器对象，充当抽象迭代器工厂角色


ConcreteAggregate（具体聚合类）：它实现了在抽象聚合类中声明的createIterator()方法，该方法返回一个与该具体聚合类对应的具体迭代器ConcreteIterator实例
```java
public class TreasureChest {
    private List<Item> items;

    public TreasureChest() {
        items = new ArrayList<>();
        items.add(new Item(ItemType.POTION, "Potion of courage"));
        items.add(new Item(ItemType.RING, "Ring of shadows"));
        items.add(new Item(ItemType.POTION, "Potion of wisdom"));
        items.add(new Item(ItemType.POTION, "Potion of blood"));
        items.add(new Item(ItemType.WEAPON, "Sword of silver +1"));
        items.add(new Item(ItemType.POTION, "Potion of rust"));
        items.add(new Item(ItemType.POTION, "Potion of healing"));
        items.add(new Item(ItemType.RING, "Ring of armor"));
        items.add(new Item(ItemType.WEAPON, "Steel halberd"));
        items.add(new Item(ItemType.WEAPON, "Dagger of poison"));
    }

    ItemIterator iterator(ItemType itemType) {
        return new TreasureChestItemIterator(this, itemType);
    }

    public List<Item> getItems() {
        List<Item> list = new ArrayList<>();
        list.addAll(items);
        return list;
    }
}
```
迭代器模式是一种使用频率非常高的设计模式，通过引入迭代器可以将数据遍历功能从聚合对象中分离出来，聚合对象只负责存储数据，而遍历数据由迭代器来完成

### 优点
1. 它支持不同方式遍历一个聚合对象，在同一个聚合对象上可以定义多种遍历方式。在迭代器模式中只需要用一个不同的迭代器来替换原有迭代器即可改变算法，可以自己定义迭代器的子类以支持新的遍历方式
2. 迭代器简化了聚合类。
3. 在迭代器模式中，由于引入了抽象层，增加新的聚合类和迭代器类都很方便，无须修改原有代码，满足“开闭原则”的要求

### 缺点
1. 由于迭代器模式将数据存储和遍历数据的职责分离，增加新的聚合类需要对应增加新的迭代器类，类的个数成对增加
2. 抽象迭代器的设计难度较大，需要充分考虑到系统将来的扩展

### 适应场景
1. 访问一个聚合对象的内容而无需暴露它的内部表示。将聚合对象的访问与内部数据的存储分离，使得访问聚合对象时无需了解其内部实现细节
2. 为一个聚合对象提供多种遍历。
3. 为遍历不同的聚合结构提供一个统一的接口(即支持多态迭代)

## Mediator(中介者模式)
* 意图: 用一个中介对象来封装一系列的对象交互。中介者使各对象不需要显示地相互引用，从而使耦合松散，而且可以独立地改变它们之间的交互。
* 适用性：①一组对象以定义良好但是复杂的方式进行通信。产生的相互依赖关系结构混乱且难以理解。
②一个对象引用其他很多对象并且直接与这些对象通信，导致难以复用该对象。
③想定制一个分布在对个类中的行为，而又不想生成太多的子类。

## Memento(备忘录模式)
* 意图 在不破坏封装性的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态，这样以后就可将该对象恢复到原先保存的状态。
* 适用性 必须保存一个对象在某一个时刻状态，这样以后需要时它才能恢复到先前的状态。
如果一个用接口来让其它对象直接得到这些状态，将会暴露对象的实现细节并破坏对象的封装性。

## Observer(观察者模式)
### 模式动机
 定义对象间的一宗一对多的依赖关系，观察者模式描述了如何建立对象与对象之间的关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。发生改变的对象称为观察目标，而被通知的对象称为观察者，一个观察目标可以对应多个观察者，而且这些观察者之间没有相互联系，可以根据需要增加和删除观察者。
### 模式定义
观察者模式(Observer Pattern):观察者模式又叫做发布-订阅(Publish/Subscribe)模式、模型-视图(Model/View)模式、源-监听器(Source/Listener)模式

Subject(目标):目标指被观察者。在目标中定义了一个观察者集合，一个观察目标可以接受任意数量的观察者来观察，它提供了一系列方法来增加和删除观察者对象，同时它定义了nofity()。
目标类可以是抽象类或者具体类
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

Observer（观察者）:观察者将堆观察目标的改变作出反应，观察者一般定义为接口，该接口声明了更新数据的`update`，因此称为抽象观察者
```java
public interface WeatherObserver {
    void update(WeatherType currentWeather);
}
```

ConcreteObserver（具体观察者）：

```java
public class Orcs implements WeatherObserver {

    public void update(WeatherType currentWeather) {
        
    }
}
```



### 优点
观察者模式的优点
- 观察者模式可以实现表示层和数据逻辑层的分离，并定义了稳定的消息更新传递机制，抽象了更新接口，使得可以有各种各样不同的表示层作为具体观察者角色。
- 观察者模式在观察目标和观察者之间建立了一个抽象的耦合。
- 观察者模式支持广播通信。
- 观察者模式符合"开闭原则"的要求

### 缺点
- 如果一个观察目标对象有很多直接和间接的观察者的话，将所有的观察者都通知会花费很多时间
- 如果观察者和观察目标之间有循环依赖的话，观察目标会触发它们之间进行循环调用，可能导致系统崩溃
- 观察者模式没有相应的机制让观察者知道所观察的目标对象是怎么发生变化的，而仅仅只是知道观察目标发生了变化

* 适用性 
    1. 当一个抽象模型有两个方面，其中一个方面依赖于另一个方面。将两者封装在独立的对象中以使它们可以各自独立地改变和复用
    2. 当对一个对象的改变需要同时改变其它对象，而不知道具体有多少对象有待改变。
    3. 当一个对象必须通知其它对象，而它又不能假定其它对象是谁。换言之，你不希望这些对象是紧密耦合的。

## State(状态模式)
允许一个对象在其内部状态改变时改变它的行为，对象看起来似乎修改了它的类。状态模式是一种对象行为型模式。

- Context(环境类)：环境类又称为上下文类，它是拥有多种状态的对象。由于环境类的状态存在多样性且在不同状态下对象的行为有所不同，因此将状态独立出去形成单独的状态类。在环境类中维护了一个抽象状态State的实例。
```java
public class Mammoth {
   private State state;

   public Mammoth() {
       state = new PeaceFulState(this);
   }

   public void timePasses() {
       if (state.getClass().equals(PeacefulState.class)) {
           changeStateTo(new AngryState(this));
       } else {
           changeStateTo(new PeaceFulState(this));
       }
   }

   private void changeStateTo(State newState) {
       this.state = newState;
       this.state.onEnterState();
   }

   public String toString() {
       return "The mammoth";
   }

   public void observer() {
       this.state.observe();
   }
}
```

- State（抽象状态类）：它用于定义一个接口以封装与环境类的一个特定状态相关的行为，在抽象状态类中声明了各种不同状态对应的方法，而在其子类中实现类这些方法，由于不同状态下的对象的行为可能不同，因此在不同子类中方法的实现可能存在不同，相同方法可以写在抽象状态类中
```java
public interface State {
    void onEnterState();

    void observe();
}
```
- ConcreteState(具体状态类)：它是抽象状态类的子类，每一个子类实现一个与环境类的一个状态相关的行为，每一个具体状态类对应环境的一个具体状态，不同具体状态类其行为有所不同。
```java
public class AngryState implements State {
    private Mammoth mammoth;

    public AngryState(Mammonth mammoth) {
        this.mammoth = mammoth;
    }

    @Override
    public void observe() {

    }

    @Overide
    public void onEnterState() {

    }
}
```

```java
public class PeacefulState implements State {
    private Mammoth mammoth;

    public PeacefulState(Mammoth mammoth) {
        this.mammoth = mammoth;
    }

    @Override
    public void observe() {

    }

    @Override
    public void onEnterState() {

    }
}
```

### 总结
状态模式将一个对象在不同状态下的不同行为封装在一个个状态类中，通过设置不同的状态对象可以让环境对象拥有不同的行为，而状态转换的细节对于客户端而言是透明的，方便了客户端的使用。

#### 优点
- 封装了状态的转换原则，在状态模式中可以将状态转换代码封装在环境类或者具体状态类中，可以对状态转换代码进行集中管理，而不是分散在一个业务方法中
- 将所有与某个状态有关的行为放到一个类中，只需要注入一个不同的状态对象即可使环境对象拥有不同行为
- 允许状态转换逻辑与状态对象合成一体，而不是提供一个巨大的条件语句块，状态模式可以避免用庞大的条件语句来将业务方法和状态转换代码交织在一起
- 可以让多个环境对象共享一个状态对象，从而减少系统中对象的个数

#### 缺点
- 增加系统中类和对象的个数，导致系统运行开销增大
- 状态模式的结构和实现都较为复杂，如果使用不当将导致程序结构和代码混乱，增加系统设计难度
- 状态模式对“开闭原则”的支持并不太好，增加新的状态类需要修改那些负责状态转换的源代码

#### 适用场景
- 对象的行为依赖于它的状态，状态的改变将导致行为的变化
- 一个操作中含有庞大的多分支的条件语句，且这些分支依赖于该对象的状态。这状态通常用一个或多个枚举常量表示。通常，有多个操作包含这一相同的条件结构。
State模式将每一个条件分支放入一个独立的类中。这使得你可以根据对象自身的情况将对象的状态作为一个对象，这一对象可以不依赖于其他对象独立变化。

## Strategy(策略模式)
定义了一系列算法类，将每一个算法封装起来，并让他们可以相互替换，策略模式让算法独立于使用它的客户而变化，也称为政策模式(Policy)。

策略模式的主要目的是将算法的定义和使用分开，也就是将算法的行为和环境分开，将算法的定义放在专门的策略类中。
每一个策略类封装了一种实现算法，使用算法的环境类针对抽象策略类进行编程，符合“依赖倒转原则”

Strategy(抽象策略类)：它为所支持的算法声明了抽象方法，是所有策略类的父类，它可以是抽象类或具体类，也可以是接口。环境类通过抽象策略类中声明的方法在运行时调用具体策略类中实现的算法。
```java
public interface DragonSlayingStrategy {
    void execute();
}
```
ConcreteStrategy(具体策略类)：
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
Context(环境类)：是需要使用算法的类。
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

### 优点
* 策略模式提供了对“开闭原则”的完美支持，用户可以在不修改原有系统的基础选择算法或行为，也可以灵活地增加新的算法或行为
* 策略模式提供了管理相关的算法族的办法。策略类的结构等级结定义了一个算法或行为族，恰当使用继承可以把公共的代码移到抽象策略类中
* 策略模式提供了一种可以替换继承关系的办法
* 使用策略模式可以避免多重条件选择语句。多重条件语句不易维护，它把采取哪一种算法或行为的逻辑与算法或者行为本身的实现逻辑混合在一起，将它们全部硬编码(Hard Coding)在一个庞大的多重条件选择语句中，比直接继承环境类的办法还要原始和落后
* 策略模式提供了一种算法的复用机制，由于将算法单独提取出来封装在策略中，因此不同的环境类可以方便地复用这些策略类

### 缺点
- 客户端必须知道所有的策略类，并自行决定使用哪一个策略类
- 策略模式将造成系统产生很多具体策略类
- 无法同时在客户端使用多个策略类

* 适用性 
1. 系统需要动态地在几种算法中选择一种，那么可以将这些算法封装到一个个的具体算法类中，而这些具体算法类都是一个抽象算法类的子类。
2. 一个对象有很多的行为，如果不用恰当的模式，这些行为就只好使用多重条件选择语句来实现。此时，使用策略设计模式，把这些行为转移到相应的具体策略类，就可以避免使用难以维护的多重条件选择语句
3. 不希望客户端知道复杂的、与算法相关的数据结构，可以提高算法的保密性与安全性

## Template Method(模板方法模式)
定义一个操作中算法的框架，而将一些步骤延迟到子类中。模板方法模式使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤


AbstractClass（抽象类）：在抽象类中定义了一系列基本操作，这些基本操作可以是具体的，也可以是抽象，每一个基本操作对应算法的一个步骤，在其子类中可以重定义或实现这些步骤
同时，在抽象类中实现了一个模板方法(Template Method)，用于定义一个算法的框架，模板方法不仅可以调用在抽象类中实现的基本方法
```java
public abstract class StealingMethod {
    protected abstract String pickTarget();

    protected abstract void confuseTarget(String target);

    protected abstract void stealTheItem(String target);

    public void steal() {
        String target = pickTaget();

        confuseTarget(target);
        stealTheItem(target);
    }
}
```
ConcreteClass（具体类）：它是抽象类的子类，用于实现在父类中声明的抽象基本操作以完成子类特定算法步骤，也可以覆盖在父类中已经实现的具体基本操作
```java
public class HitAndRunMethod extends StealingMethod {
    @Override
    protected String pickTarget() {
        return "old goblin woman";
    }

    @Override
    protected void confuseTarget(String target) {
        LOGGER.info("Approach the {} from behind.", target);
    }

    @Override
    protected void stealTheItem(String target) {
        LOGGER.info("Grab the handbag and run away fast!");
    }
}
```

```java
public class SubtleMethod extends StealingMethod {

  private static final Logger LOGGER = LoggerFactory.getLogger(SubtleMethod.class);

  @Override
  protected String pickTarget() {
    return "shop keeper";
  }

  @Override
  protected void confuseTarget(String target) {
    LOGGER.info("Approach the {} with tears running and hug him!", target);
  }

  @Override
  protected void stealTheItem(String target) {
    LOGGER.info("While in close contact grab the {}'s wallet.", target);
  }
}
```

### 总结
模板方法模式是基于继承的代码复用技术，它体现了面向对象的诸多重要思想，是一种使用频繁的模式。模板方法模式广泛应用于框架设计中，
以确保通过父类来控制处理流程的逻辑顺序

#### 优点
- 在父类形式化地定义一个算法，而由它的子类来实现细节的处理，在子类实现详细的处理算法时并不会改变算法中步骤的执行次序
- 模板方法模式是一种代码复用技术，它在类库设计中尤为重要，它提取了类库中的公共行为，将公共行为放在父类中，而通过其子类来实现不同的行为，它鼓励我们恰当使用继承来实现代码复用
- 可实现一种反向控制结果，通过子类覆盖父类的钩子方法来决定某一特定步骤是否需要执行
- 在模板方法模式中可以通过子类来覆盖父类的基本方法，不同的子类可以提供基本方法的不同实现，更换和增加新的子类很方便，符合单一职责原则和开闭原则

#### 缺点
- 需要为每一个基本方法的不同实现提供一个子类，如果父类中可变的基本方法太多，将会导致类的个数增加，系统庞大，此时可结合桥接模式来进行设计

#### 适用场景

- 对一些复杂算法进行分离，将其算法中固定不变的部分设计为模板方法和父类具体方法，而一些可以改变的细节由其子类来实现
- 各子类中公共的行为应被提取出来并集中到一个公共父类以避免代码重复
- 需要通过子类来决定父类算法中某个步骤是否执行，实现子类对父类的反向控制

## Visitor(访问者模式)

* 意图 表示一个作用于某对象结构中的各元素的操作。它使你可以在不改变各元素的前提下定义作用于这些元素的新操作。

* 适用性 一个对象结构包含很多类对象，它们由不同的接口，而你想对这些对象实施一些依赖于某具体类的操作。

需要对一个对象结构中的对象进行很多不同的并且不想关的操作，而你想避免让这些操作“污染”这些对象的类。

Visitor使得你可以将相关的操作集中起来定义在一个类中。当该对象结构被很多应用共享时，用Visitor模式让每个应用仅包含需要用到的操作。

定义的对象结构的类很少改变，但经常需要在此结构上定义新的操作。改变对象结构类需要重定义对所有访问者的接口，这可能需要很大的代价。如果对象结构类经常改变，那么可能还是在这些类中定义这些操作较好。


# 附
## Delegation(委托模式)
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
∏
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

