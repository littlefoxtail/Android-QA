[UML关系](http://www.open-open.com/lib/view/open1328059700311.html)
在UML类图中，常见的有以下几种关系：
* 泛化(Generalization) 继承关系 指向父类
* 实现(Realization) 实现关系 指向接口
* 关联(Association) 成员变量 指向成员变量
* 聚合(Aggregation) 成员变量 指向成员变量
* 组合(Composition) 成员变量 指向成员变量
* 依赖(Dependency) 局部变量、方法参数和静态方法的调用 指向被调用者

# 泛化(Generalization)
是一种继承关系，表示一般与特殊的关系，它指定了子类如何特化父类的所有特征和行为。
[箭头指向]：带三角箭头的实线，箭头指向父类。

![image](http://note.youdao.com/yws/api/personal/file/WEBc0bdfb5b8048a99ac89144769255e0d6?method=download&shareKey=953c23004a241d458bbca1a18c15370c)

# 实现(Realization)
[实现关系]：是一种类与接口的关系，表示类是接口所有特征和行为的实现。
[箭头指向]：带三角箭头的虚线，箭头指向接口

![image](http://note.youdao.com/yws/api/personal/file/WEBf2e41e25a1668d0da80f66d51dd9d28f?method=download&shareKey=5c37ad1f06fc3b34684b5b0ae8b40444)

# 关联(Association)
【关联关系】：是一种拥有的关系，它使一个类知道另一个类的属性和方法；它可以是双向的，也可以是单向的。双向的关联可以有两个箭头或者没有箭头，单向的关联有一个箭头。
【代码体现】：成员变量
【箭头及指向】：带普通箭头的实心线，指向被拥有者。

![image](http://note.youdao.com/yws/api/personal/file/WEB5958360ebd276cf1f17cfd732f561424?method=download&shareKey=6f6805a563c2047f89ab5259e11d3f74)

## 自关联：
![image](http://note.youdao.com/yws/api/personal/file/WEB8afa1cf89c051b8c2e0befa2993edcde?method=download&shareKey=e8d71b1b6b4c511859eda4e456e26bec)

4.聚合(Aggregation)
【聚合关系】：是整体与部分的关系，且部分可以离开整体而单独存在。如车和轮胎是整体和部分的关系，轮胎离开车仍然可以存在。
聚合关系时关联关系的一种，是强的关联关系；关联和聚合在语法上无法区分，必须考虑具体的逻辑关系。
【代码体现】：成员变量
【箭头及指向】：带空心菱形的实心线，菱形指向整体
![image](http://note.youdao.com/yws/api/personal/file/WEB8567c7c8fc583fb903507443197887d7?method=download&shareKey=c1544de8bf97bc15374a1f0da0a62595)

# 组合(Composition)
【组合关系】：是整体与部分的关系，但部分不能离开整体而单独存在。如公司和部门是整体和部分的关系，没有公司就不存在部门。
组合关系时关联关系的一种，是比聚合关系还要强的关系，它要求普通的聚合关系中代表整体的对象负责代表部分的对象的生命周期。
【代码体现】：成员变量
【箭头及指向】：带实心菱形的实线，菱形指向整体

![image](http://note.youdao.com/yws/api/personal/file/WEB02f477c34ded15c6c7cd002f88e63c21?method=download&shareKey=ade6b8883cbbe8534c9152d6abe35830)

# 依赖(Dependency)
【依赖关系】：是一种使用的关系，即一个类的实现需要另一个类的协助，所以尽量不使用双向的互相依赖。
【代码表现】：局部变量、方法的参数或者对静态方法的调用
【箭头指向】；带箭头的虚线，指向被使用者
![image](http://note.youdao.com/yws/api/personal/file/WEB9fded39d5a645d14d562dcb21f8603fc?method=download&shareKey=2e8d0e8ff87e389063f5bf9810de5d15)

各种关系的强弱顺序：
泛化=实现>组合>聚合>关联>依赖
下面这张UML图，比较形象的展示了各种类图关系：

![lie](http://note.youdao.com/yws/api/personal/file/WEB5df5991e85cc9a826f02fb1399160583?method=download&shareKey=310d06c2ef87f2b0f54cbdb53e79b2a5)




