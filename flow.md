需要一种方法在这些coroutines之间进行通信，而不会遇到可怕的“可变共享状态”问题。

三部分组成：
生产者：即数据被发送或发射的地方。
中介：发送时修改数据的操作者，如使用map映射，使用过滤器过滤，捕捉异常等等。
消费者：收集或观察数据流并对其作出反应。；
在开始收集之前，它本身没有任何状态。每个收集器的coroutine都会执行它自己的发射代码的实例。返回值按需计算的异步值流。

![[Pasted image 20221106173448.png]]

# StateFlow
StateFlow是一个状态容器式可观察数据流，可以向其收集器发出当前状态更新和新状态更新。还可通过其value属性读取当前状态值。

当新的使用方开始从数据流中收集数据时，它将接收信息流中的最近一个状态和任何后续状态。
和LiveData具有相似之处。两者都是可观察的数据容器类，并且在应用架构中使用时，两者都遵循相似模式。
# SharedFlow
![[Pasted image 20221106174434.png]]
```kotlin
class BroadcastEvents {
	private val _events = MutableSharedFlow<Event>()
	val event = _events.asSharedFlow()
	suspend fun postEvent(event: Event) {
		_events.emit(event) //suspends until subscribers receive it
	}
}
```
和StateFlow一样，也是热流，可以将已发送过的数据发送给新的订阅者，并且具有高的配置性。
如下场景需要使用：
- 发生订阅时，需要将过去已经更新的n个值同步给新的订阅者。
- 配置缓存策略。

它有可调整的参数，为新的订阅者keep和replay的旧事件的数量，以及为快速发射器和慢速订阅者提供缓冲的extraBufferCapacity。
当MutableSharedFlow中缓存数据量超过阈值时，emit方法和tryEmit方法的处理方式会有不同：
- emit方法：当换药策略为BufferOverflow.SUSPEND时候，emit方法会挂起，直到有新的缓存空间。
- tryEmit方法：当tryEmit会返回一个Boolean值，true代表传递成功，false代表会产生一个回调，让这次发射挂起，直到新的缓存空间。



在channel，每个事件被传递给一个订阅者。试图在没有订阅者的情况下发布事件，一旦通道缓冲区变满就会暂停，等待订阅者出现。默认情况下，发布的事件不会被丢弃。
# channel
被添加一个程序间的通信原语。channels support one-to-one，one-to-many，many-to-one，and many-to-many。
不能使用channel来分发事件或状态更新，以及允许多个订阅者独立地接收并对其作出反应。



