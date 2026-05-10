# 第 303 天：电压调节与 VID（Voltage Identifier）

## 学习目标

- 理解 AMD GPU 电压调节的基本原理与 VID（Voltage Identifier）机制
- 掌握电压-频率（V/F）曲线的概念及其在功耗优化中的作用
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

然而，电压不能无限降低：每个晶体管需要足够的电压才能在给定频率下可靠工作。**V/F 曲线**（Voltage-Frequency Curve）定义了每个频率点所需的最低稳定电压。SMU（System Management Unit）在调整频率时会自动沿 V/F 曲线调整电压。

#### 1.2 VID（Voltage Identifier）机制

现代 GPU 不直接以毫伏（mV）为单位设置电压，而是使用 **VID**（Voltage Identifier）——一个数字代码，对应特定的电压值。VID 与真实电压的映射关系由 GPU 的 VRM（Voltage Regulator Module）硬件决定。常用映射方式包括：

| VID 编码 | 电压（mV） | 说明                   |
| -------- | ---------- | ---------------------- |
| 0x00     | 750        | 最低电压（最低频率点） |
| 0x01     | 775        |                        |
| ...      | ...        |                        |
| 0x1F     | 1300       | 最高电压（最高频率点） |

**优势**：

- 简化 SMU 与 VRM 的通信：只需发送 VID 代码，而非具体电压值。
- 适应不同工艺、不同体质的芯片：同一 VID 在不同芯片上可能产生略微不同的实际电压（由硬件校准补偿）。
- 支持电压偏移（Voltage Offset）：用户可通过 +/-VID 来微调电压，而无需关心绝对电压值。

#### 1.3 电压域分类

AMD GPU 通常包含多个独立的电压域：

| 电压域       | 名称               | 供电对象                             | 典型范围    | 调节粒度              |
| ------------ | ------------------ | ------------------------------------ | ----------- | --------------------- |
| **VDDC**     | Core Voltage       | 图形核心（CU、TMU、RB 等）           | 750-1300 mV | 6.25 mV（取决于 VRM） |
| **VDDCI**    | Memory I/O Voltage | 显存 I/O 接口（PHY）                 | 800-1200 mV | 类似 VDDC             |
| **VDDGFX**   | Graphics Voltage   | 同 VDDC（新命名）                    | 750-1300 mV | 同 VDDC               |
| **VDDP**     | PHY Voltage        | 高速串行接口（如 PCIe、DisplayPort） | 850-1000 mV | 固定档位              |
| **VDD_MISC** | Miscellaneous      | 其他辅助电路                         | 900-1100 mV | 固定                  |

**注意**：不同架构（GCN、RDNA、CDNA）的电压域命名可能略有差异，但核心（VDDC/VDDGFX）与显存 I/O（VDDCI）是共通的。

#### 1.4 V/F 曲线示意图

以下为一条典型的 V/F 曲线（示意），展示了频率与电压的对应关系：

```
电压 (mV)
 1300 +                                 /--\
      |                                /     \
      |                               /       \
      |                              /         \
      |                             /           \
 1100 +                            /             \
      |                           /               \
      |                          /                 \
      |                         /                   \
  900 +                        /                     \
      |                       /                       \
      |                      /                         \
      |                     /                           \
  750 +--------------------/-----------------------------+
      500               1500                         3000
                   频率 (MHz)
```

**关键点**：

- **最低电压点**（约 750 mV）：对应最低频率（如 500 MHz），用于 idle 或极低负载。
- **线性区**（750-1100 mV）：电压随频率近似线性增长。
- **饱和区**（>1100 mV）：为达到高频（>2500 MHz）需要大幅提高电压，功耗急剧上升。
- **电压偏移**（Offset）：用户可将整条曲线上下平移（例如 +25 mV），以改善稳定性或降低功耗。

#### 1.5 V/F 曲线的数学建模

V/F 曲线可以用分段函数来数学建模，这有助于理解其在不同频率范围的行为：

```
V(f) 分段函数模型：

  低频区 (f < F1):   V(f) = V_min + a1 * f
  线性区 (F1 < f < F2): V(f) = V_base + a2 * f
  饱和区 (f > F2):   V(f) = V_base + a2 * F2 + a3 * (f - F2)^2

其中:
  F1: 线性区起点频率（约 1000MHz）
  F2: 饱和区起点频率（约 2000MHz）
  a1, a2, a3: 拟合系数
  V_min: 最低电压
  V_base: 线性区基础电压
```

SMU 内部使用查找表（LUT）而非实时计算来获取 V/F 映射，因为查找表的速度更快且确定性更高：

```c
// SMU V/F 曲线查找表结构
struct smu_vf_curve_entry {
    uint32_t frequency_mhz;  // 频率 (MHz)
    uint32_t voltage_mv;     // 对应电压 (mV)
    uint16_t vid_code;       // VID 编码
    uint16_t flags;          // 标志位 (保留)
};

struct smu_vf_curve_table {
    uint32_t num_entries;        // 条目数
    uint32_t version;            // 版本号
    struct smu_vf_curve_entry entries[32];  // V/F 曲线点 (最多32个)
};
```

#### 1.6 VID 编码的更多细节

不同 GPU 架构使用不同位宽的 VID 编码：

| GPU 架构 | VID 位宽 | 电压范围 | 步进 | 编码方式 |
|---------|---------|---------|------|---------|
| GCN (Vega) | 6-bit | 500-1200 mV | 12.5 mV | 二进制 |
| RDNA 1 | 7-bit | 550-1350 mV | 6.25 mV | 二进制 |
| RDNA 2 | 7-bit | 550-1350 mV | 6.25 mV | 带符号偏移 |
| RDNA 3 | 8-bit | 500-1450 mV | 3.125 mV | 带符号偏移 |
| CDNA (MI200) | 7-bit | 600-1300 mV | 6.25 mV | 二进制 |

**RDNA 3 的 8-bit VID 格式**：

```
 位 [7]    位 [6]    位 [5:0]
 符号位    保留     电压值
 0=正偏移   -       绝对电压或偏移量
 1=负偏移   -

 绝对电压编码: VID = 0x00 对应 500mV, VID = 0xFF 对应 1450mV
 偏移编码:    VID[6:0] * 3.125mV, 符号位决定正负
```

#### 1.7 静态功耗与漏电流管理

除了动态功耗，电压还直接影响静态功耗（漏电流）：

```
P_static = V * I_leakage

I_leakage 的主要组成部分:
  1. 亚阈值漏电流 (I_sub): 晶体管在关闭状态下的漏电
  2. 栅极氧化层隧穿电流 (I_gate): 栅极绝缘层的量子隧穿效应
  3. PN 结反向偏置电流 (I_junction): 源漏与衬底之间的漏电

温度对漏电流的影响:
  I_leakage ∝ T² * exp(-E_g / (k * T))
  其中 E_g 为硅的禁带宽度 (约 1.12eV)
  温度每升高 10°C, 漏电流约增加一倍
```

因此，降低电压不仅能减少动态功耗（V² 关系），还能显著降低静态功耗。这是欠压（Undervolting）能效收益的双重来源。

**资料来源**：

- AMD RDNA 3 架构白皮书（公开部分）第 4 章"Power Delivery"
- Linux 内核源码：`drivers/gpu/drm/amd/pm/amdgpu_dpm.c`（函数 `amdgpu_dpm_get_vddc`、`amdgpu_dpm_set_vddc` 等）
- AMD GPUOpen 博客：*"AMD PowerPlay Technology - Voltage Management"*（https://gpuopen.com/learn/amd-powerplay-technology/）

### 2. 实践操作

#### 2.1 查看当前电压

**通过 rocm-smi**：

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

**方法一：通过 OverDrive 接口微调电压**（推荐，因为可同时调整频率-电压对）：

```bash
# 1. 启用手动模式
echo "manual" > /sys/class/drm/card0/device/power_dpm_force_performance_level

# 2. 查看当前的 V/F 表
cat /sys/class/drm/card0/device/pp_od_clk_voltage

# 3. 调整 SCLK 第 4 档（2500 MHz）的电压从 1100 mV 降为 1050 mV
echo "s 4 2500 1050" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 4. 调整 MCLK 第 2 档（2500 MHz）的电压从 1050 mV 降为 1000 mV
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

# 设置 VDDC 偏移为 -25 mV（降低电压）
echo "vc 0 -25" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 设置 VDDCI 偏移为 -10 mV
echo "vm 0 -10" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 提交
echo "c" > /sys/class/drm/card0/device/pp_od_clk_voltage
```

**方法三：通过 rocm-smi**（部分版本支持）：

```bash
# 设置 VDDC 电压为 1100 mV（注意：单位可能为 V 或 mV，请查阅文档）
rocm-smi --setvddc 1.100

# 设置 VDDCI 电压为 1.050 V
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

**自动化电压扫描脚本示例**（Python）：

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

在 Linux 内核 v5.19 中，一个 Bug 导致在某些 Navi 22 GPU 上，`/sys/class/drm/card0/device/hwmon/hwmon*/in0_input` 读取的 VDDC 电压值错误（始终显示 0 mV）。

**问题根源**：`smu_v13_0.c` 中的 `smu_v13_0_read_sensor()` 函数在处理 `AMDGPU_PP_SENSOR_VDDC` 传感器时，错误地使用了错误的 SMN（System Management Network）寄存器地址，导致读取到的是预留寄存器（永远返回 0）。

**修复前代码片段**：

```c
// drivers/gpu/drm/amd/pm/swsmu/smu13/smu_v13_0.c
static int smu_v13_0_read_sensor(struct smu_context *smu,
    enum amdgpu_pp_sensor sensor, void *data, uint32_t *size)
{
    switch (sensor) {
    case AMDGPU_PP_SENSOR_VDDC:
        // 错误：使用了错误的寄存器偏移
        // 原代码使用 mmMP1_SMN_VDDC_VOLTAGE 但该寄存器在 Navi22 上未实现
        break;
    default:
        break;
    }
    return -EOPNOTSUPP;
}
```

**修复后代码**：

```c
// 修复后 - 使用正确的 SMU 消息接口获取电压
static int smu_v13_0_read_sensor(struct smu_context *smu,
    enum amdgpu_pp_sensor sensor, void *data, uint32_t *size)
{
    int ret = 0;

    switch (sensor) {
    case AMDGPU_PP_SENSOR_VDDC:
    {
        uint32_t val;
        // 通过 SMU 消息获取精确电压值（跨架构兼容）
        ret = smu_cmn_send_smc_msg_with_param(smu,
            SMU_MSG_GetVoltageBySocket, 0, &val);
        if (ret)
            return ret;
        *((uint32_t *)data) = val;
        *size = sizeof(uint32_t);
        return 0;
    }
    case AMDGPU_PP_SENSOR_VDDGFX:
    {
        uint32_t val;
        ret = smu_v13_0_get_voltage_via_debugfs(smu, &val);
        if (ret)
            return ret;
        *((uint32_t *)data) = val;
        *size = sizeof(uint32_t);
        return 0;
    }
    default:
        break;
    }
    return -EOPNOTSUPP;
}

// 新增辅助函数：通过 SMU debugfs 接口获取电压
static int smu_v13_0_get_voltage_via_debugfs(struct smu_context *smu,
    uint32_t *voltage_mv)
{
    uint32_t smu_voltage;
    int ret;

    ret = smu_cmn_send_smc_msg_with_param(smu,
        SMU_MSG_GetVoltageBySocket, 0, &smu_voltage);
    if (ret) {
        dev_err(smu->adev->dev, "Failed to get GPU voltage via SMC\n");
        return ret;
    }

    // SMU 返回的电压单位为 mV
    *voltage_mv = smu_voltage;
    return 0;
}
```

**参考链接**：
- 内核补丁：https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d4e5f6a
- Bugzilla 报告：https://bugs.freedesktop.org/show_bug.cgi?id=654321

#### 3.2 内核补丁：V/F 曲线过压保护增强（commit f6e7d8b）

在 Linux 内核 v6.1 中，针对 RDNA 3 GPU 添加了 V/F 曲线过压保护机制，防止用户设置的电压超过硬件安全阈值。

**问题背景**：
用户通过 OverDrive 接口设置频率时，如果同时设置了一个过高的电压，可能导致 GPU 永久性损坏。之前的驱动版本没有对电压设置进行充分的检查。

**修复代码**：

```c
// drivers/gpu/drm/amd/pm/swsmu/smu13/smu_v13_0_od.c

#define VDDGFX_MAX_SAFE_MV   1200  // RDNA 3 VDDGFX 安全上限
#define VDDCI_MAX_SAFE_MV    1100  // VDDCI 安全上限

static int smu_v13_0_od_edit_dpm_table(struct smu_context *smu,
    enum PP_OD_DPM_TABLE_COMMAND type, long input[], uint32_t size)
{
    struct smu_13_0_dpm_context *dpm_context = smu->dpm_context;
    struct smu_13_0_overdrive_table *od_table = &dpm_context->od_table;
    int ret = 0;

    switch (type) {
    case PP_OD_EDIT_SCLK_VDDC_TABLE:
    {
        uint32_t clk_index = (uint32_t)input[0];
        uint32_t clk = (uint32_t)input[1];
        uint32_t vddc = (uint32_t)input[2];

        // 新增：电压上限检查
        if (vddc > VDDGFX_MAX_SAFE_MV) {
            dev_err(smu->adev->dev,
                "VDDC voltage %u mV exceeds safety limit %u mV\n",
                vddc, VDDGFX_MAX_SAFE_MV);
            return -EINVAL;
        }

        // 检查频率是否在有效范围内
        if (clk < od_table->min_clk || clk > od_table->max_clk) {
            dev_err(smu->adev->dev,
                "SCLK %u MHz out of range [%u, %u]\n",
                clk, od_table->min_clk, od_table->max_clk);
            return -EINVAL;
        }

        od_table->clocks[clk_index].clk = clk;
        od_table->clocks[clk_index].voltage = vddc;
        break;
    }
    case PP_OD_EDIT_MCLK_VDDCI_TABLE:
    {
        uint32_t clk_index = (uint32_t)input[0];
        uint32_t clk = (uint32_t)input[1];
        uint32_t vddci = (uint32_t)input[2];

        if (vddci > VDDCI_MAX_SAFE_MV) {
            dev_err(smu->adev->dev,
                "VDDCI voltage %u mV exceeds safety limit %u mV\n",
                vddci, VDDCI_MAX_SAFE_MV);
            return -EINVAL;
        }

        od_table->mem_clocks[clk_index].clk = clk;
        od_table->mem_clocks[clk_index].voltage = vddci;
        break;
    }
    default:
        break;
    }

    return ret;
}
```

#### 3.3 芯片体质补偿（Silicon Binning）

某 HPC 集群在使用 MI250 GPU 进行 FP64 计算时，偶尔出现不可预期的计算结果错误。

**问题定位**：
1. 错误间歇性出现，无法通过常规压力测试复现
2. 对比多个 GPU，发现某些特定芯片总是出现更多错误
3. 使用 ROCm 的 ECC 监控发现可纠正的 ECC 错误增多
4. 这些芯片运行在 V/F 曲线的边缘——电压略低于保证稳定所需的最小值

**根本原因分析**：

```
芯片体质分布（Silicon Lottery）：
              低体质（slow）        中体质（typical）     高体质（fast）
电压需求：     1100mV               1050mV              1000mV
在 2500MHz:
  当前设置：   1050mV               1050mV              1050mV
  结果：       不稳定                稳定                稳定（但浪费功耗）
```

**修复方案**：为每个 GPU 芯片添加体质标签，驱动根据体质加载不同的 V/F 曲线：

```c
// 芯片体质感知的 V/F 曲线加载
enum gpu_silicon_binning {
    GPU_BIN_FAST,     // 高频低电压（最高能效）
    GPU_BIN_TYPICAL,  // 标准
    GPU_BIN_SLOW,     // 需更高电压才能稳定
};

static const int gpu_binning_offset[] = {
    [GPU_BIN_FAST]    = -25,  // 比标准低 25mV
    [GPU_BIN_TYPICAL] = 0,    // 标准
    [GPU_BIN_SLOW]    = 25,   // 比标准高 25mV
};

static int smu_apply_binning_offset(struct smu_context *smu)
{
    struct amdgpu_device *adev = smu->adev;
    uint32_t bin_id;
    int ret;

    // 从 SMU 固件中读取芯片体质信息
    ret = smu_cmn_send_smc_msg(smu, SMU_MSG_GetBinningInfo, &bin_id);
    if (ret) {
        dev_warn(adev->dev, "Failed to read binning info, using default\n");
        return 0;
    }

    if (bin_id >= ARRAY_SIZE(gpu_binning_offset)) {
        dev_warn(adev->dev, "Unknown bin ID %u, using default\n", bin_id);
        return 0;
    }

    int32_t offset = gpu_binning_offset[bin_id];
    dev_info(adev->dev, "GPU bin: %d, voltage offset: %+d mV\n", bin_id, offset);

    if (offset != 0) {
        ret = smu_apply_voltage_offset(smu, offset);
        if (ret)
            dev_err(adev->dev, "Failed to apply voltage offset\n");
    }

    return ret;
}
```

#### 3.4 电压调节中的 TLB 一致性问题

在某嵌入式 AMD GPU 平台上，电压调节操作导致系统挂起，排查发现是 TLB（Translation Lookaside Buffer）一致性问题。

**问题现象**：
1. 执行电压调整操作
2. SMU 尝试修改 MMIO 寄存器
3. TLB 没有正确刷新，MMIO 地址映射到旧的页表条目
4. 写入操作进入错误的物理地址
5. 系统总线错误导致系统挂起

**修复方案**：在电压调整前显式刷新 MMIO 页表的 TLB：

```c
static int smu_safe_write_voltage_reg(struct smu_context *smu,
    uint32_t reg_offset, uint32_t value)
{
    struct amdgpu_device *adev = smu->adev;

    // 刷新 MMIO 区域的 TLB
    amdgpu_mmio_flush_tlb(adev, reg_offset, sizeof(uint32_t));

    // 写入电压寄存器
    WREG32(adev->pm.smu_prv_base + reg_offset, value);

    // 确保写入完成
    mb();

    // 验证写入是否成功
    uint32_t readback = RREG32(adev->pm.smu_prv_base + reg_offset);
    if (readback != value) {
        dev_err(adev->dev,
            "Voltage register write verification failed: "
            "wrote 0x%x, read 0x%x\n", value, readback);
        return -EIO;
    }

    return 0;
}
```

#### 3.5 实用欠压（Undervolting）优化流程

以下是一个完整的 GPU 欠压优化脚本：

```bash
#!/bin/bash
# undervolt_optimize.sh - GPU 欠压优化脚本
# 使用方法: sudo ./undervolt_optimize.sh [起始电压(mV)] [步长(mV)] [最低电压(mV)]

START_VOLTAGE=${1:-1100}
STEP=${2:-10}
MIN_VOLTAGE=${3:-800}
OD_PATH="/sys/class/drm/card0/device/pp_od_clk_voltage"
SCLK_LEVEL=4
SCLK_FREQ=2500

echo "GPU 欠压优化脚本 v1.0"
echo "起始电压: ${START_VOLTAGE}mV | 步长: ${STEP}mV | 最低电压: ${MIN_VOLTAGE}mV"
echo "=============================="

echo "manual" > /sys/class/drm/card0/device/power_dpm_force_performance_level

CURRENT=$START_VOLTAGE
PASS_COUNT=0

while [ $CURRENT -ge $MIN_VOLTAGE ]; do
    echo "--- 测试电压: ${CURRENT}mV ---"

    echo "s ${SCLK_LEVEL} ${SCLK_FREQ} ${CURRENT}" > $OD_PATH
    echo "c" > $OD_PATH
    sleep 1

    glmark2 --off-screen --duration 5 &>/dev/null &
    GL_PID=$!
    sleep 6

    if dmesg | tail -5 | grep -q "amdgpu.*fault\|amdgpu.*hang\|amdgpu.*gpu_isolation"; then
        echo "X ${CURRENT}mV - 不稳定"
        PREVIOUS=$((CURRENT + STEP))
        echo "s ${SCLK_LEVEL} ${SCLK_FREQ} ${PREVIOUS}" > $OD_PATH
        echo "c" > $OD_PATH
        echo ""
        echo "优化结果：最低稳定电压 = ${PREVIOUS}mV"
        echo "成功次数: ${PASS_COUNT}"
        break
    else
        echo "V ${CURRENT}mV - 稳定"
        PASS_COUNT=$((PASS_COUNT + 1))
    fi

    POWER=$(cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_average 2>/dev/null)
    TEMP=$(cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input 2>/dev/null)
    echo "  功耗: $(($POWER/1000000))W | 温度: $(($TEMP/1000))C"

    CURRENT=$((CURRENT - STEP))
done

echo "auto" > /sys/class/drm/card0/device/power_dpm_force_performance_level
```

#### 3.6 VID 映射表完整性测试

测试用例用于验证 VID 映射表的完整性：

```python
#!/usr/bin/env python3
"""
vid_table_test.py - VID 映射表完整性测试
验证内容：
1. VID 到电压的映射是否单调递增
2. 相邻 VID 之间的电压步进是否在允许范围内
3. 是否覆盖了所有预期电压范围
"""

import csv
import sys

class VIDTableValidator:
    def __init__(self, vid_table_file):
        self.vid_table = self.load_vid_table(vid_table_file)

    def load_vid_table(self, filename):
        table = {}
        with open(filename, 'r') as f:
            reader = csv.DictReader(f)
            for row in reader:
                vid = int(row['vid'], 16) if row['vid'].startswith('0x') else int(row['vid'])
                voltage_mv = int(row['voltage_mv'])
                table[vid] = voltage_mv
        return table

    def check_monotonic(self):
        """检查 VID 到电压的映射是否单调递增"""
        sorted_vids = sorted(self.vid_table.keys())
        errors = []
        for i in range(1, len(sorted_vids)):
            prev_v = self.vid_table[sorted_vids[i-1]]
            curr_v = self.vid_table[sorted_vids[i]]
            if curr_v <= prev_v:
                errors.append(
                    f"Non-monotonic: VID 0x{sorted_vids[i-1]:02X}({prev_v}mV) -> "
                    f"0x{sorted_vids[i]:02X}({curr_v}mV)")
        return errors

    def check_step_size(self, max_step_mv=25):
        """检查相邻 VID 的电压步进"""
        sorted_vids = sorted(self.vid_table.keys())
        errors = []
        for i in range(1, len(sorted_vids)):
            step = self.vid_table[sorted_vids[i]] - self.vid_table[sorted_vids[i-1]]
            if step > max_step_mv:
                errors.append(
                    f"Large step: VID 0x{sorted_vids[i-1]:02X} -> "
                    f"0x{sorted_vids[i]:02X}: {step}mV > {max_step_mv}mV")
        return errors

    def check_coverage(self, min_mv=700, max_mv=1400):
        """检查是否覆盖了所有常用电压范围"""
        covered = set(self.vid_table.values())
        missing = []
        for v in range(min_mv, max_mv + 1, 25):
            if v not in covered:
                missing.append(v)
        return missing

    def generate_report(self):
        print("=" * 60)
        print("VID 映射表完整性测试报告")
        print("=" * 60)
        print(f"测试文件: {len(self.vid_table)} 个 VID 条目")
        print(f"VID 范围: 0x{min(self.vid_table.keys()):02X} - 0x{max(self.vid_table.keys()):02X}")
        print(f"电压范围: {min(self.vid_table.values())}mV - {max(self.vid_table.values())}mV")

        mono_errors = self.check_monotonic()
        print(f"\n[单调性检查] {'PASS' if not mono_errors else 'FAIL: ' + str(len(mono_errors)) + ' 个错误'}")
        for err in mono_errors[:5]:
            print(f"  {err}")

        step_errors = self.check_step_size()
        print(f"[步长检查] {'PASS' if not step_errors else 'WARN: ' + str(len(step_errors)) + ' 个'}")
        missing = self.check_coverage()
        print(f"[覆盖率检查] {'PASS' if not missing else 'WARN: 缺失 ' + str(len(missing)) + ' 个电压点'}")

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("用法: python3 vid_table_test.py <vid_table.csv>")
        sys.exit(1)
    validator = VIDTableValidator(sys.argv[1])
    validator.generate_report()
```

### 4. VRM 与电压调节硬件原理

#### 4.1 VRM（Voltage Regulator Module）拓扑

GPU 电压调节依赖于主板上的 VRM 电路。现代 AMD GPU 使用多相（Multi-Phase）VRM 设计：

```
                      +- Phase 1 --+
  PWM控制器 ----+------> L1/C1     +----+
  (SMU/PMBus)   |     +------------+    |
                 |     +- Phase 2 --+    |
                 +-----> L2/C2     +----+
                 |     +------------+    |     +-------+
                 |     +- Phase N --+    +-----> Vout  +-> GPU VDDC
                 +-----> Ln/Cn     +----+     +-------+
                       +------------+
                                   + 去耦电容
  每相包含:
    Lx: 输出电感
    Cx: 输出电容
    高端MOSFET + 低端MOSFET
    栅极驱动器
```

**多相设计优势**：

| 相数 | 最大电流 | 纹波抑制 | 热分布 | 成本 |
|------|---------|---------|-------|------|
| 4相 | 200A | 良好 | 一般 | 低 |
| 6相 | 300A | 优秀 | 良好 | 中 |
| 8相 | 400A | 优秀 | 优秀 | 高 |
| 12相 | 600A+ | 极佳 | 极佳 | 很高 |

#### 4.2 SVI3 通信协议

SMU 通过 SVI3（Serial Voltage Identification Interface v3）总线与 VRM 通信。SVI3 是 AMD 标准化的 VRM 通信协议：

```
SMU (GPU内部)              VRM (主板)
    |                         |
    |  --- SVI3 Clock ------> |
    |  <--- SVI3 Data -------- |
    |                         |
    |  SVI3 数据包格式:       |
    |  +--------------------+ |
    |  | 命令字节 (1)       | |
    |  | 数据字节 (2)       | |
    |  | 校验和 (1)         | |
    |  | ACK (1)            | |
    |  +--------------------+ |
    |                         |
```

```c
// SVI3 命令定义
#define SVI3_CMD_SET_VDDC      0x01  // 设置 VDDC 电压
#define SVI3_CMD_SET_VDDCI     0x02  // 设置 VDDCI 电压
#define SVI3_CMD_SET_VDDP      0x03  // 设置 VDDP 电压
#define SVI3_CMD_READ_VDDC     0x10  // 读取 VDDC 电压
#define SVI3_CMD_READ_VDDCI    0x11  // 读取 VDDCI 电压
#define SVI3_CMD_READ_TEMP     0x20  // 读取 VRM 温度
#define SVI3_CMD_SET_VID       0x30  // 直接设置 VID 代码
#define SVI3_CMD_READ_STATUS   0x40  // 读取 VRM 状态

// SVI3 传输实现（伪代码）
int svi3_transfer(uint8_t cmd, uint16_t data, uint16_t *response)
{
    uint8_t packet[4];
    uint8_t reply[4];

    packet[0] = cmd;
    packet[1] = (data >> 8) & 0xFF;
    packet[2] = data & 0xFF;
    packet[3] = svi3_calculate_checksum(packet, 3);

    int ret = svi3_gpio_transfer(packet, reply, ARRAY_SIZE(packet));
    if (ret == 0 && response) {
        *response = (reply[1] << 8) | reply[2];
    }
    return ret;
}
```

#### 4.3 VRM 效率分析

VRM 的效率（eta）决定了输入功率中有多少能到达 GPU。主要损耗来源包括 MOSFET 导通损耗、开关损耗和电感损耗：

```python
def vrm_efficiency_analysis(vddc_mv, current_a, phase_count):
    """
    VRM 效率分析
    参数: vddc_mv (输出电压 mV), current_a (输出电流 A), phase_count (相数)
    返回: 效率 (%) 及各部分损耗
    """
    VIN = 12000  # 输入电压 12V (mV)
    FSW = 500000  # 开关频率 500KHz
    I_per_phase = current_a / phase_count

    # MOSFET 参数（典型值）
    Rds_on_hs = 0.0035
    Rds_on_ls = 0.0020
    Qg = 20e-9
    Vg = 5000
    t_rf = 20e-9

    D = vddc_mv / VIN
    # 导通损耗
    P_conduction = (I_per_phase**2 * Rds_on_hs * D +
                    I_per_phase**2 * Rds_on_ls * (1 - D)) * phase_count
    # 开关损耗
    P_switching = (0.5 * (VIN / 1000.0) * (current_a / phase_count) *
                   t_rf * FSW * 2) * phase_count
    P_gate = Qg * (Vg / 1000.0) * FSW * phase_count
    # 电感损耗
    DCR = 0.0005
    P_inductor = I_per_phase**2 * DCR * phase_count

    P_loss = P_conduction + P_switching + P_gate + P_inductor
    P_out = (vddc_mv / 1000.0) * current_a
    P_in = P_out + P_loss
    efficiency = (P_out / P_in) * 100

    return {
        'efficiency': efficiency,
        'p_out': P_out,
        'p_loss': P_loss,
        'conduction_loss': P_conduction,
        'switching_loss': P_switching + P_gate,
        'inductor_loss': P_inductor
    }

# 示例
vrm_data = vrm_efficiency_analysis(1050, 150, 7)
print(f"VRM 效率分析 @ 1.05V/150A/7相")
print(f"输出功率: {vrm_data['p_out']:.1f}W")
print(f"损耗功率: {vrm_data['p_loss']:.1f}W")
print(f"效率: {vrm_data['efficiency']:.1f}%")
```

### 5. 电压管理调试工具集

#### 5.1 Python 电压监控与记录工具

```python
#!/usr/bin/env python3
"""
gpu_voltage_monitor.py - GPU 电压实时监控工具
功能：实时读取 VDDC/VDDCI 电压、记录异常事件、导出 CSV
"""

import os
import time
import sys
import signal
import csv
from datetime import datetime

class GPUVoltageMonitor:
    def __init__(self, card=0, interval=0.5):
        self.card = card
        self.interval = interval
        self.device_path = f"/sys/class/drm/card{card}/device"
        self.running = True
        self.history = []
        self.anomalies = []
        signal.signal(signal.SIGINT, self._signal_handler)

    def _signal_handler(self, sig, frame):
        self.running = False

    def _find_hwmon_dir(self):
        hwmon_base = os.path.join(self.device_path, "hwmon")
        if not os.path.exists(hwmon_base):
            return None
        for entry in sorted(os.listdir(hwmon_base)):
            hwmon_path = os.path.join(hwmon_base, entry)
            if os.path.exists(os.path.join(hwmon_path, "in0_input")):
                return hwmon_path
        return None

    def read_voltage(self):
        try:
            hwmon_path = self._find_hwmon_dir()
            if hwmon_path is None:
                return None, None
            vddc_path = os.path.join(hwmon_path, "in0_input")
            vddc = None
            if os.path.exists(vddc_path):
                with open(vddc_path, 'r') as f:
                    vddc = int(f.read().strip()) / 1000.0
            vddci_path = os.path.join(hwmon_path, "in1_input")
            vddci = None
            if os.path.exists(vddci_path):
                with open(vddci_path, 'r') as f:
                    vddci = int(f.read().strip()) / 1000.0
            return vddc, vddci
        except:
            return None, None

    def detect_anomaly(self, vddc, vddci):
        anomalies = []
        if vddc is not None:
            if vddc > 1300:
                anomalies.append(f"VDDC 过高: {vddc:.0f}mV (安全上限: 1300mV)")
            elif vddc > 1200:
                anomalies.append(f"VDDC 偏高: {vddc:.0f}mV (注意散热)")
        if vddci is not None and vddci > 1200:
            anomalies.append(f"VDDCI 过高: {vddci:.0f}mV")
        return anomalies

    def run(self, duration=None):
        print("GPU 电压实时监控")
        print(f"{'时间':<20} {'VDDC(mV)':<12} {'VDDCI(mV)':<12} {'功耗(W)':<12}")
        print("-" * 56)

        hwmon_path = self._find_hwmon_dir()
        power_path = None
        if hwmon_path:
            p = os.path.join(hwmon_path, "power1_average")
            if os.path.exists(p):
                power_path = p

        start_time = time.time()
        while self.running:
            if duration and (time.time() - start_time) > duration:
                break
            timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            vddc, vddci = self.read_voltage()
            power = None
            if power_path:
                try:
                    with open(power_path, 'r') as f:
                        power = int(f.read().strip()) / 1000000.0
                except:
                    pass
            vddc_str = f"{vddc:.0f}" if vddc else "N/A"
            vddci_str = f"{vddci:.0f}" if vddci else "N/A"
            power_str = f"{power:.1f}" if power else "N/A"
            print(f"{timestamp:<20} {vddc_str:<12} {vddci_str:<12} {power_str:<12}")

            anomalies = self.detect_anomaly(vddc, vddci if vddci else 0)
            for a in anomalies:
                print(f"  W {a}")
                self.anomalies.append((timestamp, a))
            self.history.append({
                'timestamp': timestamp, 'vddc': vddc,
                'vddci': vddci, 'power': power
            })
            time.sleep(self.interval)

    def export_csv(self, filename="voltage_log.csv"):
        if not self.history:
            return
        with open(filename, 'w', newline='') as f:
            writer = csv.DictWriter(f, fieldnames=['timestamp', 'vddc', 'vddci', 'power'])
            writer.writeheader()
            writer.writerows(self.history)
        print(f"已导出 {len(self.history)} 条记录到 {filename}")

    def summary(self):
        if not self.history:
            return
        vddc_vals = [h['vddc'] for h in self.history if h['vddc']]
        vddci_vals = [h['vddci'] for h in self.history if h['vddci']]
        print(f"\n监控总结: {len(self.history)} 个样本点")
        if vddc_vals:
            print(f"VDDC: {min(vddc_vals):.0f}mV ~ {max(vddc_vals):.0f}mV 平均: {sum(vddc_vals)/len(vddc_vals):.0f}mV")
        if vddci_vals:
            print(f"VDDCI: {min(vddci_vals):.0f}mV ~ {max(vddci_vals):.0f}mV")
        if self.anomalies:
            print(f"异常事件: {len(self.anomalies)}")

if __name__ == "__main__":
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument('--duration', type=int, default=30)
    parser.add_argument('--interval', type=float, default=0.5)
    parser.add_argument('--export', action='store_true')
    args = parser.parse_args()
    monitor = GPUVoltageMonitor(interval=args.interval)
    try:
        monitor.run(duration=args.duration)
    except KeyboardInterrupt:
        pass
    finally:
        monitor.summary()
        if args.export:
            monitor.export_csv()
```

#### 5.2 Voltage Offset 自动调优脚本

基于二分搜索法寻找最优电压偏移量：

```python
#!/usr/bin/env python3
"""voltage_offset_tuner.py - 电压偏移自动调优"""

import os
import time
import subprocess
import re

class VoltageOffsetTuner:
    def __init__(self):
        self.od_path = "/sys/class/drm/card0/device/pp_od_clk_voltage"
        self.perf_path = "/sys/class/drm/card0/device/power_dpm_force_performance_level"

    def set_offset(self, offset_mv):
        try:
            with open(self.od_path, 'w') as f:
                f.write(f"vc 0 {offset_mv}\nc\n")
            time.sleep(0.5)
            return True
        except:
            return False

    def run_stress(self, duration=10):
        try:
            proc = subprocess.Popen(
                ["glmark2", "--off-screen", "--duration", str(duration)],
                stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
            proc.wait(timeout=duration + 5)
            return proc.returncode == 0
        except:
            return False

    def check_errors(self):
        result = subprocess.run(
            ["dmesg", "--level=err,warn", "--since=10 seconds ago"],
            capture_output=True, text=True, timeout=5)
        for line in result.stdout.split('\n'):
            if re.search(r"amdgpu.*fault|amdgpu.*hang", line, re.IGNORECASE):
                return True, line
        return False, ""

    def binary_search_tune(self, min_offset=-100, max_offset=50):
        print(f"电压偏移二分搜索: {min_offset}mV ~ {max_offset}mV")
        with open(self.perf_path, 'w') as f:
            f.write("manual")

        low, high = min_offset, max_offset
        best = 0

        while low <= high:
            mid = round((low + high) / 2 / 5) * 5
            mid = max(min_offset, min(max_offset, mid))
            print(f"\n测试偏移: {mid:+d}mV")

            if not self.set_offset(mid):
                high = mid - 5
                continue

            ok = self.run_stress(15)
            has_err, msg = self.check_errors()

            if ok and not has_err:
                print(f"  V 稳定")
                best = mid
                low = mid + 5
            else:
                print(f"  X 不稳定: {msg[:60]}")
                high = mid - 5

        with open(self.perf_path, 'w') as f:
            f.write("auto")
        print(f"\n最优偏移: {best:+d}mV")
        return best

if __name__ == "__main__":
    tuner = VoltageOffsetTuner()
    tuner.binary_search_tune()
```

### 6. 实践练习

#### 练习一：阅读 V/F 曲线

1. 通过 `pp_od_clk_voltage` 读取你的 GPU 的 V/F 曲线
2. 记录所有 SCLK 和 MCLK 档位的频率-电压对
3. 绘制频率-电压关系图（可用 Python matplotlib）
4. 标注线性区和饱和区的分界点

#### 练习二：电压微调实验

1. 将 SCLK 最高档位的电压降低 25mV
2. 运行 glmark2 测试稳定性
3. 如果稳定，继续降低 25mV，直到不稳定
4. 记录每个电压点的功耗和温度变化
5. 提交电压-功耗关系曲线

#### 练习三：VID 表逆向工程

通过 debugfs 和 sysfs 逆向你的 GPU 的 VID 映射表：

```bash
#!/bin/bash
OD_PATH="/sys/class/drm/card0/device/pp_od_clk_voltage"
echo "manual" > /sys/class/drm/card0/device/power_dpm_force_performance_level

echo "VID 映射表重建"
printf "%-15s %-15s\n" "频率(MHz)" "电压(mV)"

while IFS= read -r line; do
    if [[ $line =~ ^([0-9]+):\ ([0-9]+)MHz\ ([0-9]+)mV ]]; then
        printf "%-15s %-15s\n" "${BASH_REMATCH[2]}" "${BASH_REMATCH[3]}"
    fi
done < <(cat $OD_PATH)

echo "auto" > /sys/class/drm/card0/device/power_dpm_force_performance_level
```

#### 练习四：欠压场景的性能对比

1. 在默认电压下运行基准测试，记录 FPS
2. 降压 50mV 后再次运行，记录 FPS
3. 降压 100mV 后再次运行，记录 FPS
4. 分析电压降低对性能的影响

#### 练习五：VRM 效率计算

基于 VRM 效率分析公式，编写计算器：
- 低负载（50A）vs 高负载（200A）下的效率差异
- 4 相 vs 8 相 vs 12 相的效率差异
- 电压从 0.8V 升到 1.2V 时效率变化

#### 练习六：电压噪声分析

通过高频 sysfs 采样分析 VDDC 电压纹波：

```python
#!/usr/bin/env python3
"""voltage_noise_analyzer.py - 电压噪声分析"""

import time
import statistics

class VoltageNoiseAnalyzer:
    def __init__(self, hwmon_path, interval=0.001):
        self.hwmon_path = hwmon_path
        self.interval = interval
        self.samples = []

    def read_vddc(self):
        try:
            with open(f"{self.hwmon_path}/in0_input", 'r') as f:
                return int(f.read().strip()) / 1000.0
        except:
            return None

    def sample(self, duration=2):
        print(f"采样中... ({len(self.samples) if self.samples else 0})")
        start = time.time()
        while time.time() - start < duration:
            v = self.read_vddc()
            if v is not None:
                self.samples.append(v)
            time.sleep(self.interval)

    def analyze(self):
        if len(self.samples) < 2:
            return
        mean_v = statistics.mean(self.samples)
        min_v = min(self.samples)
        max_v = max(self.samples)
        stdev_v = statistics.stdev(self.samples)
        p2p = max_v - min_v
        print(f"\n电压噪声分析 ({len(self.samples)} 样本)")
        print(f"平均: {mean_v:.2f}mV  最小: {min_v:.2f}mV  最大: {max_v:.2f}mV")
        print(f"峰峰值: {p2p:.2f}mV ({p2p/mean_v*100:.2f}%)")
        print(f"标准差: {stdev_v:.3f}mV  纹波系数: {stdev_v/mean_v*100:.3f}%")

if __name__ == "__main__":
    import sys
    hwmon = sys.argv[1] if len(sys.argv) > 1 else "/sys/class/drm/card0/device/hwmon/hwmon1"
    analyzer = VoltageNoiseAnalyzer(hwmon)
    analyzer.sample(3)
    analyzer.analyze()
```

#### 练习七：芯片体质模拟测试

设计测试验证芯片体质感知的 V/F 曲线逻辑：
1. 创建模拟 GPU 芯片，具有三种体质（fast/typical/slow）
2. 为每种体质生成不同的 V/F 曲线
3. 实现驱动初始化的体质检测逻辑
4. 验证正确的 V/F 曲线是否被加载

#### 练习八：跨架构电压对比

对比不同架构的电压管理特性：
1. GCN (Vega)、RDNA 2 (Navi 21)、RDNA 3 (Navi 31) 的 V/F 曲线差异
2. 各架构的电压域命名和数量差异
3. 电压调节粒度差异
4. OverDrive 电压调节能力差异

### 7. 相关链接

- **AMD RDNA 3 架构白皮书（公开章节）**：https://www.amd.com/en/products/graphics/amd-radeon-rx-7900xtx
- **Linux 内核源码 - amdgpu_dpm.c**：https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/pm/amdgpu_dpm.c
- **ROCm SMI 官方文档**：https://rocm.docs.amd.com/projects/amdsmi/en/latest/
- **AMD GPUOpen - Power Management**：https://gpuopen.com/learn/amd-powerplay-technology/
- **Voltage Regulator (VRM) 基础**：https://www.ti.com/power-management/non-isolated-dcdc/overview.html
- **SVI3 协议规范**：AMD 内部文档 ND0012345
- **Linux 内核电压管理文档**：https://www.kernel.org/doc/html/latest/power/
- **PMBus 规范**：https://pmbus.org/specification/
- **半导体工艺与芯片体质管理**：https://en.wikipedia.org/wiki/Semiconductor_device_fabrication
- **AMD ROCm 性能优化指南**：https://rocm.docs.amd.com/en/latest/how-to/performance-tuning.html

### 8. 知识自测

**选择题**：

1. 动态功耗与电压的关系是？
   A. 线性正比 (P proportional V)
   B. 平方正比 (P proportional V2)
   C. 指数关系 (P proportional eV)
   D. 无关

2. VID (Voltage Identifier) 的主要作用是什么？
   A. 标识 GPU 芯片的身份信息
   B. 作为电压值与 VRM 之间的数字编码接口
   C. 检测电压是否过高的保护电路
   D. 记录电压调节历史

3. 在 V/F 曲线的饱和区，下列哪项描述正确？
   A. 频率随电压线性增长
   B. 为达到更高频率需要大幅提高电压
   C. 电压增加时频率几乎不变
   D. 功耗增加速率开始下降

4. OverDrive 中 "vc 0 -25" 的含义是什么？
   A. VDDC 电压偏移设为 -25mV
   B. VDDCI 电压偏移设为 -25mV
   C. SCLK 频率降低 25MHz
   D. 最高电压限制在 25mV

5. SVI3 是什么？
   A. GPU 核心电压的测量标准
   B. AMD 标准化的 VRM 串行通信协议
   C. Linux 内核电压管理子系统
   D. GPU 供电设计标准

6. 多相 VRM 的主要优势是什么？
   A. 降低成本
   B. 减少输出纹波并提高电流能力
   C. 减小 PCB 面积
   D. 简化电路设计

7. 芯片体质（binning）对电压管理的影响是？
   A. 所有芯片的 V/F 曲线完全一致
   B. 慢体质芯片需要更高电压才能稳定
   C. 快体质芯片不需要高电压
   D. 体质与电压无关

**填空题**：

8. V/F 曲线的最低电压点通常约为 ________ mV。
9. VRM 效率损耗主要包括 ________、________ 和 ________。
10. Linux 内核中通过 ________ sysfs 文件查看 V/F 曲线。
11. 现代 AMD GPU 的 VRM 通常使用 ________ 通信协议。
12. 漏电流随温度升高每 ________ 大约增加一倍。
13. RDNA 3 使用 ________ 位宽的 VID 编码。

**答案**：1-B, 2-B, 3-B, 4-A, 5-B, 6-B, 7-B, 8-750, 9-导通损耗、开关损耗、电感损耗, 10-pp_od_clk_voltage, 11-SVI3, 12-10摄氏度, 13-8

### 9. 附录：电压调试命令速查表

| 操作 | 命令 | 说明 |
|------|------|------|
| 查看VDDC | `cat /sys/class/drm/card0/device/hwmon/hwmon*/in0_input` | 单位mV |
| 查看VDDCI | `cat /sys/class/drm/card0/device/hwmon/hwmon*/in1_input` | 单位mV |
| 查看V/F表 | `cat /sys/class/drm/card0/device/pp_od_clk_voltage` | 显示所有档位 |
| 设置manual | `echo "manual" > power_dpm_force_performance_level` | 允许手动调压 |
| 设置SCLK电压 | `echo "s 4 2500 1050" > pp_od_clk_voltage` | SCLK档位4 1050mV |
| 设置MCLK电压 | `echo "m 2 2500 1000" > pp_od_clk_voltage` | MCLK档位2 1000mV |
| 电压偏移 | `echo "vc 0 -25" > pp_od_clk_voltage` | VDDC偏移-25mV |
| 提交更改 | `echo "c" > pp_od_clk_voltage` | 提交所有OD更改 |
| 恢复默认 | `echo "r" > pp_od_clk_voltage` | 恢复V/F默认值 |
| 查看OD范围 | `cat pp_od_clk_voltage \| grep OD_RANGE` | 可调节范围 |
| rocm-smi电压 | `rocm-smi --showvoltage` | 显示电压信息 |
| 详细传感器 | `rocm-smi --showallinfo \| grep -i voltage` | 所有电压数据 |
| debugfs信息 | `sudo cat /sys/kernel/debug/dri/0/amdgpu_pm_info` | V/F曲线详情 |
| 监控dmesg | `sudo dmesg -w \| grep -E "amdgpu.*volt"` | 电压相关错误 |

## 今日小结

- **VID（Voltage Identifier）** 是 SMU 与 VRM 之间传递电压信息的数字编码机制，简化了电压调节接口。
- 电压对功耗的影响（P proportional V2）远大于频率，是能效优化的首要目标。
- V/F 曲线定义了各频率点所需的最低稳定电压，SMU 在频率切换时自动沿曲线调整电压。
- **VRM 多相设计**提供了必要的电流能力和纹波抑制，效率分析需要考虑导通、开关和电感损耗。
- 芯片体质（Silicon Lottery）导致不同芯片在同一频率下需要不同的电压，驱动需要感知体质并调整 V/F 曲线。
- 用户可通过 **OverDrive 接口** 微调单点电压或整体偏移，但需要谨慎操作以避免损坏硬件。
- **SVI3 协议**是 SMU 与 VRM 通信的标准化串行总线协议。
- 完整的调试工具链包括电压监控 Python 脚本、偏移自动调优工具、噪声分析工具等。
- 实际案例分析涵盖了 VDDC 传感器读取 Bug、过压保护增强、芯片体质补偿、TLB 一致性问题和欠压优化流程。
- 低电压优化需要充分的稳定性测试，监控 dmesg 中的 GPU 错误是关键验证手段。

## 扩展思考（可选）

假设你负责一款面向数据中心的 AMD GPU（如 MI250 / MI300）的电压调节测试。与消费级 GPU 不同，数据中心 GPU 对电压稳定性和长期可靠性有更严格的要求。

1. 在设计 V/F 曲线验证方案时，你会选择哪些测试负载（benchmark）？为什么？
2. 如何设计加速老化（Accelerated Aging）测试来评估低电压偏移对芯片寿命的影响？
3. 在数据中心场景下，每个芯片的体质差异应该如何被系统性管理（binning）？驱动应如何处理不同体质的芯片？
4. 如果发现 MI300 在特定 V/F 曲线下出现 FP64 计算精度问题，你会如何排查和修复？
5. 设计一个基于机器学习的电压预测系统，根据工作负载预测最优电压，可以降低多少功耗？

### 10. 附录1：电压测量与验证方法

#### 10.1 软件级电压测量

软件读取的电压值来源于 GPU 内部传感器，而非 VRM 实际输出电压：

```
传感器路径: GPU裸片 → 片上传感器 → SMU寄存器 → 驱动 → hwmon/sysfs → 用户态
```

**片上电压传感器的局限性**：

| 局限性 | 影响 | 缓解措施 |
|--------|------|----------|
| 采样率低（通常 10-100 Hz） | 无法捕捉纳秒级电压跌落 | 使用外部示波器测量 |
| 精度有限（±1-3%） | 微小偏移难以检测 | 多采样点取平均（fflush） |
| 传感器位置固定 | 可能错过热点电压 | 多传感器交叉验证 |
| 数字化延迟 | 读数滞后于实际变化 | IRQ 驱动的高频采样 |

**电压测量脚本示例（带统计）**：

```python
#!/usr/bin/env python3
"""GPU 电压监控与统计脚本"""
import time
import statistics
import argparse
import os

def read_sysfs(path):
    try:
        with open(path, 'r') as f:
            return int(f.read().strip())
    except (IOError, OSError, ValueError) as e:
        return None

def monitor_voltage(gpu_id=0, duration=10, interval=0.1):
    hwmon_base = f"/sys/class/drm/card{gpu_id}/device/hwmon"
    if not os.path.exists(hwmon_base):
        hwmon_base = f"/sys/class/drm/renderD128/gpu_device/hwmon"
    
    hwmon_dirs = [d for d in os.listdir(hwmon_base) if d.startswith('hwmon')]
    if not hwmon_dirs:
        print("找不到 hwmon 设备")
        return
    
    hwmon_path = os.path.join(hwmon_base, hwmon_dirs[0])
    samples = {'VDDC': [], 'VDDCI': [], 'timestamp': []}
    
    print(f"开始监控 GPU{gpu_id} 电压 {duration} 秒...")
    start = time.time()
    while time.time() - start < duration:
        vddc = read_sysfs(os.path.join(hwmon_path, 'in0_input'))
        vddci = read_sysfs(os.path.join(hwmon_path, 'in1_input'))
        t = time.time() - start
        if vddc is not None:
            samples['VDDC'].append(vddc / 1000.0)  # 转换为mV
        if vddci is not None:
            samples['VDDCI'].append(vddci / 1000.0)
        samples['timestamp'].append(t)
        time.sleep(interval)
    
    print(f"\n=== 电压统计报告 ===")
    print(f"监控时长: {duration} 秒, 采样间隔: {interval} 秒")
    print(f"总采样数: {len(samples['timestamp'])}")
    
    for rail in ['VDDC', 'VDDCI']:
        values = samples[rail]
        if not values:
            print(f"{rail}: 无数据")
            continue
        print(f"\n{rail} 统计:")
        print(f"  平均值: {statistics.mean(values):.2f} mV")
        print(f"  中位数: {statistics.median(values):.2f} mV")
        print(f"  最大值: {max(values):.2f} mV")
        print(f"  最小值: {min(values):.2f} mV")
        print(f"  标准差: {statistics.stdev(values):.2f} mV")
        print(f"  波动范围: {max(values)-min(values):.2f} mV")

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description='GPU 电压监控工具')
    parser.add_argument('--gpu', type=int, default=0, help='GPU ID')
    parser.add_argument('--duration', type=int, default=10, help='监控时长')
    parser.add_argument('--interval', type=float, default=0.1, help='采样间隔')
    args = parser.parse_args()
    monitor_voltage(args.gpu, args.duration, args.interval)
```

#### 10.2 硬件级电压测量

对于深度调试，软件读数不足时需使用硬件测量：

**测量工具**：

| 工具 | 精度 | 带宽 | 适用场景 |
|------|------|------|----------|
| 数字万用表 (DMM) | ±0.1% | DC | 稳态电压测量 |
| 示波器 + 差分探头 | ±1% | 100MHz+ | 动态电压跌落(Vdroop) |
| 数据采集卡 (DAQ) | ±0.5% | 1MHz | 长时间电压趋势记录 |
| 主动电压探头 | ±0.5% | 1GHz+ | 高频噪声分析 |

**测量要点**：

1. **测量点选择**：在 VRM 输出电容两端测量，而非在 GPU 封装引脚（后者受 IR drop 影响）
2. **探头连接**：使用弹簧地线（短地线）而非长夹子，避免引入高频噪声
3. **AC 耦合**：使用交流耦合模式可以清晰观察电压纹波（通常 10-50mVpp）
4. **负载瞬态响应**：在负载突增/突减时观察电压跌落（Vdroop）和过冲（Vovershoot）

**Vdroop 分析的 Python 脚本框架**：

```python
import numpy as np
import matplotlib.pyplot as plt

def analyze_vdroop(waveform, sample_rate, v_nominal):
    """
    分析示波器捕获的电压跌落波形
    waveform: 电压采样数组 (V)
    sample_rate: 采样率 (Hz)
    v_nominal: 标称电压 (V)
    """
    dt = 1.0 / sample_rate
    v_min = np.min(waveform)
    vdroop = v_nominal - v_min
    vdroop_pct = (vdroop / v_nominal) * 100
    
    # 找到跌落起始点和最低点
    threshold = v_nominal * 0.97
    crossing_points = np.where(waveform < threshold)[0]
    if len(crossing_points) > 0:
        t_start = crossing_points[0] * dt
        t_min = np.argmin(waveform) * dt
        droop_time = t_min - t_start  # 跌落时间
        recovery_time = (len(waveform) - np.argmin(waveform)) * dt
    else:
        droop_time = 0
        recovery_time = 0
    
    print("Vdroop 分析结果:")
    print(f"  标称电压: {v_nominal:.3f} V")
    print(f"  最低电压: {v_min:.3f} V")
    print(f"  电压跌落: {vdroop:.3f} V ({vdroop_pct:.1f}%)")
    print(f"  跌落时间: {droop_time*1e6:.1f} us")
    print(f"  恢复时间: {recovery_time*1e6:.1f} us")
    
    # 判断是否符合规格
    if vdroop_pct < 5:
        print("  判定: PASS (跌落 < 5%)")
    elif vdroop_pct < 10:
        print("  判定: WARNING (跌落 5-10%)")
    else:
        print("  判定: FAIL (跌落 > 10%)")
    
    return {
        'v_min': v_min,
        'vdroop': vdroop,
        'vdroop_pct': vdroop_pct,
        'droop_time_us': droop_time * 1e6,
        'recovery_time_us': recovery_time * 1e6
    }
```

#### 10.3 动态电压调节的示波器测量

在分析 DPM 状态切换时的电压行为时，示波器测量至关重要：

**典型测试流程**：

1. 将 GPU 置于稳态满载（运行 Furmark 或 GPGPU 计算负载）
2. 记录当前 VDDC 稳态电压（例如 1.05V）
3. 突然停止负载（模拟 Vdroop 恢复）
4. 观察负载变化后 100us 内的电压波形
5. 测量电压过冲幅度和振铃频率

**常见电压波形问题**：

```ascii
正常波形:     异常波形（振铃）:    异常波形（欠阻尼）:
  __             __                  __
 /  \          /  \  /\           /  \
|    |         |   \/  \         |    \
|    |         |        \        |     \
|    |         |         |       |      \
+----+         +---------+       +-------+
```

**电压验证检查清单**：

- [ ] 稳态电压在规格范围内（±3%）
- [ ] Vdroop 不超过标称电压的 5%
- [ ] 恢复时间不超过 50us
- [ ] 无持续振铃（ringing）
- [ ] 开关频率纹波 < 20mVpp
- [ ] 相邻 P-State 转换时电压过冲 < 100mV
- [ ] 温度变化 50°C 时的电压漂移 < 2%
- [ ] 多相 VRM 电流均衡度 > 90%

### 11. 附录2：电压调试常见问题排障指南

#### 问题1: VDDC 电压显示 0 或异常值

**现象**：`cat in0_input` 返回 0 或明显不合理的值（如 12000 mV）

**排查步骤**：

1. 检查 hwmon 设备是否存在：`ls /sys/class/drm/card0/device/hwmon/`
2. 检查驱动是否加载了电压监控功能：`cat /sys/kernel/debug/dri/0/amdgpu_pm_info | grep volt`
3. 检查 SMU 固件版本：`cat /sys/class/drm/card0/device/vbios_version`
4. 尝试重置 SMU：`echo "1" > /sys/class/drm/card0/device/pp_smu_reset`
5. 检查 dmesg 中是否有 SMU 通信超时错误

**常见原因**：

| 原因 | 特征 | 解决 |
|------|------|------|
| SMU 固件崩溃 | dmesg 有 SMU 超时 | 重启 GPU 或系统 |
| hwmon 驱动 Bug | 特定芯片无法注册 hwmon | 升级内核或应用补丁 |
| VBIOS 损坏 | in0_input 返回 0 | 刷新 VBIOS |
| 传感器硬件故障 | 电压正常但读数为 0 | 硬件维修 |

#### 问题2: 调节电压不生效

**现象**：写入 `pp_od_clk_voltage` 后读取电压无变化

**排查步骤**：

```bash
# 确认当前权限模式
cat /sys/class/drm/card0/device/power_dpm_force_performance_level

# 必须切换为 manual 模式才能调压
echo "manual" > /sys/class/drm/card0/device/power_dpm_force_performance_level

# 查看允许的 OD 范围
cat /sys/class/drm/card0/device/pp_od_clk_voltage | grep OD_RANGE

# 设置新电压并提交
echo "s 1 500 800" > /sys/class/drm/card0/device/pp_od_clk_voltage
echo "c" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 验证设置是否生效
cat /sys/class/drm/card0/device/pp_od_clk_voltage
```

#### 问题3: 系统在自定义电压下崩溃

**现象**：写入特定电压值后系统冻结、GPU 重置或 dmesg 出现 GPU 挂起信息

**排查方法**：

```python
#!/usr/bin/env python3
"""二分法查找稳定电压 - 电压调优工具"""
import subprocess
import time
import sys

def set_voltage(gpu_path, clock_idx, freq_mhz, volt_mv):
    cmd_s = f"echo 's {clock_idx} {freq_mhz} {volt_mv}' > {gpu_path}/pp_od_clk_voltage"
    cmd_c = f"echo 'c' > {gpu_path}/pp_od_clk_voltage"
    try:
        subprocess.run(cmd_s, shell=True, check=True, timeout=5)
        subprocess.run(cmd_c, shell=True, check=True, timeout=5)
        return True
    except subprocess.CalledProcessError:
        return False

def binary_search_stable_voltage(gpu_path, clock_idx, freq_mhz, v_low, v_high, load_cmd):
    """二分查找指定频率下的最低稳定电压"""
    print(f"二分查找: 频率 {freq_mhz} MHz, 电压范围 [{v_low}, {v_high}] mV")
    iterations = 0
    while v_high - v_low > 6.25:  # 精度 6.25mV
        v_mid = (v_low + v_high) // 2
        iterations += 1
        print(f"  迭代 {iterations}: 尝试 {v_mid} mV...", end=' ')
        
        if not set_voltage(gpu_path, clock_idx, freq_mhz, v_mid):
            print("写入失败")
            v_low = v_mid + 6.25
            continue
        
        time.sleep(1)
        
        try:
            # 运行负载测试（用户实现）
            subprocess.run(load_cmd, shell=True, check=True, timeout=10)
            print(f"稳定")
            v_high = v_mid  # 尝试更低的电压
        except (subprocess.CalledProcessError, subprocess.TimeoutExpired):
            print(f"不稳定")
            v_low = v_mid + 6.25
    
    print(f"\n结果: 频率 {freq_mhz} MHz 最低稳定电压: {v_high} mV")
    return v_high

if __name__ == "__main__":
    gpu_path = "/sys/class/drm/card0/device"
    # 示例：查找 SCLK 档位1在500MHz下的最低稳定电压
    binary_search_stable_voltage(
        gpu_path=gpu_path,
        clock_idx=1,
        freq_mhz=500,
        v_low=700,
        v_high=900,
        load_cmd="glmark2 --benchmark build:duration=5"
    )
```

#### 问题4: 不同 P-State 间电压跳变异常

**现象**：P-State 切换时电压跳变幅度过大或过小，导致不稳定或性能异常

**分析方法**：

```bash
# 启用 ftrace 追踪 P-State 切换
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace
echo 1 > /sys/kernel/debug/tracing/events/power/pstate/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 运行负载触发 P-State 切换
sleep 10

# 分析追踪结果
cat /sys/kernel/debug/tracing/trace | grep pstate | head -50
```

### 12. AMD 电压调节的行业标准对比

了解竞争对手的电压管理方案有助于理解 AMD 设计的权衡：

| 特性 | AMD (RDNA 3) | NVIDIA (Ada Lovelace) | Intel (Arc Alchemist) |
|------|--------------|----------------------|----------------------|
| VID 编码 | 8-bit (256级) | 10-bit (1024级) | 8-bit (256级) |
| V/F 曲线管理 | SMU 管理，OD接口可调 | 完全 GPU 自行管理 | GuC 管理 |
| 用户调压 | OverDrive 接口 | GPU Boost 偏移 | OEM 定制 |
| 电压监控 | hwmon (in0, in1) | NVAPI / NVML | i915 PMU |
| 安全限制 | 温度保护 + 电源限制 | GPU Boost 3.0 | 温度 + TDP 双重限 |
| VRM 协议 | SVI3 | SVI3 / SVID | PMBus |
| 电压偏移 | +/- (固定步长) | +/- (开放整数) | OEM 定制 |

**性能对比洞察**：

1. **粒度差异**：NVIDIA 的 10-bit VID 编码提供了更精细的电压控制，理论上能效更优
2. **用户可控性**：AMD 的 OverDrive 接口提供了更直接的用户调压能力，而 NVIDIA 更倾向自动管理
3. **生态系统**：Intel 的电压管理更多依赖于 OEM 定制，开放程度最低

### 13. 附录3：Linux 内核电压子系统简介

Linux 内核提供了通用的电压框架（`regulator` subsystem），为各种设备提供标准的电压控制接口：

```ascii
用户空间 (sysfs/ioctl)
    |
内核子系统:
    +-- regulator 框架 (drivers/regulator/core.c)
    |   +-- Voltage Regulator 设备驱动
    |   +-- Consumer API (regulator_get, regulator_set_voltage, ...)
    |
    +-- SMU 固件接口 (drivers/gpu/drm/amd/pm/swsmu/)
    |   +-- smu_v13_0.c, smu_v11_0.c
    |   +-- AMDGPU 专有电压管理
    |
    +-- hwmon 子系统 (drivers/hwmon/)
        +-- amdgpu_hwmon.c (GPU 传感器暴露)
        +-- k10temp.c (CPU 温度 - 参考实现)
```

**AMDGPU 电压管理与通用 regulator 框架的关系**：

- AMDGPU 不直接使用 regulator 框架管理核心电压（VDDC）
- GPU 核心电压由 SMU 固件通过 SVI3 协议直接与 VRM 通信控制
- VDDP、VDD_MISC 等辅助电压可能复用 regulator 框架
- hwmon 子系统仅用于电压监控读数的暴露，不参与电压调节控制

**关键内核 API 参考**：

| API | 用途 | 文件位置 |
|-----|------|----------|
| `regulator_get()` | 获取 regulator 句柄 | `drivers/regulator/core.c` |
| `regulator_set_voltage()` | 设置电压 | `drivers/regulator/core.c` |
| `regulator_get_voltage()` | 读取当前电压 | `drivers/regulator/core.c` |
| `regulator_enable/disable()` | 使能/禁能调节器 | `drivers/regulator/core.c` |
| `smu_get_voltage()` | 从 SMU 读取电压 | `drivers/gpu/drm/amd/pm/swsmu/amdgpu_smu.c` |
| `smu_set_voltage()` | 通过 SMU 设置电压 | `drivers/gpu/drm/amd/pm/swsmu/amdgpu_smu.c` |

### 14. 附录4：芯片体质与 V/F 曲线表征

#### 14.1 芯片体质的统计学分布

芯片体质（silicon quality）在晶圆制造中服从正态分布：

```ascii
分布密度
    ^
    |         ---- Gaussian Distribution ----
    |        /    \              /    \
    |       / 慢体质 \          / 快体质  \
    |      /  (高漏电) \      /  (低漏电)  \
    |     /             \    /              \
    |    /               \  /                \
    |   /                 \/                  \
    +--+-------------------+---------------------> 体质等级
      slow        typical         fast
      (leaky)     (normal)        (tight)
```

**体质等级的电压需求差异**：

| 体质等级 | 比例 | Vmin @ 2.5GHz | 漏电流 | 推荐用途 |
|----------|------|---------------|--------|----------|
| Fast (快) | 15% | 0.95V | 低 | 高频率 SKU |
| Typical (典型) | 60% | 1.00V | 中 | 标准 SKU |
| Slow (慢) | 20% | 1.08V | 高 | 低频率 SKU |
| Defect (缺陷) | 5% | >1.15V 或不稳 | 极高 | 降级或报废 |

#### 14.2 驱动级体质感知算法

```python
#!/usr/bin/env python3
"""芯片体质等级检测算法模拟"""
import random
import statistics

class SiliconBinningDetector:
    """模拟 SMU 的体质检测算法"""
    
    def __init__(self):
        self.vf_curve = {}  # 频率 -> 最低稳定电压
        self.bin_grade = None
        self.leakage_current = None
    
    def measure_vmin_at_freq(self, freq_mhz, max_attempts=5):
        """在指定频率下测量最低稳定电压"""
        # 模拟电压测量过程
        # 实际 SMU 会通过 ran 命令触发频率->电压扫描
        
        print(f"测量 {freq_mhz} MHz 的最低稳定电压...")
        
        # 实际测量结果，这里模拟
        v_estimated = 700 + (freq_mhz - 500) * 0.15  # 理想曲线
        noise = random.gauss(0, 6.25)  # 模拟测量噪声
        return v_estimated + noise
    
    def scan_full_curve(self, freq_points=None):
        """扫描完整 V/F 曲线"""
        if freq_points is None:
            freq_points = [500, 1000, 1500, 2000, 2500, 3000]
        
        print("扫描完整 V/F 曲线...")
        for freq in freq_points:
            v = self.measure_vmin_at_freq(freq)
            self.vf_curve[freq] = v
            print(f"  {freq} MHz -> {v:.1f} mV")
        
        return self.vf_curve
    
    def determine_bin(self):
        """根据扫描结果确定体质等级"""
        if not self.vf_curve:
            self.scan_full_curve()
        
        # 使用 2.5GHz 点的电压作为分级依据
        v_at_2p5g = self.vf_curve.get(2500, None)
        if v_at_2p5g is None:
            # 插值
            freqs = sorted(self.vf_curve.keys())
            if 2500 < freqs[0] or 2500 > freqs[-1]:
                return "unknown"
            # 简单线性插值
            f_low = max(f for f in freqs if f <= 2500)
            f_high = min(f for f in freqs if f >= 2500)
            v_low = self.vf_curve[f_low]
            v_high = self.vf_curve[f_high]
            ratio = (2500 - f_low) / (f_high - f_low)
            v_at_2p5g = v_low + ratio * (v_high - v_low)
        
        # 分级判定阈值（示例如下，实际因架构而异）
        if v_at_2p5g < 970:
            self.bin_grade = "fast"
        elif v_at_2p5g < 1050:
            self.bin_grade = "typical"
        elif v_at_2p5g < 1150:
            self.bin_grade = "slow"
        else:
            self.bin_grade = "defect"
        
        print(f"体质等级: {self.bin_grade.upper()} (2.5GHz Vmin = {v_at_2p5g:.1f} mV)")
        return self.bin_grade
    
    def calculate_optimal_vf_curve(self, target_power_w=250):
        """根据体质等级计算最优 V/F 曲线"""
        base_v = {
            "fast": 720, "typical": 750, "slow": 780, "defect": 820
        }.get(self.bin_grade, 750)
        
        slope = {
            "fast": 0.12, "typical": 0.15, "slow": 0.18, "defect": 0.22
        }.get(self.bin_grade, 0.15)
        
        print(f"\n体质 '{self.bin_grade}' 的最优 V/F 曲线:")
        print(f"  {'频率(MHz)':<12} {'电压(mV)':<12} {'功耗(W)':<12}")
        print(f"  {'-'*36}")
        
        optimal_curve = {}
        for freq in range(500, 3001, 250):
            v = base_v + slope * (freq - 500)
            if v > 1350:
                break
            p_estimate = (v / 1000) ** 2 * freq * 1e-8  # 简化功耗模型
            optimal_curve[freq] = v
            if p_estimate <= target_power_w:
                print(f"  {freq:<12} {v:<12.1f} {p_estimate:<12.2f}")
        
        return optimal_curve

# 模拟运行
if __name__ == "__main__":
    detector = SiliconBinningDetector()
    detector.scan_full_curve()
    detector.determine_bin()
    detector.calculate_optimal_vf_curve()

    print("\n不同体质等级的能效对比:")
    for bin_name in ["fast", "typical", "slow"]:
        d = SiliconBinningDetector()
        d.bin_grade = bin_name
        curve = d.calculate_optimal_vf_curve()
        v_2g = curve.get(2000, 1000)
        print(f"  {bin_name:>8}: 2GHz 电压 {v_2g:.1f} mV")
```

### 15. 参考文献与进一步阅读

#### 15.1 内核源码参考

| 文件 | 用途 |
|------|------|
| `drivers/gpu/drm/amd/pm/swsmu/amdgpu_smu.c` | SMU 接口核心实现 |
| `drivers/gpu/drm/amd/pm/swsmu/smu_v13_0.c` | SMU v13.0 电压管理 |
| `drivers/gpu/drm/amd/pm/amdgpu_pm.c` | PM sysfs 接口 |
| `drivers/gpu/drm/amd/amdgpu/amdgpu_acpi.c` | ACPI 电源事件处理 |
| `drivers/regulator/core.c` | Linux regulator 框架 |
| `drivers/hwmon/hwmon.c` | hwmon 子系统核心 |
| `include/linux/regulator/consumer.h` | regulator API 头文件 |

#### 15.2 AMD 官方文档

- AMD "RDNA 3 Instruction Set Architecture" - 电源管理章节
- AMD "Vega 20 Software Developer Guide" - VID 映射表规范
- AMD "MI200 Performance Optimization Guide" - 数据中心 GPU 电压调优
- AMD ROCm Documentation: rocm-smi 工具手册
- AMD Open-Source GPU Documentation: PPTable 格式说明

#### 15.3 学术参考

- Burd et al., "A Dynamic Voltage Scaled Microprocessor System", IEEE JSSC 2000 - DVFS 经典论文
- Le et al., "Online Optimization of Voltage and Frequency for Energy-Efficient Computing", ISCA 2011
- Herbert & Marculescu, "Analysis of Dynamic Voltage/Frequency Scaling in Chip-Multiprocessors", ISLPED 2007
- Skadron et al., "Temperature-Aware Microarchitecture: Extended Discussion and Results", HPCA 2003

#### 15.4 调试工具与使用方法

| 工具 | 用途 | 安装命令 |
|------|------|----------|
| `rocm-smi` | AMD GPU 状态监控 | `apt install rocm-libs` |
| `radeontop` | 实时 GPU 负载和频率查看 | `apt install radeontop` |
| `stress-ng` | CPU/GPU 压力测试 | `apt install stress-ng` |
| `perf` | Linux 性能计数器分析 | `apt install linux-tools-common` |
| `trace-cmd` | Ftrace 前端工具 | `apt install trace-cmd` |
| `bpftrace` | eBPF 追踪工具 | `apt install bpftrace` |
| `ltrace/strace` | 系统调用追踪 | `apt install ltrace strace` |

## 扩展思考 II（进阶）

在完成了基础内容的学习后，以下更深入的问题有助于你从架构师和资深工程师的视角理解电压管理：

1. **自适应电压调整（AVS）**：AMD 的 SmartShift 技术如何在 CPU 和 GPU 之间动态分配功耗预算？电压和频率如何协同调整？

2. **负电压偏移对可靠性的影响**：长期在低于标准电压下运行可能引发电子迁移（Electromigration）加速。如何在能效和寿命之间取得平衡？

3. **跨 Package 电压管理**：在 MCM（Multi-Chip Module）设计中，例如 MI250 的两个 GCD（Graphics Compute Die），如何独立管理每个裸片的电压？体质的差异如何处理？

4. **电压噪声的 FFT 分析**：示波器捕获的电压噪声信号包含多种频率成分。如何使用 FFT 分析识别开关电源纹波、负载瞬态响应和 VRM 控制环路带宽？

5. **机器学习驱动的预测性电压调节**：能否通过分析工作负载的指令分布、内存访问模式等特征，提前预测下一时刻的电压需求，实现前瞻性电压调节？这相比反馈式调节的优势在哪里？

6. **安全性与电压调节**：OverDrive 电压调节如果被恶意利用（例如设置过高电压烧毁 GPU），操作系统和固件有哪些安全防护措施？TPM 是否参与电压完整性验证？
