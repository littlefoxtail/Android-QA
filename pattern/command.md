# Command(命令模式)

需要向某些对象发送请求（调用其中的某个或某些方法），但并不知道请求的接收者是谁，也不知道请求的操作是哪个，此时希望能够以一种松耦合的方式来设计软件

命令模式(Command Pattern)：将一个请求封装为一个对象，从而让我们可用不同的请求对客户进行参数化，对请求排队或者记录请求日志，以及支持可撤销的操作，命令模式是一种对象行为型模式，其别名为动作(Action)或事务(Transaction)模式
命令模式包含的角色：

* Command(抽象命令类)：抽象命令类一般是一个抽象类或接口。在其中声明了用于执行请求的execute()等方法，通过这些方法可以调用请求接收者的相关操作

    ```java
    public abstract class Command {
        public abstract void execute(Target target);

        public abstract void undo();

        public abstract void redo();

        @Override
        publci abstarct String toString();
    }
    ```

* ConcreteCommand(具体命令类)：具体命令类是抽象命令类的子类，实现了在抽象命令类中声明的方法，它对应具体的接受者对象，将接收者对象的动作绑定其中。在实现execute()方法时，将调用接收者对象的相关操作。

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

* Invoker(调用者)：调用者即请求发送者，它通过命令对象来执行请求。一个调用者并不需要在设计时候确定其接收者，因此它只与抽象命令类之间存在关联关系。在程序运行时可以将一个具体命令对象注入其中，在调用具体命令对象的execute()方法，从而实现间接调用请求接收者的相关操作。

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

* Receiver(接收者)：接收者执行与请求相关的操作，它具体实现对请求的业务处理。

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

* Client：

    ```java
    Wizard wizard = new Wizard();
    Goblin goblin = new Goblin();

    goblin.printStatus();

    wizard.castSpell(new ShrinkSpell(), goblin);
    goblin.printStatus();
    ```

## 命令模式总结

命令模式是一种使用频率非常高的设计模式，它可以将请求发送者与接收者解耦，请求发送者通过命令对象来间接引用请求接收者，使得系统具有更好的灵活性和可扩展性。

### 命令模式优点

* 降低耦合度。由于请求者与接收者之间不存在直接引用，因此请求者与接收者完全解耦
* 新的命令可以很容易地加入到系统中，符合开闭原则
* 可以比较容易设计一个命令队列或组合命令
* 为请求的撤销和恢复操作提供了一种设计和实现方案

### 命令模式缺点

导致某些系统有过多的具体命令类

### 命令模式适用场景

* 系统需要将请求和调用者和请求接收者解耦，使得调用者不直接交互。请求调用者无须知道接收者的存在，也无须知道接收者是谁，接收者也无须关心何时被调用
* 系统需要在不同的时间制定请求、将请求排队和执行请求。一个命令对象和请求的初始调用者可以有不同的生命周期，换言之，最初的请求发出者可能已经不存在了，而命令对象本身仍然活动的，可以通过该命令对象去调用请求接收者，而无须关心请求调用者的存在性，可以通过请求日志文件等机制来具体实现
* 系统需要支持命令的撤销和恢复
* 系统需要将一组操作组在一起形成宏命令

## 命令模式实例

* [java.lang.Runnable](http://docs.oracle.com/javase/8/docs/api/java/lang/Runnable.html)
* [Netflix Hystrix](https://github.com/Netflix/Hystrix/wiki)
* [javax.swing.Action](http://docs.oracle.com/javase/8/docs/api/javax/swing/Action.html)
