
Material Theme是一种用户界面样式，它确定视图和活动开头的外观和感觉。材料主题是内置到Android 5.0，是一种内置基本样式。

Android提供三个材料主题风格：
- `Theme.Material`-深色版本的材料主题，默认风格
- `Theme.Material.Light`-材料主题的轻量版本
- `Theme.Material.Light.DarkActionBar`-轻量版本的材料主题，但是深色操作栏
![three-flavors-sml](../img/three-flavors-sml.png)

# 使用自定义主题
|属性|含义|
|:---:|:---:|
|colorPrimary|应用的主色调，actionBar默认使用该颜色，Toolbar导航栏颜色|
|colorPrimaryDark|应用的主要暗色调，statusBarColr默认使用该颜色|
|statusBarColor|状态栏颜色，默认使用colorPrimaryDark|
|windowBackground|窗口背景颜色|
|navigationBarColor|底部栏颜色|
|colorForeground|应用的前景色，ListView的分割线，switch滑动默认使用该颜色|
|colorBackground|应用的背景色，popMenu的背景默认使用该颜色|
|colorAccent|CheckBox，RadioButton，SwitchCompat等一般控件的选中效果默认采用该颜色|
|colorControlNormal|CheckBox，RadioButton，SwitchCompat等默认状态的颜色|
|colorControlHighlight|控件按压时的色调|
|colorControlActivated|控件选中时的颜色，默认使用colorAccent|
|colorButtonNormal|默认按钮的背景颜色|
|editTextColor |默认EditView输入框字体的颜色|
|textColor|Button，textView的文字颜色|
|textColorPrimaryDisableOnly|RadioButton checkbox等控件的文字|
|textColorPrimary|应用的主要文字颜色，actionBar的标题文字默认使用该颜色|
|colorSwitchThumbNormal|switch thumbs 默认状态的颜色. (switch off|

![color_theme](../img/color_theme.png)
![screen-attributes-sml](../img/screen-attributes-sml.png)


# 兼容性
在5.0以上使用Material主题，向下兼容旧的版本

- 在**values-v21/styles.xml**
```xml
<resources>
    <style name="MyCustomTheme" parent="android:Theme.Material.Light">
        <!-- Your customizations go here -->
    </style>
</resources>
```
- 在**values/styles.xml**，派生一个较旧的主题，但要使用相同的主题名称
```xml
<resources>
    <style name="MyCustomTheme" parent="android:Theme.Holo.Light">
        <!-- Your customizations go here -->
    </style>
</resources>
```

- 在AndroidManifest.xml，使用自定义主题配置应用程序
    - <application>节点
    - <activity>节点

