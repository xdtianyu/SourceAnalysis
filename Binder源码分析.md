Binder 源码分析 【进行中】
===================

本文是基于 [Android 6.0.0](https://github.com/xdtianyu/android-6.0.0_r1) 和 [kernel 3.4](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow) 源码 及 Android SDK 23 展开的。~~所有的源文件都在 GitHub 托管，在文中可以点击链接查看完整的代码。~~

目录
-----

* [简介](#简介)
* [Binder框架及Native层](#Binder框架及Native层)
* [Binder驱动](#Binder驱动)
* [Binder与AIDL](#Binder与AIDL)
* [Binder与系统服务](#Binder与系统服务)
* [Binder源码解析](#Binder源码解析)
* [Android进程间通信的其他方式](#Android进程间通信的其他方式)
* [背景](#背景)
* [参考](#参考)

简介
-----

Binder 是一种 Android 进程间通信机制，提供远程过程调用(Remote Procedure Call)功能。~~我们最直接的使用应该是在调用 `Context.getSystemService()` 来获取系统服务，或直接使用 `AIDL` 来实现多个程序(APP)间数据交互。~~

Binder 是非常重要的 Android 基础组件，几乎所有的进程间通信都是使用 Binder 机制实现的。本文将结合源码展开讲述 Binder ，同时对一些重要知识点提供扩展阅读的参考。

![android_binder](https://raw.githubusercontent.com/xdtianyu/SourceAnalysis/master/art/android_binder.png)

不管是 Android 系统服务(System services)还是用户的应用进程(User apps)，最终都会通过 binder 来实现进程间通信。上次应用首先通过 IBinder 的 transcate 方法发送命令给 libbinder， libbinder 再通过系统调用(ioctl) 发送命令到内核中的 binder 驱动，之后再由驱动完成进程间数据的交互。

我们经常使用的 Intent，Messager 数据传递也是对 Binder 更高层次的抽象和封装，最终还是会由内核中的 binder 驱动完成数据的传递。

Binder框架及Native层
-----

Binder机制使本地对象可以像操作当前对象一样调用远程对象，可以使不同的进程间互相通信。Binder 使用 Client/Server 架构，客户端通过服务端代理，经过 Binder 驱动与服务端交互。

![Binder框架图片](https://raw.githubusercontent.com/xdtianyu/SourceAnalysis/master/art/Binder.png)

Binder 机制实现进程间通信的奥秘在于 kernel 中的 Binder 驱动，将在 [Binder驱动](#Binder驱动) 小节详细讲述。

![Binder本地框架图片](https://raw.githubusercontent.com/xdtianyu/SourceAnalysis/master/art/binder_native.png)

JNI 的代码位于 [frameworks/base/core/jni](https://github.com/xdtianyu/android-6.0.0_r1/tree/master/frameworks/base/core/jni) 目录下，主要是 [android_util_Binder.cpp](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/jni/android_util_Binder.cpp) 文件和头文件 [android_util_Binder.h](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/jni/android_util_Binder.h)

Binder JNI 代码是 Binder Java 层操作到 Binder Native 层的接口封装，最后会被编译进 `libandroid_runtime.so` 系统库。

Binder 本地层的代码在 [frameworks/native/libs/binder](https://github.com/xdtianyu/android-6.0.0_r1/tree/master/frameworks/native/libs/binder) 目录下， 此目录在 Android 系统编译后会生成 `libbinder.so` 文件，供 JNI 调用。`libbinder` 封装了所有对 binder 驱动的操作，是上层应用与驱动交互的桥梁。头文件则在 [frameworks/native/include/binder](https://github.com/xdtianyu/android-6.0.0_r1/tree/master/frameworks/native/include/binder) 目录下。

Binder 本地层目录下的几个重要文件：

**IInterface.cpp**

[IInterface.cpp](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IInterface.cpp#L33) 是 Binder 本地层入口，与 java 层的 `android.os.IInterface` 对应，提供 `asBinder()` 的实现，返回 `IBinder` 对象。

在头文件中有两个类 [BnInterface (Binder Native Interface)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/include/binder/IInterface.h#L50) 和 [BpInterface (Binder Proxy Interface)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/include/binder/IInterface.h#L63), 对应于 java 层的 [Stub](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L19) 和 [Proxy](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L92)

```
sp<IBinder> IInterface::asBinder(const IInterface* iface)
{
    if (iface == NULL) return NULL;
    return const_cast<IInterface*>(iface)->onAsBinder();
}
```

```
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

Binder 本地层的整个函数/方法调用过程

![Binder本地函数调用图](https://raw.githubusercontent.com/xdtianyu/SourceAnalysis/master/art/binder_native_stack.png)

1\. Java 层 [IRemoteService.Stub.Proxy](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L147) 调用 [android.os.IBinder (实现在 android.os.Binder.BinderProxy)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/Binder.java#L501) 的 `transact()` 发送 `Stub.TRANSACTION_addUser` 命令。

2\. 由 [BinderProxy.transact()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/Binder.java#L507) 进入 native 层。

3\. 由 [jni](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/jni/android_util_Binder.cpp#L1246) 转到 [android_os_BinderProxy_transact()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/jni/android_util_Binder.cpp#L1246) 函数。

4\. 调用 [IBinder->transact](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/jni/android_util_Binder.cpp#L1124) 函数。

```
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
        jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
{
    IBinder* target = (IBinder*)
        env->GetLongField(obj, gBinderProxyOffsets.mObject);
    status_t err = target->transact(code, *data, reply, flags);
}
```
而 `gBinderProxyOffsets.mObject` 则是在 java 层调用 `IBinder.getContextObject()` 时在 [javaObjectForIBinder](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/jni/android_util_Binder.cpp#L580) 函数中设置的

```
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

```
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

```
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

```
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
    }
    ...
}
```

7\. [IPCThreadState::talkWithDriver](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L803) 函数是真正与 binder 驱动交互的实现。[ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/IPCThreadState.cpp#L856) 就是使用系统调用函数 `ioctl` 向 binder 设备文件 `/dev/binder` 发送 `BINDER_WRITE_READ` 命令。这样就将数据发送给了 Binder 驱动。

```
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

**BpBinder.cpp**

`BpBinder(Base proxy Binder)` 对应于 Java 层的 `Service Proxy`,

先查看头文件 [BpBinder.h](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/include/binder/BpBinder.h) 代码片断

```
class BpBinder : public IBinder
{
public:

    inline  int32_t     handle() const { return mHandle; }

    virtual status_t    transact(   uint32_t code,
                                    const Parcel& data,
                                    Parcel* reply,
                                    uint32_t flags = 0);

    virtual status_t    linkToDeath(const sp<DeathRecipient>& recipient,
                                    void* cookie = NULL,
                                    uint32_t flags = 0);
    virtual status_t    unlinkToDeath(  const wp<DeathRecipient>& recipient,
                                        void* cookie = NULL,
                                        uint32_t flags = 0,
                                        wp<DeathRecipient>* outRecipient = NULL);
};
```

可以看到 [BpBinder](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/include/binder/BpBinder.h#L27) 中声明了 `transact()` `linkToDeath()` 等重要函数。再看具体实现

```
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    ...
    status_t status = IPCThreadState::self()->transact(
        mHandle, code, data, reply, flags);
    ...

    return DEAD_OBJECT;
}

status_t BpBinder::linkToDeath(
    const sp<DeathRecipient>& recipient, void* cookie, uint32_t flags)
{
    ...
    IPCThreadState* self = IPCThreadState::self();
                self->requestDeathNotification(mHandle, this);
                self->flushCommands();
    ...
    return DEAD_OBJECT;
}
```

可以看出 BPBinder 是最终是通过调用 `IPCThreadState` 的函数来完成数据传递操作。


**IPCThreadState.cpp**


**AppOpsManager.cpp**

[APPOpsManager (APP Operation Manager)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/native/libs/binder/AppOpsManager.cpp) 是 应用操作管理者，实现对客户端操作的检查、启动、完成等。


Binder驱动
-----

Binder 驱动是 Binder 的最终实现， ServiceManager 和 Client/Service 进程间通信最终都是由 Binder 驱动投递的。

![Binder reference](https://raw.githubusercontent.com/xdtianyu/SourceAnalysis/master/art/binder_reference.png)

Binder 驱动的代码位于 kernel 代码的 [drivers/staging/android](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/tree/master/drivers/staging/android) 目录下。主文件是 [binder.h](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h) 和 [binder.c](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c)

进程间传输的数据被成为 Binder 对象，它是一个 [flat_binder_object](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L49),结构如下

```
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
其中 类型 [type](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L29) 描述了 Binder 对象的类型，包含如下三大类(五种)

```
enum {
    BINDER_TYPE_BINDER  = B_PACK_CHARS('s', 'b', '*', B_TYPE_LARGE),
    BINDER_TYPE_WEAK_BINDER = B_PACK_CHARS('w', 'b', '*', B_TYPE_LARGE),
    BINDER_TYPE_HANDLE  = B_PACK_CHARS('s', 'h', '*', B_TYPE_LARGE),
    BINDER_TYPE_WEAK_HANDLE = B_PACK_CHARS('w', 'h', '*', B_TYPE_LARGE),
    BINDER_TYPE_FD      = B_PACK_CHARS('f', 'd', '*', B_TYPE_LARGE),
};
```
[flags](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L52) 则表述了[传输方式](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L110)，如同步、异步等

```
enum transaction_flags {
    TF_ONE_WAY  = 0x01, /* this is a one-way call: async, no return */
    TF_ROOT_OBJECT  = 0x04, /* contents are the component's root object */
    TF_STATUS_CODE  = 0x08, /* contents are a 32-bit status code */
    TF_ACCEPT_FDS   = 0x10, /* allow replies with file descriptors */
};
```

而 `flat_binder_object` 中的 [union 联合体](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L55) 就是要传输的数据，当类型为 `BINDER` 时， 数据就是一个本地对象 [*binder](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L56)，而类型为 `HANDLE` 时，数据则是一个远程对象 [handle](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L57)。

当 `flat_binder_object` 在进程间传递时， Binder 驱动会修改它的类型和数据，交换的代码参考 [binder_transaction](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L1671) 的实现。

该如何理解本地 `BINDER` 对象和远程 `HANDLE` 对象呢？其实它们都代表同一个对象，不过是从不同的角度来看。举例来说，假如进程 A 有个对象 a，对于 A 来说，a 就是一个本地的 `BINDER` 对象；如果进程 B 通过 Binder 驱动访问 A 的 a 对象，对于 B 来说，a 就是一个 `HANDLE`。因此，从根本上来说 handle 和 binder 都指向 a。本地对象还可以带有额外的数据，保存在 [cookie](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L61) 中。

Binder 驱动直接操作的最外层数据结构是 [binder_transaction_data](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L117)， Binder 对象 `flat_binder_object` 被封装在 [binder_transaction_data](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L142) 结构体中。

`binder_transaction_data` 数据结构才是真正传输的数据，其定义如下

```
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

`flat_binder_object` 就被封装在 [*buffer](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L142)中，其中的 [unsigned int    code;](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L126) 则是传输命令，描述了 Binder 对象执行的操作。


**binder 设备的创建**

查看 [device_initcall()](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L3704) 函数

```
static struct miscdevice binder_miscdev = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = "binder",
    .fops = &binder_fops
};

static int __init binder_init(void)
{
    int ret;

    ret = misc_register(&binder_miscdev);
    return ret;
}

device_initcall(binder_init);
```

我们从 [misc_register(&binder_miscdev);](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L3716) 及 [.name = "binder"](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L3695) 可以看到， binder 向 kernel 注册了一个 `/dev/binder` 的字符设备，而文件操作都在 [binder_fops](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L3683) 结构体中。

```
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
从此结构体可以看出，主要的操作是 `binder_ioctl()` `binder_mmap()` 函数实现的。

**binder协议和数据结构**

[binder.h](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h) 文件中定义了 binder 协议和重要的数据结构。

首先在 [enum](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L29) 中定义了 binder 处理的类型，引用或是句柄

```
enum {
    BINDER_TYPE_BINDER  = B_PACK_CHARS('s', 'b', '*', B_TYPE_LARGE),
    BINDER_TYPE_WEAK_BINDER = B_PACK_CHARS('w', 'b', '*', B_TYPE_LARGE),
    BINDER_TYPE_HANDLE  = B_PACK_CHARS('s', 'h', '*', B_TYPE_LARGE),
    BINDER_TYPE_WEAK_HANDLE = B_PACK_CHARS('w', 'h', '*', B_TYPE_LARGE),
    BINDER_TYPE_FD      = B_PACK_CHARS('f', 'd', '*', B_TYPE_LARGE),
};
```
下面这段宏定义则是在 `ioctl` 函数调用时可用的具体命令。

```
#define BINDER_WRITE_READ       _IOWR('b', 1, struct binder_write_read)
#define BINDER_SET_IDLE_TIMEOUT     _IOW('b', 3, int64_t)
#define BINDER_SET_MAX_THREADS      _IOW('b', 5, size_t)
#define BINDER_SET_IDLE_PRIORITY    _IOW('b', 6, int)
#define BINDER_SET_CONTEXT_MGR      _IOW('b', 7, int)
#define BINDER_THREAD_EXIT      _IOW('b', 8, int)
#define BINDER_VERSION          _IOWR('b', 9, struct binder_version)
```
在 [BinderDriverReturnProtocol](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L166) 和 [BinderDriverCommandProtocol](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.h#L254) 中 则分别定义了 客户端调用 和 服务端 返回的命令。


**binder_ioctl() 函数**

用户态程序调用 `ioctl` 系统函数向 `/dev/binder` 设备发送数据时，会触发 `binder_ioctl` 函数响应。

[binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)](https://github.com/xdtianyu/android-msm-hammerhead-3.4-marshmallow/blob/master/drivers/staging/android/binder.c#L2734) 函数用来处理


Binder 在头文件中只要定义了两个数据类型, 一个是 `binder_write_read`

```
struct binder_write_read {
    signed long write_size; /* bytes to write */
    signed long write_consumed; /* bytes consumed by driver */
    unsigned long   write_buffer;
    signed long read_size;  /* bytes to read */
    signed long read_consumed;  /* bytes consumed by driver */
    unsigned long   read_buffer;
};
```

以及 `binder_transaction_data`

```
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

Binder与AIDL
-----

AIDL (Android Interface definition language) 是接口描述语言，用于生成在两个进程间进行通信的代码。先看 AIDL 概念图

![AIDL概念图](https://raw.githubusercontent.com/xdtianyu/SourceAnalysis/master/art/AIDL.png)

* Proxy 由 Android Sdk 自动生成，客户端通过 Proxy 与远程服务交互。

* Stub 由 Android Sdk 自动生成，包含对 IBinder 对象操作的封装，需要远程服务实现具体功能。


接下来再看具体实现， 完整源代码 [AidlExample](https://github.com/xdtianyu/AidlExample)

**AIDL 客户端**

在 Android Studio 项目上右键， `New` -> `AIDL` -> `AIDL File` 输入文件名后可以快速创建一个 AIDL 的代码结构。

`IRemoteService.aidl` 示例

    // IRemoteService.aidl
    package com.android.aidltest;
    
    interface IRemoteService {
    
        void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
                double aDouble, String aString);
    }

从生成的示例代码可以看出，AIDL 的语法类似 Java， 我们传递的参数只能是基本类型。

如果要传递自定义类型如 `User`，则需要实现 [Parcelable](http://developer.android.com/reference/android/os/Parcelable.html) 接口。`Parcelable` 是一个与 Java `Serializable` 类似的序列化接口。 

这样类 `User` 的实例就可以储存到 [Parcel](http://developer.android.com/reference/android/os/Parcel.html) 中，而 `Parcel` 则是一个可以通过 `IBinder` 发送数据或对象引用的容器。

`User.java` 示例

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
    
再向 `IRemoteService.aidl` 中添加一个 `addUser()` 方法，同时新建一个 `User.aidl` 文件。

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
    
运行编译后，会在 `generated` 文件夹中生成一个 [IRemoteService.java](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L92) 接口文件。这个接口中有两个内部类 [Stub](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L19) 和 [Stub.Proxy](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L92)。

客户端会从 [Stub.asInterface()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L34) 得到 `IRemoteService (Stub.Proxy)` 的实例，这个实例就是一个通过 Binder 传递回来的 [远程对象](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L93) 的再包装。而服务端则需要实现 [IRemoteService.addUser()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L15) 方法。

```
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

**AIDL服务端**

为了演示进程间通信，我们新建一个模块 `RemoteService` 来实现功能，并在客户端绑定服务。

按客户端的结构新建 `IRemoteService.aidl` `User.aidl` `User.java` 文件，并拷贝内容，注意如果需要请修改包名。

新建服务 `RemoteService` 并在 `onBind()` 时返回 `IRemoteService.Stub` 实例：
    
    public class RemoteService extends Service {
        @Nullable
        @Override
        public IBinder onBind(Intent intent) {
            return mBinder;
        }
    
        // 实现 IRemoteService 接口
        private IBinder mBinder = new IRemoteService.Stub() {
            @Override
            public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, String aString) throws RemoteException {
                Log.e("RemoteService", "basicTypes()");
            }
    
            @Override
            public void addUser(User user) throws RemoteException {
                Log.e("RemoteService", "addUser()");
            }
        };
    }

这样服务端就实现了 [addUser()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L15) 方法，当客户端通过远程对象调用 [IRemoteService.Stub.Proxy.addUser()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L135) 时，远程对象 [mRemote](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L147) 就会通过 [transact()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L147) 发送命令给服务端，服务端收到命令后在 [Stub.onTransact()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L76) 中读取数据并执行 [addUser()](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L84) 操作。

远程 Binder 对象 [mRemote](https://github.com/xdtianyu/AidlExample/blob/master/app/build/generated/source/aidl/debug/org/xdty/remoteservice/IRemoteService.java#L42) 是由客户端绑定服务时 [onServiceConnected()](https://github.com/xdtianyu/AidlExample/blob/master/app/src/main/java/org/xdty/aidlexample/MainActivity.java#L23) 返回的。继续追踪 [bindService()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ContextImpl.java#L1283)

```
    @Override
    public boolean bindService(Intent service, ServiceConnection conn,
            int flags) {
        warnIfCallingFromSystemProcess();
        return bindServiceCommon(service, conn, flags, Process.myUserHandle());
    }
```

可以看到最后是 [ActivityManagerNative.getDefault().bindService()](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ContextImpl.java#L1317#L1320) 来绑定服务

```
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

从上面分析可以看出， AIDL 的本质是对 Binder 的又一次抽象和封装，实际的进程间通信仍是由 Binder 完成的。

Binder与系统服务
-----

**Context.getSystemService()**

Android 系统在启动后会在后台运行很多系统服务提供给应用使用，这些 [服务](http://developer.android.com/reference/android/content/Context.html#getSystemService(java.lang.Class<T>)) 主要有 `WindowManager, LayoutInflater, ActivityManager, PowerManager, AlarmManager, NotificationManager, KeyguardManager, LocationManager, SearchManager, Vibrator, ConnectivityManager, WifiManager, AudioManager, MediaRouter, TelephonyManager, SubscriptionManager, InputMethodManager, UiModeManager, DownloadManager, BatteryManager, JobScheduler, NetworkStatsManager`

我们可以通过 `Context.getSystemService(String name)` 来获取 [服务](http://developer.android.com/reference/android/content/Context.html#getSystemService(java.lang.String))。

例如 可以通过如下方法从 xml 中插入新的视图

    LayoutInflater inflater = (LayoutInflater) getSystemService(LAYOUT_INFLATER_SERVICE);
    inflater.inflate(R.layout.view, root, true);
    
**Context.getSystemService() 源码分析**

追踪 [ContextImpl](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/ContextImpl.java#L1364) `getSystemService()` 源代码

    @Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
    
继续追踪 [SystemServiceRegistry](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/SystemServiceRegistry.java#L719) 源代码

    /**
     * Gets a system service from a given context.
     */
    public static Object getSystemService(ContextImpl ctx, String name) {
        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
        return fetcher != null ? fetcher.getService(ctx) : null;
    }
    
追踪 `SYSTEM_SERVICE_FETCHERS` 可以发现在 [SystemServiceRegistry](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/SystemServiceRegistry.java#L318) 静态区中注册了几乎所有的系统服务

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

上面代码片断中，[PhoneLayoutInflater](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/app/SystemServiceRegistry.java#L322) 最终回到了 [LayoutInflater](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/view/LayoutInflater.java#L204)。而 [LocationManager](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/location/java/android/location/LocationManager.java#L315) 则是对 [ILocationManager](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/location/java/android/location/ILocationManager.aidl#L39) 的封装。可以发现，在 [frameworks/base/location/java/android/location](https://github.com/xdtianyu/android-6.0.0_r1/tree/master/frameworks/base/location/java/android/location) 包下含有大量的 AIDL 文件。

继续追踪 [ServiceManager.getService(Context.LOCATION_SERVICE)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/ServiceManager.java#L49) 


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

    
从上面代码片断可以看出，`ServiceManager` 会从 `sCache` 缓存或 [IServiceManager](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/ServiceManagerNative.java#L33) 中查找服务并返回一个 [IBinder](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/IBinder.java#L85) 对象。这个 `IBinder` 就是一个远程对象，可以通过它与其他进程交互。 

继续深入 [getIServiceManager().getService(name)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/ServiceManager.java#L55) , 进入 [ServiceManagerNative](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/ServiceManagerNative.java#L33) 

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
    
从上边代码片断可以看到，`ServiceManager.getIServiceManager()` 返回的是一个 `ServiceManagerProxy`, 而 `ServiceManager.getService()` 则是在 `ServiceManagerProxy` 中通过 `ServiceManager` 的远程 `Binder` 对象 `mRemote`，操作 `Parcel` 数据，调用 [IBinder.transact(int code, Parcel data, Parcel reply, int flags)](https://github.com/xdtianyu/android-6.0.0_r1/blob/master/frameworks/base/core/java/android/os/IBinder.java#L223) 方法来发送请求，并通过 `reply.readStrongBinder()` 返回了要查找的服务的远程对象。

可以看到，系统服务的获取方式也是通过 AIDL 的方式实现的。


背景
-----

**为什么需要进程间通信？**

我们知道一般每个 APP 在运行时都是一个进程，而每个进程相互独立不能直接操作其他进程的数据，两个独立的进程要进行数据交互，就需要系统提供进程间通信机制。

比如 APP 运行在自己的进程中，系统的媒体播放器运行在另一进程中，APP 要操作媒体播放服务 (MediaPlayerService)，就需要通过进程间通信来管理 媒体播放器 (MediaPlayer)。

举一个现实中例子： 我乘坐 101 路公交车 到 A 站后下车，再乘坐 800 路公交车。 如果 101、800 公交线路比作两个进程，那么 `101开车门->下车->等待->800开车门->上车` 这一系列动作就可以看作是进程间通信，而我自身则是传递的数据。

*扩展阅读 [进程与线程的一个简单解释](http://www.ruanyifeng.com/blog/2013/04/processes_and_threads.html)*

**为什么进程是孤立的？**

为了安全，一个进程一定不能操作其他进程的数据。在 `Linux` 系统中，虚拟内存机制为每一个进程分配一块线性且连续的内存空间，这块空间被操作系统映射到物理内存中。

每一个进程拥有自己独立的虚拟内存空间，它只能操作自己的虚拟内存，这样它就不能操作其他进程的内存数据。只有操作系统可以访问物理内存。

进程的孤立保证了每个进程的内存安全，但是很多情况下都需要进程间的通信，所以需要操作系统提供一种机制来实现进程间通信。`Binder` 就是 Android 系统提供的进程间通信机制。

**什么是用户空间和内核空间？**


结论
-----

1\. AIDL 本质上只是一个用于封装 Binder 操作的工具，最终的进程间通信由 Binder 的 `transact` 和 `onTransact` 完成。



参考
-----

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

