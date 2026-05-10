# 第331天：AMD Display Core（DC）架构总览

## 学习目标
- 理解 AMD Display Core（DC）在整个 AMDGPU 显示驱动堆栈中的位置和作用
- 掌握 DC 的核心设计理念：硬件抽象、资源管理、状态机驱动、模块化管道
- 熟悉 DC 的关键数据结构及其关系：dc、dc_stream、dc_sink、dc_link、dc_plane、dc_state
- 理解 DC 的初始化流程和资源池（resource_pool）的构建
- 掌握 DC 的 stream/sink/link 管理机制
- 理解 DC 的验证与提交流程
- 熟悉 DC 的调试接口和 trace 机制

## 知识详解

### 1. 概念原理

#### 1.1 DC 在整个显示驱动堆栈中的位置

AMD Display Core（DC）是 AMDGPU 显示驱动的核心中间层，位于 DRM 框架和 DCN 硬件之间。其设计目标是为上层提供统一的显示抽象，同时屏蔽不同 DCN 硬件版本（DCN 2.0/2.1/3.0/3.1/3.5）的差异。

```
完整的 AMDGPU 显示驱动堆栈：
═══════════════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────────────┐
│                        User Space                                        │
│   xrandr / wayland-compositor / modetest / custom KMS app               │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │ ioctl()
┌──────────────────────────────────┴──────────────────────────────────────┐
│                        DRM Subsystem (drivers/gpu/drm/drm_*)            │
│   drm_mode_setcrtc / drm_atomic_ioctl / drm_mode_addfb                 │
│   drm_crtc_helper_set_config / drm_atomic_helper_check/commit          │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │ driver callbacks
┌──────────────────────────────────┴──────────────────────────────────────┐
│                     AMDGPU DM (Display Manager)                         │
│   drivers/gpu/drm/amd/display/amdgpu_dm/                                │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  amdgpu_dm.c        — DM 初始化、atomic check/commit 入口        │   │
│   │  amdgpu_dm_crtc.c   — CRTC 相关操作（mode set, flip, vblank）  │   │
│   │  amdgpu_dm_plane.c  — Plane 状态管理                           │   │
│   │  amdgpu_dm_connector.c — Connector 检测、EDID 解析             │   │
│   │  amdgpu_dm_irq.c    — 中断处理（vblank, HPD, MST）             │   │
│   │  amdgpu_dm_psr.c    — PSR（Panel Self Refresh）控制            │   │
│   └──────────┬──────────────────────────────────────────────────────┘   │
│              │                                                           │
│              │  (DM → DC 调用: dc_stream_create, dc_commit_state 等)     │
└──────────────┼──────────────────────────────────────────────────────────┘
               │
┌──────────────┴──────────────────────────────────────────────────────────┐
│                 ▲  AMD Display Core (DC)   ▲                             │
│                 ║  drivers/gpu/drm/amd/display/dc/                      ║
│                 ║                                                       ║
│  ┌──────────────║───────────────────────────────────────────────────┐  ║ │
│  │              ║            DC Core / DC Base                      │  ║ │
│  │              ║                                                   │  ║ │
│  │  ┌───────────╨──────────┐  ┌──────────────────┐  ┌────────────┐  │  ║ │
│  │  │ dc.c / dc_stream.c   │  │ dc_sink.c        │  │ dc_link.c  │  │  ║ │
│  │  │ dc_state.c           │  │ dc_link_dp.c     │  │ dc_dmub.c  │  │  ║ │
│  │  └───────────┬──────────┘  └──────────────────┘  └────────────┘  │  ║ │
│  │              │                                                     │  ║ │
│  │  ┌───────────┴──────────┐  ┌──────────────────┐  ┌────────────┐  │  ║ │
│  │  │ Resource Management  │  │ Bandwidth Calc   │  │ GPUV        │  │  ║ │
│  │  │ resource.c           │  │ bw_fixed.c       │  │ dcn10/dcn20 │  │  ║ │
│  │  │ dce_resource.c       │  │ dcn_calc_auto.c  │  │ dcn30/dcn31 │  │  ║ │
│  │  └──────────────────────┘  └──────────────────┘  └────────────┘  │  ║ │
│  └───────────────────────────────────────────────────────────────────┘  ║ │
│                                                                          ║ │
│  ┌───────────────────────────────────────────────────────────────────┐  ║ │
│  │           Hardware Abstraction Layer (dc->hwss)                    │  ║ │
│  │                                                                    │  ║ │
│  │  ┌──────────────────────────────────────────────────────────────┐  │  ║ │
│  │  │  Hardware Sequencer (hw_sequencer)                           │  │  ║ │
│  │  │  struct hw_sequencer_funcs {                                 │  │  ║ │
│  │  │    .apply_ctx_for_surface()       — 应用平面配置到硬件       │  │  ║ │
│  │  │    .program_front_end()           — 编程 DPP/MPC 前端       │  │  ║ │
│  │  │    .program_timing()              — 编程 OTG 时序           │  │  ║ │
│  │  │    .enable_display_power_gating() — 电源门控管理             │  │  ║ │
│  │  │    .update_plane_addr()           — 更新平面地址（flip）    │  │  ║ │
│  │  │    .set_cursor_position()         — 设置光标位置             │  │  ║ │
│  │  │  };                                                           │  │  ║ │
│  │  └──────────────────────────────────────────────────────────────┘  │  ║ │
│  │                                                                    │  ║ │
│  │  ┌──────────────────────────────────────────────────────────────┐  │  ║ │
│  │  │  DCN Version Implementations                                 │  │  ║ │
│  │  │                                                              │  │  ║ │
│  │  │  dcn10_hwseq.c  — DCN 1.0 (Ryzen APU: Raven/Picasso)       │  │  ║ │
│  │  │  dcn20_hwseq.c  — DCN 2.0 (RDNA 1: Navi 10/14)             │  │  ║ │
│  │  │  dcn21_hwseq.c  — DCN 2.1 (RDNA 1.5: Renoir APU)           │  │  ║ │
│  │  │  dcn30_hwseq.c  — DCN 3.0 (RDNA 2: Navi 21/22/23)          │  │  ║ │
│  │  │  dcn31_hwseq.c  — DCN 3.1 (RDNA 2: Navi 24/Vangogh)        │  │  ║ │
│  │  │  dcn301_hwseq.c — DCN 3.01 (RDNA 2: Green Sardine)          │  │  ║ │
│  │  │  dcn32_hwseq.c  — DCN 3.2 (RDNA 3: Navi 31/32/33)          │  │  ║ │
│  │  │  dcn35_hwseq.c  — DCN 3.5 (RDNA 3.5: Strix Point)          │  │  ║ │
│  │  └──────────────────────────────────────────────────────────────┘  │  ║ │
│  └───────────────────────────────────────────────────────────────────┘  ║ │
│                                                                          ║ │
│  ┌───────────────────────────────────────────────────────────────────┐  ║ │
│  │  IP Blocks (硬件 IP 模块实现)                                     │  ║ │
│  │                                                                    │  ║ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         │  ║ │
│  │  │ OTG/OPTC │  │ DPP/SCL  │  │ MPC      │  │ OPP      │         │  ║ │
│  │  │ dcn10_   │  │ dcn10_   │  │ dcn10_   │  │ dcn10_   │         │  ║ │
│  │  │ optc.c    │  │ dpp.c     │  │ mpc.c    │  │ opp.c    │         │  ║ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘         │  ║ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         │  ║ │
│  │  │ DIO/DP   │  │ HUBP     │  │ DCHUB    │  │ DMUB     │         │  ║ │
│  │  │ dcn10_   │  │ dcn10_   │  │ dcn10_   │  │ dmub_    │         │  ║ │
│  │  │ dio.c     │  │ hubp.c   │  │ hub.c    │  │ service.c│         │  ║ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘         │  ║ │
│  └───────────────────────────────────────────────────────────────────┘  ║ │
└──────────────────────────────────────────────────────────────────────────╨──┘
                            │
┌───────────────────────────┴──────────────────────────────────────────────┐
│                       DCN Hardware (ASIC-specific)                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │ OTG      │ │ DPP      │ │ MPC      │ │ OPP      │ │ DIO      │      │
│  │ (Timing) │ │ (Surface)│ │ (Blend)  │ │ (Output) │ │ (PHY)    │      │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘      │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 1.2 DC 的核心设计理念

DC 的设计遵循以下几个核心理念：

**1.2.1 硬件抽象（Hardware Abstraction）**

DC 通过 `dc->hwss`（Hardware Sequencer Function Table）实现硬件抽象。所有 DCN 版本相关的实现都通过函数指针表调用，DC 核心代码不直接依赖任何特定 DCN 版本的寄存器定义。

```c
// dc.h — 核心硬件序列器接口
struct hw_sequencer_funcs {
    // 状态应用
    void (*apply_ctx_for_surface)(
        struct dc *dc,
        const struct dc_stream_state *stream,
        int surface_count);

    // 前端编程（DPP/MPC）
    void (*program_front_end)(
        struct dc *dc,
        struct dc_state *context);

    // 时序编程（OTG）
    void (*program_timing)(
        struct dc *dc,
        struct dc_state *ctx,
        struct dc_stream_state *stream);

    // 电源管理
    void (*enable_display_power_gating)(
        struct dc *dc,
        bool enable);

    // 页面翻转
    void (*update_plane_addr)(
        const struct dc *dc,
        struct dc_state *ctx);

    // 光标
    void (*set_cursor_position)(
        struct dc *dc,
        struct dc_state *context,
        struct dc_stream_state *stream);
};
```

**1.2.2 资源池（Resource Pool）**

DC 使用 Resource Pool 模式管理硬件资源。每个 DCN 版本定义自己的 resource_pool，包含该版本支持的硬件 IP 模块实例。Resource Pool 在 DC 初始化时构建，提供统一的资源查询和分配接口。

```c
// dc_resource.h — 资源池定义
struct resource_pool {
    // 核心显示管道资源
    struct timing_generator *timing_generators[MAX_PIPES];     // OTG 实例
    struct pipe_ctx *pipelines[MAX_PIPES];                     // 显示管道

    // 前端处理资源
    struct dpp *dpps[MAX_PIPES];                               // DPP 实例
    struct hubp *hubps[MAX_PIPES];                             // HUBP 实例

    // 混色与输出资源
    struct mpc *mpc;                                           // MPC 实例
    struct opp *opps[MAX_PIPES];                               // OPP 实例

    // 显示输出接口
    struct link_encoder *engines[MAX_PIPES];                   // 链路编码器
    struct stream_encoder *stream_enc[MAX_PIPES];              // 流编码器
    struct hpo_dp_stream_encoder *hpo_dp_stream_enc[MAX_PIPES]; // HPO DP 编码器

    // DMUB 接口
    struct dmub_hw *dmub;                                      // DMUB 硬件实例

    // 资源计数
    int pipe_count;            // 可用管道数
    int stream_enc_count;      // 流编码器数
    int link_enc_count;        // 链路编码器数
    int timing_generator_count; // 时序生成器数

    // 资源创建函数（DCN 版本特定的工厂函数）
    struct timing_generator *(*create_timing_generator)(
        struct dc_context *ctx,
        uint32_t instance);
    struct dpp *(*create_dpp)(
        struct dc_context *ctx,
        uint32_t instance);
    // ... 其他创建函数
};
```

**1.2.3 状态机驱动（State-driven）**

DC 采用状态机模型管理显示状态。所有配置变更通过 `dc_state` 表示，`dc_commit_streams()` 和 `dc_commit_state()` 负责将新状态应用到硬件。这种模型与 DRM Atomic 框架天然契合。

```c
// dc_state.h — DC 状态定义
struct dc_state {
    // 流状态
    struct dc_stream_state *streams[MAX_PIPES];      // 当前配置的流
    int stream_count;                                // 流数量

    // 平面状态（每个流关联的平面）
    struct dc_surface_count {
        uint8_t count;
        struct dc_plane_state *surfaces[MAX_SURFACE_COUNT];
    } stream_status[MAX_PIPES];

    // 共享资源
    struct dc_resource_context res_ctx;
    struct dc_clock_config clock_config;

    // 带宽验证结果
    struct dc_validation_context bw_ctx;

    // 状态标志
    bool stream_mask;
    bool stream_changed[MAX_PIPES];

    // 调试信息
    uint64_t update_index;        // 更新序号
    uint64_t timestamp_ns;        // 时间戳
};
```

**1.2.4 模块化管道（Modular Pipeline）**

DC 的显示管道由多个可组合的硬件 IP 模块组成：OTG → HUBP → DPP → MPC → OPP → DIO。每个模块都有独立的接口定义和 DCN 版本实现，通过 Resource Pool 中的工厂函数创建。

```
显示管道模块链：
═══════════════════════════════════════════════════════════════════════

             ┌──────────┐
  Memory →   │  DCHUB   │  DCN 数据中心（Data Hub）
             │ dchub    │  - 从 DCHUBBUB 读取数据
             └────┬─────┘
                  │
             ┌────┴─────┐
             │  HUBP    │  HUBP（Hub Pipe）
             │ dcn10_   │  - 像素数据获取单元
             │ hubp.c   │  - 地址生成、tile 计算
             └────┬─────┘  - VM 地址转换
                  │
             ┌────┴─────┐
             │  DPP     │  DPP（Display Pipe and Plane Pipeline）
             │ dcn10_   │  - 缩放（SCL）
             │ dpp.c    │  - 颜色空间转换（CM）
             └────┬─────┘  - 伽马校正（TF/LUT）
                  │
             ┌────┴─────┐
             │  MPC     │  MPC（Multiple Pipe Combined）
             │ dcn10_   │  - 多平面混合（alpha blending）
             │ mpc.c    │  - 背景颜色
             └────┬─────┘  - 输出颜色矩阵（OCSC/OGAM）
                  │
             ┌────┴─────┐
             │  OPP     │  OPP（Output Plane Processor）
             │ dcn10_   │  - 色度采样
             │ opp.c    │  - 后端颜色处理
             └────┬─────┘  - 线缓冲区控制
                  │
             ┌────┴─────┐
             │  OPTC    │  OPTC（Output Timing Controller）= OTG
             │ dcn10_   │  - 时序生成（H/V Total, Sync）
             │ optc.c   │  - VBlank/VStartup 中断
             └────┬─────┘  - 触发信号生成
                  │
             ┌────┴─────┐
             │  DIO     │  DIO（Display I/O）
             │ dcn10_   │  - 流编码器（Stream Encoder）
             │ dio.c    │  - 链路编码器（Link Encoder）
             └────┬─────┘  - AUX/HPD 管理
                  │
             ┌────┴─────┐
             │  PHY     │  PHY（物理层）
             │           │  - TMDS / DP HBR+/USB-C
             └──────────┘
```

#### 1.3 DC 核心数据结构

DC 定义了一套完整的数据结构来表示显示配置。理解这些结构及其关系是掌握 DC 架构的关键。

**1.3.1 struct dc — DC 核心实例**

`struct dc` 是整个 Display Core 的顶层实例。每个 AMDGPU 设备对应一个 `struct dc` 实例，包含所有资源、回调函数和配置信息。

```c
// dc.h — DC 核心结构
struct dc {
    // 上下文和选项
    struct dc_context *ctx;                 // DC 上下文（内存分配、寄存器访问）
    struct dc_options *options;              // DC 配置选项
    struct dc_caps caps;                     // DC 能力（决定支持的显示配置）
    struct dc_debug_data *debug_data;        // 调试数据

    // 当前配置状态
    struct dc_state *current_state;          // 当前硬件状态
    struct dc_stream_state *streams[MAX_PIPES]; // 当前配置的流
    int stream_count;

    // 硬件序列器（DCN 版本特定）
    struct hw_sequencer_funcs hwss;          // 硬件序列器函数表

    // 资源池（DCN 版本特定）
    struct resource_pool *res_pool;          // 资源池

    // 链路管理
    struct dc_link *links[MAX_PIPES * MAX_CONTROLLERS]; // 所有物理链路
    int link_count;

    // 功能标志
    struct dc_debug_options debug;           // 调试控制选项
    struct dc_phy_addr_space_config config;  // 物理地址空间配置

    // 公共接口函数
    void (*hwss_apply_ctx_for_surface)(
        struct dc *dc,
        const struct dc_stream_state *stream,
        int surface_count);

    bool (*stream_adjust_vmin_vmax)(
        struct dc *dc,
        struct dc_stream_state **streams,
        int stream_count);
};

// dc_context.h — DC 上下文
struct dc_context {
    // 驱动上下文
    struct amdgpu_device *driver_context;    // 指向 amdgpu_device
    struct dc *dc;

    // GPIO 控制
    struct gpio_service *gpio_service;

    // 寄存器访问
    struct dm_pp_display_configuration pp_display_cfg;
    bool dce_version_lower_than_2_0;

    // ASIC 信息
    struct dc_bios *dc_bios;                // VBIOS 接口
    enum dce_version dce_version;           // DCE/DCN 版本枚举
    struct dc_firmware_info fw_info;         // 固件信息
    uint32_t asic_id;                       // ASIC ID

    // 时钟与电源
    struct dmub_srv *dmub_srv;              // DMUB 服务
    struct dc_power_power_state power_state;

    // 调试
    struct dc_debug_options debug;
    uint64_t fbc_gpu_addr;                  // FBC GPU 地址
};
```

**1.3.2 struct dc_stream_state — 显示流**

`dc_stream_state` 表示一个完整的显示流，包含时序参数、颜色属性、空间属性等。它是 DC 中最重要的数据结构之一，每个活动的显示输出对应一个 stream。

```c
// dc_stream.h — 显示流定义
struct dc_stream_state {
    // 流标识
    struct dc_sink *sink;                    // 关联的 sink（显示器）
    struct dc_link *link;                    // 关联的物理链路
    int stream_id;                           // 流 ID

    // 时序配置
    struct dc_crtc_timing timing;            // CRTC 时序（核心配置）
    struct dc_crtc_timing_adjust adjust;     // 时序调整（VRR/Adaptive Sync）
    enum signal_type signal;                 // 信号类型（HDMI/DP/eDP）
    enum dc_color_space output_color_space;  // 输出色彩空间

    // 显示属性
    struct dc_hdr_static_metadata hdr_static_metadata; // HDR 静态元数据
    struct dc_info_packet vrr_infopacket;    // VRR InfoFrame
    struct dc_info_packet hdr_infopacket;    // HDR InfoFrame
    struct dc_info_packet gamut_remap;       // 色域映射 InfoFrame

    // 缩放与裁剪
    struct rect src;                         // 源区域（原始图像）
    struct rect dst;                         // 目标区域（显示区域）
    struct scaling_taps scaling;             // 缩放参数（tap 数）

    // 颜色属性
    enum dc_transfer_func output_tf;         // 输出传输函数（gamma）
    struct dc_csc_transform csc_color_matrix; // CSC 颜色矩阵
    enum dc_dither_option dither_option;     // 抖动选项
    bool use_vivid;                          // 高饱和度模式

    // 电源与同步
    bool dpms_off;                           // DPMS 状态
    bool freesync_on_desktop;               // FreeSync 桌面模式
    bool phy_state;                          // PHY 电源状态
    bool ignore_msa_timing_param;            // 忽略 MSA 时序参数

    // 音频
    struct dc_audio *audio;                  // 音频实例
    uint32_t audio_output_speaker;           // 扬声器配置

    // 验证状态
    enum dc_status dpms_status;             // DPMS 状态
    bool mode_changed;                       // mode 是否变更

    // DM 层引用（返回指针）
    struct amdgpu_dm_crtc *dm_crtc;          // 指向 AMDPGU DM CRTC
    void *drm_crtc;                          // 指向 DRM CRTC
};
```

**1.3.3 struct dc_sink — 显示器抽象**

`dc_sink` 表示一个物理或逻辑显示设备。它包含 EDID 解析结果、显示能力、物理尺寸等信息。

```c
// dc_sink.h — 显示器 Sink 定义
struct dc_sink {
    // 标识
    struct dc_link *link;                    // 所属物理链路
    uint32_t sink_id;                        // Sink ID
    enum signal_type sink_signal;            // 信号类型

    // EDID 信息
    struct dc_edid dc_edid;                  // 原始 EDID
    struct dc_edid_caps edid_caps;           // EDID 能力解析结果
    struct display_sink_capabilities sink_caps; // Sink 能力

    // 显示模式
    uint32_t num_graphics_modes;             // 图形模式数量
    struct dc_mode_info *graphics_modes;     // 支持的图形模式列表
    struct dc_mode_flags_list *mode_flags_list;

    // 颜色与 HDR
    enum dc_color_space default_color_space; // 默认色彩空间
    struct dc_color_caps color_caps;         // 颜色能力

    // DSM（Display Stream Management）
    uint32_t dsc_caps;                       // DSC 能力
    bool force_original_format;              // 强制原始格式

    // 电源管理
    bool edid_managed;                       // EDID 已管理
    enum dc_power_state power_state;         // 电源状态

    // 音频
    struct audio_support audio_support;       // 音频支持

    // 引用计数
    uint32_t ref_count;                      // 引用计数
};
```

**1.3.4 struct dc_link — 物理链路**

`dc_link` 表示 GPU 与显示器之间的物理连接。它包含 PHY 配置、链路训练参数、DP MST 拓扑信息等。

```c
// dc_link.h — 物理链路定义
struct dc_link {
    // 标识
    struct dc_context *ctx;                  // DC 上下文
    uint32_t link_index;                     // 链路索引
    uint32_t link_id;                        // 链路 ID（从 VBIOS 获取）
    enum connector_id connector_id;          // Connector 类型（HDMI/DP/eDP）

    // 物理连接
    struct gpio *hpd_gpio;                   // HPD GPIO
    struct ddc_service *ddc;                 // DDC 服务（I2C/AUX）
    enum signal_type connector_signal;       // 接口信号类型

    // 链路训练状态
    struct link_training_settings cur_link_settings;  // 当前链路设置
    struct dc_link_settings preferred_link_setting;   // 首选链路设置
    enum dc_link_rate cur_link_rate;          // 当前链路速率
    uint32_t cur_lane_count;                 // 当前通道数

    // DP 相关信息
    enum dp_panel_mode dpcd_caps;             // DPCD 能力
    bool dpcd_caps_received;                  // DPCD 已接收
    struct dp_audios dp_audio;                // DP 音频
    bool aux_mode;                            // AUX 模式

    // MST 拓扑
    struct mst_mgr mst_mgr;                   // MST 管理器
    struct link_mst_stream_allocation_table mst_stream_allocation; // MST 流分配

    // EDID
    struct dc_sink *local_sink;              // 本地 Sink
    struct dc_sink *sink;                    // 当前活动 Sink
    bool edid_read;                          // EDID 已读取

    // 物理状态
    enum dc_connection_type type;            // 连接类型
    uint32_t dongle_max_pclk_mhz;            // 转接器最大像素时钟
    enum dc_power_state power_state;         // 电源状态

    // 调试
    uint32_t dpcd_pointer;                   // DPCD 指针
    bool skip_training;                      // 跳过链路训练
};
```

**1.3.5 struct dc_plane_state — 平面状态**

`dc_plane_state` 描述一个平面的完整状态，包括 framebuffer 地址、格式、缩放参数、颜色变换等。

```c
// dc_plane.h — 平面状态定义
struct dc_plane_state {
    // 图像源
    struct dc_bufobj *bufobj;                // 缓冲区对象
    uint64_t address;                        // GPU 地址
    uint64_t chroma_address;                 // 色度地址（YUV 格式）

    // 格式与尺寸
    enum surface_pixel_format format;        // 像素格式
    struct plane_size plane_size;            // 平面尺寸
    struct dc_plane_dcc_param dcc;           // DCC（Delta Color Compression）参数
    struct dc_plane_address plane_address;   // 平面地址（带 tile 信息）

    // 裁剪与缩放
    struct rect src_rect;                    // 源矩形（原始图像裁剪，16.16 定点）
    struct rect dst_rect;                    // 目标矩形（显示区域）
    struct rect clip_rect;                   // 裁剪矩形
    struct scaling_taps scaling_quality;     // 缩放质量参数

    // 颜色变换
    struct dc_csc_transform csc_amt;         // CSC 矩阵（coefficient）
    struct dc_transfer_func *in_transfer_func;   // 输入传输函数
    struct dc_transfer_func *gamma_correction;   // Gamma 校正 LUT
    struct dc_gamut_remap gamut_remap;       // 色域映射

    // Alpha 混合
    bool per_pixel_alpha;                    // 逐像素 Alpha
    uint32_t global_alpha;                   // 全局 Alpha 值
    bool global_alpha_value;                 // 全局 Alpha 启用

    // 旋转与镜像
    enum dc_rotation_angle rotation;         // 旋转角度
    bool horizontal_mirror;                  // 水平镜像

    // 立体视觉
    enum dc_stereo_format stereo_format;     // 3D 立体格式
    bool is_right_eye;                       // 右眼

    // 状态
    bool is_immediate_flip;                  // 立即翻转
    bool flip_immediate;                     // 立即翻转完成
    bool visible;                            // 可见性
    struct dc_plane_state *prev;             // 上一个状态（链表）
    struct dc_plane_state *next;             // 下一个状态（链表）

    // 引用计数
    uint32_t ref_count;
};
```

**1.3.6 struct dc_state — 完整状态描述**

`dc_state` 包含所有流和平面状态的快照，表示一次完整的显示配置。DC 通过比较 `current_state` 和新 `dc_state` 来决定需要更新哪些硬件模块。

```c
// dc_state.h — 完整状态定义
struct dc_state {
    // 流列表
    struct dc_stream_state *streams[MAX_PIPES];
    int stream_count;

    // 每个流的平面列表
    struct stream_status {
        int plane_count;
        struct dc_plane_state *plane_states[MAX_SURFACE_COUNT];
    } stream_status[MAX_PIPES];

    // 裁剪区域信息
    struct pipe_ctx res_ctx;                 // 管道上下文
    struct resource_context res_ctx;         // 资源上下文

    // 时钟配置
    struct dc_clock_config clock;            // 时钟配置
    struct dc_clocks clocks;                 // 时钟请求

    // 带宽验证上下文
    struct dc_validation_context bw_ctx;

    // 调试跟踪
    uint64_t update_index;                   // 原子更新序号
    uint64_t timestamp_ns;                   // 状态时间戳

    // 状态变更跟踪
    bool stream_changed[MAX_PIPES];
    uint8_t stream_mask;
};
```

#### 1.4 DC 核心数据模型关系图

```text
                AMD GPU DC 核心数据模型关系
                ═══════════════════════════════

  ┌─────────────────────────────────────────────────────────────────────┐
  │                          struct dc                                │
  │                  Display Core 核心实例                             │
  │                                                                     │
  │  ┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐    │
  │  │  ctx (上下文) │  │  res_pool(资源池) │  │  hwss(硬件序列器) │    │
  │  └──────────────┘  └──────────────────┘  └──────────────────┘    │
  │                                                                     │
  │  ┌──────────────────────────────────────────────────────────┐     │
  │  │                  streams[] (显示流数组)                    │     │
  │  │  0 ────> dc_stream_state ────> dc_sink ────> dc_link    │     │
  │  │  1 ────> dc_stream_state ────> dc_sink ────> dc_link    │     │
  │  │  2 ────> dc_stream_state ────> dc_sink ────> dc_link    │     │
  │  │  ...                                                     │     │
  │  └──────────────────────────────────────────────────────────┘     │
  │                                                                     │
  │  ┌──────────────────────────────────────────────────────────┐     │
  │  │                  links[] (物理链路数组)                    │     │
  │  │  0 ────> dc_link (eDP)     : HPD GPIO / DDC / PHY       │     │
  │  │  1 ────> dc_link (HDMI)    : TMDS / DDC / HPD           │     │
  │  │  2 ────> dc_link (DP)      : AUX / HPD / Lane training  │     │
  │  │  3 ────> dc_link (USB-C)   : DP Alt Mode / PD            │     │
  │  └──────────────────────────────────────────────────────────┘     │
  │                                                                     │
  │  ┌──────────────────────────────────────────────────────────┐     │
  │  │              current_state (当前硬件状态)                 │     │
  │  │  ┌──────────────────────────────────────────────────┐   │     │
  │  │  │  struct dc_state                                  │   │     │
  │  │  │  ├─ streams[]   → stream_state 引用               │   │     │
  │  │  │  ├─ stream_status[] → 每个流的 plane 状态         │   │     │
  │  │  │  ├─ res_ctx     → 管道分配信息                   │   │     │
  │  │  │  ├─ bw_ctx      → 带宽验证结果                   │   │     │
  │  │  │  └─ clocks      → 时钟配置                       │   │     │
  │  │  └──────────────────────────────────────────────────┘   │     │
  │  └──────────────────────────────────────────────────────────┘     │
  └─────────────────────────────────────────────────────────────────────┘

  dc_stream_state 详细结构：
  ┌──────────────────────────────────────────────────────────────────┐
  │  dc_stream_state                                                 │
  │  ├── sink: struct dc_sink *         → 显示器信息                 │
  │  ├── link: struct dc_link *         → 物理链路                    │
  │  ├── timing: struct dc_crtc_timing  → CRTC 时序                  │
  │  │   ├── h_addressable                  → 可见水平像素数          │
  │  │   ├── h_total                        → 水平总像素数            │
  │  │   ├── v_addressable                  → 可见垂直行数            │
  │  │   ├── v_total                        → 垂直总行数              │
  │  │   ├── pix_clk_100hz                  → 像素时钟（100Hz 单位）  │
  │  │   ├── h_sync_width                  → 行同步宽度              │
  │  │   ├── v_sync_width                  → 场同步宽度              │
  │  │   ├── flags                         → 同步极性/隔行等标志     │
  │  │   ├── display_color_depth            → 色彩深度                │
  │  │   ├── pixel_encoding                → 像素编码（RGB/YCbCr）  │
  │  │   └── timing_3d_format              → 3D 格式                  │
  │  ├── signal: enum signal_type           → HDMI/DP/eDP             │
  │  ├── output_color_space                 → 输出色彩空间            │
  │  ├── hdr_static_metadata                → HDR 静态元数据          │
  │  ├── src/dst rect                       → 源/目标区域             │
  │  ├── scaling_taps                       → 缩放参数                │
  │  ├── output_tf                          → 输出传输函数 (gamma)    │
  │  ├── csc_color_matrix                   → CSC 色彩矩阵            │
  │  ├── dpms_off                           → DPMS 状态              │
  │  └── audio                              → 音频                    │
  └──────────────────────────────────────────────────────────────────┘

  dc_plane_state 详细结构：
  ┌──────────────────────────────────────────────────────────────────┐
  │  dc_plane_state                                                  │
  │  ├── address / chroma_address          → GPU 地址                 │
  │  ├── format                             → 像素格式                │
  │  ├── plane_size                         → 平面尺寸                │
  │  ├── dcc                                → DCC 压缩参数            │
  │  ├── src_rect / dst_rect               → 源/目标矩形              │
  │  ├── scaling_quality                    → 缩放质量                │
  │  ├── in_transfer_func / gamma            → 传输函数               │
  │  ├── csc_amt                            → CSC 矩阵                │
  │  ├── gamut_remap                        → 色域映射                │
  │  ├── per_pixel_alpha / global_alpha     → Alpha 混合              │
  │  ├── rotation / mirror                  → 旋转/镜像              │
  │  ├── is_immediate_flip                  → 是否立即翻转            │
  │  └── visible                            → 可见性                  │
  └──────────────────────────────────────────────────────────────────┘
```

### 2. 实践操作

#### 2.1 DC 初始化流程

DC 的初始化在 amdgpu 驱动加载过程中完成。入口函数为 `amdgpu_dc_init()` 或 `dc_create()`。

```c
// dc.c — DC 初始化核心函数
struct dc *dc_create(struct dc_init_data *init_params)
{
    struct dc *dc = kzalloc(sizeof(*dc), GFP_KERNEL);
    if (!dc)
        return NULL;

    // 1. 保存上下文
    dc->ctx = init_params->ctx;
    dc->ctx->dc = dc;

    // 2. 检测硬件版本并创建对应的资源池
    //    根据 dce_version 选择 dcn10/dcn20/dcn30/dcn32 的资源池创建函数
    int ret = dc_detect_version(dc);
    if (ret != 0)
        goto fail;

    // 3. 创建资源池（关键步骤）
    //    dcn20_resource_construct / dcn30_resource_construct 等
    dc->res_pool = dc_create_resource_pool(
        dc,
        init_params->num_virtual_links,
        dc->ctx->dce_version);
    if (!dc->res_pool)
        goto fail;

    // 4. 初始化状态
    dc->current_state = kzalloc(sizeof(*dc->current_state), GFP_KERNEL);
    if (!dc->current_state)
        goto fail;

    // 5. 初始化链路
    dc_init_links(dc);

    // 6. 初始化 DMUB（DMCUB 微控制器通信）
    if (dc->res_pool->dmub != NULL) {
        dc_dmub_init(dc);
    }

    // 7. 设置硬件序列器函数表
    //    hwss 根据 res_pool 的 dcn 版本选择
    dc->hwss = *dc->res_pool->funcs->hwss;

    // 8. 查询硬件能力
    dc_get_caps(dc);

    return dc;

fail:
    kfree(dc);
    return NULL;
}
```

**资源池创建流程**（以 DCN 3.0 为例）：

```c
// dcn30_resource.c — DCN 3.0 资源池创建
static bool dcn30_resource_construct(
    struct dc *dc,
    void *params,
    struct resource_pool *pool)
{
    struct dc_context *ctx = dc->ctx;

    // 1. 创建时序生成器（OTG）
    for (int i = 0; i < pool->pipe_count; i++) {
        pool->timing_generators[i] = dcn30_timing_generator_create(ctx, i);
        if (!pool->timing_generators[i])
            return false;
    }

    // 2. 创建 DPP（Display Pipe Pipeline）
    for (int i = 0; i < pool->pipe_count; i++) {
        pool->dpps[i] = dcn30_dpp_create(ctx, i);
        if (!pool->dpps[i])
            return false;
    }

    // 3. 创建 HUBP（Hub Pipe — 数据获取单元）
    for (int i = 0; i < pool->pipe_count; i++) {
        pool->hubps[i] = dcn30_hubp_create(ctx, i);
        if (!pool->hubps[i])
            return false;
    }

    // 4. 创建 MPC（Multiple Pipe Combined — 混色器）
    pool->mpc = dcn30_mpc_create(ctx);
    if (!pool->mpc)
        return false;

    // 5. 创建 OPP（Output Plane Processor）
    for (int i = 0; i < pool->pipe_count; i++) {
        pool->opps[i] = dcn30_opp_create(ctx, i);
        if (!pool->opps[i])
            return false;
    }

    // 6. 创建流编码器（Stream Encoder）
    for (int i = 0; i < pool->stream_enc_count; i++) {
        pool->stream_enc[i] = dcn30_stream_encoder_create(ctx, i);
        if (!pool->stream_enc[i])
            return false;
    }

    // 7. 创建链路编码器（Link Encoder）
    for (int i = 0; i < pool->link_enc_count; i++) {
        pool->link_encoders[i] = dcn30_link_encoder_create(ctx, i);
        if (!pool->link_encoders[i])
            return false;
    }

    // 8. 设置硬件序列器函数表
    pool->funcs = &dcn30_res_pool_funcs;
    pool->hwss = &dcn30_hwseq_funcs;

    // 9. 设置时钟源
    for (int i = 0; i < pool->clk_src_count; i++) {
        pool->clock_sources[i] = dcn30_clock_source_create(ctx, i);
    }

    return true;
}
```

#### 2.2 Stream 创建与管理

当用户空间请求设置显示模式时，DM 层调用 DC 创建 stream。

```c
// dc_stream.c — Stream 创建
struct dc_stream_state *dc_stream_create(
    const struct dc_crtc_timing *timing,
    struct dc_sink *sink)
{
    struct dc_stream_state *stream;

    // 1. 分配 stream 内存
    stream = kzalloc(sizeof(*stream), GFP_KERNEL);
    if (!stream)
        return NULL;

    // 2. 复制时序配置
    stream->timing = *timing;

    // 3. 关联 sink
    stream->sink = sink;
    dc_sink_retain(sink);

    // 4. 设置默认颜色属性
    stream->output_color_space = COLOR_SPACE_SRGB;
    stream->output_tf = TRANSFER_FUNC_SRGB;

    // 5. 设置默认缩放
    stream->src.x = 0;
    stream->src.y = 0;
    stream->src.width = timing->h_addressable;
    stream->src.height = timing->v_addressable;
    stream->dst = stream->src;
    stream->scaling = (struct scaling_taps){0};

    // 6. 初始化音频
    stream->audio = NULL;

    return stream;
}

// dc_stream.c — Stream 验证
enum dc_status dc_stream_validate(
    struct dc *dc,
    struct dc_stream_state *stream)
{
    // 1. 检查时序参数合法性
    if (!dc_stream_validate_timing(stream->timing))
        return DC_FAIL_BACKEND;

    // 2. 验证链路带宽
    if (!dc_link_validate_dp_timing(dc, stream))
        return DC_FAIL_UNSUPPORTED_1;

    // 3. 验证资源可用性
    if (!dc_resource_validate(dc, stream))
        return DC_NO_CRTC_RESOURCE;

    return DC_OK;
}
```

#### 2.3 DC Commit 流程

当所有流和平面配置完成后，DM 调用 DC 的 commit 函数将配置应用到硬件。

```c
// dc.c — Streams Commit（多条流的批量提交）
bool dc_commit_streams(struct dc *dc, struct dc_state *context)
{
    int i;
    bool ret = true;

    // 1. 准备新状态
    //    复制 stream 引用到 new_state
    struct dc_state *new_state = context;

    // 2. 执行验证
    //    - 检查带宽限制
    //    - 检查资源可用性
    //    - 检查时钟约束
    if (!dc_validate_state(dc, new_state))
        return false;

    // 3. 构建管道配置
    //    - 分配 CRTC/OTG
    //    - 分配 DPP/MPC 管道
    //    - 配置链路编码器
    if (!dc_resource_build_pipeline(dc, new_state))
        return false;

    // 4. 应用状态到硬件
    //    通过 hwss 调用 DCN 版本特定的实现
    dc_commit_state(dc, new_state);

    // 5. 更新 current_state
    dc->current_state = new_state;

    return true;
}

// dc.c — State Commit（核心状态应用）
void dc_commit_state(struct dc *dc, struct dc_state *context)
{
    int i;

    // 1. 更新时钟配置
    //    根据新的显示配置计算所需时钟
    dc_update_clocks(dc, context);

    // 2. 调用硬件序列器应用前端编程
    //    - 编程 DPP（缩放、颜色、格式）
    //    - 编程 MPC（混色、输出矩阵）
    //    - 编程 OPP（输出处理）
    if (dc->hwss.program_front_end)
        dc->hwss.program_front_end(dc, context);

    // 3. 编程 OTG 时序
    //    每个活动的 stream 对应的 OTG
    for (i = 0; i < context->stream_count; i++) {
        struct dc_stream_state *stream = context->streams[i];
        if (dc->hwss.program_timing)
            dc->hwss.program_timing(dc, context, stream);
    }

    // 4. 应用平面配置到每个 stream
    for (i = 0; i < context->stream_count; i++) {
        struct dc_stream_state *stream = context->streams[i];
        int j;
        if (dc->hwss.apply_ctx_for_surface) {
            dc->hwss.apply_ctx_for_surface(
                dc, stream,
                context->stream_status[i].plane_count);
        }
    }

    // 5. 启用的输出
    //    - 启用流编码器
    //    - 启用链路编码器
    //    - 等待 VBlank 同步
    for (i = 0; i < context->stream_count; i++) {
        dc_enable_stream(dc, context->streams[i]);
    }

    // 6. 通知 DMUB（如果支持）
    if (dc->res_pool->dmub) {
        dc_dmub_notify_state(dc, context);
    }
}
```

#### 2.4 带宽验证

DC 在每次 commit 前执行严格的带宽验证，确保硬件有能力支持配置的显示设置。

```c
// dc_resource.c — 带宽验证
bool dc_validate_state(struct dc *dc, struct dc_state *ctx)
{
    int i, j;

    // 1. 验证每个 stream
    for (i = 0; i < ctx->stream_count; i++) {
        struct dc_stream_state *stream = ctx->streams[i];

        // 1.1 验证时序参数
        if (!dc_stream_validate_timing(dc, stream))
            return false;

        // 1.2 验证链路带宽
        if (!dc_link_validate_dp_timing(dc, stream))
            return false;

        // 1.3 验证 DSC（如果有需要）
        if (dc_is_dp_signal(stream->signal)) {
            if (!dc_validate_dsc(dc, stream))
                return false;
        }

        // 1.4 验证每个 plane
        for (j = 0; j < ctx->stream_status[i].plane_count; j++) {
            struct dc_plane_state *plane =
                ctx->stream_status[i].plane_states[j];

            if (!dc_validate_plane(dc, plane))
                return false;
        }
    }

    // 2. 全局带宽验证
    //    计算所有 stream 的总带宽需求
    //    检查是否超过 DCN 内部数据总线带宽
    struct dc_clock_config clock_cfg = {0};
    clock_cfg.min_pix_clk = 0;

    for (i = 0; i < ctx->stream_count; i++) {
        struct dc_stream_state *stream = ctx->streams[i];
        // 累加需要的像素时钟
        clock_cfg.min_pix_clk += stream->timing.pix_clk_100hz / 10;
    }

    // 检查 DCN 时钟限制
    if (!dc_check_clock_limit(dc, &clock_cfg))
        return false;

    // 3. 验证内存带宽
    //    计算 plane 的像素带宽（PPS: Pixels Per Second）
    //    检查是否超过 DCHUB 带宽
    for (i = 0; i < ctx->stream_count; i++) {
        for (j = 0; j < ctx->stream_status[i].plane_count; j++) {
            struct dc_plane_state *plane =
                ctx->stream_status[i].plane_states[j];

            // 检查 DCC 支持和带宽
            if (!dc_validate_dcc(dc, plane))
                return false;
        }
    }

    // 4. 检查 CRTC/OTG 资源
    if (ctx->stream_count > dc->res_pool->timing_generator_count)
        return false;

    return true;
}
```

#### 2.5 调试与跟踪

DC 提供多种调试手段，用于跟踪硬件状态和诊断显示问题。

```c
// dc_debug.c — DC 调试接口
void dc_print_state(struct dc *dc, struct dc_state *state)
{
    int i, j;

    DC_LOG("Display Core State (index=%llu, time=%llu):\n",
           state->update_index, state->timestamp_ns);

    for (i = 0; i < state->stream_count; i++) {
        struct dc_stream_state *stream = state->streams[i];

        DC_LOG("  Stream %d:\n", i);
        DC_LOG("    Timing: %dx%d @ %dHz (pix_clk=%d)\n",
               stream->timing.h_addressable,
               stream->timing.v_addressable,
               stream->timing.pix_clk_100hz / 10000,
               stream->timing.pix_clk_100hz);
        DC_LOG("    Signal: %d, Color: %d\n",
               stream->signal, stream->output_color_space);
        DC_LOG("    Sink: %s\n",
               stream->sink->edid_caps.display_name);

        for (j = 0; j < state->stream_status[i].plane_count; j++) {
            struct dc_plane_state *plane =
                state->stream_status[i].plane_states[j];
            DC_LOG("    Plane %d: format=%d, addr=0x%llx, "
                   "src=(%d,%d,%d,%d) dst=(%d,%d,%d,%d)\n",
                   j, plane->format, plane->address,
                   plane->src_rect.x, plane->src_rect.y,
                   plane->src_rect.width, plane->src_rect.height,
                   plane->dst_rect.x, plane->dst_rect.y,
                   plane->dst_rect.width, plane->dst_rect.height);
        }
    }
}

// 动态调试接口（可通过 sysfs/debugfs 触发）
void dc_dump_clocks(struct dc *dc)
{
    struct dc_clocks *clocks = &dc->current_state->bw_ctx.clk_cfg;

    printk(KERN_DEBUG "DC Clocks:\n");
    printk(KERN_DEBUG "  dcfclk: %d kHz\n", clocks->dcfclk_khz);
    printk(KERN_DEBUG "  fclk:   %d kHz\n", clocks->fclk_khz);
    printk(KERN_DEBUG "  socclk: %d kHz\n", clocks->socclk_khz);
    printk(KERN_DEBUG "  dppclk: %d kHz\n", clocks->dppclk_khz);
    printk(KERN_DEBUG "  dispclk: %d kHz\n", clocks->dispclk_khz);
}
```

**通过 sysfs/debugfs 查看 DC 状态**：

```bash
# DC 调试接口 — debugfs
cat /sys/kernel/debug/dri/0/amdgpu_dm_dc_info
# 输出示例：
# Display Core Info:
#   Version: DCN 3.0.0
#   Pipes: 6
#   Streams: 1 (active: 1)
#   Current State Index: 42

# 查看每个 CRTC 的 DC 状态
cat /sys/kernel/debug/dri/0/crtc-0/state
# 输出示例：
# crtc state:
#   mode: 1920x1080@60
#   active: yes
#   planes: 1
#   plane[0]: format=XR24 addr=0x7fabc0000000

# 查看链路状态
cat /sys/kernel/debug/dri/0/DP-1/status
# 输出示例：
# DP Link Status:
#   Link Rate: HBR2 (5.4 Gbps)
#   Lane Count: 4
#   Training Pattern: 4
#   Clock Recovery: Locked
#   Channel Equalization: Done
#   Symbol Lock: Done

# 查看时钟配置
cat /sys/kernel/debug/dri/0/amdgpu_dm_clocks
# 输出示例：
# DM Clocks:
#   display_clock: 600000 kHz
#   dpp_clock: 600000 kHz
#   soc_clock: 900000 kHz
#   fclk: 1600000 kHz
#   dcfclk: 1200000 kHz
```

#### 2.6 DMUB（Display Micro-Controller Unit Bus）

现代 AMD GPU（DCN 2.0+）包含一个专用的微控制器 DMCUB，用于处理实时显示任务。DC 通过 DMUB 接口与 DMCUB 通信。

```c
// dmub_service.h — DMUB 服务接口
struct dmub_srv {
    // 固件管理
    const struct firmware *fw;               // DMCUB 固件
    uint32_t fw_version;                     // 固件版本

    // 通信通道
    struct dmub_rb flush_queue;              // 刷新队列
    struct dmub_rb notify_queue;             // 通知队列
    struct dmub_abm_cache abm_cache;         // ABM 缓存

    // 寄存器接口
    uint32_t *fb_base;                       // Framebuffer 基址
    uint32_t *fb_offset;                     // Framebuffer 偏移

    // DMCUB 状态
    enum dmub_status status;                 // DMCUB 状态
    bool hw_init;                            // 硬件初始化完成
};

// DC 通过 DMUB 发送命令
bool dc_dmub_send_command(
    struct dc *dc,
    enum dmub_cmd_type cmd_type,
    union dmub_cmd_data *cmd_data)
{
    struct dmub_srv *srv = dc->res_pool->dmub->dmub_srv;

    // 1. 准备命令消息
    struct dmub_msg msg = {
        .type = cmd_type,
        .data = *cmd_data,
        .timestamp = ktime_get_ns(),
    };

    // 2. 写入命令队列
    if (!dmub_srv_send_cmd(srv, &msg))
        return false;

    // 3. 等待 DMCUB 处理完成
    if (!dmub_srv_wait_for_idle(srv, 1000))
        return false;

    return true;
}
```

DMUB 处理的典型命令：
- `DMUB_CMD_SET_TIMING` — 设置 OTG 时序
- `DMUB_CMD_UPDATE_PLANE` — 更新平面地址（页面翻转）
- `DMUB_CMD_SET_POWER_STATE` — 电源状态切换
- `DMUB_CMD_ABM_LEVEL` — 自动亮度控制
- `DMUB_CMD_PSR_ENTER/EXIT` — PSR 进入/退出
- `DMUB_CMD_VRR_UPDATE` — VRR 刷新率范围更新

### 3. 案例分析

#### 3.1 问题：单链路多 stream 配置（MST）

**场景**：通过 DisplayPort MST Hub 连接两台 4K 显示器，但其中一台无法正确显示。

**分析过程**：
```bash
# 查看链路训练状态
cat /sys/kernel/debug/dri/0/DP-1/status
# Link Config: HBR2 4-lane (21.6 Gbps total)

# 查看 MST 拓扑
cat /sys/kernel/debug/dri/0/DP-1/mst_topology
# MST topology:
#   root: DP-1 (rad=1)
#   └─ branch: MST Hub (rad=2)
#      ├─ sink: DELL U2723QE (rad=3)  ← 正常工作
#      └─ sink: LG 27GP950 (rad=4)    ← 无法显示

# 查看 dmesg
dmesg | grep -i "mst\|link\|bandwidth"
# [drm] MST link: rad=4, link bandwidth insufficient
# [drm] DSC required for stream at rad=4
```

**根因分析**：
```c
// DC 带宽计算 — 确定 MST 配置下每个 stream 可用的带宽

// 链路总带宽 = 链路速率 × 通道数 × 编码效率
// HBR2: 5.4 Gbps × 4 lanes × 0.8 (8b/10b) = 17.28 Gbps

// 每个 stream 所需的带宽：
// Stream 1 (DELL U2723QE): 3840x2160@60Hz
//   像素时钟 = 533.25 MHz (CVT-RB)
//   所需带宽 = 533.25 × 30 bpp = 15.9975 Gbps

// Stream 2 (LG 27GP950): 3840x2160@60Hz
//   像素时钟 = 533.25 MHz
//   所需带宽 = 533.25 × 30 bpp = 15.9975 Gbps

// MST 调度：两个 stream 分时共享链路带宽
// 总需求 = 15.9975 + 15.9975 = 31.995 Gbps > 17.28 Gbps (链路限制)

// 解决方案：启用 DSC（Display Stream Compression）
// DSC 3:1 压缩后：15.9975 / 3 = 5.3325 Gbps per stream
// 总需求 = 5.3325 + 5.3325 = 10.665 Gbps < 17.28 Gbps ✓
```

**解决方案**：
```bash
# 强制启用 DSC
echo force_dsc > /sys/kernel/debug/dri/0/amdgpu_dm_dsc_enable

# 或通过 modetest 测试
modetest -M amdgpu -D DP-1 -s 56@40:3840x2160 -d 2>&1 | grep DSC
# DSC: enabled, compression ratio 3:1

# 验证
cat /sys/kernel/debug/dri/0/DP-1/mst_topology
# 现在两个显示器都显示为 active
```

#### 3.2 问题：DC 状态转换失败导致黑屏

**场景**：系统从 suspend 恢复后，外接显示器保持黑屏。

**分析过程**：
```bash
# 检查 DC 状态
cat /sys/kernel/debug/dri/0/amdgpu_dm_dc_info
# Streams: 0 (active: 0)  ← stream 数量为 0

# 检查链路状态
cat /sys/kernel/debug/dri/0/HDMI-A-1/status
# HPD: asserted
# EDID: read successful

# 检查 dmesg
dmesg | grep -i "dc\|resume\|stream"
# [drm] DC resume: starting
# [drm] DC resume: resuming streams
# [drm] DC resume: failed to restore stream state
```

**根因分析**：
```c
// DC suspend/resume 流程

// dc_suspend() — 保存状态并关闭显示
void dc_suspend(struct dc *dc)
{
    // 1. 保存当前状态
    dc->saved_state = dc_snapshot_state(dc);

    // 2. 关闭所有 stream
    for (int i = 0; i < dc->stream_count; i++) {
        dc_disable_stream(dc, dc->streams[i]);
    }

    // 3. 关闭 DMCUB
    dmub_srv_suspend(dc->res_pool->dmub);

    // 4. 电源门控 DCN
    dc_power_down_all(dc);
}

// dc_resume() — 恢复状态
void dc_resume(struct dc *dc)
{
    // 1. 初始化 DMCUB
    dmub_srv_resume(dc->res_pool->dmub);

    // 2. 恢复硬件状态
    dc_restore_hw_state(dc);

    // 3. 恢复 stream — 依赖上层（DM）重新提交
    //    问题：DM 层在 resume 时没有重新调用 dc_commit_streams()
    //    导致 DC 有链路但没有 stream
}

// 修复：DM 层 resume 时需要重新提交 stream
// amdgpu_dm_resume() 中增加：
int amdgpu_dm_resume(struct amdgpu_device *adev)
{
    // ... 其他初始化 ...

    // 重新提交所有之前配置的 stream
    for (int i = 0; i < dc->saved_state->stream_count; i++) {
        dc_stream_retain(dc->saved_state->streams[i]);
    }
    dc_commit_streams(dc, dc->saved_state);

    return 0;
}
```

**解决方案**：内核补丁修复了 DM resume 路径，确保 suspend 前活动的 stream 在 resume 后被重新提交。

#### 3.3 问题：DC 带宽验证失败导致高刷新率不可用

**场景**：用户购买了一台 240Hz 显示器，但只能设置到 144Hz。

**分析过程**：
```bash
# 查看支持的模式
xrandr --output DP-1 --mode 2560x1440 --rate 240
# xrandr: Configure crtc 0 failed

# 查看 dmesg
dmesg | grep -i "mode\|valid\|bandwidth"
# [drm:dc_validate_state] Bandwidth validation failed for mode 2560x1440@240
# [drm:dc_link_validate_dp_timing] Required BW: 42.32 Gbps > Available: 38.69 Gbps
```

**根因分析**：
```c
// DC 带宽计算

// 2560x1440@240Hz 带宽需求：
// 像素时钟（CVT-RB）≈ 2560 × 1440 × 240 × 1.05 ≈ 928.9728 MHz
// 所需数据率 = 928.9728 MHz × 30 bpp (10bpc HDR) = 27.869 Gbps
// 链路层开销（8b/10b 编码）：27.869 / 0.8 = 34.836 Gbps

// 可用带宽（DP 1.4 HBR3 4-lane）：
// HBR3 = 8.1 Gbps per lane
// 4 lanes = 32.4 Gbps
// 实际可用（8b/10b 编码后）= 32.4 × 0.8 = 25.92 Gbps

// 34.836 Gbps > 25.92 Gbps → 验证失败 ❌

// 带宽优化策略：
// 1. 降低色深：10bpc → 8bpc
//    新需求 = 928.9728 × 24 / 0.8 = 27.869 Gbps > 25.92 Gbps ❌

// 2. 启用 DSC 1.2a（~3:1 压缩）
//    压缩后需求 = 27.869 / 3 = 9.29 Gbps < 25.92 Gbps ✓
```

**解决方案**：
```bash
# 1. 检查显示器是否支持 DSC
cat /sys/kernel/debug/dri/0/DP-1/dpcd_caps | grep -i dsc
# DSC: supported, version 1.2

# 2. 启用 DSC（如果驱动没自动启用）
echo 1 > /sys/kernel/debug/dri/0/amdgpu_dm_force_dsc

# 3. 或者降低到 8bpc（如果 DSC 不可用）
xrandr --output DP-1 --set "max bpc" 8

# 4. 或者使用 chroma subsampling
xrandr --output DP-1 --set "output_csc" YCbCr420

# 5. 验证
xrandr --output DP-1 --mode 2560x1440 --rate 240
# 应该在 DSC 启用后成功
```

### 4. 压力测试与批量验证

#### 4.1 DC 状态压力测试

```bash
#!/bin/bash
# dc_stress_test.sh — DC 状态切换压力测试

OUTPUT=$1
STREAMS=("1920x1080@60" "1920x1080@120" "2560x1440@60" "2560x1440@144" "3840x2160@60")

echo "=== DC State Stress Test ==="
echo "Testing output: $OUTPUT"
echo "Iterations: ${#STREAMS[@]}"
echo ""

for config in "${STREAMS[@]}"; do
    resolution=$(echo $config | cut -d@ -f1)
    rate=$(echo $config | cut
    rate=$(echo $config | cut -d@ -f2)
    echo "Testing: $resolution @ ${rate}Hz"

    xrandr --output $OUTPUT --mode $resolution --rate $rate 2>&1
    if [ $? -eq 0 ]; then
        echo "  ✓ PASS: $resolution@${rate}Hz"
    else
        echo "  ✗ FAIL: $resolution@${rate}Hz"
    fi

    sleep 1
done

echo ""
echo "=== Stress test complete ==="
echo "PASS: $(grep -c '✓' /tmp/dc_stress.log 2>/dev/null || echo 0)"
echo "FAIL: $(grep -c '✗' /tmp/dc_stress.log 2>/dev/null || echo 0)"
```

**运行方式**：
```bash
chmod +x dc_stress_test.sh
./dc_stress_test.sh DP-1 2>&1 | tee /tmp/dc_stress.log
```

**测试输出示例**：
```
=== DC State Stress Test ===
Testing output: DP-1
Iterations: 5

Testing: 1920x1080 @ 60Hz
  ✓ PASS: 1920x1080@60Hz
Testing: 1920x1080 @ 120Hz
  ✓ PASS: 1920x1080@120Hz
Testing: 2560x1440 @ 60Hz
  ✓ PASS: 2560x1440@60Hz
Testing: 2560x1440 @ 144Hz
  ✓ PASS: 2560x1440@144Hz
Testing: 3840x2160 @ 60Hz
  ✓ PASS: 3840x2160@60Hz

=== Stress test complete ===
PASS: 5
FAIL: 0
```

#### 4.2 IGT DC 测试

IGT（Intel GPU Tools）提供了大量 KMS/DC 测试用例，可用于验证 DC 各个子模块：

**核心 DC 测试清单**：
```bash
# 1. atomic 模式设置测试
sudo igt@kms_atomic --run-subtest basic
sudo igt@kms_atomic --run-subtest atomic_invalid_params
sudo igt@kms_atomic_transition --run-subtest 1x-modeset

# 2. plane 测试
sudo igt@kms_plane --run-subtest plane-position
sudo igt@kms_plane --run-subtest plane-panning
sudo igt@kms_plane_multiple --run-subtest plane-panning

# 3. color management 测试
sudo igt@kms_color --run-subtest deepe-pipe-*
sudo igt@kms_color --run-subtest gamma

# 4. flip 测试
sudo igt@kms_flip --run-subtest basic-flip-vs-wf_vblank
sudo igt@kms_flip --run-subtest basic-flip-vs-dpms

# 5. VBlank 测试
sudo igt@kms_vblank --run-subtest query
sudo igt@kms_vblank --run-subtest accuracy

# 6. pipe CRC 测试
sudo igt@kms_pipe_crc_basic --run-subtest *
sudo igt@kms_pipe_crc_basic --run-subtest compare-crc

# 7. 属性测试
sudo igt@kms_properties --run-subtest plane-properties
sudo igt@kms_properties --run-subtest connector-properties

# 8. fb 操作测试
sudo igt@kms_addfb_basic --run-subtest addfb25
sudo igt@kms_addfb_basic --run-subtest sizeal

# 9. cursor 测试
sudo igt@kms_cursor_legacy --run-subtest basic
sudo igt@kms_cursor_legacy --run-subtest flip-vs-cursor

# 10. rotation 测试
sudo igt@kms_rotation_crc --run-subtest primary-rotation
sudo igt@kms_rotation_crc --run-subtest sprite-rotation
```

**DC 专用测试（AMDGPU 特有）**：
```bash
# DC 使能状态检查
cat /sys/kernel/debug/dri/0/amdgpu_dm_dc_info

# 每个 pipe 的详细状态
find /sys/kernel/debug/dri/0/ -name "amdgpu_dm_dc_info" -exec cat {} \;

# DC 固件版本检查
cat /sys/kernel/debug/dri/0/amdgpu_firmware_info | grep -i dmub
cat /sys/kernel/debug/dri/0/amdgpu_firmware_info | grep -i sienna

# 所有可用 DC debugfs 入口
ls -la /sys/kernel/debug/dri/0/ | grep -i dc\|dm
```

#### 4.3 DC 诊断脚本

```bash
#!/bin/bash
# dc_diag.sh — DC 架构健康检查脚本

echo "╔══════════════════════════════════════╗"
echo "║     AMD Display Core 诊断工具        ║"
echo "╠══════════════════════════════════════╣"

# 1. 检查 DC 是否启用
echo "║ [1/6] DC 状态检查"
DC_INFO=$(cat /sys/kernel/debug/dri/0/amdgpu_dm_dc_info 2>/dev/null)
if [ -n "$DC_INFO" ]; then
    echo "║   ✓ DC 已使能"
else
    echo "║   ✗ DC 未启用"
fi

# 2. 检查驱动版本
echo "║ [2/6] 驱动版本检查"
DMUB_VER=$(cat /sys/kernel/debug/dri/0/amdgpu_firmware_info 2>/dev/null | grep -i dmub | awk '{print $NF}')
echo "║   DMUB firmware version: ${DMUB_VER:-unknown}"

# 3. 检查连接器状态
echo "║ [3/6] 连接器状态"
for conn in /sys/class/drm/*/status; do
    status=$(cat $conn 2>/dev/null)
    echo "║   $(basename $(dirname $conn)): $status"
done

# 4. 检查 CRTC 分配
echo "║ [4/6] CRTC 分配"
for crtc in /sys/kernel/debug/dri/0/crtc-*/state; do
    echo "║   --- $(basename $(dirname $crtc)) ---"
    cat $crtc 2>/dev/null | head -5 | sed 's/^/║   /'
done

# 5. 检查 plane 状态
echo "║ [5/6] Plane 状态"
for plane in /sys/kernel/debug/dri/0/plane-*/state; do
    echo "║   --- $(basename $(dirname $plane)) ---"
    cat $plane 2>/dev/null | head -3 | sed 's/^/║   /'
done

# 6. 检查带宽压力
echo "║ [6/6] 带宽状态"
for connector in /sys/class/drm/*/edid; do
    base=$(basename $(dirname $connector))
    modes=$(cat $connector 2>/dev/null | edid-decode 2>/dev/null | grep -c "Mode:")
    echo "║   $base: ${modes:-0} modes available"
done

echo "╚══════════════════════════════════════╝"
```

---

### 5. 相关链接

**内核源码（Linux v6.x）**：
1. [AMD DC 主入口](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/core/dc.c)
2. [DC 头文件](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/dc.h)
3. [DC 状态管理](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/core/dc_state.c)
4. [DC Stream 管理](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/core/dc_stream.c)
5. [DC Resource Pool](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/core/dc_resource.c)
6. [DCN30 Resource 构造](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/dcn30/dcn30_resource.c)
7. [DMUB 接口](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dmub/dmub_srv.h)
8. [AMDGPU DM 层](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c)
9. [DC 带宽校验](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/dcn30/dcn30_resource.c#L2745)

**AMD 官方文档**：
10. [AMD GPU Display Stack Overview](https://docs.kernel.org/gpu/amdgpu/display/display-overview.html)
11. [AMD DC 设计文档](https://docs.kernel.org/gpu/amdgpu/display/dc-debug.html)
12. [AMDGPU 参数文档](https://docs.kernel.org/gpu/amdgpu/module-parameters.html)

**工具与测试**：
13. [IGT GPU Tools](https://gitlab.freedesktop.org/drm/igt-gpu-tools)
14. [modetest 源码](https://gitlab.freedesktop.org/mesa/drm/-/blob/main/tests/modetest/modetest.c)
15. [umr — AMD GPU 寄存器调试器](https://gitlab.freedesktop.org/tomstdenis/umr)

---

### 6. 今日小结

**核心要点**：

1. **DC 架构定位**：DC 是 AMD 显示驱动的核心层，位于 DRM 子系统和 DCN 硬件之间，提供统一的显示抽象接口
2. **硬件抽象（hwss）**：通过 `struct hw_sequencer_funcs` 函数指针表实现对不同 DCN 版本的抽象，每个 DCN 版本提供独立的实现
3. **Resource Pool 模式**：`struct resource_pool` 管理所有硬件 IP 模块（OTG、DPP、HUBP、MPC、OPP、DIO）的生命周期
4. **State-Driven 模型**：`struct dc_state` 捕获完整的显示状态快照，支持原子切换、回滚和比较
5. **模块化数据管线**：OTG → HUBP → DPP → MPC → OPP → DIO 的模块化管线，每个模块独立可替换
6. **核心数据结构**：`struct dc`、`dc_stream_state`、`dc_sink`、`dc_link`、`dc_plane_state`、`dc_state` 构成了 DC 层的数据模型骨架
7. **三层初始化**：`dc_create()` → `dcnXX_resource_construct()` → `hwss` 和 `funcs` 函数表赋值的三层初始化流程
8. **四阶段带宽校验**：dc_validate_state() 的 per-stream → pixel clock → memory bandwidth → CRTC resource 四阶段校验
9. **DMUB 协处理器**：DMUB（DMCUB Microcontroller）负责处理时序切换、PSR、ABM 等实时性要求高的任务
10. **调试手段**：通过 `dc_print_state()`、`dc_dump_clocks()`、debugfs 和 IGT 测试全面验证 DC 状态

---

### 7. 扩展思考

1. **设计模式选择**：为什么 DC 选择基于 `dc_state` 的 state-driven 模式而不是类似 DRM 的 atomic commit 加 immediate write 模式？state-driven 模式在错误恢复、回滚和调试方面有什么优势？

2. **Resource Pool vs 工厂模式**：DC 的 resource pool 模式相比传统的工厂模式，在硬件资源管理方面有什么独特优势？为什么要将 OTG、DPP、HUBP 等 IP 模块的创建集中在一个 resource_construct 函数中？

3. **DCN 版本演进**：从 DCN 2.0（Navi10）到 DCN 3.5（RDNA3），DC 内核架构保持了高度稳定。这种稳定性是通过什么机制实现的？对比 Intel 的 display 驱动架构，AMD 的抽象层级设计有什么不同？

4. **DMUB 的设计权衡**：将部分显示控制逻辑卸载到 DMUB 协处理器有什么优缺点？实时性提升是否带来了调试复杂度的显著增加？在 suspend/resume 和 DPMS 场景中，DMUB 的角色是什么？

5. **DSC、FEC 和链路训练**：DC 中链路管理和校验（link training、DSC、FEC）的设计为什么放在了 dc_link 中而不是 dc_stream 中？这种设计对 MST 拓扑有什么影响？

---

### 附录 A：DCN 寄存器地址映射（以 DCN30 为例）

| IP 模块 | 基地址偏移 | 主要寄存器范围 | 功能 |
|---------|-----------|---------------|------|
| OTG0    | 0x6000    | 0x6000-0x65FF | 输出时序生成器 0 |
| OTG1    | 0x6800    | 0x6800-0x6DFF | 输出时序生成器 1 |
| OTG2    | 0x7000    | 0x7000-0x75FF | 输出时序生成器 2 |
| OTG3    | 0x7800    | 0x7800-0x7DFF | 输出时序生成器 3 |
| HUBP0   | 0x4000    | 0x4000-0x43FF | DCHUB 管线 0 |
| HUBP1   | 0x4400    | 0x4400-0x47FF | DCHUB 管线 1 |
| DPP0    | 0x4800    | 0x4800-0x4CFF | Display Pipe Plane 0 |
| DPP1    | 0x4D00    | 0x4D00-0x50FF | Display Pipe Plane 1 |
| MPC0    | 0x3800    | 0x3800-0x3BFF | 多 Pipe 合成器 0 |
| OPP0    | 0x3000    | 0x3000-0x33FF | 后端输出管线 0 |
| DIO0    | 0x5000    | 0x5000-0x53FF | 显示 IO 接口 0 |
| DMUB    | 0x6100    | 0x6100-0x61FF | DMCUB 微控制器 |

### 附录 B：DC 状态机转换图

```
                        ┌─────────────┐
                        │   DC_STATE   │
                        │   RESET      │
                        └──────┬──────┘
                               │ dc_create()
                               ▼
                        ┌─────────────┐
                        │   DC_STATE   │
                        │   INIT       │
                        └──────┬──────┘
                               │ dc_stream_create()
                               ▼
                        ┌─────────────┐
                        │   DC_STATE   │
                        │   STREAM_ADD │
                        └──────┬──────┘
                               │ dc_commit_streams()
                               ▼
            ┌────────────────────────────────────┐
            ▼                                    ▼
    ┌─────────────┐                    ┌──────────────────────┐
    │  DC_STATE    │  dc_commit_state() │   DC_STATE           │
    │  VALIDATE    │──────────────────► │   ACTIVE             │
    └─────────────┘                    └──────────┬───────────┘
            ▲                                     │
            │                                     │ dc_stream_update()
            └─────────────────────────────────────┘
                                                  │
                                                  │ dc_suspend()
                                                  ▼
                                        ┌──────────────────────┐
                                        │   DC_STATE           │
                                        │   SUSPEND            │
                                        └──────────┬───────────┘
                                                   │ dc_resume()
                                                   ▼
                                        ┌──────────────────────┐
                                        │   DC_STATE           │
                                        │   RESUME             │
                                        └──────────┬───────────┘
                                                   │ dc_commit_streams()
                                                   ▼
                                        ┌──────────────────────┐
                                        │   DC_STATE           │
                                        │   ACTIVE             │
                                        └──────────────────────┘
```

### 附录 C：带宽计算速查表

| 分辨率 | 刷新率 | 色深 | Pixel Clock (MHz) | 原始带宽 (Gbps) | DP 1.4 HBR3 | DP 2.0 UHBR10 |
|--------|--------|------|-------------------|----------------|-------------|---------------|
| 1920x1080 | 60  | 8bpc | 148.5 | 4.46 | ✓ | ✓ |
| 1920x1080 | 144 | 8bpc | 356.4 | 10.69 | ✓ | ✓ |
| 2560x1440 | 60  | 8bpc | 241.5 | 7.25 | ✓ | ✓ |
| 2560x1440 | 144 | 8bpc | 579.6 | 17.39 | ✓ | ✓ |
| 2560x1440 | 240 | 8bpc | 966.0 | 28.98 | ✗ → DSC ✓ | ✓ |
| 3840x2160 | 60  | 10bpc | 594.0 | 23.76 | ✓ | ✓ |
| 3840x2160 | 120 | 10bpc | 1188.0 | 47.52 | ✗ → DSC ✓ | ✓ |
| 3840x2160 | 144 | 10bpc | 1425.6 | 57.02 | ✗ | ✗ → DSC ✓ |
| 7680x4320 | 60  | 10bpc | 2376.0 | 95.04 | ✗ | ✗ → DSC ✓ |

*(计算假设：blanking overhead ~20%，8bpc=24bpp，10bpc=30bpp，DSC 压缩比 ~3:1)*

---

*本篇文档共 1800 行，系统介绍了 AMD Display Core（DC）架构总览，涵盖 DC 在驱动堆栈中的定位、核心设计原则、六大数据结构详解、初始化与提交流程、带宽验证、DMUB 接口、真实案例分析以及完整的压力测试和诊断工具。通过本篇的学习，读者应该能够理解 DC 层级的作用、核心数据结构的关系以及显示状态管线的完整生命周期。下一日将深入 DC 源码目录结构，逐层分析 `drivers/gpu/drm/amd/display/` 下各个子目录的作用和关联。*

#### 4.4 DC 状态监控脚本

```bash
#!/bin/bash
# dc_monitor.sh — 实时监控 DC 状态变化

INTERVAL=${1:-2}
echo "Monitoring DC state every ${INTERVAL}s..."
echo "Press Ctrl+C to stop"
echo ""

while true; do
    TIMESTAMP=$(date '+%H:%M:%S.%3N')
    echo "--- $TIMESTAMP ---"

    # 流状态检查
    for crtc in /sys/kernel/debug/dri/0/crtc-*/state; do
        [ -f "$crtc" ] || continue
        name=$(basename $(dirname $crtc))
        active=$(grep "active" $crtc 2>/dev/null | awk '{print $2}')
        mode=$(grep "mode:" $crtc 2>/dev/null | head -1 | cut -d: -f2- | xargs)
        echo "  $name: active=$active mode=$mode"
    done

    # 连接器热插拔监测
    for conn in /sys/class/drm/*/status; do
        [ -f "$conn" ] || continue
        name=$(basename $(dirname $conn))
        status=$(cat $conn 2>/dev/null)
        echo "  $name: $status"
    done

    # DC 内部计数器（如果可用）
    if [ -f /sys/kernel/debug/dri/0/amdgpu_dm_dc_info ]; then
        echo "  --- DC Internal ---"
        cat /sys/kernel/debug/dri/0/amdgpu_dm_dc_info 2>/dev/null | head -3 | sed 's/^/  /'
    fi

    sleep $INTERVAL
done
```

**扩展压力测试——多分辨率多刷新率组合**：

```bash
#!/bin/bash
# dc_stress_extended.sh — 扩展压力测试

OUTPUT=$1
declare -a MODES=(
    "640x480@60"
    "800x600@60"
    "1024x768@60"
    "1280x720@60"
    "1280x1024@60"
    "1366x768@60"
    "1440x900@60"
    "1600x900@60"
    "1680x1050@60"
    "1920x1080@60"
    "1920x1080@120"
    "1920x1080@144"
    "2560x1440@60"
    "2560x1440@120"
    "2560x1440@144"
    "3840x2160@30"
    "3840x2160@60"
)

LOOPS=${2:-1}
PASS=0
FAIL=0
TOTAL=$(( ${#MODES[@]} * LOOPS ))

echo "=== Extended DC Stress Test ==="
echo "Output: $OUTPUT"
echo "Modes: ${#MODES[@]}"
echo "Loops: $LOOPS"
echo "Total iterations: $TOTAL"
echo ""

for (( l=1; l<=LOOPS; l++ )); do
    echo "--- Loop $l of $LOOPS ---"
    for config in "${MODES[@]}"; do
        res=$(echo $config | cut -d@ -f1)
        rate=$(echo $config | cut -d@ -f2)
        xrandr --output $OUTPUT --mode $res --rate $rate 2>/dev/null
        if [ $? -eq 0 ]; then
            ((PASS++))
            echo "  ✓ $config"
        else
            ((FAIL++))
            echo "  ✗ $config"
        fi
        sleep 0.5
    done
done

echo ""
echo "=== Results ==="
echo "Total: $TOTAL"
echo "Pass:  $PASS"
echo "Fail:  $FAIL"
echo "Rate:  $(( 100 * PASS / TOTAL ))%"
```

**运行命令**：
```bash
# 单轮测试
chmod +x dc_stress_extended.sh
./dc_stress_extended.sh DP-1 2>&1 | tee /tmp/dc_stress_extended.log

# 多轮测试（验证状态恢复能力）
./dc_stress_extended.sh DP-1 3 2>&1 | tee /tmp/dc_stress_multi.log

# 回到默认配置
xrandr --output DP-1 --auto
```
