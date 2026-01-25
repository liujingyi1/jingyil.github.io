---
title: "Binder 驱动"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# 第二章 Binder驱动

先从驱动端讲起，先了解驱动提供了什么能力，然后在了解native层如何使用这些能力。以下都基于Android内核源码的android15-6.6分支。

### 1. 目录结构

首先，默认的aosp代码中不包含kernel代码的，所以kernel代码需要单独下载，可以从下面地址clone kernel代码：

> `git clone https://android.googlesource.com/kernel/common`

或者使用国内清华镜像

> `git clone https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/kernel/common.git`

下载后driver文件所在目录
```bash
.
├── drivers/android/
│   ├── Kconfig                   # 内核编译配置菜单。当你运行 make menuconfig 并导航到 Device Drivers -> Android 时，看到的选项（如 CONFIG_ANDROID_BINDER_IPC）就定义于此。
│   ├── Makefile                  # 构建脚本。根据 Kconfig 的选择，决定将哪些文件（如 binder.o, binder_alloc.o）编译进内核或模块。
│   ├── TEST_MAPPING
│   ├── android_debug_symbols.c
│   ├── binder.c                  # Binder IPC 驱动的主文件，
│   ├── binder_alloc.c            # Binder专用内存分配器的实现。
│   ├── binder_alloc.h
│   ├── binder_alloc_selftest.c   # 内存分配器的自测试代码。
│   ├── binder_internal.h
│   ├── binder_trace.h
│   ├── binderfs.c                # Binder文件系统（BinderFS）的实现。
│   ├── dbitmap.h                 # 用于高效管理内存页的分配状态，是 binder_alloc 的一个底层组件。
│   ├── debug_kinfo.c
│   ├── debug_kinfo.h
└── └── vendor_hooks.c            # 供应商挂钩实现。
```

```mermaid
graph TD
    A[Kconfig & Makefile] -->|构建配置| B[binder.c **核心引擎**];
    B -->|依赖| C[binder_alloc.c **内存池**];
    C -->|使用| D[dbitmap.h **位图工具**];
    D --> E[android_debug_symbols.c<br>debug_kinfo.c **调试接口**];
    
    B -->|产生事件| F[binder_trace.h **追踪点**];
    B -->|可挂接点| G[vendor_hooks.c **厂商扩展**];
    B -->|挂载为| H[binderfs.c **虚拟文件系统**];
    
    C -->|自测试| I[binder_alloc_selftest.c];
    A -->|触发测试| J[TEST_MAPPING];
```

### 2. 优越性

Binder作为Android中主要的IPC通信机制之一（在系统中还有很多其他IPC通信机制，视场景而定），相比传统Linux IPC（如管道、消息队列、Socket等）具有显著优越性，**其核心设计目标是：在资源受限的移动设备上，为大量、高频的跨进程通信提供高性能、高安全性和易于使用的框架。**

其优越性主要体现在以下几个方面：

📊 **Binder vs. 传统Linux IPC 核心对比**

| 特性维度 | Binder的优越性体现 |
| :--- | :---: |
| **性能** | **一次拷贝** 内存效率极高，延迟更低 |
| **安全性** | 基于进程UID/PID的**身份标识**与**能力控制** 内核保障，系统级安全模型 |
| **易用性** | **面向对象**（调用远程方法如同本地）、**自动代理生成** 开发者友好，降低出错率 |
| **稳定性** | **引用计数**与**生命周期管理** 由驱动管理，避免资源泄漏 |
| **设计理念** | **C/S架构**，结构清晰，与Android组件化架构完美契合 |

而这些特性中的一次拷贝的原理的体现，就是在驱动中。

### 3. 基础原理

由于Linux中已经有了很多其他的进程间通信机制，重新设计这种进程间通信机制，肯定由于Binder通信区别于其他通信机制的特性，而这种使Binder区别于其他，让Binder独有的特性的能力，并且可以使之在驱动中实现，是基于Linux可以提供的两个基础能力，一是**Linux内核的文件描述符（File Descriptor）传递机制**

这是Binder区别于所有其他IPC的根本特性。因为Binder能传递"对象引用"，而不仅仅是数据。这种特性让进程得以实现1， 跨进程对象声明周期管理。 2， 对象引用的特权控制。 3， 面向对象的RPC语义。

所以底层机制是文件描述符的跨进程传递和重映射，将fd抽象为Binder对象引用，在内核中维护对象引用到实际对象的映射关系，提供完成的引用计数、声明周期管理、权限控制。

这就是为什么Binder能实现面向对象的进程间通信，而其他IPC只能实现面向数据的进程间通信。Binder不仅传递数据，更重要的是传递能力（对象的方法调用权限），这是它区别于所有传统Linux IPC的本质特征，这些特征是革命性的。


### 4. 几个重要的结构体

```c
struct binder_ref {
	/* Lookups needed: */
	/*   node + proc => ref (transaction) */
	/*   desc + proc => ref (transaction, inc/dec ref) */
	/*   node => refs + procs (proc exit) */
	struct binder_ref_data data;
	struct rb_node rb_node_desc;
	struct rb_node rb_node_node;
	struct hlist_node node_entry;
	struct binder_proc *proc;
	struct binder_node *node;
	struct binder_ref_death *death;
	struct binder_ref_freeze *freeze;
};
```

```c
struct binder_proc {
	struct hlist_node proc_node;
	struct rb_root threads;
	struct rb_root nodes;
	struct rb_root refs_by_desc;
	struct rb_root refs_by_node;
	struct list_head waiting_threads;
	int pid;
	struct task_struct *tsk;
	const struct cred *cred;
	struct hlist_node deferred_work_node;
	int deferred_work;
	int outstanding_txns;
	bool is_dead;
	bool is_frozen;
	bool sync_recv;
	bool async_recv;
	wait_queue_head_t freeze_wait;
	struct list_head todo;
	struct binder_stats stats;
	struct list_head delivered_death;
	u32 max_threads;
	int requested_threads;
	int requested_threads_started;
	int tmp_ref;
	struct binder_priority default_priority;
	struct dentry *debugfs_entry;
	struct binder_alloc alloc;
	struct binder_context *context;
	spinlock_t inner_lock;
	spinlock_t outer_lock;
	struct dentry *binderfs_entry;
	bool oneway_spam_detection_enabled;
	ANDROID_OEM_DATA(1);
};

```

```c

struct binder_node {
	int debug_id;
	spinlock_t lock;
	struct binder_work work;
	union {
		struct rb_node rb_node;
		struct hlist_node dead_node;
	};
	struct binder_proc *proc;
	struct hlist_head refs;
	int internal_strong_refs;
	int local_weak_refs;
	int local_strong_refs;
	int tmp_refs;
	binder_uintptr_t ptr;
	binder_uintptr_t cookie;
	struct {
		/*
		 * bitfield elements protected by
		 * proc inner_lock
		 */
		u8 has_strong_ref:1;
		u8 pending_strong_ref:1;
		u8 has_weak_ref:1;
		u8 pending_weak_ref:1;
	};
	struct {
		/*
		 * invariant after initialization
		 */
		u8 sched_policy:2;
		u8 inherit_rt:1;
		u8 accept_fds:1;
		u8 txn_security_ctx:1;
		u8 min_priority;
	};
	bool has_async_transaction;
	struct list_head async_todo;
};
```

