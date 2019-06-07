# Proxy(代理模式)

代理设计模式是常用的结构型设计模式之一，当无法直接访问某个对象或者访问某个对象存在困难时候可以通过一个代理对象来间接访问。为了保证客户端使用的透明性，所访问的真实对象与代理对象需要实现相同的接口。
定义：代理模式：给某一个对象提供一个代理或占位符，并由代理对象来控制对原对象的访问。

* Subject（抽象主题角色）：它声明了真实主题和代理主题的共同接口，这样一来在任何使用真实主题的地方都可以使用代理主题，客户端通常需要针对抽象主题角色进行编程

    ```java
    public interface WizardTowser {
        void enter(Wizard wizard);
    }
    ```

* Proxy（代理主题角色）：它包含了对真实主题的引用，从而可以在任何时候操作真实主题对象；在代理主题角色中提供了一个真实主题角色相同的接口，以便在任何时候都可以替代真实主题；代理主题角色还可以控制对真实主题的使用，负责在需要的时候创建和删除真实主题对象，并对真实主题对象的使用加以约束。

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

* RealSubject（真实主题角色）：它定义了代理角色所代表的真实对象，在真实主题角色中实现了真实的业务操作，客户端可以通过代理主题角色间接调用真实主题角色中定义的操作

    ```java
    public class IvoryTowser implements WizardTowser {
        public void enter(Wizard wizard) {
            log.i("看不见");
        }
    }
    ```

代理模式根据其目的和实现方式不同可分为很多种类

1. 远程代理：为一个位于不同的地址空间的对象提供一个本地的代理对象，这个不同的地址空间可以是在同一台主机中，也可是在另一台主机中，远程代理又称为(Ambassador)
2. 虚拟代理：如果需要创建一个资源消耗较大的对象，先创建一个消耗较小的对象来表示，真实对象只在需要时才会被真正创建
3. 保护代理：控制同一个对象的访问，可以给不同的用户提供不同级别的使用权限
4. 缓冲代理：为某一个目标操作的结果提供临时的存储空间，以便多个客户端可以共享这些结果
5. 只能引用代理：当一个对象被引用时，提供一些额外的操作，例如将被调用的次数记录下来

在这些常用的代理模式中，有些代理类的设计非常复杂，例如远程代理类，它封装了底层网络通信和对远程对象的调用，其实现较为复杂。

## 代理模式总结

代理模式是常用的结构性设计模式之一，它为对象的简介访问提供了一个解决方案，可以对对象的访问进行控制。

### 代理模式优点

* 能够协调调用者和被调用者，在一定程度上降低了系统的耦合度
* 客户端可以针对抽象主题角色进行编程，增加和更换代理类无须修改源代码，符合开闭原则，系统具有较好的灵活和可扩展性

### 代理模式缺点

* 由于在客户单和真实主题之间增加了代理对象，因此有些类型的代理模式可能会造成请求的处理速度变慢
* 实现代理模式需要额外的工作，而且有些代理模式实现过程较为复杂

### 代理模式适用场景

* 当客户端对象需要访问远程主机中的对象时可以使用远程代理
* 当需要用一个消耗资源较少的对象来代表一个消耗资源较多的对象，从而降低系统开销、缩短运行时间时可以使用虚拟代理，例如一个对象需要很长时间才能完成加载

* 当需要为某一个被频繁访问的操作结果提供一个临时存储空间，以供多个客户端共享访问这些结果时可以使用缓冲代理。通过使用缓冲代理，系统无须在客户端每一次访问时都重新执行操作，只需直接从临时缓冲区获取操作结果即可。
* 当需要控制对一个对象的访问，为不同用户提供不同级别的访问权限时可以使用保护代理。
* 当需要为一个对象的访问（引用）提供一些额外的操作时可以使用智能引用代理。

## 代理模式实例

* [java.lang.reflect.Proxy](http://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Proxy.html)
* [Apache Commons Proxy](https://commons.apache.org/proper/commons-proxy/)
* Mocking frameworks Mockito, Powermock, EasyMock

![proxy](/img/proxy.png)

