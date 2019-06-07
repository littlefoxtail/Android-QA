# State(状态模式)

允许一个对象在其内部状态改变时改变它的行为，对象看起来似乎修改了它的类。状态模式是一种对象行为型模式。

* Context(环境类)：环境类又称为上下文类，它是拥有多种状态的对象。由于环境类的状态存在多样性且在不同状态下对象的行为有所不同，因此将状态独立出去形成单独的状态类。在环境类中维护了一个抽象状态State的实例。

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

* State（抽象状态类）：它用于定义一个接口以封装与环境类的一个特定状态相关的行为，在抽象状态类中声明了各种不同状态对应的方法，而在其子类中实现类这些方法，由于不同状态下的对象的行为可能不同，因此在不同子类中方法的实现可能存在不同，相同方法可以写在抽象状态类中

    ```java
    public interface State {
        void onEnterState();

        void observe();
    }
    ```

* ConcreteState(具体状态类)：它是抽象状态类的子类，每一个子类实现一个与环境类的一个状态相关的行为，每一个具体状态类对应环境的一个具体状态，不同具体状态类其行为有所不同。

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

### 状态模式总结

状态模式将一个对象在不同状态下的不同行为封装在一个个状态类中，通过设置不同的状态对象可以让环境对象拥有不同的行为，而状态转换的细节对于客户端而言是透明的，方便了客户端的使用。

### 状态模式优点

* 封装了状态的转换原则，在状态模式中可以将状态转换代码封装在环境类或者具体状态类中，可以对状态转换代码进行集中管理，而不是分散在一个业务方法中
* 将所有与某个状态有关的行为放到一个类中，只需要注入一个不同的状态对象即可使环境对象拥有不同行为
* 允许状态转换逻辑与状态对象合成一体，而不是提供一个巨大的条件语句块，状态模式可以避免用庞大的条件语句来将业务方法和状态转换代码交织在一起
* 可以让多个环境对象共享一个状态对象，从而减少系统中对象的个数

### 状态模式缺点

* 增加系统中类和对象的个数，导致系统运行开销增大
* 状态模式的结构和实现都较为复杂，如果使用不当将导致程序结构和代码混乱，增加系统设计难度
* 状态模式对“开闭原则”的支持并不太好，增加新的状态类需要修改那些负责状态转换的源代码

### 状态模式适用场景

* 对象的行为依赖于它的状态，状态的改变将导致行为的变化
* 一个操作中含有庞大的多分支的条件语句，且这些分支依赖于该对象的状态。这状态通常用一个或多个枚举常量表示。通常，有多个操作包含这一相同的条件结构。
    State模式将每一个条件分支放入一个独立的类中。这使得你可以根据对象自身的情况将对象的状态作为一个对象，这一对象可以不依赖于其他对象独立变化。

## 状态模式实例

* [javax.faces.lifecycle.Lifecycle#execute()](http://docs.oracle.com/javaee/7/api/javax/faces/lifecycle/Lifecycle.html#execute-javax.faces.context.FacesContext-) controlled by [FacesServlet](http://docs.oracle.com/javaee/7/api/javax/faces/webapp/FacesServlet.html), the behavior is dependent on current phase of lifecycle.
* [JDiameter - Diameter State Machine](https://github.com/npathai/jdiameter/blob/master/core/jdiameter/api/src/main/java/org/jdiameter/api/app/State.java)
