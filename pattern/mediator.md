## Mediator(中介者模式)

* 意图: 用一个中介对象来封装一系列的对象交互。中介者使各对象不需要显示地相互引用，从而使耦合松散，而且可以独立地改变它们之间的交互。
* 适用性：
1. 一组对象以定义良好但是复杂的方式进行通信。产生的相互依赖关系结构混乱且难以理解。
2. 一个对象引用其他很多对象并且直接与这些对象通信，导致难以复用该对象。
3. 想定制一个分布在对个类中的行为，而又不想生成太多的子类。
