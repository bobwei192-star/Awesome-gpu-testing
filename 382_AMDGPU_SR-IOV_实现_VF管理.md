# Day 382：AMDGPU SR-IOV 实现：VF（Virtual Function）管理

## 学习目标

- 理解 AMDGPU SR-IOV 中 VF 的生命周期管理机制
- 掌握 VF 的创建、初始化、运行和销毁流程
- 理解 PF-VF 通信协议（Mailbox）的实现原理
- 掌握 VF 驱动的功能限制和资源管理方式
- 理解 VF 模式下的内存管理和调度模型

## 知识详解

### 概念原理

#### AMDGPU SR-IOV 实现架构

AMDGPU SR-IOV 的实现分为三个层次：

```
         ┌─────────────────────────────────────────────┐
         │             用户空间 (User Space)             │
         │   VM0 (PF driver)   VM1 (VF driver)  ...    │
         └─────────────────┬───────────────────────────┘
                           │
         ┌─────────────────┴───────────────────────────┐
         │                内核空间 (Kernel)              │
         │  ┌─────────────────────────────────────────┐│
         │  │          AMDGPU PF 驱动模块              ││
         │  │  ┌────────────┐ ┌────────────┐          ││
         │  │  │VF 管理器    │ │Mailbox 服务 │          ││
         │  │  │- vf_ctrl    │ │- msg_recv  │          ││
         │  │  │- alloc      │ │- msg_send  │          ││
         │  │  │- free       │ │- msg_work  │          ││
         │  │  └────────────┘ └────────────┘          ││
         │  │  ┌────────────────────────────────────┐  ││
         │  │  │  资源管理器 (Resource Manager)      │  ││
         │  │  │  - VRAM 分区 / GTT 分配             │  ││
         │  │  │  - 引擎调度 / 显存带宽控制          │  ││
         │  │  └────────────────────────────────────┘  ││
         │  └─────────────────────────────────────────┘│
         │  ┌─────────────────────────────────────────┐│
         │  │         AMDGPU VF 驱动模块              ││
         │  │  ┌────────────┐ ┌────────────┐          ││
         │  │  │命令提交     │ │内存管理     │          ││
         │  │  │- ring      │ │- GTT 分配  │          ││
         │  │  │- ib        │ │- BO pin    │          ││
         │  │  └────────────┘ └────────────┘          ││
         │  │  ┌────────────┐ ┌────────────┐          ││
         │  │  │中断处理     │ │Mailbox 客户端 │          ││
         │  │  │- IH ring   │ │- 请求/响应  │          ││
         │  │  └────────────┘ └────────────┘          ││
         │  └─────────────────────────────────────────┘│
         └─────────────────┬───────────────────────────┘
                           │
         ┌─────────────────┴───────────────────────────┐
         │              GPU 固件 (Firmware)             │
         │  ┌─────────────────────────────────────────┐│
         │  │   SR-IOV 管理固件 (SRIOV Manger)         ││
         │  │  - VF 上下文管理                         ││
         │  │  - 硬件资源仲裁                           ││
         │  │  - Mailbox 路由                          ││
         │  └─────────────────────────────────────────┘│
         │  ┌──────────┐ ┌──────────┐ ┌──────────┐    ││
         │  │ PSP      │ │ SMU      │ │ DMCUB    │    ││
         │  │安全处理器 │ │电源管理  │ │显示控制  │    ││
         │  └──────────┘ └──────────┘ └──────────┘    ││
         └─────────────────────────────────────────────┘
```

#### VF 生命周期

AMDGPU VF 的生命周期分为五个阶段：

```
VF 生命周期状态机：

                ┌─────────────────┐
                │    VF_DISABLED   │
                │  (未创建/已销毁)  │
                └────────┬────────┘
                         │ PF 创建 VF (sriov_numvfs)
                         ▼
                ┌─────────────────┐
                │    VF_CREATED    │
                │  PCIe 枚举完成   │
                │  BAR 已映射      │
                └────────┬────────┘
                         │ VF 驱动加载
                         ▼
                ┌─────────────────┐
                │   VF_INIT       │◄──────────────┐
                │  - 固件加载      │               │
                │  - 内存初始化    │               │
                │  - 引擎配置      │               │
                └────────┬────────┘               │
                         │ 初始化成功               │
                         ▼                        │
                ┌─────────────────┐               │
                │  VF_ACTIVE      │               │
                │  - 命令提交      │               │
                │  - 内存操作      │               │
                │  - 中断处理      │               │
                └────────┬────────┘               │
                         │ VF 错误或关闭            │
                         ▼                        │
                ┌─────────────────┐               │
                │   VF_ERROR      │───────────────┘
                │  - 错误记录      │  (可恢复错误尝试重初始化)
                │  - 资源清理      │
                └────────┬────────┘
                         │ PF 回收
                         ▼
                ┌─────────────────┐
                │   VF_DISABLED   │
                └─────────────────┘
```

每个阶段的详细操作：

```c
// Linux 内核中 AMDGPU VF 生命周期管理的核心结构
// 源码路径: drivers/gpu/drm/amd/amdgpu/amdgpu_virt.h

enum amdgpu_vf_state {
    AMDGPU_VF_DISABLED = 0,   // VF 未创建
    AMDGPU_VF_CREATED,        // VF 已创建(PCIe 可见)
    AMDGPU_VF_INIT,           // VF 正在初始化
    AMDGPU_VF_ACTIVE,         // VF 正常运行
    AMDGPU_VF_ERROR,          // VF 错误状态
};

struct amdgpu_vf_manager {
    // VF 管理数据结构
    enum amdgpu_vf_state    vf_state[AMDGPU_MAX_VFS];
    uint64_t                vram_base[AMDGPU_MAX_VFS];  // VF 显存基址
    uint64_t                vram_size[AMDGPU_MAX_VFS];  // VF 显存大小
    uint32_t                gfx_ring_mask;               // GFX ring 分配位图
    uint32_t                sdma_ring_mask;              // SDMA ring 分配位图
    
    // Mailbox 通信
    struct mutex            mbox_lock;
    uint32_t                mbox_msg[AMDGPU_MAX_VFS];
    
    // PF 控制接口
    struct work_struct      vf_work[AMDGPU_MAX_VFS];
};
```

#### VF 创建流程（PF 侧）

当向 sysfs 写入 sriov_numvfs 时，内核 PCI 子系统调用 PF 驱动的 .sriov_configure() 回调：

```c
// PF 侧 VF 创建流程
// 简化的调用链：

// 1. PCI 子系统的入口
pci_iov_add_vf() → pci_enable_sriov() → driver->sriov_configure()

// 2. AMDGPU PF 驱动实现
int amdgpu_pci_sriov_configure(struct pci_dev *pdev, int num_vfs)
{
    struct amdgpu_device *adev = pci_get_drvdata(pdev);
    int ret;
    
    if (num_vfs == 0) {
        // 销毁所有 VF
        return amdgpu_virt_release_vfs(adev);
    }
    
    // 创建 VF
    ret = amdgpu_virt_alloc_vfs(adev, num_vfs);
    if (ret)
        return ret;
    
    // VF 创建后的初始化
    for (int i = 0; i < num_vfs; i++) {
        ret = amdgpu_virt_init_vf(adev, i);
        if (ret) {
            amdgpu_virt_release_vfs(adev);
            return ret;
        }
    }
    
    return num_vfs;
}

// 3. VF 资源分配
int amdgpu_virt_alloc_vfs(struct amdgpu_device *adev, int num_vfs)
{
    struct amdgpu_vf_manager *manager = adev->virt.vf_manager;
    uint64_t total_vram = amdgpu_vram_mgr_available(adev);
    uint64_t vf_vram_size = (total_vram * 0.8) / num_vfs;  // 80% 显存分给 VF
    
    for (int i = 0; i < num_vfs; i++) {
        // 分配显存区域
        manager->vram_base[i] = i * vf_vram_size;
        manager->vram_size[i] = vf_vram_size;
        
        // 分配 GFX ring（轮询分配）
        int ring_id = ffz(manager->gfx_ring_mask);
        manager->gfx_ring_mask |= (1 << ring_id);
        
        // 分配 SDMA 引擎
        int sdma_id = ffz(manager->sdma_ring_mask);
        manager->sdma_ring_mask |= (1 << sdma_id);
        
        // 初始化 VF 状态
        manager->vf_state[i] = AMDGPU_VF_CREATED;
    }
    
    return 0;
}
```

#### PF-VF 通信：Mailbox 协议

VF 不能直接访问 PF 的寄存器和控制接口，需要通过 Mailbox 机制进行通信：

```
Mailbox 通信架构：

┌─────────────────┐            ┌─────────────────┐
│    VF 驱动       │            │    PF 驱动       │
│                  │            │                  │
│  ┌────────────┐  │            │  ┌────────────┐  │
│  │  命令封装   │  │            │  │  命令处理   │  │
│  │  (msg类型)  │  │            │  │  (dispatch) │  │
│  └──────┬─────┘  │            │  └──────┬─────┘  │
│         │        │            │         │        │
│  ┌──────┴─────┐  │            │  ┌──────┴─────┐  │
│  │ Mailbox 写  │  │            │  │ Mailbox 读  │  │
│  │ (MMIO写)   │──┼────────────┼─▶│ (中断触发)  │  │
│  └────────────┘  │            │  └────────────┘  │
│         │        │            │         │        │
│  ┌──────┴─────┐  │            │  ┌──────┴─────┐  │
│  │  轮询等待   │  │            │  │  结果回复   │  │
│  │  (忙等待)   │◀─┼────────────┼──│ (MMIO写回)  │  │
│  └────────────┘  │            │  └────────────┘  │
└─────────────────┘            └─────────────────┘
```

Mailbox 消息格式：

```c
// AMDGPU Mailbox 消息定义
// 源码路径: drivers/gpu/drm/amd/amdgpu/amdgpu_virt.h

#define AMDGPU_MBOX_MSG_CMD_MASK    0x0000FFFF  // 命令码掩码
#define AMDGPU_MBOX_MSG_VF_ID_MASK  0x00FF0000  // VF ID 掩码
#define AMDGPU_MBOX_MSG_STATUS_MASK 0xFF000000  // 状态掩码

// 命令类型定义
enum amdgpu_mbox_msg_type {
    // 内存管理命令
    AMDGPU_MBOX_MSG_ALLOC_VRAM    = 0x01,  // 分配显存
    AMDGPU_MBOX_MSG_FREE_VRAM     = 0x02,  // 释放显存
    AMDGPU_MBOX_MSG_ALLOC_GTT     = 0x03,  // 分配 GTT 内存
    AMDGPU_MBOX_MSG_FREE_GTT      = 0x04,  // 释放 GTT 内存
    AMDGPU_MBOX_MSG_MAP_TABLE     = 0x05,  // 映射页表
    
    // 状态查询命令
    AMDGPU_MBOX_MSG_QUERY_VRAM    = 0x10,  // 查询显存状态
    AMDGPU_MBOX_MSG_QUERY_CLOCK   = 0x11,  // 查询时钟频率
    AMDGPU_MBOX_MSG_QUERY_TEMP    = 0x12,  // 查询温度
    
    // 控制命令
    AMDGPU_MBOX_MSG_GPU_RESET     = 0x20,  // 请求 GPU 重置
    AMDGPU_MBOX_MSG_FLUSH_TLB     = 0x21,  // 刷新 TLB
    AMDGPU_MBOX_MSG_UPDATE_PGTBL  = 0x22,  // 更新页表
    
    // 错误报告
    AMDGPU_MBOX_MSG_REPORT_ERR    = 0x30,  // 报告错误
    AMDGPU_MBOX_MSG_REPORT_HANG   = 0x31,  // 报告 hang
    
    // 响应状态
    AMDGPU_MBOX_MSG_SUCCESS       = 0x80,  // 成功
    AMDGPU_MBOX_MSG_FAILURE       = 0x81,  // 失败
    AMDGPU_MBOX_MSG_RETRY         = 0x82,  // 需要重试
};

// Mailbox 操作函数
struct amdgpu_mbox_ops {
    int (*send_msg)(struct amdgpu_device *adev, 
                    uint32_t msg, uint32_t param);
    int (*recv_msg)(struct amdgpu_device *adev, 
                    uint32_t *msg, uint32_t *param);
    int (*wait_response)(struct amdgpu_device *adev, 
                         uint32_t timeout_ms);
};

// VF 侧发送 Mailbox 消息的简化实现
int amdgpu_virt_mbox_send(struct amdgpu_device *adev, 
                          uint32_t cmd, uint32_t param)
{
    struct amdgpu_vf_manager *manager = adev->virt.vf_manager;
    uint32_t msg = cmd | (adev->virt.vf_id << 16);
    int ret;
    
    mutex_lock(&manager->mbox_lock);
    
    // 1. 写入 Mailbox 寄存器
    WREG32(mmMAILBOX_CMD_ADDR, msg);
    WREG32(mmMAILBOX_DATA0, param);
    
    // 2. 通知 PF（写 TRIGGER 寄存器）
    WREG32(mmMAILBOX_TRIGGER, 1);
    
    // 3. 等待 PF 响应（超时 5 秒）
    ret = amdgpu_virt_mbox_wait(adev, 5000);
    
    // 4. 读取响应
    uint32_t response = RREG32(mmMAILBOX_RESPONSE);
    
    mutex_unlock(&manager->mbox_lock);
    
    return (response == AMDGPU_MBOX_MSG_SUCCESS) ? 0 : -EIO;
}

// PF 侧 Mailbox 中断处理
void amdgpu_virt_mbox_irq_handler(struct amdgpu_device *adev)
{
    uint32_t msg = RREG32(mmMAILBOX_CMD_ADDR);
    uint32_t param = RREG32(mmMAILBOX_DATA0);
    uint32_t vf_id = (msg & AMDGPU_MBOX_MSG_VF_ID_MASK) >> 16;
    uint32_t cmd = msg & AMDGPU_MBOX_MSG_CMD_MASK;
    
    switch (cmd) {
    case AMDGPU_MBOX_MSG_ALLOC_VRAM:
        // 为 VF 分配显存
        amdgpu_virt_alloc_vram_for_vf(adev, vf_id, param);
        break;
    case AMDGPU_MBOX_MSG_QUERY_CLOCK:
        // 返回当前时钟频率
        WREG32(mmMAILBOX_RESPONSE, adev->pm.current_sclk);
        break;
    case AMDGPU_MBOX_MSG_GPU_RESET:
        // 调度 GPU 重置
        schedule_work(&adev->virt.vf_work[vf_id]);
        break;
    default:
        WREG32(mmMAILBOX_RESPONSE, AMDGPU_MBOX_MSG_FAILURE);
        break;
    }
}
```

#### VF 内存管理模型

VF 模式下的内存管理比 PF 模式复杂，因为 VF 不能直接访问 GPU 的全局页表：

```
VF 内存管理层次：

┌─────────────────────────────────────────────────────┐
│                    VF 进程 (VM Guest)                  │
│  ┌────────────────┐  ┌────────────────┐              │
│  │  VRAM 分配      │  │  GTT 分配       │              │
│  │  amdgpu_bo      │  │  amdgpu_bo      │              │
│  └───────┬────────┘  └───────┬────────┘              │
│          │                   │                        │
│  ┌───────┴────────┐  ┌───────┴────────┐              │
│  │ VF 页表 (GART)  │  │  系统内存       │              │
│  │ (VF 本地管理)   │  │ (通过 IOMMU)    │              │
│  └───────┬────────┘  └───────┬────────┘              │
└──────────┼───────────────────┼────────────────────────┘
           │                   │
           │    Mailbox        │
           │    ALLOC_VRAM     │
           ▼                   ▼
┌─────────────────────────────────────────────────────┐
│                    PF 驱动                            │
│  ┌─────────────────────────────────────────────────┐│
│  │  显存管理器 (VRAM Manager)                       ││
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐        ││
│  │  │ VF0:     │ │ VF1:     │ │ PF:      │        ││
│  │  │ 0-4GB    │ │ 4-8GB    │ │ 8-16GB   │        ││
│  │  └──────────┘ └──────────┘ └──────────┘        ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │  GTT 管理器 (GTT Manager)                       ││
│  │  - 每个 VF 独立的 GTT 分区                       ││
│  │  - 通过 IOMMU 映射到 VF 地址空间                  ││
│  └─────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

VF 内存分配流程（通过 Mailbox）：

```c
// VF 侧：请求显存分配
int amdgpu_virt_alloc_vram(struct amdgpu_device *adev, 
                           uint64_t size, 
                           uint64_t *vram_offset)
{
    int ret;
    
    // 1. 通过 Mailbox 请求 PF 分配显存
    ret = amdgpu_virt_mbox_send(adev, 
                                AMDGPU_MBOX_MSG_ALLOC_VRAM, 
                                (uint32_t)size);
    if (ret)
        return ret;
    
    // 2. 读取 PF 返回的显存偏移
    *vram_offset = RREG32(mmMAILBOX_DATA0);
    
    return 0;
}

// PF 侧：处理 VF 显存分配请求
int amdgpu_virt_alloc_vram_for_vf(struct amdgpu_device *adev,
                                   int vf_id,
                                   uint64_t size)
{
    struct amdgpu_vf_manager *manager = adev->virt.vf_manager;
    uint64_t base = manager->vram_base[vf_id];
    uint64_t max_size = manager->vram_size[vf_id];
    
    if (size > max_size)
        return -ENOMEM;
    
    // 在 VF 的显存分区中分配
    // 写入响应寄存器
    WREG32(mmMAILBOX_DATA0, (uint32_t)(base & 0xFFFFFFFF));
    WREG32(mmMAILBOX_DATA1, (uint32_t)(base >> 32));
    WREG32(mmMAILBOX_RESPONSE, AMDGPU_MBOX_MSG_SUCCESS);
    
    return 0;
}
```

#### VF 命令提交

VF 使用自己的 ring buffer 提交 GPU 命令，硬件调度器负责仲裁：

```
VF 命令提交路径：

VF 用户空间 (Mesa/ROCm)
    │
    ▼
VF 内核驱动
    │
    ├─▶ GFX Ring (专用 ring)
    │   └─▶ 硬件调度器 (HWS)
    │       ├─▶ 时间片轮转 (Round Robin)
    │       └─▶ 优先级调度 (如配置)
    │
    ├─▶ SDMA Ring (专用通道)
    │   └─▶ SDMA 硬件仲裁
    │
    └─▶ VCN Ring (共享队列)
        └─▶ 固件调度
```

```c
// VF 命令提交的核心操作
struct amdgpu_ring *vf_get_ring(struct amdgpu_device *adev, int ring_type)
{
    struct amdgpu_vf_manager *manager = adev->virt.vf_manager;
    int ring_id;
    
    // VF 只能使用分配给自己的 ring
    switch (ring_type) {
    case AMDGPU_RING_TYPE_GFX:
        ring_id = ffs(manager->gfx_ring_mask) - 1;
        break;
    case AMDGPU_RING_TYPE_SDMA:
        ring_id = ffs(manager->sdma_ring_mask) - 1;
        break;
    default:
        return -EINVAL;
    }
    
    return &adev->rings[ring_id];
}

// VF 提交命令前需要刷新 TLB
int amdgpu_virt_submit_ib(struct amdgpu_ring *ring,
                           struct amdgpu_ib *ib,
                           struct amdgpu_job *job)
{
    // 1. 通过 Mailbox 请求 TLB 刷新
    amdgpu_virt_mbox_send(ring->adev, 
                          AMDGPU_MBOX_MSG_FLUSH_TLB, 0);
    
    // 2. 写入 ring buffer
    amdgpu_ring_emit_ib(ring, ib);
    
    // 3. 更新写指针 (WPTR) 通知硬件
    amdgpu_ring_commit(ring);
    
    return 0;
}
```

#### VF 驱动功能限制

VF 驱动是 PF 驱动的轻量级版本，功能有明确限制：

```
VF 驱动 vs PF 驱动功能对比：

功能模块           PF 驱动          VF 驱动         原因
──────────────────────────────────────────────────────────
命令提交           完整支持          完整支持         VF 需要执行计算任务
显存分配           完整支持          Mailbox 代理     资源由 PF 统一管理
时钟频率控制       完整支持          不支持           电源管理由 PF 控制
电源管理           完整支持          只读查询         避免电源状态冲突
GPU 重置           完整支持          Mailbox 请求     防止影响其他 VF
OverDrive          完整支持          不支持           稳定性和公平性
RAS 管理           完整支持          只读报告         统一错误处理
性能计数器         完整支持          本 VF 范围       隔离性要求
显示输出           完整支持          不支持           数据中心场景无显示
固件更新           完整支持          不支持           安全和一致性
Debugfs 接口       完整接口          受限接口         安全和隔离
```

### 实践操作

#### 检查和操作 VF 管理接口

```bash
# 1. 查看 PF 驱动的 VF 管理功能
cat /sys/bus/pci/devices/0000:03:00.0/sriov_totalvfs
cat /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs

# 2. 查看 VF 的 mailbb

# 查看 VF 的 mailbox 状态（需要 debugfs 支持）
if [ -d /sys/kernel/debug/amdgpu ]; then
    cat /sys/kernel/debug/dri/0/amdgpu_mailbox_info
fi

# 3. 查看 VF 的虚拟化状态
cat /sys/kernel/debug/dri/0/amdgpu_virt_info

# 输出示例：
# Virtualization: SR-IOV
# VF ID: 0
# PF BDF: 0000:03:00.0
# Mailbox status: ready
# VF state: ACTIVE

# 4. 查看 VF 的资源分配
cat /sys/kernel/debug/dri/0/amdgpu_vram_usage

# 输出示例：
# Total VRAM: 8192 MB
# Used VRAM:  2048 MB
# Free VRAM:  6144 MB
# VRAM Base:  0x100000000
# VRAM Size:  0x200000000
```

#### 模拟 VF 驱动限制

```bash
# 以下操作在 VF 中会被拒绝
# 1. 尝试修改时钟频率（应返回权限错误）
echo "manual" > /sys/class/drm/card0/device/power_dpm_force_performance_level
# 预期：bash: echo: write error: Operation not permitted

# 2. 尝试 GPU 重置
echo 1 > /sys/kernel/debug/dri/0/amdgpu_gpu_recover
# 预期：bash: echo: write error: Operation not permitted

# 3. 尝试设置 OverDrive
echo "s 0 2500" > /sys/class/drm/card0/device/pp_od_clk_voltage
# 预期：bash: echo: write error: Operation not permitted

# 4. VF 中允许的操作
# 查看时钟（只读）
cat /sys/class/drm/card0/device/pp_dpm_sclk

# 查看温度（只读）
cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input

# 查看显存使用（只读）
cat /sys/class/drm/card0/device/mem_info_vram_total
cat /sys/class/drm/card0/device/mem_info_vram_used
```

#### VF 性能隔离验证

```bash
#!/bin/bash
# 文件名: vf_isolation_test.sh
# 用途: 验证 VF 间的性能和资源隔离

VF_PIDS=()
declare -A VF_BDFS

# 配置
GPU_BDF="0000:03:00.0"
VF_COUNT=2
TEST_DURATION=30
BENCHMARK="./gpu_benchmark"

setup_vfs() {
    echo "创建 ${VF_COUNT} 个 VF..."
    
    # 确保无 VF 存在
    echo 0 > /sys/bus/pci/devices/${GPU_BDF}/sriov_numvfs
    sleep 1
    
    # 创建 VF
    echo ${VF_COUNT} > /sys/bus/pci/devices/${GPU_BDF}/sriov_numvfs
    sleep 2
    
    # 记录 VF BDF
    for vf in $(ls /sys/bus/pci/devices/${GPU_BDF}/virtfn*); do
        vf_bdf=$(basename $(readlink ${vf}))
        vf_id=${vf##*virtfn}
        VF_BDFS[${vf_id}]=${vf_bdf}
        echo "  VF ${vf_id}: ${vf_bdf}"
    done
}

test_vram_isolation() {
    echo ""
    echo "=== 显存隔离测试 ==="
    
    # 在 VF0 中分配大量显存
    echo "  VF0: 分配 4GB 显存..."
    # 假设有工具可以操作 VF
    # vf_exec ${VF_BDFS[0]} "./gpu_mem_alloc 4096"
    
    # 在 VF1 中检查可用显存
    echo "  VF1: 检查可用显存..."
    # vf_exec ${VF_BDFS[1]} "cat /sys/class/drm/card1/device/mem_info_vram_free"
    
    echo "  预期: VF1 的可用显存不应比 VF0 分配前明显减少"
}

test_compute_isolation() {
    echo ""
    echo "=== 计算隔离测试 ==="
    
    echo "  测试场景: 单 VF vs 多 VF 性能对比"
    
    # 单 VF 基准测试
    # vf_exec ${VF_BDFS[0]} "${BENCHMARK}" &
    # VF_PIDS+=($!)
    
    # 等待单 VF 测试完成
    # wait ${VF_PIDS[0]}
    
    # 多 VF 并发测试
    for vf_id in "${!VF_BDFS[@]}"; do
        echo "  启动 VF${vf_id} 并发测试..."
        # vf_exec ${VF_BDFS[${vf_id}]} "${BENCHMARK} --duration ${TEST_DURATION}" &
        VF_PIDS+=($!)
    done
    
    # 监控性能
    echo "  监控性能..."
    for ((i=0; i<TEST_DURATION; i++)); do
        # 读取各 VF 的性能计数器
        for vf_id in "${!VF_BDFS[@]}"; do
            : # 性能采样逻辑
        done
        sleep 1
    done
    
    # 停止测试
    for pid in "${VF_PIDS[@]}"; do
        kill ${pid} 2>/dev/null
    done
    
    echo "  预期: 多 VF 并发时，各 VF 性能差异 < 15%"
}

test_error_isolation() {
    echo ""
    echo "=== 错误隔离测试 ==="
    
    # 在 VF0 中触发错误
    echo "  VF0: 注入 GPU hang..."
    # vf_exec ${VF_BDFS[0]} "echo 1 > /sys/kernel/debug/dri/0/amdgpu_ring_hang"
    
    sleep 5
    
    # 检查 VF1 是否受影响
    echo "  VF1: 检查运行状态..."
    # vf_exec ${VF_BDFS[1]} "./gpu_benchmark --quick"
    
    echo "  预期: VF0 hang 不影响 VF1 的正常运行"
}

cleanup() {
    echo ""
    echo "清理 VF..."
    echo 0 > /sys/bus/pci/devices/${GPU_BDF}/sriov_numvfs
    echo "完成"
}

# 主流程
echo "AMDGPU VF 隔离性测试"
echo "===================="
echo "开始时间: $(date)"

setup_vfs
test_vram_isolation
test_compute_isolation
test_error_isolation
cleanup

echo ""
echo "测试完成: $(date)"
```

#### VF 驱动调试

```bash
# 1. 启用 VF 调试日志
echo "0x1ff" > /sys/module/amdgpu/parameters/debug_large
echo "0x1ff" > /sys/module/amdgpu/parameters/debug_small

# 2. 查看 VF 初始化日志
dmesg | grep "amdgpu.*VF\|amdgpu.*virt"

# 3. 查看 Mailbox 通信日志
dmesg | grep "mbox\|MAILBOX\|virt.*msg"

# 输出示例：
# [  123.456] amdgpu 0000:03:00.1: virt: mbox send msg 0x01 param 0x10000000
# [  123.457] amdgpu 0000:03:00.1: virt: mbox recv response 0x80
# [  124.890] amdgpu 0000:03:00.1: virt: VRAM allocated at offset 0x200000000

# 4. 检查 VF 状态寄存器
# VF 状态寄存器（调试用）
# 读取 VF 的状态信息
lspci -vvv -s 03:00.1 | grep -A 20 "Capabilities"

# 5. 追踪 VF 内存操作
# 使用 ftrace 追踪 VF 内存分配
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo amdgpu_virt_alloc_vram > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... 执行 VF 操作 ...
cat /sys/kernel/debug/tracing/trace | head -50
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

### 案例分析

#### 案例一：VF 初始化失败

**问题现象：** 创建 VF 后，VF 驱动无法完成初始化，dmesg 显示 init 失败。

```bash
# dmesg 日志
[  456.789] amdgpu 0000:03:00.1: Fatal error during VF init
[  456.790] amdgpu 0000:03:00.1: amdgpu_virt_request_vram failed: -ENOMEM
[  456.791] amdgpu 0000:03:00.1: amdgpu_device_init failed
```

**诊断过程：**

```bash
# 1. 检查 PF 侧显存状态
cat /sys/class/drm/card0/device/mem_info_vram_used
cat /sys/class/drm/card0/device/mem_info_vram_total

# 2. 检查显存碎片
cat /sys/kernel/debug/dri/0/amdgpu_vram_mm

# 3. 检查 VF 配置的分区大小
cat /sys/bus/pci/devices/0000:03:00.0/vram_partition 2>/dev/null
```

**根本原因：** PF 在创建 VF 时已分配显存分区，但 PF 自身的显存占用过高，导致 VF 分区无法满足最低需求。

**解决方案：**

```bash
# 1. 释放 PF 占用的显存
# 停止使用 PF 的应用程序
sudo systemctl stop gdm  # 停止显示管理器
sudo modprobe -r amdgpu
sudo modprobe amdgpu sriov=1

# 2. 重新创建 VF
echo 0 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs
echo 2 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs

# 3. 验证 VF 初始化
dmesg | tail -10
# 预期：VF 初始化成功
```

#### 案例二：Mailbox 通信超时

**问题现象：** VF 提交命令时频繁出现 mailbox 超时错误。

```bash
# dmesg 日志
[  789.012] amdgpu 0000:03:00.1: virt: mbox timeout waiting for response
[  789.013] amdgpu 0000:03:00.1: virt: mbox msg 0x05 (MAP_TABLE) failed
[  789.014] amdgpu 0000:03:00.1: SDMA IB test failed
```

**诊断过程：**

```bash
# 1. 检查 Mailbox 中断
cat /proc/interrupts | grep amdgpu
# 检查 VF 对应的中断计数是否在增长

# 2. 检查 PF 的 Mailbox 处理队列
cat /sys/kernel/debug/dri/0/amdgpu_mailbox_stats

# 3. 检查 PF 是否处于繁忙状态（导致无法及时处理 mailbox）
cat /sys/class/drm/card0/device/gpu_busy_percent
```

**根本原因：** PF GPU 繁忙（98% 占用），导致 Mailbox 中断响应延迟，VF 请求超时。

**解决方案：**

```bash
# 1. 限制 PF 的 GPU 使用率
echo "low" > /sys/class/drm/card0/device/power_dpm_force_performance_level

# 2. 增加 Mailbox 超时时间
# 通过模块参数
echo "5000" > /sys/module/amdgpu/parameters/virt_mbox_timeout

# 3. 优化 VF 提交频率（减少不必要的 Mailbox 请求）
# 在 VF 驱动中合并页表更新操作

# 4. 长期方案：配置 QoS 确保 PF 有足够的 CPU 时间处理 Mailbox
```

### 相关链接

- [AMDGPU SR-IOV 驱动源码（amdgpu_virt.c）](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/amdgpu_virt.c)
- [AMDGPU SR-IOV 头文件（amdgpu_virt.h）](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/amdgpu_virt.h)
- [Linux 内核 SR-IOV 文档](https://www.kernel.org/doc/html/latest/PCI/pci-iov-howto.html)
- [AMD GPU SR-IOV 白皮书](https://www.amd.com/en/technologies/sr-iov)
- [KVM SR-IOV GPU 直通指南](https://www.linux-kvm.org/page/How_to_assign_devices_with_SR-IOV)

### 今日小结

AMDGPU SR-IOV 的 VF 管理是一个多层次的复杂系统：

1. **生命周期管理**：VF 经过 DISABLED → CREATED → INIT → ACTIVE → ERROR 状态转换
2. **资源分配**：显存采用静态分区，引擎采用动态分配
3. **PF-VF 通信**：Mailbox 机制是 VF 请求 PF 服务的唯一通道
4. **功能限制**：VF 驱动是 PF 驱动的轻量子集，关键操作需通过 PF 代理
5. **命令提交**：VF 有独立的 ring buffer，硬件调度器负责仲裁
6. **隔离性**：通过硬件和软件两层确保 VF 间隔离

### 扩展思考

1. 如果 Mailbox 通信延迟过高，对 VF 的计算性能有何影响？如何优化？
2. 在大规模部署场景中，如何监控和诊断 VF 的健康状态？
3. GPU SR-IOV 的 VF 密度受哪些因素限制（显存、引擎数量、PCIe 带宽）？
4. 如何在 VF 环境下实现类似 NVIDIA MIG 的细粒度资源隔离？
