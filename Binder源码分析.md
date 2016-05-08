# Binder 源码分析

本文是基于 [Android 6.0.0](https://github.com/xdtianyu/android-6.0.0_r1) 和 [kernel 3.4](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow) 源码 及 Android SDK 23 展开的。

**目录**

- [1. 简介](#1-%E7%AE%80%E4%BB%8B)
- [2. Binder 与 AIDL](#2-binder-%E4%B8%8E-aidl)
  - [2.1 AIDL 客户端](#21-aidl-%E5%AE%A2%E6%88%B7%E7%AB%AF)
  - [2.2 AIDL 服务端](#22-aidl-%E6%9C%8D%E5%8A%A1%E7%AB%AF)
  - [2.3 远程服务的获取与使用](#23-%E8%BF%9C%E7%A8%8B%E6%9C%8D%E5%8A%A1%E7%9A%84%E8%8E%B7%E5%8F%96%E4%B8%8E%E4%BD%BF%E7%94%A8)
- [3. Binder 框架及 Native 层](#3-binder-%E6%A1%86%E6%9E%B6%E5%8F%8A-native-%E5%B1%82)
  - [3.1 Binder Native 的入口](#31-binder-native-%E7%9A%84%E5%85%A5%E5%8F%A3)
  - [3.2 Binder 本地层的整个函数/方法调用过程](#32-binder-%E6%9C%AC%E5%9C%B0%E5%B1%82%E7%9A%84%E6%95%B4%E4%B8%AA%E5%87%BD%E6%95%B0%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B)
  - [3.3 Binder 设备文件的打开和读写](#33-binder-%E8%AE%BE%E5%A4%87%E6%96%87%E4%BB%B6%E7%9A%84%E6%89%93%E5%BC%80%E5%92%8C%E8%AF%BB%E5%86%99)
- [4. Binder 驱动](#4-binder-%E9%A9%B1%E5%8A%A8)
  - [4.1 binder 设备的创建](#41-binder-%E8%AE%BE%E5%A4%87%E7%9A%84%E5%88%9B%E5%BB%BA)
  - [4.2 binder协议和数据结构](#42-binder%E5%8D%8F%E8%AE%AE%E5%92%8C%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84)
  - [4.3 binder 驱动文件操作](#43-binder-%E9%A9%B1%E5%8A%A8%E6%96%87%E4%BB%B6%E6%93%8D%E4%BD%9C)
- [5. Binder 与系统服务](#5-binder-%E4%B8%8E%E7%B3%BB%E7%BB%9F%E6%9C%8D%E5%8A%A1)
  - [5.1 Context.getSystemService()](#51-contextgetsystemservice)
  - [5.2 Context.getSystemService() 源码分析](#52-contextgetsystemservice-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)
- [6. 结论](#6-%E7%BB%93%E8%AE%BA)
- [7. 参考](#7-%E5%8F%82%E8%80%83)

## 1. 简介

Binder 是一种 Android 进程间通信机制，提供远程过程调用(Remote Procedure Call)功能。我们最直接的使用是调用 `Context.getSystemService()` 来获取系统服务，或直接使用 `AIDL` 来实现多个程序(APP)间数据交互。

Binder 是非常重要的 Android 基础组件，几乎所有的进程间通信都是使用 Binder 机制实现的。本文将结合源码展开讲述 Binder ，同时对一些重要知识点提供扩展阅读的参考。

![android_binder](https://raw.githubusercontent.com/xdtianyu/SourceAnalysis/master/art/android_binder.png)

不管是 Android 系统服务(System services)还是用户的应用进程(User apps)，最终都会通过 binder 来实现进程间通信。上层应用首先通过 IBinder 的 transcate 方法发送命令给 libbinder， libbinder 再通过系统调用(ioctl) 发送命令到内核中的 binder 驱动，之后再由驱动完成进程间数据的交互。

我们经常使用的 Intent，Messager 数据传递也是对 Binder 更高层次的抽象和封装，最终还是会由内核中的 binder 驱动完成数据的传递。

## 2. Binder 与 AIDL

AIDL (Android Interface definition language) 是接口描述语言，用于生成在两个进程间进行通信的代码。先看 AIDL 概念图

![AIDL概念图](https://raw.githubusercontent.com/xdtianyu/SourceAnalysis/master/art/AIDL.png)

* Stub.Proxy 和 Stub 代码由 Android Sdk 自动生成，客户端通过 Stub.Proxy 与远程服务交互。

* Stub 包含对 IBinder 对象操作的封装，需要远程服务实现具体功能。


接下来再看具体实现， 完整源代码见 [AidlExample](https://github.com/xdtianyu/AidlExample)。在这个工程中，我们新建了两个应用， `app` 是客户端代码， `remoteservice` 则是服务端代码。

### 2.1 AIDL 客户端

在 Android Studio 项目上右键， `New` -> `AIDL` -> `AIDL File` 输入文件名后可以快速创建一个 AIDL 的代码结构。例如我们新建一个 [IRemoteService.aidl](https://github.com/xdtianyu/AidlExample/blob/master/app/src/main/aidl/org/xdty/remoteservice/IRemoteService.aidl) 文件

```java
// IRemoteService.aidl
package com.android.aidltest;

interface IRemoteService {

    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
}
```

从生成的示例代码可以看出，AIDL 的语法类似 Java， `basicTypes()` 方法传递的参数只是基本类型。

如果要传递自定义类型如 [User](https://github.com/xdtianyu/AidlExample/blob/master/remoteservice/src/main/java/org/xdty/remoteservice/User.java#L6)，则需要实现 [Parcelable](http://developer.android.com/reference/android/os/Parcelable.html) 接口。`Parcelable` 是一个与 Java `Serializable` 类似的序列化接口。 

这样类 `User` 的实例就可以储存到 [Parcel](http://developer.android.com/reference/android/os/Parcel.html) 中，而 `Parcel` 则是一个可以通过 `IBinder` 发送数据或对象引用的容器。

```java
// User.java
public class User implements Parcelable {

    private int uid;
    private String name;

    // 从 Parcel 中读取数据，顺序需要和写入保持一致
    protected User(Parcel in) {
        uid = in.readInt();
        name = in.readString();
    }

    // 必须实现，用于从 Parcel 对象中生成类实例
    public static final Creator<User> CREATOR = new Creator<User>() {
        @Override
        public User createFromParcel(Parcel in) {
            return new User(in);
        }

        @Override
        public User[] newArray(int size) {
            return new User[size];
        }
    };

    // 将数据写入到 Parcel 中， 顺序需要与读取保持一致
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(uid);
        dest.writeString(name);
    }
}
```

再向 `IRemoteService.aidl` 中添加一个 `addUser()` 方法，同时新建一个 [User.aidl](https://github.com/xdtianyu/AidlExample/blob/master/app/src/main/aidl/org/xdty/remoteservice/User.aidl) 文件。

```java
// IRemoteService.aidl
package com.android.aidltest;

import com.android.aidltest.User;

interface IRemoteService {

    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);

    // in 表示传入数据， out 表示传出数据， inout 表示双向传递。注意含有 out 时 User 类需要实现 readFromParcel() 方法
    void addUser(in User user);
}

// User.aidl
package com.android.aidltest;
parcelable User;
```
运行编译后，会在 `generated` 文件夹中生成一个 [IRemoteService.java](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L92) 接口文件。这个接口中有两个内部类 [Stub](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L19) 和 [Stub.Proxy](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L92)。注意客户端生成的`IRemoteService.java` 文件和在后文服务端生成的文件内容是相同的。

客户端会从 [Stub.asInterface()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L34) 得到 `IRemoteService (Stub.Proxy)` 的实例，这个实例就是一个通过 Binder 传递回来的 [远程对象](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L93) 的包装。而服务端则需要实现 [IRemoteService.addUser()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L15) 方法。

```java
// IRemoteService.java
public static org.xdty.remoteservice.IRemoteService asInterface(android.os.IBinder obj) {
    if ((obj == null)) {
        return null;
    }
    android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
    if (((iin != null) && (iin instanceof org.xdty.remoteservice.IRemoteService))) {
        return ((org.xdty.remoteservice.IRemoteService) iin);
    }
    return new org.xdty.remoteservice.IRemoteService.Stub.Proxy(obj);
}
```

### 2.2 AIDL 服务端

为了演示进程间通信，我们新建一个模块（应用） [RemoteService](https://github.com/xdtianyu/AidlExample/tree/master/remoteservice) 来实现功能，并在客户端绑定服务。

按客户端的结构新建 [IRemoteService.aidl](https://github.com/xdtianyu/AidlExample/blob/master/remoteservice/src/main/aidl/org/xdty/remoteservice/IRemoteService.aidl) [User.aidl](https://github.com/xdtianyu/AidlExample/blob/master/remoteservice/src/main/aidl/org/xdty/remoteservice/User.aidl) [User.java](https://github.com/xdtianyu/AidlExample/blob/master/remoteservice/src/main/java/org/xdty/remoteservice/User.java) 文件，并拷贝内容，注意如果需要请修改包名。

新建服务 [RemoteService](https://github.com/xdtianyu/AidlExample/blob/master/remoteservice/src/main/java/org/xdty/remoteservice/RemoteService.java#L29) ，覆盖(Override) `onBind()` 方法并返回 `IRemoteService.Stub` 实例 [mBinder](https://github.com/xdtianyu/AidlExample/blob/master/remoteservice/src/main/java/org/xdty/remoteservice/RemoteService.java#L11)：
  
```java
// RemoteService.java
public class RemoteService extends Service {
    private static final String TAG = RemoteService.class.getSimpleName();
    private IBinder mBinder = new IRemoteService.Stub() {
        @Override
        public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
                double aDouble, String aString) throws RemoteException {
            Log.d(TAG, "basicTypes: ");
        }

        @Override
        public void addUser(User user) throws RemoteException {
            Log.d(TAG, "addUser: " + user.name);
        }
    };

    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```

这样服务端就实现了 [addUser()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L15) 方法，当客户端通过远程对象调用 [IRemoteService.Stub.Proxy.addUser()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L135) 时，远程对象 [mRemote](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L147) 就会通过 [transact()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L147) 发送命令给服务端，服务端收到命令后在 [Stub.onTransact()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L76) 中读取数据并执行 [addUser()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L84) 方法。更多细节我们将在 [3. Binder 框架及 Native 层](#3-binder-%E6%A1%86%E6%9E%B6%E5%8F%8A-native-%E5%B1%82) 小节讲述。

```java
// IRemoteService.java
@Override
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply,
        int flags) throws android.os.RemoteException {
    switch (code) {
        ...
        case TRANSACTION_addUser: {
            data.enforceInterface(DESCRIPTOR);
            org.xdty.remoteservice.User _arg0;
            if ((0 != data.readInt())) {
                _arg0 = org.xdty.remoteservice.User.CREATOR.createFromParcel(data);
            } else {
                _arg0 = null;
            }
            this.addUser(_arg0);
            reply.writeNoException();
            return true;
        }
    }
    return super.onTransact(code, data, reply, flags);
}
```

### 2.3 远程服务的获取与使用

客户端要使用远程服务，需要绑定服务 ([bindService](https://github.com/xdtianyu/AidlExample/blob/master/app/src/main/java/org/xdty/aidlexample/MainActivity.java#L45)) 并建立服务连接 ([ServiceConnection](https://github.com/xdtianyu/AidlExample/blob/master/app/src/main/java/org/xdty/aidlexample/MainActivity.java#L19))。

```java
// MainActivity.java
public class MainActivity extends AppCompatActivity {
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            IRemoteService remoteService = IRemoteService.Stub.asInterface(service);
            try {
                remoteService.addUser(new User(1, "neo"));
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        ...
    };
    ...
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        ...
        Intent intent = new Intent().setComponent(new ComponentName(
                "org.xdty.remoteservice",
                "org.xdty.remoteservice.RemoteService"));
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }
}
```

我们可以看出，客户端通过 `binderService()` 方法，获取远程服务并在服务连接 `ServiceConnection` 中 `onServiceConnected()` 回调中得到了 `IBinder service` 实例， 最后通过上文提到的 `IRemoteService.Stub.asInterface(service)` 方法得到远程服务 `IRemoteService` 的实例。通过 `IRemoteService.addUser()` 方法我们可以像调用本地方法一样调用远程方法。在来看 `IRemoteService.addUser()` 的实现：

```java
// IRemoteService.java
public static org.xdty.remoteservice.IRemoteService asInterface(android.os.IBinder obj) {
    ...
    return new org.xdty.remoteservice.IRemoteService.Stub.Proxy(obj);
}

private static class Proxy implements org.xdty.remoteservice.IRemoteService {
    private android.os.IBinder mRemote;

    Proxy(android.os.IBinder remote) {
        mRemote = remote;
    }

    @Override
    public android.os.IBinder asBinder() {
        return mRemote;
    }
    ...
    @Override
    public void addUser(org.xdty.remoteservice.User user)
            throws android.os.RemoteException {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        try {
            _data.writeInterfaceToken(DESCRIPTOR);
            if ((user != null)) {
                _data.writeInt(1);
                user.writeToParcel(_data, 0);
            } else {
                _data.writeInt(0);
            }
            mRemote.transact(Stub.TRANSACTION_addUser, _data, _reply, 0);
            _reply.readException();
        } finally {
            _reply.recycle();
            _data.recycle();
        }
    }
}
```

可以看到客户端调用 [remoteService.addUser(new User(1, "neo"))](https://github.com/xdtianyu/AidlExample/blob/master/app/src/main/java/org/xdty/aidlexample/MainActivity.java#L25) 方法实际上是通过 [IBinder service](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L147) 实例的 [transact()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L147) 方法，发送了与服务端约定好的命令 `Stub.TRANSACTION_addUser`，并将参数按格式打包进 [Parcel](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L137) 对象。

服务端则在 [onTransact()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L51) 方法中收到命令后会对命令和参数重新解析：

```java
// IRemoteService.java
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply,
        int flags) throws android.os.RemoteException {
    switch (code) {
        ...
        case TRANSACTION_addUser: {
            data.enforceInterface(DESCRIPTOR);
            org.xdty.remoteservice.User _arg0;
            if ((0 != data.readInt())) {
                _arg0 = org.xdty.remoteservice.User.CREATOR.createFromParcel(data);
            } else {
                _arg0 = null;
            }
            this.addUser(_arg0);
            reply.writeNoException();
            return true;
        }
    }
    return super.onTransact(code, data, reply, flags);
}
```

可以看到在 `onTransact()` 中，最终 [this.addUser(_arg0)](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L84) 调用了上文提到的服务端的实现 [IRemoteService.Stub.addUser()](https://github.com/xdtianyu/AidlExample/blob/master/remoteservice/src/main/java/org/xdty/remoteservice/RemoteService.java#L19) 。

远程 Binder 对象 [mRemote](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L42) 是由客户端绑定服务时 [onServiceConnected()](https://github.com/xdtianyu/AidlExample/blob/master/app/src/main/java/org/xdty/aidlexample/MainActivity.java#L23) 返回的。继续追踪 [bindService()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ContextImpl.java#L1283)

```java
// ContextImpl.java
@Override
public boolean bindService(Intent service, ServiceConnection conn,
        int flags) {
    warnIfCallingFromSystemProcess();
    return bindServiceCommon(service, conn, flags, Process.myUserHandle());
}
```

可以看到最后是通过 [ActivityManagerNative.getDefault().bindService()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ContextImpl.java#L1317#L1320) 来绑定服务

```java
// bindServiceCommon()
int res = ActivityManagerNative.getDefault().bindService(
    mMainThread.getApplicationThread(), getActivityToken(), service,
    service.resolveTypeIfNeeded(getContentResolver()),
    sd, flags, getOpPackageName(), user.getIdentifier());

// ActivityManagerNative.getDefault().bindService()
public int bindService(IApplicationThread caller, IBinder token,
        Intent service, String resolvedType, IServiceConnection connection,
        int flags,  String callingPackage, int userId) throws RemoteException {
    ...
    data.writeStrongBinder(connection.asBinder());
    ...
    mRemote.transact(BIND_SERVICE_TRANSACTION, data, reply, 0);
    ...
}
```

追踪到 [ActivityManagerNative.getDefault().bindService()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ActivityManagerNative.java#L3740) ，可以发现 `ActivityManager` 和 [IServiceConnection](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ActivityManagerNative.java#L3750)也是一个 `AIDL` 实现。通过它的 `ActivityManagerProxy.bindService()` 将绑定请求发送给本地层。

再从 `onServiceConnected()` 回调追踪， [onServiceConnected()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/LoadedApk.java#L1223) 是由 [LoadedApk.ServiceDispatcher.doConnected()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/LoadedApk.java#L1175) 回调的。

*关于更多的 `bindService()` 远程服务创建及 `ServiceConnection` 回调， 请参考 [Android应用程序绑定服务（bindService）的过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6745181)*

从上面分析可以看出， AIDL 的本质是对 Binder 的又一次抽象和封装，实际的进程间通信仍是由 Binder 完成的。

## 3. Binder 框架及 Native 层

Binder机制使本地对象可以像操作当前对象一样调用远程对象，可以使不同的进程间互相通信。Binder 使用 Client/Server 架构，客户端通过服务端代理，经过 Binder 驱动与服务端交互。

![Binder框架图片](https://raw.githubusercontent.com/xdtianyu/SourceAnalysis/master/art/Binder.png)

Binder 机制实现进程间通信的奥秘在于 kernel 中的 Binder 驱动，将在 [4. Binder 驱动](#4-binder-%E9%A9%B1%E5%8A%A8) 小节详细讲述。

![Binder本地框架图片](https://raw.githubusercontent.com/xdtianyu/SourceAnalysis/master/art/binder_native.png)

JNI 的代码位于 [frameworks/base/core/jni](https://github.com/xdtianyu/android-6.0.0_r1/tree/master/frameworks/base/core/jni) 目录下，主要是 [android_util_Binder.cpp](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/jni/android_util_Binder.cpp) 文件和头文件 [android_util_Binder.h](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/jni/android_util_Binder.h)

Binder JNI 代码是 Binder Java 层操作到 Binder Native 层的接口封装，最后会被编译进 `libandroid_runtime.so` 系统库。

Binder 本地层的代码在 [frameworks/native/libs/binder](https://github.com/xdtianyu/android-6.0.0_r1/tree/master/frameworks/native/libs/binder) 目录下， 此目录在 Android 系统编译后会生成 `libbinder.so` 文件，供 JNI 调用。`libbinder` 封装了所有对 binder 驱动的操作，是上层应用与驱动交互的桥梁。头文件则在 [frameworks/native/include/binder](https://github.com/xdtianyu/android-6.0.0_r1/tree/master/frameworks/native/include/binder) 目录下。

### 3.1 Binder Native 的入口

[IInterface.cpp](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IInterface.cpp#L33) 是 Binder 本地层入口，与 java 层的 `android.os.IInterface` 对应，提供 `asBinder()` 的实现，返回 `IBinder` 对象。

在头文件中有两个类 [BnInterface (Binder Native Interface)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/include/binder/IInterface.h#L50) 和 [BpInterface (Binder Proxy Interface)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/include/binder/IInterface.h#L63), 对应于 java 层的 [Stub](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L19) 和 [Proxy](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L92)

```c++
sp<IBinder> IInterface::asBinder(const IInterface* iface)
{
    if (iface == NULL) return NULL;
    return const_cast<IInterface*>(iface)->onAsBinder();
}
```

```c++
template<typename INTERFACE>
class BnInterface : public INTERFACE, public BBinder
{
public:
    virtual sp<IInterface>      queryLocalInterface(const String16& _descriptor);
    virtual const String16&     getInterfaceDescriptor() const;

protected:
    virtual IBinder*            onAsBinder();
};

// ----------------------------------------------------------------------

template<typename INTERFACE>
class BpInterface : public INTERFACE, public BpRefBase
{
public:
                                BpInterface(const sp<IBinder>& remote);

protected:
    virtual IBinder*            onAsBinder();
};
```

其中 `BnInterface` 是实现Stub功能的模板，扩展BBinder的onTransact()方法实现Binder命令的解析和执行。`BpInterface` 是实现Proxy功能的模板，BpRefBase里有个mRemote对象指向一个BpBinder对象。

### 3.2 Binder 本地层的整个函数/方法调用过程

![Binder本地函数调用图](https://raw.githubusercontent.com/xdtianyu/SourceAnalysis/master/art/binder_native_stack.png)

1\. Java 层 [IRemoteService.Stub.Proxy](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L147) 调用 [android.os.IBinder (实现在 android.os.Binder.BinderProxy)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/Binder.java#L501) 的 `transact()` 发送 `Stub.TRANSACTION_addUser` 命令。

2\. 由 [BinderProxy.transact()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/Binder.java#L507) 进入 native 层。

3\. 由 [jni](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/jni/android_util_Binder.cpp#L1246) 转到 [android_os_BinderProxy_transact()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/jni/android_util_Binder.cpp#L1246) 函数。

4\. 调用 [IBinder->transact](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/jni/android_util_Binder.cpp#L1124) 函数。

```c++
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
        jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
{
    IBinder* target = (IBinder*)
        env->GetLongField(obj, gBinderProxyOffsets.mObject);
    status_t err = target->transact(code, *data, reply, flags);
}
```
而 `gBinderProxyOffsets.mObject` 则是在 java 层调用 `IBinder.getContextObject()` 时在 [javaObjectForIBinder](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/jni/android_util_Binder.cpp#L580) 函数中设置的

```c++
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);
}

jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
{
    ...
    LOGDEATH("objectForBinder %p: created new proxy %p !\n", val.get(), object);
    // The proxy holds a reference to the native object.
    env->SetLongField(object, gBinderProxyOffsets.mObject, (jlong)val.get());
    val->incStrong((void*)javaObjectForIBinder);
    ...
}
```
经过 [ProcessState::getContextObject()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/ProcessState.cpp#L85) 和 [ProcessState::getStrongProxyForHandle()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/ProcessState.cpp#L220)

```c++
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    return getStrongProxyForHandle(0);
}

sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;
    ...
    b = new BpBinder(handle); 
    result = b;
    ...
    return result;
}
```

可见 [android_os_BinderProxy_transact()]() 函数实际上调用的是 [BpBinder::transact()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/BpBinder.cpp#L159) 函数。

5\. [BpBinder::transact()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/BpBinder.cpp#L164) 则又调用了 [IPCThreadState::self()->transact()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L548) 函数。

```c++
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err = data.errorCheck();

    flags |= TF_ACCEPT_FDS;
    
    if (err == NO_ERROR) {
        LOG_ONEWAY(">>>> SEND from pid %d uid %d %s", getpid(), getuid(),
            (flags & TF_ONE_WAY) == 0 ? "READ REPLY" : "ONE WAY");
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }
    
    if ((flags & TF_ONE_WAY) == 0) {
        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
    } else {
        err = waitForResponse(NULL, NULL);
    }
    
    return err;
}

status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
    tr.target.handle = handle;
    tr.code = code;
    ...
    
    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));
    
    return NO_ERROR;
}
```

由函数内容可以看出， 数据再一次通过 [writeTransactionData()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L904) 传递给 [mOut](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L934) 进行写入操作。 `mOut` 是一个 Parcel 对象， 声明在 [IPCThreadState.h](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/include/binder/IPCThreadState.h#L123) 文件中。之后则调用 [waitForResponse()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L583) 函数。

6\. [IPCThreadState::waitForResponse()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L712) 在一个 `while` 循环里不断的调用 [talkWithDriver()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L803) 并检查是否有数据返回。

```c++
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        ...
        
        cmd = (uint32_t)mIn.readInt32();

        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:
            ...
        
        case BR_REPLY:
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));
                ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                if (err != NO_ERROR) goto finish;

                if (reply) {
                    if ((tr.flags & TF_STATUS_CODE) == 0) {
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer, this);
                    } else {
                        err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer);
                        freeBuffer(NULL,
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t), this);
                    }
                } else {
                    freeBuffer(NULL,
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t), this);
                    continue;
                }
            }
            goto finish;
        }
        
        default:
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }
    ...
}
```

7\. [IPCThreadState::talkWithDriver()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L803) 函数是真正与 binder 驱动交互的实现。[ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L856) 就是使用系统调用函数 `ioctl` 向 binder 设备文件 `/dev/binder` 发送 `BINDER_WRITE_READ` 命令。

```c++
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    if (mProcess->mDriverFD <= 0) {
        return -EBADF;
    }
    
    binder_write_read bwr;
    
    // Is the read buffer empty?
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
    
    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
    
    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }
    
    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    
#if defined(HAVE_ANDROID_OS)
        // 使用系统调用 ioctl 向 /dev/binder 发送 BINDER_WRITE_READ 命令
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
    
    do {
        if (mProcess->mDriverFD <= 0) {
            err = -EBADF;
        }
    } while (err == -EINTR);

    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < mOut.dataSize())
                mOut.remove(0, bwr.write_consumed);
            else
                mOut.setDataSize(0);
        }
        if (bwr.read_consumed > 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        return NO_ERROR;
    }
    
    return err;
}
```

经过 [IPCThreadState::talkWithDriver()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L718) ,就将数据发送给了 Binder 驱动。

继续追踪 [IPCThreadState::waitForResponse()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L723) ，可以从 第6步 发现 `IPCThreadState` 不断的循环读取 Binder 驱动返回，获取到返回命令后执行了 [executeCommand(cmd)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L787) 函数。

8\. [IPCThreadState::executeCommand()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L947) 处理 Binder 驱动返回命令

```c++
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;
    
    switch ((uint32_t)cmd) {
    ...
    
    case BR_TRANSACTION:
        {
            binder_transaction_data tr;
            result = mIn.read(&tr, sizeof(tr));
            ...
            Parcel buffer;
            buffer.ipcSetDataReference(
                reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                tr.data_size,
                reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                tr.offsets_size/sizeof(binder_size_t), freeBuffer, this);
            ...

            Parcel reply;
            status_t error;
            if (tr.target.ptr) {
                sp<BBinder> b((BBinder*)tr.cookie);
                error = b->transact(tr.code, buffer, &reply, tr.flags);

            } else {
                error = the_context_object->transact(tr.code, buffer, &reply, tr.flags);
            }
            ...
        }
        break;
    ...
}
```

9\. 可以看出其调用了 [BBinder::transact()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L1085#L1091) 函数，将数据返回给上层。

```c++
status_t BBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    data.setDataPosition(0);

    status_t err = NO_ERROR;
    switch (code) {
        case PING_TRANSACTION:
            reply->writeInt32(pingBinder());
            break;
        default:
            err = onTransact(code, data, reply, flags);
            break;
    }

    if (reply != NULL) {
        reply->setDataPosition(0);
    }

    return err;
}
```

10\. 而这里的 [b->transact(tr.code, buffer, &reply, tr.flags)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L1085#L1091) 中的 `b (BBinder)` 是 [JavaBBinder](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/jni/android_util_Binder.cpp#L217) 的实例，所以会调用 [JavaBBinder::onTransact()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/jni/android_util_Binder.cpp#L247) 函数

```c++
// frameworks/base/core/jni/android_util_Binder.cpp
virtual status_t onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0)
    {
        JNIEnv* env = javavm_to_jnienv(mVM);
        ...
        jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
            code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);
    }
    
static int int_register_android_os_Binder(JNIEnv* env)
{
    ...
    gBinderOffsets.mExecTransact = GetMethodIDOrDie(env, clazz, "execTransact", "(IJJI)Z");
    ...
}
```
11\. 可见 JNI 通过 [gBinderOffsets.mExecTransact](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/jni/android_util_Binder.cpp#L260) 最后执行了 `android.os.Binder` 的 [execTransact()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/jni/android_util_Binder.cpp#L865) 方法。

[execTransact()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/Binder.java#L442) 方法是 jni 回调的入口。

```java
// Entry point from android_util_Binder.cpp's onTransact
    private boolean execTransact(int code, long dataObj, long replyObj,
            int flags) {
        Parcel data = Parcel.obtain(dataObj);
        Parcel reply = Parcel.obtain(replyObj);
        ...
        try {
            res = onTransact(code, data, reply, flags);
        } 
        ...
    }
```

12\. 而我们则在服务端 [IRemoteService.Stub](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L19) 重载了 [onTransact()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L51) 方法，所以数据最后会回到我们的服务端并执行服务端实现的 `addUser()` 方法。

```java
public static abstract class Stub extends android.os.Binder
        implements org.xdty.remoteservice.IRemoteService {
    ...
    @Override
    public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply,
            int flags) throws android.os.RemoteException {
        switch (code) {
            case INTERFACE_TRANSACTION: {
                reply.writeString(DESCRIPTOR);
                return true;
            }
            case TRANSACTION_basicTypes: {
                ...
                return true;
            }
            case TRANSACTION_addUser: {
                data.enforceInterface(DESCRIPTOR);
                org.xdty.remoteservice.User _arg0;
                if ((0 != data.readInt())) {
                    _arg0 = org.xdty.remoteservice.User.CREATOR.createFromParcel(data);
                } else {
                    _arg0 = null;
                }
                this.addUser(_arg0);
                reply.writeNoException();
                return true;
            }
        }
        return super.onTransact(code, data, reply, flags);
    }
}
```

上述过程就是所有的 Native 层客户端到服务端的调用过程，总结下来就是 客户端进程发送 `BC_TRANSACTION` 到 Binder 驱动，服务端进程监听返回的 `BR_TRANSACTION` 命令并处理。如果是服务端向客户端返回数据，类似的是服务端发送 `BC_REPLY` 命令， 客户端监听 `BR_REPLY` 命令。

### 3.3 Binder 设备文件的打开和读写

**1. 设备的打开**

在上一小节中我们看到 JNI 过程中调用了 `ProcessState::getContextObject()` 函数， 在 [ProcessState](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/ProcessState.cpp#L340) 初始化时会打开 binder 设备

```c++
// ProcessState.cpp
ProcessState::ProcessState()
    : mDriverFD(open_driver())
    ...
{
    ...
}
```

[open_driver()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/ProcessState.cpp#L311#L337) 函数内容如下

```c++
// ProcessState.cpp
static int open_driver()
{
    // 打开设备文件
    int fd = open("/dev/binder", O_RDWR);
    if (fd >= 0) {
        fcntl(fd, F_SETFD, FD_CLOEXEC);
        int vers = 0;
        // 获取驱动版本
        status_t result = ioctl(fd, BINDER_VERSION, &vers);
        if (result == -1) {
            ALOGE("Binder ioctl to obtain version failed: %s", strerror(errno));
            close(fd);
            fd = -1;
        }
        // 检查驱动版本是否一致
        if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
            ALOGE("Binder driver protocol does not match user space protocol!");
            close(fd);
            fd = -1;
        }
        // 设置最多 15 个 binder 线程
        size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
        if (result == -1) {
            ALOGE("Binder ioctl to set max threads failed: %s", strerror(errno));
        }
    } else {
        ALOGW("Opening '/dev/binder' failed: %s\n", strerror(errno));
    }
    return fd;
}
```

**2. 设备的读写**

打开设备文件后，文件描述符被保存在 [mDriverFD](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/ProcessState.cpp#L340)， 通过系统调用 `ioctl` 函数操作 `mDriverFD` 就可以实现和 binder 驱动的交互。

对 Binder 设备文件的所有读写及关闭操作则都在 [IPCThreadState](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L805) 中，如上一小节提及到的 [IPCThreadState::talkWithDriver](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L803) 函数

`talkWithDriver()` 函数封装了 `BINDER_WRITE_READ` 命令，会从 binder 驱动读取或写入封装在 [binder_write_read](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L856) 结构体中的本地或远程对象。

```c++
// IPCThreadState.cpp
status_t IPCThreadState::talkWithDriver(bool doReceive)
{   
    binder_write_read bwr;
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
    
    // 写入数据
    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    // 读取数据
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }
    ...
    // 使用 ioctl 系统调用发送 BINDER_WRITE_READ 命令到 biner 驱动
    if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
        err = NO_ERROR;
    ...
}
```

可以看出，本地层是对应用与 binder 驱动交互的直接封装与实现，最终的数据传输仍是由驱动来完成的。本地层对底层驱动进行了完整的封装，上层应用只关心 transact() 和 onTransact() 回调，察觉不到 binder 驱动的存在，减轻了上层应用进程间通信开发的复杂度。

## 4. Binder 驱动

关于 binder 驱动建议参考另一篇文章 [深入分析Android Binder 驱动](http://blog.csdn.net/yangwen123/article/details/9316987) [原文]([Android Binder](https://web.archive.org/web/20101016004342/http://www.gmier.com/node/11)，本小节仍需要完善。

Binder 驱动是 Binder 的最终实现， ServiceManager 和 Client/Service 进程间通信最终都是由 Binder 驱动投递的。

![Binder reference](https://raw.githubusercontent.com/xdtianyu/SourceAnalysis/master/art/binder_reference.png)

Binder 驱动的代码位于 kernel 代码的 [drivers/staging/android](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/tree/master/drivers/staging/android) 目录下。主文件是 [binder.h](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h) 和 [binder.c](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c)

进程间传输的数据被称为 Binder 对象，它是一个 [flat_binder_object](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L49)，结构如下

```c
struct flat_binder_object {
    /* 8 bytes for large_flat_header. */
    unsigned long       type;
    unsigned long       flags;

    /* 8 bytes of data. */
    union {
        void        *binder;    /* local object */
        signed long handle;     /* remote object */
    };

    /* extra data associated with local object */
    void            *cookie;
};
```
其中 类型 [type](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L29) 描述了 Binder 对象的类型，包含 `BINDER`(本地对象)、`HANDLE`(远程对象)、 `FD` 三大类(五种)

```c
enum {
    BINDER_TYPE_BINDER  = B_PACK_CHARS('s', 'b', '*', B_TYPE_LARGE),
    BINDER_TYPE_WEAK_BINDER = B_PACK_CHARS('w', 'b', '*', B_TYPE_LARGE),
    BINDER_TYPE_HANDLE  = B_PACK_CHARS('s', 'h', '*', B_TYPE_LARGE),
    BINDER_TYPE_WEAK_HANDLE = B_PACK_CHARS('w', 'h', '*', B_TYPE_LARGE),
    BINDER_TYPE_FD      = B_PACK_CHARS('f', 'd', '*', B_TYPE_LARGE),
};
```
[flags](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L52) 则表述了[传输方式](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L110)，如异步、无返回等

```c
enum transaction_flags {
    TF_ONE_WAY  = 0x01, /* this is a one-way call: async, no return */
    TF_ROOT_OBJECT  = 0x04, /* contents are the component's root object */
    TF_STATUS_CODE  = 0x08, /* contents are a 32-bit status code */
    TF_ACCEPT_FDS   = 0x10, /* allow replies with file descriptors */
};
```

而 `flat_binder_object` 中的 [union 联合体](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L55) 就是要传输的数据，当类型为 `BINDER` 时， 数据就是一个本地对象 [*binder](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L56)，而类型为 `HANDLE` 时，数据则是一个远程对象 [handle](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L57)。

当 `flat_binder_object` 在进程间传递时， Binder 驱动会修改它的类型和数据，交换的代码参考 [binder_transaction](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L1671) 的实现。

该如何理解本地 `BINDER` 对象和远程 `HANDLE` 对象呢？其实它们都代表同一个对象，不过是从不同的角度来看。举例来说，假如进程 `RemoteService` 有个对象 [mBinder](https://github.com/xdtianyu/AidlExample/blob/master/remoteservice/src/main/java/org/xdty/remoteservice/RemoteService.java#L11)，对于 `RemoteService` 来说，`mBinder` 就是一个本地的 `BINDER` 对象；如果进程 `app` 通过 Binder 驱动访问 `RemoteService` 的 `mBinder` 对象，对于 `app` 来说， `mBinder` 就是一个 `HANDLE`。因此，从根本上来说 `handle` 和 `binder` 都指向 `RemoteService` 的 `mBinder`。本地对象还可以带有额外的数据，保存在 [cookie](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L61) 中。

Binder 驱动直接操作的最外层数据结构是 [binder_transaction_data](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L117)， Binder 对象 `flat_binder_object` 被封装在 [binder_transaction_data](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L142) 结构体中。

`binder_transaction_data` 数据结构才是真正传输的数据，其定义如下

```c
struct binder_transaction_data {
    /* The first two are only used for bcTRANSACTION and brTRANSACTION,
     * identifying the target and contents of the transaction.
     */
    union {
        size_t  handle; /* target descriptor of command transaction */
        void    *ptr;   /* target descriptor of return transaction */
    } target;
    void        *cookie;    /* target object cookie */
    unsigned int    code;       /* transaction command */

    /* General information about the transaction. */
    unsigned int    flags;
    pid_t       sender_pid;
    uid_t       sender_euid;
    size_t      data_size;  /* number of bytes of data */
    size_t      offsets_size;   /* number of bytes of offsets */

    /* If this transaction is inline, the data immediately
     * follows here; otherwise, it ends with a pointer to
     * the data buffer.
     */
    union {
        struct {
            /* transaction data */
            const void  *buffer;
            /* offsets from buffer to flat_binder_object structs */
            const void  *offsets;
        } ptr;
        uint8_t buf[8];
    } data;
};
```

`flat_binder_object` 就被封装在 [*buffer](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L142)中，其中的 [unsigned int   code;](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L126) 则是传输命令，描述了 Binder 对象执行的操作。


### 4.1 binder 设备的创建

[device_initcall()](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L3747) 函数是内核加载驱动的入口函数，我们先来看这个函数的调用过程。

```c
static struct miscdevice binder_miscdev = {
    .minor = MISC_DYNAMIC_MINOR,
    // 设备文件 /dev/binder
    .name = "binder",
    // 设备文件操作
    .fops = &binder_fops
};

static int __init binder_init(void)
{
    int ret;
    ...
    // 注册字符设备
    ret = misc_register(&binder_miscdev);
    ...
    // 调试文件， 在 /sys/kernel/debug/binder 目录下
    if (binder_debugfs_dir_entry_root) {
        debugfs_create_file("state",
                    S_IRUGO,
                    binder_debugfs_dir_entry_root,
                    NULL,
                    &binder_state_fops);
        ...
    }
    return ret;
}

device_initcall(binder_init);
```
可以看出 [binder_init()](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L3704) 使用 `misc_register()` 函数创建了 binder 设备。从 [misc_register(&binder_miscdev);](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L3716) 及 [.name = "binder"](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L3695) 可以看出， binder 向 kernel 注册了一个 `/dev/binder` 的字符设备，而文件操作都在 [binder_fops](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L3683) 结构体中定义。

```c
static const struct file_operations binder_fops = {
    .owner = THIS_MODULE,
    .poll = binder_poll,
    .unlocked_ioctl = binder_ioctl,
    .mmap = binder_mmap,
    .open = binder_open,
    .flush = binder_flush,
    .release = binder_release,
};
```
从上面 `binder_fops` 结构体可以看出，主要的操作是 `binder_ioctl()` `binder_mmap()` `binder_open()` 等函数实现的。

### 4.2 binder协议和数据结构

[binder.h](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h) 文件中定义了 binder 协议和重要的数据结构。

首先在 [enum](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L29) 中定义了 binder 处理的类型，引用或是句柄

```c
enum {
    BINDER_TYPE_BINDER  = B_PACK_CHARS('s', 'b', '*', B_TYPE_LARGE),
    BINDER_TYPE_WEAK_BINDER = B_PACK_CHARS('w', 'b', '*', B_TYPE_LARGE),
    BINDER_TYPE_HANDLE  = B_PACK_CHARS('s', 'h', '*', B_TYPE_LARGE),
    BINDER_TYPE_WEAK_HANDLE = B_PACK_CHARS('w', 'h', '*', B_TYPE_LARGE),
    BINDER_TYPE_FD      = B_PACK_CHARS('f', 'd', '*', B_TYPE_LARGE),
};
```
下面这段宏定义则是在 `ioctl` 函数调用时可用的具体命令。

```c
#define BINDER_WRITE_READ       _IOWR('b', 1, struct binder_write_read)
#define BINDER_SET_IDLE_TIMEOUT     _IOW('b', 3, int64_t)
#define BINDER_SET_MAX_THREADS      _IOW('b', 5, size_t)
#define BINDER_SET_IDLE_PRIORITY    _IOW('b', 6, int)
#define BINDER_SET_CONTEXT_MGR      _IOW('b', 7, int)
#define BINDER_THREAD_EXIT      _IOW('b', 8, int)
#define BINDER_VERSION          _IOWR('b', 9, struct binder_version)
```
在 [BinderDriverReturnProtocol](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L166) 和 [BinderDriverCommandProtocol](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L254) 中 则分别定义了 客户端调用 和 服务端 返回的命令。

### 4.3 binder 驱动文件操作

上文已经提到，所有的操作定义在 [binder_fops](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L3683) 结构体中，下面讲述这些操作。

**设备的打开 - binder_open() 函数**

用户空间在打开 `/dev/binder` 设备时，驱动会出发 [binder_open()](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L3004) 函数的响应。

```c
static int binder_open(struct inode *nodp, struct file *filp)
{
    struct binder_proc *proc;

    // 分配 binder_proc 数据结构内存
    proc = kzalloc(sizeof(*proc), GFP_KERNEL);
    if (proc == NULL)
        return -ENOMEM;
    
    // 增加当前线程/进程的引用计数并赋值给tsk
    get_task_struct(current);
    proc->tsk = current;
    // 初始化队列
    INIT_LIST_HEAD(&proc->todo);
    init_waitqueue_head(&proc->wait);
    proc->default_priority = task_nice(current);

    binder_lock(__func__);

    // 增加BINDER_STAT_PROC的对象计数
    binder_stats_created(BINDER_STAT_PROC);
    // 添加 proc_node 到 binder_procs 全局列表中，这样任何进程就可以访问到其他进程的 binder_proc 对象了
    hlist_add_head(&proc->proc_node, &binder_procs);
    // 保存进程 id
    proc->pid = current->group_leader->pid;
    INIT_LIST_HEAD(&proc->delivered_death);
    // 驱动文件 private_data 指向 proc
    filp->private_data = proc;

    binder_unlock(__func__);
    
    return 0;
}
```

**驱动文件释放 - binder_release() 函数**

在用户空间关闭驱动设备文件时，会调用 [binder_release()](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L3068) 函数，清理 binder_proc 对象，释放占用的内存。

```c
static int binder_release(struct inode *nodp, struct file *filp)
{
    struct binder_proc *proc = filp->private_data;
    binder_defer_work(proc, BINDER_DEFERRED_RELEASE);

    return 0;
}

static void
binder_defer_work(struct binder_proc *proc, enum binder_deferred_state defer)
{
    mutex_lock(&binder_deferred_lock);
    proc->deferred_work |= defer;
    if (hlist_unhashed(&proc->deferred_work_node)) {
        // 添加到释放队列中
        hlist_add_head(&proc->deferred_work_node,
                &binder_deferred_list);
        queue_work(binder_deferred_workqueue, &binder_deferred_work);
    }
    mutex_unlock(&binder_deferred_lock);
}
```

**内存映射 - binder_mmap() 函数**

[binder_mmap()](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L2905) 函数把设备内存映射到用户进程地址空间中，这样就可以像操作用户内存那样操作设备内存。

```c
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
    int ret;
    struct vm_struct *area;
    // 获得 binder_proc 对象
    struct binder_proc *proc = filp->private_data;
    const char *failure_string;
    struct binder_buffer *buffer;

    // 最多只分配 4M 的内存
    if ((vma->vm_end - vma->vm_start) > SZ_4M)
        vma->vm_end = vma->vm_start + SZ_4M;

    // 检查 flags
    if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {
        ret = -EPERM;
        failure_string = "bad vm_flags";
        goto err_bad_arg;
    }
    vma->vm_flags = (vma->vm_flags | VM_DONTCOPY) & ~VM_MAYWRITE;

    mutex_lock(&binder_mmap_lock);
    // 检查是否已经映射
    if (proc->buffer) {
        ret = -EBUSY;
        failure_string = "already mapped";
        goto err_already_mapped;
    }

    // 申请内核虚拟内存空间
    area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);
    if (area == NULL) {
        ret = -ENOMEM;
        failure_string = "get_vm_area";
        goto err_get_vm_area_failed;
    }
    // 将申请到的内存地址保存到 binder_proc 对象中
    proc->buffer = area->addr;
    proc->user_buffer_offset = vma->vm_start - (uintptr_t)proc->buffer;
    mutex_unlock(&binder_mmap_lock);

    // 根据请求到的内存空间大小，分配给 binder_proc 对象的 pages， 用于保存指向物理页的指针
    proc->pages = kzalloc(sizeof(proc->pages[0]) * ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL);
    if (proc->pages == NULL) {
        ret = -ENOMEM;
        failure_string = "alloc page array";
        goto err_alloc_pages_failed;
    }
    proc->buffer_size = vma->vm_end - vma->vm_start;

    vma->vm_ops = &binder_vm_ops;
    vma->vm_private_data = proc;

    // 分配一个页的物理内存
    if (binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma)) {
        ret = -ENOMEM;
        failure_string = "alloc small buf";
        goto err_alloc_small_buf_failed;
    }
    // 内存提供给 binder_buffer
    buffer = proc->buffer;
    // 初始化 proc->buffers 链表
    INIT_LIST_HEAD(&proc->buffers);
    // 将 binder_buffer 对象放入到 proc->buffers 链表中
    list_add(&buffer->entry, &proc->buffers);
    buffer->free = 1;
    binder_insert_free_buffer(proc, buffer);
    proc->free_async_space = proc->buffer_size / 2;
    barrier();
    proc->files = get_files_struct(proc->tsk);
    proc->vma = vma;
    proc->vma_vm_mm = vma->vm_mm;

    return 0;
}
```

**驱动命令接口 - binder_ioctl() 函数**

用户态程序调用 `ioctl` 系统函数向 `/dev/binder` 设备发送数据时，会触发 [binder_ioctl()](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L2734) 函数响应。

上文数据结构中已经提到了 `binder_ioctl` 可以处理的 [命令](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L87)

```c
// 核心命令，数据的读写
#define BINDER_WRITE_READ       _IOWR('b', 1, struct binder_write_read)
// 设置最大线程数
#define BINDER_SET_MAX_THREADS      _IOW('b', 5, size_t)
// 设置 context manager
#define BINDER_SET_CONTEXT_MGR      _IOW('b', 7, int)
// 线程退出命令
#define BINDER_THREAD_EXIT      _IOW('b', 8, int)
// binder 驱动的版本
#define BINDER_VERSION          _IOWR('b', 9, struct binder_version)
```

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int ret;
    struct binder_proc *proc = filp->private_data;
    struct binder_thread *thread;
    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;

    // 检查是否有错误
    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
    if (ret)
        goto err_unlocked;

    binder_lock(__func__);
    // 获取 binder_thread 对象
    thread = binder_get_thread(proc);
    if (thread == NULL) {
        ret = -ENOMEM;
        goto err;
    }

    switch (cmd) {
    case BINDER_WRITE_READ: {
        struct binder_write_read bwr;
        if (size != sizeof(struct binder_write_read)) {
            ret = -EINVAL;
            goto err;
        }
        // 从用户空间拷贝 binder_write_read 到 binder 驱动，储存在 bwr
        if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
            ret = -EFAULT;
            goto err;
        }

        if (bwr.write_size > 0) {
            // 执行写入操作
            ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
            if (ret < 0) {
                bwr.read_consumed = 0;
                if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                    ret = -EFAULT;
                goto err;
            }
        }
        if (bwr.read_size > 0) {
            // 执行读取操作
            ret = binder_thread_read(proc, thread, (void __user *)bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags & O_NONBLOCK);
            if (!list_empty(&proc->todo))
                wake_up_interruptible(&proc->wait);
            if (ret < 0) {
                if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                    ret = -EFAULT;
                goto err;
            }
        }
        // 操作完成后将数据返回给用户空间
        if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
            ret = -EFAULT;
            goto err;
        }
        break;
    }
    case BINDER_SET_MAX_THREADS:
        // 设置最大线程，从用户空间拷贝数据到 proc->max_threads
        if (copy_from_user(&proc->max_threads, ubuf, sizeof(proc->max_threads))) {
            ret = -EINVAL;
            goto err;
        }
        break;
    case BINDER_SET_CONTEXT_MGR:
        // 检查是否已经设置
        if (binder_context_mgr_node != NULL) {
            ret = -EBUSY;
            goto err;
        }
        // 设置 context manager
        ret = security_binder_set_context_mgr(proc->tsk);
        if (ret < 0)
            goto err;
        if (binder_context_mgr_uid != -1) {
            if (binder_context_mgr_uid != current->cred->euid) {
                ret = -EPERM;
                goto err;
            }
        } else
            binder_context_mgr_uid = current->cred->euid;
        // 创建 binder_context_mgr_node 节点
        binder_context_mgr_node = binder_new_node(proc, NULL, NULL);
        if (binder_context_mgr_node == NULL) {
            ret = -ENOMEM;
            goto err;
        }
        // 初始化节点数据
        binder_context_mgr_node->local_weak_refs++;
        binder_context_mgr_node->local_strong_refs++;
        binder_context_mgr_node->has_strong_ref = 1;
        binder_context_mgr_node->has_weak_ref = 1;
        break;
    case BINDER_THREAD_EXIT:
        // 线程退出，释放资源
        binder_free_thread(proc, thread);
        thread = NULL;
        break;
    case BINDER_VERSION:
        // 将 binder 驱动版本号写入到用户空间 ubuf->protocol_version 中
        if (put_user(BINDER_CURRENT_PROTOCOL_VERSION, &((struct binder_version *)ubuf)->protocol_version)) {
            ret = -EINVAL;
            goto err;
        }
        break;
    default:
        ret = -EINVAL;
        goto err;
    }
    ret = 0;
...
}
```

```c
static struct binder_node *binder_new_node(struct binder_proc *proc,
                       void __user *ptr,
                       void __user *cookie)
{
    struct rb_node **p = &proc->nodes.rb_node;
    struct rb_node *parent = NULL;
    struct binder_node *node;

    // 查找要插入节点的父节点
    while (*p) {
        parent = *p;
        node = rb_entry(parent, struct binder_node, rb_node);

        if (ptr < node->ptr)
            p = &(*p)->rb_left;
        else if (ptr > node->ptr)
            p = &(*p)->rb_right;
        else
            return NULL;
    }

    // 为要插入节点分配内存空间
    node = kzalloc(sizeof(*node), GFP_KERNEL);
    if (node == NULL)
        return NULL;
    binder_stats_created(BINDER_STAT_NODE);
    // 插入节点
    rb_link_node(&node->rb_node, parent, p);
    rb_insert_color(&node->rb_node, &proc->nodes);
    // 初始化
    node->debug_id = ++binder_last_id;
    node->proc = proc;
    node->ptr = ptr;
    node->cookie = cookie;
    node->work.type = BINDER_WORK_NODE;
    INIT_LIST_HEAD(&node->work.entry);
    INIT_LIST_HEAD(&node->async_todo);
    return node;
}
```

**BINDER_WRITE_READ 处理过程**

在 binder 本地层中，我们看到在 `IPCThreadState::talkWithDriver()` 函数中， binder 本地层通过 [ioctl()(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L856) 命令的形式，与 binder 驱动交互。

可以看出 `ioctl()` 的第三个参数是一个 [binder_write_read](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L69) 结构体

binder.h 头文件中定义了两个数据类型, 一个是 `binder_write_read`

```c
struct binder_write_read {
    signed long write_size; /* bytes to write */
    signed long write_consumed; /* bytes consumed by driver */
    unsigned long   write_buffer;
    signed long read_size;  /* bytes to read */
    signed long read_consumed;  /* bytes consumed by driver */
    unsigned long   read_buffer;
};
```
其中 `write_size` 和 `read_size` 表示需要被读写的字节数， `write_consumed` 和 `read_consumed` 表示已经被 binder 驱动读写的字节数， `write_buffer` 和 `read_buffer` 则是指向被读写数据的指针。

具体的读写操作被 [binder_thread_write](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L1852) 和 [binder_thread_read](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L2266) 实现。

**数据写入 - binder_thread_write() 函数**

将用户空间数据写入到 binder 驱动，从驱动角度来看是读取的操作。

```c
int binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,
            void __user *buffer, int size, signed long *consumed)
{
    uint32_t cmd;
    // 用户空间数据，起始地址和结束地址
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;

    // 循环读取
    while (ptr < end && thread->return_error == BR_OK) {
        // 从用户空间获取操作命令
        if (get_user(cmd, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
        if (_IOC_NR(cmd) < ARRAY_SIZE(binder_stats.bc)) {
            // 增加命令计数器
            binder_stats.bc[_IOC_NR(cmd)]++;
            proc->stats.bc[_IOC_NR(cmd)]++;
            thread->stats.bc[_IOC_NR(cmd)]++;
        }
        switch (cmd) {
        // 这四个命令用来增加或减少对象的引用计数， 操作目标 binder_ref
        case BC_INCREFS:
        case BC_ACQUIRE:
        case BC_RELEASE:
        case BC_DECREFS: {
            uint32_t target;
            struct binder_ref *ref;
            const char *debug_string;
            
            // 获取目标进程节点描述 desc
            if (get_user(target, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);
            // 索描述为 0 表示 context manager 进程
            if (target == 0 && binder_context_mgr_node &&
                (cmd == BC_INCREFS || cmd == BC_ACQUIRE)) {
                // 在 proc->refs_by_node.rb_node 红黑树中查找引用
                ref = binder_get_ref_for_node(proc,
                           binder_context_mgr_node);
            } else
                // 在 proc->refs_by_desc.rb_node 红黑树中查找引用
                ref = binder_get_ref(proc, target);
            switch (cmd) {
            case BC_INCREFS:
                debug_string = "IncRefs";
                // 增加弱引用计数
                binder_inc_ref(ref, 0, NULL);
                break;
            case BC_ACQUIRE:
                debug_string = "Acquire";
                // 增加强引用计数
                binder_inc_ref(ref, 1, NULL);
                break;
            case BC_RELEASE:
                debug_string = "Release";
                // 减少强引用计数
                binder_dec_ref(ref, 1);
                break;
            case BC_DECREFS:
            default:
                debug_string = "DecRefs";
                // 减少弱引用计数
                binder_dec_ref(ref, 0);
                break;
            }
            break;
        }
        case BC_INCREFS_DONE:
        case BC_ACQUIRE_DONE: {
            void __user *node_ptr;
            void *cookie;
            struct binder_node *node;
            
            // 从用户空间读取 node_ptr
            if (get_user(node_ptr, (void * __user *)ptr))
                return -EFAULT;
            ptr += sizeof(void *);
            // 从用户空间读取 cookie
            if (get_user(cookie, (void * __user *)ptr))
                return -EFAULT;
            ptr += sizeof(void *);
            // 获得节点
            node = binder_get_node(proc, node_ptr);
            // 没有找到则返回
            if (node == NULL) {
                binder_user_error("binder: %d:%d "
                    "%s u%p no match\n",
                    proc->pid, thread->pid,
                    cmd == BC_INCREFS_DONE ?
                    "BC_INCREFS_DONE" :
                    "BC_ACQUIRE_DONE",
                    node_ptr);
                break;
            }
            // cookie 不匹配则返回
            if (cookie != node->cookie) {
                binder_user_error("binder: %d:%d %s u%p node %d"
                    " cookie mismatch %p != %p\n",
                    proc->pid, thread->pid,
                    cmd == BC_INCREFS_DONE ?
                    "BC_INCREFS_DONE" : "BC_ACQUIRE_DONE",
                    node_ptr, node->debug_id,
                    cookie, node->cookie);
                break;
            }
            
            if (cmd == BC_ACQUIRE_DONE) {
                node->pending_strong_ref = 0;
            } else {
                node->pending_weak_ref = 0;
            }
            // 减少节点使用计数
            binder_dec_node(node, cmd == BC_ACQUIRE_DONE, 0);
            break;
        }

        // 释放 binder_bffer
        case BC_FREE_BUFFER: {
            void __user *data_ptr;
            struct binder_buffer *buffer;

            // 从用户空间获取 data_ptr
            if (get_user(data_ptr, (void * __user *)ptr))
                return -EFAULT;
            ptr += sizeof(void *);

            // 查找 binder_buffer
            buffer = binder_buffer_lookup(proc, data_ptr);
            // 没有找到则返回
            if (buffer == NULL) {
                binder_user_error("binder: %d:%d "
                    "BC_FREE_BUFFER u%p no match\n",
                    proc->pid, thread->pid, data_ptr);
                break;
            }
            // 不允许用户释放则返回
            if (!buffer->allow_user_free) {
                binder_user_error("binder: %d:%d "
                    "BC_FREE_BUFFER u%p matched "
                    "unreturned buffer\n",
                    proc->pid, thread->pid, data_ptr);
                break;
            }

            // 将 buffer->transaction 置空
            if (buffer->transaction) {
                buffer->transaction->buffer = NULL;
                buffer->transaction = NULL;
            }
            if (buffer->async_transaction && buffer->target_node) {
                if (list_empty(&buffer->target_node->async_todo))
                    buffer->target_node->has_async_transaction = 0;
                else
                    list_move_tail(buffer->target_node->async_todo.next, &thread->todo);
            }
            // 释放 binder_buffer 对象
            trace_binder_transaction_buffer_release(buffer);
            binder_transaction_buffer_release(proc, buffer, NULL);
            binder_free_buf(proc, buffer);
            break;
        }

        // binder 数据传递处理
        case BC_TRANSACTION:
        case BC_REPLY: {
            struct binder_transaction_data tr;

            // 从用户空间拷贝 binder_transaction_data 对象
            if (copy_from_user(&tr, ptr, sizeof(tr)))
                return -EFAULT;
            ptr += sizeof(tr);
            // 实际的传输函数，在下文讲解
            binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
            break;
        }

        // 设置 looper 为 BINDER_LOOPER_STATE_REGISTERED 状态
        case BC_REGISTER_LOOPER:
            if (thread->looper & BINDER_LOOPER_STATE_ENTERED) {
                thread->looper |= BINDER_LOOPER_STATE_INVALID;
                binder_user_error("binder: %d:%d ERROR:"
                    " BC_REGISTER_LOOPER called "
                    "after BC_ENTER_LOOPER\n",
                    proc->pid, thread->pid);
            } else if (proc->requested_threads == 0) {
                thread->looper |= BINDER_LOOPER_STATE_INVALID;
                binder_user_error("binder: %d:%d ERROR:"
                    " BC_REGISTER_LOOPER called "
                    "without request\n",
                    proc->pid, thread->pid);
            } else {
                proc->requested_threads--;
                proc->requested_threads_started++;
            }
            thread->looper |= BINDER_LOOPER_STATE_REGISTERED;
            break;
        // 设置 looper 为 BINDER_LOOPER_STATE_ENTERED 状态
        case BC_ENTER_LOOPER:
            if (thread->looper & BINDER_LOOPER_STATE_REGISTERED) {
                thread->looper |= BINDER_LOOPER_STATE_INVALID;
                binder_user_error("binder: %d:%d ERROR:"
                    " BC_ENTER_LOOPER called after "
                    "BC_REGISTER_LOOPER\n",
                    proc->pid, thread->pid);
            }
            thread->looper |= BINDER_LOOPER_STATE_ENTERED;
            break;
        // 设置 looper 为 BINDER_LOOPER_STATE_EXITED 状态
        case BC_EXIT_LOOPER:
            thread->looper |= BINDER_LOOPER_STATE_EXITED;
            break;

        // 发送 REQUEST_DEATH 或 CLEAR_DEATH 通知
        case BC_REQUEST_DEATH_NOTIFICATION:
        case BC_CLEAR_DEATH_NOTIFICATION: {
            uint32_t target;
            void __user *cookie;
            struct binder_ref *ref;
            struct binder_ref_death *death;
            
            // 从用户空间获取 binder_ref 描述 desc
            if (get_user(target, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);
            // 从用户空间获取 cookie
            if (get_user(cookie, (void __user * __user *)ptr))
                return -EFAULT;
            ptr += sizeof(void *);
            // 获取 binder_ref 引用
            ref = binder_get_ref(proc, target);
            if (ref == NULL) {
                binder_user_error("binder: %d:%d %s "
                    "invalid ref %d\n",
                    proc->pid, thread->pid,
                    cmd == BC_REQUEST_DEATH_NOTIFICATION ?
                    "BC_REQUEST_DEATH_NOTIFICATION" :
                    "BC_CLEAR_DEATH_NOTIFICATION",
                    target);
                break;
            }

            if (cmd == BC_REQUEST_DEATH_NOTIFICATION) {
                if (ref->death) {
                    binder_user_error("binder: %d:%"
                        "d BC_REQUEST_DEATH_NOTI"
                        "FICATION death notific"
                        "ation already set\n",
                        proc->pid, thread->pid);
                    break;
                }
                // 为 binder_ref_death 对象分配内存空间
                death = kzalloc(sizeof(*death), GFP_KERNEL);
                if (death == NULL) {
                    thread->return_error = BR_ERROR;
                    break;
                }
                // 初始化 binder_ref_death 对象
                binder_stats_created(BINDER_STAT_DEATH);
                INIT_LIST_HEAD(&death->work.entry);
                death->cookie = cookie;
                ref->death = death;
                if (ref->node->proc == NULL) {
                    ref->death->work.type = BINDER_WORK_DEAD_BINDER;
                    if (thread->looper & (BINDER_LOOPER_STATE_REGISTERED | BINDER_LOOPER_STATE_ENTERED)) {
                        list_add_tail(&ref->death->work.entry, &thread->todo);
                    } else {
                        list_add_tail(&ref->death->work.entry, &proc->todo);
                        // 唤醒目标进程
                        wake_up_interruptible(&proc->wait);
                    }
                }
            } else {
                if (ref->death == NULL) {
                    binder_user_error("binder: %d:%"
                        "d BC_CLEAR_DEATH_NOTIFI"
                        "CATION death notificat"
                        "ion not active\n",
                        proc->pid, thread->pid);
                    break;
                }
                death = ref->death;
                if (death->cookie != cookie) {
                    binder_user_error("binder: %d:%"
                        "d BC_CLEAR_DEATH_NOTIFI"
                        "CATION death notificat"
                        "ion cookie mismatch "
                        "%p != %p\n",
                        proc->pid, thread->pid,
                        death->cookie, cookie);
                    break;
                }
                // 将 ref->death 置空
                ref->death = NULL;
                if (list_empty(&death->work.entry)) {
                    death->work.type = BINDER_WORK_CLEAR_DEATH_NOTIFICATION;
                    if (thread->looper & (BINDER_LOOPER_STATE_REGISTERED | BINDER_LOOPER_STATE_ENTERED)) {
                        list_add_tail(&death->work.entry, &thread->todo);
                    } else {
                        list_add_tail(&death->work.entry, &proc->todo);
                        // 唤醒目标进程
                        wake_up_interruptible(&proc->wait);
                    }
                } else {
                    BUG_ON(death->work.type != BINDER_WORK_DEAD_BINDER);
                    death->work.type = BINDER_WORK_DEAD_BINDER_AND_CLEAR;
                }
            }
        } break;
        case BC_DEAD_BINDER_DONE: {
            struct binder_work *w;
            void __user *cookie;
            struct binder_ref_death *death = NULL;
            // 从用户空间获取 cookie
            if (get_user(cookie, (void __user * __user *)ptr))
                return -EFAULT;

            ptr += sizeof(void *);
            list_for_each_entry(w, &proc->delivered_death, entry) {
                struct binder_ref_death *tmp_death = container_of(w, struct binder_ref_death, work);
                if (tmp_death->cookie == cookie) {
                    death = tmp_death;
                    break;
                }
            }
            if (death == NULL) {
                binder_user_error("binder: %d:%d BC_DEAD"
                    "_BINDER_DONE %p not found\n",
                    proc->pid, thread->pid, cookie);
                break;
            }

            list_del_init(&death->work.entry);
            // 如果 death->work.t 为 BINDER_WORK_DEAD_BINDER_AND_CLEAR 则修改为 BINDER_WORK_CLEAR_DEATH_NOTIFICATION
            if (death->work.t == BINDER_WORK_DEAD_BINDER_AND_CLEAR ) {
                death->work.type = BINDER_WORK_CLEAR_DEATH_NOTIFICATION;
                if (thread->looper & (BINDER_LOOPER_STATE_REGISTERED | BINDER_LOOPER_STATE_ENTERED)) {
                    list_add_tail(&death->work.entry, &thread->todo);
                } else {
                    list_add_tail(&death->work.entry, &proc->todo);
                    // 唤醒目标进程
                    wake_up_interruptible(&proc->wait);
                }
            }
        } break;

        default:
            return -EINVAL;
        }
        *consumed = ptr - buffer;
    }
    return 0;
}
```

**binder_transaction() 函数**

在上文处理 [BC_TRANSACTION](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L2041) 和 [BC_REPLY](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L2042) 时，调用了 [binder_transaction()](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L1417) 函数。我们继续追踪

```c
static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,
                   struct binder_transaction_data *tr, int reply)
{
    struct binder_transaction *t;
    struct binder_work *tcomplete;
    size_t *offp, *off_end;
    struct binder_proc *target_proc;
    struct binder_thread *target_thread = NULL;
    struct binder_node *target_node = NULL;
    struct list_head *target_list;
    wait_queue_head_t *target_wait;
    struct binder_transaction *in_reply_to = NULL;
    
    if (reply) {
        // BC_REPLY 处理流程
        // 得到 binder_transaction 对象
        in_reply_to = thread->transaction_stack;
        if (in_reply_to == NULL) {
            return_error = BR_FAILED_REPLY;
            goto err_empty_call_stack;
        }
        binder_set_nice(in_reply_to->saved_priority);
        thread->transaction_stack = in_reply_to->to_parent;
        // 获取目标线程
        target_thread = in_reply_to->from;
        target_proc = target_thread->proc;
    } else {
        // BC_TRANSACTION 处理流程
        // 查找目标节点
        if (tr->target.handle) {
            struct binder_ref *ref;
            // 获取 binder_ref 对象
            ref = binder_get_ref(proc, tr->target.handle);
            target_node = ref->node;
        } else {
            // 索引为 0 则返回 context manager
            target_node = binder_context_mgr_node;
        }
        // 得到目标进程
        target_proc = target_node->proc;
        if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {
            struct binder_transaction *tmp;
            tmp = thread->transaction_stack;
            while (tmp) {
                if (tmp->from && tmp->from->proc == target_proc)
                    // 获得目标线程
                    target_thread = tmp->from;
                tmp = tmp->from_parent;
            }
        }
    }
    // 设置要处理的目标进程或目标线程任务
    if (target_thread) {
        target_list = &target_thread->todo;
        target_wait = &target_thread->wait;
    } else {
        target_list = &target_proc->todo;
        target_wait = &target_proc->wait;
    }

    // 为 binder_transaction 对象分配内存空间
    t = kzalloc(sizeof(*t), GFP_KERNEL);
    binder_stats_created(BINDER_STAT_TRANSACTION);

    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
    binder_stats_created(BINDER_STAT_TRANSACTION_COMPLETE);

    // 如果是同步传输(双向)，则将当前的 binder_thread 对象保存在 binder_transaction 对象的 from 中。
    if (!reply && !(tr->flags & TF_ONE_WAY))
        t->from = thread;
    else
        t->from = NULL;
    // 设置 binder_transaction 对象
    t->sender_euid = proc->tsk->cred->euid;
    t->to_proc = target_proc;
    t->to_thread = target_thread;
    t->code = tr->code;
    t->flags = tr->flags;
    t->priority = task_nice(current);

    // 为 binder_buffer 分配内存空间
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,
        tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));
    // 设置 binder_buffer
    t->buffer->allow_user_free = 0;
    t->buffer->debug_id = t->debug_id;
    t->buffer->transaction = t;
    t->buffer->target_node = target_node;
    if (target_node)
        binder_inc_node(target_node, 1, 0, NULL);

    offp = (size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));

    // 从用户空间拷贝数据到 binder_buffer
    if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {
        return_error = BR_FAILED_REPLY;
        goto err_copy_data_failed;
    }
    if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {
        return_error = BR_FAILED_REPLY;
        goto err_copy_data_failed;
    }

    off_end = (void *)offp + tr->offsets_size;
    for (; offp < off_end; offp++) {
        struct flat_binder_object *fp;
        // 为 flat_binder_object 赋值
        fp = (struct flat_binder_object *)(t->buffer->data + *offp);
        // 转换 binder 类型，如果是 BINDER 则转换为 HANDLE， 如果是 HANDLE 则转为 BANDLE
        switch (fp->type) {
        case BINDER_TYPE_BINDER:
        case BINDER_TYPE_WEAK_BINDER: {
            struct binder_ref *ref;
            // 获取 binder_node 节点
            struct binder_node *node = binder_get_node(proc, fp->binder);
            if (node == NULL) {
                node = binder_new_node(proc, fp->binder, fp->cookie);
                if (node == NULL) {
                    return_error = BR_FAILED_REPLY;
                    goto err_binder_new_node_failed;
                }
                node->min_priority = fp->flags & FLAT_BINDER_FLAG_PRIORITY_MASK;
                node->accept_fds = !!(fp->flags & FLAT_BINDER_FLAG_ACCEPTS_FDS);
            }
            if (fp->cookie != node->cookie) {
                goto err_binder_get_ref_for_node_failed;
            }
            // 获取 binder_ref 对象
            ref = binder_get_ref_for_node(target_proc, node);
            // 转换类型
            if (fp->type == BINDER_TYPE_BINDER)
                fp->type = BINDER_TYPE_HANDLE;
            else
                fp->type = BINDER_TYPE_WEAK_HANDLE;
            fp->handle = ref->desc;
            binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE,
                       &thread->todo);
        } break;
        case BINDER_TYPE_HANDLE:
        case BINDER_TYPE_WEAK_HANDLE: {
            // 获取 binder_ref 对象
            struct binder_ref*ref = binder_get_ref(proc, fp->handle);
            // 转换类型
            if (ref->node->proc == target_proc) {
                if (fp->type == BINDER_TYPE_HANDLE)
                    fp->type = BINDER_TYPE_BINDER;
                else
                    fp->type = BINDER_TYPE_WEAK_BINDER;
                fp->binder = ref->node->ptr;
                fp->cookie = ref->node->cookie;
                binder_inc_node(ref->node, fp->type == BINDER_TYPE_BINDER, 0, NULL);
            } else {
                struct binder_ref *new_ref;
                new_ref = binder_get_ref_for_node(target_proc, ref->node);
                fp->handle = new_ref->desc;
                binder_inc_ref(new_ref, fp->type == BINDER_TYPE_HANDLE, NULL);
            }
        } break;

        // 文件类型
        case BINDER_TYPE_FD: {
            int target_fd;
            struct file *file;
            // 获得文件对象
            file = fget(fp->handle);
            // 分配一个新的文件描述符
            target_fd = task_get_unused_fd_flags(target_proc, O_CLOEXEC);
            task_fd_install(target_proc, target_fd, file);
            fp->handle = target_fd;
        } break;

        default:
            return_error = BR_FAILED_REPLY;
            goto err_bad_object_type;
        }
    }
    if (reply) {
        // BC_REPLY 处理流程, binder_transaction 中释放 binder_transaction 对象
        binder_pop_transaction(target_thread, in_reply_to);
    } else if (!(t->flags & TF_ONE_WAY)) {
        // 同步状态(双向)需要设置回复
        t->need_reply = 1;
        t->from_parent = thread->transaction_stack;
        thread->transaction_stack = t;
    } else {
        // 异步传输不需要设置回复
        if (target_node->has_async_transaction) {
            target_list = &target_node->async_todo;
            target_wait = NULL;
        } else
            target_node->has_async_transaction = 1;
    }
    t->work.type = BINDER_WORK_TRANSACTION;
    list_add_tail(&t->work.entry, target_list);
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    list_add_tail(&tcomplete->entry, &thread->todo);
    if (target_wait)
        // 唤醒目标线程
        wake_up_interruptible(target_wait);
    return;
}
```

**数据读取 - binder_thread_read() 函数**

用户空间从 binder 驱动读取数据，从驱动角度来看是写出的操作。

```c
static int binder_thread_read(struct binder_proc *proc,
                  struct binder_thread *thread,
                  void  __user *buffer, int size,
                  signed long *consumed, int non_block)
{
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;

    int ret = 0;
    int wait_for_proc_work;

    if (*consumed == 0) {
        // 第一次操作时向用户空间返回 BR_NOOP 命令
        if (put_user(BR_NOOP, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
    }

retry:
    // 获取将要处理的任务
    wait_for_proc_work = thread->transaction_stack == NULL &&
                list_empty(&thread->todo);
    if (wait_for_proc_work) {
        if (!(thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
                    BINDER_LOOPER_STATE_ENTERED))) {
            binder_user_error("binder: %d:%d ERROR: Thread waiting "
                "for process work before calling BC_REGISTER_"
                "LOOPER or BC_ENTER_LOOPER (state %x)\n",
                proc->pid, thread->pid, thread->looper);
            wait_event_interruptible(binder_user_error_wait,
                         binder_stop_on_user_error < 2);
        }
        binder_set_nice(proc->default_priority);
        if (non_block) {
            // 非阻塞且没有数据则返回 EAGAIN
            if (!binder_has_proc_work(proc, thread))
                ret = -EAGAIN;
        } else
            // 阻塞则进入睡眠状态，等待可操作的任务
            ret = wait_event_freezable_exclusive(proc->wait, binder_has_proc_work(proc, thread));
    } else {
        if (non_block) {
            if (!binder_has_thread_work(thread))
                ret = -EAGAIN;
        } else
            ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
    }

    binder_lock(__func__);

    if (wait_for_proc_work)
        proc->ready_threads--;
    thread->looper &= ~BINDER_LOOPER_STATE_WAITING;

    if (ret)
        return ret;

    while (1) {
        uint32_t cmd;
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;

        // 获取 binder_work 对象
        if (!list_empty(&thread->todo))
            w = list_first_entry(&thread->todo, struct binder_work, entry);
        else if (!list_empty(&proc->todo) && wait_for_proc_work)
            w = list_first_entry(&proc->todo, struct binder_work, entry);
        else {
            if (ptr - buffer == 4 && !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN)) /* no data added */
                goto retry;
            break;
        }

        if (end - ptr < sizeof(tr) + 4)
            break;

        switch (w->type) {
        case BINDER_WORK_TRANSACTION: {
            // 获取 binder_transaction 对象
            t = container_of(w, struct binder_transaction, work);
        } break;
        case BINDER_WORK_TRANSACTION_COMPLETE: {
            cmd = BR_TRANSACTION_COMPLETE;
            // 返回 BR_TRANSACTION_COMPLETE 命令
            if (put_user(cmd, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);

            binder_stat_br(proc, thread, cmd);

            // 从 work 链表中删除并释放内存
            list_del(&w->entry);
            kfree(w);
            binder_stats_deleted(BINDER_STAT_TRANSACTION_COMPLETE);
        } break;
        case BINDER_WORK_NODE: {
            // 获得 binder_node 节点
            struct binder_node *node = container_of(w, struct binder_node, work);
            uint32_t cmd = BR_NOOP;
            const char *cmd_name;
            // 根据节点类型，增加/获取、减少/释放节点索引
            int strong = node->internal_strong_refs || node->local_strong_refs;
            int weak = !hlist_empty(&node->refs) || node->local_weak_refs || strong;
            // 构造 BR_* 命令
            if (weak && !node->has_weak_ref) {
                cmd = BR_INCREFS;
                cmd_name = "BR_INCREFS";
                node->has_weak_ref = 1;
                node->pending_weak_ref = 1;
                node->local_weak_refs++;
            } else if (strong && !node->has_strong_ref) {
                cmd = BR_ACQUIRE;
                cmd_name = "BR_ACQUIRE";
                node->has_strong_ref = 1;
                node->pending_strong_ref = 1;
                node->local_strong_refs++;
            } else if (!strong && node->has_strong_ref) {
                cmd = BR_RELEASE;
                cmd_name = "BR_RELEASE";
                node->has_strong_ref = 0;
            } else if (!weak && node->has_weak_ref) {
                cmd = BR_DECREFS;
                cmd_name = "BR_DECREFS";
                node->has_weak_ref = 0;
            }
            // 向用户空间返回命令
            if (cmd != BR_NOOP) {
                if (put_user(cmd, (uint32_t __user *)ptr))
                    return -EFAULT;
                ptr += sizeof(uint32_t);
                if (put_user(node->ptr, (void * __user *)ptr))
                    return -EFAULT;
                ptr += sizeof(void *);
                if (put_user(node->cookie, (void * __user *)ptr))
                    return -EFAULT;
                ptr += sizeof(void *);

                binder_stat_br(proc, thread, cmd);
            } else {
                list_del_init(&w->entry);
                if (!weak && !strong) {
                    rb_erase(&node->rb_node, &proc->nodes);
                    kfree(node);
                    binder_stats_deleted(BINDER_STAT_NODE);
                }
            }
        } break;
        case BINDER_WORK_DEAD_BINDER:
        case BINDER_WORK_DEAD_BINDER_AND_CLEAR:
        case BINDER_WORK_CLEAR_DEATH_NOTIFICATION: {
            struct binder_ref_death *death;
            uint32_t cmd;

            // 获取 binder_ref_death 对象
            death = container_of(w, struct binder_ref_death, work);
            // 构造返回命令
            if (w->type == BINDER_WORK_CLEAR_DEATH_NOTIFICATION)
                cmd = BR_CLEAR_DEATH_NOTIFICATION_DONE;
            else
                cmd = BR_DEAD_BINDER;
            // 向用户空间返回命令
            if (put_user(cmd, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);
            // 将 cookie 返回给用户空间
            if (put_user(death->cookie, (void * __user *)ptr))
                return -EFAULT;
            ptr += sizeof(void *);
            binder_stat_br(proc, thread, cmd);

            if (w->type == BINDER_WORK_CLEAR_DEATH_NOTIFICATION) {
                list_del(&w->entry);
                kfree(death);
                binder_stats_deleted(BINDER_STAT_DEATH);
            } else
                list_move(&w->entry, &proc->delivered_death);
            if (cmd == BR_DEAD_BINDER)
                goto done; /* DEAD_BINDER notifications can cause transactions */
        } break;
        }

        if (!t)
            continue;

        if (t->buffer->target_node) {
            // 获得 binder_node 节点
            struct binder_node *target_node = t->buffer->target_node;
            // 将数据封装到 binder_transaction_data 对象
            tr.target.ptr = target_node->ptr;
            tr.cookie =  target_node->cookie;
            t->saved_priority = task_nice(current);
            if (t->priority < target_node->min_priority &&
                !(t->flags & TF_ONE_WAY))
                binder_set_nice(t->priority);
            else if (!(t->flags & TF_ONE_WAY) ||
                 t->saved_priority > target_node->min_priority)
                binder_set_nice(target_node->min_priority);
            // 设置返回的命令类型
            cmd = BR_TRANSACTION;
        } else {
            tr.target.ptr = NULL;
            tr.cookie = NULL;
            cmd = BR_REPLY;
        }
        tr.code = t->code;
        tr.flags = t->flags;
        tr.sender_euid = t->sender_euid;

        if (t->from) {
            struct task_struct *sender = t->from->proc->tsk;
            tr.sender_pid = task_tgid_nr_ns(sender,
                            current->nsproxy->pid_ns);
        } else {
            tr.sender_pid = 0;
        }

        tr.data_size = t->buffer->data_size;
        tr.offsets_size = t->buffer->offsets_size;
        tr.data.ptr.buffer = (void *)t->buffer->data +
                    proc->user_buffer_offset;
        tr.data.ptr.offsets = tr.data.ptr.buffer +
                    ALIGN(t->buffer->data_size,
                        sizeof(void *));

        if (put_user(cmd, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
        // 拷贝 binder_transaction_data 对象到用户空间
        if (copy_to_user(ptr, &tr, sizeof(tr)))
            return -EFAULT;
        ptr += sizeof(tr);

        binder_stat_br(proc, thread, cmd);

        // 移除 binder_transaction 并释放空间
        list_del(&t->work.entry);
        t->buffer->allow_user_free = 1;
        // 如果是同步操作，则将 thread 对象保存在 binder_transaction 中，返回给发送方进程, 否则释放 binder_transaction 对象
        if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {
            t->to_parent = thread->transaction_stack;
            t->to_thread = thread;
            thread->transaction_stack = t;
        } else {
            t->buffer->transaction = NULL;
            kfree(t);
            binder_stats_deleted(BINDER_STAT_TRANSACTION);
        }
        break;
    }
}
```
从上述代码可以看出 binder 驱动的具体实现，以及是如何发送和接收数据的。

## 5. Binder 与系统服务

### 5.1 Context.getSystemService()

Android 系统在启动后会在后台运行很多系统服务提供给应用使用，这些 [服务](http://developer.android.com/reference/android/content/Context.html#getSystemService(java.lang.Class<T>)) 主要有 `WindowManager, LayoutInflater, ActivityManager, PowerManager, AlarmManager, NotificationManager, KeyguardManager, LocationManager, SearchManager, Vibrator, ConnectivityManager, WifiManager, AudioManager, MediaRouter, TelephonyManager, SubscriptionManager, InputMethodManager, UiModeManager, DownloadManager, BatteryManager, JobScheduler, NetworkStatsManager`

我们可以通过 `Context.getSystemService(String name)` 来获取 [服务](http://developer.android.com/reference/android/content/Context.html#getSystemService(java.lang.String))。

例如 可以通过如下方法从 xml 中插入新的视图

    LayoutInflater inflater = (LayoutInflater) getSystemService(LAYOUT_INFLATER_SERVICE);
    inflater.inflate(R.layout.view, root, true);
    
### 5.2 Context.getSystemService() 源码分析

追踪 [ContextImpl](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ContextImpl.java#L1364) `getSystemService()` 源代码

```java
    @Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
```

继续追踪 [SystemServiceRegistry](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/SystemServiceRegistry.java#L719) 源代码

```java
    /**
     * Gets a system service from a given context.
     */
    public static Object getSystemService(ContextImpl ctx, String name) {
        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
        return fetcher != null ? fetcher.getService(ctx) : null;
    }
```

追踪 `SYSTEM_SERVICE_FETCHERS` 可以发现在 [SystemServiceRegistry](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/SystemServiceRegistry.java#L318) 静态区中注册了几乎所有的系统服务

```java
    registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
            new CachedServiceFetcher<LayoutInflater>() {
        @Override
        public LayoutInflater createService(ContextImpl ctx) {
            return new PhoneLayoutInflater(ctx.getOuterContext());
        }});

    registerService(Context.LOCATION_SERVICE, LocationManager.class,
            new CachedServiceFetcher<LocationManager>() {
        @Override
        public LocationManager createService(ContextImpl ctx) {
            IBinder b = ServiceManager.getService(Context.LOCATION_SERVICE);
            return new LocationManager(ctx, ILocationManager.Stub.asInterface(b));
        }});
```

上面代码片断中，[PhoneLayoutInflater](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/SystemServiceRegistry.java#L322) 最终回到了 [LayoutInflater](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/view/LayoutInflater.java#L204)。而 [LocationManager](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/location/java/android/location/LocationManager.java#L315) 则是对 [ILocationManager](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/location/java/android/location/ILocationManager.aidl#L39) 的封装。可以发现，在 [frameworks/base/location/java/android/location](https://github.com/xdtianyu/android-6.0.0_r1/tree/master/frameworks/base/location/java/android/location) 包下含有大量的 AIDL 文件。

继续追踪 [ServiceManager.getService(Context.LOCATION_SERVICE)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/ServiceManager.java#L49) 

```java
    private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }

        // Find the service manager
        sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
        return sServiceManager;
    }

    /**
     * Returns a reference to a service with the given name.
     * 
     * @param name the name of the service to get
     * @return a reference to the service, or <code>null</code> if the service doesn't exist
     */
    public static IBinder getService(String name) {
        try {
            IBinder service = sCache.get(name);
            if (service != null) {
                return service;
            } else {
                return getIServiceManager().getService(name);
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }
```
    
从上面代码片断可以看出，`ServiceManager` 会从 `sCache` 缓存或 [IServiceManager](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/ServiceManagerNative.java#L33) 中查找服务并返回一个 [IBinder](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/IBinder.java#L85) 对象。这个 `IBinder` 就是一个远程对象，可以通过它与其他进程交互。 

继续深入 [getIServiceManager().getService(name)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/ServiceManager.java#L55) , 进入 [ServiceManagerNative](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/ServiceManagerNative.java#L33) 

```java
    /**
     * Cast a Binder object into a service manager interface, generating
     * a proxy if needed.
     */
    static public IServiceManager asInterface(IBinder obj)
    {
        if (obj == null) {
            return null;
        }
        IServiceManager in =
            (IServiceManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        
        return new ServiceManagerProxy(obj);
    }
    
    class ServiceManagerProxy implements IServiceManager {
        public ServiceManagerProxy(IBinder remote) {
            mRemote = remote;
        }
        
        public IBinder asBinder() {
            return mRemote;
        }
        
        public IBinder getService(String name) throws RemoteException {
            Parcel data = Parcel.obtain();
            Parcel reply = Parcel.obtain();
            data.writeInterfaceToken(IServiceManager.descriptor);
            data.writeString(name);
            mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
            IBinder binder = reply.readStrongBinder();
            reply.recycle();
            data.recycle();
            return binder;
        }
        
        private IBinder mRemote;
    }
```

从上边代码片断可以看到，`ServiceManager.getIServiceManager()` 返回的是一个 `ServiceManagerProxy`, 而 `ServiceManager.getService()` 则是在 `ServiceManagerProxy` 中通过 `ServiceManager` 的远程 `Binder` 对象 `mRemote`，操作 `Parcel` 数据，调用 [IBinder.transact(int code, Parcel data, Parcel reply, int flags)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/IBinder.java#L223) 方法来发送请求，并通过 `reply.readStrongBinder()` 返回了要查找的服务的远程对象。

可以看到，系统服务的获取方式也是通过 AIDL 的方式实现的。

## 6. 结论

1\. AIDL 本质上只是一个用于封装 Binder 操作的工具，最终的进程间通信由 Binder 的 `transact` 和 `onTransact` 完成。


## 7. 参考

[Android Binder机制](http://wangkuiwu.github.io/2014/09/01/Binder-Introduce/)

[Android进程间通信（IPC）机制Binder](http://blog.csdn.net/luoshengyang/article/details/6618363)

[Android Binder](https://www.nds.rub.de/media/attachments/files/2012/03/binder.pdf)

[Android Architecture Binder](http://rts.lab.asu.edu/web_438/project_final/CSE_598_Android_Architecture_Binder.pdf)

[AIDL与Binder框架浅谈](http://liuxiangtian.github.io/2016/01/07/AIDL%E4%B8%8EBinder%E6%A1%86%E6%9E%B6%E6%B5%85%E8%B0%88/)

[Binder框架解析](https://github.com/hehonghui/android-tech-frontier/blob/master/issue-22/Binder%E6%A1%86%E6%9E%B6%E8%A7%A3%E6%9E%90.md)

[Deep Dive into Android IPC/Binder Framework at Android Builders Summit 2013](http://events.linuxfoundation.org/images/stories/slides/abs2013_gargentas.pdf)

[Android Builders Summit 2013 - Deep Dive into Android IPC/Binder Framework (video)](https://www.youtube.com/watch?v=NWhyADzgoiI)

[Binder源码分析之驱动层（原）](http://blog.csdn.net/u010961631/article/details/20479507)

[深入分析Android Binder 驱动](http://blog.csdn.net/yangwen123/article/details/9316987)

[构造IOCTL命令的学习心得-----_IO, _IOR, _IOW, _IOWR 幻数的理解](http://blog.csdn.net/qq429205464/article/details/7822442)

[Service与Android系统设计（7）--- Binder驱动](http://blog.csdn.net/21cnbao/article/details/8087354)

[Android Binder](https://web.archive.org/web/20101016004342/http://www.gmier.com/node/11)

[Binder机制，从Java到C （7. Native Service）](http://www.cnblogs.com/zhangxinyan/p/3487889.html)


----------------------------

**待补充的内容**

1\. 客户端 bindService() 流程及源码分析

2\. Binder Native 层其他源码文件分析

3\. 系统服务（SystemService）详细列表及在本地层的源码分析

4\. SystemManager 源码分析

5\. 完善 binder 驱动内容，补充关系图