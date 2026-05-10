# Day 337: 平面（Plane）管理：Primary / Overlay / Cursor

## 学习目标
- 理解 DRM 平面（Plane）的概念和在 DCN 硬件中的对应关系
- 掌握 Primary / Overlay / Cursor 三种平面类型的区别和用途
- 理解 DCN 硬件平面（DPP 管道）到 DRM 平面的映射机制
- 掌握平面的创建、配置和状态管理流程
- 理解多平面合成（MPC）与硬件光标的关系
- 掌握使用 libdrm / modetest 进行平面操作的方法

## 知识详解

### 1. 概念原理

#### 1.1 DRM 平面概念

在 DRM（Direct Rendering Manager）子系统中，平面（Plane）代表一个独立的图像层，可以独立调整位置、大小、旋转和色彩属性。DRM 定义了三类平面：

```
DRM 平面类型：
══════════════════════════════════════════════════════════════════════════════════════════════

  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  Plane Type  │  DRM 常量             │  硬件对应        │  数量        │  用途      │
  ├────────────────────────────────────────────────────────────────────────────────────┤
  │  Primary     │  DRM_PLANE_TYPE_PRIMARY│  DPP 管道 0     │  每个 CRTC 1 个│  主显示层  │
  │  Overlay     │  DRM_PLANE_TYPE_OVERLAY│  DPP 管道 1-3   │  每个 CRTC N 个│  叠加层    │
  │  Cursor      │  DRM_PLANE_TYPE_CURSOR │  硬件光标引擎    │  每个 CRTC 1 个│  鼠标指针  │
  └────────────────────────────────────────────────────────────────────────────────────┘

  DRM Plane 抽象:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  struct drm_plane {                                                                │
  │      struct drm_device *dev;          // 所属 DRM 设备                            │
  │      struct drm_crtc *crtc;          // 绑定的 CRTC                              │
  │      uint32_t possible_crtcs;         // 可绑定的 CRTC 位图                       │
  │      uint32_t format_count;           // 支持的像素格式数量                        │
  │      const uint32_t *format_types;    // 支持的像素格式列表                        │
  │      struct drm_plane_state *state;   // 当前状态                                  │
  │      const struct drm_plane_funcs *funcs;  // 操作函数表                          │
  │      unsigned int index;              // 平面索引                                  │
  │  };                                                                               │
  └────────────────────────────────────────────────────────────────────────────────────┘

  平面状态 (drm_plane_state):
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  struct drm_plane_state {                                                          │
  │      struct drm_plane *plane;          // 所属平面                                │
  │      struct drm_crtc *crtc;           // 绑定的 CRTC                            │
  │      struct drm_framebuffer *fb;       // 帧缓冲                                  │
  │                                                                                    │
  │      // 源区域 (帧缓冲中的像素区域)                                                │
  │      uint32_t src_x, src_y;           // 起始坐标 (16.16 定点数)                  │
  │      uint32_t src_w, src_h;           // 宽高 (16.16 定点数)                      │
  │                                                                                    │
  │      // 目标区域 (显示画面上的位置)                                                │
  │      int32_t crtc_x, crtc_y;          // 起始坐标 (整数)                          │
  │      uint32_t crtc_w, crtc_h;         // 宽高 (整数)                              │
  │                                                                                    │
  │      // 变换属性                                                                    │
  │      unsigned int rotation;           // 旋转 (DRM_MODE_ROTATE_*)                 │
  │      unsigned int blend_mode;         // 混合模式                                  │
  │      uint16_t alpha;                  // 全局 Alpha (0-65535)                     │
  │                                                                                    │
  │      // 颜色属性                                                                    │
  │      enum drm_color_encoding color_encoding;  // 颜色编码标准                     │
  │      enum drm_color_range color_range;         // 量化范围                        │
  │  };                                                                               │
  └────────────────────────────────────────────────────────────────────────────────────┘
══════════════════════════════════════════════════════════════════════════════════════════════
```

#### 1.2 Primary Plane（主平面）

主平面是每个 CRTC 必须的平面，代表桌面的主要显示内容。在 DCN 中，主平面通常映射到管道 0（DPP0）。

```
Primary Plane 属性列表 (通过 DRM 通用属性暴露)：
══════════════════════════════════════════════════════════════════════════════════════════════

  ┌────────────────────────┬───────────────┬──────────────────────────────────────────────┐
  │  属性名称              │  类型          │  说明                                       │
  ├────────────────────────┼───────────────┼──────────────────────────────────────────────┤
  │  CRTC_ID               │  只读          │  绑定的 CRTC ID                             │
  │  FB_ID                 │  可写          │  当前帧缓冲 ID                              │
  │  CRTC_X/Y              │  可写          │  在 CRTC 上的显示位置                       │
  │  CRTC_W/H              │  可写          │  显示宽高                                   │
  │  SRC_X/Y               │  可写          │  源区域起始 (16.16 定)                      │
  │  SRC_W/H               │  可写          │  源区域尺寸 (16.16 定)                      │
  │  alpha                 │  可写          │  全局 Alpha 值 (0-65535)                    │
  │  rotation              │  可写          │  旋转 (0/90/180/270) + 反射                  │
  │  COLOR_ENCODING        │  可写          │  颜色编码 (ITU-R BT.601/709/2020)           │
  │  COLOR_RANGE           │  可写          │  量化范围 (有限/全)                          │
  │  IN_FORMATS            │  只读          │  支持的像素格式列表                         │
  │  SIZE_HINTS            │  只读          │  建议的帧缓冲尺寸                           │
  │  HOTSPOT_X/Y           │  只读          │  光标热点 (仅 Cursor 平面有效)              │
  │  type                  │  只读          │  平面类型 (Primary/Overlay/Cursor)           │
  └────────────────────────┴───────────────┴──────────────────────────────────────────────┘

  AMDGPU 扩展属性 (通过 AMDGPU_DM 属性暴露):
  ┌────────────────────────┬───────────────┬──────────────────────────────────────────────┐
  │  AMDGPU 属性名称       │  类型          │  说明                                       │
  ├────────────────────────┼───────────────┼──────────────────────────────────────────────┤
  │  COLOR_SPACE           │  可写          │  AMD DC 颜色空间枚举                        │
  │  COLOR_TRANSFORM      │  可写          │  自定义颜色转换矩阵                          │
  │  HDR_OUTPUT_METADATA  │  可写          │  HDR 静态元数据 (ST.2086)                    │
  │  DEGAMMA_LUT          │  可写          │  DeGamma 校正 LUT                           │
  │  GAMMA_LUT             │  可写          │  Gamma 校正 LUT                             │
  │  DCC_ENABLE           │  只读          │  DCC 压缩启用标志                           │
  │  TMZ_SURFACE           │  只读          │  受保护内容表面标志                         │
  └────────────────────────┴───────────────┴──────────────────────────────────────────────┘
```

**Primary Plane 的创建流程（amdgpu_dm.c）：**

```c
// amdgpu_dm_create_plane() — 创建 DRM Plane

static int amdgpu_dm_create_plane(struct amdgpu_display_manager *dm,
                                  struct amdgpu_dm_plane *dm_plane,
                                  unsigned long possible_crtcs,
                                  enum drm_plane_type type)
{
    struct drm_plane *plane;
    const uint64_t *modifiers = NULL;
    struct drm_plane_helper_funcs *helper;
    int num_formats;

    // 根据平面类型选择支持的像素格式
    switch (type) {
    case DRM_PLANE_TYPE_PRIMARY:
        // 主平面支持所有常见桌面显示格式
        formats = amdgpu_dm_primary_formats;
        num_formats = ARRAY_SIZE(amdgpu_dm_primary_formats);
        modifiers = amdgpu_dm_primary_modifiers;
        helper = &dm_plane_helper_primary_funcs;
        break;

    case DRM_PLANE_TYPE_OVERLAY:
        // 覆盖平面通常用于视频播放，需要 NV12 支持
        formats = amdgpu_dm_overlay_formats;
        num_formats = ARRAY_SIZE(amdgpu_dm_overlay_formats);
        modifiers = amdgpu_dm_overlay_modifiers;
        helper = &dm_plane_helper_overlay_funcs;
        break;

    case DRM_PLANE_TYPE_CURSOR:
        // 光标平面仅支持 ARGB8888 小尺寸格式
        formats = amdgpu_dm_cursor_formats;
        num_formats = ARRAY_SIZE(amdgpu_dm_cursor_formats);
        modifiers = NULL;
        helper = &dm_plane_helper_cursor_funcs;
        break;
    }

    // 注册通用平面
    drm_universal_plane_init(dm->ddev, plane, possible_crtcs,
                             &amdgpu_dm_plane_funcs,
                             formats, num_formats,
                             modifiers, type, NULL);

    // 注册平面辅助函数
    drm_plane_helper_add(plane, helper);

    // 注册 DRM 通用属性
    drm_plane_create_alpha_property(plane);
    drm_plane_create_rotation_property(plane,
        DRM_MODE_ROTATE_0,
        DRM_MODE_ROTATE_0 | DRM_MODE_ROTATE_90 |
        DRM_MODE_ROTATE_180 | DRM_MODE_ROTATE_270 |
        DRM_MODE_REFLECT_X | DRM_MODE_REFLECT_Y);

    // 注册颜色属性
    drm_plane_create_color_properties(plane,
        DRM_COLOR_ENCODING_BT709,
        BIT(DRM_COLOR_YCBCR_LIMITED_RANGE));

    DRM_DEBUG_KMS("DM: Plane %d created (type=%d, %d formats)\n",
                  plane->index, type, num_formats);
    return 0;
}
```

#### 1.3 Overlay Plane（覆盖平面）

覆盖平面用于在主平面上方叠加显示内容，典型应用包括视频播放、画中画、菜单叠加等。在 DCN 中，覆盖平面映射到管道 1-3（DPP1-DPP3）。

```
Overlay Plane 典型应用场景：
══════════════════════════════════════════════════════════════════════════════════════════════

  场景 1: 视频播放器叠加
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │                    ┌──────────────────────────────────────┐                        │
  │                    │  视频覆盖层 (DPP1)                     │                        │
  │                    │  格式: NV12 (YUV 420 平面)            │                        │
  │                    │  分辨率: 3840×2160                     │                        │
  │                    │  Alpha: 1.0 (完全不透明)               │                        │
  │                    │  缩放: 适配播放器窗口                   │                        │
  │                    └──────────────────────────────────────┘                        │
  │  ┌────────────────────────────────────────────────────────────────────────────────┐│
  │  │  桌面主层 (DPP0)                                                               ││
  │  │  格式: ARGB8888                                                                 ││
  │  │  分辨率: 1920×1080 (桌面壁纸 + 图标 + 窗口装饰)                                 ││
  │  └────────────────────────────────────────────────────────────────────────────────┘│
  └────────────────────────────────────────────────────────────────────────────────────┘
  MPC 混合: 主层 + 覆盖层 = 最终输出
  注意: 视频播放使用覆盖层的好处是避免将 YUV 转为 RGB 再合成，
        硬件直接处理 YUV 数据，节省带宽和功耗。

  场景 2: HDR 游戏 + SDR 直播叠加
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │                    ┌──────────────────────────────────────┐                        │
  │                    │  直播预览 (DPP2)                       │                        │
  │                    │  格式: ARGB8888                        │                        │
  │                    │  HDR→SDR 色调映射后叠加                │                        │
  │                    └──────────────────────────────────────┘                        │
  │  ┌────────────────────────────────────────────────────────────────────────────────┐│
  │  │  HDR 游戏主层 (DPP0)                                                           ││
  │  │  格式: ARGB2101010 (10bit HDR)                                                ││
  │  │  HDR 元数据: ST.2086 (MaxCLL=1000 nits, MaxFALL=400 nits)                    ││
  │  └────────────────────────────────────────────────────────────────────────────────┘│
  └────────────────────────────────────────────────────────────────────────────────────┘
  MPC 混合: 需要 HDR→SDR 色调映射，DCN 3.0+ 支持硬件色调映射

  场景 3: 画中画 (PiP)
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  ┌──────────────────────────────────────────────────────┐                          │
  │  │  主画面 (DPP0)                                         │                          │
  │  │  全屏显示                                              │                          │
  │  │  ┌──────────────────────────────────┐                  │                          │
  │  │  │  PiP 子画面 (DPP1)                │                  │                          │
  │  │  │  位置: 右下角                     │                  │                          │
  │  │  │  大小: 320×180                    │                  │                          │
  │  │  │  Alpha: 1.0                       │                  │                          │
  │  │  │  边框: MPC 背景色填充             │                  │                          │
  │  │  └──────────────────────────────────┘                  │                          │
  │  └──────────────────────────────────────────────────────┘                          │
  └────────────────────────────────────────────────────────────────────────────────────┘
══════════════════════════════════════════════════════════════════════════════════════════════
```

**Overlay Plane 的关键能力：**

| 能力 | 描述 | DCN 支持情况 |
|------|------|-------------|
| 独立缩放 | 覆盖层可以独立于主层进行缩放 | DCN 1.0+ |
| YUV 直接输入 | 支持 NV12/P010 等视频格式，无需 CPU 转换 | DCN 1.0+ |
| 独立颜色空间 | 覆盖层可使用不同于主层的颜色空间 | DCN 2.0+ |
| 逐像素 Alpha | 支持带 Alpha 通道的像素混合 | DCN 1.0+ |
| 全局 Alpha | 支持整个平面的透明度控制 | DCN 1.0+ |
| 旋转/反射 | 支持 0/90/180/270 旋转和水平/垂直反射 | DCN 2.1+ |
| HDR 色调映射 | 自动将 SDR 覆盖层映射到 HDR 输出 | DCN 3.0+ |

#### 1.4 Cursor Plane（光标平面）

光标平面是专门用于硬件鼠标光标的特殊平面，具有独立的硬件路径，不占用 DPP 管道。

```
硬件光标路径（独立于主显示管线）：
══════════════════════════════════════════════════════════════════════════════════════════════

  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  硬件光标数据流:                                                                     │
  │                                                                                    │
  │  帧缓冲中的光标图像 (cursor_buffer)                                                │
  │       │                                                                             │
  │       │  (通过 HUBP 的独立光标通道)                                                  │
  │       ▼                                                                             │
  │  ┌──────────────────┐                                                               │
  │  │  光标引擎          │                                                               │
  │  │  ┌──────────────┐ │          ┌──────────────────┐                               │
  │  │  │ 光标属性:     │ │          │  cursor_buffer    │                               │
  │  │  │  - 位置 (x,y) │ │          │  大小: 64×64     │                               │
  │  │  │  - 热点 (hx,hy)│ │          │  格式: ARGB8888  │                               │
  │  │  │  - 大小 (w,h) │ │          │  颜色键: 透明    │                               │
  │  │  │  - 显示/隐藏  │ │          └──────────────────┘                               │
  │  │  └──────────────┘ │                                                               │
  │  │  Cursor LUT       │          ┌──────────────────┐                               │
  │  │  (可选 2bit 光标) │          │  2bit 光标 LUT    │                               │
  │  │  ┌──────────────┐ │          │  - 颜色 0: 透明   │                               │
  │  │  │ 色 0→透明    │ │          │  - 颜色 1: 黑色   │                               │
  │  │  │ 色 1→白色    │ │          │  - 颜色 2: 白色   │                               │
  │  │  │ 色 2→黑色    │ │          │  - 颜色 3: XOR    │                               │
  │  │  │ 色 3→XOR     │ │          └──────────────────┘                               │
  │  │  └──────────────┘ │                                                               │
  │  └────────┬─────────┘                                                               │
  │           │ 光标像素流 (32bpp)                                                       │
  │           ▼                                                                          │
  │  ┌────────────────────────────────────────────────────────────┐                      │
  │  │  MPC (Multiple Pipe Combiner) — 光标作为顶层叠加             │                      │
  │  │  在主层和覆盖层混合完成后，在最后一层叠加光标                  │                      │
  │  │  光标覆盖在 MPC tree 的最顶层，不受缩放影响                  │                      │
  │  └────────────────────────────────────────────────────────────┘                      │
  │           │                                                                          │
  │           ▼                                                                          │
  │      OPP → OTG → DSC → Link Encoder → Display                                      │
  │                                                                                    │
  │  光标限制:                                                                           │
  │  ┌────────────────────────────────────────────────────────────────────────────────┐  │
  │  │  - 最大光标尺寸: 64×64 (典型) / 128×128 (DCN 3.0+)                              │  │
  │  │  - 不能缩放                                                                      │  │
  │  │  - 不能旋转                                                                      │  │
  │  │  - 仅限 ARGB8888 或 2bit 游标                                                   │  │
  │  │  - 每个 CRTC 仅一个光标                                                          │  │
  │  │  - 更新光标位置不触发页面翻转 (低开销)                                            │  │
  │  └────────────────────────────────────────────────────────────────────────────────┘  │
  └────────────────────────────────────────────────────────────────────────────────────┘
```

**硬件光标与软件光标对比：**

| 特性 | 硬件光标 | 软件光标 |
|------|---------|---------|
| 实现位置 | DCN 硬件 | 驱动/合成器 (Wayland/X11) |
| 更新延迟 | < 1μs | ~ 1 帧延迟 (16.7ms @ 60Hz) |
| 功耗影响 | 极低 (硬件处理) | 需要 GPU 重新合成画面 |
| 光标大小限制 | 64×64 / 128×128 | 任意尺寸 |
| 缩放支持 | 不支持 | 支持 (通过主层缩放) |
| 光标更新频率 | 独立于帧率 | 与帧率相同 |
| 多光标支持 | 每 CRTC 一个 | 软件可支持任意数量 |
| 使用场景 | 桌面环境 | 特殊需求 (大光标/多光标) |

#### 1.5 DCN 平面到硬件管道的映射

```
平面到硬件管道的完整映射 (DCN 3.1, 4 管道配置)：
══════════════════════════════════════════════════════════════════════════════════════════════

  DRM 平面           DCN 硬件管道     MPC 混合层     CRTC 绑定
  ─────────────────────────────────────────────────────────────────
  Plane 0 (Primary) → Pipe 0 (DPP0) → MPC_0 (底层) → CRTC 0
  Plane 1 (Overlay)  → Pipe 1 (DPP1) → MPC_1 (层 1) → CRTC 0
  Plane 2 (Overlay)  → Pipe 2 (DPP2) → MPC_2 (层 2) → CRTC 0
  Plane 3 (Cursor)   → Cursor Engine → MPC_0 (顶层) → CRTC 0
  Plane 4 (Primary)  → Pipe 0 (DPP0) → MPC_0 (底层) → CRTC 1
  Plane 5 (Overlay)  → Pipe 1 (DPP1) → MPC_1 (层 1) → CRTC 1
  Plane 6 (Cursor)   → Cursor Engine → MPC_0 (顶层) → CRTC 1

  在 amdgpu_dm 中的映射代码 (amdgpu_dm_plane.c):
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  static int dm_plane_atomic_check(struct drm_plane *plane,                        │
  │                                    struct drm_atomic_state *state)                │
  │  {                                                                                 │
  │      struct dm_plane_state *dm_plane_state = to_dm_plane_state(new_state);        │
  │      struct amdgpu_dm_plane *dm_plane = to_amdgpu_dm_plane(plane);                │
  │                                                                                    │
  │      // 检查硬件管道可用性                                                        │
  │      if (plane->type == DRM_PLANE_TYPE_CURSOR) {                                  │
  │          // 光标平面使用专用硬件光标引擎，不占用 DPP 管道                          │
  │          return 0;                                                                │
  │      }                                                                             │
  │                                                                                    │
  │      // 检查格式是否被 DCN 硬件支持                                                │
  │      if (!is_format_supported(plane, new_state->fb)) {                             │
  │          drm_dbg_kms(plane->dev, "Unsupported format\n");                         │
  │          return -EINVAL;                                                           │
  │      }                                                                             │
  │                                                                                    │
  │      // 检查源区域和目标区域的有效性                                              │
  │      if (new_state->src_w > (new_state->fb->width << 16) ||                       │
  │          new_state->src_h > (new_state->fb->height << 16)) {                       │
  │          return -EINVAL;                                                           │
  │      }                                                                             │
  │                                                                                    │
  │      // 检查缩放能力                                                              │
  │      if (new_state->crtc_w > dcn_max_plane_size ||                                │
  │          new_state->crtc_h > dcn_max_plane_size) {                                │
  │          drm_dbg_kms(plane->dev, "Plane size exceeds DCN limit\n");               │
  │          return -EINVAL;                                                           │
  │      }                                                                             │
  │                                                                                    │
  │      return 0;                                                                     │
  │  }                                                                                 │
  └────────────────────────────────────────────────────────────────────────────────────┘
══════════════════════════════════════════════════════════════════════════════════════════════
```

### 2. 实践操作

#### 2.1 查看当前平面的状态

```bash
# 查看所有平面及其属性
modetest -M amdgpu -p

# 示例输出:
# id    type            CRTC    formats                                    count
# 30    Primary         34      ARGB8888 XRGB8888 RGBA8888 ...            12
# 31    Overlay         34      NV12 ARGB8888 XRGB8888 ...                 8
# 32    Cursor          34      ARGB8888                                    1
# 33    Primary         35      ARGB8888 XRGB8888 RGBA8888 ...            12
# 34    Overlay         35      NV12 ARGB8888 XRGB8888 ...                 8
# 35    Cursor          35      ARGB8888                                    1

# 查看平面的详细属性
modetest -M amdgpu -p 30

# 查看当前 CRTC 的平面配置
cat /sys/kernel/debug/dri/0/state | grep -A 15 "plane"
```

#### 2.2 使用 modetest 测试平面操作

```bash
# 测试主平面显示
modetest -M amdgpu -s 34:1920x1080-60 -v

# 测试覆盖层叠加
modetest -M amdgpu \
    -s 34:1920x1080-60@30 \      # 主平面: CRTC 34, 1920x1080@60, plane 30
    -w 31:34,100,100,640x480 \   # 覆盖层: plane 31, CRTC 34, 位置(100,100), 640x480
    -v

# 测试光标操作
modetest -M amdgpu \
    -s 34:1920x1080-60@30 \
    -C 34,500,300,64x64 \        # 光标: CRTC 34, 位置(500,300), 64x64
    -v

# 测试平面 Alpha 混合
modetest -M amdgpu \
    -s 34:1920x1080-60@30 \
    -w 31:34,0,0,1920x1080,alpha=32768 \  # 覆盖层半透明 (alpha=0.5)
    -v
```

#### 2.3 使用 IGT 测试平面功能

```bash
# 安装 IGT GPU Tools
git clone https://gitlab.freedesktop.org/drm/igt-gpu-tools.git
cd igt-gpu-tools
meson build && ninja -C build

# 运行平面测试
sudo ./build/tests/kms_plane --run-subtest plane-position-overlap
sudo ./build/tests/kms_plane --run-subtest plane-position-hole
sudo ./build/tests/kms_plane --run-subtest plane-panning-top-left

# 测试覆盖平面
sudo ./build/tests/kms_plane_multiple --run-subtest atomic

# 测试光标平面
sudo ./build/tests/kms_cursor_crc --run-subtest cursor-size-64
sudo ./build/tests/kms_cursor_crc --run-subtest cursor-alpha-opaque
```

#### 2.4 使用 DRM API 直接控制平面

```c
// 使用 libdrm 直接控制平面的 C 代码示例
// 编译: gcc -o plane_test plane_test.c -ldrm

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

int main(int argc, char **argv)
{
    int fd;
    drmModePlaneRes *plane_res;
    drmModePlane *plane;
    int i, j;

    // 打开 DRM 设备
    fd = drmOpen("amdgpu", NULL);
    if (fd < 0) {
        perror("drmOpen failed");
        return 1;
    }

    // 获取平面资源
    plane_res = drmModeGetPlaneResources(fd);
    if (!plane_res) {
        perror("drmModeGetPlaneResources failed");
        return 1;
    }

    printf("Found %d planes:\n", plane_res->count_planes);

    // 枚举所有平面
    for (i = 0; i < plane_res->count_planes; i++) {
        plane = drmModeGetPlane(fd, plane_res->planes[i]);
        if (!plane) continue;

        // 确定平面类型
        const char *type_str;
        switch (plane->plane_type) {
        case DRM_PLANE_TYPE_PRIMARY:
            type_str = "Primary";
            break;
        case DRM_PLANE_TYPE_OVERLAY:
            type_str = "Overlay";
            break;
        case DRM_PLANE_TYPE_CURSOR:
            type_str = "Cursor";
            break;
        default:
            type_str = "Unknown";
        }

        printf("  Plane %d: type=%s, CRTC=%d\n",
               plane->plane_id, type_str, plane->crtc_id);
        printf("    possible_crtcs: 0x%x\n", plane->possible_crtcs);
        printf("    formats: ");

        for (j = 0; j < plane->count_formats; j++) {
            char format_str[5];
            format_str[0] = (plane->formats[j] >> 0) & 0xFF;
            format_str[1] = (plane->formats[j] >> 8) & 0xFF;
            format_str[2] = (plane->formats[j] >> 16) & 0xFF;
            format_str[3] = (plane->formats[j] >> 24) & 0xFF;
            format_str[4] = '\0';
            printf("%s ", format_str);
        }
        printf("\n");

        drmModeFreePlane(plane);
    }

    drmModeFreePlaneResources(plane_res);
    drmClose(fd);
    return 0;
}
```

#### 2.5 使用 bpftrace 追踪平面更新

```bash
# 追踪平面原子更新
sudo bpftrace -e '
    kprobe:amdgpu_dm_plane_atomic_update {
        printf("Plane update: CRTC=%d, src=(%d,%d,%d,%d), dst=(%d,%d,%d,%d)\n",
            arg0, arg2->src_x>>16, arg2->src_y>>16,
            arg2->src_w>>16, arg2->src_h>>16,
            arg2->crtc_x, arg2->crtc_y,
            arg2->crtc_w, arg2->crtc_h);
    }
    kprobe:dc_commit_planes_for_stream {
        printf("DC commit planes: surface_count=%d, stream=%p\n",
            arg0, arg2);
    }
'
```

#### 2.6 监控平面资源利用率

```bash
# 查看每个平面的使用情况
watch -n 1 '
    echo "=== Plane Status ==="
    for plane in /sys/kernel/debug/dri/0/plane-*/; do
        name=$(basename $plane)
        if [ -f "$plane/state" ]; then
            crtc=$(grep "CRTC_ID" $plane/state | awk "{print \$2}")
            fb=$(grep "FB_ID" $plane/state | awk "{print \$2}")
            echo "$name: CRTC=$crtc FB=$fb"
        fi
    done
'
```

### 3. 案例分析

#### 3.1 Bug 分析: 覆盖平面 NV12 格式缩放置乱

**Bug 信息**：
- 来源：freedesktop.org GitLab issue #21345
- 标题：NV12 overlay plane shows corrupted colors when scaled on DCN 3.1
- 链接：https://gitlab.freedesktop.org/drm/amd/-/issues/21345

**问题描述**：
在 DCN 3.1 上，使用 NV12（YUV 420）格式的覆盖平面进行缩放时，显示颜色异常，表现为绿色/紫色条带。仅在覆盖平面启用了 MPC 混合时才复现，主平面无此问题。

**根因分析**：

```
NV12 缩放颜色问题分析：
══════════════════════════════════════════════════════════════════════════════════════════════

  NV12 格式内存布局:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  NV12 是 YUV 420 平面格式:                                                         │
  │    Y 平面: 宽度 × 高度 × 1 字节 (亮度)                                           │
  │    UV 平面: 宽度/2 × 高度/2 × 2 字节 (色度, 交错)                                │
  │                                                                                    │
  │  ┌─────── Y 平面 ───────┐                                                          │
  │  │ Y00 Y01 Y02 Y03 ...   │  1920 × 1080 × 1 = 2,073,600 bytes                      │
  │  │ ...                    │                                                          │
  │  └───────────────────────┘                                                          │
  │  ┌────── UV 平面 ───────┐                                                          │
  │  │ U00 V00 U01 V01 ...   │  960 × 540 × 2 = 1,036,800 bytes                        │
  │  └───────────────────────┘                                                          │
  └────────────────────────────────────────────────────────────────────────────────────┘

  问题根因:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  DPP 缩放器在处理 NV12 时，Y 平面和 UV 平面独立缩放。                              │
  │  缩放 Y 平面时，正确的坐标为:                                                      │
  │    src_y_uv = src_y / 2 (因为 UV 平面高度是 Y 的一半)                               │
  │    src_h_uv = src_h / 2                                                             │
  │                                                                                    │
  │  Bug: DPP 缩放器在计算 UV 平面的缩放坐标时，未考虑 Y 平面的起始偏移                 │
  │  │                                                                                    │
  │  设覆盖层在帧缓冲中的偏移为 (100, 100):                                            │
  │    Y 起始: pixels[100][100] → 正确                                                  │
  │    UV 起始: pixels[1080 + (100/2)][100/2] → 但实际应为                             │
  │              pixels[1080 + 50][50]                                                   │
  │                                                                                    │
  │  修复: 在 dcn31_dpp_set_scaler() 中, 对 NV12 格式                                  │
  │  计算 UV 平面坐标时, 考虑 Y 平面的偏移量                                              │
  └────────────────────────────────────────────────────────────────────────────────────┘

  修复补丁摘要:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  diff --git a/drivers/gpu/drm/amd/display/dc/dcn31/dcn31_dpp.c                    │
  │  @@ -312,6 +312,17 @@ void dcn31_dpp_set_scaler_filter(struct dpp *dpp,          │
  │   +    // 对于 NV12 格式, 需要校正 UV 平面缩放坐标                                  │
  │   +    if (scaler_data->format == PIXEL_FORMAT_420BPP12) {                        │
  │   +        scaler_data->v_init_uv = scaler_data->v_init / 2;                      │
  │   +        scaler_data->h_init_uv = scaler_data->h_init / 2;                      │
  │   +        scaler_data->v_calc_max_uv = scaler_data->v_calc_max / 2;              │
  │   +        scaler_data->h_calc_max_uv = scaler_data->h_calc_max / 2;              │
  │   +    }                                                                          │
  └────────────────────────────────────────────────────────────────────────────────────┘
```

#### 3.2 Bug 分析: 硬件光标在 VRR 模式下的抖动

**Bug 信息**：
- 来源：GitLab issue #15678
- 标题：Hardware cursor jitters on FreeSync display when frame rate varies
- 链接：https://gitlab.freedesktop.org/drm/amd/-/issues/15678

**问题描述**：
在 FreeSync 模式下，硬件光标在帧率快速变化（如游戏场景切换）时出现明显的左右抖动。光标位置更新频率是 1000Hz（每 1ms 更新一次），但显示刷新率在 48-144Hz 之间变化。

**根因**：
光标位置寄存器更新和显示帧扫描不同步导致。驱动在用户空间更新光标位置后立即写入硬件寄存器，但由于显示扫描处于不同位置，光标在屏幕上的出现位置在不同帧之间轻微偏移。

```
光标抖动根因：
══════════════════════════════════════════════════════════════════════════════════════════════

  问题序列:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  时间    │ 帧率    │ 光标更新     │ 写入 OTG 时的扫描位置                            │
  │  T0      │ 144Hz   │ Cursor(100,100)│ 扫描至行 500 (恰好 VBlank 后)              │
  │  T0+7ms  │ 144Hz   │ Cursor(100,100)│ 扫描至行 1080 (VBlank 边界)                 │
  │  T0+14ms │ 48Hz    │ Cursor(101,101)│ 扫描至行 200                                 │
  │  T0+21ms │ 48Hz    │ Cursor(102,102)│ 扫描至行 800                                 │
  │                                                                            │
  │  结果: 光标在不同帧中出现在屏幕上的位置不同                                     │
  │  Frame N: 光标位置(100,100)，显示在行500                                      │
  │  Frame N+1(48Hz): 光标位置(102,102)，显示在行200                              │
  │  视觉上: 光标看起来在水平方向上抖动                                            │
  └────────────────────────────────────────────────────────────────────────────────────┘

  修复:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  方案: 仅在 VBlank 中断处理函数中更新光标位置                                     │
  │  ┌──────────────────────────────────────────────┐                                 │
  │  │  if (in_irq() && amdgpu_dm_is_vblank(dm)) {  │                                 │
  │  │      dc_stream_set_cursor_position(stream);   │                                 │
  │  │  }                                            │                                 │
  │  └──────────────────────────────────────────────┘                                 │
  │                                                                                    │
  │  替代方案: 使用双缓冲光标位置寄存器                                                │
  │  - 写入新位置到 shadow 寄存器                                                     │
  │  - 在 VBlank 边界硬件自动将 shadow 复制到 active 寄存器                           │
  └────────────────────────────────────────────────────────────────────────────────────┘
```

#### 3.3 Bug 分析: 多平面配置的内存带宽饱和

**Bug 信息**：
- 来源：Phoronix 测试报告
- 标题：DCN 3.1 multi-plane performance analysis with 8K displays
- 链接：https://www.phoronix.com/

**问题描述**：
在 8K 显示器上启用 4 个平面（1 Primary + 3 Overlay）时，帧率从 60FPS 降至 32FPS。HUBP 的内存读取带宽达到饱和。

```
内存带宽分析：
══════════════════════════════════════════════════════════════════════════════════════════════

  8K (7680×4320) 4 平面配置:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  Pipe 0 (Primary): 7680×4320×4B×60Hz = 7.96 GB/s                                 │
  │  Pipe 1 (Overlay):  1920×1080×4B×60Hz = 0.50 GB/s                                │
  │  Pipe 2 (Overlay):  640×480×4B×60Hz   = 0.07 GB/s                                │
  │  Pipe 3 (Overlay):  100×100×4B×60Hz   = 0.002 GB/s                               │
  │  ─────────────────────────────────────────────────                                      │
  │  总计: 8.53 GB/s                                                                    │
  │                                                                                    │
  │  DCN 3.1 HUBP 内存带宽限制:                                                        │
  │  - GDDR6 带宽: ~800 GB/s (理论)                                                   │
  │  - HUBP 读取带宽上限: ~50 GB/s (受 HUBP 内部 FIFO 限制)                            │
  │  - 多平面并发读取时的仲裁损耗: ~30%                                                │
  │                                                                                    │
  │  测试结果: 4 平面开启 → 实际可用带宽降至 ~7.5 GB/s → 帧率受限                      │
  └────────────────────────────────────────────────────────────────────────────────────┘
```

### 4. 相关链接

- AMDGPU DM 平面管理: drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_plane.c
- DRM 平面核心: drivers/gpu/drm/drm_plane.c
- DCN 颜色管理: drivers/gpu/drm/amd/display/dc/dcn31/dcn31_dpp_cm.c
- DCN 光标实现: drivers/gpu/drm/amd/display/dc/dcn31/dcn31_hwseq.c
- IGT 平面测试: tests/kms_plane.c, tests/kms_cursor_crc.c
- libdrm 平面 API: xf86drmMode.h
- Weston 合成器平面实现: https://gitlab.freedesktop.org/wayland/weston
- AMDGPU 平面属性调试: https://docs.amd.com/

## 今日小结

今日我们深入学习了 DCN 中平面（Plane）管理系统的完整架构：

1. **三类平面**：Primary（主平面，必选，Pipe 0）、Overlay（覆盖平面，可选，Pipe 1-3）、Cursor（光标平面，独立硬件引擎）。每类平面在 DRM 层有独立的对象表示。

2. **平面属性**：通过 DRM 通用属性和 AMDGPU 扩展属性暴露给用户空间，支持位置、缩放、Alpha 混合、旋转、色彩空间等控制。

3. **多平面合成**：MPC（Multiple Pipe Combiner）负责将所有平面合成为最终画面。DPP 管道处理像素级操作（缩放、色彩校正），MPC Tree 处理层间混合。

4. **硬件光标**：使用专用硬件路径，不占用 DPP 管道。支持 64x64/128x128 尺寸、ARGB8888 格式和 LUT 颜色模式。更新开销极低。

5. **平面约束**：每个平面有最大尺寸限制、格式限制、缩放比限制。DCN 3.0+ 支持的平面数量取决于硬件管道数。

**预告**：明日我们将学习页面翻转（Page Flip）与 VBlank 同步机制，理解驱动如何确保画面在垂直消隐期安全切换缓冲区而不产生撕裂。

## 扩展思考

1. **覆盖层的使用策略**：在 Wayland 合成器（如 Weston 或 KWin）中，哪些场景应该使用覆盖层，哪些应该使用主层合成？过度使用覆盖层会不会反而降低性能？

2. **硬件光标 vs 软件光标**：在哪些场景下硬件光标不可用或需要回退到软件光标？（例如：远程桌面、虚拟机、高 DPI 缩放等）驱动如何优雅地处理这种回退？

3. **平面数量与功耗**：每个额外的平面（DPP 管道）约增加多少功耗？在移动设备（笔记本 APU）上，驱动是否有策略在不需要时动态关闭未使用的 DPP 管道以节省功耗？

4. **DRM 原子模式设置与平面更新**：原子 Mode Setting 如何保证多个平面的更新是同步且一致的？如果其中一个平面的更新验证失败，如何回滚整个提交？

5. **NV12 平面性能优化**：视频播放通常使用 NV12 覆盖平面。分析 NV12 格式在 HUBP 读取、DPP 缩放和 MPC 混合各个环节的带宽和计算开销，与直接使用 ARGB8888 视频帧相比有哪些优势？
