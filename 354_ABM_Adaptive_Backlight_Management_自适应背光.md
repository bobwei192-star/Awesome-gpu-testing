# Day 354: ABM（Adaptive Backlight Management）自适应背光

## 学习目标

1. 理解 ABM 自适应背光管理的工作原理和节能机制
2. 掌握 AMDGPU ABM 的硬件实现架构（DCN 内的 ABM 模块）
3. 熟悉 ABM 级别的概念和不同级别的功耗/视觉差异
4. 学会通过 sysfs/debugfs 接口配置和调试 ABM
5. 理解 ABM 与 PSR 的协同工作关系
6. 掌握 ABM 在实际产品中的调优方法
7. 能够编写 ABM 测试和监控工具

## 知识详解

### 1. ABM 技术概述

#### 1.1 什么是 ABM

ABM（Adaptive Backlight Management，自适应背光管理）是 AMDGPU 显示驱动中的一项节能技术，通过实时分析显示内容并动态调整面板背光强度，在保持感知亮度（Perceived Brightness）不变的情况下降低背光功耗。

背光功耗在笔记本显示子系统中占据主导地位。一块典型 15.6 英寸笔记本屏幕，背光功耗通常占总显示功耗的 60-80%。ABM 通过智能背光调节，可在不影响用户体验的前提下节省 15-30% 的背光功耗。

```
笔记本显示子系统功耗分布（典型值）:

┌─────────────────────────────────────┐
│                                     │
│  面板背光 (60-80%)  ████████████    │
│                                     │
│  eDP 链路 (5-10%)   ██              │
│                                     │
│  GPU 显示引擎 (10-20%)  ███         │
│                                     │
│  面板驱动 (5-10%)     ██            │
│                                     │
└─────────────────────────────────────┘
        背光是最大的功耗来源
```

#### 1.2 ABM 与 PSR 的区别

ABM 和 PSR 是两种互补的显示节能技术：

```
技术对比:
┌────────────────────────────────────────────────────────────────┐
│ 特性          │ ABM                  │ PSR                     │
├────────────────────────────────────────────────────────────────┤
│ 节能对象      │ 面板背光 (Backlight) │ eDP 链路 (Main Link)    │
│ 工作方式      │ 动态降低背光强度     │ 关闭主链路，面板自刷新   │
│ 静态画面      │ 持续节能 (亮度降低)  │ 持续节能 (链路关闭)      │
│ 动态画面      │ 部分节能             │ 不节能 (链路激活)        │
│ 影响范围      │ 亮度感知             │ 画面完整性              │
│ 硬件依赖      │ ABM 模块 (DCN内)     │ 面板 PSR 支持           │
│ 用户可见度    │ 几乎不可见           │ 高延迟时可见闪烁         │
│ 节能量级      │ 15-30% 背光功耗     │ 50-80% 链路功耗         │
└────────────────────────────────────────────────────────────────┘
```

#### 1.3 ABM 的应用场景

- **笔记本**：延长电池续航时间，ABM 的核心应用场景
- **一体机**：降低整机功耗和散热
- **移动工作站**：在保证亮度感知的同时延长续航
- **嵌入式显示**：降低系统功耗需求

### 2. ABM 工作原理

#### 2.1 核心算法

ABM 的核心思想基于人类视觉系统的亮度感知特性：

1. **亮度适应性**：人眼会自适应环境光照，对环境亮度变化不敏感
2. **对比度保持**：观众更关注画面内容的对比度而非绝对亮度
3. **视觉掩蔽**（Visual Masking）：复杂纹理中的亮度变化更难被察觉

ABM 算法流水线：

```
输入帧 → 像素分析 → 亮度统计 → 增益计算 → PWM调整 → 背光输出
                                           ↓
                                    Content Adaptive Backlight
                                    (基于内容的背光补偿)
```

**ABM 的四个级别**：

```
Level 0: 关闭 ABM，不使用自适应背光
         背光 = 用户设置的背光值
         功耗 = 100%
         适用场景：色彩精确度要求高的专业应用

Level 1: 基础 ABM，轻度节能
         背光 = 用户背光 × 0.95 (最大降低5%)
         功耗 ≈ 95%
         适用场景：日常办公，对亮度变化敏感的用户

Level 2: 标准 ABM，中等节能
         背光 = 用户背光 × (0.85 + 内容系数)
         功耗 ≈ 80-90%
         适用场景：日常使用，平衡节能和体验

Level 3: 积极 ABM，最大节能
         背光 = 用户背光 × (0.70 + 内容系数)
         功耗 ≈ 65-80%
         适用场景：电池供电模式，优先续航
```

#### 2.2 内容自适应分析

ABM 引擎对每一帧画面进行分析，计算多个统计量：

```
┌─────────────────────────────────────────┐
│         ABM 内容分析引擎                  │
│                                         │
│  输入帧 (RGB/YUV)                        │
│         │                               │
│   ┌─────┴─────┐                        │
│   │ 亮度直方图   │                        │
│   └─────┬─────┘                        │
│         │                               │
│   ┌─────┴─────┐                        │
│   │ 平均亮度(APL) │                        │
│   └─────┬─────┘                        │
│         │                               │
│   ┌─────┴─────┐                        │
│   │ 峰值亮度     │                        │
│   └─────┬─────┘                        │
│         │                               │
│   ┌─────┴─────┐                        │
│   │ 对比度      │                        │
│   └─────┬─────┘                        │
│         │                               │
│   ┌─────┴─────┐                        │
│   │ 时空复杂度   │                        │
│   └─────┬─────┘                        │
│         │                               │
│         ▼                               │
│   背光增益系数                             │
└─────────────────────────────────────────┘
```

**关键指标**：

- **APL（Average Picture Level）**：画面平均亮度，决定基础背光调整量
- **Peak Luminance**：峰值亮度，防止高亮区域被过度压缩
- **Contrast Ratio**：对比度，低对比度画面可降低背光更多
- **Spatial-Temporal Complexity**：时空复杂度，快速变化的画面减少调整量以避免闪烁

#### 2.3 ABM 增益曲线

```c
// ABM 增益计算核心逻辑（伪代码）
// 根据 APL 计算背光增益系数

float abm_calculate_gain(int level, float apl)
{
    float gain = 1.0;

    switch (level) {
    case ABM_LEVEL_1:
        // 轻度节能：仅在 APL > 70% 时开始降低
        if (apl > 0.70)
            gain = 1.0 - (apl - 0.70) * 0.3;
        break;

    case ABM_LEVEL_2:
        // 标准节能：APL > 40% 时逐步降低
        if (apl > 0.40)
            gain = 1.0 - (apl - 0.40) * 0.5;
        break;

    case ABM_LEVEL_3:
        // 积极节能：APL > 20% 即开始大幅降低
        if (apl > 0.20)
            gain = 1.0 - (apl - 0.20) * 0.8;
        break;
    }

    // 限制最小增益，避免画面过暗
    const float MIN_GAIN[] = { 1.0, 0.85, 0.70, 0.55 };
    gain = max(gain, MIN_GAIN[level]);

    return gain;
}
```

#### 2.4 亮度补偿

单纯降低背光会导致画面变暗。ABM 通过像素补偿来维持感知亮度：

```
背光降低 + 像素增益补偿 = 感知亮度不变

示例：
  原始: 像素值=128, 背光=100% → 亮度=128
  ABM:  像素值放大至160, 背光=80% → 感知亮度≈128
  （实际效果取决于显示器的 Gamma 特性）
```

像素补偿的关键公式：

```c
// ABM 像素补偿
// 当背光降低时，增加像素值以维持亮度
// 补偿公式基于 Gamma 2.2 标准

uint8_t abm_compensate_pixel(uint8_t pixel, float backlight_ratio)
{
    // 将像素值归一化到 [0, 1]
    float normalized = pixel / 255.0;

    // Gamma 展开（假设显示器 Gamma = 2.2）
    float linear = pow(normalized, 2.2);

    // 补偿：除以背光比例
    float compensated = linear / backlight_ratio;

    // Gamma 压缩回显示空间
    float result = pow(compensated, 1.0 / 2.2);

    // 钳位并转换回 8-bit
    result = min(max(result, 0.0), 1.0);
    return (uint8_t)(result * 255.0);
}
```

### 3. AMDGPU ABM 硬件实现

#### 3.1 DCN ABM 模块架构

在 AMD DCN（Display Core Next）架构中，ABM 是集成在显示流水线中的专用硬件模块：

```
DCN 显示流水线 (ABM 位置):

┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│  DPP    │ → │  MPC    │ → │  OPTC   │ → │  DIO    │
│ (像素处理)│   │ (混合)  │   │ (时序控制)│   │ (输出)  │
└─────────┘   └─────────┘   └─────────┘   └─────────┘
                   │
                   ▼
              ┌─────────┐
              │  ABM    │
              │ (背光管理)│
              └─────────┘
                   │
                   ▼
              ┌─────────┐
              │  PWM    │ → 面板背光
              │ (脉宽调制)│
              └─────────┘
```

ABM 模块内部结构：

```
┌─────────────────────────────────────────┐
│           ABM 硬件模块 (DCN)              │
│                                         │
│  ┌─────────┐  ┌─────────┐              │
│  │ 帧捕获器  │  │ 统计引擎 │              │
│  │(Frame    │  │(Stats   │              │
│  │ Capture) │  │ Engine) │              │
│  └────┬────┘  └────┬────┘              │
│       │            │                    │
│       ▼            ▼                    │
│  ┌─────────────────────────┐           │
│  │    ABM 控制逻辑           │           │
│  │   (Gain Calculator)      │           │
│  └──────────┬──────────────┘           │
│             │                           │
│             ▼                           │
│  ┌─────────────────────────┐           │
│  │   PWM 生成器              │           │
│  │  (Pulse Width Modulator) │           │
│  └──────────┬──────────────┘           │
│             │                           │
└─────────────┼──────────────────────────-┘
              │
              ▼
        eDP BL_PWM (背光)
```

#### 3.2 关键寄存器

```c
// ABM 相关寄存器定义 (DCN 3.0+)
// 来源：drivers/gpu/drm/amd/display/dc/dcn30/dcn30_abm.h

// ABM 控制寄存器
#define ABM_CONTROL                  0x4E00
#define ABM_CONTROL_ENABLE           (1 << 0)
#define ABM_CONTROL_LEVEL_SHIFT      1
#define ABM_CONTROL_LEVEL_MASK       (0x3 << 1)

// ABM 统计寄存器
#define ABM_STATS_APL                0x4E04  // 平均画面亮度
#define ABM_STATS_PEAK               0x4E08  // 峰值亮度
#define ABM_STATS_HISTOGRAM          0x4E0C  // 亮度直方图基址

// ABM PWM 控制
#define ABM_PWM_CONTROL              0x4E20
#define ABM_PWM_PERIOD               (1 << 0)  // PWM 周期
#define ABM_PWM_DUTY_CYCLE           (1 << 4)  // PWM 占空比

// ABM 增益系数
#define ABM_GAIN                     0x4E24   // 当前增益系数
#define ABM_TARGET_GAIN              0x4E28   // 目标增益系数
#define ABM_SLEW_RATE                0x4E2C   // 增益变化速率

// ABM 配置寄存器
#define ABM_CONFIG                   0x4E30
#define ABM_CONFIG_AMBIENT_EN       (1 << 0)  // 环境光传感器启用
#define ABM_CONFIG_CONTENT_EN       (1 << 1)  // 内容自适应启用
#define ABM_CONFIG_HYSTERESIS       (1 << 2)  // 迟滞启用
```

#### 3.3 内核驱动实现

AMDGPU ABM 驱动在 DC 层实现，主要文件结构：

```
drivers/gpu/drm/amd/display/
├── dc/
│   ├── inc/
│   │   └── hw/
│   │       └── abm.h              # ABM 硬件接口定义
│   ├── dcn10/
│   │   └── dcn10_abm.c/h          # DCN 1.0 ABM 实现
│   ├── dcn20/
│   │   └── dcn20_abm.c/h          # DCN 2.0 ABM 实现
│   ├── dcn30/
│   │   └── dcn30_abm.c/h          # DCN 3.0 ABM 实现
│   └── core/
│       └── dc_abm.c               # ABM 核心逻辑
└── amdgpu_dm/
    └── amdgpu_dm_abm.c            # DM 层 ABM 管理
```

```c
// abm.h - ABM 硬件接口
// 来源：drivers/gpu/drm/amd/display/dc/inc/hw/abm.h

struct abm {
    struct hw_abm_funcs *funcs;
    struct dc_context *ctx;
    bool enabled;
    unsigned int current_level;
    unsigned int target_level;
};

struct hw_abm_funcs {
    bool (*abm_init)(struct abm *abm, struct dc_context *ctx);
    bool (*set_abm_level)(struct abm *abm, unsigned int level);
    bool (*set_abm_immediately)(struct abm *abm, unsigned int level);
    bool (*set_pipe)(struct abm *abm, unsigned int pipe_idx);
    unsigned int (*get_current_backlight)(struct abm *abm);
    bool (*set_backlight_level)(struct abm *abm,
                                unsigned int backlight_level,
                                unsigned int frame_ramp,
                                unsigned int controller_id);
};
```

```c
// dcn30_abm.c - DCN3.0 ABM 寄存器编程
// 来源：drivers/gpu/drm/amd/display/dc/dcn30/dcn30_abm.c

static bool dcn30_set_abm_level(struct abm *abm, unsigned int level)
{
    struct dcn30_abm *abm30 = TO_DCN30_ABM(abm);
    unsigned int abm_ctl;

    /* 读取当前 ABM 控制寄存器 */
    abm_ctl = REG_READ(ABM_CONTROL);

    /* 清除旧级别并设置新级别 */
    abm_ctl &= ~ABM_CONTROL_LEVEL_MASK;
    abm_ctl |= (level << ABM_CONTROL_LEVEL_SHIFT);

    REG_WRITE(ABM_CONTROL, abm_ctl);

    /* 写入目标增益 */
    if (level > 0) {
        struct abm_gain_params params;
        params.level = level;
        params.apl = get_current_apl(abm);
        params.target_gain = calculate_gain(params);

        REG_WRITE(ABM_TARGET_GAIN, params.target_gain);
        REG_WRITE(ABM_SLEW_RATE, ABM_SLEW_RATE_DEFAULT);
    }

    return true;
}
```

#### 3.4 DM 层接口

```c
// amdgpu_dm_abm.c - DM 层 ABM 管理
// 简化实现

void amdgpu_dm_update_abm_level(struct amdgpu_display_manager *dm,
                                 unsigned int level)
{
    int i;

    if (level > ABM_LEVEL_MAX)
        level = ABM_LEVEL_MAX;

    for (i = 0; i < dm->dc->sink_count; i++) {
        struct dc_sink *sink = dm->dc->sinks[i];

        if (!sink || sink->sink_signal != SIGNAL_TYPE_EDP)
            continue;

        if (sink->abm) {
            sink->abm->funcs->set_abm_level(sink->abm, level);
            DC_LOG_ABM("ABM level set to %d on sink %d\n", level, i);
        }
    }

    dm->abm_level = level;
}
```

### 4. ABM 级别详解

#### 4.1 各级别的行为差异

```c
// ABM 级别在不同场景下的背光降低比例

struct abm_level_behavior {
    const char *name;
    float dark_room_reduction;   // 暗室场景 (APL=80%)
    float office_reduction;      // 办公场景 (APL=50%)
    float bright_scene_reduction; // 明亮场景 (APL=30%)
    float hdr_reduction;         // HDR 场景
    float transition_time_ms;    // 增益过渡时间
};

struct abm_level_behavior abm_levels[] = {
    {
        .name = "Level 0 (Off)",
        .dark_room_reduction = 0,
        .office_reduction = 0,
        .bright_scene_reduction = 0,
        .hdr_reduction = 0,
        .transition_time_ms = 0,
    },
    {
        .name = "Level 1 (Light)",
        .dark_room_reduction = 5,    // 降低 5%
        .office_reduction = 3,
        .bright_scene_reduction = 0,
        .hdr_reduction = 0,
        .transition_time_ms = 500,   // 慢速过渡
    },
    {
        .name = "Level 2 (Medium)",
        .dark_room_reduction = 15,
        .office_reduction = 10,
        .bright_scene_reduction = 5,
        .hdr_reduction = 5,
        .transition_time_ms = 300,
    },
    {
        .name = "Level 3 (Aggressive)",
        .dark_room_reduction = 30,
        .office_reduction = 20,
        .bright_scene_reduction = 10,
        .hdr_reduction = 10,
        .transition_time_ms = 150,   // 快速过渡
    },
};
```

#### 4.2 级别切换的平滑过渡

ABM 使用 Slew Rate（变化速率）控制来避免背光突变导致的视觉不适：

```
ABM 增益变化曲线:

增益
1.0 │         Level 0 → Level 2
    │          ┌────────────────────
    │          │
0.9 │          │  Slew Rate 控制
    │          │  平滑过渡 ~300ms
0.8 │          │            ┌──── Level 2 稳定
    │          │            │
0.7 │          └────────────┘
    │
    └──────────────────────────────→ 时间
      0  100  200  300  400  500ms

Slew Rate 控制原理：
  每次调整只改变一小步，避免突然的背光变化。
  Δgain = (target_gain - current_gain) / slew_steps
  current_gain += Δgain (每秒重复 N 次)
```

```c
// Slew Rate 控制实现
void abm_update_gain_smooth(struct abm_control *ctrl)
{
    uint32_t current = REG_READ(ABM_GAIN);
    uint32_t target = REG_READ(ABM_TARGET_GAIN);
    uint32_t slew_rate = REG_READ(ABM_SLEW_RATE);

    if (current == target)
        return;

    // 计算每一步的增量
    int32_t step = (target > current) ? 1 : -1;
    uint32_t steps = abs(target - current);

    // 根据 slew_rate 限制单步最大变化
    if (steps > slew_rate)
        steps = slew_rate;

    // 应用新的增益
    uint32_t new_gain;
    if (step > 0)
        new_gain = min(current + steps, target);
    else
        new_gain = max(current - steps, target);

    REG_WRITE(ABM_GAIN, new_gain);
}
```

### 5. ABM 与用户空间接口

#### 5.1 sysfs 接口

```bash
# ABM 通过 backlight sysfs 接口暴露
# 路径：/sys/class/backlight/

# 查找 AMDGPU backlight 设备
ls /sys/class/backlight/
# amdgpu_bl0

# 查看当前背光亮度 (0-255)
cat /sys/class/backlight/amdgpu_bl0/brightness
# 191

# 查看最大亮度
cat /sys/class/backlight/amdgpu_bl0/max_brightness
# 255

# 查看实际亮度（应用了 ABM 调整后）
cat /sys/class/backlight/amdgpu_bl0/actual_brightness
# 172  （比设置值 191 低，说明 ABM 已生效）
```

#### 5.2 debugfs 接口

```bash
# ABM debugfs 接口
# 路径：/sys/kernel/debug/dri/0/

# 查看 ABM 级别
cat /sys/kernel/debug/dri/0/amdgpu_abm_level
# 2

# 设置 ABM 级别（0-3）
echo 1 | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level

# 查看 ABM 状态
cat /sys/kernel/debug/dri/0/amdgpu_abm_stats
# ABM Level: 2
# Current Gain: 0.85
# Target Gain: 0.85
# APL: 0.72
# Peak Luma: 0.95
# Backlight Set: 191
# Backlight Actual: 162
# Slew Rate: 10
# Transition: IDLE

# 强制禁用 ABM
echo 0 | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level

# 查看每个 eDP 显示器的 ABM 能力
cat /sys/kernel/debug/dri/0/amdgpu_abm_caps
# ABM Supported: yes
# ABM Min Level: 0
# ABM Max Level: 3
# ABM Default Level: 2
# Panel Max Backlight: 400 cd/m²
```

#### 5.3 内核模块参数

```bash
# 查看 ABM 相关内核参数
modinfo -p amdgpu | grep abm
# amdgpu_abm:ABM feature enable (0 = disable, 1 = enable) (int)
# amdgpu_abm_level:Default ABM level (0-3) (int)

# 设置默认 ABM 级别
# 在 /etc/modprobe.d/amdgpu.conf 中添加：
options amdgpu amdgpu_abm=1
options amdgpu amdgpu_abm_level=2
```

### 6. ABM 与 PSR 的协同

#### 6.1 协同调度

ABM 和 PSR 同时工作时需要协调调度：

```
时间轴上的协同工作：

          ABM 调整背光
               │
               ▼
    ┌─────────────────────────┐
    │  PSR Active             │
    │  (链路关闭，面板自刷新)   │
    │                         │
    │  ┌──── ABM 增益变化 ────┐│
    │  │   需要更新背光 PWM    ││
    │  └────────┬─────────────┘│
    │           │              │
    │           ▼              │
    │  ┌──────────────────┐   │
    │  │  PSR Exit        │   │
    │  │  短暂退出以更新背光│   │
    │  └────────┬─────────┘   │
    │           │              │
    │           ▼              │
    │  ┌──────────────────┐   │
    │  │  更新背光 PWM     │   │
    │  └────────┬─────────┘   │
    │           │              │
    │           ▼              │
    │  ┌──────────────────┐   │
    │  │  PSR Re-enter    │   │
    │  │  重新进入 PSR     │   │
    │  └──────────────────┘   │
    └─────────────────────────┘
```

#### 6.2 功耗叠加效果

```
不同节能组合的功耗对比：

场景：静态桌面（文本编辑），300nit 设定亮度

┌─────────────────────────────────────────────────┐
│ 节能组合           │ 背光功耗 │ 链路功耗 │ 总功耗  │
├─────────────────────────────────────────────────┤
│ 无节能              │ 3.0W    │ 0.5W    │ 3.5W   │
│ 仅 ABM Level 2     │ 2.4W    │ 0.5W    │ 2.9W   │
│ 仅 PSR             │ 3.0W    │ 0.05W   │ 3.05W  │
│ ABM Level 2 + PSR  │ 2.4W    │ 0.05W   │ 2.45W  │
│ ABM Level 3 + PSR  │ 2.1W    │ 0.05W   │ 2.15W  │
└─────────────────────────────────────────────────┘
          节能效果叠加，最大节能 ~39%
```

### 7. ABM 与不同显示技术的交互

#### 7.1 ABM + HDR

HDR 模式下 ABM 的行为需要特别处理：

```c
// HDR 模式下的 ABM 限制
void abm_handle_hdr_mode(struct abm *abm, bool hdr_enabled)
{
    if (hdr_enabled) {
        // HDR 模式下限制 ABM 增益
        // 避免破坏 HDR 亮度映射
        abm->max_gain_reduction = ABM_HDR_MAX_REDUCTION;
        abm->hysteresis_enabled = true;

        // 对于 HDR，只使用 Level 0 或 Level 1
        if (abm->current_level > ABM_LEVEL_1)
            abm_set_level(abm, ABM_LEVEL_1);
    } else {
        // SDR 模式恢复正常
        abm->max_gain_reduction = ABM_SDR_MAX_REDUCTION;
        abm->hysteresis_enabled = false;
    }
}
```

#### 7.2 ABM + VRR

```c
// VRR 模式下 ABM 更新时机控制
void abm_vrr_transition(struct abm *abm, uint32_t refresh_rate_hz)
{
    // VRR 低刷新率时延长 ABM 响应时间
    // 避免在帧率波动时频繁调整背光
    if (refresh_rate_hz < 60) {
        abm->slew_rate =
            abm->slew_rate * 60 / refresh_rate_hz;
    }
}
```

### 8. ABM 测试与调试

#### 8.1 基本功能测试

```bash
#!/bin/bash
# ABM 基本功能测试

echo "ABM 基本功能测试"
echo "=================="

# 检查 ABM 支持
ABM_CAPS=$(cat /sys/kernel/debug/dri/0/amdgpu_abm_caps 2>/dev/null)
if [ -z "$ABM_CAPS" ]; then
    echo "错误: ABM 不可用"
    exit 1
fi
echo "ABM 能力: $ABM_CAPS"

# 逐级测试 ABM
for level in 0 1 2 3 0; do
    echo ""
    echo "设置 ABM 级别: $level"
    echo $level | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level

    # 等待 ABM 稳定
    sleep 2

    # 读取状态
    echo "ABM 状态:"
    cat /sys/kernel/debug/dri/0/amdgpu_abm_stats 2>/dev/null

    # 读取亮度和实际亮度
    local set_brightness=$(cat /sys/class/backlight/amdgpu_bl0/brightness 2>/dev/null)
    local actual_brightness=$(cat /sys/class/backlight/amdgpu_bl0/actual_brightness 2>/dev/null)
    echo "设置亮度: $set_brightness, 实际亮度: $actual_brightness"

    if [ "$level" -gt 0 ] && [ "$set_brightness" = "$actual_brightness" ]; then
        echo "警告: ABM Level $level 已启用但亮度未调整"
    fi

    sleep 2
done

# 恢复默认
echo 2 | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level
echo ""
echo "ABM 功能测试完成"
```

#### 8.2 ABM 功耗测试

```bash
#!/bin/bash
# ABM 功耗对比测试

# 测试参数
TEST_DURATION=30      # 每个级别的测试时长（秒）
BRIGHTNESS=192        # 设置亮度 (75%)
LOG_FILE="/tmp/abm_power_test_$(date +%Y%m%d_%H%M%S).log"

# 读取 GPU 功耗
get_gpu_power() {
    local hwmon_path="/sys/class/drm/card0/device/hwmon"
    for hwmon in $hwmon_path/hwmon*/power1_average; do
        if [ -f "$hwmon" ]; then
            cat "$hwmon"
            return
        fi
    done
    echo "0"
}

# 读取面板功耗（如果可用）
get_panel_power() {
    local panel_path="/sys/class/backlight/amdgpu_bl0/device/panel_power"
    if [ -f "$panel_path" ]; then
        cat "$panel_path"
    else
        echo "N/A"
    fi
}

echo "ABM 功耗对比测试" | tee "$LOG_FILE"
echo "测试时长: ${TEST_DURATION}s/级别" | tee -a "$LOG_FILE"
echo "设置亮度: ${BRIGHTNESS}" | tee -a "$LOG_FILE"
echo "================================" | tee -a "$LOG_FILE"

# 设置初始亮度
echo "$BRIGHTNESS" | sudo tee /sys/class/backlight/amdgpu_bl0/brightness

# 预热的显示状态
sleep 5

# 测试每个 ABM 级别
for level in 0 1 2 3; do
    echo ""
    echo "测试 ABM Level $level ..." | tee -a "$LOG_FILE"

    # 设置级别
    echo $level | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level > /dev/null
    sleep 3  # 等待稳定

    # 收集功耗数据
    local total_power=0
    local max_power=0
    local min_power=99999999
    local samples=0

    for ((i=0; i<TEST_DURATION; i++)); do
        local power=$(get_gpu_power)
        if [ "$power" != "0" ]; then
            total_power=$((total_power + power))
            [ "$power" -gt "$max_power" ] && max_power=$power
            [ "$power" -lt "$min_power" ] && min_power=$power
            samples=$((samples + 1))
        fi
        sleep 1
    done

    # 计算平均值
    if [ "$samples" -gt 0 ]; then
        local avg_power=$((total_power / samples))
        local panel_power=$(get_panel_power)
        echo "  GPU 平均功耗: ${avg_power}μW" | tee -a "$LOG_FILE"
        echo "  面板功耗: ${panel_power}" | tee -a "$LOG_FILE"

        # 获取实际亮度
        local actual=$(cat /sys/class/backlight/amdgpu_bl0/actual_brightness)
        echo "  实际亮度: ${actual}/255" | tee -a "$LOG_FILE"
    fi
done

# 恢复默认 ABM 级别
echo 2 | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level > /dev/null

echo ""
echo "测试完成" | tee -a "$LOG_FILE"
echo "日志文件: $LOG_FILE"
```

#### 8.3 ABM 响应时间测试

```bash
#!/bin/bash
# ABM 响应时间测试

echo "ABM 响应时间测试"
echo "=================="

# 测量 ABM 级别切换的响应时间
measure_transition() {
    local from=$1
    local to=$2

    # 设置初始级别
    echo $from | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level > /dev/null
    sleep 3

    # 记录切换时间
    local start=$(date +%s%N)
    echo $to | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level > /dev/null

    # 轮询直到增益稳定
    local gain_prev=""
    local stable=false
    for ((i=0; i<100; i++)); do
        local stats=$(cat /sys/kernel/debug/dri/0/amdgpu_abm_stats 2>/dev/null)
        local gain=$(echo "$stats" | grep "Target Gain" | awk '{print $NF}')
        local current_gain=$(echo "$stats" | grep "Current Gain" | awk '{print $NF}')

        if [ "$current_gain" = "$gain" ]; then
            stable=true
            break
        fi
        sleep 0.01
    done

    local end=$(date +%s%N)
    local elapsed_ms=$(( (end - start) / 1000000 ))

    if $stable; then
        echo "  Level ${from}→${to}: ${elapsed_ms}ms (稳定)"
    else
        echo "  Level ${from}→${to}: ${elapsed_ms}ms (未完全稳定)"
    fi
}

echo "级别切换时间:"
measure_transition 0 1
measure_transition 1 2
measure_transition 2 3
measure_transition 3 2
measure_transition 2 1
measure_transition 1 0
```

### 9. ABM 调优指南

#### 9.1 不同场景的推荐配置

```bash
#!/bin/bash
# abm_profile.sh - ABM 场景切换脚本

set_abm_profile() {
    local profile=$1

    case $profile in
        "max-battery")
            # 电池续航优先：Level 3 + 降低基础亮度
            echo 3 | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level
            echo 128 | sudo tee /sys/class/backlight/amdgpu_bl0/brightness
            echo "已切换到最大续航模式"
            ;;

        "balanced")
            # 平衡模式：Level 2，中等亮度
            echo 2 | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level
            echo 192 | sudo tee /sys/class/backlight/amdgpu_bl0/brightness
            echo "已切换到平衡模式"
            ;;

        "performance")
            # 性能模式：Level 1，高亮度
            echo 1 | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level
            echo 255 | sudo tee /sys/class/backlight/amdgpu_bl0/brightness
            echo "已切换到性能模式"
            ;;

        "color-critical")
            # 色彩关键：关闭 ABM + 标准亮度
            echo 0 | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level
            echo 192 | sudo tee /sys/class/backlight/amdgpu_bl0/brightness
            echo "已切换到色彩精确模式"
            ;;

        "auto")
            # 自动模式：根据电源状态切换
            local power_status=$(cat /sys/class/power_supply/*/status 2>/dev/null | head -1)
            if [ "$power_status" = "Discharging" ]; then
                set_abm_profile "max-battery"
            else
                set_abm_profile "balanced"
            fi
            ;;
    esac
}

# 使用电源事件触发自动切换
# 可以放在 /etc/acpi/events/ 或 udev 规则中
case "$1" in
    ac-connect)
        set_abm_profile "balanced"
        ;;
    ac-disconnect)
        set_abm_profile "max-battery"
        ;;
    *)
        echo "用法: $0 {ac-connect|ac-disconnect|max-battery|balanced|performance|color-critical}"
        ;;
esac
```

#### 9.2 udev 规则自动切换

```bash
# /etc/udev/rules.d/99-abm-power.rules
# 电源插拔时自动切换 ABM 配置

# 接通电源
SUBSYSTEM=="power_supply", ATTR{online}=="1", \
    RUN+="/usr/local/bin/abm_profile.sh ac-connect"

# 断开电源
SUBSYSTEM=="power_supply", ATTR{online}=="0", \
    RUN+="/usr/local/bin/abm_profile.sh ac-disconnect"
```

#### 9.3 ABM SysFS 自动化监控

```python
#!/usr/bin/env python3
"""
abm_monitor.py - ABM 状态监控和日志工具
持续监控 ABM 行为并记录变化
"""

import os
import time
import logging
from datetime import datetime

ABM_DEBUGFS = "/sys/kernel/debug/dri/0"
BACKLIGHT_SYSFS = "/sys/class/backlight/amdgpu_bl0"

class ABMMonitor:
    def __init__(self, log_file="/var/log/abm_monitor.log"):
        logging.basicConfig(
            filename=log_file,
            level=logging.INFO,
            format='%(asctime)s %(message)s'
        )
        self.previous_stats = {}

    def read_sysfs(self, path):
        try:
            with open(path, 'r') as f:
                return f.read().strip()
        except (IOError, FileNotFoundError):
            return "N/A"

    def read_abm_stats(self):
        stats_file = os.path.join(ABM_DEBUGFS, "amdgpu_abm_stats")
        return self.read_sysfs(stats_file)

    def monitor_loop(self, interval=5):
        print("ABM Monitor started (Ctrl+C to stop)")
        print(f"{'Time':<20} {'Level':<8} {'Gain':<8} {'APL':<8} "
              f"{'Brightness':<12} {'Actual':<12}")

        while True:
            level = self.read_sysfs(os.path.join(
                ABM_DEBUGFS, "amdgpu_abm_level"))
            stats = self.read_abm_stats()

            bri_set = self.read_sysfs(os.path.join(
                BACKLIGHT_SYSFS, "brightness"))
            bri_act = self.read_sysfs(os.path.join(
                BACKLIGHT_SYSFS, "actual_brightness"))

            # 解析统计信息提取关键字段
            current_gain = "?"
            apl = "?"
            for line in stats.split('\n'):
                if "Current Gain:" in line:
                    current_gain = line.split(':')[1].strip()
                elif "APL:" in line:
                    apl = line.split(':')[1].strip()

            now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            print(f"{now:<20} {level:<8} {current_gain:<8} "
                  f"{apl:<8} {bri_set:<12} {bri_act:<12}")

            # 记录变化
            if self.previous_stats:
                if level != self.previous_stats.get('level'):
                    logging.info(f"ABM level changed: "
                                 f"{self.previous_stats['level']} -> {level}")
                if bri_act != self.previous_stats.get('actual'):
                    logging.info(f"Actual brightness changed: "
                                 f"{self.previous_stats['actual']} -> {bri_act}")

            self.previous_stats = {
                'level': level,
                'actual': bri_act,
                'gain': current_gain,
                'apl': apl
            }

            time.sleep(interval)

if __name__ == "__main__":
    monitor = ABMMonitor()
    try:
        monitor.monitor_loop()
    except KeyboardInterrupt:
        print("\nMonitor stopped")
```

### 10. ABM 常见问题

#### 问题 1：ABM 导致亮度闪烁

**现象**：启用 ABM 后，屏幕亮度出现周期性波动，特别是在内容切换时。

**原因**：
- ABM 的 Slew Rate 设置过快，增益变化太急
- 内容分析引擎对快速变化的场景响应过于敏感
- 面板的 PWM 调光频率与 ABM 更新频率冲突

**解决方案**：
```bash
# 降低 ABM 响应速度
# 增大迟滞（通过 debugfs）
echo 500 | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_slew_delay

# 使用较温和的 ABM 级别
echo 1 | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level
```

#### 问题 2：ABM 与色彩管理冲突

**现象**：色彩管理的应用程序（如图片编辑软件）中颜色显示不准确。

**原因**：
- ABM 的像素补偿改变了 Gamma 曲线
- 亮度补偿与 ICC 色彩配置文件的交互异常

**解决方案**：
```bash
# 对色彩敏感的应用，临时关闭 ABM
echo 0 | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level

# 或者使用自定义 udev 规则
# 在应用启动时自动关闭 ABM
```

#### 问题 3：ABM 在 HDR 视频播放时过度调暗

**现象**：播放 HDR 视频时，高亮场景被 ABM 过度降低。

**原因**：
- ABM 将 HDR 的高 APL 误判为需要降低背光
- HDR 模式的亮度补偿未正确应用

**解决方案**：
```bash
# HDR 播放时自动降低 ABM
# 可通过 DM 层的 HDR 检测自动处理
echo 1 | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level
```

## 实践操作

### 实践 1：ABM 功能验证工具

```c
/*
 * abm_verify.c - ABM 功能验证程序
 * 编译: gcc -o abm_verify abm_verify.c -Wall
 * 运行: sudo ./abm_verify
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>

#define ABM_LEVEL_PATH  "/sys/kernel/debug/dri/0/amdgpu_abm_level"
#define ABM_STATS_PATH  "/sys/kernel/debug/dri/0/amdgpu_abm_stats"
#define BRIGHTNESS_PATH "/sys/class/backlight/amdgpu_bl0/brightness"
#define ACTUAL_BRIGHT_PATH "/sys/class/backlight/amdgpu_bl0/actual_brightness"

int read_int(const char *path, int *value) {
    char buf[64];
    int fd = open(path, O_RDONLY);
    if (fd < 0) return -1;
    int n = read(fd, buf, sizeof(buf) - 1);
    close(fd);
    if (n <= 0) return -1;
    buf[n] = '\0';
    *value = atoi(buf);
    return 0;
}

int write_int(const char *path, int value) {
    char buf[16];
    int fd = open(path, O_WRONLY);
    if (fd < 0) return -1;
    int n = snprintf(buf, sizeof(buf), "%d", value);
    write(fd, buf, n);
    close(fd);
    return 0;
}

int main(int argc, char *argv[]) {
    int level, brightness, actual;

    printf("ABM 功能验证工具\n");
    printf("====================\n\n");

    /* 检查 ABM 接口 */
    if (access(ABM_LEVEL_PATH, F_OK) < 0) {
        printf("错误: ABM debugfs 接口不可用\n");
        printf("请确认 amdgpu 驱动已加载且 debugfs 已挂载\n");
        return 1;
    }

    /* 测试每个 ABM 级别 */
    for (int l = 0; l <= 3; l++) {
        printf("测试 ABM Level %d ...\n", l);

        if (write_int(ABM_LEVEL_PATH, l) < 0) {
            printf("  设置失败: %s\n", strerror(errno));
            continue;
        }

        /* 等待 ABM 稳定 */
        sleep(3);

        read_int(ABM_LEVEL_PATH, &level);
        read_int(BRIGHTNESS_PATH, &brightness);
        read_int(ACTUAL_BRIGHT_PATH, &actual);

        printf("  级别: %d (确认)\n", level);
        printf("  设置亮度: %d\n", brightness);
        printf("  实际亮度: %d\n", actual);

        if (l > 0 && brightness == actual) {
            printf("  警告: 亮度未调整，ABM 可能未生效\n");
        }

        float diff_percent = 0;
        if (brightness > 0) {
            diff_percent = (float)(brightness - actual) / brightness * 100;
        }
        printf("  背光降低: %.1f%%\n\n", diff_percent);
    }

    /* 恢复默认 */
    write_int(ABM_LEVEL_PATH, 2);
    printf("ABM 验证完成 (已恢复 Level 2)\n");

    return 0;
}
```

### 实践 2：ABM 内容自适应模拟

```c
/*
 * abm_content_sim.c - ABM 内容分析模拟器
 * 演示 ABM 如何根据画面内容调整背光
 * 编译: gcc -o abm_content_sim abm_content_sim.c -lm -Wall
 * 运行: ./abm_content_sim
 */

#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include <string.h>

#define FRAME_SIZE 1920 * 1080
#define HISTOGRAM_BINS 256

/* 模拟帧数据 */
typedef struct {
    unsigned char *pixels;
    int width;
    int height;
} frame_t;

/* APL 计算 */
float calculate_apl(const unsigned char *pixels, int count) {
    unsigned long sum = 0;
    for (int i = 0; i < count; i++) {
        sum += pixels[i];
    }
    return (float)sum / (count * 255);
}

/* 亮度直方图 */
void calculate_histogram(const unsigned char *pixels, int count,
                         unsigned int *histogram) {
    memset(histogram, 0, HISTOGRAM_BINS * sizeof(unsigned int));
    for (int i = 0; i < count; i++) {
        histogram[pixels[i]]++;
    }
}

/* 对比度计算 (RMS contrast) */
float calculate_contrast(const unsigned char *pixels, int count) {
    float mean = 0;
    for (int i = 0; i < count; i++) {
        mean += pixels[i];
    }
    mean /= count;

    float variance = 0;
    for (int i = 0; i < count; i++) {
        float diff = pixels[i] - mean;
        variance += diff * diff;
    }
    variance /= count;

    return sqrt(variance);
}

/* 峰值亮度 (第 99 百分位) */
float calculate_peak_luma(const unsigned char *pixels, int count) {
    unsigned int histogram[HISTOGRAM_BINS] = {0};
    calculate_histogram(pixels, count, histogram);

    int cumulative = 0;
    int threshold = count * 0.99;

    for (int i = HISTOGRAM_BINS - 1; i >= 0; i--) {
        cumulative += histogram[i];
        if (cumulative >= threshold) {
            return (float)i / 255.0;
        }
    }
    return 1.0;
}

/* ABM 增益计算 */
float abm_gain(int level, float apl, float contrast, float peak) {
    float base_gain = 1.0;

    switch (level) {
    case 0:
        return 1.0;
    case 1:
        base_gain = 1.0 - fmax(0, apl - 0.7) * 0.5;
        break;
    case 2:
        base_gain = 1.0 - fmax(0, apl - 0.4) * 0.7;
        break;
    case 3:
        base_gain = 1.0 - fmax(0, apl - 0.2) * 1.0;
        break;
    }

    /* 对比度修正：低对比度时允许更多降低 */
    float contrast_factor = 1.0 - (1.0 - fmin(contrast / 50, 1.0)) * 0.2;
    base_gain *= contrast_factor;

    /* 峰值修正：高峰值时限制降低 */
    float peak_factor = 1.0 + fmax(0, peak - 0.8) * 0.2;
    base_gain = fmin(base_gain * peak_factor, 1.0);

    /* 限制最小值 */
    float min_gain[] = {1.0, 0.85, 0.70, 0.55};
    return fmax(base_gain, min_gain[level]);
}

/* 模拟不同场景 */
void simulate_scene(const char *name, float apl, float contrast,
                    float peak) {
    printf("\n场景: %s\n", name);
    printf("  APL: %.2f, 对比度: %.2f, 峰值: %.2f\n",
           apl, contrast, peak);

    for (int level = 0; level <= 3; level++) {
        float gain = abm_gain(level, apl, contrast, peak);
        float backlight_reduction = (1.0 - gain) * 100;
        printf("  Level %d: 增益=%.3f, 背光降低=%.1f%%\n",
               level, gain, backlight_reduction);
    }
}

int main(void) {
    printf("ABM 内容自适应模拟器\n");
    printf("=======================\n");
    printf("演示不同画面内容对 ABM 行为的影响\n");

    /* 模拟不同场景 */
    simulate_scene("白色文档 (Word/PDF)",
                   0.85,  /* 高 APL（白底黑字） */
                   40.0,  /* 中高对比度 */
                   0.98); /* 高峰值 */

    simulate_scene("深色 IDE (代码编辑器)",
                   0.25,  /* 低 APL（暗色主题） */
                   35.0,  /* 中对比度 */
                   0.50); /* 低峰值 */

    simulate_scene("自然风光照片",
                   0.50,  /* 中等 APL */
                   55.0,  /* 高对比度 */
                   0.90); /* 高峰值 */

    simulate_scene("视频播放 (暗场景)",
                   0.35,  /* 低 APL */
                   45.0,  /* 中高对比度 */
                   0.70); /* 中峰值 */

    simulate_scene("视频播放 (明亮场景)",
                   0.70,  /* 高 APL */
                   50.0,  /* 高对比度 */
                   0.95); /* 高峰值 */

    simulate_scene("HDR 游戏场景",
                   0.45,  /* 中等 APL */
                   70.0,  /* 极高对比度 (HDR) */
                   0.99); /* 极高峰值 */

    printf("\n结论:\n");
    printf("  - 白色内容(Word)在 Level 3 可降低最多 %.0f%% 背光\n",
           (1.0 - abm_gain(3, 0.85, 40.0, 0.98)) * 100);
    printf("  - 暗色内容(IDE)在 Level 3 仅降低约 %.0f%%\n",
           (1.0 - abm_gain(3, 0.25, 35.0, 0.50)) * 100);
    printf("  - HDR 场景受峰值亮度保护，降低幅度有限\n");

    return 0;
}
```

### 实践 3：ABM 与 PSR 协同效果测试

```bash
#!/bin/bash
# abm_psr_coop_test.sh
# 测试 ABM 和 PSR 协同工作时的功耗和稳定性

DEBUGFS_BASE="/sys/kernel/debug/dri/0"
LOG_FILE="/tmp/abm_psr_coop_$(date +%Y%m%d_%H%M%S).log"

log() {
    echo "[$(date +%H:%M:%S)] $*" | tee -a "$LOG_FILE"
}

get_gpu_power() {
    for hwmon in /sys/class/drm/card0/device/hwmon/hwmon*/power1_average; do
        [ -f "$hwmon" ] && cat "$hwmon" && return
    done
    echo "0"
}

measure_combination() {
    local abm_level=$1
    local psr_enable=$2
    local duration=$3

    # 设置 ABM 级别
    echo $abm_level | tee "$DEBUGFS_BASE/amdgpu_abm_level" > /dev/null

    # 设置 PSR 状态
    if [ "$psr_enable" -eq 1 ]; then
        echo 1 | tee "$DEBUGFS_BASE/amdgpu_dm_psr_enable" > /dev/null
    else
        echo 0 | tee "$DEBUGFS_BASE/amdgpu_dm_psr_enable" > /dev/null
    fi

    # 等待状态稳定
    sleep 5

    local abm_label="ABM_$abm_level"
    local psr_label=$([ "$psr_enable" -eq 1 ] && echo "PSR_ON" || echo "PSR_OFF")
    log "测试: $abm_label + $psr_label (${duration}s)"

    local total_power=0
    local samples=0
    local psr_entries=0
    local psr_exits=0

    for ((i=0; i<duration; i++)); do
        local power=$(get_gpu_power)
        [ "$power" != "0" ] && {
            total_power=$((total_power + power))
            samples=$((samples + 1))
        }

        # 记录 PSR 统计
        local psr_stats=$(cat "$DEBUGFS_BASE/amdgpu_dm_psr_stats" 2>/dev/null)
        local entry_count=$(echo "$psr_stats" | grep "entry_count" | awk '{print $NF}')
        local exit_count=$(echo "$psr_stats" | grep "exit_count" | awk '{print $NF}')
        [ -n "$entry_count" ] && psr_entries=$entry_count
        [ -n "$exit_count" ] && psr_exits=$exit_count

        sleep 1
    done

    local avg_power=0
    [ "$samples" -gt 0 ] && avg_power=$((total_power / samples))

    log "  结果: 平均功率=${avg_power}μW, PSR进入=${psr_entries}, PSR退出=${psr_exits}"
}

log "ABM + PSR 协同效果测试"
log "========================"

# 设置固定亮度 50%
echo 128 | tee /sys/class/backlight/amdgpu_bl0/brightness > /dev/null

# 测试所有组合
for abm in 0 1 2 3; do
    for psr in 0 1; do
        measure_combination $abm $psr 10
        echo "" | tee -a "$LOG_FILE"
    done
done

# 恢复默认设置
echo 2 | tee "$DEBUGFS_BASE/amdgpu_abm_level" > /dev/null
echo 1 | tee "$DEBUGFS_BASE/amdgpu_dm_psr_enable" > /dev/null
echo 192 | tee /sys/class/backlight/amdgpu_bl0/brightness > /dev/null

log ""
log "测试完成"
log "结果已保存到: $LOG_FILE"
```

## 案例分析

### 案例一：笔记本电池续航优化

**背景**：某款 AMD Ryzen 7 6800U 笔记本在视频播放时续航仅 5 小时，低于竞品（~7 小时）。

**调查**：
```bash
# 检查当前节能配置
cat /sys/kernel/debug/dri/0/amdgpu_abm_level
# 0 (ABM 未启用)

cat /sys/kernel/debug/dri/0/amdgpu_dm_psr_state
# INACTIVE (PSR 未进入)

# 背光设置
cat /sys/class/backlight/amdgpu_bl0/brightness
# 255 (100% 亮度)

# 测量功耗
cat /sys/class/drm/card0/device/hwmon/hwmon0/power1_average
# 8500000 (8.5W)
```

**优化**：
```bash
# 1. 启用 ABM Level 2
echo 2 | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level

# 2. 降低基础亮度到 70%
echo 179 | sudo tee /sys/class/backlight/amdgpu_bl0/brightness

# 3. 确认 PSR 正常工作
echo 1 | sudo tee /sys/kernel/debug/dri/0/amdgpu_dm_psr_enable
cat /sys/kernel/debug/dri/0/amdgpu_dm_psr_state
# ACTIVE

# 4. 配置电源管理脚本
```

**结果**：显示功耗从 8.5W 降至 4.2W（降低 50%），视频播放续航从 5 小时提升至 8.5 小时。

### 案例二：ABM 导致 Adobe Photoshop 色彩偏差

**背景**：设计师反馈在 Adobe Photoshop 中编辑图片时，导出后的图片偏暗。

**根因**：
1. Photoshop 使用 ICC 色彩管理
2. ABM Level 2 的像素补偿改变了实际的 Gamma 曲线
3. 色彩管理软件未感知到 ABM 的亮度调整
4. 用户在 Photoshop 中看到的亮度比实际像素值亮

**解决方案**：
```bash
# 方案 1：Photoshop 启动时自动关闭 ABM
cat > /usr/local/bin/ps_abm_fix.sh << 'EOF'
#!/bin/bash
# 保存原始 ABM 级别
ORIG_LEVEL=$(cat /sys/kernel/debug/dri/0/amdgpu_abm_level)

# 关闭 ABM
echo 0 | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level

# 启动 Photoshop
/opt/Adobe/Photoshop/photoshop "$@"

# 退出后恢复 ABM
echo $ORIG_LEVEL | sudo tee /sys/kernel/debug/dri/0/amdgpu_abm_level
EOF
```

### 案例三：多显示器 ABM 同步问题

**背景**：笔记本连接外接显示器时，内外屏亮度感知不一致。

**现象**：外接显示器无 ABM，内屏启用 ABM 后，拖动窗口到外屏时亮度突变。

**解决方案**：
```bash
# 外接显示器连接时降低 ABM 级别
cat > /etc/udev/rules.d/99-abm-external.rules << 'EOF'
# HDMI/DP 连接时降低 ABM
SUBSYSTEM=="drm", ACTION=="change", \
    RUN+="/usr/local/bin/abm_external_monitor.sh"
EOF

cat > /usr/local/bin/abm_external_monitor.sh << 'EOF'
#!/bin/bash
# 检测外接显示器
EXTERNAL_COUNT=$(xrandr | grep " connected " | grep -v eDP | wc -l)

if [ "$EXTERNAL_COUNT" -gt 0 ]; then
    echo 1 | tee /sys/kernel/debug/dri/0/amdgpu_abm_level
else
    echo 2 | tee /sys/kernel/debug/dri/0/amdgpu_abm_level
fi
EOF
```

## 相关链接

1. [AMDGPU DC ABM 核心实现](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/core/dc_abm.c)
   - dc_abm.c：ABM 核心逻辑，包括增益计算和状态管理

2. [AMDGPU DM ABM 管理](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_abm.c)
   - amdgpu_dm_abm.c：DM 层 ABM 接口和 debugfs 实现

3. [DCN 3.0 ABM 硬件实现](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/dcn30/dcn30_abm.c)
   - dcn30_abm.c：DCN3.0 ABM 硬件寄存器编程

4. [ABM 硬件接口定义](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/inc/hw/abm.h)
   - abm.h：struct abm 和 hw_abm_funcs 定义

5. [AMDGPU backlight sysfs 接口](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_backlight.c)
   - amdgpu_dm_backlight.c：背光 sysfs 和 backlight 类设备

6. [DCN ABM 寄存器定义](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/dcn30/dcn30_abm.h)
   - dcn30_abm.h：ABM 寄存器宏定义和寄存器映射

7. [内核文档 - 显示电源管理](https://www.kernel.org/doc/html/latest/gpu/amdgpu/display/display-power-saving.html)
   - AMDGPU 显示节能特性文档

8. [VESA DisplayHDR 标准](https://vesa.org/vesa-displayhdr/)
   - HDR 亮度规范和 ABM 的 HDR 交互

9. [Linux backlight 子系统](https://www.kernel.org/doc/html/latest/admin-guide/backlight/backlight.html)
   - Linux 背光子系统架构和 sysfs 接口

10. [edp-panel-self-refresh (Allion)](https://www.allion.com/edp-panel-self-refresh/)
    - PSR 和 ABM 的综合显示节能测试方法论

11. [ACPI 电源管理事件](https://wiki.archlinux.org/title/ACPI_hotkeys)
    - ACPI 电源事件处理，用于 ABM 自动切换

12. [udev 规则编写指南](https://wiki.archlinux.org/title/Udev)
    - udev 规则用于电源状态触发的 ABM 切换

## 今日小结

### 核心概念回顾

ABM（Adaptive Backlight Management）是 AMDGPU 驱动的自适应背光管理技术，通过实时分析显示内容动态调整背光强度，在不影响视觉体验的前提下节省 15-30% 的背光功耗。

**ABM 的四大级别：**
- **Level 0**：关闭 ABM，背光不调整
- **Level 1**：轻度节能（~5% 降低），适合亮度敏感用户
- **Level 2**：标准节能（~15% 降低），日常推荐
- **Level 3**：积极节能（~30% 降低），电池续航优先

**关键机制：**
- 内容分析引擎计算 APL、峰值亮度、对比度等指标
- 增益曲线根据 APL 和目标级别计算背光降低量
- 像素补偿通过 Gamma 校正维持感知亮度
- Slew Rate 控制增益变化速率，避免背光突变

### 关键技术要点

1. ABM 通过 sysfs (`brightness/actual_brightness`) 和 debugfs (`amdgpu_abm_level/stats`) 暴露接口
2. ABM 与 PSR 可以协同工作，节能效果叠加
3. HDR 模式下 ABM 自动降级以保护 HDR 亮度映射
4. Slew Rate 控制是避免闪烁的关键参数
5. ABM 的节能效果与画面内容强相关（白色内容节能最多）

### 常见误区

- **误区 1**：ABM 在所有内容上节能效果相同。事实：白色内容（Word）节能最多，暗色内容（IDE）节能有限。
- **误区 2**：ABM 会明显降低显示质量。事实：Level 1-2 几乎不可察觉，Level 3 在大多数场景下也表现良好。
- **误区 3**：ABM 和 PSR 只能二选一。事实：两者协同工作，分别解决背光和链路两个不同的功耗来源。
- **误区 4**：`actual_brightness` 降低说明硬件有问题。事实：这正是 ABM 正常工作的表现。

## 扩展思考

### ABM 技术的未来方向

1. **机器学习的 ABM 策略**
   - 基于深度学习的场景识别，预测用户对亮度变化的需求
   - 个性化 ABM 模型，学习用户在不同应用中的亮度偏好
   - 环境光传感器 + 内容分析的双重自适应

2. **全局背光 vs 局部调光**
   - ABM 基于全局背光调整，Mini-LED 的局部调光可以实现更精细的节能
   - 未来 ABM 可能需要与局部调光算法协同
   - 每个背光分区的独立 ABM 分析

3. **跨显示设备的 ABM 同步**
   - 多显示器场景下的亮度一致性管理
   - 笔记本 + 外接显示器的统一感知亮度
   - 基于环境光传感器的全局亮度协调

4. **用户空间 ABM 策略**
   - 将 ABM 策略部分移到用户空间，允许更灵活的配置
   - Wayland 协议扩展让合成器参与 ABM 决策
   - Flatpak/Snap 等容器化应用的 ABM 策略集成
   - 基于机器学习的内容分类与自适应 ABM 级别选择

5. **ABM 与新型显示技术融合**
   - OLED 面板的 ABM 特殊性：像素老化补偿与亮度管理
   - Micro-LED 的自适应背光挑战
   - 可变刷新率（VRR）下 ABM 算法的帧率自适应调整
   - HDR 内容与 ABM 的智能切换策略

6. **ABM 标准化与生态系统**
   - 跨厂商 ABM 行为标准化（Intel、NVIDIA 对比分析）
   - Windows 与 Linux ABM 行为差异与兼容层设计
   - 能源之星/EPEAT 等节能认证对 ABM 的要求
   - 开源 ABM 测试套件（基于 IGT 的扩展）

---

### 学习建议

1. **入门路径**
   - 先通过 sysfs 和 debugfs 接口熟悉 ABM 行为
   - 阅读 `drivers/gpu/drm/amd/display/modules/abm/` 目录下的代码
   - 使用实践工具中的测试程序验证不同 ABM 级别的效果

2. **进阶方向**
   - 深入研究 DCN ABM 硬件模块的寄存器级行为
   - 分析 ABM 与其他显示特性的交互（PSR、HDR、VRR）
   - 尝试修改 ABM 算法并评估效果

3. **研究热点**
   - 基于 AI 的内容自适应背光管理
   - ABM 与游戏体验的平衡优化
   - 多屏环境下的统一亮度感知管理

---

## 附录 A：ABM 寄存器参考

### DCN3.0 ABM 关键寄存器列表

| 寄存器名 | 偏移地址 | 位域 | 描述 |
|---------|---------|------|------|
| `LB_ABM_LEVEL` | 0x6000 | [2:0] | ABM 工作级别 (0-3) |
| `LB_ABM_SLEW_RATE` | 0x6004 | [15:0] | 增益变化速率控制 |
| `LB_ABM_GAIN_CURVE` | 0x6008-0x6018 | [31:0] | 增益曲线系数表 (8 段) |
| `LB_ABM_APL_WIN` | 0x601C | [9:0] | APL 统计窗口大小 |
| `LB_ABM_PEAK_WIN` | 0x6020 | [9:0] | 峰值亮度统计窗口 |
| `LB_ABM_PIXEL_COMP` | 0x6024 | [31:0] | 像素补偿 Gamma 系数 |
| `LB_ABM_HDR_CTRL` | 0x6028 | [1:0] | HDR 模式控制 (00=关闭, 01=降级) |
| `LB_ABM_STATUS` | 0x602C | [7:0] | ABM 当前状态 |
| `LB_ABM_STATS_APL` | 0x6030 | [15:0] | 当前帧 APL 统计值 |
| `LB_ABM_STATS_PEAK` | 0x6034 | [15:0] | 当前帧峰值亮度 |
| `LB_ABM_STATS_CONTRAST` | 0x6038 | [15:0] | 当前帧对比度 |

### 寄存器编程示例

以下代码展示如何通过 MMIO 直接读取 ABM 统计寄存器：

```c
#include <stdint.h>
#include <stdio.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <unistd.h>

#define MMIO_BASE 0x6000
#define PAGE_SIZE 4096

/* 通过 PCIe BAR 映射 ABM MMIO 空间 */
uint32_t *map_abm_mmio(int pci_fd, off_t bar_base) {
    off_t offset = (bar_base + MMIO_BASE) & ~(PAGE_SIZE - 1);
    size_t map_size = 0x1000; /* 4KB 覆盖 ABM 寄存器 */

    return (uint32_t *)mmap(NULL, map_size,
                            PROT_READ | PROT_WRITE,
                            MAP_SHARED, pci_fd, offset);
}

/* 读取 ABM APL 统计值 */
uint16_t read_abm_apl(uint32_t *abm_base) {
    return (uint16_t)(abm_base[0x6030 / 4] & 0xFFFF);
}

/* 读取 ABM 峰值亮度 */
uint16_t read_abm_peak(uint32_t *abm_base) {
    return (uint16_t)(abm_base[0x6034 / 4] & 0xFFFF);
}

/* 设置 ABM 级别（直接寄存器访问） */
void set_abm_level_direct(uint32_t *abm_base, int level) {
    uint32_t val = abm_base[0x6000 / 4];
    val &= ~0x7;
    val |= (level & 0x7);
    abm_base[0x6000 / 4] = val;
}

int main(int argc, char *argv[]) {
    printf("ABM MMIO 寄存器读取示例\n");
    printf("========================\n\n");

    /* 提示：实际运行时需要 root 权限和 PCIe BAR 地址 */
    printf("此示例展示 ABM 寄存器的直接访问模式\n");
    printf("实际使用时需要通过 sysfs 获取 BAR 基地址：\n");
    printf("  $ cat /sys/bus/pci/devices/0000:03:00.0/resource\n\n");

    printf("ABM 寄存器编程要点：\n");
    printf("  1. 所有寄存器访问需要 4 字节对齐\n");
    printf("  2. 修改寄存器前应读取-修改-写入\n");
    printf("  3. Slew Rate 寄存器改变后需等待至少 1 帧\n");
    printf("  4. HDR 模式切换时需要重新配置增益曲线\n");
    printf("  5. 多显示器场景下每个 pipe 有独立的 ABM 寄存器\n");

    return 0;
}
```

## 附录 B：ABM 增益曲线算法详解

### 增益曲线数学模型

ABM 的增益曲线基于分段线性函数，将输入 APL 映射到背光增益系数：

```
对于 APL 值 a (0.0 ~ 1.0)，增益 G(a) 计算如下：

G(a) = G_base(a) * level_factor(level) * content_factor(content_type)

其中：
  - G_base(a) 是基础增益曲线
  - level_factor(level) 是级别缩放因子 [1.0, 0.95, 0.85, 0.70]
  - content_factor(content_type) 是内容类型调整因子
```

### 8 段增益曲线参数

```
段编号 | APL 范围    | 基础增益 G_base | Level 3 增益
--------|-------------|-----------------|-------------
  0     | 0.00-0.05   |     1.00        |    1.00
  1     | 0.05-0.10   |     1.00        |    0.98
  2     | 0.10-0.20   |     0.98        |    0.92
  3     | 0.20-0.35   |     0.95        |    0.85
  4     | 0.35-0.50   |     0.92        |    0.78
  5     | 0.50-0.65   |     0.88        |    0.72
  6     | 0.65-0.80   |     0.85        |    0.68
  7     | 0.80-1.00   |     0.82        |    0.65
```

### 像素补偿 Gamma 查找表

像素补偿通过对每个像素应用 Gamma 校正来维持感知亮度：

```
对于原始像素值 p (0-255)，补偿后值 p' 为：

p' = 255 * (p/255)^(1/γ)

其中 γ 动态调整：
  γ = 2.2 + (1.0 - G(a)) * 0.5
  
示例：
  - G(a) = 1.0 (无调整):    γ = 2.2, 无补偿
  - G(a) = 0.85 (Level 2):  γ = 2.275, 轻度补偿
  - G(a) = 0.70 (Level 3):  γ = 2.35, 明显补偿
```

### 性能影响数据

| 测试场景 | 面板功耗 (Level 0) | 面板功耗 (Level 2) | 面板功耗 (Level 3) |
|---------|-------------------|-------------------|-------------------|
| 白色背景（Word） | 5.2W | 4.1W (-21%) | 3.2W (-38%) |
| 混合内容（Web） | 3.8W | 3.2W (-16%) | 2.7W (-29%) |
| 视频播放（SDR） | 4.5W | 3.8W (-16%) | 3.1W (-31%) |
| 暗色背景（IDE） | 1.8W | 1.7W (-6%) | 1.5W (-17%) |
| HDR 游戏 | 8.5W | 7.8W (-8%) | 不推荐 |

### ABM 实现代码片段分析

以下分析 AMDGPU DC 中 ABM 增益计算的核心逻辑：

```c
// drivers/gpu/drm/amd/display/modules/abm/abm.c
// 简化的增益计算流程

/*
 * ABM 增益计算步骤：
 * 1. 获取当前帧的 APL 统计值（硬件自动计算）
 * 2. 根据 APL 查表获取基础增益
 * 3. 应用 Level 缩放因子
 * 4. 检查 HDR 标志，必要时降级增益
 * 5. 应用 Slew Rate 限制，输出最终增益
 */

static int get_abm_gain(int apl_percent, int level, bool hdr_active)
{
    /* 增益曲线表：8 段 APL -> 增益映射 */
    static const int gain_table[8][2] = {
        {  5, 100 },  /* APL 0-5%   -> 增益 100% */
        { 10, 100 },  /* APL 5-10%  -> 增益 100% */
        { 20,  98 },  /* APL 10-20% -> 增益 98% */
        { 35,  95 },  /* APL 20-35% -> 增益 95% */
        { 50,  92 },  /* APL 35-50% -> 增益 92% */
        { 65,  88 },  /* APL 50-65% -> 增益 88% */
        { 80,  85 },  /* APL 65-80% -> 增益 85% */
        {100,  82},   /* APL 80-100% -> 增益 82% */
    };

    static const int level_factor[4] = {
        0,    /* Level 0: 不调整 */
        -5,   /* Level 1: -5% */
        -15,  /* Level 2: -15% */
        -30,  /* Level 3: -30% */
    };

    int base_gain = 100;
    int adjusted_gain;

    /* 1. 根据 APL 查找基础增益 */
    for (int i = 0; i < 8; i++) {
        if (apl_percent <= gain_table[i][0]) {
            base_gain = gain_table[i][1];
            break;
        }
    }

    /* 2. 应用 Level 调整 */
    adjusted_gain = base_gain + level_factor[level];
    if (adjusted_gain > 100)
        adjusted_gain = 100;
    if (adjusted_gain < 40)
        adjusted_gain = 40;

    /* 3. HDR 模式下降级 */
    if (hdr_active && level > 1) {
        /* HDR 时最多允许 Level 1 的节能效果 */
        adjusted_gain = base_gain + level_factor[1];
    }

    /* 4. Slew Rate 限制 */
    static int prev_gain = 100;
    int max_change = 2; /* 每帧最多变化 2% */
    if (abs(adjusted_gain - prev_gain) > max_change) {
        if (adjusted_gain > prev_gain)
            adjusted_gain = prev_gain + max_change;
        else
            adjusted_gain = prev_gain - max_change;
    }
    prev_gain = adjusted_gain;

    return adjusted_gain;
}
```

---

## 附录 C：ABM 调试快速参考

### sysfs/debugfs 路径速查

| 接口 | 路径 | 权限 | 用途 |
|------|------|------|------|
| 背光亮度 | `/sys/class/backlight/amdgpu_bl*/brightness` | RW | 读取/设置背光 |
| 实际亮度 | `/sys/class/backlight/amdgpu_bl*/actual_brightness` | RO | 读取 ABM 调整后亮度 |
| 最大亮度 | `/sys/class/backlight/amdgpu_bl*/max_brightness` | RO | 读取最大亮度值 |
| ABM 级别 | `/sys/kernel/debug/dri/0/amdgpu_abm_level` | RW | 读取/设置 ABM 级别 |
| ABM 统计 | `/sys/kernel/debug/dri/0/amdgpu_abm_stats` | RO | 读取 ABM 详细统计 |
| ABM 能力 | `/sys/kernel/debug/dri/0/amdgpu_abm_caps` | RO | 读取 ABM 硬件能力 |
| PM 状态 | `/sys/class/drm/card0/device/power_state` | RO | 读取电源状态 |

### 快速故障诊断命令

```bash
#!/bin/bash
# abm_quick_diag.sh - ABM 快速诊断脚本

echo "=== ABM 快速诊断 ==="
echo

# 1. 检查 ABM 硬件能力
echo "--- ABM 硬件能力 ---"
cat /sys/kernel/debug/dri/0/amdgpu_abm_caps 2>/dev/null || echo "  [ABM 不可用]"

# 2. 检查当前 ABM 级别
echo -e "\n--- 当前 ABM 级别 ---"
cat /sys/kernel/debug/dri/0/amdgpu_abm_level 2>/dev/null || echo "  [不可用]"

# 3. 检查背光设置
echo -e "\n--- 背光设置 ---"
for bl in /sys/class/backlight/amdgpu_bl*; do
    if [ -d "$bl" ]; then
        echo "  设备: $(basename $bl)"
        echo "  当前: $(cat $bl/brightness 2>/dev/null)"
        echo "  实际: $(cat $bl/actual_brightness 2>/dev/null)"
        echo "  最大: $(cat $bl/max_brightness 2>/dev/null)"
        # 计算 ABM 降低百分比
        cur=$(cat $bl/brightness 2>/dev/null)
        act=$(cat $bl/actual_brightness 2>/dev/null)
        if [ -n "$cur" ] && [ -n "$act" ] && [ "$cur" -gt 0 ]; then
            pct=$(( (cur - act) * 100 / cur ))
            echo "  降低: ${pct}% (ABM 生效)"
        fi
    fi
done

# 4. 检查 PSR 状态
echo -e "\n--- PSR 状态 ---"
cat /sys/kernel/debug/dri/0/amdgpu_dm_psr_state 2>/dev/null || echo "  [不可用]"

# 5. 检查电源状态
echo -e "\n--- 电源状态 ---"
cat /sys/class/drm/card0/device/power_dpm_state 2>/dev/null || echo "  [不可用]"

# 6. ABM 统计信息
echo -e "\n--- ABM 统计 ---"
cat /sys/kernel/debug/dri/0/amdgpu_abm_stats 2>/dev/null || echo "  [不可用]"

echo -e "\n=== 诊断完成 ==="
```

---

*本文档为 AMDGPU 驱动 ABM 自适应背光管理技术的全面参考，结合理论分析、实践操作和调试方法，帮助读者深入理解 ABM 技术原理和工程应用。*