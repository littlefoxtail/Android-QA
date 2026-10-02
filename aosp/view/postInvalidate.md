# postInvalidate

时序图：
![postInvalidate](/img/postinvalidate.png)

postInvalidate是在非UI线程中调用，invalidate是在UI线程中调用

View@postInvalidate

```java
public void postInvalidate() {
   postInvalidateDelayed(0);
}
```

```java
public void postInvalidateDelayed(int delayMilliseconds) {
   final AttachInfo attachInfo = mAttachInfo;
   if (attachInfo != null) {
       attachInfo.mViewRootImpl.dispatchInvalidateDelayed(this, delayMilliseconds);
   }
}
```

ViewRootImpl@dispatchInvalidateDelayed

```java
public void dispatchInvalidateDelayed(View view, long delayMilliseconds) {
    Message msg = mHandler.obtainMessage(MSG_INVALIDATE, view);
    mHandler.sendMessageDelayed(msg, delayMilliseconds);
}
```

```java
final class ViewRootHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
       switch(msg.what) {
            case MSG_INVALIDATE:
               ((View)msg.obj).invalidate();
       }
    }
}
```

一般来说，如果View确定自身不再适合当前区域，比如说它的LayoutParams发生了改变，需要父布局对其进行重新测量、布局、绘制这三个流程，往往使用requestLayout。而invalidate则是刷新当前View，使当前View进行重绘，不会进行测量、布局流程，因此如果View只需要重绘而不需要测量，布局的时候，使用invalidate方法往往比requestLayout方法更高效
