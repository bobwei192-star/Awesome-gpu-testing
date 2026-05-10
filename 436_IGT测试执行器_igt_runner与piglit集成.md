# IGT 测试执行器：igt_runner 与 piglit 集成

## 学习目标

- 理解 igt_runner 的设计目标和架构
- 掌握 igt_runner 的命令行参数和使用方法
- 了解 piglit 测试框架及其与 IGT 的集成方式
- 学会配置和运行 IGT 测试套件
- 能够解析和生成 IGT 测试结果报告

## 知识详解

### 一、概念原理

#### 1.1 测试执行器架构

```
┌─────────────────────────────────────────────────────────────────────┐
│              IGT 测试执行器架构                                      │
│                                                                     │
│   igt_runner                                                        │
│   ├── 命令行解析                                                      │
│   │   ├── --list-tests       列出可用测试                            │
│   │   ├── --test-list        指定测试列表文件                        │
│   │   ├── --output-format    输出格式 (text/json/junit)              │
│   │   ├── --output-file      输出文件路径                            │
│   │   ├── --timeout          全局超时设置                            │
│   │   └── --parallel         并行执行数量                            │
│   │                                                                   │
│   ├── 测试发现                                                      │
│   │   ├── 扫描 tests/ 目录                                           │
│   │   ├── 解析测试元数据                                             │
│   │   └── 过滤和排序                                                 │
│   │                                                                   │
│   ├── 测试执行                                                      │
│   │   ├── 顺序执行 / 并行执行                                        │
│   │   ├── 超时控制                                                   │
│   │   ├── 结果收集                                                   │
│   │   └── 日志捕获                                                   │
│   │                                                                   │
│   └── 报告生成                                                      │
│       ├── 文本报告                                                   │
│       ├── JSON 报告                                                  │
│       ├── JUnit XML 报告                                             │
│       └── Piglit 格式报告                                            │
│                                                                     │
│   数据来源：IGT runner 源码                                          │
│   https://gitlab.freedesktop.org/drm/igt-gpu-tools/-/tree/master/runner│
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.2 piglit 集成

```
┌─────────────────────────────────────────────────────────────────────┐
│              piglit 与 IGT 集成                                      │
│                                                                     │
│   piglit 框架                                                       │
│   ├── 测试发现（test profiles）                                      │
│   ├── 测试执行（concurrent test runner）                             │
│   ├── 结果聚合（results aggregation）                                │
│   └── 报告对比（results comparison）                                 │
│                                                                     │
│   IGT 作为 piglit 的外部测试：                                       │
│   ┌─────────────────────────────────────────────────────────┐      │
│   │  piglit profile                                         │      │
│   │  ├── tests/igt.py          IGT 测试包装器               │      │
│   │  │   ├── 调用 igt_runner                                │      │
│   │  │   ├── 解析 IGT 输出                                  │      │
│   │  │   └── 转换为 piglit 结果格式                         │      │
│   │  │                                                      │      │
│   │  └── 结果文件                                           │      │
│   │      ├── IGT 原始输出                                   │      │
│   │      └── piglit 汇总报告                                │      │
│   └─────────────────────────────────────────────────────────┘      │
│                                                                     │
│   优势：                                                            │
│   ● piglit 提供跨测试框架的统一报告                                │
│   ● 支持 Mesa、VK-GL-CTS 等多框架结果对比                          │
│   ● 历史结果跟踪和回归检测                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 二、实践操作

#### 2.1 igt_runner 使用

```bash
#!/bin/bash
# run_igt_tests.sh - IGT 测试执行脚本

set -e

IGT_BUILD="${1:-$HOME/igt-gpu-tools/build}"
OUTPUT_DIR="${2:-./igt_results}"
mkdir -p "$OUTPUT_DIR"

echo "=== IGT 测试执行 ==="
echo "IGT 构建目录: $IGT_BUILD"
echo "输出目录: $OUTPUT_DIR"
echo ""

# 1. 列出所有可用测试
echo "【1】列出可用测试"
"$IGT_BUILD/runner/igt_runner" --list-tests "$IGT_BUILD/tests" | head -20
echo ""

# 2. 运行快速回归测试
echo "【2】运行快速回归测试"
"$IGT_BUILD/runner/igt_runner" \
    --test-list "$IGT_BUILD/tests/test-list.txt" \
    --output-format json \
    --output-file "$OUTPUT_DIR/quick_results.json" \
    --timeout 300 \
    "$IGT_BUILD/tests" 2>&1 | tee "$OUTPUT_DIR/quick_run.log"

echo ""

# 3. 运行指定测试
echo "【3】运行指定测试 (gem_basic)"
"$IGT_BUILD/runner/igt_runner" \
    --run-subtest "create-close" \
    --output-format text \
    "$IGT_BUILD/tests/gem_basic" 2>&1 | tee "$OUTPUT_DIR/gem_basic.log"

echo ""

# 4. 生成 JUnit 报告（CI 集成）
echo "【4】生成 JUnit 报告"
"$IGT_BUILD/runner/igt_runner" \
    --test-list "$IGT_BUILD/tests/test-list.txt" \
    --output-format junit \
    --output-file "$OUTPUT_DIR/junit_results.xml" \
    "$IGT_BUILD/tests" 2>/dev/null || true

echo ""
echo "=== 测试执行完成 ==="
echo "结果保存在: $OUTPUT_DIR"
```

#### 2.2 测试列表配置

```bash
#!/bin/bash
# create_test_list.sh - 创建自定义测试列表

set -e

IGT_BUILD="${1:-$HOME/igt-gpu-tools/build}"
OUTPUT="${2:-custom-test-list.txt}"

echo "=== 创建自定义测试列表 ==="
echo ""

# 创建快速回归测试列表
cat > "$OUTPUT" << 'EOF'
# IGT 快速回归测试列表
# 格式: 测试路径 [子测试名称]

# GEM 基础测试
gem_basic
gem_create
gem_exec_basic
gem_exec_fence
gem_busy
gem_wait

# KMS 基础测试
kms_pipe_crc_basic
kms_flip
kms_cursor_crc

# AMDGPU 基础测试
amdgpu/amd_basic
amdgpu/amd_cs_nop
amdgpu/amd_prime
EOF

echo "测试列表已创建: $OUTPUT"
echo ""
echo "内容预览:"
cat "$OUTPUT"
```

#### 2.3 结果解析脚本

```bash
#!/bin/bash
# parse_igt_results.sh - 解析 IGT 测试结果

set -e

RESULT_FILE="$1"

if [ -z "$RESULT_FILE" ]; then
    echo "用法: $0 <result-file.json>"
    exit 1
fi

echo "=== IGT 结果解析 ==="
echo "文件: $RESULT_FILE"
echo ""

# 使用 jq 解析 JSON 结果（如果可用）
if command -v jq &> /dev/null; then
    echo "总体统计:"
    jq -r '
        "总测试数: \(.tests | length)",
        "通过: \([.tests[] | select(.result == "pass")] | length)",
        "失败: \([.tests[] | select(.result == "fail")] | length)",
        "跳过: \([.tests[] | select(.result == "skip")] | length)",
        "超时: \([.tests[] | select(.result == "timeout")] | length)"
    ' "$RESULT_FILE" 2>/dev/null || echo "JSON 解析失败"
    
    echo ""
    echo "失败测试详情:"
    jq -r '.tests[] | select(.result == "fail") | "  - \(.name): \(.result)"' "$RESULT_FILE" 2>/dev/null || true
else
    echo "jq 未安装，使用 grep 解析"
    grep -o '"result":"[^"]*"' "$RESULT_FILE" | sort | uniq -c
fi

echo ""
```

### 三、案例分析

#### 3.1 CI 集成配置

```yaml
# .gitlab-ci.yml - IGT 测试集成示例

igt_regression:
  stage: test
  tags:
    - gpu-amd
    - baremetal
  timeout: 2 hours
  script:
    # 运行 IGT 快速回归测试
    - |
      $IGT_BUILD/runner/igt_runner \
        --test-list tests/quick-tests.txt \
        --output-format junit \
        --output-file igt_results.xml \
        --timeout 300 \
        $IGT_BUILD/tests
    
    # 解析结果
    - ./scripts/parse_igt_results.sh igt_results.xml
  artifacts:
    paths:
      - igt_results.xml
      - igt_run.log
    reports:
      junit: igt_results.xml
    expire_in: 1 month
  allow_failure: false
```

### 四、相关链接

- IGT runner 文档：https://drm.pages.freedesktop.org/igt-gpu-tools/igt-gpu-tools-Runner.html
- piglit 项目：https://gitlab.freedesktop.org/mesa/piglit
- JUnit 报告格式：https://llg.cubic.org/docs/junit/

## 今日小结

- igt_runner 是 IGT 的官方测试执行器，支持测试发现、执行和报告生成
- igt_runner 支持多种输出格式（text/json/junit），便于 CI 集成
- piglit 是 Mesa 项目的测试框架，可以集成 IGT 作为外部测试
- 测试列表文件用于自定义回归测试套件
- JUnit XML 格式是 CI 系统广泛支持的测试结果格式

## 扩展思考

1. 如何设计一个智能的测试选择策略，根据代码变更自动选择相关的 IGT 测试？
2. 在并行执行 IGT 测试时，如何处理测试间的资源冲突（如 DRM 设备独占访问）？
