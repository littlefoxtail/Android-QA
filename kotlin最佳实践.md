> Any fool can write code that a computer can understand. Good programmers write code that humans can understand. — Martin Fowler

对于命令式编程来说，for循环是一个很好的结构。然而有一个函数可以为你完成这项工作，那么使用该函数是一个更好的做法。
```kotlin
fun main() {
	repeat(10) {
		println(it)
	}
}
```

# scope functions
## let scope
```kotlin
fun main() {
	val age = readLine()?.toIntOrNull()
	age?.let {
		println("You are $it years old");
		
	} ?: print("Wrong")
}
```
let函数是用来绕过可能的空类型。
## apply scope
当你希望改变对象的属性或行为时，可以使用这个函数。
```kotlin
fun main() {
	val me = Person().apply {
		name = "Nishant"
		age = 19
		gender = 'M'
	}
	println(me)
}
```
## with scope 
当你想使用一个对象的属性时，这个函数就会被使用。它只是apply函数的语法糖。
```kotlin
fun main() {
	val me = with(Person()) {
		name = "Nishant"
		age = 19
		gender = 'M'
	}
	println(me)
}
```

## run scope
这个函数类似于let函数，但这里的对象引用作为this，而不是it传递下来。
```kotlin
fun main() {
	val me = Person(
		name = "Nishant"
		age = 19,
		gender = 'M'
	)
	me.run {
		println("My name is $name and i am $age years old")
	}
}
```