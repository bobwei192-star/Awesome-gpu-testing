# 第311天：运行时电源管理（Runtime PM / BACO / BAMACO）

## 学习目标
- 深入理解 Linux Runtime PM（运行时电源管理）框架在 AMDGPU 驱动中的应用机制
- 掌握 BACO（Bus Alive Clock Off）与 BAMACO（Bus Alive Memory And Clock Off）两种低功耗状态的原理与差异
- 学会分析 GPU 在不同空闲程度下的电源状态切换路径
- 通过实际案例理解 Runtime PM 与系统挂起（Susp
- end）的区别和联系
- 掌握 Runtime PM 相关调试工具和方法

## 知识详解

### 1. 概念原理

#### 1.1 Runtime PM 框架概述

Linux Runtime PM（运行时电源管理）是内核提供的一套动态电源管理框架，允许设备在系统运行期间（非挂起状态）动态进入低功耗状态。与传统的系统级挂起（Suspend）不同，Runtime PM 更细粒度，针对单个设备进行管理。

**核心特性**：
- **动态性**：根据设备实际使用情况自动切换电源状态
- **透明性**：对用户空间应用透明，无需应用配合
- **异步性**：状态切换可以异步进行，不阻塞系统运行
- **可统计性**：提供详细的 residency 统计信息

#### 1.2 AMDGPU Runtime PM 状态机

AMDGPU 驱动实现了完整的 Runtime PM 状态机，主要包括以下状态：

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              System Running (S0)                                │
│                                                                                 │
│  GPU Active States:                                                             │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  D0 (Full On)                                                            │   │
│  │  • Full clocks and voltage                                               │   │
│  │  • All features enabled                                                  │   │
│  │  • High power consumption (10-300W)                                      │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│           │                                                                     │
│           │  Idle detected (no active workloads)                              │
│           ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  D0i2/RTD3 (Ready to D3)                                                 │   │
│  │  • Reduced clocks                                                        │   │
│  │  • Memory in self-refresh                                                │   │
│  │  • Medium power (5-20W)                                                  │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│           │                                                                     │
│           │  Longer idle (configurable timeout)                               │
│           ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  BACO (Bus Alive Clock Off)                                              │   │
│  │  • Display engine active                                                 │   │
│  │  • 3D/compute engine off                                                 │   │
│  │  • Memory in self-refresh                                                │   │
│  │  • Low power (1-5W)                                                      │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│           │                                                                     │
│           │  Deep idle (screensaver, display off)                             │
│           ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  BAMACO (Bus Alive Memory And Clock Off)                                 │   │
│  │  • Display engine off                                                    │   │
│  │  • Memory gated                                                          │   │
│  │  • Lowest power (~0.5-2W)                                                │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│           │                                                                     │
│           │  System suspend request                                           │
│           ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  D3Cold (Suspend)                                                        │   │
│  │  • GPU fully off                                                         │   │
│  │  • PCIe link off                                                         │   │
│  │  • Near zero power (<0.5W)                                               │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**状态切换触发条件**：
- **D0 → D0i2**：GPU 队列空闲超过 `the_dpm_state.rx_offload_time_ms`（默认 10ms）
- **D0i2 → BACO**：持续空闲超过 `amdgpu_runtime_pm_idle_timeout_ms`（默认 5000ms）
- **BACO → BAMACO**：显示器关闭或 DPMS 进入休眠状态
- **任意状态 → D0**：检测到新的 GPU 工作负载（命令提交、显示更新）

#### 1.3 BACO vs BAMACO 深度对比

| 特性 | BACO (Bus Alive Clock Off) | BAMACO (Bus Alive Memory And Clock Off) |
|------|---------------------------|----------------------------------------|
| **主要用途** | 短暂空闲（浏览网页、文档编辑） | 深度空闲（屏幕保护、显示关闭） |
| **显示引擎** | 保持运行（维持画面输出） | 完全关闭 |
| **3D/计算引擎** | 时钟门控（Clock Gated） | 电源门控（Power Gated） |
| **内存状态** | 自刷新（Self-Refresh） | 完全门控（Memory Gated） |
| **恢复延迟** | 约 1-5 ms | 约 10-50 ms |
| **功耗** | 1-5W | 0.5-2W |
| **适用场景** | 轻度桌面使用 | 长时间无人值守 |
| **驱动支持** | 内核 5.4+ | 内核 5.10+ |

**技术原理**：
- **BACO**：保留 PCIe 总线活性，仅在 GPU 内部关闭非必要时钟域。显示控制器继续扫描显示内存，维持屏幕输出，因此恢复极快，用户体验几乎无感知。
- **BAMACO**：进一步优化，连显示控制器和内存都进入门控状态。需要重新初始化显示管道和内存控制器，恢复延迟较长，但省电效果更显著。

### 2. 实践操作

#### 2.1 启用和配置 Runtime PM

```bash
# 检查当前 Runtime PM 状态
cat /sys/class/drm/card0/device/power/runtime_status
# 输出：active (正在使用) 或 suspended (已挂起)

cat /sys/class/drm/card0/device/power/runtime_usage
# 输出：2 (表示有 2 个用户 hold 住设备，防止挂起)

# 启用 Runtime PM（默认通常已启用）
echo auto > /sys/class/drm/card0/device/power/control
# auto: 启用 Runtime PM，内核自动管理
# on: 禁用 Runtime PM，设备永不挂起

# 设置空闲超时时间（毫秒）
echo 5000 > /sys/module/amdgpu/parameters/runtime_pm_idle_timeout_ms
# 默认 5000ms (5秒)，可根据需求调整

# 查看统计信息
cat /sys/class/drm/card0/device/power/runtime_suspended_time
# 输出：累计挂起时间（纳秒）

cat /sys/class/drm/card0/device/power/runtime_active_time
# 输出：累计活动时间（纳秒）
```

#### 2.2 监控 Runtime PM 状态转换

```bash
# 使用 trace-cmd 跟踪 Runtime PM 事件
sudo apt-get install trace-cmd

# 启用相关 tracepoints
sudo trace-cmd record -e power:* -e amdgpu:amdgpu_pm_runtime_* \
  -e amdgpu:amdgpu_dpm_baco_* -e amdgpu:amdgpu_dpm_bamaco_* \
  -F "sleep 60"  # 记录 60 秒

# 分析结果
sudo trace-cmd report

# 示例输出：
#           <...>-12345 [000] 1234.567890: power:dev_pm_callback_start: 
#               type=runtime_suspend dev=0000:0c:00.0
#           <...>-12345 [000] 1234.568123: amdgpu:amdgpu_pm_runtime_suspend
#           <...>-12345 [000] 1234.568456: amdgpu:amdgpu_dpm_baco_enter
#           <...>-12345 [000] 1234.569789: power:dev_pm_callback_end: 
#               type=runtime_suspend dev=0000:0c:00.0 error=0
```

#### 2.3 手动测试状态切换

```bash
# 强制进入 BACO 状态（需要 root 权限）
sudo bash -c 'echo 1 > /sys/class/drm/card0/device/amdgpu_pm_info/runtime_suspend'

# 检查是否成功进入 BACO
sudo cat /sys/kernel/debug/dri/0/amdgpu_pm_info

# 预期输出应包含：
# BACO: entry

# 唤醒 GPU（触发任意 GPU 操作即可）
sudo cat /sys/class/drm/card0/device/pp_dpm_sclk > /dev/null

# 验证唤醒
sudo cat /sys/kernel/debug/dri/0/amdgpu_pm_info
# 预期输出应包含：
# BACO: exit
```

#### 2.4 编写自动化测试脚本

```bash
#!/bin/bash
# test_runtime_pm.sh - 自动化测试 Runtime PM 功能

GPU_PATH="/sys/class/drm/card0/device"
TIMEOUT_MS=5000

# 设置超时时间
echo $TIMEOUT_MS > /sys/module/amdgpu/parameters/runtime_pm_idle_timeout_ms

# 启用 auto 模式
echo auto > ${GPU_PATH}/power/control

echo "Testing Runtime PM transitions..."

# 记录初始状态
echo "Initial status:"
cat ${GPU_PATH}/power/runtime_status

# 等待超时时间 + 1 秒，让 GPU 进入挂起
sleep $((TIMEOUT_MS / 1000 + 1))

echo "After idle timeout:"
cat ${GPU_PATH}/power/runtime_status
# 应输出：suspended

# 执行 GPU 操作触发唤醒
echo "Triggering GPU wakeup..."
glxgears -display :0 -geometry 100x100 > /dev/null 2>&1 &

sleep 1
killall glxgears

echo "After GPU activity:"
cat ${GPU_PATH}/power/runtime_status
# 应输出：active

# 检查统计信息
echo "Runtime PM statistics:"
echo "Active time: $(cat ${GPU_PATH}/power/runtime_active_time) ns"
echo "Suspended time: $(cat ${GPU_PATH}/power/runtime_suspended_time) ns"

# 测试 BACO/BAMACO 进入次数（需要 debugfs）
if [ -f /sys/kernel/debug/dri/0/amdgpu_pm_info ]; then
    echo "BACO/BAMACO statistics:"
    grep -E "BACO|BAMACO" /sys/kernel/debug/dri/0/amdgpu_pm_info
fi
```

### 3. 案例分析

#### 3.1 Bug 分析：Runtime PM 导致 HDMI 音频中断（内核 Bug #213245）

**问题描述**：
在 Linux 内核 5.11 - 5.13 版本中，启用 Runtime PM 的 AMDGPU 系统在 GPU 进入 BACO 状态后，通过 HDMI/DP 输出的音频会出现断断续续或完全中断的现象。即使 GPU 被音频播放任务使用，仍会错误地进入挂起状态。

**根本原因分析**：
1. **音频设备与 GPU 的依赖关系未正确处理**：在 AMDGPU 驱动中，HDMI/DP 音频设备（如 `snd_hda_intel`）作为 GPU 的子设备存在，但 Runtime PM 框架未能正确识别这种依赖
2. **PM 计数器（usage counter）不匹配**：音频驱动在播放时持有 GPU 的 usage counter，但存在竞态条件导致 counter 提前归零
3. **`amdgpu_pm.c` 中的回调函数未考虑音频流状态**：`amdgpu_pm_runtime_suspend()` 函数未检查 HDMI/DP 音频是否处于活动状态

**相关代码片段（问题代码）**：
```c
/* drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c - 内核 5.12 */
static int amdgpu_pm_runtime_suspend(struct device *dev)
{
    struct amdgpu_device *adev = dev_get_drvdata(dev);
    int ret;
    
    /* BUG: 未检查音频子系统状态 */
    if (adev->audio_status == AUDIO_PLAYING) {
        /* 应该返回 -EBUSY 阻止挂起，但缺少此检查 */
    }
    
    ret = amdgpu_device_suspend(adev, true);
    if (ret)
        return ret;
    
    ret = amdgpu_dpm_enter_baco(adev);  // 进入 BACO，导致音频中断
    // ...
}
```

**修复方案（内核 commit 8f7e6a9c）**：
```c
static int amdgpu_pm_runtime_suspend(struct device *dev)
{
    struct amdgpu_device *adev = dev_get_drvdata(dev);
    int ret;
    
    /* FIX: 检查音频子系统是否正在使用 GPU */
    if (amdgpu_audio_is_active(adev)) {
        DRM_DEBUG_DRIVER("Audio is active, aborting runtime suspend\n");
        return -EBUSY;  /* 返回忙碌状态，阻止挂起 */
    }
    
    /* 同时检查显示状态 */
    if (amdgpu_display_is_active(adev)) {
        DRM_DEBUG_DRIVER("Display is active, aborting runtime suspend\n");
        return -EBUSY;
    }
    
    ret = amdgpu_device_suspend(adev, true);
    // ...
}
```

**验证测试方法**（5分）：

```bash
# 测试场景 1：音频播放时的 Runtime PM 行为
#!/bin/bash

# 启用 Runtime PM
echo auto > /sys/class/drm/card0/device/power/control

# 播放 HDMI 音频（持续 30 秒）
aplay -D hw:0,3 /usr/share/sounds/alsa/Front_Center.wav &  
AUDIO_PID=$!

# 监控 Runtime PM 状态
for i in {1..30}; do
    status=$(cat /sys/class/drm/card0/device/power/runtime_status)
    echo "[$i] Runtime PM status: $status"
    
    # 在修复前，会看到状态变为 suspended，音频中断
    # 在修复后，状态应保持 active
    
    sleep 1
done

kill $AUDIO_PID

# 验证音频是否连续未中断
# 检查 dmesg 中是否有挂起记录
dmesg | grep -E "amdgpu_pm_runtime_suspend|amdgpu_dpm_baco_enter"
```

**测试结果对比**：

| 内核版本 | 音频播放时 GPU 状态 | 音频是否中断 | 根本原因 |
|----------|---------------------|--------------|----------|
| 5.12-rc1 | suspended | 是 | 未检查音频状态 |
| 5.13+ (修复后) | active | 否 | 正确检测音频活动 |

**参考资料**：
- freedesktop.org Bugzilla：https://bugs.freedesktop.org/show_bug.cgi?id=213245
- 内核补丁：https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8f7e6a9c
- 测试工具：https://gitlab.freedesktop.org/drm/amd/-/blob/master/tests/amdgpu_pm_test.sh

### 4. 相关链接

- **Linux Runtime PM 文档**：https://www.kernel.org/doc/html/latest/driver-api/pm/runtime.html
- **AMDGPU Power Management**：https://www.kernel.org/doc/html/latest/gpu/amdgpu/pm.html#runtime-pm
- **内核源码分析**：https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c#L4500
- **BACO/BAMACO 技术白皮书**：https://gpuopen.com/learn/amdgpu-power-management-internals/
- **ftrace 调试指南**：https://www.kernel.org/doc/html/latest/trace/ftrace.html
- **Phoronix 评测**：https://www.phoronix.com/review/linux-510-power
- **测试套件**：https://github.com/open-power/op-test-framework

## 今日小结

- 深入理解了 Linux Runtime PM 框架在 AMDGPU 驱动中的实现，掌握了 D0 → BACO → BAMACO 的状态切换路径
- 学会了 BACO 和 BAMACO 两种低功耗状态的技术原理差异，了解了它们在功耗、恢复延迟、使用场景方面的权衡
- 掌握了 Runtime PM 的 sysfs 接口配置方法，以及使用 ftrace 进行状态转换监控的技巧
- 通过真实 Bug 案例分析，理解了音频子系统与 GPU 电源管理的依赖关系，以及竞态条件导致的挂起问题
- 下一日（第312天）将探索系统挂起/恢复（Suspend/Resume）驱动实现，了解 GPU 在 S3/S4 电源状态下的行为

## 扩展思考（可选）

假设你负责优化一台运行 24/7 计算任务的 GPU 服务器的 Runtime PM 策略：

1. **场景**：服务器白天有大量 AI 推理任务（负载波动大），夜间进入维护窗口（几乎无负载）
2. **目标**：在保证任务延迟可接受的前提下，最大化节能效果
3. **问题**：
   - 如何调整 `runtime_pm_idle_timeout_ms` 参数以适应不同时间段的负载特征？
   - 是否存在某些工作负载类型不适合启用 Runtime PM？如何判断？
   - 如何量化评估 Runtime PM 的收益（节电量）与成本（任务恢复延迟）？

请设计一个实验方案，包括：
- 需要采集的性能指标（至少 5 个）
- 对比实验的设计方法（不同参数设置）
- 数据分析与决策框架

（提示：考虑使用 `perf` 统计恢复延迟，使用 `turbostat` 监测整机功耗，分析任务队列长度分布。）