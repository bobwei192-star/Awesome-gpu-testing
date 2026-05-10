# [考核] IGT GEM 测试系列综合实践

## 考核说明

- 本考核涵盖 Day 446~449 的内容：gem_create、gem_mmap、gem_exec、gem_busy、gem_wait、BO 操作测试
- 总分 100 分，分为选择题、判断题、简答题和综合实践题
- 选择题每题 2 分，判断题每题 2 分，简答题每题 8 分，综合实践题每题 20 分
- 建议完成时间：120 分钟

---

## 第一部分：选择题（每题 2 分，共 20 分）

**1. gem_create 测试主要验证以下哪个功能？**

A. GPU 命令执行
B. GEM 缓冲区创建
C. 显示模式设置
D. 电源管理

**2. gem_mmap 测试验证的是：**

A. GPU 命令提交
B. 缓冲区内存映射
C. 显示输出
D. 中断处理

**3. gem_exec_basic 测试的核心功能是：**

A. 创建 BO
B. 提交 GPU 命令执行
C. 查询 BO 状态
D. 等待 BO 空闲

**4. gem_busy 返回的值表示：**

A. BO 的大小
B. BO 的忙状态
C. BO 的句柄
D. BO 的域

**5. gem_wait 的作用是：**

A. 创建 BO
B. 等待 BO 变为空闲
C. 映射 BO
D. 销毁 BO

**6. 以下哪个不是 GEM 对象的内存域？**

A. GTT
B. VRAM
C. CPU
D. SSD

**7. GPU 命令提交使用的核心 ioctl 是：**

A. DRM_IOCTL_MODE_CREATE_DUMB
B. DRM_IOCTL_I915_GEM_EXECBUFFER2
C. DRM_IOCTL_GEM_CLOSE
D. DRM_IOCTL_VERSION

**8. Fence 同步机制用于：**

A. 内存分配
B. GPU 命令间的同步
C. 设备打开
D. 模式设置

**9. 在 gem_exec_fence 测试中，syncobj 的作用是：**

A. 创建 BO
B. 同步 GPU 命令执行
C. 映射内存
D. 查询状态

**10. BO 的生命周期不包括以下哪个阶段？**

A. 创建
B. 映射
C. 编译
D. 销毁

---

## 第二部分：判断题（每题 2 分，共 10 分）

**判断以下说法是否正确，并简要说明理由。**

**11.** gem_create 测试只验证正常路径，不需要测试无效参数。

**12.** gem_mmap 映射的内存可以直接通过 CPU 读写。

**13.** gem_exec_basic 测试可以验证不同 ring（GFX/BLT/VIDEO）的命令提交。

**14.** gem_busy 返回 0 表示 BO 正在 GPU 上执行。

**15.** gem_wait 的超时参数单位是毫秒。

---

## 第三部分：简答题（每题 8 分，共 40 分）

**16. 描述 GEM 缓冲区的完整生命周期，包括创建、映射、执行、等待和销毁。**

**17. 解释 gem_exec_basic 和 gem_exec_fence 的区别，以及 fence 同步的作用。**

**18. 描述 gem_busy 和 gem_wait 的用途和使用场景。**

**19. 列举 BO 操作测试应覆盖的五个维度（功能、边界、错误、并发、压力）。**

**20. 描述如何设计一个测试来验证 GPU 命令的 fence 同步机制。**

---

## 第四部分：综合实践题（每题 15 分，共 30 分）

### 综合实践题一：分析 GEM 测试代码（15 分）

**背景**：
你需要分析以下 gem_create 测试代码片段：

```c
igt_subtest("invalid-size") {
    struct drm_mode_create_dumb create;
    int ret;
    
    memset(&create, 0, sizeof(create));
    create.width = 0;
    create.height = 0;
    create.bpp = 32;
    
    ret = drmIoctl(drm_fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);
    igt_assert_eq(ret, -1);
    igt_assert_eq(errno, EINVAL);
}
```

**要求**：

1. **解释测试目的**（3 分）
2. **解释断言含义**（4 分）
3. **设计补充测试**（4 分）：
   - 还需要测试哪些无效参数？
   - 如何测试超大大小？
4. **分析潜在问题**（4 分）：
   - 这个测试有什么局限性？
   - 如何改进？

### 综合实践题二：设计完整 BO 测试套件（15 分）

**背景**：
你需要为 AMDGPU 设计一个完整的 BO 操作测试套件。

**要求**：

1. **设计测试用例**（5 分）：
   - 列出至少 6 个子测试
   - 说明每个子测试的验证目标

2. **编写核心代码**（5 分）：
   - 编写 2 个关键子测试的代码
   - 包含设置、执行和清理

3. **设计测试策略**（5 分）：
   - 如何组织快速回归和完整覆盖？
   - 如何处理测试依赖？
   - 如何验证测试结果？

---

## 参考答案

### 第一部分：选择题

1. **B** - gem_create 验证 GEM 缓冲区创建
2. **B** - gem_mmap 验证缓冲区内存映射
3. **B** - gem_exec_basic 验证 GPU 命令提交执行
4. **B** - gem_busy 返回忙状态
5. **B** - gem_wait 等待 BO 空闲
6. **D** - SSD 不是 GEM 内存域
7. **B** - DRM_IOCTL_I915_GEM_EXECBUFFER2 用于命令提交
8. **B** - Fence 用于 GPU 命令间同步
9. **B** - syncobj 用于同步 GPU 命令
10. **C** - 编译不是 BO 生命周期阶段

### 第二部分：判断题

11. **错误**。gem_create 也测试无效参数，如零大小、超大值。

12. **正确**。gem_mmap 映射的内存可以通过 CPU 直接读写。

13. **正确**。gem_exec_basic 可以验证不同 ring 的命令提交。

14. **错误**。gem_busy 返回 0 表示 BO 空闲，非 0 表示忙。

15. **错误**。gem_wait 超时参数单位是纳秒。

### 第三部分：简答题

**16. GEM 缓冲区生命周期：**

- 创建：通过 CREATE_DUMB 或 GEM_CREATE 分配内存
- 映射：通过 MAP_DUMB 和 mmap 映射到用户空间
- 执行：通过 EXECBUFFER 提交 GPU 命令引用 BO
- 等待：通过 GEM_WAIT 等待 BO 空闲
- 销毁：通过 DESTROY_DUMB 或 GEM_CLOSE 释放内存

**17. gem_exec_basic vs gem_exec_fence：**

- gem_exec_basic：验证基本命令提交功能
- gem_exec_fence：验证命令间的 fence 同步

Fence 同步作用：确保命令按正确顺序执行，避免数据竞争。

**18. gem_busy 和 gem_wait：**

- gem_busy：查询 BO 当前忙状态，非阻塞
- gem_wait：阻塞等待 BO 变为空闲，可设置超时

使用场景：
- busy：检查状态，决定是否等待
- wait：确保 BO 空闲后再操作

**19. 五个测试维度：**

- 功能：正常创建、映射、执行、销毁
- 边界：最小/最大大小、对齐要求
- 错误：无效参数、无效句柄
- 并发：多线程/多进程操作
- 压力：大量 BO、长时间运行

**20. Fence 同步测试设计：**

```c
/* 提交命令 A，创建 fence */
/* 提交命令 B，等待 fence A */
/* 验证命令 B 在 A 完成后执行 */
```

### 第四部分：综合实践题

**综合实践题一参考答案：**

1. 测试目的：验证创建 dumb buffer 时零大小参数返回错误

2. 断言含义：
   - igt_assert_eq(ret, -1)：ioctl 应失败
   - igt_assert_eq(errno, EINVAL)：错误码应为无效参数

3. 补充测试：
   - 测试超大大小（~0ULL）
   - 测试无效 bpp（如 0、33）

4. 潜在问题：
   - 只测试了零大小，未测试其他无效值
   - 改进：添加更多边界测试

**综合实践题二参考答案：**

测试用例：
1. create-basic：基本创建和销毁
2. create-domains：不同内存域
3. create-boundary：边界大小
4. create-invalid：无效参数
5. mmap-basic：内存映射
6. concurrent：并发操作

核心代码（create-basic）：
```c
igt_subtest("create-basic") {
    amdgpu_bo_handle bo;
    bo = create_bo(4096, AMDGPU_GEM_DOMAIN_GTT);
    amdgpu_bo_free(bo);
}
```

测试策略：
- 快速回归：运行 create-basic、mmap-basic
- 完整覆盖：运行所有子测试
- 结果验证：检查 pass/fail/skip 状态

---

## 评分标准

| 部分 | 分值 | 评分要点 |
|------|------|---------|
| 选择题 | 20 分 | 每题 2 分 |
| 判断题 | 10 分 | 每题 2 分 |
| 简答题 | 40 分 | 每题 8 分 |
| 综合实践题 | 30 分 | 分析正确 15 分 + 设计合理 15 分 |
| **总分** | **100 分** | |

