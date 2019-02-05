

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [清单文件](#清单文件)
	* [何时被解析的](#何时被解析的)
	* [apk安装](#apk安装)
		* [PackageManagerService](#packagemanagerservice)

<!-- /code_chunk_output -->

# 清单文件

## 何时被解析的

- apk安装过程

- Android系统启动之后：
  - 在一个应用中打开另一个从未打开过的应用
  - 在一个应用中发送广播，如果另一个应用设置了这个广播的接收器，那么这个应用进程就会被启动并接收该广播并作出相应的处理

猜测：

- android系统在启动的时候会抓取系统中所有安装的应用信息并保存

## apk安装

```java
File apkFile;
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
intent.setDataAndType(Uri.fromFile(apkFile), "application/vnd.android.package-archive");
context.startActivity(intent);
```

AMS在处理此隐示意图，最终找到：

```text
-InstallStart.java
    -PackageInstallerActivity.java
```

安装弹窗：

```java
public PackageInstallerActivity {
    private void bindUi() {
        mOk = (Button) findViewById(R.id.ok_button);
        mCancel = (Button) findViewById(R.id.cancel_button);
        mOk.setOnClickListener(this);
        mCancel.setOnClickListener(this);
    }

    private void startInstall() {
        Intent newIntent = new Intent();
        newIntent.setData(mPackageURI);
        newIntent.setClass(this, InstallInstalling.class);
        ...
        startActivity(newIntent);

    }
}
```

```java
public class InstallInstalling {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        mSessionId = getPackageManager().getPackageInstaller().createSession(params);
    }

    protected void onResume() {
        super.onResume();

        if (mInstallingTask == null) {
            PackageInstaller installer = getPackageManager().getPackageInstaller();
            PackageInstaller.SessionInfo sessionInfo = installer.getSessionInfo(mSessionId);

            if (sessionInfo != null && !sessionInfo.isActive()) {
                mInstallingTask = new InstallingAsyncTask();
                mInstallingTask.execute();
            }
        }
    }
}
```

![installingAsyncTask_onPostExecute](/img/installingAsyncTask_onPostExecute.png)

```java
public PackageInstaller {
    public void commit(IntentSender statusReceicver) {
        mSession.commit(statusReceiver, false);
    }
}
```

```java
public class PackageInstallerSession {
    public void commit() {
        mActiveCount.incrementAndGet();

        mHandler.obtainMessage(MSG_COMMIT).sendToTarget();
    }

    private final Hanlder.Callback mHandlerCallback = new Handler.Callback() {
        @Override
        public boolean handlerMessage(Message msg) {
            case MSG_COMMIT:
                commitLocked();
        }
    }

    private void commitLocked() {
        mPm.installStage()
    }
}
```

### PackageManagerService

PackageManagerService主要负责管理系统的Package，包括APK的安装、卸载、信息的查询等
在SystemServer.initAndLoop()中启动。PackageManagerService的启动非常复杂，涉及到Setting对象、属性系统、Installer系统、PackageHandler、系统权限、
、AndroidManifest.xml、Resouce,、FileObserver已经APK的安装包的扫面等等这里我们就不过多描述了，我们只要知道这个 服务管理着apk的安装与卸载就可以了。

```java
public class PackageManagerService {

    void installStage() {
        final Message msg = mHandler.obtainMessage(INIT_COPY);//1
        ...
        mHandler.sendMesage(msg);
    }

    class PakckageHandler extends Handler {
        void doHandlerMessage(Message msg) {
            switch(msg.what) {
                case INIT_COPY:
                // 开启一个服务
                    connectToService()


                case MCS_BOUND: {
                    HandlerParams params = mPendingInstalls.get(0);
                    params.startCopy();
                }
            }
        }

        private boolean connectToService() {
            // 该服务用语检查和复制可移动文件的服务，这是一个比较耗时的操作，因此DefaultContainerService没有和PMS运行在同一进程，它运行在com.android.defcontainer进程
            Intent servie = new Intent().setComponent(DEFAULT_CONTAINER_COMPONENT);
            if (mContext.bindServiceAsUser(service, mDefContainerConn, Context.BIND_AUTO_CREATE, UserHandler.SYSTEM))
        }
    }

    final private DefaultContainerConnection mDefContainerConn = new DefaultContainerConnection();

    class DefaultContainerConnection implements ServiceConnection {
        public void onServiceConnected(ComponentName name, IBinder service) {
            if (DEBUG_SD_INSTALL) Log.i(TAG, "onServiceConnected");
            final IMediaContainerService imcs = IMediaContainerService.Stub
                    .asInterface(Binder.allowBlocking(service));
            mHandler.sendMessage(mHandler.obtainMessage(MCS_BOUND, imcs));
        }
    }

    private abstract class HandlerParams {
        final boolean startCopy() {
            
        }
    }
}
```
