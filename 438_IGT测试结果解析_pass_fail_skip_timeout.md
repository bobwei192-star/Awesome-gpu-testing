# IGT 测试结果解析：pass / fail / skip / timeout

## 学习目标

- 深入理解 IGT 测试结果的四种状态及其含义
- 掌握 IGT 测试输出的格式和解析方法
- 学会使用工具自动化解析测试结果
- 了解测试结果与 dmesg 日志的关联分析
- 能够编写结果解析和报告生成脚本

## 知识详解

### 一、概念原理

#### 1.1 测试结果状态机

```
┌─────────────────────────────────────────────────────────────────────┐
│              IGT 测试结果状态机                                      │
│                                                                     │
│   测试开始                                                          │
│      │                                                              │
│      ▼                                                              │
│   ┌─────────────┐                                                   │
│   │ 条件检查    │                                                   │
│   │ (igt_require)│                                                  │
│   └──────┬──────┘                                                   │
│          │                                                          │
│     不满足│满足                                                     │
│          ▼                                                          │
│   ┌─────────────┐    ┌─────────────┐                               │
│   │    SKIP     │    │  执行测试   │                               │
│   │  条件不满足  │    │  (igt_assert)│                              │
│   │             │    └──────┬──────┘                               │
│   │ 原因：      │           │                                       │
│   │ ● 硬件不支持│      通过│失败                                    │
│   │ ● 驱动版本  │           ▼                                       │
│   │ ● 内核配置  │    ┌─────────────┐    ┌─────────────┐           │
│   │             │    │    PASS     │    │    FAIL     │           │
│   └─────────────┘    │  断言通过   │    │  断言失败   │           │
│                      │             │    │             │           │
│                      │ 耗时记录    │    │ 错误信息    │           │
│                      └─────────────┘    │ 调用栈      │           │
│                                         └─────────────┘           │
│                                                                     │
│   特殊情况：                                                         │
│   ┌─────────────────────────────────────────────────────────┐      │
│   │  TIMEOUT: 测试执行超过预设时间（默认 30-300 秒）         │      │
│   │  CRASH:   测试进程崩溃（SIGSEGV/SIGABRT 等）             │      │
│   │  DMESG:   dmesg 中出现 WARN/ERROR（可配置为失败条件）    │      │
│   └─────────────────────────────────────────────────────────┘      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.2 结果输出格式

```
┌─────────────────────────────────────────────────────────────────────┐
│              IGT 结果输出格式                                        │
│                                                                     │
│  文本格式（默认）：                                                   │
│  ─────────────────                                                   │
│  IGT-Version: 1.28-gf7a3b2c (x86_64) (Linux: 6.8.0-amd64)          │
│  Starting subtest: create-close                                     │
│  Subtest create-close: SUCCESS (0.002s)                             │
│  Test gem_basic: SUCCESS                                            │
│                                                                     │
│  JSON 格式：                                                         │
│  ──────────                                                         │
│  {                                                                  │
│    "igt_version": "1.28",                                          │
│    "kernel_version": "6.8.0",                                      │
│    "tests": [                                                       │
│      {                                                              │
│        "name": "gem_basic",                                        │
│        "subtests": [                                                │
│          {                                                          │
│            "name": "create-close",                                 │
│            "result": "pass",                                       │
│            "time": 0.002                                           │
│          }                                                          │
│        ]                                                            │
│      }                                                              │
│    ]                                                                │
│  }                                                                  │
│                                                                     │
│  JUnit XML 格式：                                                    │
│  ──────────────                                                     │
│  <?xml version="1.0"?>                                             │
│  <testsuites>                                                       │
│    <testsuite name="igt">                                          │
│      <testcase name="gem_basic/create-close"                       │
│                   time="0.002">                                    │
│      </testcase>                                                    │
│    </testsuite>                                                     │
│  </testsuites>                                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 二、实践操作

#### 2.1 结果解析脚本

```bash
#!/bin/bash
# parse_igt_output.sh - IGT 输出解析脚本

set -e

LOG_FILE="$1"

if [ -z "$LOG_FILE" ]; then
    echo "用法: $0 <log-file>"
    exit 1
fi

echo "=== IGT 结果解析 ==="
echo "日志文件: $LOG_FILE"
echo ""

# 1. 提取 IGT 版本
echo "【1】IGT 版本信息"
grep "IGT-Version" "$LOG_FILE" | head -1 || echo "未找到版本信息"
echo ""

# 2. 统计结果
echo "【2】结果统计"
PASS_COUNT=$(grep -c "Subtest.*SUCCESS" "$LOG_FILE" || true)
FAIL_COUNT=$(grep -c "Subtest.*FAIL" "$LOG_FILE" || true)
SKIP_COUNT=$(grep -c "Subtest.*SKIP" "$LOG_FILE" || true)
TIMEOUT_COUNT=$(grep -c "Subtest.*TIMEOUT" "$LOG_FILE" || true)

echo "  通过: $PASS_COUNT"
echo "  失败: $FAIL_COUNT"
echo "  跳过: $SKIP_COUNT"
echo "  超时: $TIMEOUT_COUNT"
echo ""

# 3. 列出失败测试
echo "【3】失败测试详情"
if [ "$FAIL_COUNT" -gt 0 ]; then
    grep "Subtest.*FAIL" "$LOG_FILE" | while read line; do
        echo "  - $line"
    done
else
    echo "  无失败测试"
fi
echo ""

# 4. 列出跳过测试
echo "【4】跳过测试详情"
if [ "$SKIP_COUNT" -gt 0 ]; then
    grep "Subtest.*SKIP" "$LOG_FILE" | while read line; do
        echo "  - $line"
    done
else
    echo "  无跳过测试"
fi
echo ""

# 5. 提取耗时信息
echo "【5】耗时信息"
grep "Subtest.*SUCCESS.*s)" "$LOG_FILE" | tail -5 || true
echo ""

# 6. 检查 dmesg 错误
echo "【6】dmesg 错误检查"
if [ -f "${LOG_FILE}.dmesg" ]; then
    DMESG_ERRORS=$(grep -c "ERROR\|WARN" "${LOG_FILE}.dmesg" || true)
    echo "  dmesg 错误/警告数: $DMESG_ERRORS"
else
    echo "  未找到 dmesg 日志"
fi
echo ""
```

#### 2.2 结果对比脚本

```bash
#!/bin/bash
# compare_igt_results.sh - 对比两次 IGT 结果

set -e

RESULT_A="$1"
RESULT_B="$2"

if [ -z "$RESULT_A" ] || [ -z "$RESULT_B" ]; then
    echo "用法: $0 <result-a> <result-b>"
    exit 1
fi

echo "=== IGT 结果对比 ==="
echo "基线: $RESULT_A"
echo "当前: $RESULT_B"
echo ""

# 提取所有子测试结果
extract_results() {
    local file="$1"
    grep "Subtest" "$file" | sed 's/Subtest \(.*\): \(.*\) .*/\1:\2/'
}

# 保存到临时文件
TMP_A=$(mktemp)
TMP_B=$(mktemp)
extract_results "$RESULT_A" > "$TMP_A"
extract_results "$RESULT_B" > "$TMP_B"

# 找出差异
echo "【1】新增失败"
comm -23 <(grep ":FAIL" "$TMP_A" | sort) <(grep ":FAIL" "$TMP_B" | sort) | while read line; do
    echo "  - $line"
done

echo ""
echo "【2】修复的测试"
comm -13 <(grep ":FAIL" "$TMP_A" | sort) <(grep ":FAIL" "$TMP_B" | sort) | while read line; do
    echo "  - $line"
done

echo ""
echo "【3】状态变更"
# 使用 awk 对比
awk -F: '
    NR==FNR {a[$1]=$2; next}
    {
        if ($1 in a && a[$1] != $2) {
            print "  - " $1 ": " a[$1] " -> " $2
        }
    }
' "$TMP_A" "$TMP_B"

rm -f "$TMP_A" "$TMP_B"

echo ""
```

### 三、案例分析

#### 3.1 分析失败案例

```
示例：gem_basic 子测试 create-large 失败

日志输出：
Starting subtest: create-large
(gem_basic:1234): CRITICAL **: Assertion failure: create.handle != 0
Subtest create-large: FAIL (0.001s)

分析步骤：
1. 检查 dmesg：
   [drm:amdgpu_gem_create_ioctl] *ERROR* Failed to allocate GEM object (256MB)
   [drm:amdgpu_gem_create_ioctl] *ERROR* Not enough VRAM

2. 根因：GPU 显存不足，无法分配 256MB 的 BO

3. 解决方案：
   - 检查 GPU 显存大小
   - 关闭占用显存的应用程序
   - 或跳过该测试（如果显存确实不足）
```

### 四、相关链接

- IGT 结果格式文档：https://drm.pages.freedesktop.org/igt-gpu-tools/igt-gpu-tools-Results.html
- JUnit 格式规范：https://llg.cubic.org/docs/junit/

## 今日小结

- IGT 测试结果有五种状态：PASS、FAIL、SKIP、TIMEOUT、CRASH
- 结果输出支持文本、JSON、JUnit XML 三种格式
- 失败测试需要结合 dmesg 日志进行根因分析
- 结果对比可以识别回归和修复
- 自动化解析脚本可以提高测试效率

## 扩展思考

1. 如何处理 flaky 测试（有时通过有时失败）的结果？是否需要引入重试机制？
2. 在大型测试套件中，如何快速定位导致失败的测试子集？
