# Facade(外观模式)

## 定义

外部与一个子系统的通信必须通过一个统一的外观对象进行，为子系统中的一组接口提供一个一致的界面，外观模式定义了一个高层接口，这个接口使得这一子系统更加容易使用。

## 结构

- 意图：为子系统中的一组接口提供一个一致的界面，Facad模式定义了一个高层接口，这个接口使得这一子系统更加容易使用。

角色：

- Facade（外观角色）：在客户端可以调用它的方法，在外观角色中可以知道相关的子系统的功能和责任；在正常情况下，它将所有从客户端发来的请求委派到相应的子系统去，传递给相应的子系统对象处理

    ```java
    public class DwarvenGoldmineFacade {
        private final List<DwarvenMineWorker> workers;

        public DwarvenGoldmineFacade() {
            worker = new ArrayList<>();
            worker.add(new DwarvenGoldDigger());
            worker.add(new DwarvenCartOperator());
            worker.add(new DwarvenTunnelDigger());
        }

    }

    ```

- SubSystem（子系统角色）：在软件系统中可以有一个或者多个子系统角色，每一个子系统可以不是一个单独的类，而是一个类的集合，它实现子系统的功能；每一个子系统都可以客户端直接调用，或者被外观角色调用，它处理由外观类传过来的请求；子系统并不知道外观的存在，对于子系统而言，外观角色仅仅是另一个客户端而已。

    ```java
    public abstract class DwarvenMineWorker {
        public void goToSleep() {

        }

        public void wakeUp() {

        }

        public void goHome() {

        }

        public void goToMine() {
            
        }

        private void action(Action action) {

        }
    }
    ```

    ```java
    public class DwarvenCartOperator extends DwarvenMineWorker {
        @Override
        public void work() {

        }

        @Override
        public String name() {

        }
    }
    ```

    ```java
    public class DwarvenGoldDigger extends DwarvenMineWorker {
        @Override
        public void work() {

        }

        @Override
        public String name() {

        }
    }
    ```

## 总结

它通过引入一个外观角色来简化客户端与子系统之间的交互，为复杂的子系统调用提供一个统一的入口，使子系统与客户端的耦合度降低，且客户端调用非常方便。

它的目的在于降低系统的复杂程度，在面向对象系统中，类与类之间的关系越多，不能表示系统设计越好，反而表示系统中类之间耦合度太大，这样的系统在维护和修改时都缺乏灵活性，外观模式的引入在很大程度上降低了类与类的耦合关系。引入外观模式之后，增加新的子系统或者移除子系统都非常方便，客户类无须进行修改（或者极少的修改），只需要在外观类中增加或移除对子系统的引用即可。从这一点来说，外观模式在一定程度上并不符合开闭原则，增加新的子系统需要对原有系统进行一定的修改，虽然这个修改工作量不大。

### 优点

1. 它对客户端屏蔽了子系统组件，减少客户端所需处理的对象数目，并使得子系统使用起来更加容易。通过引入外观模式，客户端代码将变得很简单，与之关联的对象也很少。
2. 它实现了子系统与客户端之间的松耦关系，这使得子系统的变化不会影响到调用它的客户端，只需要调整外观类即可。
3. 一个子系统的修改对其他子系统没有任何影响，而且子系统内部变化也不会影响到外观

### 缺点

1. 不能很好地限制客户端使用子系统类，如果对客户端访问子系统类做大多数的限制则减少可变性和灵活性。
2. 如果设计不当，增加新的子系统可能需要修改外观类的源代码，违背了开闭原则。

### 模式适应场景

1. 当你要为一个复杂子系统提供一个简单接口时。子系统往往因为不断演化而变得越来越复杂。大多数模式使用时都会产生更多更小的类。
这使得子系统更具可重用性，也更容易对子系统进行定制，但这也给那些不需要定制的子系统的用户带来一些使用上的困难。facad可以提供一个简单的缺省视图，
这一视图对大多数用户来说已经足够，而那些需要更多的可定制性的用户可以越过facade层。
2. 客户程序与抽象类的实现部分之间存在着很大的依赖性。引入facade将这个子系统与客户以及其他子系统分离，可以提高子系统的独立性和可移植性。
3. 在层次化结构中，可以使用外观模式定义系统中每一层的入口，层与层之间不直接产生联系，而通过外观类建立联系，降低层之间的耦合度
