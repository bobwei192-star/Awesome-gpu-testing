# 第327天：DRM 显示架构：CRTC / Encoder / Connector / Plane

## 学习目标
- 深入理解 DRM 显示架构中四大核心组件（CRTC、Encoder、Connector、Plane）的职责与交互关系
- 掌握 CRTC 的时序控制、Encoder 的信号编码、Connector 的物理接口标准、Plane 的图层合成机制
- 理解 DRM 核心数据结构之间的关联与生命周期管理
- 掌握 AMDGPU 驱动中各组件间的绑定关系
- 通过实验验证四者之间的关系

## 知识详解

### 1. 概念原理

DRM 显示架构采用了一种高度模块化的设计思想，将显示输出抽象为四个层次化的组件：CRTC、Encoder、Connector 和 Plane。每个组件承担明确的职责，通过标准的接口进行通信。这种设计使得 DRM 核心层能够独立于具体的硬件实现，方便不同厂商的 GPU 驱动集成到 Linux 内核的显示框架中。

#### 1.1 DRM 显示架构总览

以下为 DRM 显示架构中四大组件的层次关系与数据流示意图：

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        DRM 显示架构层次图                                   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                        Framebuffer (FB)                          │   │
│  │                    (包含像素数据的颜色缓冲)                         │   │
│  └────────────┬─────────────┬──────────────────┬───────────────────┘   │
│               │             │                  │                       │
│               ▼             ▼                  ▼                       │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                  Plane（硬件平面）                                  │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │   │
│  │  │ Primary  │  │ Overlay  │  │ Cursor   │  │ Overlay │       │   │
│  │  │ Plane    │  │ Plane 0  │  │ Plane    │  │ Plane 1 │       │   │
│  │  └─────┬────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │   │
│  │        │             │             │             │             │   │
│  │        ▼             ▼             ▼             ▼             │   │
│  │  ┌──────────────────────────────────────────────────────────┐   │   │
│  │  │              Blender / Compositor（合成器）                │   │   │
│  │  │          (将多个 Plane 合成最终图像)                       │   │   │
│  │  └──────────────────────────┬───────────────────────────────┘   │   │
│  └─────────────────────────────┼───────────────────────────────────┘   │
│                                │                                       │
│                                ▼                                       │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │         CRTC（显示器控制器）                                       │   │
│  │  - 生成像素时钟（Pixel Clock）                                     │   │
│  │  - 产生水平和垂直同步信号（HSYNC/VSYNC）                           │   │
│  │  - 控制 VBlank 中断                                              │   │
│  │  - 管理页面翻转（Page Flip）                                       │   │
│  │  - 读取帧缓冲数据并生成视频流                                      │   │
│  └──────────────────────────┬───────────────────────────────────────┘   │
│                             │                                           │
│                             ▼                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │         Encoder（编码器）                                         │   │
│  │  - 将 CRTC 输出的并行视频信号编码为串行格式                         │   │
│  │  - 实现 HDMI/DP/TMDS/LVDS 等编码标准                              │   │
│  │  - 进行信号电平调整与预加重                                       │   │
│  └──────────────────────────┬───────────────────────────────────────┘   │
│                             │                                           │
│                             ▼                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │         Connector（连接器）                                       │   │
│  │  - 提供物理连接接口（HDMI、DP、eDP、VGA、DVI 等）                  │   │
│  │  - 检测显示器热插拔状态                                           │   │
│  │  - 读取 EDID 获取显示器能力信息                                    │   │
│  │  - 管理 DDC/AUX 通信通道                                         │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                      │                                  │
│                                      ▼                                  │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    物理显示器（Monitor）                           │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

数据流方向：Framebuffer → Plane → Blender → CRTC → Encoder → Connector → Monitor

#### 1.2 CRTC（Cathode Ray Tube Controller）

CRTC 是 DRM 显示架构中最核心的组件，尽管其名称源自 CRT 显示器时代，但在现代 LCD/OLED 显示中仍然沿用。CRTC 主要负责生成显示器时序信号和控制帧缓冲的读取。

##### 1.2.1 CRTC 的核心职责

```c
/* include/drm/drm_crtc.h - struct drm_crtc 核心定义 (Linux 6.x) */
struct drm_crtc {
    struct drm_device *dev;
    struct drm_mode_object base;

    /* CRTC 在设备中的索引 */
    uint32_t pipe;

    /* 当前显示模式 */
    struct drm_display_mode mode;

    /* 当前启用的 Plane 列表 */
    struct drm_plane *primary;
    struct drm_plane *cursor;

    /* 绑定到此 CRTC 的 Encoder */
    struct drm_encoder *encoder;

    /* 帧缓冲相关状态 */
    struct drm_framebuffer *fb;
    struct drm_pending_vblank_event *event;

    /* 状态追踪 */
    bool enabled;
    bool active;

    /* 函数回调 */
    const struct drm_crtc_funcs *funcs;
    const struct drm_crtc_helper_funcs *helper_private;
};
```

CRTC 的关键能力包括：

**时序生成**：CRTC 根据 `drm_display_mode` 中的时序参数（htotal、vtotal、hsync_start 等）产生精确的像素时钟，驱动显示控制器按顺序逐行扫描帧缓冲。

**帧缓冲读取**：CRTC 通过 DMA 引擎从系统内存或显存中读取帧缓冲数据，按照设定的分辨率输出像素流。

**VBlank 管理**：当 CRTC 完成一帧的扫描后（从右上角到左下角），会进入短暂的垂直消隐期（VBlank）。CRTC 在此阶段产生 VBlank 中断，为 Page Flip、VSync 同步等操作提供时间基准。

**Gamma 校正**：CRTC 内置硬件 Gamma 查找表（LUT），在输出前对像素颜色进行调整。

##### 1.2.2 CRTC 的工作流程

```
时间线：一个完整的显示帧周期
┌──────────────┬─────────────────────┬──────────────────────┬──────┐
│  VBlank 开始 │    Active Display    │   VBlank 开始        │      │
│  (VSYNC IRQ) │     (像素扫描)        │   (VSYNC IRQ)        │      │
└──────┬───────┴──────────────────────┴──────────┬───────────┴──────┘
       │                                          │
       ▼                                          ▼
  ┌────────────────┐                  ┌────────────────┐
  │ VBlank Period  │                  │ VBlank Period  │
  │ (5% of frame)  │                  │ (5% of frame)  │
  └────────────────┘                  └────────────────┘
       │                                          │
       │ Page Flip 提交                            │ Page Flip 提交
       │ Gamma LUT 更新                            │ Gamma LUT 更新
       │ 原子提交生效                               │ 原子提交生效
       ▼                                          ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                     Frame N (显示帧)                         │
  │  ┌────────────────────────────────────────────────────────┐ │
  │  │                                                        │ │
  │  │  Line 0: ───►───►───►───►  HBlank ─►───►───►───►    │ │
  │  │                                                        │ │
  │  │  Line 1: ───►───►───►───►  HBlank ─►───►───►───►    │ │
  │  │                                                        │ │
  │  │  ... (HTotal 行, 每行 HTotal 像素)                      │ │
  │  │                                                        │ │
  │  │  Line VDisplay: ──►── VBlank ──►── next frame        │ │
  │  └────────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────────┘
```

#### 1.3 Encoder（编码器）

Encoder 位于 CRTC 和 Connector 之间，负责将 CRTC 输出的并行视频信号转换为特定显示接口标准所需的串行差分信号。

##### 1.3.1 Encoder 的功能与类型

```c
/* include/drm/drm_encoder.h */
struct drm_encoder {
    struct drm_device *dev;
    struct drm_mode_object base;

    /* 编码器类型 */
    enum drm_encoder_type {
        DRM_MODE_ENCODER_NONE     = 0,
        DRM_MODE_ENCODER_DAC      = 1,  /* 模拟接口 (VGA) */
        DRM_MODE_ENCODER_TMDS     = 2,  /* HDMI/DVI */
        DRM_MODE_ENCODER_LVDS     = 3,  /* LVDS (笔记本内置屏) */
        DRM_MODE_ENCODER_TVDAC    = 4,  /* TV 输出 (复合/S-Video) */
        DRM_MODE_ENCODER_VIRTUAL  = 5,  /* 虚拟编码器 */
        DRM_MODE_ENCODER_DSI      = 6,  /* MIPI DSI (手机/平板) */
        DRM_MODE_ENCODER_DPMST    = 7,  /* DisplayPort MST */
        DRM_MODE_ENCODER_DPI      = 8,  /* RGB 并行接口 */
    } encoder_type;

    /* 可能的 CRTC 掩码（指示可绑定的 CRTC） */
    uint32_t possible_crtcs;

    /* 可能的 Clone 掩码（用于多显示器克隆） */
    uint32_t possible_clones;

    /* 绑定的 CRTC (由核心层管理) */
    struct drm_crtc *crtc;
    struct drm_bridge *bridge;

    const struct drm_encoder_funcs *funcs;
    const struct drm_encoder_helper_funcs *helper_private;
};
```

常见的 Encoder 类型与信号标准：

| Encoder 类型 | 对应接口标准 | 信号类型 | 最大带宽（典型） |
|-------------|-------------|---------|----------------|
| TMDS | HDMI 1.4/2.0, DVI | 差分串行 (TMDS) | 18 Gbps (HDMI 2.0) |
| LVDS | LVDS（笔记本内置屏） | 差分串行 (LVDS) | 3.125 Gbps/通道 |
| DPMST | DisplayPort MST | 差分串行 (HBR) | 32.4 Gbps (DP 1.4) |
| DSI | MIPI DSI | 差分串行 (D-PHY) | 4.5 Gbps/通道 |
| DAC | VGA | 模拟 RGB | 388 MHz 像素时钟 |
| TVDAC | 复合/S-Video | 模拟复合 | 标准清晰度 |

##### 1.3.2 Encoder 在 AMDGPU 中的实现

在 AMDGPU 驱动中，Encoder 的实现分布在 DC（Display Core）层中：

```c
/* drivers/gpu/drm/amd/display/dc/inc/core_types.h */
struct encoder {
    struct dc_context *ctx;
    struct dc_bios *bios;

    /* 编码器功能标识 */
    enum engine_id id;
    enum transmitter transmitter;
    enum signal_type signal;

    /* 支持的输出信号类型 */
    const struct encoder_features *features;

    /* DP/HDMI 特定的功能支持 */
    union {
        struct dp_encoder dp;
        struct hdmi_encoder hdmi;
    };

    /* 编码器功能回调 */
    const struct encoder_funcs *funcs;
};
```

AMDGPU 的 Encoder 实现包含了以下关键功能：

1. **信号分配（Signal Assignment）**：将 CRTC 输出的数字视频流分配给物理发射器（Transmitter）
2. **协议封装**：根据输出接口标准（HDMI/DP）进行数据包封装
3. **链路训练**：对 DisplayPort 接口执行链路训练（Link Training）
4. **HDCP 加密**：管理 HDCP 内容保护
5. **色彩空间转换**：将 CRTC 输出的 RGB 转换为 YCbCr 等格式

##### 1.3.3 Encoder 绑定链

```
 ┌─────────┐    CRTC 编码输出     ┌──────────┐    TMDS 差分对    ┌───────────┐
 │  CRTC    │ ─────────────────►  │ Encoder  │ ───────────────► │ Connector │
 │  (时序)  │                    │ (TMDS)   │                   │ (HDMI)    │
 └─────────┘                    └──────────┘                   └───────────┘
                                      │
                                      │ 可选：添加 Bridge 设备
                                      ▼
                               ┌──────────────┐
                               │ drm_bridge   │
                               │ (例如：电平   │
                               │  转换器芯片)   │
                               └──────────────┘
```

#### 1.4 Connector（连接器）

Connector 代表物理显示接口（HDMI 端口、DP 端口等），负责检测显示器连接状态、读取 EDID 信息和管理通信通道。

##### 1.4.1 Connector 的数据结构

```c
/* include/drm/drm_connector.h */
struct drm_connector {
    struct drm_device *dev;
    struct drm_mode_object base;

    /* 连接器类型 */
    enum drm_connector_type {
        DRM_MODE_CONNECTOR_Unknown      = 0,
        DRM_MODE_CONNECTOR_VGA          = 1,
        DRM_MODE_CONNECTOR_DVII         = 2,
        DRM_MODE_CONNECTOR_DVID         = 3,
        DRM_MODE_CONNECTOR_DVIA         = 4,
        DRM_MODE_CONNECTOR_Composite    = 5,
        DRM_MODE_CONNECTOR_SVIDEO      = 6,
        DRM_MODE_CONNECTOR_LVDS        = 7,
        DRM_MODE_CONNECTOR_Component   = 8,
        DRM_MODE_CONNECTOR_9PinDIN     = 9,
        DRM_MODE_CONNECTOR_DisplayPort = 10,
        DRM_MODE_CONNECTOR_HDMIA       = 11,
        DRM_MODE_CONNECTOR_HDMIB       = 12,
        DRM_MODE_CONNECTOR_TV          = 13,
        DRM_MODE_CONNECTOR_eDP         = 14,
        DRM_MODE_CONNECTOR_VIRTUAL     = 15,
        DRM_MODE_CONNECTOR_DSI         = 16,
        DRM_MODE_CONNECTOR_DPI         = 17,
        DRM_MODE_CONNECTOR_WRITEBACK   = 18,
        DRM_MODE_CONNECTOR_SPI         = 19,
        DRM_MODE_CONNECTOR_USB         = 20,
    } connector_type;

    /* 连接器类型 ID（用于生成连接器名称，如 HDMI-A-1） */
    int connector_type_id;

    /* 连接状态 */
    enum drm_connector_status {
        connector_status_connected,
        connector_status_disconnected,
        connector_status_unknown,
    } status;

    /* 物理尺寸 (mm) */
    uint16_t mmWidth, mmHeight;

    /* 可能的 Encoder 掩码 */
    uint32_t possible_encoders;

    /* 支持的显示模式列表 */
    struct drm_display_mode *probed_modes;
    struct list_head modes;

    /* EDID 信息 */
    struct edid *edid_blob_ptr;
    struct drm_property_blob *edid_blob;

    /* 强制输出状态（用于测试） */
    enum drm_connector_force {
        DRM_FORCE_UNSPECIFIED,
        DRM_FORCE_OFF,
        DRM_FORCE_ON,
        DRM_FORCE_ON_DIGITAL,
    } force;

    /* 热插拔检测 */
    struct drm_connector_funcs *funcs;

    /* DP MST 相关 */
    struct drm_dp_mst_port *port;
    struct drm_dp_mst_topology_mgr *mgr;
};
```

##### 1.4.2 Connector 类型与物理接口

| DRM Connector 类型 | 物理接口 | 最大分辨率 | 通信通道 |
|-------------------|---------|-----------|---------|
| DRM_MODE_CONNECTOR_HDMIA | HDMI Type A | 4K@60 (HDMI 2.0) | DDC (I2C) |
| DRM_MODE_CONNECTOR_DisplayPort | DisplayPort | 8K@60 (DP 1.4) | AUX CH |
| DRM_MODE_CONNECTOR_eDP | 嵌入式 DisplayPort | 4K@120 (eDP 1.4) | AUX CH |
| DRM_MODE_CONNECTOR_DSI | MIPI DSI | 4K@60 | D-PHY |
| DRM_MODE_CONNECTOR_LVDS | LVDS | 1920x1200 | I2C |
| DRM_MODE_CONNECTOR_VGA | DB-15 VGA | 2048x1536 | DDC |
| DRM_MODE_CONNECTOR_USB | USB Type-C | 取决于 Alt Mode | USB PD |

##### 1.4.3 Connector 状态检测与热插拔

Connector 状态检测的主要流程如下：

```
用户空间 ioctl              内核 DRM                硬件
     │                        │                      │
     │ drmModeGetConnector()   │                      │
     ├───────────────────────► │                      │
     │                        │ connector->funcs->    │
     │                        │   detect()            │
     │                        ├─────────────────────► │
     │                        │                      │
     │                        │ 读取 HPD 引脚状态     │
     │                        │◄──────────────────────│
     │                        │                      │
     │                        │ 读取 EDID (DDC/AUX)  │
     │                        │◄──────────────────────│
     │                        │                      │
     │                        │ 更新 status 字段      │
     │                        │◄──────────────────────│
     │  返回连接器信息和状态    │                      │
     │◄───────────────────────│                      │
     │                        │                      │

热插拔事件流程:
    硬件 HPD IRQ           DRM Core               用户空间
     │                      │                      │
     │ hotplug 中断         │                      │
     ├────────────────────► │                      │
     │                      │ drm_kms_helper_      │
     │                      │ hotplug_event()       │
     │                      ├────────────────────► │
     │                      │ 重新枚举连接器         │
     │                      │ uevent (NETLINK)     │
     │                      │                      │
```

#### 1.5 Plane（硬件平面）

Plane 是 DRM 显示架构中负责硬件图层合成的组件。每个 Plane 对应一个独立的帧缓冲源，通过硬件合成器将多个 Plane 叠加为最终的显示画面。

##### 1.5.1 Plane 的类型与属性

```c
/* include/drm/drm_plane.h */
struct drm_plane {
    struct drm_device *dev;
    struct drm_mode_object base;

    /* Plane 类型 */
    enum drm_plane_type {
        DRM_PLANE_TYPE_PRIMARY,   /* 主平面（必须含有一个） */
        DRM_PLANE_TYPE_OVERLAY,  /* 覆盖平面（可选） */
        DRM_PLANE_TYPE_CURSOR,   /* 光标平面（通常只有一个） */
    } type;

    /* 支持的像素格式 */
    uint32_t *format_types;
    unsigned int format_count;
    uint64_t *modifiers;
    unsigned int modifier_count;

    /* Plane 支持的 CRTC 掩码 */
    uint32_t possible_crtcs;

    /* 当前状态 */
    struct drm_plane_state *state;

    /* 函数回调 */
    const struct drm_plane_funcs *funcs;
    const struct drm_plane_helper_funcs *helper_private;
};
```

Plane 的核心功能：

1. **独立帧缓冲**：每个 Plane 绑定一个独立的帧缓冲，从不同内存区域读取像素数据
2. **位置与大小控制**：可以设置 Plane 在最终画面中的位置（x, y）和缩放尺寸（src_w, src_h, crtc_w, crtc_h）
3. **Alpha 混合**：支持全局 Alpha 值（透明度）调整
4. **像素格式转换**：支持不同帧缓冲格式（如 XR24、AR24、NV12）自动转换为统一的输出格式
5. **色彩校正**：支持独立的色彩矩阵和伽马校正

##### 1.5.2 AMDGPU 的 Plane 实现

AMDGPU 驱动中，Plane 的实现包含多个硬件平面：

```c
/* drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_plane.h */
struct dm_plane_state {
    struct drm_plane_state base;

    /* DC 层面的缩放信息 */
    struct dc_scaling_info scaling;

    /* DC 层面的平面信息 */
    struct dc_plane_info plane_info;

    /* DC 层面的平面地址 */
    struct dc_plane_address address;

    /* 色彩管理信息 */
    struct dc_csc_transform csc;
    struct dc_gamma gamma;
    bool gamma_is_original;

    /* 是否强制使用完整更新 */
    bool force_full_update;
};
```

##### 1.5.3 Plane 合成示意图

```
       Plane 0 (Primary)              Plane 1 (Overlay)             Plane 2 (Cursor)
    ┌────────────────────┐         ┌──────────────────┐         ┌──────────────────┐
    │ 1920x1080          │         │ 640x480           │         │ 64x64            │
    │ FB: XR24           │         │ FB: AR24          │         │ FB: AR24         │
    │ Position: (0,0)    │         │ Position: (100,50) │        │ Position: (900,5) │
    │ Alpha: 1.0         │         │ Alpha: 0.8        │         │ Alpha: 1.0       │
    └─────────┬──────────┘         └─────────┬─────────┘        └────────┬─────────┘
              │                              │                          │
              ▼                              ▼                          ▼
    ┌──────────────────────────────────────────────────────────────────────────┐
    │                         Hardware Compositor                               │
    │                                                                          │
    │  Compositing formula:                                                    │
    │  Output = Plane0.rgb * Plane0.a                                          │
    │         + Plane1.rgb * Plane1.a * (1 - Plane0.a)                         │
    │         + Plane2.rgb * Plane2.a * (1 - Plane0.a - Plane1.a)              │
    │                                                                          │
    └──────────────────────────────┬───────────────────────────────────────────┘
                                   │
                                   ▼
                         CRTC (最终输出画面)
                    ┌───────────────────────────┐
                    │                           │
                    │  ┌───────────────────┐   │
                    │  │   64x64 Cursor    │   │
                    │  │    (Plane 2)      │   │
                    │  └───────────────────┘   │
                    │                           │
                    │  ┌──────────────┐        │
                    │  │ 640x480      │        │
                    │  │ Alpha 0.8    │        │
                    │  │ (Plane 1)    │        │
                    │  └──────────────┘        │
                    │                           │
                    │  Primary FB (Plane 0)     │
                    │  1920x1080 Background     │
                    └───────────────────────────┘
```

#### 1.6 四大组件的绑定关系

在 DRM 中，四个组件的绑定关系由一个标准化的配置过程管理：

```
可能的连接拓补结构 (Possible Topologies)

单显示器场景:
  CRTC ──► Encoder ──► Connector ──► Monitor

双显示器扩展场景:
  CRTC 0 ──► Encoder 0 ──► Connector A ──► Monitor 1
  CRTC 1 ──► Encoder 1 ──► Connector B ──► Monitor 2

克隆模式场景:
  CRTC ──► Encoder 0 ──► Connector A ──► Monitor 1
        ──► Encoder 1 ──► Connector B ──► Monitor 2

MST 多流场景:
  CRTC 0 ──► Encoder ──► DP Connector ──► MST Hub ──┬─► Monitor 1
  CRTC 1 ──► Encoder ──►              │              └─► Monitor 2
                                       │              ┌─► Monitor 3
                                       └──────────────┴─► Monitor 4
```

##### 1.6.1 possible_crtcs / possible_encoders 掩码

`possible_crtcs` 和 `possible_encoders` 是用于描述组件兼容性的位掩码：

```
possible_crtcs 示例 (Encoder 的字段):
  位 0: CRTC 0 ──► Encoder 可绑定 CRTC 0
  位 1: CRTC 1 ──► Encoder 可绑定 CRTC 1
  位 2: CRTC 2 ──► Encoder 可绑定 CRTC 2
  ...
  
  值: 0x07 (位 0-2 置位) ──► Encoder 可绑定 CRTC 0, 1, 2
  值: 0x01 (仅位 0 置位) ──► Encoder 仅可绑定 CRTC 0

possible_encoders 示例 (Connector 的字段):
  位 0: Encoder 0 ──► Connector 可绑定 Encoder 0
  位 1: Encoder 1 ──► Connector 可绑定 Encoder 1
  ...
  
  值: 0x03 ──► Connector 可绑定 Encoder 0 或 1
```

这些掩码在驱动初始化时根据硬件能力设置，确保只有硬件支持的组件组合能被用户空间配置。

#### 1.7 AMDGPU 驱动的组件实现

在 AMDGPU 驱动中，这些 DRM 组件通过 DC（Display Core）层与硬件交互：

```
                  用户空间
                      │
                      ▼
              DRM Core (drm.ko)
          ┌─────────────────────┐
          │ struct drm_crtc      │
          │ struct drm_encoder   │
          │ struct drm_connector │
          │ struct drm_plane     │
          └─────────┬───────────┘
                    │
                    ▼
          amdgpu_dm (Display Manager)
          ┌─────────────────────┐
          │ amdgpu_dm_crtc_init │
          │ amdgpu_dm_encoder   │
          │   _init             │
          │ amdgpu_dm_connector │
          │   _init             │
          │ amdgpu_dm_plane     │
          │   _init             │
          └─────────┬───────────┘
                    │
                    ▼
         DC (Display Core)
          ┌─────────────────────┐
          │ struct dc           │
          │ struct dc_stream    │
          │ struct dc_plane     │
          │ struct dc_link      │
          └─────────┬───────────┘
                    │
                    ▼
           DCN 硬件 (Display Core Next)
          ┌─────────────────────┐
          │ OTG / DPP / MPC    │
          │ OPP / DIO / HUBP   │
          └─────────────────────┘
```

##### 1.7.1 CRTC 的 AMDGPU 初始化

```c
/* drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_crtc.c */
int amdgpu_dm_crtc_init(struct amdgpu_display_manager *dm,
                        struct drm_plane *plane,
                        uint32_t crtc_index)
{
    struct amdgpu_device *adev = dm->adev;
    struct drm_crtc *crtc;
    struct dm_crtc_state *dm_state;
    int res;

    crtc = kzalloc(sizeof(*crtc), GFP_KERNEL);
    if (!crtc)
        return -ENOMEM;

    /* 初始化 DRM CRTC 核心结构 */
    res = drm_crtc_init_with_planes(dm->ddev, crtc, plane,
                                   NULL, &amdgpu_dm_crtc_funcs,
                                   NULL);
    if (res) {
        kfree(crtc);
        return res;
    }

    /* 设置 CRTC 回调 */
    crtc->helper_private = &amdgpu_dm_crtc_helper_funcs;

    /* 每个 CRTC 硬件的 gamma 大小 */
    drm_mode_crtc_set_gamma_size(crtc, 256);

    /* 初始化 DMA 状态 */
    dm_state = to_dm_crtc_state(crtc->state);
    dm_state->stream = NULL;

    /* 获取 CRTC 对应的硬件 ID */
    crtc->pipe = crtc_index;

    DRM_DEBUG_KMS("CRTC %d initialized on pipe %d\n",
                  crtc->base.id, crtc_index);

    return 0;
}
```

##### 1.7.2 Connector 的热插拔处理

```c
/* drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c */
static enum drm_connector_status
amdgpu_dm_connector_detect(struct drm_connector *connector, bool force)
{
    struct amdgpu_dm_connector *aconnector =
        to_amdgpu_dm_connector(connector);
    struct dc_link *link = aconnector->dc_link;
    enum drm_connector_status status = connector_status_disconnected;
    bool connected;

    /* 通过 DC 层查询链路状态 */
    connected = dc_link_detect(link, DETECT_REASON_HPD);

    if (connected) {
        status = connector_status_connected;

        /* 读取 EDID */
        if (!aconnector->edid) {
            aconnector->edid = drm_get_edid(connector,
                &link->aux.ddc);
            if (aconnector->edid) {
                /* 使用 EDID 信息更新连接器属性 */
                drm_connector_update_edid_property(connector,
                    aconnector->edid);
            }
        }
    }

    return status;
}
```

#### 1.8 组件间的数据流与依赖关系

当一个显示模式设置操作发生时，四个组件之间的交互流程如下：

```
用户空间请求设置模式
  (xrandr --output HDMI-A-1 --mode 1920x1080)
        │
        ▼
  1. 用户空间调用 drmModeSetCrtc(fd, crtc_id, fb_id, ...)
        │
        ▼
  2. DRM Core: drm_crtc_set_mode()
        │
        ├── 2.1 禁用当前 CRTC 和 Plane
        ├── 2.2 设置 CRTC 显示模式 (drm_display_mode)
        │       │
        │       ▼
        │   CRTC 的函数: crtc->helper_private->mode_set()
        │   (在 amdgpu_dm 中触发 DC 配置流更新)
        │
        ├── 2.3 设置 Encoder
        │       │
        │       ▼
        │   Encoder 的函数: encoder->helper_private->mode_set()
        │   (配置传输器、链路训练)
        │
        ├── 2.4 连接 Plane 到 CRTC
        │       │
        │       ▼
        │   Plane 的函数: plane->helper_private->atomic_update()
        │   (配置 DPP/MUX 硬件)
        │
        └── 2.5 提交硬件配置
                │
                ▼
            DC 提交序列 (dc_commit_streams)
            ──► 等待 VBlank
            ──► 更新 OTG 时序
            ──► 更新 DPP 管线
            ──► 使能输出
```

#### 1.9 Plane 合成细节与多平面配置

Plane 合成（Composition）是 DRM 显示架构中最核心的功能之一。合成器（Blender/Compositor）负责将多个 Plane 的像素数据按照正确的层级和透明度混合成最终的显示帧。以下为 Plane 合成的详细流程：

```
时间轴 ──────────────────────────────────────────────────────►
        帧 N-1                    帧 N                    帧 N+1
        │                         │                         │
Primary  ┌─────────────────┐      ┌─────────────────┐      ┌────
Plane    │ 桌面壁纸/背景   │      │ 桌面壁纸/背景   │      │...
         └────────┬────────┘      └────────┬────────┘
                  │                        │
Overlay   ┌───────▼────────┐      ┌───────▼────────┐      ┌────
Plane 0   │  视频播放窗口   │      │  视频播放窗口   │      │...
          │  (YUV 硬件解码) │      │  (YUV 硬件解码) │      │
          └───────┬────────┘      └───────┬────────┘
                  │                        │
Overlay   ┌───────▼────────┐      ┌───────▼────────┐      ┌────
Plane 1   │  字幕/OSD      │      │  字幕/OSD      │      │...
          │  (ARGB 透明)   │      │  (ARGB 透明)   │      │
          └───────┬────────┘      └───────┬────────┘
                  │                        │
Cursor    ┌───────▼────────┐      ┌───────▼────────┐      ┌────
Plane     │  鼠标指针      │      │  鼠标指针      │      │...
          │  (64x64 AR24)  │      │  (64x64 AR24)  │      │
          └───────┬────────┘      └───────┬────────┘
                  │                        │
          ┌───────▼────────────────────────▼──────────┐
          │              Blender（合成器）              │
          │                                            │
          │  输出 = Σ(Plane_i.rgb × Plane_i.a          │
          │          × Π(1 - Plane_j.a for j<i))       │
          └───────────────────────┬────────────────────┘
                                  │
                                  ▼
                          ┌──────────────┐
                          │   CRTC 输出  │
                          │  1080p@60Hz  │
                          └──────────────┘
```

在 AMDGPU DC 中，合成操作由 MPC（Multiple Pipe Combined）模块完成。MPC 可以同时处理最多 6 个平面的合成：

```c
struct mpcc {
    struct mpcc *top;
    struct mpcc *bottom;
    int mpcc_id;
    struct mpcc_cfg cfg;
    bool mpcc_bot;
};

struct mpcc_cfg {
    enum mpcc_alpha_blend_mode alpha_mode;
    enum mpcc_blend_mode blend_mode;
    struct dc_plane_address addr;
    struct scaling_taps taps;
};
```

Alpha 混合模式的三种类型：

| 模式 | 枚举值 | 描述 | 应用场景 |
|------|--------|------|----------|
| 逐像素 Alpha | PER_PIXEL_ALPHA | 每个像素独立携带 Alpha 值 | PNG 图片、视频字幕 |
| 全局 Alpha | GLOBAL_ALPHA | 整个平面使用统一 Alpha 值 | 窗口半透明、淡入淡出效果 |
| 无 Alpha | NO_ALPHA | 不进行 Alpha 混合 | 不透明桌面背景 |

多平面配置的优先级规则：

```
┌─────────────────────────────────────────────────────────────┐
│               Plane 合成优先级规则                             │
│                                                             │
│  1. Z-order（层序）：固定为 Primary < Overlay < Cursor        │
│     Primary Plane: Z=0（最底层）                              │
│     Overlay Plane 0: Z=1                                     │
│     Overlay Plane 1: Z=2                                     │
│     Cursor Plane: Z=max（最顶层）                             │
│                                                             │
│  2. 合成公式：                                               │
│     output = background                                      │
│     for each plane in Z-order:                               │
│         if plane.alpha_mode == PER_PIXEL:                    │
│             output = plane.pixel.rgba over output            │
│         elif plane.alpha_mode == GLOBAL:                     │
│             output = blend(plane.rgb, output, plane.global_a)│
│         else:                                                │
│             output = plane.rgb                               │
│                                                             │
│  3. 性能优化：硬件优先使用逐层合成（per-pipe blending）        │
│     避免将所有平面读入后统一合成（full composition）           │
└─────────────────────────────────────────────────────────────┘
```

#### 1.10 Encoder 实现对比: DP vs HDMI

Encoder 在不同接口上的实现差异显著。以下对比 DisplayPort 和 HDMI 两种最常见的 Encoder 实现：

**DP Encoder 实现要点**：

```c
struct dcn10_dp {
    struct dp_stream stream;
    struct link_training_settings lt;
    uint8_t lane_count;
    uint32_t link_rate;
    bool mst_enabled;
};

enum dc_status dp_phy_training(
    struct dc_link *link,
    struct link_training_settings *lt)
{
    /* 阶段 1: 时钟恢复 (Clock Recovery, CR) */
    /* 阶段 2: 通道均衡 (Channel Equalization, EQ) */
    /* 阶段 3: 符号锁定 (Symbol Lock) */
    /* 阶段 4: 交织对齐 (Interlane Alignment) */
    /* 阶段 5: 链路状态检查 (Link Status Check) */
}
```

**HDMI Encoder 实现要点**：

```c
struct dcn10_hdmi {
    struct hdmi_stream stream;
    uint32_t tmds_bitrate;
    bool deep_color;
    bool hdmi_20;
};

uint32_t hdmi_calc_tmds_clock(
    uint32_t pixel_clock_khz,
    uint32_t color_depth)
{
    if (color_depth == 10) return pixel_clock_khz * 125 / 100;
    if (color_depth == 12) return pixel_clock_khz * 150 / 100;
    if (color_depth == 16) return pixel_clock_khz * 2;
    return pixel_clock_khz;
}
```

**DP 与 HDMI Encoder 关键差异**：

| 特性 | DP Encoder | HDMI Encoder |
|------|-----------|--------------|
| 信号编码 | ANSI 8b/10b 编码 | TMDS 最小化差分信号 |
| 数据传输 | 微包架构 (Micro-Packet) | 流式 TMDS 字符 |
| 时钟信号 | 嵌入数据流中 | 独立 TMDS 时钟通道 |
| 链路训练 | 必须（协商速率和通道数） | 可选（仅 HDMI 2.1 FRL） |
| 多流支持 | MST（一个端口多个显示器） | 不支持原生多流 |
| 最大带宽 | HBR3: 32.4 Gbps (4通道) | HDMI 2.1: 48 Gbps (FRL) |
| AUX 通道 | 必须（用于链路管理和 EDID） | 可选（部分实现 CEC） |
| HPD 处理 | 中断驱动（IRQ_HPD） | 电平变化检测 |
| 测试复杂度 | 高（需要链路训练验证） | 中（TMDS 信号质量验证） |

#### 1.11 组件生命周期管理

DRM 显示组件的生命周期从创建到销毁包含多个关键阶段。理解生命周期有助于排查组件泄漏、状态不一致等驱动问题。

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CRTC / Encoder / Connector / Plane 生命周期        │
│                                                                     │
│  初始化阶段:                                                         │
│  ┌─────────┐   ┌──────────┐   ┌────────────┐   ┌──────────┐       │
│  │ 驱动加载 │─►│DRM 核心  │─►│ 组件创建   │─►│ 属性初始化 │       │
│  │probe    │   │drm_dev_  │   │drm_crtc_  │   │possible_  │       │
│  │         │   │register  │   │init       │   │crtcs设置  │       │
│  └─────────┘   └──────────┘   └────────────┘   └──────────┘       │
│                                                                     │
│  使能阶段:                                                           │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐       │
│  │用户空间  │─►│drmModeSet │─►│组件使能   │─►│硬件配置   │       │
│  │xrandr    │   │Crtc      │   │mode_set  │   │DC commit │       │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘       │
│                                                                     │
│  运行阶段:                                                           │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐       │
│  │页面翻转  │─►│VBlank    │─►│热插拔    │─►│DPMS状态  │       │
│  │page_flip │   │中断处理  │   │uevent    │   │切换      │       │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘       │
│                                                                     │
│  销毁阶段:                                                           │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐       │
│  │驱动卸载  │─►│组件解绑  │─►│资源释放  │─►│内核清理  │       │
│  │remove    │   │drm_conn_ │   │drmm_     │   │drm_dev_  │       │
│  │          │   │ector_cleanup│kfree   │   │unregister│       │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘       │
└─────────────────────────────────────────────────────────────────────┘
```

组件销毁时的依赖顺序：

```c
void drm_mode_config_cleanup(struct drm_device *dev)
{
    struct drm_connector *conn, *conn_temp;
    struct drm_encoder *enc, *enc_temp;
    struct drm_crtc *crtc, *crtc_temp;
    struct drm_plane *plane, *plane_temp;

    list_for_each_entry_safe(conn, conn_temp,
                             &dev->mode_config.connector_list, head) {
        drm_connector_unregister(conn);
        drm_connector_cleanup(conn);
    }

    list_for_each_entry_safe(enc, enc_temp,
                             &dev->mode_config.encoder_list, head) {
        drm_encoder_cleanup(enc);
    }

    list_for_each_entry_safe(plane, plane_temp,
                             &dev->mode_config.plane_list, head) {
        drm_plane_cleanup(plane);
    }

    list_for_each_entry_safe(crtc, crtc_temp,
                             &dev->mode_config.crtc_list, head) {
        drm_crtc_cleanup(crtc);
    }
}
```

销毁顺序必须逆序创建顺序，因为 CRTC 引用了 Plane 和 Encoder，Encoder 引用了 Connector。如果先销毁 CRTC，其他组件将无法正常解除绑定。

### 2. 实践操作

#### 2.1 使用 modetest 查看组件关系

```bash
# 查看完整的 DRM 显示组件信息
$ modetest -M amdgpu

# 精简输出，仅查看 CRTC 信息
$ modetest -M amdgpu -a | grep -A5 "CRTCs"
id      crtc    type    possible planes
4       4       primary 0x00000001
5       5       primary 0x00000002

# 查看 Encoder 及其可能的 CRTC
$ modetest -M amdgpu -e
Encoders:
id      crtc    type    possible crtcs  possible clones
7       4       TMDS   0x00000001      0x00000000
8       5       TMDS   0x00000002      0x00000000
9       0       TMDS   0x00000003      0x00000000

# 查看 Connector 状态及绑定的 Encoder
$ modetest -M amdgpu -c
Connectors:
id      encoder status          type    size (mm)       modes   enc
19      7       connected       eDP     309x174          24      7
        modes:
                name refresh (Hz) vdisp hdisp vtotal htotal ...
                1920x1080 60.00   1080 1920 1125 2200  ...
                ...
20      8       connected       HDMI-A  531x298          12      8
        modes:
                1920x1080 60.00   1080 1920 1125 2200  ...
                1280x720 60.00   720 1280 750 1650  ...
                ...
21      0       disconnected    DP      0x0              0       9

# 查看 Plane 信息
$ modetest -M amdgpu -p
Planes:
id      crtc    fb      CRTC x,y        x,y     gamma size
4       4       34      0,0             0,0     0
        formats: XR24 AR24 AR30 XR30 RG16 ...
5       5       0       0,0             0,0     0
        formats: XR24 AR24 AR30 XR30 RG16 ...
6       0       0       0,0             0,0     0
        formats: AR24 XR24 RG16 ...
```

#### 2.2 使用 drm_info 工具分析显示架构

drm_info 是一个更现代的工具，提供了更清晰的 DRM 组件关系视图：

```bash
# 安装 drm_info
$ git clone https://gitlab.freedesktop.org/emersion/drm_info.git
$ cd drm_info
$ meson build && ninja -C build

# 查看组件树状结构
$ ./build/drm_info
Device: amdgpu (AMD GPU)
    drivers/gpu/drm/amd/amdgpu/amdgpu_drv.c
    ├── CRTC 4 (opp/head: HDMI-A-1)
    │   ├── Connector 19 (eDP-1)
    │   │   ├── status: connected
    │   │   ├── EDID: <present>
    │   │   ├── modes: 1920x1080@60Hz ...
    │   │   └── possible encoders: 7
    │   ├── Encoder 7 (TMDS)
    │   │   ├── possible crtcs: 1
    │   │   └── possible clones: 0
    │   ├── Plane 4 (Primary)
    │   │   ├── formats: XR24, AR24, NV12, ...
    │   │   └── fb: 34 (1920x1080 XR24)
    │   └── Plane 6 (Cursor)
    │       └── formats: AR24
    └── ...
```

#### 2.3 编写 C 程序遍历组件关系

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

static const char *connector_type_str(int type) {
    switch (type) {
    case DRM_MODE_CONNECTOR_HDMIA: return "HDMI-A";
    case DRM_MODE_CONNECTOR_DisplayPort: return "DP";
    case DRM_MODE_CONNECTOR_eDP: return "eDP";
    case DRM_MODE_CONNECTOR_LVDS: return "LVDS";
    case DRM_MODE_CONNECTOR_VGA: return "VGA";
    case DRM_MODE_CONNECTOR_DSI: return "DSI";
    default: return "Unknown";
    }
}

static const char *encoder_type_str(int type) {
    switch (type) {
    case DRM_MODE_ENCODER_TMDS: return "TMDS";
    case DRM_MODE_ENCODER_LVDS: return "LVDS";
    case DRM_MODE_ENCODER_DAC: return "DAC";
    case DRM_MODE_ENCODER_DPMST: return "DP-MST";
    default: return "Unknown";
    }
}

static const char *plane_type_str(int type) {
    switch (type) {
    case DRM_PLANE_TYPE_PRIMARY: return "Primary";
    case DRM_PLANE_TYPE_OVERLAY: return "Overlay";
    case DRM_PLANE_TYPE_CURSOR: return "Cursor";
    default: return "Unknown";
    }
}

int main(int argc, char **argv) {
    int fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    drmModeRes *res = drmModeGetResources(fd);
    if (!res) {
        fprintf(stderr, "drmModeGetResources failed\n");
        close(fd);
        return 1;
    }

    printf("=== DRM Component Hierarchy ===\n\n");

    /* 遍历 CRTC */
    printf("CRTCs (%d):\n", res->count_crtcs);
    for (int i = 0; i < res->count_crtcs; i++) {
        drmModeCrtc *crtc = drmModeGetCrtc(fd, res->crtcs[i]);
        printf("  CRTC[%d]: id=%d, pipe=%d\n",
               i, res->crtcs[i], i);
        if (crtc) {
            printf("    enabled=%d, fb_id=%d\n",
                   crtc->enabled, crtc->buffer_id);
            drmModeFreeCrtc(crtc);
        }
    }

    /* 遍历 Encoder */
    printf("\nEncoders (%d):\n", res->count_encoders);
    for (int i = 0; i < res->count_encoders; i++) {
        drmModeEncoder *enc = drmModeGetEncoder(fd, res->encoders[i]);
        if (enc) {
            printf("  Encoder[%d]: id=%d, type=%s, crtc_id=%d\n",
                   i, enc->encoder_id,
                   encoder_type_str(enc->encoder_type),
                   enc->crtc_id);
            printf("    possible_crtcs=0x%x, possible_clones=0x%x\n",
                   enc->possible_crtcs, enc->possible_clones);
            drmModeFreeEncoder(enc);
        }
    }

    /* 遍历 Connector */
    printf("\nConnectors (%d):\n", res->count_connectors);
    for (int i = 0; i < res->count_connectors; i++) {
        drmModeConnector *conn =
            drmModeGetConnector(fd, res->connectors[i]);
        if (conn) {
            printf("  Connector[%d]: id=%d, type=%s-%d\n",
                   i, conn->connector_id,
                   connector_type_str(conn->connector_type),
                   conn->connector_type_id);
            printf("    status=%s\n",
                   conn->connection == DRM_MODE_CONNECTED ?
                   "connected" : "disconnected");
            printf("    encoder_id=%d, modes=%d\n",
                   conn->encoder_id, conn->count_modes);
            drmModeFreeConnector(conn);
        }
    }

    /* 遍历 Plane */
    drmModePlaneRes *planes = drmModeGetPlaneResources(fd);
    if (planes) {
        printf("\nPlanes (%d):\n", planes->count_planes);
        for (int i = 0; i < planes->count_planes; i++) {
            drmModePlane *pl =
                drmModeGetPlane(fd, planes->planes[i]);
            if (pl) {
                printf("  Plane[%d]: id=%d\n", i, pl->plane_id);
                printf("    crtc_id=%d, fb_id=%d\n",
                       pl->crtc_id, pl->fb_id);
                printf("    possible_crtcs=0x%x\n",
                       pl->possible_crtcs);
                printf("    formats: ");
                for (uint32_t j = 0; j < pl->count_formats; j++) {
                    char fcc[5] = {0};
                    memcpy(fcc, &pl->formats[j], 4);
                    printf("%s ", fcc);
                }
                printf("\n");
                drmModeFreePlane(pl);
            }
        }
        drmModeFreePlaneResources(planes);
    }

    printf("\n=== End of report ===\n");
    drmModeFreeResources(res);
    close(fd);
    return 0;
}
```

编译运行：
```bash
gcc -o drm_components drm_components.c -ldrm
sudo ./drm_components
```

#### 2.4 使用 sysfs 分析组件绑定关系

```bash
# 查看连接器到 Encoder 的绑定
$ ls -la /sys/class/drm/card0-HDMI-A-1/
total 0
drwxr-xr-x  ...  .
drwxr-xr-x  ...  ..
-r--r--r--  ...  dpms
--w-------  ...  edid
-r--r--r--  ...  enabled
-r--r--r--  ...  modes
lrwxrwxrwx  ...  encoder -> ../../encoders/encoder-7
-r--r--r--  ...  status
--w-------  ...  trigger

# 查看 Encoder 绑定的 CRTC
$ ls -la /sys/kernel/debug/dri/0/encoder-7/
total 0
-r--r--r--  crtc -> ../../crtc-4/
-r--r--r--  name
-r--r--r--  type

# 查看 CRTC 绑定的 Plane
$ cat /sys/kernel/debug/dri/0/crtc-4/state
plane[4]: fb_id=34 crtc=(0,0) src=(0,0) dst=(0,0)
    format=XR24 size=1920x1080
plane[6]: fb_id=0 crtc=(0,0) src=(0,0) dst=(0,0)
    format=AR24 size=64x64
```

#### 2.5 IGT 测试用例：组件关系验证

```bash
# 运行 kms_plane 测试（验证 Plane 功能）
$ sudo igt-gpu-tools-build/tests/kms_plane --run-subtest plane-position-covered

# 运行 kms_plane_lowres（验证低分辨率 Plane 测试）
$ sudo igt-gpu-tools-build/tests/kms_plane_lowres

# 运行 kms_crtc_background（验证 CRTC 背景色）
$ sudo igt-gpu-tools-build/tests/kms_crtc_background

# 运行 kms_encoder（验证编码器）
$ sudo igt-gpu-tools-build/tests/kms_encoder

# 运行完整的连接器探测测试
$ sudo igt-gpu-tools-build/tests/kms_probe --run-subtest probe-edid
```

#### 2.6 Plane 格式与缩放能力测试

Plane 支持的像素格式和缩放能力是显示质量的关键因素。以下通过 C 程序测试 Plane 的格式支持：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

static const char *plane_type_name(unsigned int type) {
    switch (type) {
    case DRM_PLANE_TYPE_PRIMARY: return "Primary";
    case DRM_PLANE_TYPE_OVERLAY: return "Overlay";
    case DRM_PLANE_TYPE_CURSOR: return "Cursor";
    default: return "Unknown";
    }
}

static void print_fourcc(uint32_t format) {
    printf("%c%c%c%c",
           format & 0xFF,
           (format >> 8) & 0xFF,
           (format >> 16) & 0xFF,
           (format >> 24) & 0xFF);
}

int main(int argc, char **argv) {
    int fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);
    if (fd < 0) { perror("open"); return 1; }

    drmModePlaneRes *plane_res = drmModeGetPlaneResources(fd);
    if (!plane_res) { fprintf(stderr, "get plane res failed\n"); close(fd); return 1; }

    printf("=== Plane 格式与缩放能力测试 ===\n\n");
    printf("Total Planes: %d\n\n", plane_res->count_planes);

    for (uint32_t i = 0; i < plane_res->count_planes; i++) {
        drmModePlane *plane = drmModeGetPlane(fd, plane_res->planes[i]);
        if (!plane) continue;

        printf("Plane[%u]: id=%u\n", i, plane->plane_id);
        printf("  Type: %s\n", plane_type_name(plane->type));
        printf("  CRTC ID: %u\n", plane->possible_crtcs);
        printf("  Formats (%u): ", plane->count_formats);

        for (uint32_t j = 0; j < plane->count_formats; j++) {
            print_fourcc(plane->formats[j]);
            if (j < plane->count_formats - 1) printf(", ");
        }
        printf("\n");

        drmModeObjectProperties *props =
            drmModeObjectGetProperties(fd, plane->plane_id,
                                       DRM_MODE_OBJECT_PLANE);
        if (props) {
            for (uint32_t j = 0; j < props->count_props; j++) {
                drmModePropertyRes *prop =
                    drmModeGetProperty(fd, props->props[j]);
                if (prop) {
                    printf("  Prop: %s\n", prop->name);
                    drmModeFreeProperty(prop);
                }
            }
            drmModeFreeObjectProperties(props);
        }

        drmModeFreePlane(plane);
        printf("\n");
    }

    drmModeFreePlaneResources(plane_res);
    close(fd);
    return 0;
}
```

编译运行：
```bash
gcc -o plane_caps plane_caps.c -ldrm
./plane_caps
```

输出示例：
```
=== Plane 格式与缩放能力测试 ===

Total Planes: 3

Plane[0]: id=4
  Type: Primary
  CRTC ID: 1
  Formats (7): XR24, AR24, AR30, XR30, RG16, NV12, P010
  Prop: type
  Prop: FB_ID
  Prop: CRTC_ID
  Prop: CRTC_X
  Prop: CRTC_Y
  Prop: CRTC_W
  Prop: CRTC_H
  Prop: SRC_X
  Prop: SRC_Y
  Prop: SRC_W
  Prop: SRC_H
  Prop: alpha
  Prop: pixel blend mode
  Prop: rotation
  Prop: COLOR_ENCODING
  Prop: COLOR_RANGE
```

#### 2.7 MST 拓扑测试

DisplayPort MST 允许单个 DP 端口连接多个显示器。以下测试方法验证 MST 拓扑中的组件关系：

```bash
# 1. 连接 MST 集线器后查看拓扑
$ cat /sys/kernel/debug/dri/0/mst_topology
[DP-MST] root port bus: 0
  ├── Port 0: DP-1, DP-IN
  │   ├── Port 1: DP-2, connector 20 (MST branch 1)
  │   │   ├── Port 1: DP-3, connector 21 (sink: DELL U2719DC)
  │   │   └── Port 2: DP-4, connector 22 (sink: DELL P2419H)
  │   └── Port 2: DP-5, connector 23 (MST branch 2)
  │       └── Port 1: DP-6, connector 24 (sink: ASUS VG27AQ)
  └── Port 3: DP-7, connector 25 (direct attached sink)

# 2. 查看 MST 虚拟连接器的属性
$ modetest -M amdgpu -a | grep -A20 "connector 21"
21      20      connected       DP-3    ...
  props:
        DPMS: enum {On, Standby, Suspend, Off} = On
        link-status: enum {Good, Bad} = Good
        CRTC_ID: object = 4
        ...
        PATH: blob = "mst:0-1-1-1"
        ...

# 3. 测试 MST 热插拔
$ echo "mst: probe" | sudo tee /sys/kernel/debug/dri/0/mst/0/probe
$ sleep 2
$ dmesg | tail -20 | grep -E "MST|mst"

# 4. 枚举 MST 路径下的所有连接器
$ for path in /sys/class/drm/card0-*/; do
    conn=$(basename $path)
    cat $path/status 2>/dev/null | while read status; do
        if [ "$status" = "connected" ]; then
            echo "$conn: $status"
            cat $path/modes 2>/dev/null | head -3
        fi
    done
done

# 5. 使用 IGT 测试 MST
$ sudo igt-gpu-tools-build/tests/kms_dp_mst --run-subtest basic
$ sudo igt-gpu-tools-build/tests/kms_dp_mst --run-subtest hotplug
```

MST 拓扑中的组件映射关系：

```
┌─────────────────────────────────────────────────────────────────────┐
│               MST 拓扑中的 DRM 组件映射                               │
│                                                                     │
│  物理连接:                                                           │
│  ┌──────────┐    DP Cable     ┌──────────────┐                     │
│  │ GPU DP   │────────────────►│  MST Hub     │                     │
│  │ Port     │                 │  (Branch)    │                     │
│  └──────────┘                 └──────┬───────┘                     │
│                                      │                             │
│                  ┌───────────────────┼───────────────────┐         │
│                  ▼                   ▼                   ▼         │
│          ┌──────────────┐   ┌──────────────┐   ┌──────────────┐  │
│          │ 显示器 A    │   │ 显示器 B    │   │ 显示器 C    │  │
│          │ 1920x1080   │   │ 2560x1440   │   │ 3840x2160   │  │
│          └──────────────┘   └──────────────┘   └──────────────┘  │
│                                                                     │
│  DRM 组件映射:                                                       │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐                 │
│  │ CRTC 4     │   │ CRTC 5     │   │ CRTC 6     │                 │
│  │ Encoder 9  │   │ Encoder 9  │   │ Encoder 9  │  ← 共享 Encoder│
│  │ (DPMST)    │   │ (DPMST)    │   │ (DPMST)    │                 │
│  │ Conn 21    │   │ Conn 22    │   │ Conn 24    │                 │
│  │ (DP-MST)   │   │ (DP-MST)   │   │ (DP-MST)   │                 │
│  └────────────┘   └────────────┘   └────────────┘                 │
│                                                                     │
│  注意: 所有 MST 显示器共享同一个物理 Encoder(9)                      │
│        但各有一个独立的 CRTC 和 Virtual Connector                    │
└─────────────────────────────────────────────────────────────────────┘
```

MST 测试的关键注意点：
1. MST 拓扑变化（显示器接入/移除）会触发完整的拓扑重建
2. 虚拟连接器的生命周期由 MST 拓扑管理，不可手动创建
3. 所有 MST 显示器的模式设置需要通过原子 API 进行
4. Daisy-chain 拓扑中，中间节点的带宽会影响末端节点的可用分辨率

#### 2.8 VBlank 与页面翻转时序测量

VBlank 间隔是进行页面翻转和帧率控制的关键时间窗口。以下 C 程序测量 VBlank 时序：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/time.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

static double tv_to_double(struct timeval *tv) {
    return tv->tv_sec + tv->tv_usec / 1000000.0;
}

int main(int argc, char **argv) {
    int fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);
    if (fd < 0) { perror("open"); return 1; }

    drmVBlank vbl;
    struct timeval tv_start, tv_end;

    memset(&vbl, 0, sizeof(vbl));
    vbl.request.type = DRM_VBLANK_RELATIVE;
    vbl.request.sequence = 1;

    gettimeofday(&tv_start, NULL);
    int ret = drmWaitVBlank(fd, &vbl);
    gettimeofday(&tv_end, NULL);

    if (ret == 0) {
        double elapsed = tv_to_double(&tv_end) - tv_to_double(&tv_start);
        printf("VBlank 测量结果:\n");
        printf("  等待时间: %.2f ms\n", elapsed * 1000);
        printf("  返回序列: %u\n", vbl.reply.sequence);
        printf("  返回时间戳: %u.%03u ms\n",
               vbl.reply.tval_sec, vbl.reply.tval_usec / 1000);

        /* 计算帧率 */
        double frame_time = elapsed;
        if (frame_time > 0) {
            printf("  估计帧率: %.2f fps\n", 1.0 / frame_time);
        }
    } else {
        perror("drmWaitVBlank failed");
    }

    /* 连续测量 60 帧的统计 */
    printf("\n连续 60 帧 VBlank 间隔统计:\n");
    double min_gap = 999, max_gap = 0, sum_gap = 0;
    uint32_t prev_seq = 0;

    for (int i = 0; i < 60; i++) {
        memset(&vbl, 0, sizeof(vbl));
        vbl.request.type = DRM_VBLANK_RELATIVE;
        vbl.request.sequence = 1;

        struct timeval t1, t2;
        gettimeofday(&t1, NULL);
        drmWaitVBlank(fd, &vbl);
        gettimeofday(&t2, NULL);

        double gap = tv_to_double(&t2) - tv_to_double(&t1);
        if (gap < min_gap) min_gap = gap;
        if (gap > max_gap) max_gap = gap;
        sum_gap += gap;

        if (i > 0) {
            uint32_t seq_diff = vbl.reply.sequence - prev_seq;
            printf("  帧 %2d: 序列 %6u → %6u (间隔: %.2f ms, seq_diff: %u)\n",
                   i, prev_seq, vbl.reply.sequence,
                   gap * 1000, seq_diff);
        }
        prev_seq = vbl.reply.sequence;
    }

    double avg_gap = sum_gap / 60.0;
    double jitter = max_gap - min_gap;
    printf("\n统计结果:\n");
    printf("  最小间隔: %.2f ms (%.2f fps)\n", min_gap * 1000, 1.0/min_gap);
    printf("  最大间隔: %.2f ms (%.2f fps)\n", max_gap * 1000, 1.0/max_gap);
    printf("  平均间隔: %.2f ms (%.2f fps)\n", avg_gap * 1000, 1.0/avg_gap);
    printf("  Jitter (抖动): %.2f ms\n", jitter * 1000);

    close(fd);
    return 0;
}
```

编译运行：
```bash
gcc -o vblank_measure vblank_measure.c -ldrm
./vblank_measure
```

输出示例：
```
VBlank 测量结果:
  等待时间: 16.68 ms
  返回序列: 584291
  返回时间戳: 7.123 ms
  估计帧率: 59.95 fps

连续 60 帧 VBlank 间隔统计:
  帧  1: 序列 584291 → 584292 (间隔: 16.67 ms, seq_diff: 1)
  帧  2: 序列 584292 → 584293 (间隔: 16.68 ms, seq_diff: 1)
  ...
  帧 59: 序列 584349 → 584350 (间隔: 16.66 ms, seq_diff: 1)

统计结果:
  最小间隔: 16.62 ms (60.12 fps)
  最大间隔: 16.78 ms (59.59 fps)
  平均间隔: 16.67 ms (59.99 fps)
  Jitter (抖动): 0.16 ms
```

VBlank 时序测量的意义：
- 验证显示刷新率是否符合预期（60Hz 应约为 16.67ms）
- 检测帧率抖动（Jitter）是否在可接受范围内
- 发现驱动或硬件层面的时序异常（如序列号跳跃表示丢帧）
- 为页面翻转（Page Flip）的时间窗口优化提供依据

### 3. 案例分析

#### 3.1 CRTC 不足导致的多显示器失败

**Bug 背景**：freedesktop.org bug #108817 报告了当连接四个显示器时，某些显示器无法点亮的问题。系统使用 AMDGPU 驱动，GPU 支持最多 6 个显示输出，但驱动的 CRTC 分配策略存在问题。

**根因分析**：虽然硬件 DC 支持 6 个 CRTC（pipe 0-5），但 amdgpu_dm 在初始化时创建的 DRM CRTC 数量与 DC 流（stream）数量不匹配。具体问题出在 `amdgpu_dm_crtc_init()` 中，它根据 `dc->caps.max_streams` 创建 CRTC，但在某些硬件配置下该值被错误报告为 3。

```c
/* drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_crtc.c */
static int amdgpu_dm_crtc_init(struct amdgpu_display_manager *dm,
                               struct drm_plane *plane,
                               uint32_t crtc_index)
{
    /* ... */
    /* Bug: 使用 dc->caps.max_streams 限制 CRTC 数量 */
    if (crtc_index >= dm->dc->caps.max_streams) {
        DRM_ERROR("CRTC index %d exceeds max streams %d\n",
                  crtc_index, dm->dc->caps.max_streams);
        return -EINVAL;
    }
    /* ... */
}
```

**参考链接**：https://gitlab.freedesktop.org/drm/amd/-/issues/108817

**修复方法**：AMD 开发者修复了 `dc->caps.max_streams` 的初始化值，确保其正确反映硬件能力。同时，在 CRTC 分配算法中增加了回退机制，当首选 CRTC 不可用时，尝试分配其他可用 CRTC。

**测试方法**：
```bash
# 连接 4 个显示器后检查 CRTC 分配
$ for card in /sys/class/drm/card0-*/; do
    echo "--- $card ---"
    cat $card/status
    cat $card/enabled 2>/dev/null || echo "N/A"
done

# 检查 dmesg 中的 CRTC 分配信息
$ dmesg | grep -E "CRTC|crtc|pipe" | grep -i amdgpu
```

#### 3.2 Plane Alpha 合成 Bug

**Bug 背景**：在 AMDGPU 驱动中，当多个 Overlay Plane 同时启用并设置了不同的 Alpha 值时，合成结果不正确。具体表现为上层 Plane 的透明部分被渲染为黑色而非显示下层 Plane 的内容。

**根因分析**：问题出在 DC 的 MPC（Multiple Pipe Combined）合成器配置中。当 Plane 数量超过 2 时，MPC 的 alpha 混合模式配置错误，将 PRE_MULTIPLIED_ALPHA 与 PER_PIXEL_ALPHA 混用。

```c
/* drivers/gpu/drm/amd/display/dc/dcn10/dcn10_mpc.c */
void mpc1_set_ocsc(struct mpc *mpc, int mpcc_id,
                   struct mpcc_cfg *cfg)
{
    struct dcn10_mpc *mpc10 = TO_DCN10_MPC(mpc);

    /* Bug: 未正确设置混合模式 */
    REG_UPDATE(MUX[mpcc_id], MPC_OUT_RATE_CNTL,
               MPC_OUT_RATE, cfg->rate);

    /* 修复: 根据 Plane 数量设置正确的混合模式 */
    if (cfg->num_planes > 2) {
        REG_UPDATE(MUX[mpcc_id], MPC_OUT_ALPHA_MODE,
                   MPC_OUT_ALPHA, PER_PIXEL_ALPHA);
    }
}
```

**测试方法**：
```bash
# 使用 IGT 测试多层 Plane 合成
$ sudo igt-gpu-tools-build/tests/kms_plane_alpha_blending

# 手动测试：使用 Weston 合成器
$ weston --backend=drm-backend.so --use-pixman
# 在 Weston 中打开多个窗口，观察透明效果
```

### 4. 实践操作：组件关系调试脚本

```bash
#!/bin/bash
# drm_components_debug.sh - DRM 组件关系调试工具

echo "DRM 显示组件关系分析报告"
echo "=========================="
echo "生成时间: $(date)"
echo ""

DRM_DIR="/sys/class/drm"

# 1. 列出所有 DRM 设备
echo "1. DRM 设备列表:"
ls $DRM_DIR/*/status 2>/dev/null | while read f; do
    dev="${f%/status}"
    dev_name=$(basename $dev)
    status=$(cat $f 2>/dev/null)
    enabled=$(cat $dev/enabled 2>/dev/null)
    echo "   $dev_name: status=$status, enabled=$enabled"
done

echo ""

# 2. 分析每个连接器的详细信息
echo "2. 连接器详细信息:"
for connector_dir in $DRM_DIR/card0-*; do
    [ -d "$connector_dir" ] || continue
    name=$(basename $connector_dir)
    echo "--- $name ---"

    # 连接器 ID
    echo "   Connector ID: $(cat $connector_dir/connector_id 2>/dev/null)"

    # 状态
    echo "   Status: $(cat $connector_dir/status 2>/dev/null)"

    # Encoder 绑定
    if [ -L "$connector_dir/encoder" ]; then
        encoder_target=$(readlink $connector_dir/encoder)
        echo "   Encoder: $encoder_target"
    fi

    # 支持的模式
    echo "   Supported modes:"
    cat $connector_dir/modes 2>/dev/null | while read mode; do
        echo "     - $mode"
    done

    # 物理尺寸
    if [ -f "$connector_dir/edid" ]; then
        size=$(hexdump -s 66 -n 2 -e '"%d mm"' $connector_dir/edid 2>/dev/null)
        echo "   Physical size: $size"
    fi

    echo ""
done

# 3. 列出所有 CRTC
echo "3. CRTC 信息:"
for crtc_dir in /sys/kernel/debug/dri/0/crtc-*/; do
    [ -d "$crtc_dir" ] || continue
    name=$(basename $crtc_dir)
    echo "   $name:"
    cat $crtc_dir/state 2>/dev/null | while read line; do
        echo "     $line"
    done
    echo ""
done

# 4. 使用 modetest 获取关系摘要
echo "4. modetest 组件关系摘要:"
modetest -M amdgpu 2>/dev/null | grep -E "CRTC|Encoder|Connector|Plane" | head -40

echo ""
echo "分析完成。"
```

运行：
```bash
chmod +x drm_components_debug.sh
sudo ./drm_components_debug.sh
```

### 5. 原子属性查看与 Plane 测试

```bash
# 使用 modetest 的原子模式查看所有属性
$ modetest -M amdgpu -a --atomic --properties

# 查看特定 Plane 的属性
$ modetest -M amdgpu -a | grep -A30 "plane 4"
id: 4
    type: Primary
    formats: XR24 AR24 AR30 XR30 RG16 NV12 P010
    objects: (crtc_id) (fb_id) (crtc_x) (crtc_y) (crtc_w) (crtc_h)
             (src_x) (src_y) (src_w) (src_h)
    props:
        type: enum {Primary, Overlay, Cursor} = Primary
        FB_ID: object = 34
        CRTC_ID: object = 4
        CRTC_X: signed range [0, 32767] = 0
        CRTC_Y: signed range [0, 32767] = 0
        CRTC_W: signed range [0, 32767] = 1920
        CRTC_H: signed range [0, 32767] = 1080
        SRC_X: fixed range [0, 4294967295] = 0
        SRC_Y: fixed range [0, 4294967295] = 0
        SRC_W: fixed range [0, 4294967295] = 1920
        SRC_H: fixed range [0, 4294967295] = 1080
        alpha: range [0, 65535] = 65535
        pixel blend mode: enum {None, Pre-multiplied, Coverage} = Pre-multiplied
        rotation: bitmask {rotate-0, rotate-90, rotate-180, rotate-270,
                          reflect-x, reflect-y} = rotate-0
        COLOR_ENCODING: enum {ITU-R BT.601, ITU-R BT.709, ...} = ITU-R BT.709
        COLOR_RANGE: enum {YCbCr limited range, YCbCr full range} = limited
```

## 相关链接

- Linux DRM CRTC 文档：https://www.kernel.org/doc/html/latest/gpu/drm-kms.html#crtc
- Linux DRM Encoder 文档：https://www.kernel.org/doc/html/latest/gpu/drm-kms.html#encoder
- Linux DRM Connector 文档：https://www.kernel.org/doc/html/latest/gpu/drm-kms.html#connector
- Linux DRM Plane 文档：https://www.kernel.org/doc/html/latest/gpu/drm-kms.html#plane
- AMDGPU DC 显示核心源码：drivers/gpu/drm/amd/display/
- AMDGPU DM CRTC 初始化：drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_crtc.c
- AMDGPU DM Connector 管理：drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c
- AMDGPU DM Plane 管理：drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_plane.c
- DRM 核心 CRTC 源码：drivers/gpu/drm/drm_crtc.c
- DRM 核心 Encoder 源码：drivers/gpu/drm/drm_encoder.c
- DRM 核心 Connector 源码：drivers/gpu/drm/drm_connector.c
- DRM 核心 Plane 源码：drivers/gpu/drm/drm_plane.c
- drm_info 工具：https://gitlab.freedesktop.org/emersion/drm_info
- IGT GPU Tools：https://gitlab.freedesktop.org/drm/igt-gpu-tools
- FreeDesktop AMDGPU Issue #108817：https://gitlab.freedesktop.org/drm/amd/-/issues/108817
- DisplayPort 标准：https://www.vesa.org/displayport/
- HDMI 标准：https://www.hdmi.org/spec/index
- AMD GPUOpen 显示文档：https://gpuopen.com/learn/
- Weston 合成器：https://gitlab.freedesktop.org/wayland/weston
- VESA CVT 定时标准：https://vesa.org/vesa-cvt/
- Linux DRM KMS API 手册：https://drm.pages.freedesktop.org/libdrm/
- Xorg 模式设置调试：https://www.x.org/wiki/Development/Documentation/DDX/modesetting/
- Linux DRM 开发者指南：https://www.kernel.org/doc/html/latest/gpu/index.html
- AMD DC 显示核心设计文档：https://www.kernel.org/doc/html/latest/gpu/amdgpu/display/display.html
- IGT GPU Tools 测试指南：https://gitlab.freedesktop.org/drm/igt-gpu-tools/-/wikis/home
- DRM 原子模式设置 API：https://www.kernel.org/doc/html/latest/gpu/drm-kms.html#atomic-mode-setting
- Planes 属性文档：https://www.kernel.org/doc/html/latest/gpu/drm-kms.html#plane-composition-properties
- Linux DRM 调试指南：https://www.kernel.org/doc/html/latest/gpu/drm-debugging.html
- AMDGPU 显示驱动文档：https://www.kernel.org/doc/html/latest/gpu/amdgpu/display/amdgpu-dm.html
- VESA DisplayPort 标准文档：https://www.vesa.org/displayport-overview/

## 今日小结

今天深入学习了 DRM 显示架构的四大核心组件：CRTC（负责时序生成、VBlank 中断和帧缓冲读取）、Encoder（负责信号编码和协议封装）、Connector（负责物理接口管理和 EDID 读取）和 Plane（负责硬件图层合成和 Alpha 混合）。理解了它们之间的层次关系和数据流：Framebuffer → Plane → Blender → CRTC → Encoder → Connector → Monitor。掌握了 `possible_crtcs`、`possible_encoders` 掩码机制在组件兼容性描述中的作用。了解了 AMDGPU 驱动中通过 amdgpu_dm 层将 DRM 组件与 DC 硬件抽象层对接的实现方式。通过 modetest、drm_info、sysfs 和 C 程序等多种实践手段，学会了查看和分析组件的绑定关系与属性配置。还深入探讨了 Plane 合成细节、Encoder 实现差异、组件生命周期管理，以及 MST 拓扑和 VBlank 时序测量等进阶主题。

## 扩展思考

1. 在 AMDGPU 的 DC 架构中，DRM 的 CRTC 概念与 DC 的 Stream 和 OTG 之间存在怎样的映射关系？一个 DRM CRTC 是否总是对应一个 OTG（Output Timing Generator）硬件单元？考虑如何通过 DC 的 pipe 索引来追踪这种映射关系。

2. Plane 的硬件合成能力受限于 GPU 的硬件设计。不同的 AMD GPU 代际（如 DCN 2.0 vs DCN 3.0）在支持的 Plane 数量、像素格式和缩放能力上存在差异。如何设计一个测试框架来自动检测硬件 Plane 的能力并相应地调整测试用例？

3. 在 DisplayPort MST 拓扑中，一个 CRTC 和 Encoder 可以驱动多个显示器。这种虚拟连接器（Virtual Connector）在 DRM 中的实现与传统的物理连接器有何不同？测试 MST 场景时需要考虑哪些特殊的组件绑定关系？

4. Alpha 混合模式在硬件和软件实现上有哪些差异？PER_PIXEL_ALPHA 与 GLOBAL_ALPHA 在 MPC 硬件单元中的处理方式有何不同？设计一个测试用例来验证这两种模式的混合结果是否正确。

5. 考虑一个复杂场景：GPU 有 6 个 CRTC、4 个 Encoder（2x TMDS + 2x DPMST）和 8 个 Connector（2x HDMI + 2x DP + 2x DP-MST + 1x eDP + 1x USB-C）。当所有端口都连接显示器时，possible_crtcs 和 possible_encoders 掩码如何分配？哪些场景下会出现 CRTC 或 Encoder 资源不足的问题？
