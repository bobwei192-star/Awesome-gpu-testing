# Day 398：GPU 驱动 ABI 稳定性原则

## 学习目标

- 理解内核 ABI (Application Binary Interface) 和用户空间 ABI 的区别
- 掌握 DRM 子系统的 uAPI 稳定性和兼容性保证
- 学会识别和避免破坏 ABI 的变更
- 了解 ABI 变更的评估和审批流程

## 知识详解

### 概念原理

#### ABI vs API 的区别

```
ABI 与 API 对比:

┌─────────────────────────────────────────────────────────────────┐
│  API (Application Programming Interface)                        │
│                                                                 │
│  定义: 源代码级别的接口约定                                      │
│  粒度: 函数声明、数据结构定义、宏                                │
│  兼容性: 重新编译即可适配                                       │
│  示例: 调用函数 foo(int x, int y)                               │
│                                                                 │
│  保护机制: 无特殊保护，内核 API 可以在版本间变化                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  ABI (Application Binary Interface)                             │
│                                                                 │
│  定义: 二进制级别的接口约定                                      │
│  粒度: 结构体布局、函数参数传递方式、系统调用号                  │
│  兼容性: 需要二进制兼容，不能重新编译                            │
│  示例: ioctl(fd, DRM_IOCTL_AMDGPU_GEM_CREATE, &args)           │
│                                                                 │
│  保护机制: 必须向后兼容，破坏 ABI 需要非常严格的审批             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

ABI 稳定性的关键方面:

┌──────────────────────────────────────────────────────────────┐
│  1. 结构体布局 (struct layout)                                │
│     - 字段顺序不能变                                          │
│     - 字段类型不能变 (大小变化会破坏布局)                      │
│     - 只能追加字段到末尾 (使用 padding/reserved 字段)         │
│                                                              │
│  2. 枚举值 (enum values)                                      │
│     - 已有枚举值不能改变                                       │
│     - 只能追加新值                                            │
│                                                              │
│  3. 系统调用号 (syscall numbers)                               │
│     - 不能改变或移除已分配的系统调用号                         │
│                                                              │
│  4. ioctl 命令码 (ioctl command codes)                        │
│     - 每个 ioctl 有唯一的命令码                                 │
│     - 命令码编码了: 类型/序号/方向/大小                        │
│                                                              │
│  5. 设备文件语义 (device file semantics)                      │
│     - open/close/read/write/mmap 的行为                       │
│     - mmap 返回的内存布局                                     │
└──────────────────────────────────────────────────────────────┘
```

#### DRM uAPI 稳定性保证

```
DRM uAPI (用户空间 API) 的稳定性承诺:

Linux 内核社区对 DRM uAPI 的"一次提交，永远支持"原则:

  ┌──────────────────────────────────────────────────────────┐
  │  "Once a uAPI is merged and released in a kernel         │
  │   release, it must be supported indefinitely."           │
  │                                                          │
  │   — DRM uAPI Stability Policy                            │
  └──────────────────────────────────────────────────────────┘

这意味着:

  ✅ 已有 ioctl 不能移除或改变含义
  ✅ 已有结构体字段不能重用或改变含义
  ✅ 已有枚举值不能重新定义
  ❌ 不能破坏已有的用户空间程序

uAPI 的兼容性分类:

  ┌─────────────────────────────────────────────────────────┐
  │  类别              │ 示例                   │ 保证     │
  ├─────────────────────────────────────────────────────────┤
  │  ioctl 命令         │ DRM_AMDGPU_GEM_CREATE │ 永久     │
  │  ioctl 参数结构体    │ drm_amdgpu_gem_create │ 向前兼容 │
  │  sysfs 接口         │ /sys/class/drm/*       │ 稳定     │
  │  debugfs 接口        │ /sys/kernel/debug/dri/*│ 无保证   │
  │  module 参数        │ amdgpu.gpu_recovery    │ 稳定     │
  │  /dev/dri/* 语义     │ open, mmap            │ 稳定     │
  │  procfs 接口         │ /proc/dri/*           │ 无保证   │
  │  KMS 属性            │ "scaling mode"        │ 稳定     │
  │  DRM 事件            │ VBLANK, PAGE_FLIP     │ 稳定     │
  └─────────────────────────────────────────────────────────┘
```

#### AMDGPU 专用 uAPI

```
AMDGPU uAPI 层次结构:

┌─────────────────────────────────────────────────────────────────┐
│  用户空间程序                                                     │
│  (Mesa Vulkan/OpenGL, ROCm, GAMESCAN, ...)                      │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   │  ioctl() / mmap() / sysfs
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│               DRM 核心 uAPI (通用)                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  DRM_IOCTL_MODE_GETRESOURCES       - 获取显示资源                │
│  DRM_IOCTL_MODE_GETCONNECTOR       - 获取连接器信息               │
│  DRM_IOCTL_MODE_ADDFB              - 添加 framebuffer            │
│  DRM_IOCTL_MODE_SETCRTC            - 设置 CRTC 模式              │
│  DRM_IOCTL_MODE_PAGE_FLIP          - 页面翻转                    │
│  DRM_IOCTL_MODE_CREATE_DUMB        - 创建 dumb buffer            │
│  DRM_IOCTL_MODE_MAP_DUMB           - 映射 dumb buffer            │
│  DRM_IOCTL_PRIME_FD_TO_HANDLE      - PRIME 导入                  │
│  DRM_IOCTL_PRIME_HANDLE_TO_FD      - PRIME 导出                  │
│  DRM_IOCTL_SYNCOBJ_CREATE          - 同步对象创建                │
│  DRM_IOCTL_SYNCOBJ_WAIT            - 同步对象等待                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                   │
                   │  AMDGPU 专用 ioctl
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│               AMDGPU 专用 uAPI                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  DRM_AMDGPU_GEM_CREATE               - 创建 GEM 对象             │
│  DRM_AMDGPU_GEM_MMAP                 - 映射 GEM 对象             │
│  DRM_AMDGPU_GEM_WAIT_IDLE            - 等待 GPU 空闲             │
│  DRM_AMDGPU_GEM_METADATA             - GEM 元数据               │
│  DRM_AMDGPU_GEM_VA                   - 虚拟地址管理              │
│  DRM_AMDGPU_GEM_OP                   - GEM 操作                  │
│  DRM_AMDGPU_CTX                      - GPU 上下文管理            │
│  DRM_AMDGPU_VM                       - 虚拟内存管理              │
│  DRM_AMDGPU_SCHED                    - 调度参数控制              │
│  DRM_AMDGPU_BO_LIST                  - Buffer 列表管理           │
│  DRM_AMDGPU_FENCE_TO_HANDLE          - Fence 转换               │
│  DRM_AMDGPU_INFO                     - 设备信息查询              │
│  DRM_AMDGPU_WAIT_FENCES              - 等待多个 Fence            │
│  DRM_AMDGPU_GEM_USERPTR              - 用户指针注册              │
│  DRM_AMDGPU_GEM_CREATE_BO           - BO 创建                   │
│  DRM_AMDGPU_GPU_RESET                - GPU 重置控制              │
│  DRM_AMDGPU_GEM_CREATE_VM            - VM 创建                  │
│  DRM_AMDGPU_QUERY_FW_VERSION         - 固件版本查询              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                   │
                   │  KMS 属性 (sysfs/ioctl)
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│               AMDGPU KMS 属性                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  显示属性:                                                        │
│    "scaling mode"         - 缩放模式 (Full/Center/Aspect)        │
│    "underscan"            - 欠扫描设置                          │
│    "dithering"            - 抖动模式                            │
│    "freesync"             - FreeSync 开关                       │
│    "abm level"            - 自适应亮度管理                        │
│    "max bpc"              - 最大色深                            │
│                                                                  │
│  GPU 属性:                                                       │
│    "power_dpm_force_perf" - 性能等级强制                        │
│    "gpu_busy_percent"     - GPU 利用率                          │
│    "mem_busy_percent"     - 显存利用率                          │
│    "temperature"          - 温度传感器                          │
│    "fan_speed"            - 风扇转速                            │
│    "pp_od_clk_voltage"    - 超频/电压设置                        │
│    "pp_power_profile_mode" - 功耗模式                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### ioctl 命令码编码

```
Linux ioctl 命令码编码:

                      ┌─────────────────────────────┐
                      │  ioctl 命令码 = 32-bit       │
                      ├─────────────────────────────┤
                      │  bit 31-30: 方向 (2 bits)    │
                      │  bit 29-16: 类型 (8 bits)    │
                      │  bit 15-8:  序号 (8 bits)    │
                      │  bit 7-0:   大小 (8 bits)    │
                      └─────────────────────────────┘

DRM ioctl 命令码:

  #define DRM_IOCTL_BASE             'd'
  #define DRM_IOWR(nr, type)         _IOWR(DRM_IOCTL_BASE, nr, type)

  AMDGPU 示例:
  #define DRM_AMDGPU_GEM_CREATE      DRM_IOWR(0x40, union drm_amdgpu_gem_create)
  #define DRM_AMDGPU_GEM_MMAP        DRM_IOWR(0x41, union drm_amdgpu_gem_mmap)
  #define DRM_AMDGPU_CTX             DRM_IOWR(0x42, union drm_amdgpu_ctx)
  #define DRM_AMDGPU_VM              DRM_IOWR(0x43, union drm_amdgpu_vm)
  #define DRM_AMDGPU_SCHED           DRM_IOWR(0x44, union drm_amdgpu_sched)
  #define DRM_AMDGPU_INFO            DRM_IOWR(0x45, union drm_amdgpu_info)
  ...

  _IOWR 编码:
  - 方向: _IOWR = 3 (读写)
  - 类型: 'd' = 0x64
  - 序号: 0x40-0x5F (AMDGPU 专用范围)
  - 大小: sizeof(struct) 由编译器确定

  序号分配范围:
  0x00-0x3F - DRM 核心 ioctl
  0x40-0x5F - AMDGPU 专用 ioctl
  0x60-0x7F - Nouveau 专用 ioctl
  0x80-0x9F - i915 专用 ioctl
  0xA0-0xBF - vc4/v3d 专用 ioctl
  0xC0-0xFF - 保留
```

#### ABI 兼容性检查清单

```
ABI 变更兼容性检查清单:

┌─────────────────────────────────────────────────────────────────┐
│  ⚠️  以下更改会破坏 ABI，必须避免:                              │
│                                                                  │
│  1. 修改已有结构体字段的偏移                                     │
│     ❌ 在结构体中间插入新字段                                    │
│     ❌ 删除已有字段                                              │
│     ❌ 改变字段类型 (导致大小变化)                                │
│                                                                  │
│  2. 改变已有 ioctl 的行为                                        │
│     ❌ 改变已有命令码的含义                                      │
│     ❌ 要求新的 flags 位必须清除 (旧程序不会设置)                │
│     ❌ 改变返回数据的格式                                        │
│                                                                  │
│  3. 改变枚举值的含义                                             │
│     ❌ 改变已有枚举的数值                                        │
│     ❌ 删除已有枚举值                                            │
│                                                                  │
│  4. 改变设备文件语义                                             │
│     ❌ 改变 mmap 返回的页大小                                    │
│     ❌ 改变 open/close 的行为                                    │
│                                                                  │
│  ✅ 以下更改通常可以接受:                                        │
│                                                                  │
│  1. 在结构体末尾追加新字段                                       │
│     → 用户空间必须使用 sizeof 感知新旧结构体                     │
│     → 内核通过判断传入的 size 区分版本                          │
│                                                                  │
│  2. 新增 ioctl 命令码                                            │
│     → 必须在已有的命令码之后分配新的序号                         │
│     → 需要更新 DRM_AMDGPU_NUM_IOCTLS                             │
│                                                                  │
│  3. 新增枚举值                                                   │
│     → 只能追加到已有枚举的末尾                                   │
│     → 不能改变已有枚举的数值                                     │
│                                                                  │
│  4. 新增标志位 (flags)                                           │
│     → 内核必须忽略未识别的标志位                                 │
│     → 不能要求新标志位必须设置                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 实践操作

#### ABI 兼容性检查工具

```bash
#!/bin/bash
# AMDGPU ABI 兼容性检查工具

set -e

echo "=============================================="
echo "  AMDGPU ABI 兼容性检查工具"
echo "=============================================="
echo ""

KERNEL_DIR="${HOME}/linux"
ABI_REFERENCE_DIR="${HOME}/abi-references"

# 1. 生成 uAPI 结构体定义
dump_uapi_structs() {
    echo "[1/6] 导出 uAPI 结构体定义..."
    
    cd "${KERNEL_DIR}"
    
    local output_dir="${ABI_REFERENCE_DIR}/uapi-$(date +%Y%m%d)"
    mkdir -p "${output_dir}"
    
    # 从 UAPI 头文件中提取结构体定义
    local uapi_headers=(
        "include/uapi/drm/drm.h"
        "include/uapi/drm/amdgpu_drm.h"
    )
    
    for header in "${uapi_headers[@]}"; do
        if [ -f "${header}" ]; then
            local base_name=$(basename "${header}")
            cp "${header}" "${output_dir}/${base_name}"
            echo "  ✅ 已导出: ${base_name}"
        fi
    done
    
    # 提取 ioctl 定义
    echo ""
    echo "  ioctl 命令码定义:"
    grep -n "^#define DRM_AMDGPU_" "${output_dir}/amdgpu_drm.h" | \
        grep "DRM_IOWR\|DRM_IOR\|DRM_IOW" | while read line; do
        echo "    ${line}"
    done
    
    echo ""
    echo "  ✅ uAPI 结构体已导出到: ${output_dir}"
}

# 2. 结构体大小检查
check_struct_sizes() {
    echo "[2/6] 检查结构体大小变化..."
    
    local old_dir="$1"
    local new_dir="$2"
    
    if [ -z "${old_dir}" ] || [ -z "${new_dir}" ]; then
        echo "用法: $0 struct-sizes <old-dir> <new-dir>"
        echo "比较两个版本的 uAPI 结构体大小"
        return 1
    fi
    
    cd "${KERNEL_DIR}"
    
    echo "  比较:"
    echo "    旧版: ${old_dir}"
    echo "    新版: ${new_dir}"
    echo ""
    
    # 创建临时测试文件
    local test_file="/tmp/uapi_size_test.c"
    
    cat > "${test_file}" << 'EOF'
#include <stdio.h>
#include <stddef.h>
#include <drm/drm.h>
#include <drm/amdgpu_drm.h>

#define CHECK_SIZE(type) \
    printf("%-50s %4zu bytes\n", #type, sizeof(type))

#define CHECK_FIELD_OFFSET(type, field) \
    printf("%-50s %4zu bytes\n", #type "::" #field, \
           offsetof(type, field))

#define CHECK_FIELD_SIZE(type, field) \
    printf("%-50s %4zu bytes\n", #type "::" #field, \
           sizeof(((type *)0)->field))

int main() {
    printf("=== DRM 核心结构体 ===\n");
    CHECK_SIZE(drm_version);
    CHECK_SIZE(drm_unique);
    CHECK_SIZE(drm_set_version);
    
    printf("\n=== AMDGPU GEM 结构体 ===\n");
    CHECK_SIZE(drm_amdgpu_gem_create_in);
    CHECK_SIZE(drm_amdgpu_gem_create_out);
    CHECK_SIZE(drm_amdgpu_gem_mmap_in);
    CHECK_SIZE(drm_amdgpu_gem_mmap_out);
    CHECK_SIZE(drm_amdgpu_gem_wait_idle_in);
    CHECK_SIZE(drm_amdgpu_gem_metadata);
    CHECK_SIZE(drm_amdgpu_gem_va);
    
    printf("\n=== AMDGPU 上下文结构体 ===\n");
    CHECK_SIZE(drm_amdgpu_ctx_in);
    CHECK_SIZE(drm_amdgpu_ctx_out);
    CHECK_SIZE(drm_amdgpu_ctx_reg);
    
    printf("\n=== AMDGPU VM 结构体 ===\n");
    CHECK_SIZE(drm_amdgpu_vm_in);
    CHECK_SIZE(drm_amdgpu_vm_out);
    
    printf("\n=== AMDGPU 信息结构体 ===\n");
    CHECK_SIZE(drm_amdgpu_info_device);
    CHECK_SIZE(drm_amdgpu_info_hw_ip);
    CHECK_SIZE(drm_amdgpu_info_firmware);
    
    printf("\n=== AMDGPU BO 列表结构体 ===\n");
    CHECK_SIZE(drm_amdgpu_bo_list_in);
    CHECK_SIZE(drm_amdgpu_bo_list_entry);
    
    printf("\n=== AMDGPU 调度结构体 ===\n");
    CHECK_SIZE(drm_amdgpu_sched_in);
    CHECK_SIZE(drm_amdgpu_sched_out);
    
    return 0;
}
EOF
    
    # 编译旧版
    echo "  编译旧版..."
    local old_bin="/tmp/uapi_size_test_old"
    gcc -o "${old_bin}" "${test_file}" \
        -I"${old_dir}" -I"${KERNEL_DIR}/include" \
        2>/dev/null || {
        echo "  ❌ 旧版编译失败，请确认路径"
        return 1
    }
    
    # 编译新版
    echo "  编译新版..."
    local new_bin="/tmp/uapi_size_test_new"
    gcc -o "${new_bin}" "${test_file}" \
        -I"${new_dir}" -I"${KERNEL_DIR}/include" \
        2>/dev/null || {
        echo "  ❌ 新版编译失败"
        return 1
    }
    
    # 运行旧版
    echo ""
    echo "  旧版结构体大小:"
    "${old_bin}" > /tmp/uapi_old_sizes.txt
    cat /tmp/uapi_old_sizes.txt
    
    # 运行新版
    echo ""
    echo "  新版结构体大小:"
    "${new_bin}" > /tmp/uapi_new_sizes.txt
    cat /tmp/uapi_new_sizes.txt
    
    # 比较差异
    echo ""
    echo "  ⚠️  大小变化:"
    diff /tmp/uapi_old_sizes.txt /tmp/uapi_new_sizes.txt || true
    
    # 清理
    rm -f "${test_file}" "${old_bin}" "${new_bin}"
}

# 3. ioctl 命令码冲突检查
check_ioctl_numbers() {
    echo "[3/6] 检查 ioctl 命令码..."
    
    cd "${KERNEL_DIR}"
    
    local header="include/uapi/drm/amdgpu_drm.h"
    
    if [ ! -f "${header}" ]; then
        echo "  ❌ 文件不存在: ${header}"
        return 1
    fi
    
    echo "  AMDGPU ioctl 命令码:"
    echo ""
    
    # 提取所有 AMDGPU ioctl 定义
    grep -n "DRM_AMDGPU_" "${header}" | grep "DRM_IOWR\|DRM_IOR\|DRM_IOW" | \
    while read line; do
        local line_num=$(echo "${line}" | cut -d: -f1)
        local def=$(echo "${line}" | sed 's/.*#define //')
        local name=$(echo "${def}" | awk '{print $1}')
        local nr=$(echo "${def}" | grep -oP '0x[0-9a-fA-F]+' | head -1)
        
        # 提取 ioctl 号
        local ioctl_nr=$((nr))
        
        printf "    %-45s 0x%02x (%3d)\n" "${name}" "${ioctl_nr}" "${ioctl_nr}"
    done
    
    echo ""
    echo "  检查重复号..."
    local duplicates=$(grep "DRM_AMDGPU_" "${header}" | grep "DRM_IOWR\|DRM_IOR\|DRM_IOW" | \
                      sed 's/.*0x//' | cut -d, -f1 | sort | uniq -d)
    if [ -n "${duplicates}" ]; then
        echo "  ❌ 发现重复 ioctl 号:"
        echo "${duplicates}"
    else
        echo "  ✅ 无重复号"
    fi
    
    # 检查序号范围
    echo ""
    echo "  范围检查:"
    local max_nr=$(grep "DRM_AMDGPU_" "${header}" | grep "DRM_IOWR\|DRM_IOR\|DRM_IOW" | \
                  sed 's/.*0x//' | cut -d, -f1 | sort -n | tail -1)
    if [ "${max_nr}" -gt 0x5F ]; then
        echo "  ⚠️  AMDGPU 使用范围 0x40-0x5F"
        echo "  最大使用序号: 0x${max_nr}"
        echo "  ⚠️  注意: 序号超过 0x5F 可能与其他驱动冲突"
    else
        echo "  ✅ 序号在 AMDGPU 分配范围内 (0x40-0x5F)"
    fi
}

# 4. KMS 属性兼容性检查
check_kms_properties() {
    echo "[4/6] 检查 KMS 属性..."
    
    cd "${KERNEL_DIR}"
    
    # 查找 amdgpu KMS 属性定义
    local prop_files=$(grep -rl "amdgpu.*property\|amdgpu.*attr" \
                      drivers/gpu/drm/amd/amdgpu/ 2>/dev/null | head -10)
    
    echo "  KMS 属性定义文件:"
    for file in ${prop_files}; do
        echo "    ${file}"
    done
    echo ""
    
    # 提取属性名称
    echo "  amdgpu 驱动注册的 KMS 属性:"
    grep -h "drm_property_create\|drm_mode_create_property" \
        drivers/gpu/drm/amd/amdgpu/*.c 2>/dev/null | \
        grep -oP '"[^"]+"' | sort -u | while read prop; do
        echo "    ${prop}"
    done
    
    echo ""
    echo "  属性兼容性检查:"
    echo "  ✅ 属性名称不能改变 (Mesa 通过名称查找)"
    echo "  ✅ 属性值的含义不能改变"
    echo "  ✅ 可以添加新属性，但不能移除已有属性"
}

# 5. sysfs 接口检查
check_sysfs_interfaces() {
    echo "[5/6] 检查 sysfs 接口..."
    
    cd "${KERNEL_DIR}"
    
    # 查找 sysfs 接口定义
    echo "  amdgpu sysfs 接口:"
    grep -rh "DEVICE_ATTR\|DRM_DEVICE_ATTR\|amdgpu_sysfs\|sysfs_emit" \
        drivers/gpu/drm/amd/amdgpu/ 2>/dev/null | \
        grep "DEVICE_ATTR" | grep -oP '[A-Z_]+\([^)]*\)' | sort -u | \
        while read attr; do
        echo "    ${attr}"
    done
    
    echo ""
    echo "  实际 sysfs 路径检查 (当前系统):"
    local sysfs_dir="/sys/class/drm"
    if [ -d "${sysfs_dir}" ]; then
        for card in "${sysfs_dir}"/card*; do
            if [ -d "${card}/device" ]; then
                echo "  ${card}:"
                ls "${card}/device/" 2>/dev/null | grep -v "driver\|subsystem\|power" | \
                    head -10 | while read entry; do
                    echo "    ${entry}"
                done
            fi
        done
    else
        echo "  ⚠️  当前系统没有 DRM 设备"
    fi
}

# 6. 生成 ABI 报告
generate_abi_report() {
    echo "[6/6] 生成 ABI 兼容性报告..."
    
    local report_file="${HOME}/abi-report-$(date +%Y%m%d).md"
    
    cd "${KERNEL_DIR}"
    
    cat > "${report_file}" << 'HEADER'
# AMDGPU ABI 兼容性报告

## 概述
HEADER
    
    echo "" >> "${report_file}"
    echo "生成时间: $(date)" >> "${report_file}"
    echo "内核版本: $(make kernelversion 2>/dev/null || echo 'N/A')" >> "${report_file}"
    echo "" >> "${report_file}"
    
    # ioctl 列表
    echo "## AMDGPU ioctl 接口" >> "${report_file}"
    echo "| 名称 | 命令码 | 序号 | 方向 |" >> "${report_file}"
    echo "|------|--------|------|------|" >> "${report_file}"
    
    grep "DRM_AMDGPU_" include/uapi/drm/amdgpu_drm.h | \
        grep "DRM_IOWR\|DRM_IOR\|DRM_IOW" | \
    while read line; do
        local def=$(echo "${line}" | sed 's/.*#define //')
        local name=$(echo "${def}" | awk '{print $1}')
        local cmd=$(echo "${def}" | grep -oP 'DRM_IOWR|DRM_IOR|DRM_IOW')
        local nr=$(echo "${def}" | grep -oP '0x[0-9a-fA-F]+' | head -1)
        echo "| ${name} | ${cmd} | ${nr} | $(echo ${cmd} | sed 's/DRM_//') |" >> "${report_file}"
    done
    
    echo "" >> "${report_file}"
    
    # KMS 属性
    echo "## KMS 属性" >> "${report_file}" >> "${report_file}"
    grep -h "drm_property_create\|drm_mode_create_property" \
        drivers/gpu/drm/amd/amdgpu/*.c 2>/dev/null | \
        grep -oP '"[^"]+"' | sort -u | while read prop; do
        echo "- ${prop}" >> "${report_file}"
    done
    
    echo "" >> "${report_file}"
    
    # 检查最近 ABI 相关提交
    echo "## 近期 ABI 相关变更" >> "${report_file}"
    echo '```' >> "${report_file}"
    git log --oneline --all --since="6 months ago" -- \
        "include/uapi/drm/amdgpu_drm.h" \
        "include/uapi/drm/drm.h" 2>/dev/null >> "${report_file}"
    echo '```' >> "${report_file}"
    
    echo "" >> "${report_file}"
    echo "## 兼容性结论" >> "${report_file}"
    echo "- [ ] ioctl 命令码无冲突" >> "${report_file}"
    echo "- [ ] 结构体大小无变化" >> "${report_file}"
    echo "- [ ] KMS 属性未移除" >> "${report_file}"
    echo "- [ ] 无新的 ABI 破坏性变更" >> "${report_file}"
    
    echo "  ✅ 报告已生成: ${report_file}"
}

# 主流程
case "$1" in
    dump)
        dump_uapi_structs
        ;;
    struct-sizes)
        check_struct_sizes "$2" "$3"
        ;;
    ioctl)
        check_ioctl_numbers
        ;;
    kms)
        check_kms_properties
        ;;
    sysfs)
        check_sysfs_interfaces
        ;;
    report)
        generate_abi_report
        ;;
    all)
        dump_uapi_structs
        check_ioctl_numbers
        check_kms_properties
        generate_abi_report
        ;;
    *)
        echo "用法: $0 {dump|struct-sizes|ioctl|kms|sysfs|report|all}"
        echo ""
        echo "  dump         - 导出 uAPI 结构体定义"
        echo "  struct-sizes - 比较结构体大小变化"
        echo "  ioctl        - 检查 ioctl 命令码"
        echo "  kms          - 检查 KMS 属性"
        echo "  sysfs        - 检查 sysfs 接口"
        echo "  report       - 生成 ABI 兼容性报告"
        echo "  all          - 完整检查"
        ;;
esac
```

#### 结构体扩展模式

```c
// Linux 内核结构体扩展的常见模式

// 模式 1: reserved 字段预留
// 在首次定义结构体时预留 padding，为未来扩展做准备

#define DRM_AMDGPU_GEM_CREATE_PADDING 4

struct drm_amdgpu_gem_create_in {
    __u64   bo_size;            // BO 大小 (字节)
    __u64   alignment;          // 对齐要求
    __u32   domains;            // 内存域 (VRAM/GTT/CPU)
    __u32   flags;              // 创建标志
    __u32   handle;             // 输出: GEM handle
    __u32   bo_type;            // BO 类型
    __u32   reserved[DRM_AMDGPU_GEM_CREATE_PADDING];  // 预留字段
};

// 模式 2: size 字段版本检测
// 用户空间传入结构体大小，内核据此识别版本

struct drm_amdgpu_info_device {
    __u32   device_id;
    __u32   chip_class;
    __u32   pci_revision;
    __u32   asic_family;
    // ... 更多字段 ...
    
    // v2 追加字段
    __u64   gpu_core_clock;
    __u64   memory_bandwidth;
};

// 内核处理:
int amdgpu_info_ioctl(struct drm_device *dev, void *data, struct drm_file *filp)
{
    struct drm_amdgpu_info *info = data;
    
    // 通过 size 判断用户空间使用的版本
    if (info->size == sizeof(struct drm_amdgpu_info_device)) {
        // 新版结构体 - 可以访问 gpu_core_clock
    } else if (info->size >= offsetof(struct drm_amdgpu_info_device, gpu_core_clock)) {
        // 旧版结构体 - 只返回旧字段
    } else {
        return -EINVAL;
    }
}

// 模式 3: flags 字段扩展
// 使用 flags 位来控制新的行为，保留旧行为为默认

struct drm_amdgpu_gem_va {
    __u32   handle;             // GEM 对象 handle
    __u32   operation;          // MAP/UNMAP/REPLACE
    __u64   va_address;         // 虚拟地址
    __u64   offset_in_bo;       // BO 内偏移
    __u64   map_size;           // 映射大小
    __u32   flags;              // 标志位
#define AMDGPU_VM_FLAG_NEW_BEHAVIOR  (1 << 0)  // 新行为标志
#define AMDGPU_VM_FLAG_READ_ONLY     (1 << 1)  // 只读映射
#define AMDGPU_VM_FLAG_EXEC          (1 << 2)  // 可执行
    __u32   reserved;
};

// 内核必须忽略未识别的标志位:
// if (args->flags & ~AMDGPU_VM_KNOWN_FLAGS)
//     args->flags &= AMDGPU_VM_KNOWN_FLAGS;
// 不能 return -EINVAL 因为旧程序可能设置了 0

// 模式 4: 固定大小 + 扩展数据的间接方式
// 主结构体固定，通过指针指向扩展数据

struct drm_amdgpu_gem_create_ext {
    struct drm_amdgpu_gem_create_in base;  // 基础字段
    __u64   ext_data_ptr;                   // 指向扩展数据
    __u32   ext_data_size;                  // 扩展数据大小
    __u32   flags;                          // 扩展标志
};
```

### 案例分析

#### 案例一：结构体扩展导致的 ABI 破坏

**问题背景：**
在 Linux 5.10 中，`drm_amdgpu_info_device` 结构体增加了 `gpu_core_clock` 字段。开发者在结构体中间位置插入了该字段，导致后续所有字段的偏移发生变化。

**问题代码：**

```c
// Linux 5.9 (旧版)
struct drm_amdgpu_info_device {
    __u32   device_id;          // offset 0
    __u32   chip_class;         // offset 4
    __u32   pci_revision;       // offset 8
    __u32   asic_family;        // offset 12
    __u64   shadow_size;        // offset 16  ← 这里插入新字段
    __u64   gart_size;          // offset 24  ← 旧版应用中这是 offset 16
    __u32   vram_type;          // offset 32  ← 旧版应用中这是 offset 24
    // ... 后续所有字段偏移都变了
};

// Linux 5.10 (错误的新版) - 插入了 gpu_core_clock
struct drm_amdgpu_info_device {
    __u32   device_id;          // offset 0  (不变)
    __u32   chip_class;         // offset 4  (不变)
    __u32   pci_revision;       // offset 8  (不变)
    __u32   asic_family;        // offset 12 (不变)
    __u64   gpu_core_clock;     // offset 16 ← 新插入的字段
    __u64   shadow_size;        // offset 24 ← 旧: 16, 新: 24 ❌
    __u64   gart_size;          // offset 32 ← 旧: 24, 新: 32 ❌
    __u32   vram_type;          // offset 40 ← 旧: 32, 新: 40 ❌
    // ... 所有后续字段都错位了
};
```

**影响分析：**

```
旧版应用 (编译时使用 Linux 5.9 头文件):

    应用分配 sizeof = 56 字节的结构体
    设置 gart_size = 0x100000000 到 offset 24
    内核 (5.10) 在 offset 32 读取 gart_size → 读到的是 0x0 (shadow_size 的低位)
    → 内存分配错误，导致 GPU 驱动崩溃

诊断:
    1. ROCm 应用在升级内核后出现随机崩溃
    2. 只有使用较旧用户空间栈的应用受影响
    3. 新编译的应用使用新头文件，结构体大小一致
```

**修复方案：**

```c
// Linux 5.10 (正确的修复)
struct drm_amdgpu_info_device {
    __u32   device_id;          // offset 0  (不变)
    __u32   chip_class;         // offset 4  (不变)
    __u32   pci_revision;       // offset 8  (不变)
    __u32   asic_family;        // offset 12 (不变)
    __u64   shadow_size;        // offset 16 (不变)
    __u64   gart_size;          // offset 24 (不变)
    __u32   vram_type;          // offset 32 (不变)
    // ... 所有旧字段保持不变
    
    // 新字段追加到末尾
    __u64   gpu_core_clock;     // offset N  ← 追加到末尾
    __u64   memory_bandwidth;   // offset N+8
};

// 内核使用 size 检测版本:
if (info->size >= sizeof(struct drm_amdgpu_info_device)) {
    // 新用户空间，填充 gpu_core_clock
} else {
    // 旧用户空间，不填充新字段
}
```

**经验总结：**

```
结构体扩展规则:
1. 永远不要在已有字段之间插入新字段
2. 新字段只能追加到结构体末尾
3. 使用 padding/reserved 字段为未来扩展预留空间
4. 使用 size 字段检测用户空间版本
5. 旧版结构体必须保持完全不变
```

#### 案例二：flags 语义变化导致的用户空间兼容性问题

**问题背景：**
在 AMDGPU 驱动中，`DRM_AMDGPU_GEM_CREATE` 的 `flags` 字段新增了一个 `AMDGPU_GEM_CREATE_VRAM_CLEARED` 标志位，要求新创建的 VRAM BO 自动清零。但某些旧的用户空间程序恰好设置的 flags 值中该位被置位，导致意外的性能开销。

**问题分析：**

```c
// 旧版 flags 定义 (Linux 5.4)
#define AMDGPU_GEM_CREATE_CPU_ACCESS_REQUIRED  (1 << 0)
#define AMDGPU_GEM_CREATE_NO_CPU_ACCESS         (1 << 1)
#define AMDGPU_GEM_CREATE_CPU_GTT_USMC          (1 << 2)
#define AMDGPU_GEM_CREATE_VRAM_CLEARED           (1 << 3)  ← 新添加
#define AMDGPU_GEM_CREATE_SHADOW                 (1 << 4)
// ...

// 旧版用户空间程序可能设置 flags = 0x8 (bit 3)
// 在旧内核中，该位被忽略
// 在新内核中，该位触发 VRAM 清零操作

// 正确的做法:
// 新标志位必须默认关闭
// 内核必须忽略未识别的标志位

int amdgpu_gem_create_ioctl(struct drm_device *dev, void *data,
                            struct drm_file *filp)
{
    struct drm_amdgpu_gem_create_in *args = data;
    
    // 内核知道所有合法的标志位
    #define AMDGPU_GEM_CREATE_KNOWN_FLAGS \
        (AMDGPU_GEM_CREATE_CPU_ACCESS_REQUIRED | \
         AMDGPU_GEM_CREATE_NO_CPU_ACCESS | \
         AMDGPU_GEM_CREATE_CPU_GTT_USMC | \
         AMDGPU_GEM_CREATE_VRAM_CLEARED | \
         AMDGPU_GEM_CREATE_SHADOW)
    
    // 忽略未识别的标志位 - 不能返回错误
    args->flags &= AMDGPU_GEM_CREATE_KNOWN_FLAGS;
    
    // ... 正常处理
}
```

**根本原因：**

```
问题:
- 添加新标志位时，旧用户空间程序可能会"意外"设置该位
- 因为旧程序不知道新标志位的存在，该位的值可能是随机的
- 这是编译器未初始化内存的结果

正确做法:
1. flags 扩展时，新位必须表示"新增功能"
2. 未设置的位 = 旧行为
3. 内核必须用 KNOWN_FLAGS mask 清除未知位
4. 不能: 新位表示"关闭某个功能" (旧程序不会设置它，导致功能异常关闭)
```

### 相关链接

- [Linux 内核 ABI 稳定性规则](https://www.kernel.org/doc/html/latest/process/stable-api-nonsense.html)
- [DRM uAPI 添加指南](https://dri.freedesktop.org/docs/drm/gpu/drm-uapi.html)
- [ioctl 命令码规范](https://www.kernel.org/doc/html/latest/userspace-api/ioctl/ioctl-number.html)
- [Linux 内核稳定 API 承诺](https://www.kernel.org/doc/Documentation/process/stable-api-nonsense.rst)
- [DRM 维护者关于 uAPI 稳定性的讨论](https://lwn.net/Articles/283700/)

### 今日小结

GPU 驱动 ABI 稳定性是内核开发和用户空间兼容性的核心要求：

1. **ABI vs API**: ABI 是二进制接口，破坏后需要重新编译用户空间程序
2. **永久支持原则**: 一旦合并发布的 uAPI 必须永久支持
3. **扩展规则**: 结构体追加末尾、新枚举追加、新标志位必须可忽略
4. **ioctl 规范**: 唯一命令码、方向编码、大小编码
5. **KMS 属性**: 名称和值含义不能改变
6. **兼容性测试**: 结构体大小检查、flags 语义检查、sysfs 接口检查
7. **常见错误**: 结构体中间插入字段、flags 位语义反转

### 扩展思考

1. 如果必须破坏 ABI (例如安全原因)，社区中通常如何管理和沟通这个变更？
2. 如何通过自动化测试来保证 ABI 兼容性？是否可以在 CI 中集成 ABI 检查？
3. Mesa 和 ROCm 如何与内核 uAPI 版本同步？它们如何处理新旧内核的差异？
4. 在内核支持新的硬件特性时，如何设计 uAPI 以兼顾未来 5-10 年的扩展需求？
