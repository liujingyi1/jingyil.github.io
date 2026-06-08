---
title: "Binder 内存分配"
weight: 4
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Binder 内存分配器（binder_alloc）流程总结

## 核心数据结构关系

```
binder_alloc（每个进程一个）
│
├── vma（vm_area_struct *）
│   └── 描述 binder buffer 的虚拟地址空间（默认 1MB，最大 4MB）
│
├── buffer（unsigned long）
│   └── vma->vm_start，整个地址空间的基址
│
├── buffer_size（size_t）
│   └── vma 总大小
│
├── mm（struct mm_struct *）
│   └── 指向进程的内存描述符，用于保护地址空间操作
│
├── pages（struct binder_lru_page *）
│   └── 数组，每个元素对应一个物理页
│       └── binder_lru_page {
│             page_ptr: 指向物理页（struct page *），NULL 表示未分配/已回收
│             lru: 链表节点，挂在 binder_freelist 上供 shrinker 回收
│             alloc: 反指回 binder_alloc
│           }
│
├── buffers（链表）
│   └── 所有 binder_buffer（空闲+已分配），按 user_data 地址排序
│
├── free_buffers（红黑树）
│   └── 空闲 buffer，按 buffer 大小排序（最佳适配）
│
├── allocated_buffers（红黑树）
│   └── 已分配 buffer，按 user_data 地址排序
│
└── free_async_space（size_t）
    └── 异步 oneway 事务可用空间，初始 = buffer_size / 2
```

### binder_buffer

```
binder_buffer {
    entry:     链表节点（挂在 alloc->buffers）
    rb_node:   红黑树节点（挂在 free_buffers 或 allocated_buffers）
    free:      1=空闲, 0=已分配
    allow_user_free: 驱动内部同步标记，防止并发释放
    async_transaction: 是否为异步事务
    oneway_spam_suspect: 是否怀疑是 spam
    user_data: 用户空间起始地址
    data_size, offsets_size, extra_buffers_size: 数据大小
    transaction: 指向使用此 buffer 的事务
}
```

## 关键 API 和流程

### 1. 初始化：binder_alloc_mmap_handler

```c
int binder_alloc_mmap_handler(struct binder_alloc *alloc, struct vm_area_struct *vma)
```

**流程：**
1. 保存 vma 范围到 `alloc->buffer` 和 `alloc->buffer_size`
2. 分配 pages 数组：`alloc->pages = kvcalloc(buffer_size / PAGE_SIZE, ...)`
3. 初始化每个 `binder_lru_page`：设置 `alloc` 反指针，初始化 lru 链表
4. 创建第一个空闲 buffer（覆盖整个地址空间）
5. 设置 `free_async_space = buffer_size / 2`
6. 通过 `binder_alloc_set_vma` 标记初始化完成

**注意：此时只建立了 VMA，没有分配任何物理页（page_ptr 全为 NULL）。**

---

### 2. 分配 buffer：binder_alloc_new_buf

```
binder_alloc_new_buf(alloc, data_size, offsets_size, extra_buffers_size, is_async)
```

**流程：**

```
① sanitized_size()
   对 data/offsets/extra_buffers 分别对齐到 sizeof(void *)
   检查加法溢出
   最小返回 sizeof(void *)（防止空 buffer 地址冲突）
   ↓
② 预分配一个 binder_buffer（kzalloc）
   ↓
③ binder_alloc_new_buf_locked()
   在 free_buffers 红黑树中按大小查找最佳适配的空闲块
   ↓
   情况 A：正好匹配大小
     直接使用，标记为已分配
     预分配的 binder_buffer 没用上 → 在 out 处 kfree 释放
   
   情况 B：空闲块太大，需要切分
     原 buffer 前半部分 → 已分配（给事务用）
     原 buffer 后半部分 → 新空闲 buffer（new_buffer）
       new_buffer->user_data = buffer->user_data + size
       加入 alloc->buffers 链表
       加入 free_buffers 红黑树
       new_buffer 已用上 → 置 NULL，防止 out 处被 kfree
   
   ④ 从 freelist 移除被占用的页
     binder_lru_freelist_del：遍历涉及的页
     注意处理最后一页被相邻 buffer 共享的情况
   ↓
⑤ binder_install_buffer_pages()
   遍历 buffer 覆盖的所有虚拟页
   对每页：
     - 如果 page_ptr != NULL → 页已存在，跳过
     - 如果 page_ptr == NULL → 缺页，调用 binder_install_single_page
```

### 3. 安装单页：binder_install_single_page

```
binder_install_single_page(alloc, lru_page, addr)
```

**流程：**
1. `mmget_not_zero(alloc->mm)` → 确保进程的 mm_struct 还在
2. `mmap_write_lock(alloc->mm)` → 获取写锁，保护 VMA 操作
3. 再次检查 `binder_get_installed_page(lru_page)` → 防止并发竞争
4. 检查 `alloc->vma` 是否有效 → VMA 可能已被关闭
5. `alloc_page(GFP_KERNEL | __GFP_HIGHMEM | __GFP_ZERO)` → 分配物理页
6. `vm_insert_page(alloc->vma, addr, page)` → 映射到用户空间
7. `binder_set_installed_page(lru_page, page)` → 标记安装完成（带 release 屏障）

---

### 4. 释放 buffer：binder_alloc_free_buf

```
binder_alloc_free_buf(alloc, buffer)
```

**流程：**
1. 如果 `buffer->clear_on_free` → 清零数据
2. `binder_free_buf_locked()`
   - 更新 `free_async_space`（如果是 async）
   - `binder_lru_freelist_add()`：将释放的页加回 freelist
   - 从 allocated_buffers 红黑树移除
   - 检查相邻 buffer 是否也是空闲 → 合并
   - 插入 free_buffers 红黑树

---

### 5. 内存回收：shrinker

```
系统内存紧张
    ↓
kswapd 遍历所有 shrinker
    ↓
binder_shrink_count → 返回 binder_freelist 中的页数
    ↓
binder_shrink_scan → list_lru_walk → binder_alloc_free_page
    ↓
对 freelist 中每页：
    1. 从 freelist 移除
    2. vma_lookup 找到对应的 VMA
    3. zap_page_range_single 解除用户态映射
    4. __free_page 释放物理页
    5. page_ptr = NULL
```

**binder 驱动不知道何时被回收，也不需要知道。下次使用时重新分配即可。**

---

### 6. 数据拷贝

```c
binder_alloc_copy_user_to_buffer(alloc, buffer, buffer_offset, from, bytes)
binder_alloc_copy_to_buffer(alloc, buffer, buffer_offset, src, bytes)
binder_alloc_copy_from_buffer(alloc, dest, buffer, buffer_offset, bytes)
```

**流程：**
1. `check_buffer()` 验证 offset/bytes 在 buffer 范围内
2. `binder_alloc_get_page()` 通过 buffer_offset 计算：
   - `index = (buffer_space_offset) >> PAGE_SHIFT` → 第几页
   - `pgoff = buffer_space_offset & ~PAGE_MASK` → 页内偏移
   - 返回 `lru_page->page_ptr`
3. 按页分批拷贝（`kmap_local_page` + `copy_from_user` / `memcpy`）

---

## 性能分析

### 每次事务必经之路
| 操作 | 复杂度 | 说明 |
|---|---|---|
| sanitized_size | O(1) | 几个加法对齐 |
| 红黑树查找空闲块 | O(log n) | n ≈ buffer 数量，~10 次比较 |
| 红黑树插入/删除 | O(log n) | 分配/释放各一次 |
| 链表操作 | O(1) | 几个指针赋值 |
| 数据拷贝 | O(n) | **真正的性能瓶颈** |
| 页检查 | O(页数) | 大部分情况页已存在，只是读指针 |

### 偶发操作
| 操作 | 触发条件 |
|---|---|
| alloc_page + vm_insert_page | 缺页时（首次分配或被回收后） |
| shrinker 回收 | 系统内存紧张时 |
| oneway spam 检测 | async 空间低于 10% 时 |

### 瓶颈不在管理操作

```
一次典型 binder 调用（10KB 数据）：
总耗时 ≈ 10-100 微秒
  ├── 红黑树查找: < 1 微秒
  ├── 数据拷贝: 5-50 微秒 ← 大头
  ├── 进程切换: 2-5 微秒
  └── 其他: 剩余
```

## 关键设计要点

1. **延迟分配**：mmap 时不分配物理页，第一次写数据时按需分配
2. **可回收**：空闲页通过 shrinker 归还给系统
3. **最佳适配**：free_buffers 按大小排序的红黑树，减少碎片
4. **预分配**：分配时提前 kzalloc 一个 binder_buffer，用不上就释放
5. **页共享**：多个小 buffer 可以共享同一物理页，释放时需检查相邻 buffer
6. **acquire/release 屏障**：binder_set/ get_installed_page 使用 smp_store_release/ smp_load_acquire 保证多核可见性
