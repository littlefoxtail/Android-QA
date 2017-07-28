在匿名内部类中外部方法中的局部变量，我们必须手动对将这个局部变量用final关键字修饰（在JDK1.8之后不再需要显示的声明为final，因为这种情况下这个局部变量默认是final的，这是编译器为我们做的，这是JDK1.8的新特性，所以前面的结论仍然成立）

## 四种内部类
* 静态内部类（static inner class）
* 成员内部类（Method inner class）
* 局部内部类（Local inner class）
* 匿名内部类（Anonymous inner class）

### 
```java
public void start(int interval, final boolean beep)
{
    class TimePrinter implements ActionListener
    {
        public void actionPerformed(ActionEvent event)
        {
            Date now = new Date();
            System.out.println("At the tone, the time is " + now);
            if (beep) Toolkit.getDefaultToolkit().beep();
        }
    }

    ActionListener listener = new TimePrinter();
    Timer t = new Timer(interval, listener);
    t.start();
}
```

```java
class TalkingClock$1TimePrinter
{
    TalkingClock$1TimePrinter(TalkingClock, boolean);
    public void  actionPerformed(java.awt.event.ActionEvent);
    final boolean val$beep;
    final TalkingClock this$0;
}
```

局部内部类中访问的这些final修饰的局部变量，都会作为局部内部类的由final修饰的成员变量，并在构造中传入值初始化
