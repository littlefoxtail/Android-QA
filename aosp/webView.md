# WebView

[链接](http://motalks.cn/2016/09/11/Android-WebView-JavaScript-3/)  
Android WebView性能优化的常用方法

## 页面加载速度优化

### 选择合适的WebView缓存

H5常用的缓存机制的优势及使用场景

|缓存机制  |   优势   |  使用场景    |
| :----: | :----: | :----: |
| Dom Storage|HTTP协议层支持|静态文件的缓存|
| Web SQL Database|存储、管理复杂结构数据|用IndexedDB替代，不推荐使用|
| AppCache|方便构建离线App|离线App、静态文件缓存，不推荐使用|
| IndexedDB|存储任何类型数据、使用简单，支持索引|结构、关系复杂的数据存储Web SQL Database的替代|
|File System API|支持文件系统的操作|数据适合以文件进行管理的场景,Android系统还不支持|

#### 1.浏览器缓存机制：

主要前端负责，Android端不需要进行特别的配置。

#### 2.Dom Storage (Web Storage)存储机制：

配合前端使用，使用时需要打开DomStorage开关。

```java
WebView myWebView = (WebView) findViewById(R.id.webview);
WebSettings webSettings = myWebView.getSettings();
webSettings.setDomStorageEnabled(true);
```

#### 3.Application Cache存储机制

Application Cache似乎为了支持Web App离线使用而开发的缓存机制。它的缓存机制类似于浏览器的缓存
(Cache-Control和Last-Modified)机制，都是以文件为单位进行缓存，且文件有一定更新机制。但AppCache
是对浏览器缓存机制的补充，不是替代。

根据官方文档，AppCache已经不推荐使用了，标准也不支持

```java
WebView myWebView = (WebView) findViewById(R.id.webview);
WebSettings webSettings = myWebView.getSettings();
webSettings.setAppCacheEnabled(true);
final String cachePath = getApplicationContext().getDir("cache",Context.MODE_PRIVATE).getPath();
webSettings.setAppCachePath(cachePath);
webSettings.setAppCacheMaxSize(5*1024*1024);
```

#### Indexed Database 存储机制

IndexedDB也是一种数据库的存储机制，但不同于已经不再支持的Web SQL Database。IndexedDB
不是传统的关系数据库，可归为NoSQL数据库。IndexedDB又类似于Dom Storage的key-value的存储方式，但功能更强大，切存储空间更大。

Android在4.4开始加入对IndexedDB的支持，只需打开允许JS执行的开关就好

```java
WebView myWebView = (WebView) findViewById(R.id.webview);
WebSettings webSettings = myWebView.getSettings();
webSettings.setJavaScriptEnabled(true);
```

### 常用资源预加载

缓存技术，能优化二次启动WebView的加载速度，首次加载H5页面的速度呢？

1. 提前将这些资源下载好，等H5 加载时直接替换
2. 常用 JS 本地化及延迟加载

关于JS延迟加载
> Android 的 OnPageFinished 事件会在 Javascript 脚本执行完成之后才会触发。如果在页面中使 用JQuery，会在处理完 DOM 对象，执行完 $(document).ready(function() {}); 事件自会后才会渲染并显示页面。而同样的页面在 iPhone 上却是载入相当的快，因为 iPhone 是显示完页面才会触发脚本的执行。所以我们这边的解决方案延迟 JS 脚本的载入，这个方面的问题是需要Web前端工程师帮忙优化的。

### 4.使用第三方WebView内核

WebView 的兼容性一直也是困扰我们 Android 开发者的一个大问题，不说 Android 4.4 版本 Google 使用了Chromium 替代 Webkit 作为 WebView 内核，就看看国内众多的第三方 ROM 都有可能会对原生的 WebView 做出修改，这时候如果出现兼容问题，是非常难定位到问题和解决的

### 5.WebView导致的内存泄露

> Android 中的 WebView 存在很大的兼容性问题，不仅仅是 Android 系统版本的不同对 WebView 产生很大的差异，另外不同的厂商出货的 ROM 里面 WebView 也存在着很大的差异。更严重的是标准的 WebView 存在内存泄露的问题，看这里WebView causes memory leak - leaks the parent Activity。所以通常根治这个问题的办法是为 WebView 开启另外一个进程，通过 AIDL 与主进程进行通信，WebView 所在的进程可以根据业务的需要选择合适的时机进行销毁，从而达到内存的完整释放。

### 6.WebView遇到的一些坑

1. Webview打开一个链接，播放一段音乐，退出Activity时音乐还在后台播放，可以通过在Activity的onPause中调用webview.onPause()解决，并在Activity的onResume中调用webview.onResume()恢复。

2. 5.0以后api调整，设置跨域cookie读取
    ```java
        public final void setAcceptThirdPartyCookies() {
            //target 23 default false, so manual set true
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
                CookieManager.getInstance().setAcceptThirdPartyCookies(webView, true);
            }
        }
    ```

3. 5.0之后不支持Https和Http的混合模式，需要设置
    ```java
    webSetting.setMixedContentMode(webSetting.getMixedContentMode())
    ```

4. WebView与JavaScript相互调用时，如果是debug没有配置混淆时，调用时没问题的，但是当设置混淆后发现无法正常调用了

5. 沉浸式导致webview全屏遮挡虚拟按键，需要在WebChromeClient#onShowCustomView设置

```java
mActivity.getWindow().getDecorView().setSystemUiVisibility(
						View.SYSTEM_UI_FLAG_LAYOUT_STABLE |
								View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION |
								View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN |
								View.SYSTEM_UI_FLAG_HIDE_NAVIGATION |
								View.SYSTEM_UI_FLAG_FULLSCREEN |
								View.SYSTEM_UI_FLAG_IMMERSIVE);
```