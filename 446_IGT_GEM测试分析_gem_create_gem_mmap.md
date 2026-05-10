# IGT GEM 测试分析：gem_create / gem_mmap

## 学习目标

- 深入理解 IGT GEM（Graphics Execution Manager）测试的设计和实现
- 掌握 gem_create 和 gem_mmap 测试的功能和原理
- 学会分析 GEM 测试代码并理解其验证点
- 了解 GEM 对象的生命周期管理
- 能够编写类似的 GEM 功能测试

## 知识详解

### 一、概念原理

#### 1.1 GEM 概述

```
┌─────────────────────────────────────────────────────────────────────┐
│              GEM（Graphics Execution Manager）                       │
│                                                                     │
│  GEM 是 Linux DRM 子系统的内存管理框架，用于管理 GPU 图形缓冲区：    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  GEM 对象（Buffer Object）                                │       │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐     │       │
│  │  │ 创建    │  │ 映射    │  │ 使用    │  │ 销毁    │     │       │
│  │  │ CREATE  │  │ MMAP    │  │ EXEC    │  │ CLOSE   │     │       │
│  │  │         │  │         │  │         │  │         │     │       │
│  │  │ 分配    │  │ 用户空间│  │ 提交    │  │ 释放    │     │       │
│  │  │ 内存    │  │ 访问    │  │ 命令    │  │ 资源    │     │       │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘     │       │
│  │                                                         │       │
│  │  内存域：                                                │       │
│  │  ● CPU/GTT（Graphics Translation Table）                │       │
│  │  ● VRAM（Video RAM）                                    │       │
│  │  ● CPU 可访问 / GPU 可访问                              │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
│  核心 ioctl：                                                        │
│  ● DRM_IOCTL_GEM_CLOSE                                              │
│  ● DRM_IOCTL_GEM_FLINK                                              │
│  ● DRM_IOCTL_GEM_OPEN                                               │
│  ● DRM_IOCTL_I915_GEM_CREATE（Intel 专用）                          │
│  ● DRM_IOCTL_AMDGPU_GEM_CREATE（AMD 专用）                          │
│  ● DRM_IOCTL_MODE_CREATE_DUMB（通用 dumb buffer）                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.2 gem_create 测试分析

```
┌─────────────────────────────────────────────────────────────────────┐
│              gem_create 测试分析                                     │
│                                                                     │
│  测试文件：tests/gem_create.c                                       │
│                                                                     │
│  子测试列表：                                                        │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  子测试名称          验证内容                            │       │
│  │  ─────────────────────────────────────────────────────   │       │
│  │  basic               基本创建和关闭                      │       │
│  │  large               大 BO 创建（接近显存上限）          │       │
│  │  invalid-size        无效大小参数（0、超大值）           │       │
│  │  invalid-flags       无效标志                            │       │
│  │  create-close-race   并发创建/关闭竞争条件               │       │
│  │  reuse               句柄重用验证                        │       │
│  │  flink               全局名称共享                        │       │
│  │  open                通过名称打开                        │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
│  核心验证点：                                                        │
│  1. BO 创建成功返回有效句柄                                         │
│  2. 不同大小的 BO 都能正确创建                                      │
│  3. 无效参数返回适当错误码                                          │
│  4. 并发操作不会导致资源泄露                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.3 gem_mmap 测试分析

```
┌─────────────────────────────────────────────────────────────────────┐
│              gem_mmap 测试分析                                       │
│                                                                     │
│  测试文件：tests/gem_mmap.c                                         │
│                                                                     │
│  子测试列表：                                                        │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  子测试名称          验证内容                            │       │
│  │  ─────────────────────────────────────────────────────   │       │
│  │  basic               基本映射和解除映射                  │       │
│  │  bad-offset          无效偏移                            │       │
│  │  bad-size            无效大小                            │       │
│  │  bad-flag            无效标志                            │       │
│  │  copy                通过映射复制数据                    │       │
│  │  write               通过映射写入数据                    │       │
│  │  read                通过映射读取数据                    │       │
│  │  prefetch            预取测试                            │       │
│  │  coherency           一致性测试                          │       │
│  │  gtt                 GTT 映射测试                        │       │
│  │  wc                  写合并映射测试                      │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
│  核心验证点：                                                        │
│  1. MMAP 成功返回有效指针                                           │
│  2. 通过映射读写数据正确                                            │
│  3. 不同映射类型（GTT/WC）行为正确                                  │
│  4. 解除映射后访问不导致崩溃                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 二、实践操作

#### 2.1 gem_create 测试代码分析

```c
/*
 * tests/gem_create.c 简化分析
 */

#include "igt.h"

IGT_TEST_DESCRIPTION("Basic GEM buffer creation tests");

static int drm_fd;

igt_fixture {
    drm_fd = drm_open_driver(DRIVER_ANY);
    igt_require(drm_fd >= 0);
}

/* 基本创建测试 */
igt_subtest("basic") {
    struct drm_mode_create_dumb create;
    struct drm_mode_destroy_dumb destroy;
    
    /* 创建 1024x768x32bpp 的 dumb buffer */
    memset(&create, 0, sizeof(create));
    create.width = 1024;
    create.height = 768;
    create.bpp = 32;
    
    /* 执行创建 ioctl */
    do_ioctl(drm_fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);
    
    /* 验证结果 */
    igt_assert(create.handle != 0);      /* 句柄有效 */
    igt_assert(create.pitch > 0);        /* pitch 有效 */
    igt_assert(create.size > 0);         /* 大小有效 */
    
    igt_info("Created BO: handle=%u, pitch=%u, size=%llu\n",
             create.handle, create.pitch, create.size);
    
    /* 销毁 BO */
    memset(&destroy, 0, sizeof(destroy));
    destroy.handle = create.handle;
    do_ioctl(drm_fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
}

/* 大 BO 测试 */
igt_subtest("large") {
    struct drm_mode_create_dumb create;
    struct drm_mode_destroy_dumb destroy;
    
    /* 尝试创建 8192x8192x32bpp = 256MB */
    memset(&create, 0, sizeof(create));
    create.width = 8192;
    create.height = 8192;
    create.bpp = 32;
    
    do_ioctl(drm_fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);
    
    igt_assert(create.handle != 0);
    igt_info("Large BO: handle=%u, size=%llu\n",
             create.handle, create.size);
    
    destroy.handle = create.handle;
    do_ioctl(drm_fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
}

/* 无效参数测试 */
igt_subtest("invalid-size") {
    struct drm_mode_create_dumb create;
    int ret;
    
    /* 测试零大小 */
    memset(&create, 0, sizeof(create));
    create.width = 0;
    create.height = 0;
    create.bpp = 32;
    
    ret = drmIoctl(drm_fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);
    igt_assert_eq(ret, -1);
    igt_assert_eq(errno, EINVAL);
    
    /* 测试超大大小 */
    memset(&create, 0, sizeof(create));
    create.width = ~0;
    create.height = ~0;
    create.bpp = 32;
    
    ret = drmIoctl(drm_fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);
    igt_assert_eq(ret, -1);
}

igt_fixture {
    close(drm_fd);
}
```

#### 2.2 gem_mmap 测试代码分析

```c
/*
 * tests/gem_mmap.c 简化分析
 */

#include "igt.h"

IGT_TEST_DESCRIPTION("GEM buffer mmap tests");

static int drm_fd;

igt_fixture {
    drm_fd = drm_open_driver(DRIVER_ANY);
    igt_require(drm_fd >= 0);
}

/* 基本映射测试 */
igt_subtest("basic") {
    struct drm_mode_create_dumb create;
    struct drm_mode_map_dumb map;
    struct drm_mode_destroy_dumb destroy;
    void *ptr;
    
    /* 创建 BO */
    memset(&create, 0, sizeof(create));
    create.width = 1024;
    create.height = 768;
    create.bpp = 32;
    do_ioctl(drm_fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);
    
    /* 准备映射 */
    memset(&map, 0, sizeof(map));
    map.handle = create.handle;
    do_ioctl(drm_fd, DRM_IOCTL_MODE_MAP_DUMB, &map);
    
    /* 执行映射 */
    ptr = mmap(0, create.size, PROT_READ | PROT_WRITE, MAP_SHARED,
               drm_fd, map.offset);
    igt_assert(ptr != MAP_FAILED);
    
    igt_info("Mapped BO at %p, size=%llu\n", ptr, create.size);
    
    /* 解除映射 */
    igt_assert_eq(munmap(ptr, create.size), 0);
    
    /* 销毁 BO */
    destroy.handle = create.handle;
    do_ioctl(drm_fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
}

/* 数据读写测试 */
igt_subtest("read-write") {
    struct drm_mode_create_dumb create;
    struct drm_mode_map_dumb map;
    struct drm_mode_destroy_dumb destroy;
    void *ptr;
    uint32_t *data;
    
    /* 创建 BO */
    memset(&create, 0, sizeof(create));
    create.width = 256;
    create.height = 256;
    create.bpp = 32;
    do_ioctl(drm_fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);
    
    /* 映射 BO */
    memset(&map, 0, sizeof(map));
    map.handle = create.handle;
    do_ioctl(drm_fd, DRM_IOCTL_MODE_MAP_DUMB, &map);
    
    ptr = mmap(0, create.size, PROT_READ | PROT_WRITE, MAP_SHARED,
               drm_fd, map.offset);
    igt_assert(ptr != MAP_FAILED);
    
    /* 写入测试模式 */
    data = ptr;
    for (int i = 0; i < create.size / sizeof(uint32_t); i++) {
        data[i] = i;
    }
    
    /* 验证读取 */
    for (int i = 0; i < create.size / sizeof(uint32_t); i++) {
        igt_assert_f(data[i] == i,
                     "Data mismatch at offset %d: expected %u, got %u\n",
                     i, i, data[i]);
    }
    
    /* 清理 */
    munmap(ptr, create.size);
    destroy.handle = create.handle;
    do_ioctl(drm_fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
}

igt_fixture {
    close(drm_fd);
}
```

#### 2.3 运行 GEM 测试

```bash
#!/bin/bash
# run_gem_tests.sh - 运行 GEM 测试

set -e

IGT_BUILD="${1:-$HOME/igt-gpu-tools/build}"

echo "=== 运行 GEM 测试 ==="
echo ""

# 运行 gem_create
echo "【1】gem_create 测试"
sudo "$IGT_BUILD/tests/gem_create" 2>&1 | tee gem_create.log
echo ""

# 运行 gem_mmap
echo "【2】gem_mmap 测试"
sudo "$IGT_BUILD/tests/gem_mmap" 2>&1 | tee gem_mmap.log
echo ""

# 运行指定子测试
echo "【3】gem_create basic 子测试"
sudo "$IGT_BUILD/tests/gem_create" --run-subtest basic 2>&1 | tee gem_create_basic.log
echo ""

echo "=== GEM 测试完成 ==="
```

### 三、案例分析

#### 3.1 GEM 对象生命周期

```
┌─────────────────────────────────────────────────────────────────────┐
│              GEM 对象生命周期                                        │
│                                                                     │
│   创建                                                                │
│     │                                                                │
│     ▼                                                                │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐           │
│  │ CREATE_DUMB │────▶│ MAP_DUMB    │────▶│ mmap()      │           │
│  │             │     │             │     │             │           │
│  │ 分配内存    │     │ 获取偏移    │     │ 映射到进程  │           │
│  │ 返回 handle │     │ 返回 offset │     │ 返回 ptr    │           │
│  └─────────────┘     └─────────────┘     └──────┬──────┘           │
│                                                  │                  │
│                                                  ▼                  │
│                                           ┌─────────────┐          │
│                                           │ 读写数据    │          │
│                                           │             │          │
│                                           └──────┬──────┘          │
│                                                  │                  │
│                                                  ▼                  │
│  销毁                                     ┌─────────────┐          │
│     │                                     │ munmap()    │          │
│     ▼                                     │             │          │
│  ┌─────────────┐                          │ 解除映射    │          │
│  │ DESTROY_DUMB│◀─────────────────────────└─────────────┘          │
│  │             │                                                     │
│  │ 释放内存    │                                                     │
│  │ 回收 handle │                                                     │
│  └─────────────┘                                                     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 四、相关链接

- IGT GEM 测试：https://gitlab.freedesktop.org/drm/igt-gpu-tools/-/tree/master/tests
- GEM 文档：https://dri.freedesktop.org/docs/drm/gpu/drm-mm.html#gem-buffer-objects
- DRM Dumb Buffer：https://dri.freedesktop.org/docs/drm/gpu/drm-kms.html#dumb-buffer-objects

## 今日小结

- GEM 是 DRM 子系统的内存管理框架，管理 GPU 图形缓冲区
- gem_create 测试验证 BO 的创建、销毁和参数检查
- gem_mmap 测试验证 BO 的内存映射和数据访问
- GEM 对象生命周期包括创建、映射、使用和销毁四个阶段
- 测试应覆盖正常路径、错误路径和边界条件

## 扩展思考

1. GEM 和 TTM（Translation Table Manager）有什么关系？在 AMDGPU 驱动中它们如何协作？
2. 如何设计一个测试来验证 GEM 对象的内存域迁移（如 GTT -> VRAM）？
