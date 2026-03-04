# Intel GPU Xe 驱动中 P2P（Peer-to-Peer）机制深度研究报告

## 目录

1. [概述](#1-概述)
2. [P2P 技术背景](#2-p2p-技术背景)
3. [Xe 驱动架构与 P2P 的位置](#3-xe-驱动架构与-p2p-的位置)
4. [核心数据结构](#4-核心数据结构)
5. [DMA-BUF P2P 实现](#5-dma-buf-p2p-实现)
6. [SVM（共享虚拟内存）P2P 实现](#6-svm共享虚拟内存p2p-实现)
7. [P2P 互联协议与地址映射](#7-p2p-互联协议与地址映射)
8. [数据迁移引擎中的 P2P 支持](#8-数据迁移引擎中的-p2p-支持)
9. [Peer 所有权管理机制](#9-peer-所有权管理机制)
10. [KUnit 测试框架](#10-kunit-测试框架)
11. [关键代码路径分析](#11-关键代码路径分析)
12. [配置依赖与编译选项](#12-配置依赖与编译选项)
13. [总结与设计亮点](#13-总结与设计亮点)

---

## 1. 概述

Intel Xe GPU 驱动是 Linux 内核中用于支持 Intel 离散和集成 GPU 的下一代图形驱动。在多 GPU 系统或 GPU 与其他 PCIe 设备交互的场景中，**P2P（Peer-to-Peer）DMA** 允许设备之间直接传输数据，而无需经过系统主内存（System RAM）中转，从而大幅降低延迟和带宽消耗。

Xe 驱动中的 P2P 功能并非作为独立模块存在，而是**深度集成**在以下两个核心子系统中：

- **DMA-BUF（Buffer Sharing）**：用于跨设备的缓冲区共享与导入/导出。
- **SVM（Shared Virtual Memory，GPU 共享虚拟内存）**：通过 `drm_pagemap` 框架实现设备内存的映射、迁移和 P2P 访问。

其 P2P 实现依赖于 Linux 内核的 **PCI P2P DMA 子系统**（`linux/pci-p2pdma.h`）和 DRM 子系统的 **`drm_pagemap`** 框架。

---

## 2. P2P 技术背景

### 2.1 什么是 PCIe P2P DMA

PCIe Peer-to-Peer DMA 允许两个 PCIe 设备之间通过 PCIe 总线直接进行数据传输，无需 CPU 或系统内存参与。传统的数据传输路径为：

```
设备 A → 系统内存（RAM）→ 设备 B
```

P2P DMA 路径为：

```
设备 A → PCIe Switch → 设备 B
```

### 2.2 内核 PCI P2P DMA 子系统

Linux 内核提供了 `pci-p2pdma` 子系统（位于 `drivers/pci/p2pdma.c`），提供以下核心 API：

| API | 功能 |
|-----|------|
| `pci_p2pdma_distance()` | 检查两个 PCI 设备之间的 P2P DMA 可达性及"距离" |
| `dma_map_resource()` | 将设备物理地址映射为 DMA 地址供 P2P 传输使用 |
| `dma_unmap_resource()` | 取消 P2P DMA 映射 |

`pci_p2pdma_distance()` 返回值：
- **≥ 0**：P2P 可达，值越小表示拓扑距离越近
- **< 0**：P2P 不可达，需要通过系统内存中转

### 2.3 DRM Pagemap 框架

DRM 子系统提供了 `drm_pagemap` 框架（`include/drm/drm_pagemap.h`），为 GPU 驱动提供统一的设备内存管理接口。该框架定义了**互联协议**（interconnect protocol），区分不同的内存访问路径：

```c
enum drm_interconnect_protocol {
    DRM_INTERCONNECT_SYSTEM,   // 系统内存（通过 DMA 映射）
    DRM_INTERCONNECT_DRIVER,   // 驱动自定义（如本设备 VRAM）
    /* 驱动可以在此基础上扩展私有值 */
};
```

---

## 3. Xe 驱动架构与 P2P 的位置

P2P 功能在 Xe 驱动中的分布如下：

```
drivers/gpu/drm/xe/
├── xe_dma_buf.c/h          # DMA-BUF 导出/导入（P2P 附着检测）
├── xe_svm.c/h              # SVM 共享虚拟内存（P2P 映射/取消映射/所有权管理）
├── xe_migrate.c            # 迁移引擎（P2P 协议验证与页表构建）
├── tests/
│   └── xe_dma_buf.c        # P2P 相关 KUnit 测试
└── Kconfig                 # 编译配置

include/drm/
├── drm_pagemap.h           # DRM Pagemap 核心框架
└── drm_pagemap_util.h      # Peer 所有权管理工具
```

---

## 4. 核心数据结构

### 4.1 `drm_pagemap_addr` — 带协议标签的地址表示

定义于 `include/drm/drm_pagemap.h`：

```c
struct drm_pagemap_addr {
    dma_addr_t addr;    // DMA 地址或驱动自定义地址
    u64 proto : 54;     // 互联协议标识
    u64 order : 8;      // 页阶（大小 = PAGE_SIZE << order）
    u64 dir : 2;        // DMA 方向
};
```

这个结构是 P2P 的核心抽象。`proto` 字段编码了地址的来源和访问路径——是系统内存的 DMA 地址、同设备 VRAM 地址，还是 P2P 的 PCIe 地址。

### 4.2 Xe 驱动的互联协议扩展

Xe 驱动在 `xe_svm.h` 中扩展了两个私有协议值：

```c
#define XE_INTERCONNECT_VRAM  DRM_INTERCONNECT_DRIVER      // 值 = 1，同设备 VRAM 访问
#define XE_INTERCONNECT_P2P   (XE_INTERCONNECT_VRAM + 1)   // 值 = 2，跨设备 P2P 访问
```

这三种协议的含义：

| 协议值 | 宏名称 | 含义 |
|--------|--------|------|
| 0 | `DRM_INTERCONNECT_SYSTEM` | 通过系统内存 DMA 映射访问 |
| 1 | `XE_INTERCONNECT_VRAM` | 本设备 VRAM 内部访问（使用 DPA 地址） |
| 2 | `XE_INTERCONNECT_P2P` | 跨设备 PCIe P2P 访问（使用 PCIe 地址） |

### 4.3 `xe_pagemap` — Xe 设备内存管理器

```c
struct xe_pagemap {
    struct dev_pagemap pagemap;       // 内核 dev_pagemap（管理 struct page）
    struct drm_pagemap dpagemap;      // DRM pagemap（设备内存操作接口）
    struct work_struct destroy_work;  // 异步销毁工作队列
    struct drm_pagemap_peer peer;     // P2P peer 所有权追踪
    resource_size_t hpa_base;         // 主机物理地址基地址
    struct xe_vram_region *vr;        // 指向 VRAM 区域的反向指针
};
```

关键点：`peer` 成员用于 P2P 所有权计算——确定哪些设备之间具有快速互联通路。

### 4.4 `drm_pagemap_peer` — P2P Peer 结构

定义于 `include/drm/drm_pagemap_util.h`：

```c
struct drm_pagemap_peer {
    struct drm_pagemap_owner_list *list;  // 所属的 owner list
    struct list_head link;                 // 链表链接
    struct drm_pagemap_owner *owner;       // 共享 owner 指针
    void *private;                         // 子类型标识
};
```

### 4.5 `drm_pagemap_ops` — 设备映射操作

```c
struct drm_pagemap_ops {
    struct drm_pagemap_addr (*device_map)(...);    // 为设备映射页面
    void (*device_unmap)(...);                      // 取消设备映射
    int (*populate_mm)(...);                        // 填充地址空间
    void (*destroy)(...);                           // 销毁 pagemap
};
```

Xe 驱动实现了完整的操作集（`xe_drm_pagemap_ops`），其中 `device_map` 和 `device_unmap` 是 P2P 的关键路径。

---

## 5. DMA-BUF P2P 实现

### 5.1 核心文件

- **`drivers/gpu/drm/xe/xe_dma_buf.c`**

### 5.2 P2P 能力检测（`xe_dma_buf_attach`）

当一个外部设备尝试附着（attach）到 Xe 导出的 DMA-BUF 时，驱动首先检查 P2P 能力：

```c
static int xe_dma_buf_attach(struct dma_buf *dmabuf,
                             struct dma_buf_attachment *attach)
{
    struct drm_gem_object *obj = attach->dmabuf->priv;

    if (attach->peer2peer &&
        pci_p2pdma_distance(to_pci_dev(obj->dev->dev), attach->dev, false) < 0)
        attach->peer2peer = false;

    if (!attach->peer2peer && !xe_bo_can_migrate(gem_to_xe_bo(obj), XE_PL_TT))
        return -EOPNOTSUPP;

    xe_pm_runtime_get(to_xe_device(obj->dev));
    return 0;
}
```

**工作流程**：
1. 检查 `attach->peer2peer` 标志是否已请求 P2P。
2. 调用 `pci_p2pdma_distance()` 验证 PCIe 拓扑是否支持 P2P。
3. 若 P2P 不可达，将 `peer2peer` 置为 `false`。
4. 若 P2P 不可用且缓冲区无法迁移到系统内存（TT），返回 `-EOPNOTSUPP`。

### 5.3 P2P 感知的缓冲区 Pin 操作

```c
static int xe_dma_buf_pin(struct dma_buf_attachment *attach)
{
    // ...
    if (!IS_ENABLED(CONFIG_DMABUF_MOVE_NOTIFY)) {
        allow_vram = false;
    } else {
        list_for_each_entry(attach, &dmabuf->attachments, node) {
            if (!attach->peer2peer) {
                allow_vram = false;  // 任一附着不支持 P2P 则禁止 VRAM
                break;
            }
        }
    }

    if (!allow_vram) {
        ret = xe_bo_migrate(bo, XE_PL_TT, NULL, exec);  // 迁移到系统内存
        // ...
    }
    // ...
}
```

**核心逻辑**：遍历所有附着者，只有当**所有**附着者都支持 P2P 时，缓冲区才允许保留在 VRAM 中。否则必须迁移到系统内存（TT/System）。

### 5.4 P2P 感知的 Map 操作

```c
static struct sg_table *xe_dma_buf_map(struct dma_buf_attachment *attach,
                                       enum dma_data_direction dir)
{
    if (!attach->peer2peer && !xe_bo_can_migrate(bo, XE_PL_TT))
        return ERR_PTR(-EOPNOTSUPP);

    if (!xe_bo_is_pinned(bo)) {
        if (!attach->peer2peer)
            r = xe_bo_migrate(bo, XE_PL_TT, NULL, exec);  // 非 P2P：迁移到 TT
        else
            r = xe_bo_validate(bo, NULL, false, exec);      // P2P：就地验证
    }

    switch (bo->ttm.resource->mem_type) {
    case XE_PL_TT:
        // 系统内存路径：构建 scatter-gather 表
        sgt = drm_prime_pages_to_sg(...);
        dma_map_sgtable(attach->dev, sgt, dir, ...);
        break;
    case XE_PL_VRAM0:
    case XE_PL_VRAM1:
        // VRAM 路径（P2P）：使用 VRAM 管理器分配 SGT
        r = xe_ttm_vram_mgr_alloc_sgt(xe_bo_device(bo), ...);
        break;
    }
    return sgt;
}
```

### 5.5 动态附着操作与 P2P 启用

```c
static const struct dma_buf_attach_ops xe_dma_buf_attach_ops = {
    .allow_peer2peer = true,         // 允许 P2P
    .move_notify = xe_dma_buf_move_notify  // 支持移动通知
};
```

`allow_peer2peer = true` 声明告诉 DMA-BUF 框架此导入器支持 P2P 传输。

### 5.6 移动通知（Move Notify）

```c
static void xe_dma_buf_move_notify(struct dma_buf_attachment *attach)
{
    struct xe_bo *bo = gem_to_xe_bo(obj);
    XE_WARN_ON(xe_bo_evict(bo, exec));
}
```

当导出器发生内存位置变化时（例如从 VRAM 迁移到系统内存），此回调通知所有导入器驱逐其本地映射，保持缓存一致性。

---

## 6. SVM（共享虚拟内存）P2P 实现

### 6.1 核心文件

- **`drivers/gpu/drm/xe/xe_svm.c`**
- **`drivers/gpu/drm/xe/xe_svm.h`**

SVM 子系统是 Xe 驱动中 P2P 的更深层次实现，它通过 `drm_pagemap` 框架管理设备私有内存，并支持多 GPU 之间的 P2P 直接访问。

### 6.2 Peer 子类型定义

```c
/* 标识 struct drm_pagemap_peer 的子类 */
#define XE_PEER_PAGEMAP  ((void *)0ul)   // 关联到 xe_pagemap
#define XE_PEER_VM       ((void *)1ul)   // 关联到 xe_vm
```

这两个标识用于在 `xe_peer_to_dev()` 函数中根据 peer 类型提取对应的 `struct device`。

### 6.3 P2P 互联检测（`xe_has_interconnect`）

这是 P2P 能力检测的核心函数：

```c
static bool xe_has_interconnect(struct drm_pagemap_peer *peer1,
                                struct drm_pagemap_peer *peer2)
{
    struct device *dev1 = xe_peer_to_dev(peer1);
    struct device *dev2 = xe_peer_to_dev(peer2);

    if (dev1 == dev2)
        return true;  // 同设备始终互联

    return pci_p2pdma_distance(to_pci_dev(dev1), dev2, true) >= 0;
}
```

**关键设计**：
- 同设备访问（`dev1 == dev2`）始终返回 `true`。
- 跨设备访问通过 `pci_p2pdma_distance()` 检查 PCIe 拓扑。
- 第三个参数 `true` 表示启用详细日志记录。

### 6.4 Peer 设备解析（`xe_peer_to_dev`）

```c
static struct device *xe_peer_to_dev(struct drm_pagemap_peer *peer)
{
    if (peer->private == XE_PEER_PAGEMAP)
        return container_of(peer, struct xe_pagemap, peer)->dpagemap.drm->dev;

    return container_of(peer, struct xe_vm, svm.peer)->xe->drm.dev;
}
```

该函数根据 peer 的 `private` 标识，使用 `container_of` 宏从两种不同的嵌入结构中提取底层硬件 `struct device`。

---

## 7. P2P 互联协议与地址映射

### 7.1 设备地址空间

Xe 驱动中有三种关键地址：

| 地址类型 | 缩写 | 含义 |
|----------|------|------|
| Host Physical Address | HPA | 主机物理地址（通过 `devm_memremap_pages` 分配） |
| Device Physical Address | DPA | 设备物理地址（GPU 内部可见） |
| PCIe Address | - | PCIe BAR 空间地址（其他设备可通过 PCIe 总线访问） |

### 7.2 地址转换函数

```c
// 页面 → 设备物理地址（DPA）
static u64 xe_page_to_dpa(struct page *page)
{
    struct xe_pagemap *xpagemap = xe_page_to_pagemap(page);
    struct xe_vram_region *vr = xe_pagemap_to_vr(xpagemap);
    u64 hpa_base = xpagemap->hpa_base;
    u64 pfn = page_to_pfn(page);
    u64 offset = (pfn << PAGE_SHIFT) - hpa_base;
    u64 dpa = vr->dpa_base + offset;
    return dpa;
}

// 页面 → PCIe 地址（用于 P2P）
static u64 xe_page_to_pcie(struct page *page)
{
    struct xe_pagemap *xpagemap = xe_page_to_pagemap(page);
    struct xe_vram_region *vr = xe_pagemap_to_vr(xpagemap);
    return xe_page_to_dpa(page) - vr->dpa_base + vr->io_start;
}
```

**地址转换关系**：

```
HPA（struct page PFN）─────→ DPA = dpa_base + (HPA - hpa_base)
                                  │
                                  ├──→ 同设备访问：直接使用 DPA
                                  │
                                  └──→ P2P 访问：PCIe Addr = DPA - dpa_base + io_start
```

### 7.3 设备映射函数（`xe_drm_pagemap_device_map`）

这是 P2P 地址映射的核心：

```c
static struct drm_pagemap_addr
xe_drm_pagemap_device_map(struct drm_pagemap *dpagemap,
                          struct device *dev,
                          struct page *page,
                          unsigned int order,
                          enum dma_data_direction dir)
{
    struct device *pgmap_dev = dpagemap->drm->dev;
    enum drm_interconnect_protocol prot;
    dma_addr_t addr;

    if (pgmap_dev == dev) {
        // 同设备：使用 DPA 地址
        addr = xe_page_to_dpa(page);
        prot = XE_INTERCONNECT_VRAM;
    } else {
        // 跨设备 P2P：使用 PCIe 地址 + DMA 映射
        addr = dma_map_resource(dev,
                                xe_page_to_pcie(page),
                                PAGE_SIZE << order, dir,
                                DMA_ATTR_SKIP_CPU_SYNC);
        prot = XE_INTERCONNECT_P2P;
    }

    return drm_pagemap_addr_encode(addr, prot, order, dir);
}
```

**P2P 路径的关键操作**：
1. 将页面转换为 PCIe 物理地址（`xe_page_to_pcie`）。
2. 调用 `dma_map_resource()` 将 PCIe 地址映射为目标设备的 DMA 地址。
3. 标记协议为 `XE_INTERCONNECT_P2P`。
4. 编码为 `drm_pagemap_addr` 结构返回。

### 7.4 设备取消映射（`xe_drm_pagemap_device_unmap`）

```c
static void xe_drm_pagemap_device_unmap(struct drm_pagemap *dpagemap,
                                        struct device *dev,
                                        const struct drm_pagemap_addr *addr)
{
    if (addr->proto != XE_INTERCONNECT_P2P)
        return;  // 非 P2P 映射无需取消

    dma_unmap_resource(dev, addr->addr, PAGE_SIZE << addr->order,
                       addr->dir, DMA_ATTR_SKIP_CPU_SYNC);
}
```

**设计要点**：只有 P2P 映射（`XE_INTERCONNECT_P2P`）需要调用 `dma_unmap_resource()` 清理。VRAM 内部访问（`XE_INTERCONNECT_VRAM`）的 DPA 地址是设备内部地址，不需要 DMA 取消映射。

---

## 8. 数据迁移引擎中的 P2P 支持

### 8.1 迁移函数（`xe_migrate.c`）

Xe 驱动的迁移引擎支持在 VRAM 和系统内存（SRAM）之间复制数据。当涉及 P2P 地址时，迁移引擎需要正确处理协议标识：

```c
// 在构建页表更新批处理时验证协议
xe_tile_assert(m->tile, sram_addr[i].proto ==
               DRM_INTERCONNECT_SYSTEM ||
               sram_addr[i].proto == XE_INTERCONNECT_P2P);
```

这个断言确保传入的 SRAM 侧地址只能是两种合法类型：
- `DRM_INTERCONNECT_SYSTEM`：标准系统内存 DMA 地址。
- `XE_INTERCONNECT_P2P`：来自其他设备的 P2P DMA 地址。

### 8.2 数据复制函数（`xe_svm_copy`）

```c
static int xe_svm_copy(struct page **pages,
                       struct drm_pagemap_addr *pagemap_addr,
                       unsigned long npages,
                       const enum xe_svm_copy_dir dir,
                       struct dma_fence *pre_migrate_fence)
```

此函数实现了高效的批量页面复制：
- 查找物理上连续的设备页面。
- 以 **8MB 块**（`XE_MIGRATE_CHUNK_SIZE`）为单位进行 GPU 硬件复制。
- 支持两个方向的复制：
  - `XE_SVM_COPY_TO_VRAM`：系统内存 → 设备 VRAM
  - `XE_SVM_COPY_TO_SRAM`：设备 VRAM → 系统内存
- 通过 `dma_fence` 机制实现异步执行和同步等待。

### 8.3 P2P 迁移上下文

`drm_pagemap_migrate_details` 结构定义了 P2P 迁移的策略参数：

```c
struct drm_pagemap_migrate_details {
    unsigned long timeslice_ms;            // 迁移后页面保持驻留的最小时间
    u32 can_migrate_same_pagemap : 1;      // 是否允许同 pagemap 内迁移
    u32 source_peer_migrates : 1;          // P2P 迁移时由源端还是目标端执行复制
};
```

`source_peer_migrates` 标志决定了 P2P 迁移的方向：当设置为 `true` 时，使用源 `drm_pagemap` 的 `copy_to_ram()` 回调；否则使用目标 `drm_pagemap` 的 `copy_to_devmem()` 回调。

---

## 9. Peer 所有权管理机制

### 9.1 Owner List

Xe 驱动使用全局的 owner list 来追踪所有 P2P peer：

```c
static DRM_PAGEMAP_OWNER_LIST_DEFINE(xe_owner_list);
```

`drm_pagemap_owner_list` 结构维护了一个受互斥锁保护的 peer 链表。

### 9.2 所有权获取（`drm_pagemap_acquire_owner`）

在 SVM 初始化和 pagemap 创建时，peer 通过此函数注册到 owner list 中：

```c
// SVM 初始化时（xe_svm_init）
vm->svm.peer.private = XE_PEER_VM;
err = drm_pagemap_acquire_owner(&vm->svm.peer, &xe_owner_list,
                                xe_has_interconnect);

// Pagemap 创建时（xe_pagemap_create）
xpagemap->peer.private = XE_PEER_PAGEMAP;
err = drm_pagemap_acquire_owner(&xpagemap->peer, &xe_owner_list,
                                xe_has_interconnect);
```

`drm_pagemap_acquire_owner()` 函数：
1. 遍历 owner list 中的所有已注册 peer。
2. 对每对 peer 调用 `xe_has_interconnect` 检查 P2P 可达性。
3. 将具有互联能力的 peer 分组到同一个 `drm_pagemap_owner` 下。
4. 这使得后续的迁移决策可以快速判断两个 pagemap 是否可以进行 P2P 传输。

### 9.3 所有权释放

```c
// SVM 关闭时
void xe_svm_close(struct xe_vm *vm) {
    drm_pagemap_release_owner(&vm->svm.peer);
}

// Pagemap 销毁时
static void xe_pagemap_destroy_work(struct work_struct *work) {
    // ...
    drm_pagemap_release_owner(&xpagemap->peer);
    kfree(xpagemap);
}
```

### 9.4 Pagemap Owner 的作用

`pagemap->owner`（来自 `drm_pagemap_peer.owner`）被赋值给 `dev_pagemap.owner`：

```c
pagemap->owner = xpagemap->peer.owner;
```

这个 owner 指针被内核的 `dev_pagemap` 子系统用于判断两个设备私有页面是否属于同一"所有者"（即具有 P2P 互联能力），从而决定迁移策略。

---

## 10. KUnit 测试框架

### 10.1 测试文件

- **`drivers/gpu/drm/xe/tests/xe_dma_buf.c`**

### 10.2 P2P 检测辅助函数

```c
static bool p2p_enabled(struct dma_buf_test_params *params)
{
    return IS_ENABLED(CONFIG_PCI_P2PDMA) && params->attach_ops &&
           params->attach_ops->allow_peer2peer;
}
```

### 10.3 Non-P2P 附着操作（用于对比测试）

```c
static const struct dma_buf_attach_ops nop2p_attach_ops = {
    .allow_peer2peer = false,      // 禁用 P2P
    .move_notify = xe_dma_buf_move_notify
};
```

### 10.4 测试参数矩阵

测试覆盖了以下维度的组合：

| 维度 | 选项 |
|------|------|
| 内存类型 | `VRAM0`、`SYSTEM`、`VRAM0 + SYSTEM` |
| 附着操作 | P2P 启用、P2P 禁用、无动态附着（NULL） |
| 设备关系 | 同设备、模拟不同设备（`force_different_devices`） |

共定义了 **20 种**测试参数组合，覆盖了：

1. **P2P + VRAM0 + 同设备**：缓冲区保留在 VRAM。
2. **P2P + VRAM0 + 不同设备**：通过 P2P 从 VRAM 访问。
3. **No-P2P + VRAM0 + 不同设备**：P2P 不可用，迁移到系统内存。
4. **无动态附着 + VRAM0 + 不同设备**：不支持 move_notify，pin 到 TT 时失败。
5. **SYSTEM 内存各种组合**：验证系统内存路径的正确性。

### 10.5 驻留验证（`check_residency`）

测试通过 `check_residency()` 函数验证缓冲区在不同 P2P 配置下的预期内存位置：

```c
mem_type = XE_PL_VRAM0;
if (!(params->mem_mask & XE_BO_FLAG_VRAM0))
    mem_type = XE_PL_TT;                                  // 无 VRAM → TT
else if (params->force_different_devices && !p2p_enabled(params))
    mem_type = XE_PL_TT;                                  // 无 P2P → 迁移到 TT
else if (params->force_different_devices && !is_dynamic(params) &&
         (params->mem_mask & XE_BO_FLAG_SYSTEM))
    mem_type = XE_PL_TT;                                  // 非动态 pin → 迁移到 TT
```

---

## 11. 关键代码路径分析

### 11.1 DMA-BUF 导出 P2P 路径

```
应用程序调用 dma_buf_export()
  └→ xe_gem_prime_export()
       └→ 设置 buf->ops = &xe_dmabuf_ops（包含 P2P 感知的 attach/pin/map）

外部设备附着 DMA-BUF
  └→ dma_buf_dynamic_attach() with xe_dma_buf_attach_ops
       └→ .allow_peer2peer = true
       └→ xe_dma_buf_attach()
            └→ pci_p2pdma_distance() 检查 P2P 可达性
            └→ 若不可达：attach->peer2peer = false

外部设备映射 DMA-BUF
  └→ xe_dma_buf_map()
       ├→ P2P 可用：xe_bo_validate()（缓冲区保留 VRAM）
       │    └→ xe_ttm_vram_mgr_alloc_sgt()（生成 VRAM SGT）
       └→ P2P 不可用：xe_bo_migrate(XE_PL_TT)（迁移到系统内存）
            └→ drm_prime_pages_to_sg() + dma_map_sgtable()
```

### 11.2 SVM P2P 映射路径

```
GPU 页面错误（page fault）
  └→ xe_svm_handle_pagefault()
       └→ drm_gpusvm_range_get_pages()
            └→ 获取页面的 drm_pagemap
            └→ 调用 dpagemap->ops->device_map()

xe_drm_pagemap_device_map()
  ├→ 同设备（pgmap_dev == dev）：
  │    └→ addr = xe_page_to_dpa(page)
  │    └→ prot = XE_INTERCONNECT_VRAM
  └→ 跨设备 P2P（pgmap_dev != dev）：
       └→ pcie_addr = xe_page_to_pcie(page)
       └→ addr = dma_map_resource(dev, pcie_addr, ...)
       └→ prot = XE_INTERCONNECT_P2P
```

### 11.3 数据迁移 P2P 路径

```
xe_svm_copy()
  └→ 查找连续物理页面
  └→ 按 8MB 块执行 GPU 复制
       ├→ TO_VRAM: xe_migrate_to_vram(migrate, npages, pagemap_addr, vram_addr, ...)
       └→ TO_SRAM: xe_migrate_from_vram(migrate, npages, vram_addr, pagemap_addr, ...)

xe_migrate_to_vram() / xe_migrate_from_vram()
  └→ build_pt_update_batch_sram()
       └→ 验证 sram_addr[i].proto == SYSTEM 或 P2P
       └→ 编码为 GPU 页表项（PTE）
       └→ 使用 GPU 复制引擎执行 DMA 传输
```

---

## 12. 配置依赖与编译选项

### 12.1 内核配置

| 配置选项 | 说明 | P2P 影响 |
|----------|------|----------|
| `CONFIG_PCI_P2PDMA` | PCI P2P DMA 支持 | P2P 的基础依赖 |
| `CONFIG_DMABUF_MOVE_NOTIFY` | DMA-BUF 移动通知 | 动态 P2P 附着所需 |
| `CONFIG_DRM_XE_GPUSVM` | Xe GPU SVM 支持 | SVM P2P 路径所需 |
| `CONFIG_ZONE_DEVICE` | 设备内存区域 | `drm_pagemap` 框架所需 |
| `CONFIG_DRM_XE_KUNIT_TEST` | Xe KUnit 测试 | P2P 测试所需 |

### 12.2 运行时检测

P2P 支持完全在运行时动态检测，不需要额外的模块参数或编译时开关：
- `pci_p2pdma_distance()` 根据实际 PCIe 拓扑返回结果。
- `attach->peer2peer` 标志在每次附着时独立评估。
- Owner list 在设备注册/注销时动态更新。

---

## 13. 总结与设计亮点

### 13.1 架构设计亮点

1. **协议抽象（Interconnect Protocol）**：通过 `drm_pagemap_addr.proto` 字段统一编码不同的访问路径，使上层代码无需关心具体的地址类型。

2. **优雅的降级策略**：P2P 不可用时自动回退到系统内存路径，保证功能的普适性。DMA-BUF 的 `pin` 操作和 `map` 操作都实现了完整的降级逻辑。

3. **Owner 分组机制**：通过 `drm_pagemap_owner_list` 和 `drm_pagemap_acquire_owner()` 机制，在设备注册时一次性计算 P2P 互联拓扑，避免在热路径（如页面错误处理）中反复查询 PCIe 拓扑。

4. **零拷贝设计**：P2P 路径使用 `dma_map_resource()` / `dma_unmap_resource()` 进行地址映射，不涉及实际的数据拷贝，真正实现了零拷贝的设备间数据共享。

5. **统一的地址转换**：`xe_page_to_dpa()` 和 `xe_page_to_pcie()` 提供了从 `struct page`（HPA 空间）到设备地址空间（DPA 和 PCIe 地址）的清晰转换路径。

6. **异步生命周期管理**：`xe_pagemap_destroy` 支持从原子/回收上下文调用，通过工作队列延迟实际销毁，避免在关键路径中执行耗时的清理操作。

### 13.2 P2P 在 Xe 驱动中的核心价值

| 应用场景 | 无 P2P | 有 P2P |
|----------|--------|--------|
| 多 GPU 数据共享 | 数据经过系统 RAM 中转 | GPU 之间直接传输 |
| GPU-NIC 数据传输 | GPU → RAM → NIC | GPU → NIC（GPUDirect） |
| DMA-BUF 共享 | 强制迁移到系统内存 | 缓冲区保留在 VRAM |
| 内存带宽消耗 | 系统内存带宽成为瓶颈 | 减少系统内存带宽压力 |
| 延迟 | 额外的 CPU 拷贝延迟 | 最小化传输延迟 |

### 13.3 代码组织总结

Xe 驱动的 P2P 实现没有独立的 P2P 模块文件，而是有机地分布在三个层次中：

```
┌─────────────────────────────────────────────────┐
│              应用层 / 用户空间                    │
├─────────────────────────────────────────────────┤
│         DMA-BUF 导出/导入 (xe_dma_buf.c)        │
│   ┌─ attach: P2P 拓扑检查                       │
│   ├─ pin: 全局 P2P 策略决策                      │
│   └─ map: P2P 或系统内存 SGT 生成               │
├─────────────────────────────────────────────────┤
│         SVM / Pagemap (xe_svm.c)                │
│   ┌─ device_map: 地址协议标记                    │
│   ├─ device_unmap: P2P 映射清理                  │
│   └─ owner管理: P2P 互联拓扑分组                 │
├─────────────────────────────────────────────────┤
│         迁移引擎 (xe_migrate.c)                  │
│   ┌─ 协议断言: 验证合法地址类型                   │
│   └─ PTE 编码: 将 P2P/系统地址编入 GPU 页表      │
├─────────────────────────────────────────────────┤
│    Linux 内核 PCI P2P DMA 子系统                 │
│   ┌─ pci_p2pdma_distance(): 拓扑检测             │
│   ├─ dma_map_resource(): P2P 地址映射            │
│   └─ dma_unmap_resource(): P2P 地址释放          │
└─────────────────────────────────────────────────┘
```

这种分散但协调一致的设计使得 P2P 支持成为 Xe 驱动内存管理的有机组成部分，而非一个独立的可插拔模块，体现了 Linux 内核驱动设计中"关注点分离"与"内聚性"之间的平衡。
