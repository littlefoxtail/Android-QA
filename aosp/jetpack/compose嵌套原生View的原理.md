#compose 
compose是通过自己的一套重组算法来构建界面，测量和布局已经脱离了原生View体系 。
但是可以嵌套，咋弄的？
![[Pasted image 20220524000131.png]]
# 源码分析
```kotlin
@Composable  
@UiComposable  
fun <T : View> AndroidView(  
    factory: (Context) -> T,  
    modifier: Modifier = Modifier,  
    update: (T) -> Unit = NoOpUpdate  
) {  
   ...
//1. 创建ViewFactoryHolder
    ComposeNode<LayoutNode, UiApplier>(  
        factory = {  
            val viewFactoryHolder = ViewFactoryHolder<T>(context, parentReference, dispatcher)  
//2. factory 被赋值给了ViewFactoryHolder
            viewFactoryHolder.factory = factory  
//3. 从ViewFactoryHolder拿到LayoutNode
			viewFactoryHolder.layoutNode
		},..
	)

}
```
ComposeNode函数中拿到factory的返回值LayoutNode来创建一个Node节点来参与Compose的绘制。

## 赋值给ViewFactoryHolder
```kotlin
var factory: ((Context) -> T)? = null  
    set(value) { 
//1. 将factory 赋值给幕后字段
        field = value  
//2. factory不为空
        if (value != null) {  
//3. invoke factory函数，拿到原生View本身
            typedView = value(context)  
//4. 将原生View复制给View，这个属性在ViewFactoryHolder的父类AndroidViewHolder
            view = typedView  
        }  
    }
```
## 分析AndroidViewHolder
```kotlin
//1. 是一个继承自ViewGroup的原声组建
internal abstract class AndroidViewHolder(  
    context: Context,  
    parentContext: CompositionContext?,  
    private val dispatcher: NestedScrollDispatcher  
) : ViewGroup(context), NestedScrollingParent3 {

var view: View? = null  
    internal set(value) {  
        if (value !== field) {  
//2. 将view赋值给幕后字段
            field = value  
//3. 移除
            removeAllViews()  
            if (value != null) {  
//4.添加
                addView(value)  
//5.触发更新
                runUpdate()  
            }        }  
    }
```
## 分析ViewFactoryHolder.layoutNode
```kotlin
val layoutNode: LayoutNode = run {  
    // Prepare layout node that proxies measure and layout passes to the View.  
//1.准备好布局节点，代理测量和布局传递给视图
    val layoutNode = LayoutNode()  
  
    val coreModifier = Modifier  
        .pointerInteropFilter(this)  
        .drawBehind {  
            drawIntoCanvas { canvas ->  
                (layoutNode.owner as? AndroidComposeView)  
                    ?.drawAndroidView(this@AndroidViewHolder, canvas.nativeCanvas)  
            }  
        }.onGloballyPositioned {  
            // The global position of this LayoutNode can change with it being replaced. For  
            // these cases, we need to inform the View.            layoutAccordingTo(layoutNode)  
        }  
    layoutNode.modifier = modifier.then(coreModifier)  
    onModifierChanged = { layoutNode.modifier = it.then(coreModifier) }  
  
    layoutNode.density = density  
    onDensityChanged = { layoutNode.density = it }  
  
    var viewRemovedOnDetach: View? = null  
//2. 注册attach回调
    layoutNode.onAttach = { owner ->  
		// 2.1重点：将当前ViewGroup添加到AndroidComposeView中
        (owner as? AndroidComposeView)?.addAndroidView(this, layoutNode)  
        if (viewRemovedOnDetach != null) view = viewRemovedOnDetach  
    }
//3. 注册detach回调
    layoutNode.onDetach = { owner ->  
		//3.1 将当前ViewGroup从AndroidComposeView中移除
        (owner as? AndroidComposeView)?.removeAndroidView(this)  
        viewRemovedOnDetach = view  
        view = null  
    }  
//4. 注册measurePolicy 绘制策略回调
    layoutNode.measurePolicy = object : MeasurePolicy {  
        override fun MeasureScope.measure(  
            measurables: List<Measurable>,  
            constraints: Constraints  
        ): MeasureResult {  
            if (constraints.minWidth != 0) {  
                getChildAt(0).minimumWidth = constraints.minWidth  
            }  
            if (constraints.minHeight != 0) {  
                getChildAt(0).minimumHeight = constraints.minHeight  
            }  
			//4.1 layoutNode的测量，触发AndroidViewHolder的测量
            measure(  
                obtainMeasureSpec(  
                    constraints.minWidth,  
                    constraints.maxWidth,  
                    layoutParams!!.width  
                ),  
                obtainMeasureSpec(  
                    constraints.minHeight,  
                    constraints.maxHeight,  
                    layoutParams!!.height  
                )  
            )
			//4.1 layoutNode的布局，触发AndroidViewHolder的布局
            return layout(measuredWidth, measuredHeight) {  
                layoutAccordingTo(layoutNode)  
            }  
        }  
  
        //5. 返回 layoutNode
    }  
    layoutNode  
}

```
### 2.1 addAndroidView
```kotlin
internal val androidViewsHandler: AndroidViewsHandler  
    get() {  
        if (_androidViewsHandler == null) {  
            _androidViewsHandler = AndroidViewsHandler(context)
//1. 将AndroidViewHandler 添加到AndroidComposeView中
            addView(_androidViewsHandler)  
        }  
        return _androidViewsHandler!!  
    }

fun addAndroidView(view: AndroidViewHolder, layoutNode: LayoutNode) {  
    androidViewsHandler.holderToLayoutNode[view] = layoutNode
//1. 将 AndroidViewHolder 添加到 AndroidViewsHandler 中
    androidViewsHandler.addView(view)  
    androidViewsHandler.layoutNodeToHolder[layoutNode] = view  
    // Fetching AccessibilityNodeInfo from a View which is not set to  
    // IMPORTANT_FOR_ACCESSIBILITY_YES will return null.    ViewCompat.setImportantForAccessibility(  
        view,  
        ViewCompat.IMPORTANT_FOR_ACCESSIBILITY_YES  
    )  
    ...  
}
```
# 总结
最终结构为：
![[Pasted image 20220528002342.png]]
- compose是通过addView的方式，将原生View添加到AndroidComposeView中的，他依然使用的是原生布局体系。
- 嵌套原生View的测量与布局，是通过创建个人代理LayoutNode，然后添加到Compose中参与组合，并将每次重组返回的测量信息设置到原生View上，以此来改变原生View的位置与大小。