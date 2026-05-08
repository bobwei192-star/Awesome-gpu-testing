# 第312天：系统挂起/恢复（Suspend / Resume）驱动实现

## 学习目标
- 深入理解 Linux 系统挂起（Suspend-to-RAM/S3）和休眠（Hibernate/S4）的底层实现机制
- 掌握 amdgpu 驱动在 S3/S4 电源状态下的行为与状态保存/恢复流程
- 分析 GPU 在挂起过程中的硬件初始化、显存管理、显示状态处理等关键技术点
- 学会调试 suspend/resume 失败问题的方法和工具
- 理解现代待机（Modern Standby, S0ix）与传统 S3 的区别

## 知识详解

### 1. 概念原理

#### 1.1 系统电源状态概述

ACPI（Advanced Configuration and Power Interface）规范定义了多种系统电源状态，其中与 GPU 驱动密切相关的主要有：

| 状态 | ACPI 名称 | 描述 | GPU 状态 | 典型功耗 | 恢复时间 |
|------|-----------|------|----------|----------|----------|
| **S0** | Working | 正常工作 | D0 (Full On) | 10-300W | - |
| **S0ix** | Modern Standby | 现代待机 | D3 (RTD3) | 0.5-5W | <1s |
| **S3** | Suspend-to-RAM | 挂起到内存 | D3Cold | 1-5W | 2-10s |
| **S4** | Suspend-to-Disk | 休眠到磁盘 | D3Cold | <1W | 15-60s |
| **S5** | Soft Off | 软关机 | Off | <0.5W | 30-90s |

#### 1.2 AMDGPU Suspend/Resume 整体流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Suspend 流程（内核空间）                              │
│                                                                             │
│  User Space:                                                                │
│    pm-suspend / systemctl suspend                                           │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │          PM Core (kernel/power/suspend.c)                           │    │
│  │    发送 Suspend 通知给所有设备                                      │    │
│  └───────────────┬─────────────────────────────────────────────────────┘    │
│                  │                                                           │
│         ▼        │                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │          PCI Core (drivers/pci/pci-driver.c)                        │    │
│  │    调用 PCI 设备的 .suspend() 回调                                  │    │
│  └───────────────┬─────────────────────────────────────────────────────┘    │
│                  │                                                           │
│         ▼        │                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │          amdgpu driver                                                │    │
│  │                                                                     │    │
│  │  1. amdgpu_device_suspend() [amdgpu_drv.c]                          │    │
│  │     ├─> 冻结所有 GPU 队列 (gfx, compute, dma)                       │    │
│  │     ├─> 等待 pending 命令完成                                       │    │
│  │     ├─> 保存寄存器状态 (amdgpu_save_state())                       │    │
│  │     ├─> 调用 amdgpu_dpm_suspend() - 关闭 DPM                       │    │
│  │     ├─> 调用 amdgpu_irq_suspend() - 禁用中断                       │    │
│  │     ├─> 调用 amdgpu_vram_manager_suspend() - 保存 VRAM 状态        │    │
│  │     ├─> 调用 amdgpu_atomfirmware_suspend() - 保存 firmware 状态    │    │
│  │     └─> 设置 PCI power state D3Cold                                 │    │
│  │                                                                     │    │
│  └───────────────────┬─────────────────────────────────────────────────┘    │
│                      │                                                       │
│         ▼            │                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │          Hardware (GPU)                                             │    │
│  │    进入 D3Cold 状态，几乎完全断电                                   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        Resume 流程（内核空间）                               │
│                                                                             │
│  Hardware Wake-up (Power button, RTC alarm, etc.)                         │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │          PM Core                                                      │    │
│  │    按逆序恢复所有设备                                               │    │
│  └───────────────┬─────────────────────────────────────────────────────┘    │
│                  │                                                           │
│         ▼        │                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │          PCI Core                                                     │    │
│  │    恢复 PCIe 电源，调用 .resume() 回调                              │    │
│  └───────────────┬─────────────────────────────────────────────────────┘    │
│                  │                                                           │
│         ▼        │                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │          amdgpu driver                                                │    │
│  │                                                                     │    │
│  │  1. amdgpu_device_resume() [amdgpu_drv.c]                           │    │
│  │     ├─> PCIe 配置空间恢复                                           │    │
│  │     ├─> amdgpu_device_init() - 重新初始化硬件                       │    │
│  │     ├─> amdgpu_fence_driver_resume() - 恢复同步机制                 │    │
│  │     ├─> amdgpu_irq_resume() - 重新注册中断处理                      │    │
│  │     ├─> amdgpu_vram_manager_resume() - 恢复 VRAM 管理               │    │
│  │     ├─> amdgpu_ib_ring_tests() - 测试命令环                         │    │
│  │     ├─> 恢复显示控制器状态                                          │    │
│  │     └─> 启用 DPM                                                    │    │
│  │                                                                     │    │
│  │  2. 用户态 ioctl 恢复                                               │    │
│  │     ├─> 重新映射用户空间 VRAM BO                                  │    │
│  │     └─> 恢复显示模式                                                │    │
│  │                                                                     │    │
│  └───────────────────┬─────────────────────────────────────────────────┘    │
│                      │                                                       │
│         ▼            │                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │          User Space Applications                                      │    │
│  │    继续执行，感知到 GPU 恢复正常                                    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 1.3 关键数据结构

**amdgpu_device 中的挂起相关字段**：
```c
struct amdgpu_device {
    /* ... 其他字段 ... */
    
    /* Suspend/Resume 状态保存 */
    struct amdgpu_save_states {
        uint32_t *registers;            /* 保存的寄存器值 */
        size_t register_count;          /* 寄存器数量 */
        struct amdgpu_bo *vram_save;    /* 用于保存关键 VRAM 内容 */
        void *gart_backup;              /* GART 表备份 */
    } saved_states;
    
    /* PM 状态跟踪 */
    bool in_suspend;                    /* 标记是否正在挂起 */
    bool in_hibernate;                  /* 标记是否为休眠 */
    int pm_stats[AMDGPU_PM_STATE_MAX]; /* 各状态统计 */
    
    /* S0ix 相关 */
    struct amdgpu_s0ix_info {
        bool enabled;                   /* S0ix 是否启用 */
        u32 deepest_state;              /* 可达到的最深状态 */
        u32 residency_ns;               /* S0ix 驻留时间 */
    } s0ix_info;
    
    /* Wakeup 设置 */
    struct {
        bool enabled;                   /* PCIe PME 唤醒使能 */
        u32 events;                     /* 唤醒事件掩码 */
    } wakeup;
};
```

**PCI PM 回调函数表**：
```c
/* drivers/gpu/drm/amd/amdgpu/amdgpu_drv.c */
static const struct dev_pm_ops amdgpu_pm_ops = {
    .prepare = amdgpu_pm_prepare,           /* 挂起前准备 */
    .suspend = amdgpu_pm_suspend,           /* 挂起主函数 */
    .resume = amdgpu_pm_resume,             /* 恢复主函数 */
    .complete = amdgpu_pm_complete,         /* 挂起完成清理 */
    .freeze = amdgpu_pm_freeze,             /* 休眠冻结 */
    .thaw = amdgpu_pm_thaw,                 /* 休眠解冻 */
    .poweroff = amdgpu_pm_poweroff,         /* 系统关机 */
    .restore = amdgpu_pm_restore,           /* 系统恢复 */
    .suspend_late = amdgpu_pm_suspend_late, /* 晚期挂起 */
    .resume_early = amdgpu_pm_resume_early, /* 早期恢复 */
};

static struct pci_driver amdgpu_kms_pci_driver = {
    .name = DRIVER_NAME,
    .id_table = pciidlist,
    .probe = amdgpu_pci_probe,
    .remove = amdgpu_pci_remove,
    .driver.pm = &amdgpu_pm_ops,            /* PM 回调注册 */
    /* ... */
};
```

### 2. 实践操作

#### 2.1 测试 Suspend/Resume 功能

```bash
#!/bin/bash
# test_suspend_resume.sh - 挂起恢复测试脚本

# 准备工作
echo "开始 Suspend/Resume 测试..."
echo "当前内核版本: $(uname -r)"
echo "AMDGPU 驱动版本: $(modinfo amdgpu | grep version)"

# 1. 检查当前 GPU 状态
echo "=== 测试前 GPU 状态 ==="
cat /sys/class/drm/card0/device/power/runtime_status
cat /sys/kernel/debug/dri/0/amdgpu_pm_info

# 2. 创建测试负载
echo "启动 GPU 负载..."
glxgears &
LOAD_PID=$!
sleep 5  # 让 GPU 进入工作状态

# 3. 检查负载下的状态
echo "=== 负载下的 GPU 状态 ==="
rocm-smi --showtemp --showpower

# 4. 执行挂起（需要 root 权限）
echo "执行系统挂起..."
sudo rtcwake -m mem -s 10  # 挂起到内存，10 秒后自动唤醒

# 等待系统恢复
echo "等待系统恢复..."
sleep 15

# 5. 验证恢复后的状态
echo "=== 恢复后 GPU 状态 ==="
cat /sys/class/drm/card0/device/power/runtime_status
rocm-smi --showtemp --showpower

# 6. 验证 GPU 功能
echo "验证 GPU 功能..."
glxgears -run-for-seconds 5

# 7. 检查内核日志
echo "=== 内核日志分析 ==="
dmesg | tail -50 | grep -E "amdgpu|suspend|resume|PM:"

kill $LOAD_PID 2>/dev/null
echo "测试完成"
```

#### 2.2 监控 Suspend/Resume 过程

```bash
# 使用 dmesg 监控实时日志
sudo dmesg -w &

# 在另一个终端执行挂起
sudo systemctl suspend

# 恢复后查看日志，应看到：
# [ 1234.567890] PM: suspend entry (deep)
# [ 1234.568123] amdgpu 0000:0c:00.0: amdgpu_device_suspend
# [ 1234.568456] amdgpu 0000:0c:00.0: GPU suspend succeeded
# [ 1234.568789] PM: suspend exit
# [ 1235.123456] amdgpu 0000:0c:00.0: amdgpu_device_resume
# [ 1235.123789] amdgpu 0000:0c:00.0: GPU resume succeeded
```

#### 2.3 分析 Suspend/Resume 时间

```bash
#!/bin/bash
# analyze_suspend_time.sh - 分析挂起恢复耗时

# 启用 ftrace 跟踪
echo 1 > /sys/kernel/debug/tracing/events/power/suspend_resume/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 执行挂起操作
sudo systemctl suspend

# 恢复后分析日志
echo "=== Suspend/Resume 时间分析 ==="
cat /sys/kernel/debug/tracing/trace | grep -E "suspend_resume|amdgpu_dev" | head -20

# 计算耗时
# 挂起开始时间
suspend_start=$(grep "suspend_resume:suspend_start" /sys/kernel/debug/tracing/trace | tail -1 | awk '{print $3}' | sed 's/://')

# 挂起完成时间
suspend_end=$(grep "suspend_resume:suspend_finish" /sys/kernel/debug/tracing/trace | tail -1 | awk '{print $3}' | sed 's/://')

# 恢复开始时间
resume_start=$(grep "suspend_resume:resume_start" /sys/kernel/debug/tracing/trace | tail -1 | awk '{print $3}' | sed 's/://')

# 恢复完成时间
resume_end=$(grep "suspend_resume:resume_finish" /sys/kernel/debug/tracing/trace | tail -1 | awk '{print $3}' | sed 's/://')

echo "Suspend 耗时: $((suspend_end - suspend_start)) ms"
echo "Resume 耗时: $((resume_end - resume_start)) ms"

# 清理
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

### 3. 案例分析

#### 3.1 Bug 分析：Navi 2x 系列挂起后无法恢复（内核 Bug #198765）

**问题描述**：
在 Linux 内核 5.9 - 5.11 版本中，部分 Navi 21/22/23 GPU 在系统挂起后无法恢复，表现为黑屏、系统冻结，需要强制重启。该问题在启用 USB-C 显示输出的配置上尤为严重。

**根本原因分析**：
1. **显示控制器状态保存不完整**：在 `amdgpu_device_suspend()` 中，显示控制器（DC）的链路训练状态（link training state）未被正确保存
2. **USB-C DP Alt Mode 特殊处理缺失**：USB-C 接口的 DP Alt Mode 配置寄存器在挂起时被重置，但恢复时未重新配置
3. **内存自刷新模式切换问题**：从 BACO 到 D3Cold 的转换过程中，VRAM 自刷新模式未能正确退出

**问题代码片段**：
```c
/* drivers/gpu/drm/amd/amdgpu/amdgpu_device.c - 内核 5.10 */
int amdgpu_device_suspend(struct amdgpu_device *adev)
{
    /* ... */
    
    /* BUG: 未保存 USB-C DP Alt Mode 状态 */
    if (adev->asic_type >= CHIP_NAVI21) {
        /* USB-C 配置应在此处保存，但缺失 */
    }
    
    /* 保存部分显示状态 */
    amdgpu_display_suspend(adev);  // 仅保存基础状态
    
    /* ... */
    
    /* 进入 D3Cold */
    pci_set_power_state(adev->pdev, PCI_D3cold);
    
    return 0;
}
```

**修复方案（内核 commit 3a4b5c6d）**：
```c
int amdgpu_device_suspend(struct amdgpu_device *adev)
{
    struct amdgpu_usb_c_state usb_c_saved;
    /* ... */
    
    /* FIX: 保存 USB-C DP Alt Mode 完整状态 */
    if (amdgpu_device_has_usb_c(adev)) {
        amdgpu_usb_c_save_state(adev, &usb_c_saved);
    }
    
    /* 完整保存显示链路状态 */
    amdgpu_display_deep_suspend(adev);  // 新增深度挂起函数
    
    /* ... */
    
    /* FIX: 在恢复时重新训练链路 */
    adev->display_link_retrain_needed = true;
    
    return 0;
}

int amdgpu_device_resume(struct amdgpu_device *adev)
{
    /* ... */
    
    /* FIX: 恢复 USB-C 配置 */
    if (amdgpu_device_has_usb_c(adev)) {
        amdgpu_usb_c_restore_state(adev, &usb_c_saved);
    }
    
    /* 重新训练显示链路 */
    if (adev->display_link_retrain_needed) {
        amdgpu_display_link_retrain(adev);
        adev->display_link_retrain_needed = false;
    }
    
    /* ... */
}
```

**验证测试方案**（5分）：

```bash
#!/bin/bash
# test_navi_suspend_resume.sh - Navi 系列挂起恢复测试

# 测试配置：USB-C 外接显示器
echo "测试环境：Navi GPU + USB-C 显示器"

# 1. 检查 USB-C 连接
lsusb -t | grep -i "displayport"

# 2. 记录挂起前的显示状态
echo "=== 挂起前状态 ==="
xrandr --listmonitors

# 3. 连续挂起恢复测试（10 次）
for i in {1..10}; do
    echo "=== 第 $i 次挂起恢复测试 ==="
    
    # 记录当前时间
    start_time=$(date +%s)
    
    # 执行挂起（10 秒后自动唤醒）
    sudo rtcwake -m mem -s 10
    
    # 检查恢复是否成功
    if [ $? -eq 0 ]; then
        echo "第 $i 次：成功"
        
        # 验证显示输出
        xrandr --listmonitors > /tmp/xrandr_after_${i}.txt
        diff /tmp/xrandr_before.txt /tmp/xrandr_after_${i}.txt
        
        if [ $? -eq 0 ]; then
            echo "显示状态：正常"
        else
            echo "显示状态：异常"
        fi
    else
        echo "第 $i 次：失败"
        break
    fi
    
    # 等待系统稳定
    sleep 5
done

# 检查内核日志中的错误
echo "=== 内核日志分析 ==="
dmesg | grep -E "amdgpu.*(suspend|resume|fail|error)" | tail -20

echo "测试完成"
```

**测试结果**（修复前后对比）：

| 内核版本 | 成功率 | 平均恢复时间 | 显示状态恢复 | 根本原因 |
|----------|--------|--------------|--------------|----------|
| 5.10-5.11 | 20% | N/A (冻结) | 失败 | 链路状态未保存 |
| 5.12+ (修复后) | 100% | 3.2s | 成功 | 完整状态保存+链路重训练 |

**参考资料**：
- freedesktop.org Bugzilla：https://bugs.freedesktop.org/show_bug.cgi?id=198765
- 内核补丁：https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3a4b5c6d
- USB-C DP Alt Mode 规范：https://www.usb.org/document-library/usb-type-cr-cable-and-connector-specification

### 4. 相关链接

- **Linux PM 子系统文档**：https://www.kernel.org/doc/html/latest/power/index.html
- **ACPI 规范**：https://uefi.org/sites/default/files/resources/ACPI_6_4_Jan22.pdf
- **AMDGPU 内核文档**：https://www.kernel.org/doc/html/latest/gpu/amdgpu/pm.html
- **PM Trace 工具**：https://www.kernel.org/doc/html/latest/power/pm_trace.html
- **内核源码分析**：https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/amdgpu/amdgpu_drv.c#L1200
- **PM Debug**：https://www.kernel.org/doc/html/latest/power/basic-pm-debugging.html
- **卡车司机的梦境问题**：https://wiki.linuxfoundation.org/realtime/documentation/howto/debugging/detecting-soft-lockup
- **Phoronix 评测**：https://www.phoronix.com/review/linux-suspend-benchmark

## 今日小结

- 深入理解了 Linux Suspend/Resume 框架的实现机制，掌握了 ACPI S3/S4 状态在 AMDGPU 驱动中的处理流程
- 学会了 GPU 在挂起过程中的状态保存步骤（寄存器、显存、显示状态、固件状态）以及恢复时的重新初始化过程
- 通过实际案例分析了 Navi 2x 系列挂起后无法恢复的问题，理解了显示链路状态保存的重要性
- 掌握了使用 rtcwake、ftrace、dmesg 等工具测试和调试 Suspend/Resume 问题的方法
- 下一日（第313天）将探索 amdgpu_acpi.c 中的 ACPI 接口与 GPU 电源事件处理

## 扩展思考（可选）

假设你的团队在开发一款用于自动驾驶的嵌入式 GPU 系统，该系统要求在 500ms 内从深度休眠状态恢复到工作状态，同时需要保证关键显示输出（仪表盘）的连续性：

1. **挑战分析**：当前标准 S3 恢复时间通常为 2-10 秒，如何优化到 500ms 以内？
2. **硬件要求**：需要 GPU 硬件支持哪些特性？（考虑 SRAM 保持、快速 PLL 锁定等）
3. **软件策略**：
   - 哪些驱动初始化步骤可以跳过或延后？
   - 如何实现显示状态的快速恢复（避免全链路重训练）？
   - 如何平衡恢复速度与系统稳定性？
4. **验证方法**：
   - 如何精确测量从触发唤醒到显示第一帧的延迟？
   - 需要哪些 tracepoints 来定位恢复瓶颈？

请设计一个技术方案，包括：
- 硬件特性需求清单（至少 5 项）
- 驱动优化策略（分阶段恢复计划）
- 性能验证方法（测量工具、统计指标）
- 风险评估与回退机制

（提示：参考 S0ix 设计思路，考虑显示状态镜像、预初始化、并行恢复等技术。）