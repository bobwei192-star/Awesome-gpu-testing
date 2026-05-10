# IGT 测试编写基础：igt_main / igt_subtest

## 学习目标

- 掌握 IGT 测试的基本结构和编写规范
- 理解 igt_main、igt_subtest、igt_fixture 的用法
- 学会组织测试用例和子测试
- 了解 IGT 测试的编译和注册机制
- 能够编写简单的 IGT 测试程序

## 知识详解

### 一、概念原理

#### 1.1 测试结构

```
┌─────────────────────────────────────────────────────────────────────┐
│              IGT 测试程序结构                                        │
│                                                                     │
│  #include "igt.h"                                                   │
│                                                                     │
│  /* 测试描述 */                                                      │
│  IGT_TEST_DESCRIPTION("测试功能描述");                               │
│                                                                     │
│  /* 全局变量和静态函数 */                                             │
│  static int drm_fd;                                                 │
│                                                                     │
│  /* 辅助函数 */                                                       │
│  static void helper_function(void) { ... }                          │
│                                                                     │
│  /* 测试夹具：设置 */                                                 │
│  igt_fixture {                                                      │
│      /* 每个子测试前执行的准备代码 */                                 │
│      drm_fd = drm_open_driver(DRIVER_AMDGPU);                       │
│      igt_require(drm_fd >= 0);                                      │
│  }                                                                  │
│                                                                     │
│  /* 子测试 1 */                                                       │
│  igt_subtest("subtest-name-1") {                                    │
│      /* 测试代码 */                                                  │
│      igt_assert(some_condition());                                  │
│  }                                                                  │
│                                                                     │
│  /* 子测试 2 */                                                       │
│  igt_subtest("subtest-name-2") {                                    │
│      /* 测试代码 */                                                  │
│      igt_assert(another_condition());                               │
│  }                                                                  │
│                                                                     │
│  /* 测试夹具：清理 */                                                 │
│  igt_fixture {                                                      │
│      /* 每个子测试后执行的清理代码 */                                 │
│      close(drm_fd);                                                 │
│  }                                                                  │
│                                                                     │
│  /* 测试入口 */                                                       │
│  igt_main                                                           │
│  {                                                                  │
│      /* 主测试逻辑由 igt_main 宏展开 */                              │
│  }                                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.2 宏展开原理

```
┌─────────────────────────────────────────────────────────────────────┐
│              igt_main / igt_subtest 宏展开                           │
│                                                                     │
│  源码：                                                              │
│  igt_main {                                                         │
│      igt_fixture { setup(); }                                       │
│      igt_subtest("name") { test(); }                                │
│      igt_fixture { teardown(); }                                    │
│  }                                                                  │
│                                                                     │
│  展开后（简化）：                                                     │
│  int main(int argc, char **argv) {                                  │
│      /* 初始化 IGT 框架 */                                           │
│      igt_init(&argc, argv);                                         │
│                                                                     │
│      /* 注册夹具 */                                                  │
│      igt_register_fixture(setup_fn, NULL);                          │
│                                                                     │
│      /* 注册子测试 */                                                │
│      igt_register_subtest("name", test_fn);                         │
│                                                                     │
│      /* 注册夹具 */                                                  │
│      igt_register_fixture(NULL, teardown_fn);                       │
│                                                                     │
│      /* 执行所有子测试 */                                            │
│      return igt_run_subtests();                                     │
│  }                                                                  │
│                                                                     │
│  执行流程：                                                          │
│  for (each subtest) {                                               │
│      setup();                                                       │
│      subtest();                                                     │
│      teardown();                                                    │
│  }                                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 二、实践操作

#### 2.1 编写第一个 IGT 测试

```c
/*
 * tests/my_first_test.c
 * 我的第一个 IGT 测试
 */

#include "igt.h"
#include <fcntl.h>
#include <unistd.h>

IGT_TEST_DESCRIPTION("My first IGT test - basic DRM device operations");

static int drm_fd = -1;

/*
 * 测试夹具：设置
 * 每个子测试前执行
 */
igt_fixture {
    /* 打开 DRM 设备 */
    drm_fd = drm_open_driver(DRIVER_AMDGPU | DRIVER_INTEL);
    
    /* 要求设备打开成功，否则跳过测试 */
    igt_require(drm_fd >= 0);
    
    igt_info("DRM device opened: fd=%d\n", drm_fd);
}

/*
 * 子测试 1：验证设备信息
 */
igt_subtest("device-info") {
    struct drm_version version;
    char name[128] = {};
    
    memset(&version, 0, sizeof(version));
    version.name = name;
    version.name_len = sizeof(name);
    
    /* 获取驱动版本 */
    do_ioctl(drm_fd, DRM_IOCTL_VERSION, &version);
    
    igt_info("Driver: %s, Version: %d.%d.%d\n",
             version.name,
             version.version_major,
             version.version_minor,
             version.version_patchlevel);
    
    /* 验证驱动名称非空 */
    igt_assert(strlen(version.name) > 0);
}

/*
 * 子测试 2：验证 GEM 创建
 */
igt_subtest("gem-create") {
    struct drm_mode_create_dumb create;
    
    memset(&create, 0, sizeof(create));
    create.width = 1024;
    create.height = 768;
    create.bpp = 32;
    
    /* 创建 dumb buffer */
    do_ioctl(drm_fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);
    
    igt_info("GEM handle: %u, pitch: %u, size: %llu\n",
             create.handle, create.pitch, create.size);
    
    /* 验证句柄有效 */
    igt_assert(create.handle != 0);
    igt_assert(create.size > 0);
    
    /* 清理 */
    struct drm_mode_destroy_dumb destroy = {
        .handle = create.handle
    };
    do_ioctl(drm_fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
}

/*
 * 子测试 3：无效参数测试
 */
igt_subtest("gem-create-invalid") {
    struct drm_mode_create_dumb create;
    int ret;
    
    /* 测试零大小 */
    memset(&create, 0, sizeof(create));
    create.width = 0;
    create.height = 0;
    create.bpp = 32;
    
    ret = drmIoctl(drm_fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);
    
    /* 期望失败 */
    igt_assert_eq(ret, -1);
    igt_assert_eq(errno, EINVAL);
}

/*
 * 测试夹具：清理
 * 每个子测试后执行
 */
igt_fixture {
    if (drm_fd >= 0) {
        close(drm_fd);
        drm_fd = -1;
    }
    igt_info("DRM device closed\n");
}
```

#### 2.2 编译新测试

```bash
#!/bin/bash
# build_custom_test.sh - 编译自定义 IGT 测试

set -e

IGT_DIR="${1:-$HOME/igt-gpu-tools}"
TEST_FILE="$2"

if [ -z "$TEST_FILE" ]; then
    echo "用法: $0 <igt-dir> <test-file.c>"
    exit 1
fi

echo "=== 编译自定义 IGT 测试 ==="
echo "测试文件: $TEST_FILE"
echo ""

# 复制到 IGT 测试目录
TEST_NAME=$(basename "$TEST_FILE" .c)
cp "$TEST_FILE" "$IGT_DIR/tests/"

# 更新 meson.build
cd "$IGT_DIR"

# 添加测试到构建（手动编辑 meson.build）
echo "请手动编辑 tests/meson.build，添加:"
echo "  test_programs += [ '$TEST_NAME' ]"
echo ""

# 重新配置和编译
meson setup build --wipe
ninja -C build

echo ""
echo "编译完成: $IGT_DIR/build/tests/$TEST_NAME"
```

#### 2.3 运行和验证

```bash
#!/bin/bash
# run_custom_test.sh - 运行自定义测试

set -e

TEST_BIN="$1"

if [ -z "$TEST_BIN" ]; then
    echo "用法: $0 <test-binary>"
    exit 1
fi

echo "=== 运行自定义 IGT 测试 ==="
echo "测试: $TEST_BIN"
echo ""

# 列出子测试
echo "【1】列出子测试"
"$TEST_BIN" --list-subtests
echo ""

# 运行所有子测试
echo "【2】运行所有子测试"
sudo "$TEST_BIN" 2>&1 | tee "${TEST_BIN}.log"
echo ""

# 运行指定子测试
echo "【3】运行指定子测试"
sudo "$TEST_BIN" --run-subtest "gem-create" 2>&1 | tee "${TEST_BIN}_gem.log"
echo ""

# 启用调试输出
echo "【4】启用调试输出"
sudo IGT_DEBUG=1 "$TEST_BIN" --run-subtest "device-info" 2>&1 | tee "${TEST_BIN}_debug.log"
echo ""
```

### 三、案例分析

#### 3.1 完整测试示例分析

```c
/*
 * tests/amdgpu/amd_basic.c 简化分析
 * 参考：IGT 官方 amdgpu 测试
 */

#include "igt.h"
#include <amdgpu.h>
#include <amdgpu_drm.h>

IGT_TEST_DESCRIPTION("Basic AMDGPU tests");

static int drm_fd;
static uint32_t major_version;
static uint32_t minor_version;

igt_fixture {
    /* 只支持 AMDGPU */
    drm_fd = drm_open_driver(DRIVER_AMDGPU);
    igt_require(drm_fd >= 0);
    
    /* 获取 AMDGPU 版本 */
    amdgpu_device_handle device;
    igt_require(amdgpu_device_initialize(drm_fd, &major_version,
                                          &minor_version, &device) == 0);
    
    igt_info("AMDGPU version: %d.%d\n", major_version, minor_version);
}

/* 验证设备信息 */
igt_subtest("device-info") {
    amdgpu_device_handle device;
    struct drm_amdgpu_info_device info;
    
    amdgpu_device_initialize(drm_fd, &major_version, &minor_version, &device);
    
    /* 查询设备信息 */
    memset(&info, 0, sizeof(info));
    amdgpu_query_info(device, AMDGPU_INFO_DEV_INFO,
                      sizeof(info), &info);
    
    igt_info("Family: %d, Chip rev: %d\n", info.family, info.chip_rev);
    
    /* 验证基本信息 */
    igt_assert(info.family > 0);
    igt_assert(info.num_shader_engines > 0);
    
    amdgpu_device_deinitialize(device);
}

/* 验证 GEM 创建 */
igt_subtest("gem-create") {
    amdgpu_device_handle device;
    amdgpu_bo_handle bo;
    amdgpu_bo_alloc_req req;
    amdgpu_bo_alloc_result res;
    
    amdgpu_device_initialize(drm_fd, &major_version, &minor_version, &device);
    
    /* 分配 BO */
    memset(&req, 0, sizeof(req));
    req.alloc_size = 4096;
    req.phys_alignment = 4096;
    req.preferred_heap = AMDGPU_GEM_DOMAIN_GTT;
    
    igt_assert_eq(amdgpu_bo_alloc(device, &req, &bo), 0);
    
    /* 查询分配结果 */
    memset(&res, 0, sizeof(res));
    igt_assert_eq(amdgpu_bo_query_info(bo, &res), 0);
    
    igt_info("BO size: %lu, domain: %d\n", res.alloc_size, res.preferred_heap);
    
    /* 清理 */
    amdgpu_bo_free(bo);
    amdgpu_device_deinitialize(device);
}

igt_fixture {
    close(drm_fd);
}
```

### 四、相关链接

- IGT 测试编写指南：https://drm.pages.freedesktop.org/igt-gpu-tools/igt-gpu-tools-TestWriting.html
- IGT API 参考：https://drm.pages.freedesktop.org/igt-gpu-tools/igt-gpu-tools-CoreAPI.html
- AMDGPU 用户空间 API：https://dri.freedesktop.org/docs/drm/gpu/amdgpu.html#userspace-apis

## 今日小结

- IGT 测试使用 igt_main、igt_subtest、igt_fixture 宏定义结构
- igt_fixture 用于设置和清理，每个子测试前后执行
- igt_require 用于检查前置条件，不满足则跳过
- igt_assert 用于验证测试结果，失败则标记 FAIL
- 自定义测试需要编译到 IGT 构建系统中

## 扩展思考

1. igt_fixture 在每个子测试前后执行，这种设计有什么优缺点？如何处理昂贵的设置操作？
2. 如何设计一个支持参数化测试的 IGT 扩展？例如，使用不同大小的 BO 运行相同测试逻辑。
