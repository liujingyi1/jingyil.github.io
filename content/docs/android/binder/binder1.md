---
title: "Binder 代码结构"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# 第一章 Binder整体框架

---

首先，先从整体的角度看一下，Android中Binder系统的代码结构和一个完整的调用流程，这样在后面的描述中也会知道在讲的是哪一部分，同时也方便去查看代码中的具体实现。

Android代码库中关于Binder整体的代码结构

```bash
# Android Binder 实现：从最上层到最底层

# 1. Java Framework 层（应用开发者可见）
frameworks/base/core/java/android/os/
├── IInterface.java           # Java 接口基类
├── IBinder.java              # Java Binder 接口定义
├── Binder.java               # Java Binder 实现（服务端基类）
├── BinderProxy.java          # Java Binder 代理（客户端，内部使用）
├── Parcel.java               # Java 数据序列化
├── ServiceManagerNative.java # ServiceManager 包装
└── ServiceManager.java       # 系统服务管理器

# 2. AIDL 生成的代码（用户看不到的，编译时生成）
out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/gen/aidl/
├── android/os/
│   └── IActivityManager.java  # 自动生成的接口和Stub类

# 3. JNI 桥接层（连接Java和Native）
frameworks/base/core/jni/
├── android_os_Binder.cpp      # Binder JNI方法
├── android_os_BinderProxy.cpp # BinderProxy JNI方法
├── android_os_Parcel.cpp      # Parcel JNI方法
└── android_util_Binder.cpp    # Binder工具函数

# 4. Native Framework 层（C++实现）
frameworks/native/libs/binder/
├── Binder.cpp                 # BBinder/BpBinder实现
├── BpBinder.cpp              # BpBinder专门实现
├── BBinder.cpp               # BBinder专门实现
├── IBinder.cpp               # Native IBinder接口
├── IInterface.cpp            # Native接口基类
├── IServiceManager.cpp       # Native ServiceManager
├── Parcel.cpp                # Native数据序列化
├── IPCThreadState.cpp        # 与驱动通信的核心
└── ProcessState.cpp          # 进程Binder状态管理
#上面部分是用户空间的Binder实现，通过系统调用，也就是ioctrl与内核驱动通信

# 5. 内核驱动层
kernel/drivers/android/
├── binder.c                   # Binder驱动主实现
├── binder_alloc.c             # Binder内存分配
├── binderfs.c                 # Binder文件系统
└── binder_internal.h          # 内部头文件

# 6. 系统服务（具体实现）
frameworks/base/services/
├── core/java/com/android/server/
│   └── am/ActivityManagerService.java  # AMS实现
└── ...
```

**IServiceManager**

```cpp
// IServiceManager 是 Binder 上的一个服务
// 但它自己也是通过 Binder 实现的

class IServiceManager : public IInterface {
public:
    // 通过 Binder 获取服务
    virtual sp<IBinder> getService(const String16& name) = 0;
    
    // 通过 Binder 注册服务
    virtual status_t addService(const String16& name,
                               const sp<IBinder>& service) = 0;
};

// 实现：
sp<IServiceManager> defaultServiceManager() {
    // 获取 ServiceManager 的 Binder 代理
    sp<IBinder> binder = ProcessState::self()->getContextObject(NULL);
    return interface_cast<IServiceManager>(binder);
}
```

**这是一个递归的架构：**

- Binder 机制本身需要一个服务管理器
- 服务管理器通过 Binder 机制实现
- 类似于：TCP/IP 协议需要 DNS，DNS 通过 TCP/IP 实现

**各层之间映射关系**

```java
// Java层对象与Native层对象的对应关系：
Java Object                 ↔ Native Object
┌──────────────────────┐   ↔ ┌──────────────────────┐
│ Binder               │   ↔ │ BBinder              │
│   - mObject          │━━━━━┥   (C++对象指针)      │
└──────────────────────┘   ↔ └──────────────────────┘

┌──────────────────────┐   ↔ ┌──────────────────────┐
│ BinderProxy          │   ↔ │ BpBinder             │
│   - mObject          │━━━━━┥   (C++对象指针)      │
│   - mNativeData      │━━━━━┥   (Native数据)       │
└──────────────────────┘   ↔ └──────────────────────┘

┌──────────────────────┐   ↔ ┌──────────────────────┐
│ Parcel               │   ↔ │ android::Parcel      │
│   - mNativePtr       │━━━━━┥   (C++对象指针)      │
└──────────────────────┘   ↔ └──────────────────────┘

// 这些映射通过JNI维护：
static jfieldID gBinderOffsets;      // Binder.mObject
static jfieldID gBinderProxyOffsets; // BinderProxy.mObject
static jfieldID gParcelOffsets;      // Parcel.mNativePtr
```

**Binder完整调用路径**

```cpp
// 完整调用路径：
Java层: IBinder.transact() 
    ↓ 通过JNI
Native层: BpBinder::transact()
    ↓ 调用IPCThreadState
Native层: IPCThreadState::transact()
    ↓ 调用talkWithDriver
Native层: ioctl(mDriverFD, BINDER_WRITE_READ, &bwr)
    ↓ 进入内核空间
驱动层: binder_ioctl()
    ↓ 处理BINDER_WRITE_READ
驱动层: binder_thread_write()
    ↓ 事务传递
驱动层: binder_thread_read() [目标进程]
    ↓ 返回用户空间
Native层: IPCThreadState::executeCommand()
    ↓ 调用BBinder::onTransact()
Native层: BBinder::onTransact()
    ↓ 如果是Java对象，通过JNI回调
Java层: Binder.onTransact()
```

**可视化流程**

```cpp
Client Process        Kernel Driver        Server Process
    |                       |                       |
    |--- ioctl(WRITE) ---> |                       |
    |                       |-- queue transaction->|
    |                       |                       |
    |                       |                       |-- thread in binder_thread_read()
    |                       |                       |<-- gets BR_TRANSACTION
    |                       |                       |
    |                       |                       |-- executeCommand()
    |                       |                       |-- BBinder::onTransact()
```

**BpBinder 和 BnBinder**

```cpp
BpBinder = Binder Proxy Binder
           ↑          ↑
        Binder      代理

BnBinder = Binder Native Binder
           ↑          ↑
        Binder      本地
```

```cpp
// BpBinder：客户端使用的代理
// 代理远程的 Binder 对象
class BpBinder : public IBinder {
    // 内部包含一个 handle（远程引用）
    int32_t mHandle;
    
    // transact 方法将请求发送给驱动
    virtual status_t transact(uint32_t code, ...) {
        return IPCThreadState::self()->transact(mHandle, ...);
    }
};

// BBinder：服务端使用的本地对象
// 实际实现业务逻辑
class BBinder : public IBinder {
    // onTransact 方法处理来自驱动的请求
    virtual status_t onTransact(uint32_t code, ...) = 0;
};

// BnInterface 是一个模板类
// 它继承自 BBinder 和接口类
template<typename INTERFACE>
class BnInterface : public INTERFACE, public BBinder {
    // 同时是 BBinder（可以处理事务）
    // 也是 INTERFACE（实现接口方法）
};
```

**一个Binder调用实例的可视化完整调用流程**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Java Client Process                              │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. 应用程序调用:                                                          │
│    ActivityManager.getService().startActivity(...)                         │
│                                                                             │
│ 2. IActivityManager.Stub.Proxy.startActivity() [AIDL生成的代理]           │
│    - 创建 Parcel data, reply                                              │
│    - data.writeInterfaceToken(DESCRIPTOR)                                 │
│    - data.writeStrongBinder(caller)                                       │
│    - ... 序列化其他参数                                                   │
│                                                                             │
│ 3. 调用: mRemote.transact(TRANSACTION_startActivity, data, reply, 0)      │
│    ↓                                                                       │
│ 4. BinderProxy.transact()                                                 │
│    - 检查权限                                                             │
│    - 调用 native 方法: BinderProxy.transactNative()                       │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                            JNI Bridge (android_os_BinderProxy.cpp)         │
├─────────────────────────────────────────────────────────────────────────────┤
│ 5. android_os_BinderProxy_transact()                                      │
│    - 获取 Native BpBinder 指针:                                           │
│      IBinder* target = (IBinder*)                                         │
│          env->GetLongField(obj, gBinderProxyOffsets.mObject);             │
│    - 转换 Java Parcel 为 Native Parcel                                    │
│    - 调用 target->transact(code, *data, reply, flags)                    │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                        Native Client Process (C++)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│ 6. BpBinder::transact()                                                   │
│    - 获取 IPCThreadState                                                  │
│    - 调用 IPCThreadState::transact(mHandle, code, data, reply, flags)     │
│                                                                             │
│ 7. IPCThreadState::transact()                                             │
│    - 准备 binder_transaction_data                                         │
│    - 调用 writeTransactionData(BC_TRANSACTION, ...)                      │
│    - 调用 waitForResponse()                                               │
│                                                                             │
│ 8. IPCThreadState::talkWithDriver()                                       │
│    - 构造 binder_write_read bwr                                           │
│    - 调用 ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)             │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                         Linux Kernel (Binder Driver)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│ 9. binder_ioctl()                                                         │
│    - 处理 BINDER_WRITE_READ 命令                                          │
│    - 调用 binder_thread_write() 处理 BC_TRANSACTION                       │
│                                                                             │
│10. binder_thread_write()                                                  │
│    - 解析事务数据                                                         │
│    - 调用 binder_transaction()                                            │
│                                                                             │
│11. binder_transaction()                                                   │
│    - 查找目标进程: target_proc = binder_get_proc_for_handle(handle)       │
│    - 创建 binder_transaction 结构体                                       │
│    - 复制数据到目标进程空间                                                │
│    - 将事务加入目标进程的 todo 队列                                        │
│    - 唤醒目标进程: wake_up_interruptible(&target_proc->wait)             │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                         Target Process (system_server)                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                           Native Layer                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│12. 目标进程的Binder线程正在执行:                                          │
│    IPCThreadState::getAndExecuteCommand()                                 │
│    - 调用 talkWithDriver() 读取数据                                        │
│    - 从内核读取到 BR_TRANSACTION                                          │
│                                                                             │
│13. IPCThreadState::executeCommand(BR_TRANSACTION)                         │
│    - 解析 binder_transaction_data                                         │
│    - 获取 BBinder 对象:                                                   │
│      BBinder* b = reinterpret_cast<BBinder*>(tr.cookie);                  │
│    - 调用 b->onTransact(tr.code, buffer, &replyBuf, tr.flags)             │
│                                                                             │
│14. JavaBBinder::onTransact() [对于Java服务]                               │
│    - 调用 JNI 方法:                                                       │
│      jboolean res = env->CallBooleanMethod(                               │
│          mObject, gBinderOffsets.mExecTransact,                           │
│          code, reinterpret_cast<jlong>(&data),                            │
│          reinterpret_cast<jlong>(&reply), flags);                         │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                            JNI Bridge (android_os_Binder.cpp)             │
├─────────────────────────────────────────────────────────────────────────────┤
│15. Binder.execTransact() [JNI调用Java方法]                                │
│    - 调用 Java 层的 Binder.execTransact()                                 │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                           Java Layer (system_server)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│16. Binder.execTransact()                                                  │
│    - 调用 onTransact(code, data, reply, flags)                            │
│                                                                             │
│17. ActivityManagerService.onTransact()                                    │
│    - 实际上是 IActivityManager.Stub.onTransact()                          │
│    - 根据 code 调用对应方法                                               │
│                                                                             │
│18. ActivityManagerService.startActivity()                                 │
│    - 实际的业务逻辑                                                       │
│    - 将结果写入 reply Parcel                                              │
│                                                                             │
│19. 返回路径（逆序进行）:                                                 │
│    Binder.execTransact() 返回 →                                           │
│    JavaBBinder::onTransact() 返回 →                                       │
│    IPCThreadState::executeCommand() 发送 BR_REPLY →                       │
│    ... 原路返回给客户端                                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```