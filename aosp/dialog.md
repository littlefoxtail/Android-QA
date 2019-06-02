# 对话框

![dialog](/img/dialog.png)

Dialog类是对话框的基类，当应该避免直接实例化Dialog
而是使用以下子类

## AlertDialog 此对话框可显示标题，最多三个按钮、可选择项列表或自定义布局

## DatePickerDialog或TimePickerDialog

此对话框带有允许用户选择日期或时间的预定义UI

这些定义对话框样式和结构，应该将DialogFragment用作对话框的容器

## Dialog

```java
public class Dialog {
    Dialog(..) {
        mWindowManager = 获取windowManager;
        //这里用的是Activity的context，后面ViewRootImpl.setView会判断token，不是activity就报错
        mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
        final Window w = new PhoneWindow(mContext);
    }

    public void show() {
        if (!mCreated) {
            dispatchOnCreate(null);//会执行到实现类
        }
        onStart();//主要设置ActionBar
        mDecor = mWindow.getDecorView();
        mWindowManager.addView(mDecor, l);

        sendShowMessage();
    }
}
```

```java
public class AlertDialog {
    void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mAlert.installContent();
    }
}
```

```java
public class AlertController {
    public void installContent() {
        mWindow.setContentView(contentView);
        setupView(); //初始化布局
    }
}
```