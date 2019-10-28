# MultiDex

## 64K限制

64k method数量上限

1. DexOpt优化的限制：当Android系统启动一个应用的时候，有一步是对Dex进行优化，这个过程有一个专门的工具来处理，叫DexOpt。DexOpt的执行过程是在第一次加载Dex文件的时候执行的。这个过程会生成一个ODEX文件，即Optimised Dex。执行ODex的效率会比直接执行Dex文件的效率要高很多。但是在早期的Android系统中，DexOpt有一个问题，也就是这篇文章想要说明并解决的问题。DexOpt会把每一个类的方法id检索起来，存在一个链表结构里面。但是这个链表的长度是用一个short类型来保存的，导致了方法id的数目不能够超过65536个。当一个项目足够大的时候，显然这个方法数的上限是不够的。尽管在新版本的Android系统中，DexOpt修复了这个问题，但是我们仍然需要对老系统做兼容。
2. dalvik bytecode的限制：因为 Dalvik 的 invoke-kind 指令集中，method reference index 只留了 16 bits，最多能引用 65535 个方法，[参考链接](http://stackoverflow.com/questions/21490382/does-the-android-art-runtime-have-the-same-method-limit-limitations-as-dalvik/21492160#21492160，http://source.android.com/devices/tech/dalvik/dalvik-bytecode.html)

## LinearAlloc限制

即使方法数没有超过65536，能正常编译打包成apk，在安装的时候，也有可能会提示INSTALL_FAILED_DEXOPT而导致安装失败，这个一般就是因为LinearAlloc的限制导致的。这个主要是因为Dexopt 使用 LinearAlloc 来存储应用的方法信息。Dalvik LinearAlloc 是一个固定大小的缓冲区。在Android 版本的历史上，LinearAlloc 分别经历了4M/5M/8M/16M限制。Android 2.2和2.3的缓冲区只有5MB，Android 4.x提高到了8MB 或16MB。当方法数量过多导致超出缓冲区大小时，也会造成dexopt崩溃

## 源码分析

1. 先看JVM是否支持MultiDex，若JVM本身就支持则MultiDex库工程将被禁用
2. 检查SDK版本号，要求最低版本为4
3. 执行具体的MultiDex操作


```java
//MultiDex
public static void install(Context context) {
    //判断是否需要执行MultiDex
    if (IS_VM_MULTIDEX_CAPABLE) {
        return;
    }
    doInstallation(..);
}

//MultiDex
public static void doInstallation(..) {
    //方法已经调用过了，不能再调用了
    sychronized(installedApk) {
        if (installedApk.contains(sourceApk)) {
            return;
        }
        installedApk.add(sourceApk);

        ClassLoader loader = getDexClassLoader(mainContext);
        if (loader == null) {
            return;
        }

        try {
            // 清除旧的dex文件，这里不是清除上次加载的dex文件缓存
            // 获取dex缓存目录 优先获取/dex/data/<package>/code-cache作为缓存目录
            // 如果获取失败，则使用/data/data/<package>/files/secondary-dexes目录
            // 这里清除的是第二个目录
            clearOldDexDir(mainContext);
        } catch(Throwable t) {

        }
        //获取缓存目录（data/data/<package>/code-cache/secondary-dexes）
        File dexDir = getDexDir(mainContext, dataDir, secondaryFolderName);
        //加载缓存文件（如果有）
        MultiDexExtractor extractor = new MultiDexExtractor(sourceApk, dexDir);
        try {
            //从apk压缩包里面提取dex文件，得到dex的zip列表
            List<? extends File> files = extractor.load(mainContext, prefsKeyPrefix, false);
            try {
                installSecondaryDexes(..);
            } catche (IOException e) {

            }
        } catch(IOException e) {
            files = extractor.load(..);
            installSecondaryDexes(..);
        } finally {
            try {
                extractor.close();
            } catch(IOException e) {
                closeException = e;
            }
        }
    }
}

//MultiDexExtractor
MultiDexExtractor(..) {
    File lockFile = new File(dexDir, "MultiDex.lock");
    lockRaf = new RanDomAccessFile(lockFile, "rw");
    try {
        lockChannel = lockRaf.getChannel();
        // 文件锁，防止多进程冲突
        cacheLock = lockChannel.lock();
    }
}

//MultiDexExtractor
List<? extends File> load(..) {
    // 判断是否强制重新解压，第一次会优先使用已解压过的dex文件
    // 此外crc和文件修改时间，判断如果apk文件已经被修改（覆盖安装），就会跳过缓存
    if (!forceReload && isModified(context, sourceApk, sourceCrc, prefsKeyPrefix)) {
        try {
            //加载缓存的dex文件
            files = loadExistingExtractions(context, sourceApk, dexDir);
        } catch (IOException ioe) {
            //加载失败的话重新解压，并保存解压出来的dex文件的信息
            files = performExtractions(sourceApk, dexDir);
            putStoredApkInfo(..);

        }
    } else {
        files = performExtractions();
        putStoredApkInfo(..);
    }
    return files;
}
```
