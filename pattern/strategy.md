# Strategy(策略模式)

定义了一系列算法类，将每一个算法封装起来，并让他们可以相互替换，策略模式让算法独立于使用它的客户而变化，也称为政策模式(Policy)。

策略模式的主要目的是将算法的定义和使用分开，也就是将算法的行为和环境分开，将算法的定义放在专门的策略类中。
每一个策略类封装了一种实现算法，使用算法的环境类针对抽象策略类进行编程，符合“依赖倒转原则”

* Context（环境类）；
    环境类是使用算法的角色，它在解决某个问题时可以采取多种策略或者具体类。在环境类中维持一个对抽象策略的引用实例，用于定义所采用的策略

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

* Strategy(抽象策略类)：它为所支持的算法声明了抽象方法，是所有策略类的父类，它可以是抽象类或具体类，也可以是接口。环境类通过抽象策略类中声明的方法在运行时调用具体策略类中实现的算法。

    ```java
    public interface DragonSlayingStrategy {
        void execute();
    }
    ```

* ConcreteStrategy(具体策略类)：它实现了在抽象策略类中声明的算法，在运行时，具体策略类将覆盖在环境类中定义的抽象策略类对象，使用一种具体的算法实现某个业务处理。

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

## 策略模式总结

### 策略模式优点

* 策略模式提供了对“开闭原则”的完美支持，用户可以在不修改原有系统的基础选择算法或行为，也可以灵活地增加新的算法或行为
* 策略模式提供了管理相关的算法族的办法。策略类的结构等级结定义了一个算法或行为族，恰当使用继承可以把公共的代码移到抽象策略类中
* 策略模式提供了一种可以替换继承关系的办法
* 使用策略模式可以避免多重条件选择语句。多重条件语句不易维护，它把采取哪一种算法或行为的逻辑与算法或者行为本身的实现逻辑混合在一起，将它们全部硬编码(Hard Coding)在一个庞大的多重条件选择语句中，比直接继承环境类的办法还要原始和落后
* 策略模式提供了一种算法的复用机制，由于将算法单独提取出来封装在策略中，因此不同的环境类可以方便地复用这些策略类

### 策略模式缺点

* 客户端必须知道所有的策略类，并自行决定使用哪一个策略类
* 策略模式将造成系统产生很多具体策略类
* 无法同时在客户端使用多个策略类

### 策略模式适用场景

1. 系统需要动态地在几种算法中选择一种，那么可以将这些算法封装到一个个的具体算法类中，而这些具体算法类都是一个抽象算法类的子类。
2. 一个对象有很多的行为，如果不用恰当的模式，这些行为就只好使用多重条件选择语句来实现。此时，使用策略设计模式，把这些行为转移到相应的具体策略类，就可以避免使用难以维护的多重条件选择语句
3. 不希望客户端知道复杂的、与算法相关的数据结构，可以提高算法的保密性与安全性
