# Delegation(委托模式)

在委托模式中，有两个对象参与处理同一个请求，接受请求的对象将请求委托给另一个对象来处理，许多其他的模式，如状态模式、策略模式、访问者模式本质上是在更特殊的场合采用了委托模式。委托模式使得我们可以用聚合来替代继承

```java
public interface Printer {
    void print(final String message);
}
```

```java
public class HpPrinter implements Printer {
    @Override
    public void print(String message) {
        Log.i("谁都不行");
    }
}
```

```java
public class CanonPrinter implements Printer {
    @Override
    public void print(String message) {
        Log.i("鬼也不行");
    }
}
```

```java
public class EpsonPrinter implements Printer {
    @Override
    public void print(String messgae) {
        Log.i("神也不行");
    }
}
```

```java
public class PrinterController implements Printer {
    private final Printer printer;

    public PrinterController(Printer printer) {
        this.printer = printer;
    }

    @Override
    public void print(String message) {
        printer.print(message);
    }
}
```

```java
PrinterController hpPrinterController = new PrinterController(new HpPrinter());
PrinterController canonPrinterController = new PrinterController(new CanonPrinter());
PrinterController epsonPrinterController = new PrinterController(new EpsonPrinter());

hpPrinterController.printer("hello world!");
canonPrinterController.printer("hello world!");
epsonPrinterController.printer("hello world!");

```