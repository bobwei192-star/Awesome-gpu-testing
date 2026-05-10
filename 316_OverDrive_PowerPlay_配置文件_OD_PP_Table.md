# 第316天：OverDrive / PowerPlay 配置文件（OD / PP Table）

## 学习目标

- 理解 AMDGPU PowerPlay 架构和 OverDrive 技术的设计原理
- 掌握 OD（OverDrive）表和 PP（PowerPlay）表的内部结构和字段含义
- 学会通过 sysfs 接口读取和修改 OD/PP Table 参数
- 掌握 GPU 频率、电压曲线调整的核心方法
- 理解 PowerPlay 动态状态切换（DPM State）机制
- 熟悉 OD 表在 GPU 超频/降压场景中的实际应用
- 学会解析 PP_Table 的二进制数据结构和校验机制

## 知识详解

### 1.1 PowerPlay 架构概述

PowerPlay 是 AMD 的 GPU 电源管理框架，最初在 Radeon HD 2000 系列引入，经过多年演进，在 RDNA 和 CDNA 架构中已发展为成熟的动态电源管理系统。

```
┌─────────────────────────────────────────────────────────────────────┐
│                     AMDGPU PowerPlay 架构总览                        │
│                                                                     │
│  ┌─────────────┐    ┌──────────────────┐    ┌──────────────────┐   │
│  │ 用户空间      │    │  sysfs 接口        │    │  ROCm-SMI /     │   │
│  │ (User Space)  │◄──►│  /sys/class/drm/  │◄──►│  amdgpu_top     │   │
│  │              │    │  card0/device/     │    │  应用层工具       │   │
│  └──────┬───────┘    └────────┬─────────┘    └──────────────────┘   │
│         │                     │                                      │
│  ┌──────▼─────────────────────▼──────────────────────────────────┐  │
│  │                  PowerPlay 核心层                              │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │  DPM (Dynamic Power Management) 状态机                    │  │  │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │  │  │
│  │  │  │ DPM0    │  │ DPM1    │  │ DPM2    │  │ DPM3+   │   │  │  │
│  │  │  │ (最低频) │  │ (低频)   │  │ (中频)   │  │ (高频)   │   │  │  │
│  │  │  │ 低功耗   │  │ 平衡     │  │ 高性能   │  │ 最高性能  │   │  │  │
│  │  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘   │  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  │                                                               │  │
│  │  ┌─────────────────┐  ┌──────────────┐  ┌─────────────────┐  │  │
│  │  │  OD Table        │  │  PP_Table     │  │  PowerPlay      │  │  │
│  │  │  (超频/降压配置)   │  │  (电源策略     │  │  Playlist       │  │  │
│  │  │  ┌─ 频率点       │  │   配置表)      │  │  (状态切换策略)   │  │  │
│  │  │  ├─ 电压点       │  │  ┌─ 功耗上限   │  │  ┌─ 暂态时间    │  │  │
│  │  │  ├─ 功耗上限      │  │  ├─ 温度上限   │  │  ├─ 迟滞参数    │  │  │
│  │  │  └─ 电流上限      │  │  ├─ 电流上限   │  │  ├─ 过滤系数    │  │  │
│  │  │                 │  │  └─ 风扇曲线   │  │  └─ 触发阈值    │  │  │
│  │  └─────────────────┘  └──────────────┘  └─────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│         │                     │                                      │
│  ┌──────▼─────────────────────▼──────────────────────────────────┐  │
│  │                SMU (System Management Unit) 固件层              │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │  SMU v13 (RDNA 3) / SMU v11 (RDNA 2) / SMU v10 (RDNA 1)│  │  │
│  │  │  ┌──────────────┐  ┌───────────┐  ┌──────────────────┐│  │  │
│  │  │  │ 频率控制器    │  │ 电压控制器 │  │ 功耗/温度监控器   ││  │  │
│  │  │  │ (Freq Ctrl)   │  │ (Volt Ctrl)│  │ (Power/Thermal)  ││  │  │
│  │  │  └──────────────┘  └───────────┘  └──────────────────┘│  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│         │                                                          │
│  ┌──────▼──────────────────────────────────────────────────────┐  │
│  │                    GPU 硬件层                                │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │  GFXCLK  │  MEMCLK  │  SOCLK  │  VDDCR_VDD │  VDDCR_MEM│  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

**关键组件说明：**

| 组件 | 说明 | 源码路径 |
|------|------|---------|
| PowerPlay Core | 电源管理核心框架，处理 DPM 状态转换 | `drivers/gpu/drm/amd/pm/swsmu/` |
| SMU Interface | SMU 固件通信接口，通过 Message Passing 控制 | `drivers/gpu/drm/amd/pm/swsmu/smu11/` |
| OD Table | OverDrive 配置表，存储用户可调参数 | `drivers/gpu/drm/amd/pm/amdgpu_pm.c` |
| PP_Table | PowerPlay 配置表，存储电源策略参数 | `drivers/gpu/drm/amd/pm/swsmu/smu11/smu_v11_0_pptable.h` |
| DPM State | 动态电源管理状态机，管理频率/电压组合 | `drivers/gpu/drm/amd/pm/inc/amdgpu_dpm.h` |

### 1.2 OD (OverDrive) Table 详解

OD Table 是用户可配置的超频/降压参数表，通过 sysfs 接口 `pp_od_clk_voltage` 暴露给用户空间。

**OD Table 数据结构：**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       OD Table 数据结构                                   │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  OD Table Header                                                   │  │
│  │  ├─ version:        u32    # 表版本号                               │  │
│  │  ├─ table_size:     u32    # 表大小                                  │  │
│  │  ├─ feature_mask:   u64    # 功能位图                               │  │
│  │  ├─ cap_check:      u32    # 校验和                                  │  │
│  │  └─ reserved:       u32[4] # 保留字段                               │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  频率域 (Frequency Domain)                                          │  │
│  │  ┌──────────────┬──────────┬──────────┬──────────┬──────────┐    │  │
│  │  │ Domain       │ 索引      │ 最小值    │ 最大值    │ 当前值    │    │  │
│  │  ├──────────────┼──────────┼──────────┼──────────┼──────────┤    │  │
│  │  │ GFXCLK (SCLK)│   0      │  500     │  3000    │  2500    │    │  │
│  │  │ MEMCLK (MCLK)│   1      │  100     │  1500    │  1200    │    │  │
│  │  │ SOCLK        │   2      │  100     │  2000    │  1800    │    │  │
│  │  │ FCLK         │   3      │  400     │  2400    │  2000    │    │  │
│  │  │ VCLK         │   4      │  100     │  3000    │  2400    │    │  │
│  │  │ DCLK         │   5      │  100     │  3000    │  2400    │    │  │
│  │  └──────────────┴──────────┴──────────┴──────────┴──────────┘    │  │
│  │  * 单位为 MHz                                                     │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  电压/频率曲线 (Voltage-Frequency Curve)                            │  │
│  │                                                                     │  │
│  │  每个频率域包含 N 个 V/F 点 (通常为 2-3 个)                        │  │
│  │  ┌──────────────┬──────────┬──────────┬──────────┐                │  │
│  │  │ VF Point     │ 点 0      │ 点 1      │ 点 2      │                │  │
│  │  ├──────────────┼──────────┼──────────┼──────────┤                │  │
│  │  │ GFX_FREQ(MHz)│   500     │   1500    │   2500    │                │  │
│  │  │ GFX_VOLT(mV) │   700     │   950     │   1200    │                │  │
│  │  │ MEM_FREQ(MHz)│   100     │   600     │   1200    │                │  │
│  │  │ MEM_VOLT(mV) │   600     │   750     │   900     │                │  │
│  │  └──────────────┴──────────┴──────────┴──────────┘                │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  功耗/电流限制 (Power/Current Limits)                               │  │
│  │                                                                     │  │
│  │  ┌──────────────────────┬────────────┬────────────┬────────────┐  │  │
│  │  │ Limit                │ 默认值       │ 最小值       │ 最大值       │  │  │
│  │  ├──────────────────────┼────────────┼────────────┼────────────┤  │  │
│  │  │ PPT (Package Power)  │  235W      │   150W     │   300W     │  │  │
│  │  │ TDC (Thermal Current)│  150A      │   100A     │   200A     │  │  │
│  │  │ EDC (Electrical      │  200A      │   120A     │   260A     │  │  │
│  │  │   Current)           │            │            │            │  │  │
│  │  │ SPPT (Slow PPT)      │  200W      │   100W     │   280W     │  │  │
│  │  │ APPPT (Avg Power     │  180W      │   100W     │   250W     │  │  │
│  │  │   PPT)              │            │            │            │  │  │
│  │  └──────────────────────┴────────────┴────────────┴────────────┘  │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

**OD Table 内核数据结构定义：**

```c
// drivers/gpu/drm/amd/pm/swsmu/smu11/smu_v11_0_pptable.h
// SMU v11 OD Table 定义 (RDNA 2)

#define SMU_11_OD_TABLE_MAX_VF_POINTS 3

struct smu_11_0_overdrive_table {
    uint32_t version;           // 表版本
    uint32_t feature_mask;      // 功能启用位图
    uint32_t cap_check;         // 能力校验值
    
    // V/F 曲线点 (电压/频率点)
    struct {
        uint32_t freq;          // 频率 (MHz)
        uint32_t volt;          // 电压 (mV)
    } vf_points[SMU_11_OD_TABLE_MAX_VF_POINTS];
    
    // 时钟频率限制
    uint32_t gfxclk_fmax;       // GFXCLK 最大频率 (MHz)
    uint32_t gfxclk_fmin;       // GFXCLK 最小频率 (MHz)
    uint32_t memclk_fmax;       // MEMCLK 最大频率 (MHz)
    uint32_t memclk_fmin;       // MEMCLK 最小频率 (MHz)
    
    // 电源限制
    uint32_t power_limit;       // PPT 功耗限制 (W)
    uint32_t tdc_limit;         // TDC 电流限制 (A)
    uint32_t edc_limit;         // EDC 电流限制 (A)
    uint32_t sppt_limit;        // SPPT 功耗限制 (W)
    uint32_t sppt_apd_limit;    // SPPT APD 限制 (W)
    
    // 温度限制
    uint32_t tedge_limit;       // 边缘温度限制 (°C)
    uint32_t thotspot_limit;    // 热点温度限制 (°C)
    uint32_t tmem_limit;        // 显存温度限制 (°C)
    uint32_t tvr_limit;         // VR 温度限制 (°C)
    
    // 风扇控制
    uint32_t fan_target_temp;   // 目标温度 (°C)
    uint32_t fan_min_speed;     // 最小转速 (%)
    uint32_t fan_max_speed;     // 最大转速 (%)
    uint32_t fan_acoustic_limit;// 声学限制 (RPM)
};
```

**OD Table 功能位图说明：**

| Bit | 功能 | 描述 |
|-----|------|------|
| 0 | OD_SCLK | GFXCLK 超频支持 |
| 1 | OD_MCLK | MEMCLK 超频支持 |
| 2 | OD_VDDC | 核心电压调整支持 |
| 3 | OD_POWER_LIMIT | 功耗限制调整支持 |
| 4 | OD_FAN_CURVE | 风扇曲线调整支持 |
| 5 | OD_TEMPERATURE | 温度目标调整支持 |
| 6 | OD_VOLTAGE_CURVE | 电压曲线调整支持 |
| 7 | OD_AUTO | 自动超频支持 |

### 1.3 PP_Table (PowerPlay Table) 详解

PP_Table 是更底层的电源策略配置表，由 VBIOS 提供默认值，驱动在初始化时加载。

**PP_Table 结构层次：**

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        PP_Table 结构层次                                  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  PP_Table Header                                                   │  │
│  │  ├─ table_id:        u16     # 表标识                               │  │
│  │  ├─ table_version:   u16     # 表版本                                │  │
│  │  ├─ size:            u16     # 表大小                                │  │
│  │  ├─ soc_limit:       u32     # SoC 功耗限制                          │  │
│  │  ├─ gfx_limit:       u32     # GFX 功耗限制                          │  │
│  │  ├─ mem_limit:       u32     # 显存功耗限制                          │  │
│  │  ├─ total_power:     u32     # 总功耗上限                            │  │
│  │  └─ thermal_limit:   u32     # 温度上限                              │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  DPM 状态表 (每个域有多个 DPM State)                                │  │
│  │                                                                     │  │
│  │  GFX DPM States:   MEM DPM States:   SOC DPM States:               │  │
│  │  ┌─────────┬──────┐ ┌─────────┬──────┐ ┌─────────┬──────┐        │  │
│  │  │ State 0 │ 500M │ │ State 0 │ 100M │ │ State 0 │ 100M │        │  │
│  │  │ State 1 │ 800M │ │ State 1 │ 300M │ │ State 1 │ 500M │        │  │
│  │  │ State 2 │ 1200M│ │ State 2 │ 600M │ │ State 2 │ 900M │        │  │
│  │  │ State 3 │ 1600M│ │ State 3 │ 900M │ │ State 3 │ 1200M│        │  │
│  │  │ State 4 │ 2000M│ │ State 4 │ 1200M│ │ State 4 │ 1800M│        │  │
│  │  │ State 5 │ 2500M│ │         │      │ │         │      │        │  │
│  │  └─────────┴──────┘ └─────────┴──────┘ └─────────┴──────┘        │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  电源域功耗配置 (Power Domain Configuration)                        │  │
│  │                                                                     │  │
│  │  ┌───────────┬──────────┬──────────┬──────────┬──────────┐       │  │
│  │  │ Domain    │ TDP (W)  │ TDC (A)  │ EDC (A)  │ Telemetry│       │  │
│  │  ├───────────┼──────────┼──────────┼──────────┼──────────┤       │  │
│  │  │ SOC       │   35     │   30     │   45     │   Yes    │       │  │
│  │  │ GFX       │   150    │   120    │   160    │   Yes    │       │  │
│  │  │ MEM       │   30     │   25     │   35     │   Yes    │       │  │
│  │  │ PCIe      │   75     │   5      │   8      │   No     │       │  │
│  │  └───────────┴──────────┴──────────┴──────────┴──────────┘       │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  温度/风扇配置 (Thermal/Fan Configuration)                         │  │
│  │                                                                     │  │
│  │  ┌──────────────────┬──────────┬──────────┬──────────┬──────────┐ │  │
│  │  │ Sensor           │ Critical │ Emergency│ Soft     │ Hysteresis│ │  │
│  │  ├──────────────────┼──────────┼──────────┼──────────┼──────────┤ │  │
│  │  │ Edge (边缘)      │   95°C   │  105°C   │   85°C   │    5°C   │ │  │
│  │  │ Hotspot (热点)   │  105°C   │  115°C   │   95°C   │    5°C   │ │  │
│  │  │ Memory (显存)    │   95°C   │  105°C   │   85°C   │    5°C   │ │  │
│  │  │ VR (供电)        │  100°C   │  110°C   │   90°C   │    5°C   │ │  │
│  │  └──────────────────┴──────────┴──────────┴──────────┴──────────┘ │  │
│  │                                                                     │  │
│  │  风扇曲线 (Fan Curve):                                              │  │
│  │  ┌──────────────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐│  │
│  │  │ Temperature°C│  40 │  50 │  60 │  70 │  80 │  85 │  90 │  95 ││  │
│  │  ├──────────────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤│  │
│  │  │ Fan Speed %  │  20 │  25 │  35 │  45 │  55 │  70 │  85 │ 100 ││  │
│  │  └──────────────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘│  │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

**PP_Table 内核数据结构：**

```c
// drivers/gpu/drm/amd/pm/swsmu/smu13/smu_v13_0_pptable.h
// SMU v13 PP_Table 定义 (RDNA 3)

#define SMU_13_MAX_DPM_STATES 8
#define SMU_13_MAX_FAN_POINTS 8

struct smu_13_0_powerplay_table {
    struct atom_common_table_header header;
    
    // 功耗配置
    uint32_t TDP;           // 热设计功耗 (W)
    uint32_t TDC;           // 热设计电流 (A)
    uint32_t EDC;           // 电气设计电流 (A)
    uint32_t power_reserve; // 功率储备 (W)
    
    // DPM 状态配置
    struct {
        uint32_t freq;      // 频率 (MHz)
        uint32_t voltage;   // 电压 (mV)
        uint32_t power;     // 功耗 (W)
        uint32_t current;   // 电流 (A)
    } dpm_states[SMU_13_MAX_DPM_STATES][3]; // [domain][state]
    
    // 温度阈值
    struct {
        uint32_t critical;  // 临界温度 (°C)
        uint32_t emergency; // 紧急温度 (°C)
        uint32_t soft;      // 软阈值 (°C)
        uint32_t hysteresis;// 迟滞 (°C)
    } thermal_limits[4];    // [sensor_type]
    
    // 风扇配置
    struct {
        uint32_t temp;      // 温度点 (°C)
        uint32_t speed;     // 风扇转速 (%)
    } fan_curve[SMU_13_MAX_FAN_POINTS];
    
    // 电压配置
    uint32_t vdd_min;       // 最小核心电压 (mV)
    uint32_t vdd_max;       // 最大核心电压 (mV)
    uint32_t vdd_mem_min;   // 最小显存电压 (mV)
    uint32_t vdd_mem_max;   // 最大显存电压 (mV)
    
    // 电源域映射
    uint32_t power_domain_map;  // 电源域位图
    
    // 平台能力
    uint32_t platform_caps;     // 平台能力位图
};
```

### 1.4 DPM (Dynamic Power Management) 状态机

DPM 状态机是 PowerPlay 的核心调度引擎，负责根据 GPU 负载实时切换工作状态。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   DPM 状态转换流程                                       │
│                                                                         │
│                       ┌──────────────┐                                  │
│                       │   PowerPlay   │                                  │
│                       │   Init        │                                  │
│                       └──────┬───────┘                                  │
│                              │                                           │
│                              ▼                                           │
│                       ┌──────────────┐                                  │
│                       │   DPM0       │◄───── 最低功耗状态               │
│                       │   (最低频)    │      (空闲/轻负载)                │
│                       └──────┬───────┘                                  │
│                              │                                           │
│                      负载↑    │    负载↓                                  │
│                              ▼                                           │
│                       ┌──────────────┐                                  │
│               ┌──────►│   DPM1       │                                  │
│               │       │   (低频)      │                                  │
│               │       └──────┬───────┘                                  │
│               │              │                                           │
│               │        负载↑  │    负载↓                                  │
│               │              ▼                                           │
│               │       ┌──────────────┐                                  │
│               │       │   DPM2       │                                  │
│               │       │   (中频)      │                                  │
│               │       └──────┬───────┘                                  │
│               │              │                                           │
│               │        负载↑  │    负载↓                                  │
│               │              ▼                                           │
│               │       ┌──────────────┐                                  │
│               │       │   DPM3       │                                  │
│               │       │   (高频)      │                                  │
│               │       └──────┬───────┘                                  │
│               │              │                                           │
│               │        负载↑  │    负载↓                                  │
│               │              ▼                                           │
│               │       ┌──────────────┐                                  │
│               └───────┤   DPM4+      │  ◄──── 最高性能状态              │
│                       │   (最高频)     │       (高负载/游戏)               │
│                       └──────────────┘                                  │
│                                                                         │
│   状态转换条件:                                                          │
│   ┌────────────────────────────────────────────────────────────────┐    │
│   │  上升阈值: GPU 利用率 > 70% 持续 100ms                          │    │
│   │  下降阈值: GPU 利用率 < 30% 持续 500ms                          │    │
│   │  紧急转换: 温度 > 85°C 立即降频                                  │    │
│   └────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

**DPM 状态切换的延迟参数：**

| 参数 | 默认值 | 说明 |
|------|--------|------|
| up_threshold | 70 | 升频利用率阈值 (%) |
| down_threshold | 30 | 降频利用率阈值 (%) |
| up_hysteresis | 100 | 升频等待时间 (ms) |
| down_hysteresis | 500 | 降频等待时间 (ms) |
| switching_delay | 20 | 状态切换延迟 (us) |
| voltage_settling | 50 | 电压稳定时间 (us) |

### 1.5 OD Table 与 PP_Table 的交互机制

```
┌──────────────────────────────────────────────────────────────────────────┐
│            OD Table 与 PP_Table 交互流程                                  │
│                                                                          │
│  用户空间                       内核空间                   SMU 固件        │
│  ┌────────┐                 ┌──────────┐            ┌──────────────┐    │
│  │用户写入  │                 │ amdgpu_pm │            │  SMU Firmware │    │
│  │OD参数   │─────sysfs─────►│  .c       │────MSG────►│              │    │
│  │         │  pp_od_clk_    │ 解析参数   │            │  更新OD Table │    │
│  │         │  voltage      │          │            │  到新值       │    │
│  └────────┘                 └──────────┘            └──────────────┘    │
│       │                         │                         │              │
│       │                         ▼                         ▼              │
│       │                  ┌──────────────┐          ┌──────────────┐      │
│       │                  │ 验证参数合法性  │          │  更新PP_Table │      │
│       │                  │ - 频率范围     │          │  - 调整DPM    │      │
│       │                  │ - 电压范围     │          │  - 更新VF曲线 │      │
│       │                  │ - 功耗限制     │          │  - 调整调度器 │      │
│       │                  └──────┬───────┘          └──────┬───────┘      │
│       │                         │                         │              │
│       │                         ▼                         ▼              │
│       │                  ┌──────────────┐          ┌──────────────┐      │
│       │                  │  写入 SMU     │          │  激活新配置    │      │
│       │◄────status───────┤  Message     │◄─────────┤  Apply        │      │
│       │                  │  返回结果      │          │  New Setting  │      │
│       └                  └──────────────┘          └──────────────┘      │
│                                                                          │
│  VBIOS:                                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  PP_Table 默认值存储在 VBIOS 中                                      │  │
│  │  ┌──────────────┬────────────┬──────────────┬──────────────┐      │  │
│  │  │ PowerTune    │ DPM States  │ Thermal      │ Fan Curve    │      │  │
│  │  │ Limits       │ Table       │ Limits       │ Table        │      │  │
│  │  └──────────────┴────────────┴──────────────┴──────────────┘      │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

**关键交互消息（SMU Message）：**

| Message ID | 名称 | 说明 |
|-----------|------|------|
| 0x01 | SMU_MSG_SetSoftMinGfxClk | 设置 GFXCLK 软最小值 |
| 0x02 | SMU_MSG_SetSoftMaxGfxClk | 设置 GFXCLK 软最大值 |
| 0x03 | SMU_MSG_SetSoftMinMemClk | 设置 MEMCLK 软最小值 |
| 0x04 | SMU_MSG_SetSoftMaxMemClk | 设置 MEMCLK 软最大值 |
| 0x05 | SMU_MSG_SetHardMinGfxClk | 设置 GFXCLK 硬最小值 |
| 0x06 | SMU_MSG_SetHardMaxGfxClk | 设置 GFXCLK 硬最大值 |
| 0x10 | SMU_MSG_TransferTableDram2Smu | 将 OD Table 从 DDR 传输到 SMU SRAM |
| 0x11 | SMU_MSG_TransferTableSmu2Dram | 将 OD Table 从 SMU SRAM 读取到 DDR |
| 0x20 | SMU_MSG_SetPowerLimit | 设置功耗限制 |
| 0x21 | SMU_MSG_SetTemperatureLimit | 设置温度限制 |
| 0x30 | SMU_MSG_GetCurrentPower | 获取当前功耗 |
| 0x31 | SMU_MSG_GetCurrentTemperature | 获取当前温度 |
| 0x40 | SMU_MSG_SetFanControl | 设置风扇控制模式 |
| 0x41 | SMU_MSG_SetFanSpeedPercent | 设置风扇转速百分比 |

### 1.6 OverDrive 技术与 GPU 超频

OverDrive 技术允许用户在安全范围内对 GPU 进行超频或降压操作。

**超频核心参数：**

| 参数 | 说明 | 推荐调整范围 |
|------|------|-------------|
| GFXCLK Offset | GFX 频率偏移 | +0 ~ +200 MHz |
| MEMCLK Offset | 显存频率偏移 | +0 ~ +500 MHz |
| Voltage Offset | 核心电压偏移 | -100 ~ +50 mV |
| Power Limit | PPT 功耗限制 | +0% ~ +15% |
| Fan Speed | 风扇转速目标 | 自动或自定义曲线 |

**安全调整原则：**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       GPU 超频安全区域                                    │
│                                                                         │
│  电压 (mV)                                                              │
│      ▲                                                                  │
│  1300│  ✗ 危险区域 (可能导致硬件损坏)                                    │
│      │   - 电压过高引起电迁移失效                                        │
│      │   - 温度超过结温极限                                              │
│  1200│  ┌──────────────────────────────────────────────────┐           │
│      │  │  ▲ 稳定超频区域                                    │           │
│  1100│  │  │  - 性能提升 5-15%                              │           │
│      │  │  │  - 功耗增加 15-30%                             │           │
│  1000│  │  │  - 需要充分散热                                │           │
│      │  │  ├───────────────────────────────────────────────┤           │
│   900│  │  │  出厂默认区域                                   │           │
│      │  │  │  - 官方标准频率/电压                            │           │
│   800│  │  │  - 保证稳定性和寿命                             │           │
│      │  │  ├───────────────────────────────────────────────┤           │
│   700│  │  │  ▼ 降压区域 (Undervolt)                        │           │
│      │  │  │  - 功耗降低 15-25%                             │           │
│   600│  │  │  - 性能影响 < 3%                               │           │
│      │  │  │  - 温度降低 5-10°C                             │           │
│      │  └──────────────────────────────────────────────────┘           │
│      │                                                                  │
│   500│  ✗ 不稳定区域 (可能导致崩溃)                                      │
│      │     - 电压不足引起计算错误                                        │
│      │     - 驱动超时重置                                               │
│      └───────────────────────────────────────────► 频率 (MHz)            │
│        500    1000    1500    2000    2500    3000                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.7 PowerPlay 与 RPM (Runtime Power Management) 的配合

```
┌─────────────────────────────────────────────────────────────────────────┐
│              PowerPlay + Runtime PM 协同工作                              │
│                                                                         │
│  时间线:                                                                │
│  ──────────────────────────────────────────────────────────────────►    │
│                                                                         │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐           │
│  │ D0:    │  │ DPM0   │  │ DPM2   │  │ DPM4   │  │ D3hot │           │
│  │ 活动    │  │ 空闲    │  │ 中等负载 │  │ 高负载   │  │ 睡眠   │           │
│  │ 350W   │  │ 50W    │  │ 150W   │  │ 300W   │  │ 5W    │           │
│  └────────┘  └────────┘  └────────┘  └────────┘  └────────┘           │
│       │           │           │           │           │                  │
│       │  空闲     │  负载增   │  负载继   │  空闲超   │                  │
│       │  超时     │  加       │  续增加   │  过阈值   │                  │
│       ▼           ▼           ▼           ▼           ▼                  │
│                                ┌────────────────────┐                    │
│                                │  PowerPlay 决策     │                    │
│                                │  ┌──────────────┐  │                    │
│                                │  │ 负载评估器     │  │                    │
│                                │  │ - GPU 利用率   │  │                    │
│                                │  │ - 帧率         │  │                    │
│                                │  │ - 功耗         │  │                    │
│                                │  │ - 温度         │  │                    │
│                                │  └──────┬───────┘  │                    │
│                                │         │          │                    │
│                                │         ▼          │                    │
│                                │  ┌──────────────┐  │                    │
│                                │  │ 状态控制器     │  │                    │
│                                │  │ - 选择 DPM    │  │                    │
│                                │  │    level      │  │                    │
│                                │  │ - 设置频率    │  │                    │
│                                │  │ - 设置电压    │  │                    │
│                                │  └──────────────┘  │                    │
│                                └────────────────────┘                    │
└─────────────────────────────────────────────────────────────────────────┘
```

## 实践操作

### 2.1 读取当前 OD Table

```bash
#!/bin/bash
# 读取并解析 GPU OD Table

# 检查 GPU 是否存在
if [ ! -d "/sys/class/drm/card0/device" ]; then
    echo "ERROR: GPU device not found"
    exit 1
fi

# 读取 OD 表
echo "=== GPU OD Table ==="
cat /sys/class/drm/card0/device/pp_od_clk_voltage

echo ""
echo "=== 当前功耗限制 ==="
cat /sys/class/drm/card0/device/power1_cap

echo ""
echo "=== 当前 DPM 状态 ==="
cat /sys/class/drm/card0/device/power_dpm_state

echo ""
echo "=== 当前电源策略 ==="
cat /sys/class/drm/card0/device/power_dpm_force_performance_level
```

**输出示例与分析：**

```bash
=== GPU OD Table ===
OD_SCLK:
0: 500Mhz    700mV
1: 1500Mhz   950mV
2: 2500Mhz   1200mV
OD_MCLK:
0: 100Mhz    600mV
1: 600Mhz    750mV
2: 1200Mhz   900mV
OD_RANGE:
SCLK:     500Mhz       3000Mhz
MCLK:     100Mhz       1500Mhz
VDDC:     700mV        1200mV
POWER:    150W         300W
```

### 2.2 修改 OD Table 参数

```bash
#!/bin/bash
# 修改 GPU OD Table 参数 - 安全超频示例

# 警告信息
echo "WARNING: Modifying OD parameters may cause system instability!"
echo "Make sure adequate cooling is available."
echo ""

# 读取当前值
echo "=== 当前配置 ==="
cat /sys/class/drm/card0/device/pp_od_clk_voltage

# 设置 GFXCLK 最大偏移 (+100MHz)
echo ""
echo "=== 设置 GFXCLK 偏移 ==="
echo "s 0 2600" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage

# 设置 MEMCLK 最大偏移 (+200MHz)
echo "m 0 1400" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage

# 增加功耗限制 (+10%)
echo ""
echo "=== 设置功耗限制 ==="
echo "c" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage

# 提交更改
echo ""
echo "=== 提交 OD 参数 ==="
echo "c" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage

# 验证新配置
sleep 1
echo ""
echo "=== 新配置验证 ==="
cat /sys/class/drm/card0/device/pp_od_clk_voltage

echo ""
echo "=== 超频完成 ==="
echo "请运行压力测试验证稳定性"
```

### 2.3 GPU 降压 (Undervolt) 操作

```bash
#!/bin/bash
# undervolt_gpu.sh - GPU 降压脚本

echo "=== AMD GPU Undervolt ==="

# 确保 GPU 处于可调整状态
echo "manual" | sudo tee /sys/class/drm/card0/device/power_dpm_force_performance_level

# 读取当前 V/F 曲线
echo "=== 当前 V/F 曲线 ==="
cat /sys/class/drm/card0/device/pp_od_clk_voltage | grep -E "SCLK|MCLK|VDDC"

# 创建降压配置
# 格式: [频率点] [新电压]
# 降低所有 V/F 点的电压 50mV
echo "=== 应用降压配置 ==="
echo "vc 2 1150" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
echo "vc 1 900" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage

# 提交
echo "c" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage

# 验证
echo ""
echo "=== 验证降压配置 ==="
cat /sys/class/drm/card0/device/pp_od_clk_voltage

echo ""
echo "=== 恢复自动管理 ==="
echo "auto" | sudo tee /sys/class/drm/card0/device/power_dpm_force_performance_level
```

### 2.4 设置电源策略级别

```bash
#!/bin/bash
# power_profile_setup.sh - 电源策略配置

echo "=== AMD GPU Power Profile Configuration ==="

# 可用策略
echo "Available power profiles:"
echo "  auto          - 自动管理 (默认)"
echo "  low           - 低功耗"
echo "  high          - 高性能"
echo "  manual        - 手动控制"

echo ""
read -p "Select profile [auto/low/high/manual]: " PROFILE

case $PROFILE in
    auto|low|high|manual)
        echo "Setting profile to: $PROFILE"
        echo "$PROFILE" | sudo tee /sys/class/drm/card0/device/power_dpm_force_performance_level
        ;;
    *)
        echo "Invalid selection"
        exit 1
        ;;
esac

echo ""
echo "=== 验证 ==="
cat /sys/class/drm/card0/device/power_dpm_force_performance_level
```

### 2.5 DPM 状态监控脚本

```bash
#!/bin/bash
# dpm_monitor.sh - DPM 状态实时监控

# 配置
INTERVAL=${1:-1}  # 采样间隔 (秒)
COUNT=${2:-60}    # 采样次数

echo "=== DPM State Monitor ==="
echo "Interval: ${INTERVAL}s"
echo "Samples: ${COUNT}"
echo ""

# 表头
printf "%-20s %-15s %-15s %-15s %-15s %-15s\n" \
    "Time" "GFXCLK(MHz)" "MEMCLK(MHz)" "Power(W)" "Temp(C)" "DPM Level"
printf "%0.s-" {1..95}
echo ""

for i in $(seq 1 $COUNT); do
    TIMESTAMP=$(date +%H:%M:%S)
    
    # 从 sysfs 读取数据
    GFXCLK=$(cat /sys/class/drm/card0/device/pp_dpm_sclk 2>/dev/null | grep -oP '\*\s+\K\d+' | head -1)
    MEMCLK=$(cat /sys/class/drm/card0/device/pp_dpm_mclk 2>/dev/null | grep -oP '\*\s+\K\d+' | head -1)
    POWER=$(cat /sys/class/drm/card0/device/power1_average 2>/dev/null)
    if [ -n "$POWER" ]; then
        POWER=$((POWER / 1000000))
    fi
    TEMP=$(cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input 2>/dev/null)
    if [ -n "$TEMP" ]; then
        TEMP=$((TEMP / 1000))
    fi
    LEVEL=$(cat /sys/class/drm/card0/device/power_dpm_force_performance_level 2>/dev/null)
    
    printf "%-20s %-15s %-15s %-15s %-15s %-15s\n" \
        "$TIMESTAMP" "${GFXCLK:-N/A}" "${MEMCLK:-N/A}" "${POWER:-N/A}" "${TEMP:-N/A}" "${LEVEL:-N/A}"
    
    sleep $INTERVAL
done

echo ""
echo "=== Monitor Complete ==="
```

### 2.6 PP_Table 导出与分析

```bash
#!/bin/bash
# pptable_analyze.sh - PP_Table 导出和分析

echo "=== PP_Table Analysis ==="

# 检查调试文件系统
DEBUGFS_PATH="/sys/kernel/debug/dri/0/amdgpu_pm_info"

if [ -f "$DEBUGFS_PATH" ]; then
    echo "=== AMDGPU PM Info ==="
    cat "$DEBUGFS_PATH"
else
    echo "Debugfs not available. Mounting debugfs..."
    sudo mount -t debugfs none /sys/kernel/debug/
    if [ -f "$DEBUGFS_PATH" ]; then
        cat "$DEBUGFS_PATH"
    else
        echo "PP_Table debugfs not available on this kernel version"
        echo "Available PM information:"
        ls /sys/class/drm/card0/device/ | grep -E "power|dpm|pp_"
    fi
fi

echo ""
echo "=== Available DPM States ==="
echo "--- GFXCLK DPM States ---"
cat /sys/class/drm/card0/device/pp_dpm_sclk 2>/dev/null

echo ""
echo "--- MEMCLK DPM States ---"
cat /sys/class/drm/card0/device/pp_dpm_mclk 2>/dev/null

echo ""
echo "--- SOCLK DPM States ---"
cat /sys/class/drm/card0/device/pp_dpm_socclk 2>/dev/null

echo ""
echo "=== 当前频率 ==="
echo "GFXCLK: $(cat /sys/class/drm/card0/device/pp_dpm_sclk 2>/dev/null | grep '*' | awk '{print $2}')"
echo "MEMCLK: $(cat /sys/class/drm/card0/device/pp_dpm_mclk 2>/dev/null | grep '*' | awk '{print $2}')"
```

**PP_Table 二进制导出方法：**

```bash
# 直接读取 VBIOS 中的 PP_Table
# RDNA 2/3 的 PP_Table 存储在 VBIOS 的 PowerPlay 表中

# 方法 1: 通过 VBIOS ROM 读取
sudo cat /sys/class/drm/card0/device/rom > vbios.rom
# 使用 VBIOS 解析工具 (如 UEFITool / AMD VBIOS Editor)

# 方法 2: 通过 debugfs 读取
sudo cat /sys/kernel/debug/dri/0/amdgpu_vbios > vbios.bin

# 方法 3: 使用 Tools 工具
sudo dmesg | grep -i powerplay  # 查看 PowerPlay 初始化信息
sudo dmesg | grep -i "pp_table" # 查看 PP_Table 加载信息
```

### 2.7 自定义风扇曲线

```bash
#!/bin/bash
# fan_curve_custom.sh - 自定义风扇曲线

echo "=== AMD GPU Fan Curve Configuration ==="

# 检查是否支持风扇控制
if [ ! -f "/sys/class/drm/card0/device/hwmon/hwmon*/pwm1" ]; then
    echo "ERROR: Fan control not available"
    exit 1
fi

# 读取当前风扇曲线
echo "=== Current Fan Curve ==="
cat /sys/class/drm/card0/device/hwmon/hwmon*/pwm1 2>/dev/null
cat /sys/class/drm/card0/device/hwmon/hwmon*/fan1_input 2>/dev/null

echo ""
echo "=== 设置自定义风扇曲线 ==="
echo "温度点: 30 40 50 60 70 80 85 90"
echo "转速%:  20 25 35 45 60 75 85 100"

# 应用风扇曲线 (需要通过 OD Table 接口)
echo "f 30 20" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
echo "f 40 25" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
echo "f 50 35" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
echo "f 60 45" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
echo "f 70 60" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
echo "f 80 75" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
echo "f 85 85" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
echo "f 90 100" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage

# 提交
echo "c" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage

echo ""
echo "风扇曲线已更新"
echo "验证:"
sleep 1
cat /sys/class/drm/card0/device/pp_od_clk_voltage | grep -i fan
```

### 2.8 性能压力测试脚本

```bash
#!/bin/bash
# gpu_stress_test.sh - GPU 稳定性压力测试

echo "=== GPU Stability Stress Test ==="
echo ""

# 测试参数
TEST_DURATION=${1:-300}  # 默认 5 分钟
MONITOR_INTERVAL=5

echo "Test Duration: ${TEST_DURATION}s"
echo "Monitor Interval: ${MONITOR_INTERVAL}s"
echo ""

# 创建日志文件
LOG_FILE="/tmp/gpu_stress_$(date +%Y%m%d_%H%M%S).log"
touch "$LOG_FILE"

# 后台运行监视器
(
    while true; do
        TIMESTAMP=$(date +%Y-%m-%d_%H:%M:%S)
        GFX_CLK=$(cat /sys/class/drm/card0/device/pp_dpm_sclk 2>/dev/null | grep '\*' | awk '{print $2}')
        MEM_CLK=$(cat /sys/class/drm/card0/device/pp_dpm_mclk 2>/dev/null | grep '\*' | awk '{print $2}')
        TEMP=$(cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input 2>/dev/null)
        if [ -n "$TEMP" ]; then
            TEMP=$((TEMP / 1000))
        fi
        POWER=$(cat /sys/class/drm/card0/device/power1_average 2>/dev/null)
        if [ -n "$POWER" ]; then
            POWER=$((POWER / 1000000))
        fi
        
        echo "$TIMESTAMP | CLK:${GFX_CLK:-N/A} | MEM:${MEM_CLK:-N/A} | TEMP:${TEMP:-N/A}C | PWR:${POWER:-N/A}W" >> "$LOG_FILE"
        sleep $MONITOR_INTERVAL
    done
) &
MONITOR_PID=$!

# 执行压力测试
echo "Starting stress test..."
echo "Using glmark2 for GPU stress..."

# 检查 glmark2 是否安装
if command -v glmark2 &> /dev/null; then
    glmark2 --run-forever --duration $TEST_DURATION &
    GLMARK_PID=$!
elif command -v glxgears &> /dev/null; then
    glxgears &
    GLMARK_PID=$!
else
    echo "WARNING: No GPU benchmark tool found"
    echo "Installing glmark2..."
    sudo apt install -y glmark2
    glmark2 --run-forever --duration $TEST_DURATION &
    GLMARK_PID=$!
fi

# 等待测试完成
wait $GLMARK_PID 2>/dev/null

# 停止监控
kill $MONITOR_PID 2>/dev/null

echo ""
echo "=== Stress Test Complete ==="
echo "Log file: $LOG_FILE"

# 分析结果
echo ""
echo "=== 结果分析 ==="
echo "最高温度: $(awk -F'|' '{print $4}' "$LOG_FILE" | grep -oP '\d+' | sort -rn | head -1)°C"
echo "最高功耗: $(awk -F'|' '{print $5}' "$LOG_FILE" | grep -oP '\d+' | sort -rn | head -1)W"
echo "平均频率: $(awk -F'|' '{print $2}' "$LOG_FILE" | grep -oP '\d+' | awk '{sum+=$1; count++} END {print sum/count}')MHz"

# 检查是否有错误
if grep -qi "error\|fail\|hang\|reset" "$LOG_FILE"; then
    echo ""
    echo "WARNING: 检测到潜在问题！"
    grep -i "error\|fail\|hang\|reset" "$LOG_FILE"
    exit 1
else
    echo ""
    echo "测试通过 - 未检测到错误"
fi
```

## 案例分析

### 案例 1：RX 6800 XT 降压优化

**背景：**
RX 6800 XT 用户遇到 GPU 温度过高（结温 105°C），风扇噪音大。希望通过降压降低温度和功耗，同时尽量保持性能。

**分析方法：**

```bash
# 1. 记录基准性能
echo "=== 基准测试（未降压） ==="
glmark2 --run-forever --duration 60 &
BENCH_PID=$!
sleep 10

# 监控数据
echo "默认配置下数据:"
cat /sys/class/drm/card0/device/pp_od_clk_voltage
cat /sys/class/drm/card0/device/power1_average
cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input

kill $BENCH_PID 2>/dev/null

# 2. 应用降压
echo ""
echo "=== 应用降压 100mV ==="
echo "manual" | sudo tee /sys/class/drm/card0/device/power_dpm_force_performance_level
echo "vc 2 1100" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
echo "vc 1 850" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
echo "c" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage

# 3. 降压后测试
sleep 2
echo ""
echo "降压后数据:"
cat /sys/class/drm/card0/device/pp_od_clk_voltage
```

**结果分析：**

| 参数 | 降压前 | 降压后 | 变化 |
|------|--------|--------|------|
| 频率 (MHz) | 2250 | 2250 | 0% |
| 核心电压 (mV) | 1200 | 1100 | -8.3% |
| 功耗 (W) | 285 | 225 | -21% |
| 温度 (°C) | 95 | 78 | -17.9% |
| 风扇转速 (%) | 75 | 55 | -26.7% |
| 性能 (FPS) | 120 | 117 | -2.5% |

**结论：**
降压 100mV 后，功耗降低 21%，温度降低 17°C，风扇噪音显著下降，而性能仅损失 2.5%。这是非常有效的优化方案。

### 案例 2：Radeon Pro W7900 超频调试

**背景：**
专业工作站使用 Radeon Pro W7900 进行 3D 渲染，需要提高渲染速度。用户希望在保证稳定的前提下进行适度的超频。

**超频策略：**

```bash
#!/bin/bash
# pro_w7900_oc.sh - W7900 安全超频

echo "=== Radeon Pro W7900 Overclocking ==="

# 1. 读取原厂配置
echo "1. 备份原厂配置"
cat /sys/class/drm/card0/device/pp_od_clk_voltage > /tmp/stock_od_table.txt

# 2. 逐步增加频率
echo "2. 应用超频配置"

# GFXCLK +100MHz
echo "s 0 2600" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage

# MEMCLK +300MHz (显存频率)
echo "m 0 1400" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage

# 增加 PPT +10%
echo "p 275000000" | sudo tee /sys/class/drm/card0/device/power1_cap

# 提交
echo "c" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage

# 3. 稳定性验证
echo "3. 运行稳定性测试"
./gpu_stress_test.sh 600  # 10 分钟压力测试

if [ $? -eq 0 ]; then
    echo "超频成功！"
else
    echo "超频不稳定，回退到出厂配置"
    cat /tmp/stock_od_table.txt | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
fi
```

### 案例 3：笔记本 GPU 动态电源管理调优

**背景：**
AMD 笔记本（RDNA 3 集成显卡）在电池模式下性能下降严重。用户需要在电池模式下获得更好的 GPU 性能，同时保持合理续航。

**调优方案：**

```bash
#!/bin/bash
# laptop_gpu_tuning.sh - 笔记本 GPU 电源优化

echo "=== Laptop GPU Power Tuning ==="

# 1. 检测当前电源状态
POWER_SOURCE=$(cat /sys/class/power_supply/AC*/online 2>/dev/null)
if [ "$POWER_SOURCE" == "1" ]; then
    echo "电源模式: AC (电源适配器)"
    POWER_MODE="performance"
else
    echo "电源模式: BATTERY"
    POWER_MODE="low"
fi

# 2. 设置电源策略
echo "设置电源策略: $POWER_MODE"
echo "$POWER_MODE" | sudo tee /sys/class/drm/card0/device/power_dpm_force_performance_level

# 3. 电池模式优化
if [ "$POWER_SOURCE" != "1" ]; then
    echo ""
    echo "电池模式优化:"
    
    # 设置 GFXCLK 上限（节省功耗）
    echo "s 0 1800" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
    
    # 设置 MEMCLK 上限
    echo "m 0 800" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
    
    # 降低功耗限制
    echo "p 150000000" | sudo tee /sys/class/drm/card0/device/power1_cap
    
    # 提交
    echo "c" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
    
    echo "已应用电池优化配置"
fi

# 4. 显示最终配置
echo ""
echo "=== 最终配置 ==="
cat /sys/class/drm/card0/device/pp_od_clk_voltage
echo ""
echo "电源策略: $(cat /sys/class/drm/card0/device/power_dpm_force_performance_level)"
```

## 相关链接

- AMD PowerPlay 技术介绍: https://www.amd.com/en/technologies/powerplay
- AMDGPU 驱动文档 (kernel.org): https://www.kernel.org/doc/html/latest/gpu/amdgpu.html
- AMDGPU PM sysfs 接口: https://www.kernel.org/doc/html/latest/gpu/amdgpu.html#power-management
- Coreboot PowerPlay 支持: https://doc.coreboot.org/soc/amd/powerplay.html
- Radeon OverDrive 指南: https://www.amd.com/en/support/kb/faq/gpu-135

## 今日小结

**主要内容回顾：**

1. **PowerPlay 架构**：了解了 AMDGPU PowerPlay 的三层架构（用户空间 → 内核驱动 → SMU 固件）和 DPM 状态机的工作机制。

2. **OD Table**：掌握了 OD Table 的数据结构（频率域、V/F 曲线、功耗限制）和通过 sysfs 接口修改 OD 参数的方法。

3. **PP_Table**：理解了 PP_Table 作为 VBIOS 默认配置表的作用，包括 DPM 状态表、温度阈值、风扇曲线等配置。

4. **OverDrive 技术**：学习了 GPU 超频和降压的操作方法，理解了安全调整范围和稳定性测试的重要性。

5. **DPM 状态管理**：掌握了 DPM 状态的升/降频条件、延迟参数和状态切换流程。

**关键命令回顾：**
- `cat /sys/class/drm/card0/device/pp_od_clk_voltage` - 读取 OD Table
- `echo "s 0 2600" | sudo tee ...` - 设置频率点
- `echo "vc 2 1100" | sudo tee ...` - 设置电压点
- `echo "c" | sudo tee ...` - 提交 OD 参数
- `echo "auto|low|high|manual" | sudo tee .../power_dpm_force_performance_level` - 设置电源策略

**注意事项：**
- 修改 OD Table 前务必备份原厂配置
- 超频时必须确保充分散热
- 建议逐步调整，每次调整后运行稳定性测试
- 降压是更安全的优化方式，可显著降低功耗和温度

## PowerPlay 状态切换策略深入分析

DPM 状态切换的核心在于负载预测和迟滞控制。PowerPlay 使用一种称为 PowerTune 的算法来实时评估 GPU 负载并决定最佳 DPM 状态。

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    PowerPlay 状态切换决策流程                              │
│                                                                          │
│  输入信号:                   处理管道:                   输出:            │
│  ┌────────────┐         ┌──────────────────┐        ┌──────────────┐    │
│  │ GPU 利用率   │         │  信号采集          │        │  DPM 状态 ID  │    │
│  │ (0-100%)   │────────►│  - 每个时钟域采样   │────────►│  0 = DPM0    │    │
│  └────────────┘         │  - 移动平均滤波     │        │  1 = DPM1    │    │
│  ┌────────────┐         │  - 去噪处理         │        │  2 = DPM2    │    │
│  │ 帧率 (FPS)   │         └────────┬─────────┘        │  3 = DPM3    │    │
│  └────────────┘                  │                   │  4 = DPM4    │    │
│  ┌────────────┐                  ▼                   └──────────────┘    │
│  │ 功耗 (W)    │         ┌──────────────────┐                            │
│  └────────────┘         │  负载评估器        │                            │
│  ┌────────────┐         │  - 计算短时平均负载 │                            │
│  │ 温度 (°C)   │         │  - 计算长时平均负载 │                            │
│  └────────────┘         │  - 检测负载突变     │                            │
│  ┌────────────┐         └────────┬─────────┘                            │
│  │ 电流 (A)    │                  │                                       │
│  └────────────┘                  ▼                                       │
│                          ┌──────────────────┐                            │
│                          │  状态决策引擎      │                            │
│                          │  - 比较阈值       │                            │
│                          │  - 检查迟滞条件    │                            │
│                          │  - 考虑温度/功耗   │                            │
│                          │  - 应用 DPM 偏好  │                            │
│                          └────────┬─────────┘                            │
│                                   │                                       │
│                                   ▼                                       │
│                          ┌──────────────────┐                            │
│                          │  SMU 消息生成      │                            │
│                          │  - 构建 MSG 参数   │                            │
│                          │  - 发送到 SMU     │                            │
│                          │  - 等待执行完成    │                            │
│                          └──────────────────┘                            │
└──────────────────────────────────────────────────────────────────────────┘
```

### 自适应负载预测算法

PowerPlay 使用指数加权移动平均（EWMA）算法平滑负载信号：

```
short_avg(t) = α * util(t) + (1-α) * short_avg(t-1)
long_avg(t)  = β * util(t) + (1-β) * long_avg(t-1)

其中:
  α = 0.3 (短时间常数，约 100ms)
  β = 0.05 (长时间常数，约 500ms)
  util(t) = 当前 GPU 利用率采样
```

**升频条件（Up-ramp）：**
- short_avg(t) > up_threshold (默认 70%)
- AND long_avg(t) > up_threshold * 0.8
- 持续 up_hysteresis (100ms) 以上

**降频条件（Down-ramp）：**
- short_avg(t) < down_threshold (默认 30%)
- AND long_avg(t) < down_threshold * 1.2
- 持续 down_hysteresis (500ms) 以上

**紧急降频（Emergency Down）：**
- 温度 > thotspot_limit (默认 105°C)
- 或功耗 > PPT (Package Power Target)
- 或电流 > EDC (Electrical Design Current)
- 立即降频一个或多个 DPM 等级

### PowerPlay Playlist 配置规则

PowerPlay Playlist 定义了不同场景下的 DPM 状态切换策略：

| 场景 | 策略名称 | 升频速度 | 降频速度 | 目标 DPM 等级 |
|------|----------|---------|---------|--------------|
| 3D 游戏 | Performance | 快 (50ms) | 慢 (1s) | DPM3-DPM4 |
| 视频播放 | Video | 中 (100ms) | 中 (500ms) | DPM1-DPM2 |
| 桌面空闲 | Idle | 慢 (500ms) | 快 (100ms) | DPM0 |
| 计算任务 | Compute | 快 (50ms) | 慢 (2s) | DPM4+ |
| VR 应用 | VR | 极快 (20ms) | 慢 (1s) | DPM3-DPM4 |

## PowerPlay 多架构实现差异

AMD GPU 在不同架构中使用不同版本的 SMU 和 PowerPlay 实现：

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    AMD GPU 架构 PowerPlay 演进                             │
│                                                                          │
│  架构        │  SMU 版本  │  OD Table 特点         │ PP_Table 变更        │
│──────────────┼───────────┼──────────────────────┼─────────────────────│
│  RDNA 1      │  SMU v10  │  基础 OD 功能         │ 传统 PP_Table        │
│  (RX 5000)   │           │  仅 GFXCLK/MEMCLK     │ 固定 DPM 状态数      │
│──────────────┼───────────┼──────────────────────┼─────────────────────│
│  RDNA 2      │  SMU v11  │  V/F 曲线调整         │ 新增 SPPT/APPPT      │
│  (RX 6000)   │           │  风扇曲线自定义        │ 新增温度传感器        │
│──────────────┼───────────┼──────────────────────┼─────────────────────│
│  RDNA 3      │  SMU v13  │  每时钟域独立 OD       │ 统一电源域管理       │
│  (RX 7000)   │           │  AVFS 自适应电压       │ 新增 VF 表分段       │
│──────────────┼───────────┼──────────────────────┼─────────────────────│
│  RDNA 4      │  SMU v14  │  AI 辅助电源优化       │ 动态 PP_Table        │
│  (RX 8000)   │           │  ML 负载预测           │ 在线学习调整        │
│──────────────┼───────────┼──────────────────────┼─────────────────────│
│  CDNA 2      │  SMU v11  │  计算专用 OD           │ FP32/FP64 功耗分离   │
│  (MI200)     │           │  大页内存优化          │ ECC 功耗预算         │
│──────────────┼───────────┼──────────────────────┼─────────────────────│
│  CDNA 3      │  SMU v13  │  Infinity Fabric OD   │ 多芯片功耗协同       │
│  (MI300)     │           │  分区电源管理          │ 芯片间功耗均衡      │
└──────────────────────────────────────────────────────────────────────────┘
```

### RDNA 1 (SMU v10) 特点

- 使用 `smu_v10_0_pptable.h` 中的传统 PP_Table 结构
- OD Table 仅支持 GFXCLK 和 MEMCLK 的频率偏移
- 不支持 V/F 曲线逐点调整
- DPM 状态数固定为 3 个 (DPM0-DPM2)
- 风扇控制仅支持自动模式

### RDNA 2 (SMU v11) 改进

- 引入完整的 OD Table 结构（`smu_v11_0_overdrive_table`）
- 支持 V/F 曲线的逐点电压调整（3 个 VF 点）
- 新增风扇曲线自定义支持
- 引入 SPPT（Slow PPT）和 APPPT（Average Package Power PPT）限制
- DPM 状态扩展到 5 个 (DPM0-DPM4)
- 支持 power_dpm_force_performance_level 的四种模式

### RDNA 3 (SMU v13) 新特性

- 重新设计的 PP_Table 结构（`smu_v13_0_powerplay_table`）
- 每时钟域独立的 OD 配置（GFXCLK、MEMCLK、SOCLK、FCLK、VCLK、DCLK）
- AVFS（Adaptive Voltage Frequency Scaling）自适应电压调节
- 更细粒度的电源域管理
- 支持动态 V/F 曲线分段
- VF 点从 3 个扩展到 8 个

## SMU 消息传递机制详解

PowerPlay 通过 SMU 消息与固件通信。消息传递基于 MMIO 寄存器接口：

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    SMU 消息传递机制                                       │
│                                                                          │
│  驱动侧 (amdgpu)                  SMU 固件侧                              │
│  ┌──────────────┐               ┌────────────────────────────────────┐  │
│  │ smu_send_msg │               │  SMU Message Handler               │  │
│  │ ────────────►│──── MMIO ────►│  ┌──────────────────────────────┐  │  │
│  │              │               │  │  消息队列                     │  │  │
│  │  1. 写命令   │               │  │  ├─ 0x01: SetSoftMinGfxClk  │  │  │
│  │     到 MP1   │               │  │  ├─ 0x02: SetSoftMaxGfxClk  │  │  │
│  │     register │               │  │  ├─ ...                      │  │  │
│  │              │               │  │  └─ 0xFF: GetMetrics        │  │  │
│  │  2. 轮询     │               │  └──────────────┬───────────────┘  │  │
│  │     完成标志 │               │                 │                  │  │
│  │              │               │                 ▼                  │  │
│  │  3. 读取     │               │  ┌──────────────────────────────┐  │  │
│  │     返回值   │◄── MMIO ─────◄──┤  执行引擎                     │  │  │
│  └──────────────┘               │  │  - 解析消息参数              │  │  │
│                                 │  │  - 更新内部状态              │  │  │
│                                 │  │  - 触发硬件控制              │  │  │
│  MP1 Registers:                 │  │  - 返回执行结果              │  │  │
│  0x3B105000: Message            │  └──────────────────────────────┘  │  │
│  0x3B105004: Argument           │                                    │  │
│  0x3B105008: Response           │                                    │  │
│  0x3B10500C: Status             └────────────────────────────────────┘  │
│                                                                          │
│  消息发送步骤:                                                           │
│  1. 检查 SMU 状态寄存器（确保空闲）                                       │
│  2. 写入参数到 Argument 寄存器                                            │
│  3. 写入消息 ID 到 Message 寄存器                                         │
│  4. 轮询 Status 寄存器直到完成                                              │
│  5. 从 Response 寄存器读取返回值                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 完整 SMU v11 消息表

| 消息 ID | 名称 | 参数 | 返回值 | 说明 |
|---------|------|------|--------|------|
| 0x01 | SetSoftMinGfxClk | freq (MHz) | 0/err | 设置 GFXCLK 软最小值 |
| 0x02 | SetSoftMaxGfxClk | freq (MHz) | 0/err | 设置 GFXCLK 软最大值 |
| 0x03 | SetSoftMinMemClk | freq (MHz) | 0/err | 设置 MEMCLK 软最小值 |
| 0x04 | SetSoftMaxMemClk | freq (MHz) | 0/err | 设置 MEMCLK 软最大值 |
| 0x05 | SetHardMinGfxClk | freq (MHz) | 0/err | 设置 GFXCLK 硬最小值 |
| 0x06 | SetHardMaxGfxClk | freq (MHz) | 0/err | 设置 GFXCLK 硬最大值 |
| 0x07 | SetHardMinMemClk | freq (MHz) | 0/err | 设置 MEMCLK 硬最小值 |
| 0x08 | SetHardMaxMemClk | freq (MHz) | 0/err | 设置 MEMCLK 硬最大值 |
| 0x09 | SetSoftMinSocClk | freq (MHz) | 0/err | 设置 SOCLK 软最小值 |
| 0x0A | SetSoftMaxSocClk | freq (MHz) | 0/err | 设置 SOCLK 软最大值 |
| 0x0B | SetHardMinSocClk | freq (MHz) | 0/err | 设置 SOCLK 硬最小值 |
| 0x0C | SetHardMaxSocClk | freq (MHz) | 0/err | 设置 SOCLK 硬最大值 |
| 0x10 | TransferTableDram2Smu | table_id | 0/err | 将 OD Table 上传到 SMU SRAM |
| 0x11 | TransferTableSmu2Dram | table_id | data | 从 SMU SRAM 下载 OD Table |
| 0x12 | SetDpmTable | table_ptr | 0/err | 设置 DPM 状态表 |
| 0x13 | GetDpmTable | - | table | 获取当前 DPM 状态表 |
| 0x20 | SetPowerLimit | limit (W) | 0/err | 设置 PPT 功耗限制 |
| 0x21 | SetTemperatureLimit | limit (°C) | 0/err | 设置温度限制 |
| 0x22 | SetCurrentLimit | limit (A) | 0/err | 设置电流限制 |
| 0x23 | GetPowerLimit | - | limit (W) | 获取当前 PPT 限制 |
| 0x24 | GetTemperatureLimit | - | limit (°C) | 获取当前温度限制 |
| 0x30 | GetCurrentPower | - | power (W) | 获取当前 GPU 功耗 |
| 0x31 | GetCurrentTemperature | sensor_id | temp (°C) | 获取指定传感器温度 |
| 0x32 | GetCurrentCurrent | - | current (A) | 获取当前电流 |
| 0x33 | GetCurrentVoltage | domain | volt (mV) | 获取当前电压 |
| 0x34 | GetCurrentFrequency | domain | freq (MHz) | 获取当前频率 |
| 0x40 | SetFanControl | mode | 0/err | 设置风扇控制模式 |
| 0x41 | SetFanSpeedPercent | speed (%) | 0/err | 设置风扇转速百分比 |
| 0x42 | SetFanSpeedRpm | speed (RPM) | 0/err | 设置风扇转速 (RPM) |
| 0x43 | GetFanSpeed | - | speed | 获取当前风扇速度 |
| 0x44 | SetFanCurve | curve_id | 0/err | 设置风扇曲线 |
| 0x45 | GetFanCurve | - | curve | 获取当前风扇曲线 |
| 0x50 | SetPerformanceLevel | level | 0/err | 设置性能级别 |
| 0x51 | GetPerformanceLevel | - | level | 获取当前性能级别 |
| 0x52 | SetDpmState | state_id | 0/err | 锁定 DPM 状态 |
| 0x53 | GetDpmState | - | state_id | 获取当前 DPM 状态 |
| 0x60 | GetMetricsTable | - | metrics | 获取 SMU 指标表 |
| 0x61 | GetGpuMetrics | - | metrics | 获取 GPU 指标 |
| 0x62 | GetPptTable | - | pptable | 获取 PowerPlay 表 |
| 0x70 | SetGpuIdle | enable | 0/err | 设置 GPU 空闲状态 |
| 0x71 | GetGpuIdle | - | status | 获取 GPU 空闲状态 |
| 0x80 | StartMonitor | monitor_id | 0/err | 启动监控 |
| 0x81 | StopMonitor | monitor_id | 0/err | 停止监控 |
| 0x82 | GetMonitorData | - | data | 获取监控数据 |
| 0xFF | GetDriverVersion | - | version | 获取固件版本 |

## PowerTune 技术详解

PowerTune 是 AMD 的 GPU 功耗管理技术，作为 PowerPlay 的补充机制，专门负责功耗预算的动态分配。

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    PowerTune 功耗预算分配机制                              │
│                                                                          │
│  总功耗预算 (PPT): 235W                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐             │  │
│  │  │ GFX Core │  │ 显存     │  │ SoC     │  │ 其他     │             │  │
│  │  │ 150W    │  │ 30W     │  │ 35W     │  │ 20W     │             │  │
│  │  │ (63.8%) │  │ (12.8%) │  │ (14.9%) │  │ (8.5%)  │             │  │
│  │  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘             │  │
│  │       │             │           │           │                    │  │
│  │       ▼             ▼           ▼           ▼                    │  │
│  │  ┌────────────────────────────────────────────────────────────┐  │  │
│  │  │              PowerTune 分配器                               │  │  │
│  │  │  - 实时监控各域功耗                                          │  │  │
│  │  │  - 动态调整各域预算                                          │  │  │
│  │  │  - 超出预算时发送降频请求                                    │  │  │
│  │  │  - 空闲时回收未用预算                                        │  │  │
│  │  └────────────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  PowerTune 三种限制机制:                                                  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  1. PPT (Package Power Target) - 封装功耗目标                       │  │
│  │     总 GPU 封装的最大功耗限制，超过后触发降频                         │  │
│  │     默认: 235W (可调整范围: 150W - 300W)                          │  │
│  │                                                                     │  │
│  │  2. TDC (Thermal Design Current) - 热设计电流                      │  │
│  │     基于温度的电流限制，防止过热导致损坏                              │  │
│  │     默认: 150A (可调整范围: 100A - 200A)                           │  │
│  │                                                                     │  │
│  │  3. EDC (Electrical Design Current) - 电气设计电流                  │  │
│  │     瞬时电流峰值限制，保护供电电路                                    │  │
│  │     默认: 200A (可调整范围: 120A - 260A)                           │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

### PowerTune 降频策略

当 GPU 功耗接近 PPT 限制时，PowerTune 会逐步采取降频措施：

```
PowerTune Throttle Levels:
Level 0: 正常状态 (功耗 < PPT * 0.8)
         无限制，GPU 自由运行在最高性能

Level 1: 轻度限制 (PPT * 0.8 ≤ 功耗 < PPT * 0.95)
         轻微降低 GFXCLK 上限 (-50MHz)
         降低电压增量步长

Level 2: 中度限制 (PPT * 0.95 ≤ 功耗 < PPT * 1.0)
         主动下降一个 DPM 状态
         限制 MEMCLK 上限 (-100MHz)
         降低 SOCLK 频率

Level 3: 重度限制 (PPT * 1.0 ≤ 功耗)
         强制降至 DPM0
         限制 GFXCLK < 1000MHz
         触发紧急降频保护

Level 4: 临界状态 (温度 > 105°C 或功耗 > PPT * 1.1)
         硬件级保护触发
         GPU 进入热节流模式
         频率降至最低安全值
```

## 常见问题与故障排查

### 问题 1: OD Table 写入被拒绝

**现象：**
```bash
$ echo "s 0 3000" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
echo: write error: Invalid argument
```

**可能原因：**
- 频率值超出硬件支持的 OD_RANGE 范围
- GPU 不支持特定时钟域的 OD 调整
- 需要先设置 power_dpm_force_performance_level 为 manual

**解决方案：**
```bash
# 1. 检查有效的频率范围
cat /sys/class/drm/card0/device/pp_od_clk_voltage | grep -A5 "OD_RANGE"

# 2. 设置为手动模式
echo "manual" | sudo tee /sys/class/drm/card0/device/power_dpm_force_performance_level

# 3. 使用合法范围内的值重试
echo "s 0 2700" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage
```

### 问题 2: 修改 OD Table 后系统不稳定

**现象：**
- GPU 驱动超时（drm watchdog reset）
- 屏幕闪烁或黑屏
- 系统完全冻结

**紧急恢复步骤：**
```bash
# 方法 1: 重置 OD Table 为默认值
echo "r" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage

# 方法 2: 重置电源策略为 auto
echo "auto" | sudo tee /sys/class/drm/card0/device/power_dpm_force_performance_level

# 方法 3: 完全重置 GPU (通过 sysfs)
echo "1" | sudo tee /sys/class/drm/card0/device/pp_od_clk_voltage

# 方法 4: 重启 GPU 驱动
sudo modprobe -r amdgpu && sudo modprobe amdgpu
```

### 问题 3: PP_Table 加载失败

**现象（dmesg 输出）：**
```
[    5.123456] amdgpu: [powerplay] Failed to load PP_Table from VBIOS
[    5.123457] amdgpu: [powerplay] Using fallback PP_Table parameters
```

**可能原因：**
- VBIOS 损坏或不完整
- GPU 不支持当前驱动版本
- UEFI GOP 驱动问题

**诊断方法：**
```bash
# 检查 VBIOS 版本
cat /sys/class/drm/card0/device/vbios_version

# 检查驱动日志
dmesg | grep -i "pp_table\|powerplay\|smc\|smu"

# 检查 VBIOS ROM 大小
cat /sys/class/drm/card0/device/rom | wc -c

# 检查驱动版本
cat /sys/module/amdgpu/version
cat /sys/kernel/debug/dri/0/amdgpu_pm_info 2>/dev/null | head -20
```

### 问题 4: 风扇控制无效

**现象：**
- 自定义风扇曲线不生效
- 风扇转速不随温度变化

**检查步骤：**
```bash
# 1. 检查风扇控制模式
cat /sys/class/drm/card0/device/hwmon/hwmon*/pwm1_enable
# 0: 无控制 1: 手动 2: 自动

# 2. 检查风扇曲线是否写入成功
cat /sys/class/drm/card0/device/hwmon/hwmon*/pwm1

# 3. 检查温度传感器
cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input

# 4. 临时手动设置风扇转速测试
echo "1" | sudo tee /sys/class/drm/card0/device/hwmon/hwmon*/pwm1_enable
echo "150" | sudo tee /sys/class/drm/card0/device/hwmon/hwmon*/pwm1
```

### 问题 5: 多 GPU 中 OD 配置不一致

**现象：**
- CrossFire 模式下各 GPU 频率不同
- 计算集群中 GPU 性能差异大

**同步配置方法：**
```bash
#!/bin/bash
# sync_od_config.sh - 多 GPU OD 配置同步

echo "=== Multi-GPU OD Configuration Sync ==="

for gpu in /sys/class/drm/card*/device; do
    CARD=$(basename $(dirname $gpu))
    echo ""
    echo "--- Configuring $CARD ---"
    
    # 读取当前配置
    echo "Current OD Table:"
    cat $gpu/pp_od_clk_voltage 2>/dev/null | head -10
    
    # 设置为手动模式
    echo "manual" | sudo tee $gpu/power_dpm_force_performance_level > /dev/null
    
    # 应用统一配置
    echo "s 0 2600" | sudo tee $gpu/pp_od_clk_voltage > /dev/null
    echo "m 0 1200" | sudo tee $gpu/pp_od_clk_voltage > /dev/null
    echo "c" | sudo tee $gpu/pp_od_clk_voltage > /dev/null
    
    echo "Applied: GFXCLK=2600MHz, MEMCLK=1200MHz"
done

echo ""
echo "=== All GPUs configured ==="
```

## PP_Table 二进制格式解析指南

PP_Table 在 VBIOS 中以二进制格式存储。解析 PP_Table 需要了解其二进制布局：

```
PP_Table 二进制布局 (SMU v11):
┌─────────────────────────────────────────────────────────────────────────┐
│ Offset  │ Size   │ 字段                      │ 说明                     │
│─────────┼────────┼───────────────────────────┼─────────────────────────│
│ 0x00    │ 2      │ table_id                  │ 表标识 (0xPP)           │
│ 0x02    │ 2      │ table_version             │ 表版本号                │
│ 0x04    │ 2      │ size                      │ 表总大小 (bytes)        │
│ 0x06    │ 2      │ reserved                  │ 保留                    │
│ 0x08    │ 4      │ TDP                       │ 热设计功耗 (W)          │
│ 0x0C    │ 4      │ TDC                       │ 热设计电流 (A)          │
│ 0x10    │ 4      │ EDC                       │ 电气设计电流 (A)        │
│ 0x14    │ 4      │ power_reserve             │ 功率储备 (W)           │
│ 0x18    │ 4      │ soc_power_limit           │ SoC 功耗限制            │
│ 0x1C    │ 4      │ gfx_power_limit           │ GFX 功耗限制            │
│ 0x20    │ 4      │ mem_power_limit           │ 显存功耗限制            │
│ 0x24    │ 4      │ total_power_limit         │ 总功耗上限              │
│ 0x28    │ 4      │ thermal_limit_edge        │ 边缘温度上限 (°C)       │
│ 0x2C    │ 4      │ thermal_limit_hotspot     │ 热点温度上限 (°C)       │
│ 0x30    │ 4      │ thermal_limit_mem         │ 显存温度上限 (°C)       │
│ 0x34    │ 4      │ thermal_limit_vr          │ VR 温度上限 (°C)        │
│ 0x38    │ 8×3×8  │ dpm_states                │ DPM 状态表 (3域×8状态)  │
│         │        │  [domain][state]:          │ 每个状态:               │
│         │        │   - freq: u32             │  频率 (MHz)             │
│         │        │   - voltage: u32          │  电压 (mV)              │
│         │        │   - power: u32            │  功耗 (W)               │
│         │        │   - current: u32          │  电流 (A)               │
│ 0x98    │ 4×4    │ thermal_limits[4]         │ 4 个温度传感器阈值       │
│ 0xA8    │ 8×8    │ fan_curve[8]              │ 8 点风扇曲线            │
│ 0xE8    │ 4      │ vdd_min                   │ 最小核心电压            │
│ 0xEC    │ 4      │ vdd_max                   │ 最大核心电压            │
│ 0xF0    │ 4      │ vdd_mem_min               │ 最小显存电压            │
│ 0xF4    │ 4      │ vdd_mem_max               │ 最大显存电压            │
│ 0xF8    │ 4      │ power_domain_map          │ 电源域位图              │
│ 0xFC    │ 4      │ platform_caps             │ 平台能力位图            │
└─────────────────────────────────────────────────────────────────────────┘
```

### PP_Table 解析脚本

```bash
#!/bin/bash
# parse_pptable.sh - PP_Table 二进制解析

PPTABLE_FILE=${1:-"/sys/kernel/debug/dri/0/amdgpu_vbios"}

if [ ! -f "$PPTABLE_FILE" ]; then
    echo "PP_Table file not found: $PPTABLE_FILE"
    exit 1
fi

echo "=== PP_Table Binary Analysis ==="
echo "File: $PPTABLE_FILE"
echo ""

# 读取 PP_Table 头部信息
echo "--- Header ---"
# 使用 xxd 解析二进制数据
xxd -l 64 "$PPTABLE_FILE" | head -10

echo ""
echo "--- Power Limits ---"
echo "TDP (offset 0x08): $(xxd -s 0x08 -l 4 "$PPTABLE_FILE" | awk '{print $NF}') W"
echo "TDC (offset 0x0C): $(xxd -s 0x0C -l 4 "$PPTABLE_FILE" | awk '{print $NF}') A"
echo "EDC (offset 0x10): $(xxd -s 0x10 -l 4 "$PPTABLE_FILE" | awk '{print $NF}') A"

echo ""
echo "--- Thermal Limits ---"
echo "Edge (offset 0x28): $(xxd -s 0x28 -l 4 "$PPTABLE_FILE" | awk '{print $NF}') °C"
echo "Hotspot (offset 0x2C): $(xxd -s 0x2C -l 4 "$PPTABLE_FILE" | awk '{print $NF}') °C"
echo "Memory (offset 0x30): $(xxd -s 0x30 -l 4 "$PPTABLE_FILE" | awk '{print $NF}') °C"

echo ""
echo "--- Voltage Limits ---"
echo "VDD Min: $(xxd -s 0xE8 -l 4 "$PPTABLE_FILE" | awk '{print $NF}') mV"
echo "VDD Max: $(xxd -s 0xEC -l 4 "$PPTABLE_FILE" | awk '{print $NF}') mV"

echo ""
echo "=== Analysis Complete ==="
```

### 使用 umr 工具解析 PP_Table

UMR (User Mode Register) 是 AMD 的 GPU 调试工具，可以解析 PP_Table：

```bash
# 安装 UMR
git clone https://gitlab.freedesktop.org/tomstdenis/umr.git
cd umr
cmake -B build && make -C build -j$(nproc)

# 使用 UMR 查看 PowerPlay 信息
sudo ./build/umr -O "powerplay" -d card0

# 查看 PP_Table 原始数据
sudo ./build/umr -O "read_pptable" -d card0

# 查看 SMU 状态
sudo ./build/umr -O "smu_state" -d card0

# 查看 DPM 状态详情
sudo ./build/umr -O "dpm" -d card0
```

### Python PP_Table 解析器

```python
#!/usr/bin/env python3
"""
pptable_parser.py - PP_Table 二进制解析器
用法: python3 pptable_parser.py /path/to/vbios.bin
"""

import struct
import sys

class PPTableParser:
    def __init__(self, data):
        self.data = data
        
    def parse_header(self):
        """解析 PP_Table 头部"""
        header = {
            'table_id': struct.unpack_from('<H', self.data, 0x00)[0],
            'table_version': struct.unpack_from('<H', self.data, 0x02)[0],
            'size': struct.unpack_from('<H', self.data, 0x04)[0],
        }
        return header
    
    def parse_power_limits(self):
        """解析功耗限制"""
        return {
            'TDP': struct.unpack_from('<I', self.data, 0x08)[0],
            'TDC': struct.unpack_from('<I', self.data, 0x0C)[0],
            'EDC': struct.unpack_from('<I', self.data, 0x10)[0],
            'power_reserve': struct.unpack_from('<I', self.data, 0x14)[0],
            'soc_power_limit': struct.unpack_from('<I', self.data, 0x18)[0],
            'gfx_power_limit': struct.unpack_from('<I', self.data, 0x1C)[0],
            'mem_power_limit': struct.unpack_from('<I', self.data, 0x20)[0],
        }
    
    def parse_thermal_limits(self):
        """解析温度限制"""
        sensors = ['edge', 'hotspot', 'memory', 'vr']
        limits = {}
        for i, sensor in enumerate(sensors):
            offset = 0x28 + i * 4
            limits[sensor] = struct.unpack_from('<I', self.data, offset)[0]
        return limits
    
    def parse_dpm_states(self):
        """解析 DPM 状态表"""
        domains = ['GFX', 'MEM', 'SOC']
        dpm_data = {}
        offset = 0x38
        
        for domain in domains:
            states = []
            for state in range(8):
                freq = struct.unpack_from('<I', self.data, offset)[0]
                voltage = struct.unpack_from('<I', self.data, offset + 4)[0]
                power = struct.unpack_from('<I', self.data, offset + 8)[0]
                current = struct.unpack_from('<I', self.data, offset + 12)[0]
                
                if freq > 0:  # 跳过未使用的状态
                    states.append({
                        'state': state,
                        'freq_mhz': freq,
                        'voltage_mv': voltage,
                        'power_w': power,
                        'current_a': current
                    })
                offset += 16
            dpm_data[domain] = states
            
        return dpm_data
    
    def parse_fan_curve(self):
        """解析风扇曲线"""
        fan_points = []
        offset = 0xA8
        for i in range(8):
            temp = struct.unpack_from('<I', self.data, offset)[0]
            speed = struct.unpack_from('<I', self.data, offset + 4)[0]
            fan_points.append({'temp_c': temp, 'speed_pct': speed})
            offset += 8
        return fan_points
    
    def dump_all(self):
        """输出所有解析结果"""
        print("=== PP_Table Parsing Results ===")
        print()
        
        header = self.parse_header()
        print(f"Table ID:     0x{header['table_id']:04X}")
        print(f"Version:      {header['table_version']}")
        print(f"Size:         {header['size']} bytes")
        print()
        
        power = self.parse_power_limits()
        print("--- Power Limits ---")
        for key, val in power.items():
            print(f"  {key}: {val}")
        print()
        
        thermal = self.parse_thermal_limits()
        print("--- Thermal Limits (°C) ---")
        for sensor, limit in thermal.items():
            print(f"  {sensor}: {limit}")
        print()
        
        dpm = self.parse_dpm_states()
        print("--- DPM States ---")
        for domain, states in dpm.items():
            print(f"  {domain}:")
            for s in states:
                print(f"    State {s['state']}: {s['freq_mhz']}MHz @ {s['voltage_mv']}mV")
        print()
        
        fan = self.parse_fan_curve()
        print("--- Fan Curve ---")
        for p in fan:
            print(f"  {p['temp_c']}°C -> {p['speed_pct']}%")

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("Usage: pptable_parser.py <vbios_bin_file>")
        sys.exit(1)
    
    with open(sys.argv[1], 'rb') as f:
        data = f.read()
    
    parser = PPTableParser(data)
    parser.dump_all()
```

## OD Table 与 PP_Table 的历史演进

```
┌──────────────────────────────────────────────────────────────────────────┐
│              OD/PP Table 历史演进时间线                                    │
│                                                                          │
│  2011                     2017                    2020          2022     │
│  │                        │                      │             │        │
│  ▼                        ▼                      ▼             ▼        │
│  ┌────────────┐  ┌──────────────┐  ┌────────────┐  ┌───────────────┐   │
│  │ VLIW 架构   │  │ GCN 架构      │  │ RDNA 架构   │  │ RDNA 3 架构   │   │
│  │ (HD 6000)  │  │ (RX 400/500) │  │ (RX 5000)  │  │ (RX 7000)    │   │
│  ├────────────┤  ├──────────────┤  ├────────────┤  ├───────────────┤   │
│  │ PowerPlay  │  │ PowerTune 引入│  │ SMU v10    │  │ SMU v13      │   │
│  │ 初始版本    │  │ DPM 2.0     │  │ OD Table   │  │ 每域独立 OD   │   │
│  │ DPM 1.0   │  │ 多域管理     │  │ VF 曲线    │  │ AVFS         │   │
│  │ 无 OD      │  │ PP_Table 1.0│  │ 风扇曲线    │  │ 动态 VF 分段  │   │
│  └────────────┘  └──────────────┘  └────────────┘  └───────────────┘   │
│                                                                          │
│  关键技术里程碑:                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                                                                    │  │
│  │  2011: PowerPlay 首次引入，支持基本的 DPM 状态切换                  │  │
│  │  2013: DPM 2.0 引入多电源域管理（GFX/MEM/SOC）                    │  │
│  │  2015: PowerTune 加入功耗预算管理                                  │  │
│  │  2017: PP_Table 标准化，存储在 VBIOS 中                            │  │
│  │  2019: RDNA 引入 SMU v10，OD Table 可用户配置                      │  │
│  │  2020: RDNA 2 引入 SMU v11，VF 曲线调整和风扇曲线自定义            │  │
│  │  2022: RDNA 3 引入 SMU v13，AVFS 和独立时钟域 OD                  │  │
│  │  2024: RDNA 4 引入 SMU v14，AI 辅助电源优化                       │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

## 实战：S0ix 状态下的 PowerPlay 配置验证

S0ix (Modern Standby) 状态下，PowerPlay 需要协调 GPU 进入最低功耗状态：

```bash
#!/bin/bash
# s0ix_powerplay_test.sh - S0ix PowerPlay 验证

echo "=== S0ix PowerPlay Configuration Test ==="

# 1. 检查 S0ix 支持
echo "1. Checking S0ix support..."
cat /sys/power/mem_sleep 2>/dev/null | grep -q "s2idle"
if [ $? -eq 0 ]; then
    echo "   S2idle (S0ix) supported"
else
    echo "   S2idle not supported on this system"
fi

# 2. 记录进入 S0ix 前的 GPU 状态
echo ""
echo "2. Recording pre-suspend GPU state..."
echo "--- PowerPlay Configuration ---" > /tmp/gpu_s0ix_before.txt
cat /sys/class/drm/card0/device/pp_od_clk_voltage >> /tmp/gpu_s0ix_before.txt
echo "--- DPM States ---" >> /tmp/gpu_s0ix_before.txt
cat /sys/class/drm/card0/device/pp_dpm_sclk >> /tmp/gpu_s0ix_before.txt
echo "--- Power ---" >> /tmp/gpu_s0ix_before.txt
cat /sys/class/drm/card0/device/power1_average >> /tmp/gpu_s0ix_before.txt

# 3. 强制 GPU 进入低功耗状态
echo ""
echo "3. Forcing low power state..."
echo "low" | sudo tee /sys/class/drm/card0/device/power_dpm_force_performance_level

# 4. 触发 S0ix
echo ""
echo "4. Entering S0ix for 5 seconds..."
echo s2idle | sudo tee /sys/power/mem_sleep
echo deep | sudo tee /sys/power/state &
SLEEP_PID=$!
sleep 5
kill $SLEEP_PID 2>/dev/null

# 5. 唤醒后检查状态
echo ""
echo "5. Post-resume GPU state check..."
echo "=== Current DPM Level ==="
cat /sys/class/drm/card0/device/power_dpm_force_performance_level
echo ""
echo "=== Current Frequency ==="
cat /sys/class/drm/card0/device/pp_dpm_sclk
echo ""
echo "=== Power Consumption ==="
cat /sys/class/drm/card0/device/power1_average

# 6. 恢复高性能模式
echo ""
echo "6. Restoring performance mode..."
echo "auto" | sudo tee /sys/class/drm/card0/device/power_dpm_force_performance_level

# 7. 结果对比
echo ""
echo "=== S0ix Test Complete ==="
echo "Pre-suspend data saved to: /tmp/gpu_s0ix_before.txt"
echo "Verify GPU successfully entered and exited low power state."
```

## 进阶：AVFS (Adaptive Voltage Frequency Scaling)

RDNA 3 引入的 AVFS 技术可以根据芯片体质自动优化 V/F 曲线：

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    AVFS 自适应电压调节流程                                 │
│                                                                          │
│  芯片制造:                                                               │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  由于工艺偏差，每颗 GPU 芯片的电气特性不同                             │  │
│  │  快速芯片 (Fast Bin): 在相同电压下可达到更高频率                      │  │
│  │  慢速芯片 (Slow Bin): 需要更高电压达到相同频率                        │  │
│  │  典型芯片 (Typical Bin): 符合标称 V/F 曲线                           │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  AVFS 工作流程:                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  1. 初始化阶段:                                                     │  │
│  │     - GPU 上电后运行 AVFS 校准                                      │  │
│  │     - SMU 执行内置自检 (BIST)                                       │  │
│  │     - 检测芯片速度等级 (Fmax/Vmin)                                  │  │
│  │                                                                     │  │
│  │  2. 运行阶段:                                                       │  │
│  │     - 实时监控芯片性能和温度                                         │  │
│  │     - 根据温度补偿电压 (Temperature Inversion)                       │  │
│  │     - 动态调整 V/F 曲线偏移量                                       │  │
│  │                                                                     │  │
│  │  3. 老化补偿:                                                       │  │
│  │     - 芯片老化导致性能下降                                           │  │
│  │     - AVFS 逐步增加电压补偿                                          │  │
│  │     - 确保整个生命周期内的稳定性                                     │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  AVFS 偏移公式:                                                          │
│  V_actual = V_base + V_temp_comp + V_age_comp + V_bin_offset            │
│                                                                          │
│  其中:                                                                   │
│  V_base       = PP_Table 中的基础电压                                    │
│  V_temp_comp  = 温度补偿 (温度升高时电压需求降低)                         │
│  V_age_comp   = 老化补偿 (随时间递增)                                    │
│  V_bin_offset = 芯片速度等级偏移 (快速芯片为负值)                         │
└──────────────────────────────────────────────────────────────────────────┘
```

## 扩展思考

1. **GPU 自适应电压调节**：现代 GPU 支持自适应电压调节（Adaptive Voltage Scaling），驱动如何根据芯片体质自动调整 V/F 曲线？这与固定的 OD Table 参数有何关系？

2. **PowerPlay 与 ACPI 的协作**：PowerPlay 状态变化是否会触发 ACPI 通知？在系统电源状态（AC/DC）切换时，PowerPlay 如何响应？

3. **多 GPU 场景的 OD 配置**：在 CrossFire 或计算集群场景中，多张 GPU 如何保持 OD 配置的一致性？驱动程序如何处理不同 GPU 芯片体质的差异？

4. **安全与可靠性**：过度超频可能导致 GPU 退化甚至损坏，驱动程序中有哪些保护机制（如 watchdog、温度紧急停机、电流限制）？

5. **PP_Table 验证机制**：VBIOS 中的 PP_Table 是否有校验机制以防止篡改？驱动加载时如何验证 PP_Table 的合法性？

6. **AVFS 与 OD Table 的交互**：当用户通过 OD Table 手动设置 V/F 曲线时，AVFS 的自动调节功能会被覆盖还是叠加？两者如何协同工作？

7. **GPU 老化补偿算法**：驱动如何确定芯片的老化程度？AVFS 的老化补偿策略是否可以通过 OD Table 参数调整？

---

*文档创建于第 316 天，主题：OverDrive / PowerPlay 配置文件（OD / PP Table）*