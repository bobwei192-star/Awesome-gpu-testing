# syzkaller 配置与运行实践

## 学习目标

- 掌握 syzkaller 的完整配置流程，包括内核编译、虚拟机镜像创建和管理器配置
- 学会优化 syzkaller 的运行参数以提高 fuzz 效率
- 了解 syzkaller 的 Web 界面和日志分析方法
- 掌握崩溃复现和报告生成的方法
- 能够针对 AMDGPU 驱动定制 syzkaller 的测试策略

## 知识详解

### 一、概念原理

#### 1.1 syzkaller 配置要素

```
┌─────────────────────────────────────────────────────────────────────┐
│              syzkaller 配置要素                                      │
│                                                                     │
│  配置层级        关键参数              说明                         │
│  ─────────────────────────────────────────────────────────────      │
│                                                                     │
│  内核配置        CONFIG_KCOV=y         启用内核覆盖率收集           │
│  (编译时)        CONFIG_KASAN=y        启用内存错误检测             │
│                  CONFIG_DEBUG_FS=y     调试文件系统                 │
│                  CONFIG_DRM_AMDGPU=y   AMDGPU 驱动                  │
│                                                                     │
│  管理器配置      "target":             目标架构                     │
│  (drm.cfg)       "kernel_obj":         内核源码路径                 │
│                  "image":              QEMU 镜像路径                │
│                  "sshkey":             SSH 密钥                     │
│                  "vm": {               虚拟机配置                   │
│                    "count":            VM 实例数量                  │
│                    "cpu":              每 VM CPU 数                 │
│                    "mem":              每 VM 内存(MB)               │
│                  }                                                   │
│                  "enable_syscalls":    启用的 syscall 列表          │
│                                                                     │
│  运行时参数      -config               配置文件路径                 │
│                  -debug                调试模式                     │
│                  -bench                性能基准模式                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.2 覆盖率引导的 Fuzz 策略

```
┌─────────────────────────────────────────────────────────────────────┐
│              覆盖率引导 Fuzz 策略                                    │
│                                                                     │
│   输入生成 ──▶ 执行 ──▶ 覆盖率收集 ──▶ 反馈 ──▶ 语料库更新        │
│      │          │           │            │          │              │
│      │          │           │            │          │              │
│      ▼          ▼           ▼            ▼          ▼              │
│   ┌──────┐  ┌──────┐   ┌──────┐    ┌──────┐   ┌──────┐           │
│   │随机  │  │内核  │   │KCOV  │    │对比  │   │保存  │           │
│   │变异  │  │执行  │   │计数器│    │基线  │   │新输入│           │
│   │      │  │syscall│   │      │    │      │   │      │           │
│   └──────┘  └──────┘   └──────┘    └──────┘   └──────┘           │
│                                                                     │
│   关键优化：                                                         │
│   ┌─────────────────────────────────────────────────────────┐      │
│   │  1. 优先覆盖新代码的输入                                  │      │
│   │  2. 对高价值输入进行深度变异                              │      │
│   │  3. 维护最小语料库，去除冗余                              │      │
│   │  4. 跨 VM 共享有趣的输入                                  │      │
│   └─────────────────────────────────────────────────────────┘      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 二、实践操作

#### 2.1 完整配置示例

```bash
#!/bin/bash
# complete_syzkaller_setup.sh - syzkaller 完整配置

set -e

BASE_DIR="${1:-$HOME/syzkaller-drm}"
KERNEL_VERSION="6.8"
mkdir -p "$BASE_DIR"

echo "=== syzkaller 完整配置 ==="
echo "基础目录: $BASE_DIR"
echo ""

# ═══════════════════════════════════════════════════════════════
# 步骤 1: 准备内核源码
# ═══════════════════════════════════════════════════════════════

prepare_kernel() {
    echo "【步骤 1】准备内核源码"
    
    KERNEL_DIR="$BASE_DIR/linux"
    
    if [ ! -d "$KERNEL_DIR" ]; then
        echo "下载内核源码..."
        git clone --depth 1 --branch "v$KERNEL_VERSION" \
            https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git \
            "$KERNEL_DIR"
    fi
    
    cd "$KERNEL_DIR"
    
    # 创建最小化配置（用于 fuzzing）
    make defconfig
    
    # 启用必要配置
    ./scripts/config \
        --enable CONFIG_KCOV \
        --enable CONFIG_KASAN \
        --enable CONFIG_KASAN_INLINE \
        --enable CONFIG_DEBUG_FS \
        --enable CONFIG_DEBUG_KERNEL \
        --enable CONFIG_DEBUG_INFO \
        --enable CONFIG_DEBUG_INFO_DWARF5 \
        --enable CONFIG_DRM \
        --enable CONFIG_DRM_AMDGPU \
        --enable CONFIG_DRM_AMDGPU_SI \
        --enable CONFIG_DRM_AMDGPU_CIK \
        --enable CONFIG_NET \
        --enable CONFIG_INET \
        --enable CONFIG_BINFMT_MISC \
        --enable CONFIG_CONFIGFS_FS \
        --enable CONFIG_SECURITYFS \
        --enable CONFIG_PROC_FS \
        --enable CONFIG_SYSFS \
        --enable CONFIG_DEVTMPFS \
        --enable CONFIG_TMPFS \
        --enable CONFIG_BLK_DEV_INITRD \
        --enable CONFIG_RD_GZIP \
        --disable CONFIG_RANDOMIZE_BASE \
        --disable CONFIG_DEBUG_WX
    
    make olddefconfig
    
    # 编译
    make -j$(nproc)
    
    echo "内核编译完成"
    echo ""
}

# ═══════════════════════════════════════════════════════════════
# 步骤 2: 创建 QEMU 镜像
# ═══════════════════════════════════════════════════════════════

create_image() {
    echo "【步骤 2】创建 QEMU 镜像"
    
    IMAGE_DIR="$BASE_DIR/image"
    mkdir -p "$IMAGE_DIR"
    
    # 使用 debootstrap 创建 Debian 镜像（简化示例）
    # 实际应使用 syz-docker 或手动创建
    
    if [ ! -f "$IMAGE_DIR/stretch.img" ]; then
        echo "创建 Debian 镜像..."
        
        # 创建磁盘镜像
        qemu-img create -f qcow2 "$IMAGE_DIR/stretch.img" 2G
        
        # 创建分区（简化）
        # 实际应使用 debootstrap 或下载预构建镜像
        
        echo "注意：请手动创建或下载 QEMU 镜像"
        echo "参考: https://github.com/google/syzkaller/blob/master/docs/linux/setup_linux-host_qemu-vm.md"
    fi
    
    echo ""
}

# ═══════════════════════════════════════════════════════════════
# 步骤 3: 生成 SSH 密钥
# ═══════════════════════════════════════════════════════════════

generate_ssh_key() {
    echo "【步骤 3】生成 SSH 密钥"
    
    KEY_FILE="$BASE_DIR/stretch.id_rsa"
    
    if [ ! -f "$KEY_FILE" ]; then
        ssh-keygen -t rsa -b 4096 -N "" -f "$KEY_FILE"
        echo "SSH 密钥已生成"
    else
        echo "SSH 密钥已存在"
    fi
    
    echo ""
}

# ═══════════════════════════════════════════════════════════════
# 步骤 4: 创建管理器配置
# ═══════════════════════════════════════════════════════════════

create_manager_config() {
    echo "【步骤 4】创建管理器配置"
    
    CONFIG_FILE="$BASE_DIR/drm-manager.cfg"
    
    cat > "$CONFIG_FILE" << EOF
{
    "target": "linux/amd64",
    "http": ":56741",
    "workdir": "$BASE_DIR/workdir",
    "kernel_obj": "$BASE_DIR/linux",
    "kernel_src": "$BASE_DIR/linux",
    "image": "$BASE_DIR/image/stretch.img",
    "sshkey": "$BASE_DIR/stretch.id_rsa",
    "syzkaller": "$HOME/go/pkg/mod/github.com/google/syzkaller@latest",
    "procs": 8,
    "type": "qemu",
    "vm": {
        "count": 4,
        "kernel": "$BASE_DIR/linux/arch/x86/boot/bzImage",
        "cpu": 2,
        "mem": 2048,
        "cmdline": "console=ttyS0 root=/dev/sda1 debug earlyprintk=serial slub_debug=UZ"
    },
    "enable_syscalls": [
        "openat\$drm",
        "ioctl\$DRM_IOCTL_VERSION",
        "ioctl\$DRM_IOCTL_GET_UNIQUE",
        "ioctl\$DRM_IOCTL_SET_UNIQUE",
        "ioctl\$DRM_IOCTL_GET_MAGIC",
        "ioctl\$DRM_IOCTL_AUTH_MAGIC",
        "ioctl\$DRM_IOCTL_ADD_MAP",
        "ioctl\$DRM_IOCTL_RM_MAP",
        "ioctl\$DRM_IOCTL_ADD_BUFS",
        "ioctl\$DRM_IOCTL_MARK_BUFS",
        "ioctl\$DRM_IOCTL_INFO_BUFS",
        "ioctl\$DRM_IOCTL_MAP_BUFS",
        "ioctl\$DRM_IOCTL_FREE_BUFS",
        "ioctl\$DRM_IOCTL_ADD_CTX",
        "ioctl\$DRM_IOCTL_RM_CTX",
        "ioctl\$DRM_IOCTL_MOD_CTX",
        "ioctl\$DRM_IOCTL_GET_CTX",
        "ioctl\$DRM_IOCTL_SWITCH_CTX",
        "ioctl\$DRM_IOCTL_NEW_CTX",
        "ioctl\$DRM_IOCTL_RES_CTX",
        "ioctl\$DRM_IOCTL_LOCK",
        "ioctl\$DRM_IOCTL_UNLOCK",
        "ioctl\$DRM_IOCTL_CONTROL",
        "ioctl\$DRM_IOCTL_AGP_ACQUIRE",
        "ioctl\$DRM_IOCTL_AGP_RELEASE",
        "ioctl\$DRM_IOCTL_AGP_ENABLE",
        "ioctl\$DRM_IOCTL_AGP_ALLOC",
        "ioctl\$DRM_IOCTL_AGP_FREE",
        "ioctl\$DRM_IOCTL_AGP_BIND",
        "ioctl\$DRM_IOCTL_AGP_UNBIND",
        "ioctl\$DRM_IOCTL_SG_ALLOC",
        "ioctl\$DRM_IOCTL_SG_FREE",
        "ioctl\$DRM_IOCTL_WAIT_VBLANK",
        "ioctl\$DRM_IOCTL_MODESET_CTL",
        "ioctl\$DRM_IOCTL_GEM_CLOSE",
        "ioctl\$DRM_IOCTL_GEM_FLINK",
        "ioctl\$DRM_IOCTL_GEM_OPEN",
        "ioctl\$DRM_IOCTL_GET_CAP",
        "ioctl\$DRM_IOCTL_SET_CLIENT_CAP",
        "ioctl\$DRM_IOCTL_PRIME_HANDLE_TO_FD",
        "ioctl\$DRM_IOCTL_PRIME_FD_TO_HANDLE",
        "ioctl\$DRM_IOCTL_MODE_GETRESOURCES",
        "ioctl\$DRM_IOCTL_MODE_GETCRTC",
        "ioctl\$DRM_IOCTL_MODE_SETCRTC",
        "ioctl\$DRM_IOCTL_MODE_CURSOR",
        "ioctl\$DRM_IOCTL_MODE_GETGAMMA",
        "ioctl\$DRM_IOCTL_MODE_SETGAMMA",
        "ioctl\$DRM_IOCTL_MODE_GETENCODER",
        "ioctl\$DRM_IOCTL_MODE_GETCONNECTOR",
        "ioctl\$DRM_IOCTL_MODE_ATTACHMODE",
        "ioctl\$DRM_IOCTL_MODE_DETACHMODE",
        "ioctl\$DRM_IOCTL_MODE_GETPROPERTY",
        "ioctl\$DRM_IOCTL_MODE_SETPROPERTY",
        "ioctl\$DRM_IOCTL_MODE_GETPROPBLOB",
        "ioctl\$DRM_IOCTL_MODE_GETFB",
        "ioctl\$DRM_IOCTL_MODE_ADDFB",
        "ioctl\$DRM_IOCTL_MODE_RMFB",
        "ioctl\$DRM_IOCTL_MODE_PAGE_FLIP",
        "ioctl\$DRM_IOCTL_MODE_DIRTYFB",
        "ioctl\$DRM_IOCTL_MODE_CREATE_DUMB",
        "ioctl\$DRM_IOCTL_MODE_MAP_DUMB",
        "ioctl\$DRM_IOCTL_MODE_DESTROY_DUMB",
        "ioctl\$DRM_IOCTL_MODE_OBJ_GETPROPERTIES",
        "ioctl\$DRM_IOCTL_MODE_OBJ_SETPROPERTY",
        "ioctl\$DRM_IOCTL_MODE_CURSOR2",
        "ioctl\$DRM_IOCTL_MODE_ATOMIC",
        "ioctl\$DRM_IOCTL_MODE_CREATEPROPBLOB",
        "ioctl\$DRM_IOCTL_MODE_DESTROYPROPBLOB",
        "ioctl\$DRM_IOCTL_AMDGPU_INFO",
        "ioctl\$DRM_IOCTL_AMDGPU_GEM_CREATE",
        "ioctl\$DRM_IOCTL_AMDGPU_CTX",
        "ioctl\$DRM_IOCTL_AMDGPU_BO_LIST",
        "ioctl\$DRM_IOCTL_AMDGPU_CS",
        "ioctl\$DRM_IOCTL_AMDGPU_GEM_METADATA",
        "ioctl\$DRM_IOCTL_AMDGPU_GEM_MMAP",
        "ioctl\$DRM_IOCTL_AMDGPU_GEM_WAIT_IDLE",
        "ioctl\$DRM_IOCTL_AMDGPU_GEM_VA",
        "ioctl\$DRM_IOCTL_AMDGPU_WAIT_CS",
        "ioctl\$DRM_IOCTL_AMDGPU_GEM_OP",
        "ioctl\$DRM_IOCTL_AMDGPU_GEM_USERPTR"
    ],
    "suppressions": ["WARNING:"],
    "sandbox": "none"
}
EOF
    
    echo "配置已创建: $CONFIG_FILE"
    echo ""
}

# ═══════════════════════════════════════════════════════════════
# 主流程
# ═══════════════════════════════════════════════════════════════

case "${1:-all}" in
    kernel)
        prepare_kernel
        ;;
    image)
        create_image
        ;;
    ssh)
        generate_ssh_key
        ;;
    config)
        create_manager_config
        ;;
    all)
        prepare_kernel
        create_image
        generate_ssh_key
        create_manager_config
        ;;
    *)
        echo "用法: $0 {kernel|image|ssh|config|all}"
        exit 1
        ;;
esac

echo "=== 配置完成 ==="
echo ""
echo "启动命令:"
echo "  syz-manager -config=$BASE_DIR/drm-manager.cfg"
echo ""
echo "Web 界面: http://localhost:56741"
```

#### 2.2 运行监控与优化

```bash
#!/bin/bash
# monitor_fuzzing.sh - 监控 fuzz 运行状态

set -e

WORK_DIR="${1:-$HOME/syzkaller-drm/workdir}"

echo "=== Fuzz 运行监控 ==="
echo ""

# 检查进程
echo "## syz-manager 进程"
ps aux | grep syz-manager | grep -v grep || echo "syz-manager 未运行"
echo ""

# 检查覆盖率
echo "## 覆盖率统计"
if [ -f "$WORK_DIR/covercount" ]; then
    echo "覆盖的基本块数: $(cat $WORK_DIR/covercount)"
fi

if [ -d "$WORK_DIR/corpus" ]; then
    echo "语料库大小: $(ls "$WORK_DIR/corpus" | wc -l)"
fi
echo ""

# 检查崩溃
echo "## 崩溃统计"
if [ -d "$WORK_DIR/crashes" ]; then
    CRASH_COUNT=$(ls "$WORK_DIR/crashes" 2>/dev/null | wc -l)
    echo "崩溃数量: $CRASH_COUNT"
    
    if [ "$CRASH_COUNT" -gt 0 ]; then
        echo "崩溃类型分布:"
        for crash in "$WORK_DIR/crashes"/*; do
            if [ -f "$crash" ]; then
                basename "$crash"
            fi
        done
    fi
else
    echo "无崩溃目录"
fi
echo ""

# 检查日志
echo "## 最近日志"
if [ -f "$WORK_DIR/log" ]; then
    tail -20 "$WORK_DIR/log"
fi
```

### 三、案例分析

#### 3.1 崩溃复现

```bash
#!/bin/bash
# reproduce_crash.sh - 复现 syzkaller 发现的崩溃

set -e

CRASH_FILE="$1"
KERNEL_DIR="${2:-$HOME/syzkaller-drm/linux}"

if [ -z "$CRASH_FILE" ]; then
    echo "用法: $0 <crash-file> [kernel-dir]"
    exit 1
fi

echo "=== 崩溃复现 ==="
echo "崩溃文件: $CRASH_FILE"
echo ""

# 提取 reproducer
if grep -q "^# {" "$CRASH_FILE"; then
    echo "找到 reproducer 程序"
    
    # 编译 reproducer
    REPRO_C="$CRASH_FILE.repro.c"
    
    # 使用 syz-repro 工具
    syz-repro -config="$HOME/syzkaller-drm/drm-manager.cfg" \
              "$CRASH_FILE" 2>&1 | tee "$CRASH_FILE.repro.log"
    
    echo ""
    echo "复现结果保存在: $CRASH_FILE.repro.log"
else
    echo "未找到自动 reproducer"
    echo "尝试手动分析..."
    
    # 显示崩溃信息
    echo "崩溃类型:"
    grep "BUG:\|WARNING:\|KASAN\|general protection" "$CRASH_FILE" | head -5
    
    echo ""
    echo "调用栈:"
    grep -A 30 "Call Trace:" "$CRASH_FILE" | head -30
fi
```

### 四、相关链接

- syzkaller 官方文档：https://github.com/google/syzkaller/tree/master/docs
- syzkaller Linux 设置：https://github.com/google/syzkaller/blob/master/docs/linux/setup.md
- KCOV 文档：https://docs.kernel.org/dev-tools/kcov.html
- KASAN 文档：https://docs.kernel.org/dev-tools/kasan.html

## 今日小结

- syzkaller 配置包括内核编译（启用 KCOV/KASAN）、镜像创建、SSH 配置和管理器配置
- 覆盖率引导的 fuzz 策略优先覆盖新代码，提高漏洞发现效率
- Web 界面提供实时监控，包括覆盖率、语料库和崩溃统计
- 崩溃复现是验证漏洞的关键步骤，syz-repro 工具可自动生成 reproducer

## 扩展思考

1. 如何优化 syzkaller 的配置以提高 AMDGPU 特定代码的覆盖率？
2. 在资源有限的情况下，如何平衡 VM 数量和每个 VM 的 CPU/内存配置？
