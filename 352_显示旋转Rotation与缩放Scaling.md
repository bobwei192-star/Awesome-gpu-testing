# Day 352: 显示旋转（Rotation）与缩放（Scaling）

## 学习目标

1. 深入理解 DRM 子系统中旋转（Rotation）和缩放（Scaling）的硬件与软件实现机制
2. 掌握 DRM 核内旋转属性（drm_rotation）的定义、组合方式以及与 Plane 的绑定关系
3. 熟悉 AMDGPU DCN 硬件中旋转/缩放的处理流程，包括 DPP（Data Plane Pipeline）和 MPC（Multiple Pipe Combined）中的 scaler 单元
4. 学会使用用户空间工具（modetest、weston、igt）设置旋转和缩放模式
5. 理解旋转/缩放对性能、显存带宽和显示质量的影响，并能够进行针对性的调优
6. 掌握 viewport（视口）与 buffer（缓冲）尺寸的映射关系及其在驱动中的计算方式
7. 能够编写实际的 C 和 Shell 程序来测试、验证和分析旋转/缩放行为

## 知识详解

### 1. DRM 旋转与反射属性体系

#### 1.1 基本概念

在 DRM/KMS 子系统中，旋转（Rotation）和反射（Reflection）是 Plane（平面）级别的属性。它们定义了 framebuffer 内容在显示输出前的几何变换方式。这些变换由显示硬件（Display Core Next, DCN）直接完成，无需 GPU 渲染引擎参与，因此具有零性能开销的优势。

DRM 内核头文件 `include/uapi/drm/drm_mode.h` 中定义了以下旋转/反射标志：

```c
/* include/uapi/drm/drm_mode.h */
#define DRM_MODE_ROTATE_0       (1 << 0)   /* 无旋转 */
#define DRM_MODE_ROTATE_90      (1 << 1)   /* 顺时针旋转 90° */
#define DRM_MODE_ROTATE_180     (1 << 2)   /* 旋转 180° */
#define DRM_MODE_ROTATE_270     (1 << 3)   /* 顺时针旋转 270°（等价于逆时针 90°） */
#define DRM_MODE_REFLECT_X      (1 << 4)   /* 沿 X 轴反射（水平翻转） */
#define DRM_MODE_REFLECT_Y      (1 << 5)   /* 沿 Y 轴反射（垂直翻转） */
```

这些标志是位掩码，可以组合使用。例如，`DRM_MODE_ROTATE_90 | DRM_MODE_REFLECT_X` 表示先水平翻转，再顺时针旋转 90°。

#### 1.2 旋转属性与 Plane 的绑定

每个 drm_plane 可以声明它支持的旋转能力。在驱动初始化时，通过调用 `drm_plane_create_rotation_property()` 来创建旋转属性：

```c
/**
 * drm_plane_create_rotation_property - 为平面创建旋转属性
 * @plane: DRM 平面对象
 * @rotation: 初始旋转值（通常为 DRM_MODE_ROTATE_0）
 * @supported_rotations: 支持的旋转/反射位掩码
 */
int drm_plane_create_rotation_property(struct drm_plane *plane,
                                       unsigned int rotation,
                                       unsigned int supported_rotations);
```

在 AMDGPU 驱动中，`dm_plane_helper_prepare_fb()` 和 `fill_dc_plane_info_and_addr()` 等函数负责将 DRM 旋转属性转换为 DC（Display Core）层的旋转参数。

#### 1.3 旋转属性在 atomic 流程中的传递

旋转属性通过 atomic commit 流程传递到驱动层。以下是完整的数据流：

```
用户空间 (modetest/weston)
    │
    │  SETPROPERTY / ATOMIC_IOCTL
    ▼
DRM 核心 (drm_atomic.c)
    │
    │  drm_plane_state->rotation
    ▼
AMDGPU DRM 层 (amdgpu_display.c)
    │
    │  amdgpu_display_crtc_get_scanoutpos() 等
    ▼
AMDGPU DM 层 (amdgpu_dm.c)
    │
    │  fill_dc_plane_info_and_addr()
    ▼
DC Plane Info (dc_plane_info)
    │
    │  rotation, horizontal_mirror
    ▼
DCN 硬件 (dcn20/dcn30 中的 DPP)
    │
    │  DPP_SCL (scaler) + DPP_DSCL (descale)
    ▼
输出到显示器
```

### 2. 缩放模式（Scaling Mode）与属性

#### 2.1 DRM 缩放模式标志

缩放模式定义了当 framebuffer 的分辨率与显示器的原生分辨率不匹配时，如何进行缩放。DRM 核心提供了以下标准缩放模式属性：

```c
/* drm_connector.h 中的枚举 */
enum drm_scaling_mode {
    DRM_MODE_SCALE_NONE,         /* 不缩放，居中显示 */
    DRM_MODE_SCALE_FULLSCREEN,   /* 全屏缩放（拉伸） */
    DRM_MODE_SCALE_CENTER,       /* 居中显示（同 NONE） */
    DRM_MODE_SCALE_ASPECT,       /* 保持宽高比缩放 */
};
```

这些值通过 `drm_connector_attach_scaling_mode_property()` 函数附加到 Connector 属性上：

```c
int drm_connector_attach_scaling_mode_property(struct drm_connector *connector,
                                                u32 scaling_mode_mask);
```

scaling_mode_mask 参数指定了该 Connector 支持的缩放模式组合，例如：
- `BIT(DRM_MODE_SCALE_NONE) | BIT(DRM_MODE_SCALE_FULLSCREEN) | BIT(DRM_MODE_SCALE_ASPECT)`

此外，还有一个相关的 `scaling mode` 属性（注意小写），以及 `underscan` 属性用于处理 HDMI 过扫描/欠扫描。

#### 2.2 AMDGPU 中的缩放模式处理

在 AMDGPU DC 中，缩放模式的设置会影响 `struct scaler_data` 中的参数，该结构体定义了缩放器的配置：

```c
/* drivers/gpu/drm/amd/display/dc/inc/hw/scaler.h */
struct scaler_data {
    /* 源（输入）尺寸 */
    struct rect src_rect;          /* 源矩形（原始像素区域） */
    struct rect clip_rect;         /* 裁剪矩形 */

    /* 目标（输出）尺寸 */
    struct rect dst_rect;          /* 目标矩形（缩放后区域） */
    struct rect viewport;          /* 视口（实际扫描区域） */

    /* 缩放参数 */
    struct scaling_taps h_taps;    /* 水平缩放 taps */
    struct scaling_taps v_taps;    /* 垂直缩放 taps */

    /* 缩放比例（fixed-point 格式） */
    struct fixed31_32 ratio_h;     /* 水平缩放比 */
    struct fixed31_32 ratio_v;     /* 垂直缩放比 */

    /* 缩放滤波器 */
    enum dc_scr_filter_type scr_filter; /* 子像素渲染滤波器类型 */
};
```

缩放模式 → DC 层参数的映射流程：

```
connector->state->scaling_mode
    │
    ▼
dm_update_crtc_scale()  // amdgpu_dm.c
    │
    │  根据 scaling_mode 计算 dst_rect
    │  DRM_MODE_SCALE_FULLSCREEN → dst_rect = 全屏尺寸
    │  DRM_MODE_SCALE_ASPECT     → dst_rect = 等比例缩放最大尺寸
    │  DRM_MODE_SCALE_NONE       → dst_rect = 原始尺寸（居中）
    │
    ▼
fill_dc_plane_info_and_addr()
    │
    │  设置 plane_info->src_rect / dst_rect / clip_rect
    ▼
DC 的 scaler 编程
```

### 3. AMDGPU DCN 硬件的旋转/缩放实现

#### 3.1 DCN 流水线中的旋转处理

AMDGPU DCN（Display Core Next）硬件中，旋转处理发生在 DPP（Data Plane Pipeline）阶段。DCN 的每个 DPP 管道包含一个 DPP_SCL（Scaler）模块，该模块除了执行缩放外，还负责旋转/反射变换。

```
DCN 显示流水线（每个 Plane）:
    ┌─────────────┐
    │  DPP 管道    │
    │  ┌─────────┐ │
    │  │ DPP_TOP  │ │  → 格式转换、颜色调整
    │  └────┬────┘ │
    │       │      │
    │  ┌────▼────┐ │
    │  │ DPP_SCL  │ │  → 缩放 + 旋转/反射
    │  │ (Scaler) │ │
    │  └────┬────┘ │
    │       │      │
    │  ┌────▼────┐ │
    │  │ DPP_CNVC │ │  → 颜色空间转换（RGB↔YUV）
    │  └─────────┘ │
    └──────┬───────┘
           │
    ┌──────▼───────┐
    │  MPC 混合     │  → 多平面 alpha 混合
    └──────┬───────┘
           │
    ┌──────▼───────┐
    │  OPTC 输出    │  → 时序控制
    └──────────────┘
```

旋转操作在 DPP_SCL 模块中执行。DCN 硬件支持以下旋转能力（具体取决于 ASIC 版本）：

| ASIC 世代 | 支持旋转 | 支持反射 | 备注 |
|-----------|---------|---------|------|
| DCN 1.0 (Raven/Picasso) | 0°, 180° | 否 | 仅支持 180° 硬件旋转 |
| DCN 2.0 (Navi10/14) | 0°, 90°, 180°, 270° | 是 | 完整旋转支持 |
| DCN 2.1 (Navi12) | 0°, 90°, 180°, 270° | 是 | 同上 |
| DCN 3.0 (Sienna Cichlid) | 0°, 90°, 180°, 270° | 是 | 改进的缩放质量 |
| DCN 3.1 (Navy Flounder) | 0°, 90°, 180°, 270° | 是 | 无额外限制 |
| DCN 3.2 (Yellow Carp) | 0°, 90°, 180°, 270° | 是 | 支持更多格式 |
| DCN 3.5 (MI300/Strix) | 0°, 90°, 180°, 270° | 是 | 最完整支持 |

#### 3.2 旋转相关的寄存器编程

在 DCN 中，旋转参数通过 DPP_SCL 模块的寄存器进行编程。以 DCN 2.0+ 为例，关键寄存器包括：

```
DPP_SCL 寄存器组:
  ┌─────────────────────────────────────┐
  │ SCL_COEF_UPDATE_REG                  │  → 系数更新触发
  ├─────────────────────────────────────┤
  │ SCL_MODE_REG                         │  → 缩放模式选择
  ├─────────────────────────────────────┤
  │ SCL_HORZ_FILTER_SCALE_RATIO          │  → 水平缩放比例
  ├─────────────────────────────────────┤
  │ SCL_VERT_FILTER_SCALE_RATIO          │  → 垂直缩放比例
  ├─────────────────────────────────────┤
  │ SCL_ROTATION_ANGLE                   │  → 旋转角度
  ├─────────────────────────────────────┤
  │ SCL_HORZ_MIRROR                      │  → 水平镜像（反射 X）
  ├─────────────────────────────────────┤
  │ SCL_VIEWPORT_SIZE / OFFSET           │  → 视口尺寸和偏移
  └─────────────────────────────────────┘
```

这些寄存器在驱动中的编程入口位于 `drivers/gpu/drm/amd/display/dc/dcn20/dcn20_dpp_scl.c`：

```c
/* dcn20_dpp_scl.c 中的片段 */
void dpp2_set_scaler(struct dpp *dpp, struct scaling_data *data)
{
    struct dcn20_dpp *dpp20 = TO_DCN20_DPP(dpp);

    /* 配置缩放比例 */
    REG_SET(SCL_HORZ_FILTER_SCALE_RATIO, 0,
            SCL_H_FILTER_SCALE_RATIO, data->ratio_h);
    REG_SET(SCL_VERT_FILTER_SCALE_RATIO, 0,
            SCL_V_FILTER_SCALE_RATIO, data->ratio_v);

    /* 配置旋转角度 */
    REG_SET(SCL_ROTATION_ANGLE, 0,
            ROTATION_ANGLE, data->rotation);

    /* 配置镜像 */
    REG_SET(SCL_HORZ_MIRROR, 0,
            HORZ_MIRROR_EN, data->h_mirror);

    /* 配置视口 */
    REG_SET(SCL_VIEWPORT_SIZE, 0,
            VIEWPORT_WIDTH, data->viewport_width);
    REG_SET(SCL_VIEWPORT_SIZE, 0,
            VIEWPORT_HEIGHT, data->viewport_height);
}
```

#### 3.3 DC 层的旋转参数传递

从 DRM 到 DC 的旋转参数传递发生在 `fill_dc_plane_info_and_addr()` 函数中：

```c
/* drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c */
static int fill_dc_plane_info_and_addr(struct amdgpu_device *adev,
                                       struct drm_plane_state *plane_state,
                                       struct dc_plane_info *plane_info)
{
    /* ... 其他参数填充 ... */

    /* 转换 DRM 旋转标志 → DC 旋转参数 */
    switch (plane_state->rotation & DRM_MODE_ROTATE_MASK) {
    case DRM_MODE_ROTATE_0:
        plane_info->rotation = ROTATION_ANGLE_0;
        break;
    case DRM_MODE_ROTATE_90:
        plane_info->rotation = ROTATION_ANGLE_90;
        break;
    case DRM_MODE_ROTATE_180:
        plane_info->rotation = ROTATION_ANGLE_180;
        break;
    case DRM_MODE_ROTATE_270:
        plane_info->rotation = ROTATION_ANGLE_270;
        break;
    }

    /* 转换反射标志 */
    plane_info->horizontal_mirror = !!(plane_state->rotation & DRM_MODE_REFLECT_X);
    /* 注意：DRM_MODE_REFLECT_Y 通过交换 src 坐标的 y 值处理 */

    /* 转换缩放源/目标矩形 */
    /* ... src_rect, dst_rect, clip_rect ... */

    return 0;
}
```

#### 3.4 Viewport 与 Buffer 的映射关系

当旋转 90° 或 270° 时，viewport（视口）的宽度和高度需要与 buffer 的原始高度和宽度交换。这是旋转处理中最关键的部分之一。

```
原始 Buffer (1920x1080)         旋转 90° 后的视口 (1080x1920)

  ┌──────────────────┐            ┌──────────────┐
  │                  │            │              │
  │                  │    90° CW  │              │
  │   1920 x 1080    │  ────────  │  1080 x 1920 │
  │                  │            │              │
  │                  │            │              │
  └──────────────────┘            └──────────────┘
  宽 = 1920, 高 = 1080            宽 = 1080, 高 = 1920
  pitch = 1920 * bpp              pitch = 1920 * bpp (不变!)
```

注意：虽然视口的宽高互换，但 buffer 的 pitch（行跨度）保持不变。这意味着旋转 90° 后，硬件在读取像素时，会以 pitch 为步长在垂直方向扫描，而不是水平方向。

```c
/* 核心处理逻辑 — 在 drm_atomic_helper_check_plane_state() 中 */
static int drm_atomic_helper_check_plane_state(struct drm_plane_state *state)
{
    /* ... */

    /* 计算缩放后的裁剪矩形 */
    /* 对于旋转，交换源矩形的宽高 */
    if (state->rotation & DRM_MODE_ROTATE_90 ||
        state->rotation & DRM_MODE_ROTATE_270) {
        /* 交换宽高，因为旋转会交换坐标系 */
        swap(state->src_w, state->src_h);
    }

    /* 计算视口尺寸 */
    /* viewport 是根据 src 矩形和旋转计算得出的 */

    /* ... */
}
```

### 4. 缩放算法与滤波器

#### 4.1 缩放质量级别

DCN 硬件提供了多种缩放滤波器，适用于不同场景：

| 滤波器类型 | 质量级别 | 适用场景 | 性能开销 |
|-----------|---------|---------|---------|
| 最近邻 (Nearest Neighbor) | 低 | 像素艺术、放大倍数整数 | 极低 |
| 双线性 (Bilinear) | 中 | 一般用途、性能优先 | 低 |
| 双三次 (Bicubic) | 高 | 高质量缩放、照片 | 中 |
| Lanczos | 最高 | 专业图像处理 | 高 |
| FSR (FidelityFX Super Resolution) | 自适应 | 游戏、实时渲染 | 中高 |

AMDGPU DC 中的缩放滤波器系数存储在固件中，驱动根据缩放比例选择适当的系数。DCN 的 DPP_SCL 模块支持多达 8-tap 的垂直滤波器和 8-tap 的水平滤波器。

#### 4.2 fsr（FidelityFX Super Resolution）与缩放

AMD 的 FSR 技术在显示驱动层面也有集成。对于不支持 FSR 的游戏和应用，DC 子系统可以通过硬件缩放器提供类似的效果。FSR 在驱动中的标志定义：

```c
/* drivers/gpu/drm/amd/display/dc/inc/hw/scaler.h */
enum dc_scr_filter_type {
    SCALER_SCFILTER_DEFAULT = 0,     /* 默认缩放滤波器 */
    SCALER_SCFILTER_BILINEAR,        /* 双线性 */
    SCALER_SCFILTER_BICUBIC,         /* 双三次 */
    SCALER_SCFILTER_FSR,             /* FidelityFX Super Resolution */
    SCALER_SCFILTER_FSR_OLD,         /* 旧版 FSR（兼容性） */
};
```

注意：FSR 在驱动层面的集成目前处于早期阶段，大多数 FSR 处理在应用/游戏层完成，驱动主要负责检测和上报显示能力。

### 5. 旋转/缩放在 Wayland/Weston 中的实际应用

#### 5.1 Weston 的旋转处理

Wayland 合成器 Weston 使用 DRM KMS atomic 接口来配置旋转。当显示器物理安装方向改变时（例如竖屏显示器），合成器会设置相应的旋转属性。

Weston 中的旋转配置：

```c
/* weston/libweston/backend-drm/plane.c */
static int
drm_plane_set_rotation(struct drm_plane *plane,
                        struct drm_output_state *state,
                        unsigned int rotation)
{
    /* 检查硬件是否支持请求的旋转 */
    if (!(plane->props.rotation & rotation))
        return -EINVAL;

    /* 设置旋转属性 */
    return drmModeObjectSetProperty(plane->fd, plane->plane_id,
                                     plane->props.rotation_id, rotation);
}
```

#### 5.2 Mutter(GNOME) 的旋转策略

GNOME 的 Mutter 合成器使用了一种"智能旋转"策略：如果硬件支持旋转，则直接使用硬件旋转（零开销）；否则，使用 GPU shader 进行软件旋转回退。

```c
/* mutter/src/backends/native/meta-crtc-kms.c */
static gboolean
meta_crtc_kms_set_rotation(MetaCrtcKms *crtc_kms,
                            MetaMonitorTransform transform)
{
    /* 将 Mutter 变换枚举映射到 DRM 旋转标志 */
    unsigned int drm_rotation = meta_monitor_transform_to_drm(transform);

    /* 检查 KMS 平面是否支持该旋转 */
    if (drm_plane_supports_rotation(crtc_kms->plane, drm_rotation)) {
        /* 使用硬件旋转 */
        crtc_kms->rotation = drm_rotation;
        return TRUE;
    }

    /* 回退到软件旋转（使用 GPU 渲染） */
    return meta_crtc_kms_fallback_rotation(crtc_kms, transform);
}
```

### 6. IGT 测试套件中的旋转测试

IGT (Intel GPU Tools) 提供了全面的旋转测试，位于 `tests/kms_rotation_crc.c`。该测试验证各种旋转组合下的 CRC 正确性。

关键测试用例：

```c
/* IGT kms_rotation_crc 测试说明 */

/* 1. 基本旋转测试 */
IGT_TEST(rotation_0_degrees)   /* 验证 0° 旋转 CRC */
IGT_TEST(rotation_90_degrees)  /* 验证 90° 旋转 CRC */
IGT_TEST(rotation_180_degrees) /* 验证 180° 旋转 CRC */
IGT_TEST(rotation_270_degrees) /* 验证 270° 旋转 CRC */

/* 2. 组合测试 */
IGT_TEST(rotation_90_reflect_x) /* 90° + 水平翻转 */
IGT_TEST(rotation_180_reflect_y) /* 180° + 垂直翻转 */

/* 3. 多平面测试 */
IGT_TEST(rotation_plane_primary) /* 主平面旋转 */
IGT_TEST(rotation_plane_overlay) /* 覆盖平面旋转 */
IGT_TEST(rotation_plane_cursor)  /* 光标平面旋转（通常不支持） */

/* 4. 缩放 + 旋转组合测试 */
IGT_TEST(rotation_scale_full)   /* 全屏缩放 + 旋转 */
IGT_TEST(rotation_scale_aspect) /* 保持比例缩放 + 旋转 */
IGT_TEST(rotation_scale_center) /* 居中 + 旋转 */
```

运行方法：

```bash
# 运行所有旋转测试
./kms_rotation_crc --run-subtest all

# 运行特定测试
./kms_rotation_crc --run-subtest rotation-90-degrees

# 测试 180° 旋转的 CRC
./kms_rotation_crc --run-subtest rotation-180-degrees

# 测试缩放 + 旋转组合
./kms_rotation_crc --run-subtest rotation-scale-aspect

# 使用循环测试所有可能的旋转组合
for rot in 0 90 180 270; do
    for reflect in "" "reflect-x" "reflect-y" "reflect-x-reflect-y"; do
        subtest="rotation-${rot}-degrees"
        [ -n "$reflect" ] && subtest="${subtest}-${reflect}"
        echo "Running: $subtest"
        ./kms_rotation_crc --run-subtest "$subtest"
    done
done
```

### 7. 旋转/缩放对性能的影响

#### 7.1 显存带宽影响

旋转对显存访问模式有显著影响：

```
原始显存布局（线性，无旋转）：
  ┌────┬────┬────┬────┬────┬────┐
  │ P0 │ P1 │ P2 │ P3 │ P4 │ P5 │  ← 行 0 (连续访问)
  ├────┼────┼────┼────┼────┼────┤
  │ P6 │ P7 │ P8 │ P9 │P10 │P11 │  ← 行 1
  ├────┼────┼────┼────┼────┼────┤
  │P12 │P13 │P14 │P15 │P16 │P17 │  ← 行 2
  └────┴────┴────┴────┴────┴────┘
  访问模式：顺序 → 良好的缓存利用率

旋转 90° 后的显存访问：
  ┌────┬────┬────┐
  │ P0 │ P6 │P12 │  ← 原来的一列（非连续访问）
  ├────┼────┼────┤
  │ P1 │ P7 │P13 │
  ├────┼────┼────┤
  │ P2 │ P8 │P14 │
  ├────┼────┼────┤
  │ P3 │ P9 │P15 │
  ├────┼────┼────┤
  │ P4 │P10 │P16 │
  ├────┼────┼────┤
  │ P5 │P11 │P17 │
  └────┴────┴────┘
  访问模式：跨步（stride）→ 缓存利用率降低

显存带宽影响：
  ├───────────────────┬────────────┬──────────────┤
  │ 旋转角度          │ 带宽开销   │ 适用场景      │
  ├───────────────────┼────────────┼──────────────┤
  │ 0°                │ 0%         │ 标准模式      │
  │ 180°              │ ~5-10%     │ 宽高不变      │
  │ 90° / 270°        │ ~20-50%    │ 竖屏/手机      │
  └───────────────────┴────────────┴──────────────┘
```

#### 7.2 缩放性能开销

缩放的性能开销主要来自滤波器计算：

```
缩放性能对比：

  无缩放（1:1）    ───  0% 开销
       │
  双线性缩放      ───  ~3-5% 开销（每 tap 的乘加操作）
       │
  双三次缩放      ───  ~8-15% 开销（更多 tap + 权重计算）
       │
  Lanczos 缩放    ───  ~20-30% 开销（大量 sinc 计算）
       │
  FSR 缩放        ───  ~15-25% 开销（边缘检测 + 自适应重建）

注：以上为硬件缩放的开销，软件回退缩放的开销要高 10-100 倍
```

### 8. 缩放模式与 Overscan/Underscan

#### 8.1 HDMI 过扫描与欠扫描

过扫描（Overscan）是早期 CRT 电视的标准行为——图像边缘被裁剪以隐藏扫描畸变。欠扫描（Underscan）则在图像周围保留黑边。

AMDGPU 驱动中通过 Connector 属性控制：

```bash
# 查看当前 underscan 设置
cat /sys/class/drm/card0-DP-1/underscan

# 设置 underscan 模式
# 'off'  - 关闭（全屏显示，可能被裁剪）
# 'on'   - 开启（保留 5% 黑边）
# 'auto' - 根据显示器 EDID 自动选择
```

#### 8.2 缩放模式的用户空间配置

通过 modetest 或 libdrm API 设置缩放模式：

```bash
# 使用 modetest 查看缩放模式属性
modetest -M amdgpu -c | grep -A 5 "scaling"

# 通过 sysfs 设置（如果驱动支持）
echo "Full aspect" > /sys/class/drm/card0-DP-1/scaling_mode
```

支持的缩放模式值：
- `"None"`：居中显示，不缩放
- `"Full"`：全屏拉伸
- `"Center"`：居中（同 None）
- `"Full aspect"`：保持宽高比

### 9. 旋转/缩放调试工具与方法

#### 9.1 CRC 校验验证旋转正确性

使用 IGT 的 CRC 校验机制可以验证旋转是否按预期工作：

```bash
# 1. 获取 pipe CRC 源
cat /sys/class/drm/card0/crtc-0/crc/data

# 2. 捕获带旋转的 CRC
./kms_rotation_crc --run-subtest rotation-90-degrees -v

# 3. 比对 CRC
# 旋转后的 CRC 与未旋转时不应相同
```

#### 9.2 使用 drm_info 调试旋转属性

```bash
# 安装 drm_info
git clone https://gitlab.freedesktop.org/emersion/drm_info.git
cd drm_info && meson build && ninja -C build

# 查看平面的旋转能力
./build/drm_info | grep -A 10 "rotation"
```

示例输出：

```
Plane 0:
    ...
    Properties:
        9 rotation:
            flags: atomic
            enum values: "rotate-0" (0x1), "rotate-90" (0x2),
                         "rotate-180" (0x4), "rotate-270" (0x8),
                         "reflect-x" (0x10), "reflect-y" (0x20)
            current value: rotate-0
```

#### 9.3 通过 sysfs 调试旋转

```bash
# 检查 CRTC 的旋转能力
cat /sys/class/drm/card0/crtc-0/rotation

# 启用旋转调试日志（内核动态调试）
echo 'file amdgpu_dm.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file dcn20_dpp_scl.c +p' > /sys/kernel/debug/dynamic_debug/control
```

### 10. 旋转/缩放的常见问题与故障排查

#### 10.1 问题分类表

| 问题现象 | 可能原因 | 检查方法 | 解决方案 |
|---------|---------|---------|---------|
| 旋转后图像扭曲 | Viewport 计算错误 | 检查 dmesg 中的 viewport 日志 | 更新内核驱动 |
| 旋转后性能下降 | 使用了软件回退而非硬件旋转 | `cat /sys/kernel/debug/dri/0/plane_rotation` | 确认硬件是否支持该旋转 |
| 缩放后图像模糊 | 使用低质量滤波器 | 检查 scaler filter 配置 | 切换到高质量滤波器 |
| 竖屏显示黑边 | 缩放模式设置为 Center | 检查 scaling_mode 属性 | 改为 Full aspect |
| 旋转 90° 后鼠标位置偏移 | 输入事件坐标系未转换 | libinput 调试 | 检查 libinput 旋转配置 |
| HDMI 下图像被裁剪 | 过扫描未关闭 | `cat /sys/class/drm/.../underscan` | 关闭 underscan |

#### 10.2 硬件旋转不生效的排查流程

```
确认硬件是否支持请求的旋转
    │
    ├── 是 ──→ 检查 Plane 的 rotation 属性是否已设置
    │          │
    │          ├── 已设置 ──→ 检查 atomic commit 是否成功
    │          │              │
    │          │              ├── 成功 ──→ 检查 DCN 寄存器值
    │          │              │              │
    │          │              │              └── 读取 SCL_ROTATION_ANGLE
    │          │              │
    │          │              └── 失败 ──→ 检查 dmesg 错误信息
    │          │
    │          └── 未设置 ──→ 检查用户空间合成器配置
    │
    └── 否 ──→ 确认是否可降级使用软件旋转
               │
               ├── 可接受 ──→ 使用 GPU shader 旋转
               │
               └── 不可接受 ──→ 考虑升级硬件
```

### 11. DRM 旋转的原子校验逻辑

#### 11.1 drm_atomic_helper_check_plane_state 中的旋转校验

在 DRM 核心的原子检查函数中，旋转角度会影响 src/dst 矩形的合法性检查。以下是核心的校验逻辑：

```c
/*
 * drm_atomic_helper_check_plane_state() 中的旋转相关校验
 * 位于 drivers/gpu/drm/drm_atomic_helper.c
 */
int drm_atomic_helper_check_plane_state(struct drm_plane_state *state,
                                         const struct drm_crtc_state *crtc_state,
                                         int min_scale, int max_scale,
                                         bool can_position,
                                         bool can_update_disabled)
{
    struct drm_framebuffer *fb = state->fb;
    struct drm_rect src = drm_plane_state_src(state);
    struct drm_rect dest = drm_plane_state_dest(state);
    unsigned int rotation = state->rotation;

    /* 1. 检查源矩形是否在 framebuffer 范围内 */
    if (rotation & DRM_MODE_ROTATE_90 || rotation & DRM_MODE_ROTATE_270) {
        /*
         * 对于 90/270° 旋转，源矩形的宽度对应 framebuffer 的高度，
         * 源矩形的高度对应 framebuffer 的宽度
         */
        if (src.x1 < 0 || src.y1 < 0 ||
            src.x2 > drm_rect_width(&src) ||  // 实际对应 fb 高度
            src.y2 > drm_rect_height(&src)) { // 实际对应 fb 宽度
            return -EINVAL;
        }
    } else {
        /* 0/180° 旋转，正常检查 */
        if (src.x1 < 0 || src.y1 < 0 ||
            src.x2 > fb->width << 16 ||
            src.y2 > fb->height << 16) {
            return -EINVAL;
        }
    }

    /* 2. 检查缩放比例是否在允许范围内 */
    /* 对于旋转，需要交换缩放的宽高比检查 */
    if (rotation & DRM_MODE_ROTATE_90 || rotation & DRM_MODE_ROTATE_270) {
        /*
         * 旋转后，源"宽度"（旋转前高度）映射到目标宽度
         * 源"高度"（旋转前宽度）映射到目标高度
         */
        int rotated_min_scale = min_scale;
        int rotated_max_scale = max_scale;

        /* 交换宽高后检查缩放比例 */
        int hscale = drm_rect_calc_hscale_relaxed(&src, &dest,
                                                    rotated_min_scale,
                                                    rotated_max_scale);
        int vscale = drm_rect_calc_vscale_relaxed(&src, &dest,
                                                    rotated_min_scale,
                                                    rotated_max_scale);
        if (hscale < 0 || vscale < 0) {
            DRM_DEBUG_ATOMIC("缩放比例超出范围 [%d, %d]\n",
                             min_scale, max_scale);
            return -EINVAL;
        }
    } else {
        /* 非旋转，正常缩放检查 */
        int hscale = drm_rect_calc_hscale_relaxed(&src, &dest,
                                                    min_scale, max_scale);
        int vscale = drm_rect_calc_vscale_relaxed(&src, &dest,
                                                    min_scale, max_scale);
        if (hscale < 0 || vscale < 0)
            return -EINVAL;
    }

    /* 3. 检查目标矩形是否在 CRTC 范围内 */
    if (dest.x1 < 0 || dest.y1 < 0 ||
        dest.x2 > crtc_state->mode.hdisplay ||
        dest.y2 > crtc_state->mode.vdisplay) {
        DRM_DEBUG_ATOMIC("目标矩形超出 CRTC 边界\n");
        return -EINVAL;
    }

    /* 4. 对于 overlay/cursor plane，检查位置是否合法 */
    if (!can_position) {
        /* 固定位置 plane，必须覆盖整个 CRTC */
        if (dest.x1 != 0 || dest.y1 != 0 ||
            drm_rect_width(&dest) != crtc_state->mode.hdisplay ||
            drm_rect_height(&dest) != crtc_state->mode.vdisplay) {
            return -EINVAL;
        }
    }

    /* 5. 保存计算后的裁剪矩形 */
    state->visible = drm_rect_clip_scaled(&src, &dest,
                                           &fb->clip, rotation);

    return 0;
}
```

#### 11.2 DRM 核心旋转帮助函数

DRM 核心提供了一些实用的内联函数来处理旋转逻辑：

```c
/* include/drm/drm_rect.h */

/**
 * drm_rect_rotate_90 - 将矩形沿 90° 旋转
 * @r: 要旋转的矩形
 * @width: 旋转前的宽度
 * @height: 旋转前的高度
 *
 * 顺时针旋转 90°：
 * (x, y) → (y, width - x - 1)
 */
static inline void drm_rect_rotate_90(struct drm_rect *r,
                                       int width, int height)
{
    /* 交换坐标 */
    *r = DRM_RECT_FP(r->y1, width - r->x2,
                      drm_rect_height(r), drm_rect_width(r));
}

/**
 * drm_rect_rotate_180 - 将矩形沿 180° 旋转
 * @r: 要旋转的矩形
 * @width: 宽度
 * @height: 高度
 *
 * 旋转 180°：
 * (x, y) → (width - x - 1, height - y - 1)
 */
static inline void drm_rect_rotate_180(struct drm_rect *r,
                                        int width, int height)
{
    *r = DRM_RECT_FP(width - r->x2, height - r->y2,
                      drm_rect_width(r), drm_rect_height(r));
}

/**
 * drm_rect_rotate_270 - 将矩形沿 270° 旋转
 * @r: 要旋转的矩形
 * @width: 旋转前的宽度
 * @height: 旋转前的高度
 *
 * 顺时针旋转 270°（逆时针 90°）：
 * (x, y) → (height - y - 1, x)
 */
static inline void drm_rect_rotate_270(struct drm_rect *r,
                                        int width, int height)
{
    *r = DRM_RECT_FP(height - r->y2, r->x1,
                      drm_rect_height(r), drm_rect_width(r));
}

/**
 * drm_rect_rotate_inv - 反向旋转矩形，用于坐标变换
 * @r: 要反向旋转的矩形
 * @width: 反向旋转前的宽度
 * @height: 反向旋转前的高度
 * @rotation: 原始旋转角度
 */
static inline void drm_rect_rotate_inv(struct drm_rect *r,
                                        int width, int height,
                                        unsigned int rotation)
{
    /* 根据旋转角度应用反向变换 */
    switch (rotation & DRM_MODE_ROTATE_MASK) {
    case DRM_MODE_ROTATE_90:
        drm_rect_rotate_270(r, width, height);
        break;
    case DRM_MODE_ROTATE_180:
        drm_rect_rotate_180(r, width, height);
        break;
    case DRM_MODE_ROTATE_270:
        drm_rect_rotate_90(r, width, height);
        break;
    default:
        break;
    }
}
```

### 12. drm_plane_state 中的旋转字段

```c
struct drm_plane_state {
    /* ... */
    struct drm_plane *plane;        /* 所属平面 */
    struct drm_crtc *crtc;          /* 关联的 CRTC */
    struct drm_framebuffer *fb;     /* 关联的 framebuffer */

    /* 源矩形（16.16 定点格式，单位为像素） */
    uint32_t src_x, src_y;
    uint32_t src_w, src_h;

    /* 目标矩形（整数格式，单位为像素） */
    uint32_t crtc_x, crtc_y;
    uint32_t crtc_w, crtc_h;

    /* 旋转/反射标志 */
    unsigned int rotation;          /* DRM_MODE_ROTATE_* | DRM_MODE_REFLECT_* */

    /* 缩放模式 */
    unsigned int scaling_mode;      /* DRM_MODE_SCALE_* */

    /* ... 其他字段 ... */
};
```

#### 11.2 drm_mode_config 中的旋转支持声明

驱动在初始化时声明旋转支持：

```c
/* AMDGPU 驱动初始化代码片段 */
int amdgpu_dm_plane_init(struct amdgpu_display_manager *dm,
                          unsigned long possible_crtcs,
                          enum drm_plane_type type)
{
    struct drm_plane *plane;
    unsigned int supported_rotations = DRM_MODE_ROTATE_0;

    /* 根据 ASIC 版本设置支持的旋转 */
    if (dm->adev->asic_type >= CHIP_NAVI10) {
        /* Navi10+ 支持完整旋转 */
        supported_rotations |= DRM_MODE_ROTATE_90 |
                                DRM_MODE_ROTATE_180 |
                                DRM_MODE_ROTATE_270 |
                                DRM_MODE_REFLECT_X |
                                DRM_MODE_REFLECT_Y;
    } else if (dm->adev->asic_type >= CHIP_RAVEN) {
        /* Raven 仅支持 180° 旋转 */
        supported_rotations |= DRM_MODE_ROTATE_180;
    }

    /* 创建旋转属性 */
    drm_plane_create_rotation_property(plane,
                                        DRM_MODE_ROTATE_0,
                                        supported_rotations);

    /* ... */
}
```

#### 11.3 dcn20 中的旋转寄存器编程

```c
/* drivers/gpu/drm/amd/display/dc/dcn20/dcn20_dpp_scl.c */

void dpp20_set_rotation(struct dpp *dpp, enum dc_rotation_angle rotation)
{
    struct dcn20_dpp *dpp20 = TO_DCN20_DPP(dpp);

    /* 设置旋转角度寄存器 */
    switch (rotation) {
    case ROTATION_ANGLE_0:
        REG_UPDATE(SCL_ROTATION_ANGLE, ROTATION_ANGLE, 0);
        break;
    case ROTATION_ANGLE_90:
        REG_UPDATE(SCL_ROTATION_ANGLE, ROTATION_ANGLE, 1);
        break;
    case ROTATION_ANGLE_180:
        REG_UPDATE(SCL_ROTATION_ANGLE, ROTATION_ANGLE, 2);
        break;
    case ROTATION_ANGLE_270:
        REG_UPDATE(SCL_ROTATION_ANGLE, ROTATION_ANGLE, 3);
        break;
    default:
        REG_UPDATE(SCL_ROTATION_ANGLE, ROTATION_ANGLE, 0);
        break;
    }

    /* 设置水平镜像寄存器 */
    if (rotation & ROTATION_MIRROR_X)
        REG_UPDATE(SCL_HORZ_MIRROR, HORZ_MIRROR_EN, 1);
    else
        REG_UPDATE(SCL_HORZ_MIRROR, HORZ_MIRROR_EN, 0);
}
```

## 实践操作

### 实践 1：使用 libdrm 设置旋转属性

以下程序演示如何使用 libdrm API 设置 Plane 的旋转属性。

```c
/*
 * 文件名：drm_rotation_set.c
 * 功能：使用 DRM atomic API 设置平面旋转属性
 * 编译：gcc -o drm_rotation_set drm_rotation_set.c -ldrm -lgbm
 *
 * 使用说明：
 *   ./drm_rotation_set <connector_id> <plane_id> <rotation>
 *   其中 rotation: 0=0°, 1=90°, 2=180°, 3=270°, 4=reflect-x, 5=reflect-y
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

/* DRM 旋转常量 */
#ifndef DRM_MODE_ROTATE_0
#define DRM_MODE_ROTATE_0   (1 << 0)
#define DRM_MODE_ROTATE_90  (1 << 1)
#define DRM_MODE_ROTATE_180 (1 << 2)
#define DRM_MODE_ROTATE_270 (1 << 3)
#define DRM_MODE_REFLECT_X  (1 << 4)
#define DRM_MODE_REFLECT_Y  (1 << 5)
#endif

static int get_property_id(int fd, uint32_t object_id,
                            uint32_t object_type, const char *name)
{
    drmModeObjectPropertiesPtr props;
    drmModePropertyPtr prop;
    int prop_id = -1;

    props = drmModeObjectGetProperties(fd, object_id, object_type);
    if (!props) {
        fprintf(stderr, "无法获取对象 0x%x 的属性\n", object_id);
        return -1;
    }

    for (uint32_t i = 0; i < props->count_props; i++) {
        prop = drmModeGetProperty(fd, props->props[i]);
        if (prop && strcmp(prop->name, name) == 0) {
            prop_id = prop->prop_id;
            drmModeFreeProperty(prop);
            break;
        }
        drmModeFreeProperty(prop);
    }

    drmModeFreeObjectProperties(props);
    return prop_id;
}

static int set_rotation_atomic(int fd, uint32_t plane_id,
                                unsigned int rotation_value)
{
    drmModeAtomicReqPtr req;
    int ret;
    int rotation_prop_id;

    /* 获取 Plane 的 rotation 属性 ID */
    rotation_prop_id = get_property_id(fd, plane_id,
                                        DRM_MODE_OBJECT_PLANE, "rotation");
    if (rotation_prop_id < 0) {
        fprintf(stderr, "未找到 rotation 属性\n");
        return -1;
    }

    /* 创建 atomic request */
    req = drmModeAtomicAlloc();
    if (!req) {
        fprintf(stderr, "无法分配 atomic request\n");
        return -1;
    }

    /* 添加旋转属性 */
    ret = drmModeAtomicAddProperty(req, plane_id,
                                    rotation_prop_id, rotation_value);
    if (ret < 0) {
        fprintf(stderr, "添加 rotation 属性失败: %d\n", ret);
        drmModeAtomicFree(req);
        return ret;
    }

    /* 提交 atomic commit
     * DRM_MODE_ATOMIC_ALLOW_MODESET: 允许模式设置
     * DRM_MODE_ATOMIC_NONBLOCK: 非阻塞提交
     */
    ret = drmModeAtomicCommit(fd, req,
                               DRM_MODE_ATOMIC_ALLOW_MODESET |
                               DRM_MODE_ATOMIC_NONBLOCK,
                               NULL);
    if (ret < 0) {
        fprintf(stderr, "atomic commit 失败: %d (%s)\n",
                ret, strerror(-ret));
        drmModeAtomicFree(req);
        return ret;
    }

    printf("成功设置 Plane %u 的旋转为 0x%x\n", plane_id, rotation_value);
    drmModeAtomicFree(req);
    return 0;
}

static void print_rotation_help(void)
{
    printf("支持的旋转值:\n");
    printf("  0  = ROTATE_0   (无旋转)\n");
    printf("  2  = ROTATE_90  (顺时针 90°)\n");
    printf("  4  = ROTATE_180 (旋转 180°)\n");
    printf("  8  = ROTATE_270 (顺时针 270°)\n");
    printf("  16 = REFLECT_X  (水平翻转)\n");
    printf("  32 = REFLECT_Y  (垂直翻转)\n");
    printf("  可以组合使用，例如 18 = ROTATE_90 | REFLECT_X\n");
}

int main(int argc, char *argv[])
{
    int fd;
    const char *card = "/dev/dri/card0";
    uint32_t plane_id;
    unsigned int rotation_value;

    if (argc < 3) {
        fprintf(stderr, "用法: %s <plane_id> <rotation_value>\n", argv[0]);
        print_rotation_help();
        return 1;
    }

    plane_id = strtoul(argv[1], NULL, 0);
    rotation_value = strtoul(argv[2], NULL, 0);

    /* 打开 DRM 设备 */
    fd = open(card, O_RDWR | O_CLOEXEC);
    if (fd < 0) {
        fprintf(stderr, "无法打开 %s: %s\n", card, strerror(errno));
        return 1;
    }

    /* 检查 DRM 设备是否支持 atomic */
    uint64_t atomic_cap = 0;
    if (drmGetCap(fd, DRM_CAP_CRTC_IN_VBLANK_EVENT, &atomic_cap) < 0) {
        fprintf(stderr, "无法获取 DRM capabilities\n");
        close(fd);
        return 1;
    }

    /* 设置旋转属性 */
    if (set_rotation_atomic(fd, plane_id, rotation_value) < 0) {
        close(fd);
        return 1;
    }

    close(fd);
    return 0;
}
```

编译和测试：

```bash
# 编译
gcc -o drm_rotation_set drm_rotation_set.c -ldrm -lgbm

# 查看可用的 Plane
modetest -M amdgpu -p

# 将 Plane 0 设置为 90° 旋转
./drm_rotation_set 0 2

# 将 Plane 0 设置为 180° 旋转（4 = 1 << 2）
./drm_rotation_set 0 4

# 组合测试：90° + 水平翻转（18 = 2 + 16）
./drm_rotation_set 0 18
```

### 实践 2：缩放模式测试脚本

```bash
#!/bin/bash
# 文件名：scale_mode_test.sh
# 功能：测试不同缩放模式的效果
# 用法：./scale_mode_test.sh [connector_id]

CONNECTOR="${1:-$(modetest -M amdgpu -c | head -2 | tail -1 | awk '{print $1}')}"
DRM_DEVICE="/dev/dri/card0"

echo "=========================================="
echo "  缩放模式测试工具"
echo "  连接器 ID: $CONNECTOR"
echo "=========================================="

# 获取 scaling_mode 属性 ID
get_prop_id() {
    local obj_id=$1
    local obj_type=$2
    local prop_name=$3

    modetest -M amdgpu -c 2>/dev/null | \
        awk -v obj="$obj_id" -v name="$prop_name" '
        /^[0-9]+/ { id=$1 }
        /scaling_mode/ && id==obj { print $1 }
        '
}

# 设置缩放模式
set_scaling_mode() {
    local mode="$1"
    echo ""
    echo "设置缩放模式为: $mode"

    # 使用 modetest 设置属性
    modetest -M amdgpu -w "${CONNECTOR}:scaling_mode:${mode}" 2>/dev/null

    if [ $? -eq 0 ]; then
        echo "  [成功] 缩放模式已设置为 $mode"
    else
        echo "  [失败] 无法设置缩放模式为 $mode"
    fi
}

# 测试所有缩放模式
echo ""
echo "测试 1: Full (全屏拉伸)"
set_scaling_mode "Full"
sleep 2

echo ""
echo "测试 2: Center (居中)"
set_scaling_mode "Center"
sleep 2

echo ""
echo "测试 3: Full aspect (保持宽高比)"
set_scaling_mode "Full aspect"
sleep 2

echo ""
echo "测试 4: None (不缩放)"
set_scaling_mode "None"
sleep 2

echo ""
echo "=========================================="
echo "  缩放模式测试完成"
echo "  原始分辨率与显示器原生分辨率的差异决定了缩放效果"
echo "=========================================="

# 显示当前缩放模式状态
echo ""
echo "当前缩放模式配置:"
cat /sys/class/drm/card0-*/scaling_mode 2>/dev/null || \
    echo "  (sysfs 接口不可用)"
```

### 实践 3：Viewport 与 Buffer 尺寸映射验证程序

```c
/*
 * 文件名：viewport_calc.c
 * 功能：计算旋转后 viewport 与 buffer 的尺寸映射关系
 * 编译：gcc -o viewport_calc viewport_calc.c -lm
 *
 * 说明：模拟 DRM 核心中的 viewport 计算逻辑
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>

#define DRM_MODE_ROTATE_0   (1 << 0)
#define DRM_MODE_ROTATE_90  (1 << 1)
#define DRM_MODE_ROTATE_180 (1 << 2)
#define DRM_MODE_ROTATE_270 (1 << 3)
#define DRM_MODE_REFLECT_X  (1 << 4)
#define DRM_MODE_REFLECT_Y  (1 << 5)

#define ROTATE_MASK (DRM_MODE_ROTATE_0 | DRM_MODE_ROTATE_90 | \
                     DRM_MODE_ROTATE_180 | DRM_MODE_ROTATE_270)

struct buffer_info {
    uint32_t width;      /* buffer 宽度（像素） */
    uint32_t height;     /* buffer 高度（像素） */
    uint32_t pitch;      /* 行跨度（字节） */
    uint32_t bpp;        /* 每像素位数 */
};

struct viewport_info {
    uint32_t src_x;      /* 源 X 偏移 */
    uint32_t src_y;      /* 源 Y 偏移 */
    uint32_t src_w;      /* 源宽度 */
    uint32_t src_h;      /* 源高度 */
    uint32_t dst_x;      /* 目标 X 偏移 */
    uint32_t dst_y;      /* 目标 Y 偏移 */
    uint32_t dst_w;      /* 目标宽度 */
    uint32_t dst_h;      /* 目标高度 */
};

/*
 * 计算旋转后的 viewport
 * 模拟 DRM 核心中 drm_atomic_helper_check_plane_state() 的行为
 */
void calculate_rotated_viewport(struct buffer_info *buf,
                                 struct viewport_info *vp,
                                 unsigned int rotation)
{
    /* 复制原始视口参数 */
    uint32_t orig_w = vp->src_w;
    uint32_t orig_h = vp->src_h;

    printf("\n=== Viewport 计算 ===\n");
    printf("原始 Buffer:   %u x %u (pitch=%u bytes, bpp=%u)\n",
           buf->width, buf->height, buf->pitch, buf->bpp);
    printf("原始 Viewport: src=(%u,%u) %ux%u, dst=(%u,%u) %ux%u\n",
           vp->src_x, vp->src_y, vp->src_w, vp->src_h,
           vp->dst_x, vp->dst_y, vp->dst_w, vp->dst_h);

    switch (rotation & ROTATE_MASK) {
    case DRM_MODE_ROTATE_0:
        printf("旋转角度: 0° (无旋转)\n");
        printf("Viewport 不变: src_w=%u, src_h=%u\n", vp->src_w, vp->src_h);
        break;

    case DRM_MODE_ROTATE_90:
        printf("旋转角度: 90° (顺时针)\n");
        /* 交换宽高 */
        vp->src_w = orig_h;
        vp->src_h = orig_w;
        printf("Viewport 交换: src_w=%u (原高), src_h=%u (原宽)\n",
               vp->src_w, vp->src_h);
        printf("注意: pitch (%u) 不变, 垂直读取 stride=%u\n",
               buf->pitch, buf->pitch / (buf->bpp / 8));
        break;

    case DRM_MODE_ROTATE_180:
        printf("旋转角度: 180°\n");
        printf("Viewport 不变: src_w=%u, src_h=%u\n", vp->src_w, vp->src_h);
        printf("但显存扫描方向反转\n");
        break;

    case DRM_MODE_ROTATE_270:
        printf("旋转角度: 270° (逆时针 90°)\n");
        /* 交换宽高 */
        vp->src_w = orig_h;
        vp->src_h = orig_w;
        printf("Viewport 交换: src_w=%u (原高), src_h=%u (原宽)\n",
               vp->src_w, vp->src_h);
        break;
    }

    /* 计算反射 */
    if (rotation & DRM_MODE_REFLECT_X) {
        printf("反射: 水平翻转 (X轴)\n");
        printf("  dst_x 方向反转\n");
    }
    if (rotation & DRM_MODE_REFLECT_Y) {
        printf("反射: 垂直翻转 (Y轴)\n");
        printf("  dst_y 方向反转\n");
    }

    /* 计算缩放比例 */
    float scale_x = (float)vp->dst_w / vp->src_w;
    float scale_y = (float)vp->dst_h / vp->src_h;

    printf("\n缩放比例:\n");
    printf("  水平: %.4f (目标 %u / 源 %u)\n", scale_x, vp->dst_w, vp->src_w);
    printf("  垂直: %.4f (目标 %u / 源 %u)\n", scale_y, vp->dst_h, vp->src_h);

    if (fabsf(scale_x - 1.0f) < 0.001f && fabsf(scale_y - 1.0f) < 0.001f) {
        printf("  无缩放 (1:1)\n");
    } else if (fabsf(scale_x - scale_y) < 0.001f) {
        printf("  均匀缩放 (等比例) x%.4f\n", scale_x);
    } else {
        printf("  非均匀缩放 (拉伸) X:%.4f Y:%.4f\n", scale_x, scale_y);
    }

    /* 计算显存带宽影响 */
    uint32_t rotation_overhead = 0;
    switch (rotation & ROTATE_MASK) {
    case DRM_MODE_ROTATE_0:   rotation_overhead = 0; break;
    case DRM_MODE_ROTATE_180: rotation_overhead = 10; break;
    case DRM_MODE_ROTATE_90:  rotation_overhead = 30; break;
    case DRM_MODE_ROTATE_270: rotation_overhead = 30; break;
    }

    size_t bandwidth_no_rot = (size_t)buf->width * buf->height *
                               (buf->bpp / 8) * 60; /* 假设 60fps */
    size_t bandwidth_with_rot = bandwidth_no_rot *
                                 (100 + rotation_overhead) / 100;

    printf("\n带宽估算 (60fps):\n");
    printf("  无旋转: %zu MB/s\n", bandwidth_no_rot / (1024 * 1024));
    printf("  旋转后: %zu MB/s (+%d%%)\n",
           bandwidth_with_rot / (1024 * 1024), rotation_overhead);
}

int main(int argc, char *argv[])
{
    struct buffer_info buf;
    struct viewport_info vp;
    unsigned int rotation;

    /* 默认值：1080p 全屏 */
    buf.width  = 1920;
    buf.height = 1080;
    buf.pitch  = 1920 * 4;  /* 32bpp = 4 bytes/pixel */
    buf.bpp    = 32;

    vp.src_x = 0;
    vp.src_y = 0;
    vp.src_w = 1920;
    vp.src_h = 1080;
    vp.dst_x = 0;
    vp.dst_y = 0;
    vp.dst_w = 1920;
    vp.dst_h = 1080;

    /* 测试各种旋转角度 */
    unsigned int test_rotations[] = {
        DRM_MODE_ROTATE_0,
        DRM_MODE_ROTATE_90,
        DRM_MODE_ROTATE_180,
        DRM_MODE_ROTATE_270,
        DRM_MODE_ROTATE_90 | DRM_MODE_REFLECT_X,
        DRM_MODE_ROTATE_0  | DRM_MODE_REFLECT_Y,
    };
    const char *rot_names[] = {
        "0°", "90°", "180°", "270°", "90°+ReflX", "ReflY"
    };

    for (int i = 0; i < 6; i++) {
        rotation = test_rotations[i];

        /* 重置视口（每次测试重新计算） */
        vp.src_w = 1920;
        vp.src_h = 1080;
        vp.dst_w = 1920;
        vp.dst_h = 1080;

        printf("\n────────────────────────────────────────────");
        printf("\n测试 %d: 旋转 %s\n", i + 1, rot_names[i]);
        calculate_rotated_viewport(&buf, &vp, rotation);
    }

    /* 测试缩放场景：720p 内容在 1080p 显示器上 */
    printf("\n\n============================================");
    printf("\n缩放场景测试：720p → 1080p");
    printf("\n============================================\n");

    buf.width  = 1280;
    buf.height = 720;
    buf.pitch  = 1280 * 4;
    vp.src_w = 1280;
    vp.src_h = 720;
    vp.dst_w = 1920;
    vp.dst_h = 1080;

    calculate_rotated_viewport(&buf, &vp, DRM_MODE_ROTATE_0);

    /* 测试竖屏场景：1080p 竖屏显示器 */
    printf("\n\n============================================");
    printf("\n竖屏场景测试：1920x1080 → 竖屏 1080x1920");
    printf("\n============================================\n");

    buf.width  = 1920;
    buf.height = 1080;
    buf.pitch  = 1920 * 4;
    vp.src_w = 1920;
    vp.src_h = 1080;
    vp.dst_w = 1080;
    vp.dst_h = 1920;

    calculate_rotated_viewport(&buf, &vp, DRM_MODE_ROTATE_90);

    return 0;
}
```

编译和运行：

```bash
gcc -o viewport_calc viewport_calc.c -lm -Wall -Wextra
./viewport_calc
```

### 实践 4：使用 bpftrace 追踪旋转/缩放寄存器编程

```bash
#!/bin/bash
# 文件名：trace_rotation.sh
# 功能：使用 bpftrace 追踪 AMDGPU 旋转/缩放的寄存器编程

# 检查 bpftrace 是否可用
if ! command -v bpftrace &> /dev/null; then
    echo "错误: 请先安装 bpftrace"
    exit 1
fi

echo "启动旋转/缩放寄存器编程追踪..."
echo "触发方式: 在另一个终端中设置旋转属性"
echo "例如: ./drm_rotation_set 0 2"
echo ""

# bpftrace 脚本
bpftrace -e '
kprobe:dpp20_set_rotation
{
    printf("=== 设置旋转 ===\n");
    printf("  PID: %d, 进程: %s\n", pid, comm);
    printf("  DPP: 0x%lx\n", arg0);
    printf("  旋转角度: %d\n", arg1 & 0x3);
    printf("  镜像X: %d\n", (arg1 >> 2) & 1);
    printf("  时间: %lld\n", nsecs);
}

kprobe:dpp20_set_scaler
{
    printf("=== 设置缩放 ===\n");
    printf("  PID: %d, 进程: %s\n", pid, comm);
    printf("  DPP: 0x%lx\n", arg0);
    printf("  缩放数据: scale_data={...}\n");
}

kprobe:fill_dc_plane_info_and_addr
{
    printf("=== 填充 Plane 信息 ===\n");
    printf("  PID: %d, 进程: %s\n", pid, comm);
}

kprobe:amdgpu_dm_atomic_commit_tail
{
    printf("=== Atomic Commit Tail ===\n");
    printf("  PID: %d, 进程: %s\n", pid, comm);
    printf("  --- 开始原子提交 ---\n");
}

kretprobe:amdgpu_dm_atomic_commit_tail
{
    printf("=== Atomic Commit 完成 ===\n");
    printf("  返回值: %d\n", retval);
    printf("  ---\n\n");
}
'
```

### 实践 5：完整的旋转测试套件

```bash
#!/bin/bash
# 文件名：rotation_test_suite.sh
# 功能：全面的旋转测试套件
# 用法：./rotation_test_suite.sh

DRM_DEVICE="/dev/dri/card0"
OUTPUT_DIR="/tmp/rotation_test_$(date +%Y%m%d_%H%M%S)"
PLANE_ID=""
CONNECTOR_ID=""

# 颜色输出
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

pass() { echo -e "${GREEN}[PASS]${NC} $1"; }
fail() { echo -e "${RED}[FAIL]${NC} $1"; }
info() { echo -e "${YELLOW}[INFO]${NC} $1"; }

# 获取第一个可用的 Plane 和 Connector
get_available_planes() {
    # 使用 modetest 获取 plane 信息
    PLANE_ID=$(modetest -M amdgpu -p 2>/dev/null | \
               grep -B 1 "rotation" | head -1 | awk '{print $1}')

    if [ -z "$PLANE_ID" ]; then
        # 尝试其他方式获取
        PLANE_ID=$(modetest -M amdgpu -p 2>/dev/null | \
                   grep "^[0-9]" | head -1 | awk '{print $1}')
    fi

    echo "$PLANE_ID"
}

get_available_connectors() {
    CONNECTOR_ID=$(modetest -M amdgpu -c 2>/dev/null | \
                   grep "^[0-9]" | head -1 | awk '{print $1}')
    echo "$CONNECTOR_ID"
}

# 检查是否支持 rotation 属性
check_rotation_support() {
    local plane_id=$1
    modetest -M amdgpu -p "$plane_id" 2>/dev/null | grep -q "rotation"
    return $?
}

# 测试单个旋转值
test_rotation() {
    local plane_id=$1
    local rotation_val=$2
    local rotation_name=$3

    info "测试旋转: $rotation_name (0x$rotation_val)"

    # 使用 modetest 设置 rotation 属性
    local result=$(modetest -M amdgpu -w "${plane_id}:rotation:${rotation_val}" 2>&1)

    if [ $? -eq 0 ]; then
        pass "旋转 $rotation_name 设置成功"
        return 0
    else
        fail "旋转 $rotation_name 设置失败: $result"
        return 1
    fi
}

# 测试旋转组合
test_rotation_combinations() {
    local plane_id=$1

    echo ""
    echo "=========================================="
    echo "  旋转组合测试"
    echo "  Plane ID: $plane_id"
    echo "=========================================="

    # 基本旋转
    test_rotation "$plane_id" "1" "ROTATE_0"
    sleep 1
    test_rotation "$plane_id" "2" "ROTATE_90"
    sleep 1
    test_rotation "$plane_id" "4" "ROTATE_180"
    sleep 1
    test_rotation "$plane_id" "8" "ROTATE_270"
    sleep 1

    # 反射
    test_rotation "$plane_id" "16" "REFLECT_X"
    sleep 1
    test_rotation "$plane_id" "32" "REFLECT_Y"
    sleep 1

    # 组合
    test_rotation "$plane_id" "18" "ROTATE_90 | REFLECT_X"
    sleep 1
    test_rotation "$plane_id" "36" "ROTATE_180 | REFLECT_Y"
    sleep 1

    # 恢复为 0°
    test_rotation "$plane_id" "1" "ROTATE_0 (恢复)"
}

# 带 CRC 校验的旋转测试
test_rotation_crc() {
    local plane_id=$1
    local crtc_id=$2

    echo ""
    echo "=========================================="
    echo "  旋转 CRC 校验测试"
    echo "=========================================="

    local crc_file="/sys/class/drm/card0/crtc-${crtc_id}/crc/data"

    if [ ! -f "$crc_file" ]; then
        info "CRC 接口不可用，跳过 CRC 校验"
        return
    fi

    # 启用 CRC
    echo "auto" > "/sys/class/drm/card0/crtc-${crtc_id}/crc/control" 2>/dev/null

    # 获取各个旋转角度的 CRC
    for rot_name in "1:0°" "2:90°" "4:180°" "8:270°"; do
        local rot_val="${rot_name%%:*}"
        local rot_label="${rot_name##*:}"

        modetest -M amdgpu -w "${plane_id}:rotation:${rot_val}" 2>/dev/null
        sleep 0.5

        local crc=$(head -1 "$crc_file" 2>/dev/null)
        info "旋转 $rot_label: CRC = $crc"
    done

    # 恢复
    modetest -M amdgpu -w "${plane_id}:rotation:1" 2>/dev/null
    echo "disable" > "/sys/class/drm/card0/crtc-${crtc_id}/crc/control" 2>/dev/null
}

# 性能基准测试
benchmark_rotation_perf() {
    local plane_id=$1

    echo ""
    echo "=========================================="
    echo "  旋转性能基准测试"
    echo "=========================================="

    # 测试每个旋转角度下的帧率
    for rot_val in 1 2 4 8; do
        modetest -M amdgpu -w "${plane_id}:rotation:${rot_val}" 2>/dev/null

        # 使用 vblank 计数器估算帧率
        local vblank_start=$(cat /sys/class/drm/card0/crtc-0/vblank_count 2>/dev/null || echo 0)
        sleep 2
        local vblank_end=$(cat /sys/class/drm/card0/crtc-0/vblank_count 2>/dev/null || echo 0)
        local fps=$(( (vblank_end - vblank_start) / 2 ))

        info "旋转值 0x${rot_val}: 约 ${fps} fps"
    done

    # 恢复
    modetest -M amdgpu -w "${plane_id}:rotation:1" 2>/dev/null
}

# 主测试流程
main() {
    mkdir -p "$OUTPUT_DIR"

    echo "旋转测试套件"
    echo "输出目录: $OUTPUT_DIR"
    echo "开始时间: $(date)"
    echo ""

    # 获取硬件信息
    local hw_info=$(cat /sys/class/drm/card0/device/vendor 2>/dev/null)
    info "GPU 厂商 ID: $hw_info"
    info "驱动: $(cat /sys/class/drm/card0/device/driver/module/drivers/pci:amdgpu/description 2>/dev/null || echo 'amdgpu')"

    # 获取 Plane 和 Connector
    PLANE_ID=$(get_available_planes)
    CONNECTOR_ID=$(get_available_connectors)

    if [ -z "$PLANE_ID" ]; then
        fail "未找到可用的 Plane"
        exit 1
    fi

    info "使用 Plane ID: $PLANE_ID"
    info "使用 Connector ID: $CONNECTOR_ID"

    # 检查旋转支持
    if ! check_rotation_support "$PLANE_ID"; then
        fail "该 Plane 不支持旋转属性"
        exit 1
    fi

    # 运行测试
    test_rotation_combinations "$PLANE_ID"
    test_rotation_crc "$PLANE_ID" "0"
    benchmark_rotation_perf "$PLANE_ID"

    # 生成报告
    echo ""
    echo "=========================================="
    echo "  测试报告"
    echo "=========================================="
    echo "日期: $(date)"
    echo "GPU: $(lspci | grep -i amd | grep -i vga | head -1)"
    echo "Plane ID: $PLANE_ID"
    echo "支持的旋转: $(modetest -M amdgpu -p $PLANE_ID 2>/dev/null | \
                    grep -A 10 rotation | head -11)"

    # 保存测试结果
    modetest -M amdgpu -p "$PLANE_ID" > "$OUTPUT_DIR/plane_info.txt" 2>&1
    modetest -M amdgpu -c "$CONNECTOR_ID" > "$OUTPUT_DIR/connector_info.txt" 2>&1

    echo ""
    echo "详细信息已保存到: $OUTPUT_DIR"
}

main
```

### 实践 6：Wayland 合成器下的旋转配置

```bash
#!/bin/bash
# 文件名：wl_rotation_config.sh
# 功能：在 Wayland 环境下配置显示器旋转
# 适用于：GNOME (Mutter), KDE (KWin), Sway

echo "Wayland 显示器旋转配置工具"
echo ""

detect_compositor() {
    if [ -n "$WAYLAND_DISPLAY" ]; then
        case "$XDG_SESSION_DESKTOP" in
            *GNOME*|*gnome*)
                echo "gnome"
                ;;
            *KDE*|*kde*|*plasma*)
                echo "kde"
                ;;
            *sway*|*Sway*|*SWAY*)
                echo "sway"
                ;;
            *Hyprland*|*hyprland*)
                echo "hyprland"
                ;;
            *)
                echo "unknown"
                ;;
        esac
    else
        echo "x11"
    fi
}

case $(detect_compositor) in
    gnome)
        echo "检测到 GNOME (Mutter)"
        echo "使用 gnome-control-center 或 gsettings 配置旋转:"
        echo ""
        echo "  列出显示器:"
        echo "    gnome-control-center display"
        echo ""
        echo "  命令行设置旋转 (方式1):"
        echo "    gsettings set org.gnome.mutter check-rotated-display false"
        echo ""
        echo "  命令行设置旋转 (方式2):"
        echo "    # 使用 wlr-randr (如果安装了 wlr-randr 兼容层)"
        echo "    wlr-randr --output eDP-1 --transform 90"
        echo ""
        echo "  当前显示器配置:"
        echo "    $(gnome-control-center display 2>/dev/null || echo '请手动打开设置')"
        ;;

    kde)
        echo "检测到 KDE (KWin)"
        echo "使用 kscreen-doctor 配置旋转:"
        echo ""
        echo "  列出显示器:"
        echo "    kscreen-doctor -o"
        echo ""
        echo "  设置旋转 (eDP-1 为内置显示器):"
        echo "    kscreen-doctor output.eDP-1.rotation.90"
        echo ""
        echo "  支持的旋转值: 0, 90, 180, 270"
        ;;

    sway)
        echo "检测到 Sway (wlroots)"
        echo "使用 swaymsg 配置旋转:"
        echo ""
        echo "  列出输出:"
        echo "    swaymsg -t get_outputs"
        echo ""
        echo "  设置旋转:"
        echo "    swaymsg output eDP-1 transform 90"
        echo ""
        echo "  支持的 transform 值:"
        echo "    0, 90, 180, 270 (旋转)"
        echo "    flipped, flipped-90, flipped-180, flipped-270 (旋转+翻转)"
        ;;

    hyprland)
        echo "检测到 Hyprland"
        echo "编辑 ~/.config/hypr/hyprland.conf:"
        echo ""
        echo "  monitor=eDP-1,1920x1080@60,0x0,1"
        echo "  # transform 参数:"
        echo "  # 0 = 正常, 1 = 90°, 2 = 180°, 3 = 270°"
        echo "  # 4 = 翻转, 5 = 翻转90°, 6 = 翻转180°, 7 = 翻转270°"
        echo ""
        echo "  或使用 hyprctl 实时设置:"
        echo "    hyprctl keyword monitor eDP-1,1920x1080@60,0x0,1,transform,1"
        ;;

    x11)
        echo "检测到 X11"
        echo "使用 xrandr 配置旋转:"
        echo ""
        echo "  列出输出:"
        echo "    xrandr --query"
        echo ""
        echo "  设置旋转:"
        echo "    xrandr --output eDP-1 --rotate left"
        echo ""
        echo "  支持的旋转值: normal, left, right, inverted"
        ;;

    *)
        echo "未知合成器，尝试通用方法:"
        echo "  1. wlr-randr --output eDP-1 --transform 90"
        echo "  2. xrandr --output eDP-1 --rotate left"
        ;;
esac

echo ""
echo "=========================================="
echo "  提示：旋转显示器后，触摸输入也需要对应旋转"
echo "  触摸旋转配置："
echo "    libinput 通常自动处理"
echo "    如需要手动配置："
echo "    xinput set-prop <device> 'Coordinate Transformation Matrix' ..."
echo "=========================================="

## 案例分析

### 案例 1：竖屏显示器配置中的旋转问题

**问题描述**：
用户将一台物理竖屏显示器连接到 AMDGPU 系统，但显示内容仍然以横屏方式呈现。显示器 EDID 中没有包含旋转信息，导致系统无法自动识别显示器的物理方向。

**分析过程**：

```
1. 检查显示器物理安装方向：物理竖屏（旋转 90°）
2. 检查系统检测到的方向：
   $ modetest -M amdgpu -c
   Connector 0: DP-1
     modes: 1920x1080, 1680x1050, 1280x1024, ...
   → 系统报告横屏分辨率，说明 EDID 未包含旋转信息

3. 检查 Plane 支持的旋转：
   $ drm_info | grep -A 10 rotation
   → rotation: rotate-0, rotate-90, rotate-180, rotate-270, reflect-x, reflect-y
   → 硬件支持 90° 旋转

4. 解决方案：使用 atomic API 设置旋转
   $ ./drm_rotation_set 0 2  # 设置 90° 旋转
```

**解决步骤**：

```bash
#!/bin/bash
# 竖屏配置脚本：portrait_setup.sh

CONNECTOR=$(modetest -M amdgpu -c | grep "^[0-9]" | head -1 | awk '{print $1}')
PLANE=$(modetest -M amdgpu -p | grep "^[0-9]" | head -1 | awk '{print $1}')

echo "配置竖屏显示..."
echo "连接器: $CONNECTOR, 平面: $PLANE"

# 1. 首先设置缩放模式（保持宽高比）
modetest -M amdgpu -w "${CONNECTOR}:scaling_mode:Full aspect"

# 2. 设置 90° 旋转
modetest -M amdgpu -w "${PLANE}:rotation:2"

# 3. 验证设置
echo "当前旋转: $(modetest -M amdgpu -p $PLANE | grep rotation | head -1)"

echo "竖屏配置完成"
echo "注意：如果触摸输入偏移，请配置 libinput"
```

**经验总结**：
- 硬件旋转比软件旋转更高效
- 旋转后需要同步配置触摸输入映射
- 部分桌面环境（如 GNOME）会自动处理显示器旋转

### 案例 2：4K 显示器缩放性能瓶颈分析

**问题描述**：
用户在 4K 显示器上运行 1080p 游戏，发现帧率低于预期。怀疑是缩放导致的性能问题。

**分析过程**：

```
硬件配置：
  - GPU: AMD Radeon RX 6800 (Navi 21, DCN 3.0)
  - 显示器: 3840x2160 @ 60Hz
  - 游戏分辨率: 1920x1080

缩放路径分析：
  1. 游戏渲染 1920x1080 帧
  2. 驱动通过 DCN 硬件 scaler 放大到 3840x2160
  3. 缩放比例：2x (水平和垂直)

性能基准测试：
  无缩放（直接渲染 4K）：45 fps
  硬件缩放（1080p → 4K）：58 fps
  软件缩放（GPU shader）：52 fps

结论：硬件缩放性能最佳
```

```bash
#!/bin/bash
# 缩放性能测试脚本：scale_perf_test.sh
# 测试不同缩放配置下的帧率

test_scale_mode() {
    local mode=$1
    local resolution=$2

    echo "测试: 分辨率 $resolution, 缩放模式 $mode"

    # 设置分辨率和缩放
    modetest -M amdgpu -w 0:scaling_mode:"${mode}" 2>/dev/null

    # 采样 vblank 计数
    local v0=$(cat /sys/class/drm/card0/crtc-0/vblank_count 2>/dev/null)
    sleep 3
    local v1=$(cat /sys/class/drm/card0/crtc-0/vblank_count 2>/dev/null)

    local fps=$(( (v1 - v0) / 3 ))
    echo "  平均帧率: ${fps} fps"
}

# 测试各种缩放配置
echo "缩放性能测试"
echo "=============="

# 原生 4K（无缩放基准）
test_scale_mode "None" "3840x2160"

# 1080p 缩放
test_scale_mode "Full" "1920x1080"
test_scale_mode "Full aspect" "1920x1080"

echo "=============="
echo "建议: 使用 Full aspect 模式以获得最佳画质和性能平衡"
```

**经验总结**：
- 硬件缩放由 DCN 硬件完成，几乎零性能开销
- 缩放质量与性能之间存在权衡
- DCN 3.0+ 的缩放质量显著优于早期版本

### 案例 3：MST 多屏配置中的缩放和旋转问题

**问题描述**：
用户通过 DP MST（Multi-Stream Transport）连接了三个显示器，其中一个为竖屏。配置后竖屏显示器显示异常。

**根因分析**：

```
拓扑结构：
  GPU ── DP ── MST Hub ─┬─ 显示器 1 (横屏, 1920x1080)
                         ├─ 显示器 2 (竖屏, 1080x1920, 需要旋转)
                         └─ 显示器 3 (横屏, 3840x2160)

问题：
  显示器 2 的旋转属性与 MST 的虚拟连接器关联，
  需要确保旋转属性设置到正确的 Plane 和 CRTC。

调试步骤：
  1. 查看 MST 连接器拓扑：
     $ cat /sys/kernel/debug/dri/0/amdgpu_mst_topology

  2. 确认每个虚拟连接器的 ID：
     $ modetest -M amdgpu -c | grep "MST"

  3. 为竖屏显示器设置旋转：
     $ modetest -M amdgpu -w <mst_connector_id>:rotation:2
```

```bash
#!/bin/bash
# MST 配置脚本：mst_portrait_setup.sh

echo "MST 竖屏配置"
echo "============="

# 列出所有 MST 连接器
echo "MST 连接器列表:"
modetest -M amdgpu -c | while read line; do
    if echo "$line" | grep -q "MST"; then
        echo "  $line"
    fi
done

# 用户指定要配置的连接器
read -p "请输入要配置为竖屏的连接器 ID: " conn_id

# 找到与该连接器关联的 Plane
plane_id=$(modetest -M amdgpu -p | grep -B 5 "connector $conn_id" | \
           grep "^[0-9]" | head -1 | awk '{print $1}')

if [ -n "$plane_id" ]; then
    echo "设置连接器 $conn_id 的关联平面 $plane_id 为 90° 旋转"
    modetest -M amdgpu -w "${plane_id}:rotation:2"
    echo "配置完成"
else
    echo "未找到与连接器 $conn_id 关联的平面"
    echo "请手动检查: modetest -M amdgpu -p"
fi
```

**经验总结**：
- MST 拓扑中每个虚拟连接器使用独立的 CRTC 和 Plane
- 旋转属性在 Plane 级别设置，需确保设置到正确的 Plane
- MST 配置变更通常需要 modeset

## 相关链接

1. DRM 旋转/反射属性内核定义
   - include/uapi/drm/drm_mode.h
   - https://elixir.bootlin.com/linux/latest/source/include/uapi/drm/drm_mode.h

2. drm_plane_create_rotation_property() API
   - drivers/gpu/drm/drm_plane.c
   - https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/drm_plane.c

3. AMDGPU DC Plane 初始化（旋转支持声明）
   - drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_plane.c
   - https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_plane.c

4. DCN 2.0 DPP Scaler 寄存器编程
   - drivers/gpu/drm/amd/display/dc/dcn20/dcn20_dpp_scl.c
   - https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/dcn20/dcn20_dpp_scl.c

5. DCN 3.0 DPP Scaler 扩展
   - drivers/gpu/drm/amd/display/dc/dcn30/dcn30_dpp.c
   - https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/dcn30/dcn30_dpp.c

6. AMDGPU DC 层旋转参数传递
   - drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c
   - fill_dc_plane_info_and_addr() 函数

7. IGT kms_rotation_crc 测试
   - tests/kms_rotation_crc.c
   - https://gitlab.freedesktop.org/drm/igt-gpu-tools

8. Weston DRM 后端平面旋转
   - libweston/backend-drm/plane.c
   - https://gitlab.freedesktop.org/wayland/weston

9. DRM 缩放模式属性文档
   - https://dri.freedesktop.org/docs/drm/gpu/drm-kms.html

10. AMD FSR 技术白皮书
    - https://www.amd.com/en/technologies/fidelityfx-super-resolution

11. libdrm 用户空间库 API
    - https://dri.freedesktop.org/libdrm/

12. drm_info 工具
    - https://gitlab.freedesktop.org/emersion/drm-info

## 今日小结

### 核心概念回顾

Day 352 深入讲解了 DRM 子系统中旋转（Rotation）和缩放（Scaling）的完整实现：

1. **DRM 旋转/反射属性体系**：
   - 6 个旋转/反射标志：ROTATE_0/90/180/270 + REFLECT_X/Y
   - 位掩码组合，支持复杂变换
   - 在 Plane 级别通过 drm_plane_create_rotation_property() 绑定

2. **缩放模式**：
   - 4 种标准模式：None、Fullscreen、Center、Aspect
   - 通过 Connector 的 scaling_mode 属性控制
   - Overscan/Underscan 用于 HDMI 显示兼容性

3. **AMDGPU DCN 硬件实现**：
   - 旋转/缩放在 DPP_SCL 模块中完成
   - 不同 DCN 版本的旋转支持能力不同
   - 硬件旋转零性能开销
   - Viewport 与 Buffer 尺寸映射是旋转实现的关键

4. **性能影响**：
   - 旋转 90°/270° 的显存带宽开销约 20-50%
   - 缩放开销取决于滤波器质量（3-30%）
   - 硬件 vs 软件回退的性能差距可达 10-100 倍

5. **调试与测试**：
   - IGT kms_rotation_crc 提供全面的旋转测试
   - modetest、drm_info 可用于查看和设置旋转属性
   - bpftrace 可追踪旋转寄存器编程

### 关键技术要点

```
旋转/缩放调试检查清单：
□ 硬件是否支持请求的旋转角度？
□ Plane 的 rotation 属性是否已正确设置？
□ Atomic commit 是否成功？
□ Viewport 尺寸是否随旋转正确交换？
□ 缩放模式是否与预期一致？
□ 是否使用了硬件缩放（而非软件回退）？
□ 触摸/输入设备是否已同步配置？
```

### 常见误区

1. **误区**：所有 AMDGPU 都支持 90° 旋转。
   **事实**：DCN 1.0 (Raven) 仅支持 180° 旋转，90°/270° 需要 DCN 2.0+。

2. **误区**：旋转不影响性能。
   **事实**：90°/270° 旋转会导致显存访问模式变为跨步访问，带宽开销约 20-50%。

3. **误区**：缩放模式设置立即生效。
   **事实**：缩放模式是 Connector 属性，部分模式变更需要 modeset（模式重设）。

## 扩展思考

### 1. 硬件旋转的局限性

当前 DCN 硬件的旋转虽然高效，但仍有一些局限性：
- 旋转不支持任意角度，仅支持 90° 的倍数
- 旋转与某些格式组合可能不支持（如 YUV 420 + 90° 旋转）
- 多平面组合时的旋转行为需要特别关注

未来的 DCN 版本可能会支持：
- 任意角度旋转（通过更灵活的 scaler）
- 每平面独立旋转的改进
- 旋转 + HDR 的完整支持

### 2. 缩放技术的未来发展方向

```
缩放技术演进路线：
  ┌────────────────────────────────────────────────────┐
  │                        未来                         │
  │         ┌─ AI 增强缩放（基于神经网络）              │
  │         │  └─ 实时 AI 超分辨率                     │
  │         │                                          │
  │         ├─ FSR 3/4 系列                            │
  │         │  └─ 帧生成 + 超分辨率                     │
  │         │                                          │
  │   现在   ├─ FSR 2/DPC                               │
  │         │  └─ 时间累积超分辨率                       │
  │         │                                          │
  │         ├─ 传统硬件缩放                              │
  │         │  └─ 双线性/双三次/Lanczos                │
  │         │                                          │
  │    过去  └─ 最近邻缩放                              │
  │            └─ 像素化效果                            │
  └────────────────────────────────────────────────────┘
```

### 3. 与用户空间合成器的协作

旋转/缩放功能在合成器（Compositor）中的发展战略：

1. **智能旋转选择**：合成器自动检测显示器物理方向，选择最有效的旋转方式
2. **统一缩放接口**：为所有窗口管理器提供一致的缩放配置 API
3. **性能感知调度**：根据当前负载动态选择硬件或软件旋转

### 4. 实践建议

对于 AMDGPU 驱动测试开发工程师：

1. **测试重点**：
   - 所有旋转角度在每种 DCN 版本上的表现
   - 旋转 + 缩放的组合测试
   - 边界条件：0px 偏移、最大分辨率、非标准比例
   - 长时间稳定性测试（旋转状态下的 suspend/resume）

2. **性能基准**：
   - 建立旋转/缩放的性能基准测试套件
   - 测量不同旋转角度下的帧率和延迟
   - 对比硬件旋转和软件回退的性能差异

3. **兼容性测试**：
   - 不同显示器（EDID 含/不含旋转信息）
   - 不同桌面环境（GNOME、KDE、Sway）
   - 不同连接方式（DP、HDMI、USB-C）

### 5. 与其他子系统的关联

旋转/缩放与其他显示子系统的交互：

```
                     ┌──────────┐
                     │ 电源管理  │ ← 旋转/缩放影响功耗
                     └─────┬────┘
                           │
┌──────────┐         ┌─────▼────┐         ┌──────────┐
│ 颜色管理  │ ←──→    │ 旋转/缩放  │ ←──→    │ 合成器    │
│ (HDR/ICC) │         │ (KMS)    │         │ (Wayland) │
└──────────┘         └─────┬────┘         └──────────┘
                           │
                     ┌─────▼────┐
                     │ 输入系统  │ ← 旋转后需同步触摸映射
                     └──────────┘
```

### 6. 开放问题

1. **FSR 在驱动层面的集成程度**：未来是否会在 DC 层原生支持 FSR 处理？
2. **多 GPU 配置下的旋转**：Crossfire/mGPU 配置中旋转行为如何协调？
3. **虚拟化环境中的旋转**：SR-IOV VF 中的旋转能力如何提供给虚拟机？
4. **旋转状态下的颜色精度**：旋转操作是否影响颜色精度或 HDR 元数据传递？

这些问题值得在后续的学习中进一步探索。Day 353 将进入 Panel Self Refresh (PSR) 节电技术的学习，这是显示电源管理的重要一环。