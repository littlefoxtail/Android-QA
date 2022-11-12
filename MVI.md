MVI与MVVM相似，借鉴了前端框架的思想，更加强调数据的单向流动和唯一数据源。
![[Pasted image 20220521163804.png]]
1. Model：与MVVM中的Model不同的是，MVI的Model主要指UI状态（State）。例如页面加载状态、控件位置等都是一种UI状态。
2. View：与其他MVX中的View一致，可能是一个Activity或者任意UI承载单元。MVI中的View通过订阅Model的变化实现界面刷新。
3. Intent：此Intent不是Activity的Intent，用户的任何操作都被包装成Intent后发送给Model层进行数据请求。

Model层的业务结果反映为试图状态+事件。
可以吸收MVVM中的优点采用如下设计:
- 将双向绑定退化为单向绑定，View层消费UI状态流和事件流，这也意味着UI状态指责精简，它不再承载View层的用户输入等事件
- 将UI状态独立，Model层仅产生UI状态的局部变化和事件。

# 单向数据流动（Unidirectional Data Flow）
> 单向数据流（UDF）是一种设计模式，在该模式下状态向下流动，事件向上流动。通过采用单向数据流，可以在界面中显示状态可组合项与应用中存储和更改状态的部分分离开来。
![[Pasted image 20220917155159.png]]
起始于Intent，经过分类和选择性消费后产生，对应的resucer函数计算后，得到最新的State，驱动视图。
注意：
- 单向是指单一方向
- 此处的数据是广义的、宽泛的
- 仅描述数据流的变化方向，与数据流的数量无关，但一般形成有效工作均需要两条数据流（上行数据流和下行数据流）


在经典设计中，其内涵如下图：
- 按照试图的所有的内容状态，定义一个不可变的ViewState
- 按照业务初始化ViewState实例
- Model业务生成驱动ViewState变化的Result
- 计算出新状态，Reduce(Pre-ViewState,Result)->New-ViewState
- 更新数据源
- View层消费ViewState

在单向数据流动章节中，提到了MVI的UDF设计：
- 系统捕获的UI事件、其他侦听事件（例如熄屏、应用生命周期事件），生成Intent，压入Intent流中
- ViewModel层中筛选、转换、处理Intent，实际是使用Model层业务，产生业务结果，即PartialChange
- PartialChange经过Reducer计算处理得到最新的ViewState，压入ViewState流
- View层（广义的表现层）响应并呈现最新的ViewState
## MVC/MVP中的痛点
在MVC和MVP中，着眼于控制、调度，并不强调数据流的概念。
View和Model间的交互，一般有两种编码风格：双向的api调用、单向的api调用+回调。
这两种方式无法继续抽象，需根据实际业务进行命令式编码。当UI复杂时，难以写出清晰、易读的代码，维护难度激增。
## MVVM解决UI更新代码混乱问题
mvvm中通过绑定框架，讲ui事件转化为数据变化，驱动业务；业务结果表现为数据变化，驱动ui更新。
显而易见，维护朴素的数据要比直接维护复杂的ui要简单。
![[Pasted image 20220917165725.png]]
但问题也同时产生，data1变化的两个可能的原因：
- Model层业务结果使其变化，并期望它驱动更新
- View层发生事件，反馈数据变化，并期望它驱动Model层逻辑
因此，框架需要考虑标识数据变化来源、或者其他手段消除方向性所带来的问题。
并且mvvm难以灵活决定的“何时调用Model层逻辑”，即大多数业务中，都需要结合多个属性的变化形成组合条件来驱动Model层逻辑。
## 用Intent灵活决定何时调用Model
在于MVC/MVP以及MVVM对比后不难得出结论：
- mvc/mvp，view层通过调用controll/presenter层api的方式最终调用到model层业务，方式质朴、无难度。但业务量规模增大后接口方法树也会增多，导致contorll/presenter层尾大不掉，难以复用。
- mvvm中，viewModel层总是需要利用技巧进行模型概念转换，以满足业务响应实际需求
mvi在架构角色中设计了Intent的角色：
- 它包含了业务调用的意图和数据
- 从设计上可满足调用与实现的分离
- 架构模型以Intent流形式出现，下游对其筛选、转换、消费等行为可遵循fp范式（Functional Programmng Patterns），逻辑的复用粒度为方法级，复用度更高更灵活
- 解决了MVVM中方向性问题，mvc/mvp中灵活度问题等
## 单一数据源
单一可信数据源：
- 单一
- 可信
- 数据源
在mvp背景下，数据源指的是试图对应的数据实体，它代表是视图的内容状态。
可信指从数据源中获取的数据是最新的、完整的、可靠的，否则是不可信的。
# 代码结构
一些接口
```kotlin
interface UiState
interface UiEvent
interface UiEffect
```
> side effect。
> 在计算机科学中，函数的副作用指当调用函数时，除了返回可能的函数值之外，还对主调用函数产生附加的影响。例如修改全局变量，修改参数，向主调方的终端、管道输出字符或改变外部存储信息等。
> 命令式编程常常会产生副作用，声明式编程通常用来报告系统的状态，没有副作用。
- UiState 是views的状态
为了处理uiState，使用StateFlow。StateFlow就是像LiveData，但有初始值。所以我们总有一个状态。它也是SharedFlow的一种。当用户界面变得可见时，总是希望收到最后的试图状态。
- UiEvent 是用户的行为
为了处理UiEvent，我们使用SharedFlow。如果没有任何订阅者，我们想放弃事件。
- UiEffect 副作用，希望只显示一次
为了处理UiEffect，使用Channels。因为通道是热的，不需要在方向改变或用户界面再次变得可见时再次显示副作用。简单地说，想复制SingleLiveEvent

