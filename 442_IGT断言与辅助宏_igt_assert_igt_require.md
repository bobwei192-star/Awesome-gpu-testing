# IGT 断言与辅助宏：igt_assert / igt_require

## 学习目标

- 掌握 IGT 断言系统的完整功能
- 理解 igt_assert、igt_require、igt_fail 的区别和用法
- 学会使用 IGT 辅助宏简化测试代码
- 了解断言失败后的处理机制
- 能够编写健壮的测试用例

## 知识详解

### 一、概念原理

#### 1.1 断言类型对比

```
┌─────────────────────────────────────────────────────────────────────┐
│              IGT 断言类型对比                                        │
│                                                                     │
│  断言              条件不满足时行为        使用场景                   │
│  ─────────────────────────────────────────────────────────────      │
│                                                                     │
│  igt_assert(expr)  标记 FAIL，继续执行     测试核心验证              │
│                    后续子测试            验证功能正确性              │
│                                                                     │
│  igt_require(expr) 标记 SKIP，跳过当前     前置条件检查              │
│                    子测试                硬件/驱动支持检查           │
│                                                                     │
│  igt_fail(msg)     标记 FAIL，立即返回     手动标记失败              │
│                    当前子测试            复杂错误处理                │
│                                                                     │
│  igt_skip(msg)     标记 SKIP，立即返回     手动标记跳过              │
│                    当前子测试            复杂条件判断                │
│                                                                     │
│  igt_success()     标记 PASS             显式标记成功                │
│                                                                     │
│  igt_assert_f(expr,  格式化失败信息      需要详细错误信息            │
│           fmt, ...)                                                 │
│                                                                     │
│  执行流程：                                                          │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  igt_require → 失败 → SKIP → 执行下一个子测试           │       │
│  │  igt_assert  → 失败 → FAIL → 执行下一个子测试           │       │
│  │  igt_fail    → 失败 → FAIL → 执行下一个子测试           │       │
│  │  无断言      → 正常结束 → PASS                          │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.2 辅助宏分类

```
┌─────────────────────────────────────────────────────────────────────┐
│              IGT 辅助宏分类                                          │
│                                                                     │
│  类别              宏                  功能                          │
│  ─────────────────────────────────────────────────────────────      │
│                                                                     │
│  相等断言          igt_assert_eq(a,b)  a == b                       │
│                    igt_assert_neq(a,b) a != b                       │
│                    igt_assert_lt(a,b)  a < b                        │
│                    igt_assert_lte(a,b) a <= b                       │
│                    igt_assert_gt(a,b)  a > b                        │
│                    igt_assert_gte(a,b) a >= b                       │
│                                                                     │
│  指针断言          igt_assert_neq(ptr, NULL) 指针非空               │
│                    igt_assert_eq(ptr, NULL) 指针为空                │
│                                                                     │
│  返回值断言        igt_assert_eq(ret, 0)   返回 0                  │
│                    igt_assert_eq(ret, -1)  返回 -1                 │
│                                                                     │
│  条件跳过          igt_require_f(expr, fmt, ...) 格式化跳过信息     │
│                                                                     │
│  时间辅助          igt_assert_within_time(start, end, limit)       │
│                                                                     │
│  内存辅助          igt_assert_cmpfloat(a, op, b, eps) 浮点比较     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 二、实践操作

#### 2.1 断言使用示例

```c
/*
 * 断言使用示例
 */

#include "igt.h"

/* 基本断言 */
igt_subtest("basic-assertions") {
    int value = 42;
    int *ptr = &value;
    int *null_ptr = NULL;
    
    /* 相等断言 */
    igt_assert_eq(value, 42);
    igt_assert_neq(value, 0);
    
    /* 范围断言 */
    igt_assert_gt(value, 0);
    igt_assert_lt(value, 100);
    igt_assert_gte(value, 42);
    igt_assert_lte(value, 42);
    
    /* 指针断言 */
    igt_assert_neq(ptr, NULL);
    igt_assert_eq(null_ptr, NULL);
    
    /* 布尔断言 */
    igt_assert(value > 0);
    igt_assert(ptr != NULL);
}

/* 返回值断言 */
igt_subtest("return-value-assertions") {
    int fd = open("/dev/dri/card0", O_RDWR);
    
    /* 验证打开成功 */
    igt_assert_f(fd >= 0, "Failed to open DRM device: %s\n", strerror(errno));
    
    /* 验证 ioctl 成功 */
    struct drm_version version;
    memset(&version, 0, sizeof(version));
    igt_assert_eq(drmIoctl(fd, DRM_IOCTL_VERSION, &version), 0);
    
    close(fd);
}

/* 条件跳过 */
igt_subtest("conditional-skip") {
    int fd = drm_open_driver(DRIVER_AMDGPU);
    
    /* 如果不是 AMDGPU，跳过 */
    igt_require_f(fd >= 0, "AMDGPU driver not available\n");
    
    /* 检查内核版本 */
    struct utsname buf;
    uname(&buf);
    
    /* 需要 5.15+ 内核 */
    igt_require_f(strtod(buf.release, NULL) >= 5.15,
                  "Kernel %s too old, need 5.15+\n", buf.release);
    
    /* 执行测试 */
    igt_info("Running on AMDGPU with kernel %s\n", buf.release);
    
    close(fd);
}

/* 手动失败 */
igt_subtest("manual-failure") {
    int fd = drm_open_driver(DRIVER_AMDGPU);
    igt_require(fd >= 0);
    
    /* 查询设备信息 */
    struct drm_amdgpu_info info;
    int ret = amdgpu_query_info(fd, AMDGPU_INFO_DEV_INFO, sizeof(info), &info);
    
    if (ret != 0) {
        igt_fail("Failed to query AMDGPU info: %d\n", ret);
    }
    
    /* 验证设备信息 */
    if (info.family == 0) {
        igt_fail("Invalid family ID: %d\n", info.family);
    }
    
    close(fd);
}
```

#### 2.2 复杂断言场景

```c
/*
 * 复杂断言场景示例
 */

/* 浮点比较 */
igt_subtest("float-comparison") {
    float a = 0.1f + 0.2f;
    float b = 0.3f;
    
    /* 不能直接用 == 比较浮点数 */
    /* igt_assert_eq(a, b);  // 可能失败！ */
    
    /* 使用容差比较 */
    igt_assert_cmpfloat(a, ==, b, 0.0001f);
}

/* 时间断言 */
igt_subtest("timing-assertion") {
    struct timespec start, end;
    
    clock_gettime(CLOCK_MONOTONIC, &start);
    
    /* 执行操作 */
    usleep(10000);  /* 10ms */
    
    clock_gettime(CLOCK_MONOTONIC, &end);
    
    double elapsed = (end.tv_sec - start.tv_sec) * 1000.0 +
                     (end.tv_nsec - start.tv_nsec) / 1000000.0;
    
    /* 验证在合理时间内完成 */
    igt_assert_f(elapsed < 100.0, "Operation took %.2f ms, expected < 100ms\n", elapsed);
    igt_assert_f(elapsed > 5.0, "Operation took %.2f ms, expected > 5ms\n", elapsed);
}

/* 数组断言 */
igt_subtest("array-assertion") {
    int expected[] = {1, 2, 3, 4, 5};
    int actual[] = {1, 2, 3, 4, 5};
    
    for (int i = 0; i < 5; i++) {
        igt_assert_f(expected[i] == actual[i],
                     "Mismatch at index %d: expected %d, got %d\n",
                     i, expected[i], actual[i]);
    }
}
```

### 三、案例分析

#### 3.1 实际测试中的断言使用

```c
/*
 * tests/gem_create.c 中的断言使用
 */

igt_subtest("create-invalid-size") {
    struct drm_i915_gem_create create;
    int ret;
    
    /* 测试零大小 */
    memset(&create, 0, sizeof(create));
    create.size = 0;
    
    ret = drmIoctl(drm_fd, DRM_IOCTL_I915_GEM_CREATE, &create);
    
    /* 期望失败 */
    igt_assert_eq(ret, -1);
    igt_assert_eq(errno, EINVAL);
}

igt_subtest("create-valid") {
    struct drm_i915_gem_create create;
    
    memset(&create, 0, sizeof(create));
    create.size = 4096;
    
    /* 期望成功 */
    igt_assert_eq(drmIoctl(drm_fd, DRM_IOCTL_I915_GEM_CREATE, &create), 0);
    
    /* 验证句柄 */
    igt_assert_neq(create.handle, 0);
    
    /* 清理 */
    gem_close(drm_fd, create.handle);
}
```

### 四、相关链接

- IGT 断言文档：https://drm.pages.freedesktop.org/igt-gpu-tools/igt-gpu-tools-Assertions.html
- IGT 辅助宏：https://drm.pages.freedesktop.org/igt-gpu-tools/igt-gpu-tools-Helpers.html

## 今日小结

- igt_assert 用于验证测试条件，失败标记 FAIL
- igt_require 用于检查前置条件，失败标记 SKIP
- igt_fail/igt_skip 用于手动标记结果
- IGT 提供丰富的辅助宏（相等、范围、指针、时间等）
- 断言失败通过 longjmp 实现非本地跳转

## 扩展思考

1. 为什么 IGT 使用 longjmp 而不是 C++ 异常或简单的 return？这种设计有什么权衡？
2. 在测试中使用过多的断言是否会降低代码可读性？如何平衡断言密度和可读性？
