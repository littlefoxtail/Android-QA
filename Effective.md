[https://zhuanlan.zhihu.com/p/24490196](https://zhuanlan.zhihu.com/p/24490196)

#### 强制不可实例化

如果你不希望一个对象通过关键字 *new* 来创建，那么强制让它的**构造方法私有**。这尤其对一些只包含静态方法的工具类有用。

```java
class MovieUtils {
  private MovieUtils() {}

    static String titleAndYear(Movie movie) {
        [...]
    }
}
```

#### 静态工厂方法

不要使用new关键字和构造方法创建对象，而应当使用静态工厂方法（和私有构造方法）。这些工厂方法具有名字，需要每次返回一个新的对象实例，它们可以根据需求返回不同的子类型对象。

##### 优点

1. 静态工厂方法与构造器不同的优势，它们是有名称的

2. 不必在每次调用它们的时候都创建一个新对象

3. 它们可以返还原返回类型任何子类型的对象（增加了灵活性）

4. 在创建参数化类型实例的时候，它们使得代码变得更加简洁

##### 缺点

1. 类如果不含共有的或者受保护的构造器，就不能被子类化。

2. 它们与其他静态方法实际上没有任何区别

   遵守标准的命名习惯，一定量可以弥补这一劣势。

   - valueOf——这样的静态工厂方法实际上是类型转换方法。
   - of——valueOf的一种更为简洁的替代。
   - getInstance——返回的实例是通过方法的参数来描述的，但是不能够说与参数具有同样的值。对于Singleton来说，该方法没有参数，并返回唯一的实例。
   - newInstance——像getInstance一样，但newInstance能够确保返回的每个实例都与所有其他实例不同。
   - getType——像getInstance一样，但在工厂方法处于不同的类中的时候使用。Type表示工厂方法所返回的对象类型。
   - newType——像newInstance一样，但是工厂方法处于不同的类中的时候使用。Type表示工厂方法所返回的对象类型

   ​

```java
class Movie {
    [...]
    public static Movie create(String title) {
        return new Movie(title);
    }
}
```

#### 创建者模式

当对象的构造方法参数不小于3个时，可以考虑创建者模式。这可能需要更多行代码，但扩展性和可读性会很好。如果你正创建一个实体类。

##### 缺点

为了创建对象，必须先创建它的构造器。虽然创建构造器的开销在实践中可能不那么明显，但是在注重性能情况下，可能出现问题。Builder模式还比重叠构造器模式更加冗长，因此它只在有很多参数的才使用

##### 简而言之

如果类的构造器或者静态工厂中具有多个参数，设计这种累时，**Builder**模式就是种不错的选择，特别是当大多数参数都是可选的手。与重叠构造器模式比易于阅读和编写，构造器也比JavaBeans更加安全。

```
class Movie {

    static Builder newBuilder() {

        return new Builder();

    }

    static class Builder {

        String title;

        Builder withTitle(String title) {

            this.title = title;

            return this;

        }

        Movie build() {

            return new Movie(title);

        }

    }

    private Movie(String title) {

    [...]

    }

}

// Use like this:

Movie matrix = Movie.newBuilder().withTitle("The Matrix").build();

```

#### 避免可变性

不可变性是指对象在整个生命周期内一直保持不变。应将对象中所有对象中所有必要的数据在其创建时就赋值。这个做法有许多好处，比如简洁化，线程安全以及可共享性等。

```java
class Movie{
  [...]
  Movie sequel() {
    return Movie.create(this.title + " 2");
  }
}
// Use like this:
Movie toyStory = Movie.create("Toy Story");
Movie toyStory2 = toyStory.sequel();
```

#### 静态成员类

如果你定义了一个不依赖外部类的内部类，最好将其定义为静态的。否则会导致每一个内部类对象都会持有对外部类的引用。

```java
class Movie {
  [...]
  static class MovieAward {
    [...]
  }
}
```

#### 返回空值

当你方法的返回类型为list/collection时，返回空值时要避免返回null。返回一个空的集合类型，这会使得你简化接口并且避免空指针异常。就返回那个集合的空值，而不是在创建一个。

```java
List<Movie> latestMovies() {
    if (db.query().isEmpty()) {
        return Collections.emptyList();
    }
    [...]
}
```

#### 不要用"+"来连接String

必须要拼接一系列字符串，可能会使用+连字符。永远不要用它来拼接大量字符串，这样的性能很差，考虑使用StringBuilder来代替。

```java
String lastestMovieONeLine(List<Movie> movies) {
  StringBuilder sb = new StringBuilder();
  for(Movie movie : movies) {
    sb.append(movie);
  }
  return sb.toString();
}
```

Recoverable exceptions

#### Recoverable exceptions

我个人不喜欢抛出异常来指示错误，但是如果你这样做，确保这个异常被检查，确保这个**异常被捕获到**。

```java
List<Movie> latestMovies() throws MoviesNotFoundException {
    if (db.query().isEmpty()) {
throw new MoviesNotFoundException();
    }
    [...]
}
```

#### 避免创建不必要的对象

一般来说，最好能重用对象而不是