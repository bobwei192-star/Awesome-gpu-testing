# Day 386：QEMU + AMDGPU 测试环境搭建

## 学习目标

- 掌握 QEMU/KVM 虚拟化环境的安装和配置
- 理解 GPU 直通（Passthrough）的完整实现流程
- 学会配置和管理 VFIO（Virtual Function I/O）
- 能够在 QEMU 环境下独立搭建 AMDGPU 测试虚拟机
- 掌握 GPU 虚拟化环境的调试和故障排除方法

## 知识详解

### 概念原理

#### QEMU/KVM 架构概览

QEMU + KVM 是目前 Linux 上最主流的 GPU 虚拟化测试平台：

```
QEMU/KVM + GPU 直通架构：

┌──────────────────────────────────────────────────────┐
│                  Host (宿主机)                         │
│                                                      │
│  ┌──────────────────────────────────────────────────┐│
│  │            QEMU 用户空间（进程）                   ││
│  │                                                  ││
│  │  ┌──────────┐  ┌──────────┐  ┌───────────────┐  ││
│  │  │ vCPU 模拟 │  │ 内存模拟  │  │ 设备模拟      │  ││
│  │  └─────┬────┘  └────┬─────┘  │  ┌─────────┐ │  ││
│  │        │             │        │  │ virtio  │ │  ││
│  │        │             │        │  │ 设备    │ │  ││
│  │        │             │        │  └─────────┘ │  ││
│  │  ┌─────┴─────┐  ┌──┴─────┐  │  ┌─────────┐ │  ││
│  │  │  KVM 驱动  │  │ VFIO  │  │  │ VFIO   │ │  ││
│  │  │ (kvm.ko)   │  │ 驱动  │  │  │直通设备│ │  ││
│  │  └───────────┘  │vfio.ko││  │  └────┬────┘ │  ││
│  │                 └───┬───┘│  │       │      │  ││
│  └─────────────────────┼────┼──┴───────┼──────┘  │
│                        │    │          │          │
│  ┌──────────────────────┼────┼──────────┼──────┐  │
│  │      Linux 内核       │    │          │      │  │
│  │                      ▼    ▼          ▼      │  │
│  │               ┌──────────────────────┐      │  │
│  │               │    IOMMU 硬件       │      │  │
│  │               │  (VT-d / AMD-Vi)    │      │  │
│  │               └──────────┬───────────┘      │  │
│  └──────────────────────────┼───────────────────┘  │
│                             │                       │
└─────────────────────────────┼───────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │     AMD GPU       │
                    │   (PCIe设备)      │
                    └───────────────────┘
```

**关键组件说明：**

| 组件 | 功能 | 内核模块 |
|------|------|---------|
| KVM | 提供 CPU 和内存虚拟化 | kvm, kvm_amd |
| VFIO | 提供设备直通框架 | vfio, vfio_iommu_type1, vfio_pci |
| IOMMU | 提供 DMA 地址翻译和隔离 | amd_iommu |
| QEMU | 用户空间虚拟机管理器 | qemu-system-x86_64 |

#### GPU 直通前的准备工作

在将 AMD GPU 直通给虚拟机之前，需要完成以下准备工作：

```bash
#!/bin/bash
# 环境检查脚本: check_gpu_passthrough_prep.sh

echo "=========================================="
echo "  GPU 直通环境检查"
echo "=========================================="
echo ""

check_iommu() {
    echo "[1/5] 检查 IOMMU 支持..."
    
    # 检查内核启动参数
    if dmesg | grep -i "iommu" | grep -q "AMD-Vi\|Intel VT-d"; then
        echo "  ✅ IOMMU 已启用"
    else
        echo "  ❌ IOMMU 未启用"
        echo "  请在内核启动参数中添加:"
        echo "    AMD: iommu=pt amd_iommu=on"
        echo "    Intel: intel_iommu=on"
        return 1
    fi
    
    # 检查 IOMMU 设备分组
    if ls /sys/kernel/iommu_groups/ > /dev/null 2>&1; then
        local groups=$(ls -d /sys/kernel/iommu_groups/* | wc -l)
        echo "  IOMMU 组数量: ${groups}"
    else
        echo "  ⚠️ 无法访问 IOMMU 组信息"
    fi
}

check_vfio() {
    echo ""
    echo "[2/5] 检查 VFIO 模块..."
    
    local modules=("vfio" "vfio_iommu_type1" "vfio_pci" "vfio_virqfd")
    for mod in "${modules[@]}"; do
        if lsmod | grep -q "^${mod}"; then
            echo "  ✅ ${mod} 已加载"
        else
            echo "  ⚠️ ${mod} 未加载，尝试加载..."
            modprobe ${mod} 2>/dev/null && echo "  ✅ ${mod} 已加载" || echo "  ❌ ${mod} 加载失败"
        fi
    done
}

check_gpu_device() {
    echo ""
    echo "[3/5] 检查 GPU 设备..."
    
    local gpu_devices=$(lspci -D | grep -i "VGA\|Display\|3D" | grep "AMD\|ATI")
    if [ -z "${gpu_devices}" ]; then
        echo "  ❌ 未检测到 AMD GPU"
        return 1
    fi
    
    echo "  检测到 AMD GPU 设备:"
    echo "${gpu_devices}" | while read line; do
        echo "    ${line}"
    done
    
    # 获取 GPU BDF 并检查驱动
    local gpu_bdf=$(echo "${gpu_devices}" | head -1 | awk '{print $1}')
    local driver=$(readlink /sys/bus/pci/devices/${gpu_bdf}/driver 2>/dev/null | xargs basename 2>/dev/null)
    echo "  当前驱动: ${driver:-无}"
}

check_iommu_groups() {
    echo ""
    echo "[4/5] 检查 GPU IOMMU 分组..."
    
    local gpu_bdfs=$(lspci -D | grep -i "VGA\|Display\|3D" | grep "AMD\|ATI" | awk '{print $1}')
    
    for bdf in ${gpu_bdfs}; do
        # 查找 IOMMU 组
        for group in /sys/kernel/iommu_groups/*; do
            if [ -e "${group}/devices/${bdf}" ]; then
                local group_id=$(basename ${group})
                echo "  GPU ${bdf} → IOMMU Group ${group_id}"
                
                # 列出同一组内的所有设备
                echo "    同组设备:"
                for device in ${group}/devices/*; do
                    local dev_bdf=$(basename ${device})
                    local dev_info=$(lspci -s ${dev_bdf} 2>/dev/null | cut -d' ' -f2-)
                    echo "      ${dev_bdf} (${dev_info})"
                done
                break
            fi
        done
    done
}

check_hugepages() {
    echo ""
    echo "[5/5] 检查 HugePages 配置..."
    
    local hp_total=$(cat /proc/sys/vm/nr_hugepages 2>/dev/null || echo 0)
    local hp_free=$(cat /proc/meminfo | grep "HugePages_Free" | awk '{print $2}')
    
    if [ "${hp_total}" -gt 0 ]; then
        echo "  ✅ HugePages 已配置: ${hp_total} 页"
    else
        echo "  ⚠️ HugePages 未配置 (GPU 直通建议配置)"
        echo "  echo 2048 > /proc/sys/vm/nr_hugepages"
    fi
}

show_summary() {
    echo ""
    echo "=========================================="
    echo "  环境检查摘要"
    echo "=========================================="
    echo ""
    echo "  IOMMU:          $([ -n "$(dmesg | grep -i 'iommu' | grep 'AMD-Vi\|Intel VT-d')" ] && echo '✅' || echo '❌')"
    echo "  VFIO:           $(lsmod | grep -q vfio_pci && echo '✅' || echo '❌')"
    echo "  AMD GPU:        $(lspci | grep -qi 'VGA.*AMD\|Display.*AMD\|3D.*AMD' && echo '✅' || echo '❌')"
    echo "  HugePages:      $([ $(cat /proc/sys/vm/nr_hugepages 2>/dev/null) -gt 0 ] && echo '✅' || echo '⚠️')"
    echo ""
    echo "  如需启用 IOMMU:"
    echo "    1. 编辑 /etc/default/grub"
    echo "    2. 在 GRUB_CMDLINE_LINUX_DEFAULT 中添加:"
    echo "       AMD: amd_iommu=on iommu=pt"
    echo "       Intel: intel_iommu=on iommu=pt"
    echo "    3. update-grub && reboot"
}

check_iommu
check_vfio
check_gpu_device
check_iommu_groups
check_hugepages
show_summary
```

#### QEMU 命令行参数详解

```bash
# 完整的 QEMU GPU 直通启动命令
# 参数说明使用 <--- 标注

qemu-system-x86_64 \
    -name gpu-vm,process=gpu-vm \                    # VM 名称和进程名
    -machine type=q35,accel=kvm \                     # Q35 芯片组 + KVM 加速
    -cpu host,kvm=on,hv_relaxed,hv_spinlocks=0x1fff \ # CPU 直通，启用 KVM 半虚拟化
    -smp 8,maxcpus=8,sockets=1,cores=8,threads=1 \   # 8 个 vCPU
    -m 16G \                                          # 16GB 内存
    -mem-prealloc \                                   # 预分配内存
    -mem-path /dev/hugepages \                        # 使用 HugePages
    -drive file=/var/lib/libvirt/images/gpu-vm.qcow2,if=virtio \ # 系统盘
    -device vfio-pci,host=03:00.0,bus=rp2.0,addr=00.0 \  # GPU 直通
    -device vfio-pci,host=03:00.1,bus=rp2.0,addr=00.1 \  # GPU 音频功能
    -device virtio-net-pci,netdev=net0 \              # 虚拟网卡
    -netdev user,id=net0,hostfwd=tcp:2222-:22 \      # SSH 端口转发
    -vga none \                                       # 禁用 QEMU 显示
    -nographic \                                      # 无图形界面
    -monitor unix:/tmp/gpu-vm.sock,server,nowait \   # QEMU monitor socket
    -display none                                     # 无显示输出

# 简化的测试用命令：
qemu-system-x86_64 \
    -machine accel=kvm \
    -cpu host \
    -smp 4 \
    -m 8G \
    -drive file=test-vm.qcow2,format=qcow2 \
    -device vfio-pci,host=03:00.0 \
    -device virtio-net-pci,netdev=net0 \
    -netdev user,id=net0 \
    -nographic
```

#### VFIO 驱动绑定原理

```
VFIO 驱动绑定流程：

原本:
┌──────────┐   绑定    ┌──────────┐
│  GPU      │─────────│ amdgpu   │
│  (PCI)    │         │ (驱动)   │
└──────────┘          └──────────┘

VFIO 绑定后:
┌──────────┐   绑定    ┌──────────┐
│  GPU      │─────────│ vfio-pci │
│  (PCI)    │         │ (驱动)   │
└──────────┘          └──────────┘
     │                       │
     │ 直通给 VM             │ QEMU 通过 VFIO API
     ▼                       ▼ 访问设备
  ┌────────┐          ┌──────────────┐
  │ VM     │          │ /dev/vfio/N  │
  │ Guest  │◄────────│ (VFIO Group) │
  │ amdgpu │  DMA/MMIO└──────────────┘
  └────────┘
```

### 实践操作

#### 完整环境搭建流程

```bash
#!/bin/bash
# 文件名: setup_qemu_amdpgpu_env.sh
# 用途: 自动搭建 QEMU + AMDGPU 测试环境

set -e

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log_info()  { echo -e "${GREEN}[INFO]${NC} $1"; }
log_warn()  { echo -e "${YELLOW}[WARN]${NC} $1"; }
log_error() { echo -e "${RED}[ERROR]${NC} $1"; }

# 阶段 1: 安装必要的软件包
install_packages() {
    log_info "阶段 1: 安装软件包..."
    
    # 检测发行版
    if [ -f /etc/debian_version ]; then
        apt-get update
        apt-get install -y \
            qemu-system-x86-64 \
            qemu-utils \
            libvirt-daemon-system \
            libvirt-clients \
            virt-manager \
            ovmf \
            cpu-checker \
            bridge-utils
    elif [ -f /etc/redhat-release ]; then
        dnf install -y \
            @virt \
            qemu-kvm \
            libvirt \
            virt-install \
            virt-viewer \
            edk2-ovmf
    else
        log_error "不支持的操作系统"
        exit 1
    fi
    
    # 确保 KVM 模块加载
    kvm-ok || log_warn "系统可能不支持 KVM"
    modprobe kvm_amd 2>/dev/null || modprobe kvm_intel 2>/dev/null || log_warn "KVM 模块加载失败"
}

# 阶段 2: 配置内核参数
configure_kernel() {
    log_info "阶段 2: 配置内核参数..."
    
    local grub_file="/etc/default/grub"
    
    # 检查是否已配置
    if grep -q "iommu=pt" ${grub_file}; then
        log_info "IOMMU 已配置"
        return
    fi
    
    # 检测 CPU 类型
    if grep -q "AMD" /proc/cpuinfo; then
        IOMMU_PARAMS="amd_iommu=on iommu=pt"
    elif grep -q "Intel" /proc/cpuinfo; then
        IOMMU_PARAMS="intel_iommu=on iommu=pt"
    else
        log_error "未知 CPU"
        exit 1
    fi
    
    # 添加其他优化参数
    EXTRA_PARAMS="vfio-pci.ids=1002:7310,1002:ab38 kvm.ignore_msrs=1"
    
    # 更新 grub
    sed -i "s/GRUB_CMDLINE_LINUX_DEFAULT=\"/GRUB_CMDLINE_LINUX_DEFAULT=\"${IOMMU_PARAMS} ${EXTRA_PARAMS} /" ${grub_file}
    
    update-grub
    log_info "内核参数已更新，需要重启生效"
    log_info "  添加参数: ${IOMMU_PARAMS} ${EXTRA_PARAMS}"
}

# 阶段 3: 配置 VFIO
configure_vfio() {
    log_info "阶段 3: 配置 VFIO..."
    
    # 加载 VFIO 模块
    cat > /etc/modules-load.d/vfio.conf << 'EOF'
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
kvm
kvm_amd
EOF
    
    # 配置 VFIO 绑定 (基于 GPU ID)
    local gpu_ids=$(lspci -n | grep "1002" | grep "0300\|0380" | awk '{print $NF}' | tr ':' ' ')
    if [ -n "${gpu_ids}" ]; then
        # 提取 vendor:device ID
        local vendor=$(echo ${gpu_ids} | awk '{print $1}' | cut -d' ' -f1)
        local device=$(echo ${gpu_ids} | awk '{print $2}' | cut -d' ' -f1)
        
        echo "options vfio-pci ids=${vendor}:${device}" > /etc/modprobe.d/vfio.conf
        log_info "VFIO 绑定 GPU: ${vendor}:${device}"
    else
        log_warn "未检测到 AMD GPU PCI ID"
    fi
    
    # 阻止 amdgpu 自动加载
    echo "blacklist amdgpu" > /etc/modprobe.d/blacklist-amdgpu.conf
    
    # 更新 initramfs
    update-initramfs -u || dracut -f
    
    log_info "VFIO 配置完成"
}

# 阶段 4: 创建 VM 磁盘镜像
create_vm_disk() {
    local vm_path=$1
    local vm_size=$2
    
    log_info "阶段 4: 创建 VM 磁盘镜像..."
    log_info "  路径: ${vm_path}"
    log_info "  大小: ${vm_size}"
    
    if [ -f "${vm_path}" ]; then
        log_warn "磁盘镜像已存在，跳过创建"
        return
    fi
    
    qemu-img create -f qcow2 "${vm_path}" "${vm_size}"
    log_info "磁盘镜像已创建"
    
    # 下载安装 ISO (可选)
    local iso_path="/var/lib/libvirt/images/ubuntu-24.04-server.iso"
    if [ ! -f "${iso_path}" ]; then
        log_warn "请下载 Ubuntu Server ISO 到: ${iso_path}"
        log_warn "  wget https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso -O ${iso_path}"
    fi
}

# 阶段 5: 启动 VM
start_vm() {
    local vm_name=$1
    local vm_path=$2
    local gpu_bdf=$3
    local audio_bdf=$4
    
    log_info "阶段 5: 启动 VM..."
    
    # 确认 GPU 已绑定到 vfio-pci
    local current_driver=$(readlink /sys/bus/pci/devices/${gpu_bdf}/driver 2>/dev/null | xargs basename 2>/dev/null)
    if [ "${current_driver}" != "vfio-pci" ]; then
        # 动态解绑和重绑定
        echo "${gpu_bdf}" > /sys/bus/pci/devices/${gpu_bdf}/driver/unbind 2>/dev/null || true
        echo "vfio-pci" > /sys/bus/pci/devices/${gpu_bdf}/driver_override 2>/dev/null || true
        echo "${gpu_bdf}" > /sys/bus/pci/drivers/vfio-pci/bind 2>/dev/null || true
    fi
    
    # 配置 HugePages
    echo 2048 > /proc/sys/vm/nr_hugepages
    
    # QEMU 启动命令
    qemu-system-x86_64 \
        -name ${vm_name},process=${vm_name} \
        -machine type=q35,accel=kvm \
        -cpu host,kvm=on \
        -smp 4 \
        -m 8G \
        -mem-prealloc \
        -mem-path /dev/hugepages \
        -drive file=${vm_path},if=virtio,format=qcow2 \
        -device vfio-pci,host=${gpu_bdf} \
        $( [ -n "${audio_bdf}" ] && echo "-device vfio-pci,host=${audio_bdf}" ) \
        -device virtio-net-pci,netdev=net0 \
        -netdev user,id=net0,hostfwd=tcp:2222-:22 \
        -vga none \
        -nographic \
        -display none &
    
    local pid=$!
    echo ${pid} > /var/run/${vm_name}.pid
    
    log_info "VM 已启动 (PID: ${pid})"
    log_info "通过 SSH 访问: ssh -p 2222 user@localhost"
    log_info "通过 QEMU monitor: socat - UNIX-CONNECT:/tmp/${vm_name}.sock"
}

# 阶段 6: 验证 GPU 直通
verify_passthrough() {
    log_info "阶段 6: 验证 GPU 直通..."
    
    # 检查 VM 内是否识别到 GPU
    # 假设通过 SSH 执行
    local vm_ip="localhost"
    local vm_port=2222
    
    ssh -p ${vm_port} ${vm_ip} "lspci | grep -i 'VGA\|Display\|3D' && echo '✅ GPU 直通成功'"
    
    # 检查 amdgpu 驱动是否加载
    ssh -p ${vm_port} ${vm_ip} "lsmod | grep amdgpu && echo '✅ AMDGPU 驱动已加载' || echo '⚠️ AMDGPU 驱动未加载'"
    
    # 检查 GPU 信息
    ssh -p ${vm_port} ${vm_ip} "cat /sys/class/drm/card0/device/vendor && cat /sys/class/drm/card1/device/vendor" 2>/dev/null
    
    # 运行基本 GPU 测试
    log_info "运行 GPU 基准测试..."
    ssh -p ${vm_port} ${vm_ip} "glxinfo -B | grep 'OpenGL renderer'" 2>/dev/null || true
}

# 主流程
main() {
    local VM_NAME="gpu-test-vm"
    local VM_PATH="/var/lib/libvirt/images/${VM_NAME}.qcow2"
    local VM_SIZE="50G"
    local GPU_BDF="0000:03:00.0"
    local AUDIO_BDF="0000:03:00.1"
    
    log_info "QEMU + AMDGPU 测试环境搭建"
    log_info "================================"
    
    # 检查权限
    if [ "$(id -u)" != "0" ]; then
        log_error "需要 root 权限执行"
        exit 1
    fi
    
    install_packages
    configure_kernel
    configure_vfio
    
    log_warn "内核参数已更改，建议重启后再继续"
    log_warn "重启后执行: $0 --deploy"
}

if [ "$1" == "--deploy" ]; then
    # 部署阶段 (重启后执行)
    create_vm_disk "${VM_PATH}" "${VM_SIZE}"
    start_vm "${VM_NAME}" "${VM_PATH}" "${GPU_BDF}" "${AUDIO_BDF}"
    sleep 10
    verify_passthrough
else
    main "$@"
fi
```

#### VM 内 AMDGPU 驱动安装

```bash
#!/bin/bash
# VM 内 AMDGPU 驱动安装脚本

set -e

echo "========================================"
echo "  VM 内 AMDGPU 驱动安装"
echo "========================================"
echo ""

# 1. 检测 GPU
echo "[1/6] 检测 GPU..."
lspci | grep -i "VGA\|Display\|3D"
GPU_VENDOR=$(lspci -n | grep "0300" | awk '{print $NF}' | cut -d: -f1)
echo "GPU Vendor: ${GPU_VENDOR}"
if [ "${GPU_VENDOR}" != "1002" ]; then
    echo "❌ 不是 AMD GPU"
    exit 1
fi

# 2. 安装依赖
echo ""
echo "[2/6] 安装依赖..."
apt-get update
apt-get install -y \
    build-essential \
    dkms \
    linux-headers-$(uname -r) \
    mesa-utils \
    vulkan-tools \
    clinfo

# 3. 安装 AMDGPU 驱动（来自 ROCm 仓库）
echo ""
echo "[3/6] 添加 ROCm 仓库..."
wget -q -O - https://repo.radeon.com/rocm/rocm.gpg.key | apt-key add -
echo "deb [arch=amd64] https://repo.radeon.com/rocm/apt/6.0 ubuntu main" > /etc/apt/sources.list.d/rocm.list
echo "deb [arch=amd64] https://repo.radeon.com/amdgpu/6.0/ubuntu jammy main" > /etc/apt/sources.list.d/amdgpu.list

apt-get update

# 4. 安装 AMDGPU 驱动包
echo ""
echo "[4/6] 安装 AMDGPU 驱动..."
apt-get install -y amdgpu-dkms rocm-dev

# 5. 安装测试工具
echo ""
echo "[5/6] 安装测试工具..."
apt-get install -y \
    rocminfo \
    rocm-bandwidth-test \
    miopen-hip \
    tensorflow-rocm

# 6. 验证安装
echo ""
echo "[6/6] 验证安装..."
echo ""
echo "ROCm 信息:"
rocminfo | head -20 || echo "rocminfo 不可用"

echo ""
echo "GPU 信息:"
rocm-smi || amd-smi || echo "GPU 监控工具不可用"

echo ""
echo "OpenGL 信息:"
glxinfo -B 2>/dev/null | grep "OpenGL renderer" || echo "OpenGL 信息不可用"

echo ""
echo "Vulkan 信息:"
vulkaninfo --summary 2>/dev/null | grep "GPU id\|deviceName" || echo "Vulkan 信息不可用"

echo ""
echo "✅ AMDGPU 驱动安装完成"
```

#### 测试场景配置

```bash
# 场景 1: 基本 GPU 计算测试
# VM 内执行
cat > /tmp/gpu_test.sh << 'EOF'
#!/bin/bash
echo "GPU 计算测试"
echo "================"

# ROCm 信息
echo "ROCm 版本:"
cat /opt/rocm/.info/version 2>/dev/null || echo "未知"

# GPU 详细信息
echo ""
echo "GPU 详细信息:"
rocm-smi --showallinfo 2>/dev/null | head -50 || true

# 内存带宽测试 (如果有 rocm-bandwidth-test)
if command -v rocm-bandwidth-test &> /dev/null; then
    echo ""
    echo "内存带宽测试:"
    rocm-bandwidth-test 2>/dev/null | tail -20
fi

# 简单 HIP 测试 (如果有 HIP)
if command -v hipconfig &> /dev/null; then
    echo ""
    echo "HIP 配置:"
    hipconfig --full 2>/dev/null | head -20
fi
EOF

# 场景 2: 显示输出测试
cat > /tmp/display_test.sh << 'EOF'
#!/bin/bash
echo "显示输出测试"
echo "================"

# 检查 DRM 设备
echo "DRM 设备:"
ls -la /sys/class/drm/

# 检查 framebuffer 设备
echo ""
echo "Framebuffer 设备:"
ls -la /dev/fb* 2>/dev/null || echo "无 framebuffer 设备"

# 显示模式 (如果有 xrandr)
if command -v xrandr &> /dev/null; then
    echo ""
    echo "显示模式:"
    xrandr --listproviders 2>/dev/null || echo "xrandr 不可用"
fi

# 检查当前显示服务器
echo ""
echo "显示服务器:"
echo "  DISPLAY=${DISPLAY:-未设置}"
echo "  WAYLAND_DISPLAY=${WAYLAND_DISPLAY:-未设置}"
EOF

# 场景 3: 虚拟化特性测试
cat > /tmp/virt_test.sh << 'EOF'
#!/bin/bash
echo "虚拟化特性测试"
echo "================"

# 检测是否在 VM 内
echo "虚拟化检测:"
if [ -f /sys/hypervisor/type ]; then
    echo "  运行环境: 虚拟机"
    cat /sys/hypervisor/type
else
    echo "  运行环境: 物理机"
fi

# 检测 SR-IOV 支持
echo ""
echo "SR-IOV 检查:"
if [ -f /sys/class/drm/card0/device/sriov_totalvfs ]; then
    echo "  TotalVFs: $(cat /sys/class/drm/card0/device/sriov_totalvfs)"
    echo "  NumVFs:   $(cat /sys/class/drm/card0/device/sriov_numvfs)"
fi

# GPU 虚拟化能力
echo ""
echo "GPU 虚拟化能力:"
cat /sys/class/drm/card0/device/ats 2>/dev/null && echo "  ATS: 支持" || echo "  ATS: 不支持"
cat /sys/class/drm/card0/device/acs 2>/dev/null && echo "  ACS: 支持" || echo "  ACS: 不支持"

# IOMMU 信息
echo ""
echo "IOMMU 信息:"
cat /sys/kernel/iommu_groups/*/devices/*/vendor 2>/dev/null | head -5 || echo "  IOMMU 信息不可用"
EOF
```

#### 调试与故障排除

```bash
#!/bin/bash
# GPU 直通调试脚本

echo "GPU 直通调试"
echo "================================"
echo ""

check_dmesg_errors() {
    echo "1. dmesg 错误检查:"
    dmesg | grep -i "vfio\|iommu\|amdgpgu\|pci" | grep -i "error\|fail\|bug\|warn" | tail -20
    echo ""
}

check_iommu_group() {
    echo "2. IOMMU 分组:"
    for group in /sys/kernel/iommu_groups/*; do
        group_id=$(basename ${group})
        devices=""
        for device in ${group}/devices/*; do
            dev_info=$(lspci -n -s $(basename ${device}) 2>/dev/null)
            if echo "${dev_info}" | grep -q "1002"; then
                devices="${devices} ⚡⚠️GPU⚠️⚠️"
            fi
            devices="${devices} $(basename ${device})"
        done
        if echo "${devices}" | grep -qi "gpu"; then
            echo "  Group ${group_id}:${devices}"
        fi
    done
    echo ""
}

check_device_driver() {
    echo "3. GPU 设备驱动状态:"
    for dev in /sys/bus/pci/devices/*/vendor; do
        vendor=$(cat ${dev} 2>/dev/null)
        if [ "${vendor}" == "0x1002" ]; then
            bdf=$(echo ${dev} | awk -F/ '{print $(NF-1)}')
            driver=$(readlink ${dev%/*}/driver 2>/dev/null | xargs basename 2>/dev/null || echo "无")
            
            # 检查设备类
            class=$(cat ${dev%/*}/class 2>/dev/null)
            case "${class}" in
                0x030000) type="VGA GPU";;
                0x030200) type="3D GPU";;
                0x038000) type="Display GPU";;
                0x040300) type="Audio (GPU)";;
                *) type="其他 (${class})";;
            esac
            
            # 检查是否被 VFIO 占用
            vfio_owner=$(cat ${dev%/*}/iommu_group/... 2>/dev/null | head -1 || echo "")
            
            echo "  ${bdf} [${type}]"
            echo "    Driver: ${driver} $( [ "${driver}" == "vfio-pci" ] && echo '✅直通' || echo '')"
            echo "    Vendor:Device = $(cat ${dev%/*}/vendor 2>/dev/null):$(cat ${dev%/*}/device 2>/dev/null)"
        fi
    done
    echo ""
}

check_vfio_status() {
    echo "4. VFIO 状态:"
    # VFIO IOMMU 类型
    if [ -d /dev/vfio ]; then
        echo "  VFIO 设备:"
        ls -la /dev/vfio/ | tail -5
    fi
    
    # VFIO 进程占用
    echo "  VFIO 进程:"
    lsof 2>/dev/null | grep /dev/vfio || echo "    无"
    echo ""
}

check_vm_status() {
    echo "5. VM 状态:"
    virsh list --all 2>/dev/null || echo "  libvirtd 未运行或不可用"
    
    # 检查 QEMU 进程
    pgrep -a qemu 2>/dev/null || echo "  QEMU 进程: 无"
    echo ""
}

provide_solution() {
    echo "6. 常见问题与解决方案:"
    echo "  ❌ GPU 直通后 VM 无法启动:"
    echo "     → 检查 GPU 的 IOMMU 分组是否完整（包含 audio 功能）"
    echo "     → 检查 vfio-pci 是否正确绑定"
    echo "     → 尝试在 QEMU 命令行添加 -nodefaults"
    echo ""
    echo "  ❌ VM 内无法加载 amdgpu:"
    echo "     → 确认内核版本支持"  
    echo "     → 尝试安装较新内核"
    echo "     → 检查 VM 内 GPU 的 PCI ID"
    echo ""
    echo "  ❌ GPU 性能低于预期:"
    echo "     → 检查 IOMMU 是否启用 pt 模式"
    echo "     → 检查 HugePages 配置"
    echo "     → 检查 CPU pinning 配置"
    echo ""
    echo "  ❌ GPU 直通后 Host 显示异常:"
    echo "     → 使用独立 GPU 作为 Host 显示"
    echo "     → 配置 GPU 黑名单 (modprobe.d)"
}

check_dmesg_errors
check_iommu_group
check_device_driver
check_vfio_status
check_vm_status
provide_solution
```

#### libvirt/virt-manager 配置

```xml
<!-- libvirt VM 配置示例: gpu-vm.xml -->
<!-- 使用 virsh define gpu-vm.xml 导入 -->
<domain type='kvm'>
  <name>gpu-vm</name>
  <memory unit='GiB'>16</memory>
  <currentMemory unit='GiB'>16</currentMemory>
  <vcpu placement='static'>8</vcpu>
  
  <os>
    <type arch='x86_64' machine='q35'>hvm</type>
    <loader readonly='yes' type='pflash'>/usr/share/OVMF/OVMF_CODE.fd</loader>
    <nvram>/var/lib/libvirt/qemu/nvram/gpu-vm_VARS.fd</nvram>
  </os>
  
  <features>
    <acpi/>
    <apic/>
    <hyperv>
      <relaxed state='on'/>
      <spinlocks state='on' retries='8191'/>
    </hyperv>
    <kvm>
      <hidden state='on'/>   <!-- 隐藏 VM 身份，GPU 驱动需要 -->
    </kvm>
  </features>
  
  <cpu mode='host-passthrough' check='none'>
    <topology sockets='1' dies='1' cores='8' threads='1'/>
  </cpu>
  
  <devices>
    <!-- GPU 直通配置 -->
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x03' slot='0x00' function='0x0'/>
      </source>
    </hostdev>
    
    <!-- GPU 音频功能 (必须同一 IOMMU 组) -->
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x03' slot='0x00' function='0x1'/>
      </source>
    </hostdev>
    
    <!-- 磁盘 -->
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/gpu-vm.qcow2'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    
    <!-- 网络 -->
    <interface type='network'>
      <mac address='52:54:00:ab:cd:ef'/>
      <source network='default'/>
      <model type='virtio'/>
    </interface>
    
    <!-- 内存 balloon -->
    <memballoon model='virtio'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </memballoon>
  </devices>
</domain>
```

### 案例分析

#### 案例一：GPU 直通后 VM 内无法识别 GPU

**问题现象：**
```bash
# VM 内 lspci 无 GPU 输出
# dmesg 显示:
[   12.345] pci 0000:00:05.0: [Firmware Bug]: Invalid BAR (can't resize)
[   12.345] pci 0000:00:05.0: BAR 2: can't reserve [mem 0x00000000-0xbfffffff 64bit pref]
```

**诊断过程：**
```bash
# 1. Host 端检查 GPU 状态
lspci -vvs 03:00.0 | grep -i "region\|bar"
# 发现 BAR2 大小为 3GB（超过 QEMU 默认的 2GB 限制）

# 2. 检查 QEMU 内存映射
cat /proc/$(pgrep qemu)/maps | grep vfio
# 发现只有 2GB MMIO 区域被映射
```

**根本原因：** GPU 的 BAR2 需要 3GB MMIO 空间，但 QEMU 默认配置不支持大 BAR。

**解决方案：**
```bash
# 在 QEMU 命令行添加大 BAR 支持
# 添加以下参数:
-global q35-pcihost.x-config-regs=on

# 或者在 libvirt 配置中添加:
<qemu:commandline>
  <qemu:arg value='-global'/>
  <qemu:arg value='q35-pcihost.x-config-regs=on'/>
</qemu:commandline>

# 使用更新后的 QEMU 版本 (>=6.2.0)
qemu-system-x86_64 --version
# QEMU emulator version 7.2.0 (qemu-7.2.0)
```

#### 案例二：VM 内 amdgpu 加载时 GPU 重置失败

**问题现象：**
```bash
# VM 内 dmesg
[   12.345] amdgpu 0000:00:05.0: amdgpu: Resizable BAR failed
[   12.345] amdgpu 0000:00:05.0: amdgpu: Failed to initialize GPU, aborting
[   12.345] amdgpu: probe of 0000:00:05.0 failed with error -12
```

**诊断：**
```bash
# 1. 检查 VM 分配的 MMIO 空间
lspci -vvs 00:05.0 | grep "Region\|Memory"
# 发现只有 256MB MMIO 区域

# 2. 检查 QEMU 启动参数中 GPU 的 MMIO 配置
# 缺少 pci-express 配置

# 3. 检查 VM 内 IOMMU 状态
dmesg | grep -i iommu
# VM 内 IOMMU 未启用
```

**解决方案：**
```bash
# 1. 在 VM 内核启动参数中启用 IOMMU
# VM 内编辑 /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="amd_iommu=on iommu=pt"

# 2. 配置 QEMU 的 pci-root 使用 pcie-root
# 修改 QEMU 启动参数:
-machine type=q35,accel=kvm \
-global q35-pcihost.x-config-regs=on \
-global ioh3420.revision=2 \
-device pcie-root-port,slot=0,chassis=1,id=rp2.0 \
-device vfio-pci,host=03:00.0,bus=rp2.0 \
-device vfio-pci,host=03:00.1,bus=rp2.0 \
-global ioh3420.x-pcie-extcap-init=on

# 3. libvirt 中配置 PCIe root 端口
# 在 devices 部分添加:
<controller type='pci' index='0' model='pcie-root'/>
<controller type='pci' index='1' model='pcie-root-port'>
  <model name='pcie-root-port'/>
  <target chassis='1' port='0x10'/>
  <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0' multifunction='on'/>
</controller>
```

### 相关链接

- [QEMU GPU 直通文档](https://wiki.qemu.org/Features/GPU_Passthrough)
- [VFIO - "Virtual Function I/O" - Linux 内核文档](https://www.kernel.org/doc/html/latest/driver-api/vfio.html)
- [Arch Linux GPU Passthrough 指南](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)
- [AMD ROCm 安装指南](https://rocm.docs.amd.com/en/latest/deploy/linux/install.html)
- [libvirt 域 XML 格式](https://libvirt.org/formatdomain.html)

### 今日小结

QEMU + AMDGPU 测试环境的搭建是 GPU 虚拟化测试的基础技能：

1. **前置条件**：IOMMU 启用、VFIO 模块加载、内核参数配置
2. **GPU 直通**：VFIO 绑定、IOMMU 分组确认、PCIe 配置
3. **VM 配置**：QEMU 命令行参数、libvirt XML 配置、PCIe root 端口
4. **驱动安装**：VM 内 AMDGPU 驱动、ROCm 工具链、验证测试
5. **调试工具**：dmesg 分析、IOMMU 分组检查、设备驱动状态
6. **常见问题**：大 BAR 支持、PCIe 配置错误、IOMMU 分组不完整

### 扩展思考

1. 如何在不重启 Host 的情况下动态绑定/解绑 GPU 设备给 VM？
2. 多 GPU 直通时如何确保每个 GPU 的 IOMMU 分组独立？
3. GPU 直通后 Host 如何管理和监控 VM 内的 GPU 状态？
4. 在 CI/CD 环境中如何自动化 GPU 直通测试环境的搭建和销毁？
