# GPU 寄存器映射与访问方式（MMIO / SMN / PCIE）

## 学习目标

- 深入理解 MMIO、SMN、PCIE Config Space 三种寄存器访问方式的原理和差异
- 掌握 AMDGPU 驱动中三种访问方式的代码实现和适用场景
- 理解 GPU 地址空间映射和寄存器偏移计算
- 掌握寄存器访问性能特征和调试技巧

## 知识详解

### 概念原理

#### GPU 寄存器映射总体架构

```ascii
GPU 寄存器访问层次结构:

┌──────────────────────────────────────────────────────────────┐
│  CPU (x86)                                                   │
│                                                               │
│  ┌──────────────────────────────────────────────┐            │
│  │  CPU 核心                                       │            │
│  │  mov reg, [mmio_base + offset]  ← 直接访问      │            │
│  └──────────────────┬───────────────────────────┘            │
│                     │                                         │
│  ┌──────────────────▼───────────────────────────┐            │
│  │  PCI Express Root Complex                       │            │
│  │  - MMIO 事务: 直接路由到 GPU BAR               │            │
│  │  - SMN 事务: 通过 SMU (System Management Unit) │            │
│  │  - PCIE CFG: 通过配置空间访问                  │            │
│  └──────────────────┬───────────────────────────┘            │
│                     │                                         │
├─────────────────────┼─────────────────────────────────────────┤
│  GPU                │                                         │
│  ┌──────────────────▼───────────────────────────┐            │
│  │  PCIe Core / IP                                  │            │
│  │                                                   │            │
│  │  ┌──────────────┐  ┌───────────┐  ┌───────────┐  │            │
│  │  │ BAR0 (MMIO)  │  │ SMN Bus  │  │ Config    │  │            │
│  │  │ 256KB - 2MB  │  │ 完整GPU   │  │ Space     │  │            │
│  │  │ 直接映射      │  │ 地址空间    │  │ 4KB        │  │            │
│  │  └──────┬───────┘  └─────┬─────┘  └─────┬─────┘  │            │
│  └─────────┼────────────────┼───────────────┼────────┘            │
│            │                │               │                      │
│  ┌─────────▼────────────────▼───────────────▼──────────────────┐  │
│  │  GPU 内部互联 (AID/DF/IF)                                    │  │
│  │                                                                │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐           │  │
│  │  │GFX  │ │MMHUB│ │DCN  │ │VCN  │ │SDMA │ │SMU  │  ...       │  │
│  │  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘           │  │
│  │     │       │       │       │       │       │                │  │
│  │     └───────┴───────┴───────┴───────┴───────┘                │  │
│  │              IP 寄存器分布在不同地址段                          │  │
│  └──────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

#### MMIO (Memory Mapped I/O) 访问

```ascii
MMIO 访问机制:

┌──────────────────────────────────────────────────────────────┐
│ BAR0 (Base Address Register 0)                                │
│                                                               │
│ 物理地址空间:                                                 │
│ ┌──────────────────────────────────────────────────────────┐  │
│ │ CPU 物理地址: 0xF000_0000_0000 - 0xF000_001F_FFFF       │  │
│ │               (由 BIOS 分配, 在 PCI enumeration 时设置)   │  │
│ │                                                          │  │
│ │ ioremap() 后:                                            │  │
│ │ 虚拟地址: 0xFFFF_8000_0000_0000 - 0xFFFF_8000_001F_FFFF  │  │
│ │               (内核虚拟地址空间)                            │  │
│ └──────────────────────────────────────────────────────────┘  │
│                                                               │
│ MMIO 内部布局 (Navi 31 示例):                                 │
│ ┌───────────────┬───────────────┬────────────────────────────┐│
│ │ 偏移范围       │ IP 块        │ 内容                       ││
│ ├───────────────┼───────────────┼────────────────────────────┤│
│ │ 0x0000-0x3FFF │ GFX          │ 图形引擎寄存器              ││
│ │ 0x4000-0x7FFF │ MMHUB        │ 内存管理 Hub               ││
│ │ 0x8000-0xBFFF │ DCN          │ 显示控制器                  ││
│ │ 0xC000-0xFFFF │ VCN          │ 视频编解码                  ││
│ │ 0x10000-      │ SDMA         │ DMA 引擎                   ││
│ │ 0x20000-      │ SMU          │ 系统管理单元                ││
│ └───────────────┴───────────────┴────────────────────────────┘│
│                                                               │
│ MMIO 访问特性:                                                 │
│  - CPU 直接访问 (mov 指令)                                    │
│  - 延迟: ~100-500ns (取决于 PCIE 拓扑)                        │
│  - 原子性: 32-bit 读写是原子的                                 │
│  - 排序: 严格遵守 PCIE 内存排序规则                            │
│  - 大小: 通常 256KB - 2MB                                    │
└──────────────────────────────────────────────────────────────┘
```

#### SMN (System Management Network) 访问

```ascii
SMN 访问机制:

┌──────────────────────────────────────────────────────────────┐
│ SMN 是一种片上的独立总线网络，连接所有 IP 块                    │
│                                                               │
│ ┌──────────────────────────────────────────────────────────┐  │
│ │ SMN 的工作原理:                                           │  │
│ │                                                          │  │
│ │ CPU 访问 SMN 寄存器流程:                                  │  │
│ │ 1. CPU 写 SMN 地址到 SMN_ADDR 寄存器 (MMIO 偏移 0x...):  │  │
│ │    SMN_ADDR = target_smn_address                         │  │
│ │                                                          │  │
│ │ 2. CPU 读/写 SMN_DATA 寄存器                             │  │
│ │    value = SMN_DATA  (读)                                 │  │
│ │    SMN_DATA = value  (写)                                 │  │
│ │                                                          │  │
│ │ 3. 硬件自动通过 SMN 总线在目标地址执行操作                  │  │
│ │    等待 SMN 事务完成 (轮询状态或固定延迟)                  │  │
│ │                                                          │  │
│ │ SMN 地址空间:                                             │  │
│ │ ┌────────────────┬───────────┬─────────────────────────┐ │  │
│ │ │ 地址范围        │ 目标       │ 用途                    │ │  │
│ │ ├────────────────┼───────────┼─────────────────────────┤ │  │
│ │ │ 0x00000000-    │ SOC regs  │ 芯片级寄存器             │ │  │
│ │ │ 0x03FFFFFF     │           │                          │ │  │
│ │ │ 0x04000000-    │ SMU       │ 电源管理                 │ │  │
│ │ │ 0x07FFFFFF     │           │                          │ │  │
│ │ │ 0x08000000-    │ MP0/PSP   │ 安全处理器               │ │  │
│ │ │ 0x0BFFFFFF     │           │                          │ │  │
│ │ │ 0x10000000-    │ GFX       │ 图形引擎                 │ │  │
│ │ │ 0x13FFFFFF     │           │                          │ │  │
│ │ │ 0x14000000-    │ IOMMU     │ IOMMU 寄存器             │ │  │
│ │ │ 0x17FFFFFF     │           │                          │ │  │
│ │ │ 0x18000000-    │ DCN       │ 显示控制器               │ │  │
│ │ │ 0x1BFFFFFF     │           │                          │ │  │
│ │ │ ...            │           │                          │ │  │
│ │ └────────────────┴───────────┴─────────────────────────┘ │  │
│ │                                                          │  │
│ │ SMN 访问特性:                                             │  │
│ │  - 间接访问 (需要两次 MMIO 操作)                          │  │
│ │  - 延迟: ~1-10us (比 MMIO 慢 10-100x)                    │  │
│ │  - 覆盖: 完整 32-bit GPU 地址空间                         │  │
│ │  - 用途: 电源管理、时钟控制、温度传感器等                   │  │
│ │  - 32-bit 对齐访问                                        │  │
│ └──────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

#### PCIE 配置空间访问

```ascii
PCIE 配置空间:

┌──────────────────────────────────────────────────────────────┐
│ PCIE 配置空间标准 (256 字节标准 + 4KB 扩展空间)               │
│                                                               │
│ ┌───────────┬──────────────────────────────────────────────┐  │
│ │ 偏移      │ 寄存器                                        │  │
│ ├───────────┼──────────────────────────────────────────────┤  │
│ │ 0x00      │ Vendor ID (0x1002 = AMD)                     │  │
│ │ 0x02      │ Device ID (e.g., 0x7448 = Navi 31)          │  │
│ │ 0x04      │ Command / Status                             │  │
│ │ 0x08      │ Revision ID / Class Code (0x030000 = VGA)   │  │
│ │ 0x0C      │ Cache Line / Latency Timer / Header Type    │  │
│ │ 0x10      │ BAR0 - MMIO 基地址                            │  │
│ │ 0x14      │ BAR1 - VRAM (FB) 基地址                       │  │
│ │ 0x18      │ BAR2 - IO Port (可选)                         │  │
│ │ 0x1C-0x27 │ BAR3, BAR4, BAR5                              │  │
│ │ 0x28      │ Cardbus CIS Pointer                           │  │
│ │ 0x2C      │ Subsystem Vendor / Subsystem ID              │  │
│ │ 0x30      │ Expansion ROM Base Address                   │  │
│ │ 0x34      │ Capabilities Pointer                          │  │
│ │ 0x38      │ Reserved                                     │  │
│ │ 0x3C      │ Interrupt Line / Interrupt Pin                │  │
│ │ 0x3D-0x3F │ Min_Gnt / Max_Lat                            │  │
│ ├───────────┼──────────────────────────────────────────────┤  │
│ │ 0x40-0xFF │ 能力寄存器链表                                │  │
│ │           │ 0x40: PCI Express Capability                  │  │
│ │           │ 0x50: MSI Capability                          │  │
│ │           │ 0x60: Advanced Error Reporting (AER)          │  │
│ │           │ 0x80: Power Management Capability             │  │
│ │           │ 0xA0: Vendor Specific Capability (AMD)        │  │
│ │           │ 0xE0: Resizable BAR Capability                │  │
│ ├───────────┼──────────────────────────────────────────────┤  │
│ │ 0x100-    │ PCI Express Extended Configuration Space      │  │
│ │ 0xFFF     │ - AER 扩展寄存器                              │  │
│ │           │ - SR-IOV 能力                                  │  │
│ │           │ - ACS (Access Control Services)                │  │
│ │           │ - DPU (Data Processing Units)                  │  │
│ └───────────┴──────────────────────────────────────────────┘  │
│                                                               │
│ 访问方式:                                                     │
│   Linux: pci_read_config_word(pdev, offset, &value)            │
│   通过 CFG 机制: I/O 端口 0xCF8/0xCFC 或 ECAM (MMIO 映射)     │
└──────────────────────────────────────────────────────────────┘
```

#### 三种访问方式对比

```ascii
MMIO / SMN / PCIE 详细对比:

┌──────────────────────────────────────────────────────────────┐
│ 特性            │ MMIO            │ SMN           │ PCIE CFG   │
├──────────────────┼────────────────┼───────────────┼────────────┤
│ 地址空间大小     │ ~2MB (BAR0)    │ 4GB (32-bit)  │ 4KB         │
│ 访问方式         │ 直接 (mov)     │ 间接 (3步)    │ CFG cycles  │
│ 延迟             │ 100-500ns      │ 1-10us       │ 1-5us       │
│ 带宽             │ 高             │ 低            │ 低          │
│ 原子性           │ 32-bit 原子    │ 非原子        │ 非原子      │
│ 对齐要求         │ 32-bit         │ 32-bit        │ 任意        │
│ 缓存行为         │ uncacheable    │ uncacheable   │ N/A         │
│ 典型用途         │ 显示/图形      │ 电源管理      │ 设备识别    │
│ 代码封装函数     │ RREG32/WREG32  │ smn_r/w       │ pci_rd/wr   │
│ 调试接口         │ amdgpu_regs    │ smn_regs      │ pcie_cap    │
│ 安全性           │ 锁定寄存器保护 │ SMN 密码保护   │ 标准权限    │
└──────────────────────────────────────────────────────────────┘

性能特征曲线:
延迟 (log 尺度)
   10us │                          SMN
        │
    1us │                     PCIE Config
        │
  500ns │          MMIO (正常)
        │
  100ns │  MMIO (本地)
        │
        └────────────────────────────────► 寄存器位置
             本地   远端    芯片级   外部
```

### 实践操作

#### MMIO 内核实现

```c
// amdgpu_device.c - MMIO 寄存器访问
// 关键数据结构

struct amdgpu_device {
    // ...
    
    // MMIO 映射
    resource_size_t rmminfo;       // BAR0 物理地址
    void __iomem *rmmio;          // ioremap 后的虚拟地址
    u32 rmmio_size;               // BAR0 大小
    
    // 寄存器保护锁
    spinlock_t reg_lock;          // MMIO 访问保护
    
    // 寄存器访问钩子 (用于虚拟化/仿真)
    u32 (*rreg)(struct amdgpu_device *adev, u32 offset);
    void (*wreg)(struct amdgpu_device *adev, u32 offset, u32 value);
    
    // SMN 访问
    u32 (*smn_rreg)(struct amdgpu_device *adev, u32 addr);
    void (*smn_wreg)(struct amdgpu_device *adev, u32 addr, u32 value);
    
    // ...
};

// MMIO 读取函数 (默认实现)
static u32 amdgpu_mmio_rreg(struct amdgpu_device *adev, u32 offset)
{
    u32 value;
    
    // 1. 验证偏移范围
    if (WARN_ON(offset >= adev->rmmio_size))
        return 0;
    
    // 2. 内存栅栏 (确保之前的写操作完成)
    mb();
    
    // 3. 读取 MMIO 寄存器
    value = readl(adev->rmmio + offset);
    
    // 4. 内存栅栏 (确保读完成)
    mb();
    
    return value;
}

// MMIO 写入函数
static void amdgpu_mmio_wreg(struct amdgpu_device *adev, u32 offset, u32 value)
{
    // 1. 验证偏移
    if (WARN_ON(offset >= adev->rmmio_size))
        return;
    
    // 2. 内存栅栏
    wmb();
    
    // 3. 写入寄存器
    writel(value, adev->rmmio + offset);
    
    // 4. 确保写入完成 (read-back 验证)
    mb();
    readl(adev->rmmio + offset);  // flush posted write
    
    // 5. (可选) 寄存器回读验证
    if (amdgpu_debugfs_regs_replay_enabled) {
        amdgpu_debugfs_trace_reg_write(adev, offset, value);
    }
}

// 寄存器访问函数指针映射
void amdgpu_device_regs_init(struct amdgpu_device *adev)
{
    // 默认使用 MMIO
    adev->rreg = amdgpu_mmio_rreg;
    adev->wreg = amdgpu_mmio_wreg;
    
    // 虚拟化环境使用更慢但安全的访问方式
    if (adev->is_virtualization) {
        adev->rreg = amdgpu_virt_rreg;
        adev->wreg = amdgpu_virt_wreg;
    }
    
    // SMN 访问初始化
    adev->smn_rreg = amdgpu_smn_rreg;
    adev->smn_wreg = amdgpu_smn_wreg;
}
```

#### SOC15 寄存器偏移计算

```c
// soc15.h - SOC15 寄存器偏移计算宏
// AMDGPU SOC15+ 使用 IP 基地址 + 偏移的方式

// IP 块枚举
enum ip_type {
    IP_GFX     = 0,
    IP_MMHUB   = 1,
    IP_DCN     = 2,
    IP_VCN     = 3,
    IP_SDMA    = 4,
    IP_SMU     = 5,
    // ...
};

// IP 基地址结构
struct ip_base {
    uint32_t base_addr;
    uint32_t size;
};

// SOC 数据库 (每个 ASIC 不同)
struct soc15_reg_database {
    struct ip_base ip_bases[16];  // 每个 IP 的基地址和大小
};

// Navi 31 示例
static const struct soc15_reg_database navi31_reg_db = {
    .ip_bases = {
        [IP_GFX]   = { .base_addr = 0x0000, .size = 0x4000 },
        [IP_MMHUB] = { .base_addr = 0x4000, .size = 0x4000 },
        [IP_DCN]   = { .base_addr = 0x8000, .size = 0x4000 },
        [IP_VCN]   = { .base_addr = 0xC000, .size = 0x4000 },
        [IP_SDMA]  = { .base_addr = 0x10000, .size = 0x4000 },
        [IP_SMU]   = { .base_addr = 0x20000, .size = 0x4000 },
        // ...
    }
};

// 寄存器偏移计算宏
#define SOC15_REG_OFFSET(ip, inst, reg) \
    (navi31_reg_db.ip_bases[ip].base_addr + \
     (inst * 0x1000) + reg)

// 使用示例
// DCN OTG_MASTER_EN: reg偏移 0x00, 实例 0
#define mmOTG_MASTER_EN \
    SOC15_REG_OFFSET(IP_DCN, 0, 0x1A000)
// 计算结果: 0x8000 + 0 + 0x1A000 = 0x1A000

// DCN OTG_MASTER_EN 实例 1
// 计算结果: 0x8000 + 0x1000 + 0x1A000 = 0x1B000
```

#### SMN 访问实现

```c
// amdgpu_device.c - SMN 寄存器访问 (以 Navi3x 为例)

// SMN 地址寄存器 (MMIO 偏移)
#define mmSMN_ADDR   0x0000  // 需要填入实际偏移
#define mmSMN_DATA32 0x0004

// SMN 读取
u32 amdgpu_smn_rreg(struct amdgpu_device *adev, u32 addr)
{
    unsigned long flags;
    u32 value;
    
    // 1. 获取寄存器访问锁 (SMN 访问需要原子化)
    spin_lock_irqsave(&adev->smn_access_lock, flags);
    
    // 2. 写 SMN 地址
    WREG32(adev->smn_addr_reg, addr);
    
    // 3. SMN 总线传输延迟 (不同 ASIC 不同)
    // 有些使用 udelay(1), 有些需要轮询 SMN 状态寄存器
    if (adev->smn_delay_type == SMN_DELAY_FIXED) {
        udelay(adev->smn_delay_us);
    } else {
        // 轮询方式 (更可靠)
        int retries = 100;
        while (retries--) {
            if (RREG32(adev->smn_status_reg) & SMN_READY)
                break;
            udelay(1);
        }
    }
    
    // 4. 读 SMN 数据
    value = RREG32(adev->smn_data_reg);
    
    // 5. 释放锁
    spin_unlock_irqrestore(&adev->smn_access_lock, flags);
    
    return value;
}

// SMN 写入
void amdgpu_smn_wreg(struct amdgpu_device *adev, u32 addr, u32 value)
{
    unsigned long flags;
    
    spin_lock_irqsave(&adev->smn_access_lock, flags);
    
    // 1. 写 SMN 地址
    WREG32(adev->smn_addr_reg, addr);
    
    // 2. 等待 SMN 总线就绪
    if (adev->smn_delay_type == SMN_DELAY_FIXED) {
        udelay(adev->smn_delay_us);
    } else {
        int retries = 100;
        while (retries--) {
            if (RREG32(adev->smn_status_reg) & SMN_READY)
                break;
            udelay(1);
        }
    }
    
    // 3. 写 SMN 数据 (触发写入操作)
    WREG32(adev->smn_data_reg, value);
    
    // 4. 等待写入完成
    if (adev->smn_delay_type == SMN_DELAY_FIXED) {
        udelay(adev->smn_delay_us);
    } else {
        // 轮询 SMN 完成
    }
    
    spin_unlock_irqrestore(&adev->smn_access_lock, flags);
}

// SMN 地址范围检查
bool amdgpu_smn_addr_valid(u32 addr)
{
    // SMN 地址必须按 4-byte 对齐
    if (addr & 0x3)
        return false;
    
    // SMN 地址空间划分
    if (addr >= 0x00000000 && addr <= 0x03FFFFFF)
        return true;  // SOC regs
    if (addr >= 0x04000000 && addr <= 0x07FFFFFF)
        return true;  // SMU
    if (addr >= 0x08000000 && addr <= 0x0BFFFFFF)
        return true;  // MP0/PSP
    if (addr >= 0x10000000 && addr <= 0x13FFFFFF)
        return true;  // GFX
    if (addr >= 0x14000000 && addr <= 0x17FFFFFF)
        return true;  // IOMMU
    if (addr >= 0x18000000 && addr <= 0x1BFFFFFF)
        return true;  // DCN
    if (addr >= 0x1C000000 && addr <= 0x1FFFFFFF)
        return true;  // VCN
    
    return false;  // 无效地址
}
```

#### PCIE 配置空间访问

```c
// amdgpu_device.c - PCIE 配置空间访问封装

// PCIE 配置空间读取
int amdgpu_pcie_config_read(struct amdgpu_device *adev,
    u32 offset, u32 size, u32 *value)
{
    struct pci_dev *pdev = adev->pdev;
    int ret;
    
    *value = 0;
    
    switch (size) {
    case 1: {
        u8 val;
        ret = pci_read_config_byte(pdev, offset, &val);
        if (ret != PCIBIOS_SUCCESSFUL)
            return -EIO;
        *value = val;
        break;
    }
    case 2: {
        u16 val;
        ret = pci_read_config_word(pdev, offset, &val);
        if (ret != PCIBIOS_SUCCESSFUL)
            return -EIO;
        *value = val;
        break;
    }
    case 4: {
        u32 val;
        ret = pci_read_config_dword(pdev, offset, &val);
        if (ret != PCIBIOS_SUCCESSFUL)
            return -EIO;
        *value = val;
        break;
    }
    default:
        return -EINVAL;
    }
    
    return 0;
}

// 获取 BAR 信息
int amdgpu_pcie_get_bar_info(struct amdgpu_device *adev)
{
    struct pci_dev *pdev = adev->pdev;
    
    // BAR0 (MMIO)
    adev->rmminfo = pci_resource_start(pdev, 0);
    adev->rmmio_size = pci_resource_len(pdev, 0);
    
    // BAR1 (VRAM / FB)
    adev->fb_base = pci_resource_start(pdev, 1);
    adev->fb_size = pci_resource_len(pdev, 1);
    
    DRM_INFO("MMIO: 0x%llX (size: %lluMB)\n",
        adev->rmminfo, adev->rmmio_size >> 20);
    DRM_INFO("VRAM: 0x%llX (size: %lluMB)\n",
        adev->fb_base, adev->fb_size >> 20);
    
    // 验证 Resizable BAR
    u16 pcie_cap = pci_find_capability(pdev, PCI_CAP_ID_EXP);
    u32 resizable_bar;
    
    pci_read_config_dword(pdev, pcie_cap + 0xE0, &resizable_bar);
    if (resizable_bar & 0x80000000) {
        DRM_INFO("Resizable BAR enabled\n");
    }
    
    return 0;
}

// PCIE 链路状态检测
void amdgpu_pcie_link_status(struct amdgpu_device *adev)
{
    struct pci_dev *pdev = adev->pdev;
    u16 link_status, link_cap;
    u16 pcie_cap;
    
    pcie_cap = pci_find_capability(pdev, PCI_CAP_ID_EXP);
    
    // 读取链路能力
    pci_read_config_word(pdev, pcie_cap + 0x0C, &link_cap);
    int max_speed = link_cap & 0xF;
    int max_width = (link_cap >> 4) & 0x3F;
    
    // 读取链路状态
    pci_read_config_word(pdev, pcie_cap + 0x12, &link_status);
    int cur_speed = link_status & 0xF;
    int cur_width = (link_status >> 4) & 0x3F;
    
    DRM_INFO("PCIE Link: max %s x%d, current %s x%d\n",
        pcie_speed_string(max_speed), max_width,
        pcie_speed_string(cur_speed), cur_width);
    
    // 如果链路降级，发出警告
    if (cur_width < max_width || cur_speed < max_speed) {
        DRM_WARN("PCIE link degraded: expected %s x%d, got %s x%d\n",
            pcie_speed_string(max_speed), max_width,
            pcie_speed_string(cur_speed), cur_width);
    }
}

static const char *pcie_speed_string(int speed)
{
    static const char * const speeds[] = {
        "?", "Gen1", "Gen2", "Gen3", "Gen4", "Gen5", "Gen6"
    };
    if (speed < ARRAY_SIZE(speeds))
        return speeds[speed];
    return "?";
}
```

#### ioremap 与 MMIO 映射

```c
// amdgpu_device.c - BAR0 MMIO 映射

int amdgpu_device_mmio_probe(struct amdgpu_device *adev)
{
    int ret;
    
    // 1. 获取 BAR0 信息
    adev->rmminfo = pci_resource_start(adev->pdev, 0);
    adev->rmmio_size = pci_resource_len(adev->pdev, 0);
    
    if (adev->rmminfo == 0 || adev->rmmio_size == 0) {
        DRM_ERROR("BAR0 not assigned\n");
        return -ENODEV;
    }
    
    // 2. 使能 PCI 设备
    ret = pci_enable_device(adev->pdev);
    if (ret) {
        DRM_ERROR("pci_enable_device failed: %d\n", ret);
        return ret;
    }
    
    // 3. 设置 DMA 掩码 (GPU 需要 48-bit 地址)
    ret = dma_set_mask_and_coherent(&adev->pdev->dev, DMA_BIT_MASK(48));
    if (ret) {
        ret = dma_set_mask_and_coherent(&adev->pdev->dev, DMA_BIT_MASK(40));
        if (ret) {
            DRM_ERROR("No suitable DMA mask\n");
            return ret;
        }
    }
    
    // 4. 请求 MMIO 资源
    ret = pci_request_region(adev->pdev, 0, "amdgpu");
    if (ret) {
        DRM_ERROR("pci_request_region(0) failed: %d\n", ret);
        return ret;
    }
    
    // 5. ioremap BAR0 到内核虚拟地址空间
    adev->rmmio = ioremap(adev->rmminfo, adev->rmmio_size);
    if (adev->rmmio == NULL) {
        DRM_ERROR("ioremap of BAR0 failed\n");
        pci_release_region(adev->pdev, 0);
        return -ENOMEM;
    }
    
    DRM_INFO("Registered MMIO: 0x%llX-0x%llX (mapped)\n",
        adev->rmminfo,
        adev->rmminfo + adev->rmmio_size - 1);
    
    return 0;
}

void amdgpu_device_mmio_fini(struct amdgpu_device *adev)
{
    if (adev->rmmio) {
        iounmap(adev->rmmio);
        adev->rmmio = NULL;
    }
    
    pci_release_region(adev->pdev, 0);
}

// ioremap 的内存属性
// - 使用 WC (Write Combining) 模式可提高写入性能
// - 但 MMIO 寄存器通常要求 uncacheable (UC) 模式
// - UC: 每次读写直接访问硬件，无缓存
// - WC: 写入可合并，适合帧缓冲等顺序写入

static void __iomem *amdgpu_ioremap(resource_size_t addr,
    unsigned long size, bool wc)
{
    if (wc)
        return ioremap_wc(addr, size);
    return ioremap(addr, size);
}
```

#### 寄存器访问性能分析

```c
// amdgpu_device.c - 寄存器访问延迟测试

// MMIO 延迟测试
void amdgpu_test_mmio_latency(struct amdgpu_device *adev)
{
    u64 start, end;
    u32 val;
    int i;
    u64 total_ns = 0;
    
    // 执行 1000 次 MMIO 读取，计算平均延迟
    for (i = 0; i < 1000; i++) {
        // 使用时间戳计数器
        start = rdtsc();
        val = RREG32(0x8000);  // GRBM_STATUS
        end = rdtsc();
        total_ns += (end - start) * 1000 / tsc_khz;
    }
    
    u64 avg_ns = total_ns / 1000;
    DRM_INFO("MMIO read latency: %llu ns (avg of 1000 reads)\n", avg_ns);
    
    // SMN 延迟测试
    total_ns = 0;
    for (i = 0; i < 100; i++) {
        start = rdtsc();
        val = amdgpu_smn_rreg(adev, 0x13A00000);
        end = rdtsc();
        total_ns += (end - start) * 1000 / tsc_khz;
    }
    
    avg_ns = total_ns / 100;
    DRM_INFO("SMN read latency: %llu ns (avg of 100 reads)\n", avg_ns);
}

// 结果示例:
// Navi 31:
//   MMIO read latency: ~180 ns
//   SMN read latency:  ~3200 ns
//   MMIO write latency: ~120 ns (不包括 PCIE posted write)
//   SMN write latency: ~3500 ns
//
// Vega 20:
//   MMIO read latency: ~250 ns
//   SMN read latency:  ~5000 ns
```

#### 寄存器访问保护机制

```c
// amdgpu_device.c - 寄存器访问安全保护

// 1. 寄存器写保护 (防止意外修改)
struct amdgpu_reg_protection {
    u32 offset;
    u32 mask;
    u32 key;  // 解锁密钥
};

static const struct amdgpu_reg_protection protected_regs[] = {
    // 电源管理关键寄存器
    { .offset = 0x20000, .mask = 0xFFFFFFFF, .key = 0xDEADBEEF },
    // 时钟控制寄存器
    { .offset = 0x20004, .mask = 0x0000FFFF, .key = 0xCAFEBABE },
    // 复位控制寄存器
    { .offset = 0x20008, .mask = 0x00000001, .key = 0x12345678 },
    // ...
};

bool amdgpu_reg_write_allowed(struct amdgpu_device *adev,
    u32 offset, u32 value)
{
    // 检查是否在保护列表中
    for (int i = 0; i < ARRAY_SIZE(protected_regs); i++) {
        if (offset == protected_regs[i].offset) {
            // 需要正确的密钥才能写入
            u32 masked = value & protected_regs[i].mask;
            return masked == protected_regs[i].key;
        }
    }
    return true;  // 未保护的寄存器允许写入
}

// 2. 寄存器访问计数和监控
void amdgpu_reg_access_track(struct amdgpu_device *adev,
    u32 offset, bool is_write)
{
    // 记录频繁访问的寄存器 (调试)
    if (adev->reg_access_log_enabled) {
        struct reg_access_entry *entry;
        hash_for_each_possible(adev->reg_access_log,
            entry, hash_node, offset) {
            if (entry->offset == offset) {
                if (is_write)
                    entry->write_count++;
                else
                    entry->read_count++;
                return;
            }
        }
    }
}

// 3. 写入确认 (read-after-write verify)
static inline void amdgpu_reg_write_verify(struct amdgpu_device *adev,
    u32 offset, u32 expected)
{
    if (amdgpu_reg_verify_enabled) {
        u32 actual = RREG32(offset);
        if (actual != expected) {
            DRM_WARN("REG VERIFY: 0x%05X: wrote 0x%08X, read 0x%08X\n",
                offset, expected, actual);
        }
    }
}
```

### 案例分析

#### 案例 1：寄存器访问超时导致驱动 hang

```c
// amdgpu_device.c - SMN 访问超时处理

u32 amdgpu_smn_rreg_timeout(struct amdgpu_device *adev, u32 addr)
{
    unsigned long flags;
    u32 value = 0;
    int retries = 1000;  // 最大重试次数
    
    spin_lock_irqsave(&adev->smn_access_lock, flags);
    
    // 写 SMN 地址
    WREG32(adev->smn_addr_reg, addr);
    
    // 等待 SMN 就绪 (添加超时机制)
    while (retries--) {
        u32 status = RREG32(adev->smn_status_reg);
        if (status & SMN_READY) {
            value = RREG32(adev->smn_data_reg);
            spin_unlock_irqrestore(&adev->smn_access_lock, flags);
            return value;
        }
        udelay(1);
    }
    
    // 超时: 记录错误并恢复
    DRM_ERROR("SMN read timeout at 0x%08X (status=0x%08X)\n",
        addr, RREG32(adev->smn_status_reg));
    
    // 尝试 SMN 复位
    WREG32(adev->smn_reset_reg, 1);
    udelay(10);
    WREG32(adev->smn_reset_reg, 0);
    
    spin_unlock_irqrestore(&adev->smn_access_lock, flags);
    
    return 0xDEAD0000;  // 返回错误标记
}

// 问题: 休眠时 SMU 进入低功耗状态，SMN 总线挂起
// 使用 amdgpu_smn_rreg_timeout 代替 amdgpu_smn_rreg
// 避免无限等待导致的系统 hang
```

#### 案例 2：MMIO 缓存一致性

```c
// amdgpu_device.c - MMIO 与 PCIE 排序问题

void amdgpu_device_flush_mmio(struct amdgpu_device *adev)
{
    // 1. 写内存栅栏
    wmb();
    
    // 2. 读一个无害的寄存器来 flush PCIe posted writes
    //    这个读操作会等待所有之前写入完成
    (void)RREG32(0x8000);  // GRBM_STATUS
    
    // 3. 验证
    mb();
}

// 使用场景: 在关键的寄存器序列后调用
// 例如: 模式设置完成后
void amdgpu_dm_commit_planes(struct dc *dc, ...)
{
    // ... 配置平面寄存器 ...
    
    // 确保所有寄存器写入完成
    amdgpu_device_flush_mmio(dc->adev);
    
    // 触发 VUpdate
    WREG32(mmOTG_V_UPDATE, 1);
}
```

### 相关链接

- Linux PCI 子系统: https://docs.kernel.org/PCI/pci.html
- Linux MMIO 和 ioremap: https://docs.kernel.org/core-api/mm-api.html
- AMDGPU 寄存器访问代码: drivers/gpu/drm/amd/amdgpu/amdgpu_device.c
- AMDGPU SOC15 寄存器定义: drivers/gpu/drm/amd/amdgpu/soc15.h
- PCIE Base Spec: https://pcisig.com/specifications

### 今日小结

1. **三种寄存器访问方式**：MMIO（直接，快速，有限空间）、SMN（间接，慢速，全覆盖）、PCIE Config（标准接口，用于设备标识和链路管理）。

2. **地址映射**：BAR0 通过 ioremap 映射到内核虚拟地址空间，提供 ~2MB 的 MMIO 空间；SMN 使用 32-bit 地址覆盖完整 GPU 空间。

3. **性能差异**：MMIO ~180ns，SMN ~3.2us，PCIE Config ~1-5us。SMN 比 MMIO 慢约 20 倍。

4. **保护机制**：关键寄存器有写保护（需要密钥），SMN 访问有超时机制，寄存器写入可开启回读验证。

5. **调试接口**：debugfs 提供了三种访问方式的用户空间接口，umr 提供更高级的调试能力。

### 扩展思考

1. **Resizable BAR 影响**：启用 Resizable BAR (Above 4G Decoding) 后，MMIO 映射会扩大，对寄存器访问延迟有何影响？

2. **原子性保证**：在多核系统中，GPU 寄存器访问的原子性如何保证？spinlock 保护是否充分？

3. **PCIE AtomicOp**：PCIE 规范定义了 AtomicOp 事务（FetchAdd、Swap、CAS），AMDGPU 驱动是否使用这些操作来优化寄存器访问？

4. **未来演进**：随着 CXL (Compute Express Link) 的普及，GPU 寄存器访问方式会发生怎样的变化？