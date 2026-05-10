# Fuzz 测试结果分析与 Bug 分类

## 学习目标

- 掌握 Fuzz 测试发现崩溃的分类方法和优先级判定
- 学会分析崩溃日志，定位根因和影响范围
- 了解 GPU 驱动特有的崩溃类型（GPU hang、命令超时等）
- 掌握崩溃复现和最小化测试用例的技术
- 能够编写规范的漏洞报告

## 知识详解

### 一、概念原理

#### 1.1 GPU 驱动崩溃类型分类

```
┌─────────────────────────────────────────────────────────────────────┐
│              GPU 驱动崩溃类型分类                                    │
│                                                                     │
│  类别          子类型              严重程度    典型症状              │
│  ─────────────────────────────────────────────────────────────      │
│                                                                     │
│  内存安全      空指针解引用        严重        Oops / KASAN         │
│  (Memory)      越界访问            严重        KASAN / UBSAN        │
│                Use-after-free      严重        KASAN                │
│                内存泄露            中等        kmemleak             │
│                                                                     │
│  并发错误      死锁                严重        系统挂起             │
│  (Concurrency) 竞态条件            严重        数据损坏             │
│                原子操作错误        中等        计数器异常           │
│                                                                     │
│  硬件交互      GPU hang            严重        驱动恢复 / 系统崩溃  │
│  (Hardware)    命令超时            中等        dmesg 错误日志       │
│                寄存器访问错误      严重        硬件异常             │
│                固件错误            严重        PSP/SMU 超时         │
│                                                                     │
│  逻辑错误      资源泄露            中等        资源耗尽             │
│  (Logic)       状态不一致          中等        功能异常             │
│                返回值未检查        轻微        静默失败             │
│                                                                     │
│  安全漏洞      信息泄露            严重        内核信息暴露         │
│  (Security)    权限提升            严重        用户获得 root        │
│                拒绝服务            中等        系统不可用           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.2 崩溃分析流程

```
┌─────────────────────────────────────────────────────────────────────┐
│              崩溃分析流程                                            │
│                                                                     │
│   发现崩溃                                                          │
│      │                                                              │
│      ▼                                                              │
│   ┌─────────────┐                                                   │
│   │ 1. 收集信息  │                                                   │
│   │             │                                                   │
│   │ ● 崩溃日志   │                                                   │
│   │ ● dmesg     │                                                   │
│   │ ● 复现输入   │                                                   │
│   │ ● 系统状态   │                                                   │
│   └──────┬──────┘                                                   │
│          │                                                          │
│          ▼                                                          │
│   ┌─────────────┐                                                   │
│   │ 2. 分类崩溃  │                                                   │
│   │             │                                                   │
│   │ ● 内存安全?  │──是──▶ 使用 KASAN/UBSAN 定位                     │
│   │ ● 并发错误?  │──是──▶ 使用 KCSAN/lockdep 定位                   │
│   │ ● 硬件交互?  │──是──▶ 检查寄存器/固件状态                       │
│   │ ● 逻辑错误?  │──是──▶ 代码审查 + 调试                           │
│   └──────┬──────┘                                                   │
│          │                                                          │
│          ▼                                                          │
│   ┌─────────────┐                                                   │
│   │ 3. 最小化    │                                                   │
│   │             │                                                   │
│   │ ● 使用 afl-tmin│                                                 │
│   │ ● 手动裁剪   │                                                   │
│   │ ● 确认最小   │                                                   │
│   │   复现条件   │                                                   │
│   └──────┬──────┘                                                   │
│          │                                                          │
│          ▼                                                          │
│   ┌─────────────┐                                                   │
│   │ 4. 定位根因  │                                                   │
│   │             │                                                   │
│   │ ● 代码审查   │                                                   │
│   │ ● 添加调试   │                                                   │
│   │ ● 验证假设   │                                                   │
│   └──────┬──────┘                                                   │
│          │                                                          │
│          ▼                                                          │
│   ┌─────────────┐                                                   │
│   │ 5. 编写报告  │                                                   │
│   │             │                                                   │
│   │ ● 描述清晰   │                                                   │
│   │ ● 复现步骤   │                                                   │
│   │ ● 影响评估   │                                                   │
│   │ ● 修复建议   │                                                   │
│   └─────────────┘                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 二、实践操作

#### 2.1 崩溃日志分析脚本

```bash
#!/bin/bash
# analyze_crash.sh - 崩溃日志分析脚本

set -e

CRASH_LOG="$1"

if [ -z "$CRASH_LOG" ]; then
    echo "用法: $0 <crash-log-file>"
    exit 1
fi

echo "=== 崩溃日志分析 ==="
echo "文件: $CRASH_LOG"
echo ""

# 1. 检测崩溃类型
echo "## 1. 崩溃类型检测"

if grep -q "KASAN" "$CRASH_LOG"; then
    echo "类型: KASAN 内存错误"
    
    if grep -q "out-of-bounds" "$CRASH_LOG"; then
        echo "子类型: 越界访问"
    elif grep -q "use-after-free" "$CRASH_LOG"; then
        echo "子类型: Use-after-free"
    elif grep -q "stack-out-of-bounds" "$CRASH_LOG"; then
        echo "子类型: 栈越界"
    fi
    
    # 提取访问大小和地址
    grep -oP "accessed \d+ bytes" "$CRASH_LOG" || true
    grep -oP "at addr [0-9a-fx]+" "$CRASH_LOG" || true
    
elif grep -q "BUG:" "$CRASH_LOG"; then
    echo "类型: 内核 BUG"
    grep "BUG:" "$CRASH_LOG" | head -1
    
elif grep -q "WARNING:" "$CRASH_LOG"; then
    echo "类型: 内核 WARNING"
    grep "WARNING:" "$CRASH_LOG" | head -1
    
elif grep -q "general protection fault" "$CRASH_LOG"; then
    echo "类型: 通用保护错误"
    
elif grep -q " amdgpu:.*hang" "$CRASH_LOG"; then
    echo "类型: GPU hang"
    
elif grep -q " amdgpu:.*timeout" "$CRASH_LOG"; then
    echo "类型: 命令超时"
    
else
    echo "类型: 未知"
fi

echo ""

# 2. 提取调用栈
echo "## 2. 调用栈"
if grep -q "Call Trace:" "$CRASH_LOG"; then
    grep -A 30 "Call Trace:" "$CRASH_LOG" | head -30
else
    echo "未找到调用栈"
fi
echo ""

# 3. 提取 GPU 相关信息
echo "## 3. GPU 相关信息"
if grep -q "amdgpu" "$CRASH_LOG"; then
    echo "涉及 AMDGPU 驱动"
    grep "amdgpu" "$CRASH_LOG" | head -10
fi
echo ""

# 4. 提取寄存器状态
echo "## 4. 寄存器状态"
if grep -q "RIP:" "$CRASH_LOG"; then
    grep "RIP:" "$CRASH_LOG" | head -1
    grep "RSP:" "$CRASH_LOG" | head -1
fi
echo ""

# 5. 生成分析报告
echo "## 5. 分析摘要"
echo "- 崩溃文件: $CRASH_LOG"
echo "- 分析时间: $(date)"
echo "- 建议: 根据崩溃类型进行进一步分析"
```

#### 2.2 崩溃最小化

```bash
#!/bin/bash
# minimize_crash.sh - 崩溃输入最小化

set -e

CRASH_INPUT="$1"
TARGET_BIN="$2"

if [ -z "$CRASH_INPUT" ] || [ -z "$TARGET_BIN" ]; then
    echo "用法: $0 <crash-input> <target-binary>"
    exit 1
fi

echo "=== 崩溃输入最小化 ==="
echo "原始输入: $CRASH_INPUT"
echo "目标程序: $TARGET_BIN"
echo ""

# 使用 afl-tmin 最小化
if command -v afl-tmin &> /dev/null; then
    echo "使用 afl-tmin 最小化..."
    afl-tmin -i "$CRASH_INPUT" -o "$CRASH_INPUT.min" -- "$TARGET_BIN" @@
    
    ORIG_SIZE=$(stat -c%s "$CRASH_INPUT")
    MIN_SIZE=$(stat -c%s "$CRASH_INPUT.min")
    
    echo "原始大小: $ORIG_SIZE bytes"
    echo "最小化后: $MIN_SIZE bytes"
    echo "压缩比: $((100 * MIN_SIZE / ORIG_SIZE))%"
else
    echo "afl-tmin 不可用，使用手动最小化"
    cp "$CRASH_INPUT" "$CRASH_INPUT.min"
fi
```

### 三、案例分析

#### 3.1 KASAN 内存错误分析

```
=================================================================
==1234==ERROR: AddressSanitizer: heap-use-after-free on read
==1234==  at 0xffff888003456789: amdgpu_bo_ref+0x23/0x50 [amdgpu]
==1234==  at 0xffff8880034567ab: amdgpu_cs_submit+0x156/0x890 [amdgpu]
==1234==  at 0xffff8880034567cd: amdgpu_cs_ioctl+0x234/0x450 [amdgpu]
==1234==  
==1234==freed by thread T0 here:
==1234==  at 0xffff8880034567ef: amdgpu_bo_unref+0x45/0x80 [amdgpu]
==1234==  at 0xffff888003456811: amdgpu_cs_parser_fini+0x89/0x120 [amdgpu]
==1234==  
==1234==previously allocated here:
==1234==  at 0xffff888003456833: amdgpu_bo_create+0x123/0x340 [amdgpu]
==1234==  at 0xffff888003456855: amdgpu_gem_create_ioctl+0x67/0x150 [amdgpu]

分析：
1. 类型: Use-after-free（严重）
2. 根因: amdgpu_cs_submit 中引用了已被释放的 BO
3. 修复: 在 amdgpu_cs_parser_fini 中确保不再使用 BO 引用
```

### 四、相关链接

- KASAN 文档：https://docs.kernel.org/dev-tools/kasan.html
- KCSAN 文档：https://docs.kernel.org/dev-tools/kcsan.html
- lockdep 文档：https://docs.kernel.org/locking/lockdep-design.html
- 内核崩溃分析：https://docs.kernel.org/admin-guide/bug-hunting.html

## 今日小结

- GPU 驱动崩溃分为内存安全、并发错误、硬件交互、逻辑错误和安全漏洞五类
- 崩溃分析流程包括信息收集、分类、最小化、根因定位和报告编写
- KASAN、KCSAN、lockdep 等工具可辅助定位不同类型的崩溃
- 崩溃最小化是验证漏洞和编写报告的关键步骤

## 扩展思考

1. 如何自动化崩溃分类和优先级判定？是否可以训练机器学习模型来辅助分析？
2. GPU hang 和命令超时等特殊崩溃类型如何与标准内存错误区分处理？
