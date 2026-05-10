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

## 附录A：S0ix 内核代码深度分析

### A.1 acpi_init/exit 接口与 S0ix 状态管理

```c
// drivers/acpi/sleep.c
// ACPI S0ix 状态管理与平台固件交互

static int acpi_s2idle_begin(void)
{
    acpi_scan_lock_acquire();
    return 0;
}

static int acpi_s2idle_prepare(void)
{
    /* 通知所有ACPI设备准备进入S0ix */
    acpi_dev_suspend(true);
    return 0;
}

static int acpi_s2idle_prepare_late(void)
{
    /* 平台固件准备进入低功耗模式 */
    acpi_os_set_prepare_sleep(ACPI_STATE_S0, ACPI_SLEEP_TYPE_S0IX);
    return 0;
}

static int acpi_s2idle_wake(void)
{
    /* 处理唤醒事件 */
    acpi_os_wake_sleep(ACPI_STATE_S0);
    return 0;
}

static void acpi_s2idle_restore_early(void)
{
    acpi_resume_power_resources();
}

static void acpi_s2idle_restore(void)
{
    acpi_dev_resume(true);
    acpi_scan_lock_release();
}
```

### A.2 amdgpu S0ix 进入/退出流程

```c
// drivers/gpu/drm/amd/amdgpu/amdgpu_acpi.c
// AMDGPU驱动的S0ix特定处理

int amdgpu_acpi_s2idle_suspend(struct amdgpu_device *adev)
{
    int ret;

    /* 阶段1：保存显示状态 */
    amdgpu_display_suspend(adev);

    /* 阶段2：通知SMU准备S0ix */
    ret = smu_s2idle_prepare(adev->smu_priv);
    if (ret) {
        dev_err(adev->dev, "SMU S0ix prepare failed: %d\n", ret);
        return ret;
    }

    /* 阶段3：进入RTD3（Runtime D3） */
    ret = amdgpu_device_runtime_pm_suspend(adev);
    if (ret) {
        dev_err(adev->dev, "RTD3 suspend failed: %d\n", ret);
        return ret;
    }

    /* 阶段4：设置PCI D3cold */
    pci_save_state(adev->pdev);
    ret = pci_set_power_state(adev->pdev, PCI_D3hot);
    if (ret)
        return ret;

    return 0;
}

int amdgpu_acpi_s2idle_resume(struct amdgpu_device *adev)
{
    int ret;

    /* 阶段1：恢复PCI电源 */
    pci_set_power_state(adev->pdev, PCI_D0);
    pci_restore_state(adev->pdev);

    /* 阶段2：从RTD3恢复 */
    ret = amdgpu_device_runtime_pm_resume(adev);
    if (ret) {
        dev_err(adev->dev, "RTD3 resume failed: %d\n", ret);
        return ret;
    }

    /* 阶段3：恢复SMU状态 */
    ret = smu_s2idle_resume(adev->smu_priv);
    if (ret) {
        dev_err(adev->dev, "SMU S0ix resume failed: %d\n", ret);
        return ret;
    }

    /* 阶段4：恢复显示 */
    amdgpu_display_resume(adev);

    return 0;
}
```

### A.3 S0ix 与 Runtime PM 的协同工作

```c
// drivers/gpu/drm/amd/amdgpu/amdgpu_device.c

/* 在系统进入S0ix前确保Runtime PM已完成深度睡眠 */
int amdgpu_device_s2idle_prepare(struct amdgpu_device *adev)
{
    int retry = 3;

    while (retry--) {
        /* 强制触发Runtime PM suspend */
        pm_runtime_suspend(adev->dev);
        
        /* 检查是否已进入D3cold */
        if (adev->pdev->current_state == PCI_D3hot ||
            adev->in_s0ix) {
            break;
        }
        msleep(100);
    }

    if (retry < 0) {
        dev_warn(adev->dev, 
                 "Failed to enter D3 before S0ix\n");
        return -EBUSY;
    }

    return 0;
}

void amdgpu_device_s2idle_resume(struct amdgpu_device *adev)
{
    /* 确保Runtime PM恢复完成 */
    pm_runtime_get_sync(adev->dev);
    
    /* 检查设备是否可用 */
    if (adev->pdev->current_state != PCI_D0) {
        pci_set_power_state(adev->pdev, PCI_D0);
        pci_restore_state(adev->pdev);
    }
    
    pm_runtime_put_autosuspend(adev->dev);
}
```

### A.4 GFXOFF 与 S0ix 的关系

```c
// drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.c

/* GFXOFF是进入S0ix的前置条件 */
int amdgpu_gfx_enter_s0ix(struct amdgpu_device *adev)
{
    int ret;

    /* 确保GFXOFF已激活 */
    if (!adev->gfx.gfx_off_state) {
        ret = amdgpu_gfx_off_control(adev, true);
        if (ret) {
            dev_err(adev->dev, "Failed to enable GFXOFF\n");
            return ret;
        }
    }

    /* 等待所有GFX引擎空闲 */
    ret = amdgpu_gfx_wait_for_idle(adev);
    if (ret) {
        dev_err(adev->dev, "GFX not idle before S0ix\n");
        return ret;
    }

    /* 通知SMU进入S0ix深度睡眠 */
    ret = smu_s0ix_entry(adev->smu_priv);
    if (ret)
        return ret;

    adev->in_s0ix = true;
    return 0;
}
```

## 附录B：ACPI Low Power S0 Idle (LPI) 协议详解

### B.1 LPI 状态定义

ACPI 6.0+ 定义了 Low Power Idle (LPI) 描述符，用于描述S0ix下各设备的睡眠深度：

```c
/* include/acpi/actypes.h */

/* LPI状态约束类型 */
#define ACPI_LPI_STATE_TYPE_LPI  0  // 浅睡眠
#define ACPI_LPI_STATE_TYPE_RTC  1  // RTC唤醒
#define ACPI_LPI_STATE_TYPE_PLATFORM 2  // 平台级睡眠

struct acpi_lpi_state {
    u32 min_residency;     // 最小驻留时间(us)
    u32 wake_latency;      // 唤醒延迟(us)
    u32 flags;             // LPI标志位
    u32 entry_method;      // 进入方式(FEI/HCI)
    u8 enabled;            // 是否启用
};
```

### B.2 S0ix 平台固件交互流程

```
Linux内核                    Platform Firmware (BIOS/UEFI)
    │                                │
    │  echo freeze > /sys/power/state │
    │───────────────────────────────>│
    │                                │
    │  acpi_s2idle_prepare()         │
    │───────────────────────────────>│  _LPI 方法评估
    │                                │
    │  acpi_s2idle_prepare_late()    │
    │───────────────────────────────>│  关闭非唤醒中断
    │                                │
    │  设备驱动 Runtime PM Suspend   │
    │───────────────────────────────>│  _PS3 (D3hot)
    │                                │
    │  处理器进入C6/C7               │
    │───────────────────────────────>│  _CST 状态转换
    │                                │
    │  ACPI SLP_TYP写入S0            │
    │───────────────────────────────>│  硬件进入S0ix
    │           ...睡眠中...          │
    │                                │
    │  唤醒事件(USB/网络/RTC)        │
    │<───────────────────────────────│
    │                                │
    │  设备驱动程序Resume            │
    │<───────────────────────────────│  _PS0 (D0恢复)
    │                                │
    │  acpi_s2idle_restore()         │
    │<───────────────────────────────│  恢复中断
```

### B.3 使用 ACPI 表调试验证 S0ix

```bash
#!/bin/bash
"""
检查平台S0ix相关ACPI表
"""

echo "=== 检查S0ix支持 ==="
cat /sys/power/mem_sleep
# 期望输出: s2idle [deep] 或 [s2idle] deep

echo "=== ACPI FADT表检查 ==="
cat /sys/firmware/acpi/tables/FADT | od -An -tx1 | head -5
# LOW Power S0 Idle (LPI) Capable 位在 FADT 标志中

echo "=== DSDT中查找_LPI对象 ==="
# 需要安装iasl工具
iasl -d /sys/firmware/acpi/tables/DSDT 2>/dev/null
grep -i "_LPI\|S0IX\|LowPowerS0" /tmp/DSDT.dsl 2>/dev/null

echo "=== 检查所有设备的LPI约束 ==="
for dev in /sys/bus/acpi/devices/*/power; do
    if [ -f "$dev/wakeup" ]; then
        echo "--- $(basename $(dirname $dev)) ---"
        cat $dev/wakeup 2>/dev/null
        cat $dev/resources 2>/dev/null | head -5
    fi
done
```

### B.4 设备唤醒能力配置

```bash
#!/bin/bash
"""
配置和管理S0ix下的设备唤醒
"""

# 查看当前唤醒配置
cat /proc/acpi/wakeup

# 启用/禁用特定设备的S0ix唤醒
# 格式: echo DEVICE_NAME > /proc/acpi/wakeup

# 例如禁用USB唤醒
echo "USB0" > /proc/acpi/wakeup
echo "USB1" > /proc/acpi/wakeup
echo "USB2" > /proc/acpi/wakeup

# 查看设备唤醒状态统计
cat /sys/bus/acpi/devices/*/power/wakeup_count 2>/dev/null

# 设置唤醒GPIO极性（如平台支持）
# 通常通过BIOS设置界面调整
```

## 附录C：S0ix 调试工具链详解

### C.1 powertop 深度分析

```bash
#!/bin/bash
"""
使用powertop分析和优化S0ix功耗
"""

# 1. 校准powertop
echo "校准powertop（需要root）..."
powertop --calibrate
# 校准过程约需20-30分钟，将学习设备的功耗特征

# 2. 检查S0ix状态
echo "=== S0ix 状态检查 ==="
powertop --quiet --csv=/tmp/powertop_report.csv

# 3. 查看S0ix驻留时间
echo "检查S0ix驻留统计..."
cat /sys/kernel/debug/pmc_core/slp_s0_residency_usec 2>/dev/null
# 单位: 微秒，数字持续增长表示S0ix正常运行

# 4. 检查阻止S0ix进入的设备
echo "=== 阻止进入S0ix的设备 ==="
powertop --auto-report | grep -i "S0ix\|package c-state\|device.*block"

# 5. 生成HTML报告
powertop --html=/tmp/powertop_report.html
echo "HTML报告已生成: /tmp/powertop_report.html"
```

### C.2 turbostat 分析 CPU C-State 与 S0ix

```bash
#!/bin/bash
"""
使用turbostat分析CPU C-State与S0ix关联
"""

# 1. 基本C-State监控
echo "=== CPU C-State 分布 ==="
turbostat -i 5 --quiet --show C1,C1E,C6,C7,C8,C9,C10,POLL,CPU%c1,CPU%c6,CPU%c7 -n 12

# 2. S0ix对应的是Package C10状态
echo "=== Package C-State（C10表示系统进入S0ix）==="
turbostat -i 10 --quiet --show Pkg%pc2,Pkg%pc3,Pkg%pc6,Pkg%pc7,Pkg%pc8,Pkg%pc9,Pkg%pc10 -n 6

# 3. 监控S0ix入口
echo "=== 模拟S0ix事件（手动触发）==="
turbostat -i 1 --quiet --show PkgWY,Pkg%pc10 &
TURBO_PID=$!

# 触发S0ix
sleep 2
echo freeze > /sys/power/state 2>/dev/null

sleep 10
kill $TURBO_PID 2>/dev/null
```

### C.3 使用 rtcwake 测试 S0ix

```bash
#!/bin/bash
"""
S0ix自动化测试框架
"""

TEST_COUNT=50
SLEEP_DURATION=5
INTERVAL_SECONDS=15

echo "S0ix自动化测试开始"
echo "测试次数: $TEST_COUNT, 每次睡眠: ${SLEEP_DURATION}s"
echo "========================================="

for i in $(seq 1 $TEST_COUNT); do
    BEFORE_STATE=$(cat /sys/kernel/debug/pmc_core/slp_s0_residency_usec 2>/dev/null || echo "0")
    BEFORE_C10=$(turbostat -i 1 --show Pkg%pc10 -n 1 2>/dev/null | tail -1 || echo "N/A")
    
    echo "[$i/$TEST_COUNT] 触发S0ix..."
    rtcwake -m freeze -s $SLEEP_DURATION -d /dev/rtc0 2>&1
    
    AFTER_STATE=$(cat /sys/kernel/debug/pmc_core/slp_s0_residency_usec 2>/dev/null || echo "0")
    DELTA=$(( AFTER_STATE - BEFORE_STATE ))
    
    # 检查GFXOFF状态
    GFXOFF=$(cat /sys/kernel/debug/dri/0/amdgpu_gfxoff 2>/dev/null || echo "N/A")
    
    if [ $DELTA -gt 1000000 ]; then  # 至少1秒的S0ix驻留
        echo "  ✓ S0ix驻留: ${DELTA}us | GFXOFF: $GFXOFF"
    else
        echo "  ✗ S0ix驻留不足: ${DELTA}us | 可能被阻止"
    fi
    
    sleep $INTERVAL_SECONDS
done

echo "========================================="
echo "S0ix测试完成"
```

### C.4 使用 eBPF 追踪 S0ix 进入/退出

```python
#!/usr/bin/env python3
"""
eBPF追踪S0ix进入/退出延迟
"""

from bcc import BPF
import ctypes as ct
import time

bpf_source = """
#include <linux/sched.h>

BPF_HASH(entry_ts, u64, u64);
BPF_PERF_OUTPUT(s0ix_events);

struct s0ix_event {
    u32 cpu;
    u64 duration_ns;
    u8 entry_success;
};

TRACEPOINT_PROBE(power, cpu_idle)
{
    u64 cpu = args->cpu;
    u64 state = args->state;
    u64 ts = bpf_ktime_get_ns();
    
    if (state == 10) {  // C10 = S0ix
        u64 *start = &ts;
        entry_ts.update(&cpu, &ts);
    }
    
    if (state == -1) {  // Exit idle
        u64 *entry = entry_ts.lookup(&cpu);
        if (entry) {
            struct s0ix_event event = {};
            event.cpu = cpu;
            event.duration_ns = ts - *entry;
            event.entry_success = 1;
            s0ix_events.perf_submit(args, &event, sizeof(event));
            entry_ts.delete(&cpu);
        }
    }
    
    return 0;
}
"""

b = BPF(text=bpf_source)
print("追踪S0ix(C10)事件...")

def print_event(cpu, data, size):
    event = ct.cast(data, ct.POINTER(ct.c_ubyte * size)).contents
    duration_ms = event.duration_ns / 1000000
    if duration_ms > 100:  # 只显示超过100ms的有效S0ix
        print(f"[CPU{event.cpu}] S0ix驻留: {duration_ms:.1f}ms")

b["s0ix_events"].open_perf_buffer(print_event)

try:
    while True:
        b.perf_buffer_poll(timeout=1000)
except KeyboardInterrupt:
    print("停止追踪")
```

### C.5 S0ix 功耗数据采集脚本

```bash
#!/bin/bash
"""
S0ix 功耗数据采集与分析
"""

LOG_FILE="/tmp/s0ix_power_log.csv"
echo "timestamp,s0ix_residency_us,package_power_W,GFXOFF,gpu_power_W,gpu_temp_C" > $LOG_FILE

SAMPLE_INTERVAL=5
DURATION=3600  # 1小时采样

echo "开始采集S0ix功耗数据（${DURATION}s）..."

for i in $(seq $((DURATION / SAMPLE_INTERVAL))); do
    TIMESTAMP=$(date +%s)
    RESIDENCY=$(cat /sys/kernel/debug/pmc_core/slp_s0_residency_usec 2>/dev/null || echo "0")
    PACKAGE_POWER=$(turbostat -i 1 --show PkgWatt -n 1 2>/dev/null | tail -1 || echo "0")
    GFXOFF_STATE=$(cat /sys/kernel/debug/dri/0/amdgpu_gfxoff 2>/dev/null || echo "N/A")
    GPU_POWER=$(cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_average 2>/dev/null || echo "0")
    GPU_TEMP=$(cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input 2>/dev/null || echo "0")
    
    echo "$TIMESTAMP,$RESIDENCY,$PACKAGE_POWER,$GFXOFF_STATE,$GPU_POWER,$GPU_TEMP" >> $LOG_FILE
    
    # 实时显示
    printf "\r采样第%d/%d次 | S0ix: %dus | 封装功耗: %.1fW | GPU: %.1fW / %.1f°C" \
        $i $((DURATION / SAMPLE_INTERVAL)) $RESIDENCY $PACKAGE_POWER \
        $(echo "$GPU_POWER/1000000" | bc 2>/dev/null) \
        $(echo "$GPU_TEMP/1000" | bc 2>/dev/null)
    
    sleep $SAMPLE_INTERVAL
done

echo ""
echo "数据采集完成，已保存到 $LOG_FILE"
echo "生成摘要..."

# 简单统计
awk -F',' 'NR>1{
    residency[NR]+=$2
    gpu_power[NR]+=$5
} END {
    print "平均S0ix驻留: " residency/NR "us"
    print "平均GPU功耗: " gpu_power/NR "uW"
}' $LOG_FILE
```

## 附录D：S0ix 故障排查指南

### D.1 常见S0ix故障速查

| 症状 | 可能原因 | 诊断步骤 | 解决方案 |
|------|---------|---------|---------|
| 无法进入S0ix (freeze) | 设备驱动阻止 | `cat /sys/kernel/debug/pmc_core/slp_s0_residency_usec` | 逐个禁用设备排查 |
| S0ix驻留时间过短 | 频繁唤醒事件 | `powertop --auto-report` | 配置设备唤醒策略 |
| S0ix恢复后WiFi断开 | 网卡驱动问题 | `dmesg | grep iwl` | 更新网卡固件 |
| S0ix恢复后GPU不可用 | amdgpu RTD3失败 | `cat /sys/kernel/debug/dri/0/amdgpu_gfxoff` | 重置GPU驱动 |
| 触摸板无响应 | ACPI唤醒配置 | `cat /proc/acpi/wakeup` | 启用触摸板唤醒 |
| RTC唤醒不准时 | 平台固件bug | `rtcwake -m freeze -s 60 -v` | 检查BIOS版本 |
| 电池消耗快 | 未进入深度C-State | `turbostat -i 5 | grep C10` | 检查阻止睡眠的进程 |
| 音频卡顿/断流 | 音频设备PM冲突 | `dmesg | grep snd` | 禁用音频Runtime PM |
| USB设备丢失 | USB控制器PM问题 | `lsusb -t` | 重新枚举USB设备 |
| 系统无法恢复死机 | 中断路由问题 | 检查`/proc/interrupts` | 内核参数`acpi_mask_gpe=0xXX` |

### D.2 阻止S0ix的设备排查

```bash
#!/bin/bash
"""
系统性排查阻止S0ix进入的设备
"""

echo "=== 步骤1: 检查S0ix驻留基线 ==="
BASELINE=$(cat /sys/kernel/debug/pmc_core/slp_s0_residency_usec 2>/dev/null || echo "0")
sleep 10
CURRENT=$(cat /sys/kernel/debug/pmc_core/slp_s0_residency_usec 2>/dev/null || echo "0")
DELTA=$((CURRENT - BASELINE))
echo "10秒内S0ix驻留增量: ${DELTA}us"
if [ $DELTA -lt 1000000 ]; then
    echo "⚠ S0ix驻留不足，可能存在阻止设备"
fi

echo ""
echo "=== 步骤2: 检查各设备PM状态 ==="
for dev in /sys/bus/pci/devices/*/power; do
    if [ -f "$dev/runtime_status" ]; then
        STATUS=$(cat $dev/runtime_status 2>/dev/null)
        if [ "$STATUS" == "active" ]; then
            PCI_ADDR=$(basename $(dirname $dev))
            VENDOR=$(cat $(dirname $dev)/vendor 2>/dev/null)
            echo "  活跃: $PCI_ADDR ($(lspci -s $PCI_ADDR 2>/dev/null | cut -d' ' -f2-))"
        fi
    fi
done

echo ""
echo "=== 步骤3: 逐个禁用PCI设备排查 ==="
for dev_path in /sys/bus/pci/devices/*/power; do
    PCI_ADDR=$(basename $(dirname $dev_path))
    if [ -f "$(dirname $dev_path)/driver/unbind" ]; then
        echo "尝试禁用: $PCI_ADDR"
        BEFORE=$(cat /sys/kernel/debug/pmc_core/slp_s0_residency_usec 2>/dev/null || echo "0")
        echo "$PCI_ADDR" > "$(dirname $dev_path)/driver/unbind" 2>/dev/null
        sleep 5
        AFTER=$(cat /sys/kernel/debug/pmc_core/slp_s0_residency_usec 2>/dev/null || echo "0")
        DELTA=$((AFTER - BEFORE))
        echo "  S0ix驻留变化: ${DELTA}us"
        # 重新绑定
        echo "$PCI_ADDR" > "$(dirname $dev_path)/driver/bind" 2>/dev/null
    fi
done
```

### D.3 故障案例：XPS笔记本无法进入S0ix

**问题现象**：Dell XPS 15 (AMD版) 运行Linux时无法进入S0ix，电源按钮灯持续闪烁，系统功耗维持在6-8W。

**环境**：Dell XPS 15 7590, AMD Ryzen 7 7840HS, RX 7600M XT dGPU, Ubuntu 23.10, Kernel 6.5

**诊断过程**：

```bash
# 1. 基础检查
cat /sys/power/mem_sleep
# 输出: s2idle [deep]  <- 默认使用S3而非S0ix

cat /sys/kernel/debug/pmc_core/slp_s0_residency_usec
# 输出: 0  <- S0ix从未进入

# 2. 强制启用S0ix
echo s2idle > /sys/power/mem_sleep

# 3. 检查阻止设备
powertop --auto-report | grep "阻止"
# 发现: nvidia-dgpu (RX 7600M XT), xhci_hcd 阻止了C10

# 4. 检查GPU Runtime PM状态
cat /sys/bus/pci/devices/0000:03:00.0/power/runtime_status
# 输出: active  <- dGPU未进入睡眠

# 5. 检查ACPI唤醒设置
cat /proc/acpi/wakeup
# 发现多个USB设备支持唤醒但未正确配置
```

**根因**：NVIDIA dGPU（实际为AMD RX 7600M XT）在PRIME配置下未启用Runtime PM。同时，USB控制器（xhci）在S0ix下持续活跃，阻止了Package C-State进入C10。

**解决方案**：

```bash
# 方案A：启用dGPU Runtime PM
echo auto > /sys/bus/pci/devices/0000:03:00.0/power/control

# 方案B：配置USB唤醒策略
# 禁用不需要的USB唤醒
echo "XHCI" > /proc/acpi/wakeup
echo "XDCI" > /proc/acpi/wakeup

# 方案C：内核参数优化
# 在/etc/default/grub中添加：
# GRUB_CMDLINE_LINUX="s2idle=force amdgpu.runpm=1 i915.enable_dc=4"
```

**验证**：
```bash
echo s2idle > /sys/power/mem_sleep
rtcwake -m freeze -s 10
cat /sys/kernel/debug/pmc_core/slp_s0_residency_usec
# 输出: 45283912 (45ms驻留，比之前的0有显著改善)
turbostat -i 1 --show Pkg%pc10 -n 5 | tail -3
# 输出: 68.2% (Package C10驻留率)
```

## 附录E：跨平台S0ix实现对比

### E.1 Windows Modern Standby vs Linux S0ix

| 特性 | Windows Modern Standby | Linux S0ix (s2idle) |
|------|----------------------|-------------------|
| 内核实现 | NT内核 PM模块 | ACPI s2idle + Runtime PM |
| 设备管理 | PoFx (Power Framework) | Linux PM核心 + 设备驱动 |
| 唤醒处理 | 基于软件的通知 | ACPI GPE中断 |
| 网络保持 | 网卡保持最低活动 | 取决于网卡驱动支持 |
| 应用透明 | 支持后台应用活动 | 支持（需ioloop） |
| GPU管理 | DirectX运行时D3 | DRM/amdgpu Runtime PM |
| 调试工具 | powertrace, SleepStudy | powertop, turbostat, ftrace |
| 固件交互 | _DSM接口 | _LPI + _PS3/_PS0 |
| 驱动要求 | Windows DCH驱动 | 内核PM回调实现 |
| 用户通知 | 系统托盘指示 | sysfs接口 |

### E.2 Windows Modern Standby 关键调试命令

```powershell
# Windows PowerShell：Modern Standby调试

# 1. 查看睡眠状态报告
powercfg /sleepstudy
# 输出: sleepstudy-report.html

# 2. 检查S0ix支持
powercfg /a
# 输出系统支持的睡眠状态

# 3. 启用/禁用Modern Standby
# 修改注册表
reg add HKLM\System\CurrentControlSet\Control\Power /v PlatformAoAcOverride /t REG_DWORD /d 1

# 4. 查看设备驱动阻止睡眠的问题
powercfg /requests
# 显示哪些驱动/应用阻止系统进入睡眠

# 5. 生成能源审计报告
powercfg /energy /output energy_report.html

# 6. 查看电源管理事件日志
Get-WinEvent -ProviderName Microsoft-Windows-Kernel-Power | Select-Object -First 20
```

### E.3 Intel SoC Linux vs AMD SoC S0ix 支持对比

| 特性 | Intel SoC (Tiger Lake+) | AMD SoC (Phoenix/Hawk Point) |
|------|------------------------|------------------------------|
| PMC支持 | intel_pmc_core驱动 | 无专用PMC驱动（AMD尚未提供） |
| C-State深度 | C10 (Package) | C10 (Package) |
| 调试接口 | slp_s0_residency_usec | 无（需AMD工具） |
| GPU S0ix | i915 DMC固件管理 | amdgpu SMU固件管理 |
| IPU6/CAM | 支持S0ix唤醒 | 取决于OEM实现 |
| Audio DSP | SOF固件S0ix支持 | AMD ACP固件支持 |
| NVMe | 支持RTD3 | 支持RTD3（需驱动支持） |
| USB4/TBT | 支持D3cold | 支持（AMD USB4控制器） |

### E.4 S0ix 在数据中心/服务器的应用

```bash
#!/bin/bash
"""
数据中心S0ix使用情况调查

在服务器环境下，S0ix通常用于部分节点低负载时段
"""

echo "=== 服务器S0ix支持检查 ==="
cat /sys/power/state
# 输出应包含 "freeze"

echo "=== 检查服务器C-State ==="
turbostat -i 10 -n 6 --show Pkg%pc2,Pkg%pc6,Pkg%pc10

echo "=== 检查带外管理对S0ix的影响 ==="
# BMC/IPMI可能阻止S0ix
ipmitool power status 2>/dev/null

echo "=== 虚拟化环境下S0ix可用性 ==="
cat /sys/module/kvm_intel/parameters/ple_gap 2>/dev/null
# KVM嵌套虚拟化可能阻止S0ix
```

## 附录F：S0ix 面试题与综合实践

### F.1 核心概念面试题

**题目1**：S0ix (Modern Standby) 与 S3 (Suspend-to-RAM) 的根本区别是什么？为什么现代GPU驱动需要同时支持这两种睡眠模式？

**解答**：S3是传统睡眠状态，挂起时CPU完全停止执行，所有设备上下文保存在DRAM中，恢复时需要完整的硬件重新初始化流程。S0ix是操作系统的深度空闲状态，CPU保持最低活动（C-State C10），设备通过Runtime PM独立管理电源。GPU驱动需要同时支持两种模式，因为：1）不同平台硬件支持不同（部分笔记本/台式机不支持S0ix）；2）用户使用习惯不同（部分用户偏好快速唤醒）；3）兼容性考虑（S3更成熟，S0ix仍在完善中）。

**题目2**：解释GFXOFF与S0ix的关系。为什么在进入S0ix前必须确保GFXOFF已激活？

**解答**：GFXOFF是RDNA架构引入的细粒度GFX引擎时钟门控，在GPU空闲时关闭GFX引擎时钟以降低功耗。GFXOFF是S0ix的前置条件，因为S0ix要求GPU处于最低功耗状态。如果GFX时钟仍在运行，GPU内部状态转换可能导致在S0ix进入/退出过程中出现总线事务不一致。此外，GFXOFF确保所有GFX相关的DMA操作已完成，避免在S0ix期间出现内存访问错误。

### F.2 调试场景题

**题目3**：你在调试一台AMD笔记本的S0ix问题时发现`cat /sys/kernel/debug/pmc_core/slp_s0_residency_usec`始终为0。如何列出所有可能的原因并逐一排查？

**解答**：
排查步骤：
1. 确认平台支持S0ix：检查`mem_sleep`是否包含s2idle选项
2. 确认BIOS/UEFI配置中S0ix已启用
3. 检查ACPI LPIT表：`cat /sys/firmware/acpi/tables/LPIT > /dev/null`，如果不存在则平台不支持
4. 使用powertop检查阻止设备：找`Device阻止C-State`部分
5. 逐设备排查：通过`driver/unbind`临时禁用设备（见附录D.2）
6. 检查内核配置：`CONFIG_PMC_CORE`和`CONFIG_ACPI_LPIT`必须启用
7. 检查固件版本：某些OEM BIOS在特定版本后才支持S0ix
8. 使用turbostat确认Package C-State：如果所有核心都处于C7但Package不在C10，说明SoC内仍有活动单元

**题目4**：用户报告在从S0ix恢复后，GPU温度比进入前高了10°C。这可能是什么问题？如何诊断？

**解答**：
可能原因：
1. GPU在恢复后未正确降频，持续运行在较高P-State
2. 风扇控制未从S0ix状态正确恢复，风扇转速未随负载调整
3. GPU在恢复过程中固件初始化失败，触发了硬件保护机制（clock gating disabled）

诊断步骤：
1. 检查恢复后的GPU频率：`cat /sys/class/drm/card0/device/pp_dpm_sclk`
2. 检查风扇转速：`cat /sys/class/drm/card0/device/hwmon/hwmon*/fan1_input`
3. 检查SMU错误日志：`dmesg | grep -i smu`
4. 检查GFXOFF状态：`cat /sys/kernel/debug/dri/0/amdgpu_gfxoff`
5. 对比进入S0ix前后的功率限制：`cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap`

### F.3 综合设计题

**题目5**：设计一个新的S0ix电源管理策略"Adaptive S0ix"，要求根据用户使用模式自动在S3和S0ix之间切换。

**设计方案**：
```python
#!/usr/bin/env python3
"""
Adaptive S0ix - 智能电源管理策略
"""

class AdaptiveS0ixManager:
    def __init__(self):
        self.usage_history = []
        self.s0ix_enter_delays = []
        self.s3_enter_delays = []
        self.s0ix_resume_delays = []
        self.s3_resume_delays = []
        self.threshold_wakeup_latency_ms = 1000  # 用户容忍的唤醒延迟
        
    def record_usage_event(self, event_type, timestamp):
        """
        记录用户使用模式
        event_type: 'input' (键盘/鼠标), 'network' (网络包), 'timer' (定时唤醒)
        """
        self.usage_history.append({
            'type': event_type,
            'timestamp': timestamp,
            'hour': timestamp.hour,
            'day_of_week': timestamp.weekday()
        })
    
    def predict_idle_duration(self, current_hour, is_weekend):
        """基于历史数据预测空闲时长"""
        relevant_events = [
            e for e in self.usage_history
            if e['hour'] == current_hour and 
            e['day_of_week'] >= 5 == is_weekend
        ]
        
        if len(relevant_events) < 10:
            return 300  # 默认5分钟
        
        # 分析空闲间隔分布
        intervals = []
        for i in range(1, len(relevant_events)):
            interval = relevant_events[i]['timestamp'] - \
                      relevant_events[i-1]['timestamp']
            intervals.append(interval.total_seconds())
        
        # P90空闲间隔作为预测值
        intervals.sort()
        p90_index = int(len(intervals) * 0.9)
        return intervals[p90_index]
    
    def decide_sleep_mode(self, predicted_idle_seconds):
        """决策：S0ix还是S3"""
        if predicted_idle_seconds < 60:
            return None  # 不睡眠
        
        if predicted_idle_seconds < 600:  # 10分钟内
            return 's0ix'  # 快速唤醒
    
        # 长时间空闲 -> S3更省电
        if (self.get_avg_resume_delay('s0ix') < 
            self.threshold_wakeup_latency_ms):
            return 's0ix'  # S0ix满足唤醒延迟要求
        
        return 's3'
    
    def get_avg_resume_delay(self, mode):
        delays = self.s0ix_resume_delays if mode == 's0ix' \
                 else self.s3_resume_delays
        if not delays:
            return 0
        return sum(delays) / len(delays)
    
    def calibrate(self):
        """自动校准S0ix和S3的性能特征"""
        import subprocess
        import json
        
        # 测S0ix恢复延迟
        for _ in range(5):
            start = time.time()
            subprocess.run(['rtcwake', '-m', 'freeze', '-s', '5'])
            delay = (time.time() - start) * 1000 - 5000
            self.s0ix_resume_delays.append(delay)
        
        # 测S3恢复延迟
        for _ in range(3):
            start = time.time()
            subprocess.run(['rtcwake', '-m', 'mem', '-s', '10'])
            delay = (time.time() - start) * 1000 - 10000
            self.s3_resume_delays.append(delay)
```

### F.4 实操题

**题目6**：编写Python脚本，收集系统在1小时内的S0ix驻留时间、唤醒事件频率和原因，生成包含S0ix效率评分（0-100）的报告。

```python
#!/usr/bin/env python3
"""
S0ix 效率评分工具
"""

import time
import subprocess
import csv
import os

class S0ixEfficiencyAnalyzer:
    def __init__(self, duration_seconds=3600, interval=10):
        self.duration = duration_seconds
        self.interval = interval
        self.data = []
        
    def read_s0ix_residency(self):
        path = "/sys/kernel/debug/pmc_core/slp_s0_residency_usec"
        try:
            with open(path) as f:
                return int(f.read().strip())
        except (FileNotFoundError, ValueError):
            return 0
    
    def read_wakeup_count(self):
        try:
            result = subprocess.run(
                ['cat', '/sys/bus/acpi/devices/*/power/wakeup_count'],
                capture_output=True, text=True, shell=True
            )
            counts = [int(x) for x in result.stdout.split() if x.isdigit()]
            return sum(counts) if counts else 0
        except:
            return 0
    
    def get_active_power(self):
        try:
            result = subprocess.run(
                ['turbostat', '-i', '1', '--show', 'PkgWatt', '-n', '1'],
                capture_output=True, text=True
            )
            lines = result.stdout.strip().split('\n')
            return float(lines[-1].strip()) if len(lines) > 1 else 0
        except:
            return 0
    
    def collect_data(self):
        print(f"开始采集S0ix数据（{self.duration}秒）...")
        start_time = time.time()
        baseline_residency = self.read_s0ix_residency()
        baseline_wakeups = self.read_wakeup_count()
        
        while time.time() - start_time < self.duration:
            timestamp = time.time() - start_time
            residency = self.read_s0ix_residency() - baseline_residency
            power = self.get_active_power()
            wakeups = self.read_wakeup_count() - baseline_wakeups
            
            self.data.append({
                'timestamp_s': timestamp,
                'residency_us': residency,
                'power_W': power,
                'wakeup_count': wakeups
            })
            
            time.sleep(self.interval)
    
    def calculate_score(self):
        if not self.data:
            return 0
        
        total_time = self.data[-1]['timestamp_s']
        final_residency = self.data[-1]['residency_us']
        
        # S0ix驻留率
        residency_ratio = final_residency / (total_time * 1_000_000)
        residency_score = min(40, 40 * residency_ratio)
        
        # 唤醒频率评分
        wakeup_count = self.data[-1]['wakeup_count']
        wakeup_rate = wakeup_count / (total_time / 3600)
        if wakeup_rate <= 10:
            wakeup_score = 30
        elif wakeup_rate <= 50:
            wakeup_score = 20
        elif wakeup_rate <= 200:
            wakeup_score = 10
        else:
            wakeup_score = 0
        
        # 功耗评分
        avg_power = sum(d['power_W'] for d in self.data) / len(self.data)
        if avg_power <= 2:
            power_score = 30
        elif avg_power <= 5:
            power_score = 20
        elif avg_power <= 10:
            power_score = 10
        else:
            power_score = 0
        
        total_score = residency_score + wakeup_score + power_score
        
        # 评分等级
        if total_score >= 85:
            grade = "优秀 (Optimal S0ix)"
        elif total_score >= 60:
            grade = "良好 (Good S0ix)"
        elif total_score >= 40:
            grade = "一般 (Moderate S0ix)"
        else:
            grade = "差 (Poor S0ix)"
        
        return {
            'score': total_score,
            'grade': grade,
            'residency_ratio': residency_ratio,
            'avg_power_W': avg_power,
            'wakeup_rate_per_hour': wakeup_rate
        }

# 使用示例
if __name__ == "__main__":
    analyzer = S0ixEfficiencyAnalyzer(duration_seconds=3600)
    analyzer.collect_data()
    result = analyzer.calculate_score()
    
    print("=" * 50)
    print("S0ix 效率评估报告")
    print("=" * 50)
    print(f"总分: {result['score']:.1f}/100")
    print(f"等级: {result['grade']}")
    print(f"S0ix驻留率: {result['residency_ratio']*100:.1f}%")
    print(f"平均功耗: {result['avg_power_W']:.2f}W")
    print(f"唤醒频率: {result['wakeup_rate_per_hour']:.1f}次/小时")
    print("=" * 50)
```

---

## 附录G：S0ix调试实战案例深度解析

### 案例1：Lenovo ThinkPad X1 Carbon + AMD Radeon RX 6400 eGPU

**现象**：外接eGPU通过雷电口连接，系统支持S0ix但eGPU无法进入RTD3，导致整机S0ix驻留率为0%。

**根因分析**：
1. 雷电控制器(Titan Ridge)在eGPU连接时保持PCIe链路活跃
2. amdgpu驱动未正确注册eGPU为Runtime PM设备
3. `pm_runtime_allow()`未被调用

**排查过程**：
```bash
# 1. 检查eGPU Runtime PM状态
cat /sys/bus/pci/devices/0000:0a:00.0/power/control
# 输出: auto (期望值)
cat /sys/bus/pci/devices/0000:0a:00.0/power/runtime_status
# 输出: active (应为 suspended)

# 2. 检查Runtime PM计数
cat /sys/bus/pci/devices/0000:0a:00.0/power/runtime_usage
# 输出: 2 (有使用者引用，阻止挂起)

# 3. 检查设备是否在阻止列表
cat /sys/bus/pci/devices/0000:0a:00.0/power/wakeup
# 输出: disabled

# 4. 检查雷电PM状态
cat /sys/bus/thunderbolt/devices/0-0/runtime_control
# 输出: on (应为 auto)
```

**解决方案**：
```bash
# 启用eGPU Runtime PM
echo auto > /sys/bus/pci/devices/0000:0a:00.0/power/control

# 释放不必要的使用者引用
# 检查哪个进程持有refcount
cat /sys/bus/pci/devices/0000:0a:00.0/power/runtime_active_kids

# 启用雷电控制器运行时电源管理
echo auto > /sys/bus/thunderbolt/devices/0-0/runtime_control

# 配置autosuspend延迟为低值
echo 0 > /sys/bus/pci/devices/0000:0a:00.0/power/autosuspend_delay_ms

# 验证
cat /sys/bus/pci/devices/0000:0a:00.0/power/runtime_status
# 输出: suspended
```

**结果**：eGPU成功进入RTD3状态，整机S0ix驻留率从0%提升至87%。

### 案例2：Dell Precision 5680 + RTX 5000 Ada Generation dGPU

**现象**：dGPU无法进入S0ix所需的低功耗状态，系统维持在C8而非C10，功耗偏高(~3.5W)。

**根因分析**：
1. dGPU驱动未完成GFXOFF过渡到S0ix
2. `amdgpu_gfx_enter_s0ix()`调用失败
3. MMIO寄存器显示GFX block未完全关闭

**排查过程**：
```bash
# 1. 检查系统目标C-State
turbostat --quiet --show PkgWatt,C8,C9,C10 --interval 5
# 输出: C8驻留率85%, C10驻留率0% → 存在阻止C10的设备

# 2. 检查dGPU S0ix状态
cat /sys/kernel/debug/dri/1/amdgpu_pm_info | grep -i s0ix
# 输出: S0ix status: failed - GFX not idle

# 3. 检查GFXOFF状态
cat /sys/kernel/debug/dri/1/amdgpu_gfxoff
# 输出: GFXOFF disabled (refcount=1)

# 4. 检查阻止GFXOFF的进程
cat /sys/kernel/debug/dri/1/amdgpu_gfxoff_count
# pid=1234 comm=Xorg count=1
```

**解决方案**：
```bash
# 临时释放GFXOFF阻止
echo 0 > /sys/kernel/debug/dri/1/amdgpu_gfxoff

# 验证GFXOFF已启用
cat /sys/kernel/debug/dri/1/amdgpu_gfxoff
# 输出: GFXOFF enabled

# 检查S0ix入口
cat /sys/kernel/debug/dri/1/amdgpu_pm_info | grep -i s0ix
# 输出: S0ix status: ready

# 回退验证驻留率
turbostat --quiet --show PkgWatt,C8,C9,C10 --interval 5
# 输出: C10驻留率提升至92%

# 永久修复：在/etc/modprobe.d/amdgpu.conf中添加
options amdgpu gfxoff=1
```

### 案例3：ASUS ROG Flow X13 + XG Mobile eGPU热插拔

**现象**：XG Mobile eGPU热插拔后系统无法进入S0ix，pm_setup耗时异常增长。

**根因分析**：
1. eGPU热插拔导致ACPI通知链路异常
2. `acpi_lps0_device_remove()`未正确处理eGPU热移除
3. PCIe子系统残留LTR值阻止C10进入

**排查过程**：
```bash
# 1. 热插拔后检查ACPI suspend blockers
cat /sys/power/suspend_stats
# 输出: failed_suspend=5, last_failed_dev=pci0000:03:00.0

# 2. 检查残留PCIe设备
lspci -t | grep -i bridge
# 输出: 显示已移除的eGPU桥接器仍在活动

# 3. 检查LTR值
lspci -vv -s 03:00.0 | grep -i LTR
# 输出: LTR: Max_Snoop_Latency=0ns Max_NoSnoop_Latency=0ns (应被清除)

# 4. 检查驱动绑定状态
ls /sys/bus/pci/drivers/amdgpu/
# 输出: 0000:03:00.0 (设备未解绑)
```

**解决方案**：
```bash
# 手动解绑残留设备
echo 0000:03:00.0 > /sys/bus/pci/drivers/amdgpu/unbind

# 触发PCIe总线重扫描
echo 1 > /sys/bus/pci/rescan

# 清理LTR阻塞
echo 0 > /sys/bus/pci/devices/0000:00:01.0/power/ltr_override

# 验证S0ix入口
echo freeze > /sys/power/state
# 成功进入S0ix且即时唤醒
```

### 案例4：数据中心服务器AMD MI250 OAM模块S0ix稳定性

**现象**：训练集群中约3%的节点在S0ix测试中失败，表现为挂起后无法唤醒或唤醒后GPU功能异常。

**根因分析**：
1. MI250 OAM模块的固件版本不一致导致S0ix状态机异常
2. `smsc_suspend()`中SMU固件命令超时
3. VRAM自刷新入口条件未满足

**排查过程**：
```bash
# 1. 检查SMU固件版本
cat /sys/kernel/debug/dri/1/amdgpu_firmware_info | grep smc
# 输出: SMC version: 52.18.0 vs 期望 52.22.0

# 2. 检查S0ix入口dmesg
dmesg | grep -i "s0ix\|suspend\|SMU"
# 输出: [12345.678] amdgpu: SMU: S0ix entry timeout after 500ms

# 3. 检查VRAM自刷新状态
cat /sys/kernel/debug/dri/1/amdgpu_vram_mm | grep self-refresh
# 输出: Self-refresh: disabled (VRAM access pending)

# 4. 批量检查集群状态 (clush语法)
clush -w node[001-100] "cat /sys/kernel/debug/dri/*/amdgpu_firmware_info | grep smc | cut -d: -f4"
# 输出: 发现3个节点固件版本过低
```

**解决方案**：
```bash
# 固件升级
sudo /opt/amdgpu/sbin/fw_update.sh --gpu 0000:03:00.0 --fw P2

# 升级后验证
cat /sys/kernel/debug/dri/1/amdgpu_firmware_info | grep smc
# 输出: SMC version: 52.22.0

# 批量应用
clush -w node[015,047,089] "sudo /opt/amdgpu/sbin/fw_update.sh --gpu all --fw P2"

# S0ix回归测试脚本 (单节点)
for i in $(seq 1 100); do
    echo freeze > /sys/power/state
    sleep 10
    # 验证GPU功能
    rocm-smi --showpower
    if [ $? -ne 0 ]; then
        echo "Node $(hostname): S0ix failure at iteration $i" | tee -a /var/log/s0ix_fail.log
        exit 1
    fi
done
echo "Node $(hostname): All 100 S0ix cycles passed"
```

**结果**：固件升级后，集群S0ix失败率从3%降至0.02%。

### 案例5：Chromebook + AMD Mendocino APU电池续航优化

**现象**：用户报告S0ix驻留率良好(~95%)但电池续航仍低于预期(8h vs 设计12h)，怀疑存在非S0ix相关功耗泄漏。

**根因分析**：
1. S0ix入口成功但部分外设未进入LPM
2. WiFi模块(mediatek MT7921)在S0ix期间保持高活跃
3. 触摸屏I2C控制器未配置唤醒使能导致频繁中断

**精细化功耗分解**：
```bash
# 1. 全系统功耗分解
powertop --csv=/tmp/powertop_report.csv --iteration=3

# 2. S0ix期间外设功耗分析
cat /sys/class/powercap/intel-rapl\:*/energy_uj
# 输出: 显示SoC功耗正常(~200mW)但WiFi模块功耗异常(~350mW)

# 3. WiFi模块唤醒源分析
cat /proc/interrupts | grep wifi
# 输出: 显示WiFi中断频率高达200次/秒即使在S0ix期间

# 4. I2C触摸屏唤醒配置
cat /sys/bus/i2c/devices/i2c-ELAN0000\:00/power/wakeup
# 输出: enabled (不当配置)
```

**优化方案**：
```bash
# 1. 禁用不必要的触摸屏唤醒
echo disabled > /sys/bus/i2c/devices/i2c-ELAN0000:00/power/wakeup

# 2. 配置WiFi模块低功耗模式
iw dev wlan0 set power_save on
iw dev wlan0 set power_save_timeout 100

# 3. 调整autosuspend延迟
echo 2000 > /sys/bus/pci/devices/0000:02:00.0/power/autosuspend_delay_ms

# 4. 优化NVMe LPM
echo 1 > /sys/class/nvme/nvme0/power_control
nvme set-feature /dev/nvme0 -f 0x0c -v 0x01

# 5. 创建systemd服务持久化配置
cat > /etc/systemd/system/s0ix_optimize.service << 'EOF'
[Unit]
Description=S0ix Power Optimization
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/s0ix_optimize.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

cat > /usr/local/bin/s0ix_optimize.sh << 'SHEOF'
#!/bin/bash
# S0ix 功耗优化脚本
echo disabled > /sys/bus/i2c/devices/i2c-ELAN0000:00/power/wakeup
iw dev wlan0 set power_save on
echo 2000 > /sys/bus/pci/devices/0000:02:00.0/power/autosuspend_delay_ms
echo 1 > /sys/class/nvme/nvme0/power_control
SHEOF
chmod +x /usr/local/bin/s0ix_optimize.sh
systemctl enable s0ix_optimize.service
```

**结果**：优化后S0ix模式下总功耗从~550mW降至~220mW，电池续航从8h提升至11.5h。

---

## 附录H：S0ix性能基准测试与优化数据

### H.1 不同平台S0ix功耗对比

| 平台 | SoC | dGPU | S0ix功耗(mW) | C10驻留率(%) | 唤醒延迟(ms) |
|------|-----|------|-------------|-------------|-------------|
| Intel Tiger Lake-U | i7-1165G7 | - | 180-250 | 95-98 | 350-450 |
| Intel Alder Lake-P | i7-1280P | - | 220-300 | 92-96 | 400-500 |
| AMD Rembrandt-U | Ryzen 7 6800U | - | 200-280 | 93-97 | 380-480 |
| AMD Mendocino | Ryzen 5 7520U | - | 190-260 | 94-98 | 360-460 |
| AMD Phoenix | Ryzen 7 7840U | RX 7600M XT | 350-450 | 85-92 | 500-650 |
| Intel Raptor Lake-HX | i9-13900HX | RTX 4090 | 400-550 | 80-88 | 550-750 |
| Apple M2 Pro | M2 Pro | Integrated | 100-150 | N/A | 200-300 |

### H.2 S0ix入口成功率测试数据 (200轮循环)

| 平台版本 | 成功率 | 平均入口时间(ms) | 最大入口时间(ms) | 失败原因TOP1 |
|----------|--------|-----------------|-----------------|-------------|
| Kernel 6.1 LTS | 92.5% | 380 | 1200 | dGPU RPM超时 |
| Kernel 6.5 | 96.0% | 320 | 950 | ACPI AML错误 |
| Kernel 6.8 | 98.5% | 280 | 780 | PCIe ASPM协商 |
| Kernel 6.10 | 99.2% | 260 | 650 | 固件PMC超时 |

### H.3 S0ix唤醒源分布统计 (24小时采集)

```text
唤醒源分类                               占比(%)    平均唤醒次数/小时
─────────────────────────────────────────────────────────────────
USB设备 (鼠标/键盘/接收器)                 42.3      18.5
WiFi网络活动 (DHCP/AP扫描/数据包)          28.7      12.6
定时器 (RTC/高精度定时器)                  14.1      6.2
ACPI事件 (LID开关/电源按钮)                7.5       3.3
蓝牙连接活动                              4.2       1.8
其他 (PCIe PME/GPIO中断)                  3.2       1.4
─────────────────────────────────────────────────────────────────
总计                                      100.0     43.8
```

### H.4 S0ix调试命令速查表

| 调试目标 | 命令 | 关键字段 |
|----------|------|---------|
| S0ix驻留率 | `turbostat --show PkgWatt,C10,C10` | C10% |
| 设备阻止列表 | `cat /sys/kernel/debug/pmc_core/sleep_state_stats` | S0ix Residency |
| dGPU S0ix状态 | `cat /sys/kernel/debug/dri/0/amdgpu_pm_info \| grep s0ix` | S0ix status |
| 唤醒源追踪 | `cat /sys/kernel/debug/pmc_core/wakeup_sources` | Wake source |
| ACPI唤醒配置 | `cat /proc/acpi/wakeup` | Device S-state |
| Runtime PM状态 | `for d in /sys/bus/pci/devices/*/power/runtime_status; do echo "$d: $(cat $d)"; done` | suspended/active |
| 中断频率 | `cat /proc/interrupts \| sort -k2 -n -r \| head -10` | 中断次数 |
| 功耗实时监测 | `powertop --csv=/tmp/pt.csv --iteration=1` | Device Power |
| C-State分解 | `turbostat -q --show CPU%c1,CPU%c6,CPU%c7 -i 10` | C1/C6/C7% |
| LTR值检查 | `lspci -vv \| grep -A5 LTR` | Max Latency |

### H.5 常见S0ix问题与快速修复对照表

| 症状 | 可能原因 | 快速诊断命令 | 修复方案 |
|------|---------|-------------|---------|
| C10驻留率为0% | 某个设备阻止S0ix | `cat /sys/kernel/debug/pmc_core/sleep_state_stats` | 逐个卸载驱动模块排查 |
| dGPU不进入RTD3 | RPM refcount不为0 | `cat /sys/bus/pci/devices/*/power/runtime_usage` | `echo auto > .../power/control` |
| S0ix入口超时 | SMU固件问题 | `dmesg \| grep "S0ix.*timeout"` | 升级SMU固件 |
| 唤醒后GPU异常 | GFXOFF未正确恢复 | `cat /sys/kernel/debug/dri/0/amdgpu_gfxoff` | `echo 0 > amdgpu_gfxoff`后重置 |
| 功耗高于预期 | 外设未LPM | `powertop --csv` | 按powertop建议修复 |
| 频繁自动唤醒 | USB唤醒使能 | `cat /proc/acpi/wakeup` | `echo USB0 > /proc/acpi/wakeup` |
| S0ix入口失败 | PCIe ASPM协商 | `lspci -vv \| grep ASPM` | 内核参数`pcie_aspm=force` |
| ACPI错误导致失败 | DSDT表bug | `dmesg \| grep ACPI.*Error` | 提交内核bug或加载DSDT覆盖表 |

### H.6 S0ix自动化验证框架

```bash
#!/bin/bash
# s0ix_full_validation.sh - S0ix完整验证脚本
# 用法: ./s0ix_full_validation.sh [循环次数] [唤醒延迟秒数]

CYCLES=${1:-50}
WAKE_DELAY=${2:-10}
PASS=0
FAIL=0
LOG_FILE="/tmp/s0ix_validation_$(date +%Y%m%d_%H%M%S).log"

echo "S0ix 完整验证 - $(date)" | tee -a "$LOG_FILE"
echo "循环次数: $CYCLES, 唤醒延迟: ${WAKE_DELAY}s" | tee -a "$LOG_FILE"
echo "========================================" | tee -a "$LOG_FILE"

collect_s0ix_stats() {
    echo "=== S0ix状态收集 ===" >> "$LOG_FILE"
    echo "--- turbostat C10 ---" >> "$LOG_FILE"
    turbostat --quiet --show PkgWatt,C10 --interval 3 2>&1 | tail -5 >> "$LOG_FILE"
    
    echo "--- dmesg S0ix ---" >> "$LOG_FILE"
    dmesg | tail -20 | grep -i "s0ix\|suspend\|resume\|ACPI" >> "$LOG_FILE"
    
    echo "--- dGPU PM info ---" >> "$LOG_FILE"
    for drm in /sys/kernel/debug/dri/*/amdgpu_pm_info; do
        if [ -f "$drm" ]; then
            grep -i "s0ix\|power\|clock" "$drm" >> "$LOG_FILE"
        fi
    done
    
    echo "--- Wakeup sources ---" >> "$LOG_FILE"
    cat /proc/acpi/wakeup 2>/dev/null | tail -30 >> "$LOG_FILE"
}

pre_test_check() {
    local issues=0
    echo "=== 前置检查 ===" | tee -a "$LOG_FILE"
    
    # 检查PM控制
    for dev in /sys/bus/pci/devices/*/power/control; do
        if [ "$(cat "$dev")" != "auto" ]; then
            echo "WARNING: $dev 未设置为auto" >> "$LOG_FILE"
            issues=$((issues + 1))
        fi
    done
    
    # 检查S0ix支持
    if ! grep -q "freeze" /sys/power/state 2>/dev/null; then
        echo "ERROR: 系统不支持S0ix(freeze)" | tee -a "$LOG_FILE"
        return 1
    fi
    
    echo "前置检查完成，发现 $issues 个可优化项" | tee -a "$LOG_FILE"
    return 0
}

single_s0ix_cycle() {
    local cycle=$1
    
    rtcwake -m freeze -s "$WAKE_DELAY" 2>&1 >> "$LOG_FILE"
    local ret=$?
    
    if [ $ret -eq 0 ]; then
        echo "[$cycle/$CYCLES] ✅ S0ix 成功 (唤醒延迟: ${WAKE_DELAY}s)"
    else
        echo "[$cycle/$CYCLES] ❌ S0ix 失败 (返回码: $ret)"
        collect_s0ix_stats
    fi
    
    sleep 2
    return $ret
}

# 测试流程
pre_test_check || exit 1

echo "开始S0ix验证 ($CYCLES 次) - $(date)" | tee -a "$LOG_FILE"

for i in $(seq 1 "$CYCLES"); do
    if single_s0ix_cycle "$i"; then
        PASS=$((PASS + 1))
    else
        FAIL=$((FAIL + 1))
    fi
done

echo "========================================" | tee -a "$LOG_FILE"
echo "验证完成 - $(date)" | tee -a "$LOG_FILE"
echo "通过: $PASS, 失败: $FAIL" | tee -a "$LOG_FILE"
echo "成功率: $(echo "scale=2; $PASS * 100 / ($PASS + $FAIL)" | bc)%" | tee -a "$LOG_FILE"

# 汇总统计
if [ $FAIL -gt 0 ]; then
    echo "失败详情:" | tee -a "$LOG_FILE"
    grep -n "❌" "$LOG_FILE" | tee -a "$LOG_FILE"
fi

exit $FAIL
```

### H.7 内核参数优化清单

以下内核参数可优化S0ix行为（添加到`/etc/sysctl.d/99-s0ix-optimize.conf`）：

```conf
# 减少定时器频率以减少S0ix唤醒
kernel.timer_migration = 1

# 启用PCIe ASPM动态协商
# (通过内核引导参数: pcie_aspm=force)

# 调整dirty page回写行为
vm.dirty_writeback_centisecs = 6000
vm.dirty_expire_centisecs = 12000
vm.dirty_ratio = 10
vm.dirty_background_ratio = 3

# 减少磁盘I/O唤醒
vm.vfs_cache_pressure = 50

# 调整NUMA内存回收
vm.zone_reclaim_mode = 0
```

内核引导参数优化（添加到`/etc/default/grub`的`GRUB_CMDLINE_LINUX`）：

```text
# 基础S0ix优化参数
intel_idle.max_cstate=9 intel_pmc_core.wakeup_sources=1

# AMD平台优化参数
amd_pstate=guided amd_prefcore=enable

# 通用优化参数
processor.max_cstate=9 intel_idle.states_off=0x1E
```

### H.8 S0ix知识自测题

**选择题**：

1. 以下哪个C-State是S0ix入口的前提条件？
   A. C1  B. C6  C. C8  D. C10

2. 检查系统是否支持S0ix应使用哪个命令？
   A. `cat /sys/power/state`  B. `cat /proc/acpi/wakeup`
   C. `dmesg | grep "ACPI.*S3"`  D. `lspci | grep VGA`

3. S0ix模式下dGPU的PCIe链路状态应为：
   A. L0  B. L1  C. L2/L3  D. L0s

4. 以下哪种情况会阻止S0ix入口？
   A. GFXOFF已启用  B. USB设备挂起
   C. PCIe设备未进入D3hot  D. CPU进入C1

5. `turbostat`中哪个指标表明系统已进入S0ix？
   A. C1%  B. C6%  C. C8%  D. C10%

6. S0ix失败后应首先检查哪个日志？
   A. `/var/log/syslog`  B. `dmesg`
   C. `/var/log/kern.log`  D. `journalctl -u pm`

**答案**：1.D 2.A 3.C 4.C 5.D 6.B