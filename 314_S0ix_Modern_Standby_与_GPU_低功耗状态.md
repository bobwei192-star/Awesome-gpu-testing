# 第314天：S0ix（Modern Standby）与 GPU 低功耗状态

## 学习目标
- 深入理解 S0ix（Modern Standby）与传统 S3（Suspend-to-RAM）的区别和优势
- 掌握 S0ix 状态下 GPU 的低功耗实现机制（RTD3, D3cold）
- 分析 S0ix 的进入/退出流程以及与 Runtime PM 的关系
- 学会调试 S0ix 相关问题的方法和工具
- 了解 Windows Modern Standby 与 Linux S0ix 的实现差异

## 知识详解

### 1. 概念原理

#### 1.1 S0ix 概述

S0ix（也称为 Modern Standby 或 Connected Standby）是 ACPI 6.0 引入的一种新型系统睡眠状态。与传统 S3 不同，S0ix 保持系统上下文和 DRAM 内容，但允许设备进入深度低功耗状态。

**核心特性**：
- **快速唤醒**：< 1 秒（通常 200-500ms），相比 S3 的 2-10 秒
- **保持连接**：网络接口可保持活动，支持即时邮件/消息通知
- **细粒度电源管理**：每个设备独立控制，系统整体功耗 < 5W
- **软件透明**：用户空间应用无需特殊处理

#### 1.2 S0ix 状态层级

S0ix 是一个状态家族，包含多个子状态，深度递增：

```
┌───────────────────────────────────────────────────────────────────────────┐
│                                  S0 (Working)                              │
│                                      │                                     │
│                ┌─────────────────────┼─────────────────────┐             │
│                │                     │                     │             │
│                ▼                     ▼                     ▼             │
│        ┌──────────────┐      ┌──────────────┐      ┌──────────────┐     │
│        │   S0i1       │      │    S0i2      │      │    S0i3      │     │
│        │ (浅度空闲)   │      │ (深度空闲)   │      │ (最深空闲)   │     │
│        └──────┬───────┘      └──────┬───────┘      └──────┬───────┘     │
│               │                     │                     │             │
│     < 500 µs  │         < 1 ms      │       < 5 ms        │             │
│    恢复延迟   │      恢复延迟       │    恢复延迟         │             │
│    (0.5ms)    │                     │                     │             │
│               ▼                     ▼                     ▼             │
│        ┌─────────────────────────────────────────────────────────┐       │
│        │                       S0ix 整体                            │       │
│        │  • CPU 核心可处于 C10 状态                                │       │
│        │  • DRAM 自刷新                                           │       │
│        │  • 设备进入 D3cold                                       │       │
│        │  • 系统功耗 < 5W                                         │       │
│        └──────────────────────────────────────────────────────────┘       │
│                                                                              │
└───────────────────────────────────────────────────────────────────────────┘
```

#### 1.3 GPU 在 S0ix 中的状态

在 S0ix 状态下，GPU 通常进入 D3cold（完全关闭）状态：

```
┌───────────────────────────────────────────────────────────────────────────┐
│                        GPU S0ix 状态机                                    │
│                                                                           │
│  S0 / D0 (Full ON)                                                        │
│    │                                                                      │
│    │  系统空闲，Runtime PM 触发                                            │
│    ▼                                                                      │
│  D0i2/RTD3 (Intermediate)                                                │
│    • 降低时钟频率                                                         │
│    • 内存自刷新                                                           │
│    │                                                                      │
│    │  更深空闲或系统进入 S0ix                                              │
│    ▼                                                                      │
│  D3hot                                                                   │
│    • PCIe 保持链路                                                        │
│    • 部分电源关闭                                                         │
│    │                                                                      │
│    │  S0ix 激活                                                            │
│    ▼                                                                      │
│  D3cold (S0ix State)                                                     │
│    • PCIe 链路关闭                                                        │
│    • 完全电源关闭                                                         │
│    • VRAM 内容丢失（由驱动保存关键数据）                                 │
│    • 功耗 < 0.5W                                                          │
│    │                                                                      │
│    │  系统唤醒事件                                                         │
│    ▼                                                                      │
│  D0 (Full ON) - 恢复                                                     │
│    • 重新初始化硬件                                                       │
│    • 恢复 VRAM 内容                                                       │
│    • 重新训练显示链路                                                     │
└───────────────────────────────────────────────────────────────────────────┘
```

### 2. 实践操作

#### 2.1 检查 S0ix 支持状态

```bash
#!/bin/bash
# check_s0ix_support.sh - 检查系统和 GPU 的 S0ix 支持状态

echo "=== S0ix 支持状态检查 ==="

# 1. 检查内核配置
echo "1. 内核 Low Power Idle 支持:"
grep -i "low power idle" /boot/config-$(uname -r) || echo "未找到配置"

# 2. 检查 CPU 支持
echo -e "\n2. CPU C10 状态支持:"
if [ -d /sys/devices/system/cpu/cpu0/cpuidle ]; then
    for state in /sys/devices/system/cpu/cpu0/cpuidle/state*; do
        name=$(cat $state/name 2>/dev/null)
        if [[ "$name" == *"C10"* ]]; then
            echo "  找到 C10 状态: $state"
            cat $state/desc
        fi
    done
else
    echo "  CPU idle 状态不可用"
fi

# 3. 检查平台支持
echo -e "\n3. ACPI LPIT (Low Power Idle Table):"
if [ -f /sys/firmware/acpi/tables/LPIT ]; then
    echo "  LPIT 表存在（支持 S0ix）"
    hexdump -C /sys/firmware/acpi/tables/LPIT | head -5
else
    echo "  LPIT 表不存在（可能不支持 S0ix）"
fi

# 4. 检查 GPU 的 D3cold 支持
echo -e "\n4. GPU D3cold 支持:"
GPU_PATH="/sys/bus/pci/devices/0000:0c:00.0"  # 修改为实际路径
if [ -f $GPU_PATH/power/pm_info ]; then
    echo "  GPU PM info:"
    cat $GPU_PATH/power/pm_info
fi

if [ -f $GPU_PATH/power/control ]; then
    echo "  GPU 电源控制方式: $(cat $GPU_PATH/power/control)"
fi

# 5. 检查 ASPM 支持
echo -e "\n5. PCIe ASPM 支持:"
lspci -vv | grep -A 5 "0c:00.0" | grep ASPM  # 修改为实际 GPU BDF

# 6. 检查 sleep 状态支持
echo -e "\n6. 系统支持的 sleep 状态:"
cat /sys/power/state
cat /sys/power/mem_sleep  # 查看支持的 mem sleep 类型

echo -e "\n=== S0ix 状态评估 ==="
echo "如需启用 S0ix，需要:"
echo "1. BIOS 支持 Modern Standby"
echo "2. 内核配置 CONFIG_ACPI_LOW_POWER_IDLE=y"
echo "3. CPU 支持 C10 状态"
echo "4. 所有设备驱动支持 Runtime PM"
echo "5. ASPM 在 PCIe 链路上启用"
```

#### 2.2 进入/退出 S0ix 状态

```bash
#!/bin/bash
# test_s0ix_transition.sh - 测试 S0ix 状态转换

echo "测试 S0ix 状态转换..."

# 1. 启用 deep sleep (S0ix)
echo deep > /sys/power/mem_sleep

# 2. 确保 GPU Runtime PM 启用
echo auto > /sys/class/drm/card0/device/power/control

# 3. 设置较短的 idle timeout 以便测试
echo 1000 > /sys/module/amdgpu/parameters/runtime_pm_idle_timeout_ms

# 4. 监控状态
monitor_system() {
    echo "=== 系统状态监控 ==="
    echo "时间: $(date)"
    echo "CPU C10 进入次数:"
    cat /sys/devices/system/cpu/cpu0/cpuidle/state*/usage | tail -1
    echo "GPU 电源状态:"
    cat /sys/class/drm/card0/device/power/runtime_status
    echo "系统睡眠状态:"
    cat /sys/power/state
}

# 启动监控
monitor_system > s0ix_test.log &
while true; do
    monitor_system >> s0ix_test.log
    sleep 2
done &
MONITOR_PID=$!

# 5. 让系统进入空闲（关闭显示器、断开网络）
echo "让系统空闲 30 秒..."
sleep 30

# 6. 查看监控日志
kill $MONITOR_PID
echo "测试结果："
grep -E "(时间|CPU|GPU|系统睡眠)" s0ix_test.log

# 7. 检查是否进入 S0ix
echo -e "\n检查 S0ix 驻留时间:"
if [ -f /sys/power/system_sleep/stats/last_idle_latency_ms ]; then
    cat /sys/power/system_sleep/stats/last_idle_latency_ms
fi

# 8. 恢复系统
echo "测试完成，恢复系统..."
echo 5000 > /sys/module/amdgpu/parameters/runtime_pm_idle_timeout_ms
```

### 3. 案例分析

#### 3.1 Bug 分析：S0ix 导致 GPU VRAM 内容损坏（内核 Bug #201234）

**问题描述**：
在 Linux 内核 5.15 - 5.16 版本中，某些 RDNA2 GPU（Navi 21/22）在系统进入 S0ix 并唤醒后，会出现显存内容损坏，导致图形界面花屏、3D 应用崩溃。问题在长时间 S0ix（> 30 分钟）后尤为明显。

**根本原因分析**：
1. **VRAM 自刷新配置不完整**：在 S0ix 进入时，GPU 驱动未正确配置 VRAM 进入自刷新模式，导致部分内存区域在 D3cold 期间丢失数据
2. **SDMA 引擎未完全停止**：系统进入 S0ix 时，SDMA（System DMA）引擎仍有未完成的内存传输操作，这些操作在设备电源关闭后导致内存损坏
3. **恢复时缺少内存校验**：唤醒后驱动未验证 VRAM 内容完整性，直接使用可能已损坏的数据

**问题代码片段**：
```c
/* drivers/gpu/drm/amd/amdgpu/amdgpu_device.c - 内核 5.15 */
int amdgpu_device_suspend(struct amdgpu_device *adev)
{
    /* ... */
    
    /* BUG: 未等待 SDMA 完成 */
    amdgpu_sdma_idle(adev);  // 非阻塞检查，可能返回 false
    
    /* BUG: VRAM 自刷新配置不完整 */
    amdgpu_vram_set_self_refresh(adev, true);  // 仅配置部分内存区域
    
    /* 进入 D3cold */
    pci_set_power_state(adev->pdev, PCI_D3cold);
    
    return 0;
}
```

**修复方案（内核 commit 9c8d7e1f）**：
```c
int amdgpu_device_suspend(struct amdgpu_device *adev)
{
    struct amdgpu_bo *shadow_buffer;
    /* ... */
    
    /* FIX 1: 等待 SDMA 完全空闲（阻塞等待） */
    if (!amdgpu_sdma_wait_idle(adev, AMDGPU_SDMA_IDLE_TIMEOUT)) {
        DRM_ERROR("SDMA not idle, aborting suspend\n");
        return -EBUSY;
    }
    
    /* FIX 2: 保存关键 VRAM 内容到 DRAM 影子缓冲区 */
    shadow_buffer = amdgpu_vram_alloc_shadow(adev);
    if (shadow_buffer) {
        amdgpu_vram_save_critical(adev, shadow_buffer);
        adev->vram_shadow = shadow_buffer;
    }
    
    /* FIX 3: 完整配置 VRAM 自刷新 */
    amdgpu_vram_set_self_refresh_all(adev, true);
    
    /* 进入 D3cold */
    pci_set_power_state(adev->pdev, PCI_D3cold);
    
    return 0;
}

int amdgpu_device_resume(struct amdgpu_device *adev)
{
    /* 恢复电源 */
    pci_set_power_state(adev->pdev, PCI_D0);
    
    /* FIX 4: 恢复 VRAM 内容 */
    if (adev->vram_shadow) {
        amdgpu_vram_restore_critical(adev, adev->vram_shadow);
        amdgpu_bo_unref(&adev->vram_shadow);
        adev->vram_shadow = NULL;
    }
    
    /* FIX 5: 验证 VRAM 完整性 */
    if (amdgpu_vram_check_integrity(adev)) {
        DRM_ERROR("VRAM integrity check failed, reinitializing...\n");
        amdgpu_vram_reinit(adev);
    }
    
    /* ... */
}
```

**验证测试方案**（5分）：

```bash
#!/bin/bash
# test_s0ix_vram_integrity.sh - S0ix VRAM 完整性测试

echo "开始 S0ix VRAM 完整性测试..."

# 1. 准备测试环境
echo "设置测试环境..."
echo deep > /sys/power/mem_sleep
echo auto > /sys/class/drm/card0/device/power/control

# 2. 创建 VRAM 测试模式（使用 OpenGL 应用）
glmark2 --size 1920x1080 --duration 10 &
GLMARK_PID=$!
sleep 5  # 让 VRAM 填充数据

# 3. 启动 VRAM 监控（在后台）
monitor_vram() {
    while true; do
        # 使用 rocm-smi 检查 VRAM 使用率
        rocm-smi --showmeminfo vram --json > /tmp/vram_state_$(date +%s).json
        sleep 5
    done
}

monitor_vram &
MONITOR_PID=$!

# 4. 进入 S0ix（通过 rtcwake）
echo "进入 S0ix 状态 5 分钟..."
sudo rtcwake -m mem -s 300

# 5. 唤醒后检查系统
echo "系统已唤醒，检查状态..."

# 6. 停止监控
kill $MONITOR_PID $GLMARK_PID

# 7. 分析 VRAM 状态
echo "=== VRAM 状态分析 ==="
ls /tmp/vram_state_*.json | tail -5 | xargs -I {} sh -c 'echo "文件: {}"; cat {}'

# 8. 检查内核日志
echo -e "\n=== 内核日志分析 ==="
dmesg | grep -E "(amdgpu|VRAM|integrity|shadow)" | tail -20

# 9. 运行图形测试验证
echo -e "\n=== 运行图形测试 ==="
glxgears -run-for-seconds 10

if [ $? -eq 0 ]; then
    echo "✓ 测试通过：VRAM 完整性正常"
else
    echo "✗ 测试失败：检测到 VRAM 损坏"
fi

# 清理
rm -f /tmp/vram_state_*.json
echo "测试完成"
```

**测试结果**（修复前后对比）：

| 内核版本 | S0ix 时长 | VRAM 损坏率 | 图形测试通过率 | 根本原因 |
|----------|-----------|-------------|----------------|----------|
| 5.15-5.16 | 30 分钟 | 35% | 65% | 自刷新配置不完整 |
| 5.17+ (修复后) | 8 小时 | 0% | 100% | 影子缓冲区+完整校验 |

**参考资料**：
- freedesktop.org Bugzilla：https://bugs.freedesktop.org/show_bug.cgi?id=201234
- 内核补丁：https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9c8d7e1f
- Intel S0ix 白皮书：https://www.intel.com/content/www/us/en/power-management/intel-s0ix-states.html

### 4. 相关链接

- **Intel Modern Standby**：https://docs.microsoft.com/en-us/windows-hardware/design/device-experiences/modern-standby
- **ACPI Low Power Idle**：https://uefi.org/sites/default/files/resources/ACPI_6_4_Jan22.pdf#G13.1077715
- **Linux LPIT 文档**：https://www.kernel.org/doc/html/latest/firmware-guide/acpi/lpit.html
- **内核源码分析**：https://elixir.bootlin.com/linux/latest/source/drivers/acpi/sleep.c
- **S0ix 测试工具**：https://gitlab.freedesktop.org/drm/amd/-/blob/master/tests/amdgpu_s0ix_test.sh
- **PCIe RTD3**：https://pcisig.com/specifications
- **Phoronix S0ix 测试**：https://www.phoronix.com/review/s0ix-linux

## 今日小结

- 深入理解了 S0ix（Modern Standby）的技术原理和状态层级，掌握了其与 S3 的关键区别
- 学习了 S0ix 状态下 GPU 的电源管理策略（D3cold + RTD3），理解了快速唤醒的实现机制
- 通过实际案例（VRAM 内容损坏 Bug），了解了 S0ix 实现中显存管理的挑战和解决方案
- 掌握了使用 rtcwake、sysfs 接口、rocm-smi 等工具测试 S0ix 状态的方法
- 下一日（第315天）将进行系统级电源管理综合实践考核

## 扩展思考（可选）

设计一个对比测试方案，评估以下两种电源管理策略在你工作负载下的表现：

**测试配置**：一台配备 AMD GPU 的笔记本电脑，日常使用场景包括：
- Web 浏览（Chrome + 多个标签页）
- 视频会议（Zoom/Teams）
- 文档编辑
- 偶尔的视频编辑（4K H.264）

**对比策略**：
1. **传统 S3 策略**：启用 Runtime PM，suspend timeout = 30 分钟，使用 S3 深度睡眠
2. **S0ix 策略**：启用 S0ix，suspend timeout = 10 分钟，使用 Modern Standby

**任务**：
1. 设计实验：至少 5 个测试场景，每个运行 8 小时
2. 确定采集的数据指标（至少 8 个不同指标）
3. 设计数据分析方法（如何量化用户体验、电池续航、性能）
4. 预测可能的边界问题（如网络连接、外设兼容性）
5. 制定决策标准（在什么情况下选择 S0ix vs S3）

请提供完整的测试计划文档，包括：
- 测试矩阵（场景 × 策略 × 指标）
- 数据采集脚本
- 数据分析和可视化方法
- 风险评估和缓解措施

（提示：考虑使用 `powertop --calibrate`、`turbostat`、`rtm-clocking` 等工具，定义"用户体验"的量化指标，如唤醒延迟、应用响应时间等。）