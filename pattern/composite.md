# Composite(组合模式)

组合多个对象形成树形结构以表示具有“整体-部分”关系的层次结构。组合模式对单个对象和组合对象的使用具有一致性，组合模式又可以称为“整体-部分”模式，它是一种对象结构性模式。

* Component（抽象构件）：可以是接口或抽象类，为叶子构件和容器构件对象声明接口，在该角色中可以包含所有子类共有行为的声明和实现。
    在抽象构件中定义了访问及管理它的子构件的方法

    ```java
    public abstract class LetterComposite {
        private List<LetterComposite> children = new ArrayList<>();

        public void add(LetterComposite letter) {
            childred.add(letter);
        }

        public int count() {
            return children.size();
        }

        protected void printThisBefore(){}

        protected void printThisAfter(){}

        public void print() {
            printThisBefore();
            for(LetterComposite letter : childred) {
                letter.print();
            }
            printThisAfter();
        }
    }
    ```

* Leaf（叶子构件）：它在组合结构中表示叶子节点对象，容器节点包含子节点，其子节点可以是叶子节点，也可以容器节点，它提供一个集合用于存储子节点，
    实现了在抽象构件中定义的行为，包括哪些访问及管理子构件的方法，在其业务方法中可以递归调用其子节点的业务方法

    ```java
    public class Letter extends LetterComposite {
        private char c;
        public Letter(char c) {
            this.c = c;
        }
        @Override
        protected void printThisBefore() {
            System.out.print(c);
        }
    }
    ```

* Composite（容器构件）：它在组合结构中表示容器节点对象，容器节点包含子节点，其子节点可以是叶子节点，也可以是容器节点，它提供一个集合用于存储子节点，实现了在抽象构件中定义的行为
  
    ```java
    public class Word extends LetterComposite {
        public Word(List<Letter> letters) {
            for (Letter l : letters) {
                this.add(l);
            }
        }

        @Override
        protected void printThisBefore() {
            System.out.print(" ");
        }
    }
    ```

    ```java
    public class Sentence extends LetterComposite {
        public Sentence(List<Word> words) {
            for(Word w : words) {
                this.add(w);
            }
        }

        @Override
        protected void printThisAfter() {
            System.out.print(" ");
        }
    }
    ```

## 组合模式总结

组合模式的关键是定义了一个抽象构件类，它既可以代表叶子，又可以代表容器，而客户端对该抽象构件类进行编程
组合模式使用面向对象的思想来实现树形结构的构建与处理，描述了如何将容器对象和叶子对象进行递归组合，实现简单，灵活性好

### 组合模式优点

* 组着模式可以清楚地定义分层次的复杂对象，表示对象的全部或部分层次，它让客户端忽略了层次的差异，方便对整个层次结构进行控制
* 客户端可以一致的使用一个组合结构或其中单个对象，不必关心处理的是单个对象还是整个组合结构，简化了客户端代码
* 在组合模式增加新的容器构件和叶子构件都很方面，符合“开闭原则”
* 组合模式为树形结构的面向对象实现提供了一种灵活的解决方案，通过叶子对象和容器对象的递归组合，可以形成复杂的树形结构，但对树形结构的控制却非常简单

### 组合模式缺点

在增加新的构件时难对容器中的构建类型进行限制。

### 组合模式场景

* 在具有整体和部分的层次结构中，希望通过一种方式忽略整体与部分的差异，客户端可以一致地对待它们
* 需要处理一种树形结构
* 在一个系统中能够分离出叶子对象和容器对象，而且它们的类型不固定，需要增加一些新的类型

## 组合模式实例

* [java.awt.Container](http://docs.oracle.com/javase/8/docs/api/java/awt/Container.html) and [java.awt.Component](http://docs.oracle.com/javase/8/docs/api/java/awt/Component.html)
* [Apache Wicket](https://github.com/apache/wicket) component tree, see [Component](https://github.com/apache/wicket/blob/91e154702ab1ff3481ef6cbb04c6044814b7e130/wicket-core/src/main/java/org/apache/wicket/Component.java) and [MarkupContainer](https://github.com/apache/wicket/blob/b60ec64d0b50a611a9549809c9ab216f0ffa3ae3/wicket-core/src/main/java/org/apache/wicket/MarkupContainer.java)