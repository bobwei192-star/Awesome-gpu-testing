# 第366天：AMDGPU debugfs 接口全面解析

## 学习目标
- 全面掌握 AMDGPU 驱动提供的 debugfs 调试接口
- 理解每个 debugfs 文件的作用、数据来源和使用场景
- 能够通过 debugfs 接口诊断 GPU 驱动问题
- 掌握 debugfs 接口的注册机制和实现原理

## 知识详解

### 1. debugfs 概述

debugfs 是 Linux 内核提供的一种虚拟文件系统，用于向用户空间导出内核调试信息。与 procfs 和 sysfs 不同，debugfs 专门用于调试目的，不保证 ABI 稳定性，因此内核开发者可以自由地向其中添加调试接口。

AMDGPU 驱动充分利用了 debugfs，提供了丰富的调试接口，这些接口通常位于 `/sys/kernel/debug/dri/<num>/` 目录下，其中 `<num>` 为 DRM 设备编号（通常为 0）。

#### 1.1 debugfs 挂载

```bash
# 手动挂载 debugfs（通常系统已自动挂载）
sudo mount -t debugfs none /sys/kernel/debug

# 检查是否已挂载
mount | grep debugfs

# 查看 AMDGPU debugfs 入口
ls /sys/kernel/debug/dri/0/
```

#### 1.2 AMDGPU debugfs 文件分类

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AMDGPU DebugFS 接口总览                            │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  寄存器访问类                                                    │   │
│  │  ├── amdgpu_regs         MMIO 寄存器直接读写                    │   │
│  │  ├── amdgpu_regs2        IOCTL 接口寄存器访问                  │   │
│  │  ├── amdgpu_gfxoff       GFXOFF 状态控制                      │   │
│  │  └── amdgpu_smu_debug     SMU 调试接口                         │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  性能监控类                                                    │   │
│  │  ├── amdgpu_pm_info      电源管理完整状态信息                  │   │
│  │  ├── amdgpu_sensors      传感器读数（温度、功耗等）              │   │
│  │  ├── amdgpu_benchmark     DMA 引擎基准测试                     │   │
│  │  └── amdgpu_gpu_recovery  GPU 恢复控制                        │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  内存管理类                                                    │   │
│  │  ├── amdgpu_gem_info     GEM 对象信息                        │   │
│  │  ├── amdgpu_vm_info      虚拟内存信息                        │   │
│  │  ├── amdgpu_eviction_info  驱逐统计                         │   │
│  │  └── amdgpu_vram_info    VRAM 使用统计                      │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  可靠性类                                                      │   │
│  │  ├── ras/                   RAS 子系统接口                    │   │
│  │  │   ├── ras_ctrl          错误注入与查询                      │   │
│  │  │   ├── ras_eeprom_table   EEPROM 错误记录表                 │   │
│  │  │   ├── umc_err_count      UMC 错误计数                     │   │
│  │  │   └── gfx_err_count      GFX 错误计数                     │   │
│  │  └── amdgpu_ras_eeprom     RAS EEPROM 操作                   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  显示子系统类                                                  │   │
│  │  ├── amdgpu_dm_dc_info    DC 信息                            │   │
│  │  ├── amdgpu_dm_dtn_log    DCN 状态追踪日志                    │   │
│  │  ├── amdgpu_dm_visual_confirm  视觉确认                       │   │
│  │  ├── amdgpu_dm_psr/       PSR 调试目录                       │   │
│  │  └── amdgpu_dm_dmub/      DMUB 调试目录                     │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  固件信息类                                                    │   │
│  │  ├── amdgpu_firmware_info    固件版本信息                     │   │
│  │  ├── amdgpu_ip_info         IP 块信息                        │   │
│  │  └── amdgpu_gpu_metrics     GPU 度量数据                     │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 2. debugfs 核心注册机制

#### 2.1 内核 debugfs API

```c
// drivers/gpu/drm/amd/amdgpu/amdgpu_debugfs.c

#include <linux/debugfs.h>
#include <linux/seq_file.h>

/*
 * debugfs 核心 API 函数:
 *
 * debugfs_create_dir()  - 创建调试目录
 * debugfs_create_file() - 创建调试文件
 * debugfs_create_u32()  - 创建 u32 类型调试文件
 * debugfs_create_bool() - 创建 bool 类型调试文件
 * debugfs_create_x32()  - 创建十六进制 u32 调试文件
 */

// seq_file 操作结构体：允许内核生成多行文本输出
struct seq_operations {
    void *(*start)(struct seq_file *m, loff_t *pos);
    void (*stop)(struct seq_file *m, void *v);
    void *(*next)(struct seq_file *m, void *v, loff_t *pos);
    int (*show)(struct seq_file *m, void *v);
};

// file_operations 与 seq_file 的关联
static int debugfs_open(struct inode *inode, struct file *file)
{
    return single_open(file, debugfs_show_func, inode->i_private);
}

static const struct file_operations debugfs_fops = {
    .owner = THIS_MODULE,
    .open  = debugfs_open,
    .read  = seq_read,
    .llseek = seq_lseek,
    .release = single_release,
};
```

#### 2.2 AMDGPU debugfs 初始化流程

```c
// amdgpu_debugfs_init() - 注册所有 AMDGPU 调试接口
void amdgpu_debugfs_init(struct amdgpu_device *adev)
{
    struct drm_minor *minor = adev_to_drm(adev)->primary;
    struct dentry *root = minor->debugfs_root;
    struct dentry *dir;
    int i;

    // 1. 创建寄存器访问文件
    debugfs_create_file("amdgpu_regs", 0644, root,
                        adev, &amdgpu_debugfs_regs_fops);

    debugfs_create_file("amdgpu_regs2", 0644, root,
                        adev, &amdgpu_debugfs_regs2_fops);

    // 2. 创建电源管理文件
    debugfs_create_file("amdgpu_pm_info", 0444, root,
                        adev, &amdgpu_debugfs_pm_info_fops);

    // 3. 创建传感器文件
    debugfs_create_file("amdgpu_sensors", 0444, root,
                        adev, &amdgpu_debugfs_sensors_fops);

    // 4. 创建基准测试文件
    debugfs_create_file("amdgpu_benchmark", 0644, root,
                        adev, &amdgpu_debugfs_benchmark_fops);

    // 5. 创建 GEM 信息文件
    debugfs_create_file("amdgpu_gem_info", 0444, root,
                        adev, &amdgpu_debugfs_gem_info_fops);

    // 6. 创建 VRAM 信息文件
    debugfs_create_file("amdgpu_vram_info", 0444, root,
                        adev, &amdgpu_debugfs_vram_info_fops);

    // 7. 创建固件信息文件
    debugfs_create_file("amdgpu_firmware_info", 0444, root,
                        adev, &amdgpu_debugfs_firmware_info_fops);

    // 8. 创建 IP 信息文件
    debugfs_create_file("amdgpu_ip_info", 0444, root,
                        adev, &amdgpu_debugfs_ip_info_fops);

    // 9. 创建 GPU 度量文件
    debugfs_create_file("amdgpu_gpu_metrics", 0444, root,
                        adev, &amdgpu_debugfs_gpu_metrics_fops);

    // 10. 创建 RAS 目录（如果支持）
    if (adev->ras_enabled) {
        dir = debugfs_create_dir("ras", root);
        // RAS 子文件将在 RAS 初始化时创建
    }

    // 11. 创建显示调试目录
    if (adev->dm_enabled) {
        // DC 调试文件由 display 子系统注册
    }

    DRM_INFO("AMDGPU debugfs interface initialized\n");
}
```

#### 2.3 seq_file 输出机制

debugfs 文件的核心输出机制是 Linux 内核的 seq_file（序列文件）接口。它允许内核模块以流式方式生成大量文本输出，而不需要预先分配大缓冲区。

```c
/*
 * seq_file 操作示例：
 *
 * seq_printf()  - 格式化输出
 * seq_puts()    - 输出字符串
 * seq_putc()    - 输出单个字符
 * seq_hex_dump()- 输出十六进制转储
 * seq_write()   - 直接写入数据
 */

// amdgpu_pm_info 的 seq_file show 函数
static int amdgpu_debugfs_pm_info_show(struct seq_file *m, void *unused)
{
    struct amdgpu_device *adev = m->private;
    uint32_t gfx_clock, mem_clock, voltage, temperature;
    uint32_t power, fan_speed, gpu_load;

    // 读取 GPU 时钟
    gfx_clock = amdgpu_get_gfx_clock(adev);
    mem_clock = amdgpu_get_mem_clock(adev);

    // 读取电压
    voltage = amdgpu_get_gfx_voltage(adev);

    // 读取温度
    temperature = amdgpu_get_temperature(adev);

    // 读取功耗
    power = amdgpu_get_power(adev);

    // 读取风扇速度
    fan_speed = amdgpu_get_fan_speed(adev);

    // 读取 GPU 负载
    gpu_load = amdgpu_get_gpu_load(adev);

    // 输出信息头
    seq_printf(m, "AMDGPU Power Management Info\n");
    seq_printf(m, "============================\n\n");

    // 时钟信息
    seq_printf(m, "GFX Clock:         %u MHz\n", gfx_clock);
    seq_printf(m, "Memory Clock:      %u MHz\n", mem_clock);
    seq_printf(m, "\n");

    // 电压信息
    seq_printf(m, "GFX Voltage:       %u mV\n", voltage);
    seq_printf(m, "\n");

    // 温度信息
    seq_printf(m, "Temperature:       %u C\n", temperature);
    seq_printf(m, "\n");

    // 功耗信息
    seq_printf(m, "Power (average):   %u W\n", power);
    seq_printf(m, "\n");

    // 风扇信息
    seq_printf(m, "Fan Speed:         %u RPM\n", fan_speed);
    seq_printf(m, "\n");

    // 负载信息
    seq_printf(m, "GPU Load:          %u%%\n", gpu_load);

    return 0;
}
```

### 3. 寄存器访问类 debugfs 接口

#### 3.1 amdgpu_regs 接口

`amdgpu_regs` 是最基础的寄存器访问接口，允许通过 MMIO 直接读写 GPU 寄存器。

```bash
# 读取寄存器（偏移地址 0x1234）
cat /sys/kernel/debug/dri/0/amdgpu_regs
echo "0x1234" > /sys/kernel/debug/dri/0/amdgpu_regs
cat /sys/kernel/debug/dri/0/amdgpu_regs
# 输出示例：
# Register 0x1234: 0xABCDEF01

# 写入寄存器
echo "0x1234 0xDEADBEAF" > /sys/kernel/debug/dri/0/amdgpu_regs
```

```c
// amdgpu_regs 内核实现
static ssize_t amdgpu_debugfs_regs_read(struct file *f, char __user *buf,
                                         size_t size, loff_t *pos)
{
    struct amdgpu_device *adev = f->f_inode->i_private;
    ssize_t result = 0;
    uint32_t value;
    char tmp[32];

    // 偏移地址验证
    if (*pos & 0x3 || *pos >= adev->rmmio_size) {
        DRM_ERROR("Invalid register offset 0x%llx\n", *pos);
        return -EINVAL;
    }

    // 读取 MMIO 寄存器
    value = RREG32(*pos);

    // 格式化输出
    snprintf(tmp, sizeof(tmp), "0x%08X\n", value);
    if (copy_to_user(buf, tmp, strlen(tmp)))
        return -EFAULT;

    result = strlen(tmp);
    return result;
}

static ssize_t amdgpu_debugfs_regs_write(struct file *f,
                                          const char __user *buf,
                                          size_t size, loff_t *pos)
{
    struct amdgpu_device *adev = f->f_inode->i_private;
    unsigned long offset, value;
    char kbuf[128];

    // 从用户空间复制数据
    if (size >= sizeof(kbuf))
        return -EINVAL;

    if (copy_from_user(kbuf, buf, size))
        return -EFAULT;

    kbuf[size] = '\0';

    // 解析格式: "offset" 或 "offset value"
    if (sscanf(kbuf, "%lx %lx", &offset, &value) >= 1) {
        if (offset & 0x3 || offset >= adev->rmmio_size)
            return -EINVAL;

        // 设置读取/写入位置
        *pos = offset;

        if (value) {
            // 写入寄存器
            WREG32(offset, value);
            DRM_DEBUG("Write reg 0x%lx = 0x%lx\n", offset, value);
        }
    }

    return size;
}
```

#### 3.2 amdgpu_regs2 接口

`amdgpu_regs2` 是更现代的寄存器访问接口，支持更灵活的访问方式和更广泛的寄存器地址范围（包括 SMN 地址空间）。

```c
/*
 * amdgpu_regs2 IOCTL 接口
 *
 * 适用于需要原子性读-修改-写操作的场景
 * 支持 32 位完整 MMIO 地址范围
 * 支持 SRBM/GRBM bank 切换
 */

struct amdgpu_debugfs_regs2_data {
    uint32_t offset;    // 寄存器偏移地址
    uint32_t value;     // 读取/写入的值
    uint32_t mask;      // 位掩码（用于修改操作）
    uint32_t flags;     // 操作标志
};

// 操作标志定义
#define AMDGPU_REGS2_FLAGS_READ    0
#define AMDGPU_REGS2_FLAGS_WRITE   BIT(0)
#define AMDGPU_REGS2_FLAGS_MODIFY  BIT(1)
#define AMDGPU_REGS2_FLAGS_SMN     BIT(2)  // SMN 地址空间
#define AMDGPU_REGS2_FLAGS_SRBM    BIT(3)  // SRBM bank 选择

static int amdgpu_debugfs_regs2_ioctl(struct drm_device *dev, void *data,
                                       struct drm_file *filp)
{
    struct amdgpu_device *adev = drm_to_adev(dev);
    struct amdgpu_debugfs_regs2_data *args = data;
    uint32_t value;

    // 根据 flags 选择寄存器空间
    if (args->flags & AMDGPU_REGS2_FLAGS_SMN) {
        // SMN 地址空间访问
        if (args->flags & AMDGPU_REGS2_FLAGS_WRITE) {
            WREG32_SMN(args->offset, args->value);
        } else if (args->flags & AMDGPU_REGS2_FLAGS_MODIFY) {
            value = RREG32_SMN(args->offset);
            value &= ~args->mask;
            value |= (args->value & args->mask);
            WREG32_SMN(args->offset, value);
        } else {
            args->value = RREG32_SMN(args->offset);
        }
    } else {
        // MMIO 地址空间访问
        if (args->flags & AMDGPU_REGS2_FLAGS_WRITE) {
            WREG32(args->offset, args->value);
        } else if (args->flags & AMDGPU_REGS2_FLAGS_MODIFY) {
            value = RREG32(args->offset);
            value &= ~args->mask;
            value |= (args->value & args->mask);
            WREG32(args->offset, value);
        } else {
            args->value = RREG32(args->offset);
        }
    }

    return 0;
}
```

#### 3.3 amdgpu_smu_debug 接口

SMU 调试接口用于与 System Management Unit 固件交互，读取 SMU 状态和控制 SMU 行为。

```bash
# SMU 调试接口
cat /sys/kernel/debug/dri/0/amdgpu_smu_debug

# 输出示例：
# SMU Debug Information
# =====================
# SMU Firmware Version: 84.52.0
# SMU Feature Mask:     0xFFFFFFFF
# SMU Enabled Features: 23
# SMU Metrics Address:  0x00000000DEADBEEF

# 当前 PM 状态
# Power Source: AC
# Performance Level: Auto
#  ...
```

```c
// SMU debugfs 实现
static int amdgpu_debugfs_smu_info_show(struct seq_file *m, void *unused)
{
    struct amdgpu_device *adev = m->private;
    struct smu_context *smu = &adev->smu;
    struct smu_metrics metrics;
    int ret;

    seq_printf(m, "SMU Firmware Version: %u.%u.0\n",
               smu->smc_fw_version >> 16,
               (smu->smc_fw_version >> 8) & 0xFF);

    // 读取 SMU 度量数据
    ret = smu_get_metrics_table(smu, &metrics, sizeof(metrics));
    if (ret) {
        seq_printf(m, "Failed to get SMU metrics: %d\n", ret);
        return 0;
    }

    seq_printf(m, "\nSMU Metrics:\n");
    seq_printf(m, "-------------\n");
    seq_printf(m, "GFX Avg Clock:    %u MHz\n", metrics.GfxclkFrequency);
    seq_printf(m, "GFX Max Clock:    %u MHz\n", metrics.GfxclkFrequencyMax);
    seq_printf(m, "Mem Avg Clock:    %u MHz\n", metrics.MemclkFrequency);
    seq_printf(m, "SOC Avg Clock:    %u MHz\n", metrics.SocclkFrequency);
    seq_printf(m, "GFX Avg Voltage:  %u mV\n", metrics.GfxVoltage);
    seq_printf(m, "GFX Avg Power:    %u W\n", metrics.AverageGfxPower);
    seq_printf(m, "Socket Power:     %u W\n", metrics.CurrentSocketPower);
    seq_printf(m, "GFX Temperature:  %u C\n", metrics.GfxTemperature);
    seq_printf(m, "Memory Temp:      %u C\n", metrics.MemTemperature);
    seq_printf(m, "GFX Utilization:  %u%%\n", metrics.GfxBusyPercent);
    seq_printf(m, "Memory Bandwidth: %u MB/s\n",
               metrics.MemoryBandwidth);

    // VF 曲线信息
    seq_printf(m, "\nVoltage-Frequency Points:\n");
    for (int i = 0; i < metrics.NumVFPoints; i++) {
        seq_printf(m, "  Point %d: %u MHz @ %u mV\n",
                   i, metrics.VFCurve[i].frequency,
                   metrics.VFCurve[i].voltage);
    }

    return 0;
}
```

### 4. 性能监控类 debugfs 接口

#### 4.1 amdgpu_pm_info 接口

这是最常用的电源管理信息接口，提供 GPU 状态的全面快照。

```bash
# 读取完整电源管理信息
cat /sys/kernel/debug/dri/0/amdgpu_pm_info

# 典型输出：
# AMDGPU Power Management Info
# ============================
#
# GFX Clock:         2450 MHz
# Memory Clock:      1000 MHz
# SOC Clock:         1200 MHz
#
# GFX Voltage:       1150 mV
# SOC Voltage:       950 mV
# Memory Voltage:    1350 mV
#
# Temperature:       72 C
#
# Power (average):   185.0 W
# Power Limit (slow): 220.0 W
# Power Limit (fast): 250.0 W
#
# Fan Speed:         1850 RPM
# Fan Speed (%):     42%
# Fan PWM:           128
#
# GPU Load:          78%
# Memory Load:       15%
#
# GFX Clock Gating:  Enabled
# Memory Clock Gating: Enabled
# CG Flags:          0x0000003F
#
# Performance Level: Auto
# Power Supply Mode: AC
# Throttle Status:   Not Throttling
# Throttle Reasons:  None
#
# P-State:           6 (Max: 6)
# Current UCLK:      1000 MHz
# Current SOCLK:     1200 MHz
# Current FCLK:      1600 MHz
#
# VDDGFX:            1150 mV
# VDDCI:             900 mV
# MVDD:              1350 mV
```

```c
// amdgpu_pm_info 详细实现
static int amdgpu_debugfs_pm_info_show(struct seq_file *m, void *unused)
{
    struct amdgpu_device *adev = m->private;
    int ret;

    // 1. 获取 SMU 度量
    struct smu_context *smu = &adev->smu;
    struct smu_metrics metrics;

    ret = smu_get_metrics_table(smu, &metrics, sizeof(metrics));
    if (ret) {
        seq_printf(m, "Metrics unavailable: %d\n", ret);
        return 0;
    }

    // 2. 获取温度（通过多个传感器）
    uint32_t gfx_temp = amdgpu_get_temperature(adev);
    uint32_t mem_temp = amdgpu_get_mem_temperature(adev);
    uint32_t hotspot = amdgpu_get_hotspot_temperature(adev);

    // 3. 获取功耗
    uint32_t avg_power = amdgpu_get_average_power(adev);
    uint32_t slow_limit = amdgpu_get_power_limit(adev, POWER_LIMIT_SLOW);
    uint32_t fast_limit = amdgpu_get_power_limit(adev, POWER_LIMIT_FAST);

    // 4. 获取风扇信息
    uint32_t fan_speed = amdgpu_get_fan_speed(adev);
    uint32_t fan_pwm = amdgpu_get_fan_pwm(adev);

    // 5. 获取负载
    uint32_t gpu_busy = amdgpu_get_gpu_busy_percent(adev);
    uint32_t mem_busy = amdgpu_get_mem_busy_percent(adev);

    // 6. 获取节流状态
    uint32_t throttle_status = amdgpu_get_throttle_status(adev);
    const char *throttle_strs[] = {
        "Not Throttling",
        "Power Throttling",
        "Thermal Throttling",
        "Current Throttling"
    };

    seq_printf(m, "AMDGPU Power Management Info\n");
    seq_printf(m, "============================\n\n");

    // --- 时钟域 ---
    seq_printf(m, "--- Clock Domains ---\n");
    seq_printf(m, "GFX Clock:             %u MHz\n",
               metrics.GfxclkFrequency);
    seq_printf(m, "Memory Clock:          %u MHz\n",
               metrics.MemclkFrequency);
    seq_printf(m, "SOC Clock:             %u MHz\n",
               metrics.SocclkFrequency);
    seq_printf(m, "FCLK:                  %u MHz\n",
               metrics.FclkFrequency);
    seq_printf(m, "VCLK:                  %u MHz\n",
               metrics.VclkFrequency);
    seq_printf(m, "DCLK:                  %u MHz\n",
               metrics.DclkFrequency);
    seq_printf(m, "\n");

    // --- 电压 ---
    seq_printf(m, "--- Voltages ---\n");
    seq_printf(m, "GFX Voltage:           %u mV\n",
               metrics.GfxVoltage);
    seq_printf(m, "SOC Voltage:           %u mV\n",
               metrics.SocVoltage);
    seq_printf(m, "Memory Voltage:        %u mV\n",
               metrics.MemVddciVoltage);
    seq_printf(m, "\n");

    // --- 温度 ---
    seq_printf(m, "--- Temperatures ---\n");
    seq_printf(m, "GFX Temperature:       %u C\n", gfx_temp);
    seq_printf(m, "Memory Temperature:    %u C\n", mem_temp);
    seq_printf(m, "Hotspot Temperature:   %u C\n", hotspot);
    seq_printf(m, "\n");

    // --- 功耗 ---
    seq_printf(m, "--- Power ---\n");
    seq_printf(m, "Average Power:        %.1f W\n",
               metrics.AverageGfxPower / 1000.0);
    seq_printf(m, "Socket Power:         %.1f W\n",
               metrics.CurrentSocketPower / 1000.0);
    seq_printf(m, "Power Limit (slow):   %.1f W\n",
               slow_limit / 1000.0);
    seq_printf(m, "Power Limit (fast):   %.1f W\n",
               fast_limit / 1000.0);
    seq_printf(m, "\n");

    // --- 风扇 ---
    seq_printf(m, "--- Fan ---\n");
    seq_printf(m, "Fan Speed:            %u RPM\n", fan_speed);
    seq_printf(m, "Fan PWM:              %u/255\n", fan_pwm);
    seq_printf(m, "\n");

    // --- 负载 ---
    seq_printf(m, "--- Load ---\n");
    seq_printf(m, "GFX Busy:             %u%%\n", gpu_busy);
    seq_printf(m, "Memory Busy:          %u%%\n", mem_busy);
    seq_printf(m, "\n");

    // --- 电源状态 ---
    seq_printf(m, "--- Power State ---\n");
    seq_printf(m, "Performance Level:    %s\n",
               smu->power_profile_mode ? "Manual" : "Auto");
    seq_printf(m, "Throttle Status:      %s\n",
               throttle_strs[throttle_status]);
    seq_printf(m, "P-State:              %d\n",
               metrics.GfxclkPstate);
    seq_printf(m, "\n");

    // --- 时钟门控 ---
    seq_printf(m, "--- Clock Gating ---\n");
    seq_printf(m, "GFX Clock Gating:     %s\n",
               adev->cg_flags & AMD_CG_SUPPORT_GFX_MGCG ?
               "Enabled" : "Disabled");
    seq_printf(m, "Memory Clock Gating:  %s\n",
               adev->cg_flags & AMD_CG_SUPPORT_MC_MGCG ?
               "Enabled" : "Disabled");
    seq_printf(m, "CG Flags:             0x%08X\n",
               adev->cg_flags);

    return 0;
}
```

#### 4.2 amdgpu_sensors 接口

传感器接口提供对 GPU 上各种物理传感器的原始数据访问。

```bash
# 读取传感器数据
cat /sys/kernel/debug/dri/0/amdgpu_sensors

# 输出示例：
# AMDGPU Sensor Information
# ==========================
#
# Temperature Sensors:
#   GPU Temperature:      72.0 C
#   Memory Temperature:   68.0 C
#   Hotspot Temperature:  85.0 C
#   VR GFX Temperature:   65.0 C
#   VR MEM Temperature:   62.0 C
#   Liquid Temperature:   45.0 C
#   PLX Temperature:      55.0 C
#
# Power Sensors:
#   GPU Power:            185.0 W
#   Socket Power:         220.0 W
#   GFX Power:            150.0 W
#   SOC Power:            25.0 W
#   Memory Power:         30.0 W
#
# Current Sensors:
#   GFX Current:          120.0 A
#   SOC Current:          15.0 A
#
# Voltage Sensors:
#   GFX Voltage:          1.150 V
#   SOC Voltage:          0.950 V
#   Memory Voltage:       1.350 V
```

```c
// amdgpu_sensors 实现 - 多传感器读取
static int amdgpu_debugfs_sensors_show(struct seq_file *m, void *unused)
{
    struct amdgpu_device *adev = m->private;
    struct smu_context *smu = &adev->smu;
    struct smu_metrics metrics;

    if (smu_get_metrics_table(smu, &metrics, sizeof(metrics)))
        return 0;

    seq_printf(m, "AMDGPU Sensor Information\n");
    seq_printf(m, "==========================\n\n");

    // 温度传感器
    seq_printf(m, "Temperature Sensors:\n");
    seq_printf(m, "  GPU Temperature:      %.1f C\n",
               amdgpu_get_temperature(adev) / 10.0);
    seq_printf(m, "  Memory Temperature:   %.1f C\n",
               amdgpu_get_mem_temperature(adev) / 10.0);
    seq_printf(m, "  Hotspot Temperature:  %.1f C\n",
               amdgpu_get_hotspot_temperature(adev) / 10.0);
    seq_printf(m, "  Edge Temperature:     %.1f C\n",
               metrics.EdgeTemperature / 100.0);
    seq_printf(m, "\n");

    // 功耗传感器
    seq_printf(m, "Power Sensors:\n");
    seq_printf(m, "  GPU Power:            %.1f W\n",
               metrics.AverageGfxPower / 1000.0);
    seq_printf(m, "  Socket Power:         %.1f W\n",
               metrics.CurrentSocketPower / 1000.0);
    seq_printf(m, "\n");

    // 电流传感器
    seq_printf(m, "Current Sensors:\n");
    seq_printf(m, "  GFX Current:          %.1f A\n",
               metrics.GfxCurrent / 100.0);
    seq_printf(m, "  SOC Current:          %.1f A\n",
               metrics.SocCurrent / 100.0);
    seq_printf(m, "\n");

    // 电压传感器
    seq_printf(m, "Voltage Sensors:\n");
    seq_printf(m, "  GFX Voltage:          %.3f V\n",
               metrics.GfxVoltage / 1000.0);
    seq_printf(m, "  SOC Voltage:          %.3f V\n",
               metrics.SocVoltage / 1000.0);
    seq_printf(m, "  Memory Voltage:       %.3f V\n",
               metrics.MemVddciVoltage / 1000.0);

    return 0;
}
```

#### 4.3 amdgpu_benchmark 接口

DMA 引擎基准测试接口，用于测试 GPU 内存带宽。

```bash
# 运行基准测试
echo "1" > /sys/kernel/debug/dri/0/amdgpu_benchmark

# 读取结果
cat /sys/kernel/debug/dri/0/amdgpu_benchmark

# 输出示例：
# AMDGPU Benchmark Results
# ========================
#
# DMA Benchmark: 256 iterations
#   Copy 1MB:    12.5 GB/s
#   Copy 8MB:    14.2 GB/s
#   Copy 64MB:   15.8 GB/s
#   Copy 256MB:  16.1 GB/s
#   Fill 1MB:    18.3 GB/s
#   Fill 64MB:   22.5 GB/s
```

```c
// amdgpu_benchmark 实现
enum amdgpu_benchmark_mode {
    AMDGPU_BENCHMARK_COPY,
    AMDGPU_BENCHMARK_FILL,
};

struct amdgpu_benchmark_result {
    size_t size;
    double bandwidth;  // GB/s
};

static void amdgpu_benchmark_do_copy(struct amdgpu_device *adev,
                                      size_t size,
                                      struct amdgpu_benchmark_result *res)
{
    struct amdgpu_bo *src, *dst;
    int i, ret;
    ktime_t start, end;
    uint64_t elapsed_ns;

    // 分配源和目标 BO
    ret = amdgpu_bo_create_kernel(adev, size, PAGE_SIZE,
                                   AMDGPU_GEM_DOMAIN_VRAM,
                                   &src, NULL, NULL);
    if (ret)
        return;

    ret = amdgpu_bo_create_kernel(adev, size, PAGE_SIZE,
                                   AMDGPU_GEM_DOMAIN_VRAM,
                                   &dst, NULL, NULL);
    if (ret) {
        amdgpu_bo_free_kernel(&src, NULL, NULL);
        return;
    }

    // 执行复制并计时
    start = ktime_get();
    for (i = 0; i < 256; i++) {
        amdgpu_copy_mem_to_mem(adev, src, dst, size, 0, 0, NULL);
    }
    end = ktime_get();

    // 计算带宽
    elapsed_ns = ktime_to_ns(ktime_sub(end, start));
    res->size = size;
    res->bandwidth = (double)size * 256 / elapsed_ns * 1000;

    amdgpu_bo_free_kernel(&src, NULL, NULL);
    amdgpu_bo_free_kernel(&dst, NULL, NULL);
}

static int amdgpu_debugfs_benchmark_show(struct seq_file *m,
                                          void *unused)
{
    struct amdgpu_device *adev = m->private;
    struct amdgpu_benchmark_result result;

    seq_printf(m, "AMDGPU Benchmark Results\n");
    seq_printf(m, "========================\n\n");

    // 测试不同大小的拷贝
    seq_printf(m, "DMA Benchmark:\n");

    amdgpu_benchmark_do_copy(adev, 1 * 1024 * 1024, &result);
    seq_printf(m, "  Copy 1MB:    %.1f GB/s\n", result.bandwidth);

    amdgpu_benchmark_do_copy(adev, 8 * 1024 * 1024, &result);
    seq_printf(m, "  Copy 8MB:    %.1f GB/s\n", result.bandwidth);

    amdgpu_benchmark_do_copy(adev, 64 * 1024 * 1024, &result);
    seq_printf(m, "  Copy 64MB:   %.1f GB/s\n", result.bandwidth);

    amdgpu_benchmark_do_copy(adev, 256 * 1024 * 1024, &result);
    seq_printf(m, "  Copy 256MB:  %.1f GB/s\n", result.bandwidth);

    return 0;
}
```

### 5. 固件信息类 debugfs 接口

#### 5.1 amdgpu_firmware_info

```bash
# 查看所有固件版本信息
cat /sys/kernel/debug/dri/0/amdgpu_firmware_info

# 输出示例：
# AMDGPU Firmware Information
# ============================
#
# Feature Flags:       0x00000000
# Firmware Flags:      0x80000000
#
# Firmware Versions:
#   VCE:               0x07001234
#   UVD:               0x04001234
#   VCN:               0x01002200
#   MC:                0x00001234
#   ME:                0x00004A80
#   PFP:               0x00004A80
#   CE:                0x00004A80
#   RLC:               0x0000002F
#   MEC:               0x00004A82
#   MEC2:              0x00004A82
#   SDMA0:             0x00002B80
#   SDMA1:             0x00002B80
#   PSP:               0x00330043
#   SMU:               0x00540030
#   DMCU:              0x000000F0
#   DMCUB:             0x01020040
#
# IP Discovery:
#   Harvest Config:    0x00000000
#   Active IPs:
#     GC:              11.0.0
#     SDMA:            6.0.0
#     GMC:             11.0.0
#     PSP:             13.0.0
#     SMU:             13.0.0
#     DCN:             3.0.0
```

```c
// amdgpu_firmware_info 实现
static int amdgpu_debugfs_firmware_info_show(struct seq_file *m,
                                              void *unused)
{
    struct amdgpu_device *adev = m->private;

    seq_printf(m, "AMDGPU Firmware Information\n");
    seq_printf(m, "============================\n\n");

    seq_printf(m, "Feature Flags:       0x%08X\n", adev->feature_flags);
    seq_printf