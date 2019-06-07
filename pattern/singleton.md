# 单例模式（Singleton）

* 意图：保证一个类仅有一个实例，并提供一个访问它的全局访问点。
* 适用性：
    1. 当类只能有一个实例而且客户可以从一个众所周知的访问点访问它时。
    2. 当这个唯一的实例应该是通过子类化可扩展的，并且客户应该无需更改代码就能使用一个扩展的实例时。

## （单例模式）优点

* 单例模式提供了对唯一实例的受控访问。因为单例模式封装了它的唯一实例，所以它可以严格控制客户这样以及何时访问它
* 节省系统资源，对一些频繁创建和销毁的对象单例模式无疑可以提高系统的性能
* 允许可变数目的实例。基于单例模式可以进行扩展，使用与单例控制相似的方法来获得指定个数的对象实例

## （单例模式）缺点

* 由于单例模式中没有抽象层，因此单例类的扩展有很大的困难
* 单例类的职责过重，在一定程度上违背了单一职责。因为单例类既充当了工厂角色，提供了工厂方法，同时又充当了产品角色，包含一些业务方法，将产品的创建和产品的本身的功能融合到一起