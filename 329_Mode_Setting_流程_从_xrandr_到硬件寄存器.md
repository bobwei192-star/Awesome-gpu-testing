# 第329天：Mode Setting 流程：从 xrandr 到硬件寄存器

## 学习目标

1. 理解从用户空间 xrandr 命令到硬件寄存器的完整 mode 设置流程
2. 掌握 xrandr → libXrandr → libdrm → DRM ioctl → DRM core → AMDGPU DM → DC → DCN 的逐层调用链
3. 学会使用 strace、modetest、register dump 等工具跟踪 mode 设置的全过程
4. 理解传统 (legacy) 与原子 (atomic) 两种 mode 设置路径的差异
5. 掌握每个层级的关键数据结构转换和适配逻辑

## 1 知识详解

### 1.1 整体流程概览

用户执行 `xrandr --output HDMI-A-1 --mode 1920x1080` 时，完整的调用链如下：

```
┌─────────────────────────────────────────────────────────────┐
│                    用户空间 (Userspace)                       │
│                                                             │
│  xrandr 命令行工具                                           │
│    ↓ libXrandr (X11 扩展库)                                  │
│    ↓ libdrm (DRM 用户空间库)                                 │
│    ↓ ioctl() 系统调用                                        │
├─────────────────────────────────────────────────────────────┤
│                    内核空间 (Kernel)                         │
│                                                             │
│  DRM core (drm_ioctl)                                       │
│    ↓ drm_mode_setcrtc                                       │
│    ↓ drm_atomic_commit (原子路径)                            │
│    ↓                                                         │
│  AMDGPU DM (Display Manager)                                 │
│    ↓ amdgpu_dm_setcrtc → amdgpu_dm_commit_planes            │
│    ↓ dm_connector_mode_to_dc_timing                         │
│    ↓                                                         │
│  AMDGPU DC (Display Core)                                    │
│    ↓ dc_commit_streams → dc_commit_state                    │
│    ↓ resource_build_scaling_params                          │
│    ↓                                                         │
│  DCN 硬件层                                                  │
│    ↓ dcn20_set_otg_timing → REG_SET()                      │
│    ↓ OTG → DPP → MPC → OPP → OPTC 硬件流水线                │
│    ↓                                                         │
│  GPU 硬件寄存器                                              │
│    OTG_V_TOTAL, H_TOTAL, OTG_V_SYNC_A_END ...               │
└─────────────────────────────────────────────────────────────┘
```

每个层次的职责：

| 层次 | 模块 | 职责 |
|------|------|------|
| 1 | xrandr | 解析用户参数，构造 mode 请求 |
| 2 | libXrandr | X11 扩展，将请求转发到 X Server 或直接通过 DRM 接口 |
| 3 | libdrm | 封装 DRM ioctl，提供 drmModeSetCrtc 等 API |
| 4 | DRM core | ioctl 分发，参数校验，crtc/connector/fb 状态管理 |
| 5 | AMDGPU DM | 将 DRM 对象映射到 DC 对象，验证 mode 兼容性 |
| 6 | AMDGPU DC | 抽象显示流水线，构建 stream 状态 |
| 7 | DCN | 硬件寄存器编程，配置 OTG/DPP/MPC/OPP |
| 8 | GPU 硬件 | 按寄存器配置输出 video timing |

以下各节将逐层深入分析每一级的关键数据结构、函数调用和参数转换。

---

### 1.2 xrandr 用户空间层

`xrandr` 是 X Resize and Rotate 扩展的命令行工具。当用户执行 `xrandr --output HDMI-A-1 --mode 1920x1080`，内部处理流程如下：

#### 1.2.1 参数解析

xrandr 使用 `getopt` 解析命令行参数，核心数据结构：

```c
// xrandr.c (xrandr 源码)
struct mode_info {
    char name[256];
    int width, height;
    Bool preferred;
    RRMode mode_id;   // X Resource 中的 mode ID
};

struct output_info {
    char name[256];
    int crtc_id;
    int mm_width, mm_height;
    Bool connected;
    Bool is_primary;
    int nmodes;
    struct mode_info *modes;
    Rotation rotation;
};
```

#### 1.2.2 XRRSetCrtcConfig

xrandr 通过 XRRSetCrtcConfig X11 扩展函数设置 mode：

```c
// xrandr.c main() 处理 --mode 参数后
if (set_mode) {
    Status s = XRRSetCrtcConfig(
        dpy,        // X Display
        output->crtc_info->crtc,
        CurrentTime,
        output->x, output->y,
        mode_info->mode_id,   // RRMode
        rotation,
        output->crtc_info->noutput,
        output->crtc_info->outputs
    );
    if (s != RRSetConfigSuccess) {
        fprintf(stderr, "Failed to set mode\n");
    }
}
```

#### 1.2.3 X Server 到 DRM 的转发

在 X Server 内部（xf86-video-amdgpu DDX 驱动），XRRSetCrtcConfig 最终调用 DRM ioctl：

```c
// xf86-video-amdgpu/src/amdgpu_kms.c
Bool AMDGPUModeSet(ScrnInfoPtr scrn, DisplayModePtr mode) {
    struct amdgpu_encoder *enc;
    drmModeCrtcPtr crtc;
    uint32_t fb_id;
    drmModeModeInfo drm_mode;

    // 将 X 的 DisplayMode 转换为 DRM drmModeModeInfo
    xf86_to_drm_mode(mode, &drm_mode);

    // 获取 framebuffer
    fb_id = amdgpu_get_fb(scrn);

    // 调用 libdrm 的 DRM ioctl
    crtc = drmModeGetCrtc(pAMDGPUEnt->fd, enc->crtc_id);
    drmModeSetCrtc(pAMDGPUEnt->fd, crtc->crtc_id, fb_id,
                   0, 0, &conn_id, 1, &drm_mode);
}
```

---

### 1.3 libdrm 层

libdrm 封装了内核 DRM 的 ioctl 接口，提供类型安全的用户空间 API。

#### 1.3.1 drmModeSetCrtc

```c
// xf86drmMode.c (libdrm)
int drmModeSetCrtc(int fd, uint32_t crtc_id, uint32_t buffer_id,
                  uint32_t x, uint32_t y, uint32_t *connectors,
                  int count, drmModeModeInfoPtr mode)
{
    struct drm_mode_crtc crtc;

    memcpy(&crtc.mode, mode, sizeof(struct drm_mode_modeinfo));
    crtc.crtc_id = crtc_id;
    crtc.fb_id = buffer_id;
    crtc.x = x;
    crtc.y = y;
    crtc.set_connectors_ptr = (uintptr_t)connectors;
    crtc.count_connectors = count;

    // 发送 DRM_IOCTL_MODE_SETCRTC
    int ret = drmIoctl(fd, DRM_IOCTL_MODE_SETCRTC, &crtc);
    return ret;
}
```

#### 1.3.2 drmModeAtomicCommit

在原子模式设置路径中，使用 drmModeAtomicCommit：

```c
// xf86drmMode.c
int drmModeAtomicCommit(int fd, drmModeAtomicReqPtr req,
                        uint32_t flags, void *user_data)
{
    struct drm_mode_atomic atomic;

    atomic.flags = flags;           // DRM_MODE_ATOMIC_ALLOW_MODESET
    atomic.count_objs = req->cursor;
    atomic.objs_ptr = (uintptr_t)req->objs;
    atomic.count_props_ptr = (uintptr_t)req->count_props;
    atomic.props_ptr = (uintptr_t)req->props;
    atomic.prop_values_ptr = (uintptr_t)req->prop_values;
    atomic.user_data = (uint64_t)(uintptr_t)user_data;

    return drmIoctl(fd, DRM_IOCTL_MODE_ATOMIC, &atomic);
}
```

#### 1.3.3 关键 DRM ioctl 命令

| ioctl 命令 | 功能 | 传入结构体 | 路径 |
|-----------|------|-----------|------|
| DRM_IOCTL_MODE_SETCRTC | 设置 CRTC mode | drm_mode_crtc | 传统 (legacy) |
| DRM_IOCTL_MODE_ATOMIC | 原子模式设置 | drm_mode_atomic | 原子 (atomic) |
| DRM_IOCTL_MODE_SETPROPERTY | 设置属性 | drm_mode_getproperty | 传统 |
| DRM_IOCTL_MODE_PAGE_FLIP | 页面翻转 | drm_mode_crtc_page_flip | 传统 |
| DRM_IOCTL_MODE_CURSOR | 设置光标 | drm_mode_cursor | 传统 |

---

### 1.4 DRM core ioctl 入口

内核收到 ioctl 后，调用 drm_ioctl 分发到对应的处理函数。

#### 1.4.1 ioctl 分发表

```c
// drivers/gpu/drm/drm_ioctl.c
DRM_IOCTL_DEF(DRM_IOCTL_MODE_SETCRTC, drm_mode_setcrtc, DRM_UNLOCKED),
DRM_IOCTL_DEF(DRM_IOCTL_MODE_ATOMIC, drm_mode_atomic_ioctl, DRM_UNLOCKED),
DRM_IOCTL_DEF(DRM_IOCTL_MODE_PAGE_FLIP, drm_mode_page_flip_ioctl, DRM_UNLOCKED),
DRM_IOCTL_DEF(DRM_IOCTL_MODE_SETPROPERTY, drm_mode_obj_set_property_ioctl, DRM_UNLOCKED),
```

#### 1.4.2 drm_mode_setcrtc 实现

```c
// drivers/gpu/drm/drm_crtc.c
int drm_mode_setcrtc(struct drm_device *dev, void *data,
                     struct drm_file *file_priv)
{
    struct drm_mode_crtc *crtc_req = data;
    struct drm_crtc *crtc;
    struct drm_connector **connector_set = NULL;
    struct drm_framebuffer *fb = NULL;
    struct drm_display_mode *mode = NULL;
    int ret;

    // 1. 查找 CRTC 对象
    crtc = drm_crtc_find(dev, file_priv, crtc_req->crtc_id);
    if (!crtc)
        return -ENOENT;

    // 2. 查找 Framebuffer（如果提供了 fb_id）
    if (crtc_req->fb_id) {
        fb = drm_framebuffer_lookup(dev, file_priv, crtc_req->fb_id);
        if (!fb)
            return -ENOENT;
    }

    // 3. 从用户空间复制 mode 信息
    if (crtc_req->mode.width || crtc_req->mode.height) {
        mode = drm_mode_create(dev);
        if (!mode) {
            ret = -ENOMEM;
            goto out;
        }
        // 将 drm_mode_modeinfo (用户空间) 转换为 drm_display_mode (内核)
        ret = drm_mode_convert_umode(dev, mode, &crtc_req->mode);
        if (ret) {
            drm_mode_destroy(dev, mode);
            mode = NULL;
            goto out;
        }
    }

    // 4. 复制 connector 列表
    if (crtc_req->count_connectors > 0 && crtc_req->set_connectors_ptr) {
        connector_set = kcalloc(crtc_req->count_connectors,
                                sizeof(*connector_set), GFP_KERNEL);
        // 从用户空间复制 connector ID 列表
        // ...
    }

    // 5. 调用 CRTC 的 set_config 回调
    //    这将最终到达 AMDGPU 驱动
    crtc->funcs->set_config(&set, &ret);

out:
    // 清理
    if (mode)
        drm_mode_destroy(dev, mode);
    if (fb)
        drm_framebuffer_put(fb);
    kfree(connector_set);
    return ret;
}
```

#### 1.4.3 drm_mode_convert_umode 详细转换

```c
// drivers/gpu/drm/drm_modes.c
int drm_mode_convert_umode(struct drm_device *dev,
                           struct drm_display_mode *out,
                           const struct drm_mode_modeinfo *in)
{
    // clock: kHz 值直接复制
    out->clock = in->clock;
    out->hdisplay = in->hdisplay;
    out->hsync_start = in->hsync_start;
    out->hsync_end = in->hsync_end;
    out->htotal = in->htotal;
    out->hskew = in->hskew;
    out->vdisplay = in->vdisplay;
    out->vsync_start = in->vsync_start;
    out->vsync_end = in->vsync_end;
    out->vtotal = in->vtotal;
    out->vscan = in->vscan;
    out->vrefresh = in->vrefresh;
    out->flags = in->flags;
    out->picture_aspect_ratio = in->picture_aspect_ratio;

    // type 字段：用户空间的 type 不直接映射到内核 type
    // 需要设置 DRM_MODE_TYPE_USERDEF 标记
    if (in->type & DRM_MODE_TYPE_USERDEF)
        out->type |= DRM_MODE_TYPE_USERDEF;

    // name 字符串复制（最大 32 字符）
    strncpy(out->name, in->name, DRM_DISPLAY_MODE_LEN);
    out->name[DRM_DISPLAY_MODE_LEN - 1] = 0;

    return 0;
}
```

---

### 1.5 DRM core set_config 调用链

set_config 是 CRTC 的核心回调，针对 AMDGPU 驱动，最终调用到 amdgpu_dm 层的实现。

#### 1.5.1 drm_mode_set_config_internal

```c
// drivers/gpu/drm/drm_crtc.c
int drm_mode_set_config_internal(struct drm_mode_set *set)
{
    struct drm_crtc *crtc = set->crtc;
    struct drm_framebuffer *fb;

    // 保存旧的 framebuffer
    fb = crtc->primary->fb;

    // 调用驱动的 set_config 实现
    // 对 AMDGPU 来说这是 amdgpu_dm_atomic_set_config
    crtc->funcs->set_config(set);

    // 管理 framebuffer 的引用计数
    if (fb)
        drm_framebuffer_put(fb);
    if (set->fb)
        drm_framebuffer_get(set->fb);

    return 0;
}
```

#### 1.5.2 AMDGPU set_config 钩子

AMDGPU 驱动将 set_config 封装为原子提交：

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c
static const struct drm_crtc_funcs amdgpu_dm_crtc_funcs = {
    .set_config = drm_atomic_helper_set_config,
    .page_flip = drm_atomic_helper_page_flip,
    .atomic_duplicate_state = amdgpu_dm_crtc_duplicate_state,
    .atomic_destroy_state = amdgpu_dm_crtc_destroy_state,
};
```

`drm_atomic_helper_set_config` 会将传统的 set_config 调用自动转换为原子接口：

```c
// drivers/gpu/drm/drm_atomic_helper.c
int drm_atomic_helper_set_config(struct drm_mode_set *set)
{
    struct drm_atomic_state *state;
    struct drm_crtc *crtc = set->crtc;
    struct drm_crtc_state *crtc_state;
    struct drm_plane_state *primary_state;
    int ret;

    // 1. 分配原子状态
    state = drm_atomic_state_alloc(crtc->dev);
    if (!state)
        return -ENOMEM;

    // 2. 获取 CRTC 状态
    crtc_state = drm_atomic_get_crtc_state(state, crtc);
    if (IS_ERR(crtc_state))
        return PTR_ERR(crtc_state);

    // 3. 获取 primary plane 状态
    primary_state = drm_atomic_get_plane_state(state, crtc->primary);
    if (IS_ERR(primary_state))
        return PTR_ERR(primary_state);

    // 4. 设置 mode 到 CRTC 状态
    if (set->mode) {
        drm_mode_copy(&crtc_state->mode, set->mode);
        crtc_state->enable = true;
        drm_calc_timestamping_constants(crtc, &crtc_state->mode);
    } else {
        crtc_state->enable = false;
    }

    // 5. 设置 framebuffer 到 plane 状态
    if (set->fb) {
        primary_state->fb = set->fb;
        drm_framebuffer_get(set->fb);
    }

    // 6. 设置 connector 到 CRTC 状态
    // ...

    // 7. 原子提交！触发整个 atomic check 和 commit 流程
    ret = drm_atomic_commit(state);

    drm_atomic_state_put(state);
    return ret;
}
```

---

### 1.6 AMDGPU DM (Display Manager) 层

原子提交最终进入 AMDGPU DM 层的 atomic_commit 实现。

#### 1.6.1 amdgpu_dm_atomic_commit

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c
static int amdgpu_dm_atomic_commit(struct drm_device *dev,
                                   struct drm_atomic_state *state,
                                   bool nonblock)
{
    int ret;

    // Phase 1: Atomic check - 验证所有状态是否合法
    ret = drm_atomic_helper_check(dev, state);
    if (ret)
        return ret;

    // Phase 2: 准备硬件资源
    // - 分配 DC 所需的 stream/plane 对象
    // - 计算缩放参数
    // - 验证链路带宽

    // Phase 3: 提交到 DC 层
    ret = amdgpu_dm_atomic_commit_tail(state, NULL);

    return ret;
}
```

#### 1.6.2 amdgpu_dm_atomic_commit_tail 核心实现

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c
static void amdgpu_dm_atomic_commit_tail(struct drm_atomic_state *state)
{
    struct dm_atomic_state *dm_state;
    struct amdgpu_device *adev;
    struct drm_crtc *crtc;
    int i;

    // 1. 处理每个 CRTC 的状态更新
    for_each_new_crtc_in_state(state, crtc, crtc_state, i) {
        struct dm_crtc_state *dm_crtc_state = to_dm_crtc_state(crtc_state);

        // 2. 将 DRM mode 转换为 DC timing
        if (crtc_state->enable && crtc_state->active) {
            struct dc_stream_state *stream = dm_crtc_state->stream;

            // 关键转换：DRM mode → DC timing
            dm_connector_mode_to_dc_timing(crtc_state->mode, stream);
        }
    }

    // 3. 获取 DM 原子状态
    dm_state = dm_atomic_get_new_state(state);

    // 4. 调用 DC 层提交 streams
    //    dc_commit_streams 是进入 DC 层的入口
    dc_commit_streams(adev->dm.dc, dm_state->streams, dm_state->stream_count);
}
```

#### 1.6.3 dm_connector_mode_to_dc_timing

这是 DRM 模式到 DC 时序转换的核心函数：

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c
static void dm_connector_mode_to_dc_timing(
    const struct drm_display_mode *drm_mode,
    struct dc_stream_state *stream)
{
    struct dc_crtc_timing *timing = &stream->timing;

    // 像素时钟：DRM 使用 kHz，DC 使用 100Hz 单位
    timing->pix_clk_100hz = drm_mode->clock * 10;

    // 水平参数
    timing->h_addressable = drm_mode->hdisplay;
    timing->h_border_left = 0;
    timing->h_border_right = 0;
    timing->h_front_porch = drm_mode->hsync_start - drm_mode->hdisplay;
    timing->h_sync_width = drm_mode->hsync_end - drm_mode->hsync_start;
    timing->h_total = drm_mode->htotal;

    // 垂直参数
    timing->v_addressable = drm_mode->vdisplay;
    timing->v_border_top = 0;
    timing->v_border_bottom = 0;
    timing->v_front_porch = drm_mode->vsync_start - drm_mode->vdisplay;
    timing->v_sync_width = drm_mode->vsync_end - drm_mode->vsync_start;
    timing->v_total = drm_mode->vtotal;

    // 同步极性
    timing->flags.HSYNC_NEGATIVE =
        !(drm_mode->flags & DRM_MODE_FLAG_PHSYNC);
    timing->flags.VSYNC_NEGATIVE =
        !(drm_mode->flags & DRM_MODE_FLAG_PVSYNC);

    // 隔行扫描
    timing->flags.INTERLACE =
        !!(drm_mode->flags & DRM_MODE_FLAG_INTERLACE);

    // 像素编码格式
    timing->pixel_encoding = PIXEL_ENCODING_RGB;
    timing->display_color_depth = COLOR_DEPTH_888;

    // 定时标准来源
    timing->timing_3d_format = TIMING_3D_FORMAT_NONE;
    timing->scan_type = SCANNING_TYPE_UNDEFINED;

    // 计算刷新率
    if (timing->h_total && timing->v_total && timing->pix_clk_100hz)
        timing->refresh_rate = timing->pix_clk_100hz * 100 /
            (timing->h_total * timing->v_total);
}
```

#### 1.6.4 stream 状态构建

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c
struct dc_stream_state *dc_create_stream_for_sink(
    const struct drm_display_mode *drm_mode,
    struct drm_connector *drm_connector,
    struct dc_sink *sink)
{
    struct dc_stream_state *stream;
    struct dc_sink *dc_sink = sink;

    stream = kzalloc(sizeof(struct dc_stream_state), GFP_KERNEL);
    if (!stream)
        return NULL;

    // 1. 设置 DC timing
    dm_connector_mode_to_dc_timing(drm_mode, stream);

    // 2. 设置 sink 信息
    stream->sink = dc_sink;
    stream->sink_address = dc_sink->sink_signal;
    stream->signal = sink->sink_signal;

    // 3. 设置颜色属性
    stream->output_color_space = COLOR_SPACE_SRGB;
    stream->dc_output = true;

    // 4. 设置 audio 信息（如果有 audio 能力）
    stream->audio_info.mode_count = 0;

    // 5. 设置 stream 标志
    stream->apply_edid_fixup = true;

    return stream;
}
```

---

### 1.7 DC (Display Core) 层

DC 层是 AMDGPU 显示驱动的核心抽象层，管理显示流水线的状态转换。

#### 1.7.1 dc_commit_streams

```c
// drivers/gpu/drm/amd/display/dc/core/dc.c
bool dc_commit_streams(struct dc *dc,
                       struct dc_stream_state *streams[],
                       int stream_count)
{
    struct dc_state *context;
    enum dc_status result = DC_OK;

    // 1. 创建新的 DC 状态
    context = dc_create_state();
    if (!context)
        return false;

    // 2. 将 streams 复制到新的 context
    for (int i = 0; i < stream_count; i++) {
        context->streams[i] = streams[i];
        context->stream_count++;
    }

    // 3. 资源验证 - 检查是否有足够的硬件资源
    //    - OTG 数量是否足够
    //    - DPP 管线是否足够
    //    - 带宽是否充足
    result = dc_validate_global_state(dc, context, true);
    if (result != DC_OK) {
        dc_release_state(context);
        return false;
    }

    // 4. 构建缩放参数
    for (int i = 0; i < stream_count; i++) {
        result = resource_build_scaling_params(context, context->streams[i]);
        if (result != DC_OK)
            goto fail;
    }

    // 5. 提交状态到硬件
    //    这一步最终调用 DCN 硬件编程
    result = dc_commit_state(dc, context);

    dc_release_state(context);
    return result == DC_OK;

fail:
    dc_release_state(context);
    return false;
}
```

#### 1.7.2 dc_commit_state

```c
// drivers/gpu/drm/amd/display/dc/core/dc.c
static enum dc_status dc_commit_state(struct dc *dc,
                                      struct dc_state *context)
{
    enum dc_status result = DC_ERROR_UNEXPECTED;
    int i;

    // Phase 1: 编程前端 (Front End)
    //    - DPP (Display Pipe and Plane) 设置
    //    - MPC (Multiple Pipe Combined) 设置
    //    - 色彩管理 (CM, gamut remap)
    for (i = 0; i < context->stream_count; i++) {
        struct pipe_ctx *pipe = &context->res_ctx.pipe_ctx[i];

        // DPP 编程
        if (pipe->plane_state) {
            dpp_program_gamut_remap(pipe->plane_res.scl_data.lb_params);
            dpp_program_degamma(pipe->plane_res.dpp,
                                pipe->plane_state->gamma_correction);
        }
    }

    // Phase 2: 编程后端 (Back End)
    //    - OTG (Output Timing Generator) 设置
    //    - OPTC (Output Timing Combiner) 设置
    //    - 链路设置 (DP/HDMI)
    for (i = 0; i < context->stream_count; i++) {
        // OTG timing 编程
        // 这是实际写入硬件寄存器的步骤
        result = dcn20_program_timing_generator(context, i);
        if (result != DC_OK)
            goto done;

        // 链路编码设置
        result = link_encoder_setup(pipe->stream->link,
                                    pipe->stream->signal,
                                    pipe->stream->timing);
        if (result != DC_OK)
            goto done;
    }

    // Phase 3: 等待 VBlank
    //    确保所有寄存器写入在 VBlank 期间生效
    dc->hwss.wait_for_mpcc_commit(1);

    // Phase 4: 更新状态
    dc->current_state = context;

done:
    return result;
}
```

---

### 1.8 DCN 硬件寄存器编程

DCN (Display Core Next) 是 AMDGPU 的显示硬件块。以下以 DCN 2.0 (Navi 10) 为例。

#### 1.8.1 OTG Timing 寄存器编程

```c
// drivers/gpu/drm/amd/display/dc/dcn20/dcn20_hwseq.c
static enum dc_status dcn20_program_timing_generator(
    struct dc *dc,
    struct dc_state *context,
    int pipe_idx)
{
    struct pipe_ctx *pipe_ctx = &context->res_ctx.pipe_ctx[pipe_idx];
    struct timing_generator *tg = pipe_ctx->stream_res.tg;
    struct dc_crtc_timing *timing = &pipe_ctx->stream->timing;

    // 计算 OTG 时序参数
    struct dc_crtc_timing hw_timing;
    hw_timing.h_total = timing->h_total;
    hw_timing.h_addressable = timing->h_addressable;
    hw_timing.h_front_porch = timing->h_front_porch;
    hw_timing.h_sync_width = timing->h_sync_width;

    hw_timing.v_total = timing->v_total;
    hw_timing.v_addressable = timing->v_addressable;
    hw_timing.v_front_porch = timing->v_front_porch;
    hw_timing.v_sync_width = timing->v_sync_width;

    // 调用 OTG 的编程函数
    // 这些函数最终写入硬件寄存器
    tg->funcs->program_timing(tg, &hw_timing, 0);

    return DC_OK;
}
```

#### 1.8.2 OTG 寄存器写入实现

```c
// drivers/gpu/drm/amd/display/dc/dcn20/dcn20_optc.c
void optc2_program_timing(struct timing_generator *optc,
                          const struct dc_crtc_timing *timing,
                          int vready_offset)
{
    struct optc *optc_dcn = DCN_TO_OPTC(optc);

    // H Total - 水平总像素数
    REG_SET(OTG_H_TOTAL, 0,
            OTG_H_TOTAL, timing->h_total - 1);

    // V Total - 垂直总行数
    REG_SET(OTG_V_TOTAL, 0,
            OTG_V_TOTAL, timing->v_total - 1);

    // H Sync 起始和结束
    uint32_t hsync_start = timing->h_total - timing->h_front_porch;
    uint32_t hsync_end = hsync_start - timing->h_sync_width;

    REG_SET_2(OTG_H_SYNC_A, 0,
              OTG_H_SYNC_A_START, hsync_start,
              OTG_H_SYNC_A_END, hsync_end);

    // 同步极性
    REG_SET(OTG_H_SYNC_A_CNTL, 0,
            OTG_H_SYNC_A_POL, !timing->flags.HSYNC_NEGATIVE);

    // V Sync 起始和结束
    uint32_t vsync_start = timing->v_total - timing->v_front_porch;
    uint32_t vsync_end = vsync_start - timing->v_sync_width;

    REG_SET_2(OTG_V_SYNC_A, 0,
              OTG_V_SYNC_A_START, vsync_start,
              OTG_V_SYNC_A_END, vsync_end);

    REG_SET(OTG_V_SYNC_A_CNTL, 0,
            OTG_V_SYNC_A_POL, !timing->flags.VSYNC_NEGATIVE);

    // H Blanking Start/End
    uint32_t h_blank_start = timing->h_total - timing->h_front_porch;
    uint32_t h_blank_end = h_blank_start - timing->h_addressable;

    REG_SET_2(OTG_H_BLANK, 0,
              OTG_H_BLANK_START, h_blank_start,
              OTG_H_BLANK_END, h_blank_end);

    // V Blanking Start/End
    uint32_t v_blank_start = timing->v_total - timing->v_front_porch;
    uint32_t v_blank_end = v_blank_start - timing->v_addressable;

    REG_SET_2(OTG_V_BLANK, 0,
              OTG_V_BLANK_START, v_blank_start,
              OTG_V_BLANK_END, v_blank_end);

    // Interlaced 模式控制
    if (timing->flags.INTERLACE) {
        REG_SET(OTG_INTERLACE, 1,
                OTG_INTERLACE_ENABLE, 1);
    } else {
        REG_SET(OTG_INTERLACE, 0,
                OTG_INTERLACE_ENABLE, 0);
    }

    // VCO 分频设置（时钟同步）
    // 根据 pixel clock 调整 VCO 分频系数
    // ...
}
```

#### 1.8.3 REG_SET 宏展开

DC 的 REG_SET 宏最终调用 MMIO 写入：

```c
// drivers/gpu/drm/amd/display/dc/dc_reg_helper.h
// 宏定义
#define REG_SET(reg, v, field, val) { \
    uint32_t value = REG_READ(reg); \
    value = set_reg_field_value(value, val, reg, field); \
    REG_WRITE(reg, value); \
}

// REG_WRITE 最终调用
#define REG_WRITE(reg, v) \
    dc_write(optc->base.ctx, reg##_BASE_IDX + reg, v)

// dc_write 实现
void dc_write(struct dc_context *ctx, uint32_t addr, uint32_t value)
{
    // 通过 amdgpu 的 MMIO 写入函数
    amdgpu_device_wreg(ctx->driver_context,
                       addr, value,
                       0);  // 不需要延迟
}
```

这就是 mode 参数最终到达 GPU 硬件寄存器的完整路径。

---

### 1.9 硬件寄存器的内存映射与写入

AMDGPU GPU 的寄存器映射在 PCIe BAR 空间中。DCN 寄存器位于特定的 MMIO 偏移区间。

#### 1.9.1 MMIO 地址映射

```c
// DCN 2.0 寄存器基址 (Navi 10)
// 寄存器基址定义在 dcn20/dcn20_optc.h
#define OTC_BASE_INNER 0x6000
#define OTC_BASE 0x6000

// OTG 寄存器的具体偏移
#define mmOTG_H_TOTAL 0x6040
#define mmOTG_V_TOTAL 0x6041
#define mmOTG_H_SYNC_A 0x6044
#define mmOTG_V_SYNC_A 0x6048
#define mmOTG_H_BLANK 0x604C
#define mmOTG_V_BLANK 0x6050
#define mmOTG_INTERLACE 0x6054
```

每个 DCN IP 版本 (DCN 1.0, 2.0, 2.1, 3.0, 3.01, 3.02, 3.1) 有不同的寄存器偏移和功能。例如 DCN 3.0 (RDNA 2) 增加了对 HDMI 2.1 FRL 的支持，需要额外的寄存器配置。

#### 1.9.2 写入时序示例

当 AMDGPU 驱动写入 OTG_H_TOTAL 寄存器时，硬件会在下一个 VBlank 边界加载新值，实现无缝模式切换（不产生撕裂）。

```
时间线:
VSync ────────┐    ┌────┐    ┌────┐    ┌────
              │    │    │    │    │    │
              └────┘    └────┘    └────┘
                    ^         ^
                    |         |
  REG_SET(OTG_H