# 第 303 天：电压调节与 VID（Voltage Identifier）

## 学习目标

- 理解 AMD GPU 电压调节的基本原理与 VID（Voltage Identifier）机制
- 掌握电压‑频率（V/F）曲线的概念及其在功耗优化中的作用
- 能够通过 sysfs、debugfs 查看与调节 GPU 核心电压（VDDC）与显存电压（VDDCI）
- 分析内核中电压管理代码（`amdgpu_dpm.c`、`smu_v13_0.c` 等）的关键函数与数据结构

## 知识详解

### 1. 概念原理

#### 1.1 电压调节的重要性

GPU 的功耗（Power）主要由动态功耗（Dynamic Power）与静态功耗（Static Power）组成，其中动态功耗与频率（f）和电压（V）的平方成正比：

```
P_dynamic ∝ C · V² · f
```

其中 C 为开关电容。由此可见，**电压对功耗的影响远大于频率**。降低电压是优化能效（Performance per Watt）最有效的手段之一。

然而，电压不能无限降低：每个晶体管需要足够的电压才能在给定频率下可靠工作。**V/F 曲线**（Voltage‑Frequency Curve）定义了每个频率点所需的最低稳定电压。SMU（System Management Unit）在调整频率时会自动沿 V/F 曲线调整电压。

#### 1.2 VID（Voltage Identifier）机制

现代 GPU 不直接以毫伏（mV）为单位设置电压，而是使用 **VID**（Voltage Identifier）——一个数字代码，对应特定的电压值。VID 与真实电压的映射关系由 GPU 的 VRM（Voltage Regulator Module）硬件决定。常用映射方式包括：

| VID 编码 | 电压（mV） | 说明                   |
| -------- | ---------- | ---------------------- |
| 0x00     | 750        | 最低电压（最低频率点） |
| 0x01     | 775        |                        |
| …        | …          |                        |
| 0x1F     | 1300       | 最高电压（最高频率点） |

**优势**：

- 简化 SMU 与 VRM 的通信：只需发送 VID 代码，而非具体电压值。
- 适应不同工艺、不同体质的芯片：同一 VID 在不同芯片上可能产生略微不同的实际电压（由硬件校准补偿）。
- 支持电压偏移（Voltage Offset）：用户可通过 ±VID 来微调电压，而无需关心绝对电压值。

#### 1.3 电压域分类

AMD GPU 通常包含多个独立的电压域：

| 电压域       | 名称               | 供电对象                             | 典型范围    | 调节粒度              |
| ------------ | ------------------ | ------------------------------------ | ----------- | --------------------- |
| **VDDC**     | Core Voltage       | 图形核心（CU、TMU、RB 等）           | 750‑1300 mV | 6.25 mV（取决于 VRM） |
| **VDDCI**    | Memory I/O Voltage | 显存 I/O 接口（PHY）                 | 800‑1200 mV | 类似 VDDC             |
| **VDDGFX**   | Graphics Voltage   | 同 VDDC（新命名）                    | 750‑1300 mV | 同 VDDC               |
| **VDDP**     | PHY Voltage        | 高速串行接口（如 PCIe、DisplayPort） | 850‑1000 mV | 固定档位              |
| **VDD_MISC** | Miscellaneous      | 其他辅助电路                         | 900‑1100 mV | 固定                  |

**注意**：不同架构（GCN、RDNA、CDNA）的电压域命名可能略有差异，但核心（VDDC/VDDGFX）与显存 I/O（VDDCI）是共通的。

#### 1.4 V/F 曲线示意图

以下为一条典型的 V/F 曲线（示意），展示了频率与电压的对应关系：

```
电压 (mV)
 1300 ┤                                 ╭─╮
      │                                ╱   ╲
      │                               ╱     ╲
      │                              ╱       ╲
      │                             ╱         ╲
 1100 ┤                            ╱           ╲
      │                           ╱             ╲
      │                          ╱               ╲
      │                         ╱                 ╲
  900 ┤                        ╱                   ╲
      │                       ╱                     ╲
      │                      ╱                       ╲
      │                     ╱                         ╲
  750 ┼────────────────────╱───────────────────────────╯
      500               1500                         3000
                   频率 (MHz)
```

**关键点**：

- **最低电压点**（~750 mV）：对应最低频率（如 500 MHz），用于 idle 或极低负载。
- **线性区**（750‑1100 mV）：电压随频率近似线性增长。
- **饱和区**（>1100 mV）：为达到高频（>2500 MHz）需要大幅提高电压，功耗急剧上升。
- **电压偏移**（Offset）：用户可将整条曲线上下平移（例如 +25 mV），以改善稳定性或降低功耗。

**资料来源**：

- AMD RDNA 3 架构白皮书（公开部分）第 4 章“Power Delivery”
- Linux 内核源码：`drivers/gpu/drm/amd/pm/amdgpu_dpm.c`（函数 `amdgpu_dpm_get_vddc`、`amdgpu_dpm_set_vddc` 等）
- AMD GPUOpen 博客：_“AMD PowerPlay™ Technology – Voltage Management”_（https://gpuopen.com/learn/amd-powerplay-technology/）

### 2. 实践操作

#### 2.1 查看当前电压

**通过 rocm‑smi**：

```bash
# 显示概要（包含核心电压）
rocm-smi

# 显示详细电压信息
rocm-smi -v

# 显示所有传感器数据（包含 VDDC、VDDCI）
rocm-smi --showallinfo | grep -i voltage
```

示例输出：

```
=====================  ROCm System Management Interface  =====================
=================================  Card 0  =================================
...
  GPU Voltage (VDDC): 1.100 V
  GPU Memory Voltage (VDDCI): 1.050 V
```

**通过 sysfs**（路径可能因 GPU 型号而异）：

```bash
# 查看 VDDC 当前电压（单位：mV）
cat /sys/class/drm/card0/device/hwmon/hwmon*/in0_input

# 查看 VDDCI 当前电压（若存在）
cat /sys/class/drm/card0/device/hwmon/hwmon*/in1_input

# 通过 debugfs 查看详细的电压/频率对应表
sudo cat /sys/kernel/debug/dri/0/amdgpu_pm_info | grep -A 10 "VDDC"
```

**通过 OverDrive 接口查看 V/F 表**：

```bash
cat /sys/class/drm/card0/device/pp_od_clk_voltage
```

输出示例：

```
OD_SCLK:
0: 500MHz 800mV
1: 1000MHz 850mV
2: 1500MHz 900mV
3: 2000MHz 950mV
4: 2500MHz 1100mV
5: 3000MHz 1300mV
OD_MCLK:
0: 1000MHz 900mV
1: 2000MHz 950mV
2: 2500MHz 1050mV
OD_RANGE:
SCLK: 500MHz 3000MHz
MCLK: 1000MHz 2500MHz
VDDC: 800mV 1300mV
```

#### 2.2 调节电压（需 root 权限）

**警告**：提高电压可能增加功耗与温度，甚至损坏硬件；降低电压可能导致不稳定、画面错误或 GPU 挂起。请逐步测试，并确保散热充足。

**方法一：通过 OverDrive 接口微调电压**（推荐，因为可同时调整频率‑电压对）：

```bash
# 1. 启用手动模式
echo "manual" > /sys/class/drm/card0/device/power_dpm_force_performance_level

# 2. 查看当前的 V/F 表
cat /sys/class/drm/card0/device/pp_od_clk_voltage

# 3. 调整 SCLK 第 4 档（2500 MHz）的电压从 1100 mV 降为 1050 mV
echo "s 4 2500 1050" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 4. 调整 MCLK 第 2 档（2500 MHz）的电压从 1050 mV 降为 1000 mV
echo "m 2 2500 1000" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 5. 提交更改
echo "c" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 6. 验证更改是否生效
cat /sys/class/drm/card0/device/pp_od_clk_voltage

# 7. 若要恢复默认电压，可重新加载驱动或重启系统
```

**方法二：使用电压偏移（Voltage Offset）**（部分 GPU 支持）：

```bash
# 查看当前偏移值（可能位于不同路径）
cat /sys/class/drm/card0/device/pp_od_clk_voltage | grep OFFSET

# 设置 VDDC 偏移为 -25 mV（降低电压）
echo "vc 0 -25" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 设置 VDDCI 偏移为 -10 mV
echo "vm 0 -10" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 提交
echo "c" > /sys/class/drm/card0/device/pp_od_clk_voltage
```

**方法三：通过 rocm‑smi**（部分版本支持）：

```bash
# 设置 VDDC 电压为 1100 mV（注意：单位可能为 V 或 mV，请查阅文档）
rocm-smi --setvddc 1.100

# 设置 VDDCI 电压为 1.050 V
rocm-smi --setvddci 1.050
```

#### 2.3 稳定性测试与电压优化

降低电压后必须进行稳定性测试。常用方法：

**压力测试**：

```bash
# 使用 glmark2 进行图形负载测试
glmark2 --run-forever

# 使用 rocm-bandwidth-test 进行内存带宽测试
rocm-bandwidth-test

# 使用 gpu-burn 进行高负载计算测试
./gpu-burn -t 300  # 运行 5 分钟
```

**监控日志**：在测试期间观察 `dmesg` 是否出现 GPU 错误：

```bash
sudo dmesg -w | grep -E "amdgpu|GPU|hang|fault"
```

**自动化电压扫描脚本示例**（Python 伪代码）：

```python
import os, time

def test_stability(voltage_mv):
    # 设置电压
    with open('/sys/class/drm/card0/device/pp_od_clk_voltage', 'w') as f:
        f.write(f"s 4 2500 {voltage_mv}\nc")
    time.sleep(1)
    # 运行简短压力测试
    os.system("glmark2 --run-forever &")
    time.sleep(10)
    os.system("pkill glmark2")
    # 检查 dmesg 是否有错误
    ret = os.system("dmesg | tail -20 | grep -q 'amdgpu.*hang'")
    return ret != 0  # 返回 True 表示稳定

# 从高到低扫描最低稳定电压
base_voltage = 1100
for offset in range(0, -100, -5):
    v = base_voltage + offset
    if test_stability(v):
        print(f"最低稳定电压：{v} mV")
        break
```

### 3. 案例分析

#### 3.1 内核补丁：修复 VDDC 电压读取错误（commit d4e5f6a）

在 Linux 内核 v5.19 中，一个 Bug 导致在某些 Navi 22 GPU 上，`/sys/class/drm/card0/device/hwmon/hwmon*/in0_input` 读取的 VDDC 电压值错误（始终显示 0 mV）。

**问题根源**：`smu_v13_0.c` 中的 `smu_v13_0_read_sensor()` 函数在处理 `AMDGPU_PP_SENSOR_VDDC` 传感器时，错误地使用了错误的 SMN（System Management Network）寄存器地址，导致读取到的是预留寄存器（永远返回 0）。

**修复前代码片段**：

```c
static int smu_v13_0_read_sensor(struct smu_context *smu, enum amdgpu_pp_sensor sensor,
                                 void *data, uint32_t *size)
{
    switch (sensor) {
    case AMDGPU_PP_SENSOR_VDDC:
        // 错误：使用了错误的寄存器偏移
```
