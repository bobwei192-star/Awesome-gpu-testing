# IGT GEM 测试分析：gem_busy / gem_wait

## 学习目标

- 理解 GPU 缓冲区忙状态检测机制
- 掌握 gem_busy 测试的功能和原理
- 掌握 gem_wait 测试的功能和原理
- 了解 GPU 命令完成检测的方法
- 能够编写缓冲区状态相关的测试

## 知识详解

### 一、概念原理

#### 1.1 GPU 缓冲区状态

```
┌─────────────────────────────────────────────────────────────────────┐
│              GPU 缓冲区状态                                          │
│                                                                     │
│  GEM 对象（Buffer Object）生命周期中的状态：                         │
│                                                                     │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐        │
│  │ 创建    │───▶│ 空闲    │───▶│ 忙      │───▶│ 完成    │        │
│  │ CREATED │    │ IDLE    │    │ BUSY    │    │ DONE    │        │
│  └─────────┘    └────┬────┘    └────┬────┘    └────┬────┘        │
│                      │              │              │               │
│                      │              │              │               │
│  状态查询：           │              │              │               │
│  ● gem_busy() ◀─────┘              │              │               │
│    返回当前状态                      │              │               │
│                                      │              │               │
│  等待完成：                          │              │               │
│  ● gem_wait() ──────────────────────┘              │               │
│    阻塞直到完成                                    │               │
│                                                    │               │
│  状态转换：                                        │               │
│  ● GPU 命令提交引用 BO ──▶ BUSY                   │               │
│  ● GPU 命令完成 ──▶ DONE ──▶ IDLE                 │               │
│                                                                     │
│  忙状态含义：                                                        │
│  ● 有 GPU 命令正在读取该 BO                                         │
│  ● 有 GPU 命令正在写入该 BO                                         │
│  ● CPU 不能安全访问该 BO                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.2 gem_busy 测试分析

```
┌─────────────────────────────────────────────────────────────────────┐
│              gem_busy 测试分析                                       │
│                                                                     │
│  测试文件：tests/gem_busy.c                                         │
│                                                                     │
│  测试目标：验证 GEM 缓冲区忙状态查询功能                             │
│                                                                     │
│  子测试列表：                                                        │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  子测试名称          验证内容                            │       │
│  │  ─────────────────────────────────────────────────────   │       │
│  │  basic               基本忙状态查询                      │       │
│  │  after-execution     执行后状态变化                      │       │
│  │  concurrent          并发执行时的状态                    │       │
│  │  invalid-handle      无效句柄测试                        │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
│  核心 ioctl：                                                        │
│  ● DRM_IOCTL_I915_GEM_BUSY（Intel）                                 │
│  ● DRM_IOCTL_AMDGPU_GEM_OP（AMD，带 AMDGPU_GEM_OP_GET_BUSY）        │
│                                                                     │
│  返回值：                                                            │
│  ● 0：BO 空闲                                                       │
│  ● >0：BO 忙（正在 GPU 上执行）                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.3 gem_wait 测试分析

```
┌─────────────────────────────────────────────────────────────────────┐
│              gem_wait 测试分析                                       │
│                                                                     │
│  测试文件：tests/gem_wait.c                                         │
│                                                                     │
│  测试目标：验证 GEM 缓冲区等待完成功能                               │
│                                                                     │
│  子测试列表：                                                        │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  子测试名称          验证内容                            │       │
│  │  ─────────────────────────────────────────────────────   │       │
│  │  basic               基本等待功能                        │       │
│  │  timeout             超时等待                            │       │
│  │  invalid-timeout     无效超时值                          │       │
│  │  invalid-handle      无效句柄测试                        │       │
│  │  wait-fd             基于文件描述符的等待                │       │
│  │  busy                忙状态与等待结合                    │       │
│  │  reset               GPU reset 后等待                    │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
│  核心 ioctl：                                                        │
│  ● DRM_IOCTL_I915_GEM_WAIT（Intel）                                 │
│  ● DRM_IOCTL_AMDGPU_GEM_WAIT_IDLE（AMD）                            │
│                                                                     │
│  等待模式：                                                          │
│  ● 阻塞等待：直到 BO 空闲或超时                                     │
│  ● 非阻塞：立即返回当前状态                                         │
│  ● 基于 fd：返回等待 fd，可配合 poll/epoll 使用                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 二、实践操作

#### 2.1 gem_busy 测试代码

```c
/*
 * tests/gem_busy.c 简化分析
 */

#include "igt.h"

IGT_TEST_DESCRIPTION("GEM buffer busy state tests");

static int drm_fd;

igt_fixture {
    drm_fd = drm_open_driver(DRIVER_INTEL);
    igt_require(drm_fd >= 0);
}

/* 基本忙状态测试 */
igt_subtest("basic") {
    struct drm_i915_gem_create create;
    struct drm_i915_gem_busy busy;
    
    /* 创建 BO */
    memset(&create, 0, sizeof(create));
    create.size = 4096;
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_CREATE, &create);
    
    /* 查询忙状态 */
    memset(&busy, 0, sizeof(busy));
    busy.handle = create.handle;
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_BUSY, &busy);
    
    /* 新创建的 BO 应该是空闲的 */
    igt_assert_eq(busy.busy, 0);
    igt_info("New BO busy state: %u (expected 0)\n", busy.busy);
    
    /* 清理 */
    gem_close(drm_fd, create.handle);
}

/* 执行后的状态变化 */
igt_subtest("after-execution") {
    struct drm_i915_gem_create create;
    struct drm_i915_gem_execbuffer2 execbuf;
    struct drm_i915_gem_exec_object2 exec_object;
    struct drm_i915_gem_busy busy;
    uint32_t batch[16];
    
    /* 创建 BO */
    memset(&create, 0, sizeof(create));
    create.size = 4096;
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_CREATE, &create);
    
    /* 准备 batch buffer */
    memset(batch, 0, sizeof(batch));
    batch[0] = MI_NOOP;
    batch[1] = MI_BATCH_BUFFER_END;
    gem_write(drm_fd, create.handle, 0, batch, sizeof(batch));
    
    /* 设置执行 */
    memset(&exec_object, 0, sizeof(exec_object));
    exec_object.handle = create.handle;
    
    memset(&execbuf, 0, sizeof(execbuf));
    execbuf.buffers_ptr = (uintptr_t)&exec_object;
    execbuf.buffer_count = 1;
    execbuf.batch_len = sizeof(batch);
    
    /* 提交前查询 */
    memset(&busy, 0, sizeof(busy));
    busy.handle = create.handle;
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_BUSY, &busy);
    igt_assert_eq(busy.busy, 0);
    igt_info("Before execution: busy=%u\n", busy.busy);
    
    /* 提交执行 */
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_EXECBUFFER2, &execbuf);
    
    /* 提交后立即查询（可能忙） */
    memset(&busy, 0, sizeof(busy));
    busy.handle = create.handle;
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_BUSY, &busy);
    igt_info("After execution: busy=%u\n", busy.busy);
    
    /* 等待完成 */
    struct drm_i915_gem_wait wait;
    memset(&wait, 0, sizeof(wait));
    wait.bo_handle = create.handle;
    wait.timeout_ns = -1;  /* 无限等待 */
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_WAIT, &wait);
    
    /* 完成后查询 */
    memset(&busy, 0, sizeof(busy));
    busy.handle = create.handle;
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_BUSY, &busy);
    igt_assert_eq(busy.busy, 0);
    igt_info("After wait: busy=%u\n", busy.busy);
    
    /* 清理 */
    gem_close(drm_fd, create.handle);
}

igt_fixture {
    close(drm_fd);
}
```

#### 2.2 gem_wait 测试代码

```c
/*
 * tests/gem_wait.c 简化分析
 */

#include "igt.h"

IGT_TEST_DESCRIPTION("GEM buffer wait tests");

static int drm_fd;

igt_fixture {
    drm_fd = drm_open_driver(DRIVER_INTEL);
    igt_require(drm_fd >= 0);
}

/* 基本等待测试 */
igt_subtest("basic") {
    struct drm_i915_gem_create create;
    struct drm_i915_gem_wait wait;
    
    /* 创建 BO */
    memset(&create, 0, sizeof(create));
    create.size = 4096;
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_CREATE, &create);
    
    /* 等待 BO 空闲 */
    memset(&wait, 0, sizeof(wait));
    wait.bo_handle = create.handle;
    wait.timeout_ns = -1;  /* 无限等待 */
    
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_WAIT, &wait);
    
    igt_info("Wait completed, BO is idle\n");
    
    /* 清理 */
    gem_close(drm_fd, create.handle);
}

/* 超时等待测试 */
igt_subtest("timeout") {
    struct drm_i915_gem_create create;
    struct drm_i915_gem_execbuffer2 execbuf;
    struct drm_i915_gem_exec_object2 exec_object;
    struct drm_i915_gem_wait wait;
    uint32_t batch[16];
    int ret;
    
    /* 创建长时间执行的 BO */
    memset(&create, 0, sizeof(create));
    create.size = 4096;
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_CREATE, &create);
    
    /* 准备长时间执行的 batch */
    memset(batch, 0, sizeof(batch));
    /* 添加大量 NOP 命令 */
    for (int i = 0; i < 1000; i++) {
        batch[i % 16] = MI_NOOP;
    }
    batch[15] = MI_BATCH_BUFFER_END;
    gem_write(drm_fd, create.handle, 0, batch, sizeof(batch));
    
    /* 设置执行 */
    memset(&exec_object, 0, sizeof(exec_object));
    exec_object.handle = create.handle;
    
    memset(&execbuf, 0, sizeof(execbuf));
    execbuf.buffers_ptr = (uintptr_t)&exec_object;
    execbuf.buffer_count = 1;
    execbuf.batch_len = sizeof(batch);
    
    /* 提交执行 */
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_EXECBUFFER2, &execbuf);
    
    /* 短时间等待（应该超时） */
    memset(&wait, 0, sizeof(wait));
    wait.bo_handle = create.handle;
    wait.timeout_ns = 1;  /* 1ns 超时 */
    
    ret = drmIoctl(drm_fd, DRM_IOCTL_I915_GEM_WAIT, &wait);
    
    /* 可能超时或刚好完成 */
    if (ret == -1) {
        igt_assert_eq(errno, ETIME);
        igt_info("Wait timed out as expected\n");
    } else {
        igt_info("Wait completed quickly\n");
    }
    
    /* 清理 */
    gem_close(drm_fd, create.handle);
}

/* 无效句柄测试 */
igt_subtest("invalid-handle") {
    struct drm_i915_gem_wait wait;
    int ret;
    
    memset(&wait, 0, sizeof(wait));
    wait.bo_handle = 0xDEADBEEF;
    wait.timeout_ns = -1;
    
    ret = drmIoctl(drm_fd, DRM_IOCTL_I915_GEM_WAIT, &wait);
    igt_assert_eq(ret, -1);
    igt_assert_eq(errno, ENOENT);
}

igt_fixture {
    close(drm_fd);
}
```

#### 2.3 运行测试

```bash
#!/bin/bash
# run_gem_busy_wait.sh - 运行 gem_busy 和 gem_wait 测试

set -e

IGT_BUILD="${1:-$HOME/igt-gpu-tools/build}"

echo "=== 运行 gem_busy 和 gem_wait 测试 ==="
echo ""

# 运行 gem_busy
echo "【1】gem_busy 测试"
sudo "$IGT_BUILD/tests/gem_busy" 2>&1 | tee gem_busy.log
echo ""

# 运行 gem_wait
echo "【2】gem_wait 测试"
sudo "$IGT_BUILD/tests/gem_wait" 2>&1 | tee gem_wait.log
echo ""

# 运行指定子测试
echo "【3】gem_busy after-execution 子测试"
sudo "$IGT_BUILD/tests/gem_busy" --run-subtest after-execution 2>&1 | tee gem_busy_after.log
echo ""

echo "=== 测试完成 ==="
```

### 三、案例分析

#### 3.1 忙状态检测的实际应用

```
场景：应用程序需要修改一个 BO 的内容

问题：如果 GPU 正在读取该 BO，CPU 修改会导致数据竞争

解决方案：
1. 查询 BO 忙状态
2. 如果忙，等待完成或创建新 BO
3. 如果不忙，安全修改

代码示例：
```c
/* 安全修改 BO */
void safe_modify_bo(int drm_fd, uint32_t handle, void *data, size_t size) {
    struct drm_i915_gem_busy busy;
    struct drm_i915_gem_wait wait;
    
    /* 检查忙状态 */
    memset(&busy, 0, sizeof(busy));
    busy.handle = handle;
    drmIoctl(drm_fd, DRM_IOCTL_I915_GEM_BUSY, &busy);
    
    if (busy.busy) {
        /* BO 忙，等待完成 */
        memset(&wait, 0, sizeof(wait));
        wait.bo_handle = handle;
        wait.timeout_ns = -1;
        drmIoctl(drm_fd, DRM_IOCTL_I915_GEM_WAIT, &wait);
    }
    
    /* 现在可以安全修改 */
    gem_write(drm_fd, handle, 0, data, size);
}
```

### 四、相关链接

- IGT GEM 测试：https://gitlab.freedesktop.org/drm/igt-gpu-tools/-/tree/master/tests
- DRM 缓冲区管理：https://dri.freedesktop.org/docs/drm/gpu/drm-mm.html
- dma_fence：https://dri.freedesktop.org/docs/drm/driver-api/dma-buf.html

## 今日小结

- GEM 缓冲区有三种状态：空闲、忙、完成
- gem_busy 查询缓冲区的当前忙状态
- gem_wait 阻塞等待缓冲区变为空闲
- 忙状态检测是避免 CPU/GPU 数据竞争的重要手段
- 测试应覆盖正常查询、执行后状态变化和超时场景

## 扩展思考

1. 在异步 GPU 编程中，忙状态检测和 fence 同步有什么区别？各自适用于什么场景？
2. 如何设计一个测试来验证 GPU 命令的优先级和抢占机制？
