# IGT 核心库（lib/）架构分析

## 学习目标

- 深入理解 IGT 核心库的设计架构和关键数据结构
- 掌握 igt_main、igt_subtest、igt_fixture 等核心机制
- 理解 IGT 的测试生命周期管理
- 学会使用 IGT 的断言和辅助宏
- 了解 IGT 的硬件抽象层设计

## 知识详解

### 一、概念原理

#### 1.1 核心库架构

```
┌─────────────────────────────────────────────────────────────────────┐
│              IGT 核心库架构                                          │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────┐      │
│   │                    测试程序 (tests/*.c)                  │      │
│   │                                                         │      │
│   │  igt_main({                                             │      │
│   │    igt_fixture { setup(); }                             │      │
│   │    igt_subtest("name") { test_body(); }                 │      │
│   │    igt_fixture { teardown(); }                          │      │
│   │  })                                                     │      │
│   └─────────────────────────┬───────────────────────────────┘      │
│                             │                                       │
│                             ▼                                       │
│   ┌─────────────────────────────────────────────────────────┐      │
│   │                    核心库 (libigt)                       │      │
│   │                                                         │      │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │      │
│   │  │ 测试框架    │  │ 断言系统    │  │ 日志系统    │     │      │
│   │  │ igt_core    │  │ igt_aux     │  │ igt_log     │     │      │
│   │  │             │  │             │  │             │     │      │
│   │  │ ● 子测试    │  │ ● igt_assert│  │ ● igt_info  │     │      │
│   │  │ ● 夹具      │  │ ● igt_require│ │ ● igt_debug │     │      │
│   │  │ ● 跳过逻辑  │  │ ● igt_fail  │  │ ● igt_warn  │     │      │
│   │  └─────────────┘  └─────────────┘  └─────────────┘     │      │
│   │                                                         │      │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │      │
│   │  │ DRM 包装    │  │ KMS 辅助    │  │ GEM 辅助    │     │      │
│   │  │ ioctl_wrappers│ │ igt_kms    │  │ igt_gem     │     │      │
│   │  │             │  │             │  │             │     │      │
│   │  │ ● 安全 ioctl│  │ ● CRTC管理  │  │ ● BO 操作   │     │      │
│   │  │ ● 错误处理  │  │ ● Mode设置  │  │ ● 执行列表  │     │      │
│   │  │ ● 重试逻辑  │  │ ● Plane管理 │  │ ● MMAP      │     │      │
│   │  └─────────────┘  └─────────────┘  └─────────────┘     │      │
│   │                                                         │      │
│   └─────────────────────────┬───────────────────────────────┘      │
│                             │                                       │
│                             ▼                                       │
│   ┌─────────────────────────────────────────────────────────┐      │
│   │                    内核 DRM 驱动                         │      │
│   └─────────────────────────────────────────────────────────┘      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.2 测试生命周期

```
┌─────────────────────────────────────────────────────────────────────┐
│              IGT 测试生命周期                                        │
│                                                                     │
│   程序启动                                                            │
│      │                                                              │
│      ▼                                                              │
│   ┌─────────────┐                                                   │
│   │ igt_main()  │  ← 解析命令行参数                                  │
│   │             │  ← 初始化日志                                      │
│   └──────┬──────┘                                                   │
│          │                                                          │
│          ▼                                                          │
│   ┌─────────────┐                                                   │
│   │ igt_fixture │  ← 执行一次（全局设置）                            │
│   │ (setup)     │                                                   │
│   └──────┬──────┘                                                   │
│          │                                                          │
│          ▼                                                          │
│   ┌─────────────────────────────────────────────────────────┐      │
│   │              子测试循环                                  │      │
│   │                                                         │      │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │      │
│   │  │subtest 1│  │subtest 2│  │subtest 3│  │subtest N│   │      │
│   │  │         │  │         │  │         │  │         │   │      │
│   │  │ ● 检查  │  │ ● 检查  │  │ ● 检查  │  │ ● 检查  │   │      │
│   │  │   条件  │  │   条件  │  │   条件  │  │   条件  │   │      │
│   │  │ ● 执行  │  │ ● 执行  │  │ ● 执行  │  │ ● 执行  │   │      │
│   │  │ ● 断言  │  │ ● 断言  │  │ ● 断言  │  │ ● 断言  │   │      │
│   │  │ ● 记录  │  │ ● 记录  │  │ ● 记录  │  │ ● 记录  │   │      │
│   │  └─────────┘  └─────────┘  └─────────┘  └─────────┘   │      │
│   │                                                         │      │
│   └─────────────────────────────────────────────────────────┘      │
│          │                                                          │
│          ▼                                                          │
│   ┌─────────────┐                                                   │
│   │ igt_fixture │  ← 执行一次（全局清理）                            │
│   │ (teardown)  │                                                   │
│   └──────┬──────┘                                                   │
│          │                                                          │
│          ▼                                                          │
│   ┌─────────────┐                                                   │
│   │ 生成报告    │  ← pass/fail/skip 统计                             │
│   │ 退出程序    │                                                   │
│   └─────────────┘                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 二、实践操作

#### 2.1 核心数据结构

```c
/*
 * IGT 核心数据结构简化版
 * 参考：lib/igt_core.h
 */

/* 子测试描述符 */
struct igt_subtest {
    const char *name;           /* 子测试名称 */
    void (*fn)(void);           /* 测试函数 */
    bool enabled;               /* 是否启用 */
    enum igt_result result;     /* 执行结果 */
};

/* 测试结果枚举 */
enum igt_result {
    IGT_RESULT_PASS,            /* 通过 */
    IGT_RESULT_FAIL,            /* 失败 */
    IGT_RESULT_SKIP,            /* 跳过 */
    IGT_RESULT_TIMEOUT,         /* 超时 */
    IGT_RESULT_CRASH            /* 崩溃 */
};

/* 测试状态 */
struct igt_test_state {
    int argc;                   /* 命令行参数 */
    char **argv;
    
    struct igt_subtest *subtests;  /* 子测试数组 */
    int subtest_count;          /* 子测试数量 */
    
    bool list_subtests;         /* 仅列出子测试 */
    bool run_single;            /* 仅运行单个测试 */
    const char *filter;         /* 子测试过滤 */
    
    int timeout;                /* 超时设置（秒） */
    bool fork;                  /* 是否 fork 子进程 */
};
```

#### 2.2 断言系统实现

```c
/*
 * IGT 断言宏简化版
 * 参考：lib/igt_core.h
 */

/* 基本断言 */
#define igt_assert(expr) \
    do { \
        if (!(expr)) { \
            igt_fail("Assertion failed: %s\n", #expr); \
        } \
    } while (0)

/* 要求条件（不满足则跳过） */
#define igt_require(expr) \
    do { \
        if (!(expr)) { \
            igt_skip("Requirement not met: %s\n", #expr); \
        } \
    } while (0)

/* 检查 errno */
#define igt_assert_eq(a, b) \
    do { \
        if ((a) != (b)) { \
            igt_fail("Assertion failed: %s (%d) != %s (%d)\n", \
                     #a, (a), #b, (b)); \
        } \
    } while (0)

/* 失败处理 */
void igt_fail(const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);
    
    /* 记录失败信息 */
    vfprintf(stderr, fmt, args);
    
    /* 标记测试失败 */
    current_subtest->result = IGT_RESULT_FAIL;
    
    /* 清理并退出 */
    va_end(args);
    longjmp(test_jmpbuf, 1);  /* 非本地跳转回测试框架 */
}

/* 跳过处理 */
void igt_skip(const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);
    
    vfprintf(stderr, fmt, args);
    
    current_subtest->result = IGT_RESULT_SKIP;
    
    va_end(args);
    longjmp(test_jmpbuf, 1);
}
```

### 三、案例分析

#### 3.1 简单测试分析

```c
/*
 * tests/gem_basic.c 简化分析
 * 参考：IGT 官方 gem_basic 测试
 */

#include "igt.h"

IGT_TEST_DESCRIPTION("Basic GEM buffer object tests");

/* 全局夹具：每个子测试前执行 */
static int drm_fd;
static uint32_t devid;

igt_fixture {
    /* 打开 DRM 设备 */
    drm_fd = drm_open_driver_master(DRIVER_INTEL | DRIVER_AMDGPU);
    
    /* 获取设备 ID */
    devid = intel_get_drm_devid(drm_fd);
    
    /* 检查设备是否支持基本功能 */
    igt_require_gem(drm_fd);
}

/* 子测试 1：创建和关闭 BO */
igt_subtest("create-close") {
    struct drm_i915_gem_create create;
    
    /* 创建 BO */
    memset(&create, 0, sizeof(create));
    create.size = 4096;
    
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_CREATE, &create);
    
    /* 验证句柄有效 */
    igt_assert(create.handle != 0);
    
    /* 关闭 BO */
    gem_close(drm_fd, create.handle);
}

/* 子测试 2：创建大 BO */
igt_subtest("create-large") {
    struct drm_i915_gem_create create;
    
    memset(&create, 0, sizeof(create));
    create.size = 256 * 1024 * 1024;  /* 256MB */
    
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_CREATE, &create);
    igt_assert(create.handle != 0);
    
    gem_close(drm_fd, create.handle);
}

/* 子测试 3：无效参数测试 */
igt_subtest("create-invalid-size") {
    struct drm_i915_gem_create create;
    int ret;
    
    memset(&create, 0, sizeof(create));
    create.size = 0;  /* 无效大小 */
    
    /* 期望失败 */
    ret = drmIoctl(drm_fd, DRM_IOCTL_I915_GEM_CREATE, &create);
    igt_assert_eq(ret, -1);
    igt_assert_eq(errno, EINVAL);
}

igt_fixture {
    /* 清理：关闭 DRM 设备 */
    close(drm_fd);
}
```

### 四、相关链接

- IGT 核心 API：https://drm.pages.freedesktop.org/igt-gpu-tools/igt-gpu-tools-CoreAPI.html
- IGT 测试编写指南：https://drm.pages.freedesktop.org/igt-gpu-tools/igt-gpu-tools-TestWriting.html

## 今日小结

- IGT 核心库提供测试框架、断言系统、日志系统和 DRM 辅助功能
- igt_main 定义测试入口，igt_fixture 定义设置/清理，igt_subtest 定义测试用例
- 断言系统支持 igt_assert（失败）、igt_require（跳过）等多种语义
- 测试通过 longjmp 实现非本地跳转，支持失败后的清理

## 扩展思考

1. IGT 的 longjmp 错误处理机制有什么优缺点？与 C++ 异常相比如何？
2. 如何设计一个支持多线程测试的 IGT 扩展？
