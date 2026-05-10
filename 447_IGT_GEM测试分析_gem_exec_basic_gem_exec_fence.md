# IGT GEM 测试分析：gem_exec_basic / gem_exec_fence

## 学习目标

- 深入理解 GPU 命令提交和执行机制
- 掌握 gem_exec_basic 测试的功能和原理
- 掌握 gem_exec_fence 测试的功能和原理
- 了解 dma_fence 在 GPU 命令同步中的作用
- 能够编写命令提交相关的测试

## 知识详解

### 一、概念原理

#### 1.1 GPU 命令提交模型

```
┌─────────────────────────────────────────────────────────────────────┐
│              GPU 命令提交模型                                        │
│                                                                     │
│   用户空间                                                            │
│   ──────────                                                         │
│   ┌─────────────┐                                                   │
│   │ 应用程序    │                                                   │
│   │             │                                                   │
│   │ 1. 创建 BO  │                                                   │
│   │ 2. 填充命令 │                                                   │
│   │ 3. 提交执行 │───▶ ioctl(DRM_IOCTL_I915_GEM_EXECBUFFER2)        │
│   │ 4. 等待完成 │                                                   │
│   └─────────────┘                                                   │
│                              │                                       │
│   内核空间                    ▼                                       │
│   ──────────               ┌─────────────┐                          │
│                            │ DRM 驱动    │                          │
│                            │             │                          │
│                            │ 1. 验证参数 │                          │
│                            │ 2. 绑定 BO  │                          │
│                            │ 3. 提交到   │                          │
│                            │    GPU 队列 │                          │
│                            │ 4. 返回     │                          │
│                            │    fence    │                          │
│                            └──────┬──────┘                          │
│                                   │                                 │
│                                   ▼                                 │
│                            ┌─────────────┐                          │
│                            │ GPU 硬件    │                          │
│                            │             │                          │
│                            │ 执行命令    │                          │
│                            │ 更新 fence  │                          │
│                            └─────────────┘                          │
│                                                                     │
│  关键概念：                                                          │
│  ● Command Buffer：包含 GPU 命令的缓冲区                            │
│  ● Ring Buffer：GPU 命令队列                                        │
│  ● Fence：命令完成信号                                              │
│  ● Execution Unit：GPU 执行单元                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.2 gem_exec_basic 测试分析

```
┌─────────────────────────────────────────────────────────────────────┐
│              gem_exec_basic 测试分析                                 │
│                                                                     │
│  测试文件：tests/gem_exec_basic.c                                   │
│                                                                     │
│  测试目标：验证基本的 GPU 命令提交和执行功能                         │
│                                                                     │
│  子测试列表：                                                        │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  子测试名称          验证内容                            │       │
│  │  ─────────────────────────────────────────────────────   │       │
│  │  basic               基本命令提交                        │       │
│  │  basic-batch         基本 batch buffer 提交              │       │
│  │  no-relocs           无重定位提交                        │       │
│  │  readonly            只读缓冲区提交                      │       │
│  │  writeonly           只写缓冲区提交                      │       │
│  │  invalid-flags       无效标志测试                        │       │
│  │  invalid-handle      无效句柄测试                        │       │
│  │  invalid-ring        无效 ring 测试                      │       │
│  │  invalid-buffers     无效缓冲区列表测试                  │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
│  核心验证点：                                                        │
│  1. 基本命令可以成功提交和执行                                      │
│  2. 不同 ring（GFX/BLT/VIDEO）可以正确提交                          │
│  3. 无效参数返回适当错误码                                          │
│  4. 缓冲区访问权限（读/写）正确验证                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.3 gem_exec_fence 测试分析

```
┌─────────────────────────────────────────────────────────────────────┐
│              gem_exec_fence 测试分析                                 │
│                                                                     │
│  测试文件：tests/gem_exec_fence.c                                   │
│                                                                     │
│  测试目标：验证 GPU 命令执行的 fence 同步机制                        │
│                                                                     │
│  子测试列表：                                                        │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  子测试名称          验证内容                            │       │
│  │  ─────────────────────────────────────────────────────   │       │
│  │  basic               基本 fence 同步                     │       │
│  │  wait                等待 fence 完成                     │       │
│  │  signal              fence 信号测试                      │       │
│  │  export              导出 fence 到 sync_file             │       │
│  │  import              从 sync_file 导入 fence             │       │
│  │  busy                fence 忙状态测试                    │       │
│  │  reset               fence 重置测试                      │       │
│  │  cross-ring          跨 ring fence 同步                  │       │
│  │  cross-ctx           跨上下文 fence 同步                 │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
│  Fence 同步原理：                                                    │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  命令 A ──▶ GPU 执行 ──▶ Fence A 信号                   │       │
│  │                              │                          │       │
│  │                              ▼                          │       │
│  │  命令 B ──▶ 等待 Fence A ──▶ GPU 执行                   │       │
│  │                                                                     │
│  │  保证命令 B 在命令 A 完成后执行                          │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 二、实践操作

#### 2.1 命令提交数据结构

```c
/*
 * Intel GPU 命令提交结构（简化）
 */

/* 执行缓冲区 */
struct drm_i915_gem_execbuffer2 {
    __u64 buffers_ptr;          /* 缓冲区列表指针 */
    __u32 buffer_count;         /* 缓冲区数量 */
    __u32 batch_start_offset;   /* batch buffer 起始偏移 */
    __u32 batch_len;            /* batch buffer 长度 */
    __u32 DR1;                  /* 保留 */
    __u32 DR4;                  /* 保留 */
    __u64 num_cliprects;        /* cliprect 数量 */
    __u64 cliprects_ptr;        /* cliprect 指针 */
    __u64 flags;                /* 执行标志 */
    __u64 rsvd1;                /* 保留 */
    __u64 rsvd2;                /* 保留 */
};

/* 执行对象（缓冲区描述） */
struct drm_i915_gem_exec_object2 {
    __u64 handle;               /* GEM 句柄 */
    __u64 relocation_count;     /* 重定位数量 */
    __u64 relocs_ptr;           /* 重定位指针 */
    __u64 alignment;            /* 对齐要求 */
    __u64 offset;               /* 执行时的偏移 */
    __u64 flags;                /* 对象标志 */
    __u64 rsvd1;                /* 保留 */
    __u64 rsvd2;                /* 保留 */
};

/* Fence 同步 */
struct drm_i915_gem_exec_fence {
    __u32 handle;               /* syncobj 句柄 */
    __u32 flags;                /* I915_EXEC_FENCE_* */
};
```

#### 2.2 gem_exec_basic 测试代码

```c
/*
 * tests/gem_exec_basic.c 简化分析
 */

#include "igt.h"

IGT_TEST_DESCRIPTION("Basic GEM execution tests");

static int drm_fd;

igt_fixture {
    drm_fd = drm_open_driver(DRIVER_INTEL);
    igt_require(drm_fd >= 0);
}

/* 基本执行测试 */
igt_subtest("basic") {
    struct drm_i915_gem_execbuffer2 execbuf;
    struct drm_i915_gem_exec_object2 exec_object;
    struct drm_i915_gem_create create;
    uint32_t batch[16];
    
    /* 创建 batch buffer */
    memset(&create, 0, sizeof(create));
    create.size = 4096;
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_CREATE, &create);
    
    /* 填充 NOP 命令 */
    memset(batch, 0, sizeof(batch));
    batch[0] = MI_NOOP;  /* NOP 命令 */
    batch[1] = MI_BATCH_BUFFER_END;  /* 结束命令 */
    
    /* 写入 batch buffer */
    gem_write(drm_fd, create.handle, 0, batch, sizeof(batch));
    
    /* 设置执行对象 */
    memset(&exec_object, 0, sizeof(exec_object));
    exec_object.handle = create.handle;
    
    /* 设置执行缓冲区 */
    memset(&execbuf, 0, sizeof(execbuf));
    execbuf.buffers_ptr = (uintptr_t)&exec_object;
    execbuf.buffer_count = 1;
    execbuf.batch_start_offset = 0;
    execbuf.batch_len = sizeof(batch);
    execbuf.flags = I915_EXEC_RENDER;
    
    /* 提交执行 */
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_EXECBUFFER2, &execbuf);
    
    igt_info("Batch executed on ring %d\n", execbuf.flags & I915_EXEC_RING_MASK);
    
    /* 清理 */
    gem_close(drm_fd, create.handle);
}

/* 无效参数测试 */
igt_subtest("invalid-handle") {
    struct drm_i915_gem_execbuffer2 execbuf;
    struct drm_i915_gem_exec_object2 exec_object;
    int ret;
    
    /* 使用无效句柄 */
    memset(&exec_object, 0, sizeof(exec_object));
    exec_object.handle = 0xDEADBEEF;
    
    memset(&execbuf, 0, sizeof(execbuf));
    execbuf.buffers_ptr = (uintptr_t)&exec_object;
    execbuf.buffer_count = 1;
    execbuf.batch_len = 8;
    
    ret = drmIoctl(drm_fd, DRM_IOCTL_I915_GEM_EXECBUFFER2, &execbuf);
    igt_assert_eq(ret, -1);
    igt_assert_eq(errno, ENOENT);
}

igt_fixture {
    close(drm_fd);
}
```

#### 2.3 gem_exec_fence 测试代码

```c
/*
 * tests/gem_exec_fence.c 简化分析
 */

#include "igt.h"

IGT_TEST_DESCRIPTION("GEM execution fence tests");

static int drm_fd;

igt_fixture {
    drm_fd = drm_open_driver(DRIVER_INTEL);
    igt_require(drm_fd >= 0);
}

/* 基本 fence 同步 */
igt_subtest("basic") {
    struct drm_i915_gem_execbuffer2 execbuf;
    struct drm_i915_gem_exec_object2 exec_object;
    struct drm_syncobj_create syncobj_create;
    struct drm_syncobj_wait syncobj_wait;
    struct drm_i915_gem_create create;
    uint32_t batch[16];
    
    /* 创建 syncobj */
    memset(&syncobj_create, 0, sizeof(syncobj_create));
    do_ioctl(drm_fd, DRM_IOCTL_SYNCOBJ_CREATE, &syncobj_create);
    
    /* 创建 batch buffer */
    memset(&create, 0, sizeof(create));
    create.size = 4096;
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_CREATE, &create);
    
    /* 填充命令 */
    memset(batch, 0, sizeof(batch));
    batch[0] = MI_NOOP;
    batch[1] = MI_BATCH_BUFFER_END;
    gem_write(drm_fd, create.handle, 0, batch, sizeof(batch));
    
    /* 设置执行对象 */
    memset(&exec_object, 0, sizeof(exec_object));
    exec_object.handle = create.handle;
    
    /* 设置执行缓冲区，带 fence */
    memset(&execbuf, 0, sizeof(execbuf));
    execbuf.buffers_ptr = (uintptr_t)&exec_object;
    execbuf.buffer_count = 1;
    execbuf.batch_len = sizeof(batch);
    execbuf.flags = I915_EXEC_RENDER;
    
    /* 设置输出 fence */
    execbuf.rsvd2 = syncobj_create.handle;
    
    /* 提交执行 */
    do_ioctl(drm_fd, DRM_IOCTL_I915_GEM_EXECBUFFER2, &execbuf);
    
    /* 等待 fence */
    memset(&syncobj_wait, 0, sizeof(syncobj_wait));
    syncobj_wait.handles = (uintptr_t)&syncobj_create.handle;
    syncobj_wait.handle_count = 1;
    syncobj_wait.timeout_nsec = 5 * NSEC_PER_SEC;
    syncobj_wait.flags = DRM_SYNCOBJ_WAIT_FLAGS_WAIT_ALL;
    
    do_ioctl(drm_fd, DRM_IOCTL_SYNCOBJ_WAIT, &syncobj_wait);
    
    igt_info("Fence signaled, execution completed\n");
    
    /* 清理 */
    struct drm_syncobj_destroy syncobj_destroy = {
        .handle = syncobj_create.handle
    };
    do_ioctl(drm_fd, DRM_IOCTL_SYNCOBJ_DESTROY, &syncobj_destroy);
    gem_close(drm_fd, create.handle);
}

igt_fixture {
    close(drm_fd);
}
```

### 三、案例分析

#### 3.1 Fence 同步机制

```
┌─────────────────────────────────────────────────────────────────────┐
│              Fence 同步机制详解                                      │
│                                                                     │
│  场景：提交两个有依赖关系的命令                                      │
│                                                                     │
│  命令 A（渲染）                                                      │
│     │                                                                │
│     ▼                                                                │
│  ┌─────────────┐                                                    │
│  │ 提交命令 A  │───▶ GPU 执行                                       │
│  │ 创建 Fence 1│                                                    │
│  └─────────────┘                                                    │
│           │                                                          │
│           │ Fence 1 未信号                                           │
│           ▼                                                          │
│  ┌─────────────┐                                                    │
│  │ 提交命令 B  │───▶ 等待 Fence 1                                   │
│  │ 等待 Fence 1│        │                                           │
│  └─────────────┘        │                                           │
│                         ▼                                           │
│                    ┌─────────────┐                                  │
│                    │ Fence 1 信号│                                  │
│                    │ 开始执行 B  │                                  │
│                    └─────────────┘                                  │
│                                                                     │
│  实现方式：                                                          │
│  1. 创建 syncobj（同步对象）                                        │
│  2. 命令 A 提交时指定输出 fence = syncobj                           │
│  3. 命令 B 提交时指定输入 fence = syncobj（等待）                   │
│  4. GPU 完成 A 后自动信号 syncobj                                   │
│  5. GPU 开始执行 B                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 四、相关链接

- IGT GEM 执行测试：https://gitlab.freedesktop.org/drm/igt-gpu-tools/-/tree/master/tests
- DRM Syncobj：https://dri.freedesktop.org/docs/drm/gpu/drm-uapi.html#sync-objects
- Intel GPU 命令：https://01.org/linuxgraphics/documentation

## 今日小结

- GPU 命令提交通过 ioctl 将命令缓冲区提交到 GPU 队列
- gem_exec_basic 验证基本的命令提交功能和参数检查
- gem_exec_fence 验证命令间的 fence 同步机制
- dma_fence/syncobj 是 GPU 命令同步的核心机制
- 测试应覆盖不同 ring、不同同步场景和错误路径

## 扩展思考

1. 在多 GPU 系统中，fence 同步如何跨 GPU 工作？ROCm 的 HIP 流同步与 DRM syncobj 有什么关系？
2. 如何设计一个测试来验证 GPU 命令的超时和取消机制？
