# 第326天：KMS（Kernel Mode Setting）基础概念

## 学习目标
- 理解 Kernel Mode Setting（KMS）的核心概念及其在 Linux 图形栈中的位置
- 掌握 KMS 与用户空间 Mode Setting（UMS）的区别与演进历史
- 了解 KMS 的初始化流程、关键数据结构与内核 API
- 能够通过 modetest 等工具验证 KMS 的基本功能
- 掌握 AMDGPU 驱动中 KMS 的实现入口
- 理解 CRTC 时序、帧缓冲格式、VBlank 同步等核心机制

## 知识详解

### 1. 概念原理

#### 1.1 什么是 Kernel Mode Setting

Kernel Mode Setting（KMS）是 Linux 内核提供的一套显示输出管理机制，它将显示模式设置（如分辨率、刷新率、像素格式）的控制权从用户空间移到内核空间。KMS 是 DRM（Direct Rendering Manager）子系统的一部分，位于 `drivers/gpu/drm/` 目录下。

在 KMS 出现之前，Linux 使用 User Space Mode Setting（UMS），即由 X Server 等用户空间程序直接操作显卡寄存器来设置显示模式。这种方式存在多个问题：

- 多用户空间程序同时操作寄存器导致的竞态条件
- 系统启动过程中显示输出的闪烁/黑屏
- 虚拟终端切换时的显示不一致
- 电源管理状态切换时的显示恢复不可靠

KMS 在 2008 年左右随 Linux 内核 2.6.26 引入，由 Intel 和 AMD 等厂商推动。AMD GPU 的 KMS 支持最早出现在 Radeon 驱动中，后来演变为当前的 AMDGPU 驱动。

以下为 Linux 图形栈中 KMS 的位置示意图：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           User Space                                         │
│                                                                              │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐                │
│  │   Xorg    │  │  Wayland  │  │  Weston   │  │  Other    │                │
│  │  (X Server)│  │ Compositor│  │           │  │  Apps     │                │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘                │
│        │               │               │              │                      │
│        └───────────────┴───────────────┴──────────────┘                      │
│                              │                                               │
│  ┌───────────────────────────┴───────────────────────────────────────────┐  │
│  │                     libdrm (drm library)                               │  │
│  │  drmModeGetResources, drmModeSetCrtc, drmModeAddFB, ...               │  │
│  └───────────────────────────┬───────────────────────────────────────────┘  │
├──────────────────────────────┼──────────────────────────────────────────────┤
│                       Kernel Space                                          │
│                              │                                               │
│  ┌───────────────────────────┴───────────────────────────────────────────┐  │
│  │                    DRM Core (drm.ko)                                    │  │
│  │                                                                         │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────────┐   │  │
│  │  │   GEM (Graphics │  │   KMS (Kernel   │  │   DRM Misc          │   │  │
│  │  │   Execution     │  │   Mode Setting) │  │   (syncobj, ...)    │   │  │
│  │  │   Manager)      │  │                 │  │                      │   │  │
│  │  └─────────────────┘  └─────────────────┘  └──────────────────────┘   │  │
│  │                              │                                         │  │
│  │  ┌───────────────────────────┴─────────────────────────────────────┐  │  │
│  │  │               Driver Callbacks (struct drm_mode_config_funcs)   │  │  │
│  │  │  fb_create, output_poll_changed, atomic_check, atomic_commit   │  │  │
│  │  └───────────────────────────┬─────────────────────────────────────┘  │  │
│  └──────────────────────────────┼─────────────────────────────────────────┘  │
│                                 │                                            │
│  ┌──────────────────────────────┴─────────────────────────────────────────┐  │
│  │                     GPU Driver (amdgpu.ko)                              │  │
│  │                                                                         │  │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────────┐   │  │
│  │  │   AMD KMS (DM)   │  │   AMD GEM        │  │   AMD Display     │   │  │
│  │  │   (amdgpu_dm.c)  │  │   (amdgpu_gem.c) │  │   Core (DC)       │   │  │
│  │  └──────────────────┘  └──────────────────┘  └────────────────────┘   │  │
│  │                              │                                         │  │
│  │  ┌───────────────────────────┴─────────────────────────────────────┐  │  │
│  │  │                Display Core Next (DCN) Hardware Layer            │  │  │
│  │  │  OTG, DPP, MPC, OPP, DIO, HUBP, ...                             │  │  │
│  │  └──────────────────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                              │                                               │
├──────────────────────────────┼──────────────────────────────────────────────┤
│                        Hardware                                              │
│                              │                                               │
│  ┌───────────────────────────┴───────────────────────────────────────────┐  │
│  │                    AMD GPU with Display Controller                     │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────────┐   │  │
│  │  │  OTG     │  │  DPP     │  │  MPC     │  │  Display Output    │   │  │
│  │  │ (Timing) │→│(Frontend)│→│ (Blend)  │→│  (DIO + PHY)        │   │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┬─────────┘   │  │
│  │                                                        │              │  │
│  │                                                        ▼              │  │
│  │                                              ┌──────────────────┐    │  │
│  │                                              │  HDMI / DP / USB-C│   │  │
│  │                                              └──────────────────┘    │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 1.2 KMS 核心概念

KMS 围绕以下几个核心抽象概念组织：

**CRTC (Cathode Ray Tube Controller)**
历史遗留命名，实际上表示显示控制器（Display Controller）。CRTC 负责生成显示时序信号（水平同步、垂直同步、像素时钟），从帧缓冲中读取像素数据并发送到编码器。在现代 AMD GPU 中，CRTC 对应于 OTG（Output Timing Generator）。

**Encoder（编码器）**
将 CRTC 输出的像素数据转换为特定显示接口所需的信号格式。例如，将数字 RGB 信号转换为 TMDS（HDMI/DVI）或 DisplayPort 差分信号。

**Connector（连接器）**
物理显示接口的抽象，如 HDMI、DisplayPort、VGA、DVI-D、USB-C 等。Connector 代表物理端口，连接到显示器。

**Plane（平面）**
显示平面的抽象。每个 CRTC 可以叠加多个平面（主平面、游标平面、覆盖平面），由硬件混合器（MPC）合成最终图像。

以下为这四个核心对象的层次关系图：

```
┌──────────────────────────────────────────────────────────────────┐
│                    DRM KMS 核心对象层次关系                        │
│                                                                  │
│   ┌─────────────┐          ┌─────────────┐                      │
│   │   Framebuffer│         │   Framebuffer│                      │
│   │   (FB 1)    │         │   (FB 2)    │                      │
│   └──────┬──────┘         └──────┬──────┘                      │
│          │                       │                              │
│          ▼                       ▼                              │
│   ┌─────────────┐          ┌─────────────┐                      │
│   │   Plane 0   │          │   Plane 1   │                      │
│   │ (Primary)   │          │ (Overlay)   │                      │
│   └──────┬──────┘          └──────┬──────┘                      │
│          │                       │                              │
│          └───────────┬───────────┘                              │
│                      │                                          │
│                      ▼                                          │
│   ┌─────────────────────────────────────────────┐              │
│   │                CRTC (Controller)             │              │
│   │  • 生成时序信号                               │              │
│   │  • 读取帧缓冲像素数据                          │              │
│   │  • 管理 VBlank 中断                           │              │
│   └──────────────────────┬──────────────────────┘              │
│                          │                                      │
│                          ▼                                      │
│   ┌─────────────────────────────────────────────┐              │
│   │               Encoder (编码器)               │              │
│   │  • 将像素数据转换为接口信号                    │              │
│   │  • 管理链路训练 (DP)                         │              │
│   │  • 控制输出使能                              │              │
│   └──────────────────────┬──────────────────────┘              │
│                          │                                      │
│                          ▼                                      │
│   ┌─────────────────────────────────────────────┐              │
│   │              Connector (连接器)               │              │
│   │  • 物理端口抽象 (HDMI/DP/USB-C)               │              │
│   │  • EDID 读取与解析                            │              │
│   │  • 热插拔检测                                 │              │
│   │  • 显示设备状态管理                            │              │
│   └───────────────────────────────────────────────┘              │
│                                                                  │
│   可能的连接拓扑:                                                │
│   CRTC ──→ Encoder ──→ Connector ──→ Monitor                    │
│   CRTC ──→ Encoder ──→ Encoder ──→ Connector ──→ Monitor        │
│   CRTC ──→ Encoder ──→ Connector ──→ Connector ──→ Monitor      │
│   (Encoder 和 Connector 之间可能是一对一或一对多关系)            │
└──────────────────────────────────────────────────────────────────┘
```

#### 1.3 显示时序详解

CRTC 的核心功能之一是生成精确的显示时序。显示时序定义了如何将像素数据从帧缓冲传输到显示设备，包括水平同步、垂直同步、消隐区间和前廊/后廊等参数。

以下为显示时序的详细示意图：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        水平扫描时序 (Horizontal Timing)                       │
│                                                                              │
│   ┌─────────┐                                                               │
│   │  HSYNC  │         ┌────────────────────┐                                │
│   │         │         │                    │                                │
│   │         │         │                    │                                │
│   │         │         │    显示区域         │                                │
│   │         │         │   (hdisplay)       │                                │
│   │         │         │                    │                                │
│   │         │         │                    │                                │
│   └─────────┴─────────┴────────────────────┴────────────────────────────────┘
│   ↑        ↑              ↑                    ↑                             │
│   │        │              │                    │                             │
│   │   hsync_start   hsync_end            htotal                              │
│   │                                                                          │
│   │← hsync_len →←  hbp  →←   hdisplay    →←  hfp  →                        │
│   │             (后廊)        (有效像素)      (前廊)                          │
│                                                                              │
│                                                                              │
│                        垂直扫描时序 (Vertical Timing)                         │
│                                                                              │
│   ┌─────────┐                                                               │
│   │  VSYNC  │                                                               │
│   │         │                                                               │
│   │         │                                                               │
│   │         │                     ┌────────────────────┐                    │
│   │         │                     │                    │                    │
│   │         │                     │    有效显示行      │                    │
│   │         │                     │   (vdisplay)       │                    │
│   │         │                     │                    │                    │
│   │         │                     │                    │                    │
│   └─────────┴─────────────────────┴────────────────────┴────────────────────┘
│   ↑        ↑              ↑                    ↑                             │
│   │        │              │                    │                             │
│   │   vsync_start   vsync_end            vtotal                              │
│   │                                                                          │
│   │← vsync_len →←  vbp  →←   vdisplay    →←  vfp  →                        │
│   │             (后廊)        (有效行)        (前廊)                          │
└─────────────────────────────────────────────────────────────────────────────┘

各参数含义:
  hdisplay: 水平有效像素数（如1920）
  hsync_start: 水平同步起始位置 = hdisplay + hfp
  hsync_end: 水平同步结束位置 = hdisplay + hfp + hsync_len
  htotal: 水平总像素数 = hdisplay + hfp + hsync_len + hbp
  vdisplay: 垂直有效行数（如1080）
  vsync_start: 垂直同步起始位置 = vdisplay + vfp
  vsync_end: 垂直同步结束位置 = vdisplay + vfp + vsync_len
  vtotal: 垂直总线数 = vdisplay + vfp + vsync_len + vbp

刷新率计算公式:
  刷新率 (Hz) = pixel_clock (Hz) / (htotal * vtotal)

  例如: 1920x1080 @ 60Hz
  pixel_clock = 148500 kHz = 148500000 Hz
  htotal = 2200, vtotal = 1125
  刷新率 = 148500000 / (2200 * 1125) = 60.00 Hz
```

#### 1.4 CRTC 时序计算示例

以下展示几个常见显示模式的时序参数：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    常见显示模式时序参数表 (VESA 标准)                         │
│                                                                              │
│  模式           像素时钟    hdisp  hfp  hsync  hbp  htotal  vdisp  vfp  vsync│
│                  (MHz)                         hbp+hsync          vbp+vsync │
│  ─────────────────────────────────────────────────────────────────────────── │
│  640x480 @ 60Hz  25.175    640    16     96   48    800     480   11     2  │
│  800x600 @ 60Hz  40.000    800    40    128   88   1056     600    1     4  │
│  1024x768 @ 60Hz 65.000    1024   24    136  160   1344     768    3     6  │
│  1280x720 @ 60Hz 74.250    1280  110     40  220   1650     720    5     5  │
│  1280x1024 @ 60Hz 108.00   1280   48    112  248   1688    1024    1     3  │
│  1920x1080 @ 60Hz 148.50   1920   88     44  148   2200    1080    4     5  │
│  1920x1080 @ 50Hz 148.50   1920  528     44  148   2640    1080    4     5  │
│  1920x1080 @ 30Hz 74.250   1920   88     44  148   2200    1080    4     5  │
│  1920x1200 @ 60Hz 154.00   1920   48    112  248   2328    1200    1     3  │
│  2560x1440 @ 60Hz 241.50   2560   88     44  148   2840    1440    4     5  │
│  3840x2160 @ 60Hz 594.00   3840  176     88  296   4400    2160    8    10  │
│  3840x2160 @ 30Hz 297.00   3840  176     88  296   4400    2160    8    10  │
│                                                                              │
│  注: 上表参数来源于 VESA Coordinated Video Timing (CVT) 标准                   │
│  和 CEA-861 标准。实际驱动实现中可能略有差异。                               │
│  参考: https://vesa.org/vesa-modes/                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 1.5 帧缓冲与像素格式

KMS 使用帧缓冲（Framebuffer）作为显示内容的来源。帧缓冲是一块连续的内存区域，存储了要显示的像素数据。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        帧缓冲 (Framebuffer) 结构                              │
│                                                                              │
│  帧缓冲是一个抽象对象，关联到 Planes，存储要显示的内容。                       │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────┐        │
│  │                      Framebuffer Object                          │        │
│  │                                                                  │        │
│  │  ┌──────────────────────────────────────────────────────────┐   │        │
│  │  │  struct drm_framebuffer {                                │   │        │
│  │  │      struct drm_device *dev;     // 所属 DRM 设备        │   │        │
│  │  │      uint32_t width;             // 帧缓冲宽度(像素)     │   │        │
│  │  │      uint32_t height;            // 帧缓冲高度(像素)     │   │        │
│  │  │      uint32_t pitches[4];        // 每行字节数(每平面)   │   │        │
│  │  │      uint32_t offsets[4];        // 平面偏移             │   │        │
│  │  │      uint64_t modifier;          // 内存布局修饰符       │   │        │
│  │  │      struct drm_gem_object *obj[4]; // GEM 对象指针     │   │        │
│  │  │      const struct drm_framebuffer_funcs *funcs;          │   │        │
│  │  │  };                                                      │   │        │
│  │  └──────────────────────────────────────────────────────────┘   │        │
│  └──────────────────────────────────────────────────────────────────┘        │
│                                                                              │
│  常见像素格式 (DRM FourCC 编码):                                            │
│  ───────────────────────────────────────────────────────────────             │
│                                                                              │
│  格式编码    描述                   每像素位数  布局                         │
│  ────────    ────                   ────────   ────                         │
│  XR24        不透明 RGB 24-bit      32-bit     Packed: X8R8G8B8             │
│  AR24        透明 RGB 24-bit        32-bit     Packed: A8R8G8B8             │
│  XR12        不透明 RGB 12-bit      16-bit     Packed: X1R5G5B5             │
│  RG16        RGB 565                16-bit     Packed: R5G6B5               │
│  NV12        YUV 420 2-plane        12-bit     Planar: Y + UV              │
│  P010        YUV 420 10-bit 2-plane  15-bit    Planar: Y + UV (HDR)        │
│  AR30        透明 RGB 30-bit (10:10:10:2)  32-bit  Packed: A2R10G10B10     │
│  AB30        透明 RGB 30-bit (10:10:10:2)  32-bit  Packed: A2B10G10R10     │
│                                                                              │
│  modifier (内存布局修饰符) 说明:                                             │
│  DRM_FORMAT_MOD_LINEAR          = 0:  线性布局 (逐行连续)                   │
│  DRM_FORMAT_MOD_VENDOR_AMD      = 0x08: AMD 厂商 ID                        │
│  AMD 常用修饰符:                                                             │
│    AMD_FMT_MOD: tile 模式 (GFX9+, DCN 使用)                                 │
│    - 256B_2D: 256 字节块的 2D 平铺                                          │
│    - 4KB_2D: 4KB 块的 2D 平铺                                               │
│    - 64KB_2D: 64KB 块的 2D 平铺                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

帧缓冲的创建通过用户空间调用 `DRM_IOCTL_MODE_ADDFB2` ioctl 完成。内核中对应的处理函数链如下：

```
用户空间调用 drmModeAddFB2() 
  → DRM_IOCTL_MODE_ADDFB2 (drm_mode_addfb2)
    → drm_internal_framebuffer_create()
      → drm_mode_config_funcs->fb_create() (驱动回调)
        → amdgpu_dm_fb_create() (AMDGPU KMS)
          → amdgpu_dm_verify_plane() (验证像素格式)
          → amdgpu_gem_object_create() (分配 GEM 对象)
          → drm_framebuffer_init() (初始化帧缓冲)
```

常见的帧缓冲内存分配方式：

```c
// 用户空间分配 dumb buffer
#include <xf86drm.h>
#include <xf86drmMode.h>

int fd = open("/dev/dri/card0", O_RDWR);
struct drm_mode_create_dumb create = {0};
create.width = 1920;
create.height = 1080;
create.bpp = 32;

// 创建 dumb buffer
drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);

// create.handle, create.pitch, create.size 现在包含有效值

// 将 dumb buffer 添加到帧缓冲
uint32_t fb_id;
drmModeAddFB2(fd, 1920, 1080, DRM_FORMAT_XRGB8888,
              &create.handle, &create.pitch, &create.offset,
              &fb_id, DRM_MODE_FB_MODIFIERS);

// 映射到用户空间
struct drm_mode_map_dumb map = {0};
map.handle = create.handle;
drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map);

uint32_t *vaddr = mmap(0, create.size, PROT_READ | PROT_WRITE,
                       MAP_SHARED, fd, map.offset);

// 填充像素数据 (红色渐变)
for (int y = 0; y < 1080; y++) {
    for (int x = 0; x < 1920; x++) {
        vaddr[y * (create.pitch / 4) + x] = 
            0xFF000000 | (x * 255 / 1920) << 16 | 
            (y * 255 / 1080) << 8;
    }
}
```

#### 1.6 KMS 数据结构

在 DRM 内核代码中，KMS 的核心数据结构定义在 `include/drm/drm_modeset.h` 和 `include/drm/drm_crtc.h` 中。以下列出最关键的结构体：

```c
/* DRM 模式配置核心结构体 - 表示一个显示配置状态 */
struct drm_mode_config {
    struct mutex mutex;            /* 配置互斥锁 */
    struct drm_device *dev;        /* 所属 DRM 设备 */
    
    /* 核心对象 IDR 表 */
    struct idr crtc_idr;           /* CRTC ID 分配器 */
    struct idr tile_idr;           /* Tile group ID */
    
    /* 最小/最大尺寸限制 */
    uint32_t min_width, max_width; /* 帧缓冲最小/最大宽度 */
    uint32_t min_height, max_height; /* 帧缓冲最小/最大高度 */
    
    /* 函数指针表 - 驱动需要实现的回调 */
    const struct drm_mode_config_funcs *funcs;
    
    /* 辅助结构 */
    struct drm_mode_config_helper_funcs *helper_private;
    
    /* 对象链接列表 */
    struct list_head crtc_list;     /* 所有 CRTC 的链表 */
    struct list_head connector_list; /* 所有 Connector 的链表 */
    struct list_head encoder_list;  /* 所有 Encoder 的链表 */
    struct list_head plane_list;    /* 所有 Plane 的链表 */
    
    /* 属性相关 */
    struct list_head property_list; /* 所有属性的链表 */
    
    /* 输出轮询 */
    bool poll_enabled;
    bool poll_running;
    struct delayed_work output_poll_work; /* 热插拔轮询工作项 */
    
    /* 帧缓冲相关 */
    struct list_head fb_list;       /* 帧缓冲链表 */
    struct idr fb_idr;              /* 帧缓冲 ID 分配器 */
    
    /* 原子操作支持 */
    bool atomic_capable;           /* 是否支持原子模式设置 */
    
    /* 其他 */
    int num_connector;             /* 连接器数量 */
    int num_encoder;               /* 编码器数量 */
    int num_crtc;                  /* CRTC 数量 */
    int num_plane;                 /* 平面数量 */
    
    /* 异步页面翻转 */
    int async_page_flip;
};
```

```c
/* CRTC 结构体 - 表示显示控制器 */
struct drm_crtc {
    struct drm_device *dev;           /* 所属 DRM 设备 */
    struct list_head head;            /* 全局 CRTC 链表节点 */
    
    /* 对象基础结构 */
    struct drm_mode_object base;
    
    /* 当前模式 */
    struct drm_display_mode mode;     /* 当前显示模式 */
    
    /* 当前帧缓冲 */
    struct drm_framebuffer *primary_fb;
    
    /* 函数指针 */
    const struct drm_crtc_funcs *funcs;
    const struct drm_crtc_helper_funcs *helper_private;
    
    /* 当前启用的平面 */
    struct drm_plane *primary;        /* 主平面 */
    struct drm_plane *cursor;         /* 游标平面 */
    
    /* 索引 */
    unsigned int index;               /* 在 drm_mode_config 中的索引 */
    
    /* 位置 */
    int x, y;                         /* CRTC 在全局帧缓冲中的位置 */
    
    /* 是否启用 */
    bool enabled;
    
    /* VBlank 相关 */
    int vblank_enabled;
    wait_queue_head_t vblank_queue;
};
```

```c
/* 显示模式结构体 - 表示一个显示时序配置 */
struct drm_display_mode {
    /* 时钟频率 (kHz) */
    int clock;
    
    /* 水平参数 */
    int hdisplay;   /* 水平显示像素数 */
    int hsync_start;
    int hsync_end;
    int htotal;     /* 水平总像素数（含消隐） */
    int hskew;
    
    /* 垂直参数 */
    int vdisplay;   /* 垂直显示线数 */
    int vsync_start;
    int vsync_end;
    int vtotal;     /* 垂直总线数（含消隐） */
    
    /* 同步极性 */
    int flags;
    #define DRM_MODE_FLAG_PHSYNC   (1<<0) /* 水平同步极性高有效 */
    #define DRM_MODE_FLAG_NHSYNC   (1<<1) /* 水平同步极性低有效 */
    #define DRM_MODE_FLAG_PVSYNC   (1<<2) /* 垂直同步极性高有效 */
    #define DRM_MODE_FLAG_NVSYNC   (1<<3) /* 垂直同步极性低有效 */
    #define DRM_MODE_FLAG_INTERLACE (1<<4) /* 隔行扫描 */
    #define DRM_MODE_FLAG_DBLCLK   (1<<5) /* 双倍像素时钟 */
    #define DRM_MODE_FLAG_DBLSCAN  (1<<6) /* 双倍扫描 */
    
    /* 模式名称 */
    char name[DRM_DISPLAY_MODE_LEN];
    
    /* 刷新率 (mHz) - VFreq = clock * 1000 / (htotal * vtotal) */
    int vrefresh;
    
    /* 类型标志 */
    unsigned int type;
    #define DRM_MODE_TYPE_BUILTIN   (1<<0) /* 内建模式（从 EDID 读取） */
    #define DRM_MODE_TYPE_PREFERRED (1<<1) /* 首选模式（显示器推荐） */
    #define DRM_MODE_TYPE_DEFAULT   (1<<2) /* 默认模式 */
    #define DRM_MODE_TYPE_USERDEF   (1<<3) /* 用户自定义模式 */
    #define DRM_MODE_TYPE_DRIVER    (1<<4) /* 驱动定义模式 */
};
```

```c
/* DRM 连接器结构体 - 表示物理显示端口 */
struct drm_connector {
    struct drm_device *dev;
    struct device *kdev;          /* sysfs 设备节点 */
    struct drm_mode_object base;
    
    /* 连接器属性 */
    int connector_type;           /* HDMI, DP, eDP, USB-C, VGA, ... */
    int connector_type_id;        /* 同类型连接器的编号 */
    bool interlace_allowed;       /* 是否允许隔行扫描 */
    bool doublescan_allowed;      /* 是否允许双倍扫描 */
    bool stereo_allowed;          /* 是否允许立体显示 */
    
    /* 连接状态 */
    enum drm_connector_status status; /* connected, disconnected, unknown */
    
    /* 物理尺寸 (mm) */
    uint16_t mm_width, mm_height;
    
    /* 支持的显示模式列表 */
    struct list_head modes;       /* 从 EDID 解析的模式链表 */
    struct list_head probed_modes;  /* 探测到的模式 */
    struct list_head user_modes;     /* 用户自定义模式 */
    
    /* EDID 相关 */
    struct edid *edid_blob_ptr;   /* EDID 原始数据 */
    struct drm_property_blob *edid_blob; /* EDID blob 属性 */
    
    /* 热插拔检测 */
    struct drm_cmdline_mode cmdline_mode;
    enum drm_connector_force force;
    
    /* 函数指针 */
    const struct drm_connector_funcs *funcs;
    const struct drm_connector_helper_funcs *helper_private;
    
    /* 显示控制器拓扑 */
    struct drm_encoder *encoder;   /* 当前绑定的编码器 */
    struct drm_encoder *encoder_ids[DRM_CONNECTOR_MAX_ENCODER];
    
    /* 属性列表 */
    struct drm_object_properties properties;
    
    /* DP MST 相关 */
    struct drm_dp_mst_topology_mgr *mst_mgr;
    struct drm_dp_mst_port *port;
};
```

以上数据结构定义参见 Linux 内核源码 `include/drm/drm_crtc.h` 和 `include/drm/drm_modes.h`。

#### 1.7 KMS 初始化流程

KMS 子系统的初始化流程从 DRM 设备的注册开始，经历以下主要阶段：

```
KMS 初始化序列图:

┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  DRM Core   │    │  GPU Driver │    │   KMS Subsys│    │   Hardware  │
│  (drm.ko)   │    │ (amdgpu.ko) │    │ (DRM KMS)   │    │ (GPU/DC)    │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                   │                   │                   │
       │  drm_dev_alloc()  │                   │                   │
       │──────────────────>│                   │                   │
       │                   │                   │                   │
       │  drm_dev_register()│                  │                   │
       │──────────────────>│                   │                   │
       │                   │                   │                   │
       │                   │ driver->load()   │                   │
       │                   │ or ->probe()     │                   │
       │                   │──────────────────>│                   │
       │                   │                   │                   │
       │                   │                   │ drm_mode_config   │
       │                   │                   │ _init()           │
       │                   │                   │───┬──────────────>│
       │                   │                   │   │               │
       │                   │                   │   │ 初始化互斥锁  │
       │                   │                   │   │ 初始化 IDR    │
       │                   │                   │   │ 设置 min/max  │
       │                   │                   │   │ 设置 funcs    │
       │                   │                   │<──┘               │
       │                   │                   │                   │
       │                   │                   │ 驱动初始化 CRTC  │
       │                   │                   │───┬──────────────>│
       │                   │                   │   │ 读取硬件能力  │
       │                   │                   │   │ 创建 CRTC     │
       │                   │                   │   │ 注册 IRQ      │
       │                   │                   │<──┘               │
       │                   │                   │                   │
       │                   │                   │ 驱动初始化 Plane │
       │                   │                   │───┬──────────────>│
       │                   │                   │   │                │
       │                   │                   │<──┘               │
       │                   │                   │                   │
       │                   │                   │ 驱动初始化        │
       │                   │                   │ Encoder/Connector │
       │                   │                   │───┬──────────────>│
       │                   │                   │   │ 读取物理端口  │
       │                   │                   │   │ 读取 EDID     │
       │                   │                   │   │ 创建连接器    │
       │                   │                   │<──┘               │
       │                   │                   │                   │
       │                   │                   │ drm_mode_config   │
       │                   │                   │ _resigter()       │
       │                   │                   │                   │
       │<──────────────────│───────────────────│                   │
       │                   │                   │                   │
       │                   │                   │                   │
       │  /dev/dri/card0   │                   │                   │
       │  设备节点创建     │                   │                   │
```

AMDGPU 驱动中 KMS 初始化的具体调用链：

```
amdgpu_pci_probe()                                  // PCI 探测入口
  → amdgpu_device_init()                            // 设备初始化
    → amdgpu_display_modeset_probe()                // 显示模式探测
      → amdgpu_dm_init()                            // Display Manager 初始化
        → dm_create_persistent_data()               // 创建持久数据
        → dm_helpers_construct()                    // 构建辅助函数
        → dce110_register_irq_handlers()            // 注册中断处理
        → amdgpu_dm_initialize_drm_device()          // 初始化 DRM 设备
          → drm_mode_config_init()                   // 初始化 KMS 配置
          → amdgpu_dm_create_crtc()                  // 创建 CRTC
          → amdgpu_dm_create_planes()                // 创建 Plane
          → amdgpu_dm_connector_init()               // 初始化连接器
          → drm_mode_config_reset()                  // 重置配置
    → drm_dev_register()                            // 注册 DRM 设备
```

#### 1.8 DRM 属性 (Property) 系统

DRM KMS 提供了一套灵活的属性 (Property) 系统，用于控制显示配置的各个方面。每个 CRTC、Connector、Plane 都可以关联多个属性。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      DRM 属性系统架构                                         │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────┐        │
│  │  struct drm_property {                                          │        │
│  │      struct list_head head;        // 全局属性链表               │        │
│  │      struct drm_device *dev;       // 所属设备                   │        │
│  │      uint32_t flags;               // 属性类型标志               │        │
│  │      char name[DRM_PROP_NAME_LEN]; // 属性名称                   │        │
│  │      uint32_t num_values;          // 值数量                     │        │
│  │      uint64_t *values;             // 枚举值列表                 │        │
│  │      struct drm_property_enum_list *enum_list; // 枚举名列表    │        │
│  │  };                                                              │        │
│  └──────────────────────────────────────────────────────────────────┘        │
│                                                                              │
│  属性类型标志:                                                                │
│  DRM_MODE_PROP_IMMUTABLE  (1<<0)  // 不可变属性（只读）                      │
│  DRM_MODE_PROP_RANGE      (1<<1)  // 范围属性（有最小值/最大值）             │
│  DRM_MODE_PROP_ENUM       (1<<2)  // 枚举属性（选择列表）                    │
│  DRM_MODE_PROP_BLOB       (1<<3)  // 二进制大对象属性                        │
│  DRM_MODE_PROP_BITMASK    (1<<4)  // 位掩码属性                              │
│  DRM_MODE_PROP_ATOMIC     (1<<5)  // 原子属性                                │
│                                                                              │
│  Connector 常见属性:                                                          │
│  ─────────────────                                                           │
│  "EDID"          BLOB  只读  显示器 EDID 数据                                │
│  "DPMS"          ENUM  RW    电源管理状态 (On/Standby/Suspend/Off)           │
│  "link-status"   ENUM  RW    链路状态 (Good/Bad)                            │
│  "non-desktop"   RANGE RO    非桌面显示器标志                                │
│  "content type"  ENUM  RW    内容类型 (Graphics/Cinema/etc)                 │
│  "scaling mode"  ENUM  RW    缩放模式 (None/Full/Center/Full Aspect)        │
│  "max bpc"       RANGE RW    最大每分量比特数 (8/10/12/16)                  │
│  "HDR output metadata" BLOB RW  HDR 元数据                                   │
│  "Colorspace"    ENUM  RW    色彩空间 (Default/BT2020_RGB/BT709/etc)        │
│  "vrr_capable"   RANGE RO    可变刷新率能力                                  │
│  "adaptive_sync" ENUM  RW    自适应同步                                      │
│                                                                              │
│  Plane 常见属性:                                                              │
│  ────────────                                                                 │
│  "type"          ENUM  RO    平面类型 (Primary/Overlay/Cursor)               │
│  "rotation"      BITMASK RW   旋转 (rotate-0/90/180/270, reflect-x/y)       │
│  "alpha"         RANGE RW    全局透明度 (0~65535)                            │
│  "pixel blend mode" ENUM RW  像素混合模式 (None/Pre-multiplied/Coverage)  │
│  "color_encoding" ENUM RW    色彩编码 (BT.601/BT.709/BT.2020)               │
│  "color_range"   ENUM RW    色彩范围 (Limited/Full)                         │
│  "fb_id"         RANGE RW    关联的帧缓冲 ID（原子操作）                     │
│  "crtc_id"       RANGE RW    关联的 CRTC ID（原子操作）                      │
│  "src_x/y/w/h"   RANGE RW    源区域（固定点 16.16）                          │
│  "crtc_x/y/w/h"  RANGE RW    目标区域                                        │
│                                                                              │
│  CRTC 常见属性:                                                               │
│  ────────────                                                                 │
│  "mode"          BLOB  RW    当前显示模式                                    │
│  "active"        RANGE RW    CRTC 是否激活                                   │
│  "out_fence_ptr" RANGE RW    输出 fence 指针 (原子操作)                      │
│  "scaling filter" ENUM RW    缩放滤波器 (Default/Nearest Neighbor)           │
└─────────────────────────────────────────────────────────────────────────────┘
```

通过 `modetest` 查看属性：

```bash
# 查看 Plane 属性
$ modetest -M amdgpu -p --props
Planes:
19      props: 12
        1 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: Primary
        2 fb_id:
                flags: atomic range
                values: 0 4294967295
                value: 31
        3 crtc_id:
                flags: atomic range
                values: 0 4294967295
                value: 0
        4 src_x:
                flags: atomic range
                values: 0 4294967295
                value: 0
        5 src_y:
                flags: atomic range
                values: 0 4294967295
                value: 0
        6 src_w:
                flags: atomic range
                values: 0 4294967295
                value: 1920
        7 src_h:
                flags: atomic range
                values: 0 4294967295
                value: 1080
        8 crtc_x:
                flags: atomic range
                values: -4294967295 4294967295
                value: 0
        9 crtc_y:
                flags: atomic range
                values: -4294967295 4294967295
                value: 0
        10 crtc_w:
                flags: atomic range
                values: 0 4294967295
                value: 1920
        11 crtc_h:
                flags: atomic range
                values: 0 4294967295
                value: 1080
        12 rotation:
                flags: atomic bitmask
                values: 0x1=rotate-0 0x2=rotate-90 0x4=rotate-180 0x8=rotate-270 0x10=reflect-x 0x20=reflect-y
                value: rotate-0
```

#### 1.9 KMS 与 Atomic Mode Setting

KMS 最初支持的是传统（Legacy）模式设置接口，通过独立的 ioctl 调用来设置 CRTC、Plane、Encoder 等。Linux 内核 4.2 引入了 Atomic Mode Setting，允许在一个原子事务中一次性提交多个显示配置的更改。

传统 KMS 与 Atomic KMS 的对比：

| 特性 | 传统模式设置 | 原子模式设置 |
|------|------------|------------|
| 提交方式 | 多个独立 ioctl 调用 `drmModeSetCrtc`、`drmModeSetPlane` 等 | 单次 `DRM_IOCTL_MODE_ATOMIC` 提交 |
| 回滚 | 部分失败无法回滚 | 全有或全无，失败自动回滚 |
| 中间状态 | 可能看到中间帧（撕裂、闪烁） | 无中间状态，切换原子化 |
| 依赖管理 | 用户空间负责依赖顺序 | 内核自动检测依赖关系 |
| 测试模式 | 无 | 支持 `DRM_MODE_ATOMIC_TEST_ONLY` 模式 |
| 性能 | 每个操作独立提交，延迟累加 | 批量提交，整体延迟更低 |

Atomic Mode Setting 的 ioctl 定义：

```c
#define DRM_IOCTL_MODE_ATOMIC \
    DRM_IOWR(DRM_COMMAND_BASE + 0xBC, struct drm_mode_atomic)

struct drm_mode_atomic {
    __u32 flags;              /* 标志位 */
    #define DRM_MODE_ATOMIC_TEST_ONLY 0x0100  /* 仅测试不真正执行 */
    #define DRM_MODE_ATOMIC_NONBLOCK  0x0200  /* 非阻塞提交 */
    #define DRM_MODE_ATOMIC_ALLOW_MODESET 0x0400 /* 允许模式切换 */
    
    __u32 count_objs;         /* 对象数量 */
    __u64 objs_ptr;           /* 对象 ID 数组指针 */
    __u64 count_props_ptr;    /* 每个对象的属性数量数组 */
    __u64 props_ptr;          /* 属性 ID 数组指针 */
    __u64 prop_values_ptr;    /* 属性值数组指针 */
    __u64 user_data;          /* 用户数据 */
};
```

#### 1.10 VBlank 与页面翻转

VBlank（垂直消隐区间）是 CRTC 完成一帧扫描到下一帧开始之间的间隔时间。在此期间，可以安全地更新帧缓冲内容而不会产生撕裂（Tearing）。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     VBlank 与页面翻转时序图                                   │
│                                                                              │
│  ________________________________________________________________________    │
│ │                                                                          │ │
│ │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │ │
│ │   │   Frame N    │  │   Frame N+1  │  │   Frame N+2  │                  │ │
│ │   │   (显示中)   │  │   (显示中)   │  │   (显示中)   │                  │ │
│ │   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                  │ │
│ │          │                 │                 │                            │ │
│ │          ▼                 ▼                 ▼                            │ │
│ │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │ │
│ │   │  VBlank N    │  │  VBlank N+1  │  │  VBlank N+2  │                  │ │
│ │   │   (消隐期)   │  │   (消隐期)   │  │   (消隐期)   │                  │ │
│ │   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                  │ │
│ │          │                 │                 │                            │ │
│ │          │ ← page flip →  │                 │                            │ │
│ │          │  (更新 FB)     │                 │                            │ │
│ │          ▼                 ▼                 ▼                            │ │
│ │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │ │
│ │   │ VBlank IRQ   │  │ VBlank IRQ   │  │ VBlank IRQ   │                  │ │
│ │   │ (触发 fence) │  │ (触发 fence) │  │ (触发 fence) │                  │ │
│ │   └──────────────┘  └──────────────┘  └──────────────┘                  │ │
│ │__________________________________________________________________________│ │
│                                                                              │
│  ________________________________________________________________________    │
│ │                                                                          │ │
│ │  页面翻转类型:                                                            │ │
│ │  ────────────                                                             │ │
│ │                                                                          │ │
│ │  1. 同步页面翻转 (Synchronous Page Flip)                                  │ │
│ │     - 在 VBlank 中断中执行翻转                                            │ │
│ │     - 保证无撕裂                                                          │ │
│ │     - 阻塞调用直到下一个 VBlank                                           │ │
│ │                                                                          │ │
│ │  2. 异步页面翻转 (Async Page Flip)                                        │ │
│ │     - 立即执行翻转，不等待 VBlank                                         │ │
│ │     - 可能产生撕裂                                                        │ │
│ │     - 低延迟                                                              │ │
│ │                                                                          │ │
│ │  3. 原子页面翻转 (Atomic Page Flip)                                       │ │
│ │     - 在原子事务中提交                                                    │ │
│ │     - 支持多个平面同时翻转                                                 │ │
│ │     - 支持 fence 同步                                                     │ │
│ │__________________________________________________________________________│ │
│                                                                              │
│  VBlank 中断处理流程:                                                        │
│                                                                              │
│  ┌──────────────┐                                                           │
│  │  硬件 VBlank │                                                           │
│  │  中断触发    │                                                           │
│  └──────┬───────┘                                                           │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────┐                                                       │
│  │ drm_handle_vblank│  → 更新 vblank 计数                                   │
│  └──────┬───────────┘                                                       │
│         │                                                                    │
│         ├───────────────────────────────┐                                   │
│         ▼                               ▼                                   │
│  ┌──────────────────┐         ┌──────────────────┐                          │
│  │  唤醒等待队列    │         │  处理页面翻转     │                          │
│  │  vblank_queue    │         │  page flip       │                          │
│  └──────────────────┘         └──────────────────┘                          │
│         │                               │                                   │
│         ▼                               ▼                                   │
│  ┌──────────────────┐         ┌──────────────────┐                          │
│  │  发送 VBlank     │         │  触发完成回调    │                          │
│  │  事件到用户空间  │         │  page_flip_cb    │                          │
│  └──────────────────┘         └──────────────────┘                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

VBlank 相关的内核 API：

```c
#include <drm/drm_vblank.h>

/* 获取 VBlank 计数器 */
u64 drm_crtc_vblank_count(struct drm_crtc *crtc);

/* 等待 VBlank */
int drm_crtc_wait_one_vblank(struct drm_crtc *crtc);

/* 启用 VBlank 中断 */
int drm_crtc_vblank_get(struct drm_crtc *crtc);
void drm_crtc_vblank_put(struct drm_crtc *crtc);

/* 打开/关闭 VBlank */
int drm_vblank_init(struct drm_device *dev, unsigned int num_crtcs);

/* VBlank 中断处理函数 (驱动调用) */
bool drm_handle_vblank(struct drm_device *dev, unsigned int pipe);

/* 注册 VBlank 回调 (用于页面翻转完成通知) */
void drm_crtc_add_vblank_event(struct drm_crtc *crtc,
                                struct drm_pending_event *e);
```

AMDGPU 驱动中 VBlank IRQ 的注册路径：

```
amdgpu_dm_init()
  → dm_create_persistent_data()
    → register_hotplug_hpd_irq_handler()
    → register_hpd_handlers()
      → amdgpu_dm_irq_register_interrupt()
        → dc_interrupt_set()         // 设置 DC 中断
        → amdgpu_irq_add_id()        // 注册 IRQ 号
          → amdgpu_dm_irq_handler()  // AMDGPU KMS IRQ 处理入口
            → dm_vblank_handler()    // VBlank 中断处理
              → drm_crtc_handle_vblank()
```

#### 1.11 KMS fbdev (帧缓冲设备) 回退

在 KMS 架构中，还提供了一个兼容层 fbdev，用于在系统启动初期提供帧缓冲控制台支持：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    KMS fbdev 兼容层                                          │
│                                                                              │
│  当内核配置了 CONFIG_DRM_FBDEV_EMULATION 时，DRM KMS 会创建一个               │
│  /dev/fb0 设备，使得传统的 fbdev 程序也能在内核模式设置下工作。                │
│                                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                   │
│  │  Xorg/Wayland│    │  fbdev apps  │    │  Kernel      │                   │
│  │  (DRM 原生)  │    │  (/dev/fb0)  │    │  Console     │                   │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘                   │
│         │                   │                   │                            │
│         ▼                   ▼                   ▼                            │
│  ┌─────────────────────────────────────────────────────┐                    │
│  │               DRM Core (drm.ko)                      │                    │
│  │  ┌─────────────────┐  ┌─────────────────────────┐   │                    │
│  │  │   KMS API       │  │   DRM FBDEV Helper      │   │                    │
│  │  │   (ioctl)       │  │   (drm_fb_helper.c)     │   │                    │
│  │  └─────────────────┘  └─────────────────────────┘   │                    │
│  └──────────────────────────────┬──────────────────────┘                    │
│                                 │                                           │
│                                 ▼                                           │
│  ┌─────────────────────────────────────────────────────┐                    │
│  │            AMDGPU Driver (amdgpu.ko)                  │                    │
│  │  ┌──────────────────┐  ┌─────────────────────────┐   │                    │
│  │  │   amdgpu_dm.c    │  │   amdgpu_fb.c           │   │                    │
│  │  │   (KMS 主入口)   │  │   (fbdev 控制台)        │   │                    │
│  │  └──────────────────┘  └─────────────────────────┘   │                    │
│  └──────────────────────────────────────────────────────┘                    │
│                                                                              │
│  fbdev 兼容层初始化:                                                         │
│  1. amdgpu_fbdev_init() 创建 fbdev 辅助器                                    │
│  2. drm_fb_helper_init() 初始化辅助器                                        │
│  3. drm_fb_helper_single_add_all_connectors() 添加所有连接器                 │
│  4. drm_fb_helper_initial_config() 初始配置 (分配 fb, 设置模式)             │
│  5. drm_fb_helper_fbdev_setup() 设置 fbdev 设备                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 1.12 KMS 事件模型

KMS 使用事件 (Event) 机制来通知用户空间异步操作完成：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     KMS 事件模型                                             │
│                                                                              │
│  用户空间通过 DRM 文件描述符等待事件：                                       │
│  poll() / select() / epoll() 监听 POLLIN                                    │
│                                                                              │
│  KMS 事件类型:                                                               │
│  ───────────                                                                 │
│                                                                              │
│  DRM_EVENT_FLIP_COMPLETE:  页面翻转完成                                     │
│  ─────────────────────────                                                   │
│  用户空间调用 drmModePageFlip() 或原子提交后，                               │
│  内核在 VBlank 中断中完成翻转，向用户空间发送事件。                          │
│                                                                              │
│  struct drm_event_vblank {                                                   │
│      struct drm_event base;                                                 │
│      __u32 user_data;       // 用户自定义数据                                │
│      __u32 tv_sec;          // 时间戳 (秒)                                  │
│      __u32 tv_usec;         // 时间戳 (微秒)                                 │
│      __u32 sequence;        // VBlank 序列号                                 │
│      __u32 crtc_id;         // CRTC ID                                      │
│  };                                                                          │
│                                                                              │
│  DRM_EVENT_VBLANK:         VBlank 事件                                      │
│  ───────────────────                                                         │
│  用户空间通过 DRM_IOCTL_WAIT_VBLANK ioctl 等待 VBlank。                     │
│  或通过 drmWaitVBlank() libdrm 函数调用。                                   │
│                                                                              │
│  使用示例:                                                                   │
│  int fd = open("/dev/dri/card0", O_RDWR);                                    │
│  drmVBlank vbl;                                                              │
│  vbl.request.type = DRM_VBLANK_RELATIVE;                                     │
│  vbl.request.sequence = 1;  // 等待下一个 VBlank                             │
│  drmWaitVBlank(fd, &vbl);                                                    │
│  printf("VBlank seq: %lu\n", vbl.reply.sequence);                           │
│                                                                              │
│  DRM_EVENT_CRTC_SEQUENCE:  CRTC 序列事件 (内核 5.x+)                        │
│  ───────────────────────────                                                  │
│  更精确的 CRTC 时序事件，允许指定目标序列号触发。                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2. 实践操作

#### 2.1 查看 KMS 设备信息

使用 modetest 工具（来自 libdrm 测试集）查看 KMS 设备：

```bash
# 查看 DRM 设备列表
$ ls -la /dev/dri/
crw-rw---- 1 root video 226,   0 May  9 10:00 card0
crw-rw---- 1 root video 226,  64 May  9 10:00 controlD64
crw-rw---- 1 root video 226, 128 May  9 10:00 renderD128

# 使用 modetest 查看 card0 的 KMS 信息
$ modetest -M amdgpu -c

# 输出示例（RDNA 3 GPU）:
Connectors:
id      encoder status          type    size (mm)       modes   enc name
5       4       connected        DP-1    600x340          22      4
  modes:
        index   name refresh (Hz) hdisp hss hse htot vdisp vss vse vtot)
  #0 1920x1080 60.00 1920 2008 2052 2200 1080 1084 1089 1125 148500 flags: phsync, pvsync; type: preferred, driver
  #1 1920x1080 59.94 1920 2008 2052 2200 1080 1084 1089 1125 148352 flags: phsync, pvsync; type: driver
  #2 1920x1080 50.00 1920 2008 2052 2200 1080 1084 1089 1125 148500 flags: phsync, pvsync; type: driver
  #3 1680x1050 59.88 1680 1728 1760 1840 1050 1053 1059 1080 119000 flags: nhsync, nvsync; type: driver
  ...

# 查看 CRTC 信息
$ modetest -M amdgpu -e
Encoders:
id      crtc    type    possible crtcs  possible clones
4       0       TMDS   0x00000001      0x00000000

# 查看 Plane 信息
$ modetest -M amdgpu -p
Planes:
id      crtc    fb      CRTC x,y        x,y     gamma size       possible crtcs
19      0       31      0,0             0,0     0               0x00000001
  formats: XR24 AR24 XR12 AR12 XR12 AR12 RG16 BG16 XR24 AR24 XR24 AR24 RG16 BG16
20      0       0       0,0             0,0     0               0x00000001
  formats: AR24 AR24 AR24 AR24 AR24 AR24 AR24 AR24 AR24 AR24 XR24 XR24
21      0       0       0,0             0,0     0               0x00000001
  formats: AR24 AR24 AR24 AR24 AR24 AR24 AR24 AR24 AR24 AR24 XR24 XR24
```

modetest 的完整用法：

```bash
# modetest 参数说明
modetest [options]

# 常用选项:
  -M <module>     指定 DRM 模块 (如 amdgpu, radeon, i915)
  -c              显示 Connector 信息
  -e              显示 Encoder 信息
  -p              显示 Plane 信息
  -f              显示 Framebuffer 信息
  -s <connector>:<mode>  设置显示模式
  -a              显示所有信息
  -v              显示版本信息
  -w <id>:<prop>:<value>  设置属性
  -C              测试 CRC 校验
  --props         显示属性信息

# 显示所有 KMS 信息
$ modetest -M amdgpu -a

# 设置显示模式
$ modetest -M amdgpu -s 5@4:1920x1080

# 设置显示模式并指定刷新率
$ modetest -M amdgpu -s 5@4:1920x1080@60

# 将单个平面设置为某个帧缓冲
$ modetest -M amdgpu -P 19@4:1920x1080@XR24

# 测试原子模式设置 (不真正提交)
$ modetest -M amdgpu -a --atomic
```

#### 2.2 使用 libdrm API 进行模式设置

以下示例代码展示如何使用 libdrm 的 KMS API 设置显示模式：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

int main(int argc, char **argv) {
    int fd;
    drmModeRes *resources;
    drmModeConnector *connector = NULL;
    drmModeEncoder *encoder = NULL;
    drmModeCrtc *crtc = NULL;
    drmModeModeInfo *mode = NULL;
    int i, j;

    /* 打开 DRM 设备 */
    fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);
    if (fd < 0) {
        perror("Failed to open /dev/dri/card0");
        return 1;
    }

    /* 获取 DRM 资源 */
    resources = drmModeGetResources(fd);
    if (!resources) {
        fprintf(stderr, "Failed to get DRM resources\n");
        close(fd);
        return 1;
    }

    printf("DRM Resources:\n");
    printf("  CRTC count: %d\n", resources->count_crtcs);
    printf("  Encoder count: %d\n", resources->count_encoders);
    printf("  Connector count: %d\n", resources->count_connectors);
    printf("  FB count: %d\n", resources->count_fbs);

    /* 查找第一个已连接的 Connector */
    for (i = 0; i < resources->count_connectors && !connector; i++) {
        connector = drmModeGetConnector(fd, resources->connectors[i]);
        if (connector && connector->connection == DRM_MODE_CONNECTED) {
            printf("\nFound connected connector: %d\n", connector->connector_id);
            printf("  Connector type: ");
            switch (connector->connector_type) {
                case DRM_MODE_CONNECTOR_HDMIA: printf("HDMI-A\n"); break;
                case DRM_MODE_CONNECTOR_DisplayPort: printf("DP\n"); break;
                case DRM_MODE_CONNECTOR_eDP: printf("eDP\n"); break;
                case DRM_MODE_CONNECTOR_USB: printf("USB-C\n"); break;
                default: printf("Other (%d)\n", connector->connector_type);
            }
            printf("  Width: %d mm, Height: %d mm\n",
                   connector->mmWidth, connector->mmHeight);
            printf("  Mode count: %d\n", connector->count_modes);

            /* 选择首选模式 */
            for (j = 0; j < connector->count_modes; j++) {
                printf("    Mode %d: %s @ %d Hz\n",
                       j,
                       connector->modes[j].name,
                       connector->modes[j].vrefresh);
                if (connector->modes[j].type & DRM_MODE_TYPE_PREFERRED) {
                    mode = &connector->modes[j];
                }
            }

            if (!mode && connector->count_modes > 0) {
                mode = &connector->modes[0];
            }
        } else {
            if (connector) {
                drmModeFreeConnector(connector);
                connector = NULL;
            }
        }
    }

    if (!connector || !mode) {
        fprintf(stderr, "No connected connector or no valid mode found\n");
        drmModeFreeResources(resources);
        close(fd);
        return 1;
    }

    /* 查找匹配的 Encoder */
    for (i = 0; i < resources->count_encoders; i++) {
        encoder = drmModeGetEncoder(fd, resources->encoders[i]);
        if (!encoder) continue;
        if (encoder->encoder_id == connector->encoder_id) {
            break;
        }
        drmModeFreeEncoder(encoder);
        encoder = NULL;
    }

    if (!encoder) {
        /* 尝试通过可能的 CRTC 掩码查找 */
        for (i = 0; i < resources->count_encoders; i++) {
            encoder = drmModeGetEncoder(fd, resources->encoders[i]);
            if (!encoder) continue;
            if (encoder->possible_crtcs & (1 << 0)) {
                break;
            }
            drmModeFreeEncoder(encoder);
            encoder = NULL;
        }
    }

    if (!encoder) {
        fprintf(stderr, "No suitable encoder found\n");
        drmModeFreeConnector(connector);
        drmModeFreeResources(resources);
        close(fd);
        return 1;
    }

    /* 获取当前 CRTC 状态 */
    crtc = drmModeGetCrtc(fd, resources->crtcs[0]);
    if (!crtc) {
        fprintf(stderr, "Failed to get CRTC\n");
        drmModeFreeEncoder(encoder);
        drmModeFreeConnector(connector);
        drmModeFreeResources(resources);
        close(fd);
        return 1;
    }

    printf("\nCurrent CRTC state:\n");
    printf("  CRTC ID: %d\n", crtc->crtc_id);
    printf("  (x, y): (%d, %d)\n", crtc->x, crtc->y);
    printf("  FB ID: %d\n", crtc->buffer_id);

    /* 分配帧缓冲并设置模式 */
    printf("\nSetting mode: %s @ %d Hz\n", mode->name, mode->vrefresh);

    uint32_t handles[4] = {0}, pitches[4] = {0}, offsets[4] = {0};
    uint32_t fb_id;
    unsigned char *framebuf;
    struct drm_mode_create_dumb dumb = {0};
    struct drm_mode_destroy_dumb dumb_destroy = {0};
    struct drm_mode_map_dumb dumb_map = {0};
    unsigned int fb_width = mode->hdisplay;
    unsigned int fb_height = mode->vdisplay;
    unsigned int fb_bpp = 32;
    unsigned int fb_pitch;

    /* 创建 dumb buffer */
    dumb.width = fb_width;
    dumb.height = fb_height;
    dumb.bpp = fb_bpp;
    if (drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &dumb) < 0) {
        fprintf(stderr, "Failed to create dumb buffer: %s\n", strerror(errno));
        drmModeFreeCrtc(crtc);
        drmModeFreeEncoder(encoder);
        drmModeFreeConnector(connector);
        drmModeFreeResources(resources);
        close(fd);
        return 1;
    }
    fb_pitch = dumb.pitch;

    /* 添加帧缓冲 */
    if (drmModeAddFB(fd, fb_width, fb_height, 24, 32,
                     fb_pitch, dumb.handle, &fb_id) < 0) {
        /* 如果 24/32 失败，尝试 DRM_FORMAT_XRGB8888 */
        uint32_t format = DRM_FORMAT_XRGB8888;
        if (drmModeAddFB2(fd, fb_width, fb_height, format,
                          handles, pitches, offsets, &fb_id, 0) < 0) {
            fprintf(stderr, "Failed to add FB: %s\n", strerror(errno));
            dumb_destroy.handle = dumb.handle;
            drmIoctl(fd, DRM_IOCTL_MODE_DESTROY_DUMB, &dumb_destroy);
            drmModeFreeCrtc(crtc);
            drmModeFreeEncoder(encoder);
            drmModeFreeConnector(connector);
            drmModeFreeResources(resources);
            close(fd);
            return 1;
        }
    }

    /* 映射帧缓冲到用户空间 */
    dumb_map.handle = dumb.handle;
    if (drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &dumb_map) < 0) {
        fprintf(stderr, "Failed to map dumb buffer: %s\n", strerror(errno));
        goto cleanup;
    }

    framebuf = mmap(0, dumb.size, PROT_READ | PROT_WRITE,
                    MAP_SHARED, fd, dumb_map.offset);
    if (framebuf == MAP_FAILED) {
        fprintf(stderr, "Failed to mmap framebuffer: %s\n", strerror(errno));
        goto cleanup;
    }

    /* 填充帧缓冲 - 绘制渐变条纹 */
    for (unsigned int y = 0; y < fb_height; y++) {
        for (unsigned int x = 0; x < fb_width; x++) {
            unsigned int offset = y * fb_pitch + x * 4;
            framebuf[offset + 0] = (x * 255) / fb_width;     /* B */
            framebuf[offset + 1] = (y * 255) / fb_height;    /* G */
            framebuf[offset + 2] = 128;                        /* R */
            framebuf[offset + 3] = 0xff;                        /* A */
        }
    }

    /* 设置 CRTC 模式 */
    if (drmModeSetCrtc(fd, crtc->crtc_id, fb_id, 0, 0,
                       &connector->connector_id, 1, mode) < 0) {
        fprintf(stderr, "Failed to set CRTC mode: %s\n", strerror(errno));
        munmap(framebuf, dumb.size);
        goto cleanup;
    }

    printf("Mode set successfully! Press Enter to exit...\n");
    getchar();

    /* 恢复原始 CRTC 状态 */
    drmModeSetCrtc(fd, crtc->crtc_id, crtc->buffer_id,
                   crtc->x, crtc->y,
                   &connector->connector_id, 1, &crtc->mode);

    munmap(framebuf, dumb.size);

cleanup:
    dumb_destroy.handle = dumb.handle;
    drmIoctl(fd, DRM_IOCTL_MODE_DESTROY_DUMB, &dumb_destroy);
    if (fb_id) drmModeRmFB(fd, fb_id);
    if (crtc) drmModeFreeCrtc(crtc);
    if (encoder) drmModeFreeEncoder(encoder);
    if (connector) drmModeFreeConnector(connector);
    drmModeFreeResources(resources);
    close(fd);
    return 0;
}
```

编译命令：
```bash
gcc -o kms_demo kms_demo.c -ldrm -lm
sudo ./kms_demo
```

#### 2.3 通过 sysfs 查看 KMS 状态

Linux 内核通过 sysfs 暴露 KMS 设备的状态信息，位于 `/sys/class/drm/` 目录下：

```bash
# 列出所有 DRM 设备及其连接状态
$ ls /sys/class/drm/
card0  card0-HDMI-A-1  card0-DP-1  card0-eDP-1  renderD128

# 查看连接器状态
$ cat /sys/class/drm/card0-HDMI-A-1/status
connected

# 查看当前使能状态
$ cat /sys/class/drm/card0-HDMI-A-1/enabled
enabled

# 查看 EDID（原始二进制数据）
$ cat /sys/class/drm/card0-HDMI-A-1/edid | hexdump -C | head -20

# 查看支持的显示模式
$ cat /sys/class/drm/card0-HDMI-A-1/modes
1920x1080
1600x900
1280x1024
1280x720
1024x768
800x600
640x480

# 查看 DPMS 状态
$ cat /sys/class/drm/card0-HDMI-A-1/dpms
On

# 触发连接器重新探测
$ echo detect > /sys/class/drm/card0-HDMI-A-1/trigger

# 查看 GPU 所属的 DRM 设备
$ readlink /sys/class/drm/card0/device
../../../0000:03:00.0

# 查看 PCI 信息
$ cat /sys/class/drm/card0/device/vendor
0x1002
$ cat /sys/class/drm/card0/device/device
0x7310
```

sysfs 接口提供了一种不需要任何工具即可查看 KMS 状态的方法，非常适合脚本自动化检测显示器状态。

#### 2.4 使用 xrandr 管理显示配置

xrandr（X Resize and Rotate）是 X11 环境下的标准显示配置工具，它通过 libdrm 与内核 KMS 交互：

```bash
# 查看当前显示配置
$ xrandr
Screen 0: minimum 320 x 200, current 1920 x 1080, maximum 16384 x 16384
eDP connected primary 1920x1080+0+0 (normal left inverted right) 309mm x 174mm
   1920x1080     60.00*+  60.00    50.00
   1680x1050     60.00
   1280x1024     60.00
   1440x900      59.90
   1280x720      60.00
DP-1 disconnected (normal left inverted right)
HDMI-A-1 connected 1920x1080+0+0 (normal left inverted right) 531mm x 298mm
   1920x1080     60.00*+  50.00    59.94
   1280x1024     60.00
   1280x720     60.00

# 设置单显示器分辨率
$ xrandr --output eDP --mode 1920x1080 --rate 60

# 设置双显示器扩展模式
$ xrandr --output eDP --auto --output HDMI-A-1 --auto --right-of eDP

# 设置双显示器镜像模式
$ xrandr --output eDP --auto --output HDMI-A-1 --auto --same-as eDP

# 关闭某个显示器
$ xrandr --output HDMI-A-1 --off

# 设置缩放
$ xrandr --output eDP --scale 1.5x1.5

# 查看当前刷新率
$ xrandr --verbose | grep -i rate

# 使用 --prop 查看连接器属性
$ xrandr --prop
HDMI-A-1 connected 1920x1080+1920+0 (normal left inverted right) 531mm x 298mm
   1920x1080     60.00*+  50.00    59.94
   ...
   EDID:
       00ffffffffffff004ca34e324c554132
       001c0103803c2278ea9de5ae4d47a924
   ...
   non-desktop: 0
        supported: 0, 1
   Content Protection: Undesired
        supported: Undesired, Desired, Enabled
```

在 Wayland 环境下，使用 `wlr-randr`（基于 wlroots 的合成器）或 `kanshci`（KDE KWin）：

```bash
# wlr-randr 用法
$ wlr-randr
eDP-1: 1920x1080 @ 60.000000 Hz (DP-1 connected)
  Preferred mode: 1920x1080 @ 60.000000 Hz

# 设置分辨率
$ wlr-randr --output eDP-1 --mode 1920x1080
```

### 3. 案例分析

#### 3.1 KMS 初始化阶段的显示闪烁问题

**Bug 背景**：在 Linux 内核 5.4 到 5.10 版本间，多个 AMDGPU 用户报告系统启动时外部显示器出现短暂闪烁或黑屏，特别是在使用 DisplayPort 连接时。

**根因分析**：该问题源于 KMS 初始化过程中 CRTC 与 Connector 的绑定时序。在 `amdgpu_dm.c` 的 `amdgpu_dm_init()` 函数中，DC（Display Core）的初始化完成后，DRM 核心层开始枚举连接器并创建 CRTC。然而，如果 DC 硬件检测尚未完成（如 DP 接收器的 AUX 通道通信未就绪），EDID 读取会失败，导致连接器被标记为 `disconnected`。随后当热插拔事件触发重新检测时，KMS 需要重新分配 CRTC，造成中间状态的闪烁。

**参考链接**：https://gitlab.freedesktop.org/drm/amd/-/issues/1123

**修复方法**：AMD 开发者在 `amdgpu_dm_connector_funcs.detect` 中添加了延迟检测机制，确保在初始化阶段等待 AUX 通道就绪后再读取 EDID。同时，DC 的 `s3_handle_dmub_irq()` 中增加了对热插拔中断的早期处理。

```c
/* drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c */
static int amdgpu_dm_init(struct amdgpu_device *adev) {
    /* ... 初始化 DC ... */

    /* 增加初始化延迟检测等待 */
    if (adev->dm.dc->caps.ips_support) {
        /* 等待 DMUB 固件完成硬件检测 */
        dmub_srv_wait_for_idle(adev->dm.dmub_srv, 100000);
    }

    /* ... 创建 CRTC 和连接器 ... */
}
```

**测试方法**：
1. 反复重启系统并观察外部显示器的输出状态
2. 使用 `dmesg | grep -i amdgpu` 检查是否有 `EDID` 或 `Aux` 相关错误
3. 编写脚本循环执行 `modetest -M amdgpu` 并在热插拔后验证连接器状态

```bash
#!/bin/bash
# display_flicker_test.sh - KMS 闪烁测试脚本
for i in {1..50}; do
    echo "=== Boot cycle $i ===" >> test_results.log
    sudo reboot
    sleep 30
    ssh user@host "dmesg | grep -E 'amdgpu.*EDID|amdgpu.*Aux|dm.*connect'" >> test_results.log
    ssh user@host "modetest -M amdgpu -c" >> test_results.log
done
```

#### 3.2 热插拔（Hotplug）状态同步问题

**Bug 背景**：freedesktop.org bug #110632 报告了当通过 DP MST（多流传输）集线器连接多个显示器时，热插拔后某些显示器无法正确启用，表现为 `xrandr` 显示 `disconnected` 但物理显示器实际已连接。

**根因分析**：问题出在 DRM 核心的连接器状态缓存机制。当 MST 拓扑变化时，`drm_dp_mst_topology_mgr` 中的 `mst_state` 更新与 `drm_connector` 的状态同步存在窗口期。具体来说，`drm_dp_mst_connector_destroy()` 在拓扑变化时销毁旧的连接器，而 `drm_dp_mst_connector_init()` 创建的新连接器尚未完成 EDID 读取，此时 `drm_connector->status` 被错误地设为 `connector_status_disconnected`。

**参考链接**：https://bugs.freedesktop.org/show_bug.cgi?id=110632

**修复方法**：在 `drivers/gpu/drm/drm_dp_mst_topology.c` 中增加了 `drm_dp_mst_connector_late_register()` 回调，在注册完成后重新读取连接器状态：

```c
/* drivers/gpu/drm/drm_dp_mst_topology.c */
int drm_dp_mst_connector_late_register(struct drm_connector *connector,
                                       struct drm_dp_mst_port *port)
{
    struct drm_dp_mst_topology_mgr *mgr = port->mgr;
    int ret;

    ret = drm_dp_mst_register_i2c_bus(&port->aux);
    if (ret < 0)
        return ret;

    /* 重新探测连接器状态 */
    connector->status = drm_helper_probe_detect(connector, NULL, false);
    drm_kms_helper_hotplug_event(connector->dev);

    return 0;
}
```

**测试方法**：
1. 连接 DP MST 集线器和多个显示器
2. 反复热插拔单个显示器并检查连接器状态
3. 使用 IGT 的 `kms_hotplug` 测试用例进行验证

```bash
# 安装 IGT 并运行热插拔测试
$ sudo igt-gpu-tools-build/tests/kms_hotplug --run-subtest hotplug-dp

# 手动触发热插拔测试
$ while true; do
    echo 1 > /sys/class/drm/card0-HDMI-A-1/trigger
    sleep 1
    cat /sys/class/drm/card0-HDMI-A-1/status
    sleep 2
done
```

### 4. 实践操作：KMS 压力测试脚本

以下是一个完整的 KMS 压力测试脚本，用于验证模式切换的稳定性：

```bash
#!/bin/bash
# kms_stress_test.sh - KMS 模式切换压力测试
# 测试内容：循环切换分辨率、刷新率、DPMS 状态
# 参考：https://drm.pages.freedesktop.org/igt-gpu-tools/

DRM_CARD=${1:-"/dev/dri/card0"}
CONNECTOR=${2:-"HDMI-A-1"}
MODES=("1920x1080" "1280x720" "1024x768" "800x600" "640x480")
REFRESH_RATES=("60" "50" "30")
ITERATIONS=${3:-100}

echo "KMS Stress Test Starting"
echo "DRM Device: $DRM_CARD"
echo "Connector: $CONNECTOR"
echo "Iterations: $ITERATIONS"
echo "========================================"

# 检查依赖
for cmd in xrandr modetest cat; do
    if ! command -v $cmd &> /dev/null; then
        echo "ERROR: $cmd not found"
        exit 1
    fi
done

# 保存初始状态
INITIAL_MODE=$(xrandr --query | grep "$CONNECTOR connected" | \
               grep -oP '\d+x\d+' | head -1)
echo "Initial mode: $INITIAL_MODE"

test_count=0
fail_count=0

for ((i=1; i<=ITERATIONS; i++)); do
    echo "--- Iteration $i/$ITERATIONS ---"

    # 随机选择测试操作
    CASE=$((RANDOM % 5))

    case $CASE in
        0)
            # 测试 1：切换分辨率
            MODE=${MODES[$((RANDOM % ${#MODES[@]}))]}
            echo "Test: Set resolution $MODE"
            xrandr --output $CONNECTOR --mode $MODE
            RESULT=$?
            ;;
        1)
            # 测试 2：切换刷新率
            RATE=${REFRESH_RATES[$((RANDOM % ${#REFRESH_RATES[@]}))]}
            echo "Test: Set refresh rate ${RATE}Hz"
            xrandr --output $CONNECTOR --rate $RATE
            RESULT=$?
            ;;
        2)
            # 测试 3：DPMS 开关
            echo "Test: DPMS off/on"
            xrandr --output $CONNECTOR --dpms off
            sleep 0.5
            xrandr --output $CONNECTOR --dpms on
            RESULT=$?
            ;;
        3)
            # 测试 4：使用 modetest 设置模式
            echo "Test: modetest atomic test"
            modetest -M amdgpu -a --atomic 2>/dev/null
            RESULT=$?
            ;;
        4)
            # 测试 5：连接器状态读取
            echo "Test: sysfs status read"
            STATUS=$(cat /sys/class/drm/card0-$CONNECTOR/status 2>/dev/null)
            echo "  Connector status: $STATUS"
            RESULT=0
            ;;
    esac

    if [ $RESULT -ne 0 ]; then
        echo "  FAIL: Exit code $RESULT"
        ((fail_count++))
    else
        echo "  PASS"
    fi

    ((test_count++))
    sleep 0.2
done

# 恢复初始模式
echo "========================================"
echo "Restoring initial mode: $INITIAL_MODE"
xrandr --output $CONNECTOR --mode $INITIAL_MODE

echo "========================================"
echo "KMS Stress Test Complete"
echo "Total tests: $test_count"
echo "Passed: $((test_count - fail_count))"
echo "Failed: $fail_count"
echo "Success rate: $(( (test_count - fail_count) * 100 / test_count ))%"

# 如果有失败，输出 dmesg 信息
if [ $fail_count -gt 0 ]; then
    echo "Check dmesg for errors:"
    dmesg | grep -E "drm|amdgpu" | tail -20
fi
```

运行压力测试：
```bash
chmod +x kms_stress_test.sh
# 运行 50 次迭代测试
./kms_stress_test.sh /dev/dri/card0 HDMI-A-1 50

# 长时间稳定性测试（1000 次迭代）
./kms_stress_test.sh /dev/dri/card0 eDP-1 1000
```

### 5. 实践操作：IGT KMS 测试用例

IGT GPU Tools（igt-gpu-tools）是 KMS 测试的官方工具集。以下是与 KMS 基础功能相关的常用测试用例：

```bash
# 确认 IGT 已安装
$ igt-gpu-tools-build/tests/kms_flip --list-subtests
basic-flip
basic-flip-vs-wf
basic-flip-vs-dpms
basic-flip-vs-modeset
plain-flip
plain-flip-ts-check
busy-flip
...

# 运行基本 flip 测试
$ sudo igt-gpu-tools-build/tests/kms_flip --run-subtest basic-flip

# 测试模式设置基本功能
$ sudo igt-gpu-tools-build/tests/kms_mode --run-subtest basic

# 测试连接器状态探测
$ sudo igt-gpu-tools-build/tests/kms_probe

# 测试原子模式设置
$ sudo igt-gpu-tools-build/tests/kms_atomic --run-subtest basic

# 测试 VBlank 事件
$ sudo igt-gpu-tools-build/tests/kms_vblank --run-subtest basic

# 测试帧缓冲创建与销毁
$ sudo igt-gpu-tools-build/tests/kms_addfb_basic

# 测试 fbdev 回退路径
$ sudo igt-gpu-tools-build/tests/kms_fbcon

# 运行完整的 KMS 测试集
$ sudo igt-gpu-tools-build/tests/kms_all

# 查看单个测试的代码
$ grep -r "igt_main" igt-gpu-tools/tests/kms_flip.c | head -5
```

IGT 测试是验证 KMS 功能正确性的重要手段。每个测试用例都包含详细的参数化和非参数化子测试，测试覆盖了正常路径和错误路径。

以下是一个 IGT KMS 测试的基本结构示例（简化版）：

```c
/* igt-gpu-tools/tests/kms_example.c - 简化的 IGT KMS 测试结构 */
#include "igt.h"

IGT_TEST_DESCRIPTION("Example KMS test demonstrating IGT structure");

static void test_connector_state(int fd)
{
    drmModeRes *res;
    drmModeConnector *conn;
    int i;

    res = drmModeGetResources(fd);
    igt_assert(res != NULL);

    for (i = 0; i < res->count_connectors; i++) {
        conn = drmModeGetConnector(fd, res->connectors[i]);
        igt_assert(conn != NULL);
        igt_info("Connector %d: type %d, status %d, modes %d\n",
                 conn->connector_id, conn->connector_type,
                 conn->connection, conn->count_modes);
        igt_assert(conn->count_modes > 0);
        drmModeFreeConnector(conn);
    }

    drmModeFreeResources(res);
}

igt_main
{
    int fd;

    igt_fixture {
        fd = drm_open_driver_master(DRIVER_ANY);
        igt_require(fd >= 0);
    }

    igt_subtest("connector-state")
        test_connector_state(fd);

    igt_fixture {
        close(fd);
    }
}
```

## 相关链接

- Linux Kernel DRM KMS 文档：https://www.kernel.org/doc/html/latest/gpu/drm-kms.html
- AMDGPU 驱动文档 (KMS 部分)：https://dri.freedesktop.org/docs/drm/gpu/amdgpu.html
- libdrm API 文档 (xf86drmMode.h)：https://dri.freedesktop.org/docs/drm/libdrm/
- IGT GPU Tools 源码与文档：https://gitlab.freedesktop.org/drm/igt-gpu-tools
- modetest 源码：https://gitlab.freedesktop.org/mesa/drm/-/blob/main/tests/modetest/modetest.c
- DRM KMS 内核源码：drivers/gpu/drm/drm_crtc.c
- AMDGPU Display Core 源码：drivers/gpu/drm/amd/display/
- KMS 事件模型文档：https://www.kernel.org/doc/html/latest/gpu/drm-uapi.html
- VESA CVT 标准：https://vesa.org/vesa-cvt/
- FreeDesktop AMDGPU Bug 跟踪：https://gitlab.freedesktop.org/drm/amd/-/issues
- Wayland 与 DRM 交互：https://wayland.freedesktop.org/docs/html/
- Linux 图形栈简介：https://01.org/linuxgraphics/documentation
- drm_info 工具：https://gitlab.freedesktop.org/emersion/drm_info
- Linux kernel drm 提交日志：https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/drivers/gpu/drm
- AMDGPU 显示问题跟踪：https://gitlab.freedesktop.org/drm/amd/-/issues/?label_name%5B%5D=display
- IGT GPU Tools kms_flip 测试源码：https://gitlab.freedesktop.org/drm/igt-gpu-tools/-/blob/master/tests/kms_flip.c
- drm-misc 维护者分支：https://cgit.freedesktop.org/drm/drm-misc/
- Weston 合成器参考实现：https://gitlab.freedesktop.org/wayland/weston
- DRM 格式修饰符（DRM Format Modifiers）文档：https://www.kernel.org/doc/html/latest/gpu/drm-kms.html#drm-format-modifiers
- xrandr 源码：https://gitlab.freedesktop.org/xorg/app/xrandr
- wlr-randr 源码（wlroots）：https://gitlab.freedesktop.org/wlroots/wlr-randr
- libdrm 核心头文件（drm.h）：https://gitlab.freedesktop.org/mesa/drm/-/blob/main/include/drm/drm.h
- DRM uAPI 文档：https://www.kernel.org/doc/html/latest/gpu/drm-uapi.html
- Linux 内核图形子系统概述：https://www.kernel.org/doc/html/latest/gpu/index.html

## 今日小结

今天系统学习了 KMS（Kernel Mode Setting）的基础概念，包括其与 UMS 的区别、在 Linux 图形栈中的位置、核心组件（CRTC、Encoder、Connector、Plane）的定义与相互关系。深入理解了显示时序的水平和垂直同步机制、帧缓冲与像素格式的表示方法、CRTC 时序计算的 VESA CVT 标准。掌握了 KMS 的核心数据结构（drm_mode_config、drm_crtc、drm_display_mode、drm_connector）的字段含义。了解了 DRM 属性系统（Property）如何将 KMS 的控制接口暴露给用户空间，以及 Legacy KMS 与 Atomic Mode Setting 的区别。学习 VBlank 同步和页面翻转机制，理解了防止撕裂的关键技术。通过 modetest、libdrm API、sysfs、xrandr 等多个层面的实践操作，掌握了 KMS 的调试和测试方法。通过两个真实 Bug 的案例分析，了解了 KMS 初始化闪烁和热插拔状态同步问题的根因与修复方案。

KMS 作为 Linux 显示栈的基础，是后续深入学习 DRM 显示架构、AMD DC 显示核心、原子模式设置等高级主题的必备知识。掌握好 KMS 的概念和实践，对于显示驱动的开发和测试工作至关重要。

## 扩展思考

1. KMS 中的 Plane（硬件平面）概念在视频播放和游戏渲染中如何实现硬件覆盖层？对比软件合成与硬件合成的性能差异，考虑如何设计测试用例来验证 Plane 的正确性。

2. 分析 Atomic Mode Setting 相对于 Legacy Mode Setting 的优势。在设计测试框架时，如何确保测试用例在 Legacy 和 Atomic 两种模式下都能正确执行？考虑设计一个测试适配器来统一两种模式的测试接口。

3. 探讨 VBlank 同步在 VR/AR 设备中的特殊要求。VR 设备通常需要低延迟（<20ms）和高刷新率（90Hz+），这对 KMS 的 VBlank 中断处理和页面翻转机制提出了哪些额外的性能要求？如何设计测试来评估 KMS 的 VBlank 延迟特性？

4. 对比分析 AMDGPU 与 i915（Intel）驱动在 KMS 实现上的差异。两种驱动在 CRTC 初始化、Plane 管理、原子提交路径上有哪些不同的设计选择？这些差异对测试用例的设计有何影响？

5. 考虑一个场景：系统同时连接了 4K@120Hz 显示器（通过 DP 1.4）和 1080p@60Hz 显示器（通过 HDMI 2.0）。分析在这种混合配置下，KMS 如何协调两个 CRTC 的不同时序要求？Page flip 事件如何在不同刷新率的显示器上保持同步？
