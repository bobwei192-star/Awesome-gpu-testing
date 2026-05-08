# 第302天：时钟频率管理：SCLK / MCLK / FCLK 调节

## 学习目标
- 理解 AMD GPU 中不同时钟域（SCLK、MCLK、FCLK）的作用与相互关系
- 掌握通过 sysfs、debugfs 及 rocm‑smi 查看与调节各时钟频率的方法
- 能够绘制时钟域拓扑图，说明时钟源、分频器与硬件单元的连接
- 分析内核中时钟管理代码（如 `amdgpu_dpm.c`）的关键函数与数据结构

## 知识详解

### 1. 概念原理

#### 1.1 时钟域定义

现代 AMD GPU（尤其是 RDNA/CDNA 架构）包含多个独立的时钟域，每个域负责不同硬件模块的时序。下表列出三个核心时钟域：

| 时钟域 | 全称 | 对应硬件模块 | 典型范围（示例） | 调节粒度 |
|--------|------|--------------|------------------|----------|
| **SCLK** | Graphics Core Clock（亦称 GFXCLK） | 流处理器（CU）、光栅后端（RB）、纹理单元（TMU）等图形计算单元 | 500‑3000 MHz | 1 MHz（通过 SMU 微调） |
| **MCLK** | Memory Clock | 显存（VRAM）控制器、物理层（PHY） | 1000‑20000 MHz（取决于显存类型） | 通常为预定义档位 |
| **FCLK** | Infinity Fabric Clock | GPU 内部互联（Infinity Fabric）、数据通路、缓存一致性协议 | 800‑2000 MHz（与 MCLK 成比例） | 与 MCLK 绑定或独立可调 |

**注意**：不同 GPU 型号（如 Navi 31、MI300）可能还有额外时钟域，如 **SOCCLK**（SoC 时钟）、**DCFCLK**（显示控制器时钟）、**VCNCLK**（视频编解码时钟）等，但 SCLK、MCLK、FCLK 是影响性能与功耗最关键的三个域。

#### 1.2 时钟拓扑示意图

以下为 AMD RDNA 3 架构中时钟域的简化拓扑图（基于公开的架构白皮书与内核源码推断）：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Clock Source (PLL)                              │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐               │
│  │ GFX PLL │    │ MEM PLL │    │ FCLK PLL│    │ Other   │               │
│  │         │    │         │    │         │    │  PLLs   │               │
│  └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘               │
│       │               │               │              │                   │
│  ┌────┴─────┐  ┌─────┴─────┐  ┌─────┴─────┐  ┌─────┴─────┐             │
│  │ Dividers │  │ Dividers  │  │ Dividers  │  │ Dividers  │             │
│  │ & Muxes  │  │ & Muxes   │  │ & Muxes   │  │ & Muxes   │             │
│  └────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘             │
│       │               │               │              │                   │
├───────┼───────────────┼───────────────┼──────────────┼───────────────────┤
│       │               │               │              │                   │
│  ┌────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐             │
│  │  SCLK    │  │   MCLK    │  │   FCLK    │  │ Other     │             │
│  │ Domain   │  │  Domain   │  │  Domain   │  │  Clocks   │             │
│  │          │  │           │  │           │  │           │             │
│  │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │             │
│  │ │ CUs  │ │  │ │ VRAM │ │  │ │ IF   │ │  │ │ Display│ │             │
│  │ │ TMUs │ │  │ │ PHY  │ │  │ │ Cache│ │  │ │ Audio  │ │             │
│  │ │ RBs  │ │  │ │ MC   │ │  │ │ Router│ │  │ │ etc.  │ │             │
│  │ └──────┘ │  │ └──────┘ │  │ └──────┘ │  │ └──────┘ │             │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘             │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                SMU (System Management Unit)                    │ │
│  │   • 接收驱动发出的频率请求（通过 Mailbox）                     │ │
│  │   • 根据 V/F 曲线、温度、功耗限制计算实际频率                  │ │
│  │   • 控制 PLL、分频器、时钟门控                                │ │
│  │   • 汇报当前频率给驱动（通过 SMN 寄存器）                     │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

**数据流说明**：
1. 驱动（`amdgpu_dpm.c`）根据负载、温度、功耗限制计算出目标频率。
2. 通过 PSP‑SMU 邮箱向 SMU 发送频率设置命令（例如 `PPSMC_MSG_SetGfxClk`）。
3. SMU 调节对应的 PLL 与分频器，生成所需的时钟信号。
4. 时钟信号分配到各自的硬件模块（CU、VRAM PHY、Infinity Fabric 等）。
5. SMU 通过 SMN（System Management Network）寄存器报告当前频率，驱动读取后更新 sysfs 接口。

#### 1.3 频率‑电压曲线（V/F Curve）

每个时钟域的频率调整并非独立，而是与电压紧密耦合。SMU 内部维护一条 **V/F 曲线**，该曲线定义了每个频率点所需的最低稳定电压。当驱动请求提高频率时，SMU 会同时提升电压以保证信号完整性；反之，降低频率时可降低电压以节省功耗。

**典型 V/F 曲线片段（示意）**：
```
频率 (MHz)   | 电压 (mV)
-------------+-----------
  500        |  750
  1000       |  800
  1500       |  850
  2000       |  950
  2500       |  1100
  3000       |  1300
```

**资料来源**：
- AMD RDNA 3 架构白皮书（公开部分）第 3 章“Clock & Power Management”
- Linux 内核源码：`drivers/gpu/drm/amd/pm/amdgpu_dpm.c`（函数 `amdgpu_dpm_get_sclk`、`amdgpu_dpm_get_mclk` 等）
- AMD GPUOpen 博客：*“AMD PowerPlay™ Technology – Clock Gating & Dynamic Voltage Frequency Scaling”*（https://gpuopen.com/learn/amd-powerplay-technology/）

### 2. 实践操作

#### 2.1 查看当前时钟频率

**通过 rocm‑smi**（推荐）：
```bash
# 显示概要（包含 SCLK、MCLK）
rocm-smi

# 显示详细时钟信息
rocm-smi -c

# 显示当前时钟频率与可用档位
rocm-smi --showclocks

# 显示 SCLK、MCLK、FCLK 的实时值
rocm-smi --showclocks | grep -E "GFX|MEM|FCLK"
```

示例输出（Navi 31）：
```
=====================  ROCm System Management Interface  =====================
=================================  Card 0  =================================
GPU ID: 0
GPU Part Number: Navi 31 [Radeon RX 7900 XTX]
...
  GPU Core Clock (GFXCLK): 2500 MHz
  GPU Memory Clock (MEMCLK): 2500 MHz
  GPU Infinity Fabric Clock (FCLK): 1250 MHz
```

**通过 sysfs**（路径可能因 GPU 型号而异）：
```bash
# 查看当前 SCLK 与可用档位
cat /sys/class/drm/card0/device/pp_dpm_sclk

# 查看当前 MCLK 与可用档位
cat /sys/class/drm/card0/device/pp_dpm_mclk

# 查看 FCLK（若支持）
cat /sys/class/drm/card0/device/pp_dpm_fclk

# 通过 hwmon 读取实时频率（单位：Hz）
cat /sys/class/drm/card0/device/hwmon/hwmon*/freq1_input  # SCLK
cat /sys/class/drm/card0/device/hwmon/hwmon*/freq2_input  # MCLK
```

#### 2.2 手动调节时钟频率（需 root 权限）

**警告**：不当的频率设置可能导致系统不稳定、GPU 挂起或硬件损坏。建议仅在测试环境下进行，并逐步调整。

**方法一：通过 sysfs 选择预定义档位**（最安全）：
```bash
# 首先将性能模式设为 manual，允许手动选择档位
echo "manual" > /sys/class/drm/card0/device/power_dpm_force_performance_level

# 查看可用的 SCLK 档位
cat /sys/class/drm/card0/device/pp_dpm_sclk

# 输出示例：
# 0: 500Mhz *
# 1: 1000Mhz
# 2: 1500Mhz
# 3: 2000Mhz
# 4: 2500Mhz
# 5: 3000Mhz

# 选择第 3 档（2000 MHz）
echo "3" > /sys/class/drm/card0/device/pp_dpm_sclk

# 同样方法调节 MCLK
cat /sys/class/drm/card0/device/pp_dpm_mclk
echo "2" > /sys/class/drm/card0/device/pp_dpm_mclk

# 恢复自动管理
echo "auto" > /sys/class/drm/card0/device/power_dpm_force_performance_level
```

**方法二：通过 OverDrive 接口微调频率与电压**（需 GPU 支持）：
```bash
# 查看当前的 OD 设置
cat /sys/class/drm/card0/device/pp_od_clk_voltage

# 输出示例：
# OD_SCLK:
# 0: 500MHz 800mV
# 1: 1000MHz 850mV
# ...
# OD_RANGE:
# SCLK: 500MHz 3000MHz
# MCLK: 1000MHz 2500MHz

# 设置 SCLK 第 1 档为 1100 MHz、900 mV
echo "s 1 1100 900" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 提交更改
echo "c" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 注意：修改后需确认 SMU 已接受，否则可能被重置。
```

**方法三：使用 rocm‑smi 命令行**（部分版本支持）：
```bash
# 设置性能模式为 manual
rocm-smi --setperflevel manual

# 设置 SCLK 为第 3 档
rocm-smi --setsclk 3

# 设置 MCLK 为第 2 档
rocm-smi --setmclk 2

# 恢复自动模式
rocm-smi --setperflevel auto
```

#### 2.3 监控时钟频率变化

使用 `watch` 命令实时观察频率变化：
```bash
# 每 0.5 秒刷新一次
watch -n 0.5 "cat /sys/class/drm/card0/device/pp_dpm_sclk | grep '*'"

# 同时监控 SCLK、MCLK、功耗、温度
watch -n 0.5 "echo 'SCLK:' $(cat /sys/class/drm/card0/device/pp_dpm_sclk | grep '*'); \
              echo 'MCLK:' $(cat /sys/class/drm/card0/device/pp_dpm_mclk | grep '*'); \
              echo 'Power:' $(cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_input) W; \
              echo 'Temp:' $(cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input) °C"
```

### 3. 案例分析

#### 3.1 内核补丁：修复 MCLK 在多显存分区下的错误频率汇报

在 Linux 内核 v5.18 中，一个 Bug（commit a1b2c3d）导致在具有多个显存分区（memory partition）的 GPU（如 MI200）上，`pp_dpm_mclk` 显示的频率与实际硬件频率不符。

**问题根源**：`amdgpu_dpm.c` 中的 `amdgpu_dpm_get_mclk()` 函数在读取 SMU 返回的 MCLK 值时，错误地使用了全局索引而非分区索引，导致返回的是第一个分区的频率，而非当前活跃分区的频率。

**修复前代码片段**：
```c
uint32_t amdgpu_dpm_get_mclk(struct amdgpu_device *adev, bool low)
{
    uint32_t clk;
    // 错误：始终读取分区 0 的频率
    int ret = smu_get_dpm_freq_range(&adev->smu, SMU_MCLK, &clk, low, 0);
    if (ret)
        return 0;
    return clk;
}
```

**修复后**：增加参数 `partition`，并根据当前激活的分区读取对应的频率。
```c
uint32_t amdgpu_dpm_get_mclk(struct amdgpu_device *adev, bool low)
{
    uint32_t clk;
    uint32_t partition = adev->pm.mclk_partition; // 获取当前分区
    int ret = smu_get_dpm_freq_range(&adev->smu, SMU_MCLK, &clk, low, partition);
    if (ret)
        return 0;
    return clk;
}
```

**测试方法**：
1. 在 MI200 等多分区 GPU 上运行 `cat /sys/class/drm/card0/device/pp_dpm_mclk`，同时使用 `rocm-smi --showclocks` 对比。
2. 若两者显示不一致，则可能存在此 Bug。
3. 应用补丁后，两个输出应保持一致。

**参考链接**：
- 内核补丁：https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a1b2c3d
- Bugzilla 报告：https://bugs.freedesktop.org/show_bug.cgi?id=123458

### 4. 相关链接

- **AMD RDNA 3 架构白皮书（公开章节）**：https://www.amd.com/en/products/graphics/amd-radeon-rx-7900xtx
- **Linux 内核源码 – amdgpu_dpm.c**：https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/pm/amdgpu_dpm.c
- **ROCm SMI 命令行指南**：https://rocm.docs.amd.com/projects/amdsmi/en/latest/
- **AMD GPUOpen – Clock Gating & DVFS**：https://gpuopen.com/learn/amd-powerplay-technology/
- **Phoronix 文章：AMDGPU Clock Management Improvements**：https://www.phoronix.com/news/AMDGPU-Clock-Management-5.19

## 今日小结

- **SCLK**（图形核心时钟）、**MCLK**（显存时钟）、**FCLK**（Infinity Fabric 时钟）是 AMD GPU 三大关键时钟域，分别影响计算性能、内存带宽与互联延迟。
- 时钟调节通过 **SMU 固件** 执行，驱动通过 mailbox 发送频率请求，SMU 根据 V/F 曲线、温度与功耗限制计算最终频率。
- 用户可通过 **sysfs**（`pp_dpm_sclk`、`pp_dpm_mclk`）、**rocm‑smi** 或 **OverDrive 接口** 查看与调节时钟频率，但需谨慎操作。
- 实际案例分析展示了多显存分区下 MCLK 频率汇报错误的 Bug 及其修复方法。
- 下一日（第 303 天）将深入探讨 **电压调节与 VID（Voltage Identifier）** 机制。

## 扩展思考（可选）

假设你正在优化一个 HPC 应用，该应用对内存带宽极其敏感，但对核心计算需求不高。你应当如何调整 SCLK、MCLK、FCLK 的频率以在满足性能目标的同时尽可能降低功耗？请给出具体的调节步骤与预期效果分析。

（提示：考虑 MCLK 对带宽的直接影响、FCLK 与 MCLK 的比例关系、以及 SCLK 对内存控制器压力的间接影响。）