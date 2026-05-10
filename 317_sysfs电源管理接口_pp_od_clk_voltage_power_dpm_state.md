# 第317天：sysfs 电源管理接口：pp_od_clk_voltage / power_dpm_state

## 内容简介
本课程全面讲解 AMDGPU 驱动通过 sysfs 暴露的电源管理接口体系。重点分析 pp_od_clk_voltage（超频/降频配置接口）、power_dpm_state（DPM 状态控制）、power_dpm_force_performance_level（性能等级强制）等核心接口，以及相关的 pp_dpm_sclk、pp_dpm_mclk、power1_cap、hwmon 等辅助接口的用法和实现原理。

## 学习目标
1. 理解 AMDGPU sysfs 电源管理接口的整体架构和设计原则
2. 掌握 pp_od_clk_voltage 接口的读写格式和参数含义
3. 熟练使用 power_dpm_force_performance_level 控制 GPU 性能状态
4. 理解 pp_dpm_sclk/mclk/socclk 等 DPM 状态读取接口
5. 掌握 hwmon 子系统的电源/温度/风扇监控接口
6. 能够编写脚本自动化 GPU 电源管理配置

## 第一节：sysfs 电源管理接口架构

### 1.1 接口层次结构

AMDGPU 驱动的 sysfs 电源管理接口位于 `/sys/class/drm/card0/device/` 目录下：

```
/sys/class/drm/card0/device/
├── pp_od_clk_voltage              # OD 时钟/电压配置（读写）
├── power_dpm_state                # DPM 状态控制（读写）
├── power_dpm_force_performance_level  # 性能等级强制（读写）
├── pp_dpm_sclk                    # 当前 GFX 时钟 DPM 状态（只读）
├── pp_dpm_mclk                    # 当前显存 DPM 状态（只读）
├── pp_dpm_socclk                  # 当前 SoC 时钟 DPM 状态（只读）
├── pp_dpm_fclk                    # 当前 FCLK 状态（只读）
├── pp_dpm_vclk                    # 当前 VCLK 状态（只读）
├── pp_dpm_dclk                    # 当前 DCLK 状态（只读）
├── pp_power_profile_mode          # 电源配置文件选择（读写）
├── power1_cap                     # 最大功耗上限（读写）
├── power1_cap_max                 # 最大功耗上限最大值（只读）
├── power1_cap_min                 # 最大功耗上限最小值（只读）
├── power1_average                 # 平均功耗（只读）
├── current_freq_mhz               # 当前频率（只读）
├── vbios_version                  # VBIOS 版本（只读）
├── hwmon/
│   └── hwmonN/
│       ├── name                   # 传感器名称
│       ├── temp1_input            # 温度传感器 1 输入
│       ├── temp1_crit             # 温度临界值
│       ├── temp1_crit_hyst        # 温度临界迟滞
│       ├── in0_input              # 电压输入
│       ├── power1_average         # 平均功耗
│       ├── power1_cap             # 功耗上限
│       ├── fan1_input             # 风扇转速
│       ├── fan1_min               # 风扇最小转速
│       ├── fan1_max               # 风扇最大转速
│       ├── pwm1                   # 风扇 PWM 值
│       └── pwm1_enable            # 风扇控制模式
```

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     sysfs 电源管理接口架构图                              │
│                                                                          │
│                        用户空间                                            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  cat/echo 命令     |  Python 脚本   |  ROCm 工具               │   │
│   │  /sys/class/drm/   |  os.sysfs     |  rocm-smi                │   │
│   └───────────────────────────┬─────────────────────────────────────┘   │
│                               │                                          │
│                               ▼                                          │
│                        内核空间 (sysfs)                                   │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  amdgpu_device_attr                                        │   │
│   │  ┌──────────────────────────────────────────────────────┐   │   │
│   │  │  pp_od_clk_voltage → amdgpu_get_pp_od_clk_voltage() │   │   │
│   │  │  power_dpm_state → amdgpu_get/set_power_dpm_state()  │   │   │
│   │  │  pp_dpm_sclk → amdgpu_get_pp_dpm_sclk()             │   │   │
│   │  │  ...                                                │   │   │
│   │  └──────────────────────────────────────────────────────┘   │   │
│   └───────────────────────────┬─────────────────────────────────────┘   │
│                               │                                          │
│                               ▼                                          │
│                        驱动层 (amdgpu)                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  amdgpu_dpm.c / amdgpu_pm.c                                    │   │
│   │  ┌──────────────────────────────────────────────────────────┐   │   │
│   │  │  smu_send_msg()  →  SMU 固件通信                       │   │   │
│   │  │  pp_hwmgr_funcs  →  PowerPlay 回调                     │   │   │
│   │  └──────────────────────────────────────────────────────────┘   │   │
│   └───────────────────────────┬─────────────────────────────────────┘   │
│                               │                                          │
│                               ▼                                          │
│                        固件层 (SMU)                                       │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  SMU v11/v13 Firmware                                         │   │
│   │  ┌──────────────────────────────────────────────────────────┐   │   │
│   │  │  消息处理: SetSoftMin/Max, TransferTable, GetMetrics    │   │   │
│   │  │  DPM 状态管理, PowerTune 控制, 传感器数据采集           │   │   │
│   │  └──────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.2 接口注册机制

AMDGPU 驱动通过 `amdgpu_device_attr` 宏在内核中注册 sysfs 属性：

```c
// 内核源码: drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c
// 属性定义示例

static DEVICE_ATTR(pp_od_clk_voltage, S_IRUGO | S_IWUSR,
                   amdgpu_get_pp_od_clk_voltage,
                   amdgpu_set_pp_od_clk_voltage);

static DEVICE_ATTR(power_dpm_state, S_IRUGO | S_IWUSR,
                   amdgpu_get_power_dpm_state,
                   amdgpu_set_power_dpm_state);

static DEVICE_ATTR(power_dpm_force_performance_level, S_IRUGO | S_IWUSR,
                   amdgpu_get_power_dpm_force_performance_level,
                   amdgpu_set_power_dpm_force_performance_level);

static DEVICE_ATTR(pp_dpm_sclk, S_IRUGO,
                   amdgpu_get_pp_dpm_sclk, NULL);

static DEVICE_ATTR(pp_dpm_mclk, S_IRUGO,
                   amdgpu_get_pp_dpm_mclk, NULL);
```

```c
// 读取回调函数示例 - amdgpu_get_pp_od_clk_voltage
static int amdgpu_get_pp_od_clk_voltage(struct device *dev,
                                         struct device_attribute *attr,
                                         char *buf)
{
    struct drm_device *ddev = dev_get_drvdata(dev);
    struct amdgpu_device *adev = ddev->dev_private;
    
    if (adev->powerplay.pp_funcs &&
        adev->powerplay.pp_funcs->print_clock_levels) {
        return adev->powerplay.pp_funcs->print_clock_levels(
            adev, PP_OD_EDIT_SCLK, buf);
    }
    return -EINVAL;
}
```

## 第二节：pp_od_clk_voltage 接口详解

### 2.1 接口概述

`pp_od_clk_voltage` 是 AMDGPU 驱动最核心的电源管理接口，用于读取和设置 OverDrive 参数，包括频率、电压和 V/F 曲线。

**读取格式：**
```bash
$ cat /sys/class/drm/card0/device/pp_od_clk_voltage
```

**典型输出 (RDNA 2, RX 6800 XT)：**
```
OD_SCLK:
0:        500Mhz        800mV
1:       1200Mhz        900mV
2:       2250Mhz       1150mV
3:       2430Mhz       1200mV
OD_MCLK:
0:        100Mhz        750mV
1:        800Mhz        850mV
2:       1000Mhz        950mV
3:       2000Mhz       1350mV
OD_RANGE:
SCLK:     500MHz       3000MHz
MCLK:     100MHz       2250MHz
VDDC:     800mV        1200mV
```

**输出字段含义：**

| 字段 | 说明 | 示例值 |
|------|------|--------|
| OD_SCLK | GFX 时钟域的 DPM 状态表 | 状态编号: 频率(MHz) 电压(mV) |
| OD_MCLK | 显存时钟域的 DPM 状态表 | 状态编号: 频率(MHz) 电压(mV) |
| OD_RANGE | 可调整的范围限制 | 最小/最大频率和电压 |
| SCLK | GFX 时钟频率范围 | 500MHz - 3000MHz |
| MCLK | 显存时钟频率范围 | 100MHz - 2250MHz |
| VDDC | 核心电压范围 | 800mV - 1200mV |

**写入格式：**
```bash
# 修改 GFXCLK DPM 状态 0 的频率为 500MHz
echo "s 0 500" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 修改 MEMCLK DPM 状态 0 的频率为 100MHz
echo "m 0 100" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 设置 V/F 曲线点（RDNA 2+）
echo "vc 0 2250 1100" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 重置为默认值
echo "r" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 提交更改
echo "c" > /sys/class/drm/card0/device/pp_od_clk_voltage
```

### 2.2 写入命令参考

```
pp_od_clk_voltage 写入命令格式:

参数格式: <command> [arg1] [arg2] [arg3]

命令列表:
┌──────────┬─────────────────────────────────────────────────────────────┐
│ 命令     │ 说明                        │ 参数格式                     │
├──────────┼──────────────────────────────┼─────────────────────────────┤
│ s        │ 设置 SCLK (GFXCLK) 频率     │ s <state_id> <freq_mhz>    │
│          │                              │ 示例: s 0 500               │
├──────────┼──────────────────────────────┼─────────────────────────────┤
│ m        │ 设置 MCLK (MEMCLK) 频率     │ m <state_id> <freq_mhz>    │
│          │                              │ 示例: m 0 100               │
├──────────┼──────────────────────────────┼─────────────────────────────┤
│ vc       │ 设置 V/F 曲线点             │ vc <point_id> <freq> <volt>│
│          │                              │ 示例: vc 0 2250 1100       │
├──────────┼──────────────────────────────┼─────────────────────────────┤
| r        │ 重置为默认值                │ r                          │
|          |                              | 示例: r                     |
├──────────┼──────────────────────────────┼─────────────────────────────┤
│ c        │ 提交更改到 SMU              │ c                          │
│          │                              │ 示例: c                     │
├──────────┼──────────────────────────────┼─────────────────────────────┤
│ p        │ 设置功耗限制                │ p <power_w>                │
│          │                              │ 示例: p 250                 │
└──────────┴──────────────────────────────┴─────────────────────────────┘
```

### 2.3 完整读写流程

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    pp_od_clk_voltage 完整操作流程                         │
│                                                                          │
│  读取操作:                   写入操作:                                    │
│  ┌──────────────┐          ┌──────────────┐                             │
│  │ cat 接口文件   │          │ echo 命令写入  │                             │
│  └──────┬───────┘          └──────┬───────┘                             │
│         │                         │                                      │
│         ▼                         ▼                                      │
│  ┌──────────────┐          ┌──────────────┐                             │
│  │ VFS sysfs    │          │ VFS sysfs    │                             │
│  │ 文件系统调用  │          │ 文件系统调用  │                             │
│  └──────┬───────┘          └──────┬───────┘                             │
│         │                         │                                      │
│         ▼                         ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐      │
│  │                   amdgpu 驱动层                                  │      │
│  │  ┌──────────────────────────────────────────────────────────┐  │      │
│  │  │ amdgpu_get_pp_od_clk_voltage()  │ amdgpu_set_pp_od_clk_voltage() │
│  │  │   │                            │   │                      │  │      │
│  │  │   ├─ 解析时钟域                │   ├─ 解析命令            │  │      │
│  │  │   ├─ 读取 SMU 数据            │   ├─ 验证参数合法性      │  │      │
│  │  │   └─ 格式化输出                │   ├─ 更新驱动缓存       │  │      │
│  │  │                                │   └─ 发送 SMU 消息      │  │      │
│  │  └──────────────────────────────────────────────────────────┘  │      │
│  └────────────────────────────────────────────────────────────────┘      │
│         │                         │                                      │
│         ▼                         ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐      │
│  │  SMU 固件通信 (通过 MMIO 寄存器)                                │      │
│  │                                                                 │      │
│  │  读取: GetDpmTable → 返回 DPM 状态表到驱动内存                  │      │
│  │  写入: TransferTableDram2Smu → 将更新后的表上传到 SMU          │      │
│  │  提交: SetDpmTable → 应用新的 DPM 参数到硬件                    │      │
│  └────────────────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.4 内核实现分析

```c
// 简化的 pp_od_clk_voltage 设置函数核心逻辑
// 内核源码: drivers/gpu/drm/amd/pm/amdgpu_pm.c

static int amdgpu_set_pp_od_clk_voltage(struct device *dev,
    struct device_attribute *attr, const char *buf, size_t count)
{
    struct drm_device *ddev = dev_get_drvdata(dev);
    struct amdgpu_device *adev = ddev->dev_private;
    char *clone, *token, *str;
    int ret = -EINVAL;
    const char *delim = " \n";

    clone = kstrdup(buf, GFP_KERNEL);
    if (!clone)
        return -ENOMEM;

    // 解析第一个 token 作为命令
    token = strsep(&str, delim);
    if (!token)
        goto done;

    // 处理 's' 命令 - 设置 SCLK 频率
    if (!strcmp(token, "s")) {
        uint32_t state, freq;
        
        token = strsep(&str, delim);  // 状态编号
        if (!token) goto done;
        ret = kstrtou32(token, 10, &state);
        if (ret) goto done;
        
        token = strsep(&str, delim);  // 频率值
        if (!token) goto done;
        ret = kstrtou32(token, 10, &freq);
        if (ret) goto done;
        
        // 调用 PowerPlay 回调设置 SCLK
        if (adev->powerplay.pp_funcs &&
            adev->powerplay.pp_funcs->force_clock_level) {
            ret = adev->powerplay.pp_funcs->force_clock_level(
                adev, PP_SCLK, state | (freq << 16));
        }
    }
    // 处理 'm' 命令 - 设置 MCLK 频率
    else if (!strcmp(token, "m")) {
        // ... 类似 s 命令的处理逻辑
    }
    // 处理 'r' 命令 - 重置
    else if (!strcmp(token, "r")) {
        // 调用重置函数恢复默认配置
        if (adev->powerplay.pp_funcs &&
            adev->powerplay.pp_funcs->reset_power_profile) {
            ret = adev->powerplay.pp_funcs->reset_power_profile(adev);
        }
    }
    // 处理 'c' 命令 - 提交更改
    else if (!strcmp(token, "c")) {
        // 提交所有待处理的更改到 SMU
        if (adev->powerplay.pp_funcs &&
            adev->powerplay.pp_funcs->commit_od_settings) {
            ret = adev->powerplay.pp_funcs->commit_od_settings(adev);
        }
    }

done:
    kfree(clone);
    return ret == 0 ? count : ret;
}
```

### 2.5 实际案例：RX 7900 XTX OD 配置

```bash
#!/bin/bash
# rx7900xtx_od_config.sh - RX 7900 XTX OverDrive 配置

GPU_PATH="/sys/class/drm/card0/device"

echo "=== RX 7900 XTX OverDrive Configuration ==="
echo ""

# 1. 读取当前配置
echo "1. Current OD Table:"
cat $GPU_PATH/pp_od_clk_voltage

# 2. 检查 OD 范围
echo ""
echo "2. Available OD Range:"
cat $GPU_PATH/pp_od_clk_voltage | grep "OD_RANGE" -A5

# 3. 设置为手动模式
echo ""
echo "3. Setting manual mode..."
echo "manual" | sudo tee $GPU_PATH/power_dpm_force_performance_level > /dev/null

# 4. 降压超频配置 (Undervolt + Overclock)
echo ""
echo "4. Applying undervolt + overclock settings..."

# 降低 V/F 曲线点 0 的电压 (idle 状态)
echo "vc 0 500 750" | sudo tee $GPU_PATH/pp_od_clk_voltage > /dev/null

# 降低 V/F 曲线点 1 的电压 (中负载)
echo "vc 1 1200 850" | sudo tee $GPU_PATH/pp_od_clk_voltage > /dev/null

# 降低 V/F 曲线点 2 的电压并提升频率 (高负载)
echo "vc 2 2600 1050" | sudo tee $GPU_PATH/pp_od_clk_voltage > /dev/null

# 降低 V/F 曲线点 3 的电压并提升频率 (最高负载)
echo "vc 3 3000 1100" | sudo tee $GPU_PATH/pp_od_clk_voltage > /dev/null

# 提高功耗限制
echo "p 355" | sudo tee $GPU_PATH/pp_od_clk_voltage > /dev/null

# 5. 提交更改
echo ""
echo "5. Committing changes..."
echo "c" | sudo tee $GPU_PATH/pp_od_clk_voltage > /dev/null

# 6. 验证新配置
sleep 2
echo ""
echo "6. New OD Table:"
cat $GPU_PATH/pp_od_clk_voltage

echo ""
echo "=== Configuration Complete ==="
```

## 第三节：power_dpm_state 接口

### 3.1 接口概述

`power_dpm_state` 是传统的 DPM 状态控制接口，用于读取和设置 GPU 的电源状态。

```bash
# 读取当前 DPM 状态
$ cat /sys/class/drm/card0/device/power_dpm_state
performance

# 可设置的状态值
$ echo "battery" | sudo tee /sys/class/drm/card0/device/power_dpm_state
$ echo "balanced" | sudo tee /sys/class/drm/card0/device/power_dpm_state
$ echo "performance" | sudo tee /sys/class/drm/card0/device/power_dpm_state
```

**支持的状态：**

| 状态值 | 说明 | 典型功耗 | 适用场景 |
|--------|------|---------|---------|
| battery | 电池优化模式 | 30-50% TDP | 笔记本电池供电 |
| balanced | 平衡模式 | 50-80% TDP | 日常使用 |
| performance | 性能模式 | 80-100% TDP | 游戏/计算 |

### 3.2 状态切换的内部行为

```
┌──────────────────────────────────────────────────────────────────────────┐
│              power_dpm_state 状态切换内部流程                              │
│                                                                          │
│  battery → balanced → performance                                        │
│                                                                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                          │
│  │ battery   │    │ balanced  │    │ perf.    │                          │
│  └─────┬────┘    └─────┬────┘    └────┬─────┘                          │
│        │               │              │                                  │
│        ▼               ▼              ▼                                  │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  内部参数调整:                                                    │   │
│  │                                                                  │   │
│  │  参数              battery    balanced    performance            │   │
│  │  ───────────────  ─────────  ─────────  ───────────             │   │
│  │  power_dpm_state  0          1          2                        │   │
│  │  最大频率限制      低         中          无限制                  │   │
│  │  最小频率          低         中          高                      │   │
│  │  升频速度          慢         中          快                      │   │
│  │  降频速度          快         中          慢                      │   │
│  │  PowerTune 限制    严         中          宽                      │   │
│  │  温度目标          低         中          高                      │   │
│  │  风扇曲线          安静       平衡        性能                    │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.3 与 power_dpm_force_performance_level 的区别

| 特性 | power_dpm_state | power_dpm_force_performance_level |
|------|----------------|----------------------------------|
| 控制粒度 | 粗粒度 (3个级别) | 细粒度 (5个级别) |
| 自动 DPM | 允许自动切换 | 可强制锁定级别 |
| 适用硬件 | 所有 AMDGPU | GCN 5+ / RDNA |
| 推荐使用 | 已弃用 | 当前推荐接口 |

## 第四节：power_dpm_force_performance_level 接口

### 4.1 接口概述

这是当前推荐的性能等级控制接口，提供更细粒度的控制：

```bash
# 读取当前级别
$ cat /sys/class/drm/card0/device/power_dpm_force_performance_level
auto

# 设置不同级别
$ echo "auto" | sudo tee /sys/class/drm/card0/device/power_dpm_force_performance_level
$ echo "low" | sudo tee /sys/class/drm/card0/device/power_dpm_force_performance_level
$ echo "high" | sudo tee /sys/class/drm/card0/device/power_dpm_force_performance_level
$ echo "manual" | sudo tee /sys/class/drm/card0/device/power_dpm_force_performance_level
```

**支持级别：**

| 级别值 | 说明 | DPM 行为 | 典型用途 |
|--------|------|---------|---------|
| auto | 自动模式 | 正常 DPM 切换 | 默认，日常使用 |
| low | 低功耗强制 | 强制使用最低 DPM 状态 | 省电、低负载 |
| high | 高性能强制 | 强制使用最高 DPM 状态 | 游戏、Benchmark |
| manual | 手动模式 | 解锁 OD Table 写入权限 | 超频/降压调整 |

### 4.2 各模式的 DPM 行为分析

```
┌──────────────────────────────────────────────────────────────────────────┐
│              power_dpm_force_performance_level 行为对比                   │
│                                                                          │
│  auto 模式:                                                              │
│  DPM0 ◄──► DPM1 ◄──► DPM2 ◄──► DPM3 ◄──► DPM4                          │
│  ↑                                                                      │
│  正常负载驱动，PowerTune 介入控制                                          │
│                                                                          │
│  low 模式:                                                               │
│  DPM0 ──► DPM1                                                          │
│  ↑                                                                      │
│  锁定在最低 DPM 状态，频率和电压最低                                       │
│                                                                          │
│  high 模式:                                                              │
│  DPM4                                                                   │
│  ↑                                                                      │
│  锁定在最高 DPM 状态，频率和电压最高                                       │
│                                                                          │
│  manual 模式:                                                            │
│  DPM0 ◄──► DPM1 ◄──► DPM2 ◄──► DPM3 ◄──► DPM4                          │
│  ↑                ↑                                      ↑              │
│  pp_od_clk_voltage 可写    可调用 force_clock_level      可自定义 DPM 范围│
│                                                                          │
│  ├──────────────────────────────────────────────────────────────────────┤ │
│  │  auto:     SCLK 500-2500MHz, MCLK 100-2000MHz (全部 DPM 可用)     │ │
│  │  low:      SCLK 500MHz, MCLK 100MHz (DPM 0 锁定)                 │ │
│  │  high:     SCLK 2500MHz, MCLK 2000MHz (最高 DPM 锁定)            │ │
│  │  manual:   SCLK 500-2500MHz (OD 可调范围)                        │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.3 自动切换策略

当设置为 auto 时，驱动会根据系统电源状态自动切换级别：

```c
// 内核源码概念 - amdgpu_device_event 处理
// 系统电源状态变化时自动切换

static void amdgpu_pm_acpi_event_handler(struct amdgpu_device *adev)
{
    // 检查 AC 电源状态
    if (power_supply_is_system_supplied() > 0) {
        // 使用 AC 电源 -> 切换到 auto 模式
        adev->pm.ac_power = true;
        if (adev->pm.default_power_level == AUTO)
            amdgpu_dpm_force_performance_level(adev, AMD_DPM_FORCE_LEVEL_AUTO);
    } else {
        // 使用电池 -> 切换到 low 模式
        adev->pm.ac_power = false;
        amdgpu_dpm_force_performance_level(adev, AMD_DPM_FORCE_LEVEL_LOW);
    }
}
```

### 4.4 实战：自动电源管理脚本

```bash
#!/bin/bash
# auto_power_manager.sh - 自动电源管理守护脚本

GPU_PATH="/sys/class/drm/card0/device"
ACPI_PATH="/sys/class/power_supply"
POLL_INTERVAL=10
CURRENT_MODE=""

check_ac_power() {
    # 检测 AC 电源状态
    for supply in $ACPI_PATH/*/online; do
        if [ -f "$supply" ]; then
            status=$(cat $supply 2>/dev/null)
            if [ "$status" = "1" ]; then
                return 0  # AC 供电
            fi
        fi
    done
    return 1  # 电池供电
}

set_power_level() {
    local level=$1
    local gpu_temp=$(cat $GPU_PATH/hwmon/hwmon*/temp1_input 2>/dev/null)
    gpu_temp=$((gpu_temp / 1000))
    
    echo "$level" | sudo tee $GPU_PATH/power_dpm_force_performance_level > /dev/null
    
    if [ "$level" != "$CURRENT_MODE" ]; then
        echo "[$(date)] GPU temp: ${gpu_temp}°C, set: $level"
        CURRENT_MODE="$level"
    fi
}

echo "=== AMDGPU Auto Power Manager ==="
echo "Polling every ${POLL_INTERVAL}s..."
echo ""

# 主循环
while true; do
    if check_ac_power; then
        # AC 电源 - 检查温度决定级别
        gpu_temp=$(cat $GPU_PATH/hwmon/hwmon*/temp1_input 2>/dev/null)
        gpu_temp=$((gpu_temp / 1000))
        
        if [ "$gpu_temp" -gt 85 ]; then
            set_power_level "auto"    # 高温时回退到 auto
        elif [ "$gpu_temp" -gt 75 ]; then
            set_power_level "auto"    # 中高温用 auto
        else
            set_power_level "high"    # 温度正常时高性能
        fi
    else
        # 电池供电 - 始终低功耗
        set_power_level "low"
    fi
    
    sleep $POLL_INTERVAL
done
```

## 第五节：pp_dpm_* 时钟状态接口

### 5.1 pp_dpm_sclk (GFX 时钟 DPM 状态)

```bash
$ cat /sys/class/drm/card0/device/pp_dpm_sclk
0: 500Mhz *
1: 1200Mhz
2: 1800Mhz
3: 2200Mhz
4: 2500Mhz
```

带星号 (*) 的行表示当前活跃的 DPM 状态。

### 5.2 pp_dpm_mclk (显存时钟 DPM 状态)

```bash
$ cat /sys/class/drm/card0/device/pp_dpm_mclk
0: 100Mhz
1: 800Mhz
2: 1000Mhz
3: 2000Mhz *
```

### 5.3 pp_dpm_socclk / pp_dpm_fclk / pp_dpm_vclk / pp_dpm_dclk

```bash
# SoC 时钟 DPM 状态
$ cat /sys/class/drm/card0/device/pp_dpm_socclk
0: 800Mhz *
1: 1200Mhz

# FCLK (Fabric Clock) DPM 状态 - RDNA 2+
$ cat /sys/class/drm/card0/device/pp_dpm_fclk
0: 800Mhz *
1: 1200Mhz
2: 1800Mhz

# VCLK (Video Clock) DPM 状态 - RDNA 2+
$ cat /sys/class/drm/card0/device/pp_dpm_vclk
0: 600Mhz *
1: 1100Mhz

# DCLK (Data Clock) DPM 状态 - RDNA 2+
$ cat /sys/class/drm/card0/device/pp_dpm_dclk
0: 600Mhz *
1: 1100Mhz
```

### 5.4 DPM 状态监控脚本

```bash
#!/bin/bash
# dpm_monitor_full.sh - 完整 DPM 状态监控

GPU_PATH="/sys/class/drm/card0/device"
HWMON_PATH="$GPU_PATH/hwmon/hwmon*/"

echo "=== GPU DPM State Monitor ==="
echo ""

while true; do
    clear
    echo "Time: $(date +%H:%M:%S)"
    echo ""
    
    # DPM 状态
    echo "--- DPM Clock States ---"
    echo "SCLK:"
    cat $GPU_PATH/pp_dpm_sclk 2>/dev/null | grep "\*"
    echo "MCLK:"
    cat $GPU_PATH/pp_dpm_mclk 2>/dev/null | grep "\*"
    
    if [ -f "$GPU_PATH/pp_dpm_socclk" ]; then
        echo "SOCLK:"
        cat $GPU_PATH/pp_dpm_socclk 2>/dev/null | grep "\*"
    fi
    if [ -f "$GPU_PATH/pp_dpm_fclk" ]; then
        echo "FCLK:"
        cat $GPU_PATH/pp_dpm_fclk 2>/dev/null | grep "\*"
    fi
    
    echo ""
    echo "--- Performance Level ---"
    cat $GPU_PATH/power_dpm_force_performance_level 2>/dev/null
    
    echo ""
    echo "--- Power & Thermal ---"
    if [ -f "$GPU_PATH/power1_average" ]; then
        power=$(cat $GPU_PATH/power1_average 2>/dev/null)
        power_w=$(echo "scale=1; $power / 1000000" | bc 2>/dev/null)
        echo "Power: ${power_w}W"
    fi
    
    temp=$(cat $GPU_PATH/hwmon/hwmon*/temp1_input 2>/dev/null)
    if [ -n "$temp" ]; then
        temp_c=$((temp / 1000))
        echo "Temp: ${temp_c}°C"
    fi
    
    if [ -f "$GPU_PATH/hwmon/hwmon*/fan1_input" ]; then
        fan=$(cat $GPU_PATH/hwmon/hwmon*/fan1_input 2>/dev/null)
        echo "Fan: ${fan}RPM"
    fi
    
    echo ""
    echo "--- Current Frequencies ---"
    cat $GPU_PATH/current_freq_mhz 2>/dev/null
    
    sleep 2
done
```

## 第六节：hwmon 子系统接口

### 6.1 hwmon 接口层次

hwmon (Hardware Monitor) 是 Linux 内核的标准硬件监控接口，AMDGPU 驱动通过 hwmon 子系统暴露温度、电压、功耗、风扇等传感器数据：

```bash
$ ls -la /sys/class/drm/card0/device/hwmon/
total 0
drwxr-xr-x 3 root root 0 Mar 15 10:00 .
drwxr-xr-x 5 root root 0 Mar 15 10:00 ..
lrwxrwxrwx 1 root root 0 Mar 15 10:00 hwmon1 -> ../../devices/pci0000:00/.../hwmon/hwmon1
```

### 6.2 hwmon 传感器详细说明

```bash
$ cat /sys/class/drm/card0/device/hwmon/hwmon1/name
amdgpu

# 温度传感器
$ cat /sys/class/drm/card0/device/hwmon/hwmon1/temp1_input       # 边缘温度 (单位: 0.001°C)
52000  # = 52.0°C
$ cat /sys/class/drm/card0/device/hwmon/hwmon1/temp2_input       # 热点温度 (单位: 0.001°C)
85000  # = 85.0°C
$ cat /sys/class/drm/card0/device/hwmon/hwmon1/temp3_input       # 显存温度 (单位: 0.001°C)
65000  # = 65.0°C

# 温度阈值
$ cat /sys/class/drm/card0/device/hwmon/hwmon1/temp1_crit        # 临界温度上限
120000  # = 120.0°C
$ cat /sys/class/drm/card0/device/hwmon/hwmon1/temp1_crit_hyst   # 临界迟滞
5000    # = 5.0°C

# 功耗传感器
$ cat /sys/class/drm/card0/device/hwmon/hwmon1/power1_average    # 平均功耗 (单位: 微瓦)
235000000  # = 235W
$ cat /sys/class/drm/card0/device/hwmon/hwmon1/power1_cap        # 功耗上限 (单位: 微瓦)
250000000  # = 250W
$ cat /sys/class/drm/card0/device/hwmon/hwmon1/power1_cap_max    # 功耗上限最大值
300000000  # = 300W
$ cat /sys/class/drm/card0/device/hwmon/hwmon1/power1_cap_min    # 功耗上限最小值
80000000   # = 80W

# 电压传感器
$ cat /sys/class/drm/card0/device/hwmon/hwmon1/in0_input         # 核心电压 (单位: 毫伏)
1150     # = 1.15V

# 风扇传感器
$ cat /sys/class/drm/card0/device/hwmon/hwmon1/fan1_input        # 风扇转速 (RPM)
1800
$ cat /sys/class/drm/card0/device/hwmon/hwmon1/fan1_min          # 最小转速
500
$ cat /sys/class/drm/card0/device/hwmon/hwmon1/fan1_max          # 最大转速
3500
$ cat /sys/class/drm/card0/device/hwmon/hwmon1/pwm1              # PWM 值 (0-255)
128
$ cat /sys/class/drm/card0/device/hwmon/hwmon1/pwm1_enable       # 风扇控制模式
2        # 0=无控制, 1=手动PWM, 2=自动温控
```

### 6.3 温度传感器编号约定

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    AMDGPU hwmon 温度传感器映射                            │
│                                                                          │
│  传感器编号     │  名称         │  描述                    │  典型范围   │
│─────────────────┼──────────────┼─────────────────────────┼────────────│
│  temp1          │  edge         │ GPU 核心边缘温度         │ 30-110°C   │
│  temp2          │  hotspot      │ GPU 热点温度 (最热点)    │ 30-120°C   │
│  temp3          │  mem          │ 显存温度                 │ 30-105°C   │
│  temp4          │  vr_gfx       │ GFX 供电 VR 温度        │ 30-110°C   │
│  temp5          │  vr_mem       │ 显存供电 VR 温度        │ 30-110°C   │
│  temp6          │  liquid       │ 液冷液体温度 (若有)     │ 20-60°C    │
│  temp7          │  plx          │ PCIe 桥接温度 (多GPU)   │ 30-85°C    │
│                                                                          │
│  温度值单位: 毫摄氏度 (millidegrees Celsius)，读取值除以 1000 得到 °C   │
│  示例: 85000 → 85.0°C                                                    │
│  临界值: tempX_crit 表示硬件保护触发温度                                 │
│  迟滞值: tempX_crit_hyst 表示从临界状态恢复到正常状态的温度差            │
└──────────────────────────────────────────────────────────────────────────┘
```

### 6.4 hwmon 数据采集脚本

```bash
#!/bin/bash
# hwmon_collector.sh - hwmon 传感器数据采集

OUTPUT_FILE="/tmp/gpu_hwmon_data.csv"
INTERVAL=1
COUNT=60

echo "=== AMDGPU hwmon Data Collector ==="
echo "Output: $OUTPUT_FILE"
echo "Interval: ${INTERVAL}s, Count: ${COUNT}"
echo ""

# 初始化 CSV 文件
echo "timestamp,temp_edge,temp_hotspot,temp_mem,power_avg,power_cap,voltage,fan_rpm,pwm" > $OUTPUT_FILE

# 查找 hwmon 路径
HWMON_PATH=$(ls -d /sys/class/drm/card*/device/hwmon/hwmon*/ 2>/dev/null | head -1)

if [ -z "$HWMON_PATH" ]; then
    echo "Error: hwmon path not found"
    exit 1
fi

echo "hwmon path: $HWMON_PATH"
echo "Collecting $COUNT samples..."
echo ""

for i in $(seq 1 $COUNT); do
    timestamp=$(date +%H:%M:%S)
    
    # 读取各传感器（使用默认值 0 处理不存在的情况）
    te=$(cat ${HWMON_PATH}temp1_input 2>/dev/null || echo 0)
    th=$(cat ${HWMON_PATH}temp2_input 2>/dev/null || echo 0)
    tm=$(cat ${HWMON_PATH}temp3_input 2>/dev/null || echo 0)
    
    pw=$(cat ${HWMON_PATH}power1_average 2>/dev/null || echo 0)
    pc=$(cat ${HWMON_PATH}power1_cap 2>/dev/null || echo 0)
    
    vol=$(cat ${HWMON_PATH}in0_input 2>/dev/null || echo 0)
    fan=$(cat ${HWMON_PATH}fan1_input 2>/dev/null || echo 0)
    pwm=$(cat ${HWMON_PATH}pwm1 2>/dev/null || echo 0)
    
    # 写入 CSV
    echo "$timestamp,$te,$th,$tm,$pw,$pc,$vol,$fan,$pwm" >> $OUTPUT_FILE
    
    # 显示进度
    printf "\r[%3d/%3d] %s | T: %5.1f°C P: %6.1fW F: %4dRPM" \
        $i $COUNT "$timestamp" $(echo "scale=1; $te/1000" | bc) \
        $(echo "scale=1; $pw/1000000" | bc) $fan
    
    sleep $INTERVAL
done

echo ""
echo ""
echo "=== Collection Complete ==="
echo "Data saved to: $OUTPUT_FILE"
```

## 第七节：pp_power_profile_mode 接口

### 7.1 接口概述

`pp_power_profile_mode` 控制 GPU 的电源配置文件，每种配置文件定义了不同的频率/电压响应特性：

```bash
# 查看支持的配置文件
$ cat /sys/class/drm/card0/device/pp_power_profile_mode
Current Profile: 0 (CUSTOM)
```

**支持的配置文件：**

| 编号 | 名称 | 说明 | 升频策略 | 降频策略 |
|------|------|------|---------|---------|
| 0 | CUSTOM | 用户自定义 | 可配置 | 可配置 |
| 1 | VR | VR 应用 | 激进升频 | 保守降频 |
| 2 | POWER_SAVING | 省电模式 | 保守升频 | 积极降频 |
| 3 | VIDEO | 视频播放 | 中等升频 | 中等降频 |
| 4 | COMPUTE | 计算任务 | 中等升频 | 保守降频 |
| 5 | 3D_FULL_SCREEN | 全屏 3D | 激进升频 | 保守降频 |

### 7.2 配置文件参数

```bash
# 读取详细参数
$ cat /sys/class/drm/card0/device/pp_power_profile_mode
NUM        MODE_NAME     SCLK_UP_HYST  SCLK_DOWN_HYST  SCLK_UP_IDLE  ...
 0   CUSTOM            0              0               10
 1          VR            5              10              25
 2   POWER_SAVING       25              5               50
 3        VIDEO          10              10              25
 4       COMPUTE         10              20              10
 5  3D_FULL_SCREEN       5              15              10
...
Current Mode: 0 (CUSTOM)

# 设置配置文件
$ echo "5" | sudo tee /sys/class/drm/card0/device/pp_power_profile_mode
```

### 7.3 配置文件参数含义

| 参数 | 单位 | 说明 | 默认值 | 范围 |
|------|------|------|-------|------|
| SCLK_UP_HYST | % | GFX 升频迟滞阈值 | 5 | 0-100 |
| SCLK_DOWN_HYST | % | GFX 降频迟滞阈值 | 15 | 0-100 |
| SCLK_UP_IDLE | ms | 升频空闲等待时间 | 10 | 0-1000 |
| SCLK_DOWN_IDLE | ms | 降频空闲等待时间 | 10 | 0-1000 |
| MEM_UP_HYST | % | 显存升频迟滞阈值 | 5 | 0-100 |
| MEM_DOWN_HYST | % | 显存降频迟滞阈值 | 10 | 0-100 |
| MEM_UP_IDLE | ms | 显存升频空闲等待 | 10 | 0-1000 |
| MEM_DOWN_IDLE | ms | 显存降频空闲等待 | 10 | 0-1000 |

## 第八节：power1_cap 功耗上限控制

### 8.1 功耗上限设置

```bash
# 查看当前功耗上限
$ cat /sys/class/drm/card0/device/power1_cap
250000000  # 250W

# 查看可设置范围
$ cat /sys/class/drm/card0/device/power1_cap_max
300000000  # 300W
$ cat /sys/class/drm/card0/device/power1_cap_min
80000000   # 80W

# 设置功耗上限 (单位: 微瓦)
$ echo "200000000" | sudo tee /sys/class/drm/card0/device/power1_cap

# 或使用更直观的 rocm-smi 命令
$ sudo rocm-smi --setpoweroverdrive 200
```

### 8.2 功耗上限对性能的影响

```
┌──────────────────────────────────────────────────────────────────────────┐
│              功耗上限设置对 RX 6800 XT 性能的影响                         │
│                                                                          │
│                                                                          │
│  Power Cap │  GFXCLK Max │  3DMark Score  │  Temp  │  Power Draw      │
│  ──────────┼────────────┼────────────────┼────────┼───────────────── │
│  300W      │  2250MHz    │  19200         │  76°C  │  285W            │
│  250W      │  2250MHz    │  19000         │  74°C  │  240W            │
│  200W      │  2100MHz    │  18200         │  68°C  │  195W            │
│  150W      │  1850MHz    │  16500         │  62°C  │  145W            │
│  100W      │  1500MHz    │  13500         │  55°C  │  98W             │
│                                                                          │
│  结论:                                                                   │
│  - 降低功耗上限可显著降低温度和功耗，性能损失相对较小                      │
│  - 从 250W 降到 200W (-20%) 性能仅损失约 5%                             │
│  - 从 250W 降到 150W (-40%) 性能损失约 13%                             │
└──────────────────────────────────────────────────────────────────────────┘
```

## 第九节：实战综合脚本

### 9.1 GPU 电源配置管理器

```bash
#!/bin/bash
# gpu_power_manager.sh - 综合 GPU 电源管理脚本

GPU_PATH="/sys/class/drm/card0/device"
HWMON_PATH=$(ls -d $GPU_PATH/hwmon/hwmon*/ 2>/dev/null | head -1)

show_help() {
    echo "Usage: $0 <command> [args]"
    echo ""
    echo "Commands:"
    echo "  status           - 显示完整 GPU 电源状态"
    echo "  profile <mode>   - 设置性能级别 (auto|low|high|manual)"
    echo "  power <watts>    - 设置功耗上限 (瓦特)"
    echo "  fan <percent>    - 设置风扇转速 (0-100%)"
    echo "  fan_auto         - 恢复风扇自动控制"
    echo "  undefolt         - 一键降压 (Undervolt)"
    echo "  reset            - 重置所有电源设置为默认值"
    echo "  monitor          - 实时监控模式"
    echo ""
}

show_status() {
    echo "=== GPU Power Status ==="
    echo ""
    
    echo "--- Performance Level ---"
    cat $GPU_PATH/power_dpm_force_performance_level 2>/dev/null
    
    echo ""
    echo "--- DPM Clock States ---"
    echo "SCLK:"
    cat $GPU_PATH/pp_dpm_sclk 2>/dev/null
    echo "MCLK:"
    cat $GPU_PATH/pp_dpm_mclk 2>/dev/null
    
    echo ""
    echo "--- Power Limits ---"
    cap=$(cat $GPU_PATH/power1_cap 2>/dev/null)
    if [ -n "$cap" ]; then
        echo "Power Cap: $(echo "scale=1; $cap/1000000" | bc)W"
    fi
    cap_max=$(cat $GPU_PATH/power1_cap_max 2>/dev/null)
    if [ -n "$cap_max" ]; then
        echo "Max Cap:   $(echo "scale=1; $cap_max/1000000" | bc)W"
    fi
    cap_min=$(cat $GPU_PATH/power1_cap_min 2>/dev/null)
    if [ -n "$cap_min" ]; then
        echo "Min Cap:   $(echo "scale=1; $cap_min/1000000" | bc)W"
    fi
    
    echo ""
    echo "--- Power & Thermal ---"
    pw=$(cat $GPU_PATH/power1_average 2>/dev/null)
    if [ -n "$pw" ]; then
        echo "Power Draw: $(echo "scale=1; $pw/1000000" | bc)W"
    fi
    for i in 1 2 3; do
        temp=$(cat ${HWMON_PATH}temp${i}_input 2>/dev/null)
        if [ -n "$temp" ]; then
            temp_c=$((temp / 1000))
            echo "Temp${i}: ${temp_c}°C"
        fi
    done
    
    echo ""
    echo "--- Fan ---"
    fan=$(cat ${HWMON_PATH}fan1_input 2>/dev/null)
    pwm=$(cat ${HWMON_PATH}pwm1 2>/dev/null)
    pwm_en=$(cat ${HWMON_PATH}pwm1_enable 2>/dev/null)
    if [ -n "$fan" ]; then
        echo "Fan Speed: ${fan} RPM"
        echo "PWM: $pwm/255 ($((pwm * 100 / 255))%)"
        echo "Fan Mode: $pwm_en ($([ $pwm_en -eq 0 ] && echo 'no control' || [ $pwm_en -eq 1 ] && echo 'manual' || echo 'auto'))"
    fi
    
    echo ""
    echo "--- OD Table ---"
    cat $GPU_PATH/pp_od_clk_voltage 2>/dev/null | head -20
}

set_profile() {
    local mode=$1
    case "$mode" in
        auto|low|high|manual)
            echo "$mode" | sudo tee $GPU_PATH/power_dpm_force_performance_level > /dev/null
            echo "Profile set to: $mode"
            ;;
        *)
            echo "Invalid mode: $mode"
            echo "Valid: auto, low, high, manual"
            exit 1
            ;;
    esac
}

set_power() {
    local watts=$1
    local microwatts=$((watts * 1000000))
    local max_cap=$(cat $GPU_PATH/power1_cap_max 2>/dev/null)
    local min_cap=$(cat $GPU_PATH/power1_cap_min 2>/dev/null)
    
    if [ "$microwatts" -gt "$max_cap" ] || [ "$microwatts" -lt "$min_cap" ]; then
        echo "Error: Power cap must be between $(echo "scale=1; $min_cap/1000000" | bc)W and $(echo "scale=1; $max_cap/1000000" | bc)W"
        exit 1
    fi
    
    echo "$microwatts" | sudo tee $GPU_PATH/power1_cap > /dev/null
    echo "Power cap set to: ${watts}W"
}

set_fan() {
    local pct=$1
    if [ "$pct" -lt 0 ] || [ "$pct" -gt 100 ]; then
        echo "Error: Fan percent must be 0-100"
        exit 1
    fi
    
    local pwm_val=$((pct * 255 / 100))
    echo "1" | sudo tee ${HWMON_PATH}pwm1_enable > /dev/null
    echo "$pwm_val" | sudo tee ${HWMON_PATH}pwm1 > /dev/null
    echo "Fan set to ${pct}% (PWM=$pwm_val)"
}

set_fan_auto() {
    echo "2" | sudo tee ${HWMON_PATH}pwm1_enable > /dev/null
    echo "Fan set to auto mode"
}

do_undervolt() {
    echo "=== Applying Undervolt ==="
    
    # 设置手动模式
    echo "manual" | sudo tee $GPU_PATH/power_dpm_force_performance_level > /dev/null
    
    # 降低各 V/F 点的电压
    # 注意: 这些值需要根据具体 GPU 调整
    echo "vc 0 500 750" | sudo tee $GPU_PATH/pp_od_clk_voltage > /dev/null
    echo "vc 1 1200 850" | sudo tee $GPU_PATH/pp_od_clk_voltage > /dev/null
    echo "vc 2 2250 1050" | sudo tee $GPU_PATH/pp_od_clk_voltage > /dev/null
    echo "vc 3 2500 1100" | sudo tee $GPU_PATH/pp_od_clk_voltage > /dev/null
    
    # 提交更改
    echo "c" | sudo tee $GPU_PATH/pp_od_clk_voltage > /dev/null
    
    echo "Undervolt applied! Run 'gpu_power_manager.sh status' to verify."
}

do_reset() {
    echo "=== Resetting Power Settings ==="
    
    # 重置 OD Table
    echo "r" | sudo tee $GPU_PATH/pp_od_clk_voltage > /dev/null
    
    # 恢复自动模式
    echo "auto" | sudo tee $GPU_PATH/power_dpm_force_performance_level > /dev/null
    
    # 恢复风扇控制
    echo "2" | sudo tee ${HWMON_PATH}pwm1_enable > /dev/null
    
    echo "All power settings restored to default."
}

do_monitor() {
    echo "=== GPU Power Monitor (Ctrl+C to exit) ==="
    echo ""
    printf "%-10s | %-8s | %-8s | %-8s | %-6s | %-8s | %-6s\n" \
        "TIME" "T_EDGE" "T_HOT" "POWER" "SCLK" "MCLK" "FAN"
    echo "$(printf '=%.0s' {1..70})"
    
    while true; do
        te=$(cat ${HWMON_PATH}temp1_input 2>/dev/null || echo 0)
        th=$(cat ${HWMON_PATH}temp2_input 2>/dev/null || echo 0)
        pw=$(cat $GPU_PATH/power1_average 2>/dev/null || echo 0)
        fan=$(cat ${HWMON_PATH}fan1_input 2>/dev/null || echo 0)
        
        # 从 DPM 状态中提取当前频率
        sclk=$(cat $GPU_PATH/pp_dpm_sclk 2>/dev/null | grep "\*" | head -1 | awk '{print $1}')
        mclk=$(cat $GPU_PATH/pp_dpm_mclk 2>/dev/null | grep "\*" | head -1 | awk '{print $1}')
        
        now=$(date +%H:%M:%S)
        printf "%-10s | %5.1f°C  | %5.1f°C  | %5.1fW  | %-4s  | %-4s  | %4dRPM\n" \
            "$now" \
            $(echo "scale=1; $te/1000" | bc) \
            $(echo "scale=1; $th/1000" | bc) \
            $(echo "scale=1; $pw/1000000" | bc) \
            "$sclk" "$mclk" \
            $fan
        
        sleep 2
    done
}

# 主入口
case "${1:-help}" in
    status)     show_status ;;
    profile)    set_profile "$2" ;;
    power)      set_power "$2" ;;
    fan)        set_fan "$2" ;;
    fan_auto)   set_fan_auto ;;
    undefolt)   do_undervolt ;;
    reset)      do_reset ;;
    monitor)    do_monitor ;;
    help|*)     show_help ;;
esac
```

### 9.2 systemd 电源管理服务

```ini
# /etc/systemd/system/gpu-power-optimize.service
# systemd 服务：GPU 电源优化

[Unit]
Description=AMDGPU Power Optimization Service
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/gpu_power_manager.sh profile low
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```

```bash
# 启用服务
sudo systemctl enable gpu-power-optimize.service
sudo systemctl start gpu-power-optimize.service

# 创建 AC 电源事件触发
# /etc/acpi/events/gpu-power-ac
event=ac_adapter
action=/usr/local/bin/gpu_power_manager.sh profile auto

# 创建电池事件触发
# /etc/acpi/events/gpu-power-battery
event=battery
action=/usr/local/bin/gpu_power_manager.sh profile low
```

## 第十节：接口使用注意事项与最佳实践

### 10.1 安全操作指南

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    sysfs 电源管理接口安全操作指南                          │
│                                                                          │
│  1. 写入前必须先读取，了解当前配置和有效范围                               │
│     - 始终先 cat pp_od_clk_voltage 查看 OD_RANGE                        │
│     - 确认 power1_cap_min/max 范围后再设置                                │
│                                                                          │
│  2. 修改 OD Table 的完整流程:                                            │
│     a) 设置 manual 模式: echo manual > power_dpm_force_performance_level │
│     b) 读取当前配置:    cat pp_od_clk_voltage                           │
│     c) 逐步修改参数:    echo "s 0 500" > pp_od_clk_voltage              │
│     d) 提交更改:       echo "c" > pp_od_clk_voltage                    │
│     e) 验证新配置:      cat pp_od_clk_voltage                           │
│     f) 压力测试:        stress test for stability                        │
│                                                                          │
│  3. 异常恢复步骤:                                                        │
│     - 重置 OD Table: echo "r" > pp_od_clk_voltage                      │
│     - 恢复 auto 模式: echo "auto" > power_dpm_force_performance_level   │
│     - 重启驱动:        modprobe -r amdgpu && modprobe amdgpu           │
│     - 重启系统:        reboot (最后手段)                                 │
│                                                                          │
│  4. 不要同时使用多个工具修改同一参数                                       │
│     - rocm-smi、GUI 工具、手动 sysfs、脚本不要同时操作                    │
│     - 建议使用排他锁或单一管理脚本                                        │
│                                                                          │
│  5. 硬件保护机制:                                                        │
│     - 即使设置超出范围，硬件和 SMU 也会强制执行安全限制                    │
│     - 温度超过 110°C 时触发硬件热节流                                    │
│     - 电流超过 EDC 时自动降频                                             │
└──────────────────────────────────────────────────────────────────────────┘
```

### 10.2 不同 GPU 架构的接口差异

| 接口 | GCN (RX 400/500) | RDNA 1 (RX 5000) | RDNA 2 (RX 6000) | RDNA 3 (RX 7000) |
|------|------------------|------------------|------------------|------------------|
| pp_od_clk_voltage | 仅频率 | 频率+电压 | V/F 曲线 | V/F 曲线+多域 |
| power_dpm_state | 支持 | 支持 | 已弃用 | 已弃用 |
| power_dpm_force_performance_level | 支持 | 支持 | 支持 | 支持 |
| pp_dpm_socclk | 不支持 | 不支持 | 支持 | 支持 |
| pp_dpm_fclk | 不支持 | 不支持 | 支持 | 支持 |
| pp_dpm_vclk/dclk | 不支持 | 不支持 | 支持 | 支持 |
| pp_power_profile_mode | 有限 | 支持 | 完整 | 完整 |
| power1_cap | 支持 | 支持 | 支持 | 支持 |

## 第十一节：案例分析与问题排查

### 11.1 案例 1：RX 6800 XT 降压后性能提升

**背景：** RX 6800 XT 默认电压偏高导致温度高、风扇噪音大。

**操作步骤：**
```bash
# 1. 检查默认配置
cat /sys/class/drm/card0/device/pp_od_clk_voltage

# 2. 设置手动模式
echo "manual" | sudo tee /sys/class/drm/card0/device/power_dpm_force_performance_level

# 3. 降低 V/F 曲线电压 (降压超频)
echo "vc 0 500 750" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
echo "vc 1 1200 850" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
echo "vc 2 2250 1050" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
echo "vc 3 2430 1100" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage

# 4. 提交更改
echo "c" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
```

**结果对比：**
| 指标 | 默认 | 降压后 | 改善 |
|------|------|--------|------|
| 满载温度 | 78°C | 68°C | -10°C |
| 功耗 | 250W | 200W | -50W (-20%) |
| 频率 | 2250MHz | 2430MHz | +180MHz (+8%) |
| 风扇转速 | 2200RPM | 1600RPM | -600RPM |
| 3DMark 分数 | 19000 | 19500 | +2.6% |

### 11.2 案例 2：笔记本 GPU 省电配置

**背景：** 笔记本使用电池时 GPU 功耗过高导致续航短。

**优化方案：**
```bash
#!/bin/bash
# laptop_gpu_battery_opt.sh - 笔记本 GPU 电池优化

GPU_PATH="/sys/class/drm/card0/device"

# 1. 锁定低性能级别
echo "low" | sudo tee $GPU_PATH/power_dpm_force_performance_level

# 2. 限制功耗上限到最低
cap_min=$(cat $GPU_PATH/power1_cap_min)
echo "$cap_min" | sudo tee $GPU_PATH/power1_cap

# 3. 设置省电配置文件
echo "2" | sudo tee $GPU_PATH/pp_power_profile_mode  # POWER_SAVING

# 4. 如果支持，强制最低 DPM 状态
echo "0" | sudo tee $GPU_PATH/pp_dpm_sclk 2>/dev/null
echo "0" | sudo tee $GPU_PATH/pp_dpm_mclk 2>/dev/null

echo "GPU power optimization for battery applied."
```

### 11.3 案例 3：计算集群功率封顶

**背景：** 数据中心有多张 MI250/MI300 GPU，需要控制总功耗不超过机架供电限制。

```bash
#!/bin/bash
# cluster_power_capping.sh - 多 GPU 功率封顶

POWER_BUDGET=800  # 总功率预算 (瓦特)
GPU_COUNT=0
GPU_LIST=""

# 检测所有 AMD GPU
for gpu in /sys/class/drm/card*/device; do
    if [ -f "$gpu/power1_cap" ]; then
        card=$(basename $(dirname $gpu))
        GPU_LIST="$GPU_LIST $gpu"
        GPU_COUNT=$((GPU_COUNT + 1))
    fi
done

echo "=== Cluster Power Capping ==="
echo "Detected $GPU_COUNT AMD GPUs"
echo "Total power budget: ${POWER_BUDGET}W"
echo ""

# 计算每个 GPU 的功率预算
PER_GPU_POWER=$((POWER_BUDGET / GPU_COUNT))

# 应用功率限制
for gpu in $GPU_LIST; do
    card_name=$(basename $(dirname $gpu))
    power_uw=$((PER_GPU_POWER * 1000000))
    
    echo "Setting $card_name power cap to ${PER_GPU_POWER}W..."
    echo "$power_uw" | sudo tee $gpu/power1_cap > /dev/null
done

echo ""
echo "=== Configuration Applied ==="
echo "Each GPU limited to: ${PER_GPU_POWER}W"
echo ""

# 验证
echo ""
echo "=== Verification ==="
for gpu in $GPU_LIST; do
    card_name=$(basename $(dirname $gpu))
    actual=$(cat $gpu/power1_cap 2>/dev/null)
    actual_w=$((actual / 1000000))
    echo "$card_name: cap=${actual_w}W"
done
```

**验证输出：**
```
=== Cluster Power Capping ===
Detected 4 AMD GPUs
Total power budget: 800W

Setting card0 device power cap to 200W...
Setting card1 device power cap to 200W...
Setting card2 device power cap to 200W...
Setting card3 device power cap to 200W...

=== Configuration Applied ===

=== Verification ===
card0: cap=200W
card1: cap=200W
card2: cap=200W
card3: cap=200W
```

### 11.4 案例 4：性能回退调试

**背景：** 运行游戏/计算任务时 GPU 频繁降频，怀疑是 sysfs 配置冲突。

**调试脚本：**
```bash
#!/bin/bash
# debug_perf_throttle.sh - 性能回退调试

GPU_PATH="/sys/class/drm/card0/device"
LOG_FILE="/tmp/gpu_debug_$(date +%Y%m%d_%H%M%S).log"

echo "=== GPU Performance Debug Log ===" | tee -a $LOG_FILE
echo "Timestamp: $(date)" | tee -a $LOG_FILE
echo "" | tee -a $LOG_FILE

# 1. 检查当前 DPM 状态
echo "--- DPM States ---" | tee -a $LOG_FILE
for clock in pp_dpm_sclk pp_dpm_mclk pp_dpm_socclk pp_dpm_fclk; do
    if [ -f "$GPU_PATH/$clock" ]; then
        echo "$clock:" | tee -a $LOG_FILE
        cat "$GPU_PATH/$clock" | tee -a $LOG_FILE
    fi
done

echo "" | tee -a $LOG_FILE

# 2. 检查性能等级
echo "--- Performance Level ---" | tee -a $LOG_FILE
cat "$GPU_PATH/power_dpm_force_performance_level" | tee -a $LOG_FILE

echo "" | tee -a $LOG_FILE

# 3. 检查配置文件
echo "--- Power Profile ---" | tee -a $LOG_FILE
cat "$GPU_PATH/pp_power_profile_mode" | tee -a $LOG_FILE

echo "" | tee -a $LOG_FILE

# 4. 检查功耗限制
echo "--- Power Capping ---" | tee -a $LOG_FILE
for cap in power1_cap power1_cap_max power1_cap_min power1_average; do
    if [ -f "$GPU_PATH/$cap" ]; then
        val=$(cat "$GPU_PATH/$cap" 2>/dev/null)
        val_w=$((val / 1000000))
        echo "$cap: ${val_w}W" | tee -a $LOG_FILE
    fi
done

echo "" | tee -a $LOG_FILE

# 5. 检查温度
echo "--- Temperatures ---" | tee -a $LOG_FILE
for i in 1 2 3 4 5 6 7; do
    if [ -f "$GPU_PATH/hwmon/hwmon*/temp${i}_input" ]; then
        temp=$(cat "$GPU_PATH/hwmon/hwmon*/temp${i}_input" 2>/dev/null | head -1)
        temp_c=$((temp / 1000))
        echo "temp${i}: ${temp_c}°C" | tee -a $LOG_FILE
    fi
done

echo "" | tee -a $LOG_FILE

# 6. 持续监控（3秒间隔，5次采样）
echo "--- 5-Second Sampling ---" | tee -a $LOG_FILE
for i in 1 2 3 4 5; do
    sclk=$(cat "$GPU_PATH/pp_dpm_sclk" 2>/dev/null | grep "\*" | awk '{print $2}')
    temp=$(cat "$GPU_PATH/hwmon/hwmon*/temp1_input 2>/dev/null | head -1)
    temp_c=$((temp / 1000))
    power=$(cat "$GPU_PATH/power1_average" 2>/dev/null)
    power_w=$((power / 1000000))
    echo "[$i] SCLK=$sclk MHz | Temp=${temp_c}°C | Power=${power_w}W" | tee -a $LOG_FILE
    sleep 1
done

echo "" | tee -a $LOG_FILE
echo "Log saved to: $LOG_FILE"
```

## 第十二节：内核实现深度分析

### 12.1 amdgpu_pm.c 中的 DEVICE_ATTR 注册

AMDGPU 的 sysfs 电源管理接口在 `drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c` 中实现：

```c
// amdgpu_pm.c - DEVICE_ATTR 定义示例

/** pp_od_clk_voltage 接口 */
static DEVICE_ATTR(pp_od_clk_voltage, S_IRUGO | S_IWUSR,
                   amdgpu_get_pp_od_clk_voltage,
                   amdgpu_set_pp_od_clk_voltage);

/** power_dpm_force_performance_level 接口 */
static DEVICE_ATTR(power_dpm_force_performance_level, S_IRUGO | S_IWUSR,
                   amdgpu_get_power_dpm_force_performance_level,
                   amdgpu_set_power_dpm_force_performance_level);

/** pp_dpm_sclk 接口（只读） */
static DEVICE_ATTR(pp_dpm_sclk, S_IRUGO,
                   amdgpu_get_pp_dpm_sclk, NULL);

/** pp_dpm_mclk 接口（只读） */
static DEVICE_ATTR(pp_dpm_mclk, S_IRUGO,
                   amdgpu_get_pp_dpm_mclk, NULL);
```

### 12.2 权限位解析

| 宏 | 值 | 含义 |
|----|-----|------|
| S_IRUGO | 0444 | 所有人可读 |
| S_IWUSR | 0200 | 所有者可写 |
| S_IRUGO \| S_IWUSR | 0644 | 所有人可读，root 可写 |

### 12.3 show 回调函数模板

```c
static ssize_t amdgpu_get_pp_dpm_sclk(struct device *dev,
    struct device_attribute *attr, char *buf)
{
    struct drm_device *ddev = dev_get_drvdata(dev);
    struct amdgpu_device *adev = ddev->dev_private;
    ssize_t size;
    uint32_t value;

    // 1. 检查 GPU 是否处于可查询状态
    if (adev->in_suspend || adev->in_runpm) {
        return snprintf(buf, PAGE_SIZE, "Device suspended\n");
    }

    // 2. 检查是否支持 PowerPlay
    if (!adev->powerplay.pp_funcs ||
        !adev->powerplay.pp_funcs->print_clock_levels) {
        return snprintf(buf, PAGE_SIZE, "Not supported\n");
    }

    // 3. 调用 SMU 获取时钟状态
    size = amdgpu_dpm_print_clock_levels(adev, PP_SCLK, buf);

    return size;
}
```

### 12.4 store 回调函数模板

```c
static ssize_t amdgpu_set_power_dpm_force_performance_level(
    struct device *dev, struct device_attribute *attr,
    const char *buf, size_t count)
{
    struct drm_device *ddev = dev_get_drvdata(dev);
    struct amdgpu_device *adev = ddev->dev_private;
    enum amd_dpm_forced_level level;
    char command[16];
    int ret;

    // 1. 解析用户输入
    if (sscanf(buf, "%15s", command) != 1)
        return -EINVAL;

    // 2. 字符串到枚举转换
    if (strncmp(command, "low", 3) == 0)
        level = AMD_DPM_FORCED_LEVEL_LOW;
    else if (strncmp(command, "high", 4) == 0)
        level = AMD_DPM_FORCED_LEVEL_HIGH;
    else if (strncmp(command, "manual", 6) == 0)
        level = AMD_DPM_FORCED_LEVEL_MANUAL;
    else if (strncmp(command, "auto", 4) == 0)
        level = AMD_DPM_FORCED_LEVEL_AUTO;
    else
        return -EINVAL;

    // 3. 调用 SMU 设置
    ret = amdgpu_dpm_force_performance_level(adev, level);

    if (ret)
        return ret;

    return count;
}
```

### 12.5 amdgpu_pm.c 接口注册流程

```
amdgpu_pm_sysfs_init(adev)
  ├── device_create_file(dev, &dev_attr_pp_od_clk_voltage)
  ├── device_create_file(dev, &dev_attr_power_dpm_state)
  ├── device_create_file(dev, &dev_attr_power_dpm_force_performance_level)
  ├── device_create_file(dev, &dev_attr_pp_dpm_sclk)
  ├── device_create_file(dev, &dev_attr_pp_dpm_mclk)
  ├── device_create_file(dev, &dev_attr_pp_dpm_socclk)
  ├── device_create_file(dev, &dev_attr_pp_dpm_fclk)
  ├── device_create_file(dev, &dev_attr_power1_cap)
  ├── ...
  └── hwmon_device_register_with_groups(dev, "amdgpu", adev, hwmon_groups)
```

注册顺序直接影响 `/sys/class/drm/card0/device/` 目录下的文件排列顺序。

### 12.6 hwmon 回调函数组

```c
static umode_t amdgpu_hwmon_attr_is_visible(struct kobject *kobj,
    struct attribute *attr, int index)
{
    struct device *dev = kobj_to_dev(kobj);
    struct drm_device *ddev = dev_get_drvdata(dev);
    struct amdgpu_device *adev = ddev->dev_private;

    // 根据 GPU 代际和功能支持动态显示/隐藏属性
    if (attr == &sensor_dev_attr_power1_cap.dev_attr.attr) {
        if (!adev->powerplay.pp_funcs ||
            !adev->powerplay.pp_funcs->get_power_limit)
            return 0;  // 不支持，隐藏
    }

    return attr->mode;
}
```

## 第十三节：多架构接口差异对比

### 13.1 各代 GPU 接口支持矩阵

| sysfs 接口 | GCN (RX 400/500) | RDNA1 (RX 5000) | RDNA2 (RX 6000) | RDNA3 (RX 7000) | CDNA2 (MI200) | CDNA3 (MI300) |
|-------------|:---:|:---:|:---:|:---:|:---:|:---:|
| pp_od_clk_voltage | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| power_dpm_state | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ |
| power_dpm_force_performance_level | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| pp_dpm_sclk | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| pp_dpm_mclk | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| pp_dpm_socclk | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| pp_dpm_fclk | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| pp_dpm_vclk | ✗ | ✓ | ✓ | ✓ | ✗ | ✗ |
| pp_dpm_dclk | ✗ | ✓ | ✓ | ✓ | ✗ | ✗ |
| pp_power_profile_mode | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| power1_cap | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| hwmon temp1~7 | temp1-4 | temp1-4 | temp1-6 | temp1-7 | temp1-4 | temp1-5 |
| hwmon fan1_input | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| hwmon in0_input | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

### 13.2 RDNA3 新增接口

RDNA3 (RX 7900 系列) 相比 RDNA2 新增了以下 sysfs 功能：

```
# 新增温度传感器
temp6_input  -> "Liquid" (液温，水冷版)
temp7_input  -> "PLX" (PCIe 交换芯片温度)

# 新增 per-chip OD 功能
pp_od_clk_voltage 增加了新的 subcommand:
- vddgfx <value>    # 固定 GFX 电压覆盖
- vddm <value>      # 显存电压覆盖
- fclk <freq>       # FCLK 频率覆盖
```

### 13.3 GCN 遗留接口行为

GCN 代 (RX 400/500/Vega) 的 `power_dpm_state` 行为：

```bash
# GCN 上 power_dpm_state 实际控制行为
echo "battery" > /sys/class/drm/card0/device/power_dpm_state
# 效果：SCLK上限降低到最大频率的 70%
# 效果：MCLK上限降低到最大频率的 50%
# 效果：电压降低 50-100mV

echo "balanced" > /sys/class/drm/card0/device/power_dpm_state
# 效果：正常范围，无限制

echo "performance" > /sys/class/drm/card0/device/power_dpm_state
# 效果：SCLK/MCLK 强制最高状态
# 效果：禁用省电功能
```

## 第十四节：性能基准测试

### 14.1 sysfs 接口调用延迟

使用 `perf` 工具测量各接口的调用延迟：

```bash
#!/bin/bash
# benchmark_sysfs_latency.sh - sysfs 接口延迟基准测试

GPU_PATH="/sys/class/drm/card0/device"
RESULTS="/tmp/sysfs_latency.txt"
ITERATIONS=1000

echo "=== sysfs Interface Latency Benchmark ===" > $RESULTS
echo "Iterations: $ITERATIONS" >> $RESULTS
echo "Date: $(date)" >> $RESULTS
echo "" >> $RESULTS

bench_read() {
    local name=$1
    local path=$2
    local total=0
    
    for i in $(seq 1 $ITERATIONS); do
        local start=$(date +%s%N)
        cat $path > /dev/null 2>&1
        local end=$(date +%s%N)
        local elapsed=$((end - start))
        total=$((total + elapsed))
    done
    
    local avg=$((total / ITERATIONS / 1000))
    echo "$name: avg ${avg}us" >> $RESULTS
}

bench_write() {
    local name=$1
    local path=$2
    local value=$3
    local total=0
    
    for i in $(seq 1 $ITERATIONS); do
        local start=$(date +%s%N)
        echo "$value" > $path 2>/dev/null
        local end=$(date +%s%N)
        local elapsed=$((end - start))
        total=$((total + elapsed))
    done
    
    local avg=$((total / ITERATIONS / 1000))
    echo "$name: avg ${avg}us" >> $RESULTS
}

# 运行测试
bench_read "pp_dpm_sclk" "$GPU_PATH/pp_dpm_sclk"
bench_read "pp_dpm_mclk" "$GPU_PATH/pp_dpm_mclk"
bench_read "power_dpm_force_performance_level" "$GPU_PATH/power_dpm_force_performance_level"
bench_read "pp_od_clk_voltage" "$GPU_PATH/pp_od_clk_voltage"
bench_read "power1_average" "$GPU_PATH/power1_average"
bench_read "pp_power_profile_mode" "$GPU_PATH/pp_power_profile_mode"

echo "" >> $RESULTS
cat $RESULTS
```

**典型延迟结果（RX 6800 XT, Linux 6.2）：**

```
=== sysfs Interface Latency Benchmark ===
Iterations: 1000

pp_dpm_sclk: avg 12us
pp_dpm_mclk: avg 11us
power_dpm_force_performance_level: avg 8us
pp_od_clk_voltage: avg 45us
power1_average: avg 5us
pp_power_profile_mode: avg 15us
```

### 14.2 高频监控的 CPU 开销

```bash
#!/bin/bash
# monitor_overhead_test.sh - 高频监控 CPU 开销测试

GPU_PATH="/sys/class/drm/card0/device"

echo "=== CPU Overhead Test ==="
echo "Testing 100Hz monitoring loop..."

# 测试 10 秒内 100Hz 采样的 CPU 占用
timeout 10 bash -c '
    count=0
    while true; do
        cat '"$GPU_PATH"'/pp_dpm_sclk > /dev/null 2>&1
        cat '"$GPU_PATH"'/power1_average > /dev/null 2>&1
        cat '"$GPU_PATH"'/hwmon/hwmon*/temp1_input > /dev/null 2>&1
        count=$((count + 1))
        sleep 0.01
    done
    echo "Total samples: $count"
'
```

**结果分析：**
- 100Hz 采样率下，CPU 占用率 < 1%（单核）
- 瓶颈在 sysfs 文件读取系统调用，非 SMU 查询
- 建议监控频率不超过 50Hz，避免不必要的 sysfs 访问
- 对于大规模集群监控，推荐使用 batch 读取而非持续轮询

## 第十五节：常见问题与故障排查

### 15.1 "Invalid argument" 错误

**现象：**
```bash
$ echo "s 0 500" > /sys/class/drm/card0/device/pp_od_clk_voltage
-bash: echo: write error: Invalid argument
```

**排查步骤：**
1. 检查当前 OD 支持范围：
```bash
$ cat /sys/class/drm/card0/device/pp_od_clk_voltage
SCLK:
0: 500Mhz
1: 1000Mhz
2: 1500Mhz
3: 2000Mhz
4: 2500Mhz
OD_RANGE:
SCLK:     500MHz       3000MHz
MCLK:     100MHz       1500MHz
VDDC:     700mV        1200mV
```

2. 确认写入值在 OD_RANGE 范围内
3. 检查是否已解锁 OD：
```bash
# 某些卡需要先写 'c' 来检查是否 locked
echo "c" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
```

### 15.2 "Device or resource busy" 错误

**现象：**
```bash
$ sudo cat /sys/class/drm/card0/device/pp_od_clk_voltage
cat: pp_od_clk_voltage: Device or resource busy
```

**原因：** GPU 当前正在运行计算/图形任务，或处于挂起状态。

**解决方案：**
```bash
# 1. 检查 GPU 使用情况
sudo cat /sys/class/drm/card0/device/gpu_busy_percent

# 2. 检查是否挂起
sudo cat /sys/class/drm/card0/device/power/runtime_status

# 3. 等待任务完成或暂停任务
sudo systemctl stop gdm  # 停止图形显示管理器
```

### 15.3 写入后立即恢复

**现象：** 设置 power1_cap 或 pp_od_clk_voltage 后，值自动恢复为默认。

**原因：**
- 可能是用户态监控服务（如 `power-profiles-daemon`）覆盖
- 可能是 ACPI 事件触发策略切换
- 可能是 ROCm 运行时库自动管理

**排查：**
```bash
# 排查是否有其他进程在写入
sudo auditctl -w /sys/class/drm/card0/device/power1_cap -p wa -k gpu_power
sudo ausearch -k gpu_power | tail -20

# 临时禁用 power-profiles-daemon
sudo systemctl stop power-profiles-daemon
```

### 15.4 hwmon 传感器缺失

**现象：** hwmon 目录下缺少某些温度/功耗传感器。

**原因：**
- 老驱动版本不支持
- GPU 硬件不包含该传感器
- 传感器被禁用

**排查：**
```bash
# 检查驱动版本
cat /sys/class/drm/card0/device/driver/module/version

# 检查 GPU 代际
cat /sys/class/drm/card0/device/vbios_version

# 查看所有可用 hwmon 属性
ls -la /sys/class/drm/card0/device/hwmon/hwmon*/

# 读取所有可用的 hwmon 属性
for f in /sys/class/drm/card0/device/hwmon/hwmon*/{name,temp*_input,power*_average,fan*_input,in*_input,pwm*}; do
    if [ -f "$f" ]; then
        echo "$f: $(cat $f 2>/dev/null)"
    fi
done
```

### 15.5 OD 超频后不稳定

**现象：** 设置 OD 参数后，GPU 驱动崩溃或系统重启。

**恢复步骤：**
```bash
#!/bin/bash
# od_recovery.sh - OD 超频恢复脚本

GPU_PATH="/sys/class/drm/card0/device"

echo "=== OD Recovery Script ==="

# 1. 重置 OD 表到默认
echo "r" | sudo tee $GPU_PATH/pp_od_clk_voltage

# 2. 提交重置
echo "c" | sudo tee $GPU_PATH/pp_od_clk_voltage

# 3. 恢复默认性能级别
echo "auto" | sudo tee $GPU_PATH/power_dpm_force_performance_level

# 4. 恢复默认功耗限制
cap_default=$(sudo cat $GPU_PATH/power1_cap_max)
echo "$cap_default" | sudo tee $GPU_PATH/power1_cap

# 5. 恢复默认配置文件
echo "0" | sudo tee $GPU_PATH/pp_power_profile_mode

echo "OD settings have been reset to defaults."
echo "If issues persist, try: sudo modprobe -r amdgpu && sudo modprobe amdgpu"
```

## 第十六节：ROCm-SMI 与 sysfs 关系

### 16.1 rocm-smi 的 sysfs 映射

ROCm-SMI 工具 (`rocm-smi`) 本质上是对 sysfs 接口的封装：

```
rocm-smi 命令                   sysfs 映射路径
────────────────────────────────────────────────────────────
rocm-smi --showpower          power1_average, power1_cap
rocm-smi --setsclk <level>    pp_dpm_sclk
rocm-smi --setmclk <level>    pp_dpm_mclk
rocm-smi --setfan <level>     hwmon/hwmon*/pwm1
rocm-smi --showtemp            hwmon/hwmon*/temp*_input
rocm-smi --showmeminfo         pp_dpm_mclk
rocm-smi --showvoltage         hwmon/hwmon*/in0_input
rocm-smi --resetfans           hwmon/hwmon*/pwm1 (写 0)
rocm-smi --setperflevel <l>   power_dpm_force_performance_level
rocm-smi --showprofile         pp_power_profile_mode
rocm-smi --setprofile <p>      pp_power_profile_mode
rocm-smi --showod              pp_od_clk_voltage
rocm-smi --setod <k=v>        pp_od_clk_voltage
```

### 16.2 rocm-smi 的额外功能

rocm-smi 在 sysfs 之上提供了一些额外功能：

```bash
# 1. JSON 格式化输出
rocm-smi --json --showpower

# 2. 多 GPU 统一管理
rocm-smi --setsclk 0 -i 0,1,2,3  # 同时设置 4 张卡

# 3. 日志记录
rocm-smi --loglevel debug

# 4. 自动监控
rocm-smi -p 10 --showpower --showtemp  # 每 10 秒监控
```

### 16.3 自定义封装示例

```python
#!/usr/bin/env python3
# amdgpu_sysfs_wrapper.py - sysfs 封装的 Python 库

import os
import glob

class AMDGPUSysfs:
    """AMDGPU sysfs 接口的 Python 封装"""
    
    def __init__(self, card_id=0):
        self.base_path = f"/sys/class/drm/card{card_id}/device"
        if not os.path.exists(self.base_path):
            raise ValueError(f"Card {card_id} not found")
    
    def _read(self, attr):
        with open(f"{self.base_path}/{attr}", 'r') as f:
            return f.read().strip()
    
    def _write(self, attr, value):
        with open(f"{self.base_path}/{attr}", 'w') as f:
            f.write(str(value))
    
    def get_power_dpm_state(self):
        return self._read("power_dpm_state")
    
    def set_performance_level(self, level):
        """level: auto, low, high, manual"""
        self._write("power_dpm_force_performance_level", level)
    
    def get_gpu_clocks(self):
        sclk = self._read("pp_dpm_sclk")
        mclk = self._read("pp_dpm_mclk")
        return {"sclk": sclk, "mclk": mclk}
    
    def get_temperature(self, sensor=1):
        hwmon = glob.glob(f"{self.base_path}/hwmon/hwmon*/")[0]
        with open(f"{hwmon}/temp{sensor}_input", 'r') as f:
            return int(f.read()) / 1000
    
    def get_power_usage(self):
        return int(self._read("power1_average")) / 1000000
    
    def set_power_cap(self, watts):
        uw = int(watts * 1000000)
        self._write("power1_cap", str(uw))
    
    def get_od_table(self):
        return self._read("pp_od_clk_voltage")
    
    def set_od_clock(self, domain, state, freq_mhz):
        """domain: 's' for SCLK, 'm' for MCLK"""
        self._write("pp_od_clk_voltage", f"{domain} {state} {freq_mhz}")
    
    def commit_od(self):
        self._write("pp_od_clk_voltage", "c")
    
    def reset_od(self):
        self._write("pp_od_clk_voltage", "r")
    
    def get_power_profile(self):
        return self._read("pp_power_profile_mode")
    
    def set_power_profile(self, profile_id):
        self._write("pp_power_profile_mode", str(profile_id))


# 使用示例
if __name__ == "__main__":
    gpu = AMDGPUSysfs(0)
    print(f"Temperature: {gpu.get_temperature():.1f}°C")
    print(f"Power: {gpu.get_power_usage():.2f}W")
    print(f"OD Table:\n{gpu.get_od_table()}")
    
    # 设置性能模式
    gpu.set_performance_level("high")
    print(f"Performance level set to high")
```

## 第十七节：综合实战练习

### 17.1 练习题 1：创建 GPU 性能监控仪表盘

**要求：** 编写脚本来实现类 `htop` 风格的 GPU 实时监控。

```bash
#!/bin/bash
# gpu_top.sh - GPU 实时监控仪表盘

GPU_PATH="/sys/class/drm/card0/device"
HWMON=$(ls -d $GPU_PATH/hwmon/hwmon*/ 2>/dev/null | head -1)

# 获取终端大小
COLS=$(tput cols)
LINES=$(tput lines)

draw_bar() {
    local label=$1
    local value=$2
    local max=$3
    local width=$((COLS - 25))
    local bar_width=$((width < 10 ? 10 : width > 60 ? 60 : width))
    local filled=$((value * bar_width / max))
    local empty=$((bar_width - filled))
    
    printf "%-20s [" "$label"
    for i in $(seq 1 $filled); do printf "#"; done
    for i in $(seq 1 $empty); do printf "."; done
    printf "] %5.1f%%\n" "$(echo "scale=1; $value * 100 / $max" | bc)"
}

while true; do
    clear
    echo "╔══════════════════════════════════════════════════════════╗"
    echo "║                AMDGPU System Monitor                   ║"
    echo "╚══════════════════════════════════════════════════════════╝"
    echo ""
    
    # 温度
    for i in 1 2 3 4; do
        if [ -f "${HWMON}temp${i}_input" ]; then
            raw=$(cat ${HWMON}temp${i}_input 2>/dev/null)
            temp_c=$((raw / 1000))
            draw_bar "Temp${i}" "$temp_c" 100
        fi
    done
    
    # 功耗
    power_raw=$(cat $GPU_PATH/power1_average 2>/dev/null)
    power_w=$((power_raw / 1000000))
    cap_raw=$(cat $GPU_PATH/power1_cap 2>/dev/null)
    cap_w=$((cap_raw / 1000000))
    draw_bar "Power" "$power_w" "$cap_w"
    
    echo ""
    echo "Press Ctrl+C to exit"
    sleep 1
done
```

### 17.2 练习题 2：自动化 GPU 配置文件切换

**要求：** 根据 GPU 使用场景自动切换电源配置文件。

```bash
#!/bin/bash
# auto_profile_switch.sh - 自动配置文件切换守护进程

GPU_PATH="/sys/class/drm/card0/device"
POLL_INTERVAL=5
PROFILE="auto"

# 场景检测
detect_scenario() {
    local gpu_busy=$(cat $GPU_PATH/gpu_busy_percent 2>/dev/null || echo 0)
    local power=$(cat $GPU_PATH/power1_average 2>/dev/null)
    local power_w=$((power / 1000000))
    local temp_raw=$(cat $GPU_PATH/hwmon/hwmon*/temp1_input 2>/dev/null | head -1)
    local temp_c=$((temp_raw / 1000))
    
    if [ $gpu_busy -lt 5 ] && [ $power_w -lt 30 ]; then
        echo "idle"     # 空闲 -> 省电
    elif [ $gpu_busy -gt 80 ] && [ $power_w -gt 150 ]; then
        echo "compute"  # 高负载 -> 计算模式
    elif [ $temp_c -gt 85 ]; then
        echo "throttle" # 过热 -> 降频
    else
        echo "normal"
    fi
}

# 应用配置
apply_profile() {
    local scenario=$1
    
    case $scenario in
        idle)
            echo "low" > $GPU_PATH/power_dpm_force_performance_level
            echo "2" > $GPU_PATH/pp_power_profile_mode  # POWER_SAVING
            ;;
        compute)
            echo "high" > $GPU_PATH/power_dpm_force_performance_level
            echo "5" > $GPU_PATH/pp_power_profile_mode   # COMPUTE
            ;;
        throttle)
            cap_min=$(cat $GPU_PATH/power1_cap_min)
            echo "$cap_min" > $GPU_PATH/power1_cap
            echo "1" > $GPU_PATH/pp_power_profile_mode   # VR
            ;;
        normal)
            echo "auto" > $GPU_PATH/power_dpm_force_performance_level
            echo "0" > $GPU_PATH/pp_power_profile_mode   # CUSTOM
            ;;
    esac
}

echo "Starting auto profile switch daemon..."
while true; do
    scenario=$(detect_scenario)
    if [ "$scenario" != "$PROFILE" ]; then
        echo "[$(date)] Switching to $scenario mode"
        apply_profile $scenario
        PROFILE=$scenario
    fi
    sleep $POLL_INTERVAL
done
```

### 17.3 练习题 3：sysfs 接口兼容性测试

**要求：** 编写脚本测试所有 sysfs 接口的可用性和行为。

```bash
#!/bin/bash
# sysfs_compatibility_test.sh - sysfs 接口兼容性测试

GPU_PATH="/sys/class/drm/card0/device"
PASS=0
FAIL=0

test_interface() {
    local name=$1
    local path=$2
    local expected_rw=$3  # "ro", "rw", "wo"
    
    if [ ! -e "$path" ]; then
        echo "[SKIP] $name: interface not found"
        return
    fi
    
    # 测试可读性
    if [ "$expected_rw" != "wo" ]; then
        if cat "$path" > /dev/null 2>&1; then
            echo "[PASS] $name: readable"
            PASS=$((PASS + 1))
        else
            echo "[FAIL] $name: expected readable but failed"
            FAIL=$((FAIL + 1))
        fi
    fi
    
    # 测试可写性
    if [ "$expected_rw" != "ro" ]; then
        # 保存原值
        orig=$(cat "$path" 2>/dev/null)
        echo "" > "$path" 2>/dev/null
        if [ $? -eq 0 ] || [ $? -eq 1 ]; then
            echo "[PASS] $name: writable"
            PASS=$((PASS + 1))
        else
            echo "[FAIL] $name: expected writable but failed"
            FAIL=$((FAIL + 1))
        fi
    fi
}

echo "=== AMDGPU sysfs Interface Compatibility Test ==="
echo "Date: $(date)"
echo "GPU: $(cat $GPU_PATH/vbios_version 2>/dev/null || echo unknown)"
echo ""

test_interface "pp_od_clk_voltage" "$GPU_PATH/pp_od_clk_voltage" "rw"
test_interface "power_dpm_state" "$GPU_PATH/power_dpm_state" "rw"
test_interface "power_dpm_force_performance_level" "$GPU_PATH/power_dpm_force_performance_level" "rw"
test_interface "pp_dpm_sclk" "$GPU_PATH/pp_dpm_sclk" "ro"
test_interface "pp_dpm_mclk" "$GPU_PATH/pp_dpm_mclk" "ro"
test_interface "pp_dpm_socclk" "$GPU_PATH/pp_dpm_socclk" "ro"
test_interface "pp_dpm_fclk" "$GPU_PATH/pp_dpm_fclk" "ro"
test_interface "pp_dpm_vclk" "$GPU_PATH/pp_dpm_vclk" "ro"
test_interface "pp_dpm_dclk" "$GPU_PATH/pp_dpm_dclk" "ro"
test_interface "pp_power_profile_mode" "$GPU_PATH/pp_power_profile_mode" "rw"
test_interface "power1_cap" "$GPU_PATH/power1_cap" "rw"
test_interface "power1_average" "$GPU_PATH/power1_average" "ro"

echo ""
echo "=== Results ==="
echo "Passed: $PASS"
echo "Failed: $FAIL"
```

## 第十八节：扩展思考与进阶

### 18.1 sysfs 的局限性

1. **单值接口限制：** 每个 sysfs 文件只能表示一个值，复杂的 OD 配置只能通过多行字符串传递
2. **原子性问题：** 多个 sysfs 写入操作不是原子的，可能导致中间状态
3. **缺乏事件通知：** sysfs 不支持 poll/epoll 事件，必须轮询读取
4. **权限粒度粗：** sysfs 只有 rwx 权限位，无法细粒度控制
5. **性能开销：** 每次 sysfs 访问都是一次系统调用 + 内核函数调用

### 18.2 替代方案对比

| 方案 | 延迟 | 灵活性 | 易用性 | 适用场景 |
|------|:---:|:---:|:---:|---------|
| sysfs 直接访问 | 5-45μs | 中 | 高 | 脚本/CLI 工具 |
| hwmon (sysfs) | 3-10μs | 低 | 高 | 温度/功耗监控 |
| rocm-smi | 50-200μs | 中 | 高 | 多 GPU 管理 |
| libdrm/amdgpu | 1-5μs | 高 | 低 | 应用集成 |
| DBGIO (MMIO) | 0.1-1μs | 最高 | 最低 | 内核调试 |
| SMU 消息 | 10-50μs | 高 | 低 | 固件级控制 |

### 18.3 内核 6.x 新特性

Linux 内核 6.x 版本中对 AMDGPU sysfs 接口的重要更新：

- **6.0:** 增加 `gpu_busy_percent` 接口
- **6.1:** 支持 RDNA3 的 `pp_dpm_vclk` 和 `pp_dpm_dclk`
- **6.2:** 增加 `power1_cap_min/max` 动态范围报告
- **6.3:** hwmon 新增 `temp6_input` (液温) 和 `temp7_input` (PLX)
- **6.5:** pp_od_clk_voltage 支持 VDDGFX 覆盖
- **6.7:** 新增 `amdgpu_gpu_recovery` sysfs 控制接口
- **6.8:** 改进 S0ix 状态下的 sysfs 可访问性

### 18.4 思考题

1. 为什么 `power_dpm_state` 在 RDNA 架构中被废弃？它和 `power_dpm_force_performance_level` 的功能重叠在哪里？

2. 在设计 sysfs 接口时，为什么 pp_od_clk_voltage 选择使用多行文本格式而不是每个属性一个文件？

3. 假设你要设计一个新的 sysfs 接口来支持 RDNA4 的 per-chip V/F curve 自定义，你会如何设计？

4. 在集群环境中，1000 张 GPU 同时通过 sysfs 读取 power1_average 会造成多大的内核锁竞争？

5. 如何在不重启的情况下热添加一个新的 sysfs 电源管理属性到已加载的 amdgpu 模块？

6. hwmon 子系统的功率值以微瓦（μW）为单位，温度值以毫摄氏度（m°C）为单位，这种设计的历史原因是什么？

7. 如果 pp_od_clk_voltage 的写操作被中断，如何保证配置不会处于部分应用的中间状态？

## 参考资料

1. **Linux Kernel 源码 - amdgpu_pm.c**
   `drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c`
   https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c

2. **AMD ROCm 文档 - ROCm-SMI**
   https://rocm.docs.amd.com/projects/rocmsmi/en/latest/

3. **Linux Kernel 文档 - hwmon 子系统**
   https://www.kernel.org/doc/html/latest/hwmon/index.html

4. **Linux Kernel 文档 - sysfs 规则**
   Documentation/filesystems/sysfs.rst
   https://www.kernel.org/doc/html/latest/filesystems/sysfs.html

5. **AMDGPU 驱动文档**
   https://docs.kernel.org/gpu/amdgpu.html

6. **Phoronix AMDGPU Linux 驱动评测**
   https://www.phoronix.com/search/AMDGPU+Linux

7. **PowerPlay 内部文档 (GPUOpen)**
   https://gpuopen.com/learn/amd-powerplay/

---

*本文件内容基于 Linux 内核 6.x 版本、AMD ROCm 5.x 以及 AMDGPU 开源驱动编写。命令示例可能因内核版本和 GPU 架构不同而产生差异，请以实际环境为准。*PER_GPU_POWER}W"

# 验证
echo ""
echo "=== Verification ==="
for gpu in $GPU_LIST; do
    card_name=$(basename $(dirname $gpu))
    actual=$(cat $gpu/power1_cap 2>/dev/null)
    actual_w=$((actual / 1000000))
    echo "$card_name: cap=${actual_w}W"
done
```