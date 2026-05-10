# Day 387：KVM GPU 直通（Passthrough）配置

## 学习目标

- 深入理解 VFIO 驱动的架构和工作原理
- 掌握 ACS（Access Control Service）和 IOMMU 分组的配置方法
- 学会 GPU 直通的完整配置流程（从硬件准备到 VM 运行）
- 掌握 GPU 重置（Reset）机制和问题处理
- 理解 vCPU 绑定（Pinning）和 NUMA 亲和性优化

## 知识详解

### 概念原理

#### VFIO 驱动栈详解

VFIO 是 Linux 内核中实现设备直通的核心框架：

```
VFIO 完整驱动栈：

┌─────────────────────────────────────────────────────┐
│                   用户空间 (QEMU)                     │
│                                                     │
│  ┌─────────────────────────────────────────────────┐│
│  │          打开 /dev/vfio/N (组文件)               ││
│  │  1. ioctl(VFIO_GET_IOMMU_GROUP_INFO)           ││
│  │  2. ioctl(VFIO_SET_IOMMU)                      ││
│  │  3. ioctl(VFIO_GROUP_GET_DEVICE_FD, "0000:...) ││
│  │  4. ioctl(VFIO_DEVICE_GET_REGION_INFO)         ││
│  │  5. mmap(设备 BAR 区域)                        ││
│  │  6. ioctl(VFIO_DEVICE_GET_IRQ_INFO)            ││
│  └─────────────────────────────────────────────────┘│
│                      │                               │
└──────────────────────┼───────────────────────────────┘
                       │ 系统调用
┌──────────────────────┼───────────────────────────────┐
│                  内核空间                             │
│                      ▼                               │
│  ┌─────────────────────────────────────────────────┐│
│  │                VFIO 核心层                        ││
│  │  vfio.ko                                        ││
│  │  - IOMMU 组管理                                  ││
│  │  - 设备生命周期管理                              ││
│  │  - 安全审计                                     ││
│  └──────┬─────────────────────────────────┬────────┘│
│         │                                 │          │
│         ▼                                 ▼          │
│  ┌──────────────────┐    ┌─────────────────────────┐│
│  │ vfio_iommu_type1  │    │    vfio_pci (PCI 后端)  ││
│  │ (IOMMU 后端)      │    │                         ││
│  │                   │    │ - 设备配置空间访问       ││
│  │ - DMA 映射管理    │    │ - MMIO BAR 管理          ││
│  │ - IOMMU 页表操作  │    │ - PCIe 配置空间          ││
│  │ - 地址翻译        │    │ - MSI/MSI-X 中断管理     ││
│  │ - 内存固定        │    │ - 设备重置              ││
│  └────────┬──────────┘    └───────────┬─────────────┘│
│           │                           │              │
│           └──────┬────────────────────┘              │
│                  ▼                                   │
│  ┌─────────────────────────────────────────────────┐│
│  │                IOMMU 硬件层                      ││
│  │  (AMD-Vi / Intel VT-d)                          ││
│  │  - DMA Remapping                                ││
│  │  - Interrupt Remapping                          ││
│  │  - Device TLB 控制                             ││
│  └─────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

**VFIO 数据结构：**

```c
// 内核源码: drivers/vfio/vfio.c
// VFIO IOMMU 组结构
struct vfio_group {
    struct kref             kref;
    int                     minor;          // 设备文件 minor
    struct iommu_group      *iommu_group;   // 对应的 IOMMU 组
    struct list_head        device_list;    // 组内设备列表
    struct notifier_block   nb;             // IOMMU 通知器
    struct list_head        container_next; // container 链表
    enum vfio_group_type    type;           // 组类型
};

// VFIO container (IOMMU 域)
struct vfio_container {
    struct kref             kref;
    struct list_head        group_list;     // 组链表
    struct list_head        iommu_data_list; // IOMMU 数据
    struct vfio_iommu_driver *iommu_driver; // IOMMU 驱动
    bool                    noiommu;        // 是否无 IOMMU
};

// VFIO PCI 设备
struct vfio_pci_device {
    struct pci_dev          *pdev;          // PCI 设备
    void __iomem            *barmap[PCI_STD_NUM_BARS]; // BAR 映射
    struct vfio_pci_region  *region;        // 设备区域
    int                     num_regions;
    struct vfio_pci_irq_ctx *ctx;           // 中断上下文
    int                     num_ctx;
    struct mutex            reflck;         // 重置锁
    bool                    reset_works;    // 是否支持重置
    bool                    extended_caps;  // PCIe 扩展能力
};
```

#### IOMMU 分组与 ACS

IOMMU 分组是 GPU 直通的核心限制因素：

```
IOMMU 分组原理：

物理拓扑:
┌───────────── PCIe Root Complex ─────────────┐
│                                              │
│  ┌────────── PCIe Switch ──────────┐         │
│  │                                 │         │
│  │  ┌────┐  ┌────┐  ┌────┐  ┌────┐│         │
│  │  │GPU │  │网卡 │  │USB │  │NVMe││         │
│  │  │    │  │    │  │    │  │    ││         │
│  │  └────┘  └────┘  └────┘  └────┘│         │
│  └─────────────────────────────────┘         │
└──────────────────────────────────────────────┘

IOMMU 分组 (无 ACS):
┌───────────────────────────────────────────────┐
│  IOMMU Group 13:                               │
│  ├── 03:00.0 GPU (VGA)                        │
│  ├── 03:00.1 GPU (Audio)                      │
│  └── 03:00.2 GPU (USB)    ← 需要一起直通      │
└───────────────────────────────────────────────┘

IOMMU 分组 (有 ACS):
┌───────────────────────────────────────────────┐
│  IOMMU Group 13: 03:00.0 GPU                 │
│  IOMMU Group 14: 03:00.1 GPU Audio           │  ← 可独立直通
│  IOMMU Group 15: 03:00.2 GPU USB             │
└───────────────────────────────────────────────┘
```

**ACS (Access Control Service) 的作用：**

```
ACS 是 PCIe 规范中的可选功能，控制同一 PCIe Switch 下不同设备间的 DMA 访问：

ACS 功能矩阵：
┌────────────────────────────────────────────────────────┐
│ ACS 类型               │ 功能描述          │ 必需?    │
├────────────────────────────────────────────────────────┤
│ Source Validation       │ 验证请求来源有效性  │ Yes     │
│ Translation Blocking   │ 阻止 P2P 地址翻译  │ Yes     │
│ P2P Request Redirect   │ P2P 请求重定向      │ Optional │
│ P2P Completion Redirect│ P2P 完成重定向      │ Optional │
│ Upstream Forwarding    │ 上行转发控制        │ Optional │
│ P2P Egress             │ P2P 出口控制        │ Optional │
│ Direct Translated P2P  │ 直接翻译 P2P       │ Optional │
└────────────────────────────────────────────────────────┘

没有 ACS 的情况下，同一 Switch 下的设备可以：
  - GPU → 网卡: 直接 DMA (绕过 IOMMU) ← 安全隐患
  - 网卡 → GPU: 直接读取显存数据
```

#### GPU 重置机制

GPU 直通中的重置（Reset）是确保 VM 启动时 GPU 处于干净状态的关键：

```
GPU 重置机制层次:

1. 功能级别重置 (Function Level Reset - FLR)
   ┌─────────────────────────────────────────┐
   │ 通过 PCIe 配置空间触发                  │
   │ 写 1 到 PCI_EXP_DEVCTL_FLR bit          │
   │ 效果: 重置单个 PCIe 功能，不影响其他    │
   │ 延迟: ~100ms                           │
   │ 支持: 大部分现代 GPU 都支持             │
   └─────────────────────────────────────────┘

2. 热重置 (Secondary Bus Reset - SBR)
   ┌─────────────────────────────────────────┐
   │ 通过 PCIe 桥控制寄存器触发              │
   │ 效果: 重置桥下所有设备                  │
   │ 延迟: ~200ms                           │
   │ 用途: FLR 不够时的备选方案              │
   └─────────────────────────────────────────┘

3. GPU 特定重置 (AMDGPU GPU Reset)
   ┌─────────────────────────────────────────┐
   │ 通过 GPU 固件触发                       │
   │ 效果: 完全重置 GPU 状态                 │
   │ 过程:                                   │
   │   1. 停止所有调度队列                   │
   │   2. 通知固件执行重置                   │
   │   3. 等待重置完成                       │
   │   4. 重新初始化 GPU                     │
   │ 延迟: ~2-5s                            │
   └─────────────────────────────────────────┘

VRAM 持久性:
┌─────────────────────────────────────────────┐
│ 重置类型    │ VRAM 内容 │ 需要重新初始化?   │
├─────────────────────────────────────────────┤
│ FLR         │ 丢失      │ 是               │
│ SBR         │ 丢失      │ 是               │
│ Warm Reset  │ 保留      │ 部分             │
│ Cold Reset  │ 丢失      │ 完全             │
└─────────────────────────────────────────────┘
```

```c
// VFIO PCI 重置流程 (drivers/vfio/pci/vfio_pci_core.c)
static int vfio_pci_ioctl_set_irq(struct vfio_pci_device *vdev, ...)
{
    // ... 其他处理
    
    // 设备重置 IOCTL
    case VFIO_DEVICE_RESET:
        return vfio_pci_dev_try_reset(vdev);
}

static int vfio_pci_dev_try_reset(struct vfio_pci_device *vdev)
{
    struct pci_dev *pdev = vdev->pdev;
    int ret;
    
    // 1. 尝试 FLR
    ret = pcie_flr(pdev);
    if (ret == 0) {
        pci_info(pdev, "FLR successful\n");
        return 0;
    }
    
    // 2. FLR 失败，尝试 PM Reset (D3hot → D0)
    ret = pci_pm_reset(pdev, 0);
    if (ret == 0) {
        pci_info(pdev, "PM reset successful\n");
        return 0;
    }
    
    // 3. 尝试 AMD 特定重置 (如果有)
    ret = amdgpgu_gpu_reset(vdev);
    if (ret == 0) {
        pci_info(pdev, "AMDGPU reset successful\n");
        return 0;
    }
    
    // 4. 最后尝试 SBR (需要 PCIe 桥)
    ret = pci_bridge_secondary_bus_reset(pdev);
    if (ret == 0) {
        pci_info(pdev, "SBR successful\n");
        return 0;
    }
    
    return -ENODEV;  // 所有重置方法均失败
}
```

#### vCPU Pinning 与 NUMA 亲和性

```
NUMA 架构下的 GPU 直通优化：

物理 NUMA 拓扑:
┌─────────────────────────┬─────────────────────────┐
│      NUMA Node 0        │      NUMA Node 1        │
│                         │                         │
│  ┌───┐  ┌───┐  ┌───┐  │  ┌───┐  ┌───┐  ┌───┐  │
│  │C0 │  │C1 │  │C2 │  │  │C4 │  │C5 │  │C6 │  │  │
│  └───┘  └───┘  └───┘  │  └───┘  └───┘  └───┘  │
│                         │                         │
│  ┌─────────────────┐   │  ┌─────────────────┐   │
│  │  Memory (32GB)  │   │  │  Memory (32GB)  │   │
│  └─────────────────┘   │  └─────────────────┘   │
│         │              │         │              │
│         ▼              │         ▼              │
│  ┌──────────┐          │  ┌──────────┐          │
│  │ GPU 0   │          │  │ GPU 1   │          │
│  │ (PCIe)  │          │  │ (PCIe)  │          │
│  └──────────┘          │  └──────────┘          │
└─────────────────────────┴─────────────────────────┘

优化配置: GPU 0 在 NUMA Node 0 上
  - vCPU 绑定到 Node 0 的核心 (C0-C3)
  - VM 内存分配在 Node 0
  - 避免跨 NUMA 访问（延迟增加 30-50%）

不优化时: GPU 0 在 NUMA Node 0, VM 在 Node 1
  - 每次 GPU DMA 操作都需要跨 NUMA 访问
  - 性能损失: 20-40%
```

### 实践操作

#### 完整的 GPU 直通配置流程

```bash
#!/bin/bash
# 文件名: gpu_passthrough_setup.sh
# 用途: 完整的 GPU 直通配置流程

set -e

echo "=============================================="
echo "  KVM GPU 直通配置脚本"
echo "=============================================="
echo ""

# 步骤 1: 检查和启用 IOMMU
step_check_iommu() {
    echo "[1/7] 检查 IOMMU..."
    
    if dmesg | grep -qi "AMD-Vi\|Intel VT-d"; then
        echo "  ✅ IOMMU 已启用"
    else
        echo "  ⚠️ IOMMU 未启用"
        echo "  正在启用 IOMMU..."
        
        # 检测 CPU 并配置
        if grep -qi "amd" /proc/cpuinfo; then
            IOMMU_PARAM="amd_iommu=on iommu=pt"
        else
            IOMMU_PARAM="intel_iommu=on iommu=pt"
        fi
        
        grubby --args="${IOMMU_PARAM}" --update-kernel=DEFAULT 2>/dev/null || \
        sed -i "s/GRUB_CMDLINE_LINUX_DEFAULT=\"/GRUB_CMDLINE_LINUX_DEFAULT=\"${IOMMU_PARAM} /" /etc/default/grub
        
        if command -v update-grub &> /dev/null; then
            update-grub
        elif command -v grub2-mkconfig &> /dev/null; then
            grub2-mkconfig -o /boot/grub2/grub.cfg
        fi
        
        echo "  ⚠️ 请重启系统后继续"
        echo "  重启后重新执行: $0 --continue"
        exit 0
    fi
}

# 步骤 2: 识别 GPU 和 IOMMU 分组
step_identify_gpu() {
    echo ""
    echo "[2/7] 识别 GPU 设备和 IOMMU 分组..."
    
    # 列出所有 AMD GPU
    echo "  AMD GPU 设备:"
    GPU_LIST=$(lspci -D | grep -i "AMD\|ATI" | grep -i "VGA\|Display\|3D")
    echo "${GPU_LIST}" | while read line; do
        echo "    ${line}"
    done
    
    # 选择直通的 GPU (如果不是第一块)
    read -p "  请输入要直通的 GPU BDF (例如 0000:03:00.0): " GPU_BDF
    
    if [ -z "${GPU_BDF}" ]; then
        # 自动选择非启动 GPU
        GPU_BDF=$(echo "${GPU_LIST}" | tail -1 | awk '{print $1}')
        echo "  自动选择: ${GPU_BDF}"
    fi
    
    # 查找 IOMMU 分组
    for group in /sys/kernel/iommu_groups/*; do
        if [ -e "${group}/devices/${GPU_BDF}" ]; then
            IOMMU_GROUP=$(basename ${group})
            echo "  GPU 在 IOMMU Group ${IOMMU_GROUP}"
            
            # 列出同组所有设备
            echo "  同组设备 (必须全部直通):"
            for dev in ${group}/devices/*; do
                DEV_BDF=$(basename ${dev})
                DEV_NAME=$(lspci -s ${DEV_BDF} 2>/dev/null | cut -d' ' -f2-)
                echo "    ${DEV_BDF} → ${DEV_NAME}"
            done
            break
        fi
    done
    
    # 导出变量供后续步骤使用
    echo "export GPU_BDF=${GPU_BDF}" > /tmp/gpu_passthrough_env.sh
    echo "export IOMMU_GROUP=${IOMMU_GROUP}" >> /tmp/gpu_passthrough_env.sh
    echo "export GPU_DOMAIN=$(echo ${GPU_BDF} | cut -d: -f1)" >> /tmp/gpu_passthrough_env.sh
    echo "export GPU_BUS=$(echo ${GPU_BDF} | cut -d: -f2)" >> /tmp/gpu_passthrough_env.sh
    echo "export GPU_SLOT=$(echo ${GPU_BDF} | cut -d: -f3 | cut -d. -f1)" >> /tmp/gpu_passthrough_env.sh
    echo "export GPU_FUNC=$(echo ${GPU_BDF} | cut -d. -f2)" >> /tmp/gpu_passthrough_env.sh
}

# 步骤 3: 配置 VFIO 驱动绑定
step_configure_vfio() {
    echo ""
    echo "[3/7] 配置 VFIO 驱动绑定..."
    
    source /tmp/gpu_passthrough_env.sh
    
    # 获取 GPU 的 Vendor:Device ID
    GPU_VENDOR=$(cat /sys/bus/pci/devices/${GPU_BDF}/vendor | sed 's/0x//')
    GPU_DEVICE=$(cat /sys/bus/pci/devices/${GPU_BDF}/device | sed 's/0x//')
    
    # 获取同组其他设备的 ID
    ALL_IDS="${GPU_VENDOR}:${GPU_DEVICE}"
    for group in /sys/kernel/iommu_groups/*; do
        if [ -e "${group}/devices/${GPU_BDF}" ]; then
            for dev in ${group}/devices/*; do
                DEV_BDF=$(basename ${dev})
                if [ "${DEV_BDF}" != "${GPU_BDF}" ]; then
                    VENDOR=$(cat ${dev}/vendor | sed 's/0x//')
                    DEV=$(cat ${dev}/device | sed 's/0x//')
                    ALL_IDS="${ALL_IDS},${VENDOR}:${DEV}"
                fi
            done
            break
        fi
    done
    
    echo "  设备 IDs: ${ALL_IDS}"
    
    # 配置 vfio-pci
    cat > /etc/modprobe.d/vfio-pci.conf << EOF
options vfio-pci ids=${ALL_IDS}
options vfio-pci disable_vga=1
softdep amdgpu pre: vfio-pci
EOF
    
    # 配置模块加载
    cat > /etc/modules-load.d/vfio-pci.conf << EOF
vfio
vfio_iommu_type1
vfio_pci
EOF
    
    # 更新 initramfs
    echo "  更新 initramfs..."
    update-initramfs -u || dracut -f
    
    echo "  ✅ VFIO 配置完成"
    echo "  ⚠️ 需要重启系统才能生效"
}

# 步骤 4: 配置 HugePages
step_configure_hugepages() {
    echo ""
    echo "[4/7] 配置 HugePages..."
    
    # 计算建议的 HugePages 数量
    local vm_memory_gb=$1
    local suggested_hugepages=$((vm_memory_gb * 1024 / 2))  # 2MB per page
    
    echo "  配置 ${suggested_hugepages} 个 HugePages (${vm_memory_gb}GB VM)"
    
    # 持久化配置
    echo "vm.nr_hugepages=${suggested_hugepages}" > /etc/sysctl.d/99-hugepages.conf
    
    # 立即生效
    sysctl -w vm.nr_hugepages=${suggested_hugepages}
    
    # 创建 HugePages 挂载点
    mkdir -p /dev/hugepages
    mount -t hugetlbfs hugetlbfs /dev/hugepages
    
    # 持久化挂载
    if ! grep -q "hugetlbfs" /etc/fstab; then
        echo "hugetlbfs /dev/hugepages hugetlbfs defaults 0 0" >> /etc/fstab
    fi
    
    echo "  ✅ HugePages 配置完成"
}

# 步骤 5: 创建 VM 启动脚本
step_create_vm_script() {
    echo ""
    echo "[5/7] 创建 VM 启动脚本..."
    
    source /tmp/gpu_passthrough_env.sh
    
    local vm_name="gpu-vm"
    local vm_memory="16"
    local vm_cpus="8"
    
    cat > /usr/local/bin/start-${vm_name}.sh << 'VMSCRIPT'
#!/bin/bash
# GPU 直通 VM 启动脚本
    
VM_NAME="gpu-vm"
GPU_BDF=$(cat /tmp/gpu_passthrough_env.sh | grep GPU_BDF | cut -d= -f2)
MEMORY="16G"
CPUS="8"
VM_IMAGE="/var/lib/libvirt/images/${VM_NAME}.qcow2"

# NUMA 优化: 绑定 vCPU 到 GPU 所在 NUMA 节点
# 检测 GPU 的 NUMA 节点
GPU_NUMA_NODE=$(cat /sys/bus/pci/devices/${GPU_BDF}/numa_node 2>/dev/null || echo 0)

# 获取该 NUMA 节点的 CPU 列表
NUMA_CPUS=$(lscpu | grep "NUMA node${GPU_NUMA_NODE}" | awk -F: '{print $2}' | tr -d ' ')

echo "启动 GPU 直通 VM..."
echo "  GPU: ${GPU_BDF}"
echo "  NUMA Node: ${GPU_NUMA_NODE}"
echo "  CPU Affinity: ${NUMA_CPUS}"
echo "  Memory: ${MEMORY}"
echo ""

# 配置 vCPU 亲和性 (使用 taskset)
CPU_MASK=""
if [ -n "${NUMA_CPUS}" ]; then
    # 解析 CPU 列表为 mask
    # 示例: 0-3,8-11 → 000000000000FF0F
    CPU_MASK=$(echo "${NUMA_CPUS}" | \
        awk -F, '{for(i=1;i<=NF;i++){if($i~/-/){split($i,a,"-");for(j=a[1];j<=a[2];j++)printf"%s ",j}else printf"%s ",$i}}' | \
        awk '{mask=0; for(i=1;i<=NF;i++) mask+=2^$i; printf "0x%x", mask}')
    
    echo "  CPU Mask: ${CPU_MASK}"
fi

# 启动 QEMU (使用 numactl 绑定 NUMA)
exec numactl --cpunodebind=${GPU_NUMA_NODE} --membind=${GPU_NUMA_NODE} \
    qemu-system-x86_64 \
        -name ${VM_NAME},process=${VM_NAME} \
        -machine type=q35,accel=kvm \
        -cpu host,kvm=on,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time \
        -smp ${CPUS} \
        -m ${MEMORY} \
        -mem-prealloc \
        -mem-path /dev/hugepages \
        -drive file=${VM_IMAGE},if=virtio,format=qcow2 \
        -device vfio-pci,host=${GPU_BDF} \
        $(lspci -s ${GPU_BDF%%.*}.1 > /dev/null 2>&1 && echo "-device vfio-pci,host=${GPU_BDF%%.*}.1") \
        -device virtio-net-pci,netdev=net0 \
        -netdev user,id=net0,hostfwd=tcp:2222-:22 \
        -vga none \
        -nographic \
        -display none \
        -monitor unix:/tmp/${VM_NAME}.sock,server,nowait \
        -global q35-pcihost.x-config-regs=on
VMSCRIPT
    
    chmod +x /usr/local/bin/start-${vm_name}.sh
    echo "  ✅ VM 启动脚本已创建: /usr/local/bin/start-${vm_name}.sh"
    echo "  启动: sudo /usr/local/bin/start-${vm_name}.sh"
}

# 步骤 6: ACS 补丁和解决 IOMMU 分组问题
step_acs_patch() {
    echo ""
    echo "[6/7] ACS 覆盖配置..."
    
    source /tmp/gpu_passthrough_env.sh
    
    # 检查是否需要 ACS 补丁
    echo "  检查 PCIe Switch ACS 支持..."
    
    # 找到 GPU 的上行端口 (Upstream Port)
    GPU_BUS_HEX=$(printf "%d" 0x$(echo ${GPU_BDF} | cut -d: -f2))
    for dev in /sys/bus/pci/devices/*; do
        SEC_BUS=$(cat ${dev}/pci_bus/... 2>/dev/null || echo "")
        SUB_BUS=$(cat ${dev}/subordinate_bus 2>/dev/null || echo "")
        
        if [ -n "${SEC_BUS}" ] && [ -n "${SUB_BUS}" ]; then
            if [ "${GPU_BUS_HEX}" -ge "${SEC_BUS}" ] && [ "${GPU_BUS_HEX}" -le "${SUB_BUS}" ]; then
                BRIDGE_BDF=$(basename ${dev})
                ACS_CAP=$(setpci -s ${BRIDGE_BDF} CAP_EXP+0x10.L 2>/dev/null || echo "")
                echo "  PCIe Bridge: ${BRIDGE_BDF}"
                echo "  ACS Capability: ${ACS_CAP:-不支持}"
                break
            fi
        fi
    done
    
    # 如果 PCIe Switch 不支持 ACS，需要内核补丁
    echo ""
    echo "  如果 GPU 和其他设备在同一 IOMMU 组:"
    echo "  选项 1: 应用 ACS 覆盖内核参数"
    echo "    pcie_acs_override=downstream,multifunction"
    echo ""
    echo "  选项 2: 分离设备到独立 IOMMU 组 (不推荐)"
    echo "    - 修改内核源码 drivers/pci/quirks.c"
    echo "    - 重新编译内核"
    echo ""
    
    read -p "  是否需要启用 ACS 覆盖? (y/N): " enable_acs
    if [ "${enable_acs}" == "y" ] || [ "${enable_ats}" == "Y" ]; then
        grubby --args="pcie_acs_override=downstream,multifunction" --update-kernel=DEFAULT 2>/dev/null || \
        sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="/GRUB_CMDLINE_LINUX_DEFAULT="pcie_acs_override=downstream,multifunction /' /etc/default/grub
        
        if command -v update-grub &> /dev/null; then
            update-grub
        elif command -v grub2-mkconfig &> /dev/null; then
            grub2-mkconfig -o /boot/grub2/grub.cfg
        fi
        
        echo "  ✅ ACS 覆盖已配置，重启生效"
    fi
}

# 步骤 7: 验证和测试
step_verify() {
    echo ""
    echo "[7/7] 验证配置..."
    
    source /tmp/gpu_passthrough_env.sh
    
    # 1. 验证 IOMMU 启用
    echo "  IOMMU: $(dmesg | grep -qi 'AMD-Vi\|Intel VT-d' && echo '✅' || echo '❌')"
    
    # 2. 验证 VFIO 模块
    for mod in vfio vfio_iommu_type1 vfio_pci; do
        echo "  ${mod}: $(lsmod | grep -q ^${mod} && echo '✅' || echo '❌')"
    done
    
    # 3. 验证 GPU 驱动绑定
    CURRENT_DRIVER=$(readlink /sys/bus/pci/devices/${GPU_BDF}/driver 2>/dev/null | xargs basename)
    echo "  GPU 驱动: ${CURRENT_DRIVER} $([ "${CURRENT_DRIVER}" == "vfio-pci" ] && echo '✅' || echo '⚠️ (需要重启)')"
    
    # 4. 验证 IOMMU 分组
    for group in /sys/kernel/iommu_groups/*; do
        if [ -e "${group}/devices/${GPU_BDF}" ]; then
            GROUP_DEVICES=$(ls ${group}/devices/ | wc -l)
            echo "  IOMMU 组: $(basename ${group}) (${GROUP_DEVICES} 设备)"
            echo "    ✅ 配置就绪"
        fi
    done
    
    # 5. 验证 HugePages
    HP_TOTAL=$(cat /proc/sys/vm/nr_hugepages)
    HP_FREE=$(grep HugePages_Free /proc/meminfo | awk '{print $2}')
    echo "  HugePages: ${HP_TOTAL} 总, ${HP_FREE} 空闲"
    
    echo ""
    echo "=============================================="
    echo "  配置完成! 请重启系统后启动 VM"
    echo "=============================================="
    echo ""
    echo "  启动 VM: sudo /usr/local/bin/start-gpu-vm.sh"
    echo "  连接 VM: ssh -p 2222 user@localhost"
    echo "  Monitor: socat - UNIX-CONNECT:/tmp/gpu-vm.sock"
    echo ""
    echo "  故障排除:"
    echo "    dmesg | grep -i 'vfio\|iommu\|amdgpgu'"
    echo "    cat /proc/$(pgrep qemu)/status | grep -i vms"
}

# 主流程
if [ "$1" == "--continue" ]; then
    # 跳过 IOMMU 检查继续
    . /tmp/gpu_passthrough_env.sh 2>/dev/null || true
    step_identify_gpu
    step_configure_vfio
    step_configure_hugepages 16
    step_create_vm_script
    step_acs_patch
    step_verify
else
    step_check_iommu
    step_identify_gpu
    step_configure_vfio
    step_configure_hugepages 16
    step_create_vm_script
    step_acs_patch
    step_verify
fi
```

#### RESET 问题处理和自定义重置

```bash
#!/bin/bash
# GPU 直通重置工具

echo "GPU 直通重置工具"
echo "========================"
echo ""

# 检测 GPU 重置支持
check_reset_capability() {
    local gpu_bdf=$1
    
    echo "检查 GPU 重置能力:"
    echo "  GPU: ${gpu_bdf}"
    
    # 1. 检查 FLR 支持
    local flr_support=$(setpci -s ${gpu_bdf} CAP_EXP+0x08.W 2>/dev/null)
    if [ -n "${flr_support}" ]; then
        local flr_bit=$(( (0x${flr_support} >> 15) & 1 ))
        if [ "${flr_bit}" == "1" ]; then
            echo "  ✅ FLR: 支持"
        else
            echo "  ❌ FLR: 不支持"
        fi
    fi
    
    # 2. 检查 PM 重置 (D3hot → D0)
    local pm_cap=$(setpci -s ${gpu_bdf} CAP_EXP+0x02.W 2>/dev/null)
    echo "  ✅ PM Reset: 通常可用"
    
    # 3. 检查 AMDGPU 特定重置
    local vendor=$(cat /sys/bus/pci/devices/${gpu_bdf}/vendor 2>/dev/null)
    if [ "${vendor}" == "0x1002" ]; then
        echo "  ✅ AMDGPU Reset: AMD GPU 通用重置"
    fi
    
    # 4. 检查 VFIO 重置接口
    if [ -f /sys/bus/pci/devices/${gpu_bdf}/reset ]; then
        echo "  ✅ VFIO Reset: 可通过 sysfs 触发"
    fi
}

# 手动触发 FLR
trigger_flr() {
    local gpu_bdf=$1
    
    echo "触发 FLR..."
    echo "  GPU: ${gpu_bdf}"
    
    # 方法 1: 通过 setpci
    # 设置 FLR bit (PCIe 设备控制寄存器, offset 0x08, bit 15)
    local orig_val=$(setpci -s ${gpu_bdf} CAP_EXP+0x08.W 2>/dev/null)
    local flr_val=$(( 0x${orig_val} | 0x8000 ))
    setpci -s ${gpu_bdf} CAP_EXP+0x08.W=0x$(printf '%04X' ${flr_val})
    
    # 等待重置完成 (最多 5s)
    for i in $(seq 1 50); do
        if setpci -s ${gpu_bdf} VENDOR0.L > /dev/null 2>&1; then
            echo "  ✅ FLR 完成 (${i}00ms)"
            return 0
        fi
        sleep 0.1
    done
    
    echo "  ❌ FLR 超时"
    return 1
}

# 触发 AMDGPU 特定重置
trigger_amdgpgu_reset() {
    local gpu_bdf=$1
    
    echo "触发 AMDGPU 特定重置..."
    echo "  GPU: ${gpu_bdf}"
    
    # 检查是否有 amdgpu 重置接口
    local reset_path="/sys/bus/pci/devices/${gpu_bdf}/reset_method"
    if [ -f "${reset_path}" ]; then
        echo "  支持的 reset 方法: $(cat ${reset_path})"
        echo "  当前 reset 方法: $(cat ${reset_path}/... 2>/dev/null || echo '默认')"
    fi
    
    # 通过 debugfs 触发 GPU 重置
    local debugfs_reset="/sys/kernel/debug/dri/0/amdgpu_gpu_recover"
    if [ -f "${debugfs_reset}" ]; then
        echo "  触发 GPU recovery..."
        echo 1 > "${debugfs_reset}"
        echo "  ✅ GPU recovery 已触发"
    else
        echo "  ⚠️ 无法直接触发 GPU reset"
        echo "  QEMU 在启动 VM 时会自动触发"
    fi
}

# 处理 VM 退出后的 GPU 状态清理
cleanup_gpu_after_vm() {
    local gpu_bdf=$1
    
    echo "清理 VM 退出后的 GPU 状态..."
    
    # 1. 确保 GPU 重新绑定到 vfio-pci
    local current_driver=$(readlink /sys/bus/pci/devices/${gpu_bdf}/driver 2>/dev/null | xargs basename)
    if [ "${current_driver}" != "vfio-pci" ]; then
        echo "  重新绑定到 vfio-pci..."
        echo "${gpu_bdf}" > /sys/bus/pci/drivers/vfio-pci/bind 2>/dev/null || true
    fi
    
    # 2. 触发重置
    echo "  触发设备重置..."
    echo 1 > /sys/bus/pci/devices/${gpu_bdf}/reset 2>/dev/null || true
    
    sleep 2
    
    # 3. 验证 VFIO 设备状态
    vfio_group=$(readlink /sys/bus/pci/devices/${gpu_bdf}/iommu_group 2>/dev/null | xargs basename)
    if [ -c "/dev/vfio/${vfio_group}" ]; then
        echo "  ✅ VFIO 设备就绪 (Group ${vfio_group})"
    else
        echo "  ⚠️ VFIO 设备不可用"
    fi
}

# 重置所有 VFIO 设备 (VM crash 恢复)
reset_all_vfio() {
    echo "重置所有 VFIO 设备..."
    
    # 杀掉所有持 QEMU 进程
    pkill -9 qemu-system-x86_64 2>/dev/null || true
    sleep 2
    
    # 遍历所有 VFIO 设备
    for vfio_dev in /sys/bus/pci/drivers/vfio-pci/*/reset; do
        if [ -f "${vfio_dev}" ]; then
            local bdf=$(echo ${vfio_dev} | awk -F/ '{print $(NF-1)}')
            echo "  重置: ${bdf}"
            echo 1 > "${vfio_dev}" 2>/dev/null || true
        fi
    done
    
    echo "  ✅ 所有 VFIO 设备已重置"
}

# 主菜单
case "$1" in
    check)
        check_reset_capability "$2"
        ;;
    flr)
        trigger_flr "$2"
        ;;
    amd)
        trigger_amdgpgu_reset "$2"
        ;;
    cleanup)
        cleanup_gpu_after_vm "$2"
        ;;
    reset-all)
        reset_all_vfio
        ;;
    *)
        echo "用法: $0 {check|flr|amd|cleanup|reset-all} [BDF]"
        echo ""
        echo "示例:"
        echo "  $0 check 0000:03:00.0   # 检查重置能力"
        echo "  $0 flr 0000:03:00.0     # 触发 FLR"
        echo "  $0 amd 0000:03:00.0     # AMD GPU 重置"
        echo "  $0 cleanup 0000:03:00.0 # VM 退出后清理"
        echo "  $0 reset-all            # 重置所有 VFIO 设备"
        ;;
esac
```

#### vCPU Pinning 和 NUMA 优化

```bash
#!/bin/bash
# VM CPU 和内存优化配置

echo "VM vCPU Pinning 和 NUMA 优化"
echo "================================"
echo ""

# 检测 NUMA 拓扑
echo "NUMA 拓扑:"
numactl --hardware | while read line; do
    echo "  ${line}"
done

echo ""

# 检测 GPU 所在 NUMA 节点
GPU_BDF="0000:03:00.0"  # 替换为实际的 GPU BDF
GPU_NUMA_NODE=$(cat /sys/bus/pci/devices/${GPU_BDF}/numa_node 2>/dev/null)
echo "GPU ${GPU_BDF} 所在 NUMA 节点: ${GPU_NUMA_NODE:-0}"

# 获取指定 NUMA 节点的 CPU 列表
get_numa_cpus() {
    local node=$1
    lscpu | grep "NUMA node${node}" | awk -F: '{print $2}' | tr -d ' '
}

NUMA_NODE=${GPU_NUMA_NODE:-0}
NODE_CPUS=$(get_numa_cpus ${NUMA_NODE})
echo "NUMA Node ${NUMA_NODE} 的 CPU: ${NODE_CPUS}"

# 生成 libvirt XML 的 CPU pinning 配置
echo ""
echo "libvirt CPU pinning 配置:"
echo ""

cat << XML_CFG
<vcpu placement='static'>8</vcpu>
<cputune>
XML_CFG

# 为每个 vCPU 分配物理核心
CPU_INDEX=0
for cpu in ${NODE_CPUS}; do
    echo "  <vcpupin vcpu='${CPU_INDEX}' cpuset='${cpu}'/>"
    CPU_INDEX=$((CPU_INDEX + 1))
    if [ ${CPU_INDEX} -ge 8 ]; then
        break
    fi
done

cat << XML_CFG
</cputune>

<numatune>
  <memory mode='strict' nodeset='${NUMA_NODE}'/>
  <memnode cellid='0' mode='strict' nodeset='${NUMA_NODE}'/>
</numatune>
XML_CFG

echo ""

# 生成 QEMU 命令行 NUMA 优化的启动脚本
echo "QEMU 带 NUMA 优化的启动命令:"
echo ""

cat << QEMU_CMD
numactl --cpunodebind=${NUMA_NODE} --membind=${NUMA_NODE} \\
    qemu-system-x86_64 \\
        -machine type=q35,accel=kvm \\
        -cpu host,kvm=on,hv_relaxed,hv_spinlocks=0x1fff \\
        -smp 8 \\
        -m 16G \\
        -object memory-backend-file,id=mem0,size=16G,mem-path=/dev/hugepages,share=on \\
        -numa node,memdev=mem0 \\
        -device vfio-pci,host=${GPU_BDF} \\
        ...
QEMU_CMD

echo ""

# 验证当前 vCPU 绑定
echo "当前 QEMU 进程的 CPU 绑定:"
for pid in $(pgrep -f "qemu.*gpu-vm" 2>/dev/null); do
    echo "  PID ${pid}:"
    taskset -cp ${pid} 2>/dev/null || echo "  无法读取"
done
```

### 案例分析

#### 案例一：GPU 直通后 VM 内 GPU 重置失败

**问题现象：**
```bash
# VM 内 dmesg 循环重置
[   12.345] amdgpu 0000:00:05.0: [drm] GPU reset begin
[   12.345] amdgpu 0000:00:05.0: [drm] GPU reset end
[   12.350] amdgpu 0000:00:05.0: [drm] GPU reset begin  ← 循环
[   12.350] amdgpu 0000:00:05.0: [drm] GPU reset end
# 无限循环，VM 无法正常使用
```

**诊断过程：**
```bash
# 1. Host 端检查 GPU 重置接口
cat /sys/bus/pci/devices/0000:03:00.0/reset_method
# 输出: flr (只支持 FLR)

# 2. 检查 FLR 执行
echo 1 > /sys/bus/pci/devices/0000:03:00.0/reset
dmesg | tail -5
# [  123.456] vfio-pci 0000:03:00.0: FLR in progress
# [  123.789] vfio-pci 0000:03:00.0: FLR failed (-ENODEV)  ← FLR 失败

# 3. 检查 GPU 的 PCIe 能力
lspci -vvvs 03:00.0 | grep -i "devctl\|FLR"
# DevCtl: Report errors... ExtTag- PhantFunc- AuxP- NoSnoop- FLR-
# FLR- 表示不支持 FLR
```

**根本原因：** 该 GPU 不支持 FLR，PM reset 也没有正确触发。

**解决方案：**
```bash
# 方案 1: 添加 ACS 覆盖并使用 SBR
# 在 Host 内核参数中添加:
pcie_acs_override=downstream

# 重启后验证 SBR
echo 1 > /sys/bus/pci/devices/0000:00:02.0/reset  # PCIe 桥 SBR

# 方案 2: 使用 vendor 特定重置 (如果有)
# 修改 QEMU 源码或使用补丁
# 在 vfio-pci 中添加 AMDGPU 重置支持

# 方案 3: 软件重置 + GPU 重新初始化
# 在 VM 启动脚本中添加预重置步骤:
cat > /usr/local/bin/pre-vm-reset.sh << 'EOF'
#!/bin/bash
# VM 启动前的 GPU 预重置

GPU_BDF="0000:03:00.0"

# 1. 确保 VFIO 绑定
echo "${GPU_BDF}" > /sys/bus/pci/drivers/vfio-pci/bind 2>/dev/null

# 2. 尝试 PM 重置
echo "auto" > /sys/bus/pci/devices/${GPU_BDF}/power/control
echo 1000 > /sys/bus/pci/devices/${GPU_BDF}/power/autosuspend_delay_ms

# 3. 触发 D3cold 进入和退出
echo 0 > /sys/bus/pci/devices/${GPU_BDF}/power/control
# PCIe PM: 设置 D3hot
setpci -s ${GPU_BDF} CAP_PM+0x04.B=0x03
sleep 1
# PCIe PM: 设置 D0
setpci -s ${GPU_BDF} CAP_PM+0x04.B=0x00
sleep 1

echo "GPU 预重置完成"
EOF

chmod +x /usr/local/bin/pre-vm-reset.sh
```

#### 案例二：IOMMU 分组不完整导致的直通失败

**问题现象：**
```bash
# QEMU 启动时报错
qemu-system-x86_64: -device vfio-pci,host=03:00.0: 
  vfio 0000:03:00.0: device is not in an active DMA mapping state
# 或者
qemu-system-x86_64: -device vfio-pci,host=03:00.0: 
  vfio: failed to set iommu for container: Failed to attach device to IOMMU group
```

**诊断过程：**
```bash
# 1. 检查 IOMMU 分组
ls -la /sys/kernel/iommu_groups/*/devices/ | grep 03:00
# 发现 03:00.0 (GPU) 和 03:00.2 (USB Controller) 在同一组

# 2. 只直通了 GPU，未直通 USB 控制器
# VM 无法获取完整的 IOMMU 组

# 3. 检查 USB 控制器
lspci -vvvs 03:00.2
# 03:00.2 USB controller: Advanced Micro Devices, Inc.
# 与 GPU 在同一 PCIe Switch 下
```

**解决方案：**
```bash
# 方案 1: 将 USB 控制器一起直通
# 在 QEMU 命令中增加:
-device vfio-pci,host=03:00.2

# 方案 2: 使用 ACS 覆盖分离 IOMMU 组
# 在内核参数中添加:
pcie_acs_override=downstream,multifunction

# 重启后验证:
ls -la /sys/kernel/iommu_groups/*/devices/ | grep 03:00
# 期望输出:
# 03:00.0 在独立的 IOMMU 组
# 03:00.2 在另一个独立组

# 方案 3: 使用 NO-IOMMU 模式 (不推荐，有安全风险)
# 加载 vfio 时启用 no-iommu:
modprobe vfio enable_unsafe_noiommu_mode=1
modprobe vfio_pci

# 然后直通，但无 IOMMU 保护
```

### 相关链接

- [VFIO 内核文档](https://www.kernel.org/doc/html/latest/driver-api/vfio.html)
- [Linux IOMMU 文档](https://www.kernel.org/doc/html/latest/core-api/iommu.html)
- [PCIe ACS 覆盖补丁](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Bypassing_the_IOMMU_groups_(ACS_override_patch))
- [AMDGPU GPU 重置文档](https://docs.kernel.org/gpu/amdgpu.html#gpu-reset)
- [NUMA 架构和性能调优](https://www.kernel.org/doc/html/latest/admin-guide/numa_perf.html)

### 今日小结

KVM GPU 直通配置的核心要点：

1. **IOMMU**: 必须启用，提供 DMA 隔离保护
2. **VFIO**: 内核驱动框架，管理设备直通的生命周期
3. **IOMMU 分组**: ACS 决定分组大小，同组设备必须一起直通
4. **GPU 重置**: FLR 是主要重置方法，失败时需 vendor 特定重置
5. **NUMA 优化**: vCPU Pinning + 内存绑定，避免跨 NUMA 访问
6. **HugePages**: 1GB/2MB 大页，减少 TLB 缺失，提高性能
7. **调试工具**: dmesg、VFIO 状态、IOMMU 分组检查

### 扩展思考

1. GPU 直通后如何在 Host 和 VM 之间共享 GPU 的部分功能（如视频编解码）？
2. 如何在不重启 Host 的情况下解决 VM 异常退出后 GPU 状态卡死的问题？
3. 多 GPU 直通时如何确保每张 GPU 都有独立的 IOMMU 组？
4. 对于 GPU 驱动测试而言，直通环境有哪些独特的测试场景（如重置测试、D3hot 转换测试）？
