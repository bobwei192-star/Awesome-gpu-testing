# Day 353: Panel Self Refresh（PSR）节电技术

## 学习目标

1. 深入理解 Panel Self Refresh（PSR）技术在嵌入式 DisplayPort（eDP）和 DisplayPort 上的工作原理
2. 掌握 PSR 的两种主要规范——PSR1（标准 PSR）和 PSR2（选择性更新）的区别与实现细节
3. 熟悉 DRM 子系统中的 PSR 基础设施，包括 drm_dp_psr 核心和 AMDGPU DC 的 PSR 实现
4. 理解 PSR 的进入/退出条件、时序约束及其对用户体验的影响
5. 学会使用 debugfs 和 bpftrace 等工具诊断 PSR 问题
6. 掌握 PSR 与 Panel Replay（面板回放）的关系，以及 AMDGPU 中的实现差异
7. 能够编写 Shell 和 C 程序来测试、监控和调试 PSR 行为

## 知识详解

### 1. PSR 技术概述

#### 1.1 什么是 Panel Self Refresh

Panel Self Refresh（PSR）是 VESA（视频电子标准协会）在嵌入式 DisplayPort（eDP）标准中定义的一项节能技术。当显示器显示静态画面时，PSR 允许 GPU 进入省电状态（如关闭显示核心的时钟或进入低功耗模式），而显示器面板内部的缓冲器（Panel Buffer）继续刷新显示内容。

```
传统模式（无 PSR）：
   ┌──────────────────┐          ┌──────────────────┐
   │     GPU/DCN      │  eDP     │   显示面板        │
   │  每帧渲染 →      │─────────→│  实时刷新显示      │
   │  持续传输帧数据   │  链路    │  无内部缓存        │
   └──────────────────┘          └──────────────────┘
   功耗：~1-3W（仅显示）         功耗：~3-5W（面板）

PSR 模式（节能）：
   静态画面时：
   ┌──────────────────┐          ┌──────────────────┐
   │     GPU/DCN      │  eDP     │   显示面板        │
   │  进入省电模式 ←   │─────────→│  使用内部缓冲刷新  │
   │  主链路关闭       │  休眠    │  Self-Refresh     │
   └──────────────────┘          └──────────────────┘
   功耗：~0.5W（省电模式）       功耗：~3-5W（面板）
```

#### 1.2 PSR 的节能效果

PSR 的节能效果取决于使用场景：

| 使用场景 | 无 PSR 功耗（GPU+显示） | 有 PSR 功耗 | 节省比例 |
|---------|----------------------|------------|---------|
| 静态桌面 | 4-8W | 1.5-3W | 50-70% |
| 阅读/浏览 | 4-8W | 1.5-3W | 50-70% |
| 视频播放 | 6-12W | 4-10W | 20-30% |
| 动态游戏 | 10-30W | 9-29W | ~5% |

注意：PSR 的主要收益场景是静态画面和轻度交互场景，在持续动态内容（如 3D 游戏）中节能效果有限。

### 2. PSR 工作原理详解

#### 2.1 PSR1（标准 PSR）

PSR1 是 VESA eDP 标准中定义的第一代 PSR 技术。其基本工作流程如下：

```
PSR1 状态机：
                    ┌─────────────┐
                    │  PSR 关闭    │
                    │  (Active)    │
                    └──────┬──────┘
                           │ 进入条件满足
                           ▼
                    ┌─────────────┐
                    │  PSR 进入    │
                    │  (入口)      │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  PSR 活跃    │ ◄──── 帧缓冲区自刷新
                    │  (Self-      │
                    │   Refresh)   │
                    └──────┬──────┘
                           │ 画面变化检测
                           ▼
                    ┌─────────────┐
                    │  PSR 退出    │
                    │  (出口)      │
                    └──────┬──────┘
                           │ 退出完成
                           ▼
                    ┌─────────────┐
                    │  PSR 关闭    │
                    │  (Active)    │  ← 返回正常刷新
                    └─────────────┘
```

PSR1 的关键特征：

1. **全帧更新**：当画面变化时，PSR1 要求传输完整的帧数据到面板缓冲区
2. **主链路关闭**：PSR 激活期间，eDP 主链路（Main Link）进入休眠模式，仅辅助通道（AUX）保持活跃
3. **面板缓冲区**：面板内部集成的显存（通常为 1-2 帧容量）存储最后一帧的像素数据
4. **退出延迟**：从 PSR 退出到显示新内容需要约 1-5ms

```
PSR1 时序图：

正常刷新时（每帧传输）：
  Frame N    Frame N+1  Frame N+2  Frame N+3
  ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐
  │VSync │   │VSync │   │VSync │   │VSync │
  └──┬───┘   └──┬───┘   └──┬───┘   └──┬───┘
     │          │          │          │
  ┌──▼───┐   ┌──▼───┐   ┌──▼───┐   ┌──▼───┐
  │XMIT  │   │XMIT  │   │XMIT  │   │XMIT  │
  │Frame │   │Frame │   │Frame │   │Frame │
  │  N   │   │ N+1  │   │ N+2  │   │ N+3  │
  └──────┘   └──────┘   └──────┘   └──────┘
  主链路：   活跃       活跃       活跃       活跃
  面板：     刷新       刷新       刷新       刷新

进入 PSR（静态画面出现后）：
  Frame N    Frame N    Frame N    Frame N
  ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐
  │VSync │   │VSync │   │VSync │   │VSync │
  └──┬───┘   └──┬───┘   └──┬───┘   └──┬───┘
     │          │          │          │
  ┌──▼───┐   ┌──────┐   ┌──────┐   ┌──────┐
  │XMIT  │   │SLEEP │   │SLEEP │   │SLEEP │
  │Last  │   │      │   │      │   │      │
  │Frame │   │      │   │      │   │      │
  └──────┘   └──────┘   └──────┘   └──────┘
  主链路：   活跃       休眠       休眠       休眠
  面板：     缓存帧     自刷新     自刷新     自刷新
```

#### 2.2 PSR2（选择性更新）

PSR2 是 eDP 1.4a 中引入的增强型 PSR，允许只传输画面中变化的部分（选择性更新），而不是完整的帧。这大幅降低了退出 PSR 时的带宽需求和功耗。

```
PSR1 退出 → 传输全帧：
  ┌──────────────────────────────────────┐
  │                                      │
  │  全帧传输（1920x1080）              │
  │  ██████████████████████████████████  │  ← 整帧传输
  │  ██████████████████████████████████  │     带宽：~3.2 Gbps
  │  ██████████████████████████████████  │     （1080p@60Hz）
  │  ██████████████████████████████████  │
  │  ██████████████████████████████████  │
  └──────────────────────────────────────┘

PSR2 选择性更新：
  ┌──────────────────────────────────────┐
  │                                      │
  │  ┌──────────────────┐               │
  │  │  变化区域 (SU)    │               │  ← 只传输变化区域
  │  │  ████████████████ │               │     带宽：~10-100 Mbps
  │  └──────────────────┘               │     （取决于变化大小）
  │                                      │
  └──────────────────────────────────────┘
```

PSR2 的关键特征：

1. **选择性更新（SU, Selective Update）**：使用 DP 辅助通道传输变化区域
2. **更低的退出延迟**：~0.5-2ms（vs PSR1 的 1-5ms）
3. **更细粒度的控制**：驱动可以指定脏区域（dirty rectangle）来部分更新
4. **主链路保持活跃**：进入 PSR2 后，主链路降速但不完全关闭

```
PSR2 选择性更新时序：

  Frame N    Frame N    Frame N+SU  Frame N
  ┌──────┐   ┌──────┐   ┌────────┐   ┌──────┐
  │VSync │   │VSync │   │ VSync  │   │VSync │
  └──┬───┘   └──┬───┘   └───┬────┘   └──┬───┘
     │          │           │           │
  ┌──▼───┐   ┌──────┐   ┌──▼─────┐   ┌──────┐
  │XMIT  │   │SLEEP │   │SU XMIT │   │SLEEP │
  │Frame │   │      │   │Region  │   │      │
  │  N   │   │      │   │  A     │   │      │
  └──────┘   └──────┘   └────────┘   └──────┘
  主链路：  活跃      低速      活跃(部分)  低速
  面板：    缓存帧    自刷新    更新区域A  自刷新

  用户感知：静态       静态      鼠标移动    静态
```

#### 2.3 PSR1 vs PSR2 对比

| 特性 | PSR1 | PSR2 |
|------|------|------|
| eDP 标准版本 | eDP 1.3+ | eDP 1.4a+ |
| 更新方式 | 全帧更新 | 选择性区域更新 |
| 主链路状态 | 完全关闭 | 降速 |
| 退出延迟 | 1-5ms | 0.5-2ms |
| 功耗节省（静态） | 高（~60-70%） | 中高（~50-65%） |
| 功耗节省（轻度交互） | 中（~30-40%） | 高（~40-60%） |
| 对合成器要求 | 简单（全帧检测） | 复杂（区域跟踪） |
| 面板硬件要求 | 较低 | 较高（支持 SU） |
| 兼容性 | 广泛支持 | 较新面板支持 |

### 3. DRM PSR 基础设施

#### 3.1 DRM DP PSR 核心

Linux 内核 DRM 子系统提供了通用的 DP PSR 支持，位于 `drivers/gpu/drm/display/drm_dp_helper.c` 和 `drm_dp_psr.c` 中。

核心数据结构：

```c
/* drivers/gpu/drm/i915/display/intel_psr.h - 参考实现 */
struct drm_dp_psr {
    struct drm_dp_aux *aux;           /* DP 辅助通道 */
    struct mutex lock;                /* PSR 状态锁 */
    enum psr_state state;             /* 当前 PSR 状态 */
    bool psr2_enabled;                /* 是否启用 PSR2 */
    bool panel_replay;                /* 是否支持 Panel Replay */
    ktime_t last_entry_time;          /* 上次进入 PSR 的时间 */
    ktime_t last_exit_time;           /* 上次退出 PSR 的时间 */
    u32 entry_count;                  /* 进入 PSR 的总次数 */
    u32 exit_count;                   /* 退出 PSR 的总次数 */
    u32 busy_count;                   /* 因忙碌无法进入 PSR 的次数 */
    u32 irq_count;                    /* PSR 相关 IRQ 计数 */

    /* PSR2 选择性更新相关 */
    struct drm_rect su_region;        /* 当前选择性更新区域 */
    bool su_valid;                    /* 选择性更新区域是否有效 */
    u32 su_blocks;                    /* 选择性更新的块数 */

    /* 调试统计 */
    u32 frame_count_gpu;              /* GPU 渲染帧数 */
    u32 frame_count_psr;              /* PSR 自刷新帧数 */
};
```

PSR 状态枚举：

```c
enum psr_state {
    PSR_STATE_NONE,            /* PSR 未启用 */
    PSR_STATE_INACTIVE,        /* PSR 未激活 */
    PSR_STATE_ENTERING,        /* 正在进入 PSR */
    PSR_STATE_ACTIVE,          /* PSR 活跃（自刷新中） */
    PSR_STATE_EXITING,         /* 正在退出 PSR */
    PSR_STATE_ERROR,           /* PSR 错误 */
    PSR_STATE_DISABLED,        /* PSR 已禁用 */
};
```

#### 3.2 DPCD 寄存器与 PSR 能力检测

PSR 能力通过读取 DPCD（DisplayPort Configuration Data）寄存器来检测。关键寄存器包括：

```
DPCD 地址 0x070 (DP_PSR_SUPPORT):
  Bit 0: PSR1 支持
  Bit 1: PSR2 支持
  Bit 2: Panel Replay 支持（eDP 1.4b+）

DPCD 地址 0x071 (DP_PSR_CAPS):
  Bit 0: 帧缓冲区容量（0=1帧, 1=2帧）
  Bit 1: 选择性更新能力
  Bit 2: Y 坐标精度（SU 使用）
  Bit 3-4: 链路训练要求
  Bit 5: 帧 CRC 校验支持

DPCD 地址 0x080 (DP_PSR_ENABLE):
  写入启用/禁用 PSR
  Bit 0: PSR1 启用
  Bit 1: PSR2 启用
  Bit 2: CRC 校验启用

DPCD 地址 0x081-0x082 (DP_PSR_STATUS):
  Bit 0-2: PSR 状态
    0x0 = 未激活
    0x1 = 过渡到活跃
    0x2 = 活跃（自刷新）
    0x3 = 过渡到非活跃
    0x4 = 非活跃（帧传输中）
  Bit 3: 错误状态
  Bit 4: 链路 CRC 错误
```

在 AMDGPU 驱动中，PSR 能力检测发生在 DC 初始化阶段：

```c
/* drivers/gpu/drm/amd/display/modules/power/power_helpers.c */
bool mod_power_calc_psr_config(struct dc_link *link,
                                struct psr_config *psr_config)
{
    struct dpcd_caps *dpcd_caps = &link->dpcd_caps;
    union dp_psr_support psr_support;

    /* 读取 DPCD 0x070 获取 PSR 支持 */
    psr_support.raw = dpcd_caps->psr_info.psr_version;

    if (!psr_support.bits.psr1_supported &&
        !psr_support.bits.psr2_supported) {
        /* 面板不支持 PSR */
        return false;
    }

    /* 配置 PSR 参数 */
    psr_config->psr_level = psr_support.bits.psr2_supported ?
                              PSR_LEVEL_2 : PSR_LEVEL_1;
    psr_config->frame_buffer_capacity =
        dpcd_caps->psr_info.psr_caps.bits.frame_buffer_capacity;
    psr_config->selective_update =
        dpcd_caps->psr_info.psr_caps.bits.selective_update;

    return true;
}
```

#### 3.3 AMDGPU DC PSR 架构

AMDGPU 的 PSR 实现主要在 DC（Display Core）层和 DM（Display Manager）层。DC 层负责与面板的 PSR 硬件交互，DM 层负责与 DRM 核心集成。

```
AMDGPU PSR 架构：

  ┌─────────────────────────────────────────────┐
  │              用户空间（桌面环境）              │
  │  - Weston / GNOME / KDE                     │
  │  - 只管理显示内容，不直接控制 PSR            │
  └──────────────────┬──────────────────────────┘
                     │ DRM/KMS API
  ┌──────────────────▼──────────────────────────┐
  │          AMDGPU DM（Display Manager）         │
  │                                              │
  │  - amdgpu_dm_psr.c         │                 │
  │  - 管理 PSR 状态                           │
  │  - 响应 atomic commit 变化                  │
  │  - 控制 PSR 进入/退出                      │
  └──────────────────┬──────────────────────────┘
                     │ DC 接口
  ┌──────────────────▼──────────────────────────┐
  │          DC（Display Core）                  │
  │                                              │
  │  - dcn20/dcn30 中的 PSR 支持                │
  │  - mod_power_helpers.c（PSR 策略）          │
  │  - 寄存器级 PSR 编程                        │
  └──────────────────┬──────────────────────────┘
                     │ DCN 寄存器/MMIO
  ┌──────────────────▼──────────────────────────┐
  │          DCN 硬件                            │
  │                                              │
  │  - PSR 控制寄存器（DCPG_PSR_*）             │
  │  - AUX 通道控制器                           │
  │  - 面板缓冲接口                              │
  └─────────────────────────────────────────────┘
```

### 4. AMDGPU PSR 实现细节

#### 4.1 PSR 启用和禁用

PSR 的启用和禁用通过 DC 接口完成：

```c
/* drivers/gpu/drm/amd/display/dc/dc_dp_types.h */
struct dc_psr_config {
    bool psr_enabled;              /* PSR 是否启用 */
    bool psr2_enabled;             /* PSR2 是否启用 */
    bool allow_smu_optimizations;  /* 允许 SMU 优化 */
    unsigned int psr_dispatch_delay; /* PSR 进入延迟（帧数） */
    unsigned int psr_exit_multiplier; /* PSR 退出时间乘数 */
};

struct dc_psr_context {
    unsigned int vsync_rate_hz;    /* 垂直同步频率 */
    unsigned int sink_caps;        /* 接收端能力 */
    unsigned int su_blocks;        /* 选择性更新块数 */
    bool su_valid;                 /* 选择性更新是否有效 */
};
```

DC PSR 接口函数：

```c
/* drivers/gpu/drm/amd/display/dc/inc/hw/dc.h */

/* 设置 PSR 配置 */
bool dc_set_psr_config(struct dc_link *link,
                        struct dc_psr_config *config);

/* 启用 PSR */
bool dc_set_psr_enable(struct dc_link *link, bool enable);

/* 获取 PSR 状态 */
bool dc_get_psr_state(struct dc_link *link,
                       enum dc_psr_state *state);

/* 获取 PSR 统计信息 */
void dc_get_psr_stats(struct dc_link *link,
                       struct dc_psr_stats *stats);
```

#### 4.2 PSR 进入条件

PSR 进入需要满足多个条件，AMDGPU DC 层在每次 atomic commit 后检查这些条件：

```c
/* 伪代码：PSR 进入条件检查 */
bool should_enter_psr(struct dc_link *link,
                       struct dc_state *context)
{
    /* 1. PSR 已启用 */
    if (!link->psr_config.psr_enabled)
        return false;

    /* 2. 显示器处于静态画面 */
    if (is_display_content_changing(context))
        return false;

    /* 3. 没有正在进行的热插拔事件 */
    if (link->hpd_status != HPD_READY)
        return false;

    /* 4. 没有 pending 的 modeset */
    if (context->streams[0]->mode_changed)
        return false;

    /* 5. 没有 cursor 移动（如果 cursor 触发退出） */
    if (link->psr_config.psr_level == PSR_LEVEL_1 &&
        has_cursor_movement(context))
        return false;

    /* 6. 没有音频流 */
    if (link->audio_stream_active)
        return false;

    /* 7. 亮度没有变化（ABM 未激活） */
    if (is_abm_active(link))
        return false;

    /* 8. 满足空闲帧数要求 */
    if (link->psr_idle_frames < MIN_PSR_IDLE_FRAMES)
        return false;

    return true;
}
```

#### 4.3 PSR 退出条件

以下事件会导致 PSR 退出：

```
PSR 退出事件触发优先级：

  优先级 1（立即退出）：
    ├── 鼠标/指针移动
    ├── 键盘输入
    ├── 触摸事件
    ├── 新帧提交（atomic commit）
    └── 光标属性变化

  优先级 2（延迟退出）：
    ├── 亮度变化请求
    ├── 颜色变化
    └── 音频流启动

  优先级 3（非显示退出）：
    ├── 热插拔事件
    ├── DPMS 变化
    ├── suspend/resume
    └── 调试接口访问
```

#### 4.4 PSR 寄存器编程

在 DCN 硬件中，PSR 控制寄存器位于 DCPG（Display Clock and Power Gating）模块：

```c
/* 伪代码：DCN PSR 寄存器编程 */

/* PSR 控制寄存器 */
#define DCPG_PSR_CONTROL               0x6000
#define DCPG_PSR_STATUS                0x6004
#define DCPG_PSR_FRAME_COUNT           0x6008
#define DCPG_PSR_ERROR_STATUS          0x600C
#define DCPG_PSR_CRC_VAL               0x6010

/* DCPG_PSR_CONTROL 位域 */
#define PSR_ENABLE                     (1 << 0)
#define PSR2_ENABLE                    (1 << 1)
#define PSR_ACTIVE                     (1 << 2)
#define PSR_FORCE_ENTER                (1 << 3)
#define PSR_FORCE_EXIT                 (1 << 4)
#define PSR_HPD_IRQ_TRIGGER            (1 << 5)
#define PSR_CRC_ENABLE                 (1 << 6)
#define PSR_SU_ENABLE                  (1 << 7)

/* PSR 状态位 */
#define PSR_STATE_MASK                 0x7
#define PSR_STATE_SHIFT                0
/* PSR_STATE 值：
 *   0 = OFF
 *   1 = SLEEP
 *   2 = ACTIVE (Self-Refresh)
 *   3 = WAKEUP
 *   4 = ERROR
 */

static void dcn20_set_psr(struct dc_link *link, bool enable)
{
    uint32_t psr_control = 0;
    struct dcn20_dccg *dccg = TO_DCN20_DCCG(link->dc->res_pool->dccg);

    if (enable) {
        /* 配置 PSR 控制寄存器 */
        psr_control |= PSR_ENABLE;

        if (link->psr_config.psr2_enabled)
            psr_control |= PSR2_ENABLE;

        /* 设置进入延迟 */
        psr_control |= (link->psr_config.psr_dispatch_delay << 8);

        /* 启用 CRC 校验（如果支持） */
        if (link->dpcd_caps.psr_info.psr_caps.bits.crc_verification)
            psr_control |= PSR_CRC_ENABLE;

        REG_SET(DCPG_PSR_CONTROL, 0, psr_control);
    } else {
        /* 禁用 PSR */
        REG_UPDATE(DCPG_PSR_CONTROL, PSR_ENABLE, 0);
    }
}
```

### 5. Panel Replay 技术

#### 5.1 Panel Replay 概述

Panel Replay（面板回放）是 eDP 1.4b+ 标准中引入的新技术，可以看作是 PSR 的增强版本。与 PSR 不同，Panel Replay 不需要面板内部缓冲区（Panel Buffer），而是利用 eDP 链路的 ALPM（Alternate Low Power Mode）机制实现节能。

```
Panel Replay 工作原理：

  正常模式：
  ┌──────────────────┐   全速链路    ┌──────────────────┐
  │     GPU/DCN      │──────────────→│   显示面板        │
  │  每帧传输        │   HBR2/HBR3   │  实时显示          │
  └──────────────────┘              └──────────────────┘

  Panel Replay 模式：
  ┌──────────────────┐   降速链路    ┌──────────────────┐
  │     GPU/DCN      │──────────────→│   显示面板        │
  │  进入低功耗       │   ALPM       │  使用帧重复        │
  │  DCN 时钟门控     │   ~100Mbps   │  驱动自行刷新      │
  └──────────────────┘              └──────────────────┘
```

#### 5.2 PSR vs Panel Replay 对比

| 特性 | PSR | Panel Replay |
|------|-----|---------------|
| 需要面板缓冲区 | 是 | 否 |
| 主链路状态 | 完全关闭 | 降速（ALPM） |
| eDP 标准 | eDP 1.3+ | eDP 1.4b+ |
| 退出延迟 | 1-5ms | <1ms（更快） |
| 面板兼容性 | 需要专用硬件 | 更多面板支持 |
| AMDGPU 支持版本 | DCN 2.0+ | DCN 3.5+ |

### 6. PSR 对用户体验的影响

#### 6.1 PSR 引起的闪烁问题

PSR 最常见的用户可见问题是闪烁（Flicker）。当 PSR 进入/退出频繁交替时，可能导致以下表现：

```
PSR 闪烁问题分类：

  1. 进入闪烁（Entry Flicker）：
     ┌──────┐  ┌──────┐  ┌──────┐
     │正常  │  │进入  │  │PSR   │
     │显示  │  │PSR   │  │稳定  │
     └──────┘  └──────┘  └──────┘
      帧 N      帧 N+1    帧 N+2
                ↑
          可能出现的闪烁点

  2. 退出闪烁（Exit Flicker）：
     ┌──────┐  ┌──────┐  ┌──────┐
     │PSR   │  │退出  │  │正常  │
     │稳定  │  │PSR   │  │显示  │
     └──────┘  └──────┘  └──────┘
      帧 N      帧 N+1    帧 N+2
                ↑
          可能出现的闪烁点

  3. 频繁切换闪烁（Rapid Transition Flicker）：
     ┌──┬──┬──┬──┬──┬──┬──┬──┐
     │PSR│出│PSR│出│PSR│出│PSR│出│
     │   │  │   │  │   │  │   │  │
     └──┴──┴──┴──┴──┴──┴──┴──┘
     ↑  ↑  ↑  ↑  ↑  ↑  ↑  ↑  ← 可见闪烁
     持续交替进出 PSR
```

#### 6.2 PSR 延迟问题

PSR 退出延迟可能导致用户输入的响应延迟增加：

```c
/* PSR 延迟计算 */
struct psr_latency {
    unsigned int exit_latency_us;    /* PSR 退出延迟（微秒） */
    unsigned int entry_latency_us;   /* PSR 进入延迟（微秒） */
    unsigned int su_latency_us;      /* 选择性更新延迟 */
};

/* 典型延迟值 */
static const struct psr_latency psr1_latency = {
    .exit_latency_us  = 3000,   /* 3ms */
    .entry_latency_us = 1000,   /* 1ms */
    .su_latency_us    = 0,      /* PSR1 不支持 SU */
};

static const struct psr_latency psr2_latency = {
    .exit_latency_us  = 1000,   /* 1ms */
    .entry_latency_us = 500,    /* 0.5ms */
    .su_latency_us    = 200,    /* 0.2ms - 只更新部分区域 */
};

/* 对用户体验的影响：
 * - 60fps 下，每帧时间为 16.67ms
 * - PSR1 退出延迟 3ms 占帧时间的 18%
 * - PSR2 退出延迟 1ms 占帧时间的 6%
 * - PSR2 SU 仅 0.2ms 额外延迟，几乎不可感知
 */
```

### 7. PSR 测试与调试

#### 7.1 debugfs 接口

AMDGPU 提供了丰富的 debugfs 接口用于 PSR 调试：

```bash
# PSR 状态查询
cat /sys/kernel/debug/dri/0/amdgpu_dm_psr_state

# 示例输出：
# PSR State: ACTIVE
# PSR Entry Count: 1234
# PSR Exit Count: 567
# PSU Selective Update: enabled
# Last Entry: 2026-05-09 10:30:45.123456789
# Last Exit: 2026-05-09 10:30:48.987654321

# PSR 统计信息
cat /sys/kernel/debug/dri/0/amdgpu_dm_psr_stats

# 示例输出：
# Total Frames Rendered: 123456
# Total Frames PSR: 78901
# PSR Efficiency: 63.9%
# Average PSR Entry Latency: 1.2ms
# Average PSR Exit Latency: 3.1ms
# CRC Errors: 0
# Link CRC Errors: 2
```

#### 7.2 动态 PSR 控制

通过 debugfs 可以手动控制 PSR：

```bash
# 强制禁用 PSR
echo 0 > /sys/kernel/debug/dri/0/amdgpu_dm_psr_enable

# 强制启用 PSR
echo 1 > /sys/kernel/debug/dri/0/amdgpu_dm_psr_enable

# 设置 PSR 级别（0=禁用，1=PSR1，2=PSR2）
echo 2 > /sys/kernel/debug/dri/0/amdgpu_dm_psr_level

# 强制进入 PSR（调试用）
echo 1 > /sys/kernel/debug/dri/0/amdgpu_dm_psr_force_entry

# 强制退出 PSR
echo 1 > /sys/kernel/debug/dri/0/amdgpu_dm_psr_force_exit
```

#### 7.3 DRM DP PSR 调试信息

```bash
# 查看 DPCD PSR 能力
cat /sys/kernel/debug/dri/0/DP-1/psr_sink_support

# 查看 PSR 调试计数器
cat /sys/kernel/debug/dri/0/amdgpu_dm_dp_psr_debug

# 启用 PSR 详细日志
echo 'file amdgpu_dm_psr.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file dcn20_dccg.c +p' > /sys/kernel/debug/dynamic_debug/control
```

### 8. IGT PSR 测试

IGT 测试套件包含专门的 PSR 测试，位于 `tests/kms_psr.c` 和 `tests/kms_psr2_su.c`。

```bash
# 运行所有 PSR 测试
./kms_psr --run-subtest all

# 测试 PSR1 基本功能
./kms_psr --run-subtest psr_basic

# 测试 PSR 退出/重新进入
./kms_psr --run-subtest psr_exit

# 测试 PSR 与光标交互
./kms_psr --run-subtest psr_cursor

# 测试 PSR2 选择性更新
./kms_psr2_su --run-subtest psr2_su_basic
./kms_psr2_su --run-subtest psr2_su_frontbuffer

# 测试 PSR 与 DPMS
./kms_psr --run-subtest psr_dpms

# 测试长时间 PSR 稳定性
./kms_psr --run-subtest psr_stability --repeat 100
```

IGT PSR 测试的核心逻辑：

```c
/* 伪代码：IGT PSR 测试逻辑 */

/* 测试 PSR 基本进入/退出 */
static void test_psr_basic(int fd, uint32_t crtc_id, uint32_t plane_id)
{
    igt_assert(psr_wait_entry(fd, PSR_TIMEOUT_MS));

    /* 触发屏幕变化 */
    drmModeAtomicReq *req = drmModeAtomicAlloc();
    add_plane_properties(req, plane_id, ...);
    drmModeAtomicCommit(fd, req, DRM_MODE_ATOMIC_NONBLOCK, NULL);

    /* 验证 PSR 退出 */
    igt_assert(psr_wait_exit(fd, PSR_TIMEOUT_MS));

    /* 等待 PSR 重新进入 */
    igt_assert(psr_wait_entry(fd, PSR_TIMEOUT_MS));
}

/* 等待 PSR 进入 */
static bool psr_wait_entry(int fd, int timeout_ms)
{
    struct timespec tv = { .tv_sec = timeout_ms / 1000,
                           .tv_nsec = (timeout_ms % 1000) * 1000000 };
    int debug_fd = igt_debugfs_open(fd, "amdgpu_dm_psr_state", O_RDONLY);

    return igt_debugfs_monitor(debug_fd, "ACTIVE", &tv);
}

/* 等待 PSR 退出 */
static bool psr_wait_exit(int fd, int timeout_ms)
{
    struct timespec tv = { .tv_sec = timeout_ms / 1000,
                           .tv_nsec = (timeout_ms % 1000) * 1000000 };
    int debug_fd = igt_debugfs_open(fd, "amdgpu_dm_psr_state", O_RDONLY);

    return igt_debugfs_monitor(debug_fd, "INACTIVE", &tv);
}
```

### 9. 内核参数与 PSR 调优

#### 9.1 AMDGPU PSR 内核模块参数

```bash
# 查看可用的 PSR 参数
modinfo amdgpu | grep psr

# 示例输出：
# parm:  amdgpu_psr:PSR enable (0 = disable, 1 = enable) (int)
# parm:  amdgpu_psr_level:PSR level (0 = off, 1 = PSR1, 2 = PSR2) (int)

# 在模块加载时设置 PSR 参数
options amdgpu amdgpu_psr=1 amdgpu_psr_level=2
```

#### 9.2 系统级 PSR 调优

```bash
#!/bin/bash
# PSR 调优脚本：psr_tune.sh

echo "PSR 调优配置"
echo "============="

# 1. 检查当前 PSR 状态
echo "1. 当前 PSR 状态:"
cat /sys/kernel/debug/dri/0/amdgpu_dm_psr_state 2>/dev/null || \
    echo "  debugfs 不可用"

# 2. 检查 PSR 级别
echo "2. PSR 级别设置:"
cat /sys/kernel/debug/dri/0/amdgpu_dm_psr_level 2>/dev/null || \
    echo "  使用默认级别"

# 3. 设置 PSR 级别
case "$1" in
    off)
        echo 0 > /sys/kernel/debug/dri/0/amdgpu_dm_psr_level 2>/dev/null
        echo "  PSR 已禁用"
        ;;
    psr1)
        echo 1 > /sys/kernel/debug/dri/0/amdgpu_dm_psr_level 2>/dev/null
        echo "  PSR1 已启用"
        ;;
    psr2)
        echo 2 > /sys/kernel/debug/dri/0/amdgpu_dm_psr_level 2>/dev/null
        echo "  PSR2 已启用"
        ;;
    *)
        echo "  用法: $0 {off|psr1|psr2}"
        ;;
esac

# 4. 设置进入延迟（帧数延迟后进入 PSR）
echo "4. 设置 PSR 进入延迟..."
# 更长的延迟 → 更少的进入/退出切换 → 更少的闪烁
# 较短的延迟 → 更多的节能
echo 4 > /sys/kernel/debug/dri/0/amdgpu_dm_psr_entry_delay 2>/dev/null

# 5. 查看统计信息
echo "5. PSR 统计信息:"
cat /sys/kernel/debug/dri/0/amdgpu_dm_psr_stats 2>/dev/null || \
    echo "  统计信息不可用"
```

### 10. 常见 PSR 问题与解决方案

#### 10.1 PSR 问题分类表

| 问题 | 症状 | 原因 | 解决方案 |
|------|------|------|---------|
| 屏幕闪烁 | 进入/退出 PSR 时短暂闪烁 | 面板时序不兼容 | 增加进入延迟或禁用 PSR |
| 鼠标卡顿 | 移动鼠标时不流畅 | PSR 退出延迟 | 升级到 PSR2 或降低退出延迟 |
| 黑屏 | PSR 进入后无法恢复 | 面板缓冲区损坏 | 禁用 PSR，检查面板固件 |
| 画面撕裂 | 退出 PSR 时显示撕裂 | PSR 退出与 VSync 不同步 | 同步 PSR 退出与 VSync |
| 无法进入 PSR | PSR 始终不激活 | 条件不满足 | 检查进入条件日志 |
| CRC 错误 | PSR 自刷新数据错误 | 链路噪声 | 启用 CRC 纠错或禁用 PSR |

#### 10.2 排查 PSR 进入失败

```bash
#!/bin/bash
# PSR 诊断脚本：psr_diagnose.sh
# 诊断 PSR 无法进入的问题

echo "PSR 诊断工具"
echo "============="
echo ""

# 1. 检查硬件支持
echo "1. 检查 PSR 硬件支持..."
dpcd_val=$(cat /sys/kernel/debug/dri/0/DP-1/dpcd 2>/dev/null | \
           xxd -p | dd bs=1 skip=$((0x070)) count=1 2>/dev/null)

if [ -n "$dpcd_val" ]; then
    psr1=$(( (0x$dpcd_val) & 1 ))
    psr2=$(( (0x$dpcd_val >> 1) & 1 ))
    echo "  PSR1 支持: $([ $psr1 -eq 1 ] && echo '是' || echo '否')"
    echo "  PSR2 支持: $([ $psr2 -eq 1 ] && echo '是' || echo '否')"
else
    echo "  无法读取 DPCD（可能需要 root）"
fi

# 2. 检查 PSR 启用状态
echo "2. 检查 PSR 启用状态..."
psr_enable=$(cat /sys/kernel/debug/dri/0/amdgpu_dm_psr_enable 2>/dev/null)
echo "  PSR 启用: $psr_enable"

# 3. 检查 PSR 状态
echo "3. PSR 当前状态:"
psr_state=$(cat /sys/kernel/debug/dri/0/amdgpu_dm_psr_state 2>/dev/null)
echo "  $psr_state"

# 4. 检查可能的阻塞因素
echo "4. 检查 PSR 阻塞因素..."

# 检查光标是否在移动
cursor_pos=$(cat /sys/class/drm/card0/crtc-0/cursor_position 2>/dev/null)
echo "  光标位置: $cursor_pos"

# 检查显示器刷新率
refresh=$(modetest -M amdgpu -c 2>/dev/null | grep " 1920x1080" | \
          grep -oP '\d+\.\d+' | head -1)
echo "  刷新率: $refresh Hz"

# 5. 检查 PSR 统计信息
echo "5. PSR 统计信息:"
cat /sys/kernel/debug/dri/0/amdgpu_dm_psr_stats 2>/dev/null || \
    echo "  不可用"

# 6. 建议
echo ""
echo "诊断建议:"
if [ "$psr_enable" != "1" ]; then
    echo "  - PSR 未启用，尝试: echo 1 > /sys/kernel/debug/dri/0/amdgpu_dm_psr_enable"
fi
if [ $(( (0x$dpcd_val) & 1 )) -eq 0 ]; then
    echo "  - 面板不支持 PSR，无法启用"
fi
echo "  - 详细日志: echo 'file amdgpu_dm_psr.c +p' > /sys/kernel/debug/dynamic_debug/control"
```

### 11. 使用 bpftrace 追踪 PSR 事件

```bash
#!/bin/bash
# PSR 事件追踪脚本：trace_psr.sh

echo "启动 PSR 事件追踪..."
echo "此脚本将追踪 PSR 相关内核函数调用"
echo ""

bpftrace -e '
kprobe:dc_set_psr_config
{
    printf("=== PSR 配置 ===\n");
    printf("  PID: %d, 进程: %s\n", pid, comm);
    printf("  link: 0x%lx\n", arg0);
    printf("  config: 0x%lx\n", arg1);
    printf("  时间: %lld\n", nsecs);
}

kprobe:dc_set_psr_enable
{
    printf("=== PSR 启用/禁用 ===\n");
    printf("  PID: %d, 进程: %s\n", pid, comm);
    printf("  link: 0x%lx\n", arg0);
    printf("  enable: %d\n", arg1);
    printf("  时间: %lld\n", nsecs);
}

kprobe:should_enter_psr
{
    printf("=== PSR 进入检查 ===\n");
    printf("  PID: %d, 进程: %s\n", pid, comm);
}

kretprobe:should_enter_psr
{
    if (retval == 1) {
        printf("  → 允许进入 PSR\n");
    } else {
        printf("  → 不允许进入 PSR\n");
    }
}

kprobe:dcn20_set_psr
{
    printf("=== DCN20 PSR 寄存器编程 ===\n");
    printf("  PID: %d, 进程: %s\n", pid, comm);
    printf("  link: 0x%lx\n", arg0);
    printf("  enable: %d\n", arg1);
}

kprobe:amdgpu_dm_set_psr
{
    printf("=== DM PSR 控制 ===\n");
    printf("  PID: %d, 进程: %s\n", pid, comm);
    printf("  crtc: 0x%lx\n", arg0);
    printf("  enable: %d\n", arg1);
}

kprobe:psr_handle_irq
{
    printf("=== PSR IRQ 处理 ===\n");
    printf("  PID: %d, 进程: %s\n", pid, comm);
    printf("  IRQ 类型: 0x%x\n", arg1);
}
'
```

### 12. PSR 与 eDP 电源管理的关系

#### 12.1 eDP 电源状态与 PSR

eDP 接口定义了多种电源状态，PSR 是其中重要的一环：

```
eDP 电源状态层次：

  D0 (正常工作)
    │
    ├── 全速模式 (Full Link)
    │   ├── 正常帧传输
    │   └── PSR 禁用
    │
    ├── PSR 模式
    │   ├── 主链路休眠（PSR1）
    │   └── 主链路降速（PSR2）
    │
    └── Panel Replay 模式
        └── ALPM 链路

  D3 (睡眠)
    ├── 面板关闭
    ├── eDP 链路关闭
    └── 最小功耗
```

#### 12.2 PSR 与 SMU（System Management Unit）交互

AMDGPU 中，PSR 状态变化需要通知 SMU 以协调整体电源管理：

```c
/* 伪代码：PSR 与 SMU 交互 */

/* 通知 SMU PSR 状态变化 */
static int amdgpu_dm_psr_notify_smu(struct amdgpu_device *adev,
                                      bool psr_active)
{
    uint32_t message;

    if (psr_active) {
        /* PSR 活跃，允许更激进的电源门控 */
        message = SMU_MSG_SetPSRFlag;
        /* 降低 DCN 时钟 */
        smu_send_message(adev, SMU_MSG_SetDCNClock,
                         DCN_CLOCK_MIN);
    } else {
        /* PSR 不活跃，恢复正常时钟 */
        message = SMU_MSG_ClearPSRFlag;
        smu_send_message(adev, SMU_MSG_SetDCNClock,
                         DCN_CLOCK_NORMAL);
    }

    return smu_send_message(adev, message, 0);
}
```

## 实践操作

### 实践 1：PSR 状态监控工具

```c
/*
 * 文件名：psr_monitor.c
 * 功能：监控 PSR 状态变化并输出统计信息
 * 编译：gcc -o psr_monitor psr_monitor.c
 *
 * 说明：读取 debugfs PSR 状态文件，监控状态变化
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <time.h>
#include <errno.h>

#define PSR_STATE_PATH "/sys/kernel/debug/dri/0/amdgpu_dm_psr_state"
#define PSR_STATS_PATH "/sys/kernel/debug/dri/0/amdgpu_dm_psr_stats"
#define POLL_INTERVAL_MS 100

struct psr_stats {
    unsigned int entry_count;
    unsigned int exit_count;
    unsigned long total_entry_time_us;
    unsigned long total_exit_time_us;
    unsigned int current_entry_duration_s;
    unsigned int crc_errors;
    unsigned int link_crc_errors;
};

static int read_line(const char *path, char *buf, size_t size)
{
    int fd = open(path, O_RDONLY);
    if (fd < 0)
        return -1;

    ssize_t n = read(fd, buf, size - 1);
    close(fd);

    if (n <= 0)
        return -1;

    buf[n] = '\0';
    /* 去除换行符 */
    size_t len = strlen(buf);
    if (len > 0 && buf[len - 1] == '\n')
        buf[len - 1] = '\0';

    return 0;
}

static int get_psr_state(char *state_str, size_t size)
{
    return read_line(PSR_STATE_PATH, state_str, size);
}

static int get_psr_stats(struct psr_stats *stats)
{
    char buf[1024];
    if (read_line(PSR_STATS_PATH, buf, sizeof(buf)) < 0)
        return -1;

    /* 解析统计信息行 */
    /* 格式: "Total Frames Rendered: %u ..." */
    sscanf(buf,
           "Total Frames PSR: %u\n"
           "PSR Efficiency: %*f%%\n"
           "CRC Errors: %u",
           &stats->entry_count,
           &stats->crc_errors);

    return 0;
}

static void print_psr_status(const char *state,
                              const struct psr_stats *stats,
                              time_t elapsed)
{
    time_t now = time(NULL);
    struct tm *tm = localtime(&now);

    printf("\033[2J\033[H");  /* 清屏 */
    printf("═══════════════════════════════════════════\n");
    printf("  PSR 状态监控 (更新时间: %02d:%02d:%02d)\n",
           tm->tm_hour, tm->tm_min, tm->tm_sec);
    printf("═══════════════════════════════════════════\n\n");

    /* PSR 状态 */
    printf("  PSR 状态: ");
    if (strstr(state, "ACTIVE")) {
        printf("\033[32m活跃 (Self-Refresh)\033[0m\n");
    } else if (strstr(state, "INACTIVE") ||
               strstr(state, "SLEEP")) {
        printf("\033[33m非活跃\033[0m\n");
    } else if (strstr(state, "ERROR")) {
        printf("\033[31m错误！\033[0m\n");
    } else {
        printf("\033[37m%s\033[0m\n", state);
    }

    printf("  详细信息: %s\n\n", state);

    /* 统计信息 */
    printf("  ┌──────────────────────────────┐\n");
    printf("  │ 统计项目            │ 数值    │\n");
    printf("  ├──────────────────────────────┤\n");
    printf("  │ 进入 PSR 次数       │ %-7u │\n", stats->entry_count);
    printf("  │ CRC 错误数          │ %-7u │\n", stats->crc_errors);
    printf("  │ 运行时间            │ %-7lus │\n", elapsed);
    printf("  └──────────────────────────────┘\n");

    /* 状态指示器 */
    printf("\n  状态: ");
    if (strstr(state, "ACTIVE")) {
        printf("\033[32m████████ PSR 节能中 ████████\033[0m\n");
        printf("  GPU 显示功耗: 约 0.5-1W\n");
    } else {
        printf("\033[33m░░░░░░░░ 正常显示 ░░░░░░░░\033[0m\n");
        printf("  GPU 显示功耗: 约 3-8W\n");
    }

    printf("\n  提示: 移动鼠标或按任意键将触发 PSR 退出\n");
    printf("═══════════════════════════════════════════\n");
    fflush(stdout);
}

int main(int argc, char *argv[])
{
    char state[256];
    struct psr_stats stats = {0};
    time_t start_time = time(NULL);
    int repeat = -1;  /* -1 表示无限循环 */

    if (argc > 1) {
        repeat = atoi(argv[1]);
        if (repeat <= 0) repeat = 1;
    }

    printf("PSR 状态监控工具\n");
    printf("监控路径: %s\n", PSR_STATE_PATH);
    printf("按 Ctrl+C 退出\n\n");
    sleep(2);

    int count = 0;
    while (repeat < 0 || count < repeat) {
        if (get_psr_state(state, sizeof(state)) < 0) {
            fprintf(stderr, "无法读取 PSR 状态: %s\n",
                    strerror(errno));
            fprintf(stderr, "请确认: 1) debugfs 已挂载 2) amdgpu 驱动已加载\n");
            return 1;
        }

        get_psr_stats(&stats);

        time_t elapsed = time(NULL) - start_time;
        print_psr_status(state, &stats, elapsed);

        usleep(POLL_INTERVAL_MS * 1000);
        count++;
    }

    return 0;
}
```

编译和运行：

```bash
# 编译
gcc -o psr_monitor psr_monitor.c -Wall

# 运行（持续监控）
sudo ./psr_monitor

# 运行 10 次后退出
sudo ./psr_monitor 10
```

### 实践 2：PSR 节能效果测试脚本

```bash
#!/bin/bash
# 文件名：psr_power_test.sh
# 功能：测试 PSR 开启和关闭时的功耗差异
# 用法：sudo ./psr_power_test.sh [test_duration_seconds]

DURATION=${1:-30}
POWER_PATH="/sys/class/drm/card0/device/hwmon/hwmon*/power1_average"

echo "=========================================="
echo "  PSR 节能效果测试"
echo "  测试时长: ${DURATION}秒"
echo "=========================================="

# 函数：获取当前功耗（mW）
get_power_mw() {
    local power_file=$(ls $POWER_PATH 2>/dev/null | head -1)
    if [ -n "$power_file" ]; then
        cat "$power_file" 2>/dev/null || echo 0
    else
        echo 0
    fi
}

# 函数：测试指定 PSR 状态下的功耗
test_psr_mode() {
    local mode_name="$1"
    local psr_level=$2

    echo ""
    echo "测试模式: $mode_name"
    echo "------------------------"

    # 设置 PSR 级别
    if [ -f /sys/kernel/debug/dri/0/amdgpu_dm_psr_level ]; then
        echo $psr_level > /sys/kernel/debug/dri/0/amdgpu_dm_psr_level
        echo "  PSR 级别已设置为: $psr_level"
    fi

    # 等待状态稳定
    echo "  等待 ${DURATION} 秒..."
    sleep 2

    # 采样功耗
    local total_power=0
    local samples=0

    for ((i=0; i<DURATION; i++)); do
        local power=$(get_power_mw)
        if [ "$power" -gt 0 ]; then
            total_power=$((total_power + power))
            samples=$((samples + 1))
        fi
        sleep 1
    done

    # 计算平均值
    if [ "$samples" -gt 0 ]; then
        local avg_power=$((total_power / samples))
        echo "  平均功耗: ${avg_power} mW ($(echo "scale=1; $avg_power/1000" | bc) W)"
        echo "$avg_power"
    else
        echo "  无法读取功耗数据"
        echo "0"
    fi
}

# 确保有 root 权限
if [ "$EUID" -ne 0 ]; then
    echo "请使用 sudo 运行此脚本"
    exit 1
fi

# 挂载 debugfs（如果需要）
mount -t debugfs none /sys/kernel/debug 2>/dev/null

# 测试 PSR 关闭
power_off=$(test_psr_mode "PSR 关闭" 0)

# 测试 PSR1
power_psr1=$(test_psr_mode "PSR1（标准PSR）" 1)

# 测试 PSR2（如果支持）
power_psr2=0
if [ -f /sys/kernel/debug/dri/0/amdgpu_dm_psr_level ]; then
    power_psr2=$(test_psr_mode "PSR2（选择性更新）" 2)
fi

# 恢复默认设置
echo 1 > /sys/kernel/debug/dri/0/amdgpu_dm_psr_level 2>/dev/null

# 显示结果
echo ""
echo "=========================================="
echo "  测试结果汇总"
echo "=========================================="
echo ""
echo "  模式          | 平均功耗 | 节省比例"
echo "  ─────────────────────────────────────"
if [ "$power_off" -gt 0 ]; then
    echo "  PSR 关闭       | ${power_off} mW  | 基准"
fi
if [ "$power_psr1" -gt 0 ]; then
    local save_psr1=$(( (power_off - power_psr1) * 100 / power_off ))
    echo "  PSR1（标准）   | ${power_psr1} mW  | ${save_psr1}%"
fi
if [ "$power_psr2" -gt 0 ]; then
    local save_psr2=$(( (power_off - power_psr2) * 100 / power_off ))
    echo "  PSR2（SU）     | ${power_psr2} mW  | ${save_psr2}%"
fi
echo ""
echo "=========================================="
echo "  测试完成"
echo "=========================================="
```

### 实践 3：PSR 闪烁检测程序

```c
/*
 * 文件名：psr_flicker_detect.c
 * 功能：检测 PSR 进入/退出时的屏幕闪烁
 * 编译：gcc -o psr_flicker_detect psr_flicker_detect.c -lm
 *
 * 说明：使用 CRC 校验检测 PSR 相关的图像异常
 * 需要显示稳定的测试图案来进行 CRC 比对
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <time.h>

#define CRC_CONTROL_PATH  "/sys/class/drm/card0/crtc-0/crc/control"
#define CRC_DATA_PATH     "/sys/class/drm/card0/crtc-0/crc/data"
#define PSR_STATE_PATH    "/sys/kernel/debug/dri/0/amdgpu_dm_psr_state"

#define MAX_CRC_SAMPLES   1000
#define FLICKER_THRESHOLD 5  /* CRC 连续异常阈值 */

struct crc_sample {
    unsigned int crc_value;
    struct timespec timestamp;
    char psr_state[64];
};

static int write_file(const char *path, const char *value)
{
    int fd = open(path, O_WRONLY);
    if (fd < 0) return -1;

    int ret = write(fd, value, strlen(value));
    close(fd);
    return ret > 0 ? 0 : -1;
}

static int read_crc(unsigned int *crc_value)
{
    char buf[64];
    int fd = open(CRC_DATA_PATH, O_RDONLY);
    if (fd < 0) return -1;

    ssize_t n = read(fd, buf, sizeof(buf) - 1);
    close(fd);

    if (n <= 0) return -1;
    buf[n] = '\0';

    /* CRC 格式: "XXXXXXXX 00000000\n" */
    return sscanf(buf, "%x", crc_value) == 1 ? 0 : -1;
}

static int read_psr_state(char *state, size_t size)
{
    int fd = open(PSR_STATE_PATH, O_RDONLY);
    if (fd < 0) {
        snprintf(state, size, "unknown");
        return -1;
    }

    ssize_t n = read(fd, state, size - 1);
    close(fd);

    if (n <= 0) {
        snprintf(state, size, "unknown");
        return -1;
    }

    state[n] = '\0';
    /* 去除换行 */
    size_t len = strlen(state);
    if (len > 0 && state[len - 1] == '\n')
        state[len - 1] = '\0';

    return 0;
}

int main(int argc, char *argv[])
{
    struct crc_sample samples[MAX_CRC_SAMPLES];
    unsigned int duration_s = 30;  /* 默认测试 30 秒 */
    int sample_count = 0;
    int flicker_count = 0;
    unsigned int crc_reference = 0;
    int has_reference = 0;

    if (argc > 1)
        duration_s = atoi(argv[1]);

    printf("PSR 闪烁检测程序\n");
    printf("================\n\n");

    /* 启用 CRC */
    if (write_file(CRC_CONTROL_PATH, "auto") < 0) {
        fprintf(stderr, "无法启用 CRC（需要 root 权限？）\n");
        return 1;
    }

    /* 等待 CRC 稳定 */
    sleep(1);

    /* 获取参考 CRC */
    if (read_crc(&crc_reference) == 0) {
        has_reference = 1;
        printf("参考 CRC: 0x%08X\n", crc_reference);
    }

    printf("开始检测闪烁（测试时长: %d 秒）...\n\n", duration_s);

    time_t start = time(NULL);
    while (time(NULL) - start < duration_s &&
           sample_count < MAX_CRC_SAMPLES) {
        struct crc_sample *s = &samples[sample_count];

        clock_gettime(CLOCK_MONOTONIC, &s->timestamp);
        read_psr_state(s->psr_state, sizeof(s->psr_state));

        if (read_crc(&s->crc_value) == 0) {
            /* CRC 读取有效 */
        } else {
            s->crc_value = 0xFFFFFFFF;
        }

        sample_count++;

        /* 检查 CRC 是否异常 */
        if (has_reference &&
            sample_count > 1 &&
            s->crc_value != crc_reference &&
            s->crc_value != samples[sample_count - 2].crc_value) {

            /* CRC 变化可能是闪烁 */
            printf("  [%4d] CRC: 0x%08X (参考: 0x%08X) PSR: %s\n",
                   sample_count, s->crc_value, crc_reference,
                   s->psr_state);
            flicker_count++;
        }

        usleep(16667);  /* 约 60fps 采样间隔 */
    }

    /* 禁用 CRC */
    write_file(CRC_CONTROL_PATH, "disable");

    /* 输出结果 */
    printf("\n================\n");
    printf("  检测结果\n");
    printf("================\n");
    printf("  总采样数: %d\n", sample_count);
    printf("  CRC 异常次数: %d\n", flicker_count);
    printf("  异常率: %.1f%%\n",
           (float)flicker_count / sample_count * 100);

    if (flicker_count > FLICKER_THRESHOLD) {
        printf("\n  检测到频繁的图像变化！\n");
        printf("  可能原因:\n");
        printf("    1. PSR 频繁进入/退出\n");
        printf("    2. 显示内容正在变化\n");
        printf("  建议: 在静态显示内容时重新测试\n");
    } else {
        printf("\n  未检测到明显的 PSR 相关闪烁\n");
    }

    return flicker_count > FLICKER_THRESHOLD ? 1 : 0;
}
```

编译和运行：

```bash
# 编译
gcc -o psr_flicker_detect psr_flicker_detect.c -lm

# 运行检测
sudo ./psr_flicker_detect

# 指定测试时长（60秒）
sudo ./psr_flicker_detect 60
```

### 实践 4：PSR 强制进入/退出测试

```bash
#!/bin/bash
# 文件名：psr_force_test.sh
# 功能：强制 PSR 进入/退出，测试系统稳定性

DEBUGFS_BASE="/sys/kernel/debug/dri/0"

echo "PSR 强制测试工具"
echo "================="
echo ""

# 检查 debugfs 接口
check_interface() {
    if [ ! -f "$DEBUGFS_BASE/amdgpu_dm_psr_enable" ]; then
        echo "错误: PSR debugfs 接口不可用"
        echo "请确认: sudo mount -t debugfs none /sys/kernel/debug"
        exit 1
    fi
}

# 强制 PSR 进入
force_psr_entry() {
    echo "强制 PSR 进入..."
    echo 1 > "$DEBUGFS_BASE/amdgpu_dm_psr_force_entry" 2>/dev/null
    sleep 0.5

    local state=$(cat "$DEBUGFS_BASE/amdgpu_dm_psr_state" 2>/dev/null)
    if echo "$state" | grep -q "ACTIVE"; then
        echo "  [成功] PSR 已进入激活状态"
    else
        echo "  [失败] PSR 未能进入（状态: $state）"
    fi
}

# 强制 PSR 退出
force_psr_exit() {
    echo "强制 PSR 退出..."
    echo 1 > "$DEBUGFS_BASE/amdgpu_dm_psr_force_exit" 2>/dev/null
    sleep 0.5

    local state=$(cat "$DEBUGFS_BASE/amdgpu_dm_psr_state" 2>/dev/null)
    if echo "$state" | grep -q "INACTIVE"; then
        echo "  [成功] PSR 已退出"
    else
        echo "  [失败] PSR 未能退出（状态: $state）"
    fi
}

# 循环压力测试
stress_test() {
    local iterations=${1:-100}

    echo "PSR 压力测试: $iterations 次循环"
    echo ""

    local pass=0
    local fail=0

    for ((i=1; i<=iterations; i++)); do
        echo -ne "  循环 $i/$iterations\r"

        # 强制进入
        echo 1 > "$DEBUGFS_BASE/amdgpu_dm_psr_force_entry" 2>/dev/null
        usleep 500000  # 500ms

        # 检查状态
        local state=$(cat "$DEBUGFS_BASE/amdgpu_dm_psr_state" 2>/dev/null)
        if echo "$state" | grep -q "ACTIVE"; then
            pass=$((pass + 1))
        else
            fail=$((fail + 1))
        fi

        # 强制退出
        echo 1 > "$DEBUGFS_BASE/amdgpu_dm_psr_force_exit" 2>/dev/null
        usleep 500000
    done

    echo ""
    echo "压力测试完成"
    echo "  成功: $pass"
    echo "  失败: $fail"
}

# 显示菜单
show_menu() {
    echo ""
    echo "PSR 测试菜单:"
    echo "  1) 查看当前 PSR 状态"
    echo "  2) 强制进入 PSR"
    echo "  3) 强制退出 PSR"
    echo "  4) 压力测试（100次循环）"
    echo "  5) 启用 PSR 详细日志"
    echo "  6) 退出"
    echo -n "请选择 [1-6]: "
}

# 启用 PSR 日志
enable_psr_log() {
    local log_file="/tmp/psr_trace_$(date +%Y%m%d_%H%M%S).log"

    echo "启用 PSR 详细日志（输出到 $log_file）..."
    echo "按 Ctrl+C 停止记录"

    # 同时监控多个 PSR 相关事件
    {
        echo "=== PSR 日志开始 $(date) ==="
        while true; do
            echo "[$(date +%H:%M:%S.%N)] PSR状态: $(cat $DEBUGFS_BASE/amdgpu_dm_psr_state 2>/dev/null)"
            echo "[$(date +%H:%M:%S.%N)] PSR统计: $(cat $DEBUGFS_BASE/amdgpu_dm_psr_stats 2>/dev/null)"
            sleep 0.1
        done
    } > "$log_file" 2>&1 &

    local pid=$!
    trap "kill $pid 2>/dev/null; echo '日志已保存到 $log_file'; exit" INT
    wait $pid
}

# 主菜单循环
main() {
    check_interface

    while true; do
        show_menu
        read -r choice
        case $choice in
            1)
                echo ""
                echo "当前 PSR 状态:"
                cat "$DEBUGFS_BASE/amdgpu_dm_psr_state" 2>/dev/null || echo "不可用"
                echo ""
                echo "PSR 统计信息:"
                cat "$DEBUGFS_BASE/amdgpu_dm_psr_stats" 2>/dev/null || echo "不可用"
                ;;
            2)
                force_psr_entry
                ;;
            3)
                force_psr_exit
                ;;
            4)
                stress_test 100
                ;;
            5)
                enable_psr_log
                ;;
            6)
                echo "退出"
                exit 0
                ;;
            *)
                echo "无效选项，请选择 1-6"
                ;;
        esac
        echo ""
        echo -n "按回车键继续..."
        read -r
    done
}

main
```

编译和运行：

```bash
chmod +x psr_force_test.sh
sudo ./psr_force_test.sh
```

## 案例分析

### 案例一：笔记本合盖/开盖后的 PSR 恢复失败

**问题现象：**
某搭载 AMD Ryzen 7 5800U 的笔记本在合盖休眠后开盖，eDP 屏幕显示正常但 PSR 无法再次进入。`cat amdgpu_dm_psr_state` 始终显示 `INACTIVE`，PSR 统计数据显示 `entry_fail_count` 持续增加。

```
PSR 统计样例：
  psr_version: 1
  psr_state: INACTIVE
  entry_count: 42
  exit_count: 38
  entry_fail_count: 7    ← 异常增多
  last_entry_fail_reason: LINK_NOT_ACTIVE
```

**根因分析：**
1. 合盖时 eDP 链路进入 D3 状态，PSR 上下文丢失
2. 开盖后链路重新训练，但 PSR 配置未正确恢复
3. DPCD 寄存器 0x080（PSR_CONFIG）值被重置为 0
4. DC 状态机中 PSR 上下文标志 `ctx->psr_context_valid` 未清除导致恢复失败

**解决方案：**
```c
// 内核修复：在 amdgpu_dm_resume 中添加 PSR 重建逻辑
void amdgpu_dm_resume_psr(struct amdgpu_display_manager *dm)
{
    struct dm_crtc_state *dm_state;
    struct drm_crtc *crtc;

    if (!dm->psr_feature_enabled)
        return;

    /* 强制重建 PSR 上下文 */
    dm->dc->hwss.edp_backlight_control(dm->dc, NULL, false);

    /* 重新初始化 DPCD PSR 寄存器 */
    drm_dp_dpcd_writeb(&dm->dp_aux,
                       DP_PSR_ENABLE, 0);
    msleep(10);
    drm_dp_dpcd_writeb(&dm->dp_aux,
                       DP_PSR_ENABLE,
                       DP_PSR_ENABLE_ENABLE);

    /* 清除无效的 PSR 上下文 */
    dm->dc->current_state->psr_context_valid = false;
    DC_LOG_PSR("PSR context rebuilt after resume\n");
}
```

**经验教训：** PSR 状态与 eDP 链路状态深度耦合，电源状态转换（S3/S4）后必须重建 PSR 配置。

### 案例二：PSR 导致的外接显示器闪烁

**问题现象：**
用户通过 USB-C/DP 外接 4K 显示器时，在桌面静止 10-15 秒后显示器出现周期性微闪（约每 5 秒一次）。内置 eDP 屏幕正常，仅外接 DP 显示器受影响。

```
dmesg 日志：
[12345.678] amdgpu: [psr] PSR entry succeeded
[12345.685] amdgpu: [psr] PSR exit due to DPCD update
[12345.690] amdgpu: [psr] PSR entry succeeded
[12345.697] amdgpu: [psr] PSR exit due to DPCD update
```

**根因分析：**
1. 外接 DP 显示器虽然支持 PSR，但 DPCD 寄存器 0x071（PSR_CAPS）的 `PSR_CAPS_SETUP_TIME` 字段值为 0，表示设置时间为 0
2. AMDGPU DC 层使用默认的 PSR 设置时间参数，与该显示器不兼容
3. 显示器在接收到 PSR 进入命令后无法在指定时间内完成 Self-Refresh 配置，导致反复进入/退出

**解决方案：**
```bash
# 临时解决方案：禁用 PSR
echo 0 | sudo tee /sys/kernel/debug/dri/0/amdgpu_dm_psr_enable

# 永久解决方案：内核参数
# 在 /etc/modprobe.d/amdgpu.conf 中添加：
options amdgpu amdgpu_psr=0

# 更精准的修复：通过 EDID 覆盖或驱动 quirk
# 在 amdgpu_dm_psr.c 中添加显示器 quirk：
static const struct psr_quirk psr_quirks[] = {
    { "DEL", "DELL U2723QE", PSR_SETUP_TIME_EXTENDED },
    { "SAM", "SAMSUNG S27A70", PSR_SETUP_TIME_EXTENDED },
};
```

**经验教训：** 并非所有标称支持 PSR 的显示器实现都完全符合规范，需要在实际硬件上验证兼容性。

### 案例三：KDE Plasma 桌面环境下 PSR 无法进入

**问题现象：**
KDE Plasma Wayland 会话中，桌面静止超过 30 秒后 PSR 仍无法进入。GNOME 下相同硬件 PSR 正常工作。

```
PSR 状态（KDE）：
  psr_state: INACTIVE
  last_exit_reason: CURSOR_UPDATE
  entry_fail_count: 0

PSR 状态（GNOME）：
  psr_state: ACTIVE
  entry_count: 15
```

**根因分析：**
1. KDE Plasma 的 KWin 合成器默认每 50ms 发送一次光标位置更新，即使鼠标静止（用于光标呼吸效果）
2. 每次光标更新都触发 PSR 退出，导致 PSR 永远无法稳定进入
3. GNOME 的 Mutter 合成器在检测到无输入事件后会停止光标更新

```c
// KWin 中导致 PSR 退出的光标更新代码（简化）
void KWin::Cursor::updatePosition()
{
    if (m_breatheEffect) {
        QPointF offset(cos(m_angle) * 0.5, sin(m_angle) * 0.5);
        move(m_pos + offset);
        m_angle += 0.1;
    }
}
```

**解决方案：**
```bash
# 方案 1：禁用 KWin 光标呼吸效果
kwriteconfig5 --file kwinrc \
    --group Effects \
    --key CursorBreathe false
qdbus org.kde.KWin /KWin reconfigure

# 方案 2：增加 PSR 进入延迟容忍度
echo 3000 | sudo tee /sys/kernel/debug/dri/0/amdgpu_dm_psr_idle_threshold

# 方案 3：使用内核参数提高 PSR 进入阈值
# /etc/modprobe.d/amdgpu.conf
options amdgpu amdgpu_psr_idle_threshold=3000
```

**经验教训：** Wayland 合成器的渲染策略对 PSR 行为有显著影响。

## 相关链接

1. [Linux 内核 DRM DP PSR 核心](https://elixir.bootlin.com/linux/latest/source/include/drm/drm_dp_helper.h)
   - drm_dp_helper.h：DPCD 寄存器定义（0x070-0x082 的 PSR 相关寄存器）

2. [AMDGPU DC PSR 实现](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/core/dc.c)
   - `dc_psr_config()` / `dc_psr_enable()` / `dc_psr_disable()` 接口

3. [AMDGPU DM PSR 层](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_psr.c)
   - amdgpu_dm_psr.c：DM 层的 PSR 管理和 debugfs 接口

4. [DRM DPCD PSR 寄存器定义](https://elixir.bootlin.com/linux/latest/source/include/drm/drm_dp_helper.h#L1268)
   - DP_PSR_SUPPORT (0x070), DP_PSR_CAPS (0x071), DP_PSR_ENABLE (0x080), DP_PSR_STATUS (0x081)

5. [IGT GPU 工具 - PSR 测试](https://gitlab.freedesktop.org/drm/igt-gpu-tools/-/tree/master/tests/kms_psr.c)
   - kms_psr: PSR1 测试套件, kms_psr2_su: PSR2 选择性更新测试, kms_panel_replay: Panel Replay 测试

6. [VESA eDP 标准 (需会员)](https://vesa.org/standards/)
   - eDP v1.4: PSR1 规范, eDP v1.4a: PSR2 选择性更新, eDP v1.5: Panel Replay

7. [AMDGPU DC PSR 头文件](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/inc/hw/psr.h)
   - struct dc_psr_config / struct dc_psr_context 定义

8. [AMDGPU 显示节能内核文档](https://www.kernel.org/doc/html/latest/gpu/amdgpu/display/display-power-saving.html)
   - AMDGPU 显示节能特性的内核文档和模块参数说明

9. [DP PHY 测试模式与 PSR 调试](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/dcn10/dcn10_hw_sequencer.c)
   - DCN PSR 寄存器编程：PSR_ENTRY, PSR_EXIT, PSR_FRAME_UPDATE

10. [eDP Panel Self Refresh 白皮书](https://www.allion.com/edp-panel-self-refresh/)
    - PSR 工作原理的可视化图解和测试方法论

11. [SystemTap PSR 追踪示例](https://sourceware.org/systemtap/examples/)
    - 使用 SystemTap 追踪 PSR 进入/退出事件和延迟分析

12. [amd-gfx 邮件列表 PSR 讨论](https://lore.kernel.org/amd-gfx/)
    - amd-gfx 邮件列表中的 PSR 相关补丁讨论和问题修复上下文

## 今日小结

### 核心概念回顾

Panel Self Refresh（PSR）是 eDP 标准中一项关键的显示节能技术，通过利用面板内部的帧缓冲器，在静态画面下关闭 GPU 到面板的主链路，从而降低显示子系统的功耗。

**PSR 技术三剑客：**
- **PSR1**：标准 PSR，全帧缓冲，主链路完全关闭，适合纯静态场景
- **PSR2**：选择性更新（Selective Update），仅传输变化区域，主链路保持低速率
- **Panel Replay**：基于 ALPM 的演进技术，无需面板缓冲器，兼容性更好

**PSR 的核心设计权衡：**
- 节能效果 vs 退出延迟：PSR1 节能更佳但退出延迟大（~3ms），PSR2 退出更快（~1ms）
- 节能效果 vs 用户体验：频繁进入/退出反而可能增加总体功耗并引入闪烁
- 硬件兼容性 vs 通用性：不同显示器的 PSR 实现差异大，需要 per-panel 调优

### 关键技术要点

1. **PSR 状态机**：OFF → SLEEP → ACTIVE(Self-Refresh) → WAKEUP → OFF
2. **PSR 进入条件**：连续静态帧 > 阈值（默认 1-3 秒），无用户输入，无 modeset
3. **PSR 退出触发器**：光标移动、页面滚动、视频播放、HPD 事件、DPMS 切换
4. **PSR 调试工具箱**：debugfs (state/stats/enable/force_entry/force_exit)
5. **内核参数调优**：`amdgpu_psr`, `amdgpu_psr_idle_threshold`, `amdgpu_panel_replay`

### 常见误区

- **误区 1**：PSR 在所有场景下都省电。事实：频繁进入/退出（如快速滚动网页）可能增加功耗。
- **误区 2**：PSR 不影响显示质量。事实：部分显示器存在 PSR 进入/退出时的闪烁问题。
- **误区 3**：PSR 只与面板有关。事实：PSR 行为受 GPU 驱动、合成器（KWin/Mutter）、输入设备等多因素影响。
- **误区 4**：PSR2 完全替代 PSR1。事实：PSR2 需要显示器额外支持，实际部署中 PSR1 更为普遍。

## 扩展思考

### PSR 技术的未来演进

1. **Panel Replay 的普及趋势**
   - Panel Replay 不依赖面板缓冲器，降低了显示器的硬件成本
   - AMD RX 7000 系列及更新 GPU 已支持 Panel Replay
   - 预计 Panel Replay 将逐步取代 PSR1/PSR2 成为主流 eDP 节能方案

2. **自适应 PSR 策略**
   - 基于内容分析的智能 PSR 决策：利用 GPU 的视觉分析能力预判画面变化
   - 机器学习预测 PSR 退出时机，减少不必要的状态切换
   - 场景感知：检测到视频播放时自动切换到 PSR2 模式

3. **跨显示器的 PSR 协调**
   - 多显示器场景下不同面板的 PSR 状态独立管理
   - 系统级 PSR 调度：避免一个显示器的 PSR 退出影响其他显示器的节能状态
   - PSR 与 VRR 的协同工作优化

4. **与合成器的深度集成**
   - Wayland 合成器感知 PSR 状态，优化合成策略
   - KWin/Mutter 可以为 PSR 优化帧提交时机
   - 用户空间 API：让应用程序知晓 PSR 状态从而调整渲染策略

### 开放问题

1. **PSR 与 HDR 的兼容性**：HDR 内容的静态画面检测更加复杂，PSR 在 HDR 模式下的行为尚未完全标准化。
2. **PSR 的 QoS 保证**：如何确保 PSR 退出延迟不影响实时应用（如 VR/AR）的帧时序要求。
3. **PSR 能耗模型**：建立精确的 PSR 能耗模型，量化不同场景下的实际节能效果，指导 PSR 策略调优。
4. **跨厂商互操作性**：不同 GPU 厂商（AMD/NVIDIA/Intel）的 PSR 实现与不同面板的组合兼容性测试。

### 下一步学习建议

1. 深入学习 IGT PSR 测试框架的实现，编写自定义 PSR 测试用例
2. 阅读 AMDGPU DC 层的 PSR 源代码，理解硬件寄存器编程细节
3. 在实际硬件上搭建 PSR 测试环境，收集不同场景下的 PSR 行为数据
4. 关注内核邮件列表（amd-gfx）中的 PSR 相关补丁和讨论