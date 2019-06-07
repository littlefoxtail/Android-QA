# 原型模式（Prototype）

使用原型实例指定创建对象的种类，并且通过拷贝这些原型创建新的对象。原型模式是一种对象创建型模式

原型模式的工作原理：将一个原型对象传给那个要发动创建的对象，这个要发动创建的对象通过请求原型对象拷贝自己来实现创建过程。

* Prototype（抽象原型类）：它是声明克隆方法的接口，是所有具体原型类的公共父类，可以是抽象类也可是接口，甚至还可以是具体实现类
  
    ```java
    public abstract class Prototype implements Cloneable {
        public abstract Object copy() throws CloneNotSupportedException;
    }
    ```

* ConcretePrototype（具体原型类）：它实现在抽象原型类中声明的克隆方法，在克隆方法中返回自己的一个克隆对象
  
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

* Client（客户端）：让一个原型对象克隆自身从而创造一个新的对象，在客户类中只需要直接实例化或通过工厂方法等方式创建一个原型对象，再通过调用该对象的克隆方法即可得到多个相同的对象

    ```java
    Prototype obj1  = new OrcBeast("laser");
    Prototype obj2 = obj.copy();
    ```

## 原型模式总结

原型模式作为一种快速创建大量相同或者相似对象的方式，在软件开发中应用较为广泛，很多软件提供的复制(Ctrl + C)和粘贴(Ctrl + V)操作就是原型模式的典型应用，下面对该模式的使用效果和适用情况进行简单的总结。

### 原型模式优点

* 当创建新的对象实例较为复杂时，使用原型模式可以简化的创建过程，通过复制一个已有实例可以提高新实例的创建效率
* 扩展性较好，由于在原型模式中提供了抽象原型类，在客户端可以针对抽象原型类进行编程，而将具体原型类写在配置文件中，增加或减少产品类对原有系统都没有任何影响
* 原型模式提供了简化的创建结构，工厂方法模式常常需要有一个与产品类等级结构相同的工厂等级结构，而原型模式就不需要这样，原型模式中产品的复制是通过封装在原型类中的克隆方法实现的，无须专门的工厂类来创建产品
* 可以使用深克隆的方式保存对象的状态，使用原型模式将对象复制一份并将其状态保存起来，以便在需要的时候使用，可辅助实现撤销操作

### 原型模式缺点

* 需要为每一个类配备一个克隆方法，而且该克隆方法位于一个类的内部，当对已有的类进行改造时，需要修改源代码，违背了“开闭原则”
* 在实现深克隆时需要编写较为复杂的代码，而且当对象之间存在多重的嵌套引用时，为了实现深克隆，每一层对象对应的类都必须支持深克隆，实现起来可能会比较麻烦

### 原型模式适用场景

* 创建新对象成本较大，新的对象可以通过原型模式对已有对象进行复制来获得，如果是相似对象，则可以对其成员变量稍作修改
* 如果系统要保存对象的状态，而对象的状态变化很小，或者对象本身占用内存较少时，可以使用原型模式配合备忘录模式来实现。
* 需要避免使用分层次的工厂类来创建分层次的对象，并且类的实例对象只有一个或很少的几个组合状态，通过复制原型对象得到新实例可能比使用构造函数创建一个新实例更加方便

## 原型模式实例

* [java.lang.Object#clone()](http://docs.oracle.com/javase/8/docs/api/java/lang/Object.html#clone%28%29)