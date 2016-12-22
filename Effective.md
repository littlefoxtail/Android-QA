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

```java
class Movie {
    [...]
    public static Movie create(String title) {
        return new Movie(title);
    }
}
```

创建者模式

当对象的构造方法参数不小于3个时，可以考虑创建者模式。这可能需要更多行代码，但扩展性和可读性会很好。如果你正创建一个实体类。

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



