# 测试支持库

Instrumentation 是Android自带的一个测试框架，是很多框架的基础，可以同进程中加载被测组件。但instrumentation不支持跨应用，导致基于instrumentation的框架都继承了这个缺点

UIAutomator和Espresso也是基于Instrumentation的，更偏向于UI方面的自测化测试

InstrumentationTestRunner：在API24 被AndroidUnitRunner取代

AndroidJunitRunnter：是一个Junit测试运行器，可以让Android设备上运行Junit3或者Junit4样式测试类。测试运行器可以将测试软件包和要测试的应用加载到设备、运行测试并报告测试结果。

## 测试支持库功能

Android测试支持库包括以下自动化测试工具：

- AndroidJunitRunner：适用于Android且与Junit4兼容的测试运行器
- Espresso： UI测试框架；适合应用中的功能性UI测试。（Espresso基于AndroidJUnitRunner）
- UI Automator：UI测试框架；适合跨系统和已安装应用的跨应用功能性UI测试
- robolectric
- Mockito

### mock

单元测试的思路是在不涉及依赖关系的情况下测试代码，所以测试与其他类或者系统的关系应该尽量被消除。一个可行的消除方法是替换掉依赖类。

#### Mock对象的产生

你可以手动创建一个Mock对象或者使用Mock框架来模拟这些类，Mock框架允许你在运行时创建Mock对象并且定义它的行为。

典型的例子就是把Mock对象模拟成数据的提供者。在正式生产环境中它会被实现用来连接数据源

Mock对象可以被提供来进行测试。因此，我们的测试的类应该避免任何外部数据的强依赖

通过Mock对象或者Mock框架，可以测试代码中期望的行为

#### 使用Mockito生成Mock对象

Mockito是一个流行mock框架，可以和JUnit结合起来使用。Mockito允许你创建和配置mock对象。使用mockito可以明显简化对外部依赖的测试类的开发。

一般Mockito需要执行下面三步：

- 模拟并替换测试代码中外部依赖
- 执行测试代码
- 验证测试代码是否被正确执行

![mockito_step](/img/mockito_step)

#### 使用Mockito API

##### 静态引用

如果在代码中静态引用`org.mockito.Mockito.*;`，就可以直接调用静态的方法和变量

##### 使用Mockito创建和配置mock对象

除了使用mock()静态方法外，Mockito还支持通过`@Mock`注解的方式来创建mock对象。

如果使用注解，那么必须实例化mock对象。Mockito在遇到使用注解的字段的时候，会调用`MockitoAnnotations.initMocks(this)`来初始化该mock对象。另外也可以通过使用`@RunWith(MockitoJnitRunner.class)`来达到相同的效果。

例子：

```java
public class MockitoTest {
    @Mock
    MyDatabase databaseMock;(1)

    @Rule
    public MockitRule mockitoRule = MockitoJunit.rule();(2)

    @Test
    public void testQuery() {
        ClassToTest t  = new ClassToTest(databaseMock); (3)
        boolean check = t.query("* from t"); (4)
        assertTrue(check); (5)
        verify(databaseMock).query("* from t"); (6)
    }


}
```

1. 告诉Mockito模拟databaseMock实例
2. Mockito通过`@mock`注解创建mock对象
3. 使用以及创建的mock初始化这个类
4. 在测试环境下，执行测试类的代码
5. 使用断言确保调用的方法返回值为true
6. 验证query方法是否被`MyDatabase`的mock对象调用

##### 配置mock

当需要配置某个方法的返回值的时候，Mockito提供了链式API供调用

`when(..).thenReturn(..)`可以被用来定义当条件满足时函数的返回值，如果需要定义多个返回值，可以多次定义。当你多次调用函数的时候，Mockito会根据你定义的先后顺序来返回返回值。Mockito还可以根据传入参数的不同的定义不同的返回值。

```java
@Test
public void test1() {
    // 创建mock
    MyClass test = Mockito.mock(MyClass.class);
    // 自定义getUniqueId()返回值
    when(test.getUniqueId()).thenReturn(43);
    // 在测试中使用mock对象
    assertEquals(tes.getUniqueId(), 43);
}
// 返回多个值

@Test
public void testMoreThanOneReturnValue() {
    Iterator i= mock(Iterator.class);
    when(i.next()).thenReturn("Mockito").thenReturn("rocks");
    String result=i.next()+" "+i.next();
    // 断言
    assertEquals("Mockito rocks", result);
}
```

