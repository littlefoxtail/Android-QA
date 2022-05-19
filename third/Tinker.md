# Tinker

## 自动生成TinkerApplication

```java
@DefaultLifeCycle(application = "tinker.sample.android.app.SampleApplication",
                  flags = ShareConstants.TINKER_ENABLE_ALL,
                  loadVerifyFlag = false)
public class SmapleApplicationLike extends DefaultApplication {

}
```

```java
//AnnotationProcessor
@Override
public Set<String> getSupportedAnnotationTypes() {
    final Set<String> supportedAnnotationTypes = new LinkedHashSet<>();

    supportedAnnotationTypes.add(DefaultLifeCycle.class.getName());

    return supportedAnnotationTypes;
}

@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    processDefaultLifeCycle(roundEnv.getElementsAnnotatedWith(DefaultLifeCycle.class));
    return true;
}
```

整个 processDefaultLifeCycle 方法看下来，其实主要在做的就是去读取一份模版，然后用注解中设置的值替换里面的一些占位符。这个模版就是 resouces/TinkerAnnoApplication.tmpl

```java
package %PACKAGE%;

import com.tencent.tinker.loader.app.TinkerApplication;

/**
 *
 * Generated application for tinker life cycle
 *
 */
public class %APPLICATION% extends TinkerApplication {

    public %APPLICATION%() {
        super(%TINKER_FLAGS%, "%APPLICATION_LIFE_CYCLE%", "%TINKER_LOADER_CLASS%", %TINKER_LOAD_VERIFY_FLAG%);
    }

}
```

最终生成：

```java
/**
 *
 * Generated application for tinker life cycle
 *
 */
public class SampleApplication extends TinkerApplication {

    public SampleApplication() {
        super(7, "tinker.sample.android.app.SampleApplicationLike", "com.tencent.tinker.loader.TinkerLoader", false);
    }

}
```

## 解析TinkerApplication

```java
// TinkerApplication
protected void attachBaseContext(Context base) {
    onBaseContextAttached(base);
}

/**
* Hook for sub-classes to run logic after the {@link Application#attachBaseContext} has been
* called but before the delegate is created. Implementors should be very careful what they do
* here since {@link android.app.Application#onCreate} will not have yet been called.
*/
private void onBaseContextAttached(Context base) {
    try {
        applicationStartElapsedTime = SystemClock.elapsedRealtime();
        applicationStartMillisTime = System.currentTimeMillis();
        loadTinker();
        ensureDelegate();
        invokeAppLikeOnBaseContextAttached(applicationLike, base);
        //reset save mode
        if (useSafeMode) {
            ShareTinkerInternals.setSafeModeCount(this, 0);
        }
    } catch (TinkerRuntimeException e) {
        throw e;
    } catch (Throwable thr) {
        throw new TinkerRuntimeException(thr.getMessage(), thr);
    }
}

```

```java
//TinkerLoader
public Intent tryLoad(TinkerApplication app) {
    Intent resultIntent = new Intent();

    long begin = SystemClock.elapsedRealtime();
    tryLoadPatchFilesInternal(app, resultIntent);
    long cost = SystemClock.elapsedRealtime() - begin;
    ShareIntentUtil.setIntentPatchCostTime(resultIntent, cost);
    return resultIntent;
}
//TinkerLoader
private void tryLoadPatchFilesInternal(..) {
    //获取tinker目录，/data/data/<packagename>/tinker
    File patchDirectoryFile = SharePathFileUtil.getPatchDirectory(app);
    //文件目录 /data/data/<packagename>/tinker/patch.info
    File pathInfoFile = SharePathFileUtil.getPatchInfoFile(patchDirectoryPath);
    //检查patch.info补丁信息文件是否存在
    if (!patchInfoFile.exists()) {
        ShareIntentUtil.setIntentReturnCode(..);
        return;
    }
    //tinker/info.lock
    File patchInfoLockFile = SharePatchFileUtil.getPatchInfoLockFile(patchDirectoryPath);
    // 检查patch info文件中的补丁版本信息
    patchInfo = SharePatchInfo.readAndCheckPropertyWithLock(patchInfoFile, patchInfoLockFile);
    if (patchInfo == null) {
        ShareIntentUtil.setIntentReturnCode(resultIntent, ShareConstants.ERROR_LOAD_PATCH_INFO_CORRUPTED);
        return;
    }
    //检查读取出来的patchInfo补丁版本信息
    String oldVersion = patchInfo.oldVersion;
    String newVersion = patchInfo.newVersion;
    String oatDex = patchInfo.oatDir;

    if (mainProcess && isRemoveNewVersion) {
        //获取新的补丁文件夹，例如patch-2c150d85
        String patchName = SharePatchFileUtil.getPatchVersionDirectory(newVersion);
        if (patchName != null) {
            //删除新的补丁插件
            String patchVersionDirFullPath = patchDirectoryPath + "/" + patchName;
            SharePatchFileUtil.deleteDir(patchVersionDirFullPath);
            //如果旧版本和新版本一致，就把oldVersion和newVersion设置为空来清除补丁
            if (oldVersion.equals(newVersion)) {
                oldVersion = "";
            }
            // 如果!oldVersion.equals(newVersion)意味着新补丁已经应用了，需要回退到原来的旧版本
            newVersion = oldVersion;
            patchInfo.oldVersion = oldVersion;
            patchInfo.newVersion = newVersion;
            //把数据重新写入 patchInfo文件中
            SharePatchInfo.rewritePatchInfoFileWithLock(..);
            //杀掉主进程以外的所有进程
            ShareTinkerInternals.killProcessExceptionMain(app);
        }
    }
    resultIntent.putExtra(ShareIntentUtil.INTENT_PATCH_OLD_VERSION, oldVersion);
    resultIntent.putExtra(ShareIntentUtil.INTENT_PATCH_NEW_VERSION, newVersion);

    boolean versionChanged = !(oldVersion.equals(newVersion));
}

```

```java
// Tinker
public void install(Intent intentResult) {
    install(intentResult, DefaultTinkerResultService.class, new UpgradePatch());
}


```

## oat文件的生成过程

```java
// DexDiffPatchInternal
private static boolean dexOptimizeDexFiles(..) {
    // 参数dexFiles表示需要优化的dex文件列表
    // optimizeDexDirectory为oat文件的存放路径
}

```