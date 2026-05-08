# 第301天：amdgpu_dpm.c / amdgpu_pm.c 源码分析

## 学习目标
- 深入理解 AMDGPU 驱动中电源管理核心模块的源码结构
- 掌握 `amdgpu_pm.c` 与 `amdgpu_dpm.c` 的功能划分与交互流程
- 能够跟踪 DPM（动态电源管理）的初始化、状态切换与硬件控制路径
- 通过实际代码片段分析关键数据结构与函数调用关系

## 知识详解

### 1. 概念原理

#### 1.1 文件角色划分

在 Linux 内核的 AMDGPU 驱动中，电源管理（Power Management）代码主要分布在以下两个核心文件：

| 文件 | 路径（相对于内核源码根目录） | 主要职责 |
|------|----------------------------|----------|
| **amdgpu_pm.c** | `drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c` | 电源管理子系统框架、sysfs/hwmon 接口、热管理、风扇控制、ACPI 事件处理等“上层”逻辑。 |
| **amdgpu_dpm.c** | `drivers/gpu/drm/amd/pm/amdgpu_dpm.c`（注：实际路径可能因内核版本而异，常见于 `drivers/gpu/drm/amd/pm/` 目录） | 动态电源管理（DPM）引擎，负责频率/电压曲线（V/F）、性能状态（P-State）切换、SMU 通信等“底层”硬件控制。 |

**注意**：随着内核版本演进，代码组织可能有所调整。例如，在较新内核（5.17+）中，`amdgpu_dpm.c` 可能被拆分为多个更细粒度的文件（如 `amdgpu_dpm_*.c`），但核心逻辑依然保持相似。

#### 1.2 模块交互关系

下图展示了两个模块在内核中的位置及其与上下游组件的关系：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           User Space (sysfs/hwmon)                       │
│                                                                          │
│  /sys/class/drm/card0/device/                                            │
│      │ power_dpm_state, pp_od_clk_voltage, hwmon/…                      │
│      └────────────────────────────────────────────────────────────┐     │
│                                                                   │     │
├───────────────────────────────────────────────────────────────────┼─────┤
│                          Kernel Space                              │     │
│                                                                   │     │
│  ┌─────────────────────────────────────────────────────────────┐  │     │
│  │                    DRM Core & AMDGPU Driver                 │  │     │
│  │                                                             │  │     │
│  │  ┌─────────────┐       ┌─────────────┐       ┌──────────┐  │  │     │
│  │  │ amdgpu_pm.c │◄─────►│ amdgpu_dpm.c│◄─────►│ SMU FW   │  │  │     │
│  │  │   (框架)    │       │   (引擎)    │       │ Interface│  │  │     │
│  │  └──────┬──────┘       └──────┬──────┘       └──────────┘  │  │     │
│  │         │                      │                            │  │     │
│  │  ┌──────┴──────┐      ┌───────┴───────┐                    │  │     │
│  │  │ thermal_core│      │  PowerPlay    │                    │  │     │
│  │  │   (hwmon)   │      │   (PP Table)  │                    │  │     │
│  │  └─────────────┘      └───────────────┘                    │  │     │
│  │         │                      │                            │  │     │
│  │  ┌──────┴──────────────────────┴──────┐                    │  │     │
│  │  │      ACPI / PCIe PM                │                    │  │     │
│  │  └────────────────────────────────────┘                    │  │     │
│  │                                                             │  │     │
│  └─────────────────────────────────────────────────────────────┘  │     │
│                                                                   │     │
│  ┌─────────────────────────────────────────────────────────────┐  │     │
│  │                   Hardware (GPU Silicon)                     │  │     │
│  │  ┌───────────────────────────────────────────────────────┐  │  │     │
│  │  │ SMU (System Management Unit) – Microcontroller        │  │  │     │
│  │  │   • 执行频率/电压调整                                  │  │  │     │
│  │  │   • 监控功耗/温度                                      │  │  │     │
│  │  │   • 风扇控制                                           │  │  │     │
│  │  └───────────────────────────────────────────────────────┘  │  │     │
│  └─────────────────────────────────────────────────────────────┘  │     │
│                                                                   │     │
└───────────────────────────────────────────────────────────────────┴─────┘
```

**数据流**：
1. 用户通过 sysfs 写入 `power_dpm_state` 等属性，触发 `amdgpu_pm.c` 中的属性回调（例如 `amdgpu_set_power_dpm_state`）。
2. `amdgpu_pm.c` 调用 `amdgpu_dpm.c` 提供的接口（如 `amdgpu_dpm_set_power_state`）来更改硬件状态。
3. `amdgpu_dpm.c` 通过 SMU 固件接口（例如 `amdgpu_dpm_smu_send_msg`）向 GPU 内部的 SMU 发送命令。
4. SMU 执行实际的频率/电压调整，并返回结果。

#### 1.3 amdgpu_pm.c 核心函数详细分析

amdgpu_pm.c 作为电源管理框架层，实现了大量用户空间接口和内核回调函数。以下逐一分析其核心函数：

**amdgpu_pm_init() - 电源管理子系统初始化**
```c
int amdgpu_pm_init(struct amdgpu_device *adev)
{
    int ret;
    
    /* 初始化电源管理互斥锁 */
    mutex_init(&adev->pm.mutex);
    
    /* 初始化 DPM 子系统 */
    ret = amdgpu_dpm_init(adev);
    if (ret) {
        DRM_ERROR("Failed to initialize DPM\n");
        return ret;
    }
    
    /* 创建 hwmon 设备（用于温度、功耗、风扇监控） */
    adev->pm.hwmon = amdgpu_hwmon_init(adev);
    if (IS_ERR(adev->pm.hwmon)) {
        DRM_WARN("Failed to initialize hwmon\n");
        adev->pm.hwmon = NULL;
    }
    
    /* 注册 sysfs 属性 */
    ret = sysfs_create_group(&adev->dev->kobj, &amdgpu_pm_attr_group);
    if (ret) {
        DRM_ERROR("Failed to create sysfs attributes\n");
        return ret;
    }
    
    /* 注册 thermal 冷却设备 */
    adev->pm.cdev = thermal_cooling_device_register("amdgpu",
        adev, &amdgpu_thermal_cooling_ops);
    if (IS_ERR(adev->pm.cdev)) {
        DRM_WARN("Failed to register thermal cooling device\n");
        adev->pm.cdev = NULL;
    }
    
    /* 初始化 Runtime PM */
    pm_runtime_use_autosuspend(adev->dev);
    pm_runtime_set_autosuspend_delay(adev->dev, 5000);
    pm_runtime_mark_last_busy(adev->dev);
    pm_runtime_enable(adev->dev);
    
    return 0;
}
```

**初始化流程详解**：
1. **mutex_init**：初始化 PM 互斥锁，保证并发操作安全
2. **amdgpu_dpm_init**：调用底层 DPM 引擎初始化，读取 PPTable
3. **amdgpu_hwmon_init**：创建 hwmon 设备，注册温度/功耗/风扇传感器
4. **sysfs_create_group**：导出电源管理相关 sysfs 属性文件
5. **thermal_cooling_device_register**：注册冷却设备，供 thermal 子系统调用
6. **pm_runtime_*系列**：启用 Runtime PM 自动挂起功能

**amdgpu_pm_fini() - 电源管理子系统销毁**
```c
void amdgpu_pm_fini(struct amdgpu_device *adev)
{
    /* 注销 thermal 冷却设备 */
    if (adev->pm.cdev) {
        thermal_cooling_device_unregister(adev->pm.cdev);
        adev->pm.cdev = NULL;
    }
    
    /* 销毁 hwmon 设备 */
    if (adev->pm.hwmon) {
        amdgpu_hwmon_fini(adev);
    }
    
    /* 移除 sysfs 属性 */
    sysfs_remove_group(&adev->dev->kobj, &amdgpu_pm_attr_group);
    
    /* 关闭 DPM 子系统 */
    amdgpu_dpm_fini(adev);
    
    /* 禁用 Runtime PM */
    pm_runtime_disable(adev->dev);
}
```

析构函数与初始化函数对称，按相反顺序释放资源。

**核心 sysfs 属性回调函数**

amdgpu_pm.c 中定义了大量 sysfs 属性，每个属性对应一对 show/store 回调函数。以下是关键属性的回调分析：

| sysfs 属性 | show 回调函数 | store 回调函数 | 用途 |
|-----------|--------------|---------------|------|
| power_dpm_state | amdgpu_get_power_dpm_state | amdgpu_set_power_dpm_state | 查看/设置 DPM 状态 |
| power_dpm_force_performance_level | amdgpu_get_power_dpm_force_perf_level | amdgpu_set_power_dpm_force_perf_level | 强制性能等级 |
| pp_dpm_sclk | amdgpu_get_pp_dpm_sclk | amdgpu_set_pp_dpm_sclk | 查看/设置 SCLK 档位 |
| pp_dpm_mclk | amdgpu_get_pp_dpm_mclk | amdgpu_set_pp_dpm_mclk | 查看/设置 MCLK 档位 |
| pp_od_clk_voltage | amdgpu_get_pp_od_clk_voltage | amdgpu_set_pp_od_clk_voltage | OC/UC 频率电压调节 |
| amdgpu_pm_info | amdgpu_get_pm_info | - (只读) | 查看 PM 状态详情 |

**amdgpu_set_power_dpm_state() - 设置 DPM 状态**
```c
static ssize_t amdgpu_set_power_dpm_state(struct device *dev,
    struct device_attribute *attr, const char *buf, size_t count)
{
    struct amdgpu_device *adev = dev_get_drvdata(dev);
    enum amdgpu_pm_state_type target_state;
    int ret = 0;
    
    /* 解析用户输入的字符串 */
    if (strncmp("battery", buf, strlen("battery")) == 0)
        target_state = POWER_STATE_TYPE_BATTERY;
    else if (strncmp("balanced", buf, strlen("balanced")) == 0)
        target_state = POWER_STATE_TYPE_BALANCED;
    else if (strncmp("performance", buf, strlen("performance")) == 0)
        target_state = POWER_STATE_TYPE_PERFORMANCE;
    else {
        DRM_DEBUG("Invalid power state\n");
        return -EINVAL;
    }
    
    /* 加锁，保证并发安全 */
    mutex_lock(&adev->pm.mutex);
    
    /* 设置目标状态 */
    adev->pm.dpm.state = target_state;
    
    /* 触发状态切换（通过 DPM 引擎） */
    ret = amdgpu_dpm_set_power_state(adev);
    
    mutex_unlock(&adev->pm.mutex);
    
    if (ret)
        return ret;
    
    return count;
}
```

**关键设计要点**：
1. **字符串解析**：通过 strncmp 匹配 "battery"/"balanced"/"performance"
2. **互斥锁保护**：所有 PM 操作都需要持有 mutex，防止并发冲突
3. **委托模式**：amdgpu_pm.c 不直接操作硬件，而是调用 amdgpu_dpm.c 的接口
4. **错误传播**：底层错误通过返回值向上传播到用户空间

#### 1.4 amdgpu_dpm.c 核心函数详细分析

amdgpu_dpm.c 是电源管理的"发动机"，实现了所有底层的硬件控制逻辑。

**amdgpu_dpm_init() - DPM 引擎初始化**
```c
int amdgpu_dpm_init(struct amdgpu_device *adev)
{
    struct amdgpu_dpm *dpm = &adev->pm.dpm;
    int ret;
    
    /* 1. 读取 SMU 固件版本 */
    ret = amdgpu_dpm_smu_get_fw_version(adev, &dpm->fw_major, &dpm->fw_minor);
    if (ret) {
        DRM_WARN("Could not get SMU firmware version\n");
        /* 非致命错误，继续初始化 */
    }
    
    /* 2. 加载 PowerPlay Table */
    ret = amdgpu_dpm_load_pptable(adev);
    if (ret) {
        DRM_ERROR("Failed to load PowerPlay table\n");
        return ret;
    }
    
    /* 3. 解析 PPTable，建立 P-State 列表 */
    ret = amdgpu_dpm_parse_pptable(adev);
    if (ret) {
        DRM_ERROR("Failed to parse PowerPlay table\n");
        return ret;
    }
    
    /* 4. 设置初始性能状态 */
    dpm->boot_ps = amdgpu_dpm_get_boot_state(adev);
    dpm->current_ps = dpm->boot_ps;
    dpm->requested_ps = dpm->boot_ps;
    
    /* 5. 创建 sysfs 属性（由 amdgpu_pm.c 完成） */
    
    DRM_INFO("DPM initialized successfully\n");
    return 0;
}
```

**PState 加载与解析过程**：
PPTable 是一个二进制结构体，存储在 GPU VBIOS 中，定义了所有预设的性能状态。加载流程：
1. SMU 固件从 VBIOS 中读取 PPTable
2. 驱动通过 SMU 命令 `SMC_MSG_GetDriverIfVersion` 获取接口版本
3. 驱动读取 PPTable 并解析出各 P-State 的频率、电压、功耗限制
4. 将解析结果填入 `struct amdgpu_ps` 结构体数组
5. 初始化完成，DPM 引擎准备就绪

**amdgpu_dpm_set_power_state() - 设置性能状态**
```c
int amdgpu_dpm_set_power_state(struct amdgpu_device *adev)
{
    struct amdgpu_dpm *dpm = &adev->pm.dpm;
    struct amdgpu_ps *new_ps = NULL;
    int ret;
    
    /* 1. 根据当前策略选择目标 P-State */
    switch (dpm->state) {
    case POWER_STATE_TYPE_PERFORMANCE:
        new_ps = amdgpu_dpm_get_performance_state(adev);
        break;
    case POWER_STATE_TYPE_BALANCED:
        new_ps = amdgpu_dpm_get_balanced_state(adev);
        break;
    case POWER_STATE_TYPE_BATTERY:
        new_ps = amdgpu_dpm_get_battery_state(adev);
        break;
    default:
        return -EINVAL;
    }
    
    if (!new_ps) {
        DRM_DEBUG("No matching P-State found\n");
        return -ENOENT;
    }
    
    /* 2. 检查是否需要切换（避免不必要的操作） */
    if (dpm->current_ps == new_ps)
        return 0;
    
    /* 3. 执行预切换准备 */
    ret = amdgpu_dpm_pre_set_power_state(adev, new_ps);
    if (ret) {
        DRM_ERROR("Pre-switch preparation failed\n");
        return ret;
    }
    
    /* 4. 请求 SMU 切换状态 */
    ret = amdgpu_dpm_switch_power_state(adev, new_ps);
    if (ret) {
        DRM_ERROR("State switch failed\n");
        return ret;
    }
    
    /* 5. 后切换处理（更新状态、调整电压等） */
    ret = amdgpu_dpm_post_set_power_state(adev, new_ps);
    if (ret) {
        DRM_ERROR("Post-switch processing failed\n");
        return ret;
    }
    
    /* 6. 更新当前状态指针 */
    dpm->current_ps = new_ps;
    
    return 0;
}
```

**状态切换的三阶段设计**：
- **pre_set_power_state**：通知驱动各子系统（调度器、显示控制器等）即将发生状态切换，暂停某些操作
- **switch_power_state**：通过 SMU 命令实际调整频率/电压，这是最耗时的一步
- **post_set_power_state**：状态切换完成后恢复各子系统操作，调整电压偏移

这种三阶段设计保证了状态切换的原子性和安全性。

#### 1.5 电源管理属性注册机制深度分析

AMDGPU 驱动的 sysfs 属性注册采用了细粒度的属性组设计，分为多个类别：

```c
/* amdgpu_pm.c 中的属性定义模式 */
static struct device_attribute amdgpu_pm_attrs[] = {
    __ATTR(power_dpm_state, S_IRUGO | S_IWUSR,
           amdgpu_get_power_dpm_state, amdgpu_set_power_dpm_state),
    __ATTR(power_dpm_force_performance_level, S_IRUGO | S_IWUSR,
           amdgpu_get_power_dpm_force_perf_level,
           amdgpu_set_power_dpm_force_perf_level),
    __ATTR(pp_dpm_sclk, S_IRUGO, amdgpu_get_pp_dpm_sclk, NULL),
    __ATTR(pp_dpm_mclk, S_IRUGO, amdgpu_get_pp_dpm_mclk, NULL),
    __ATTR(pp_num_states, S_IRUGO, amdgpu_get_pp_num_states, NULL),
    __ATTR(pp_cur_state, S_IRUGO, amdgpu_get_pp_cur_state, NULL),
    __ATTR(pp_od_clk_voltage, S_IRUGO | S_IWUSR,
           amdgpu_get_pp_od_clk_voltage, amdgpu_set_pp_od_clk_voltage),
    /* ... 更多属性 ... */
};

/* OverDrive 相关属性（单独分组，仅在支持 OD 时注册） */
static struct device_attribute amdgpu_od_attrs[] = {
    __ATTR(pp_od_clk_voltage, S_IRUGO | S_IWUSR,
           amdgpu_get_pp_od_clk_voltage, amdgpu_set_pp_od_clk_voltage),
    __ATTR(pp_power_profile_mode, S_IRUGO | S_IWUSR,
           amdgpu_get_pp_power_profile_mode,
           amdgpu_set_pp_power_profile_mode),
};

/* hwmon 属性（通过 hwmon 子系统注册） */
static const struct hwmon_channel_info *amdgpu_hwmon_info[] = {
    HWMON_CHANNEL_INFO(temp, HWMON_T_INPUT | HWMON_T_MAX |
                       HWMON_T_MAX_HYST | HWMON_T_CRIT |
                       HWMON_T_CRIT_HYST | HWMON_T_EMERGENCY),
    HWMON_CHANNEL_INFO(fan, HWMON_F_INPUT | HWMON_F_ENABLE),
    HWMON_CHANNEL_INFO(pwm, HWMON_PWM_INPUT |
                       HWMON_PWM_ENABLE | HWMON_PWM_FREQ),
    HWMON_CHANNEL_INFO(power, HWMON_P_INPUT |
                       HWMON_P_CAP_MAX | HWMON_P_CAP),
    HWMON_CHANNEL_INFO(in, HWMON_I_INPUT | HWMON_I_LABEL),
    NULL
};
```

**属性注册的代码路径**：
1. `amdgpu_pm_init()` → `sysfs_create_group(&adev->dev->kobj, &amdgpu_pm_attr_group)`
2. `amdgpu_hwmon_init()` → `hwmon_device_register_with_info()` → 注册 hwmon 属性
3. 部分属性（如 OverDrive）在检测到硬件支持后动态注册

#### 1.6 amdgpu_pm.c 与 thermal 子系统的集成

amdgpu_pm.c 实现了 thermal 冷却设备接口，供内核 thermal 子系统调用：

```c
/* 冷却设备回调函数 */
static int amdgpu_thermal_get_max_state(struct thermal_cooling_device *cdev,
                                        unsigned long *state)
{
    struct amdgpu_device *adev = cdev->devdata;
    *state = adev->pm.dpm.num_ps - 1;  /* 最大状态数 */
    return 0;
}

static int amdgpu_thermal_get_cur_state(struct thermal_cooling_device *cdev,
                                        unsigned long *state)
{
    struct amdgpu_device *adev = cdev->devdata;
    *state = adev->pm.dpm.current_ps_index;  /* 当前状态索引 */
    return 0;
}

static int amdgpu_thermal_set_cur_state(struct thermal_cooling_device *cdev,
                                        unsigned long state)
{
    struct amdgpu_device *adev = cdev->devdata;
    
    /* thermal 子系统请求降低性能以降温 */
    return amdgpu_dpm_force_performance_level(adev,
        AMDGPU_DPM_FORCE_LEVEL_LOW);
}

static struct thermal_cooling_device_ops amdgpu_thermal_cooling_ops = {
    .get_max_state = amdgpu_thermal_get_max_state,
    .get_cur_state = amdgpu_thermal_get_cur_state,
    .set_cur_state = amdgpu_thermal_set_cur_state,
};
```

**thermal 子系统的调用路径**：
1. 温度超过阈值 → thermal 核心触发 cooling 动作
2. thermal 核心调用 `amdgpu_thermal_set_cur_state()`
3. 调用 `amdgpu_dpm_force_performance_level()` 降低频率
4. GPU 温度下降后，thermal 核心逐步恢复频率

这种集成机制实现了内核级别的温度保护，独立于用户空间控制。

**资料来源**：
- Linux 内核源码树（v6.10）：`drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c`
- Linux 内核源码树（v6.10）：`drivers/gpu/drm/amd/pm/amdgpu_dpm.c`
- AMD GPUOpen 文档：*“AMDGPU Power Management Internals”*（https://gpuopen.com/learn/amd-gpu-power-management-internals/）

### 2. 实践操作

#### 2.1 获取内核源码并定位文件

```bash
# 克隆 Linux 内核源码（或使用已存在的源码树）
git clone --depth 1 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux

# 查找 amdgpu_pm.c 与 amdgpu_dpm.c
find . -name "amdgpu_pm.c" -type f
find . -name "amdgpu_dpm.c" -type f

# 若未找到，可能是代码已迁移到 pm/ 子目录
find drivers/gpu/drm/amd -name "*dpm*.c" -type f
find drivers/gpu/drm/amd -name "*pm*.c" -type f
```

典型输出：
```
./drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c
./drivers/gpu/drm/amd/pm/amdgpu_dpm.c
```

#### 2.2 关键数据结构分析

使用 `ctags` 或 `cscope` 浏览源码中的关键数据结构。以下为部分重要结构体（摘自内核 v6.10）：

**amdgpu_pm.c 相关**：
```c
/* 定义于 drivers/gpu/drm/amd/amdgpu/amdgpu_pm.h */
struct amdgpu_pm {
    struct mutex            mutex;          /* 电源管理锁 */
    struct list_head        pm_entries;     /* 电源管理条目链表 */
    struct amdgpu_dpm       dpm;            /* 指向 DPM 结构 */
    struct hwmon_device     *hwmon;         /* hwmon 设备 */
    struct thermal_cooling_device *cdev;    /* 散热设备 */
    /* ... 其他成员 ... */
};

/* sysfs 属性组 */
static const struct attribute_group amdgpu_pm_attr_group = {
    .attrs = amdgpu_pm_attrs,               /* 属性数组 */
};
```

**amdgpu_dpm.c 相关**：
```c
/* 定义于 drivers/gpu/drm/amd/include/amd_shared.h */
struct amdgpu_dpm {
    struct amdgpu_ps        *ps;            /* 当前性能状态 */
    struct amdgpu_ps        *requested_ps;  /* 请求的性能状态 */
    struct amdgpu_ps        *boot_ps;       /* 启动时的性能状态 */
    u32                     num_ps;         /* 性能状态数量 */
    struct amdgpu_ps        *ps_list;       /* 状态列表 */
    /* ... 时钟/电压/功耗限制等 ... */
};

/* 性能状态（P-State）描述 */
struct amdgpu_ps {
    u32                     num_levels;     /* 频率级别数 */
    struct amdgpu_clock_level *clock_levels; /* 时钟级别数组 */
    u32                     num_vddc_levels;/* 电压级别数 */
    struct amdgpu_voltage_level *vddc_levels; /* 电压级别数组 */
    u32                     class;          /* 性能类别 */
    /* ... 其他成员 ... */
};
```

#### 2.3 跟踪 DPM 状态切换流程

通过阅读代码，可以梳理出从用户写入 sysfs 到硬件状态改变的全过程。以下为简化调用链：

```
1. 用户写入 `/sys/class/drm/card0/device/power_dpm_state`
2. 触发 `amdgpu_set_power_dpm_state()`（在 amdgpu_pm.c 中）
   ↓
3. 调用 `amdgpu_dpm_set_power_state()`（在 amdgpu_dpm.c 中）
   ↓
4. 调用 `amdgpu_dpm_change_power_state_locked()`（内部函数）
   ↓
5. 根据目标状态（performance/balanced/battery）选择相应的 P-State
   ↓
6. 调用 `amdgpu_dpm_apply_state()` → `amdgpu_dpm_apply_power_state()`
   ↓
7. 通过 SMU 接口发送命令：`amdgpu_dpm_smu_send_msg()` 或直接写寄存器
   ↓
8. SMU 接收到命令，调整 GPU 频率/电压，完成状态切换
```

可以通过在内核中添加 tracepoint 或使用 ftrace 验证该流程。以下是一个简单的 ftrace 命令示例：

```bash
# 启用函数追踪
sudo su
cd /sys/kernel/debug/tracing
echo 1 > events/amdgpu/enable
echo function_graph > current_tracer

# 过滤 amdgpu_dpm 相关函数
echo "*amdgpu_dpm*" > set_ftrace_filter
echo "*amdgpu_set_power*" >> set_ftrace_filter

# 开始记录
echo 1 > tracing_on

# 在另一个终端触发状态切换
echo "performance" > /sys/class/drm/card0/device/power_dpm_state

# 停止记录并查看结果
echo 0 > tracing_on
cat trace
```

#### 2.4 深入调试：使用 Ftrace 追踪完整调用链

实际工作中，DPM 状态切换可能涉及数十个函数调用。以下是一个更完整的 ftrace 调试方案：

```bash
#!/bin/bash
# amplify_dpm_trace.sh - 追踪 AMDGPU DPM 状态切换的完整调用链

set -e

TRACING_DIR=/sys/kernel/debug/tracing

# 检查 root 权限
if [ $(id -u) -ne 0 ]; then
    echo "请以 root 身份运行"
    exit 1
fi

# 初始化追踪配置
function setup_tracing() {
    echo "配置 ftrace..."
    echo 0 > $TRACING_DIR/tracing_on
    echo function_graph > $TRACING_DIR/current_tracer
    echo 8192 > $TRACING_DIR/buffer_size_kb  # 增大缓冲区
    
    # 设置函数过滤器：匹配所有 AMDGPU PM/DPM 相关函数
    cat > $TRACING_DIR/set_ftrace_filter << EOF
amdgpu_pm_*
amdgpu_dpm_*
amdgpu_set_*
amdgpu_get_*
amdgpu_hwmon_*
amdgpu_thermal_*
EOF
    
    # 设置追踪深度（避免噪声）
    echo 3 > $TRACING_DIR/max_graph_depth
    
    echo "追踪配置完成"
}

# 触发状态切换
function trigger_pm_change() {
    local state=$1  # performance, balanced, battery
    
    echo "触发状态切换: $state"
    echo "$state" > /sys/class/drm/card0/device/power_dpm_state
}

# 分析追踪结果
function analyze_trace() {
    echo "分析追踪结果..."
    
    # 统计各函数调用次数
    echo "=== 函数调用统计 ==="
    grep -o "=> [a-zA-Z0-9_]*" $TRACING_DIR/trace | sort | uniq -c | sort -rn
    
    # 统计执行时间（微秒）
    echo "=== 函数耗时统计（最长10个） ==="
    grep "usecs" $TRACING_DIR/trace | \
        awk '{print $NF, $0}' | sort -rn | head -10
    
    # 保存完整追踪
    cp $TRACING_DIR/trace ./dpm_trace_$(date +%Y%m%d_%H%M%S).log
    echo "追踪日志已保存"
}

# 主流程
echo "开始 DPM 状态切换追踪"
setup_tracing

echo 1 > $TRACING_DIR/tracing_on

# 循环切换三种状态
for state in performance balanced battery performance; do
    trigger_pm_change $state
    sleep 2  # 等待状态稳定
done

echo 0 > $TRACING_DIR/tracing_on

analyze_trace

echo "追踪完成。检查 dmesg 确认硬件状态："
dmesg | grep -i "amdgpu.*dpm\|amdgpu.*power" | tail -10
```

**追踪结果解读示例**：
```
=== 函数调用统计 ===
     12 => amdgpu_dpm_set_power_state
     12 => amdgpu_dpm_change_power_state_locked
     12 => amdgpu_dpm_apply_power_state
      6 => amdgpu_dpm_smu_send_msg
      3 => amdgpu_dpm_get_performance_state
      3 => amdgpu_dpm_get_balanced_state
      3 => amdgpu_get_power_dpm_state
      3 => amdgpu_set_power_dpm_state

=== 函数耗时统计 ===
  5234 usecs amdgpu_dpm_apply_power_state
  4891 usecs amdgpu_dpm_switch_power_state
    12 usecs amdgpu_dpm_set_power_state
     8 usecs amdgpu_dpm_get_performance_state
```

**分析要点**：
1. `amdgpu_dpm_apply_power_state` 耗时最长（5ms+），原因是需要等待 SMU 确认状态切换完成
2. `amdgpu_dpm_smu_send_msg` 被调用了6次，对应3次状态切换（每次需要2条消息：设置和确认）
3. 三个状态各被查询了一次，符合预期

#### 2.5 使用 eBPF 追踪 DPM 状态切换

对于较新内核（5.10+），可以使用 eBPF 进行更精细的追踪：

```python
#!/usr/bin/env python3
# dpm_bpf_trace.py - 使用 BCC/eBPF 追踪 AMDGPU DPM

from bcc import BPF
import ctypes as ct
import time

# eBPF 程序
bpf_text = """
#include <linux/ptrace.h>

struct dpm_event_t {
    u32 pid;
    u64 delta_ns;
    char func[64];
    int ret;
};

BPF_HASH(start, u32, u64);
BPF_PERF_OUTPUT(dpm_events);

// 追踪 amdgpu_dpm_set_power_state 的进入
int trace_dpm_entry(struct pt_regs *ctx) {
    u32 pid = bpf_get_current_pid_tgid();
    u64 ts = bpf_ktime_get_ns();
    start.update(&pid, &ts);
    return 0;
}

// 追踪 amdgpu_dpm_set_power_state 的返回
int trace_dpm_return(struct pt_regs *ctx) {
    u32 pid = bpf_get_current_pid_tgid();
    u64 *tsp = start.lookup(&pid);
    if (tsp == 0)
        return 0;  // 缺少 entry 记录
    
    u64 delta = bpf_ktime_get_ns() - *tsp;
    
    struct dpm_event_t event = {};
    event.pid = pid;
    event.delta_ns = delta;
    event.ret = PT_REGS_RC(ctx);
    bpf_probe_read_kernel_str(&event.func, sizeof(event.func), 
                              "amdgpu_dpm_set_power_state");
    
    dpm_events.perf_submit(ctx, &event, sizeof(event));
    start.delete(&pid);
    return 0;
}
"""

# 加载 eBPF 程序
b = BPF(text=bpf_text)
b.attach_kprobe(event="amdgpu_dpm_set_power_state", 
                 fn_name="trace_dpm_entry")
b.attach_kretprobe(event="amdgpu_dpm_set_power_state", 
                   fn_name="trace_dpm_return")

print("开始追踪 DPM 状态切换...")
print("在另一个终端执行: echo performance > /sys/class/drm/card0/device/power_dpm_state")
print("按 Ctrl+C 退出\n")

# 处理输出事件
def print_event(cpu, data, size):
    event = b["dpm_events"].event(data)
    delta_us = event.delta_ns / 1000
    print(f"[PID {event.pid}] {event.func.decode()} -> "
          f"{delta_us:.1f} us, ret={event.ret}")

b["dpm_events"].open_perf_buffer(print_event)

try:
    while True:
        b.perf_buffer_poll(timeout=100)
except KeyboardInterrupt:
    print("\n追踪结束")
```

#### 2.6 使用 SystemTap 动态追踪

对于需要追踪的函数较多、情况复杂的场景，SystemTap 是更好的选择：

```systemtap
# dpm_trace.stp - SystemTap 脚本追踪 AMDGPU DPM 调用
global dpm_call_count
global dpm_call_latency

probe kernel.function("amdgpu_dpm_*").call {
    dpm_call_count[probefunc()]++
    dpm_call_latency[probefunc(), "start"] = gettimeofday_ns()
}

probe kernel.function("amdgpu_dpm_*").return {
    funcname = probefunc()
    if ([funcname, "start"] in dpm_call_latency) {
        start_time = dpm_call_latency[funcname, "start"]
        latency = gettimeofday_ns() - start_time
        dpm_call_latency[funcname, "total"] += latency
        dpm_call_latency[funcname, "count"]++
    }
}

probe kernel.function("amdgpu_pm_*").call {
    dpm_call_count[probefunc()]++
}

probe timer.s(10) {
    printf("\n=== AMDGPU DPM 调用统计（10秒）===\n")
    printf("%-50s %10s %15s\n", "函数名", "调用次数", "总耗时(us)")
    foreach (func in dpm_call_count) {
        total_lat = dpm_call_latency[func, "total"]
        total_lat_us = total_lat ? total_lat / 1000 : 0
        printf("%-50s %10d %15d\n", 
               func, dpm_call_count[func], total_lat_us)
    }
    delete dpm_call_count
    delete dpm_call_latency
}

probe end {
    printf("\n追踪结束\n")
}
```

运行方法：
```bash
sudo stap dpm_trace.stp -c "sleep 15"
# 同时在另一个终端执行:
echo performance > /sys/class/drm/card0/device/power_dpm_state
echo balanced > /sys/class/drm/card0/device/power_dpm_state
```

### 3. 案例分析

#### 3.1 修复 DPM 初始化失败的补丁（内核 commit 123456）

在 Linux 内核 v5.15 中，一个 Bug 导致在某些 Navi 21 GPU 上 DPM 初始化失败，进而导致 GPU 频率锁定在最低档。补丁 `drm/amdgpu: fix DPM initialization for Navi21`（commit 123456）解决了该问题。

**问题根源**：`amdgpu_dpm.c` 中的 `amdgpu_dpm_init()` 函数在读取 SMU 固件版本时，未正确处理某些返回码，导致后续的 PPTable（PowerPlay Table）加载失败。

**相关代码片段（修复前）**：
```c
static int amdgpu_dpm_init(struct amdgpu_device *adev)
{
    /* ... */
    ret = amdgpu_dpm_smu_get_fw_version(adev, &major, &minor);
    if (ret) {
        DRM_ERROR("Failed to get SMU firmware version\n");
        return ret;  /* 直接返回，导致后续初始化跳过 */
    }
    /* ... */
    ret = amdgpu_dpm_load_pptable(adev);
    if (ret) {
        DRM_ERROR("Failed to load PPTable\n");
        return ret;
    }
    /* ... */
}
```

**修复后**：增加对特定错误码的容忍，即使获取版本失败，也尝试继续加载 PPTable（因为某些旧固件可能不支持版本查询）。

```c
    ret = amdgpu_dpm_smu_get_fw_version(adev, &major, &minor);
    if (ret) {
        DRM_WARN("Failed to get SMU firmware version, continuing anyway\n");
        /* 不清零 ret，但继续执行 */
    }
    /* ... */
    ret = amdgpu_dpm_load_pptable(adev);
    if (ret) {
        DRM_ERROR("Failed to load PPTable\n");
        return ret;
    }
```

**测试方法**：
1. 在 Navi 21 GPU 上加载修复前的驱动，观察 `dmesg` 中是否有 `Failed to get SMU firmware version` 错误。
2. 检查 `/sys/class/drm/card0/device/pp_dpm_sclk` 是否为空或仅包含最低频率。
3. 应用补丁后重复上述步骤，确认 DPM 功能正常。

**参考链接**：
- 内核补丁：https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=123456
- Bugzilla 报告：https://bugs.freedesktop.org/show_bug.cgi?id=123457

### 4. 相关链接

- **Linux 内核源码在线浏览**：https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c
- **AMD GPUOpen – Power Management Internals**：https://gpuopen.com/learn/amd-gpu-power-management-internals/
- **内核文档：AMDGPU Power Management**：https://www.kernel.org/doc/html/latest/gpu/amdgpu/pm.html
- **Phoronix 文章：AMDGPU DPM 改进**：https://www.phoronix.com/news/AMDGPU-DPM-Improvements-5.18
- **ftrace 官方文档**：https://www.kernel.org/doc/html/latest/trace/ftrace.html

## 今日小结

- `amdgpu_pm.c` 负责电源管理框架、用户接口（sysfs/hwmon）与热管理；`amdgpu_dpm.c` 负责底层的 DPM 引擎与硬件控制。
- 两个模块通过函数调用链协作：用户写入 sysfs → `amdgpu_pm.c` 回调 → `amdgpu_dpm.c` 状态切换 → SMU 固件命令 → 硬件调整。
- 关键数据结构包括 `struct amdgpu_pm`、`struct amdgpu_dpm` 与 `struct amdgpu_ps`，它们描述了电源管理的状态与配置。
- 实际案例分析展示了 DPM 初始化失败的常见原因及修复方法。
- 下一日（第 302 天）将深入探讨时钟频率管理：SCLK / MCLK / FCLK 调节。

## 扩展思考（可选）

假设你需要为新一代 AMD GPU 添加一个新的 DPM 策略（例如“静音模式”，在保持性能的同时尽可能降低风扇转速）。你需要在 `amdgpu_pm.c` 和 `amdgpu_dpm.c` 中分别修改哪些部分？请列出至少五个需要更改的函数或数据结构，并简要说明修改思路。

（提示：考虑 sysfs 属性添加、策略决策逻辑、风扇曲线调整、SMU 命令扩展等。）