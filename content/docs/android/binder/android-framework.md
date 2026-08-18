---
title: "Binder Framework"
weight: 6
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---


# Android Binder C++ 框架核心类：BBinder、BpBinder、BnInterface 与 BpInterface

在 Android 的 Binder C++ 框架中，`BBinder`、`BpBinder`、`BnInterface` 和 `BpInterface` 是核心类。下面按层次介绍它们的角色、区别和联系。

## 1. 抽象基类：IBinder

`android::IBinder` 是所有 Binder 对象的抽象基类，定义了 Binder 通信的基本协议，包括：

- `transact()`：发送事务，可以是远程调用，也可以是本地调用。
- `queryLocalInterface()`：查询当前 Binder 对象是否提供本地接口。
- `localBinder()` / `remoteBinder()`：获取本地 Binder 或远程 Binder 指针。
- `isBinderAlive()`：判断 Binder 对象是否仍然存活。
- `linkToDeath()`：注册 Binder 死亡通知。

`IBinder` 是一个抽象接口，不关心具体对象是本地实现还是远程代理。

---

## 2. 本地对象基类：BBinder

`android::BBinder` 继承自 `IBinder`，代表**本地 Binder 对象**，通常对应服务端实现。

它的主要特点如下：

- 实现了 `transact()`。
- 对于本地调用，默认逻辑是在当前线程中直接调用 `onTransact()`。
- `onTransact()` 是服务端处理客户端事务的入口。
- 服务端通常不会直接继承 `BBinder`，而是通过 `BnInterface` 间接继承。

```cpp
class BBinder : public IBinder {
public:
    virtual status_t onTransact(
        uint32_t code,
        const Parcel& data,
        Parcel* reply,
        uint32_t flags
    );
};
```

在实际服务端代码中，通常需要重写 `onTransact()`，根据事务码 `code` 解析 `Parcel`，再调用对应的业务方法。

---

## 3. 远程代理基类：BpBinder

`android::BpBinder` 继承自 `IBinder`，代表**远程 Binder 对象的代理**，通常由客户端持有。

它的主要特点如下：

- 内部保存一个由 Binder 驱动管理的整型 `handle`。
- 它的 `transact()` 不会在当前进程中直接处理事务。
- `transact()` 会通过 `IPCThreadState` 将事务发送给 Binder 驱动。
- Binder 驱动再将事务转发到目标服务进程。
- 客户端通常不会直接使用 `BpBinder`，而是通过 `BpInterface` 对它进行业务接口封装。

```cpp
class BpBinder : public IBinder {
private:
    int32_t mHandle;  // 远程 Binder 对象的句柄

public:
    status_t transact(
        uint32_t code,
        const Parcel& data,
        Parcel* reply,
        uint32_t flags
    ) override;
};
```

其事务发送逻辑可以概括为：

```cpp
IPCThreadState::self()->transact(
    mHandle,
    code,
    data,
    reply,
    flags
);
```

---

## 4. 接口模板：BnInterface 与 BpInterface

实际的 Binder 服务通常需要定义业务接口，例如 `IMyService`。Android Binder 框架提供了 `BnInterface` 和 `BpInterface` 两个模板，将业务接口与底层 Binder 对象结合起来。

### 4.1 BnInterface\<INTERFACE\>

`BnInterface<INTERFACE>` 用于服务端，将业务接口与 `BBinder` 结合起来。

```cpp
template <typename INTERFACE>
class BnInterface : public INTERFACE, public BBinder {
public:
    virtual sp<IInterface> queryLocalInterface(
        const String16& descriptor
    ) override;

    virtual const String16& getInterfaceDescriptor() const override;
};
```

它通过多重继承同时继承：

1. 业务接口 `INTERFACE`；
2. 本地 Binder 基类 `BBinder`。

服务端实现类通常继承自 `BnInterface<IMyService>`，并实现业务方法及事务分发逻辑。

因此，一个服务端对象同时具备两种身份：

- 它是一个业务接口对象，例如 `IMyService`；
- 它也是一个本地 Binder 对象，可以接收和处理 Binder 事务。

### 4.2 BpInterface\<INTERFACE\>

`BpInterface<INTERFACE>` 用于客户端，将业务接口与远程 Binder 引用结合起来。

```cpp
template <typename INTERFACE>
class BpInterface : public INTERFACE, public BpRefBase {
public:
    virtual sp<IInterface> queryLocalInterface(
        const String16& descriptor
    ) override;

    virtual const String16& getInterfaceDescriptor() const override;

protected:
    explicit BpInterface(const sp<IBinder>& remote);
};
```

它同时继承：

1. 业务接口 `INTERFACE`；
2. `BpRefBase`。

`BpRefBase` 内部持有一个远程 `IBinder` 引用，可以通过 `remote()` 获取。这个远程 `IBinder` 在典型的跨进程场景中通常对应一个 `BpBinder`。

客户端代理类继承自 `BpInterface<IMyService>`，并实现业务方法。业务方法内部负责：

1. 创建并填写请求 `Parcel`；
2. 调用 `remote()->transact(...)`；
3. 读取响应 `Parcel`；
4. 将返回值转换成业务接口所需的类型。

因此，客户端代理对象同时具备两种能力：

- 对上表现为业务接口，例如 `IMyService`；
- 对下持有远程 Binder 对象，能够发起跨进程调用。

---

## 5. 典型使用模式

假设定义一个 Binder 业务接口 `IMyService`。

### 5.1 定义业务接口

```cpp
class IMyService : public IInterface {
public:
    virtual int add(int a, int b) = 0;

    DECLARE_META_INTERFACE(MyService);
};
```

`IMyService` 继承自 `IInterface`，对客户端和服务端公开统一的业务方法。

### 5.2 服务端实现

```cpp
class BnMyService : public BnInterface<IMyService> {
public:
    status_t onTransact(
        uint32_t code,
        const Parcel& data,
        Parcel* reply,
        uint32_t flags
    ) override;
};
```

`BnMyService` 通常在 `onTransact()` 中完成事务分发：

```cpp
status_t BnMyService::onTransact(
    uint32_t code,
    const Parcel& data,
    Parcel* reply,
    uint32_t flags
) {
    switch (code) {
        case ADD_CODE: {
            int32_t a = data.readInt32();
            int32_t b = data.readInt32();
            int32_t result = add(a, b);
            reply->writeInt32(result);
            return NO_ERROR;
        }
        default:
            return BBinder::onTransact(code, data, reply, flags);
    }
}
```

真正的业务服务类继承自 `BnMyService`：

```cpp
class MyService : public BnMyService {
public:
    int add(int a, int b) override {
        return a + b;
    }
};
```

此时：

- `MyService::add()` 负责业务逻辑；
- `BnMyService::onTransact()` 负责 Binder 协议解析和事务分发；
- `BBinder` 负责接收本地 Binder 事务。

### 5.3 客户端代理

```cpp
class BpMyService : public BpInterface<IMyService> {
public:
    explicit BpMyService(const sp<IBinder>& remote)
        : BpInterface<IMyService>(remote) {}

    int add(int a, int b) override {
        Parcel data;
        Parcel reply;

        data.writeInt32(a);
        data.writeInt32(b);

        remote()->transact(
            ADD_CODE,
            data,
            &reply
        );

        return reply.readInt32();
    }
};
```

客户端从 `ServiceManager` 获取服务时，得到的基础类型是 `sp<IBinder>`。

在跨进程场景中，这个 `IBinder` 通常是一个 `BpBinder`。随后，接口转换逻辑会将它包装为 `BpMyService`，最终以 `sp<IMyService>` 的形式提供给客户端。

客户端调用：

```cpp
int result = service->add(1, 2);
```

实际会经过以下调用链：

```text
客户端业务代码
    ↓
BpMyService::add()
    ↓
remote()->transact()
    ↓
BpBinder::transact()
    ↓
IPCThreadState::transact()
    ↓
Binder 驱动
    ↓
服务端 Binder 线程
    ↓
BBinder::transact()
    ↓
BnMyService::onTransact()
    ↓
MyService::add()
```

---

## 6. 关于 BnBinder

`BnBinder` 并不是 Android Binder C++ 框架中的标准类名，通常可能是对 `BnInterface` 的误写，或者是对服务端 Binder 类的泛称。

标准命名习惯如下：

- `Bn + 接口名`：服务端 Binder 桩，例如 `BnMyService`；
- `Bp + 接口名`：客户端 Binder 代理，例如 `BpMyService`。

例如：

```cpp
class BnMyService : public BnInterface<IMyService> {
    // 服务端事务分发
};

class BpMyService : public BpInterface<IMyService> {
    // 客户端代理实现
};
```

底层对应的核心 Binder 类是：

- `BBinder`：本地 Binder 对象；
- `BpBinder`：远程 Binder 代理。

---

## 7. 与 AIBinder、ABBinder 的关系

在 NDK Binder 框架中：

- `AIBinder` 是面向 NDK API 暴露的 Binder 类型；
- 它在内部与 C++ Binder 框架中的 `IBinder` 对象相关联；
- `ABBinder` 用于表示本地 NDK Binder 对象；
- 本地 NDK Binder 对象最终仍需要具备类似 `BBinder` 的本地事务处理能力；
- 远程 NDK Binder 代理内部则会关联远程 `IBinder`，其底层通常对应 `BpBinder` 或等价机制。

从整体角色上看，可以进行如下类比：

```text
C++ Binder 本地对象：BBinder / BnInterface
NDK Binder 本地对象：AIBinder 对应的本地实现（如 ABBinder）

C++ Binder 远程对象：BpBinder / BpInterface
NDK Binder 远程对象：AIBinder 对应的远程代理
```

---

## 8. 核心类总结

| 类名 | 角色 | 侧重点 | 是否经常直接使用 |
|---|---|---|---|
| `IBinder` | Binder 抽象基类 | 定义 Binder 通用接口 | 很少直接实现 |
| `BBinder` | 本地 Binder 对象基类 | 服务端事务处理入口 `onTransact()` | 一般通过 `BnInterface` 间接继承 |
| `BpBinder` | 远程 Binder 代理基类 | 保存远程句柄并发送事务 | 一般通过 `BpInterface` 封装 |
| `BnInterface<INTERFACE>` | 服务端接口模板 | 结合业务接口和 `BBinder` | 服务端 Binder 桩的基础 |
| `BpInterface<INTERFACE>` | 客户端接口模板 | 结合业务接口和远程 Binder 引用 | 客户端代理的基础 |

---

## 9. 核心关系图

```text
                        IInterface
                            ▲
                            │
                      IMyService
                       ▲       ▲
                       │       │
              BnInterface     BpInterface
                   ▲               ▲
                   │               │
             BnMyService       BpMyService
                   ▲               │
                   │               │ remote()
              MyService            ▼
                   │             IBinder
                   ▼               ▲
                BBinder         BpBinder
                   │               │
                   └──── Binder 驱动 ────┘
```

需要注意的是，`BnInterface` 和 `BpInterface` 都会与业务接口产生继承关系，但它们在 Binder 通信中的职责完全不同：

- `BnInterface` 位于服务端，负责接收并分发事务；
- `BpInterface` 位于客户端，负责将业务方法转换成 Binder 事务；
- `BBinder` 是本地 Binder 对象的基础；
- `BpBinder` 是远程 Binder 对象代理的基础。

---

## 10. 总结

Android Binder C++ 框架通过这些类实现了业务接口与 IPC 细节的分离：

- `IBinder` 定义统一的 Binder 对象抽象；
- `BBinder` 表示本地服务对象；
- `BpBinder` 表示远程服务代理；
- `BnInterface` 将服务端业务接口与 `BBinder` 结合；
- `BpInterface` 将客户端业务接口与远程 Binder 引用结合。

客户端只需要面向 `IMyService` 等业务接口编程，不需要直接处理 Binder 驱动、句柄、事务分发和跨进程传输等底层细节。

这种设计实现了：

1. **接口与实现分离**；
2. **本地调用与远程调用形式统一**；
3. **业务逻辑与 IPC 协议解耦**；
4. **客户端代理和服务端实现职责清晰**。


