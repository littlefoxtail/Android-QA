# Bridge(桥接模式)

桥接模式是一种很实用的结构型设计模式，如果软件系统中某个类存在两个独立变化的维度，通过该模式可以将两个维度分离出来，使两者可以独立扩展，让系统更加符合“单一职责原则”。与多层继承方案不同，它将两个独立变化的维度设计为两个独立的继承结构，并且在抽象层建立一个抽象关联，该关联关系类似一条连接两个独立继承结构的桥，故名桥接模式。

桥接模式用一种巧妙的方式处理多层继承存在的问题，用抽象关联取代了传统的多层继承，将类之间的静态继承关系转换为动态的对象组合关系

* Abstraction（抽象类）：用于定义抽象类的接口，定义了一个Implementor(实现类接口)类型的对象并可以维护该对象，它与Implementor之间具有关联关系，它即可以包含抽象业务方法，也可以包含具体业务方法

    ```java
    public interface Weapon {
        void wield();

        void swing();

        void unwield();

        Enchantment getEnchantment();
    }
    ```

* RefinedAbstraction（扩充抽象类）：扩充由于Abstraction定义的接口，通常情况下它不再是抽象类而是具体类，它实现了抽象类声明的抽象业务方法

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

* Implementor（实现类接口）：这个接口不一定要与Abstraction的接口完全一致，实际上可以完全不同，一般而言，Implementor接口仅提供基本操作，而Abstraction定义的接口可能做更多复杂的事。Abstraction中不仅拥有自己的方法，还可以调用到Implementor中定义的方法，使用关联关系来替代继承关系

    ```java
    public interface Enchantment {
        void onActivate();

        void apply();

        void onDeactivate();
    }
    ```

* ConcreteImplementor（具体实现类）：具体实现Implementor接口，在不同的ConcreteImplementor中提供基本操作的不同实现，在程序运行时，ConcreteImplementor将替换其父类对象，提供给抽象类具体的业务操作方法

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

* Client（客户端）：
  
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

## 桥接模式总结

在软件开发中如果一个类或一个系统有多个变化维度时，都可以尝试使用桥接模式对其进行设计。桥接模式为多维度变化的系统提供了一套完整的解决方案，并且降低了系统的复杂度

使用桥接模式时，首先应该识别出一个类所具有的两个独立变化的维度，将它们设计为两个独立的继承等级结构，为两个维度都提供抽象层，为两个维度都提供抽象层，并建立抽象耦合

* 意图：将抽象部分与它的实现部分分离，使它们都可以独立地变化。
 (其实就是，子类有两个维度的排列组合 用桥接模式)
  脱耦：脱耦就是将抽象化和实现化之间的耦合解脱开，或者说是将它们之间的强关联换成弱关联，将两个角色之间的继承关系
  改为关联关系。桥接模式中所谓的脱耦，就是指一个软件系统的抽象化和实现化之间使用关联关系而不是继承关系。

### 桥接模式优点

* 分离抽象接口及其实现部分
* 桥接模式有时类似于多继承方案，但是多继承方案违背了类的单一职责原则，复用性较差，而且多继承结构中类的个数非常庞大，桥接模式是比多继承方案更好的解决方法。
* 桥接模式提高了系统的可扩充性，在两个变化唯独中任意扩展一个维度，都不需要修改原有系统。
* 实现细节对客户透明，可以对用户隐藏实现细节

### 桥接模式缺点

* 桥接模式引入会增加系统的理解和设计难度，由于聚合关联关系建立在抽象层，要求开发者针对抽象进行设计和编程
* 桥接模式要求正确识别出系统中两个独立变化的维度，因此其使用范围有一定的局限

### 桥接模式适用场景

* 如果一个系统需要在抽象化和具体化之间增加更多的灵活性，避免在两个层次之间建立静态的继承关系，通过桥接模式可以使它们在抽象层建立一个关联关系。
* 抽象部分和实现部分可以以继承方式独立扩展而互不影响，在程序运行时可以动态将一个抽象化子类的对象和一个实现化子类的对象进行组合。
* 一个类存在两个（或者多个）不同变化的维度，且这两个（或多个）维度都需要独立进行扩展。
* 对那些不希望使用继承或因为多层继承导致系统类的个数急剧增加的系统，桥接模式尤为适用
