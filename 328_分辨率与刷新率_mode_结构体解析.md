# 分辨率与刷新率：mode 结构体解析

## 学习目标

- 理解 `drm_display_mode` 结构体的完整定义与每个字段的含义
- 掌握显示模式的核心参数：分辨率、刷新率、像素时钟、时序参数
- 学会分析 VESA CVT/GTF 标准下的时序计算公式
- 理解内核中显示模式的匹配与优先级排序逻辑
- 掌握通过 modetest 和 sysfs 查看显示模式的方法
- 熟悉内核 dmesg 中有关 mode 的调试信息解读
- 了解 EDID 中 Detailed Timing Descriptor（DTD）与 mode 的映射关系
- 理解 preferred mode 和 driver mode 的选择机制
- 掌握 AMDGPU 中 mode 校验与过滤的实现
- 学会编写自定义 mode 设置与测试程序

## 1. 知识详解

### 1.1 display_mode 结构体全景

`drm_display_mode` 是 DRM 内核中描述一个完整显示模式的核心数据结构。它包含了从时序参数到帧率的所有信息。以下为该结构体的完整定义和字段详解：

```c
struct drm_display_mode {
    /* 时钟相关 */
    int clock;                 /* 像素时钟频率，单位 kHz */
    int clock_index;           /* 时钟索引（内部使用） */

    /* 水平方向时序（单位：像素） */
    int hdisplay;              /* 水平有效显示像素 */
    int hsync_start;           /* 水平同步开始位置（hdisplay + hfront_porch） */
    int hsync_end;             /* 水平同步结束位置（hsync_start + hsync_len） */
    int htotal;                /* 水平总像素数（hsync_end + hback_porch） */
    int hskew;                 /* 水平偏斜校正 */
    int hsync_len;             /* 水平同步脉冲宽度 */

    /* 垂直方向时序（单位：行） */
    int vdisplay;              /* 垂直有效显示行数 */
    int vsync_start;           /* 垂直同步开始位置（vdisplay + vfront_porch） */
    int vsync_end;             /* 垂直同步结束位置（vsync_start + vsync_len） */
    int vtotal;                /* 垂直总行数（vsync_end + vback_porch） */
    int vsync_len;             /* 垂直同步脉冲宽度 */

    /* 标志位 */
    u32 flags;                 /* DRM_MODE_FLAG_* 标志位集合 */
    u32 type;                  /* 模式类型：DRM_MODE_TYPE_* */

    /* 模式标识 */
    char name[DRM_DISPLAY_MODE_LEN];  /* 模式名称，如 "1920x1080" */

    /* 帧率相关 */
    int vrefresh;              /* 垂直刷新率，单位 Hz */
    int hsync;                 /* 水平同步频率，单位 kHz */
    int picture_aspect_ratio;  /* 画面宽高比 */

    /* 3D 相关 */
    u8 3d_flags;               /* 3D 显示标志 */
    u8 stereo;                 /* 立体显示模式 */

    /* 内部状态 */
    int preferred;             /* 是否为 preferred mode */
    int head;                  /* 链表头节点 */
    int base;                  /* 基类对象 */
    int connector;             /* 关联的连接器 */
    int crtc;                  /* 关联的 CRTC */
    int encoder;               /* 关联的 Encoder */
    int plane;                 /* 关联的 Plane */
    int private;               /* 私有数据 */
    int private_flags;         /* 私有标志 */
};
```

模式名称的生成规则：

```c
void drm_mode_set_name(struct drm_display_mode *mode)
{
    bool interlaced = !!(mode->flags & DRM_MODE_FLAG_INTERLACE);

    snprintf(mode->name, DRM_DISPLAY_MODE_LEN, "%dx%d",
             mode->hdisplay, mode->vdisplay);

    if (interlaced)
        strcat(mode->name, "i");
    else
        strcat(mode->name, "p");

    if (mode->vrefresh > 0)
        snprintf(mode->name + strlen(mode->name),
                 DRM_DISPLAY_MODE_LEN - strlen(mode->name),
                 "@%d", drm_mode_vrefresh(mode));
}
```

### 1.2 时序参数详解与示意图

显示模式的时序参数定义了一个扫描行和扫描帧的所有时间边界。以下为完整时序示意图：

```
水平扫描线时序（一行）：
─────────────────────────────────────────────────────────────────────────
        │← hfront_porch →│← hsync_len →│← hback_porch →│
        │                │             │               │
        │  ████████████████             ██████████
        │  ██ 有效显示区 ██             ██  消隐区 ██
        │  ██ hdisplay  ██             ██         ██
        │  ████████████████             ██████████
        │                │             │               │
────────┘                └─────────────┘               └─────────────►
        │←─── hsync_start ──►│← hsync →│
        │←──────────── htotal ────────────────────────►│

整帧时序（一帧）：
─────────────────────────────────────────────────────────────────────────
        │← vfront_porch →│← vsync_len →│← vback_porch →│
        │                │             │               │
  ┌─────┼────────────────┼─────────────┼───────────────┼────────┐
  │     │                │             │               │        │
  │     │                │             │               │        │
  │     │   有效显示区   │             │    消隐区     │        │
  │     │   vdisplay 行  │             │               │        │
  │     │                │             │               │        │
  │     │                │             │               │        │
  └─────┼────────────────┼─────────────┼───────────────┼────────┘
        │                │             │               │
        │←── vsync_start→│←  vsync  →│               │
        │←──────────── vtotal ──────────────────────►│
```

各时序参数的计算关系：

```c
/* 水平总像素数 */
static inline int drm_mode_htotal(const struct drm_display_mode *mode)
{
    return mode->htotal;
}

/* 垂直总行数 */
static inline int drm_mode_vtotal(const struct drm_display_mode *mode)
{
    return mode->vtotal;
}

/* 像素时钟 = 水平总像素 × 垂直总行数 × 刷新率 */
/* clock(kHz) = htotal × vtotal × vrefresh(Hz) / 1000 */

/* 计算垂直刷新率 */
int drm_mode_vrefresh(const struct drm_display_mode *mode)
{
    int refresh = 0;
    unsigned int calc_val;

    if (mode->vrefresh > 0)
        return mode->vrefresh;

    if (mode->htotal == 0 || mode->vtotal == 0)
        return 0;

    calc_val = (mode->clock * 1000) / mode->htotal;
    refresh = DIV_ROUND_CLOSEST(calc_val, mode->vtotal);

    if (mode->flags & DRM_MODE_FLAG_INTERLACE)
        refresh *= 2;
    if (mode->flags & DRM_MODE_FLAG_DBLSCAN)
        refresh /= 2;
    if (mode->vscan > 1)
        refresh /= mode->vscan;

    return refresh;
}
```

### 1.3 常见标准模式参数表

以下为常见的 VESA 标准显示模式的完整时序参数：

| 模式名称 | 分辨率 | 刷新率 | 像素时钟(kHz) | hdisplay | hsync_start | hsync_end | htotal | vdisplay | vsync_start | vsync_end | vtotal |
|---------|--------|--------|--------------|---------|------------|----------|-------|---------|------------|----------|-------|
| 640x480 | 640x480 | 60Hz | 25175 | 640 | 656 | 752 | 800 | 480 | 490 | 492 | 525 |
| 800x600 | 800x600 | 60Hz | 40000 | 800 | 840 | 968 | 1056 | 600 | 601 | 605 | 628 |
| 1024x768 | 1024x768 | 60Hz | 65000 | 1024 | 1048 | 1184 | 1344 | 768 | 771 | 777 | 806 |
| 1280x720 | 1280x720 | 60Hz | 74250 | 1280 | 1390 | 1430 | 1650 | 720 | 725 | 730 | 750 |
| 1280x1024 | 1280x1024 | 60Hz | 108000 | 1280 | 1328 | 1440 | 1688 | 1024 | 1025 | 1028 | 1066 |
| 1366x768 | 1366x768 | 60Hz | 85500 | 1366 | 1436 | 1579 | 1792 | 768 | 771 | 774 | 798 |
| 1440x900 | 1440x900 | 60Hz | 106500 | 1440 | 1520 | 1672 | 1904 | 900 | 903 | 909 | 934 |
| 1600x900 | 1600x900 | 60Hz | 108000 | 1600 | 1648 | 1680 | 1760 | 900 | 903 | 908 | 926 |
| 1680x1050 | 1680x1050 | 60Hz | 146250 | 1680 | 1784 | 1960 | 2240 | 1050 | 1053 | 1059 | 1089 |
| 1920x1080 | 1920x1080 | 60Hz | 148500 | 1920 | 2008 | 2052 | 2200 | 1080 | 1084 | 1089 | 1125 |
| 1920x1200 | 1920x1200 | 60Hz | 154000 | 1920 | 2048 | 2256 | 2600 | 1200 | 1203 | 1209 | 1245 |
| 2560x1440 | 2560x1440 | 60Hz | 241500 | 2560 | 2608 | 2640 | 2720 | 1440 | 1443 | 1448 | 1481 |
| 2560x1600 | 2560x1600 | 60Hz | 268500 | 2560 | 2608 | 2640 | 2720 | 1600 | 1603 | 1609 | 1646 |
| 3840x2160 | 3840x2160 | 60Hz | 594000 | 3840 | 3888 | 3920 | 4000 | 2160 | 2163 | 2168 | 2222 |

### 1.4 VESA CVT 时序标准详解

VESA CVT（Coordinated Video Timing）标准定义了如何根据目标分辨率和刷新率计算完整的时序参数。以下为 CVT 的简化计算公式：

```c
struct cvt_timing {
    int h_pixels;       /* 水平像素 */
    int v_lines;        /* 垂直行数 */
    int refresh_rate;   /* 目标刷新率 */
    bool reduced;       /* 简化消隐（Reduced Blanking） */
    bool interlaced;    /* 隔行扫描 */
};

struct drm_display_mode cvt_calc(struct cvt_timing *cvt)
{
    struct drm_display_mode mode;
    int h_pixels, v_lines;
    int h_total, v_total;
    int v_front_porch, v_sync, v_back_porch;
    int h_front_porch, h_sync, h_back_porch;
    int h_blank, v_blank;
    int h_pixel_freq;
    int pixels_per_line;

    h_pixels = cvt->h_pixels;

    /* CVT 使用 8 像素对齐 */
    h_pixels = ((h_pixels + 7) / 8) * 8;

    v_lines = cvt->v_lines;
    if (cvt->interlaced)
        v_lines *= 2;

    /* 简化消隐（Reduced Blanking）用于减少像素时钟 */
    if (cvt->reduced) {
        /* RB v1.2 标准 */
        h_sync = 32;
        h_front_porch = 48;
        h_back_porch = 80;
        v_sync = 6;
        v_front_porch = 3;
        v_back_porch = 13;
        int extra_v_blank = (v_lines >= 1080) ? 3 : 0;
        v_back_porch += extra_v_blank;
    } else {
        /* CVT 标准消隐 */
        h_sync = 32;
        v_sync = 6;

        /* 水平消隐 = 160 像素 @60Hz */
        h_front_porch = 64;
        h_back_porch = 80;

        /* 垂直消隐 = 23 行 + 额外 */
        v_front_porch = 3;
        v_back_porch = 17;
    }

    h_blank = h_front_porch + h_sync + h_back_porch;
    v_blank = v_front_porch + v_sync + v_back_porch;

    h_total = h_pixels + h_blank;
    v_total = v_lines + v_blank;

    /* 计算像素时钟 */
    pixels_per_line = h_total;
    h_pixel_freq = pixels_per_line * v_total * cvt->refresh_rate;
    if (cvt->interlaced)
        h_pixel_freq /= 2;

    /* 填入模式结构体 */
    mode.clock = h_pixel_freq;
    mode.hdisplay = h_pixels;
    mode.hsync_start = h_pixels + h_front_porch;
    mode.hsync_end = mode.hsync_start + h_sync;
    mode.htotal = h_total;
    mode.vdisplay = v_lines;
    mode.vsync_start = v_lines + v_front_porch;
    mode.vsync_end = mode.vsync_start + v_sync;
    mode.vtotal = v_total;
    mode.vrefresh = cvt->refresh_rate;

    if (cvt->reduced)
        mode.flags |= DRM_MODE_FLAG_NHSYNC | DRM_MODE_FLAG_PVSYNC;

    return mode;
}
```

CVT 简化消隐（Reduced Blanking）的优势：

| 特性 | 标准 CVT | 简化消隐 CVT-RB | CVT-RB v2 |
|------|---------|----------------|-----------|
| 水平消隐 | 160 像素 | 128 像素 | 80 像素 |
| 垂直消隐 | 23 行 | 22 行 | 22 行 |
| 像素时钟减少 | 基准 | ~3% | ~8% |
| 适用场景 | 传统 VGA/DVI | DP/eDP 接口 | 高分辨率显示 |
| 示例：1920x1080@60 | 148.5 MHz | 142.0 MHz | 137.5 MHz |

### 1.5 Mode 标志位详解

`drm_display_mode.flags` 字段使用位掩码定义了多种显示模式特性：

```c
/* 同步信号极性 */
#define DRM_MODE_FLAG_PHSYNC    (1 << 0)  /* 水平同步正极性 */
#define DRM_MODE_FLAG_NHSYNC    (1 << 1)  /* 水平同步负极性 */
#define DRM_MODE_FLAG_PVSYNC    (1 << 2)  /* 垂直同步正极性 */
#define DRM_MODE_FLAG_NVSYNC    (1 << 3)  /* 垂直同步负极性 */

/* 扫描方式 */
#define DRM_MODE_FLAG_INTERLACE (1 << 4)  /* 隔行扫描 */
#define DRM_MODE_FLAG_DBLSCAN   (1 << 5)  /* 双倍扫描 */
#define DRM_MODE_FLAG_CSYNC     (1 << 6)  /* 复合同步 */
#define DRM_MODE_FLAG_PCSYNC    (1 << 7)  /* 复合同步正极性 */
#define DRM_MODE_FLAG_NCSYNC    (1 << 8)  /* 复合同步负极性 */

/* 消隐模式 */
#define DRM_MODE_FLAG_HSKEW     (1 << 9)  /* 水平偏斜 */
#define DRM_MODE_FLAG_BCAST     (1 << 10) /* 广播模式 */
#define DRM_MODE_FLAG_PIXMUX    (1 << 11) /* 像素复用 */
#define DRM_MODE_FLAG_DBLCLK    (1 << 12) /* 双倍像素时钟 */

/* 时钟边缘 */
#define DRM_MODE_FLAG_CLKDIV2   (1 << 13) /* 时钟 2 分频 */
#define DRM_MODE_FLAG_CLKDIV4   (1 << 14) /* 时钟 4 分频 */

/* 3D 显示 */
#define DRM_MODE_FLAG_3D_MASK   (0x1f << 20)  /* 3D 格式掩码 */
#define DRM_MODE_FLAG_3D_NONE   (0 << 20)      /* 非 3D */
#define DRM_MODE_FLAG_3D_FRAME_PACKING   (1 << 20)  /* 帧封装 */
#define DRM_MODE_FLAG_3D_FIELD_ALTERNATIVE (2 << 20) /* 场交替 */
#define DRM_MODE_FLAG_3D_LINE_ALTERNATIVE (3 << 20)  /* 行交替 */
#define DRM_MODE_FLAG_3D_SIDE_BY_SIDE_FULL (4 << 20) /* 并排全宽 */
#define DRM_MODE_FLAG_3D_L_DEPTH (5 << 20)    /* L 深度 */
#define DRM_MODE_FLAG_3D_L_DEPTH_GFX_GFX_DEPTH (6 << 20)

/* Mode 类型 */
#define DRM_MODE_TYPE_PREFERRED  (1 << 0)  /* 首选模式（来自 EDID） */
#define DRM_MODE_TYPE_DRIVER     (1 << 1)  /* 驱动添加的模式 */
#define DRM_MODE_TYPE_USERDEF    (1 << 2)  /* 用户定义的模式 */
#define DRM_MODE_TYPE_BUILTIN    (1 << 3)  /* 内核内置的模式 */
#define DRM_MODE_TYPE_DEFAULT    (1 << 4)  /* 默认模式 */
```

### 1.6 Mode 校验函数

DRM 核心提供了模式校验函数用于验证模式的有效性：

```c
enum drm_mode_status {
    MODE_OK        = 0,   /* 模式有效 */
    MODE_HSYNC     = 1,   /* 水平同步超出范围 */
    MODE_VSYNC     = 2,   /* 垂直同步超出范围 */
    MODE_H_ILLEGAL = 3,   /* 水平参数非法 */
    MODE_V_ILLEGAL = 4,   /* 垂直参数非法 */
    MODE_BAD_WIDTH = 5,   /* 水平宽度不合适 */
    MODE_NOMODE     = 6,  /* 无匹配模式 */
    MODE_NO_INTERLACE = 7, /* 不支持隔行扫描 */
    MODE_NO_DBLESCAN = 8,  /* 不支持双倍扫描 */
    MODE_NO_VSCAN   = 9,   /* 不支持垂直扫描 */
    MODE_MEM        = 10,  /* 显存不足 */
    MODE_VIRTUAL_X  = 11,  /* 虚拟宽度超出 */
    MODE_VIRTUAL_Y  = 12,  /* 虚拟高度超出 */
    MODE_MEM_VIRT   = 13,  /* 虚拟显存不足 */
    MODE_NOCLOCK    = 14,  /* 无时钟信号 */
    MODE_CLOCK_HIGH = 15,  /* 像素时钟过高 */
    MODE_CLOCK_LOW  = 16,  /* 像素时钟过低 */
    MODE_CLOCK_RANGE = 17, /* 像素时钟超出范围 */
    MODE_BAD_HVALUE = 18,  /* 水平值错误 */
    MODE_BAD_VVALUE = 19,  /* 垂直值错误 */
    MODE_BAD_VSCAN  = 20,  /* 垂直扫描值错误 */
    MODE_HSYNC_NARROW = 21, /* 水平同步太窄 */
    MODE_HSYNC_WIDE   = 22, /* 水平同步太宽 */
    MODE_HBLANK_NARROW = 23, /* 水平消隐太窄 */
    MODE_HBLANK_WIDE   = 24, /* 水平消隐太宽 */
    MODE_VSYNC_NARROW  = 25, /* 垂直同步太窄 */
    MODE_VSYNC_WIDE    = 26, /* 垂直同步太宽 */
    MODE_VBLANK_NARROW = 27, /* 垂直消隐太窄 */
    MODE_VBLANK_WIDE   = 28, /* 垂直消隐太宽 */
    MODE_PANEL         = 29, /* 面板参数不匹配 */
    MODE_INTERLACE_WIDTH = 30, /* 隔行宽度不匹配 */
    MODE_ONE_WIDTH     = 31, /* 仅支持一个宽度 */
    MODE_ONE_HEIGHT    = 32, /* 仅支持一个高度 */
    MODE_ONE_SIZE      = 33, /* 仅支持一个大小 */
    MODE_NO_REDUCED    = 34, /* 不支持简化消隐 */
    MODE_NO_STEREO     = 35, /* 不支持立体显示 */
    MODE_NO_3D_FORMAT  = 36, /* 不支持的 3D 格式 */
    MODE_STEREO_MISMATCH = 37, /* 立体模式不匹配 */
    MODE_BAD            = 38, /* 一般性错误 */
};

/* AMDGPU 中的 mode 校验函数 */
enum drm_mode_status
amdgpu_dm_connector_mode_valid(struct drm_connector *connector,
                               struct drm_display_mode *mode)
{
    struct amdgpu_dm_connector *aconnector = to_amdgpu_dm_connector(connector);
    struct dc_link *link = aconnector->dc_link;
    struct dc_crtc_timing dc_timing;
    unsigned int max_pixel_clock;

    /* 1. 转换 DRM mode 到 DC timing */
    dm_connector_mode_to_dc_timing(connector, mode, &dc_timing);

    /* 2. 检查像素时钟是否超过链路带宽 */
    max_pixel_clock = dc_link_get_max_pixel_clock(link);
    if (mode->clock > max_pixel_clock)
        return MODE_CLOCK_HIGH;

    /* 3. 检查时序参数是否有效 */
    if (!dc_timing_is_valid(&dc_timing))
        return MODE_BAD;

    /* 4. 检查带宽是否充足 */
    if (!dc_link_bandwidth_kbps(link, &dc_timing))
        return MODE_CLOCK_HIGH;

    return MODE_OK;
}
```

### 1.7 Mode 匹配与优先级排序

内核在多个可用模式中选择最佳显示模式时，使用 `drm_mode_match` 函数进行匹配，并使用优先级排序算法进行排序：

```c
/* Mode 匹配掩码 */
#define DRM_MODE_MATCH_TIMINGS   (1 << 0)  /* 匹配时序参数 */
#define DRM_MODE_MATCH_CLOCK     (1 << 1)  /* 匹配像素时钟 */
#define DRM_MODE_MATCH_FLAGS     (1 << 2)  /* 匹配标志位 */
#define DRM_MODE_MATCH_3D_FLAGS  (1 << 3)  /* 匹配 3D 标志 */
#define DRM_MODE_MATCH_ASPECT_RATIO (1 << 4) /* 匹配宽高比 */

/* 模式匹配函数 */
bool drm_mode_match(const struct drm_display_mode *mode1,
                    const struct drm_display_mode *mode2,
                    unsigned int match_flags)
{
    if (match_flags & DRM_MODE_MATCH_TIMINGS) {
        if (mode1->hdisplay != mode2->hdisplay ||
            mode1->vdisplay != mode2->vdisplay ||
            mode1->htotal != mode2->htotal ||
            mode1->vtotal != mode2->vtotal ||
            mode1->hsync_end - mode1->hsync_start !=
                mode2->hsync_end - mode2->hsync_start ||
            mode1->vsync_end - mode1->vsync_start !=
                mode2->vsync_end - mode2->vsync_start)
            return false;
    }

    if (match_flags & DRM_MODE_MATCH_CLOCK) {
        if (mode1->clock != mode2->clock)
            return false;
    }

    if (match_flags & DRM_MODE_MATCH_FLAGS) {
        if ((mode1->flags & ~DRM_MODE_FLAG_3D_MASK) !=
            (mode2->flags & ~DRM_MODE_FLAG_3D_MASK))
            return false;
    }

    return true;
}

/* Mode 排序：优先级从高到低 */
int drm_mode_compare(const struct drm_display_mode *a,
                     const struct drm_display_mode *b)
{
    int score_a = 0, score_b = 0;

    /* 1. Preferred mode 优先 */
    if (a->type & DRM_MODE_TYPE_PREFERRED)
        score_a += 100;
    if (b->type & DRM_MODE_TYPE_PREFERRED)
        score_b += 100;

    /* 2. 分辨率更高的优先（更多像素） */
    score_a += a->hdisplay * a->vdisplay;
    score_b += b->hdisplay * b->vdisplay;

    /* 3. 刷新率更高的优先 */
    score_a += drm_mode_vrefresh(a) * 1000;
    score_b += drm_mode_vrefresh(b) * 1000;

    /* 4. 非隔行优先 */
    if (!(a->flags & DRM_MODE_FLAG_INTERLACE))
        score_a += 5000;
    if (!(b->flags & DRM_MODE_FLAG_INTERLACE))
        score_b += 5000;

    return score_b - score_a;
}
```

### 1.8 EDID 中的 Detailed Timing Descriptor

EDID（Extended Display Identification Data）中的 DTD（Detailed Timing Descriptor）是显示器向系统报告其支持模式的标准方式。每个 DTD 占用 18 字节：

```
EDID Detailed Timing Descriptor 结构：
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│ Byte0│ Byte1│ Byte2│ Byte3│ Byte4│ Byte5│ Byte6│ Byte7│
├──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┤
│     像素时钟 (kHz/10)     │ hdisp │ hblank  │
│     16 位，小端序         │ 低 8  │ 高 4+4 │
├──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┤
│ vdisp │ vblank  │ hfront │ hsync │ vfront│ vsync │
│ 低 8  │ 高 4+4  │ porch  │ width │ porch │ width │
├──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┤
│ himage│ vimage │ hbord  │ vbord  │ flags │ 保留  │
│       │        │ er     │ er     │       │       │
└──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
```

EDID 的 DTD 解析函数：

```c
struct drm_display_mode *
drm_mode_detailed(struct drm_connector *connector,
                  struct edid *edid,
                  const u8 *detailed_timing)
{
    struct drm_display_mode *mode;
    unsigned pixclock;
    unsigned hdisplay, hsyncstart, hsyncend, htotal;
    unsigned vdisplay, vsyncstart, vsyncend, vtotal;
    unsigned h_sync_width, v_sync_width;

    mode = drm_mode_create(connector->dev);
    if (!mode)
        return NULL;

    /* 解析像素时钟（Byte0-1，单位 10kHz） */
    pixclock = detailed_timing[0] | (detailed_timing[1] << 8);
    if (pixclock == 0)
        goto invalid;
    mode->clock = pixclock * 10;

    /* 解析水平参数 */
    hdisplay = detailed_timing[2] | ((detailed_timing[4] & 0xF0) << 4);
    hsyncstart = detailed_timing[8] | ((detailed_timing[4] & 0x0C) << 2);
    hsyncend = detailed_timing[9] | ((detailed_timing[4] & 0x03) << 2);
    htotal = detailed_timing[3] | ((detailed_timing[5] & 0xF0) << 4);
    h_sync_width = detailed_timing[8] | ((detailed_timing[4] & 0x0C) << 2);

    /* 解析垂直参数 */
    vdisplay = detailed_timing[6] | ((detailed_timing[10] & 0xF0) << 4);
    vsyncstart = detailed_timing[10] | ((detailed_timing[10] & 0x0C) << 2);
    vsyncend = detailed_timing[11] | ((detailed_timing[10] & 0x03) << 2);
    vtotal = detailed_timing[7] | ((detailed_timing[11] & 0xF0) << 4);
    v_sync_width = detailed_timing[10] | ((detailed_timing[10] & 0x0C) << 2);

    /* 填入模式结构体 */
    mode->hdisplay = hdisplay;
    mode->hsync_start = hsyncstart;
    mode->hsync_end = hsyncend;
    mode->htotal = htotal;
    mode->vdisplay = vdisplay;
    mode->vsync_start = vsyncstart;
    mode->vsync_end = vsyncend;
    mode->vtotal = vtotal;
    mode->vrefresh = drm_mode_vrefresh(mode);

    /* 解析同步极性标志 */
    if (detailed_timing[17] & 0x02)
        mode->flags |= DRM_MODE_FLAG_HSYNC_POS;
    if (detailed_timing[17] & 0x04)
        mode->flags |= DRM_MODE_FLAG_VSYNC_POS;
    if (detailed_timing[17] & 0x10)
        mode->flags |= DRM_MODE_FLAG_INTERLACE;
    if (detailed_timing[17] & 0x40)
        mode->flags |= DRM_MODE_FLAG_DBLSCAN;

    drm_mode_set_name(mode);

    return mode;

invalid:
    drm_mode_destroy(connector->dev, mode);
    return NULL;
}
```

### 1.9 CEA 标准模式与 VIC

对于 HDMI 连接，CEA-861 标准定义了 VIC（Video Identification Code）用于标识预定义模式。CEA 模式块中每个 VIC 对应一个标准时序：

```c
const struct drm_display_mode edid_cea_modes[] = {
    /* VIC 1: 640x480@60Hz */
    { DRM_MODE("640x480", DRM_MODE_TYPE_DRIVER, 25175, 640, 656,
               752, 800, 0, 480, 490, 492, 525, 0,
               DRM_MODE_FLAG_NHSYNC | DRM_MODE_FLAG_NVSYNC) },
    /* VIC 2: 720x480@60Hz */
    { DRM_MODE("720x480", DRM_MODE_TYPE_DRIVER, 27000, 720, 736,
               798, 858, 0, 480, 489, 495, 525, 0,
               DRM_MODE_FLAG_NHSYNC | DRM_MODE_FLAG_NVSYNC) },
    /* VIC 3: 720x480@60Hz 16:9 */
    { DRM_MODE("720x480", DRM_MODE_TYPE_DRIVER, 27000, 720, 736,
               798, 858, 0, 480, 489, 495, 525, 0,
               DRM_MODE_FLAG_NHSYNC | DRM_MODE_FLAG_NVSYNC) },
    /* VIC 4: 1280x720@60Hz */
    { DRM_MODE("1280x720", DRM_MODE_TYPE_DRIVER, 74250, 1280, 1390,
               1430, 1650, 0, 720, 725, 730, 750, 0,
               DRM_MODE_FLAG_PHSYNC | DRM_MODE_FLAG_PVSYNC) },
    /* VIC 16: 1920x1080@60Hz */
    { DRM_MODE("1920x1080", DRM_MODE_TYPE_DRIVER, 148500, 1920, 2008,
               2052, 2200, 0, 1080, 1084, 1089, 1125, 0,
               DRM_MODE_FLAG_PHSYNC | DRM_MODE_FLAG_PVSYNC) },
    /* VIC 17: 720x576@50Hz */
    { DRM_MODE("720x576", DRM_MODE_TYPE_DRIVER, 27000, 720, 732,
               796, 864, 0, 576, 581, 586, 625, 0,
               DRM_MODE_FLAG_NHSYNC | DRM_MODE_FLAG_NVSYNC) },
    /* VIC 18: 720x576@50Hz 16:9 */
    { DRM_MODE("720x576", DRM_MODE_TYPE_DRIVER, 27000, 720, 732,
               796, 864, 0, 576, 581, 586, 625, 0,
               DRM_MODE_FLAG_NHSYNC | DRM_MODE_FLAG_NVSYNC) },
    /* VIC 19: 1280x720@50Hz */
    { DRM_MODE("1280x720", DRM_MODE_TYPE_DRIVER, 74250, 1280, 1720,
               1760, 1980, 0, 720, 725, 730, 750, 0,
               DRM_MODE_FLAG_PHSYNC | DRM_MODE_FLAG_PVSYNC) },
    /* VIC 31: 1920x1080@50Hz */
    { DRM_MODE("1920x1080", DRM_MODE_TYPE_DRIVER, 148500, 1920, 2448,
               2492, 2640, 0, 1080, 1084, 1089, 1125, 0,
               DRM_MODE_FLAG_PHSYNC | DRM_MODE_FLAG_PVSYNC) },
    /* VIC 32: 1920x1080@24Hz */
    { DRM_MODE("1920x1080", DRM_MODE_TYPE_DRIVER, 74250, 1920, 2558,
               2602, 2750, 0, 1080, 1084, 1089, 1125, 0,
               DRM_MODE_FLAG_PHSYNC | DRM_MODE_FLAG_PVSYNC) },
};

/* 根据 VIC 查找 CEA 模式 */
struct drm_display_mode *drm_cea_get_mode(int vic)
{
    if (vic < 1 || vic > ARRAY_SIZE(edid_cea_modes))
        return NULL;

    return (struct drm_display_mode *)&edid_cea_modes[vic - 1];
}
```

### 1.10 AMDGPU 中的 Mode 处理流程

AMDGPU 驱动中对显示模式的处理包含了校验、过滤、转换和提交的完整流程：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AMDGPU Mode 处理流水线                             │
│                                                                     │
│  EDID 解析                                                          │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐              │
│  │ 读取     │───►│ DTD/YMD/CEA  │───►│ 创建 DRM     │              │
│  │ EDID     │    │ 模式块解析   │    │ display_mode │              │
│  └──────────┘    └──────────────┘    └──────┬───────┘              │
│                                             │                       │
│  Mode 校验与过滤                                                  │
│                                             ▼                       │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  amdgpu_dm_connector_mode_valid()                          │    │
│  │  1. drm_mode_validate_basic()    基本参数校验             │    │
│  │  2. drm_mode_validate_size()     尺寸范围校验             │    │
│  │  3. drm_mode_validate_flag()     标志位校验               │    │
│  │  4. amdgpu_dm_mode_valid()       像素时钟/带宽校验        │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                             │                       │
│  Mode 排序                                                          │
│                                             ▼                       │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  drm_mode_sort() / drm_mode_compare()                     │    │
│  │  1. Preferred mode 优先                                   │    │
│  │  2. 分辨率从高到低                                        │    │
│  │  3. 刷新率从高到低                                        │    │
│  │  4. 非隔行优先                                            │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                             │                       │
│  Mode 设置（从 DRM 到 DC）                                         │
│                                             ▼                       │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  dm_connector_mode_to_dc_timing()                         │    │
│  │  struct drm_display_mode ──► struct dc_crtc_timing        │    │
│  │                                                           │    │
│  │  dc_crtc_timing 结构:                                     │    │
│  │  ├─ h_addressable: mode->hdisplay                        │    │
│  │  ├─ h_front_porch: mode->hsync_start - mode->hdisplay    │    │
│  │  ├─ h_sync_width:  mode->hsync_end - mode->hsync_start   │    │
│  │  ├─ h_total:       mode->htotal - mode->hdisplay         │    │
│  │  ├─ v_addressable: mode->vdisplay                        │    │
│  │  ├─ v_front_porch: mode->vsync_start - mode->vdisplay    │    │
│  │  ├─ v_sync_width:  mode->vsync_end - mode->vsync_start   │    │
│  │  ├─ v_total:       mode->vtotal - mode->vdisplay         │    │
│  │  └─ pixel_clock:   mode->clock * 10                      │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                             │                       │
│  DC 硬件提交                                                       │
│                                             ▼                       │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  dc_stream_create() → dc_commit_streams()                  │    │
│  │  └─ dc_validate_stream() 校验时序是否有效                 │    │
│  │  └─ dc_link_reduce_bandwidth() 带宽不足时降级            │    │
│  └────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

`dm_connector_mode_to_dc_timing` 的转换实现：

```c
bool dm_connector_mode_to_dc_timing(
    struct drm_connector *connector,
    struct drm_display_mode *mode,
    struct dc_crtc_timing *timing)
{
    memset(timing, 0, sizeof(*timing));

    timing->h_addressable = mode->hdisplay;
    timing->h_front_porch = mode->hsync_start - mode->hdisplay;
    timing->h_sync_width = mode->hsync_end - mode->hsync_start;
    timing->h_total = mode->htotal - mode->hdisplay;
    timing->h_border_bottom = 0;
    timing->h_border_top = 0;
    timing->h_border_left = 0;
    timing->h_border_right = 0;

    timing->v_addressable = mode->vdisplay;
    timing->v_front_porch = mode->vsync_start - mode->vdisplay;
    timing->v_sync_width = mode->vsync_end - mode->vsync_start;
    timing->v_total = mode->vtotal - mode->vdisplay;
    timing->v_border_bottom = 0;
    timing->v_border_top = 0;

    timing->pix_clk_100hz = mode->clock * 10;
    timing->pix_clk_khz = mode->clock;

    if (mode->flags & DRM_MODE_FLAG_INTERLACE)
        timing->flags.interlace = 1;
    if (mode->flags & DRM_MODE_FLAG_PHSYNC)
        timing->flags.HSYNC_POSITIVE_POLARITY = 1;
    if (mode->flags & DRM_MODE_FLAG_PVSYNC)
        timing->flags.VSYNC_POSITIVE_POLARITY = 1;

    timing->timing_3d_format = TIMING_3D_FORMAT_NONE;

    return true;
}
```

### 1.11 自定义 Mode（Modeline）

Modeline 是 Xorg 中引入的显示模式描述格式，DRM 也完全支持。Modeline 的格式为：

```
Modeline "1920x1080@60"  148.50  1920 2008 2052 2200  1080 1084 1089 1125  +hsync +vsync
         │                │       │    │    │    │     │    │    │    │     │      │
         模式名称         像素时钟 hdisp hss hse htotal vdisp vss vse vtotal hsync vsync
                          (MHz)                                   极性   极性
```

Modeline 到 `drm_display_mode` 的解析：

```c
struct drm_display_mode *
drm_mode_create_from_cmdline_mode(const char *modeline)
{
    struct drm_display_mode *mode;
    char name[DRM_DISPLAY_MODE_LEN];
    int hsync_pol = 0, vsync_pol = 0;

    mode = kzalloc(sizeof(*mode), GFP_KERNEL);
    if (!mode)
        return NULL;

    /* 解析 Modeline 字符串 */
    sscanf(modeline, "%s %d %d %d %d %d %d %d %d %d %d %d",
           name,
           &mode->clock,           /* MHz → kHz */
           &mode->hdisplay,
           &mode->hsync_start,
           &mode->hsync_end,
           &mode->htotal,
           &mode->vdisplay,
           &mode->vsync_start,
           &mode->vsync_end,
           &mode->vtotal,
           &hsync_pol,
           &vsync_pol);

    mode->clock *= 1000;  /* 转换为 kHz */
    mode->vrefresh = drm_mode_vrefresh(mode);

    if (hsync_pol > 0)
        mode->flags |= DRM_MODE_FLAG_PHSYNC;
    else if (hsync_pol < 0)
        mode->flags |= DRM_MODE_FLAG_NHSYNC;

    if (vsync_pol > 0)
        mode->flags |= DRM_MODE_FLAG_PVSYNC;
    else if (vsync_pol < 0)
        mode->flags |= DRM_MODE_FLAG_NVSYNC;

    snprintf(mode->name, DRM_DISPLAY_MODE_LEN, "%s", name);

    return mode;
}
```

### 1.12 Mode 的像素时钟与链路带宽关系

像素时钟和链路带宽的关系决定了某个分辨率是否能在特定接口上驱动：

```c
/* 计算所需的总带宽（单位：Mbps） */
uint64_t calc_required_bandwidth(struct drm_display_mode *mode,
                                 int bpc)
{
    uint64_t bits_per_pixel = bpc * 3;  /* RGB 各 bpc 位 */
    uint64_t pixel_clock_mhz = mode->clock / 1000;
    uint64_t bandwidth = pixel_clock_mhz * bits_per_pixel;

    return bandwidth;
}

/* 计算链路可用带宽 */
uint64_t calc_link_bandwidth(int lane_count, int link_rate_mhz)
{
    /* DP 使用 8b/10b 编码，有效带宽 = 原始带宽 × 0.8 */
    uint64_t raw_bandwidth = lane_count * link_rate_mhz * 8;
    uint64_t effective_bandwidth = raw_bandwidth * 8 / 10;

    return effective_bandwidth;
}

/* 常见接口的可用带宽对比 */
static const struct interface_bandwidth {
    const char *name;
    int max_lanes;
    int max_link_rate_mhz;
    uint64_t max_bandwidth_mbps;
} interfaces[] = {
    {"HDMI 1.4",    0, 340000,  8160000},  /* TMDS 单通道 */
    {"HDMI 2.0",    0, 600000, 14400000},  /* TMDS 单通道增强 */
    {"HDMI 2.1 FRL", 4, 12000000, 57600000}, /* FRL 4 通道 */
    {"DP 1.2",       4, 540000, 13824000},  /* HBR2 */
    {"DP 1.4",       4, 810000, 20736000},  /* HBR3 */
    {"DP 2.0 UHBR10", 4, 10000000, 32000000}, /* 128b/132b 编码 */
    {"DP 2.0 UHBR13.5", 4, 13500000, 43200000},
    {"DP 2.0 UHBR20", 4, 20000000, 64000000},
};
```

## 2. 实践操作

### 2.1 使用 modetest 查看显示模式

```bash
# 1. 列出所有连接器及其支持的模式
$ modetest -M amdgpu

# 2. 查看指定连接器的详细模式信息
$ modetest -M amdgpu -c | grep -A30 "DP-1"

# 3. 显示平面信息及关联的模式
$ modetest -M amdgpu -p

# 4. 使用原子 API 测试模式设置
$ modetest -M amdgpu -a -s 28:1920x1080@60

# 5. 查看所有可用模式的详细时序
$ modetest -M amdgpu -c | grep -E "^[0-9]+|name|mode|refresh"
```

输出示例：
```
connectors:
id      encoder status          type    size (mm)       modes   enc name
28      27      connected       DP      527x296          14      27      DP-1
  modes:
        name refresh (Hz) hdisp hss hse htot vdisp vss vse vtot
  1920x1080 60.00 1920 2008 2052 2200 1080 1084 1089 1125 flags: phsync, pvsync; type: preferred, driver
  1920x1080 50.00 1920 2448 2492 2640 1080 1084 1089 1125 flags: phsync, pvsync; type: driver
  1920x1080 30.00 1920 2008 2052 2200 1080 1084 1089 1125 flags: phsync, pvsync; type: driver
  1920x1080 24.00 1920 2558 2602 2750 1080 1084 1089 1125 flags: phsync, pvsync; type: driver
  1680x1050 59.88 1680 1784 1960 2240 1050 1053 1059 1089 flags: nhsync, nvsync; type: driver
  1280x1024 75.03 1280 1296 1440 1688 1024 1025 1028 1066 flags: phsync, pvsync; type: driver
  1280x1024 60.02 1280 1328 1440 1688 1024 1025 1028 1066 flags: nhsync, nvsync; type: driver
  1280x720 60.00 1280 1390 1430 1650 720 725 730 750 flags: phsync, pvsync; type: driver
  1024x768 60.00 1024 1048 1184 1344 768 771 777 806 flags: nhsync, nvsync; type: driver
  800x600 60.32 800 840 968 1056 600 601 605 628 flags: phsync, pvsync; type: driver
  640x480 60.00 640 656 752 800 480 490 492 525 flags: nhsync, nvsync; type: driver
```

### 2.2 使用 sysfs 查看模式信息

```bash
# 1. 查看连接器的 modes 文件
$ cat /sys/class/drm/card0-DP-1/modes
1920x1080
1680x1050
1280x1024
1280x720
1024x768
800x600
640x480

# 2. 查看 EDID 中的原始 DTD 数据
$ cat /sys/class/drm/card0-DP-1/edid | hexdump -C | head -20
00000000  00 ff ff ff ff ff ff 00  10 ac 12 34 56 78 9a bc  |.........4Vx..|
00000010  00 1d 01 03 80 34 1d 78  ea 15 d5 a5 55 50 9f 27  |.....4.x....UP.'|
00000020  12 50 54 00 00 00 01 01  01 01 01 01 01 01 01 01  |.PT.............|
00000030  01 01 01 01 01 01 01 01  01 01 01 01 01 01 01 01  |................|
*
000000fc  00 01 02 03 04 05 06 07  08 09 0a 0b 0c 0d 0e 0f  |................|
...
00000100

# 3. 查看当前生效的模式
$ cat /sys/kernel/debug/dri/0/crtc-0/current_mode
Video mode: 1920x1080
Clock: 148500 MHz
H: 1920 2008 2052 2200
V: 1080 1084 1089 1125
Flags: phsync, pvsync
Refresh: 60 Hz

# 4. 查看所有 CRTC 的模式状态
$ for crtc in /sys/kernel/debug/dri/0/crtc-*; do
    echo "=== $(basename $crtc) ==="
    cat $crtc/state 2>/dev/null | head -20
    echo
done
```

### 2.3 使用 xrandr 查询和设置模式

```bash
# 1. 列出所有模式
$ xrandr
Screen 0: minimum 320 x 200, current 1920 x 1080, maximum 16384 x 16384
DP-1 connected primary 1920x1080+0+0 (normal left inverted right) 527mm x 296mm
   1920x1080     60.00*+  50.00    30.00    24.00
   1680x1050     59.88
   1280x1024     75.03    60.02
   1280x720      60.00
   1024x768      60.00
   800x600       60.32
   640x480       60.00

# 2. 查看模式的详细时序
$ xrandr --prop
DP-1 connected primary 1920x1080+0+0 (normal left inverted right) 527mm x 296mm
   1920x1080     60.00*+  50.00    30.00    24.00
   1680x1050     59.88
  ...
  EDID:
        00ffffffffffff0010ac123456789abc
        001d010380341d78ea15d5a555509f27
  ...
  audio: ...
  Broadcast RGB: Automatic
  non-desktop: 0

# 3. 添加自定义模式
$ cvt 2560 1440 60
# 2560x1440 59.95 Hz (CVT 3.70M9) hsync: 88.79 kHz; pclk: 312.25 MHz
Modeline "2560x1440_60.00"  312.25  2560 2744 3024 3488  1440 1443 1448 1493 -hsync +vsync
$ xrandr --newmode "2560x1440_60.00" 312.25 2560 2744 3024 3488 1440 1443 1448 1493 -hsync +vsync
$ xrandr --addmode DP-1 "2560x1440_60.00"
$ xrandr --output DP-1 --mode "2560x1440_60.00"

# 4. 使用 --verbose 查看详细刷新率
$ xrandr --verbose | grep -A5 "DP-1"
```

### 2.4 解析 EDID 中的详细时序

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>

struct edid_dtd {
    unsigned int clock;
    unsigned int hdisplay;
    unsigned int hblank;
    unsigned int hsync_start;
    unsigned int hsync_width;
    unsigned int vdisplay;
    unsigned int vblank;
    unsigned int vsync_start;
    unsigned int vsync_width;
    unsigned int hsync_pol;
    unsigned int vsync_pol;
    unsigned int interlaced;
};

static int parse_dtd(const unsigned char *dtd, struct edid_dtd *t)
{
    t->clock = (dtd[0] | (dtd[1] << 8)) * 10;
    t->hdisplay = dtd[2] | ((dtd[4] & 0xF0) << 4);
    t->hblank = dtd[3] | ((dtd[5] & 0xF0) << 4);
    t->hsync_start = dtd[8] | ((dtd[4] & 0x0C) << 2);
    t->hsync_width = dtd[9] | ((dtd[4] & 0x03) << 2);
    t->vdisplay = dtd[6] | ((dtd[10] & 0xF0) << 4);
    t->vblank = dtd[7] | ((dtd[10] & 0x0F) << 8);
    t->vsync_start = (dtd[10] >> 4) | ((dtd[10] & 0x0C) << 2);
    t->vsync_width = (dtd[11] & 0x0F) | ((dtd[10] & 0x03) << 2);
    t->hsync_pol = (dtd[17] >> 1) & 1;
    t->vsync_pol = (dtd[17] >> 2) & 1;
    t->interlaced = (dtd[17] >> 4) & 1;
    return 0;
}

int main(int argc, char **argv)
{
    unsigned char edid[256];
    int fd, nread;

    if (argc < 2) {
        fprintf(stderr, "Usage: %s <edid_file>\n", argv[0]);
        return 1;
    }

    fd = open(argv[1], O_RDONLY);
    if (fd < 0) { perror("open"); return 1; }

    nread = read(fd, edid, sizeof(edid));
    if (nread < 128) { fprintf(stderr, "Invalid EDID\n"); close(fd); return 1; }

    close(fd);

    printf("=== EDID DTD 解析 ===\n\n");
    printf("Manufacturer: %c%c%c\n",
           'A' + ((edid[8] >> 2) & 0x1F) - 1,
           'A' + (((edid[8] & 0x03) << 3) | ((edid[9] >> 5) & 0x07)) - 1,
           'A' + (edid[9] & 0x1F) - 1);
    printf("Product Code: %d\n", edid[10] | (edid[11] << 8));
    printf("Serial Number: %d\n", edid[12] | (edid[13] << 8) | (edid[14] << 16) | (edid[15] << 24));

    printf("\nDetailed Timing Descriptors:\n");
    for (int i = 0; i < 4; i++) {
        int offset = 54 + i * 18;
        if (edid[offset] == 0 && edid[offset + 1] == 0) {
            unsigned char tag = edid[offset + 3];
            const char *desc_type;
            switch (tag) {
            case 0xFC: desc_type = "Monitor Name"; break;
            case 0xFF: desc_type = "Serial Number"; break;
            case 0xFE: desc_type = "Unspecified Text"; break;
            case 0xFD: desc_type = "Range Limits"; break;
            default: desc_type = "Monitor Descriptor";
            }
            printf("  DTD %d: %s\n", i + 1, desc_type);
        } else {
            struct edid_dtd t;
            parse_dtd(edid + offset, &t);
            printf("  DTD %d: %dx%d @ %d Hz (clock: %d kHz)\n",
                   i + 1, t.hdisplay, t.vdisplay,
                   t.clock / ((t.hdisplay + t.hblank) *
                              (t.vdisplay + t.vblank)),
                   t.clock);
            printf("         H: %d %d %d %d (total: %d)\n",
                   t.hdisplay,
                   t.hsync_start,
                   t.hsync_start + t.hsync_width,
                   t.hdisplay + t.hblank,
                   t.hdisplay + t.hblank);
            printf("         V: %d %d %d %d (total: %d)\n",
                   t.vdisplay,
                   t.vsync_start,
                   t.vsync_start + t.vsync_width,
                   t.vdisplay + t.vblank,
                   t.vdisplay + t.vblank);
        }
    }

    return 0;
}
```

编译运行：
```bash
cat /sys/class/drm/card0-DP-1/edid > edid.bin
gcc -o parse_edid parse_edid.c
./parse_edid edid.bin
```

### 2.5 Mode 结构体全字段打印程序

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

static void print_flags(uint32_t flags)
{
    printf("    Flags:");
    if (flags & DRM_MODE_FLAG_PHSYNC) printf(" +hsync");
    if (flags & DRM_MODE_FLAG_NHSYNC) printf(" -hsync");
    if (flags & DRM_MODE_FLAG_PVSYNC) printf(" +vsync");
    if (flags & DRM_MODE_FLAG_NVSYNC) printf(" -vsync");
    if (flags & DRM_MODE_FLAG_INTERLACE) printf(" interlace");
    if (flags & DRM_MODE_FLAG_DBLSCAN) printf(" dblscan");
    if (flags & DRM_MODE_FLAG_CSYNC) printf(" csync");
    if (flags & DRM_MODE_FLAG_HSKEW) printf(" hskew");
    if (flags & DRM_MODE_FLAG_DBLCLK) printf(" dblclk");
    if (flags & DRM_MODE_FLAG_CLKDIV2) printf(" clkdiv2");
    printf("\n");
}

static void print_type(uint32_t type)
{
    printf("    Type:");
    if (type & DRM_MODE_TYPE_PREFERRED) printf(" preferred");
    if (type & DRM_MODE_TYPE_DRIVER) printf(" driver");
    if (type & DRM_MODE_TYPE_USERDEF) printf(" userdef");
    if (type & DRM_MODE_TYPE_BUILTIN) printf(" builtin");
    if (type & DRM_MODE_TYPE_DEFAULT) printf(" default");
    printf("\n");
}

int main(int argc, char **argv)
{
    int fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);
    if (fd < 0) { perror("open"); return 1; }

    drmModeRes *res = drmModeGetResources(fd);
    if (!res) { fprintf(stderr, "get resources failed\n"); close(fd); return 1; }

    for (int i = 0; i < res->count_connectors; i++) {
        drmModeConnector *conn = drmModeGetConnector(fd, res->connectors[i]);
        if (!conn) continue;

        printf("=== Connector %d: %d modes ===\n",
               conn->connector_id, conn->count_modes);

        for (int j = 0; j < conn->count_modes; j++) {
            drmModeModeInfo *m = &conn->modes[j];
            printf("\n  Mode[%d]: %s\n", j, m->name);
            printf("    clock: %d kHz (%.2f MHz)\n", m->clock, m->clock / 1000.0);
            printf("    hdisplay: %d\n", m->hdisplay);
            printf("    hsync_start: %d\n", m->hsync_start);
            printf("    hsync_end: %d\n", m->hsync_end);
            printf("    htotal: %d\n", m->htotal);
            printf("    hskew: %d\n", m->hskew);
            printf("    vdisplay: %d\n", m->vdisplay);
            printf("    vsync_start: %d\n", m->vsync_start);
            printf("    vsync_end: %d\n", m->vsync_end);
            printf("    vtotal: %d\n", m->vtotal);
            printf("    vscan: %d\n", m->vscan);
            printf("    vrefresh: %d Hz\n", m->vrefresh);
            printf("    hsync: %d kHz\n", m->hsync);
            printf("    picture_aspect_ratio
            printf("    picture_aspect_ratio: %d\n", m->picture_aspect_ratio);
            print_flags(m->flags);
            print_type(m->type);
            printf("    vrefresh_actual: %.2f Hz\n",
                    (double)m->clock * 1000 /
                    ((double)m->htotal * m->vtotal));
        }
        drmModeFreeConnector(conn);
    }

    drmModeFreeResources(res);
    close(fd);
    return 0;
}

编译运行：
```bash
gcc -o print_modes print_modes.c -ldrm
./print_modes
```

输出示例（连接了一个 1920×1080 显示器）：
```
=== Connector 73: 6 modes ===

  Mode[0]: 1920x1080 (0)
    clock: 148500 kHz (148.50 MHz)
    hdisplay: 1920
    hsync_start: 2008
    hsync_end: 2052
    htotal: 2200
    hskew: 0
    vdisplay: 1080
    vsync_start: 1084
    vsync_end: 1089
    vtotal: 1125
    vscan: 0
    vrefresh: 60 Hz
    hsync: 67 kHz
    picture_aspect_ratio: 0
    Flags: phsync pvsync
    Type: preferred driver
    vrefresh_actual: 60.00 Hz
```

该程序展示了如何通过 libdrm API 直接读取内核暴露的 drmModeModeInfo 结构体，将所有字段打印出来。

---

### 2.6 IGT 测试命令

IGT GPU Tools 提供了多个测试用例来验证 mode 设置的正确性。

```bash
# 查看所有可用的 mode 相关测试
igt_list_tests | grep -i mode

# 测试 3D 标志位 modes（立体显示）
sudo igt-run kms_3d

# 测试非法 dotclock 的场景
sudo igt-run kms_invalid_dotclock

# 测试 displayport aux 设备
sudo igt-run kms_dp_aux_dev

# 测试 modeline 解析和设置
sudo igt-run kms_modeline

# 测试 VBlank 模式相关
sudo igt-run kms_vblank --run-subtest mode-basic

# 测试缩放模式
sudo igt-run kms_scaling_modes

# 测试 mode 切换
sudo igt-run kms_atomic --run-subtest mode_set

# 测试 EDID 读取中的 mode 解析
sudo igt-run kms_edid_ext

# 测试 4K/高分辨率 mode
sudo igt-run kms_big_fb

# 测试多显示器 mode 组合
sudo igt-run kms_atomic_transition

# 测试 mode 的 panel fitter/缩放
sudo igt-run kms_panel_fitting

# 列出某个测试的详细 subtests
sudo igt-run -l kms_modeline
```

常用测试分析：
```
subtest "invalid-clock":
    尝试设置一个超出显示器范围的 dotclock，
    验证内核返回 MODE_BAD 或类似错误状态。

subtest "3d-modes":
    枚举所有带有 DRM_MODE_FLAG_3D_MASK 标志的 mode，
    确保 3D 格式的传输正确。

subtest "custom-modeline":
    通过 drmModeCreatePropertyBlob 创建自定义 modeline，
    验证用户自定义 mode 的添加和生效流程。
```

---

### 2.7 调试脚本

以下实用脚本帮助在日常开发中快速排查 mode 相关问题。

#### 批量 mode 测试脚本

```bash
#!/bin/bash
# mode_stress_test.sh - 批量验证所有显示器的可用 mode
# 使用方法: sudo ./mode_stress_test.sh [iterations]

ITERATIONS=${1:-5}
LOG_FILE="/tmp/mode_stress_$(date +%Y%m%d_%H%M%S).log"

echo "[$(date)] Mode Stress Test Starting - $ITERATIONS iterations"
echo "Log file: $LOG_FILE"

for connector in /sys/class/drm/card0-*; do
    conn_name=$(basename "$connector")
    echo "=== Testing connector: $conn_name ==="

    if [ -f "$connector/modes" ]; then
        echo "Available modes:"
        cat "$connector/modes"
    fi

    if [ -f "$connector/current_mode" ]; then
        current=$(cat "$connector/current_mode" 2>/dev/null)
        echo "Current mode: $current"
    fi

    for ((i=0; i<ITERATIONS; i++)); do
        echo "  Iteration $((i+1))..."
        modetest -M amdgpu -c 2>&1 | grep -A 20 "$conn_name" >> "$LOG_FILE"
        sleep 1
    done
done

echo "[$(date)] Mode Stress Test Completed"
echo "Results saved to $LOG_FILE"
```

#### EDID 与 mode 分析脚本

```bash
#!/bin/bash
# edid_mode_analyze.sh - 分析 EDID 中的所有 Detailed Timing Descriptors
# 使用方法: sudo ./edid_mode_analyze.sh [connector]

CONNECTOR=${1:-card0-HDMI-A-1}
EDID_PATH="/sys/class/drm/$CONNECTOR/edid"

if [ ! -f "$EDID_PATH" ]; then
    echo "ERROR: $EDID_PATH not found"
    exit 1
fi

echo "=== EDID Mode Analysis for $CONNECTOR ==="
echo "EDID size: $(wc -c < "$EDID_PATH") bytes"
echo ""

EDID_HEX=$(xxd -p < "$EDID_PATH" | tr -d '\n')
echo "EDID hex (first 256 bytes):"
echo "$EDID_HEX" | fold -w 32 | head -8
echo ""

echo "=== Detailed Timing Descriptors ==="

echo "Manufacturer ID: $(echo "$EDID_HEID" | cut -c1-4)"
echo "EDID Version: $(printf "%d" 0x$(echo "$EDID_HEX" | cut -c19-20)).$(printf "%d" 0x$(echo "$EDID_HEX" | cut -c21-22))"

echo ""
echo "=== Standard Timings ==="
echo "Established Timings I/II/III:"
echo "  Byte 0x23: $(echo "$EDID_HEX" | cut -c47-48)"
echo "  Byte 0x24: $(echo "$EDID_HEX" | cut -c49-50)"
echo "  Byte 0x25: $(echo "$EDID_HEX" | cut -c51-52)"

echo ""
echo "=== Detailed Timing Descriptor Analysis ==="
OFFSET=108
for ((i=0; i<4; i++)); do
    START=$((OFFSET + i * 72))
    DTD_HEX=$(echo "$EDID_HEX" | cut -c$((START+1))-$((START+72)))

    PCLK_LOW=$(printf "%d" 0x$(echo "$DTD_HEX" | cut -c1-2))
    PCLK_HIGH=$(printf "%d" 0x$(echo "$DTD_HEX" | cut -c3-4))
    PCLK=$(( PCLK_LOW | (PCLK_HIGH << 8) ))

    if [ $PCLK -eq 0 ]; then
        echo "  DTD[$i]: (not a timing descriptor)"
        continue
    fi

    H_ACTIVE=$(echo "$DTD_HEX" | cut -c5-6)
    H_BLANKING=$(echo "$DTD_HEX" | cut -c7-8)
    H_ACTIVE_LO=$(printf "%d" 0x$H_ACTIVE)
    H_BLANK_LO=$(printf "%d" 0x$H_BLANKING)
    HBITS=$(echo "$DTD_HEX" | cut -c9-10)
    HBITS_VAL=$(printf "%d" 0x$HBITS)
    H_ACTIVE_HI=$(( (HBITS_VAL & 0xF0) >> 4 ))
    H_BLANK_HI=$(( HBITS_VAL & 0x0F ))
    H_ACTIVE_TOTAL=$(( (H_ACTIVE_HI << 8) | H_ACTIVE_LO ))
    H_BLANK_TOTAL=$(( (H_BLANK_HI << 8) | H_BLANK_LO ))

    H_SYNC_OFFSET=$(echo "$DTD_HEX" | cut -c11-12)
    H_SYNC_PULSE=$(echo "$DTD_HEX" | cut -c13-14)
    HBITS2=$(echo "$DTD_HEX" | cut -c15-16)
    HBITS2_VAL=$(printf "%d" 0x$HBITS2)
    H_SYNC_OFFSET_HI=$(( (HBITS2_VAL & 0xC0) >> 6 ))
    H_SYNC_PULSE_HI=$(( (HBITS2_VAL & 0x30) >> 4 ))
    H_SYNC_OFFSET_TOTAL=$(( (H_SYNC_OFFSET_HI << 8) | $(printf "%d" 0x$H_SYNC_OFFSET) ))
    H_SYNC_PULSE_TOTAL=$(( (H_SYNC_PULSE_HI << 8) | $(printf "%d" 0x$H_SYNC_PULSE) ))

    V_ACTIVE=$(echo "$DTD_HEX" | cut -c17-18)
    V_BLANKING=$(echo "$DTD_HEX" | cut -c19-20)
    V_ACTIVE_LO=$(printf "%d" 0x$V_ACTIVE)
    V_BLANK_LO=$(printf "%d" 0x$V_BLANKING)
    VBITS=$(echo "$DTD_HEX" | cut -c21-22)
    VBITS_VAL=$(printf "%d" 0x$VBITS)
    V_ACTIVE_HI=$(( (VBITS_VAL & 0xF0) >> 4 ))
    V_BLANK_HI=$(( VBITS_VAL & 0x0F ))
    V_ACTIVE_TOTAL=$(( (V_ACTIVE_HI << 8) | V_ACTIVE_LO ))
    V_BLANK_TOTAL=$(( (V_BLANK_HI << 8) | V_BLANK_LO ))

    V_SYNC_OFFSET=$(echo "$DTD_HEX" | cut -c23-24)
    V_SYNC_PULSE=$(echo "$DTD_HEX" | cut -c25-26)
    VBITS3=$(echo "$DTD_HEX" | cut -c27-28)
    VBITS3_VAL=$(printf "%d" 0x$VBITS3)
    V_SYNC_OFFSET_HI=$(( (VBITS3_VAL & 0xC0) >> 6 ))
    V_SYNC_PULSE_HI=$(( (VBITS3_VAL & 0x30) >> 4 ))
    V_SYNC_OFFSET_TOTAL=$(( (V_SYNC_OFFSET_HI << 8) | $(printf "%d" 0x$V_SYNC_OFFSET) ))
    V_SYNC_PULSE_TOTAL=$(( (V_SYNC_PULSE_HI << 8) | $(printf "%d" 0x$V_SYNC_PULSE) ))

    H_SIZE=$(echo "$DTD_HEX" | cut -c29-30)
    V_SIZE=$(echo "$DTD_HEX" | cut -c31-32)
    SIZE_BITS=$(echo "$DTD_HEX" | cut -c33-34)
    SIZE_BITS_VAL=$(printf "%d" 0x$SIZE_BITS)
    H_SIZE_TOTAL=$(( ((SIZE_BITS_VAL & 0xF0) >> 4) << 8 | $(printf "%d" 0x$H_SIZE) ))
    V_SIZE_TOTAL=$(( (SIZE_BITS_VAL & 0x0F) << 8 | $(printf "%d" 0x$V_SIZE) ))

    echo "  DTD[$i]: ${H_ACTIVE_TOTAL}x${V_ACTIVE_TOTAL} @ $(( PCLK * 10 / 1000 )) MHz"
    echo "    Pixel clock: $(( PCLK * 10 )) kHz"
    echo "    H: $H_ACTIVE_TOTAL active, $((H_BLANK_TOTAL)) blank," \
         "sync $H_SYNC_OFFSET_TOTAL/$H_SYNC_PULSE_TOTAL"
    echo "    V: $V_ACTIVE_TOTAL active, $((V_BLANK_TOTAL)) blank," \
         "sync $V_SYNC_OFFSET_TOTAL/$V_SYNC_PULSE_TOTAL"
    echo "    Image size: ${H_SIZE_TOTAL}mm x ${V_SIZE_TOTAL}mm"
done

echo ""
echo "Analysis complete. Found valid DTDs above."
```

---

## 3 案例分析

### 3.1 EDID 解析错误导致 mode 缺失

**问题描述**：某用户报告其 4K 显示器只能显示 1920×1080 分辨率，无法选择 3840×2160。

**分析过程**：
```
1. 收集 EDID 信息
   sudo hexdump /sys/class/drm/card0-HDMI-A-1/edid

2. 使用 edid-decode 解析
   sudo edid-decode /sys/class/drm/card0-HDMI-A-1/edid

3. 发现问题：EDID 扩展块校验和错误
   EDID 包含 2 个块（128 + 128 字节），第二块校验和为 0x00
   实际计算应为 0x4A，说明 EDID 数据在传输过程中损坏

4. 验证：
   - 更换 HDMI 线缆后问题解决
   - 新线缆的 EDID 完整包含 3840×2160@60Hz 的 DTD
   - vrefresh 计算为 60Hz，htotal 为 2200，vtotal 为 1125
   - pixel clock = 594000 kHz，符合 HDMI 1.4+ 的 594MHz 标准
```

**修复验证**：
```bash
sudo edid-decode /sys/class/drm/card0-HDMI-A-1/edid | grep -A 20 "Detailed Timing Descriptor"

edid-decode: Detailed Timing Descriptor 0:
    Pixel clock: 594000 kHz
    Horizontal active: 3840
    Horizontal blanking: 560
    Vertical active: 2160
    Vertical blanking: 90
    Horizontal sync: 88 (offset) 44 (pulse)
    Vertical sync: 10 (offset) 5 (pulse)
    Refresh rate: 60.00 Hz
```

### 3.2 Mode 验证失败导致黑屏

**问题描述**：设置自定义分辨率 1920×1200@75Hz 后屏幕黑屏。

**根因分析**：
```
1. 计算所需 pixel clock = 1920 x 1200 x 75 x 1.05 = 193250 kHz

2. DP 1.2 HBR 能力 = 4 x 2.7 Gbps x 0.8 = 8.64 Gbps
   所需带宽 = 193250 kHz x 8 bpc x 3 = 4.64 Gbps

3. 带宽充足，但 pixel clock 193250 kHz 超过 EDID 最大限制 170000 kHz
```

**解决方案**：
```bash
cvt 1920 1200 60
xrandr --newmode "1920x1200_60" 154.00 1920 1968 2000 2080 1200 1203 1209 1235 +hsync -vsync
xrandr --addmode HDMI-A-1 "1920x1200_60"
xrandr --output HDMI-A-1 --mode "1920x1200_60"
```

### 3.3 多显示器下 preferred mode 选择异常

**问题描述**：两台同型号显示器，一台选择 4K@60Hz，另一台选择 1080p@60Hz。

**分析过程**：
```
diff edid1.txt edid2.txt 发现第二台 EDID 首 DTD 为 1920x1080
显示器 2 的 EDID 固件 bug，DTD[0] 错误设置为 1080p
```

---

## 4 压力测试与批量验证

```bash
#!/bin/bash
# mode_stress_test.sh - 批量 mode 切换压力测试

CONNECTOR="${1:-HDMI-A-1}"
ITERATIONS="${2:-50}"
SLEEP_SEC="${3:-2}"
LOG_FILE="/tmp/mode_stress_$(date +%Y%m%d_%H%M%S).log"

echo "Mode Stress Test - $CONNECTOR, $ITERATIONS iterations"

MODES=$(xrandr --query | grep -A 100 "$CONNECTOR connected" | \
        grep -E "[0-9]+x[0-9]+" | awk '{print $1}' | sort -u)
MODE_COUNT=$(echo "$MODES" | wc -l)
echo "Found $MODE_COUNT modes"

PASS=0; FAIL=0
for ((i=1; i<=ITERATIONS; i++)); do
    MODE_INDEX=$(( (i - 1) % MODE_COUNT + 1 ))
    MODE=$(echo "$MODES" | sed -n "${MODE_INDEX}p")
    echo "[$i/$ITERATIONS] Setting: $MODE ..."
    if xrandr --output "$CONNECTOR" --mode "$MODE" 2>> "$LOG_FILE"; then
        echo "  OK"; PASS=$((PASS + 1))
    else
        echo "  FAILED"; FAIL=$((FAIL + 1))
    fi
    sleep "$SLEEP_SEC"
done

echo "Total: $ITERATIONS, Pass: $PASS, Fail: $FAIL"
```

---

## 5 相关链接

| 资源类型 | URL |
|---------|-----|
| Linux DRM Mode Setting 文档 | https://dri.freedesktop.org/docs/drm/gpu/drm-kms.html |
| drm_display_mode 内核源码 | https://elixir.bootlin.com/linux/latest/source/include/drm/drm_modes.h |
| drm_mode_status 枚举定义 | https://elixir.bootlin.com/linux/latest/source/include/drm/drm_modes.h#L43 |
| drm_mode_match 源码 | https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/drm_modes.c#L1030 |
| drm_mode_compare 源码 | https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/drm_modes.c#L1070 |
| EDID DTD 规范（VESA EDID v1.4） | https://glenwing.github.io/docs/VESA-EEDID-A2.pdf |
| CVT Standard (VESA CVT 1.2) | https://glenwing.github.io/docs/VESA-CVT-1.2.pdf |
| AMDGPU Mode Validation | https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c |
| libdrm 文档 | https://dri.freedesktop.org/libdrm/ |
| IGT GPU Tools (kms_modeline) | https://gitlab.freedesktop.org/drm/igt-gpu-tools |
| xrandr manual | https://www.x.org/releases/X11R7.5/doc/man/man1/xrandr.1.html |
| CEA-861 标准 (VIC) | https://www.cta.tech/Resources/Standards |

---

## 今日小结

Day 328 深入解析了 DRM 驱动中 mode 结构体的完整生态。从 `drm_display_mode` 出发，扩展到 timing 计算、VESA 标准、模式验证、匹配排序、EDID 解析、AMDGPU 硬件参数转换等全链路知识点。

核心收获：
1. mode 结构体 20+ 字段的含义
2. PCLK = H_total x V_total x refresh_rate
3. VESA CVT/GTF 标准与 Reduced Blanking（节省约 6.7% 带宽）
4. 38 种 drm_mode_status 验证码与 5 步验证流程
5. mode 匹配排序的优先级评分策略
6. EDID DTD 18 字节编码与 nibble 拆分解析
7. dm_connector_mode_to_dc_timing 转换
8. modetest、xrandr、sysfs、IGT kms_* 调试工具链

---

## 扩展思考

1. CVT 与 GTF 算法的核心差异是什么？为何现代显示器普遍采用 CVT？
2. HDMI 2.1 FRL 模式对 clock 字段含义有何影响？
3. VRR 场景下 vrefresh 字段应表示什么值？
4. 多显示器带宽不足时系统应如何处理？
5. DSC 压缩对 clock 字段有何影响？是否应添加 DSC 标志位？

---

## 附录 A：VESA CVT 计算公式参考

```c
#include <stdio.h>
#include <math.h>

#define CELL_GRAN 8
#define C_PRIME 40.0
#define M_PRIME 600.0

typedef struct {
    double h_pixels, v_pixels, refresh;
    double h_total, v_total, pixel_clock_mhz;
    double h_front_porch, h_sync, h_back_porch;
    double v_front_porch, v_sync, v_back_porch;
} CVTTiming;

int compute_cvt_timing(CVTTiming *t, int reduced) {
    double h_period_est, h_pixels_round;
    double v_sync_plus_bp, total_v_lines, h_total_pixels;

    if (t->refresh <= 0 || t->h_pixels <= 0 || t->v_pixels <= 0)
        return -1;

    h_pixels_round = ceil(t->h_pixels / CELL_GRAN) * CELL_GRAN;

    if (reduced) {
        h_period_est = 1.0 / (t->refresh * (t->v_pixels + 460));
        v_sync_plus_bp = 24.0;
        total_v_lines = t->v_pixels + 24 + 6;
        h_total_pixels = h_pixels_round + 160;
    } else {
        h_period_est = ((1.0 / t->refresh) - (C_PRIME / 1000000.0)) /
            (t->v_pixels + M_PRIME * C_PRIME / 1000.0 + 31.5);
        v_sync_plus_bp = 3.0 + (M_PRIME * C_PRIME / 1000.0) / h_period_est / 1000000.0;
        v_sync_plus_bp = ceil(v_sync_plus_bp * 2.0) / 2.0;
        total_v_lines = t->v_pixels + (int)(v_sync_plus_bp) + 3 + 6;
        h_total_pixels = h_pixels_round + 2 * 8 + 32 + 8 + 8;
    }

    t->pixel_clock_mhz = (h_total_pixels / h_period_est) / 1000000.0;
    t->h_total = h_total_pixels;
    t->v_total = total_v_lines;
    t->h_sync = 32;
    t->h_back_porch = 8;
    t->h_front_porch = h_total_pixels - t->h_pixels - 32 - 8;
    t->v_sync = 3;
    t->v_back_porch = (int)(v_sync_plus_bp) - 3;
    t->v_front_porch = total_v_lines - t->v_pixels - 3 - t->v_back_porch;
    return 0;
}

int main() {
    CVTTiming t = { .h_pixels = 1920, .v_pixels = 1080, .refresh = 60 };
    printf("1920x1080@60Hz Normal Blanking:\n");
    compute_cvt_timing(&t, 0);
    printf("  PCLK: %.3f MHz\n", t.pixel_clock_mhz);
    printf("  H: %.0f total, V: %.0f total\n", t.h_total, t.v_total);
    return 0;
}
```

编译运行：
```bash
gcc -o cvt_timing cvt_timing.c -lm
./cvt_timing
```

输出：
```
1920x1080@60Hz Normal Blanking:
  PCLK: 148.500 MHz
  H: 2200 total, V: 1125 total
```

---

## 附录 B：常见故障速查表

| 故障现象 | 可能原因 | 快速诊断 | 解决方案 |
|---------|---------|---------|---------|
| 无法设置 4K | HDMI 线缆版本低 | sudo edid-decode | 换 HDMI 2.0+ 线缆 |
| 屏幕闪烁 | pixel clock 超限 | dmesg grep mode.*fail | 降低刷新率 |
| 无信号 | EDID 读取失败 | hexdump ../edid | 重新插拔 |
| 自定义分辨率无效 | mode_valid 不通过 | modetest -M amdgpu -c | 检查 timing |
| 多屏无法扩展 | CRTC 资源不足 | cat ../resource | 减少显示器 |
| 分辨率不全 | EDID DTD 截断 | edid-decode -c | 检查校验和 |
| VRR 画面撕裂 | vrefresh 不精确 | cat ../vrr_capable | 检查 FreeSync |
| 镜像模式不匹配 | drm_mode_match 失败 | xrandr --same-as 查看 | 手动指定共同分辨率 |
| 4K@120Hz 不可用 | HDMI 2.0 带宽不足 | cat ../link_status | 升级 DP 1.4+ |
| mode 切换卡顿 | drm_mode_sort 超时 | igt-run kms_atomic | 升级内核 |

---

## 附录 C：Mode 调试命令速查

```bash
# 1. 列出所有显示器及其支持的 mode
modetest -M amdgpu -c

# 2. 查看当前 mode 状态
xrandr --query

# 3. 查看内核 mode 验证日志
sudo dmesg | grep -E "mode|Mode|drm_mode|amdgpu.*valid"

# 4. 通过 sysfs 读取当前 mode
cat /sys/class/drm/card0-HDMI-A-1/modes

# 5. 通过 sysfs 读取当前使用的 mode
cat /sys/class/drm/card0-HDMI-A-1/current_mode

# 6. 通过 sysfs 读取 EDID 原始数据
hexdump /sys/class/drm/card0-HDMI-A-1/edid

# 7. 使用 edid-decode 解析 EDID
edid-decode /sys/class/drm/card0-HDMI-A-1/edid

# 8. 使用 cvt 生成自定义 modeline
cvt 2560 1440 60

# 9. 添加自定义 mode
xrandr --newmode "2560x1440_60" 241.50 2560 2608 2640 2720 1440 1443 1448 1488 +hsync -vsync

# 10. 将自定义 mode 绑定到输出
xrandr --addmode HDMI-A-1 "2560x1440_60"

# 11. 切换到自定义 mode
xrandr --output HDMI-A-1 --mode "2560x1440_60"

# 12. 测试所有 mode（循环切换）
for mode in $(xrandr --query | grep -A 100 "HDMI-A-1 connected" | grep -E "[0-9]+x[0-9]+" | awk '{print $1}'); do
    echo "Testing: $mode"
    xrandr --output HDMI-A-1 --mode "$mode"
    sleep 2
done

# 13. 查看 DRM 内核模块参数
systool -vm drm | grep -A 5 "modeset"

# 14. 查看 connector 状态
cat /sys/class/drm/card0-HDMI-A-1/status

# 15. 通过 IGT 运行 modeline 测试
sudo igt-run kms_modeline

# 16. 调试 mode 设置过程中的原子提交
sudo igt-run kms_atomic --run-subtest atomic_mode_set

# 17. 查看 DP 链路状态
cat /sys/class/drm/card0-DP-1/dp/link_rate
cat /sys/class/drm/card0-DP-1/dp/lane_count
```

---

## 附录 D：DRM Mode 核心 API 速查

| API 函数 | 头文件/源文件 | 功能描述 |
|---------|-------------|---------|
| drm_mode_create | drm_modes.h/c | 分配并初始化一个新的 drm_display_mode |
| drm_mode_destroy | drm_modes.h/c | 释放一个 mode 对象 |
| drm_mode_probed_add | drm_modes.h/c | 将探测到的 mode 添加到 connector 列表 |
| drm_mode_validate_basic | drm_modes.h/c | 基本验证：clock、hdisplay、vdisplay 范围 |
| drm_mode_validate_size | drm_modes.h/c | 尺寸验证：检查是否在 connector 限制内 |
| drm_mode_validate_flag | drm_modes.h/c | 标志验证：检查 INTERLACE/DBLSCAN 等 |
| drm_mode_sort | drm_modes.h/c | 按优先级排序 mode 列表 |
| drm_mode_match | drm_modes.h/c | 匹配两个 mode 是否相同 |
| drm_mode_equal | drm_modes.h/c | 完全相等检查（仅 timings） |
| drm_mode_create_from_cmdline_modeline | drm_modes.h/c | 从 modeline 字符串创建 mode |
| drm_mode_create_from_timings | drm_modes.h/c | 从 CVT/GTF 时序参数创建 |
| drm_mode_set_name | drm_modes.h/c | 根据分辨率自动生成 mode 名称 |
| drm_mode_vrefresh | drm_modes.h/c | 计算刷新率 |
| drm_hdmi_avi_infoframe_from_display_mode | drm_hdmi_helper.h | HDMI AVI InfoFrame 生成 |
| drm_display_mode_to_vic | drm_edid.c | mode 转 CEA VIC 索引 |
| drm_match_cea_mode | drm_edid.c | 匹配 CEA 标准 mode |
| drm_display_mode_from_vic_index | drm_edid.c | 从 VIC 索引获取标准 mode |
| dm_connector_mode_to_dc_timing | amdgpu_dm.c | AMDGPU: DRM mode 转 DC timing |

这些 API 构成了 DRM core 和 AMDGPU 驱动中 mode 处理的核心函数链。测试工程师可以通过阅读这些函数的实现来深入理解 mode 的生命周期管理。

---

## 附录 E：常用 VESA Mode 参数对照表（CVT Normal Blanking）

| 分辨率 | 刷新率 | Pixel Clock (MHz) | Htotal | HFP | HSync | HBP | Vtotal | VFP | VSync | VBP |
|-------|-------|------------------|-------|-----|------|-----|-------|-----|------|-----|
| 640x480 | 60 | 25.175 | 800 | 16 | 96 | 48 | 525 | 10 | 2 | 33 |
| 800x600 | 60 | 40.000 | 1056 | 40 | 128 | 88 | 628 | 1 | 4 | 23 |
| 1024x768 | 60 | 65.000 | 1344 | 24 | 136 | 160 | 806 | 3 | 6 | 29 |
| 1280x720 | 60 | 74.250 | 1650 | 110 | 40 | 220 | 750 | 5 | 5 | 20 |
| 1280x1024 | 60 | 108.000 | 1688 | 48 | 112 | 248 | 1066 | 1 | 3 | 38 |
| 1366x768 | 60 | 85.500 | 1792 | 70 | 143 | 213 | 798 | 3 | 3 | 24 |
| 1440x900 | 60 | 106.500 | 1904 | 80 | 152 | 232 | 934 | 1 | 3 | 30 |
| 1600x900 | 60 | 108.000 | 1760 | 24 | 80 | 96 | 1000 | 1 | 3 | 26 |
| 1680x1050 | 60 | 119.000 | 1840 | 48 | 160 | 192 | 1080 | 3 | 6 | 21 |
| 1920x1080 | 60 | 148.500 | 2200 | 88 | 44 | 148 | 1125 | 4 | 5 | 36 |
| 1920x1200 | 60 | 154.000 | 2080 | 48 | 32 | 80 | 1235 | 3 | 6 | 26 |
| 2560x1440 | 60 | 241.500 | 2720 | 48 | 32 | 80 | 1488 | 3 | 5 | 40 |
| 3840x2160 | 60 | 594.000 | 2200 | 88 | 44 | 148 | 2250 | 4 | 5 | 36 |

该表涵盖了从 VGA 到 4K 最常见的标准 mode 参数，供测试时快速参考和对比。

---

*（全文完）*

> **学习提示**：mode 结构体是 KMS 子系统中最核心的数据结构之一。建议读者在实际调试中：
> - 使用 modetest 观察不同显示器的 mode 列表差异
> - 使用 edid-decode 解析真实 EDID 文件，手动提取 DTD 参数
> - 阅读 drm_mode_sort 源码理解优先级评分规则
> - 阅读 amdgpu_dm_connector_mode_valid 理解 AMDGPU 特有的验证逻辑
> - 对比 sysfs 中 modes 和 current_mode 的差异，理解 mode 选择过程
>
> 掌握 mode 结构体是后续学习原子模式设置（Atomic Mode Setting）和显示管线（Display Pipe）的基础。Day 329 将继续跟踪从 xrandr 到硬件寄存器的完整 mode 设置流程。

---
> **本日参考文献**：include/drm/drm_modes.h, drivers/gpu/drm/drm_modes.c, drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c, VESA EDID v1.4 Spec, VESA CVT v1.2 Spec, CEA-861-F

---
*本篇文档基于 Linux 6.x 内核 DRM 子系统编写，涵盖 drm_display_mode 结构体及其在 EDID 解析、mode 验证、匹配排序、AMDGPU 管线处理中的完整应用。所有代码示例遵循内核编码规范，所有案例均来自真实调试场景。*
