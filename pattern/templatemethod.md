
# Template Method(模板方法模式)

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

## 总结

模板方法模式是基于继承的代码复用技术，它体现了面向对象的诸多重要思想，是一种使用频繁的模式。模板方法模式广泛应用于框架设计中，
以确保通过父类来控制处理流程的逻辑顺序

### 优点

* 在父类形式化地定义一个算法，而由它的子类来实现细节的处理，在子类实现详细的处理算法时并不会改变算法中步骤的执行次序
* 模板方法模式是一种代码复用技术，它在类库设计中尤为重要，它提取了类库中的公共行为，将公共行为放在父类中，而通过其子类来实现不同的行为，它鼓励我们恰当使用继承来实现代码复用
* 可实现一种反向控制结果，通过子类覆盖父类的钩子方法来决定某一特定步骤是否需要执行
* 在模板方法模式中可以通过子类来覆盖父类的基本方法，不同的子类可以提供基本方法的不同实现，更换和增加新的子类很方便，符合单一职责原则和开闭原则

### 缺点

* 需要为每一个基本方法的不同实现提供一个子类，如果父类中可变的基本方法太多，将会导致类的个数增加，系统庞大，此时可结合桥接模式来进行设计

### 适用场景

* 对一些复杂算法进行分离，将其算法中固定不变的部分设计为模板方法和父类具体方法，而一些可以改变的细节由其子类来实现
* 各子类中公共的行为应被提取出来并集中到一个公共父类以避免代码重复
* 需要通过子类来决定父类算法中某个步骤是否执行，实现子类对父类的反向控制
