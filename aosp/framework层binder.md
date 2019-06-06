# 通信过程

## IActivityManager：

```java
@Override 
public android.content.ComponentName startService(android.app.IApplicationThread caller, android.content.Intent service, java.lang.String resolvedType, boolean requireForeground, java.lang.String callingPackage, int userId) throws android.os.RemoteException {
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    android.content.ComponentName _result;
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        _data.writeStrongBinder((((caller!=null))?(caller.asBinder()):(null)));
        if ((service!=null)) {
        _data.writeInt(1);
        service.writeToParcel(_data, 0);
    }
    else {
        _data.writeInt(0);
    }
    // 写入Parcel数据
    _data.writeString(resolvedType);
    _data.writeInt(((requireForeground)?(1):(0)));
    _data.writeString(callingPackage);
    _data.writeInt(userId);
    // 通过Binder传递数据
    mRemote.transact(Stub.TRANSACTION_startService, _data, _reply, 0);
    // 读取应答消息的异常情况
    _reply.readException();
    if ((0!=_reply.readInt())) {
        // 根据reply数据来创建ComponentName对象
        _result = android.content.ComponentName.CREATOR.createFromParcel(_reply);
    }
    else {
        _result = null;
    }
    }
    finally {
        _reply.recycle();
        _data.recycle();
    }
    return _result;
}
```

主要功能：

- 获取或创建两个Parcel对象，data用于发送数据，reply用于接收应答数据
- 将startService相关数据都封装到Parcel对象data，其中descritor="android.app.IActivityManager"
- 通过Binder传递数据，并将应答消息写入reply
- 读取reply应答消息的异常情况和组件对象

## Parcel.obtain

```java
public static Parcel obtain() {
    final Parcel[] pool = sOwnedPool;
    synchronized(pool) {
        Parcel p;
        for (int i=0; i<POOL_SIZE; i++) {
            p = pool[i];
            if (p != null) {
                pool[i] = null;
                p.mReadWriteHelper = ReadWriteHelper.DEFAULT;
                return p;
            }
        }
    }
    return new Parcel(0);
}
```

1. 先尝试从缓存池sOwnedPool中查询是否存在缓存Parcel对象，当存在则直接返回该对象
2. 如果没有可用的Parcel对象，则直接创建Parcel对象

## new Parcel

```java
private Parcel(long nativePtr) {
    init(nativePtr);
}

private void init(long nativePtr) {
    if (nativePtr != 0) {
            mNativePtr = nativePtr;
            mOwnsNativeParcelObject = false;
        } else {
            mNativePtr = nativeCreate();
            mOwnsNativeParcelObject = true;
        }
}
```

nativeCreate这是native方法，经过JNI进入native层，调用android_os_Parcel_create()方法

```c++
static jlong android_os_Parcel_create(JNIEnv* env, jclass clazz) {
    Parcel* parcel = new Parcel();
    return reinterpret_cast<jlong>(parcel);
}
```

## Parcel.recycle

```java
public final void recycle() {
    // 释放native parcel对象
    freeBuffer();

    final Parcel[] pool;
    // 根据情况来选择加入相应池
    if (mOwnsNativeParcelObject) {
        pool = sOwnedPool;
    } else {
        mNativePtr = 0;
        pool = sHolderPool;
    }

    synchronized(pool) {
        for (int i=0; i<POOL_SIZE; i++) {
            if (pool[i] == null) {
                pool[i] = this;
                return;
            }
        }
    }
}
```

将不再使用的Parcel对象放入缓存池，可回收重复利用，当缓存池已满则不再加入缓存池。这里有两个Parcel线程池：

- mOwnsNativeParcelObject=true，即调用不带参数obtain()方法获取的对象，回收时会放入sOwnedPool对象池;
- mOwnsNativeParcelObject=false, 即调用带nativePtr参数的obtain(long)方法获取的对象, 回收时会放入sHolderPool对象池;

## Parcel.writeString

Parcel.java:

```java
public final void writeString(String val) {
     mReadWriteHelper.writeString(this, val);
}
```

Parcel.java:

```java
public void writeString(Parcel p, String s) {
    nativeWriteString(p.mNativePtr, s);
}
```

android_os_Parcel.cpp:

```c++
static void android_os_Parcel_writeString(JNIEnv* env, jclass clazz, jlong nativePtr, jstring val)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        status_t err = NO_MEMORY;
        if (val) {
            const jchar* str = env->GetStringCritical(val, 0);
            if (str) {
                err = parcel->writeString16(
                    reinterpret_cast<const char16_t*>(str),
                    env->GetStringLength(val));
                env->ReleaseStringCritical(val, str);
            }
        } else {
            err = parcel->writeString16(NULL, 0);
        }
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
}
```

Parcel.cpp:

```c++
status_t Parcel::writeString16(const char16_t* str, size_t len)
{
    if (str == NULL) return writeInt32(-1);

    status_t err = writeInt32(len);
    if (err == NO_ERROR) {
        len *= sizeof(char16_t);
        uint8_t* data = (uint8_t*)writeInplace(len+sizeof(char16_t));
        if (data) {
            memcpy(data, str, len);
            *reinterpret_cast<char16_t*>(data+len) = 0;
            return NO_ERROR;
        }
        err = mError;
    }
    return err;
}
```

除了writeString(),在Parcel.java中大量的native方法, 都是调用android_os_Parcel.cpp相对应的方法, 该方法再调用Parcel.cpp中对应的方法. 
调用流程: Parcel.java –> android_os_Parcel.cpp –> Parcel.cpp.

## mRemote

ServiceManager:

```java
public static IActivityManager getService() {
    return IActivityManagerSingleton.get();
}
```

ServiceManager:

```java
private static final Singleton<IActivityManager> IActivityManagerSingleton = 
    new Singleton<IActivityManager>() {
        @Override
        protected IActivityManager create() {
            final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
            final IActivityManager am = IActivityManager.Stub.asInterface(b);
            return am;
        }
    }
```

ServiceManager.getService(“activity”)返回的是指向目标服务AMS的代理对象BinderProxy对象，由该代理对象可以找到目标服务AMS所在进程

ActivityManagerNative:

```java
static public IActivityManager asInterface(IBinder obj) {
    return IActivityManager.Stub.asInterface(obj);
}
```

IActivityManager:

```java
public static IActivityManager asInterface(IBinder obj) {
    if (obj == null) {
        return null;
    }
    //此处obj = BinderProxy
    IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
    // iin为null
    if ((iin != null) && (iin instanceof IActivityManager)) {
        return (IActivityManager)(iin);
    }
    return new IActivityManager.Stub.Proxy(obj);
}
```

Binder.java:

```java
public class Binder implements IBinder {
    public IInterface queryLocalInterface(String descriptor) {
        if (mDescriptor.equals(descriptor)) {
            return mOwner;
        }
        return null;
    }
}

final class BinderProxy implements IBinder {
    //BinderProxy对象的调用，则防止为空
    public IInterface queryLocalInterface(String descriptor) {
        return null;
    }
}
```

对于Binder IPC的过程中, 同一个进程的调用则会是asInterface()方法返回的便是本地的Binder对象;对于不同进程的调用则会是远程代理对象BinderProxy

## 创建IActivityManager.Proxy

IActivityManager.java:

```java
private static class Proxy implements IActivityManager {
    private IBinder mRemote;
    Proxy(IBidner remote) {
        // mRemote便是指向AMS服务的BinderProxy对象
        mRemote = remote;
    }
}
```

## mRemote.transact

Binder.java::BinderProxy

```java
public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
     return transactNative(code, data, reply, flags);
}
```

## Binder.execTransact

经过很多native代码，从C++回到了Java代码
Binder.java:

```java
private boolean execTransact(int code, long dataObj, long replyObj, int flags) {
    Parcel data = Parcel.obtain(dataObj);
    Parcel reply = Parcel.obtain(replyObj);

    boolean res;

    try {
        res = onTransact(code, data, reply, flags);
    } catch (RemoteException e) {
        if ((flags & FLAG_ONEWAY) != 0) {
            if (e instanceof RemoteException) {
                Log.w(TAG, "Binder call failed.", e);
            } else {
                Log.w(TAG, "Caught a RuntimeException from the binder stub implementation.", e);
            }
        } else {
            reply.setDataPosition(0);
            reply.writeException(e);
        }
        res = true;
    } finally {
        checkParcel(this, code, reply, "Unreasonably large binder reply buffer");
        reply.recycle();
        data.recycle();
    }
    return res;
}
```

## IActivityManager.onTransact

```java
public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    switch(code) {
        case TRANSACTION_startActivity:{
            data.enforceInterface(DESCRIPTOR);
            //生成IApplicationThread的代理对象，即IApplicationThread.Stub.Proxy
            IApplicationThread _arg0 = IApplicationThread.Stub.asInterface(data.readStrongBinder());
            Intent _arg1;
            if ((0!=data.readInt())) {
                _arg1 = Intent.CREATOR.createFromParcel(data);
            }
            else {
                _arg1 = null;
            }
            String _arg2 = data.readString();
            boolean _arg3 = (0!=data.readInt());
            String _arg4 = data.readString();
            int _arg5 = data.readInt();
            // 调用ActivityManagerService的startService()方法
            android.content.ComponentName _result = this.startService(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5);
            reply.writeNoException();
            if ((_result!=null)) {
                reply.writeInt(1);
                _result.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
            }
            else {
                reply.writeInt(0);
            }
            return true;
            }
    }
}
```

```java
public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, boolean requireForeground, String callingPackage, int userId)
            throws TransactionTooLargeException {
    try {
        res = mServices.startServiceLocked(caller, service, resolvedType, callingPid, callingUid,
requireForeground, callingPackage, userId);
    } finally {
        Binder.restoreCallingIdentity(origId);
    }
}
```

历经千山万水，总算进入了AMS.startService。当system_service收到BR_TRANSACTION的过程后，通信并没有完全结束，还需将服务启动完成的回应消息，告诉给发起端进程

## 总结

Binder一步步调用进入到system_server进程的AMS.startSevice，整个过程涉及Java Framework，native，kernel driver各个层面知识
![binder_ipc_process](/img/binder_ipc_process.jpg)