# Fuzz 测试基础：syzkaller 在 DRM 驱动中的应用

## 学习目标

- 理解 Fuzz 测试的原理和在 GPU 驱动安全测试中的价值
- 掌握 syzkaller 工具的安装、配置和运行方法
- 学会编写和定制 syzkaller 的测试描述（syscall descriptions）
- 了解 Fuzz 测试结果的分析和漏洞分类方法
- 能够将 Fuzz 测试集成到 GPU 驱动的安全测试流程中

## 知识详解

### 一、概念原理

#### 1.1 Fuzz 测试原理

Fuzz 测试（Fuzzing）是一种自动化软件测试技术，通过向程序输入大量随机或半随机数据来发现崩溃、内存泄露和其他安全漏洞：

```
┌─────────────────────────────────────────────────────────────────────┐
│              Fuzz 测试原理与 GPU 驱动应用                            │
│                                                                     │
│  传统测试                Fuzz 测试                                   │
│  ─────────────           ─────────────                             │
│                                                                     │
│  ● 基于规格编写用例      ● 自动生成输入                              │
│  ● 验证预期行为          ● 发现未预期行为                            │
│  ● 覆盖已知场景          ● 探索未知场景                              │
│  ● 人工设计边界          ● 自动发现边界                              │
│                                                                     │
│  Fuzz 测试在 GPU 驱动中的价值：                                     │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  1. 内核驱动权限高，漏洞影响大（权限提升、信息泄露）      │       │
│  │  2. ioctl 接口复杂，手动测试难以覆盖所有参数组合          │       │
│  │  3. 用户空间可直接访问内核，是主要攻击面                  │       │
│  │  4. GPU 驱动代码量大，人工审查难以覆盖所有路径            │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
│  Fuzz 测试类型：                                                     │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  类型          原理              适用场景                 │       │
│  │  ─────────────────────────────────────────────────────   │       │
│  │  盲 fuzz       完全随机输入      快速发现明显崩溃        │       │
│  │  基于生成      按语法生成        结构化输入（如 ioctl）  │       │
│  │  基于变异      变异合法输入      已有样本基础上探索      │       │
│  │  覆盖引导      优先覆盖新代码    深入发现复杂漏洞        │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.2 syzkaller 架构

syzkaller 是 Google 开发的开源内核 Fuzz 测试工具，被广泛用于 Linux 内核安全测试：

```
┌─────────────────────────────────────────────────────────────────────┐
│              syzkaller 架构                                          │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────┐      │
│   │                    syz-manager (主控)                    │      │
│   │                                                         │      │
│   │  ● 管理多个 VM 实例                                      │      │
│   │  ● 分发测试程序                                          │      │
│   │  ● 收集崩溃信息                                          │      │
│   │  ● 维护语料库（corpus）                                  │      │
│   │  ● 生成覆盖率报告                                        │      │
│   └─────────────────────────┬───────────────────────────────┘      │
│                             │                                       │
│              ┌──────────────┼──────────────┐                     │
│              │              │              │                     │
│              ▼              ▼              ▼                     │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│   │  VM 实例 1   │  │  VM 实例 2   │  │  VM 实例 N   │             │
│   │             │  │             │  │             │             │
│   │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │             │
│   │ │syz-fuzzer│ │  │ │syz-fuzzer│ │  │ │syz-fuzzer│ │             │
│   │ │         │ │  │ │         │ │  │ │         │ │             │
│   │ │● 生成   │ │  │ │● 生成   │ │  │ │● 生成   │ │             │
│   │ │● 执行   │ │  │ │● 执行   │ │  │ │● 执行   │ │             │
│   │ │● 覆盖   │ │  │ │● 覆盖   │ │  │ │● 覆盖   │ │             │
│   │ │● 反馈   │ │  │ │● 反馈   │ │  │ │● 反馈   │ │             │
│   │ └────┬────┘ │  │ └────┬────┘ │  │ └────┬────┘ │             │
│   │      │      │  │      │      │  │      │      │             │
│   │      ▼      │  │      ▼      │  │      ▼      │             │
│   │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │             │
│   │ │ 内核    │ │  │ │ 内核    │ │  │ │ 内核    │ │             │
│   │ │ ● DRM   │ │  │ │ ● DRM   │ │  │ │ ● DRM   │ │             │
│   │ │ ● AMDGPU│ │  │ │ ● AMDGPU│ │  │ │ ● AMDGPU│ │             │
│   │ │ ● KFD   │ │  │ │ ● KFD   │ │  │ │ ● KFD   │ │             │
│   │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │             │
│   └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                     │
│   工作流：                                                           │
│   1. syz-manager 启动多个 VM                                        │
│   2. 每个 VM 运行 syz-fuzzer                                        │
│   3. syz-fuzzer 生成随机 syscall 序列                               │
│   4. 执行 syscall 并收集覆盖率                                      │
│   5. 覆盖新代码的输入加入语料库                                     │
│   6. 发现崩溃时报告给 syz-manager                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**参考来源**：syzkaller 官方文档 (https://github.com/google/syzkaller/blob/master/docs/linux/setup.md)

### 二、实践操作

#### 2.1 syzkaller 安装与配置

```bash
#!/bin/bash
# install_syzkaller.sh - syzkaller 安装脚本
# 参考：syzkaller 官方安装指南

set -e

SYZ_DIR="${1:-$HOME/syzkaller}"
GO_VERSION="1.21.0"

echo "=== syzkaller 安装 ==="
echo "安装目录: $SYZ_DIR"
echo ""

# 1. 安装依赖
echo "【步骤 1】安装依赖"
sudo apt-get update
sudo apt-get install -y \
    git \
    build-essential \
    qemu-system-x86 \
    debootstrap \
    gcc \
    g++ \
    make \
    libncurses5-dev

# 2. 安装 Go
echo "【步骤 2】安装 Go"
if ! command -v go &> /dev/null || [ "$(go version | awk '{print $3}')" != "go$GO_VERSION" ]; then
    wget "https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz"
    sudo rm -rf /usr/local/go
    sudo tar -C /usr/local -xzf "go${GO_VERSION}.linux-amd64.tar.gz"
    rm "go${GO_VERSION}.linux-amd64.tar.gz"
    
    export PATH=$PATH:/usr/local/go/bin
    echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
fi

echo "Go 版本: $(go version)"

# 3. 下载 syzkaller
echo "【步骤 3】下载 syzkaller"
mkdir -p "$SYZ_DIR"
cd "$SYZ_DIR"

go env -w GOPROXY=https://proxy.golang.org,direct
go install github.com/google/syzkaller/syz-manager@latest

echo "syzkaller 安装完成"
echo ""

# 4. 验证安装
echo "【步骤 4】验证安装"
which syz-manager || echo "syz-manager 在 $HOME/go/bin/syz-manager"
```

#### 2.2 DRM 驱动 Fuzz 配置

```bash
#!/bin/bash
# setup_drm_fuzzing.sh - DRM 驱动 Fuzz 测试配置

set -e

WORK_DIR="${1:-$HOME/drm-fuzzing}"
KERNEL_SRC="${2:-$HOME/linux}"
mkdir -p "$WORK_DIR"

echo "=== DRM 驱动 Fuzz 测试配置 ==="
echo ""

# 1. 配置内核（启用 KCOV 和必要的调试选项）
echo "【步骤 1】配置内核"
cd "$KERNEL_SRC"

cat >> .config << 'EOF'
# KCOV（内核覆盖率收集）
CONFIG_KCOV=y
CONFIG_KCOV_INSTRUMENT_ALL=y
CONFIG_KCOV_IRQ_AREA_SIZE=0x40000

# 调试选项
CONFIG_DEBUG_FS=y
CONFIG_DEBUG_KERNEL=y
CONFIG_DEBUG_INFO=y
CONFIG_DEBUG_INFO_DWARF5=y

# 内存检测
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y

# DRM 驱动
CONFIG_DRM=y
CONFIG_DRM_AMDGPU=y
CONFIG_DRM_AMDGPU_SI=y
CONFIG_DRM_AMDGPU_CIK=y

# 网络（syzkaller 需要）
CONFIG_NET=y
CONFIG_INET=y
EOF

make olddefconfig
make -j$(nproc)
make modules_install INSTALL_MOD_PATH="$WORK_DIR/modules"

echo "内核编译完成"
echo ""

# 2. 创建 syzkaller 配置文件
echo "【步骤 2】创建 syzkaller 配置"
cat > "$WORK_DIR/drm.cfg" << EOF
{
    "target": "linux/amd64",
    "http": ":56741",
    "workdir": "$WORK_DIR/workdir",
    "kernel_obj": "$KERNEL_SRC",
    "image": "$WORK_DIR/stretch.img",
    "sshkey": "$WORK_DIR/stretch.id_rsa",
    "syzkaller": "$HOME/go/pkg/mod/github.com/google/syzkaller@*/",
    "procs": 4,
    "type": "qemu",
    "vm": {
        "count": 4,
        "kernel": "$KERNEL_SRC/arch/x86/boot/bzImage",
        "cpu": 2,
        "mem": 2048
    },
    "enable_syscalls": [
        "openat$drm",
        "ioctl$DRM_IOCTL_VERSION",
        "ioctl$DRM_IOCTL_GET_UNIQUE",
        "ioctl$DRM_IOCTL_SET_UNIQUE",
        "ioctl$DRM_IOCTL_GET_MAGIC",
        "ioctl$DRM_IOCTL_AUTH_MAGIC",
        "ioctl$DRM_IOCTL_ADD_MAP",
        "ioctl$DRM_IOCTL_RM_MAP",
        "ioctl$DRM_IOCTL_ADD_BUFS",
        "ioctl$DRM_IOCTL_MARK_BUFS",
        "ioctl$DRM_IOCTL_INFO_BUFS",
        "ioctl$DRM_IOCTL_MAP_BUFS",
        "ioctl$DRM_IOCTL_FREE_BUFS",
        "ioctl$DRM_IOCTL_ADD_CTX",
        "ioctl$DRM_IOCTL_RM_CTX",
        "ioctl$DRM_IOCTL_MOD_CTX",
        "ioctl$DRM_IOCTL_GET_CTX",
        "ioctl$DRM_IOCTL_SWITCH_CTX",
        "ioctl$DRM_IOCTL_NEW_CTX",
        "ioctl$DRM_IOCTL_RES_CTX",
        "ioctl$DRM_IOCTL_LOCK",
        "ioctl$DRM_IOCTL_UNLOCK",
        "ioctl$DRM_IOCTL_CONTROL",
        "ioctl$DRM_IOCTL_AGP_ACQUIRE",
        "ioctl$DRM_IOCTL_AGP_RELEASE",
        "ioctl$DRM_IOCTL_AGP_ENABLE",
        "ioctl$DRM_IOCTL_AGP_ALLOC",
        "ioctl$DRM_IOCTL_AGP_FREE",
        "ioctl$DRM_IOCTL_AGP_BIND",
        "ioctl$DRM_IOCTL_AGP_UNBIND",
        "ioctl$DRM_IOCTL_SG_ALLOC",
        "ioctl$DRM_IOCTL_SG_FREE",
        "ioctl$DRM_IOCTL_WAIT_VBLANK",
        "ioctl$DRM_IOCTL_MODESET_CTL",
        "ioctl$DRM_IOCTL_GEM_CLOSE",
        "ioctl$DRM_IOCTL_GEM_FLINK",
        "ioctl$DRM_IOCTL_GEM_OPEN",
        "ioctl$DRM_IOCTL_GET_CAP",
        "ioctl$DRM_IOCTL_SET_CLIENT_CAP",
        "ioctl$DRM_IOCTL_PRIME_HANDLE_TO_FD",
        "ioctl$DRM_IOCTL_PRIME_FD_TO_HANDLE",
        "ioctl$DRM_IOCTL_MODE_GETRESOURCES",
        "ioctl$DRM_IOCTL_MODE_GETCRTC",
        "ioctl$DRM_IOCTL_MODE_SETCRTC",
        "ioctl$DRM_IOCTL_MODE_CURSOR",
        "ioctl$DRM_IOCTL_MODE_GETGAMMA",
        "ioctl$DRM_IOCTL_MODE_SETGAMMA",
        "ioctl$DRM_IOCTL_MODE_GETENCODER",
        "ioctl$DRM_IOCTL_MODE_GETCONNECTOR",
        "ioctl$DRM_IOCTL_MODE_ATTACHMODE",
        "ioctl$DRM_IOCTL_MODE_DETACHMODE",
        "ioctl$DRM_IOCTL_MODE_GETPROPERTY",
        "ioctl$DRM_IOCTL_MODE_SETPROPERTY",
        "ioctl$DRM_IOCTL_MODE_GETPROPBLOB",
        "ioctl$DRM_IOCTL_MODE_GETFB",
        "ioctl$DRM_IOCTL_MODE_ADDFB",
        "ioctl$DRM_IOCTL_MODE_RMFB",
        "ioctl$DRM_IOCTL_MODE_PAGE_FLIP",
        "ioctl$DRM_IOCTL_MODE_DIRTYFB",
        "ioctl$DRM_IOCTL_MODE_CREATE_DUMB",
        "ioctl$DRM_IOCTL_MODE_MAP_DUMB",
        "ioctl$DRM_IOCTL_MODE_DESTROY_DUMB",
        "ioctl$DRM_IOCTL_MODE_OBJ_GETPROPERTIES",
        "ioctl$DRM_IOCTL_MODE_OBJ_SETPROPERTY",
        "ioctl$DRM_IOCTL_MODE_CURSOR2",
        "ioctl$DRM_IOCTL_MODE_ATOMIC",
        "ioctl$DRM_IOCTL_MODE_CREATEPROPBLOB",
        "ioctl$DRM_IOCTL_MODE_DESTROYPROPBLOB",
        "ioctl$DRM_IOCTL_AMDGPU_INFO",
        "ioctl$DRM_IOCTL_AMDGPU_GEM_CREATE",
        "ioctl$DRM_IOCTL_AMDGPU_CTX",
        "ioctl$DRM_IOCTL_AMDGPU_BO_LIST",
        "ioctl$DRM_IOCTL_AMDGPU_CS",
        "ioctl$DRM_IOCTL_AMDGPU_GEM_METADATA",
        "ioctl$DRM_IOCTL_AMDGPU_GEM_MMAP",
        "ioctl$DRM_IOCTL_AMDGPU_GEM_WAIT_IDLE",
        "ioctl$DRM_IOCTL_AMDGPU_GEM_VA",
        "ioctl$DRM_IOCTL_AMDGPU_WAIT_CS",
        "ioctl$DRM_IOCTL_AMDGPU_GEM_OP",
        "ioctl$DRM_IOCTL_AMDGPU_GEM_USERPTR"
    ]
}
EOF

echo "配置文件已创建: $WORK_DIR/drm.cfg"
echo ""

# 3. 创建 QEMU 镜像（简化示例）
echo "【步骤 3】准备 QEMU 镜像"
echo "注意：请手动创建或下载 QEMU 镜像"
echo "参考: https://github.com/google/syzkaller/blob/master/docs/linux/setup_linux-host_qemu-vm.md"
```

#### 2.3 运行 Fuzz 测试

```bash
#!/bin/bash
# run_drm_fuzzing.sh - 运行 DRM Fuzz 测试

set -e

WORK_DIR="${1:-$HOME/drm-fuzzing}"

echo "=== 运行 DRM Fuzz 测试 ==="
echo ""

# 检查配置
if [ ! -f "$WORK_DIR/drm.cfg" ]; then
    echo "错误: 配置文件不存在"
    exit 1
fi

# 启动 syz-manager
echo "启动 syz-manager..."
syz-manager -config="$WORK_DIR/drm.cfg" 2>&1 | tee "$WORK_DIR/fuzzing.log" &
MANAGER_PID=$!

echo "syz-manager PID: $MANAGER_PID"
echo ""

# 监控运行
echo "Fuzz 测试运行中..."
echo "Web 界面: http://localhost:56741"
echo ""
echo "按 Ctrl+C 停止"

# 等待用户中断
trap "echo '停止 syz-manager...'; kill $MANAGER_PID 2>/dev/null; exit" INT
wait $MANAGER_PID
```

#### 2.4 崩溃分析

```bash
#!/bin/bash
# analyze_crashes.sh - 分析 syzkaller 发现的崩溃

set -e

WORK_DIR="${1:-$HOME/drm-fuzzing}"
CRASH_DIR="$WORK_DIR/workdir/crashes"

echo "=== 崩溃分析 ==="
echo ""

if [ ! -d "$CRASH_DIR" ]; then
    echo "未找到崩溃目录"
    exit 0
fi

# 统计崩溃
echo "崩溃统计:"
ls -la "$CRASH_DIR" 2>/dev/null || echo "无崩溃"
echo ""

# 分析每个崩溃
for crash in "$CRASH_DIR"/*; do
    if [ -f "$crash" ]; then
        echo "--- 崩溃: $(basename $crash) ---"
        
        # 提取崩溃类型
        if grep -q "KASAN" "$crash" 2>/dev/null; then
            echo "类型: KASAN 内存错误"
        elif grep -q "BUG:" "$crash" 2>/dev/null; then
            echo "类型: 内核 BUG"
        elif grep -q "WARNING:" "$crash" 2>/dev/null; then
            echo "类型: 内核 WARNING"
        elif grep -q "general protection fault" "$crash" 2>/dev/null; then
            echo "类型: 通用保护错误"
        else
            echo "类型: 未知"
        fi
        
        # 提取调用栈
        echo "调用栈:"
        grep -A 20 "Call Trace:" "$crash" 2>/dev/null | head -20 || true
        
        echo ""
    fi
done
```

### 三、案例分析

#### 3.1 真实案例：通过 syzkaller 发现 AMDGPU ioctl 漏洞

**背景**：syzkaller 在 fuzz AMDGPU ioctl 接口时发现了一个空指针解引用漏洞。

**崩溃日志**：

```
BUG: kernel NULL pointer dereference, address: 0000000000000000
#PF: supervisor read access in kernel mode
#PF: error_code(0x0000) - not-present page
PGD 0 P4D 0
Oops: 0000 [#1] PREEMPT SMP KASAN
CPU: 2 PID: 1234 Comm: syz-executor Not tainted 6.8.0
RIP: 0010:amdgpu_cs_ioctl+0x234/0x890 [amdgpu]
Call Trace:
 <TASK>
 drm_ioctl_kernel+0x15a/0x1a0 [drm]
 drm_ioctl+0x2e8/0x540 [drm]
 __x64_sys_ioctl+0xd4/0x100
 do_syscall_64+0x3b/0x90
 entry_SYSCALL_64_after_hwframe+0x46/0x4e
```

**根因分析**：
- `amdgpu_cs_ioctl` 在处理特定参数组合时未验证指针有效性
- syzkaller 生成了包含无效 GEM 对象句柄的 ioctl 调用序列
- 驱动代码直接使用了未初始化的指针

**修复**：
```c
// 修复前
struct amdgpu_bo *bo = amdgpu_gem_object_lookup(filp, handle);
// 直接使用 bo，未检查 NULL

// 修复后
struct amdgpu_bo *bo = amdgpu_gem_object_lookup(filp, handle);
if (!bo) {
    DRM_ERROR("Invalid GEM handle: %u\n", handle);
    return -EINVAL;
}
```

#### 3.2 AMDGPU 专用 syscall 描述

```
# amdgpu.txt - AMDGPU 专用 syscall 描述
# 参考：syzkaller syscall 描述语法

# DRM 设备打开
openat$drm(fd const[AT_FDCWD], file ptr[in, string["/dev/dri/card0"]], flags flags[open_flags], mode const[0]) fd_drm

# AMDGPU info ioctl
ioctl$DRM_IOCTL_AMDGPU_INFO(fd fd_drm, cmd const[DRM_IOCTL_AMDGPU_INFO], arg ptr[in, amdgpu_info_args])

amdgpu_info_args {
    return_pointer	ptr64[out, array[int8]]
    query		int32[0:AMDGPU_INFO_MAX]
    pad		int32
    union		amdgpu_info_union
}

amdgpu_info_union [
    mode		amdgpu_mode_info
    vbios		amdgpu_vbios_info
    sensor		amdgpu_sensor_info
    fw			amdgpu_fw_info
    num_handles	int32
    mem			amdgpu_mem_info
    vce_clock_table	amdgpu_vce_clock_table
    vbios_size	int32
    vbios_image	array[int8, 65536]
    num_vram_cpu_page_faults	int64
    vram_lost_counter	int32
]

# AMDGPU GEM 创建 ioctl
ioctl$DRM_IOCTL_AMDGPU_GEM_CREATE(fd fd_drm, cmd const[DRM_IOCTL_AMDGPU_GEM_CREATE], arg ptr[inout, amdgpu_gem_create_inout])

amdgpu_gem_create_inout {
    in	amdgpu_gem_create_in
    out	amdgpu_gem_create_out
}

amdgpu_gem_create_in {
    bo_size		int64
    alignment	int64
    domains		flags[gem_domains, int32]
    domain_flags	flags[gem_domain_flags, int64]
    alloc_flags	flags[gem_alloc_flags, int64]
    preferred_heap	int32
    pad		int32
}

amdgpu_gem_create_out {
    handle	int32
    pad		int32
}

gem_domains = AMDGPU_GEM_DOMAIN_CPU, AMDGPU_GEM_DOMAIN_GTT, AMDGPU_GEM_DOMAIN_VRAM, AMDGPU_GEM_DOMAIN_GDS, AMDGPU_GEM_DOMAIN_GWS, AMDGPU_GEM_DOMAIN_OA
gem_domain_flags = AMDGPU_GEM_CREATE_CPU_ACCESS_REQUIRED, AMDGPU_GEM_CREATE_CPU_GTT_USWC, AMDGPU_GEM_CREATE_VRAM_CONTIGUOUS, AMDGPU_GEM_CREATE_VM_ALWAYS_VALID
gem_alloc_flags = AMDGPU_GEM_CREATE_VRAM_CLEARED, AMDGPU_GEM_CREATE_SHADOW, AMDGPU_GEM_CREATE_CPU_GTT_USWC, AMDGPU_GEM_CREATE_ENCRYPTED
```

### 四、相关链接

- syzkaller 官方文档：https://github.com/google/syzkaller/tree/master/docs
- syzkaller Linux 设置指南：https://github.com/google/syzkaller/blob/master/docs/linux/setup.md
- syzkaller syscall 描述语法：https://github.com/google/syzkaller/blob/master/docs/syscall_descriptions.md
- 内核 KCOV 文档：https://docs.kernel.org/dev-tools/kcov.html
- KASAN 文档：https://docs.kernel.org/dev-tools/kasan.html
- DRM ioctl 定义：include/uapi/drm/amdgpu_drm.h

## 今日小结

- Fuzz 测试通过自动生成输入发现 GPU 驱动中的安全漏洞
- syzkaller 是 Linux 内核 Fuzz 测试的主流工具，采用覆盖引导的 fuzz 策略
- DRM 驱动 ioctl 接口是主要 fuzz 目标，需要编写专用的 syscall 描述
- KCOV 和 KASAN 是内核 Fuzz 测试必备的内核配置
- 发现的崩溃需要分类分析，区分内存错误、逻辑错误和竞争条件

## 扩展思考

1. syzkaller 发现的崩溃中有大量是"不可复现"的（由于竞争条件或特定状态），如何设计更可靠的复现机制？
2. GPU 驱动的硬件依赖性使得某些漏洞只能在特定 GPU 上触发，如何扩展 syzkaller 支持多 GPU 硬件测试？
