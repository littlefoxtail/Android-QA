# Compose所解决的问题
关注分离点（Separation of concerns，SOC)是一个众所周知的软件设计原则。
处理紧耦合的模块，对一个地方的代码改动，意味着对其他模块作出许多其他改动。更糟的是，耦合常常是隐式的，以及看起来毫无关联的修改，却会造成了意料之外的错误发生。
关注点分离尽可能的将相关的代码组织在一起，以便可以轻松地维护它们，并方便我们随着应用规模的增长而扩展我们的代码。
## 实例
视图模型（view model）和XML布局为例：
![[Pasted image 20220522195602.png]]
视图模型会向布局提供数据。事实证明，这里隐藏了许多依赖关系：试图模型与布局间存在许多耦合。

```kotlin
public fun ComponentActivity.setContent(  
    parent: CompositionContext? = null,  
    content: @Composable () -> Unit  
) {  
	//1. 通过decorView找到ContentView，再获取第一个ComposeView
    val existingComposeView = window.decorView  
        .findViewById<ViewGroup>(android.R.id.content)  
        .getChildAt(0) as? ComposeView  
	//2. 如果ComposeView为空，则走初始化流程
    if (existingComposeView != null) with(existingComposeView) {
        setParentCompositionContext(parent)  
        setContent(content)  
    } else ComposeView(this).apply {  
        // Set content and parent **before** setContentView  
        // to have ComposeView create the composition on attach
		//3. 初始化ComposeView肯定为空，则进入这边
		setParentCompositionContext(parent)  
        setContent(content) 
        // Set the view tree owners before setting the content view so that the inflation process  
        // and attach listeners will see them already present        setOwners()  
		//5. 把ComposeView设置进ContentView
        setContentView(this, DefaultActivityContentLayoutParams)  
    }  
}
```

当前窗口的decorView中android.R.id.content中的第0个位置，会存在一个ComposeView，所以结构图可以理解为：
![[Pasted image 20220521171902.png]]
1. setContent存储入口函数对象，等待拿走。
2. setContentView，添加到android.R.id.content中
# 附加到Window
```kotlin
//AbstractComposeView
override fun onAttachedToWindow() {  
    super.onAttachedToWindow()  
  
    previousAttachedWindowToken = windowToken  
  //ComposeView修改后预备状态
    if (shouldCreateCompositionOnAttachedToWindow) {  
		//真正启动的地方
        ensureCompositionCreated()  
    }  
}
```

```kotlin
//AbstractComposeView
@Suppress("DEPRECATION") // Still using ViewGroup.setContent for now  
private fun ensureCompositionCreated() {
	//composition 初始化流程为空
    if (composition == null) {  
        try {  
            creatingComposition = true  
			//又来一个setContent?
            composition = setContent(resolveParentCompositionContext()) {
			//抽象方法
                Content()  
            }  
        } finally {  
            creatingComposition = false  
        }  
    }  
}
```


```java
//AbstractComposeView
private fun resolveParentCompositionContext() = parentContext  
    ?: findViewTreeCompositionContext()?.cacheIfAlive()  
    ?: cachedViewTreeCompositionContext?.get()?.takeIf { it.isAlive }  
    ?: windowRecomposer.cacheIfAlive()
```
确定正确的CompositionContext来作为这个视图的父级组成。这可能会导致缓存 一个查找的，以便以后使用。
都没找到会向windowRecomposer求助，可以lazily创建一个recomposer

```kotlin
//windowRecomposer
internal val View.windowRecomposer: Recomposer  
    get() {  
        check(isAttachedToWindow) {  
            "Cannot locate windowRecomposer; View $this is not attached to a window"        }  
			//扩展函数拿到contenChild
        val rootView = contentChild  
        return when (val rootParentRef = rootView.compositionContext) {  
		//初始化流程为null
            null -> WindowRecomposerPolicy.createAndInstallWindowRecomposer(rootView)  
            is Recomposer -> rootParentRef  
            else -> error("root viewTreeParentCompositionContext is not a Recomposer")  
        }  
    }
```

```kotlin
//WindowRecomposerPolicy
internal fun createAndInstallWindowRecomposer(rootView: View): Recomposer {  
//1. get 创建 WindowRecomposerFactory
//2. createRecomposer 调用WindowRecomposerFactory.createRecomposer()
    val newRecomposer = factory.get().createRecomposer(rootView)  
//3. compositionContext 赋值到ComposeView Tag数组
    rootView.compositionContext = newRecomposer  
  
    
    return newRecomposer  
}
```
## AbstractComposeView.setContent
```kotlin
internal fun AbstractComposeView.setContent(  
    parent: CompositionContext,  
    content: @Composable () -> Unit  
): Composition {  
    GlobalSnapshotManager.ensureStarted()  
//获取AndroidComposeView
    val composeView =  
        if (childCount > 0) {  
            getChildAt(0) as? AndroidComposeView  
        } else {  
            removeAllViews(); null  
        } ?: AndroidComposeView(context).also {
		//添加到ViewGroup（ComposeView）****
			addView(it.view, DefaultLayoutParams) }  
    return doSetContent(composeView, parent, content)  
}
```
![[Pasted image 20220522135138.png]]
```kotlin
private fun doSetContent(  
    owner: AndroidComposeView,  
    parent: CompositionContext,  
    content: @Composable () -> Unit  //入口函数对象
): Composition {  
//创建Composition
    val original = Composition(UiApplier(owner.root), parent)  
//常见WrappedComposition并赋值到AndroidComposeView的Tag中
    val wrapped = owner.view.getTag(R.id.wrapped_composition_tag)  
        as? WrappedComposition  
        ?: WrappedComposition(owner, original).also {  
            owner.view.setTag(R.id.wrapped_composition_tag, it)  
        }  

    wrapped.setContent(content)  
    return wrapped  
}
```
### UiApplier
LayoutNode：布局层次中的一个元素，用compose UI构建。
*AndroidComposeView.root*创建
```kotlin
//AndroidComposeView
override val root = LayoutNode().also {  
    it.measurePolicy = RootMeasurePolicy  
    // Composed modifiers cannot be added here directly  
    it.modifier = Modifier  
        .then(semanticsModifier)  
        .then(rotaryInputModifier)  
        .then(_focusManager.modifier)  
        .then(keyInputModifier)  
    it.density = density  
}
```
是一个视图工具类，让其他使用者更方便的操作*root: LayoutNode*。内部维护了一个*stack = mutableListOf*代表视图树。

### composition
UiApplier是Composition的入参，哪个对象来操作UiApplier？
CompositionImpl：一个组合对象通常是为你构建，并从一个API中返回，用于最初的UI组合
```kotlin
//CompositionImpl
//保存所有试图组合的信息（核心数据结构）
internal val slotTable = SlotTable()

private val composer: ComposerImpl =  
    ComposerImpl(  
        applier = applier,  
        parentContext = parent,  
        slotTable = slotTable,  
        abandonSet = abandonSet,  
        changes = changes,  
        lateChanges = lateChanges,  
        composition = this  
    ).also {  
        parent.registerComposer(it)  
    }
```
Composition是一个连接器，把视图树（UiAapplier）和调度程序（Recomposer）连接到了一起，当监听到数据（mutableState）变化或者添加视图等。
### WrappedComposition.setContent
```kotlin
override fun setContent(content: @Composable () -> Unit) { 
//AndroidComposeView生命周期监听
    owner.setOnViewTreeOwnersAvailable {  
        if (!disposed) {  
            val lifecycle = it.lifecycleOwner.lifecycle  
            lastContent = content  
            if (addedToLifecycle == null) {  
//1. 初始化流程第一次会进入
                addedToLifecycle = lifecycle  
                // this will call ON_CREATE synchronously if we already created  
//2. 设置生命周期监听
                lifecycle.addObserver(this)  
            } else if (lifecycle.currentState.isAtLeast(Lifecycle.State.CREATED)) {  **
			//触发生命周期Lifecycle.Event.ON_CREATE
//Composition的setContent()->composeContent()
//调用连接器的setContent
                original.setContent {  
  
                    @Suppress("UNCHECKED_CAST")  
                    val inspectionTable =  
                        owner.getTag(R.id.inspection_slot_table_set) as?  
                            MutableSet<CompositionData>  
                            ?: (owner.parent as? View)?.getTag(R.id.inspection_slot_table_set)  
                                as? MutableSet<CompositionData>  
                    if (inspectionTable != null) {  
                        inspectionTable.add(currentComposer.compositionData)  
                        currentComposer.collectParameterInformation()  
                    }  
  
                    LaunchedEffect(owner) { owner.keyboardVisibilityEventLoop() }  
                    LaunchedEffect(owner) { owner.boundsUpdatesEventLoop() }  
//入口函数对象
                    CompositionLocalProvider(LocalInspectionTables provides inspectionTable) {  
                        ProvideAndroidCompositionLocals(owner, content)  
                    }  
                }            }  
        }  
    }  
}
```