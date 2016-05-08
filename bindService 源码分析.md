# bindService 源码分析 【进行中】

## 简介

客户端通过 ContextWrapper.bindService() 方法来绑定服务，本地对 bindService 的过程进行源码分析。

**整体时序图**

![bindService 时序图](https://github.com/xxx)

## 源码分析

1\. 客户端调用 [bindService()](https://github.com/xdtianyu/AidlExample/blob/master/app/src/main/java/org/xdty/aidlexample/MainActivity.java#L45) 绑定服务

```java
// MainActivity.java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    Intent intent = new Intent().setComponent(new ComponentName(
            "org.xdty.remoteservice",
            "org.xdty.remoteservice.RemoteService"));
    bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
}
```

`Activity.bindService()` 最终调用的是 [ContextImpl.bindService()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ContextImpl.java#L1283) 方法

2\. ContextImpl.bindService()

```java
// ContextImpl.java

@Override
public boolean bindService(Intent service, ServiceConnection conn,
        int flags) {
    warnIfCallingFromSystemProcess();
    return bindServiceCommon(service, conn, flags, Process.myUserHandle());
}

private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
        UserHandle user) {
    IServiceConnection sd;

    sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(),
                mMainThread.getHandler(), flags);
    ...
    try {
        IBinder token = getActivityToken();
        ...
        int res = ActivityManagerNative.getDefault().bindService(
            mMainThread.getApplicationThread(), getActivityToken(), service,
            service.resolveTypeIfNeeded(getContentResolver()),
            sd, flags, getOpPackageName(), user.getIdentifier());
        ...
        return res != 0;
    } catch (RemoteException e) {
        ...
    }
}   
```

可见接着调用了 [ActivityManagerNative.getDefault().bindService()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ActivityManagerNative.java#L3740) 方法

这里的 `mPackageInfo` 是一个 [LoadedApk](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/LoadedApk.java#L977) 实例。`mMainThread` 则是一个 [ActivityThread](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ActivityThread.java#L1705)。

3\. ActivityManagerNative.getDefault().bindService()

注意 `ActivityManagerNative.java` 文件中有两个类 [ActivityManagerNative](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ActivityManagerNative.java#L61) [ActivityManagerProxy](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ActivityManagerNative.java#L2619), 他们都实现了 [IActivityManager](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/IActivityManager.java#L66)接口，而 [ActivityManagerNative]() 则继承了 `Binder` 对象，相当于 AIDL 中的 Stub，``ActivityManagerProxy` 则相当于 Proxy。

首先看 [ActivityManagerNative.getDefault()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ActivityManagerNative.java#L84)

```java
static public IActivityManager getDefault() {
    return gDefault.get();
}
```
再找 [gDefault](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ActivityManagerNative.java#L2610)

```java
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        IBinder b = ServiceManager.getService("activity");
        IActivityManager am = asInterface(b);
        return am;
    }
};
```

再来看 [asInterface()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ActivityManagerNative.java#L67) 方法

```java
static public IActivityManager asInterface(IBinder obj) {
    if (obj == null) {
        return null;
    }
    IActivityManager in =
        (IActivityManager)obj.queryLocalInterface(descriptor);
    if (in != null) {
        return in;
    }

    return new ActivityManagerProxy(obj);
}
```
很容易发现其返回了一个 `ActivityManagerProxy` 对象，所以调用的 `bindService()` 方法实现在 `ActivityManagerProxy` 类中。注意这里不要和抽象类 `ActivityManagerNative.class` 继承的 `bindService()` 方法混淆，它的实现在子类 ActivityManagerService 中。接下来我们继续看 Proxy 中的实现：

```java
// ActivityManagerNative.java

// ActivityManagerProxy.class
public int bindService(IApplicationThread caller, IBinder token,
        Intent service, String resolvedType, IServiceConnection connection,
        int flags,  String callingPackage, int userId) throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(caller != null ? caller.asBinder() : null);
    data.writeStrongBinder(token);
    service.writeToParcel(data, 0);
    data.writeString(resolvedType);
    data.writeStrongBinder(connection.asBinder());
    data.writeInt(flags);
    data.writeString(callingPackage);
    data.writeInt(userId);
    mRemote.transact(BIND_SERVICE_TRANSACTION, data, reply, 0);
    reply.readException();
    int res = reply.readInt();
    data.recycle();
    reply.recycle();
    return res;
}
```

可见这里将要绑定的服务信息打包，通过一个 binder 对象，发送了 `BIND_SERVICE_TRANSACTION` 命令给 `ActivityManager` 的服务端。而 `ActivityManager` 的服务端就是 `ActivityManagerNative`，所以接着命令会被 [ActivityManagerNative.onTransact()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ActivityManagerNative.java#L143) 处理

4\. ActivityManagerNative.onTransact()

`ActivityManagerNative` 对 [BIND_SERVICE_TRANSACTION](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ActivityManagerNative.java#L973) 命令重新解包并通过子类实现的 [ActivityManagerNative.bindService()]() 处理，注意这里的 `bindService()` 和上文的 `ActivityManagerProxy.bindService()` 不同。

```java
 public abstract class ActivityManagerNative extends Binder implements IActivityManager
{
    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        ...
        case BIND_SERVICE_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();
            IApplicationThread app = ApplicationThreadNative.asInterface(b);
            IBinder token = data.readStrongBinder();
            Intent service = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();
            b = data.readStrongBinder();
            int fl = data.readInt();
            String callingPackage = data.readString();
            int userId = data.readInt();
            IServiceConnection conn = IServiceConnection.Stub.asInterface(b);
            int res = bindService(app, token, service, resolvedType, conn, fl,
                    callingPackage, userId);
            reply.writeNoException();
            reply.writeInt(res);
            return true;
        }
        ...
}
```

5\. ActivityManagerService.bindService()

我们继续看 `ActivityManagerNative` 的子类 [ActivityManagerService](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java#L15897) 对 `bindService()` 方法的实现：

```java
public int bindService(IApplicationThread caller, IBinder token, Intent service,
        String resolvedType, IServiceConnection connection, int flags, String callingPackage,
        int userId) throws TransactionTooLargeException {
    ...
    synchronized(this) {
        return mServices.bindServiceLocked(caller, token, service,
                resolvedType, connection, flags, callingPackage, userId);
    }
}
```

可见通过 `mServices.bindServiceLocked()` 方法又传递给了下一层，这里的 [mServices](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java#L2309) 则是一个 `ActiveServices` 实例。我们继续追踪 [ActiveServices.bindServiceLocked()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/services/core/java/com/android/server/am/ActiveServices.java#L697)

6\. ActiveServices.bindServiceLocked()

接下来看 bindServiceLocked() 关键代码片断。

```java
int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
        String resolvedType, IServiceConnection connection, int flags,
        String callingPackage, int userId) throws TransactionTooLargeException {
    // 获取当前应用的进程记录
    final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);

    // 获取当前 Activity 记录
    ActivityRecord activity = null;
    if (token != null) {
        activity = ActivityRecord.isInStackLocked(token);
    }
    ...

    // 获取 ServiceLookupResult，其中 res.record 就是我们要启动的 service
    ServiceLookupResult res =
        retrieveServiceLocked(service, resolvedType, callingPackage,
                Binder.getCallingPid(), Binder.getCallingUid(), userId, true, callerFg);
    ServiceRecord s = res.record;
    
    mAm.startAssociationLocked(callerApp.uid, callerApp.processName,
            s.appInfo.uid, s.name, s.processName);

    AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
    
    // 封装客户端传入的 IServiceConnection，用于绑定后回调
    ConnectionRecord c = new ConnectionRecord(b, activity,
            connection, flags, clientLabel, clientIntent);

    // 保存 connection 记录到列表中
    IBinder binder = connection.asBinder();
    ArrayList<ConnectionRecord> clist = s.connections.get(binder);
    if (clist == null) {
        clist = new ArrayList<ConnectionRecord>();
        s.connections.put(binder, clist);
    }
    clist.add(c);
    b.connections.add(c);
    if (activity != null) {
        if (activity.connections == null) {
            activity.connections = new HashSet<ConnectionRecord>();
        }
        activity.connections.add(c);
    }
    b.client.connections.add(c);
    if ((c.flags&Context.BIND_ABOVE_CLIENT) != 0) {
        b.client.hasAboveClient = true;
    }
    if (s.app != null) {
        updateServiceClientActivitiesLocked(s.app, c, true);
    }
    clist = mServiceConnections.get(binder);
    if (clist == null) {
        clist = new ArrayList<ConnectionRecord>();
        mServiceConnections.put(binder, clist);
    }
    clist.add(c);

    // 继续进入 bringUpServiceLocked() 处理
    if ((flags&Context.BIND_AUTO_CREATE) != 0) {
        s.lastActivity = SystemClock.uptimeMillis();
        if (bringUpServiceLocked(s, service.getFlags(), callerFg, false) != null) {
            return 0;
        }
    }
    ...
}
```

7\. ActiveServices.bringUpServiceLocked()

继续查看 [bringUpServiceLocked()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/services/core/java/com/android/server/am/ActiveServices.java#L1362) 方法

```java

private final String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
        boolean whileRestarting) throws TransactionTooLargeException {
    ...

    final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
    final String procName = r.processName;
    ProcessRecord app;

    // 获取进程记录
    app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
    if (app != null && app.thread != null) {
        try {
            app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);
            // 启动服务
            realStartServiceLocked(r, app, execInFg);
            return null;
        } catch (TransactionTooLargeException e) {
            ...
        }
    }
    ...
}
```
8\. ActiveServices.realStartServiceLocked()

继续追踪 [ActiveServices.realStartServiceLocked()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/services/core/java/com/android/server/am/ActiveServices.java#L1492)

```java
private final void realStartServiceLocked(ServiceRecord r,
        ProcessRecord app, boolean execInFg) throws RemoteException {
    ...
    try {
        ...
        // 创建服务，服务端最终会响应 onCreate() 方法。
        app.thread.scheduleCreateService(r, r.serviceInfo,
                mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                app.repProcState);
        ...
    } catch (DeadObjectException e) {
        ...
    }

    // 绑定服务， 服务最终端会响应 onBind() 方法
    requestServiceBindingsLocked(r, execInFg);
    ...
}
```

## 参考

[Android应用程序绑定服务（bindService）的过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6745181)
