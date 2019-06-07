# Observer(观察者模式)

## 模式动机

 定义对象间的一宗一对多的依赖关系，观察者模式描述了如何建立对象与对象之间的关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。发生改变的对象称为观察目标，而被通知的对象称为观察者，一个观察目标可以对应多个观察者，而且这些观察者之间没有相互联系，可以根据需要增加和删除观察者。

## 模式定义

观察者模式(Observer Pattern):观察者模式又叫做发布-订阅(Publish/Subscribe)模式、模型-视图(Model/View)模式、源-监听器(Source/Listener)模式

* Subject(目标):目标指被观察者。在目标中定义了一个观察者集合，一个观察目标可以接受任意数量的观察者来观察，它提供了一系列方法来增加和删除观察者对象，同时它定义了nofity()。目标类可以是抽象类或者具体类

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

* ConcreteSubject（具体目标）：具体目标是目标类的子类，通常包含有经常发生改变的数据，当它的状态发生改变时，向它的各个观察者发出通知；同时它还实现了在目标类中定义的抽象业务逻辑方法。如果无须扩展目标类，则具体目标类可以省略

* Observer（观察者）:观察者将堆观察目标的改变作出反应，观察者一般定义为接口，该接口声明了更新数据的`update`，因此称为抽象观察者

    ```java
    public interface WeatherObserver {
        void update(WeatherType currentWeather);
    }
    ```

* ConcreteObserver（具体观察者）：在具体观察者中维护一个指向具体目标对象的引用，它存储具体观察者的有关状态，这些状态需要和具体目标的状态保持一致；它实现了抽象观察者Observer中定义的update方法。通常实现时，可以调用具体目标类的attach将自己添加到目标类的集合/detach将自己从目标类的集合中删除

    ```java
    public class Orcs implements WeatherObserver {

        public void update(WeatherType currentWeather) {
            Log.i("update weather update")
        }
    }
    ```

## 观察者模式总结

观察者模式实现对象之间的联动提供了一套完整的解决方案，凡是涉及到一对一或者一对多的对象交互场景都可以使用观察者模式

### 观察者模式优点

* 观察者模式可以实现表示层和数据逻辑层的分离，并定义了稳定的消息更新传递机制，抽象了更新接口，使得可以有各种各样不同的表示层作为具体观察者角色。
* 观察者模式在观察目标和观察者之间建立了一个抽象的耦合。
* 观察者模式支持广播通信。
* 观察者模式符合"开闭原则"的要求

### 观察者模式缺点

* 如果一个观察目标对象有很多直接和间接的观察者的话，将所有的观察者都通知会花费很多时间
* 如果观察者和观察目标之间有循环依赖的话，观察目标会触发它们之间进行循环调用，可能导致系统崩溃
* 观察者模式没有相应的机制让观察者知道所观察的目标对象是怎么发生变化的，而仅仅只是知道观察目标发生了变化

* 适用性
    1. 当一个抽象模型有两个方面，其中一个方面依赖于另一个方面。将两者封装在独立的对象中以使它们可以各自独立地改变和复用
    2. 当对一个对象的改变需要同时改变其它对象，而不知道具体有多少对象有待改变。
    3. 当一个对象必须通知其它对象，而它又不能假定其它对象是谁。换言之，你不希望这些对象是紧密耦合的。

### 观察者模式使用场景

* 一个抽象模型有两个方面，其中一个方面依赖于另一个方面，将两个方面封装在独立的对象中使他们可以各自独立地改变和复用
* 一个对象的改变将导致一个或多个对象也发生改变，而并不知道具体有多少对象将发生改变，也不知道这些对象是谁
* 需要在系统中创建一个触发链，A对象的行为将影响B对象，B对象的行为将影响C对象，可以使用观察者模式创建一种链式触发机制

## 观察者模式实例

* [java.util.Observer](http://docs.oracle.com/javase/8/docs/api/java/util/Observer.html)
* [java.util.EventListener](http://docs.oracle.com/javase/8/docs/api/java/util/EventListener.html)
* [javax.servlet.http.HttpSessionBindingListener](http://docs.oracle.com/javaee/7/api/javax/servlet/http/HttpSessionBindingListener.html)
* [RxJava](https://github.com/ReactiveX/RxJava)
