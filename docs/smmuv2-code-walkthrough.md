# ARM SMMU v2 驱动源码解析与学习路线

> 适用读者：希望快速上手 Linux 内核 `drivers/iommu/arm/arm-smmu.c`/`arm-smmu-v3.c` 以前的 ARM System MMU v2 驱动，并理解其与硬件协作机制的嵌入式/系统工程师。

---

## 1. 为什么要关心 SMMU v2？

- **SMMU（System Memory Management Unit）** 位于外设与主存之间，为 I/O 访问引入地址转换与权限控制，是 ARM SoC 实现设备虚拟化与安全隔离的关键。
- **v2 版本** 是 ARMv7/v8 SoC 中最常见的一代，广泛部署在移动终端、汽车域控制器以及部分服务器平台。
- Linux IOMMU 框架将 SMMU 封装为 `iommu_ops`，直接影响 GPU/NPU/ISP 等外设的 DMA 稳定性和安全性。

掌握 SMMU v2 驱动有助于：

1. 理解 SoC 外设虚拟地址空间的建立过程。
2. 排查 DMA Fault / Translation Fault 等现场问题。
3. 为安全需求定制 Stage-1/Stage-2 页表策略。

---

## 2. 硬件视角速览

| 模块             | 作用 | Linux 驱动中的对应抽象 |
|------------------|------|-------------------------|
| Stream ID (SID)  | 外设发出的标识，区分发起者 | `arm_smmu_master` 中的 `streamids[]` |
| Context Bank (CB)| 每个 SID 映射到的上下文，保存 TTBR/MAIR/TTBCR 等寄存器集 | `arm_smmu_cb` 结构体 |
| Translation Stages | Stage-1/Stage-2 地址转换链路 | 通过 `arm_smmu_domain` 建立页表 |
| TLB + Micro-TLB  | 缓存转换结果 | 刷新接口 `arm_smmu_tlb_inv_*` |
| Command Queue (CMDQ) | 下发失效、配置命令 | `arm_smmu_cmdq_issue_cmd` |
| Event/Fault Queue | 上报异常（Translation Fault, Permission Fault） | `arm_smmu_evtq_thread`, `arm_smmu_handle_event` |

> **提示**：v2 SMMU 没有 v3 的 Stream Table/Context Descriptor 概念，SID → CB 的映射由固定寄存器配置完成。

### 2.1 地址转换路径

```
设备发出 40-bit IO VA
   ↓ (以 SID 区分)
SMMU 查找 Context Bank → 读取 TTBRx
   ↓
阶段 1 页表 (Stage-1) → 可选 Stage-2 → 物理地址
   ↓
主存
```

- Stage-1 常用于给设备提供与 CPU 相同的虚拟地址。
- Stage-2 常用于虚拟化场景，由 Hypervisor 管理。
- v2 支持三种域类型：**Identity**（直通）、**Stage-1**、**Stage-2**。

---

## 3. Linux 驱动文件结构

| 文件/模块 | 说明 |
|-----------|------|
| `drivers/iommu/arm/arm-smmu.c` | v1/v2 通用驱动主体，注册 `iommu_ops`，处理大部分硬件细节 |
| `drivers/iommu/arm/arm-smmu-regs.h` | 寄存器定义、位域宏 |
| `drivers/iommu/arm/arm-smmu-debug.c`（如启用） | debugfs 接口、状态转储 |

源代码遵循 Linux IOMMU 框架：

- **probe** 阶段解析设备树/ACPI，映射寄存器并初始化队列。
- **domain lifecycle**：`domain_alloc` → `domain_init` → `attach_dev` → `map/unmap`。
- **fault/interrupt**：注册 IRQ 处理，内核线程消费事件队列。

---

## 4. 核心数据结构拆解

### 4.1 `struct arm_smmu_device`

负责描述一颗 SMMU 实例：

```c
struct arm_smmu_device {
    struct device             *dev;
    void __iomem              *base;
    u32                        num_context_banks;
    u32                        num_s2_context_banks;
    u32                        num_mapping_groups;
    struct arm_smmu_cb        *cbs;        // 上下文寄存器缓存
    struct arm_smmu_master    *masters;    // 每个绑定外设的信息
    const struct arm_smmu_impl *impl;      // 平台定制实现
    struct arm_smmu_cmdq      cmdq;
    struct arm_smmu_evtq      evtq;
    struct arm_smmu_priq      priq;       // ATS PRI 队列（如支持）
    spinlock_t                global_sync_lock;
    ...
};
```

关键成员：

- `impl`：不同 SoC 可能存在 Errata，借此覆盖默认回调。
- `cmdq/evtq/priq`：队列元数据 + ring buffer 内存。
- `global_sync_lock`：保护跨 CB 的同步操作，例如 TLB 失效。

### 4.2 `struct arm_smmu_domain`

对应 `iommu_domain`，描述一个上下文域：

```c
struct arm_smmu_domain {
    struct arm_smmu_device *smmu;
    struct io_pgtable_ops  *pgtbl_ops;
    enum io_pgtable_fmt     fmt;      // 页表格式（ARM64_LPAE_S1/S2 等）
    struct mutex            init_mutex;
    struct arm_smmu_cfg     cfg;      // 与硬件 Context Bank 直接相关
    struct iommu_domain     domain;   // 嵌入式结构
};
```

- `pgtbl_ops`：与 `io-pgtable` 库结合，完成页表分配和编码。
- `cfg`：包含 CB 号、页表基址、属性寄存器值。

### 4.3 `struct arm_smmu_master`

表示一个连接到 SMMU 的设备：

```c
struct arm_smmu_master {
    struct list_head        list;
    struct device          *dev;
    u16                     streamids[MAX_STREAMIDS];
    int                     num_streamids;
    bool                    ats_enabled;
};
```

- 设备树中的 `iommus = <&smmu SID ...>` 信息在这里落地。

---

## 5. 驱动执行流程

### 5.1 探测（Probe）阶段

1. `arm_smmu_device_probe`
   - 获取资源（内存映射、时钟、复位）。
   - 解析 `of_arm_smmu_configure` 提供的拓扑信息。
   - 初始化命令/事件队列内存并配置寄存器。
2. 注册 IOMMU ops：

```c
static const struct iommu_ops arm_smmu_ops = {
    .capable        = arm_smmu_capable,
    .domain_alloc   = arm_smmu_domain_alloc,
    .domain_free    = arm_smmu_domain_free,
    .attach_dev     = arm_smmu_attach_dev,
    .detach_dev     = arm_smmu_detach_dev,
    .map            = arm_smmu_map,
    .unmap          = arm_smmu_unmap,
    .flush_iotlb_all= arm_smmu_iotlb_sync,
    .iotlb_sync     = arm_smmu_iotlb_sync,
    .add_device     = arm_smmu_add_device,
    .remove_device  = arm_smmu_remove_device,
    .pgsize_bitmap  = (1ULL << 12) | (1ULL << 21) | (1ULL << 30),
};
```

3. 注册 IRQ：CMDQ Completion / EVTQ（Fault）/ PRI（ATS）。

### 5.2 设备绑定流程

1. 外设驱动调用 `of_dma_configure()` → `iommu_fwspec_init()`，从 DT 读取 SID。
2. 当外设请求域：
   - `arm_smmu_domain_alloc` 分配 `arm_smmu_domain`，并创建页表。
   - `arm_smmu_domain_finalise` 根据 `type`（Identity/S1/S2）初始化配置寄存器。
3. `arm_smmu_attach_dev`
   - 将 `arm_smmu_master` 挂到 `domain`。
   - 编程 SID ↔ Context Bank 映射寄存器。
   - 通过 CMDQ 使能上下文（写 `SCTLR`, `TTBR`, `TTBCR` 等）。

### 5.3 IOVA 映射

- `map()` → 调用 `io_pgtable_ops->map()` 分配页表条目并写入 Stage-1/Stage-2 PTE。
- 若页表修改成功，调用 `arm_smmu_tlb_inv_range` 刷新硬件缓存。

### 5.4 中断与 Fault 处理

- EVTQ IRQ → `arm_smmu_evtq_irq` → 唤醒 `arm_smmu_evtq_thread`。
- 线程循环 `arm_smmu_handle_event`：
  - 解析 Fault 信息（SID、地址、FSYNR/FAR）。
  - 打印 `dev_err_ratelimited`，触发回调（`report_iommu_fault`）。
- 若开启 PRI/ATS，`arm_smmu_priq_thread` 负责处理页面请求。

### 5.5 TLB 失效（TLB maintenance）

关键调用栈：

```
arm_smmu_map()
  → arm_smmu_tlb_inv_range()
     → arm_smmu_cmdq_issue_cmd( CMD_TLB_INV )
     → arm_smmu_cmdq_poll_until()
```

- 对全局刷新使用 `TLBI_ALL`；针对段落/页使用 `TLBI_VA`。
- 等待 `CMD_SYNC` 保证命令完成，常见死锁点在这里——需避免在持有全局锁时长时间阻塞。

---

## 6. “上课”式学习建议

### 6.1 第一课：环境搭建

1. 选定内核版本（建议 LTS，例如 6.1）。
2. 打开 `drivers/iommu/arm/arm-smmu.c`，配合文档《ARM® System Memory Management Unit Architecture Specification version 2.0》阅读。
3. 利用 `cscope`/`clangd` 构建索引，方便查找函数跳转。

**作业**：画出 `arm_smmu_device_probe` 函数的调用图。

### 6.2 第二课：域与映射实践

- 在开发板上启用 `CONFIG_IOMMU_DEBUGFS`。
- 编写简单的 DMA 驱动，调用 `dma_alloc_attrs()`，观察 `/sys/kernel/debug/iommu/` 下的页表。
- 修改驱动，测试 `iommu_map()`/`iommu_unmap()` 产生的 TLB 刷新。

**作业**：在 fault 日志中定位 SID，对应到设备树节点。

### 6.3 第三课：调试与性能

- 打开 `CONFIG_ARM_SMMU_DISABLE_BYPASS_BY_DEFAULT`，确保绑定设备必须显式建立域。
- 使用 `trace-cmd record -e arm_smmu_*` 捕获 TLB/Fault 事件。
- 学习如何调整 `queue_wrap_mask` 解决 CMDQ 溢出。

**作业**：模拟命令超时，分析 `arm_smmu_cmdq_issue_cmdlist` 中的超时路径。

### 6.4 讨论课：虚拟化与 Stage-2

- 阅读 `kvm_arm_setup_stage2` 如何与 SMMU 协作。
- 关注 `ANCIENT_HW` 等 Errata 处理宏。
- 探索 GPU 使用 Stage-2（guest @ host）场景。

---

## 7. 常见踩坑与排查清单

| 现象 | 可能原因 | 建议排查 |
|------|----------|-----------|
| `Translation fault` | 页表缺失或权限不符 | 检查 `iommu_map()` 调用是否成功；确认页大小匹配 |
| `CMD_SYNC timeout` | 命令队列阻塞 | 查看 IRQ 是否正常触发；确认 `cmdq_cons` 前进 |
| 设备直通失败 | 使用 Identity 域但硬件禁用了 BYPASS | 检查 `ARM_SMMU_FEAT_TRANS_OPS` 能力，必要时设置 `arm_smmu_ops.identity_domain` |
| ATS/PRI 返回 `Unsupported` | 平台未实现 | 查询 `ARM_SMMU_FEAT_PRI` 标志，关闭驱动中 ATS 支持 |

---

## 8. 进一步阅读

1. ARM® SMMU Architecture Specification (ARM DEN 0020B).
2. Linux Documentation: `Documentation/driver-api/iommu.rst`。
3. 相关补丁系列：Rob Clark、Will Deacon 等维护者在 LKML 上的讨论。
4. 调试工具：`perf c2c`、`trace-cmd`, `debugfs`。

---

## 9. 小结

- SMMU v2 驱动是 Linux IOMMU 框架中典型的硬件抽象实现，理解其数据结构与执行流程能大幅提升在复杂 SoC 上调试 DMA 的效率。
- 建议从硬件规格 → 驱动初始化 → 映射流程 → 故障处理逐层深入，结合实机实验加深记忆。

> 如果你已经完成了以上“课程”，不妨尝试阅读下一代 `arm-smmu-v3.c`，对比 v2/v3 在 Stream Table、ATS 支持等方面的差异，这是验证掌握程度的最好方式。
