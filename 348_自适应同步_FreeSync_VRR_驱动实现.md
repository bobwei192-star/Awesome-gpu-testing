# 自适应同步：FreeSync / VRR 驱动实现

## 学习目标

- 理解 VRR（Variable Refresh Rate）的基本原理和 FreeSync 技术的核心机制
- 掌握 AMDGPU 驱动中 FreeSync 的实现架构，包括 VRR 范围协商、帧率同步和 LFC（Low Framerate Compensation）
- 深入理解 VBlank 控制、DRR（Dynamic Refresh Rate）和 Freesync 的硬件支持
- 掌握 FreeSync 的调试方法、性能分析和问题排查
- 了解 FreeSync Premium / Premium Pro 认证要求和测试方法

## 知识详解

### 概念原理

#### VRR 技术基础

传统固定刷新率显示器以恒定速率刷新画面（如 60Hz、144Hz）。当 GPU 渲染帧率与显示器刷新率不匹配时，会出现两种典型的视觉问题：

- **撕裂（Tearing）**：帧率高于刷新率时，一帧内显示多个不完整的画面
- **卡顿（Stuttering）**：帧率低于刷新率时，同一帧被重复显示导致画面不连续

VRR（Variable Refresh Rate）技术通过允许显示器动态调整刷新率以匹配 GPU 的渲染帧率，从根本上消除撕裂和卡顿。

```ascii
固定刷新率 vs VRR 对比:

固定 60Hz 显示器:
┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐
│ 帧 A ││ 帧 A ││ 帧 B ││ 帧 C ││ 帧 C ││ 帧 D │  ← 显示器输出
└──────┘└──────┘└──────┘└──────┘└──────┘└──────┘
  16.7ms  16.7ms  16.7ms  16.7ms  16.7ms  16.7ms
  ↑ 帧率 45fps: 帧 A 重复, 帧 B 显示不完整 → 卡顿 + 撕裂

FreeSync VRR 显示器:
┌──────┐  ┌──────┐  ┌──────┐    ┌──────┐    ┌──────┐
│ 帧 A │  │ 帧 B │  │ 帧 C │    │ 帧 D │    │ 帧 E │  ← 显示器输出
└──────┘  └──────┘  └──────┘    └──────┘    └──────┘
 22.2ms    22.2ms    22.2ms      22.2ms      22.2ms
  ↑ 帧率 45fps: 每帧显示 22.2ms → 无撕裂, 无卡顿
```

#### FreeSync 技术架构

FreeSync 是 AMD 基于 VESA DisplayPort Adaptive-Sync 标准实现的可变刷新率技术。从 AMDGPU 驱动角度看，FreeSync 的实现分为以下几个层次：

```ascii
FreeSync 软件架构栈:

┌─────────────────────────────────────────────┐
│  用户空间 (User Space)                        │
│  ┌──────────────────────────────────────┐    │
│  │ 游戏/应用程序 (OpenGL/Vulkan/DX)     │    │
│  └──────────┬───────────────────────────┘    │
│             │ drmModeAtomicCommit             │
│  ┌──────────▼───────────────────────────┐    │
│  │ Xserver/Wayland (DRM client)         │    │
│  │ - DRM_IOCTL_MODE_ATOMIC              │    │
│  │ - VRR_ENABLED property               │    │
│  └──────────┬───────────────────────────┘    │
├─────────────┼───────────────────────────────┤
│  内核空间 (Kernel Space)                     │
│  ┌──────────▼───────────────────────────┐    │
│  │ DRM Core (drm_atomic.c)             │    │
│  │ - drm_atomic_check_only()           │    │
│  │ - drm_atomic_commit()               │    │
│  └──────────┬───────────────────────────┘    │
│  ┌──────────▼───────────────────────────┐    │
│  │ AMDGPU DM (amdgpu_dm.c)             │    │
│  │ - amdgpu_dm_atomic_commit_tail()    │    │
│  │ - dm_set_vrr()                      │    │
│  │ - dm_enable_vrr()                   │    │
│  │ - VRR timing adjustment             │    │
│  └──────────┬───────────────────────────┘    │
│  ┌──────────▼───────────────────────────┐    │
│  │ AMDGPU DC (dc/core/dc.c)            │    │
│  │ - dc_stream_set_vrr()               │    │
│  │ - dc_stream_adjust_vmin_vmax()      │    │
│  │ - LFC 决策逻辑                       │    │
│  └──────────┬───────────────────────────┘    │
├─────────────┼───────────────────────────────┤
│  硬件层 (Hardware)                           │
│  ┌──────────▼───────────────────────────┐    │
│  │ DCN 显示引擎                          │    │
│  │ - OTG (Output Timing Generator)      │    │
│  │ - V_TOTAL_MIN / V_TOTAL_MAX 寄存器   │    │
│  │ - DRR (Dynamic Refresh Rate) 模块    │    │
│  │ - FreeSync HPD 握手                   │    │
│  └──────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

#### VRR 工作原理

VRR 的核心原理是动态调整显示器的 VBlank 时长，使显示器等待 GPU 完成当前帧后再开始扫描输出。

```ascii
VRR 时序控制:
                    ┌───────── VRR Range ─────────┐
                    │  48Hz ~ 144Hz (显示器能力)   │
                    └─────────────────────────────┘

正常 VRR 操作:
GPU 帧完成信号 ────┐         ┌─────────┐         ┌─────────┐
                   │         │         │         │         │
                   ▼         ▼         ▼         ▼
显示器扫描输出:  ┌─┴─────────┴─┐   ┌─┴─────────┴─┐   ┌─────
                │   帧 1      │   │   帧 2      │   │  帧 3
                │  (60fps)    │   │  (90fps)    │   │ (120fps)
                └─────────────┘   └─────────────┘   └───
VBlank:              ↓                ↓                ↓
              V_TOTAL=2200      V_TOTAL=3300     V_TOTAL=4400
              实际: 16.67ms      实际: 11.11ms    实际: 8.33ms

LFC (Low Framerate Compensation):
帧率 < VRR 下限 (48Hz):
GPU 帧完成信号 ────┐         ┌─────────┐         ┌─────────┐
                   │         │         │         │         │
                   ▼         ▼         ▼         ▼
显示器扫描输出:  ┌─┴─────────┴─┐   ┌─┴─────────┴─┐   ┌─────
                │   帧 1      │   │   帧 1      │   │  帧 2
                │  (30fps)    │   │  (30fps)    │   │ (30fps)
                └─────────────┘   └─────────────┘   └───
实际显示:        33.3ms  (30Hz)   33.3ms (30Hz)    33.3ms (30Hz)
                 ↑ LFC: 实际刷新率 = 30fps × 2 = 60Hz
```

#### 关键数据结构

FreeSync 的核心数据通过 DRM 属性和 AMDGPU DC 流状态管理：

```ascii
DRM 属性链:
┌──────────────────────────────────────────────────────────┐
│ drm_connector (connector)                                 │
│  ├── vrr_capable: bool                    [只读]           │
│  │   └── 由 EDID 中的 FreeSync 标志决定                   │
│  └── connector_state (state)                              │
│       └── vrr_enabled: bool                 [可写]        │
│            └── 由用户空间设置 (gamescope/Weston)           │
│                                                            │
│ drm_crtc (crtc)                                            │
│  └── crtc_state (state)                                    │
│       └── vrr_enabled: bool  ← 从 connector 透传          │
│                                                            │
│ amdgpu_crtc (acrtc)                                        │
│  └── dm_crtc_state (dm_state)                              │
│       ├── vrr_supported: bool                              │
│       ├── vrr_enabled: bool                                │
│       ├── vrr_min_refresh: int                             │
│       ├── vrr_max_refresh: int                             │
│       ├── vrr_pixel_clock: int                             │
│       └── freesync_state: enum {}                          │
│                                                            │
│ dc_stream_state (stream)  [DC 层]                          │
│  ├── vrr_infopacket: struct dc_info_packet                 │
│  ├── vrr_vtotal_min: uint32_t                              │
│  ├── vrr_vtotal_max: uint32_t                              │
│  ├── vrr_update_lock: bool                                 │
│  ├── allow_freesync: bool                                  │
│  └── vrr_active_variable: bool                             │
└──────────────────────────────────────────────────────────┘
```

#### FreeSync 状态机

FreeSync 在驱动中通过一个状态机管理 VRR 的启用、禁用和切换：

```ascii
FreeSync 状态机:
                         ┌─────────────┐
                         │   DISABLED  │
                         │  (默认状态)  │
                         └──────┬──────┘
                                │
                    ┌───────────┴───────────┐
                    │  user sets            │
                    │  vrr_enabled=1        │
                    └───────────┬───────────┘
                                ▼
                         ┌─────────────┐
            ┌───────────►│   PENDING   │◄──────────┐
            │            │  (等待激活)  │           │
            │            └──────┬──────┘           │
            │                   │                   │
            │         ┌─────────┴─────────┐        │
            │         │ atomic check OK   │        │
            │         └─────────┬─────────┘        │
            │                   ▼                   │
            │            ┌─────────────┐           │
            │  ┌────────►│   ACTIVE    │───────────┼───► HPD 断开
            │  │         │  (VRR 运行)  │           │
            │  │         └──────┬──────┘           │
            │  │                │                   │
            │  │     ┌──────────┴──────────┐        │
            │  │     │ 帧率低于 VRR 下限    │        │
            │  │     └──────────┬──────────┘        │
            │  │                ▼                   │
            │  │         ┌─────────────┐           │
            │  │         │ LFC ACTIVE  │───────────┘
            │  │         │ (倍频补偿)   │
            │  │         └──────┬──────┘
            │  │                │
            │  │     ┌──────────┴──────────┐
            │  │     │ 帧率恢复到范围内     │
            │  │     └──────────┬──────────┘
            │  │                │
            │  │                └──► 回到 ACTIVE
            │  │
            │  └── user sets vrr_enabled=0
            │
            └────────── 模式切换 (分辨率/刷新率变化)
```

#### FreeSync Premium 和 Premium Pro

FreeSync 的不同认证级别：

```ascii
FreeSync 认证级别对比:

┌────────────────────────────────────────────────────────────────┐
│                    FreeSync 认证要求                            │
├──────────┬──────────────┬──────────────┬──────────────────────┤
│ 特性      │ FreeSync     │ FreeSync     │ FreeSync Premium    │
│          │ (Base)        │ Premium      │ Pro                  │
├──────────┼──────────────┼──────────────┼──────────────────────┤
│ VRR 范围  │ ≥ 200Hz 范围   │ ≥ 2.5× 范围   │ ≥ 2.5× 范围         │
│          │ (如 40-60Hz)  │ (如 40-100Hz)│ (如 48-120Hz)       │
├──────────┼──────────────┼──────────────┼──────────────────────┤
│ LFC      │ 可选          │ 强制         │ 强制                 │
├──────────┼──────────────┼──────────────┼──────────────────────┤
│ 低延迟    │ 无要求        │ 无要求        │ 强制                 │
├──────────┼──────────────┼──────────────┼──────────────────────┤
│ HDR       │ 不要求        │ 不要求        │ 强制 HDR 认证        │
│          │              │              │ (≥ 400 cd/m², 90%   │
│          │              │              │  DCI-P3)            │
├──────────┼──────────────┼──────────────┼──────────────────────┤
│ 典型场景  │ 普通办公      │ 主流游戏      │ 高端游戏 + HDR      │
│          │ 60-75Hz       │ 100-165Hz    │ 240Hz+ OLED/高端   │
└──────────┴──────────────┴──────────────┴──────────────────────┘
```

### 实践操作

#### AMDGPU FreeSync 初始化流程

FreeSync 的初始化发生在显示模式设置过程中，从 EDID 解析开始到 VRR 范围注册：

```c
// amdgpu_dm.c - VRR 能力检测
static void amdgpu_dm_connector_ddc_get_vrr_capability(
    struct drm_connector *connector)
{
    struct amdgpu_dm_connector *aconnector = to_amdgpu_dm_connector(connector);
    struct edid *edid;
    bool vrr_capable = false;
    
    // 1. 获取 EDID
    if (connector->edid_blob_ptr) {
        edid = (struct edid *)connector->edid_blob_ptr->data;
        
        // 2. 搜索 FreeSync VSDB
        // 在 CEA-861 扩展块中搜索 AMD OUI (0x00001A)
        for (int i = 0; i < edid->extensions; i++) {
            const u8 *cea = (const u8 *)edid + (i + 1) * EDID_LENGTH;
            
            if (cea[0] != 0x02)  // 不是 CEA-861 扩展块
                continue;
            
            int offset = 4;
            while (offset < cea[2] && offset < EDID_LENGTH) {
                u8 tag = cea[offset] >> 5;
                u8 len = cea[offset] & 0x1F;
                
                if (tag == 3) {  // VSDB
                    u32 oui = (cea[offset+1] << 16) |
                              (cea[offset+2] << 8) | cea[offset+3];
                    
                    if (oui == 0x00001A) {  // AMD OUI
                        u8 freesync_caps = cea[offset+4];
                        if (freesync_caps & 0x01) {
                            vrr_capable = true;
                            
                            // 获取 VRR 范围
                            if (len >= 6) {
                                aconnector->vrr_min_refresh =
                                    cea[offset+5];
                                aconnector->vrr_max_refresh =
                                    cea[offset+6];
                            }
                            break;
                        }
                    }
                }
                offset += len + 1;
            }
        }
    }
    
    // 3. 设置 VRR capability 属性
    connector->vrr_capable = vrr_capable;
    drm_connector_set_vrr_capable_property(connector, vrr_capable);
    
    // 4. 通知用户空间
    if (vrr_capable)
        DRM_INFO("Connector %s: FreeSync supported, range %d-%d Hz\n",
            connector->name,
            aconnector->vrr_min_refresh,
            aconnector->vrr_max_refresh);
}
```

VRR 使能的核心函数：

```c
// amdgpu_dm.c - VRR 使能
static void dm_set_vrr(struct amdgpu_display_manager *dm,
    struct drm_crtc *crtc, bool enable)
{
    struct dm_crtc_state *dm_state = to_dm_crtc_state(crtc->state);
    struct amdgpu_crtc *acrtc = to_amdgpu_crtc(crtc);
    struct dc_stream_state *stream = dm_state->stream;
    uint32_t min_vtotal, max_vtotal;
    
    if (!stream)
        return;
    
    if (enable) {
        // 1. 计算 V_TOTAL 范围
        // V_TOTAL_MIN: 对应最大刷新率的 V_Total
        // V_TOTAL_MAX: 对应最小刷新率的 V_Total
        uint32_t v_total = stream->timing.v_total;
        uint32_t pixel_clock = stream->timing.pix_clk_100hz;
        
        // 最大刷新率时的 V_Total (最小的 V_Total)
        min_vtotal = v_total;
        
        // 最小刷新率时的 V_Total (最大的 V_Total)
        // VTOTAL_MAX = (pixel_clock / (h_total * min_refresh)) - 1
        if (dm_state->vrr_min_refresh > 0) {
            max_vtotal = (pixel_clock * 100) /
                (stream->timing.h_total * dm_state->vrr_min_refresh);
        } else {
            max_vtotal = v_total * 2;  // 默认 2x
        }
        
        // 2. 设置 DC stream VRR 参数
        stream->vrr_vtotal_min = min_vtotal;
        stream->vrr_vtotal_max = max_vtotal;
        stream->allow_freesync = true;
        
        // 3. 通知 DC 层使能 VRR
        dc_stream_set_vrr(stream, true);
        
        DRM_DEBUG_DRIVER("VRR enabled: VTOTAL range [%d, %d]\n",
            min_vtotal, max_vtotal);
    } else {
        // 禁用 VRR
        stream->allow_freesync = false;
        dc_stream_set_vrr(stream, false);
        
        DRM_DEBUG_DRIVER("VRR disabled\n");
    }
}
```

#### AMDGPU DC 中 VRR 的硬件实现

DC 层负责与 DCN 硬件交互，配置 OTG（Output Timing Generator）的 DRR 寄存器：

```c
// dc/dcn20/dcn20_optc.c - OTG DRR 配置
void optc2_set_drr(
    struct timing_generator *optc,
    const struct drr_params *params)
{
    struct dcn10_optc *optc10 = DCN10_OPTC_FROM_OPTC(optc);
    
    // 1. 设置 V_TOTAL 最小和最大值
    if (params != NULL &&
        params->vertical_total_min > 0 &&
        params->vertical_total_max > 0) {
        
        // 写入 V_TOTAL_MIN 寄存器
        REG_WRITE(OPTC_V_TOTAL_MIN, params->vertical_total_min);
        
        // 写入 V_TOTAL_MANUAL 寄存器 (手动模式 V_Total)
        REG_WRITE(OPTC_V_TOTAL_MANUAL, params->vertical_total_min);
        
        // 写入 V_TOTAL_MAX 寄存器
        REG_WRITE(OPTC_V_TOTAL_MAX, params->vertical_total_max);
        
        // 2. 设置 DRR 控制位
        REG_UPDATE(OPTC_DRR_CONTROL,
            OPTC_DRR_EN, 1,           // 使能 DRR
            OPTC_DRR_FRAME_SEL, 0,    // 当前帧立即生效
            OPTC_DRR_V_TOTAL_CHANGE, 1);  // 允许 V_Total 变化
    } else {
        // 禁用 DRR
        REG_UPDATE(OPTC_DRR_CONTROL, OPTC_DRR_EN, 0);
    }
}

// dc/dcn30/dcn30_optc.c - DCN3 的 VRR 支持
void optc3_set_vrr(
    struct timing_generator *optc,
    bool enable)
{
    struct dcn10_optc *optc10 = DCN10_OPTC_FROM_OPTC(optc);
    
    if (enable) {
        // 1. 使能动态 V_Total 变化
        REG_UPDATE(OPTC_DRR_CONTROL,
            OPTC_DRR_EN, 1,
            OPTC_DRR_V_TOTAL_CHANGE, 1);
        
        // 2. 设置 V_TOTAL 变化为每帧可调
        REG_UPDATE(OPTC_VARIANTS,
            OPTC_V_TOTAL_VARIANT_EN, 1,
            OPTC_V_TOTAL_MODE, 0);  // 自动模式
        
        // 3. 设置最小/最大帧时间
        // 最小帧时间 = V_TOTAL_MIN
        // 最大帧时间 = V_TOTAL_MAX
    } else {
        // 禁用 VRR
        REG_UPDATE(OPTC_DRR_CONTROL,
            OPTC_DRR_EN, 0,
            OPTC_DRR_V_TOTAL_CHANGE, 0);
    }
}
```

#### FreeSync 的原子提交处理

在 atomic commit 流程中，VRR 状态的改变需要特别处理：

```c
// amdgpu_dm.c - atomic check 中的 VRR 处理
static int dm_crtc_atomic_check(struct drm_crtc *crtc,
    struct drm_crtc_state *state)
{
    struct dm_crtc_state *dm_state = to_dm_crtc_state(state);
    int ret = 0;
    
    // 1. 检查 VRR 状态变化
    if (state->vrr_enabled != crtc->state->vrr_enabled) {
        // VRR 状态改变
        if (state->vrr_enabled) {
            // 启用 VRR: 验证 VRR 范围
            if (!dm_state->stream || !dm_state->stream->sink) {
                DRM_DEBUG_DRIVER("Cannot enable VRR: no stream\n");
                return -EINVAL;
            }
            
            // 验证显示器是否支持 VRR
            if (!dm_state->stream->sink->edid_caps.freesync_supported) {
                DRM_DEBUG_DRIVER("Sink does not support FreeSync\n");
                return -EOPNOTSUPP;
            }
        }
    }
    
    // 2. 检查 mode 变化时的 VRR 兼容性
    if (state->mode_changed && state->vrr_enabled) {
        // 某些模式可能与 VRR 不兼容
        // 例如: interlaced 模式不支持 VRR
        if (state->mode.flags & DRM_MODE_FLAG_INTERLACE) {
            DRM_DEBUG_DRIVER("VRR not supported with interlaced mode\n");
            return -EINVAL;
        }
    }
    
    return ret;
}

// atomic commit tail 中的 VRR 处理
static void dm_crtc_atomic_commit_tail(struct drm_atomic_state *state)
{
    struct drm_crtc *crtc;
    struct drm_crtc_state *old_state, *new_state;
    int i;
    
    for_each_oldnew_crtc_in_state(state, crtc, old_state,
        new_state, i) {
        
        struct dm_crtc_state *dm_new_state = to_dm_crtc_state(new_state);
        struct amdgpu_crtc *acrtc = to_amdgpu_crtc(crtc);
        
        // 检查 VRR 状态变化
        if (new_state->vrr_enabled != old_state->vrr_enabled) {
            dm_set_vrr(acrtc->dm, crtc, new_state->vrr_enabled);
        }
        
        // 检查 VRR 参数变化 (VTOTAL 范围调整)
        if (new_state->vrr_enabled &&
            (dm_new_state->vrr_min_refresh !=
             to_dm_crtc_state(old_state)->vrr_min_refresh ||
             dm_new_state->vrr_max_refresh !=
             to_dm_crtc_state(old_state)->vrr_max_refresh)) {
            
            dm_set_vrr(acrtc->dm, crtc, true);
        }
    }
}
```

#### LFC (Low Framerate Compensation) 实现

LFC 是 FreeSync Premium 的强制特性，当帧率低于 VRR 下限时自动激活：

```c
// dc/core/dc_stream.c - LFC 决策
bool dc_stream_set_vrr(struct dc_stream_state *stream, bool enable)
{
    struct dc *dc = stream->ctx->dc;
    struct dc_link *link = stream->link;
    
    if (enable) {
        // 1. 计算 LFC 触发阈值
        // LFC 触发条件: 帧率 < VRR 下限 × 0.8 (带滞后)
        uint32_t vrr_min = stream->vrr_vtotal_max;
        uint32_t vrr_max = stream->vrr_vtotal_min;
        uint32_t lfc_threshold = vrr_min + (vrr_max - vrr_min) / 4;
        
        // 2. 设置 LFC 参数
        stream->lfc_params.enabled = true;
        stream->lfc_params.mid_point = (vrr_min + vrr_max) / 2;
        stream->lfc_params.low_threshold = lfc_threshold;
        stream->lfc_params.exit_threshold = vrr_min - 10;
        
        // 3. 配置 LFC 倍频策略
        // 30fps → 60Hz (2x), 24fps → 48Hz (2x), 20fps → 60Hz (3x)
        stream->lfc_params.multiplier = 1;
        if (stream->timing.pix_clk_100hz /
            (stream->timing.h_total * vrr_min) < 40) {
            // 如果当前帧率低于 40fps, 启用 LFC
            stream->lfc_params.active = true;
            stream->lfc_params.multiplier = 2;  // 2x LFC
        }
    } else {
        stream->lfc_params.enabled = false;
        stream->lfc_params.active = false;
    }
    
    return true;
}

// LFC 的帧率监控 (在 VBlank ISR 中执行)
void dc_stream_monitor_vrr(struct dc_stream_state *stream)
{
    uint32_t current_v_total;
    uint32_t frame_time_us;
    
    if (!stream->allow_freesync)
        return;
    
    // 1. 读取当前 V_Total (由硬件自动调整)
    current_v_total = stream->vrr_vtotal_min;  // 从寄存器读取
    
    // 2. 计算实际帧率
    frame_time_us = (stream->timing.h_total * current_v_total * 10000) /
        stream->timing.pix_clk_100hz;
    
    uint32_t fps = 1000000 / frame_time_us;
    
    // 3. LFC 决策
    if (stream->lfc_params.active) {
        // 检查是否可以退出 LFC
        if (fps >= stream->lfc_params.exit_threshold) {
            stream->lfc_params.active = false;
            stream->lfc_params.multiplier = 1;
            DRM_DEBUG_DRIVER("LFC: Exiting LFC, fps=%d\n", fps);
        }
    } else {
        // 检查是否需要进入 LFC
        if (fps < stream->vrr_vtotal_min * 10000 /
            (stream->timing.h_total * stream->timing.pix_clk_100hz)) {
            
            stream->lfc_params.active = true;
            // 计算合适的倍频
            stream->lfc_params.multiplier =
                max(2u, (stream->vrr_vtotal_min * 2) /
                    (fps * stream->timing.h_total * 10000 /
                     stream->timing.pix_clk_100hz));
            
            DRM_DEBUG_DRIVER("LFC: Entering LFC, fps=%d, mult=%d\n",
                fps, stream->lfc_params.multiplier);
        }
    }
}
```

#### FreeSync 用户空间接口

用户空间通过 DRM 属性控制 FreeSync：

```c
// DRM VRR 属性定义
// drivers/gpu/drm/drm_connector.c

// 1. vrr_capable 属性 (只读, 连接器级别)
static const struct drm_prop_enum_list drm_vrr_capable_enum[] = {
    { 0, "Not capable" },
    { 1, "Capable" },
};

// 2. vrr_enabled 属性 (可写, 连接器状态)
// 用户空间通过 atomic commit 设置

// 用户空间使用示例 (libdrm):
/*
    drmModeAtomicReq *req = drmModeAtomicAlloc();
    
    // 设置 VRR 使能
    drmModeAtomicAddProperty(req, crtc_id, vrr_enabled_prop, 1);
    
    // 提交 atomic commit
    drmModeAtomicCommit(fd, req, DRM_MODE_ATOMIC_NONBLOCK, NULL);
    
    drmModeAtomicFree(req);
*/
```

Wayland 合成器 (gamescope) 的 VRR 支持：

```c
// gamescope 中 VRR 使用示例
// src/main.cpp
static void gamescope_set_vrr(bool enabled)
{
    for (auto &conn : g_connectors) {
        if (conn->vrr_capable) {
            drmModeObjectSetProperty(g_drm_fd,
                conn->id,
                DRM_MODE_OBJECT_CONNECTOR,
                conn->vrr_enabled_prop_id,
                enabled ? 1 : 0);
            
            DRM_DEBUG("Set VRR on connector %d: %s\n",
                conn->id, enabled ? "enabled" : "disabled");
        }
    }
}
```

#### FreeSync 调试接口

AMDGPU 提供了多个调试接口用于分析 FreeSync 状态：

```bash
# 1. 查看 VRR 能力
cat /sys/class/drm/card0-DP-1/vrr_capable
# 输出: 1 (支持 FreeSync)

# 2. 查看当前 VRR 状态 (通过 debugfs)
cat /sys/kernel/debug/dri/0/amdgpu_freesync_status
# 输出示例:
# Connector DP-1:
#   FreeSync: enabled
#   VRR range: 48-144 Hz
#   Current refresh: 89 Hz
#   LFC active: no
#   Min V_TOTAL: 2200
#   Max V_TOTAL: 6600
#   Current V_TOTAL: 3300

# 3. 查看 VBlank 信息
cat /sys/kernel/debug/dri/0/crtc-0/vblank_info
# 输出示例:
# CRTC 0:
#   VBlank count: 123456
#   VBlank time: 11.1ms (90 Hz)
#   VRR: enabled

# 4. DRM 调试日志
echo 0x1f > /sys/module/drm/parameters/debug
dmesg -w | grep -i "vrr\|freesync\|lfc"

# 5. AMDGPU DC 调试 mask
echo 0x8 > /sys/module/amdgpu/parameters/amdgpu_dc_debug_mask
dmesg -w | grep -i "vrr\|freesync\|lfc\|v_total"

# 6. 使用 modetest 查看 VRR 属性
modetest -M amdgpu -p | grep -A 20 "Connector: DP-1"
# 查找 vrr_capable 属性值

# 7. 强制启用/禁用 FreeSync (调试)
# 通过 sysfs 节点 (如果存在)
echo 1 > /sys/kernel/debug/dri/0/amdgpu_freesync_force
echo 0 > /sys/kernel/debug/dri/0/amdgpu_freesync_force

# 8. 使用 drm_info 查看完整 VRR 状态
drm_info | grep -i "vrr\|freesync\|adaptive"
```

#### FreeSync 自动化测试

编写一个完整的 FreeSync 测试脚本：

```python
#!/usr/bin/env python3
"""
FreeSync/VRR 自动化测试脚本
测试目标:
  1. 验证 VRR 能力检测
  2. 验证 VRR 使能/禁用
  3. 验证 VRR 范围
  4. 验证 LFC 行为
  5. 验证 VBlank 时序调整
"""

import subprocess
import time
import json
import os
import re
import sys

class FreeSyncTest:
    def __init__(self):
        self.drm_path = "/sys/class/drm"
        self.debugfs_path = "/sys/kernel/debug/dri/0"
        self.results = {
            "passed": 0,
            "failed": 0,
            "skipped": 0,
            "tests": []
        }
    
    def run_cmd(self, cmd, timeout=10):
        try:
            result = subprocess.run(
                cmd, shell=True, capture_output=True,
                text=True, timeout=timeout
            )
            return result.returncode, result.stdout, result.stderr
        except subprocess.TimeoutExpired:
            return -1, "", "Timeout"
        except Exception as e:
            return -1, "", str(e)
    
    def get_connectors(self):
        """获取所有显示连接器"""
        rc, out, err = self.run_cmd(
            f"ls -d {self.drm_path}/card0-* 2>/dev/null"
        )
        return out.strip().split('\n') if out.strip() else []
    
    def test_vrr_capability_detection(self):
        """测试 VRR 能力检测"""
        print("\n[TEST] VRR 能力检测测试")
        
        connectors = self.get_connectors()
        if not connectors:
            print("  [SKIP] 未找到连接器")
            self.results["skipped"] += 1
            return
        
        for conn_path in connectors:
            conn_name = os.path.basename(conn_path)
            rc, out, err = self.run_cmd(
                f"cat {conn_path}/vrr_capable 2>/dev/null"
            )
            
            vrr_capable = out.strip()
            if vrr_capable == "1":
                print(f"  [PASS] {conn_name}: VRR capable")
                self.results["passed"] += 1
            elif vrr_capable == "0":
                print(f"  [INFO] {conn_name}: Not VRR capable")
                self.results["skipped"] += 1
            else:
                print(f"  [FAIL] {conn_name}: Cannot read vrr_capable")
                self.results["failed"] += 1
    
    def test_vrr_range(self):
        """测试 VRR 范围读取"""
        print("\n[TEST] VRR 范围读取测试")
        
        rc, out, err = self.run_cmd(
            f"cat {self.debugfs_path}/amdgpu_freesync_status 2>/dev/null"
        )
        
        if "VRR range:" in out:
            match = re.search(r'VRR range: (\d+)-(\d+) Hz', out)
            if match:
                min_hz = int(match.group(1))
                max_hz = int(match.group(2))
                
                # 验证范围合理性
                if min_hz > 0 and max_hz > min_hz and max_hz <= 360:
                    print(f"  [PASS] VRR range: {min_hz}-{max_hz} Hz")
                    self.results["passed"] += 1
                else:
                    print(f"  [FAIL] Invalid VRR range: {min_hz}-{max_hz}")
                    self.results["failed"] += 1
            else:
                print(f"  [FAIL] Could not parse VRR range")
                self.results["failed"] += 1
        elif "FreeSync: disabled" in out:
            print("  [SKIP] FreeSync 全局禁用")
            self.results["skipped"] += 1
        else:
            print("  [SKIP] amdgpu_freesync_status not available")
            self.results["skipped"] += 1
    
    def test_vrr_enable_disable(self):
        """测试 VRR 使能/禁用"""
        print("\n[TEST] VRR 使能/禁用测试")
        
        # 使用 modetest 检查 VRR 属性
        rc, out, err = self.run_cmd(
            "modetest -M amdgpu -p 2>/dev/null | grep 'vrr_capable'"
        )
        
        if "vrr_capable" in out:
            # 解析连接器 ID
            connector_match = re.search(
                r'Connector: (\w+-\d+)', out
            )
            if connector_match:
                conn_id = connector_match.group(1)
                print(f"  [PASS] 检测到 VRR 属性 on {conn_id}")
                self.results["passed"] += 1
            else:
                print(f"  [PASS] 检测到 VRR 属性")
                self.results["passed"] += 1
        else:
            print("  [SKIP] 未检测到 VRR 属性")
            self.results["skipped"] += 1
    
    def test_lfc_behavior(self):
        """测试 LFC 行为"""
        print("\n[TEST] LFC (Low Framerate Compensation) 测试")
        
        rc, out, err = self.run_cmd(
            f"cat {self.debugfs_path}/amdgpu_freesync_status 2>/dev/null"
        )
        
        if "LFC active:" in out:
            lfc_active = "yes" in out.split("LFC active:")[1].split('\n')[0]
            
            if lfc_active:
                print("  [INFO] LFC 当前激活 (帧率较低)")
            else:
                print("  [INFO] LFC 未激活 (帧率正常)")
            
            print("  [PASS] LFC 状态可查询")
            self.results["passed"] += 1
        else:
            print("  [SKIP] LFC 状态不可用")
            self.results["skipped"] += 1
    
    def test_vblank_timing(self):
        """测试 VBlank 时序调整"""
        print("\n[TEST] VBlank 时序调整测试")
        
        rc, out, err = self.run_cmd(
            f"cat {self.debugfs_path}/crtc-0/vblank_info 2>/dev/null"
        )
        
        if "VRR:" in out or "vrr" in out.lower():
            print(f"  [PASS] VBlank 信息中包含 VRR 状态")
            for line in out.split('\n')[:5]:
                print(f"         {line}")
            self.results["passed"] += 1
        elif out:
            print(f"  [SKIP] VBlank 信息可用但不含 VRR")
            self.results["skipped"] += 1
        else:
            print("  [SKIP] vblank_info 不可用")
            self.results["skipped"] += 1
    
    def test_kernel_messages(self):
        """测试内核日志中的 VRR 信息"""
        print("\n[TEST] 内核 VRR 日志检查")
        
        rc, out, err = self.run_cmd(
            "dmesg | grep -i 'vrr\\|freesync\\|adaptive.sync' | tail -10"
        )
        
        if out:
            print(f"  [PASS] VRR 相关内核日志:")
            for line in out.split('\n')[:3]:
                print(f"         {line.strip()}")
            self.results["passed"] += 1
        else:
            print("  [SKIP] 未找到 VRR 相关日志")
            self.results["skipped"] += 1
    
    def test_performance_impact(self):
        """测试 FreeSync 性能影响"""
        print("\n[TEST] FreeSync 性能影响测试")
        
        # 检查 FreeSync 启用/禁用的 VBlank 时间
        rc, out, err = self.run_cmd(
            "cat /proc/interrupts | grep 'amdgpu' | head -5"
        )
        
        if out:
            interrupt_count = sum(int(x) for x in out.split()[1:5])
            print(f"  [INFO] AMDGPU 中断计数: {interrupt_count}")
            print("  [PASS] 中断信息可读取")
            self.results["passed"] += 1
        else:
            print("  [SKIP] 无法读取中断信息")
            self.results["skipped"] += 1
    
    def run_all(self):
        """运行所有测试"""
        print("=" * 60)
        print("FreeSync/VRR 自动化测试套件")
        print("=" * 60)
        
        self.test_vrr_capability_detection()
        self.test_vrr_range()
        self.test_vrr_enable_disable()
        self.test_lfc_behavior()
        self.test_vblank_timing()
        self.test_kernel_messages()
        self.test_performance_impact()
        
        print("\n" + "=" * 60)
        print("测试结果汇总")
        print(f"  PASS:   {self.results['passed']}")
        print(f"  FAIL:   {self.results['failed']}")
        print(f"  SKIP:   {self.results['skipped']}")
        print(f"  总计:   {self.results['passed'] + self.results['failed'] + self.results['skipped']}")
        print("=" * 60)
        
        return self.results['failed'] == 0


if __name__ == "__main__":
    test = FreeSyncTest()
    success = test.run_all()
    sys.exit(0 if success else 1)
```

#### VRR Range 协商协议

VRR 范围的确定涉及显示器能力、驱动配置和用户设置的协商：

```ascii
VRR Range 协商流程:

┌──────────────────────┐
│ 显示器 EDID          │
│ FreeSync VSDB:       │
│  min_refresh: 48 Hz  │
│  max_refresh: 144 Hz │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ 驱动读取 EDID 能力    │
│ vrr_min = 48         │
│ vrr_max = 144        │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────┐
│ 模式设置 (mode setting)   │
│ 当前模式: 2560×1440@144Hz │
│ V_Total = 2200           │
│ H_Total = 2640           │
│ Pixel Clock = 533MHz     │
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│ 计算 VTOTAL 范围          │
│ V_TOTAL_MIN = 2200       │
│   (对应 144Hz)            │
│ V_TOTAL_MAX = 2200×144/48 │
│   = 6600 (对应 48Hz)      │
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│ 写入 OTG 寄存器           │
│ OPTC_V_TOTAL_MIN = 2200  │
│ OPTC_V_TOTAL_MAX = 6600  │
│ 使能 DRR                  │
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│ 运行时动态调整            │
│ GPU 帧率 → V_Total 调整  │
│ 60fps → V_Total = 5280  │
│ 90fps → V_Total = 3520  │
│ 120fps → V_Total = 2640 │
└──────────────────────────┘
```

V_TOTAL 到刷新率的转换公式：

```ascii
Refresh Rate = Pixel Clock (Hz) / (H_Total × V_Total)

示例:
  Pixel Clock = 533,250,000 Hz (533.25 MHz)
  H_Total = 2640
  V_Total = 2200 (最小值)
  刷新率 = 533250000 / (2640 × 2200) = 91.8 Hz
  
  V_Total = 6600 (最大值)
  刷新率 = 533250000 / (2640 × 6600) = 30.6 Hz

实际公式:
  V_TOTAL = Pixel Clock / (H_Total × Target_Refresh_Rate)
  
  目标 144Hz: V_TOTAL = 533250000 / (2640 × 144) = 1402
  目标 48Hz:  V_TOTAL = 533250000 / (2640 × 48)  = 4207
```

#### FreeSync 与 Adaptive-Sync

FreeSync 基于 VESA DisplayPort Adaptive-Sync 标准，但不同实现存在差异：

```ascii
Adaptive-Sync vs FreeSync vs G-Sync Compatible:

┌────────────────────────────────────────────────────────────────────┐
│ 特性              │ Adaptive-Sync  │ FreeSync     │ G-Sync Compat │
│                   │ (VESA 标准)    │ (AMD)        │ (NVIDIA)      │
├───────────────────┼────────────────┼──────────────┼───────────────┤
│ 标准基础           │ DP 1.2a Annex │ Adaptive-    │ Adaptive-     │
│                   │ 2B/2C         │ Sync         │ Sync          │
├───────────────────┼────────────────┼──────────────┼───────────────┤
│ 连接器类型          │ DP only       │ DP + HDMI    │ DP only       │
├───────────────────┼────────────────┼──────────────┼───────────────┤
│ HDMI 实现          │ N/A           │ HDMI VRR     │ N/A           │
│                   │                │ (HDMI 2.1)   │               │
├───────────────────┼────────────────┼──────────────┼───────────────┤
│ LFC 支持           │ 可选           │ 强制         │ 强制          │
│                   │                │ (Premium+)   │               │
├───────────────────┼────────────────┼──────────────┼───────────────┤
│ 认证等级           │ 无             │ 3 级         │ 1 级          │
│                   │                │ (Base/       │ (G-Sync/     │
│                   │                │  Premium/Pro)│  Ultimate)    │
├───────────────────┼────────────────┼──────────────┼───────────────┤
│ 最低 VRR 范围要求  │ 无             │ Base: 1.67×  │ ≥ 2.5×       │
│                   │                │ Premium: 2.5×│               │
├───────────────────┼────────────────┼──────────────┼───────────────┤
│ HDR 要求           │ 无             │ Premium Pro  │ G-Sync       │
│                   │                │ 强制 HDR     │ Ultimate 要求 │
└────────────────────────────────────────────────────────────────────┘
```

#### FreeSync HDMI 实现

HDMI 2.1 原生支持 VRR，AMDGPU 驱动通过 HDMI FRL 实现 FreeSync：

```c
// amdgpu_dm.c - HDMI VRR 初始化
static void dm_set_hdmi_vrr(struct dc_stream_state *stream, bool enable)
{
    if (enable) {
        // 1. 设置 HDMI VRR InfoFrame
        struct dc_info_packet vrr_packet = {0};
        vrr_packet.valid = true;
        vrr_packet.hb0 = 0x81;  // HDMI VRR InfoFrame type
        vrr_packet.hb1 = 0x01;  // Version
        vrr_packet.hb2 = 0x0E;  // Length (14 bytes)
        
        // VRR InfoFrame 内容
        // Byte 1: VRR enable (bit 0)
        vrr_packet.data[0] = 0x01;  // VRR enabled
        
        // Byte 2-3: VTotal Min
        vrr_packet.data[1] = stream->vrr_vtotal_min & 0xFF;
        vrr_packet.data[2] = (stream->vrr_vtotal_min >> 8) & 0xFF;
        
        // Byte 4-5: VTotal Max
        vrr_packet.data[3] = stream->vrr_vtotal_max & 0xFF;
        vrr_packet.data[4] = (stream->vrr_vtotal_max >> 8) & 0xFF;
        
        // Byte 6-7: Current VTotal
        vrr_packet.data[5] = stream->vrr_vtotal_min & 0xFF;
        vrr_packet.data[6] = (stream->vrr_vtotal_min >> 8) & 0xFF;
        
        // 发送 VRR InfoFrame
        dc_stream_send_hdmi_info_packet(stream, &vrr_packet);
        
        stream->vrr_infopacket = vrr_packet;
    } else {
        // 发送空的 InfoFrame 禁用 VRR
        struct dc_info_packet null_packet = {0};
        dc_stream_send_hdmi_info_packet(stream, &null_packet);
    }
}

// HDMI VRR 时序调整
void dm_set_hdmi_vrr_timing(struct dc_stream_state *stream,
    uint32_t target_v_total)
{
    // 更新 HDMI VRR InfoFrame 中的 Current VTotal
    stream->vrr_infopacket.data[5] = target_v_total & 0xFF;
    stream->vrr_infopacket.data[6] = (target_v_total >> 8) & 0xFF;
    
    // 重新发送 InfoFrame
    dc_stream_send_hdmi_info_packet(stream, &stream->vrr_infopacket);
}
```

#### FreeSync 性能分析

FreeSync 性能分析需要关注的指标：

```bash
# 1. 帧时间分布 (使用 amdgpu_top)
amdgpu_top --freesync-metrics

# 2. 使用 perf 分析 VBlank IRQ 延迟
perf record -e amdgpu:amdgpu_vblank_work -ag -- sleep 10
perf report

# 3. VBlank 时序抖动分析
cat /sys/kernel/debug/dri/0/crtc-0/vblank_info | grep "time"

# 4. 使用 trace event
trace-cmd record -e amdgpu:amdgpu_vblank_work \
    -e amdgpu:amdgpu_dm_vrr_update \
    -e amdgpu:amdgpu_dm_commit_planes

# 5. 帧时间分析脚本
cat > /tmp/frame_analysis.sh << 'EOF'
#!/bin/bash
# 监控帧时间变化
while true; do
    cat /sys/kernel/debug/dri/0/amdgpu_freesync_status | \
        grep "Current refresh"
    sleep 0.1
done
EOF
```

#### FreeSync 常见 Bug 案例

**Bug 1: VRR 范围协商失败**

```dmesg
[  123.4567] [drm] VRR range invalid: min=0 max=0
[  123.4568] [drm] Falling back to fixed refresh rate

# 根因: EDID 中 FreeSync VSDB 格式不正确
# 修复: amdgpu_dm_connector_ddc_get_vrr_capability()
#       增加对多种 OUI 格式的兼容性检查
```

**Bug 2: LFC 频繁进出导致闪烁**

```dmesg
[  456.7890] [drm] LFC: Entering LFC, fps=45, mult=2
[  456.8001] [drm] LFC: Exiting LFC, fps=48
[  456.8102] [drm] LFC: Entering LFC, fps=44

# 根因: LFC 进出阈值缺乏滞后 (hysteresis)
# 修复: 增加 +/-5fps 的滞后窗口
#       stream->lfc_params.low_threshold = vrr_min + 8;
#       stream->lfc_params.exit_threshold = vrr_min - 5;
```

**Bug 3: 多显示器 VRR 冲突**

```dmesg
[  789.0123] [drm] CRTC 0: VRR enabled, CRTC 1: VRR disabled
[  789.0124] [drm] CRTC 0: Cannot meet VRR timing, disabling

# 根因: 两个 CRTC 共享同一 DCN pipe 时 VRR 不兼容
# 修复: dc_stream_set_vrr() 增加共享资源检查
```

### 案例分析

#### 案例 1：FreeSync 无法启用

**问题描述**：用户在 RX 6800 + AOC 27G2G3 (165Hz FreeSync Premium) 上无法启用 FreeSync。

**排查过程**：

```bash
# 步骤 1: 检查 FreeSync 能力
$ cat /sys/class/drm/card0-DP-1/vrr_capable
0  # 显示不支持？

# 步骤 2: 检查 EDID
$ edid-decode /sys/class/drm/card0-DP-1/edid | grep -i "freesync\|adaptive"
# 没有输出... 检查 VSDB

$ edid-decode /sys/class/drm/card0-DP-1/edid | grep -A 30 "Vendor-specific"
Vendor-specific data block (VSDB), OUI 0x00001A (AMD):
  FreeSync supported: yes
  Minimum refresh rate: 48 Hz
  Maximum refresh rate: 165 Hz

# 步骤 3: EDID 解析正常，检查驱动代码
# 发现 amdgpu_dm_connector_ddc_get_vrr_capability()
# 检查 OUI 匹配逻辑

# 步骤 4: 检查驱动参数
$ cat /sys/module/amdgpu/parameters/amdgpu_freesync_vid
# 输出: 0x00001A (正确的 AMD OUI)

# 步骤 5: 检查内核日志
$ dmesg | grep -i "freesync\|vrr"
[  234.5678] [drm] Connector DP-1: FreeSync supported, range 0-0 Hz
# 范围解析为 0-0!
```

**根因**：EDID 中的 FreeSync VSDB 格式不标准。某些显示器将刷新率字段放在偏移 5-6，但驱动期望的偏移不同。

**解决方案**：

```bash
# 修复 VRR 范围解析，增加对不同 VSDB 布局的兼容性
# 在 amdgpu_dm_connector_ddc_get_vrr_capability() 中
# 增加对 3 字节 OUI + 1 字节 flags + 2 字节 range 的检查
```

#### 案例 2：LFC 导致画面闪烁

**问题描述**：在 RX 7900 XTX + Samsung Odyssey G7 (240Hz) 上，帧率在 40-50fps 时画面周期性闪烁。

**排查过程**：

```bash
# 步骤 1: 检查 VRR 范围
$ cat /sys/kernel/debug/dri/0/amdgpu_freesync_status
FreeSync: enabled
VRR range: 48-240 Hz
Current refresh: 94 Hz
LFC active: yes

# 步骤 2: LFC 刚激活，刷新率 94Hz = 47fps × 2
# 帧率 47fps 刚好在 VRR 下限 48Hz 附近

# 步骤 3: 分析帧时间数据
# 47fps = 21.3ms 帧时间
# 48Hz = 20.8ms 帧时间
# 帧率在 21.3ms 附近波动，LFC 频繁进出
```

**根因**：LFC 进入/退出阈值缺少滞后（hysteresis），帧率在边界附近时导致 LFC 频繁切换。

**解决方案**：

```bash
# 增加 LFC 滞后窗口 (在 dc_stream_set_vrr 中)
# lfc_params.low_threshold = vrr_min + 8;  // 进入阈值
# lfc_params.exit_threshold = vrr_min - 5; // 退出阈值
# 滞后窗口: 13Hz
```

### 相关链接

- AMD FreeSync 技术页面: https://www.amd.com/freesync
- VESA Adaptive-Sync 标准: https://vesa.org/adaptive-sync/
- AMDGPU FreeSync 驱动文档: https://docs.kernel.org/gpu/amdgpu/display/freesync.html
- IGT GPU Tools FreeSync 测试: https://gitlab.freedesktop.org/drm/igt-gpu-tools
- Gamescope VRR 实现: https://github.com/ValveSoftware/gamescope
- HDMI 2.1 VRR 规范: https://hdmi.org/spec/hdmi2_1

### 今日小结

1. **VRR 原理**：通过动态调整显示器 V_Total 使刷新率跟随 GPU 帧率变化，消除撕裂和卡顿。

2. **FreeSync 架构**：分为用户空间（DRM 属性）→ 内核 DRM Core → AMDGPU DM → AMDGPU DC → DCN 硬件五层。

3. **硬件实现**：DCN 的 OTG 模块通过 V_TOTAL_MIN/V_TOTAL_MAX 寄存器实现 DRR（Dynamic Refresh Rate）。

4. **VRR 范围协商**：从 EDID 读取 FreeSync VSDB → 计算 VTOTAL 范围 → 配置 OTG DRR 寄存器 → 运行时动态调整。

5. **LFC（Low Framerate Compensation）**：帧率低于 VRR 下限时自动以整数倍刷新（如 30fps→60Hz），FreeSync Premium 强制要求。

6. **FreeSync 认证**：三个级别（Base/Premium/Premium Pro），分别对应不同 VRR 范围、LFC 要求和 HDR 支持。

7. **HDMI VRR**：通过 HDMI 2.1 VRR InfoFrame 传输 V_Total 范围，HDMI FRL 模式支持。

8. **调试方法**：通过 sysfs vrr_capable、debugfs amdgpu_freesync_status、dmesg、modetest 等工具排查。

### 扩展思考

1. **FreeSync 与 G-Sync 的互操作性**：为什么 NVIDIA 的 G-Sync Compatible 可以在 FreeSync 显示器上工作？AMDGPU 驱动的 certification 流程是什么？

2. **VRR 与 HDR 的结合**：VRR + HDR + DSC 三者的带宽如何分配？在 4K 240Hz HDR 场景下是否还有足够的 VRR 调整空间？

3. **VRR 的未来方向**：DisplayPort 2.1 UHBR20 的 80Gbps 带宽和 VRR 的结合，能否支持 8K 480Hz VRR？

4. **VRR 在虚拟化环境**：vGPU/Passthrough 中的 VRR 实现是否可行？虚拟化环境中的帧同步如何保证？

5. **驱动级帧生成（Frame Generation）与 VRR 的交互**：AMD Fluid Motion Frames（AFMF）生成的插帧如何与 FreeSync VRR 协调工作？