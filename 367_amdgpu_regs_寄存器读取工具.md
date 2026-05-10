# amdgpu_regs 寄存器读取工具

## 学习目标

- 理解 amdgpu_regs 工具的定位和作用，掌握其在内核调试工具链中的位置
- 掌握使用 amdgpu_regs 读取 GPU 寄存器的基本方法和高级用法
- 理解 MMIO、SMN、PCIE 配置空间等不同寄存器访问方式
- 掌握通过寄存器读取定位 GPU 硬件问题的方法
- 了解 amdgpu_regs 工具的代码实现和扩展机制

## 知识详解

### 概念原理

#### AMDGPU 寄存器访问架构

AMDGPU 驱动提供了一组 debugfs 接口，允许用户空间直接读取 GPU 寄存器的内容：

```ascii
AMDGPU 寄存器访问架构:

┌──────────────────────────────────────────────────────────────┐
│  用户空间                                                      │
│                                                               │
│  ┌─────────────┐  ┌──────────┐  ┌─────────────────┐          │
│  │ cat/debugfs │  │ umr      │  │ rocm-smi        │          │
│  │  shell工具   │  │用户态工具│  │ 监控工具         │          │
│  └──────┬──────┘  └────┬─────┘  └────────┬────────┘          │
│         │              │                 │                    │
├─────────┼──────────────┼─────────────────┼────────────────────┤
│  内核空间│              │                 │                    │
│         │              │                 │                    │
│  ┌──────▼──────────────▼─────────────────▼──────────────┐    │
│  │  AMDGPU DebugFS (drivers/gpu/drm/amd/amdgpu/amdgpu_debugfs.c) │
│  │                                                       │    │
│  │  amdgpu_regs: MMIO 寄存器读写                          │    │
│  │  amdgpu_smn_regs: SMN 寄存器读写                       │    │
│  │  amdgpu_pcie_cap: PCIE 配置空间访问                     │    │
│  │  amdgpu_gpu_offset: GPU 地址偏移                       │    │
│  └───────────────────────┬───────────────────────────────┘    │
│                          │                                     │
│  ┌──────────────────────▼──────────────────────────────────┐  │
│  │  AMDGPU 寄存器映射层                                      │  │
│  │                                                          │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │  │
│  │  │ MMIO (BAR0) │  │ SMN (System  │  │ PCIE Config    │  │  │
│  │  │ GPU 核心     │  │  Management  │  │ Space (BDF)    │  │  │
│  │  │ 寄存器       │  │  Network)    │  │                │  │  │
│  │  └─────────────┘  └──────────────┘  └────────────────┘  │  │
│  └──────────────────────┬──────────────────────────────────┘  │
│                         │                                      │
├─────────────────────────┼──────────────────────────────────────┤
│  硬件层                  │                                      │
│  ┌──────────────────────▼──────────────────────────────────┐  │
│  │  GPU ASIC 寄存器空间                                      │  │
│  │                                                          │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ GPU Memory Map:                                    │  │  │
│  │  │  0x00000 - 0x3FFFF:  GFX IP 寄存器                  │  │  │
│  │  │  0x40000 - 0x7FFFF:  MMHUB 寄存器                    │  │  │
│  │  │  0x80000 - 0xBFFFF:  DCN 显示寄存器                  │  │  │
│  │  │  0xC0000 - 0xFFFFF:  VCN 多媒体寄存器                │  │  │
│  │  │  ...                                                │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

#### 寄存器访问方式

```ascii
三种寄存器访问方式对比:

┌──────────────────────────────────────────────────────────────┐
│ 1. MMIO (Memory Mapped I/O)                                  │
│    ┌────────────────────────────────────────────────────┐     │
│    │ 访问方式: GPU BAR0 被映射到 CPU 地址空间            │     │
│    │ 地址范围: 0 ~ BAR0 大小 (典型 256KB - 2MB)         │     │
│    │ 访问速度: 快，CPU 直接内存访问                       │     │
│    │ 使用场景: GPU 核心寄存器 (GFX, MMHUB, DCN)          │     │
│    │ 读写函数: RREG32 / WREG32                          │     │
│    │ 代码路径: amdgpu_device_rreg / amdgpu_device_wreg  │     │
│    │                                                    │     │
│    │ 示例: amdgpu_regs 工具默认使用 MMIO 访问            │     │
│    │ # cat /sys/kernel/debug/dri/0/amdgpu_regs 0x1A000  │     │
│    └────────────────────────────────────────────────────┘     │
│                                                              │
│ 2. SMN (System Management Network)                           │
│    ┌────────────────────────────────────────────────────┐     │
│    │ 访问方式: 通过 SMN 总线间接访问 GPU 寄存器          │     │
│    │ 地址范围: 0 ~ 2^32 (完整的 GPU 地址空间)            │     │
│    │ 访问速度: 较慢，需要 SMN 总线事务                    │     │
│    │ 使用场景: SOC 级寄存器, Power Management 寄存器     │     │
│    │ 读写函数: amdgpu_device_smn_rreg / wreg            │     │
│    │                                                    │     │
│    │ SMN 协议:                                           │     │
│    │   1. 写地址到 SMN_ADDR 寄存器                       │     │
│    │   2. 读写 SMN_DATA 寄存器                           │     │
│    │   3. 等待 SMN 事务完成                               │     │
│    └────────────────────────────────────────────────────┘     │
│                                                              │
│ 3. PCIE Config Space                                         │
│    ┌────────────────────────────────────────────────────┐     │
│    │ 访问方式: PCIE 配置空间标准访问 (CFG cycles)        │     │
│    │ 地址范围: 0x00 - 0xFF (标准头) / 0x100+ (扩展)     │     │
│    │ 访问速度: 通过 PCIE root complex                     │     │
│    │ 使用场景: PCIE 能力, BAR, LTR, AER 寄存器          │     │
│    │ 读写函数: pci_read_config_word / pci_write          │     │
│    │                                                    │     │
│    │ 示例:                                              │     │
│    │ # cat /sys/kernel/debug/dri/0/amdgpu_pcie_cap 0x40 │     │
│    │ 读取 PCIE 链路状态和能力寄存器                       │     │
│    └────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────┘
```

#### GPU 寄存器空间布局

```ascii
AMD GPU (Navi3x / RDNA3) 寄存器内存映射:

GPU 虚拟地址空间:
┌──────────────────────────────────────────────────────────────┐
│ 0x0000_0000 ┌────────────────────────────────────────────┐    │
│             │  GFX IP (图形引擎)                          │    │
│ 0x0003_FFFF │  - 着色器 / 命令处理器 (CP)                  │    │
│             │  - 资源调度器 (GRBM)                        │    │
│             │  - 图形管线 (PA, SC, SPI, CB, DB)          │    │
│             └────────────────────────────────────────────┘    │
│ 0x0004_0000 ┌────────────────────────────────────────────┐    │
│             │  MMHUB (内存管理 Hub)                       │    │
│ 0x0007_FFFF │  - 页表 / TLB 管理                          │    │
│             │  - 内存仲裁                                  │    │
│             └────────────────────────────────────────────┘    │
│ 0x0008_0000 ┌────────────────────────────────────────────┐    │
│             │  DCN (Display Core Next)                    │    │
│ 0x000B_FFFF │  - OTG / DPP / MPC / OPP                   │    │
│             │  - HUBP / DCHUB                             │    │
│             │  - 显示时钟 / PLL                           │    │
│             └────────────────────────────────────────────┘    │
│ 0x000C_0000 ┌────────────────────────────────────────────┐    │
│             │  VCN (Video Core Next) 视频编解码           │    │
│ 0x000F_FFFF │  - 解码器 / 编码器                          │    │
│             │  - JPEG                                     │    │
│             └────────────────────────────────────────────┘    │
│ ...          ┌────────────────────────────────────────────┐    │
│             │  SDMA (System DMA)                          │    │
│             │  - 复制引擎                                  │    │
│             └────────────────────────────────────────────┘    │
│             ┌────────────────────────────────────────────┐    │
│             │  PSP (Platform Security Processor)          │    │
│             │  - 固件加载 / 安全功能                       │    │
│             └────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘

IP 寄存器偏移计算:
  IP_BASE = soc_get_ip_base(ip_type)  // 从 SOC 数据库获取基地址
  REG_ADDR = IP_BASE + REG_OFFSET     // 完整 MMIO 地址
```

### 实践操作

#### amdgpu_regs debugfs 接口

AMDGPU 通过 debugfs 提供了多个寄存器读写接口：

```c
// amdgpu_debugfs.c - debugfs 寄存器接口注册
int amdgpu_debugfs_init(struct amdgpu_device *adev)
{
    struct dentry *root = adev->ddev->primary->debugfs_root;
    
    // 1. MMIO 寄存器访问 (读/写)
    debugfs_create_file("amdgpu_regs", 0600, root, adev,
                        &amdgpu_debugfs_regs_fops);
    
    // 2. SMN 寄存器访问
    debugfs_create_file("amdgpu_smn_regs", 0600, root, adev,
                        &amdgpu_debugfs_smn_regs_fops);
    
    // 3. PCIE 配置空间访问
    debugfs_create_file("amdgpu_pcie_cap", 0600, root, adev,
                        &amdgpu_debugfs_pcie_cap_fops);
    
    // 4. GPU 寄存器偏移 (用于计算物理地址)
    debugfs_create_file("amdgpu_gpu_offset", 0400, root, adev,
                        &amdgpu_debugfs_gpu_offset_fops);
    
    // 5. 寄存器回读验证
    debugfs_create_file("amdgpu_regs_replay", 0600, root, adev,
                        &amdgpu_debugfs_regs_replay_fops);
    
    return 0;
}
```

#### MMIO 寄存器读取实现

```c
// amdgpu_debugfs.c - MMIO 寄存器读取实现
static void amdgpu_debugfs_regs_read(struct amdgpu_device *adev,
    loff_t pos, size_t count, char *buf)
{
    int i, r;
    uint32_t value;
    
    // 1. 验证地址范围
    // MMIO 地址必须在 [0, adev->rmmio_size) 范围内
    if (pos >= adev->rmmio_size)
        return;
    
    // 2. 限制读取长度 (防止过多寄存器读取)
    count = min(count, (size_t)(adev->rmmio_size - pos));
    count = min_t(size_t, count, 128);  // 最多读取 32 个寄存器 (32-bit each)
    
    // 3. 读取寄存器值
    i = 0;
    while (i < count) {
        // 4-byte 对齐
        if (pos % 4 == 0 && i + 4 <= count) {
            value = RREG32(pos);
            memcpy(buf + i, &value, 4);
            i += 4;
            pos += 4;
        } else {
            // 非对齐读取: 逐个字节
            value = RREG32(round_down(pos, 4));
            buf[i] = (value >> ((pos % 4) * 8)) & 0xFF;
            i++;
            pos++;
        }
    }
}

// debugfs 文件操作回调
static const struct file_operations amdgpu_debugfs_regs_fops = {
    .owner = THIS_MODULE,
    .read = amdgpu_debugfs_regs_read_wrapper,
    .write = amdgpu_debugfs_regs_write_wrapper,
    .llseek = default_llseek,
};
```

#### SMN 寄存器读取

SMN 寄存器通过间接访问方式读取，适用于完整的 GPU 地址空间：

```c
// amdgpu_debugfs.c - SMN 寄存器读取
static ssize_t amdgpu_debugfs_smn_regs_read(struct file *f,
    char __user *buf, size_t size, loff_t *pos)
{
    struct amdgpu_device *adev = f->f_inode->i_private;
    int r;
    uint32_t value;
    
    // 1. SMN 地址验证 (0 - 0xFFFFFFFF)
    if (*pos & 0x3)  // 4-byte 对齐要求
        return -EINVAL;
    
    // 2. SMN 间接读取
    // 写 SMN 地址到地址寄存器
    WREG32(SOC15_REG_OFFSET(NBIF, 0, regSMN_ADDR), *pos & 0xFFFFFFFF);
    
    // 等待地址写入完成
    udelay(1);
    
    // 读 SMN 数据寄存器
    value = RREG32(SOC15_REG_OFFSET(NBIF, 0, regSMN_DATA32));
    
    // 3. 复制到用户空间
    r = copy_to_user(buf, &value, min(size, sizeof(value)));
    
    // 4. 更新文件位置
    *pos += 4;
    
    return min(size, sizeof(value));
}

// SMN 批量读取示例
void read_smn_regs_batch(struct amdgpu_device *adev,
    uint32_t addr, uint32_t *values, int count)
{
    int i;
    
    for (i = 0; i < count; i++) {
        // 每个 SMN 寄存器需要独立的地址写 + 数据读
        WREG32_SMN(addr, addr + i * 4);
        // SMN 传输延迟 (典型 1-5 us)
        udelay(2);
        values[i] = RREG32_SMN(addr + i * 4);
    }
    
    // 观察: SMN 批量读取延迟累计
    // 100 个寄存器: ~500 us
    // 1000 个寄存器: ~5 ms
}
```

#### PCIE 配置空间访问

```c
// amdgpu_debugfs.c - PCIE 配置空间读取
static int amdgpu_debugfs_pcie_cap_read(struct amdgpu_device *adev,
    loff_t pos, u32 *value)
{
    struct pci_dev *pdev = adev->pdev;
    u32 v = 0;
    int ret;
    
    switch (pos & 0x3) {
    case 0:
        // 4-byte 对齐读取
        ret = pci_read_config_dword(pdev, pos, &v);
        if (ret != PCIBIOS_SUCCESSFUL)
            return -EIO;
        *value = v;
        break;
    case 2:
        // 2-byte 读取
        ret = pci_read_config_word(pdev, pos, (u16 *)&v);
        if (ret != PCIBIOS_SUCCESSFUL)
            return -EIO;
        *value = v & 0xFFFF;
        break;
    default:
        // 1-byte 读取
        ret = pci_read_config_byte(pdev, pos, (u8 *)&v);
        if (ret != PCIBIOS_SUCCESSFUL)
            return -EIO;
        *value = v & 0xFF;
        break;
    }
    
    return 0;
}

// PCIE 配置空间常见偏移
// 0x00: Vendor ID / Device ID
// 0x04: Command / Status
// 0x08: Revision ID / Class Code
// 0x10: BAR0 (MMIO base address)
// 0x14: BAR1 (FB aperture base address)
// 0x18: BAR2 (IO port base - 可选)
// 0x3C: Interrupt Line / Pin
// 0x40+: 能力寄存器
//   0x40: PCI Express Capability
//   0x50: MSI Capability
//   0x60: Advanced Error Reporting (AER)
//   0x80: Power Management Capability
//   0xA0: Vendor Specific Capability (AMD 专用)
//   0xE0: Resizable BAR Capability
```

#### 使用 amdgpu_regs 工具

通过命令行使用 amdgpu_regs debugfs 接口：

```bash
# 1. 列出可用的寄存器调试接口
ls -la /sys/kernel/debug/dri/0/amdgpu_regs*

# 输出:
# --w------- 1 root root 0 May 10 10:00 amdgpu_regs
# --w------- 1 root root 0 May 10 10:00 amdgpu_smn_regs
# -r-------- 1 root root 0 May 10 10:00 amdgpu_gpu_offset
# --w------- 1 root root 0 May 10 10:00 amdgpu_pcie_cap

# 2. 读取单个 MMIO 寄存器
# 格式: echo <offset> <count> > amdgpu_regs; cat amdgpu_regs
# 读取 DCN OTG0 控制寄存器 (偏移 0x1A000)
echo 0x1A000 > /sys/kernel/debug/dri/0/amdgpu_regs
cat /sys/kernel/debug/dri/0/amdgpu_regs

# 输出示例:
# 0x1A000: 0x00000001  (OTG_MASTER_EN = 1, 表示 OTG 已启用)

# 3. 批量读取多个寄存器
echo 0x1A000 8 > /sys/kernel/debug/dri/0/amdgpu_regs
cat /sys/kernel/debug/dri/0/amdgpu_regs
# 输出:
# 0x1A000: 0x00000001  OTG_MASTER_EN
# 0x1A004: 0x00000000  OTG_CONTROL
# 0x1A008: 0x00000004  OTG_H_TOTAL = 3840+... = 4400
# 0x1A00C: 0x00000000  OTG_V_TOTAL
# 0x1A010: 0x00000000  OTG_H_SYNC_A
# 0x1A014: 0x00000000  OTG_H_SYNC_B
# 0x1A018: 0x00000000  OTG_V_SYNC_A
# 0x1A01C: 0x00000000  OTG_V_SYNC_B

# 4. 读取 SMN 寄存器 (访问完整 GPU 地址空间)
# 读取 SMU 温度传感器寄存器 (Navi31)
echo 0x13A00000 > /sys/kernel/debug/dri/0/amdgpu_smn_regs
cat /sys/kernel/debug/dri/0/amdgpu_smn_regs

# 输出:
# 0x13A00000: 0x00000041  (温度 = 0x41 = 65°C)

# 5. 读取 PCIE 配置空间
echo 0x40 > /sys/kernel/debug/dri/0/amdgpu_pcie_cap
cat /sys/kernel/debug/dri/0/amdgpu_pcie_cap
# 输出:
# 0x40: 0x00100010  (PCIe Capability: version 2, device/port type)
#   bit[15:8]: 0x10 = PCI Express Capability version 2
#   bit[3:0]: 0x00 = PCI Express endpoint

# 6. 解码 PCIE 链路状态
echo 0x42 > /sys/kernel/debug/dri/0/amdgpu_pcie_cap
cat /sys/kernel/debug/dri/0/amdgpu_pcie_cap
# 输出: 0x42: 0x0042
#   bit[7:4]: 0x4 = Link Width x16
#   bit[3:0]: 0x2 = Link Speed 5.0 GT/s (Gen2)
#   (实际应为 Gen4 16x: 需要检查连接问题)

# 7. 检查 GPU 偏移
cat /sys/kernel/debug/dri/0/amdgpu_gpu_offset
# 输出:
# GPU offset: 0x000000007fed0000 (GPU 地址偏移量)
```

#### 寄存器读取脚本工具

创建批处理脚本简化寄存器读取：

```bash
#!/bin/bash
# amdgpu_regs_tool.sh - AMDGPU 寄存器读取辅助脚本
# 用法: ./amdgpu_regs_tool.sh <command> [args]

AMDGPU_DEBUGFS="/sys/kernel/debug/dri/0"

read_reg() {
    local offset=$1
    local count=${2:-1}
    
    echo "读取 MMIO 寄存器: offset=0x${offset}, count=${count}"
    
    # 写入偏移和计数
    echo "$((16#${offset})) ${count}" > ${AMDGPU_DEBUGFS}/amdgpu_regs
    
    # 读取结果
    cat ${AMDGPU_DEBUGFS}/amdgpu_regs
    
    echo ""
}

read_smn() {
    local offset=$1
    
    echo "读取 SMN 寄存器: 0x${offset}"
    
    # SMN 只读取 1 个 4-byte 值
    echo "$((16#${offset}))" > ${AMDGPU_DEBUGFS}/amdgpu_smn_regs
    cat ${AMDGPU_DEBUGFS}/amdgpu_smn_regs
    
    echo ""
}

# 读取 DCN 寄存器 (显示管线状态)
read_dcn_status() {
    echo "===== DCN 状态 ====="
    
    # OTG 控制
    read_reg 1A000 4
    echo "--- OTG 控制寄存器 ---"
    
    # DPP 控制
    read_reg 40000 4
    echo "--- DPP 控制寄存器 ---"
    
    # MPC 控制
    read_reg 48000 4
    echo "--- MPC 控制寄存器 ---"
}

# 读取电源相关寄存器
read_power_status() {
    echo "===== 电源状态 ====="
    
    # SMU 温度
    read_smn 13A00000
    read_smn 13A00004
    
    # GFX 时钟频率
    read_smn 14000000
    read_smn 14000004
    
    echo "--- 电源状态 ---"
    read_smn 13B00000 8
}

# 读取 PCIE 状态
read_pcie_status() {
    echo "===== PCIE 状态 ====="
    echo "--- PCIE 能力 ---"
    read_reg 40 4
    echo "--- 链路状态 ---"
    read_reg 42 4
    echo "--- 设备状态 ---"
    read_reg 44 4
    echo "--- 链路控制 ---"
    read_reg 48 4
}

# 主命令分发
case "$1" in
    reg)
        shift
        read_reg "$@"
        ;;
    smn)
        shift
        read_smn "$@"
        ;;
    dcn)
        read_dcn_status
        ;;
    power)
        read_power_status
        ;;
    pcie)
        read_pcie_status
        ;;
    all)
        read_dcn_status
        read_power_status
        read_pcie_status
        ;;
    *)
        echo "用法: $0 {reg|smn|dcn|power|pcie|all} [offset] [count]"
        echo "示例:"
        echo "  $0 reg 1A000     # 读取 MMIO 偏移 0x1A000"
        echo "  $0 reg 1A000 8   # 批量读取 8 个寄存器"
        echo "  $0 smn 13A00000  # 读取 SMN 寄存器"
        echo "  $0 dcn           # 读取 DCN 状态"
        echo "  $0 power         # 读取电源状态"
        echo "  $0 pcie          # 读取 PCIE 状态"
        echo "  $0 all           # 读取所有状态"
        exit 1
        ;;
esac
```

#### 寄存器映射查找工具

```python
#!/usr/bin/env python3
# gpu_regs_map.py - AMD GPU 寄存器映射查找工具

import sys
import re

# AMD GPU 寄存器映射数据库 (Navi 3x 部分寄存器)
REG_MAP = {
    # GFX IP
    "GRBM_STATUS":             {"offset": 0x8000, "type": "MMIO", "desc": "GFX 资源调度器状态"},
    "GRBM_SOFT_RESET":         {"offset": 0x8018, "type": "MMIO", "desc": "GFX 软复位"},
    "CP_STATUS":              {"offset": 0x8200, "type": "MMIO", "desc": "命令处理器状态"},
    "CP_CPC_STATUS":           {"offset": 0x8300, "type": "MMIO", "desc": "CPC (计算管道) 状态"},
    
    # DCN 显示
    "OTG_MASTER_EN":           {"offset": 0x1A000, "type": "MMIO", "desc": "OTG 主使能"},
    "OTG_CONTROL":             {"offset": 0x1A004, "type": "MMIO", "desc": "OTG 控制"},
    "OTG_H_TOTAL":             {"offset": 0x1A008, "type": "MMIO", "desc": "水平总像素"},
    "OTG_V_TOTAL":             {"offset": 0x1A00C, "type": "MMIO", "desc": "垂直总行数"},
    "OTG_H_SYNC_A":            {"offset": 0x1A010, "type": "MMIO", "desc": "水平同步开始"},
    "OTG_H_SYNC_B":            {"offset": 0x1A014, "type": "MMIO", "desc": "水平同步结束"},
    "OTG_V_SYNC_A":            {"offset": 0x1A018, "type": "MMIO", "desc": "垂直同步开始"},
    "OTG_V_SYNC_B":            {"offset": 0x1A01C, "type": "MMIO", "desc": "垂直同步结束"},
    "OTG_V_UPDATE_POSITION":   {"offset": 0x1A284, "type": "MMIO", "desc": "VUpdate 位置"},
    "OTG_V_BLANK_START_END":   {"offset": 0x1A024, "type": "MMIO", "desc": "VBlank 开始/结束"},
    "OPTC_DRR_CONTROL":        {"offset": 0x1A0C8, "type": "MMIO", "desc": "DRR (VRR) 控制"},
    "OPTC_V_TOTAL_MIN":        {"offset": 0x1A1C0, "type": "MMIO", "desc": "VRR V_Total 最小值"},
    "OPTC_V_TOTAL_MAX":        {"offset": 0x1A1C4, "type": "MMIO", "desc": "VRR V_Total 最大值"},
    
    # HUBP / DCHUB
    "HUBP_VTG_FP2":            {"offset": 0x4000, "type": "MMIO", "desc": "HUBP 垂直时序"},
    "DCHUBP_CNTL":             {"offset": 0x4004, "type": "MMIO", "desc": "HUBP 控制"},
    
    # SMU 电源管理 (SMN 空间)
    "SMU_SMU_STATUS":          {"offset": 0x13A00000, "type": "SMN", "desc": "SMU 状态"},
    "SMU_GFX_CLK_STATUS":      {"offset": 0x13A00004, "type": "SMN", "desc": "GFX 时钟状态"},
    "SMU_SOFT_RESET":          {"offset": 0x13A00008, "type": "SMN", "desc": "SMU 软复位"},
    "SMU_TEMPERATURE":         {"offset": 0x13A0000C, "type": "SMN", "desc": "温度传感器"},
    "SMU_POWER_AVERAGE":       {"offset": 0x13A00010, "type": "SMN", "desc": "平均功耗"},
    
    # PCIE 配置空间
    "PCI_VENDOR_DEVICE":       {"offset": 0x00, "type": "PCIE", "desc": "Vendor/Device ID"},
    "PCI_COMMAND_STATUS":      {"offset": 0x04, "type": "PCIE", "desc": "Command/Status"},
    "PCI_BAR0":               {"offset": 0x10, "type": "PCIE", "desc": "BAR0 (MMIO)"},
    "PCI_BAR1":               {"offset": 0x14, "type": "PCIE", "desc": "BAR1 (FB)"},
    "PCI_EXPRESS_CAP":        {"offset": 0x40, "type": "PCIE", "desc": "PCIe 能力"},
    "PCI_LINK_STATUS":        {"offset": 0x42, "type": "PCIE", "desc": "链路状态"},
    "PCI_DEVICE_STATUS":      {"offset": 0x44, "type": "PCIE", "desc": "设备状态"},
    "PCI_AER_CAP":            {"offset": 0x60, "type": "PCIE", "desc": "AER 能力"},
    "PCI_MSI_CAP":            {"offset": 0x50, "type": "PCIE", "desc": "MSI 能力"},
}


def lookup_reg(name_or_offset):
    """查找寄存器映射"""
    # 按名称查找
    if name_or_offset.upper() in REG_MAP:
        info = REG_MAP[name_or_offset.upper()]
        print(f"寄存器: {name_or_offset.upper()}")
        print(f"  偏移: 0x{info['offset']:X}")
        print(f"  类型: {info['type']}")
        print(f"  描述: {info['desc']}")
        return
    
    # 按偏移查找
    try:
        offset = int(name_or_offset, 16) if name_or_offset.startswith("0x") else int(name_or_offset)
        found = False
        for name, info in REG_MAP.items():
            if info["offset"] == offset:
                print(f"偏移 0x{offset:X} -> {name}")
                print(f"  类型: {info['type']}")
                print(f"  描述: {info['desc']}")
                found = True
                break
        if not found:
            print(f"未找到偏移 0x{offset:X} 对应的寄存器")
    except ValueError:
        print(f"无效的输入: {name_or_offset}")


def list_by_type(reg_type):
    """按类型列出寄存器"""
    print(f"\n{reg_type} 寄存器列表:")
    print("-" * 60)
    for name, info in sorted(REG_MAP.items(), key=lambda x: x[1]["offset"]):
        if info["type"] == reg_type:
            print(f"  0x{info['offset']:08X}  {name:30s}  {info['desc']}")
    print()


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("AMD GPU 寄存器映射查找工具")
        print()
        print("用法:")
        print(f"  {sys.argv[0]} <寄存器名称或偏移>")
        print(f"  {sys.argv[0]} --list-mem    # 列出 MMIO 寄存器")
        print(f"  {sys.argv[0]} --list-smn    # 列出 SMN 寄存器")
        print(f"  {sys.argv[0]} --list-pcie   # 列出 PCIE 寄存器")
        print()
        print("示例:")
        print(f"  {sys.argv[0]} OTG_MASTER_EN")
        print(f"  {sys.argv[0]} 0x1A000")
        print()
        sys.exit(1)
    
    if sys.argv[1] == "--list-mem":
        list_by_type("MMIO")
    elif sys.argv[1] == "--list-smn":
        list_by_type("SMN")
    elif sys.argv[1] == "--list-pcie":
        list_by_type("PCIE")
    else:
        lookup_reg(sys.argv[1])
```

#### 高级调试：寄存器回读验证

```c
// amdgpu_debugfs.c - 寄存器回读验证 (用于调试寄存器写入问题)
static int amdgpu_debugfs_regs_replay_read(struct amdgpu_device *adev,
    char *buf, size_t size)
{
    // 回放最近写入的寄存器序列，验证是否能正确回读
    // 用于检测:
    // 1. 寄存器写入是否生效
    // 2. 寄存器是否被其他 IP 意外修改
    // 3. 电源门控导致的寄存器丢失
    
    for (int i = 0; i < adev->regs_replay_count; i++) {
        uint32_t expected = adev->regs_replay[i].value;
        uint32_t actual = RREG32(adev->regs_replay[i].offset);
        
        if (expected != actual) {
            DRM_WARN("REG REPLAY: offset 0x%05X: expected 0x%08X, actual 0x%08X (diff=%d)\n",
                adev->regs_replay[i].offset,
                expected, actual,
                adev->regs_replay[i].offset);
        }
        
        seq_printf(m, "0x%05X: wrote 0x%08X, read 0x%08X %s\n",
            adev->regs_replay[i].offset,
            expected, actual,
            expected == actual ? "[OK]" : "[MISMATCH]");
    }
    
    return 0;
}

// 寄存器写入追踪 (通过 RREG32/WREG32 包装)
void amdgpu_debugfs_trace_reg_write(struct amdgpu_device *adev,
    uint32_t offset, uint32_t value)
{
    if (adev->regs_replay_count < MAX_REGS_REPLAY) {
        int idx = adev->regs_replay_count++;
        adev->regs_replay[idx].offset = offset;
        adev->regs_replay[idx].value = value;
    }
}
```

### 案例分析

#### 案例 1：通过寄存器定位显示无输出问题

**问题描述**：系统启动后显示器无信号，GPU 驱动已加载但无画面输出。

**排查过程**：

```bash
# 步骤 1: 检查驱动加载状态
dmesg | grep amdgpu | tail -20

# 步骤 2: 检查 CRTC/Connector 状态
cat /sys/kernel/debug/dri/0/amdgpu_connectors

# 步骤 3: 读取 DCN OTG 状态
echo 0x1A000 4 > /sys/kernel/debug/dri/0/amdgpu_regs
cat /sys/kernel/debug/dri/0/amdgpu_regs

# 输出:
# 0x1A000: 0x00000000  ← OTG_MASTER_EN = 0 (OTG 未使能!)
# 正常应为 0x00000001

# 步骤 4: 检查 CRTC 寄存器
echo 0x1A004 > /sys/kernel/debug/dri/0/amdgpu_regs
cat /sys/kernel/debug/dri/0/amdgpu_regs
# 0x1A004: 0x00000000  ← OTG 完全未被编程

# 结论: Atomic mode set 未成功提交导致 OTG 未使能
# 可能是 framebuffer 未正确创建或原子检查失败
```

#### 案例 2：PCIE 链路降级检测

**问题描述**：GPU 性能低于预期，怀疑 PCIE 链路运行在低速率。

**排查过程**：

```bash
# 步骤 1: 检查 PCIE 链路状态
echo 0x42 > /sys/kernel/debug/dri/0/amdgpu_pcie_cap
cat /sys/kernel/debug/dri/0/amdgpu_pcie_cap
# 0x42: 0x2001
# bit[7:4] = 0x1 = Link Width x1 (应为 x16)
# bit[3:0] = 0x2 = Link Speed 5.0 GT/s (Gen2)

# 步骤 2: 检查 AER 错误
echo 0x60 8 > /sys/kernel/debug/dri/0/amdgpu_pcie_cap
cat /sys/kernel/debug/dri/0/amdgpu_pcie_cap
# 0x60: 0x00014001
# 0x64: 0x00000000
# 0x68: 0x00000000
# 0x6C: 0x00000000
# 0x70: 0x00000000
# 0x74: 0x00000000
# 0x78: 0x00000000
# 0x7C: 0x00000000

# 步骤 3: 检查 PCIE 错误计数器
lspci -vv -s 03:00.0 | grep -i "correctable\|non-fatal"
# 发现大量 Correctable Error Count

# 结论: PCIE 链路信号完整性差，PCIE Retrain 后降级到 Gen2 x1
# 解决方案: 重插 GPU 或检查 PCIE 延长线/Riser 卡
```

### 相关链接

- AMDGPU Debugfs 文档: https://docs.kernel.org/gpu/amdgpu/amdgpu-debugfs.html
- Linux PCI 配置空间访问: https://docs.kernel.org/PCI/pci.html
- AMD GPU 寄存器参考手册 (需 NDA): https://developer.amd.com/
- UMR 工具: https://gitlab.freedesktop.org/tomstdenis/umr
- GPU 寄存器解码器: https://github.com/GPUOpen-Tools/GPU-PerfStudio

### 今日小结

1. **三种寄存器访问方式**：MMIO (BAR0 直接映射，快速)、SMN (间接总线访问，覆盖完整空间)、PCIE Config Space (标准配置空间)。

2. **amdgpu_regs debugfs 接口**：提供 amdgpu_regs (MMIO)、amdgpu_smn_regs (SMN)、amdgpu_pcie_cap (PCIE Config)、amdgpu_gpu_offset 四个文件节点。

3. **寄存器偏移查找**：不同 GPU IP (GFX/MMHUB/DCN/VCN/SMU) 有不同的基地址偏移，需要 SOC 数据库查询。

4. **调试应用**：寄存器读取工具可用于排查显示无输出、PCIE 链路降级、电源状态异常等问题。

5. **回放验证**：通过寄存器回读验证机制检查写入的寄存器值是否保持，检测电源门控或意外修改问题。

### 扩展思考

1. **寄存器安全**：直接寄存器写入可能导致 GPU 崩溃或硬件损坏，debugfs 接口为什么需要 root 权限保护？内核如何防止危险寄存器写入？

2. **寄存器虚拟化**：在 SR-IOV 环境中，VF (Virtual Function) 能否访问所有寄存器？哪些寄存器被 VF 独占，哪些被 PF 管理？

3. **寄存器与调试效率**：相比使用 IGT/uMR 等高级工具，直接读取寄存器的方式在什么场景下更高效？什么场景下不够用？

4. **自监测寄存器 (SMC)**：现代 GPU (RDNA3/CDNA3) 有哪些硬件自监测能力？SMC (System Management Controller) 如何帮助调试？