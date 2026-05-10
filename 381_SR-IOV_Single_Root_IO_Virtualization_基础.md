# Day 381：SR-IOV（Single Root I/O Virtualization）基础

## 学习目标

- 理解 SR-IOV 的核心概念、架构原理和 PCIe 规范中的定义
- 掌握 PF（Physical Function）与 VF（Virtual Function）的区别与作用
- 理解 SR-IOV 在 GPU 虚拟化中的应用场景和限制
- 学会在 Linux 系统中启用和管理 SR-IOV 功能
- 掌握 SR-IOV 的性能隔离与资源分配模型

## 知识详解

### 概念原理

#### SR-IOV 的诞生背景

传统 I/O 虚拟化面临一个核心矛盾：虚拟机需要直接访问硬件设备以获得高性能，但直接访问会带来安全性和隔离性问题。在 SR-IOV 出现之前，主要有三种 I/O 虚拟化方案：

```
方案        | 优点              | 缺点
全虚拟化    | 无需修改 Guest OS  | 性能差，需 trap 模拟
半虚拟化    | 性能较好           | 需修改 Guest OS
设备直通    | 性能接近原生       | 每个设备只能分配给一个 VM
```

SR-IOV（Single Root I/O Virtualization）由 PCI-SIG 在 2007 年提出，作为 PCIe 规范的可选扩展能力。其核心思想是：让**单个物理设备在硬件层面呈现为多个独立的虚拟设备**，每个虚拟设备拥有独立的 DMA 地址空间、中断资源和配置空间，虚拟机可以直接访问而无需 Hypervisor 介入数据传输路径。

```
┌─────────────────────────────────────────────────────────┐
│                     Hypervisor / Host OS                  │
│  ┌─────────────────────────────────────────────────────┐ │
│  │                 PCIe 配置空间管理                    │ │
│  └─────────────────────────────────────────────────────┘ │
│                         │                                 │
│         ┌───────────────┴───────────────┐                 │
│         ▼                               ▼                 │
│  ┌──────────┐                  ┌──────────────┐          │
│  │   PF 驱动  │                  │  VF 驱动管理  │          │
│  │  (完整功能) │                  │  (创建/销毁)  │          │
│  └─────┬────┘                  └──────┬───────┘          │
│        │                              │                    │
└────────┼──────────────────────────────┼────────────────────┘
         │                              │
         ▼                              ▼
┌──────────────────────────────────────────────┐
│              GPU 硬件（SR-IOV）                │
│  ┌────────────────────────────────────────┐  │
│  │           PF（Physical Function）       │  │
│  │  - 完整配置空间 (BDF + VF BAR)         │  │
│  │  - 完整 GPU 功能 (引擎/内存/调度)      │  │
│  │  - VF 管理 (创建/销毁/资源分配)        │  │
│  └────────────┬───────────────────────────┘  │
│               │                               │
│    ┌──────────┼──────────┐                    │
│    ▼          ▼          ▼                    │
│ ┌──────┐ ┌──────┐ ┌──────┐                   │
│ │VF 0  │ │VF 1  │ │VF n  │                   │
│ │独立BAR│ │独立BAR│ │独立BAR│                   │
│ │独立MSI│ │独立MSI│ │独立MSI│                   │
│ │独立DMA│ │独立DMA│ │独立DMA│                   │
│ │ 队列  │ │ 队列  │ │ 队列  │                   │
│ └──────┘ └──────┘ └──────┘                   │
│    │         │         │                       │
│    └─────────┼─────────┘                       │
│              ▼                                 │
│    ┌──────────────────┐                        │
│    │ 物理硬件资源仲裁器 │                        │
│    │ (HW Scheduler)   │                        │
│    └──────────────────┘                        │
└──────────────────────────────────────────────┘
```

#### 关键术语

**Physical Function（PF）：**
PF 是物理设备上完整的 PCIe 功能，拥有完整的配置空间、BAR 空间和全部硬件能力。PF 驱动负责管理整个设备，包括 VF 的创建、销毁和资源分配。在 AMDGPU 中，PF 驱动拥有完整的 GPU 控制权，包括时钟管理、电源管理和内存管理。

**Virtual Function（VF）：**
VF 是由 PF 在硬件层面创建的轻量级 PCIe 功能。每个 VF 拥有：
- 独立的 PCIe 配置空间（BDF、 Vendor ID、 Device ID、 Revision ID）
- 独立的 BAR 空间（通常映射部分 GPU 资源）
- 独立的 MSI/MSI-X 中断向量
- 独立的 DMA 地址空间（IOMMU 保护）
- 部分硬件队列（如 GFX 管道、SDMA 引擎的独立通道）

VF 驱动是轻量级驱动，仅有完整驱动的子集功能，主要提供：
- 命令提交接口（ring buffer）
- 内存分配接口（通过 PF 代理）
- 中断处理
- 状态查询

**ATS（Address Translation Services）：**
PCIe ATS 允许设备缓存 IOMMU 地址转换结果，减少每次 DMA 的转换开销。在 SR-IOV 环境中，ATS 对 VF 的性能至关重要。

**ACS（Access Control Services）：**
PCIe ACS 提供点对点访问控制，防止 VF 之间或 VF 与 PF 之间的非授权直接内存访问，是 SR-IOV 安全隔离的基础。

#### SR-IOV 规范的 PCIe 扩展

SR-IOV 在 PCIe 配置空间中添加了新的能力结构（Capability Structure）：

```
PCIe 配置空间中的 SR-IOV 能力结构偏移：

├── Capability ID (0x10)          // SR-IOV 能力 ID
├── Next Capability Pointer
├── SR-IOV Control Register       // 启用/禁用 VF
│   ├── Bit 0: VF Enable
│   ├── Bit 1: VF Migration Enable
│   ├── Bit 3: ARI Capable Hierarchy
│   └── Bit 8: VF MSE (Memory Space Enable)
├── SR-IOV Status Register        // VF 状态
├── TotalVFs                      // 设备支持的最大 VF 数量
├── InitialVFs                    // 初始启用的 VF 数量
├── TotalVFs                      // 物理支持的最大 VF 数
├── Function Dependency Link      // PF/VF 依赖关系
├── First VF Offset               // 第一个 VF 的偏移
├── VF Stride                     // VF 之间的 BDF 步长
├── VF Device ID                  // VF 的设备 ID
├── VF BARs (0-5)                 // VF 的 BAR 空间定义
└── VF Migration State Array      // VF 迁移状态（可选）
```

关键字段说明：

**TotalVFs** 是硬件实际支持的最大 VF 数量，受物理资源限制（如显存大小、引擎数量）。**InitialVFs** 是启动时默认启用的 VF 数量。**VF Stride** 定义了 VF 之间的 BDF 间隔，例如 stride=2 表示 PF 在 BDF 00.0，VF0 在 00.1，VF1 在 00.3（跳过 00.2）。

VF 的 BAR 空间与 PF 的 BAR 空间不同。VF BAR 通常映射了：
- VF 专用的 MMIO 空间（doorbell、command queue）
- 部分显存空间（通过 GPU 内部地址重映射）
- VF 状态寄存器

#### SR-IOV 的硬件资源分区模型

GPU 的 SR-IOV 硬件分区与网卡不同，因为 GPU 的资源类型更复杂：

```
GPU SR-IOV 硬件分区层次：

┌────────────────────────────────────────────────────┐
│                   GPU 物理资源                       │
├────────────────────────────────────────────────────┤
│  ┌──── PCIe 接口 (Root Complex) ─────────────────┐ │
│  │  PF: BDF 03:00.0    VF0: 03:00.1  VF1: 03:00.2│ │
│  └────────────────────────────────────────────────┘ │
│  ┌──── 计算单元 (SE/SH/CU) ──────────────────────┐ │
│  │  VF0: CU 0-7     VF1: CU 8-15   PF独占: CU 16 │ │
│  └────────────────────────────────────────────────┘ │
│  ┌──── 显存控制器 (UMC/HBM) ─────────────────────┐ │
│  │  VF0: 0-8GB     VF1: 8-16GB   PF: 16GB+     │ │
│  └────────────────────────────────────────────────┘ │
│  ┌──── DMA 引擎 (SDMA) ──────────────────────────┐ │
│  │  VF0: SDMA0    VF1: SDMA1    PF: SDMA2       │ │
│  └────────────────────────────────────────────────┘ │
│  ┌──── 多媒体引擎 (VCN) ─────────────────────────┐ │
│  │  VF0: VCN0 queue  VF1: VCN1 queue            │ │
│  └────────────────────────────────────────────────┘ │
│  ┌──── 硬件调度器 ───────────────────────────────┐ │
│  │  VF0 ring: 0-31    VF1 ring: 32-63           │ │
│  └────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────┘
```

资源分区有两种模式：

1. **静态分区（Static Partitioning）：** 在 VF 创建时固定分配资源，不可动态调整。优点：简单、隔离性好。缺点：资源利用率低。

2. **动态分区（Dynamic Partitioning）：** 根据 VF 负载实时调整资源分配。优点：资源利用率高。缺点：实现复杂，需要硬件支持。

AMDGPU SR-IOV 采用**混合模式**：显存和 PCIe 资源使用静态分区，计算单元使用动态调度。

#### SR-IOV 与 GPU 虚拟化

SR-IOV 在 GPU 虚拟化中有其独特的定位：

```
虚拟化方案对比矩阵：

                   全虚拟化     API 转发    设备直通      SR-IOV
                   (软件模拟)   (Mesa)      (Passthrough)  (硬件虚拟化)
─────────────────────────────────────────────────────────────────────
性能              极低(1-5%)   高(80-90%)  极高(95%+)   极高(90-95%)
隔离性            极高          中          高            高
VM 密度           高           高          低(1 VM/GPU)  中(2-8 VM/GPU)
GPU 共享          是           是          否            是
驱动修改          无           无          无            轻量 VF 驱动
硬件支持          无           无          无            需要
API 兼容性        完全          部分 API    完全          完全
迁移支持          是           是          否            有限
─────────────────────────────────────────────────────────────────────
```

SR-IOV 的核心优势是**兼顾性能和多租户**。对于 GPU 驱动测试而言，SR-IOV 允许在单张 GPU 上创建多个隔离的测试环境，每个环境有独立的显存空间和引擎通道，互不干扰。

#### AMDGPU SR-IOV 架构总览

AMDGPU SR-IOV 实现是 AMD 的硬件辅助 GPU 虚拟化方案，主要面向数据中心和云计算场景：

```
AMDGPU SR-IOV 软件架构：

┌─────────────────────────────────────────────────────────────────────┐
│                            Host Linux (Dom0)                         │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    amdgpu PF 驱动模块                          │  │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────────────────┐    │  │
│  │  │ 设备管理    │ │ VF 管理    │ │ 资源管理               │    │  │
│  │  │ - PCI 枚举  │ │ - VF 创建  │ │ - 显存分区 (VRAM)     │    │  │
│  │  │ - BAR 映射  │ │ - VF 销毁  │ │ - 引擎分配 (GFX/SDMA) │    │  │
│  │  │ - IRQ 设置  │ │ - VF 状态  │ │ - 带宽限制             │    │  │
│  │  └────────────┘ └────────────┘ └────────────────────────┘    │  │
│  │  ┌────────────────────────────────────────────────────────┐    │  │
│  │  │                 SR-IOV 后端服务                         │    │  │
│  │  │  - 消息传递 (Mailbox)                                  │    │  │
│  │  │  - 页表管理 (GART 共享)                                │    │  │
│  │  │  - 错误上报 (RAS 转发)                                 │    │  │
│  │  └────────────────────────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                               │                                       │
└───────────────────────────────┼───────────────────────────────────────┘
                                │
         ┌──────────────────────┼──────────────────────┐
         ▼                      ▼                      ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│   Guest 0 (DomU)  │ │   Guest 1 (DomU)  │ │   Guest n (DomU)  │
│ ┌────────────────┐│ │ ┌────────────────┐│ │ ┌────────────────┐│
│ │ amdgpu VF 驱动 ││ │ │ amdgpu VF 驱动 ││ │ │ amdgpu VF 驱动 ││
│ │ - 命令提交      ││ │ │ - 命令提交      ││ │ │ - 命令提交      ││
│ │ - 内存分配      ││ │ │ - 内存分配      ││ │ │ - 内存分配      ││
│ │ - 中断处理      ││ │ │ - 中断处理      ││ │ │ - 中断处理      ││
│ └────────────────┘│ │ └────────────────┘│ │ └────────────────┘│
└──────────────────┘ └──────────────────┘ └──────────────────┘
        │                      │                      │
        └──────────────────────┼──────────────────────┘
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                   GPU 硬件 (SR-IOV Enabled)                    │
│  ┌────────────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌────────────┐  │
│  │    PF      │ │ VF 0 │ │ VF 1 │ │ VF n │ │ 硬件调度器  │  │
│  │ 完整功能    │ │轻量级 │ │轻量级 │ │轻量级 │ │ 时间片轮转  │  │
│  └────────────┘ └──────┘ └──────┘ └──────┘ └────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### 实践操作

#### 检查系统 SR-IOV 支持

在启用 SR-IOV 之前，需要确认硬件和软件支持：

```bash
# 1. 检查 GPU 是否支持 SR-IOV
lspci -vvv -s $(lspci | grep AMD | grep "VGA\|Display" | head -1 | cut -d' ' -f1) | grep -i "sriov"

# 输出示例：
# Capabilities: [300 v1] Single Root I/O Virtualization (SR-IOV)
#   IOVCap: Migration-, ARIHint-, ARICapable-
#   IOVCtl: Enable- Migration- Interrupt- MSE- ARIHierarchy-
#   IOVSta: Migration- PageRequest+
#   InitialVFs: 0, TotalVFs: 15, Function Dependency Link: 00
#   First VF Offset: 0x0001, VF Stride: 0x0002

# 2. 检查内核是否支持 SR-IOV
cat /boot/config-$(uname -r) | grep -i "SRIOV\|PCI_IOV"

# 预期输出：
# CONFIG_PCI_IOV=y
# CONFIG_PCI_PRI=y
# CONFIG_PCI_PASID=y

# 3. 检查 IOMMU 是否启用
dmesg | grep -i "iommu\|amd_iommu"

# 预期输出示例：
# [    0.123456] AMD-Vi: IOMMU performance counters supported
# [    0.123789] AMD-Vi: Found IOMMU at ...
# [    0.456789] AMD-Vi: Initialized for Passthrough Mode
```

SR-IOV 的硬件依赖关系：

```
必要条件清单：

硬件条件:
├── GPU 支持 SR-IOV (TotalVFs > 0)
├── CPU/主板支持 IOMMU (AMD-Vi / Intel VT-d)
└── 足够的内存和显存资源

软件条件:
├── Linux 内核支持 PCI_IOV (CONFIG_PCI_IOV=y)
├── Linux 内核支持 IOMMU (CONFIG_IOMMU_SUPPORT=y)
├── AMDGPU 驱动编译了 SR-IOV 支持
├── 虚拟机监视器支持 SR-IOV (KVM/Xen)
└── VF 驱动 (VF driver) 可用
```

#### 启用 SR-IOV

```bash
# 1. 在内核命令行启用 IOMMU
# 编辑 /etc/default/grub，在 GRUB_CMDLINE_LINUX 中添加：
# 对于 AMD CPU：
#   amd_iommu=on iommu=pt
# 对于 Intel CPU：
#   intel_iommu=on iommu=pt

# 示例：
# GRUB_CMDLINE_LINUX="amd_iommu=on iommu=pt"

# 更新 grub 配置：
sudo grub-mkconfig -o /boot/grub/grub.cfg   # Ubuntu/Debian
# sudo grub2-mkconfig -o /boot/grub2/grub.cfg  # RHEL/CentOS

# 2. 重启系统
sudo reboot

# 3. 重启后验证 IOMMU
dmesg | grep "AMD-Vi\|DMAR\|IOMMU"

# 4. 加载 amdgpu 驱动时启用 SR-IOV
# 方式一：模块参数
echo "options amdgpu sriov=1" | sudo tee /etc/modprobe.d/amdgpu-sriov.conf
sudo update-initramfs -u

# 方式二：sysfs 动态启用
# 写入 VF 数量到 sriov_numvfs 文件
echo 4 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs

# 5. 验证 VF 创建
lspci | grep "AMD\|ATI" | grep "Virtual Function"

# 输出示例：
# 03:00.0 VGA compatible controller: AMD ... (Physical Function)
# 03:00.1 AMD ... Virtual Function
# 03:00.3 AMD ... Virtual Function
# 03:00.5 AMD ... Virtual Function
# 03:00.7 AMD ... Virtual Function
# (注意 VF Stride=2，所以 BDF 为 00.1, 00.3, 00.5, 00.7)
```

#### 管理 VF 生命周期

```bash
# 1. 查询 VF 信息
# 查看当前 VF 数量
cat /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs

# 查看最大 VF 数量
cat /sys/bus/pci/devices/0000:03:00.0/sriov_totalvfs

# 查看 VF 的链路状态
cat /sys/bus/pci/devices/0000:03:00.0/sriov_vf_device

# 2. 动态调整 VF 数量
# 设置为 0 表示销毁所有 VF
echo 0 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs

# 重新创建 2 个 VF
echo 2 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs

# 3. VF 的 MAC 地址和 VLAN 设置（类似网卡 VF 管理）
# 注意：GPU SR-IOV 通常不支持 MAC/VLAN 设置，但 PCIe ACS 仍需配置

# 4. 绑定 VF 到 VFIO 驱动（用于直通到 VM）
# 对于 GPU VF，通常使用 vfio-pci 驱动
echo "vfio-pci" > /sys/bus/pci/devices/0000:03:00.1/driver_override
echo 0000:03:00.1 > /sys/bus/pci/devices/0000:03:00.1/driver/unbind
echo 0000:03:00.1 > /sys/bus/pci/drivers/vfio-pci/bind

# 5. 检查 VF 的 MMIO 和内存资源
lspci -vvv -s 03:00.1 | grep -A 10 "Region\|Memory"

# 输出示例：
# Region 0: Memory at c0000000 (64-bit, prefetchable) [size=256M]
# Region 2: Memory at d0000000 (64-bit, prefetchable) [size=2M]
# Region 4: Memory at d0200000 (64-bit, prefetchable) [size=1M]
```

#### SR-IOV 配置脚本示例

```bash
#!/bin/bash
# 文件名: sriov-gpu-setup.sh
# 用途: 自动配置 AMDGPU SR-IOV 环境

set -euo pipefail

# 配置参数
GPU_BDF="${GPU_BDF:-0000:03:00.0}"
VF_COUNT="${VF_COUNT:-4}"
IOMMU_DOMAIN="${IOMMU_DOMAIN:-pt}"

log() {
    echo "[$(date '+%H:%M:%S')] $*"
}

check_prerequisites() {
    log "检查 SR-IOV 前置条件..."

    # 检查内核支持
    if [ ! -f /sys/bus/pci/devices/${GPU_BDF}/sriov_totalvfs ]; then
        log "错误: GPU ${GPU_BDF} 不支持 SR-IOV 或驱动未加载"
        exit 1
    fi

    # 检查 IOMMU
    if [ ! -d /sys/kernel/iommu_groups ]; then
        log "警告: IOMMU 可能未启用"
    fi

    TOTAL_VFS=$(cat /sys/bus/pci/devices/${GPU_BDF}/sriov_totalvfs)
    log "GPU 支持的最大 VF 数量: ${TOTAL_VFS}"

    if [ "${VF_COUNT}" -gt "${TOTAL_VFS}" ]; then
        log "错误: 请求 ${VF_COUNT} 个 VF，但最多支持 ${TOTAL_VFS}"
        exit 1
    fi
}

enable_sriov() {
    log "启用 SR-IOV，创建 ${VF_COUNT} 个 VF..."

    # 先销毁已有 VF
    if [ "$(cat /sys/bus/pci/devices/${GPU_BDF}/sriov_numvfs)" -ne 0 ]; then
        log "销毁已有 VF..."
        echo 0 > /sys/bus/pci/devices/${GPU_BDF}/sriov_numvfs
        sleep 2
    fi

    # 创建 VF
    echo ${VF_COUNT} > /sys/bus/pci/devices/${GPU_BDF}/sriov_numvfs
    sleep 1

    log "VF 创建完成"
}

verify_vfs() {
    log "验证 VF..."

    local count=0
    for vf in $(ls /sys/bus/pci/devices/${GPU_BDF}/virtfn* 2>/dev/null); do
        local vf_bdf=$(basename $(readlink ${vf}))
        log "  VF ${count}: ${vf_bdf}"

        # 检查 VF BAR
        local bar0=$(cat /sys/bus/pci/devices/${vf_bdf}/resource0 2>/dev/null | wc -c)
        log "    BAR0: ${bar0} bytes"

        count=$((count + 1))
    done

    log "已创建 ${count}/${VF_COUNT} 个 VF"
}

setup_iommu() {
    log "配置 IOMMU 分组..."

    for vf in $(ls /sys/bus/pci/devices/${GPU_BDF}/virtfn* 2>/dev/null); do
        local vf_bdf=$(basename $(readlink ${vf}))
        local iommu_group=$(readlink /sys/bus/pci/devices/${vf_bdf}/iommu_group 2>/dev/null | xargs basename 2>/dev/null)

        if [ -n "${iommu_group}" ]; then
            log "  ${vf_bdf} -> IOMMU Group ${iommu_group}"
        else
            log "  ${vf_bdf} -> 无 IOMMU 分组"
        fi
    done
}

show_summary() {
    log ""
    log "=============================================="
    log "  SR-IOV 配置摘要"
    log "=============================================="
    log "  GPU BDF:      ${GPU_BDF}"
    log "  VF 数量:      ${VF_COUNT}"
    log "  IOMMU Domain: ${IOMMU_DOMAIN}"
    log ""
    log "  使用 libvirt/QEMU 将 VF 分配给 VM:"
    log "  virsh nodedev-detach pci_0000_03_00_1"
    log "  virsh nodedev-detach pci_0000_03_00_3"
    log "=============================================="
}

main() {
    log "AMDGPU SR-IOV 环境配置开始"

    check_prerequisites
    enable_sriov
    verify_vfs
    setup_iommu
    show_summary

    log "配置完成"
}

main
```

#### VF 驱动配置

如果主机内核包含了 amdgpu VF 驱动，VF 创建后会自动绑定到 amdgpu 驱动。但对于直通到虚拟机的场景，需要在 VM 中安装 VF 驱动：

```bash
# 在 VM Guest 中检查 VF 是否可见
lspci | grep "AMD\|ATI"

# 加载 amdgpu VF 模式
# VF 驱动通常以独立模块或参数形式提供
modprobe amdgpu

# 验证 VF 驱动加载
dmesg | grep "amdgpu.*VF\|amdgpu.*virtual"

# 检查 VF 功能
cat /sys/kernel/debug/dri/0/amdgpu_vm_info
cat /sys/kernel/debug/dri/0/amdgpu_mem_info

# VF 模式下受限的命令示例：
# 以下操作在 VF 中会被拒绝（由 PF 控制）：
# - 调整时钟频率 （set_sclk）
# - 修改电源策略 （set_power_profile）
# - 设置 OverDrive （set_od）
# - 全局 GPU 重置 （gpu_recover）
```

#### SR-IOV 环境下的测试注意事项

```bash
# 1. 测试 VF 间的隔离性
# 在 VF0 中分配大量显存
# VF0 中执行：
./gpu_mem_test --allocate 2048 --pattern 0xAAAAAAAA

# 在 VF1 中读取相同地址范围
# VF1 中执行：
./gpu_mem_test --read 0x10000000 --size 4096

# 预期：VF1 应返回 0（未初始化）或自己的数据，而非 VF0 的数据

# 2. 测试 VF 的性能隔离
# VF0 运行计算密集型负载
./gpu_burn.sh &

# VF1 测量渲染性能
./gpu_benchmark --frames 1000

# 对比：单 VF 运行时 vs 多 VF 并发时的性能差异

# 3. 测试 VF 恢复
# 模拟 VF hang
echo 1 > /sys/kernel/debug/dri/0/amdgpu_ring_hang

# 检查 VF 恢复日志
dmesg | tail -20
# 预期：仅该 VF 受影响，其他 VF 继续运行
```

### 案例分析

#### 案例一：数据中心 GPU 虚拟化部署

某 AI 训练平台需要在一张 AMD Instinct MI250 GPU 上同时运行 4 个深度学习训练任务，每个任务需要独立的显存空间和计算资源。

**需求分析：**

```
部署要求：
├── GPU: AMD Instinct MI250 (128GB HBM2e)
├── VF 数量: 4
├── 每个 VF:
│   ├── 显存: 28 GB   (合计 112GB，预留 16GB 给 PF)
│   ├── CU: 8 个 (动态调度)
│   └── 带宽: 25% PCIe 带宽
└── 工作负载: PyTorch 训练任务
```

**配置方案：**

```bash
#!/bin/bash
# 数据中心 SR-IOV 部署脚本

# 1. 创建 4 个 VF
echo 4 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs

# 2. 配置显存分区（需要 PF 驱动支持）
echo "vf0:0-28G vf1:28G-56G vf2:56G-84G vf3:84G-112G" > \
    /sys/bus/pci/devices/0000:03:00.0/vram_partition

# 3. 配置带宽限制（需要硬件支持）
echo "vf0:25 vf1:25 vf2:25 vf3:25" > \
    /sys/bus/pci/devices/0000:03:00.0/bandwidth_limit

# 4. 绑定 VF 到 vfio-pci（准备直通）
for vf_bdf in 03:00.1 03:00.3 03:00.5 03:00.7; do
    virsh nodedev-detach pci_0000_${vf_bdf//[:.]/_}
done

# 5. 启动虚拟机（使用 libvirt）
for i in {0..3}; do
    cat > /etc/libvirt/qemu/gpu-vm-${i}.xml << EOF
<domain type='kvm'>
  <name>gpu-vm-${i}</name>
  <memory unit='GiB'>32</memory>
  <vcpu>16</vcpu>
  <devices>
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x03' slot='0x00' function='$((i * 2 + 1))'/>
      </source>
    </hostdev>
  </devices>
</domain>
EOF
    virsh define /etc/libvirt/qemu/gpu-vm-${i}.xml
    virsh start gpu-vm-${i}
done
```

**测试验证：**

```bash
# 验证各 VM 中的 GPU 可见性
for i in {0..3}; do
    echo "VM ${i}:"
    virsh qemu-agent-command gpu-vm-${i} \
        '{"execute":"guest-exec","arguments":{"path":"/usr/bin/lspci","arg":["-d","1002:"]}}'
done

# 并发运行训练任务
for i in {0..3}; do
    virsh qemu-agent-command gpu-vm-${i} \
        '{"execute":"guest-exec","arguments":{"path":"/usr/bin/python3","arg":["train.py","--epochs","100"]}}'
done

# 监控性能隔离
watch -n 1 'cat /sys/bus/pci/devices/0000:03:00.0/vf_utilization'
```

**结果分析：**
- 4 个 VF 成功创建，每个 VM 识别到 GPU VF
- 并发训练运行时，各 VF 的性能方差 < 5%
- 其中一个 VF 的 PyTorch 任务因 OOM 失败时，其他 VF 不受影响
- PF 仍可正常管理和监控所有 VF 状态

#### 案例二：SR-IOV 故障排除

**问题现象：** 创建 VF 时返回 -ENOSPC（No space left on device）错误。

```bash
# 尝试创建 VF
echo 4 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs
# bash: echo: write error: No space left on device
```

**诊断过程：**

```bash
# 1. 检查可用 VF 数量
cat /sys/bus/pci/devices/0000:03:00.0/sriov_totalvfs
# 输出: 4

# 2. 检查 dmesg 获取详细错误
dmesg | tail -20
# [12345.678901] amdgpu 0000:03:00.0: VF BAR0 allocation failed
# [12345.678902] amdgpu 0000:03:00.0: SR-IOV: Failed to allocate VF resources

# 3. 检查 BAR 空间占用
lspci -vvv -s 03:00.0 | grep -E "Region|Memory"

# Region 0: Memory at a0000000 (64-bit, prefetchable) [size=256M]
# Region 2: Memory at b0000000 (64-bit, prefetchable) [size=16M]

# 4. 检查 iommu 内存保留
cat /proc/iomem | grep "PCI"

# 5. 检查大页预留
cat /proc/meminfo | grep -i "huge\|Huge"
```

**根本原因：** 系统预留的 IOMMU 地址空间不足，无法为 VF 分配 DMA 地址空间。

**解决方案：**

```bash
# 1. 增加 IOMMU 地址空间
# 编辑内核命令行，添加 iommu 参数
# amd_iommu=on iommu=pt iommu_size=256M

# 2. 释放不必要的内存预留
# 减少大页预留
echo 0 > /proc/sys/vm/nr_hugepages

# 3. 重新分配 VF 数量
echo 0 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs
sleep 2
echo 2 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs  # 先减少到 2

# 4. 如果仍然失败，检查 BIOS 设置
# 确保 BIOS 中启用了：
# - SR-IOV Support
# - Above 4G Decoding
# - Resizable BAR Capability
```

### 相关链接

- [PCI-SIG SR-IOV 规范](https://pcisig.com/specifications/iov/single_root/)
- [Linux 内核 SR-IOV 文档](https://www.kernel.org/doc/Documentation/PCI/pci-iov-howto.rst)
- [AMDGPU SR-IOV 驱动文档](https://docs.amd.com/)
- [KVM SR-IOV 配置指南](https://www.linux-kvm.org/page/How_to_assign_devices_with_SR-IOV)
- [IOMMU 与 PCIe ATS 文档](https://www.kernel.org/doc/Documentation/x86/intel-iommu.txt)

### 今日小结

SR-IOV 是 GPU 虚拟化的关键技术，通过硬件辅助方式实现高性能的多租户 GPU 共享：

1. **核心概念**：PF 管理整卡资源，VF 提供轻量级独立功能
2. **硬件要求**：GPU 固件支持 SR-IOV，主板支持 IOMMU
3. **资源分区**：显存静态分区，计算单元动态调度
4. **隔离性**：PCIe ACS + IOMMU + 硬件仲裁三重隔离
5. **管理方式**：sysfs 接口动态创建/销毁 VF
6. **驱动模型**：PF 驱动完整功能，VF 驱动轻量级子集

### 扩展思考

1. SR-IOV 在 GPU 上的实现比网卡复杂得多，主要挑战来自哪些方面（资源类型多样性、状态依赖性、调度公平性）？
2. 如何在大规模数据中心中实现 SR-IOV VF 的热迁移？需要哪些硬件和软件支持？
3. 对比 NVIDIA MIG（Multi-Instance GPU）与 AMD SR-IOV 的设计差异，两者在分区粒度、隔离性和性能方面的取舍有何不同？
4. 对于 GPU 驱动测试而言，SR-IOV 环境下的测试策略需要额外关注哪些方面（VF 隔离性测试、并发测试、恢复测试）？
