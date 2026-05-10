# gcov / lcov 代码覆盖率分析基础

## 学习目标

- 理解代码覆盖率在 GPU 驱动测试中的意义和局限性
- 掌握 gcov 内核覆盖率收集的配置和使用方法
- 学会使用 lcov 生成可视化的覆盖率报告
- 了解如何解读覆盖率数据并识别测试盲区
- 能够将覆盖率分析集成到 CI 流水线中

## 知识详解

### 一、概念原理

#### 1.1 代码覆盖率的概念与类型

代码覆盖率（Code Coverage）度量测试用例执行了多少比例的源代码。在 GPU 驱动测试中，覆盖率是评估测试充分性的重要指标：

```
┌─────────────────────────────────────────────────────────────────────┐
│              代码覆盖率类型与 GPU 驱动应用                           │
│                                                                     │
│  覆盖率类型        定义                  GPU 驱动应用               │
│  ─────────────────────────────────────────────────────────────      │
│                                                                     │
│  语句覆盖          执行的语句比例        基础指标，容易达成         │
│  (Statement)       ├─ if 条件分支        但高语句覆盖不等于         │
│                    ├─ 循环体             高质量测试                 │
│                    └─ 函数调用                                          │
│                                                                     │
│  分支覆盖          执行的分支比例        更重要：GPU 驱动中         │
│  (Branch)          ├─ if/else 分支       错误处理路径往往           │
│                    ├─ switch case        包含关键逻辑               │
│                    └─ 三元运算符       目标：>80% 分支覆盖         │
│                                                                     │
│  函数覆盖          调用的函数比例        确保所有 API 入口          │
│  (Function)        ├─ 导出函数         都有基础测试                 │
│                    ├─ 内部函数                                           │
│                    └─ 回调函数                                          │
│                                                                     │
│  行覆盖            执行的代码行比例      最常用指标                 │
│  (Line)            ├─ 可执行行         与语句覆盖类似               │
│                    └─ 声明行                                           │
│                                                                     │
│  条件覆盖          布尔子条件            最严格：每个子条件         │
│  (Condition)       ├─ a && b             都独立为 true/false        │
│                    └─ a || b           目标：关键路径 100%         │
│                                                                     │
│  路径覆盖          执行的路径比例        理论上最完整               │
│  (Path)            ├─ 所有可能路径     实际不可行（指数爆炸）       │
│                    └─ 独立路径                                          │
│                                                                     │
│  GPU 驱动测试目标：                                                  │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  语句覆盖 > 80%    分支覆盖 > 70%    函数覆盖 > 90%     │       │
│  │  错误处理路径 > 60%  关键函数 100%    新代码 100%       │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**重要提醒**：高覆盖率不等于高质量测试。覆盖率只能回答"测了多少"，不能回答"测得好不好"。

#### 1.2 内核覆盖率收集的挑战

内核代码覆盖率收集比用户空间程序更复杂：

```
┌─────────────────────────────────────────────────────────────────────┐
│              内核覆盖率收集的挑战与解决方案                          │
│                                                                     │
│  挑战                    原因                  解决方案             │
│  ─────────────────────────────────────────────────────────────      │
│                                                                     │
│  需要重新编译内核        gcov 需要编译时        配置 CONFIG_GCOV    │
│  启用 gcov 支持          插入计数器代码         使用特殊构建        │
│                                                                     │
│  多文件覆盖聚合          内核包含数千个         使用 lcov 聚合      │
│                          源文件                 或 genhtml 生成报告 │
│                                                                     │
│  模块动态加载            驱动可作为模块         分别收集 vmlinux   │
│                          动态加载               和 .ko 覆盖率       │
│                                                                     │
│  覆盖率数据持久化        内核崩溃时数据         定期同步到文件      │
│                          可能丢失               系统或 sysrq        │
│                                                                     │
│  性能开销                gcov 计数器增加        仅在调试构建        │
│                          运行时开销             启用，发布构建禁用  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 二、实践操作

#### 2.1 内核 gcov 配置与编译

```bash
#!/bin/bash
# kernel_gcov_setup.sh - 内核 gcov 覆盖率收集配置
# 参考：内核文档 Documentation/dev-tools/gcov.rst

set -e

KERNEL_DIR="${1:-/usr/src/linux}"
cd "$KERNEL_DIR"

echo "=== 内核 gcov 覆盖率配置 ==="
echo ""

# 1. 配置内核启用 gcov
echo "## 1. 启用内核 gcov 配置"
cat >> .config << 'EOF'
# GCOV 覆盖率支持
CONFIG_DEBUG_FS=y
CONFIG_GCOV_KERNEL=y
# 使用内核默认的 gcov 格式（与 gcc 兼容）
CONFIG_GCOV_FORMAT_AUTODETECT=y
# 或明确指定格式
# CONFIG_GCOV_FORMAT_4_7=y
EOF

echo "已添加 gcov 配置到 .config"
echo ""

# 2. 验证配置
echo "## 2. 验证配置"
make oldconfig

echo "配置验证结果:"
grep CONFIG_GCOV .config || true
echo ""

# 3. 编译带覆盖率的内核
echo "## 3. 编译内核"
make -j$(nproc)

echo ""
echo "编译完成。覆盖率数据将在运行时收集。"
echo ""

# 4. 挂载 debugfs（gcov 数据通过 debugfs 暴露）
echo "## 4. 挂载 debugfs"
if ! mountpoint -q /sys/kernel/debug; then
    sudo mount -t debugfs none /sys/kernel/debug
    echo "debugfs 已挂载"
else
    echo "debugfs 已挂载"
fi

echo ""
echo "gcov 数据位置: /sys/kernel/debug/gcov/"
ls -la /sys/kernel/debug/gcov/ 2>/dev/null || echo "请重启到新内核后查看"
```

#### 2.2 覆盖率数据收集与报告生成

```bash
#!/bin/bash
# collect_coverage.sh - 覆盖率数据收集脚本
# 参考：lcov 官方文档 (https://github.com/linux-test-project/lcov)

set -e

KERNEL_DIR="${1:-/usr/src/linux}"
OUTPUT_DIR="${2:-./coverage_report}"
mkdir -p "$OUTPUT_DIR"

echo "=== 覆盖率数据收集 ==="
echo "内核目录: $KERNEL_DIR"
echo "输出目录: $OUTPUT_DIR"
echo ""

# 1. 从 debugfs 复制 gcov 数据
echo "## 1. 收集 gcov 数据"
GCOV_DIR="/sys/kernel/debug/gcov"

if [ ! -d "$GCOV_DIR" ]; then
    echo "错误: $GCOV_DIR 不存在，请确保："
    echo "  1. 内核启用了 CONFIG_GCOV_KERNEL"
    echo "  2. debugfs 已挂载"
    exit 1
fi

# 创建临时目录存放 gcov 数据
TMP_DIR=$(mktemp -d)
trap "rm -rf $TMP_DIR" EXIT

# 复制 gcov 数据（包括 .gcno 和 .gcda 文件）
rsync -av "$GCOV_DIR/" "$TMP_DIR/" 2>/dev/null || \
cp -r "$GCOV_DIR"/* "$TMP_DIR/" 2>/dev/null || true

echo "gcov 数据已复制到: $TMP_DIR"
echo ""

# 2. 使用 lcov 生成覆盖率信息
echo "## 2. 生成 lcov 信息文件"

# 初始化 lcov 数据
lcov --capture --directory "$TMP_DIR" \
     --output-file "$OUTPUT_DIR/coverage.info" \
     --rc lcov_branch_coverage=1 2>&1 | tail -20

echo ""

# 3. 过滤只保留 AMDGPU 相关代码
echo "## 3. 过滤 AMDGPU 代码"
lcov --extract "$OUTPUT_DIR/coverage.info" \
     "*/drivers/gpu/drm/amd/*" \
     --output-file "$OUTPUT_DIR/amdgpu_coverage.info" \
     --rc lcov_branch_coverage=1 2>&1 | tail -10

echo ""

# 4. 移除无关代码（如测试代码自身）
echo "## 4. 清理数据"
lcov --remove "$OUTPUT_DIR/amdgpu_coverage.info" \
     "*/tests/*" \
     "*/usr/include/*" \
     --output-file "$OUTPUT_DIR/amdgpu_coverage_clean.info" \
     --rc lcov_branch_coverage=1 2>&1 | tail -10

echo ""

# 5. 生成 HTML 报告
echo "## 5. 生成 HTML 报告"
genhtml "$OUTPUT_DIR/amdgpu_coverage_clean.info" \
        --output-directory "$OUTPUT_DIR/html" \
        --branch-coverage \
        --show-details \
        --legend \
        --title "AMDGPU 驱动代码覆盖率报告" 2>&1 | tail -20

echo ""
echo "=== 覆盖率报告已生成 ==="
echo "HTML 报告: $OUTPUT_DIR/html/index.html"
echo ""

# 6. 输出摘要
echo "## 覆盖率摘要"
lcov --summary "$OUTPUT_DIR/amdgpu_coverage_clean.info" \
     --rc lcov_branch_coverage=1 2>&1
```

#### 2.3 覆盖率数据解读

```bash
#!/bin/bash
# coverage_analysis.sh - 覆盖率数据分析
# 参考：内核覆盖率分析最佳实践

set -e

COVERAGE_INFO="${1:-./coverage_report/amdgpu_coverage_clean.info}"

if [ ! -f "$COVERAGE_INFO" ]; then
    echo "错误: 覆盖率文件不存在: $COVERAGE_INFO"
    exit 1
fi

echo "=== 覆盖率数据分析 ==="
echo ""

# 1. 总体覆盖率
echo "## 1. 总体覆盖率"
lcov --summary "$COVERAGE_INFO" --rc lcov_branch_coverage=1 2>&1
echo ""

# 2. 按目录统计
echo "## 2. 按目录统计覆盖率"
echo ""
echo "| 目录 | 行覆盖 | 函数覆盖 | 分支覆盖 |"
echo "|------|--------|----------|----------|"

for dir in $(lcov --list "$COVERAGE_INFO" 2>/dev/null | grep "drivers/gpu/drm/amd" | cut -d'/' -f1-6 | sort -u); do
    # 提取该目录的覆盖率
    DIR_INFO=$(mktemp)
    lcov --extract "$COVERAGE_INFO" "${dir}/*" --output-file "$DIR_INFO" 2>/dev/null
    
    LINE=$(lcov --summary "$DIR_INFO" 2>/dev/null | grep "lines" | awk '{print $2}')
    FUNC=$(lcov --summary "$DIR_INFO" 2>/dev/null | grep "functions" | awk '{print $2}')
    BRANCH=$(lcov --summary "$DIR_INFO" 2>/dev/null | grep "branches" | awk '{print $2}')
    
    echo "| $(basename $dir) | ${LINE:-N/A} | ${FUNC:-N/A} | ${BRANCH:-N/A} |"
    
    rm -f "$DIR_INFO"
done
echo ""

# 3. 识别低覆盖率文件
echo "## 3. 低覆盖率文件（行覆盖 < 50%）"
echo ""

# 使用 lcov 列出文件并筛选
lcov --list "$COVERAGE_INFO" 2>/dev/null | while read file; do
    if [[ "$file" == *".c"* ]] || [[ "$file" == *".h"* ]]; then
        FILE_INFO=$(mktemp)
        lcov --extract "$COVERAGE_INFO" "$file" --output-file "$FILE_INFO" 2>/dev/null
        
        LINE_PCT=$(lcov --summary "$FILE_INFO" 2>/dev/null | grep "lines" | grep -oP '\d+\.\d+%' | cut -d'%' -f1)
        
        if [ -n "$LINE_PCT" ] && (( $(echo "$LINE_PCT < 50" | bc -l) )); then
            echo "- $file: ${LINE_PCT}%"
        fi
        
        rm -f "$FILE_INFO"
    fi
done

echo ""

# 4. 识别未覆盖函数
echo "## 4. 未覆盖函数（覆盖率 0%）"
echo ""
echo "以下函数未被任何测试调用："

# 使用 nm 和 gcov 数据对比（简化示例）
# 实际应使用更精确的方法
lcov --list "$COVERAGE_INFO" 2>/dev/null | grep -E "\.c$|\.h$" | head -20 | while read file; do
    echo "检查: $file"
done
```

### 三、案例分析

#### 3.1 AMDGPU 驱动覆盖率分析案例

**背景**：分析 amdgpu 内核驱动的测试覆盖率，识别测试盲区。

```
┌─────────────────────────────────────────────────────────────────────┐
│              AMDGPU 驱动覆盖率分析结果示例                           │
│                                                                     │
│  模块                  行覆盖    分支覆盖    函数覆盖    状态        │
│  ─────────────────────────────────────────────────────────────      │
│                                                                     │
│  amdgpu_device.c       78%       65%        85%        良好        │
│  amdgpu_cs.c           82%       71%        88%        良好        │
│  amdgpu_ttm.c          75%       62%        80%        良好        │
│  amdgpu_vm.c           68%       55%        75%        需改进      │
│  amdgpu_dpm.c          45%       38%        60%        不足        │
│  amdgpu_ras.c          40%       35%        55%        不足        │
│  amdgpu_acpi.c         35%       30%        50%        严重不足    │
│                                                                     │
│  测试盲区识别：                                                      │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  1. 错误处理路径：GPU hang 恢复、内存分配失败           │       │
│  │  2. 电源管理：不同 DPM 状态切换、温度阈值处理           │       │
│  │  3. 显示接口：HDMI/DP 链路训练失败路径                  │       │
│  │  4. 虚拟化：SR-IOV VF 管理、vGPU 调度                   │       │
│  │  5. 调试功能：debugfs 接口、性能计数器读取              │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
│  改进建议：                                                          │
│  ● 为 amdgpu_dpm.c 增加电源状态切换测试                             │
│  ● 为 amdgpu_ras.c 增加错误注入和恢复测试                            │
│  ● 为 amdgpu_acpi.c 增加系统挂起/恢复测试                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 3.2 CI 集成覆盖率门禁

```yaml
# .gitlab-ci.yml 覆盖率门禁配置
# 参考：Mesa 项目覆盖率实践

coverage_check:
  stage: verify
  image: kernel-build-env:latest
  script:
    # 运行测试并收集覆盖率
    - ./run_tests_with_coverage.sh
    
    # 检查覆盖率阈值
    - |
      LINE_PCT=$(lcov --summary coverage.info | grep "lines" | grep -oP '\d+\.\d+%' | cut -d'%' -f1)
      echo "行覆盖率: ${LINE_PCT}%"
      
      if (( $(echo "$LINE_PCT < 70" | bc -l) )); then
        echo "错误：行覆盖率 ${LINE_PCT}% 低于阈值 70%"
        exit 1
      fi
    
    - |
      BRANCH_PCT=$(lcov --summary coverage.info | grep "branches" | grep -oP '\d+\.\d+%' | cut -d'%' -f1)
      echo "分支覆盖率: ${BRANCH_PCT}%"
      
      if (( $(echo "$BRANCH_PCT < 60" | bc -l) )); then
        echo "错误：分支覆盖率 ${BRANCH_PCT}% 低于阈值 60%"
        exit 1
      fi
    
    # 检查新增代码覆盖率
    - ./check_new_code_coverage.sh
  artifacts:
    paths:
      - coverage_report/
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
```

### 四、相关链接

- 内核 gcov 文档：https://docs.kernel.org/dev-tools/gcov.html
- lcov 项目：https://github.com/linux-test-project/lcov
- gcov 工具文档：https://gcc.gnu.org/onlinedocs/gcc/Gcov.html
- 内核代码覆盖率最佳实践：https://docs.kernel.org/dev-tools/index.html
- Cobertura 覆盖率格式：http://cobertura.github.io/cobertura/

## 今日小结

- 代码覆盖率是评估测试充分性的重要指标，但高覆盖率不等于高质量测试
- 内核覆盖率收集需要启用 CONFIG_GCOV_KERNEL 并挂载 debugfs
- lcov + genhtml 是生成可视化覆盖率报告的标准工具链
- 分支覆盖率比语句覆盖率更能反映测试质量
- 覆盖率分析应识别低覆盖率模块和未覆盖的错误处理路径
- CI 中可设置覆盖率门禁，确保新增代码有足够测试覆盖

## 扩展思考

1. 覆盖率目标是否应该因模块而异？例如，核心调度代码是否应该要求 100% 分支覆盖，而调试代码可以放宽要求？
2. 如何处理硬件相关代码（如寄存器访问）的覆盖率收集？这些代码在模拟环境中可能无法执行。
