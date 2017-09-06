# 1.在Finally中清理资源或者使用Try-With-Resource语句
通常情况下，你在try中使用了一个资源，比如InputStream，之后需要关闭它。在这种情况下，一个常见的错误是在try
的末尾关闭了资源。

```java
public void doNotCloseResourceInTry() {
    FileInputStream inputStream = null;
    try {
        File file = new File("./tmp.txt");
        inputStream = new FileInputStream(file);
        // do Not do this f**k
        inputStream.close();
    } catch (FileNotFoundException e) {
        log.error(e);
    } catch (IOException e) {
        log.error(e);
    }

}
```
只要不抛出异常，这种方法就可以很好的运行。try内的所有语句都将被执行，资源也会别关闭。
但是你在try里调用了一个或多个可能抛出异常的方法，或者自己抛出异常。这意味着可能无法到达try的末尾。因此，将不会关闭这些资源。
所以应该将清理资源的代码放入Finally中，或者使用Try-With-Resource语句。

## 使用Finally
相比于try，无论是在成功执行try里的代码后，或是catch中处理了一个异常后，Finally里的内容是一定会被执行的。因此，可以确保清理所有已打开的资源。

```java
public void closeResourceInFinally() {
    FileInputStream inputStream = null;
    try {
        File file = new File("./tmp.txt");
        inputStream = new FileInputStream(file);
        // use the inputStream to read a file
    } catch (FileNotFoundException e) {
        log.error(e);
    } finally {
        if (inputStream != null) {
            try {
                inputStream.close();
            } catch (IOException e) {
                log.error(e);
            }
        }
    }
}
```

## Java 7 Try-With-Resource语句
[introduction to Java exception handling](https://stackify.com/specify-handle-exceptions-java/#tryWithResource)
如果你的资源实现了AutoCloseable接口，就可以使用它，这正式大多数Java标准资源所做的。当你在try子句中被执行后自动关闭，或者处理一个异常。

```java
public void automaticallyCloseResource() {
    File file = new File("./tmp.txt");
    try (FileInputStream inputStream = new FileInputStream(file);) {
        // use the inputStream to read a file
    } catch (FileNotFoundException e) {
        log.error(e);
    } catch (IOException e) {
        log.error(e);
    }
}
```

# 2.给出准确的异常处理信息
因此，请确保提供尽可能多的信息，这会使你的API更容易理解。因此，你方法的调用者将能够更好地处理异常，或者通过额外的检查来避免它。

所以，要尽量能更好地描述你的异常处理信息，比如用NumberFormatException代替IllegalArgumentException，避免抛出一个不具体的异常。
```java
public void doNotDoThis() throws Exception {
    ...
}
public void doThis() throws NumberFormatException {
    ...
}
```
# 3.记录你所指定的异常
当你在方法中指定一个异常时，你应该在Javadoc中记录下它。这与前面提到的方法有着相同的目标：
为调用者提供尽可能多的信息，这样他们就可以避免异常或者更容易地处理异常。
因此，请确保在JavaDoc中添加一个@throws声明，并描述可能导致的异常情况。
```java
/**
 * This method does something extremely useful ...
 *
 * @param input
 * @throws MyBusinessException if ... happens
 */
public void doSomething(String input) throws MyBusinessException {
    ...
}
```
# 4.使用描述性消息抛出异常
应该尽可能准确地描述问题，并提供相关的信息来了解异常事件。

如果抛出一个特定的异常，它的类名很可能已经描述了这种类型的错误。所以，你不需要提供很多额外的信息。一个很好的例子就是，当你以错误的格式使用字符串时，如NumberFormatException，它就会被类 java.lang.Long的构造函数抛出。
```java
try {
    new Long("xyz");
} catch (NumberFormatException e) {
    log.error(e);
}
```

# 5.最先捕获特定的异常
```java
public void catchMostSpecificExceptionFirst() {
    try {
        doSomething("A message");
    } catch (NumberFormatException e) {
        log.error(e);
    } catch (IllegalArgumentException e) {
        log.error(e)
    }
}
```

# 6.不要在catch中使用Throwable
Throwable是exceptions 和 errors的父类。当然，你可以在catch子句中使用它，但其实你不应该这样做。

如果你在catch子句中使用Throwable，它将不仅捕获所有的异常，还会捕获所有错误。JVM会抛出错误，这是应用程序不打算处理的严重问题。典型的例子是OutOfMemoryError或StackOverflowError。这两种情况都是由应用程序控制之外的情况引起的，无法处理。

所以，最好不要在catch中使用Throwable，除非你完全确定自己处于一个特殊的情况下，并且你需要处理一个错误。

```java
public void doNotCatchThrowable() {
    try {
        // do something
    } catch (Throwable t) {
        // don't do this!
    }
}
```
# 7.不要忽略Exceptions


# 8.不要记录和抛出一个异常
不添加任何额外的信息。正如在上述第4个中所解释的那样，异常消息应该描述异常事件。堆栈会告诉你在哪个类、方法和行中异常被抛出。

如果你需要添加额外的信息，应该捕获异常并将其包装在一个自定义的信息中。但要确保遵循下面的第9条。
```java
public void wrapException(String input) throws MyBusinessException {
    try {
        // do something
    } catch (NumberFormatException e) {
        throw new MyBusinessException("A message that describes the error.", e);
    }
}
```
# 9.包装异常
有时最好捕获一个标准异常并将其封装到一个定制的异常中。此类异常的典型例子是应用程序或框架特定的业务异常。这允许你添加额外的信息，并且也可以为异常类实现一个特殊的处理。

当你这样做时，确保引用原始的异常处理。Exception类提供了一些特定的构造函数方法，这些方法可以接受Throwable作为参数。否则，你将丢失原始异常的堆栈跟踪和消息，这将使你很难分析导致异常的事件。
```java
public void wrapException(String input) throws MyBusinessException {
    try {
        // do something
    } catch (NumberFormatException e) {
        throw new MyBusinessException("A message that describes the error.", e);
    }
}
```