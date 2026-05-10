# [考核] Fuzz 测试技术综合报告

## 考核说明

- 本考核涵盖 Day 426~429 的内容：Fuzz 测试基础、syzkaller、AFL/libFuzzer、结果分析
- 总分 100 分，分为选择题、判断题、简答题和综合实践题
- 选择题每题 2 分，判断题每题 2 分，简答题每题 8 分，综合实践题每题 20 分
- 建议完成时间：120 分钟

---

## 第一部分：选择题（每题 2 分，共 20 分）

**1. syzkaller 主要用于 fuzz 测试以下哪个组件？**

A. 用户空间应用程序
B. Linux 内核驱动
C. Web 服务
D. 数据库系统

**2. 以下哪个内核配置选项是 syzkaller 必需的？**

A. CONFIG_DEBUG_INFO
B. CONFIG_KCOV
C. CONFIG_PERF_EVENTS
D. CONFIG_TRACING

**3. AFL 的核心机制是：**

A. 符号执行
B. 编译时插桩 + 覆盖率反馈
C. 静态分析
D. 模型检查

**4. 以下哪个工具最适合 fuzz Mesa 着色器编译器？**

A. syzkaller
B. AFL
C. smatch
D. sparse

**5. KASAN 的主要功能是：**

A. 性能分析
B. 内存错误检测
C. 并发错误检测
D. 代码覆盖率收集

**6. 在 fuzz 测试中，"语料库"（corpus）指的是：**

A. 测试用例集合
B. 崩溃日志集合
C. 源代码集合
D. 配置文件集合

**7. 以下哪种崩溃类型最严重？**

A. 内存泄露
B. Use-after-free
C. 返回值未检查
D. 代码风格警告

**8. syzkaller 的管理器通过哪个端口提供 Web 界面？**

A. 8080
B. 56741
C. 3000
D. 22

**9. libFuzzer 的 fuzz 目标函数签名是：**

A. int main(int argc, char** argv)
B. int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size)
C. void fuzz(const char* input)
D. int test_input(void* data, int len)

**10. 以下哪个不是 GPU 驱动特有的崩溃类型？**

A. GPU hang
B. 命令超时
C. 空指针解引用
D. 固件错误

---

## 第二部分：判断题（每题 2 分，共 10 分）

**判断以下说法是否正确，并简要说明理由。**

**11.** syzkaller 可以直接在裸金属上运行，不需要虚拟机。

**12.** AFL 的覆盖率反馈机制可以指导 fuzz 引擎优先探索未覆盖的代码路径。

**13.** KASAN 可以在生产环境中启用，对性能影响很小。

**14.** 用户空间 fuzz（AFL）和内核 fuzz（syzkaller）是互补的，应该同时使用。

**15.** 所有 fuzz 发现的崩溃都需要立即修复，不需要区分优先级。

---

## 第三部分：简答题（每题 8 分，共 40 分）

**16. 描述 syzkaller 的架构和工作流程。**

**17. 解释 AFL 的编译时插桩和覆盖率反馈机制。**

**18. 列举至少四种 GPU 驱动崩溃类型，并说明每种类型对应的检测工具。**

**19. 描述如何对 fuzz 发现的崩溃进行最小化和复现。**

**20. 在 GPU 驱动测试中，如何设计一个结合 syzkaller 和 AFL 的完整 fuzz 测试策略？**

---

## 第四部分：综合实践题（每题 15 分，共 30 分）

### 综合实践题一：配置 syzkaller 进行 DRM 驱动 fuzz（15 分）

**背景**：
你需要配置 syzkaller 对 AMDGPU DRM 驱动进行 fuzz 测试。

**要求**：

1. **编写内核配置脚本**（5 分）：
   - 启用 KCOV、KASAN 和 AMDGPU 驱动
   - 禁用可能影响 fuzz 的随机化选项

2. **编写 syzkaller 配置文件**（5 分）：
   - 配置 QEMU 虚拟机参数
   - 列出需要 fuzz 的 DRM ioctl
   - 设置合理的 VM 数量和资源限制

3. **设计监控方案**（5 分）：
   - 如何监控 fuzz 进度？
   - 如何收集和分类崩溃？
   - 如何设置自动化报告？

### 综合实践题二：分析 fuzz 发现的崩溃（15 分）

**背景**：
你在 fuzz AMDGPU 驱动时发现了以下崩溃日志：

```
==1234==ERROR: AddressSanitizer: heap-use-after-free on read
==1234==  at 0xffff888003456789: amdgpu_bo_ref+0x23/0x50 [amdgpu]
==1234==  at 0xffff8880034567ab: amdgpu_cs_submit+0x156/0x890 [amdgpu]
==1234==freed by thread T0 here:
==1234==  at 0xffff8880034567ef: amdgpu_bo_unref+0x45/0x80 [amdgpu]
```

**要求**：

1. **分析崩溃类型和根因**（5 分）
2. **设计修复方案**（5 分）
3. **编写测试用例验证修复**（5 分）

---

## 参考答案

### 第一部分：选择题

1. **B** - syzkaller 专门用于 Linux 内核驱动 fuzz
2. **B** - CONFIG_KCOV 是 syzkaller 必需的覆盖率收集机制
3. **B** - AFL 使用编译时插桩和覆盖率反馈
4. **B** - AFL 适合用户空间组件如 Mesa 编译器
5. **B** - KASAN 是内存错误检测工具
6. **A** - 语料库是测试用例集合
7. **B** - Use-after-free 是严重的内存安全漏洞
8. **B** - syzkaller 默认使用 56741 端口
9. **B** - libFuzzer 使用 LLVMFuzzerTestOneInput
10. **C** - 空指针解引用是通用崩溃类型，不是 GPU 特有

### 第二部分：判断题

11. **错误**。syzkaller 设计为在 QEMU 虚拟机中运行，以隔离崩溃影响。

12. **正确**。AFL 通过插桩收集边覆盖信息，指导变异引擎优先探索新路径。

13. **错误**。KASAN 有显著性能开销（2-3x），不适合生产环境。

14. **正确**。用户空间 fuzz 和内核 fuzz 覆盖不同层面，互补使用更全面。

15. **错误**。崩溃应按严重程度和影响范围区分优先级，不是全部立即修复。

### 第三部分：简答题

**16. syzkaller 架构：**

syzkaller 由 syz-manager（主控）和多个 syz-fuzzer（工作进程）组成。syz-manager 管理 QEMU 虚拟机，分发测试程序，收集崩溃信息。syz-fuzzer 在 VM 中生成随机 syscall 序列，通过 KCOV 收集覆盖率，将覆盖新代码的输入加入语料库。

**17. AFL 插桩机制：**

AFL 使用 afl-gcc/afl-clang 在编译时插入覆盖率计数器。运行时记录基本块跳转的边覆盖。变异引擎根据覆盖率反馈，优先选择能覆盖新代码的输入进行深度变异。

**18. GPU 驱动崩溃类型：**

| 类型 | 检测工具 |
|------|---------|
| 内存错误 | KASAN |
| 并发错误 | KCSAN, lockdep |
| GPU hang | dmesg, amdgpu_regs |
| 命令超时 | dmesg, rocm-smi |

**19. 崩溃最小化：**

使用 afl-tmin 自动最小化输入，或手动裁剪输入文件。验证最小化后的输入仍能复现崩溃。记录复现步骤和环境配置。

**20. 完整 fuzz 策略：**

- 内核层：syzkaller fuzz AMDGPU ioctl 接口
- 用户空间：AFL fuzz Mesa 编译器和 libdrm
- 结果关联：用户空间发现的无效输入是否触发内核崩溃
- CI 集成：每日定时运行，自动报告新崩溃

### 第四部分：综合实践题

**综合实践题一参考答案：**

内核配置：
```bash
./scripts/config \
    --enable CONFIG_KCOV \
    --enable CONFIG_KASAN \
    --enable CONFIG_DRM_AMDGPU \
    --disable CONFIG_RANDOMIZE_BASE
```

syzkaller 配置：
```json
{
    "target": "linux/amd64",
    "http": ":56741",
    "workdir": "./workdir",
    "kernel_obj": "./linux",
    "image": "./image.img",
    "sshkey": "./id_rsa",
    "syzkaller": "./syzkaller",
    "procs": 8,
    "type": "qemu",
    "vm": {
        "count": 4,
        "kernel": "./linux/arch/x86/boot/bzImage",
        "cpu": 2,
        "mem": 2048
    },
    "enable_syscalls": [
        "ioctl$DRM_IOCTL_AMDGPU_GEM_CREATE",
        "ioctl$DRM_IOCTL_AMDGPU_CS",
        "ioctl$DRM_IOCTL_AMDGPU_CTX"
    ]
}
```

监控方案：
- Web 界面实时监控覆盖率和崩溃
- 脚本定期检查 crashes 目录
- 自动发送邮件报告新崩溃

**综合实践题二参考答案：**

分析：
- 类型: Use-after-free
- 根因: amdgpu_cs_submit 引用了已被 amdgpu_bo_unref 释放的 BO

修复：
```c
// 在 amdgpu_cs_parser_fini 中确保不再使用 BO
void amdgpu_cs_parser_fini(struct amdgpu_cs_parser *parser) {
    // 先清理提交，再释放 BO
    if (parser->job)
        amdgpu_job_free(parser->job);
    
    for (i = 0; i < parser->nbo; i++)
        amdgpu_bo_unref(&parser->bos[i]);
}
```

测试：
```c
// 测试用例：并发创建和释放 BO
static void test_bo_lifecycle(void **state) {
    // 创建 BO
    // 提交命令引用 BO
    // 释放 BO
    // 验证无 use-after-free
}
```

---

## 评分标准

| 部分 | 分值 | 评分要点 |
|------|------|---------|
| 选择题 | 20 分 | 每题 2 分 |
| 判断题 | 10 分 | 每题 2 分 |
| 简答题 | 40 分 | 每题 8 分 |
| 综合实践题 | 30 分 | 方案合理 15 分 + 实现正确 15 分 |
| **总分** | **100 分** | |
