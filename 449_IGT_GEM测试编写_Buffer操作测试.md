# IGT GEM 测试编写：Buffer 操作测试

## 学习目标

- 掌握编写 GEM 缓冲区操作测试的方法
- 理解 BO 生命周期管理的测试要点
- 学会设计边界条件和错误路径测试
- 了解并发和竞争条件测试的设计
- 能够编写完整的 GEM 功能测试套件

## 知识详解

### 一、概念原理

#### 1.1 BO 操作测试设计

```
┌─────────────────────────────────────────────────────────────────────┐
│              BO 操作测试设计                                         │
│                                                                     │
│  测试维度：                                                          │
│  ──────────                                                          │
│                                                                     │
│  1. 功能测试                                                         │
│     ├── 创建（不同大小、不同域）                                     │
│     ├── 映射（读写、不同标志）                                       │
│     ├── 执行（命令提交）                                             │
│     ├── 等待（忙状态、超时）                                         │
│     └── 销毁（正常、重复）                                           │
│                                                                     │
│  2. 边界测试                                                         │
│     ├── 最小大小（0、1、页大小）                                     │
│     ├── 最大大小（显存上限、整数溢出）                               │
│     ├── 对齐要求（不同对齐值）                                       │
│     └── 句柄边界（0、最大值）                                        │
│                                                                     │
│  3. 错误测试                                                         │
│     ├── 无效句柄                                                     │
│     ├── 无效大小                                                     │
│     ├── 无效标志                                                     │
│     ├── 无效偏移                                                     │
│     └── 权限错误                                                     │
│                                                                     │
│  4. 并发测试                                                         │
│     ├── 多线程创建/销毁                                              │
│     ├── 多进程访问                                                   │
│     ├── 读写竞争                                                     │
│     └── 执行竞争                                                     │
│                                                                     │
│  5. 压力测试                                                         │
│     ├── 大量 BO（内存压力）                                          │
│     ├── 长时间运行（泄露检测）                                       │
│     └── 快速创建/销毁（碎片测试）                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 二、实践操作

#### 2.1 完整 BO 操作测试

```c
/*
 * tests/amdgpu/amd_bo_operations.c
 * AMDGPU 缓冲区操作完整测试
 */

#include "igt.h"
#include <amdgpu.h>
#include <amdgpu_drm.h>

IGT_TEST_DESCRIPTION("AMDGPU buffer object operations test suite");

static int drm_fd = -1;
static amdgpu_device_handle device = NULL;

/*
 * 辅助函数：创建 BO
 */
static amdgpu_bo_handle create_bo(uint64_t size, uint32_t domain)
{
    amdgpu_bo_handle bo;
    amdgpu_bo_alloc_req req;
    int ret;
    
    memset(&req, 0, sizeof(req));
    req.alloc_size = size;
    req.phys_alignment = 4096;
    req.preferred_heap = domain;
    
    ret = amdgpu_bo_alloc(device, &req, &bo);
    igt_assert_f(ret == 0, "Failed to allocate BO: %d\n", ret);
    
    return bo;
}

/*
 * 辅助函数：验证 BO 信息
 */
static void verify_bo_info(amdgpu_bo_handle bo, uint64_t expected_size)
{
    amdgpu_bo_alloc_result res;
    int ret;
    
    memset(&res, 0, sizeof(res));
    ret = amdgpu_bo_query_info(bo, &res);
    igt_assert_eq(ret, 0);
    igt_assert_eq(res.alloc_size, expected_size);
}

/*
 * 测试夹具：初始化
 */
igt_fixture {
    uint32_t major, minor;
    int ret;
    
    drm_fd = drm_open_driver(DRIVER_AMDGPU);
    igt_require_f(drm_fd >= 0, "AMDGPU driver not available\n");
    
    ret = amdgpu_device_initialize(drm_fd, &major, &minor, &device);
    igt_require_f(ret == 0, "Failed to initialize AMDGPU: %d\n", ret);
    
    igt_info("AMDGPU %d.%d initialized\n", major, minor);
}

/*
 * 子测试 1：基本创建和销毁
 */
igt_subtest("create-destroy-basic") {
    amdgpu_bo_handle bo;
    int ret;
    
    /* 创建最小 BO */
    bo = create_bo(4096, AMDGPU_GEM_DOMAIN_GTT);
    verify_bo_info(bo, 4096);
    ret = amdgpu_bo_free(bo);
    igt_assert_eq(ret, 0);
    
    /* 创建中等 BO */
    bo = create_bo(1024 * 1024, AMDGPU_GEM_DOMAIN_GTT);
    verify_bo_info(bo, 1024 * 1024);
    ret = amdgpu_bo_free(bo);
    igt_assert_eq(ret, 0);
    
    igt_info("Basic create/destroy passed\n");
}

/*
 * 子测试 2：不同内存域
 */
igt_subtest("create-domains") {
    amdgpu_bo_handle bo;
    int ret;
    
    /* GTT 域 */
    bo = create_bo(4096, AMDGPU_GEM_DOMAIN_GTT);
    ret = amdgpu_bo_free(bo);
    igt_assert_eq(ret, 0);
    
    /* VRAM 域 */
    bo = create_bo(4096, AMDGPU_GEM_DOMAIN_VRAM);
    ret = amdgpu_bo_free(bo);
    igt_assert_eq(ret, 0);
    
    /* CPU 域 */
    bo = create_bo(4096, AMDGPU_GEM_DOMAIN_CPU);
    ret = amdgpu_bo_free(bo);
    igt_assert_eq(ret, 0);
    
    igt_info("Domain tests passed\n");
}

/*
 * 子测试 3：边界大小
 */
igt_subtest("create-boundary-sizes") {
    amdgpu_bo_handle bo;
    amdgpu_bo_alloc_req req;
    int ret;
    
    /* 测试大小数组 */
    uint64_t sizes[] = {
        1,          /* 最小 */
        4095,       /* 页大小 - 1 */
        4096,       /* 一页 */
        4097,       /* 页大小 + 1 */
        65536,      /* 16 页 */
        1024 * 1024, /* 1MB */
        16 * 1024 * 1024, /* 16MB */
    };
    
    for (int i = 0; i < ARRAY_SIZE(sizes); i++) {
        igt_info("Testing size: %lu\n", sizes[i]);
        
        bo = create_bo(sizes[i], AMDGPU_GEM_DOMAIN_GTT);
        
        /* 实际大小可能大于请求（对齐） */
        amdgpu_bo_alloc_result res;
        memset(&res, 0, sizeof(res));
        ret = amdgpu_bo_query_info(bo, &res);
        igt_assert_eq(ret, 0);
        igt_assert_f(res.alloc_size >= sizes[i],
                     "Allocated size %lu < requested %lu\n",
                     res.alloc_size, sizes[i]);
        
        ret = amdgpu_bo_free(bo);
        igt_assert_eq(ret, 0);
    }
    
    igt_info("Boundary size tests passed\n");
}

/*
 * 子测试 4：无效参数
 */
igt_subtest("create-invalid") {
    amdgpu_bo_handle bo;
    amdgpu_bo_alloc_req req;
    int ret;
    
    /* 零大小 */
    memset(&req, 0, sizeof(req));
    req.alloc_size = 0;
    req.preferred_heap = AMDGPU_GEM_DOMAIN_GTT;
    ret = amdgpu_bo_alloc(device, &req, &bo);
    igt_assert_f(ret != 0, "Zero-size allocation should fail\n");
    
    /* 无效域 */
    memset(&req, 0, sizeof(req));
    req.alloc_size = 4096;
    req.preferred_heap = 0xFFFFFFFF;  /* 无效域 */
    ret = amdgpu_bo_alloc(device, &req, &bo);
    igt_assert_f(ret != 0, "Invalid domain should fail\n");
    
    igt_info("Invalid parameter tests passed\n");
}

/*
 * 子测试 5：内存映射
 */
igt_subtest("mmap-basic") {
    amdgpu_bo_handle bo;
    void *ptr;
    int ret;
    
    /* 创建 BO */
    bo = create_bo(4096, AMDGPU_GEM_DOMAIN_GTT);
    
    /* 映射 */
    ret = amdgpu_bo_cpu_map(bo, &ptr);
    igt_assert_f(ret == 0, "Failed to map BO: %d\n", ret);
    igt_assert_neq(ptr, NULL);
    
    /* 写入 */
    memset(ptr, 0xAB, 4096);
    
    /* 验证 */
    for (int i = 0; i < 4096; i++) {
        igt_assert_f(((uint8_t *)ptr)[i] == 0xAB,
                     "Data mismatch at %d\n", i);
    }
    
    /* 解除映射 */
    ret = amdgpu_bo_cpu_unmap(bo);
    igt_assert_eq(ret, 0);
    
    /* 清理 */
    ret = amdgpu_bo_free(bo);
    igt_assert_eq(ret, 0);
    
    igt_info("Mmap test passed\n");
}

/*
 * 子测试 6：并发创建
 */
igt_subtest("concurrent-create") {
    const int num_bos = 100;
    amdgpu_bo_handle bos[num_bos];
    int ret;
    
    /* 并发创建 */
    for (int i = 0; i < num_bos; i++) {
        bos[i] = create_bo(4096, AMDGPU_GEM_DOMAIN_GTT);
    }
    
    igt_info("Created %d BOs\n", num_bos);
    
    /* 并发销毁 */
    for (int i = 0; i < num_bos; i++) {
        ret = amdgpu_bo_free(bos[i]);
        igt_assert_eq(ret, 0);
    }
    
    igt_info("Concurrent create/destroy test passed\n");
}

/*
 * 子测试 7：压力测试
 */
igt_subtest("stress-create-destroy") {
    const int iterations = 1000;
    amdgpu_bo_handle bo;
    int ret;
    
    for (int i = 0; i < iterations; i++) {
        bo = create_bo(65536, AMDGPU_GEM_DOMAIN_GTT);
        ret = amdgpu_bo_free(bo);
        igt_assert_eq(ret, 0);
    }
    
    igt_info("Stress test: %d iterations passed\n", iterations);
}

/*
 * 测试夹具：清理
 */
igt_fixture {
    if (device) {
        amdgpu_device_deinitialize(device);
        device = NULL;
    }
    if (drm_fd >= 0) {
        close(drm_fd);
        drm_fd = -1;
    }
    igt_info("Cleanup completed\n");
}
```

#### 2.2 编译和运行

```bash
#!/bin/bash
# build_and_run_bo_test.sh

set -e

IGT_DIR="${1:-$HOME/igt-gpu-tools}"
TEST_NAME="amd_bo_operations"

echo "=== 编译和运行 AMDGPU BO 测试 ==="
echo ""

# 复制测试文件
cp "${TEST_NAME}.c" "$IGT_DIR/tests/amdgpu/"

# 更新 meson.build
cd "$IGT_DIR"
if ! grep -q "$TEST_NAME" tests/amdgpu/meson.build; then
    echo "test_progs += [ '$TEST_NAME' ]" >> tests/amdgpu/meson.build
fi

# 编译
ninja -C build

# 运行
TEST_BIN="$IGT_DIR/build/tests/amdgpu/$TEST_NAME"

if [ -f "$TEST_BIN" ]; then
    echo "列出子测试:"
    "$TEST_BIN" --list-subtests
    
    echo ""
    echo "运行所有子测试:"
    sudo "$TEST_BIN" 2>&1 | tee "${TEST_NAME}.log"
else
    echo "编译失败"
    exit 1
fi

echo ""
echo "=== 完成 ==="
```

### 三、案例分析

#### 3.1 测试覆盖分析

```
┌─────────────────────────────────────────────────────────────────────┐
│              amd_bo_operations 测试覆盖分析                          │
│                                                                     │
│  功能覆盖：                                                          │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  功能          子测试              覆盖状态              │       │
│  │  ─────────────────────────────────────────────────────   │       │
│  │  创建          create-destroy-basic ✓                  │       │
│  │  销毁          create-destroy-basic ✓                  │       │
│  │  不同域        create-domains       ✓                  │       │
│  │  边界大小      create-boundary-sizes ✓                 │       │
│  │  无效参数      create-invalid       ✓                  │       │
│  │  内存映射      mmap-basic           ✓                  │       │
│  │  并发操作      concurrent-create    ✓                  │       │
│  │  压力测试      stress-create-destroy ✓                 │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
│  边界条件覆盖：                                                      │
│  ● 大小：0, 1, 4095, 4096, 4097, 65536, 1MB, 16MB                  │
│  ● 域：GTT, VRAM, CPU                                               │
│  ● 并发：100 个 BO 同时创建/销毁                                    │
│  ● 压力：1000 次创建/销毁循环                                       │
│                                                                     │
│  错误路径覆盖：                                                      │
│  ● 零大小                                                           │
│  ● 无效域                                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 四、相关链接

- IGT 测试编写指南：https://drm.pages.freedesktop.org/igt-gpu-tools/igt-gpu-tools-TestWriting.html
- AMDGPU 用户空间 API：https://dri.freedesktop.org/docs/drm/gpu/amdgpu.html#userspace-apis
- GEM 设计文档：https://lwn.net/Articles/283798/

## 今日小结

- BO 操作测试应覆盖功能、边界、错误、并发和压力五个维度
- 测试应验证不同大小、不同域的 BO 创建和销毁
- 内存映射测试需要验证读写正确性
- 并发测试可以检测资源竞争和泄露
- 压力测试可以验证长期稳定性

## 扩展思考

1. 如何设计一个测试来检测 GPU 内存碎片问题？
2. 在多进程环境中，BO 的共享和同步需要哪些额外的测试？
