# IGT 文件描述符管理：drm_open_driver

## 学习目标

- 理解 IGT 中 DRM 设备打开和管理机制
- 掌握 drm_open_driver 和相关函数的用法
- 学会处理多 GPU 场景下的设备选择
- 了解 DRM 设备权限和访问控制
- 能够正确管理 DRM 文件描述符生命周期

## 知识详解

### 一、概念原理

#### 1.1 DRM 设备打开流程

```
┌─────────────────────────────────────────────────────────────────────┐
│              DRM 设备打开流程                                        │
│                                                                     │
│  drm_open_driver(driver_filter)                                     │
│      │                                                              │
│      ▼                                                              │
│  ┌─────────────┐                                                    │
│  │ 扫描 /dev/dri/                                                    │
│  │ 查找匹配的 card* 设备                                             │
│  └──────┬──────┘                                                    │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────┐                                                    │
│  │ 检查驱动类型                                                      │
│  │ (DRIVER_AMDGPU/DRIVER_INTEL/...)                                  │
│  └──────┬──────┘                                                    │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────┐                                                    │
│  │ 打开设备 (open)                                                   │
│  │ 设置主设备权限 (setmaster)                                        │
│  └──────┬──────┘                                                    │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────┐                                                    │
│  │ 返回文件描述符                                                    │
│  └─────────────┘                                                    │
│                                                                     │
│  驱动过滤器：                                                        │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  DRIVER_ANY        任何 DRM 驱动                        │       │
│  │  DRIVER_AMDGPU     AMD GPU 驱动                         │       │
│  │  DRIVER_INTEL      Intel GPU 驱动                       │       │
│  │  DRIVER_I915       Intel i915 驱动（旧）                │       │
│  │  DRIVER_RADEON     AMD Radeon 驱动（旧）                │       │
│  │  DRIVER_NOUVEAU    NVIDIA Nouveau 驱动                  │       │
│  │  DRIVER_VC4        Raspberry Pi VC4 驱动                │       │
│  │  DRIVER_V3D        Raspberry Pi V3D 驱动                │       │
│  │  DRIVER_VGEM       虚拟 GEM 驱动（测试用）              │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.2 设备管理函数

```
┌─────────────────────────────────────────────────────────────────────┐
│              IGT DRM 设备管理函数                                    │
│                                                                     │
│  函数                              功能                             │
│  ─────────────────────────────────────────────────────────────      │
│                                                                     │
│  drm_open_driver(filter)           打开匹配的 DRM 设备              │
│  drm_open_driver_render(filter)    打开 render 节点（非主设备）     │
│  drm_open_any()                    打开任何可用的 DRM 设备          │
│  drm_open_any_render()             打开任何可用的 render 节点       │
│                                                                     │
│  drm_close_driver(fd)              关闭 DRM 设备                    │
│  drm_set_master(fd)                设置 DRM Master 权限             │
│  drm_drop_master(fd)               放弃 DRM Master 权限             │
│                                                                     │
│  is_amdgpu_device(fd)              检查是否为 AMDGPU                │
│  is_i915_device(fd)                检查是否为 Intel i915            │
│  is_vc4_device(fd)                 检查是否为 VC4                   │
│                                                                     │
│  __drm_open_driver_helper()        内部辅助函数                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 二、实践操作

#### 2.1 设备打开示例

```c
/*
 * DRM 设备打开和管理示例
 */

#include "igt.h"

static int drm_fd = -1;

igt_fixture {
    /* 方法 1：打开特定驱动 */
    drm_fd = drm_open_driver(DRIVER_AMDGPU);
    igt_require_f(drm_fd >= 0, "AMDGPU driver not found\n");
    
    /* 方法 2：打开任何可用驱动 */
    /* drm_fd = drm_open_any(); */
    
    /* 方法 3：打开 render 节点（不需要 master 权限） */
    /* drm_fd = drm_open_driver_render(DRIVER_AMDGPU); */
    
    igt_info("Opened DRM device: fd=%d\n", drm_fd);
}

/* 验证驱动类型 */
igt_subtest("check-driver") {
    igt_assert(is_amdgpu_device(drm_fd));
    
    /* 获取驱动版本 */
    struct drm_version version;
    char name[128] = {};
    char date[128] = {};
    char desc[256] = {};
    
    memset(&version, 0, sizeof(version));
    version.name = name;
    version.name_len = sizeof(name);
    version.date = date;
    version.date_len = sizeof(date);
    version.desc = desc;
    version.desc_len = sizeof(desc);
    
    igt_assert_eq(drmIoctl(drm_fd, DRM_IOCTL_VERSION, &version), 0);
    
    igt_info("Driver: %s\n", version.name);
    igt_info("Version: %d.%d.%d\n",
             version.version_major,
             version.version_minor,
             version.version_patchlevel);
    igt_info("Date: %s\n", version.date);
    igt_info("Description: %s\n", version.desc);
}

/* 多设备测试 */
igt_subtest("multi-device") {
    int fd1, fd2;
    
    /* 打开第一个设备 */
    fd1 = drm_open_driver(DRIVER_AMDGPU);
    igt_require(fd1 >= 0);
    
    /* 尝试打开第二个设备 */
    fd2 = drm_open_driver(DRIVER_AMDGPU);
    
    if (fd2 >= 0) {
        igt_info("Found 2 AMD GPUs\n");
        
        /* 比较设备信息 */
        struct drm_version v1, v2;
        char name1[128], name2[128];
        
        memset(&v1, 0, sizeof(v1));
        v1.name = name1;
        v1.name_len = sizeof(name1);
        drmIoctl(fd1, DRM_IOCTL_VERSION, &v1);
        
        memset(&v2, 0, sizeof(v2));
        v2.name = name2;
        v2.name_len = sizeof(name2);
        drmIoctl(fd2, DRM_IOCTL_VERSION, &v2);
        
        igt_info("GPU 1: %s\n", v1.name);
        igt_info("GPU 2: %s\n", v2.name);
        
        close(fd2);
    } else {
        igt_info("Only 1 AMD GPU found\n");
    }
    
    close(fd1);
}

igt_fixture {
    if (drm_fd >= 0) {
        close(drm_fd);
        drm_fd = -1;
    }
}
```

#### 2.2 设备权限管理

```bash
#!/bin/bash
# check_drm_permissions.sh - 检查 DRM 设备权限

set -e

echo "=== DRM 设备权限检查 ==="
echo ""

# 列出 DRM 设备
echo "【1】DRM 设备列表"
ls -la /dev/dri/ | grep "card\|render" || true
echo ""

# 检查当前用户权限
echo "【2】当前用户权限"
echo "用户: $(whoami)"
echo "组: $(groups)"
echo ""

# 检查 video 组
echo "【3】video 组成员"
getent group video || echo "video 组不存在"
echo ""

# 检查 render 组
echo "【4】render 组成员"
getent group render || echo "render 组不存在"
echo ""

# 测试打开设备
echo "【5】测试打开 DRM 设备"
for dev in /dev/dri/card*; do
    if [ -c "$dev" ]; then
        if python3 -c "import os; fd = os.open('$dev', os.O_RDWR); os.close(fd)" 2>/dev/null; then
            echo "  ✓ $dev: 可读写"
        else
            echo "  ✗ $dev: 无权限"
        fi
    fi
done
echo ""
```

### 三、案例分析

#### 3.1 实际测试中的设备管理

```c
/*
 * tests/amdgpu/amd_basic.c 设备管理分析
 */

#include "igt.h"
#include <amdgpu.h>

IGT_TEST_DESCRIPTION("Basic AMDGPU tests");

static int drm_fd;
static amdgpu_device_handle amdgpu_device;

igt_fixture {
    /* 明确要求 AMDGPU 驱动 */
    drm_fd = drm_open_driver(DRIVER_AMDGPU);
    igt_require_f(drm_fd >= 0,
                  "No AMDGPU device found. Skipping test.\n");
    
    /* 初始化 AMDGPU 用户空间库 */
    uint32_t major, minor;
    int ret = amdgpu_device_initialize(drm_fd, &major, &minor,
                                       &amdgpu_device);
    igt_require_f(ret == 0,
                  "Failed to initialize AMDGPU device: %d\n", ret);
    
    igt_info("AMDGPU device initialized (version %d.%d)\n",
             major, minor);
}

/* 测试完成后清理 */
igt_fixture {
    if (amdgpu_device) {
        amdgpu_device_deinitialize(amdgpu_device);
        amdgpu_device = NULL;
    }
    
    if (drm_fd >= 0) {
        close(drm_fd);
        drm_fd = -1;
    }
}
```

### 四、相关链接

- IGT DRM 辅助文档：https://drm.pages.freedesktop.org/igt-gpu-tools/igt-gpu-tools-DRM.html
- DRM 设备节点：https://dri.freedesktop.org/docs/drm/gpu/drm-uapi.html#primary-nodes

## 今日小结

- drm_open_driver 用于打开指定类型的 DRM 设备
- DRIVER_AMDGPU、DRIVER_INTEL 等过滤器限制驱动类型
- drm_open_driver_render 打开 render 节点，不需要 master 权限
- 设备打开后需要验证驱动类型和版本
- 多 GPU 系统需要处理多个设备文件

## 扩展思考

1. 在多 GPU 系统中，如何设计测试以覆盖所有 GPU？是否需要并行测试？
2. render 节点和 primary 节点有什么区别？什么场景下应该使用 render 节点？
