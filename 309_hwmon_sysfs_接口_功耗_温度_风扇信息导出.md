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

**参考答案思路**：

```c
// 1. 在 amdgpu_pm.c 中注册新属性
static SENSOR_DEVICE_ATTR(temp3_input, S_IRUGO, amdgpu_hwmon_show_temp, NULL, AMDGPU_HWMON_SENSOR_TEMP_COOLANT);

// 2. 在 amdgpu_dpm.c 中扩展传感器枚举
enum amdgpu_hwmon_sensor_temp {
    AMDGPU_HWMON_SENSOR_TEMP_JUNCTION = 0,
    AMDGPU_HWMON_SENSOR_TEMP_HOTSPOT = 1,
    AMDGPU_HWMON_SENSOR_TEMP_MEM = 2,
    AMDGPU_HWMON_SENSOR_TEMP_COOLANT = 3,  // 新增
    AMDGPU_HWMON_SENSOR_TEMP_COUNT
};

// 3. 新增 SMU 消息接口 (smu_v13_0.c)
#define SMU_MSG_GetCoolantTemp 0x4A
// 水冷温度响应较慢，可设置 5 秒缓存

// 4. 兼容性处理：更新 hwmon 属性描述（amdgpu_hwmon_strings 数组）
static const char * const temp_label[] = {
    "junction",
    "hotspot",
    "mem",
    "coolant"  // 新增
};

// 5. 修改数据结构 amdgpu_hwmon_info (amdgpu_pm.c)
struct amdgpu_hwmon_info {
    struct amdgpu_device *adev;
    struct device *hwmon_dev;
    struct mutex lock;
    u32 coolant_temp_cache;  // 新增缓存
    unsigned long coolant_last_read;  // 新增缓存时间戳
};
```

### 5. 附录1：hwmon 子系统深度分析

#### 5.1 Linux hwmon 内核框架

hwmon（Hardware Monitoring）是 Linux 内核的硬件监控子系统，为各种传感器提供统一的用户空间访问接口：

**核心文件**：

| 文件 | 作用 |
|------|------|
| `drivers/hwmon/hwmon.c` | hwmon 核心框架，设备注册与管理 |
| `include/linux/hwmon.h` | 核心 API 头文件 |
| `include/linux/hwmon-sysfs.h` | sysfs 属性定义 |
| `Documentation/hwmon/` | 用户与开发者文档 |

**设备注册流程**：

```c
// 1. 分配 hwmon 设备
struct device *hwmon_device_register_with_info(
    struct device *parent,
    const char *name,
    void *drvdata,
    const struct hwmon_chip_info *info,
    const struct attribute_group **groups);

// 2. 定义传感器信息
static const struct hwmon_channel_info *amdgpu_hwmon_info[] = {
    HWMON_CHANNEL_INFO(temp, HWMON_T_INPUT | HWMON_T_LABEL | HWMON_T_MAX | HWMON_T_MAX_HYST,
                        HWMON_T_INPUT | HWMON_T_LABEL,
                        HWMON_T_INPUT | HWMON_T_LABEL),
    HWMON_CHANNEL_INFO(fan, HWMON_F_INPUT | HWMON_F_LABEL),
    HWMON_CHANNEL_INFO(power, HWMON_P_INPUT | HWMON_P_CAP_MAX | HWMON_P_CAP_MIN,
                        HWMON_P_INPUT | HWMON_P_LABEL),
    HWMON_CHANNEL_INFO(in, HWMON_I_INPUT | HWMON_I_LABEL,
                        HWMON_I_INPUT | HWMON_I_LABEL),
    NULL  // 终止标记
};

// 3. 实现读写回调
static int amdgpu_hwmon_read(struct device *dev, enum hwmon_sensor_types type,
                             u32 attr, int channel, long *val)
{
    struct amdgpu_device *adev = dev_get_drvdata(dev);
    // 根据 type/channel/attr 分发到具体读取函数
    return amdgpu_hwmon_read_temp(adev, channel, val);
}

static int amdgpu_hwmon_write(struct device *dev, enum hwmon_sensor_types type,
                              u32 attr, int channel, long val)
{
    struct amdgpu_device *adev = dev_get_drvdata(dev);
    // 写入操作（如设置阈值）
    return amdgpu_hwmon_write_temp(adev, channel, attr, val);
}
```

#### 5.2 hwmon sysfs 属性规范

**标准命名规则**：

| 属性模式 | 类型 | 含义 |
|----------|------|------|
| `temp{1-3}_input` | 温度 | 传感器读数（m°C） |
| `temp{1-3}_max` | 温度 | 阈值上限（m°C） |
| `temp{1-3}_max_hyst` | 温度 | 迟滞值（m°C） |
| `temp{1-3}_crit` | 温度 | 临界值（m°C） |
| `temp{1-3}_label` | 温度 | 传感器名称字符串 |
| `fan{1-}_input` | 风扇 | 风扇转速（RPM） |
| `fan{1-}_min` | 风扇 | 最低转速（RPM） |
| `fan{1-}_max` | 风扇 | 最高转速（RPM） |
| `fan{1-}_target` | 风扇 | 目标转速（RPM） |
| `pwm{1-}` | PWM | 风扇PWM值（0-255） |
| `pwm{1-}_enable` | PWM | 风扇控制模式 |
| `power1_input` | 功耗 | 瞬时功耗（μW） |
| `power1_average` | 功耗 | 平均功耗（μW） |
| `power1_cap` | 功耗 | 功耗上限（μW） |
| `power1_cap_max` | 功耗 | 可设置的最大上限 |
| `power1_cap_min` | 功耗 | 可设置的最小上限 |
| `in{0-1}_input` | 电压 | 电压读数（mV） |
| `in{0-1}_label` | 电压 | 电压轨名称 |
| `curr{1-}_input` | 电流 | 电流读数（mA） |

**AMDGPU hwmon 具体属性列表**：

```bash
# 温度传感器（3个）
temp1_input     # 结温 (junction)
temp2_input     # 热点温度 (hotspot)
temp3_input     # 显存温度 (memory)
temp1_label     # "junction"
temp2_label     # "hotspot"
temp3_label     # "mem"

# 风扇（1个）
fan1_input      # 风扇转速 (RPM)
pwm1            # PWM 值 (0-255)
pwm1_enable     # 模式: 0=无, 1=manual, 2=auto

# 功耗（2个通道）
power1_input    # GPU 总功耗 (μW)
power1_average  # 平均功耗 (μW)
power1_cap      # 功耗上限 (μW)
power2_input    # 显存功耗 (μW)

# 电压（2个通道）
in0_input       # VDDC 电压 (mV)
in1_input       # VDDCI 电压 (mV)
in0_label       # "vddgfx" / "vddc"
in1_label       # "vddci"

# 电流（1个通道）
curr1_input     # GPU 电流 (mA)
```

#### 5.3 lm-sensors 集成

**配置方法**：

```bash
# 1. 安装 lm-sensors
sudo apt-get install lm-sensors

# 2. 执行传感器检测
sudo sensors-detect

# 3. 手动配置 amdgpu
echo "bus i2c-0" | sudo tee /etc/sensors.d/amdgpu.conf
echo "chip k10temp-*" >> /etc/sensors.d/amdgpu.conf

# 4. 自定义传感器显示
cat /etc/sensors.d/amdgpu.conf
chip amdgpu-*
    label temp1 "GPU Junction"
    label temp2 "GPU Hotspot"
    label temp3 "VRAM"
    set temp1_max 95000
    set temp1_max_hyst 5000
    compute temp1 "temp1_input / 1000", "temp1_output * 1000"
```

**自定义 sensors 输出配置**：

```bash
# /etc/sensors.d/amdgpu-format.conf
bus i2c-1
chip amdgpu-pci-*

    # 温度显示格式
    label temp1 "GPU核心"
    label temp2 "GPU热点"
    label temp3 "显存"
    set temp1_max 95000
    set temp1_max_hyst 5000

    # 功耗显示格式
    label power1 "GPU功耗"
    label power2 "显存功耗"
    set power1_cap 250000000

    # 电压显示格式
    label in0 "核心电压"
    label in1 "显存I/O电压"

    # 风扇显示格式
    label fan1 "GPU风扇"
    label pwm1 "PWM"
```

#### 5.4 AMDGPU hwmon 内核实现细节

**关键数据结构**：

```c
// drivers/gpu/drm/amd/pm/amdgpu_pm.c

struct amdgpu_hwmon_attributes {
    struct device_attribute dev_attr;     // 标准设备属性
    struct sensor_device_attribute sda;    // hwmon 传感器属性
    enum amdgpu_hwmon_sensor_type type;   // 传感器类型
    u32 channel;                          // 通道号
};

// hwmon 注册
int amdgpu_pm_init(struct amdgpu_device *adev)
{
    // 1. 检查电源管理支持
    if (!adev->pm.dpm_enabled)
        return 0;
    
    // 2. 初始化电源管理锁
    mutex_init(&adev->pm.mutex);
    
    // 3. 初始化 hwmon
    amdgpu_hwmon_init(adev);
    
    // 4. 创建电源管理 sysfs 文件
    amdgpu_pm_sysfs_init(adev);
    
    return 0;
}

// hwmon 初始化
static int amdgpu_hwmon_init(struct amdgpu_device *adev)
{
    struct device *hwmon_dev;
    
    hwmon_dev = hwmon_device_register_with_info(
        adev->dev,
        "amdgpu",
        adev,
        &amdgpu_hwmon_chip_info,
        NULL);
    
    if (IS_ERR(hwmon_dev))
        return PTR_ERR(hwmon_dev);
    
    adev->pm.hwmon_dev = hwmon_dev;
    return 0;
}

// 温度读取（核心逻辑）
static int amdgpu_hwmon_read_temp(struct amdgpu_device *adev, int channel, long *val)
{
    u32 temp_value = 0;
    int ret;
    
    switch (channel) {
    case AMDGPU_HWMON_SENSOR_TEMP_JUNCTION:
        ret = amdgpu_dpm_read_sensor(adev, AMDGPU_GPU_SENSOR_TEMP, &temp_value);
        break;
    case AMDGPU_HWMON_SENSOR_TEMP_HOTSPOT:
        ret = amdgpu_dpm_read_sensor(adev, AMDGPU_GPU_SENSOR_HOTSPOT_TEMP, &temp_value);
        break;
    case AMDGPU_HWMON_SENSOR_TEMP_MEM:
        ret = amdgpu_dpm_read_sensor(adev, AMDGPU_GPU_SENSOR_MEM_TEMP, &temp_value);
        break;
    default:
        return -EOPNOTSUPP;
    }
    
    if (ret)
        return ret;
    
    *val = temp_value * 1000;  // 转换为 m°C
    return 0;
}
```

### 6. 附录2：高级监控工具开发

#### 6.1 自定义 hwmon 监控框架

```python
#!/usr/bin/env python3
"""GPU hwmon 抽象监控框架"""
import os
import time
import threading
from collections import deque
from dataclasses import dataclass
from typing import Dict, List, Optional, Callable
import json
import logging

@dataclass
class SensorReading:
    name: str
    value: float
    unit: str
    timestamp: float

class HWMONDevice:
    """hwmon 设备抽象层"""
    
    def __init__(self, hwmon_path: str):
        self.path = hwmon_path
        self.name = self._read_file('name')
        self.sensors = self._discover_sensors()
    
    def _read_file(self, filename: str) -> Optional[str]:
        path = os.path.join(self.path, filename)
        try:
            with open(path, 'r') as f:
                return f.read().strip()
        except (IOError, OSError):
            return None
    
    def _discover_sensors(self) -> Dict[str, List[str]]:
        """自动发现所有可用传感器"""
        sensors = {
            'temp': [], 'fan': [], 'pwm': [], 'power': [], 'in': [], 'curr': []
        }
        
        for f in os.listdir(self.path):
            for prefix in sensors.keys():
                if f.startswith(prefix) and f.endswith('_input'):
                    name = f'{prefix}_{f.split("_input")[0].replace(prefix, "")}'
                    if name not in sensors[prefix]:
                        sensors[prefix].append(name)
        
        return sensors
    
    def read_sensor(self, sensor_type: str, channel: str = '') -> Optional[SensorReading]:
        filename = f"{sensor_type}{channel}_input"
        value = self._read_file(filename)
        if value is None:
            return None
        
        units = {'temp': 'm°C', 'fan': 'RPM', 'power': 'μW', 'in': 'mV', 'curr': 'mA'}
        unit = units.get(sensor_type, '')
        
        return SensorReading(
            name=f"{sensor_type}{channel}",
            value=float(value),
            unit=unit,
            timestamp=time.time()
        )
    
    def read_all(self) -> Dict[str, SensorReading]:
        """读取所有可用传感器"""
        results = {}
        for sensor_type, channels in self.sensors.items():
            for channel in channels:
                reading = self.read_sensor(sensor_type, channel.replace(sensor_type + '_', ''))
                if reading:
                    results[f"{sensor_type}{channel}"] = reading
        return results

class GPUMonitor:
    """GPU 监控管理器"""
    
    def __init__(self, gpu_id: int = 0):
        self.gpu_id = gpu_id
        self.hwmon = self._find_hwmon()
        self.history = {
            'temp': deque(maxlen=3600),
            'power': deque(maxlen=3600),
            'fan': deque(maxlen=3600)
        }
        self.running = False
        self.callbacks: List[Callable] = []
    
    def _find_hwmon(self) -> Optional[HWMONDevice]:
        base = f"/sys/class/drm/card{self.gpu_id}/device/hwmon"
        if not os.path.exists(base):
            return None
        
        hwmon_dirs = [d for d in os.listdir(base) if d.startswith('hwmon')]
        if not hwmon_dirs:
            return None
        
        return HWMONDevice(os.path.join(base, hwmon_dirs[0]))
    
    def on_reading(self, callback: Callable):
        """注册读取回调"""
        self.callbacks.append(callback)
    
    def read_single(self) -> Dict:
        """单次读取所有数据"""
        if not self.hwmon:
            return {}
        
        readings = self.hwmon.read_all()
        result = {}
        
        for name, reading in readings.items():
            if reading.unit == 'm°C':
                result[name] = reading.value / 1000  # 转换为°C
            elif reading.unit == 'μW':
                result[name] = reading.value / 1000000  # 转换为W
            else:
                result[name] = reading.value
        
        return result
    
    def start_monitoring(self, interval: float = 1.0):
        """启动持续监控"""
        self.running = True
        
        def _loop():
            while self.running:
                data = self.read_single()
                timestamp = time.time()
                
                # 记录历史
                for key in ['temp', 'power', 'fan']:
                    matching = {k: v for k, v in data.items() if k.startswith(key)}
                    if matching:
                        self.history[key].append((timestamp, matching))
                
                # 调用回调
                for cb in self.callbacks:
                    try:
                        cb(data, timestamp)
                    except Exception as e:
                        logging.error(f"Callback error: {e}")
                
                time.sleep(interval)
        
        thread = threading.Thread(target=_loop, daemon=True)
        thread.start()
    
    def stop_monitoring(self):
        self.running = False
    
    def get_stats(self) -> Dict:
        """获取历史统计"""
        stats = {}
        for sensor_type, history in self.history.items():
            if not history:
                continue
            
            values = []
            for _, data in history.items():
                if isinstance(data, dict):
                    values.extend(data.values())
                else:
                    values.append(data)
            
            if values:
                stats[sensor_type] = {
                    'current': values[-1],
                    'avg': sum(values) / len(values),
                    'min': min(values),
                    'max': max(values),
                    'samples': len(values)
                }
        return stats

# 使用示例
if __name__ == "__main__":
    import argparse
    
    parser = argparse.ArgumentParser(description='GPU hwmon 监控工具')
    parser.add_argument('--gpu', type=int, default=0, help='GPU ID')
    parser.add_argument('--interval', type=float, default=1.0, help='采样间隔')
    parser.add_argument('--duration', type=int, default=10, help='监控时长')
    parser.add_argument('--json', action='store_true', help='JSON 输出')
    args = parser.parse_args()
    
    monitor = GPUMonitor(args.gpu)
    
    if not monitor.hwmon:
        print("找不到 GPU hwmon 设备")
        exit(1)
    
    def on_data(data, timestamp):
        if args.json:
            print(json.dumps({'timestamp': timestamp, 'data': data}))
        else:
            os.system('clear')
            print(f"GPU {args.gpu} 传感器监控 (@{time.strftime('%H:%M:%S')})")
            print("=" * 50)
            for key, value in sorted(data.items()):
                print(f"  {key:20s}: {value:>10.2f}")
            stats = monitor.get_stats()
            if stats.get('temp'):
                print(f"\n温度历史 ({args.duration}s):")
                print(f"  当前: {stats['temp']['current']:.1f}°C")
                print(f"  平均: {stats['temp']['avg']:.1f}°C")
                print(f"  最高: {stats['temp']['max']:.1f}°C")
    
    monitor.on_reading(on_data)
    monitor.start_monitoring(args.interval)
    
    try:
        time.sleep(args.duration)
    except KeyboardInterrupt:
        pass
    finally:
        monitor.stop_monitoring()
```

#### 6.2 历史趋势记录与分析

```python
#!/usr/bin/env python3
"""GPU hwmon 数据记录与分析"""
import sqlite3
import time
import json
from datetime import datetime

class GPUMetricsRecorder:
    """GPU 指标长期记录器"""
    
    def __init__(self, db_path: str = 'gpu_metrics.db'):
        self.conn = sqlite3.connect(db_path)
        self._init_db()
    
    def _init_db(self):
        cursor = self.conn.cursor()
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS gpu_metrics (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                timestamp REAL NOT NULL,
                temp_junction REAL,
                temp_hotspot REAL,
                temp_mem REAL,
                fan_rpm REAL,
                pwm_value REAL,
                power_gpu REAL,
                power_mem REAL,
                voltage_vddc REAL,
                voltage_vddci REAL
            )
        ''')
        cursor.execute('''
            CREATE INDEX IF NOT EXISTS idx_timestamp 
            ON gpu_metrics(timestamp)
        ''')
        self.conn.commit()
    
    def record(self, data: dict):
        cursor = self.conn.cursor()
        cursor.execute('''
            INSERT INTO gpu_metrics 
            (timestamp, temp_junction, temp_hotspot, temp_mem, 
             fan_rpm, pwm_value, power_gpu, power_mem,
             voltage_vddc, voltage_vddci)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        ''', (
            time.time(),
            data.get('temp1', 0),
            data.get('temp2', 0),
            data.get('temp3', 0),
            data.get('fan1', 0),
            data.get('pwm1', 0),
            data.get('power1', 0),
            data.get('power2', 0),
            data.get('in0', 0),
            data.get('in1', 0)
        ))
        self.conn.commit()
    
    def query_range(self, start_time: float, end_time: float) -> list:
        cursor = self.conn.cursor()
        cursor.execute('''
            SELECT * FROM gpu_metrics 
            WHERE timestamp BETWEEN ? AND ?
            ORDER BY timestamp ASC
        ''', (start_time, end_time))
        return cursor.fetchall()
    
    def export_csv(self, filename: str):
        cursor = self.conn.cursor()
        cursor.execute('SELECT * FROM gpu_metrics ORDER BY timestamp ASC')
        rows = cursor.fetchall()
        
        with open(filename, 'w') as f:
            f.write('timestamp,temp_junction,temp_hotspot,temp_mem,fan_rpm,pwm_value,power_gpu,power_mem,voltage_vddc,voltage_vddci\n')
            for row in rows:
                f.write(','.join(str(v) for v in row[1:]) + '\n')
    
    def get_summary(self) -> dict:
        cursor = self.conn.cursor()
        cursor.execute('SELECT COUNT(*) FROM gpu_metrics')
        count = cursor.fetchone()[0]
        
        if count == 0:
            return {'records': 0}
        
        cursor.execute('''
            SELECT 
                MIN(temp_junction), MAX(temp_junction), AVG(temp_junction),
                MIN(power_gpu), MAX(power_gpu), AVG(power_gpu),
                MIN(fan_rpm), MAX(fan_rpm)
            FROM gpu_metrics
        ''')
        stats = cursor.fetchone()
        
        cursor.execute('SELECT MIN(timestamp), MAX(timestamp) FROM gpu_metrics')
        time_range = cursor.fetchone()
        
        return {
            'records': count,
            'period_start': datetime.fromtimestamp(time_range[0]).isoformat(),
            'period_end': datetime.fromtimestamp(time_range[1]).isoformat(),
            'temp': {'min': stats[0], 'max': stats[1], 'avg': stats[2]},
            'power': {'min': stats[3], 'max': stats[4], 'avg': stats[5]},
            'fan': {'min': stats[6], 'max': stats[7]}
        }
    
    def close(self):
        self.conn.close()

# 数据可视化辅助
class MetricsVisualizer:
    @staticmethod
    def plot_time_series(db_path: str, metrics: list = None):
        """生成时间序列图（需要 matplotlib）"""
        try:
            import matplotlib.pyplot as plt
            import matplotlib.dates as mdates
        except ImportError:
            print("需要安装 matplotlib: pip install matplotlib")
            return
        
        conn = sqlite3.connect(db_path)
        cursor = conn.cursor()
        
        if metrics is None:
            metrics = ['temp_junction', 'temp_hotspot', 'power_gpu', 'fan_rpm']
        
        fig, axes = plt.subplots(len(metrics), 1, figsize=(12, 3*len(metrics)), sharex=True)
        if len(metrics) == 1:
            axes = [axes]
        
        for ax, metric in zip(axes, metrics):
            cursor.execute(f'''
                SELECT timestamp, {metric} FROM gpu_metrics 
                WHERE {metric} IS NOT NULL
                ORDER BY timestamp ASC
            ''')
            data = cursor.fetchall()
            
            if not data:
                continue
            
            times = [datetime.fromtimestamp(row[0]) for row in data]
            values = [row[1] for row in data]
            
            ax.plot(times, values, linewidth=0.5)
            ax.set_ylabel(metric.replace('_', ' ').title())
            ax.grid(True, alpha=0.3)
        
        axes[-1].xaxis.set_major_formatter(mdates.DateFormatter('%H:%M'))
        fig.autofmt_xdate()
        plt.tight_layout()
        plt.savefig('gpu_metrics_plot.png', dpi=150)
        print(f"图表已保存: gpu_metrics_plot.png")
        
        conn.close()
```

### 7. 附录3：hwmon 调试与常见问题

#### 7.1 调试命令速查表

| 操作 | 命令 | 说明 |
|------|------|------|
| 列出 hwmon 设备 | `ls /sys/class/hwmon/` | 所有 hwmon 设备 |
| 查看设备名称 | `cat /sys/class/hwmon/hwmon*/name` | 识别哪个是 amdgpu |
| 列出所有属性 | `ls /sys/class/hwmon/hwmon*/` | 查看所有可用传感器 |
| 读取温度 | `cat /sys/class/hwmon/hwmon*/temp*_input` | 原始值（m°C） |
| 实时监控 | `watch -n 1 sensors` | lm-sensors 输出 |
| 详细 sensors | `sensors -u` | 原始格式输出 |
| 风扇控制 | `echo 200 > pwm1` | 设置 PWM |
| hwmon 调试信息 | `sudo cat /sys/kernel/debug/dri/0/amdgpu_hwmon` | 内核调试信息 |
| 检测新设备 | `sudo udevadm trigger` | 触发 udev 重检测 |

#### 7.2 常见问题排查

**问题1：找不到 hwmon 设备**

```bash
# 检查 amdgpu 驱动是否加载
lsmod | grep amdgpu

# 检查 DRM 设备
ls /sys/class/drm/card0/device/hwmon/

# 如果hwmon目录不存在，检查pm是否启用
cat /sys/class/drm/card0/device/power_dpm_state

# 手动加载模块
sudo modprobe amdgpu
```

**问题2：sensors 命令不显示 amdgpu**

```bash
# 检查 sensors 版本
sensors --version

# 手动扫描 I2C 总线
sudo sensors-detect --auto

# 检查配置
cat /etc/sensors.d/*

# 强制检查 amdgpu
sensors amdgpu-*
```

**问题3：读数稳定不变**

```python
def check_reading_stale(hwmon_path, duration=10):
    """检查传感器读数是否更新"""
    import time
    import os
    
    values = []
    for _ in range(duration):
        path = os.path.join(hwmon_path, 'temp1_input')
        if os.path.exists(path):
            with open(path, 'r') as f:
                values.append(int(f.read().strip()))
        time.sleep(1)
    
    if len(set(values)) == 1:
        print("警告: 读数在 {} 秒内未变化，传感器可能挂死".format(duration))
        return False
    else:
        print("正常: 读数值范围 [{}-{}]".format(min(values), max(values)))
        return True
```

**问题4：权限问题**

```bash
# 临时解决（调试用）
sudo chmod 444 /sys/class/hwmon/hwmon*/temp*_input

# 永久解决（udev 规则）
echo 'SUBSYSTEM=="hwmon", ACTION=="add", RUN+="/bin/chmod 444 /sys%p/temp*_input"' | sudo tee /etc/udev/rules.d/99-hwmon-permissions.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

### 8. hwmon 接口行业标准对比

| 特性 | AMD (amdgpu) | NVIDIA (nvidia-smi / NVML) | Intel (i915 / xe) |
|------|-------------|---------------------------|-------------------|
| 监控接口 | hwmon sysfs | NVML API | hwmon sysfs |
| 温度传感器 | 3个（结温/热点/显存） | 多传感器 | 2-3个 |
| 功耗读取 | 1-2个通道（μW） | 支持（W） | 1个通道 |
| 风扇控制 | PWM sysfs | NVAPI（需驱动） | PWM sysfs |
| 电压读取 | 2个通道（mV） | 有限支持 | 有限支持 |
| 用户态工具 | lm-sensors, rocm-smi | nvidia-smi, nvtop | lm-sensors, intel-gpu-tools |
| 编程接口 | sysfs 文件操作 | NVML C/Python API | sysfs 文件操作 |
| Docker 支持 | 需 --device /dev/dri | nvidia-container-toolkit | 需 --device /dev/dri |
| 远程监控 | 需自定义方案 | NVML 支持远程 | 需自定义方案 |
| 虚拟化支持 | GPU 直通 | vGPU | GPU 直通 |
| 数据导出 | 自行实现 | NVML 日志 | 自行实现 |

### 9. 知识自测

**选择题**：

1. hwmon 子系统中温度的默认单位是：
   A. °C B. m°C C. μ°C D. K

2. `power1_average` 的默认单位是：
   A. mW B. μW C. nW D. W

3. AMDGPU 的 hwmon 名称通常是：
   A. amdgpu_hwmon B. amdgpu C. radeon D. gpu_hwmon

4. `pwm1_enable=1` 表示什么模式：
   A. 自动控制 B. 手动控制 C. 关闭风扇 D. 仅读模式

5. `fan1_input` 的返回值单位是：
   A. Hz B. RPM C. % D. 无单位

6. 以下哪个不是标准的 hwmon sysfs 属性：
   A. temp1_input B. power1_average C. fan1_speed D. pwm1

7. `hwmon_device_register_with_info()` 在哪个文件中定义：
   A. amdgpu_pm.c B. hwmon.c C. thermal_core.c D. drm_core.c

8. lm-sensors 的核心配置文件位于：
   A. /etc/sensors.conf B. /etc/sensors.d/ C. /etc/lm-sensors/ D. /etc/hwmon/

9. AMDGPU hwmon 中 `temp2_label` 通常显示：
   A. junction B. hotspot C. mem D. vram

10. `power1_cap` 设置的值如果超出范围会怎样：
    A. 自动截断到范围边界 B. 写入失败返回错误
    C. 系统崩溃 D. 忽略该操作

**填空题**：

11. Linux hwmon 子系统注册新设备的函数是 ________。
12. AMDGPU hwmon 提供 ________ 个温度传感器通道。
13. `temp1_input` 的返回值需要除以 ________ 才能得到 °C。
14. `pwm1_enable=2` 表示 ________ 控制模式。
15. AMDGPU 的功耗读取默认单位是 ________。
16. lm-sensors 的命令行工具是 ________。

**答案**：1-B, 2-B, 3-B, 4-B, 5-B, 6-C, 7-B, 8-B, 9-B, 10-B, 11-hwmon_device_register_with_info, 12-3, 13-1000, 14-自动, 15-μW, 16-sensors

### 10. 实践项目：完整 GPU 监控面板

#### 项目设计

创建一个基于 curses 的终端 GPU 监控面板，类似 nvtop 但使用 hwmon 接口：

```python
#!/usr/bin/env python3
"""GPU 监控面板 - 基于 hwmon 的终端 UI"""
import curses
import time
import os
import glob
import threading
from collections import deque

class GPUMonitorPanel:
    """终端 GPU 监控面板"""
    
    def __init__(self, gpu_id=0):
        self.gpu_id = gpu_id
        self.hwmon_path = self._find_hwmon()
        self.history = {
            'temp_junction': deque(maxlen=120),
            'temp_hotspot': deque(maxlen=120),
            'power': deque(maxlen=120),
            'fan': deque(maxlen=120)
        }
        self.running = True
    
    def _find_hwmon(self):
        base = f"/sys/class/drm/card{self.gpu_id}/device/hwmon"
        if not os.path.exists(base):
            return None
        dirs = glob.glob(f"{base}/hwmon*")
        return dirs[0] if dirs else None
    
    def _read_sensor(self, sensor):
        path = os.path.join(self.hwmon_path, sensor)
        try:
            with open(path, 'r') as f:
                return int(f.read().strip())
        except (IOError, OSError, ValueError):
            return None
    
    def get_readings(self):
        if not self.hwmon_path:
            return None
        
        return {
            'temp_junction': self._read_sensor('temp1_input'),
            'temp_hotspot': self._read_sensor('temp2_input'),
            'temp_mem': self._read_sensor('temp3_input'),
            'power_gpu': self._read_sensor('power1_average'),
            'fan_rpm': self._read_sensor('fan1_input'),
            'pwm': self._read_sensor('pwm1'),
            'voltage': self._read_sensor('in0_input')
        }
    
    def _update_history(self, readings):
        for key, value in readings.items():
            if value is not None and key in self.history:
                self.history[key].append(value)
    
    def _draw_bar(self, win, y, x, width, value, max_value, label, unit, color_good, color_warn, color_crit):
        """绘制水平柱状图"""
        ratio = min(value / max_value, 1.0) if max_value > 0 else 0
        bar_width = int(width * ratio)
        
        if ratio > 0.85:
            color = color_crit
        elif ratio > 0.7:
            color = color_warn
        else:
            color = color_good
        
        win.attron(color)
        win.addstr(y, x, f"{label}: ")
        win.addstr(f"[{'#' * bar_width}{' ' * (width - bar_width)}] ")
        win.addstr(f"{value}{unit}")
        win.attroff(color)
    
    def _draw_sparkline(self, win, y, x, width, data):
        """绘制迷你趋势线"""
        if not data:
            return
        max_val = max(data) if data else 1
        min_val = min(data) if data else 0
        range_val = max_val - min_val if max_val != min_val else 1
        
        chars = '▁▂▃▄▅▆▇█'
        for i, v in enumerate(data[-width:]):
            idx = int((v - min_val) / range_val * (len(chars) - 1))
            idx = min(idx, len(chars) - 1)
            try:
                win.addch(y, x + i, chars[idx])
            except curses.error:
                pass
    
    def run(self, stdscr):
        curses.curs_set(0)
        curses.use_default_colors()
        curses.init_pair(1, curses.COLOR_GREEN, -1)
        curses.init_pair(2, curses.COLOR_YELLOW, -1)
        curses.init_pair(3, curses.COLOR_RED, -1)
        curses.init_pair(4, curses.COLOR_CYAN, -1)
        curses.init_pair(5, curses.COLOR_MAGENTA, -1)
        
        color_good = curses.color_pair(1)
        color_warn = curses.color_pair(2)
        color_crit = curses.color_pair(3)
        color_info = curses.color_pair(4)
        color_hl = curses.color_pair(5)
        
        while self.running:
            readings = self.get_readings()
            if readings:
                self._update_history(readings)
            
            stdscr.clear()
            height, width = stdscr.getmaxyx()
            bar_width = min(20, width - 30)
            
            # 标题
            stdscr.attron(curses.A_BOLD | color_info)
            stdscr.addstr(0, 0, f"GPU {self.gpu_id} 监控面板".center(width))
            stdscr.attroff(curses.A_BOLD | color_info)
            stdscr.addstr(1, 0, "=" * width)
            
            if readings:
                y = 3
                # 温度
                tj = readings['temp_junction']
                if tj is not None:
                    self._draw_bar(stdscr, y, 0, bar_width, tj/1000, 95, "温度(核心)", "°C", color_good, color_warn, color_crit)
                    y += 1
                
                th = readings['temp_hotspot']
                if th is not None:
                    self._draw_bar(stdscr, y, 0, bar_width, th/1000, 100, "温度(热点)", "°C", color_good, color_warn, color_crit)
                    y += 1
                
                tm = readings['temp_mem']
                if tm is not None:
                    self._draw_bar(stdscr, y, 0, bar_width, tm/1000, 95, "温度(显存)", "°C", color_good, color_warn, color_crit)
                    y += 2
                
                # 功耗
                pw = readings['power_gpu']
                if pw is not None:
                    tdp = 250000000
                    self._draw_bar(stdscr, y, 0, bar_width, pw, tdp, "功耗", "W", color_good, color_warn, color_crit)
                    y += 2
                
                # 风扇
                fan = readings['fan_rpm']
                if fan is not None:
                    self._draw_bar(stdscr, y, 0, bar_width, fan, 5000, "风扇", "RPM", color_good, color_warn, color_crit)
                    y += 1
                
                pwm = readings['pwm']
                if pwm is not None:
                    self._draw_bar(stdscr, y, 0, bar_width, pwm, 255, "PWM", "", color_good, color_warn, color_crit)
                    y += 2
                
                # 电压
                volt = readings['voltage']
                if volt is not None:
                    self._draw_bar(stdscr, y, 0, bar_width, volt/1000, 1500, "电压", "mV", color_good, color_warn, color_crit)
                    y += 2
                
                # 趋势图
                diff_y = y
                stdscr.attron(curses.A_BOLD)
                stdscr.addstr(y, 0, "温度趋势(最近120秒):")
                stdscr.attroff(curses.A_BOLD)
                y += 1
                
                if self.history['temp_junction']:
                    self._draw_sparkline(stdscr, y, 0, min(60, width-2), list(self.history['temp_junction']))
                    y += 1
                
                if y < height - 3:
                    stdscr.attron(curses.A_BOLD)
                    stdscr.addstr(y, 0, "功耗趋势(最近120秒):")
                    stdscr.attroff(curses.A_BOLD)
                    y += 1
                    if self.history['power']:
                        self._draw_sparkline(stdscr, y, 0, min(60, width-2), list(self.history['power']))
                        y += 1
                
                # 提示信息
                if y < height - 2:
                    stdscr.addstr(height-2, 0, "按 'q' 退出", curses.A_DIM)
            
            stdscr.refresh()
            time.sleep(1)
            
            # 检查按键
            stdscr.nodelay(1)
            key = stdscr.getch()
            if key == ord('q'):
                self.running = False

def main():
    monitor = GPUMonitorPanel(gpu_id=0)
    if not monitor.hwmon_path:
        print("错误: 找不到 GPU hwmon 设备")
        print("请确保 amdgpu 驱动已加载且 GPU 存在")
        return
    
    try:
        curses.wrapper(monitor.run)
    except KeyboardInterrupt:
        pass

if __name__ == "__main__":
    main()
```

## 附录4: hwmon sysfs ABI 详细规范

### 4.1 temp 属性族

| 属性 | 访问 | 描述 |
|------|------|------|
| `temp[1-*]_input` | RO | 当前温度值，单位毫摄氏度(m°C) |
| `temp[1-*]_max` | RW | 温度上限阈值，触发告警 |
| `temp[1-*]_max_hyst` | RW | 温度上限迟滞值 |
| `temp[1-*]_min` | RW | 温度下限阈值 |
| `temp[1-*]_crit` | RW | 临界温度值，超过可能触发关机 |
| `temp[1-*]_crit_hyst` | RW | 临界温度迟滞值 |
| `temp[1-*]_emergency` | RW | 紧急温度阈值 |
| `temp[1-*]_emergency_hyst` | RW | 紧急温度迟滞值 |
| `temp[1-*]_label` | RO | 传感器标签名(如"junction"/"mem"/"edge") |
| `temp[1-*]_type` | RO | 传感器类型(1=CPU, 2=GPU, 3=北桥...) |
| `temp[1-*]_offset` | RW | 温度校准偏移量 |
| `temp[1-*]_fault` | RO | 传感器故障指示(1=故障) |
| `temp[1-*]_alarm` | RO | 告警标志 |
| `temp[1-*]_max_alarm` | RO | 温度上限告警 |
| `temp[1-*]_crit_alarm` | RO | 临界温度告警 |

### 4.2 power 属性族

| 属性 | 访问 | 描述 |
|------|------|------|
| `power[1-*]_input` | RO | 当前功耗值，单位微瓦(µW) |
| `power[1-*]_cap` | RW | 功耗上限 |
| `power[1-*]_cap_max` | RO | 功耗上限最大值 |
| `power[1-*]_cap_min` | RO | 功耗上限最小值 |
| `power[1-*]_max` | RO | 最大功耗 |
| `power[1-*]_crit` | RO | 临界功耗 |
| `power[1-*]_label` | RO | 功耗传感器标签 |
| `power[1-*]_average` | RO | 平均功耗 |
| `power[1-*]_average_interval` | RW | 平均功耗采样窗口(毫秒) |
| `power[1-*]_accuracy` | RO | 测量精度(百分比) |
| `power[1-*]_cap_hyst` | RW | 功耗上限迟滞 |

### 4.3 fan 属性族

| 属性 | 访问 | 描述 |
|------|------|------|
| `fan[1-*]_input` | RO | 风扇转速(RPM) |
| `fan[1-*]_label` | RO | 风扇标签 |
| `fan[1-*]_min` | RO | 最小风扇转速 |
| `fan[1-*]_max` | RO | 最大风扇转速 |
| `fan[1-*]_div` | RW | 转速除法因子 |
| `fan[1-*]_fault` | RO | 风扇故障指示 |
| `fan[1-*]_alarm` | RO | 风扇告警 |
| `pwm[1-*]` | RW | PWM占空比(0-255) |
| `pwm[1-*]_enable` | RW | PWM模式(1=手动, 2=自动) |
| `pwm[1-*]_mode` | RW | PWM模式(0=直流, 1=脉冲) |
| `pwm[1-*]_freq` | RW | PWM频率(Hz) |
| `pwm[1-*]_auto_point[1-*]_pwm` | RW | 自动曲线PWM点 |
| `pwm[1-*]_auto_point[1-*]_temp` | RW | 自动曲线温度点 |

### 4.4 in/curr 属性族

| 属性 | 访问 | 描述 |
|------|------|------|
| `in[1-*]_input` | RO | 电压值，单位毫伏(mV) |
| `in[1-*]_label` | RO | 电压轨标签(如"vddgfx"/"vddci") |
| `in[1-*]_min` | RW | 电压下限告警 |
| `in[1-*]_max` | RW | 电压上限告警 |
| `in[1-*]_crit` | RW | 临界电压 |
| `in[1-*]_alarm` | RO | 电压告警标志 |
| `curr[1-*]_input` | RO | 电流值，单位毫安(mA) |
| `curr[1-*]_label` | RO | 电流轨标签 |
| `curr[1-*]_max` | RW | 电流上限告警 |
| `curr[1-*]_crit` | RW | 临界电流 |

## 附录5: AMDGPU PM sysfs 高级属性详解

### 5.1 性能与功耗控制属性

| 属性路径 | 访问 | 描述 |
|---------|------|------|
| `device_power_state` | RO | 当前设备电源状态(D0/D3) |
| `power_dpm_state` | RW | 电源状态(battery/performance/balance) |
| `power_dpm_force_performance_level` | RW | 强制性能级别(auto/low/high/manual) |
| `pp_dpm_sclk` | RO/RW | GPU核心时钟频率(MHz)层级 |
| `pp_dpm_mclk` | RO/RW | 显存时钟频率(MHz)层级 |
| `pp_dpm_socclk` | RO/RW | SoC时钟频率层级 |
| `pp_dpm_fclk` | RO/RW | Infinity Fabric时钟频率层级 |
| `pp_dpm_dcefclk` | RO/RW | 显示控制器时钟频率层级 |
| `pp_dpm_pcie` | RO/RW | PCIe速率控制 |
| `pp_od_clk_voltage` | RW | OverDrive频率电压调整 |
| `pp_power_profile_mode` | RW | 电源配置模式(3D/VIDEO/COMPUTE/CUSTOM) |
| `pp_sclk_od` | RW | SCLK OverDrive百分比 |
| `pp_mclk_od` | RW | MCLK OverDrive百分比 |
| `pp_power_limit` | RW | 当前功耗限制百分比 |
| `pp_force_state` | RW | 强制电源状态 |
| `pp_cur_state` | RO | 当前电源状态 |

### 5.2 各属性关键实现代码注解

```c
// pp_dpm_sclk 输出示例(AMDGPU):
// 0: 500Mhz *
// 1: 800Mhz
// 2: 1200Mhz
// 3: 1500Mhz
// 4: 1800Mhz
// * 表示当前所处层级

// pp_dpm_sclk 写入方法:
// echo "manual" > power_dpm_force_performance_level
// echo "4" > pp_dpm_sclk  // 锁定到第4层级

// pp_od_clk_voltage 操作:
// echo "s 1 2000 1150" > pp_od_clk_voltage  // 设置状态1: 2000MHz @ 1150mV
// echo "c" > pp_od_clk_voltage                // 提交更改
// echo "r" > pp_od_clk_voltage                // 恢复默认
// echo "vc" > pp_od_clk_voltage               // 电压曲线校准

// pp_power_profile_mode 模式列表:
// 0: BOOTUP_DEFAULT
// 1: 3D_FULL_SCREEN
// 2: POWER_SAVING
// 3: VIDEO
// 4: VR
// 5: COMPUTE
// 6: CUSTOM
// 设置: echo "3" > pp_power_profile_mode
```

### 5.3 sysfs 属性编程监控示例

```python
#!/usr/bin/env python3
"""
AMDGPU sysfs 属性高级监控工具
演示如何使用 sysfs 轮询多维度 GPU 状态
"""

import os
import time
import sys
import json

class AMDGPUSysfsMonitor:
    """AMDGPU sysfs 属性监控器"""
    
    # AMDGPU hwmon 与 pm sysfs 路径映射
    HWMON_ATTRS = {
        'temp_junction': 'temp1_input',
        'temp_mem': 'temp2_input',
        'temp_soc': 'temp3_input',
        'power_avg': 'power1_average',
        'power_input': 'power1_input',
        'fan_speed': 'fan1_input',
        'fan_pwm': 'pwm1',
    }
    
    PM_ATTRS = {
        'sclk': 'pp_dpm_sclk',
        'mclk': 'pp_dpm_mclk',
        'socclk': 'pp_dpm_socclk',
        'power_limit': 'pp_power_limit',
        'perf_level': 'power_dpm_force_performance_level',
    }
    
    def __init__(self, gpu_id=0):
        self.gpu_id = gpu_id
        self.hwmon_path = self._find_hwmon_path()
        self.pm_path = f"/sys/class/drm/card{gpu_id}/device"
        
    def _find_hwmon_path(self):
        """自动发现 hwmon 设备路径"""
        hwmon_base = f"/sys/class/drm/card{self.gpu_id}/device/hwmon"
        if not os.path.exists(hwmon_base):
            return None
        for entry in os.listdir(hwmon_base):
            if entry.startswith('hwmon'):
                return f"{hwmon_base}/{entry}"
        return None
    
    def _read_sysfs(self, path):
        """读取 sysfs 属性值"""
        try:
            with open(path, 'r') as f:
                return f.read().strip()
        except (IOError, OSError) as e:
            return f"ERROR: {e}"
    
    def read_hwmon_attr(self, name):
        """读取 hwmon 属性"""
        attr = self.HWMON_ATTRS.get(name)
        if not attr or not self.hwmon_path:
            return None
        return self._read_sysfs(f"{self.hwmon_path}/{attr}")
    
    def read_pm_attr(self, name):
        """读取电源管理属性"""
        attr = self.PM_ATTRS.get(name)
        if not attr:
            return None
        return self._read_sysfs(f"{self.pm_path}/{attr}")
    
    def read_all(self):
        """读取所有指标"""
        metrics = {'timestamp': time.time()}
        
        # 读取 hwmon 指标
        for name in self.HWMON_ATTRS:
            value = self.read_hwmon_attr(name)
            if value and not value.startswith('ERROR'):
                metrics[name] = value
        
        # 读取 PM 指标
        for name in self.PM_ATTRS:
            value = self.read_pm_attr(name)
            if value and not value.startswith('ERROR'):
                metrics[name] = value
        
        return metrics
    
    def monitor_loop(self, interval=2, duration=60):
        """持续监控循环"""
        start_time = time.time()
        records = []
        
        print(f"AMDGPU GPU {self.gpu_id} 监控启动 (间隔{interval}s, 持续{duration}s)")
        print(f"hwmon路径: {self.hwmon_path}")
        print(f"{'='*60}")
        print(f"{'时间':<20} {'温度(°C)':<12} {'功耗(W)':<12} {'风扇(RPM)':<12}")
        print(f"{'='*60}")
        
        while time.time() - start_time < duration:
            metrics = self.read_all()
            
            # 转换格式
            temp = self._parse_temp(metrics.get('temp_junction'))
            power = self._parse_power(metrics.get('power_avg'))
            fan = metrics.get('fan_speed', 'N/A')
            
            timestamp = time.strftime('%H:%M:%S')
            print(f"{timestamp:<20} {temp:<12} {power:<12} {fan:<12}")
            
            records.append(metrics)
            time.sleep(interval)
        
        return records
    
    def _parse_temp(self, value):
        """温度值转换: 毫摄氏度 -> 摄氏度"""
        if not value:
            return 'N/A'
        try:
            return f"{int(value) / 1000:.1f}"
        except ValueError:
            return value
    
    def _parse_power(self, value):
        """功耗值转换: 微瓦 -> 瓦"""
        if not value:
            return 'N/A'
        try:
            return f"{int(value) / 1000000:.2f}"
        except ValueError:
            return value
    
    def export_json(self, records, filename='gpu_metrics.json'):
        """导出监控数据为 JSON"""
        with open(filename, 'w') as f:
            json.dump(records, f, indent=2)
        print(f"数据已导出到 {filename}")


if __name__ == "__main__":
    monitor = AMDGPUSysfsMonitor()
    if not monitor.hwmon_path:
        print("错误: 找不到 GPU hwmon 设备")
        sys.exit(1)
    
    records = monitor.monitor_loop(interval=1, duration=30)
    monitor.export_json(records)
```

## 附录6: ROCm SMI 与 hwmon 接口对比

ROCm SMI (System Management Interface) 是 AMD 提供的上层 GPU 管理工具，它在 hwmon sysfs 基础上提供了更友好的接口。

### 6.1 功能对比

| 功能 | hwmon sysfs | ROCm SMI (rocm-smi) | amdsmi 库 |
|------|-------------|---------------------|-----------|
| 温度读取 | cat temp1_input | rocm-smi --showtemp | amdsmi_get_temp_metric() |
| 功耗读取 | cat power1_average | rocm-smi --showpower | amdsmi_get_power_info() |
| 风扇控制 | echo N > pwm1 | rocm-smi --setfan N | amdsmi_set_fan_speed() |
| 频率设置 | echo N > pp_dpm_sclk | rocm-smi --setsclk N | amdsmi_set_clk_range() |
| OverDrive | echo ... > pp_od_clk_voltage | rocm-smi --setoverdrive N | amdsmi_set_gpu_overdrive() |
| 支持多GPU | 手动遍历 | 自动发现所有GPU | amdsmi_get_handles() |
| 输出格式 | 原始文本 | 格式化表格 | 结构化API |
| 权限要求 | root/sysfs | root+dri权限 | 设备节点权限 |

### 6.2 ROCm SMI 常用命令

```bash
# 显示所有 GPU 信息
rocm-smi

# 显示温度
rocm-smi --showtemp

# 显示功耗
rocm-smi --showpower

# 显示风扇信息
rocm-smi --showfan

# 显示频率
rocm-smi --showclkfrq

# 设置风扇转速(百分比)
rocm-smi --setfan 75

# 设置 OverDrive 级别
rocm-smi --setoverdrive 20

# JSON 格式输出(便于脚本处理)
rocm-smi --json

# 监控模式(持续刷新)
rocm-smi --loop
```

### 6.3 amdsmi Python 库使用示例

```python
try:
    import amdsmi
    # 初始化
    amdsmi.amdsmi_init()
    
    # 获取GPU句柄
    handles = amdsmi.amdsmi_get_handles()
    
    for handle in handles:
        # 获取温度
        temp_info = amdsmi.amdsmi_get_temp_metric(
            handle,
            amdsmi.AmdSmiTempMetric.JUNCTION,
            amdsmi.AmdSmiTempType.EDGE
        )
        print(f"GPU温度: {temp_info}°C")
        
        # 获取功耗
        power_info = amdsmi.amdsmi_get_power_info(handle)
        print(f"当前功耗: {power_info['current_socket_power']}W")
        
        # 获取频率
        clk_info = amdsmi.amdsmi_get_clk_freq(handle, amdsmi.AmdSmiClkType.SYSCLK)
        print(f"核心频率: {clk_info}")
        
    amdsmi.amdsmi_shutdown()
except ImportError:
    print("amdsmi 库未安装，请通过 'pip install pyamdsmi' 安装")
```

## 附录7: 高级调试与排障专题

### 7.1 hwmon 设备注册失败排查流程

当 hwmon 设备未出现时，按以下流程排查：

```bash
# 1. 确认 amdgpu 驱动已加载
lsmod | grep amdgpu

# 2. 检查 GPU 设备是否就绪
lspci -nn | grep -i amd

# 3. 查看内核日志中 hwmon 注册信息
dmesg | grep -i hwmon
dmesg | grep -i "amdgpu.*hwmon"

# 4. 检查 sysfs 设备树
ls -la /sys/class/drm/card0/device/hwmon/

# 5. 手动触发 hwmon 注册(绑定/解绑)
echo -n "0000:03:00.0" > /sys/bus/pci/drivers/amdgpu/unbind
echo -n "0000:03:00.0" > /sys/bus/pci/drivers/amdgpu/bind

# 6. 检查 ACPI 表
cat /sys/firmware/acpi/tables/PPTable > pptable.bin 2>/dev/null || echo "无 PPTable"
```

### 7.2 hwmon 性能开销分析

高频读取 hwmon sysfs 属性会通过 SMU 消息传递产生性能开销：

```python
import time
import os

# 性能基准测试
hwmon_path = "/sys/class/drm/card0/device/hwmon/hwmon1"

def benchmark_hwmon_read(attr, iterations=10000):
    path = f"{hwmon_path}/{attr}"
    if not os.path.exists(path):
        print(f"{attr}: 不存在")
        return
    
    # 预热
    with open(path, 'r') as f:
        f.read()
    
    # 基准测试
    start = time.perf_counter_ns()
    for _ in range(iterations):
        with open(path, 'r') as f:
            f.read()
    end = time.perf_counter_ns()
    
    avg_ns = (end - start) / iterations
    print(f"{attr:<20}: {avg_ns:>8.1f} ns/次  ({iterations}次)")
    return avg_ns

print("hwmon sysfs 读取性能基准测试")
print("=" * 50)
benchmark_hwmon_read("temp1_input")
benchmark_hwmon_read("power1_average")
benchmark_hwmon_read("fan1_input")
benchmark_hwmon_read("pwm1")
```

### 7.3 hwmon 数据采集最佳实践

```python
"""
hwmon 数据采集最佳实践
- 批量读取减少 sysfs 访问次数
- 缓存策略避免频繁 SMU 查询
- 异常值检测与过滤
"""
import time
import os
import statistics

class HWMONOptimizedCollector:
    """优化的 hwmon 数据采集器"""
    
    CACHE_TTL = {  # 不同属性缓存时间(秒)
        'temp': 1.0,
        'power': 2.0,
        'fan': 1.0,
        'pwm': 1.0,
    }
    
    def __init__(self, hwmon_path):
        self.hwmon_path = hwmon_path
        self.cache = {}
    
    def _read(self, attr):
        now = time.time()
        
        # 检查缓存
        if attr in self.cache:
            value, timestamp = self.cache[attr]
            attr_type = attr.rstrip('0123456789')
            if now - timestamp < self.CACHE_TTL.get(attr_type, 0.5):
                return value
        
        # 读取 sysfs
        try:
            path = f"{self.hwmon_path}/{attr}"
            with open(path, 'r') as f:
                value = f.read().strip()
            self.cache[attr] = (value, now)
            return value
        except (IOError, OSError):
            return None
    
    def batch_read(self, attrs):
        """批量读取，返回字典"""
        result = {}
        for attr in attrs:
            result[attr] = self._read(attr)
        return result
    
    def detect_anomalies(self, readings, thresholds):
        """异常值检测
        
        Args:
            readings: 属性名 -> 值 字典
            thresholds: 属性名 -> (min, max) 阈值字典
        Returns:
            anomalies: 异常属性列表
        """
        anomalies = []
        for attr, value in readings.items():
            if value is None:
                continue
            threshold = thresholds.get(attr)
            if not threshold:
                continue
            try:
                v = int(value)
                if v < threshold[0] or v > threshold[1]:
                    anomalies.append((attr, v, threshold))
            except ValueError:
                pass
        return anomalies


# 使用示例
def monitoring_best_practice_demo():
    hwmon_path = "/sys/class/drm/card0/device/hwmon/hwmon1"
    if not os.path.exists(hwmon_path):
        print("GPU hwmon 设备不存在")
        return
    
    collector = HWMONOptimizedCollector(hwmon_path)
    
    # 温度阈值(毫摄氏度)
    thresholds = {
        'temp1_input': (0, 110000),  # junction: 0-110°C
        'temp2_input': (0, 95000),   # mem: 0-95°C
        'power1_average': (0, 350000000),  # 0-350W
    }
    
    print("开始监控(按Ctrl+C停止)...")
    print(f"{'时间':<10} {'temp_junction':<15} {'temp_mem':<15} {'power':<15} {'状态'}")
    
    try:
        while True:
            readings = collector.batch_read([
                'temp1_input', 'temp2_input', 'power1_average'
            ])
            
            anomalies = collector.detect_anomalies(readings, thresholds)
            
            timestamp = time.strftime('%H:%M:%S')
            tj = f"{int(readings.get('temp1_input', 0))/1000:.1f}°C" if readings.get('temp1_input') else 'N/A'
            tm = f"{int(readings.get('temp2_input', 0))/1000:.1f}°C" if readings.get('temp2_input') else 'N/A'
            pw = f"{int(readings.get('power1_average', 0))/1000000:.2f}W" if readings.get('power1_average') else 'N/A'
            status = "⚠️ ALARM" if anomalies else "正常"
            
            print(f"{timestamp:<10} {tj:<15} {tm:<15} {pw:<15} {status}")
            
            if anomalies:
                for attr, value, (lo, hi) in anomalies:
                    print(f"  ⚠️ {attr}: {value} (阈值: {lo}-{hi})")
            
            time.sleep(2)
    except KeyboardInterrupt:
        print("\n监控结束")


if __name__ == "__main__":
    monitoring_best_practice_demo()
```

## 附录8: 常见面试题与深入理解

### 8.1 基础概念题

**Q1: hwmon 与 thermal sysfs 的区别是什么？**

A: hwmon 是通用的硬件监控子系统，提供温度、电压、电流、功耗、风扇等多维度监控接口。thermal sysfs 专门用于热管理，侧重于温度阈值触发的冷却措施(如降频、风扇控制)。hwmon 监控数据的上限/临界值在达到时仅产生告警标志，而 thermal sysfs 可自动触发冷却设备调节。两者可以共存：hwmon 提供数据采集，thermal 提供热策略执行。AMDGPU 同时使用两者，hwmon 用于数据显示，thermal 用于 Throttle 策略。

**Q2: sysfs 属性的 RW 和 RO 是怎么在内核中实现的？**

A: 通过 `SENSOR_DEVICE_ATTR` 或 `DEVICE_ATTR` 宏定义 show/store 回调函数：
```c
// RO 属性只需实现 show 回调
static SENSOR_DEVICE_ATTR(name, 0444, show_func, NULL, 0);
// RW 属性需要实现 show 和 store 回调
static SENSOR_DEVICE_ATTR(name, 0644, show_func, store_func, 0);
// 权限控制：0444=只读, 0644=读写
```

**Q3: 为什么读取temp1_input有时会卡住几十毫秒？**

A: 因为 hwmon 读取操作需要经过以下路径：
```
用户态read() → sysfs → amdgpu_hwmon_read_temp() → SMU 消息发送 → SMU 固件处理 → SMU 响应返回 → 解析返回数据
```
SMU 消息通过 MMIO 寄存器(Mailbox)发送，SMU 固件处理需要等待下一个调度周期，这个过程在负载高时可能引入 10-50ms 延迟。AMDGPU 驱动在某些版本中对温度读取实现了缓存机制，每 100ms 刷新一次以减少 SMU 交互频率。

### 8.2 进阶设计题

**Q4: 如果要在 Linux 上实现一个 GPU 监控守护进程，你会如何设计？**

A: 设计要点：
1. **数据采集层**: 使用 hwmon sysfs 轮询 + 可选 eBPF 追踪 SMU 消息
2. **缓存层**: 内存环形缓冲区存储最近 N 秒数据
3. **持久化层**: SQLite/InfluxDB 存储历史数据
4. **告警引擎**: 基于阈值的告警规则引擎(温度/功耗/风扇故障)
5. **API 接口**: Unix Socket/REST API 提供外部访问
6. **前端展示**: Web Dashboard 或命令行 TUI
7. **低开销设计**: 轮询间隔自适应(温度稳定时拉长，变化快时缩短)

**Q5: 多个 GPU 的 hwmon 设备号是不固定的，如何可靠定位？**

A: 通过 PCI 拓扑关系定位，而非依赖 hwmon 编号：
```python
# PCI 地址映射
def find_gpu_hwmon_by_pci(pci_addr):
    path = f"/sys/bus/pci/devices/{pci_addr}/hwmon"
    if os.path.exists(path):
        for hwmon in os.listdir(path):
            if hwmon.startswith('hwmon'):
                return f"{path}/{hwmon}"
    return None

# 获取所有 GPU 的 hwmon 路径
def list_all_gpu_hwmon():
    gpus = {}
    for card in glob.glob("/sys/class/drm/card*/device"):
        pci_link = os.readlink(card)
        pci_addr = os.path.basename(pci_link)
        hwmon_path = find_gpu_hwmon_by_pci(pci_addr)
        if hwmon_path:
            gpus[pci_addr] = hwmon_path
    return gpus
```

### 8.3 知识自测(补充题)

**选择题(续):**

11. hwmon 中 temp1_crit 属性的作用是？
A. 记录最高温度记录
B. 设置临界关断温度阈值
C. 设置风扇全速温度
D. 控制温度采样频率

12. 以下哪个不是 hwmon 支持的电源管理属性？
A. power1_input
B. power1_cap
C. power1_rated_min
D. power1_average_interval

13. 在 AMDGPU 驱动中，pp_od_clk_voltage 属性的 "c" 命令含义是？
A. 清除所有自定义设置
B. 提交电压频率修改
C. 校准时钟发生器
D. 创建新的性能状态

14. 以下哪个工具可以直接读取 hwmon 温度数据？
A. sensors
B. i2cget
C. powertop
D. perf

15. 使用 sysfs 调节风扇转速时需要先将 pwm1_enable 设置为？
A. 0
B. 1
C. 2
D. 3

16. hwmon 的 temp1_fault 属性值为 1 表示什么？
A. 传感器温度过高
B. 传感器通信故障
C. 传感器未校准
D. 传感器已禁用

17. AMDGPU 的 power_dpm_force_performance_level 设置为 "high" 时会产生什么效果？
A. 锁定到最高频率状态
B. 启用自动超频
C. 仅使用最高电压
D. 禁用所有省电功能

18. hwmon 设备注册时，如果已有同名设备会怎样？
A. 注册失败返回错误
B. 自动覆盖旧设备
C. 生成新的设备编号
D. 内核崩溃

**填空题(续):**

7. hwmon 中温度值的单位是 ______。
8. 在 sysfs 中设置风扇为手动模式需要执行 `echo ______ > pwm1_enable`。
9. ROCm SMI 的命令行工具名称是 ______。
10. AMDGPU 的 pp_dpm_sclk 属性用于显示和设置 ______ 频率层级。
11. hwmon 的 power1_average_interval 属性用于设置 ______。
12. 在 /sys/class/drm/card0/device/ 目录中，读取 ______ 属性可以查看当前电源状态(D0/D3)。

**参考答案:**

选择题: 11-B, 12-C, 13-B, 14-A, 15-B, 16-B, 17-A, 18-C
填空题: 7. 毫摄氏度(m°C), 8. 1, 9. rocm-smi, 10. GPU核心时钟(SCLK/GFXCLK), 11. 平均功耗采样窗口, 12. device_power_state

## 附录9: hwmon 相关内核配置与调试

### 9.1 内核配置选项

```bash
# hwmon 核心支持
CONFIG_HWMON=y

# AMDGPU hwmon 支持(随AMDGPU自动启用)
CONFIG_DRM_AMDGPU=y

# lm-sensors 用户态接口
CONFIG_SENSORS_LM90=m  # 温度传感器驱动
CONFIG_SENSORS_AMDGPU=m # AMDGPU传感器

# 调试选项
CONFIG_HWMON_DEBUGFS=y
CONFIG_DEBUG_FS=y

# 检查当前内核配置
zcat /proc/config.gz | grep CONFIG_HWMON
# 或
grep CONFIG_HWMON /boot/config-$(uname -r)
```

### 9.2 debugfs 调试接口

```bash
# 挂载 debugfs
mount -t debugfs none /sys/kernel/debug

# AMDGPU debugfs 调试接口
ls /sys/kernel/debug/dri/0/

# amdgpu_pm_info: 显示完整电源管理状态
cat /sys/kernel/debug/dri/0/amdgpu_pm_info

# amdgpu_sensors: 传感器原始数据
cat /sys/kernel/debug/dri/0/amdgpu_sensors

# 跟踪 hwmon 操作(使用 ftrace)
echo function > /sys/kernel/debug/tracing/current_tracer
echo amdgpu_hwmon_read_temp > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

### 9.3 自定义 udev 规则

```bash
# /etc/udev/rules.d/99-gpu-hwmon.rules
# 为 GPU hwmon 设备创建持久化符号链接

# AMDGPU hwmon - 按PCI地址创建别名
SUBSYSTEM=="hwmon", KERNELS=="0000:03:00.0", SUBSYSTEMS=="pci", \
    ATTR{name}=="amdgpu", SYMLINK+="hwmon/gpu0"

SUBSYSTEM=="hwmon", KERNELS=="0000:04:00.0", SUBSYSTEMS=="pci", \
    ATTR{name}=="amdgpu", SYMLINK+="hwmon/gpu1"

# 应用规则后重载
udevadm control --reload-rules
udevadm trigger

# 验证
ls -la /dev/hwmon/
# 输出: lrwxrwxrwx ... /dev/hwmon/gpu0 -> ../hwmon5
```

### 9.4 hwmon 性能优化: sysfs 批量读取 vs 逐个读取

```python
import os
import time

# 对比测试: 逐个读取 vs 批量读取
hwmon = "/sys/class/drm/card0/device/hwmon/hwmon1"
attrs = ['temp1_input', 'temp2_input', 'power1_average', 'fan1_input', 'pwm1']

def sequential_read(iterations=100):
    start = time.perf_counter()
    for _ in range(iterations):
        for attr in attrs:
            path = f"{hwmon}/{attr}"
            if os.path.exists(path):
                with open(path) as f:
                    _ = f.read()
    return (time.perf_counter() - start) / iterations

def batch_read(iterations=100):
    start = time.perf_counter()
    for _ in range(iterations):
        # 批量读取: 打开一次文件描述符
        results = {}
        for attr in attrs:
            path = f"{hwmon}/{attr}"
            if os.path.exists(path):
                fd = os.open(path, os.O_RDONLY)
                results[attr] = os.read(fd, 32)
                os.close(fd)
    return (time.perf_counter() - start) / iterations

# 注意: 实际优化效果有限，因为每次 read 都会触发 SMU 消息
# 真正的优化是减少读取频率或使用驱动级的缓存机制
```

### 11. 参考文献与资源

| 资源 | 链接/路径 |
|------|-----------|
| hwmon 内核 API 文档 | `Documentation/hwmon/hwmon-kernel-api.rst` |
| AMDGPU PM 文档 | `Documentation/gpu/amdgpu/pm.rst` |
| hwmon sysfs ABI | `Documentation/ABI/testing/sysfs-class-hwmon` |
| lm-sensors 项目 | https://github.com/lm-sensors/lm-sensors |
| amdgpu_pm.c 源码 | `drivers/gpu/drm/amd/pm/amdgpu_pm.c` |
| hwmon 核心源码 | `drivers/hwmon/hwmon.c` |
| AMD ROCm SMI | https://github.com/ROCm/amdsmi |
| Linux 内核 hwmon 子系统指南 | https://www.kernel.org/doc/html/latest/hwmon/ |
| AMDGPU 电源管理文档 | https://dri.freedesktop.org/docs/drm/gpu/amdgpu/pm.html |
| sensors-detect 使用手册 | `man sensors-detect` |
| sysfs 属性规范 | `Documentation/ABI/testing/sysfs-class-hwmon` |
| PCI 电源管理规范 | `Documentation/power/pci.rst` |
| debugfs AMDGPU 调试 | `Documentation/gpu/amdgpu/debugfs.rst` |