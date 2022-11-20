# 为什么需要一个clean architecture 
编码是一门艺术。一个应用程序并不意味着它可以工作和满足ui/ux要求。它还需要易于理解、灵活、可测试、可扩展和可维护。
一个需要遵循的模版。
三个主要层次：
1. Domain：执行于独立于任何层的业务逻辑，它只是一个纯kotlin包，没有安卓任何特定依赖。它包括Model、Use case等
2. Data：通过实现域层暴露的接口，为应用程序提供所需的数据。它包括远程/本地数据源、api实现等。
3. Presenter/Application：包含执行用户界面逻辑的android特定代码。它包括view、fragment等
# TLDR
- Domain Layer不能访问任何Layer，Data layer只能访问Domain Layer，而App layer可以访问这两层。
- UseCase用于接受请求并将数据传递给主要在Persenter/App layers层的请求。UserCase是在Domain Layer。
- Domain Layer只有Kotlin/Java特定的代码。没有框架相关的（Android）的代码。
- repository的接口在Domain Layer声明，在Data layer中实现。
- UserCase使用Repository作为构造器注入，Repository使用本地/远程数据源作为构造器注入。
- Data flow：
	1. 来自App/Persentation layer的请求，通过UseCase(Domain layer) 
	2. UseCase将通过repository联系到Data layer。
	3. Data layer将通过repository返回数据给UseCase。 
	5. UseCase将把数据返回给Presentation/App layers，UseCase属于Domain layer

# Domain Layer
Domain layer由modell、repository和useCase。
UseCase负责从本地/远程数据源获取数据，并将数据反馈给请求。
```kotlin
class GetPopularMoviesUseCase(private val movieRepository: MovieRepository) {

suspend operator fun invoke()=movieRepository.getPopularMovies()

}
```
MovieRepository是一个接口，它将与数据层联系并获得数据。Movie Repository接口的实现将在Data layer中实现。
# Data Layer
数据层由api和repository组成。不要把repository和data layer搞混。
MovieRepositoryImpl是领域层的资源库接口的实现。这个资源库包用来实现本地/远程数据源的。
# App/Presentation Layer
这一层由Android UI专用代码组成。
这个包由依赖性注入相关的代码组成