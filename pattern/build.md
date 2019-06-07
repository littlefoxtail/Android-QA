# 建造者模式（Builder）

**通俗来讲： 允许您创建不同的对象风格，同时避免构造器污染。 当可能有几种风格的对象时很有用。 或者当创建对象时涉及很多步骤**
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

* Builder（抽象建造者）：为创建一个产品对象的各个部件指定抽象接口，在该接口中一般声明两类方法，一类方法是buildPartX()，它们用于创建复杂对象的各个部件；另一类方法是getResult()，它们用于返回复杂对象，Builder既可以是抽象类，也可以是接口。

* ConcreteBuilder（具体建造者）:实现了Builder接口，实现各个部件的具体构造和装配方法，定义并明确它所创建的复杂对象，也可以提供一个方法创建返回好的复杂产品对象。

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

* Product（产品角色）：它是被构建的复杂对象，包含多个组成组件，具体建造者创建该产品的内部表示定义它的装配过程

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

* Director（指挥者）:它负责安排复杂对象的建造次序，指挥者与抽象建造者之间存在关联关系，可以调用建造者对象部件构造与装配方法，完成复杂对象的建造，客户端一般只需要与指挥者交互，确定具体的建造者的类型

    ```java
    public class App {
        public static void main(String[] args) {
            Hero mage = new Hero.Builder(Profession.Name, "Riobard").withHairColor(HairColor.BLACK).withWeapon(WeaponD.AGGER).build();
        }
    }
    ```

## 建造者总结

建造者的核心在于如何一步步构件一个包含多个组成部分的完整对象，使用相同的构建不同的产品。

### 建造者优点

* 在建造者模式中，客户端不必知道产品内部组成的细节，将产品本身与产品的创建过程解耦，使得相同的创建过程可以创建不同的产品对象
* 每一个具体建造者都相对独立，而与其他的具体建造者无关，因此可以很方便地替换具体建造者或增加新的具体建造者，用户使用不同的具体建造者即可得到不同的产品对象。
* 可以更加精细地控制产品的创建过程。将复杂产品的创建步骤分解在不同的方法中，使得创建过程更加清晰，也更方便使用程序来控制创建过程。
* 增加新的具体建造者无需修改原有类库的代码，指挥者类针对抽象建造者类编程，系统扩展方便，符合“开闭原则”

### 建造者缺点

* 建造者模式所创建的产品一般具有较多的共同点，其组成部分相似，如果产品之间的差异性很大，则不适合使用建造者模式，因此其使用范围受到一定的限制。
* 如果产品的内部变化复杂，可能会导致需要定义很多具体建造者类来实现这种变化，导致系统变得很庞大。

### 建造者适用场景

* 需要生成的对产品对象有复杂的内部结构，这些产品对象包含多个成员属性
* 需要生成的产品对象的属性相互依赖，需要执行其生成顺序
* 对象的场景过程独立于创建该对象的类。在建造者模式中引入了指挥者类，将创建过程封装在指挥者类中，而不再建造者和客户类中
* 隔离复杂对象的创建和使用，并使得相同的创建过程可以创建不同的产品

## 建造者实例

* [java.lang.StringBuilder](http://docs.oracle.com/javase/8/docs/api/java/lang/StringBuilder.html)
* [java.nio.ByteBuffer](http://docs.oracle.com/javase/8/docs/api/java/nio/ByteBuffer.html#put-byte-) as well as similar buffers such as FloatBuffer, IntBuffer and so on.
* [java.lang.StringBuffer](http://docs.oracle.com/javase/8/docs/api/java/lang/StringBuffer.html#append-boolean-)
* All implementations of [java.lang.Appendable](http://docs.oracle.com/javase/8/docs/api/java/lang/Appendable.html)
* [Apache Camel builders](https://github.com/apache/camel/tree/0e195428ee04531be27a0b659005e3aa8d159d23/camel-core/src/main/java/org/apache/camel/builder)