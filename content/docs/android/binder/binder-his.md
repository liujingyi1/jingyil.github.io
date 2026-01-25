---
title: "Binder 历史"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

Binder 被正式合入标准 Linux 内核主线（mainline）的时间点是 2017年9月3日，随 Linux 内核 4.14 版本 的合并窗口引入。
最初路径是放在drivers/staging/android/目录下，现在在任何 Linux 4.14 及以上版本 的官方内核源码中，都能在 drivers/android/（后期从 staging 目录移出）找到 Binder 驱动的源代码，可以通过配置选项 `CONFIG_ANDROID_BINDER_IPC` 来启用它。这也意味着Android 可以基于更接近纯净版的标准 Linux 内核运行，大大减少了维护成本和内核碎片化。是两大开源项目融合的关键一步。

然而在Binder合入Linux内核主线之前，经历一个近十年的"Google申请合入Linux内核，然后被Linux社区拒绝"的这样一个反复的过程，而Binder被Linux内核主线拒绝合入，根源在于两者在设计哲学、安全模型、架构理念等多方面的深刻冲突。


### 1. 设计哲学上的冲突

Binder是为移动设备优化的专用IPC，其在设计上追求的是“做一件事，做到极致”。而Linux 主流哲学是“机制与策略分离”，
提供基础原语，让用户空间决定策略。

除此还有Binder深度耦合Android框架,假设上层是Android框架,Linux内核要求保持内核简洁、通用，不假设任何特定用户空间。

Linux 内核维护者（特别是 Greg Kroah-Hartman、Arnd Bergmann 等）的观点：

> “内核应该提供基础构建块，而不是完整的解决方案。Binder 是一个完整的 RPC 系统，包含了太多策略决策。”

### 2. 安全模型冲突：能力 vs 传统权限

Binder 的安全模型：基于能力的细粒度控制

```c
// Binder 的安全检查（简化）
struct binder_transaction_data {
    uid_t sender_euid;      // 发送者UID
    pid_t sender_pid;       // 发送者PID
    // 额外的安全上下文
};
// 可以基于服务接口、方法名进行权限控制
```

Linux 传统安全模型：
- 基于 UID/GID 的 DAC（自主访问控制）
- 后来加入 capabilities（能力机制）
- SELinux/AppArmor 等 MAC（强制访问控制）

冲突点：Binder 在 IPC 层面内建了复杂的安全策略，这被视作“策略”而非“机制”。

### 3. 架构冲突：面向对象 vs 过程式

Binder 的面向对象设计
```text
进程A                进程B
  |                    |
  |-- 获取服务引用 --->|
  |                    |
  |-- 调用方法 ------->|
  |    (参数打包)      |
  |<-- 返回结果 -------|
```

Linux 传统 IPC 设计

```text
进程A                进程B
  |                    |
  |-- 管道/消息队列 -->|
  |-- 共享内存 ------->|
  |-- 信号量同步 ----->|
```

内核维护者的批评：

> “Binder 将面向对象的概念（对象引用、方法调用）带入内核，这与内核的 C 语言、过程式风格格格不入。”

### 4. 技术实现冲突

Binder 的“创新”引起争议

| **Binder 特性**      | **Linux 社区反对理由**                         |
| :------------------- | :--------------------------------------------- |
| **引用计数跨进程**   | 内核对象生命周期应由内核管理，而非用户空间逻辑 |
| **线程池模型内建**   | 线程调度是内核核心功能，不应与特定 IPC 耦合    |
| **复杂的内存管理**   | Binder 的共享内存机制（ashmem）同样被拒绝      |
| **事务的优先级继承** | 复杂且可能与内核调度器冲突                     |

内核维护者评价：

> “这更像是用户空间库的代码，不应该在内核中出现。”

### 5. 性能优化的代代价

Binder 的激进优化：

1. **零拷贝传输**：通过 mmap 共享内存
2. **同步调用语义**：简化编程模型
3. **死亡通知机制**：监控进程/服务状态

社区的反对理由

> 这些优化高度特化于 Android 的使用模式，破坏了内核的通用性。

### 6.历史的拒绝记录

**关键时间线和事件**

2008-2009：第一次尝试

- 提案：将 Binder 提交到 Linux 2.6 内核
- 主要反对者：Andrew Morton、Arnd Bergmann
- 结果：拒绝
- 理由：“这更像是一个用户空间库的功能”

2011-2012：第二次尝试（BinderFS）

- 提案：至少将 Binder 的设备节点管理现代化
- 反对点：BinderFS 仍然依赖于整个 Binder 驱动
- 结果：部分接受，但核心 Binder 仍被拒

2014-2015：Google 的妥协尝试

bash

```
# Google 试图将 Binder 拆分为“通用”部分
# 但社区仍不满意
diff --git a/drivers/android/binder.c b/drivers/android/binder.c
# 大量的重构，但本质未变
```

2019-2020：最近的讨论**

- 新论点：Binder 已被数十亿设备使用
- 社区回应：“使用广泛不代表设计优秀”

而这些差异基于更深层次的文化冲突，Linux社区更理想主义，注重技术纯洁性，和分散治理，Google/Android更注重务实，产品导向，集中控制。这种冲突并非谁对谁错，而是不同目标和约束下的必然结果。Binder 在 Android 生态中极其成功，证明了其设计在特定场景下的价值；Linux 主线保持简洁通用，也维护了内核的长期健康。

通过这些冲突，可以从侧面了解Binder的一些特性、设计思想和理念。

而这种类似的事情，其实还有ashmem，ashmem也是Android针对自己特定的使用场景，而设计出来的一种共享内存，而这种共享内存在Android系统中的大量地方使用着，比如最重要的显示系统，比如多媒体框架，这些都建立在高效的使用ashmem的基础上，所以在其他的章节，会介绍ashmem，这部分也是内核中的实现。