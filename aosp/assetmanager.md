# 资源处理

## getResources

```java
@Override
public Resources getResources() {
    return mResources;
}
```

## mResources何时赋值

入口比较多，这里选了createApplicationContext：

```java
class ContextImpl {
    @Override
    public Context createApplicationContext(..) {
        c.setResources(createResources(mActivityToken, pi, null, displayId, null, ..));
    }

    private static Resources createResources(..) {

        final String[] splitResDirs;
        final ClassLoader classLoader;
        try {
            classloader = pi.getSplitClassLoader(splitName);
        } catch (NameNotFoundException e) {

        }
        return ResourcesManager.getInstance().getResources(..);
    }

}
```

转移到ResourcesManager：

```java
public class ResourcesManager {
    public @Nullable Resources getResources(..) {
        final ResoucesKey key = new ResourcesKey(..);
        return getOrCreateResources(activityToken, key, classloader);
    }

    private Resources getOrCreateResources(..) {
        //创建ResourcesImpl：资源访问的实现。该类包含AssetManager和所有缓存与之相关
        ResourcesImpl resourcesImpl = createResourcesImpl(key);
        resources = getOrCreateResourcesLocked(..);
        return resources;
    }

    private ResourcesImpl createResourcesImpl() {
        final AssetManager assets = createAssetManager(key);
        final ResourcesImpl impl = new ResourcesImpl(assets);
        return impl;
    }

    protected AssetManager createAssetManager(..) {

    }
}
```

## AssetManager

AssetManager类有三个成员变量mAssetPaths、mResources和mConfig。

mAssetPaths保存的是资源存放目录，mResources指向的是一个资源索引表，而mConfig是设备的本地配置信息

### 5.0

ensureStringBlocks:

```java
public class AssetManager {
    final void ensureStringBlocks() {
        if (mStringBlocks == null) {
            synchronized(this) {
                if (mStringBlocks == null) {
                    makeStringBlocks(sSystem.mStringBlocks);
                }
            }
        }
    }

    final void makeStringBlocks(StringBlocks[] seed) {
        final int seedNum = (seed != null) ? seed.length : 0;
        // num是包含了系统资源表的个数sysNum
        final int num = getStringBlockCount();
        mStringBlocks = new StringBlock[num];

        for (int i=0; i<num; i++) {
            if (i < seedNum) {
                mStringBlocks[i] = seed[i];
            } else {
                mStringBlocks[i] = new StringBlock(getNativeStringBlock(i), true);
            }
        }
    }
}
```

```java
public class Resouces {
    // 缓存Resources的对象池
    final SynchronizedPool<TypedArray> mTypedArrayPool = new SynchronizedPool<TypedArray>(5);
}
```

## 资源获取

```java
//AssetManager
boolean getResouceValue(@AnyRes int resId, int densityDpi, @NonNull TypedValue outValue) {
    Preconditions.checkNotNull(outValue, "outValue");
    synchronized (this) {
        ensureValidLocked();
        //1. 通过native方法，进行资源的查找操作
        final int cookie = nativeGetResourceValue(
                mObject, resId, (short) densityDpi, outValue, resolveRefs);
        if (cookie <= 0) {
            return false;
        }

        // Convert the changing configurations flags populated by native code.
        outValue.changingConfigurations = ActivityInfo.activityInfoConfigNativeToJava(
                outValue.changingConfigurations);
        //2.若资源值为String类型，则从ApkAssets的GlobalStringPool中获取对应的数据
        if (outValue.type == TypedValue.TYPE_STRING) {
            outValue.string = mApkAssets[cookie - 1].getStringFromPool(outValue.data);
        }
        return true;
    }
}

```

## 资源获取native部分

[NativeGetResourceValue](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_AssetManager.cpp;bpv=1;bpt=1;l=743?gsn=NativeGetResourceValue&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Dc%252B%252B%3Fpath%3Dframeworks%2Fbase%2Fcore%2Fjni%2Fandroid_util_AssetManager.cpp%23JZ98mMUdAAti3l0eFhgdHTZ3ZXORLtrc1LLyaG4Xpn4)

```cpp
static jint NativeGetResourceValue(JNIEnv* env, jclass /*clazz*/, jlong ptr, jint resid,
                                   jshort density, jobject typed_value,
                                   jboolean resolve_references) {
  ScopedLock<AssetManager2> assetmanager(AssetManagerFromLong(ptr));
  Res_value value;
  ResTable_config selected_config;
  uint32_t flags;
  ApkAssetsCookie cookie =
      assetmanager->GetResource(static_cast<uint32_t>(resid), false /*may_be_bag*/,
                                static_cast<uint16_t>(density), &value, &selected_config, &flags);
  if (cookie == kInvalidCookie) {
    return ApkAssetsCookieToJavaCookie(kInvalidCookie);
  }

  uint32_t ref = static_cast<uint32_t>(resid);
  if (resolve_references) {
    cookie = assetmanager->ResolveReference(cookie, &value, &selected_config, &flags, &ref);
    if (cookie == kInvalidCookie) {
      return ApkAssetsCookieToJavaCookie(kInvalidCookie);
    }
  }
  return CopyValue(env, cookie, value, ref, flags, &selected_config, typed_value);
}
```

[ApkAssetsCookie](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/androidfw/include/androidfw/AssetManager2.h;bpv=0;bpt=1;l=37)

```cpp
using ApkAssetsCookie = int32_t;

enum : ApkAssetsCookie {
  kInvalidCookie = -1,
}; 
```

### GetResource

[AssetManager2::GetResource](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/androidfw/AssetManager2.cpp;bpv=0;bpt=1;l=**691**)

```cpp
ApkAssetsCookie AssetManager2::GetResource(uint32_t resid, bool may_be_bag,
                                           uint16_t density_override, Res_value* out_value,
                                           ResTable_config* out_selected_config,
                                           uint32_t* out_flags) const {
  FindEntryResult entry;
  //通过FindEntry，查找每一个资源entry，每一个entry包含真实资源数据，这个时候out_value会获取当前entry中的真实数据
  ApkAssetsCookie cookie = FindEntry(resid, density_override, false /* stop_at_first_match */,
                                     false /* ignore_configuration */, &entry);
  if (cookie == kInvalidCookie) {
    return kInvalidCookie;
  }
    //当前引用比较复杂的时候，是一个引用，并非真实的资源数据，则data返回的是当前应用的resid
  if (dtohs(entry.entry->flags) & ResTable_entry::FLAG_COMPLEX) {
    if (!may_be_bag) {
      LOG(ERROR) << base::StringPrintf("Resource %08x is a complex map type.", resid);
      return kInvalidCookie;
    }

    // Create a reference since we can't represent this complex type as a Res_value.
    out_value->dataType = Res_value::TYPE_REFERENCE;
    out_value->data = resid;
    *out_selected_config = entry.config;
    *out_flags = entry.type_flags;
    return cookie;
  }

  const Res_value* device_value = reinterpret_cast<const Res_value*>(
      reinterpret_cast<const uint8_t*>(entry.entry) + dtohs(entry.entry->size));
  out_value->copyFrom_dtoh(*device_value);
    // 接着会覆盖当前的资源id，以及相关的配置等信息。
  // Convert the package ID to the runtime assigned package ID.
  entry.dynamic_ref_table->lookupResourceValue(out_value);

  *out_selected_config = entry.config;
  *out_flags = entry.type_flags;
  return cookie;
}
```

### FindEntry

```cpp

```
