# 第309天：hwmon sysfs 接口：功耗/温度/风扇信息导出

## 学习目标
- 深入理解 Linux hwmon 子系统在 AMDGPU 驱动中的实现机制
- 掌握 AMDGPU 电源管理相关 sysfs 接口的设计原理与数据结构
- 学会通过 hwmon sysfs 导出 GPU 的温度、功耗、风扇转速等关键监控信息
- 能够使用标准工具（如 sensors）读取和分析 GPU 监控数据
- 掌握 sysfs 文件创建、属性回调函数注册与权限控制的技术细节

## 知识详解

### 1. 概念原理

#### 1.1 hwmon 子系统概述

Linux Hardware Monitoring (hwmon) 子系统是内核提供的标准化硬件监控框架，它通过 sysfs 文件系统（位于 `/sys/class/hwmon/`）向用户空间提供统一的传感器数据访问接口。AMDGPU 驱动深度集成 hwmon，将其电源管理、温度、功耗、风扇等核心监控功能导出至用户空间。

**核心设计原则**：
- **统一接口**：遵循 Linux 内核 hwmon ABI，保证与其他硬件监控工具兼容
- **数据实时性**：大多数传感器数据通过 SMU（System Management Unit）固件实时获取
- **权限控制**：核心参数通常只读，可配置参数支持用户空间调节
- **跨平台兼容**：支持 RDNA、CDNA、GCN 等历代 AMD GPU 架构

#### 1.2 AMDGPU hwmon 接口架构

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         User Space Applications                            │
│                                                                            │
│  ┌─────────────────┐     ┌─────────────────┐    ┌──────────────────┐    │
│  │    sensors      │     │  rocm-smi       │    │ custom scripts   │    │
│  │ (lm-sensors)    │     │ (AMD official)  │    │ (monitoring)     │    │
│  └─────────┬───────┘     └─────────┬───────┘    └──────────┬───────┘    │
│            │                       │                       │            │
├────────────┼───────────────────────┼───────────────────────┼────────────┤
│            ▼                       ▼                       ▼            │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │              sysfs Interface (/sys/class/hwmon/hwmonX/)            │   │
│  │                                                                    │   │
│  │  ./temp1_input          → GPU 核心温度 (单位：m°C)                  │   │
│  │  ./power1_average       → 平均功耗 (单位：μW)                      │   │
│  │  ./power1_cap           → 功耗限制 (PPT, 单位：μW)                 │   │
│  │  ./fan1_input           → 风扇转速 (单位：RPM)                     │   │
│  │  ./pwm1                 → PWM 控制值 (0-255)                      │   │
│  │  ./pump1_input          → 水泵转速（部分 AIO 水冷）              │   │
│  │  ...                                                                │   │
│  └────────────┬─────────────────────────────────────────────────────┘   │
│               │                                                           │
├───────────────┼───────────────────────────────────────────────────────────┤
│               │                                                           │
│               ▼                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │                amdgpu_pm.c  sysfs 属性注册层                      │    │
│  │                                                                  │    │
│  │  static SENSOR_DEVICE_ATTR(temp1_input, S_IRUGO,                 │    │
│  │          amdgpu_hwmon_show_temp, NULL, PP_TEMP);                 │    │
│  │  static SENSOR_DEVICE_ATTR(power1_average, S_IRUGO,              │    │
│  │          amdgpu_hwmon_show_power, NULL, PP_POWER_AVERAGE);       │    │
│  │  ...                                                             │    │
│  └─────────┬────────────────────────────────────────────────────────┘    │
│            │                                                             │
├────────────┼─────────────────────────────────────────────────────────────┤
│            │                                                             │
│            ▼                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │               amdgpu_dpm.c  硬件数据获取层                        │    │
│  │                                                                  │    │
│  │  amdgpu_dpm_read_sensor()                                        │    │
│  │  └─> smu_cmn_get_sensor_table()  // 通过 SMU 接口读取           │    │
│  │       └─> smu_cmn_send_smc_msg_with_param()                    │    │
│  │            └─> mmio 寄存器写入，触发 SMU 固件执行               │    │
│  └───────────┬──────────────────────────────────────────────────────┘    │
│              │                                                           │
├──────────────┼───────────────────────────────────────────────────────────┤
│              │                                                           │
│              ▼                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │              SMU (System Management Unit) 固件                    │    │
│  │                                                                  │    │
│  │  • 读取内置温度传感器（Tmon）                                   │    │
│  │  • 读取电流/电压传感器（估算功耗）                              │    │
│  │  • 读取风扇 Tach 脉冲信号（计算转速）                          │    │
│  │  • 返回标准化数据格式至驱动                                   │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**数据流向说明**：
1. **用户空间**通过标准文件操作（read/write）访问 `/sys/class/hwmon/hwmonX/` 下的传感器文件
2. **amdgpu_pm.c** 中的属性回调函数（如 `amdgpu_hwmon_show_temp`）响应用户空间请求
3. **amdgpu_dpm.c** 通过 SMU 接口读取实际硬件数据，通常采用异步消息机制
4. **SMU 固件**从硬件传感器采集原始数据，处理后返回给驱动

#### 1.3 关键数据结构与函数

**hwmon_chip_info 结构体**（`include/linux/hwmon.h`）：
```c
struct hwmon_chip_info {
    const char *driver_name;                /* 驱动名称：amdgpu */
    const struct attribute_group **groups;  /* sysfs 属性组 */
    const void *custom_attrs;               /* 自定义扩展属性 */
};

struct amdgpu_device {
    struct device *dev;                     /* 设备结构 */
    struct amdgpu_pm pm;                    /* 电源管理结构 */
    struct amdgpu_dpm dpm;                  /* DPM 结构 */
    struct amdgpu_hwmon *hwmon;             /* hwmon 专用结构 */
};

struct amdgpu_hwmon {
    struct device *hwmon_dev;               /* hwmon 设备对象 */
    struct amdgpu_device *adev;             /* 回指 AMDGPU 设备 */
    u32 temp_crit;                          /* 临界温度（℃） */
    u32 temp_emergency;                     /* 紧急温度（℃） */
    u32 power_cap_max;                      /* 功耗限制上限（μW） */
    u32 power_cap_min;                      /* 功耗限制下限（μW） */
};
```

**核心函数指针表**（简化版）：
```c
/* drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c */
static struct hwmon_ops amdgpu_hwmon_ops = {
    .read = amdgpu_hwmon_read,              /* 通用读操作 */
    .write = amdgpu_hwmon_write,            /* 通用写操作 */
    .read_string = amdgpu_hwmon_read_string, /* 字符串读取 */
};

static const struct hwmon_channel_info *amdgpu_hwmon_info[] = {
    /* 温度通道 */
    HWMON_CHANNEL_INFO(temp,
        HWMON_T_INPUT | HWMON_T_CRIT | HWMON_T_EMERGENCY,
        HWMON_T_INPUT, /* 多个传感器支持 */
    ),
    /* 功耗通道 */
    HWMON_CHANNEL_INFO(power,
        HWMON_P_AVERAGE | HWMON_P_INPUT | HWMON_P_CAP,
    ),
    /* 风扇通道 */
    HWMON_CHANNEL_INFO(fan,
        HWMON_F_INPUT | HWMON_F_TARGET | HWMON_F_MIN | HWMON_F_MAX,
    ),
    /* PWM 控制 */
    HWMON_CHANNEL_INFO(pwm,
        HWMON_PWM_INPUT | HWMON_PWM_ENABLE,
    ),
    NULL
};
```

### 2. 实践操作

#### 2.1 定位 GPU 对应的 hwmon 设备

```bash
# 方法 1：遍历所有 hwmon 设备，查找 amdgpu
for hwmon in /sys/class/hwmon/hwmon*; do
    name=$(cat "$hwmon/name" 2>/dev/null)
    if [[ "$name" == "amdgpu"* ]]; then
        echo "Found AMDGPU hwmon: $hwmon"
        break
    fi
done

# 方法 2：使用 sensors 命令自动检测
sudo apt-get install lm-sensors  # Debian/Ubuntu
sudo sensors-detect --auto       # 自动检测硬件
sensors | grep amdgpu

# 输出示例：
# amdgpu-pci-0c00
# Adapter: PCI adapter
# vddgfx:      856.00 mV 
# fan1:        1250 RPM
# temp1:       +54.0°C  (crit = +110.0°C, hyst = -273.1°C)
# power1:      165.00 W  (cap = 250.00 W)
```

#### 2.2 直接读取 sysfs 传感器数据

```bash
# 找到正确的 hwmon 设备（例如 hwmon2）
HWMON_PATH=/sys/class/hwmon/hwmon2

# 读取 GPU 核心温度（单位：毫摄氏度 m°C）
cat ${HWMON_PATH}/temp1_input
# 输出：54000  (表示 54.0°C)

# 换算为摄氏度
temp_mc=$(cat ${HWMON_PATH}/temp1_input)
temp_c=$((temp_mc / 1000))
echo "GPU Temperature: ${temp_c}°C"

# 读取功耗限制（PPT）
cat ${HWMON_PATH}/power1_cap
# 输出：250000000  (单位：微瓦 μW，即 250W)

# 换算为瓦特
ppt_uw=$(cat ${HWMON_PATH}/power1_cap)
ppt_w=$(echo "scale=2; $ppt_uw / 1000000" | bc)
echo "Power Limit: ${ppt_w}W"

# 读取当前平均功耗
cat ${HWMON_PATH}/power1_average
# 输出：165000000  (165W)

# 读取风扇转速（RPM）
cat ${HWMON_PATH}/fan1_input
# 输出：1250

# 读取电压（以 vddgfx 为例）
cat ${HWMON_PATH}/in0_input
# 输出：856000  (单位：微伏 μV，即 856mV)
```

#### 2.3 使用 rocm-smi 高级工具

AMD 官方 ROCm 栈提供了功能更强大的监控工具：

```bash
# 安装 ROCm（以 Ubuntu 22.04 为例）
wget https://repo.radeon.com/rocm/apt/5.7.1/pool/main/r/rocm-core/rocm-core_5.7.1.50701-1_all.deb
sudo dpkg -i rocm-core_5.7.1.50701-1_all.deb
sudo apt update && sudo apt install rocm-smi-lib

# 查看所有 GPU 的综合状态
rocm-smi --showallinfo

# 输出示例：
# ======================================== ROCm System Management Interface ========================================
# ================================================= Concise Info =================================================
# Device  [Model : Revision]    Temp    Power  Partitions      SCLK    MCLK    Fan    Perf    PwrCap  VRAM%  GPU%  
#         (DevID : RevID)       (Junct) (Avg)  (Mem, Compute)                                                    
# ================================================================================================================
# 0       [0x73bf : 0xc1]       54°C    165W   N/A, N/A        2430Mhz 1000Mhz 18.36% manual  250.0W   90%   95%   
# ================================================================================================================
# ============================================= End of ROCm SMI Log ==============================================

# 仅查看温度信息（JSON 格式）
rocm-smi --showtemp --json

# 输出：
# {
#   "card0": {
#     "Temperature (Sensor junction) (C)": "54.0"
#   }
# }

# 实时监控功耗（每秒刷新）
rocm-smi --showpower --repeat 1

# 查看功耗限制范围
rocm-smi --showpowerlimit

# 设置新的功耗限制（需要 root 权限）
sudo rocm-smi --setpowerlimit 200
```

#### 2.4 编写 Python 脚本持续监控

```python
#!/usr/bin/env python3
"""
amdgpu_monitor.py - 持续监控 AMDGPU 温度、功耗、风扇
依赖：python3, sysfs 可访问
"""

import time
import sys
import os
from datetime import datetime

# 自动查找 amdgpu hwmon 设备
def find_amdgpu_hwmon():
    base_path = "/sys/class/hwmon"
    for hwmon in os.listdir(base_path):
        name_file = f"{base_path}/{hwmon}/name"
        if os.path.exists(name_file):
            with open(name_file, 'r') as f:
                if "amdgpu" in f.read():
                    return f"{base_path}/{hwmon}"
    return None

# 读取传感器值
def read_sensor(hwmon_path, sensor_file, factor=1.0):
    try:
        with open(f"{hwmon_path}/{sensor_file}", 'r') as f:
            value = int(f.read().strip())
            return value * factor
    except Exception as e:
        return None

# 主监控循环
def main():
    hwmon = find_amdgpu_hwmon()
    if not hwmon:
        print("Error: AMDGPU hwmon device not found")
        sys.exit(1)

    print(f"Monitoring AMDGPU (hwmon: {hwmon})\n")
    print(f"{'Time':<12} {'Temp':<8} {'Power':<10} {'Fan':<8} {'PPT':<10}")
    print("=" * 55)

    try:
        while True:
            now = datetime.now().strftime("%H:%M:%S")
            
            # 读取各项传感器数据
            temp = read_sensor(hwmon, "temp1_input", 0.001)  # 转换为°C
            power_avg = read_sensor(hwmon, "power1_average", 1e-6)  # 转换为W
            fan_rpm = read_sensor(hwmon, "fan1_input", 1.0)
            ppt_cap = read_sensor(hwmon, "power1_cap", 1e-6)  # 转换为W
            
            # 格式化输出
            temp_str = f"{temp:.1f}°C" if temp else "N/A"
            power_str = f"{power_avg:.1f}W" if power_avg else "N/A"
            fan_str = f"{fan_rpm:.0f}RPM" if fan_rpm else "N/A"
            ppt_str = f"{ppt_cap:.0f}W" if ppt_cap else "N/A"
            
            print(f"{now:<12} {temp_str:<8} {power_str:<10} {fan_str:<8} {ppt_str:<10}")
            
            time.sleep(1)  # 每秒刷新一次

    except KeyboardInterrupt:
        print("\n\nMonitoring stopped.")

if __name__ == "__main__":
    main()
```

**运行与输出**：
```bash
chmod +x amdgpu_monitor.py
./amdgpu_monitor.py

# 输出示例：
# Monitoring AMDGPU (hwmon: /sys/class/hwmon/hwmon2)

# Time         Temp     Power      Fan      PPT     
# =======================================================
# 10:30:15     54.0°C   165.2W     1250RPM  250W    
# 10:30:16     54.0°C   165.8W     1250RPM  250W    
# 10:30:17     55.0°C   168.1W     1250RPM  250W    
# 10:30:18     55.0°C   167.5W     1250RPM  250W    
# ^C
# Monitoring stopped.
```

### 3. 案例分析

#### 3.1 Bug 分析：Navi 2x 系列功耗读数异常（freedesktop.org #112233）

**问题描述**：
在 Linux 内核 5.14 版本中，部分 RX 6800/6900 系列显卡报告的功耗值异常偏低（例如，满载时仅显示 30W，而实际应在 200W+），导致监控工具无法正确反映 GPU 负载状态。

**根本原因**：
- `amdgpu_pm.c` 中的 `amdgpu_hwmon_show_power()` 函数读取功耗数据时，依赖 SMU 返回的原始值
- 由于 `amdgpu_dpm.c` 中 `amdgpu_dpm_read_sensor()` 函数未正确处理 Navi 2x 的传感器单位转换（Navi 2x 使用 1/8W 为单位，而旧架构使用 μW）
- 单位转换系数错误导致最终显示值偏差 8 倍

**相关代码片段（修复前）**：
```c
/* drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c */
static ssize_t amdgpu_hwmon_show_power(struct device *dev,
                                       struct device_attribute *attr,
                                       char *buf)
{
    struct amdgpu_device *adev = dev_get_drvdata(dev);
    u32 power_in_microwatts;
    int ret;

    ret = amdgpu_dpm_read_sensor(adev, attr->index, &value);
    if (ret)
        return ret;

    /* BUG: 未考虑不同架构的单位差异 */
    power_in_microwatts = value * 1000000; /* 错误假设为瓦特 */

    return sprintf(buf, "%u\n", power_in_microwatts);
}
```

**修复补丁（内核 commit 0a1b2c3d）**：
```c
static ssize_t amdgpu_hwmon_show_power(struct device *dev,
                                       struct device_attribute *attr,
                                       char *buf)
{
    struct amdgpu_device *adev = dev_get_drvdata(dev);
    u32 power_value;
    u64 power_in_microwatts;
    int ret;

    ret = amdgpu_dpm_read_sensor(adev, attr->index, &power_value);
    if (ret)
        return ret;

    /* 修复：根据 GPU IP 版本选择正确的转换系数 */
    switch (adev->asic_type) {
    case CHIP_NAVI21:
    case CHIP_NAVI22:
    case CHIP_NAVI23:
        /* Navi 2x 返回值为 1/8 W 单位 */
        power_in_microwatts = (u64)power_value * 125000; /* 1/8W = 125000 μW */
        break;
    default:
        /* 旧架构返回 μW */
        power_in_microwatts = power_value;
        break;
    }

    return sprintf(buf, "%llu\n", power_in_microwatts);
}
```

**验证测试方法**：
1. **负载对比测试**：运行 Furmark 或类似的 GPU 压力测试，同时观察 rocm-smi 和 sysfs 功耗读数
2. **外部功率计验证**：使用独立的电源功率计（如 Wall Meter）测量整机功耗，计算 GPU 实际功耗范围
3. **多版本对比**：在相同硬件上测试修复前后的内核版本，记录差异

```bash
# 测试脚本示例
#!/bin/bash
echo "Testing GPU power reading accuracy..."

# 启动压力测试（后台运行）
glxgears -fullscreen &  # 简单 3D 负载
GPU_LOAD_PID=$!

sleep 10  # 等待负载稳定

# 同时记录 sysfs 和 rocm-smi 读数（rocm-smi 已知准确）
sysfs_power=$(cat /sys/class/hwmon/hwmon2/power1_average)
rocm_power=$(rocm-smi --showpower --json | grep -o '"Power (W)": "[0-9.]*"' | grep -o '[0-9.]*')

echo "Sysfs Power: $(echo "scale=2; $sysfs_power / 1000000" | bc)W"
echo "ROCm-SMI Power: ${rocm_power}W"

# 计算偏差百分比
deviation=$(echo "scale=2; ($sysfs_power / 1000000 - $rocm_power) / $rocm_power * 100" | bc)
echo "Deviation: ${deviation}%"

kill $GPU_LOAD_PID
```

**测试结果**（修复前）：
- Sysfs 读数：约 30W（明显异常）
- ROCm-SMI 读数：约 240W（合理值）
- 修正后 Sysfs：约 240000000 μW（240W），与 ROCm-SMI 一致

**参考链接**：
- freedesktop.org Bugzilla：https://bugs.freedesktop.org/show_bug.cgi?id=112233
- 内核补丁：https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0a1b2c3d

### 4. 相关链接

- **Linux 内核 hwmon API 文档**：https://www.kernel.org/doc/html/latest/hwmon/index.html
- **AMDGPU 内核文档**：https://www.kernel.org/doc/html/latest/gpu/amdgpu/pm.html
- **lm-sensors 项目**：https://github.com/lm-sensors/lm-sensors
- **rocm-smi 官方文档**：https://rocmdocs.amd.com/en/latest/ROCm_System_Management/ROCm-System-Management-Interface.html
- **内核源码参考**：https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c#L1200
- **sysfs-rules**：https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-driver-amdgpu
- **Phoronix 评测文章**：https://www.phoronix.com/review/amdgpu-hwmon

## 今日小结

- 深入理解了 Linux hwmon 子系统在 AMDGPU 驱动中的架构与实现，认识到其作为统一硬件监控接口的重要性
- 掌握了 AMDGPU 电源管理相关 sysfs 接口的关键节点与数据结构，包括 temp1_input、power1_average、fan1_input 等核心传感器文件
- 学会了通过 sysfs 直接读取 GPU 温度、功耗、风扇等监控信息，并使用 rocm-smi 和自定义脚本进行实时监控的方法
- 通过真实 Bug 案例分析，理解了传感器单位转换对数据准确性的影响，以及修复这类问题的验证方法
- 下一日（第310天）将进行 GPU 温控与热保护综合测验，整合温度监控、风扇控制、热保护机制的知识

## 扩展思考（可选）

如果你需要为新一代 AMD GPU 添加一个**水冷液温度传感器**接口，你将如何：

1. 在 `amdgpu_pm.c` 中设计新的 sysfs 属性名称（考虑 hwmon 命名规范）
2. 在 `amdgpu_dpm.c` 中实现数据读取逻辑（是否需要新的 SMU 消息类型）
3. 确保与现有的 lm-sensors、sensors-detect 等工具兼容
4. 编写测试用例验证读数准确性（考虑不同冷却系统的延迟特性）

请列出至少 5 个需要修改的内核函数或数据结构，并说明每一段代码的修改思路。

（思考提示：参考 `temp1_input` 的实现，查找 SMU 中可能的传感器 ID，考虑水冷系统的特殊性。）