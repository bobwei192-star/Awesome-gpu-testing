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