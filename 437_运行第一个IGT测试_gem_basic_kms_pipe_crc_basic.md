# 运行第一个 IGT 测试：gem_basic / kms_pipe_crc_basic

## 学习目标

- 掌握运行 IGT 测试的基本方法
- 理解 gem_basic 测试的功能和原理
- 理解 kms_pipe_crc_basic 测试的功能和原理
- 学会解读 IGT 测试的输出结果
- 能够排查测试运行中的常见问题

## 知识详解

### 一、概念原理

#### 1.1 gem_basic 测试

```
┌─────────────────────────────────────────────────────────────────────┐
│              gem_basic 测试                                          │
│                                                                     │
│  功能：验证 GEM（Graphics Execution Manager）缓冲区对象的基本操作    │
│                                                                     │
│  测试内容：                                                          │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  子测试名称          测试内容                            │       │
│  │  ─────────────────────────────────────────────────────   │       │
│  │  create-close        创建和关闭 BO                       │       │
│  │  create-large        创建大 BO（256MB）                  │       │
│  │  create-invalid-size 无效大小参数测试                    │       │
│  │  create-zero-size    零大小参数测试                      │       │
│  │  mmap                BO 内存映射测试                     │       │
│  │  mmap-bad-flag       无效 mmap 标志测试                  │       │
│  │  pwrite              向 BO 写入数据                      │       │
│  │  pread               从 BO 读取数据                      │       │
│  │  set-domain          设置内存域                          │       │
│  │  get-tiling          获取 tiling 模式                    │       │
│  │  set-tiling          设置 tiling 模式                    │       │
│  │  swapping            BO 交换测试                         │       │
│  │  close-race          并发关闭竞争测试                    │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
│  涉及 ioctl：                                                        │
│  ● DRM_IOCTL_I915_GEM_CREATE / DRM_IOCTL_AMDGPU_GEM_CREATE         │
│  ● DRM_IOCTL_I915_GEM_MMAP / DRM_IOCTL_AMDGPU_GEM_MMAP             │
│  ● DRM_IOCTL_I915_GEM_PWRITE / DRM_IOCTL_AMDGPU_GEM_OP             │
│  ● DRM_IOCTL_I915_GEM_PREAD                                        │
│  ● DRM_IOCTL_I915_GEM_SET_DOMAIN                                   │
│  ● DRM_IOCTL_I915_GEM_SET_TILING                                   │
│  ● DRM_IOCTL_GEM_CLOSE                                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.2 kms_pipe_crc_basic 测试

```
┌─────────────────────────────────────────────────────────────────────┐
│              kms_pipe_crc_basic 测试                                 │
│                                                                     │
│  功能：验证 KMS（Kernel Mode Setting）显示管线的 CRC 校验功能        │
│                                                                     │
│  测试原理：                                                          │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  1. 设置显示模式（分辨率、刷新率）                       │       │
│  │  2. 绘制已知图案到帧缓冲区                               │       │
│  │  3. 读取显示管线的 CRC32 校验值                          │       │
│  │  4. 验证 CRC 值与预期值匹配                              │       │
│  │  5. 重复测试不同模式和图案                               │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
│  CRC 校验值用途：                                                    │
│  ● 验证显示输出正确性（无像素损坏）                                  │
│  ● 检测页面翻转是否成功                                              │
│  ● 验证不同模式设置的一致性                                          │
│                                                                     │
│  涉及 ioctl：                                                        │
│  ● DRM_IOCTL_MODE_GETRESOURCES                                     │
│  ● DRM_IOCTL_MODE_GETCRTC / DRM_IOCTL_MODE_SETCRTC                 │
│  ● DRM_IOCTL_MODE_GETCONNECTOR                                     │
│  ● DRM_IOCTL_MODE_CREATE_DUMB / DRM_IOCTL_MODE_DESTROY_DUMB        │
│  ● DRM_IOCTL_MODE_PAGE_FLIP                                        │
│  ● DRM_IOCTL_AMDGPU_INFO (for CRC)                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 二、实践操作

#### 2.1 运行 gem_basic 测试

```bash
#!/bin/bash
# run_gem_basic.sh - 运行 gem_basic 测试

set -e

IGT_BUILD="${1:-$HOME/igt-gpu-tools/build}"

echo "=== 运行 gem_basic 测试 ==="
echo ""

# 检查测试程序是否存在
if [ ! -f "$IGT_BUILD/tests/gem_basic" ]; then
    echo "错误: 未找到 gem_basic 测试程序"
    exit 1
fi

# 运行所有子测试
echo "【1】运行所有子测试"
sudo "$IGT_BUILD/tests/gem_basic" 2>&1 | tee gem_basic_full.log

echo ""

# 运行指定子测试
echo "【2】运行指定子测试 (create-close)"
sudo "$IGT_BUILD/tests/gem_basic" --run-subtest "create-close" 2>&1 | tee gem_basic_create.log

echo ""

# 列出所有子测试
echo "【3】列出所有子测试"
"$IGT_BUILD/tests/gem_basic" --list-subtests 2>&1

echo ""

# 解析结果
echo "【4】结果解析"
if grep -q "PASS" gem_basic_full.log; then
    PASS_COUNT=$(grep -c "PASS" gem_basic_full.log || true)
    echo "通过测试数: $PASS_COUNT"
fi

if grep -q "FAIL" gem_basic_full.log; then
    FAIL_COUNT=$(grep -c "FAIL" gem_basic_full.log || true)
    echo "失败测试数: $FAIL_COUNT"
fi

echo ""
echo "=== gem_basic 测试完成 ==="
```

#### 2.2 运行 kms_pipe_crc_basic 测试

```bash
#!/bin/bash
# run_kms_crc.sh - 运行 kms_pipe_crc_basic 测试

set -e

IGT_BUILD="${1:-$HOME/igt-gpu-tools/build}"

echo "=== 运行 kms_pipe_crc_basic 测试 ==="
echo ""

# 检查测试程序
if [ ! -f "$IGT_BUILD/tests/kms_pipe_crc_basic" ]; then
    echo "错误: 未找到 kms_pipe_crc_basic 测试程序"
    exit 1
fi

# 运行测试
echo "【1】运行测试"
sudo "$IGT_BUILD/tests/kms_pipe_crc_basic" 2>&1 | tee kms_crc.log

echo ""

# 运行指定子测试
echo "【2】运行指定子测试 (read-crc-pipe-A)"
sudo "$IGT_BUILD/tests/kms_pipe_crc_basic" --run-subtest "read-crc-pipe-A" 2>&1 | tee kms_crc_pipeA.log

echo ""

# 解析结果
echo "【3】结果解析"
grep -E "IGT-Version|Subtest|PASS|FAIL|SKIP" kms_crc.log || true

echo ""
echo "=== kms_pipe_crc_basic 测试完成 ==="
```

#### 2.3 输出结果解读

```
┌─────────────────────────────────────────────────────────────────────┐
│              IGT 测试输出解读                                        │
│                                                                     │
│  示例输出：                                                          │
│  ──────────                                                          │
│  IGT-Version: 1.28-gf7a3b2c (x86_64) (Linux: 6.8.0-amd64)          │
│                                                                     │
│  Starting subtest: create-close                                     │
│  Subtest create-close: SUCCESS (0.002s)                             │
│                                                                     │
│  Starting subtest: create-large                                     │
│  Subtest create-large: SUCCESS (0.015s)                             │
│                                                                     │
│  Starting subtest: create-invalid-size                              │
│  Subtest create-invalid-size: SUCCESS (0.001s)                      │
│                                                                     │
│  Test gem_basic: SUCCESS                                            │
│                                                                     │
│  结果状态：                                                          │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │  状态      含义              处理方式                     │       │
│  │  ─────────────────────────────────────────────────────   │       │
│  │  SUCCESS   测试通过          正常                         │       │
│  │  FAIL      测试失败          需要调查                     │       │
│  │  SKIP      条件不满足跳过    检查硬件/驱动支持            │       │
│  │  TIMEOUT   测试超时          可能 hang，需收集日志        │       │
│  │  CRASH     测试崩溃          严重问题，需立即调查         │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 三、常见问题排查

```
┌─────────────────────────────────────────────────────────────────────┐
│              常见问题排查                                            │
│                                                                     │
│  问题                          原因              解决方案           │
│  ─────────────────────────────────────────────────────────────      │
│                                                                     │
│  Permission denied             无 DRM 设备权限   sudo 或加入 video │
│                                                                     │
│  No DRM device found           驱动未加载        lsmod 检查驱动    │
│                                                                     │
│  Subtest skipped               硬件不支持        检查 GPU 型号     │
│                                                                     │
│  Timeout                       测试 hang         收集 dmesg        │
│                                                                     │
│  CRC mismatch                  显示输出错误      检查显示器连接    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 四、相关链接

- IGT 测试文档：https://drm.pages.freedesktop.org/igt-gpu-tools/igt-gpu-tools-Test.html
- GEM 文档：https://dri.freedesktop.org/docs/drm/gpu/drm-mm.html
- KMS 文档：https://dri.freedesktop.org/docs/drm/gpu/drm-kms.html

## 今日小结

- gem_basic 测试验证 GEM 缓冲区对象的基本操作（创建、映射、读写、关闭）
- kms_pipe_crc_basic 测试验证 KMS 显示管线的 CRC 校验功能
- IGT 测试输出包含版本信息、子测试名称、结果状态和耗时
- 测试结果分为 SUCCESS、FAIL、SKIP、TIMEOUT、CRASH 五种状态
- 运行 IGT 测试需要 DRM 设备访问权限

## 扩展思考

1. CRC 校验在显示测试中的局限性是什么？什么类型的显示错误可能无法被 CRC 检测？
2. 如何设计一个自动化脚本，定期运行 gem_basic 和 kms_pipe_crc_basic 并报告回归？
