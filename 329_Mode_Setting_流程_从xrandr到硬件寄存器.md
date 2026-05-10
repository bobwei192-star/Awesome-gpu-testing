# 第329天：Mode Setting 流程：从 xrandr 到硬件寄存器

## 学习目标

1. 理解从 xrandr 用户态命令到内核态 DRM ioctl 的完整调用链路
2. 掌握 drm_mode_setcrtc ioctl 在内核中的处理流程
3. 深入分析 AMDGPU DM 层中 mode setting 的实现细节
4. 理解 DC（Display Core）层的 stream 提交与硬件寄存器编程
5. 掌握通过 modetest、strace、ftrace 等工具追踪 mode setting 流程的方法

---

## 1 知识详解

### 1.1 Mode Setting 总体流程概览

当用户在终端执行 `xrandr --output HDMI-A-1 --mode 1920x1080 --rate 60` 时，历经以下层次：

```
┌─────────────────────────────────────────────────────────┐
│                   用户态应用程序                          │
│  xrandr → libXrandr → libdrm → DRM ioctl                │
└──────────────────────┬──────────────────────────────────┘
                       │ 系统调用 (ioctl)
                       ▼
┌─────────────────────────────────────────────────────────┐
│                   DRM Core (内核态)                       │
│   drm_ioctl → drm_mode_setcrtc → drm_crtc_helper_set_config│
└──────────────────────┬──────────────────────────────────┘
                       │ 调用 .mode_set 回调
                       ▼
┌─────────────────────────────────────────────────────────┐
│                 AMDGPU DM 层                             │
│  amdgpu_dm_crtc_mode_set → dm_crtc_helper_mode_set      │
│  → amdgpu_dm_commit_planes → amdgpu_dm_atomic_commit    │
└──────────────────────┬──────────────────────────────────┘
                       │ DC API 调用
                       ▼
┌─────────────────────────────────────────────────────────┐
│                 DC (Display Core) 层                     │
│  dc_commit_streams → dc_commit_state →                  │
│  dcn10_apply_ctx_for_surface → ...                      │
└──────────────────────┬──────────────────────────────────┘
                       │ 硬件寄存器编程
                       ▼
┌─────────────────────────────────────────────────────────┐
│                 DCN 硬件寄存器层                          │
│  REG_SET/REG_GET → MMIO → GPU 显示引擎                   │
│  OTG 时序寄存器 → DPP 管线 → MPC 合成 → OPP 输出       │
└─────────────────────────────────────────────────────────┘
```

整个流程可分为 5 个层次：
1. **用户态**：xrandr 通过 libXrandr 和 libdrm 发起 ioctl 系统调用
2. **DRM Core**：处理 drm_mode_setcrtc ioctl，调用 helper 函数
3. **AMDGPU DM**：AMDGPU 驱动显示管理层，转换 DRM 对象为 DC 对象
4. **DC 层**：AMD 的通用显示核心，管理 stream、surface、timing
5. **DCN 硬件**：最终写入 OTG/DPP/MPC/OPP 寄存器

---

### 1.2 xrandr 用户态流程

xrandr 是 X11 协议中 RandR（Resize and Rotate）扩展的命令行工具。

```c
// xrandr.c 简化流程（伪代码）
int main(int argc, char **argv) {
    // 1. 连接 X server
    Display *dpy = XOpenDisplay(NULL);

    // 2. 获取 RandR 版本和屏幕信息
    int major, minor;
    XRRQueryVersion(dpy, &major, &minor);

    // 3. 解析命令行参数
    //    --output HDMI-A-1 --mode 1920x1080 --rate 60
    XRRScreenResources *res = XRRGetScreenResources(dpy, window);
    RROutput output = XRRGetOutputByName(dpy, res, "HDMI-A-1");
    XRROutputInfo *output_info = XRRGetOutputInfo(dpy, res, output);

    // 4. 查找匹配的 mode
    RRMode mode_id = 0;
    for (int i = 0; i < output_info->nmode; i++) {
        XRRModeInfo *mi = &res->modes[output_info->modes[i]];
        if (strcmp(mi->name, "1920x1080") == 0)
            mode_id = output_info->modes[i];
    }

    // 5. 查找匹配的 crtc
    RRCrtc crtc = output_info->crtc;

    // 6. 调用 XRRSetCrtcConfig 设置模式
    Status s = XRRSetCrtcConfig(dpy, res, crtc, CurrentTime,
                                0, 0, mode_id, RR_Rotate_0, 1, &output);
    // 该函数内部最终通过 X 协议请求到达 X server
}
```

在 X server 内部，RandR 扩展的处理函数会调用内核的 DRM 接口：

```c
// X server RandR 处理（简化）
static void ProcRRSetCrtcConfig(ClientPtr client) {
    // 1. 解析 X 协议请求
    xRRSetCrtcConfigReq *req = (xRRSetCrtcConfigReq *)client->requestBuffer;

    // 2. 找到对应的 DRM CRTC
    RRCrtcPtr rr_crtc = LookupCrtcByID(req->crtc);
    xf86CrtcPtr crtc = rr_crtc->devPrivate;

    // 3. 通过 xf86CrtcSetMode 调用内核 mode set
    if (xf86CrtcSetMode(crtc, mode, rotation, x, y)) {
        // 成功：更新 X server 内部状态
    }
}

// xf86CrtcSetMode 最终调用内核 drm_mode_setcrtc
static Bool xf86CrtcSetMode(xf86CrtcPtr crtc, DisplayModePtr mode, ...) {
    // 调用驱动程序的内核 mode set 函数
    // 对于 modesetting 驱动：调用 drmmode_set_mode_major
    // 对于 AMDGPU 驱动：调用 drmModeSetCrtc (libdrm)
}
```

实际上对于现代 modesetting 驱动，X server 直接通过 libdrm 调用内核 DRM ioctl：

```c
// libdrm drmModeSetCrtc 实现
int drmModeSetCrtc(int fd, uint32_t crtcId, uint32_t bufferId,
                   uint32_t x, uint32_t y, uint32_t *connectors,
                   int connector_count, drmModeModeInfo *mode) {
    struct drm_mode_crtc crtc;

    memset(&crtc, 0, sizeof(crtc));
    crtc.crtc_id = crtcId;
    crtc.fb_id = bufferId;
    crtc.x = x;
    crtc.y = y;
    crtc.set_connectors_ptr = (uintptr_t)connectors;
    crtc.count_connectors = connector_count;

    if (mode) {
        memcpy(&crtc.mode, mode, sizeof(struct drm_mode_modeinfo));
        crtc.mode_valid = 1;
    }

    // 发起 DRM_IOCTL_MODE_SETCRTC ioctl
    return drmIoctl(fd, DRM_IOCTL_MODE_SETCRTC, &crtc);
}
```

通过 strace 可以观察到实际的系统调用：

```bash
# strace 追踪 xrandr 的系统调用
strace -e ioctl xrandr --output HDMI-A-1 --mode 1920x1080 2>&1 | grep DRM_IOCTL

# 输出示例：
# ioctl(4, DRM_IOCTL_MODE_GETRESOURCES, ...) = 0
# ioctl(4, DRM_IOCTL_MODE_GETCONNECTOR, ...) = 0
# ioctl(4, DRM_IOCTL_MODE_GETENCODER, ...) = 0
# ioctl(4, DRM_IOCTL_MODE_GETCRTC, ...) = 0
# ioctl(4, DRM_IOCTL_MODE_SETCRTC, {crtc_id=73, fb_id=0, ...}) = 0
# ioctl(4, DRM_IOCTL_MODE_GETPROPERTY, ...) = 0
# ioctl(4, DRM_IOCTL_MODE_DIRTYFB, ...) = 0
```

---

### 1.3 DRM Core 层：drm_mode_setcrtc ioctl 处理

在内核侧，DRM core 注册了 `DRM_IOCTL_MODE_SETCRTC` 的处理函数。

```c
// drivers/gpu/drm/drm_ioctl.c
static const struct drm_ioctl_desc drm_ioctls[] = {
    // ...
    DRM_IOCTL_DEF(DRM_IOCTL_MODE_SETCRTC, drm_mode_setcrtc, DRM_MASTER),
    // ...
};
```

`drm_mode_setcrtc` 函数是 mode setting 的入口：

```c
// drivers/gpu/drm/drm_crtc.c
int drm_mode_setcrtc(struct drm_device *dev, void *data,
                     struct drm_file *file_priv) {
    struct drm_mode_crtc *crtc_req = data;
    struct drm_crtc *crtc;
    struct drm_mode_set set;
    struct drm_framebuffer *fb = NULL;
    int ret;

    // 1. 参数校验
    if (!drm_core_check_feature(dev, DRIVER_MODESET))
        return -EOPNOTSUPP;

    // 2. 查找 CRTC 对象
    crtc = drm_crtc_find(dev, file_priv, crtc_req->crtc_id);
    if (!crtc)
        return -ENOENT;

    // 3. 查找 framebuffer（如果指定了 fb_id）
    if (crtc_req->fb_id) {
        fb = drm_framebuffer_lookup(dev, file_priv, crtc_req->fb_id);
        if (!fb)
            return -ENOENT;
    }

    // 4. 构建 mode_set 结构
    memset(&set, 0, sizeof(set));
    set.crtc = crtc;
    set.x = crtc_req->x;
    set.y = crtc_req->y;
    set.fb = fb;
    set.mode = (crtc_req->mode_valid) ? &crtc_req->mode : NULL;
    set.connectors = NULL;
    set.num_connectors = 0;

    // 5. 解析 connector 列表
    if (crtc_req->count_connectors > 0) {
        uint32_t __user *connector_ids =
            (uint32_t __user *)(unsigned long)crtc_req->set_connectors_ptr;
        set.connectors = kcalloc(crtc_req->count_connectors,
                                 sizeof(*set.connectors), GFP_KERNEL);
        if (!set.connectors) {
            ret = -ENOMEM;
            goto out;
        }

        for (int i = 0; i < crtc_req->count_connectors; i++) {
            uint32_t conn_id;
            if (get_user(conn_id, connector_ids + i)) {
                ret = -EFAULT;
                goto out;
            }
            set.connectors[i] = drm_connector_lookup(dev, file_priv, conn_id);
            if (!set.connectors[i]) {
                ret = -ENOENT;
                goto out;
            }
        }
        set.num_connectors = crtc_req->count_connectors;
    }

    // 6. 调用核心设置函数
    ret = drm_mode_set_config_internal(&set);

out:
    // 清理...
    for (int i = 0; i < set.num_connectors; i++) {
        if (set.connectors[i])
            drm_connector_put(set.connectors[i]);
    }
    kfree(set.connectors);
    if (fb)
        drm_framebuffer_put(fb);
    return ret;
}
```

`drm_mode_set_config_internal` 进一步调用 driver 的回调：

```c
// drivers/gpu/drm/drm_crtc.c
static int drm_mode_set_config_internal(struct drm_mode_set *set) {
    struct drm_crtc *crtc = set->crtc;
    struct drm_framebuffer *fb;
    struct drm_crtc *tmp;
    int ret;

    // 1. 检查是否正在设置相同的 mode（去重优化）
    if (set->mode && crtc->state->active &&
        drm_mode_equal(set->mode, &crtc->state->mode)) {
        DRM_DEBUG_KMS("Set same mode, skipping\n");
        // 只是更新 front buffer
        fb = crtc->primary->state->fb;
        if (fb && fb != set->fb) {
            ret = crtc->funcs->page_flip(crtc, set->fb, NULL, 0, NULL);
            return ret;
        }
        return 0;
    }

    // 2. 调用 driver 的 set_config 回调
    ret = crtc->funcs->set_config(set, NULL);

    return ret;
}
```

`crtc->funcs->set_config` 由每个驱动注册。对于 AMDGPU，就是 `dm_crtc_helper_set_config`。

---

### 1.4 DRM Helper：drm_crtc_helper_set_config

在 AMDGPU DM 层注册 `set_config` 回调之前，通常会调用 DRM 的 helper 函数。

```c
// drivers/gpu/drm/drm_crtc_helper.c
int drm_crtc_helper_set_config(struct drm_mode_set *set,
                               struct drm_modeset_acquire_ctx *ctx) {
    struct drm_crtc **save_crtcs, *new_crtc;
    struct drm_encoder **save_encoders, *new_encoder;
    struct drm_framebuffer *old_fb;
    int ret, i, fail;
    bool save_enabled;

    // 1. 保存当前配置状态
    save_crtcs = kcalloc(MAX_CONNECTORS, sizeof(*save_crtcs), GFP_KERNEL);
    save_encoders = kcalloc(MAX_CONNECTORS, sizeof(*save_encoders), GFP_KERNEL);

    for (int i = 0; i < set->num_connectors; i++) {
        struct drm_connector *connector = set->connectors[i];
        save_crtcs[i] = connector->encoder->crtc;
        save_encoders[i] = connector->encoder;
    }

    // 2. 更新 encoder 到 connector 的绑定
    for (int i = 0; i < set->num_connectors; i++) {
        struct drm_connector *connector = set->connectors[i];
        struct drm_encoder *encoder;

        // 查找合适的 encoder
        encoder = drm_encoder_find(dev, NULL,
                                   connector->state->best_encoder->base.id);
        set->crtc->encoder = encoder;

        // 更新 connector 状态
        connector->state->crtc = set->crtc;
        connector->state->best_encoder = encoder;
        encoder->crtc = set->crtc;
    }

    // 3. 分配 CRTC 资源
    // ...

    // 4. 调用 crtc_helper 的 mode_set 回调
    if (set->mode) {
        // 调用 crtc->helper_private->mode_set
        drm_helper_crtc_mode_set(set->crtc, set->mode,
                                 set->x, set->y, old_fb);
    }

    // 5. 调用 encoder 的 mode_set 回调
    for (int i = 0; i < set->num_connectors; i++) {
        struct drm_encoder *encoder = set->crtc->encoder;
        if (encoder && encoder->helper_private &&
            encoder->helper_private->mode_set) {
            encoder->helper_private->mode_set(encoder, set->mode,
                                              set->crtc->hwmode);
        }
    }

    // 6. 最终提交（commit）
    drm_helper_crtc_commit_set(set);

    return 0;
}
```

对应的回调函数定义在 AMDGPU 驱动的初始化中：

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c
static const struct drm_crtc_helper_funcs amdgpu_dm_crtc_helper_funcs = {
    .mode_set = dm_crtc_helper_mode_set,
    .mode_set_base = dm_crtc_helper_mode_set_base,
    .disable = dm_crtc_helper_disable,
    .prepare = dm_crtc_helper_prepare,
    .commit = dm_crtc_helper_commit,
};
```

---

### 1.5 AMDGPU DM 层：dm_crtc_helper_mode_set

DM 层的 `dm_crtc_helper_mode_set` 是连接 DRM core 和 DC 层的关键桥梁。

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c
static void dm_crtc_helper_mode_set(struct drm_crtc *crtc,
                                    struct drm_display_mode *mode,
                                    struct drm_display_mode *adjusted_mode,
                                    int x, int y,
                                    struct drm_framebuffer *old_fb) {
    struct dm_crtc_state *acrtc_state = to_dm_crtc_state(crtc->state);

    // 1. 将 DRM mode 转换为 DC timing 参数
    struct dc_crtc_timing dc_crtc_timing;
    memset(&dc_crtc_timing, 0, sizeof(dc_crtc_timing));

    dc_crtc_timing.h_addressable = mode->hdisplay;
    dc_crtc_timing.h_front_porch = mode->hsync_start - mode->hdisplay;
    dc_crtc_timing.h_sync_width = mode->hsync_end - mode->hsync_start;
    dc_crtc_timing.h_total = mode->htotal;

    dc_crtc_timing.v_addressable = mode->vdisplay;
    dc_crtc_timing.v_front_porch = mode->vsync_start - mode->vdisplay;
    dc_crtc_timing.v_sync_width = mode->vsync_end - mode->vsync_start;
    dc_crtc_timing.v_total = mode->vtotal;

    // pixel clock in 100Hz units
    dc_crtc_timing.pix_clk_100hz = mode->clock * 10;

    // 2. 设置 timing 标志
    dc_crtc_timing.flags.INTERLACE = !!(mode->flags & DRM_MODE_FLAG_INTERLACE);
    dc_crtc_timing.flags.HSYNC_POSITIVE_POLARITY =
        !!(mode->flags & DRM_MODE_FLAG_PHSYNC);
    dc_crtc_timing.flags.VSYNC_POSITIVE_POLARITY =
        !!(mode->flags & DRM_MODE_FLAG_PVSYNC);

    // 3. 设置颜色深度（默认 8bit）
    dc_crtc_timing.display_color_depth = COLOR_DEPTH_888;

    // 4. 保存 timing 到 crtc state
    acrtc_state->timing = dc_crtc_timing;

    // 5. 保存 stream 信息用于后续 commit
    //（实际 stream 创建在 atomic_check 阶段完成）
    DRM_DEBUG_DRIVER("DM: Mode set %dx%d@%dHz\n",
                     mode->hdisplay, mode->vdisplay,
                     drm_mode_vrefresh(mode));
}
```

---

### 1.6 AMDGPU DM 层：Atomic Commit 路径

在现代内核中，AMDGPU 使用 atomic modesetting 接口。`set_config` 回调实际指向 `dm_atomic_helper_set_config`：

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c
static const struct drm_crtc_funcs amdgpu_dm_crtc_funcs = {
    .reset = amdgpu_dm_crtc_reset_state,
    .destroy = amdgpu_dm_crtc_destroy,
    .set_config = drm_atomic_helper_set_config,
    .page_flip = drm_atomic_helper_page_flip,
    .atomic_duplicate_state = amdgpu_dm_crtc_duplicate_state,
    .atomic_destroy_state = amdgpu_dm_crtc_destroy_state,
};
```

`drm_atomic_helper_set_config` 将 legacy setcrtc 转换为 atomic commit：

```c
// drivers/gpu/drm/drm_atomic_helper.c
int drm_atomic_helper_set_config(struct drm_mode_set *set,
                                 struct drm_modeset_acquire_ctx *ctx) {
    struct drm_atomic_state *state;
    struct drm_crtc *crtc = set->crtc;
    int ret;

    // 1. 创建 atomic state
    state = drm_atomic_state_alloc(crtc->dev);
    if (!state)
        return -ENOMEM;
    state->acquire_ctx = ctx;

    // 2. 获取 CRTC state
    struct drm_crtc_state *crtc_state =
        drm_atomic_get_crtc_state(state, crtc);
    if (IS_ERR(crtc_state)) {
        ret = PTR_ERR(crtc_state);
        goto fail;
    }

    // 3. 设置 mode
    if (set->mode) {
        drm_mode_copy(&crtc_state->mode, set->mode);
        crtc_state->enable = true;
        drm_mode_copy(&crtc_state->adjusted_mode, set->mode);
    } else {
        crtc_state->enable = false;
    }

    // 4. 设置 connectors
    for (int i = 0; i < set->num_connectors; i++) {
        struct drm_connector_state *conn_state =
            drm_atomic_get_connector_state(state, set->connectors[i]);
        if (IS_ERR(conn_state)) {
            ret = PTR_ERR(conn_state);
            goto fail;
        }
        conn_state->crtc = crtc;
    }

    // 5. 设置 primary plane state
    struct drm_plane_state *plane_state =
        drm_atomic_get_plane_state(state, crtc->primary);
    if (IS_ERR(plane_state)) {
        ret = PTR_ERR(plane_state);
        goto fail;
    }
    plane_state->crtc = crtc;
    plane_state->fb = set->fb;
    plane_state->crtc_x = set->x;
    plane_state->crtc_y = set->y;

    // 6. 执行 atomic check 和 commit
    ret = drm_atomic_check_only(state);
    if (ret == 0)
        ret = drm_atomic_commit(state);

fail:
    drm_atomic_state_put(state);
    return ret;
}
```

---

### 1.7 AMDGPU DM：Atomic Check 阶段

在 atomic check 阶段，AMDGPU DM 层创建 DC stream 并验证可行性：

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c
int amdgpu_dm_atomic_check(struct drm_device *dev,
                           struct drm_atomic_state *state) {
    struct dm_atomic_state *dm_state = to_dm_atomic_state(state);
    struct amdgpu_device *adev = dev->dev_private;
    int ret;

    // 1. 遍历所有 CRTC，为每个启用的 CRTC 创建 DC stream
    for (int i = 0; i < state->num_crtc; i++) {
        struct drm_crtc *crtc = state->crtcs[i].ptr;
        struct drm_crtc_state *crtc_state = state->crtcs[i].new_state;
        struct dm_crtc_state *dm_crtc_state = to_dm_crtc_state(crtc_state);

        if (!crtc_state->enable)
            continue;

        // 2. 创建 DC stream
        struct dc_stream_state *stream = kzalloc(sizeof(*stream), GFP_KERNEL);
        if (!stream) {
            ret = -ENOMEM;
            goto fail;
        }

        // 3. 填充 stream 的 timing 参数
        stream->timing = dm_crtc_state->timing;

        // 4. 查找对应的 connector 获取 sink 信息
        struct drm_connector *connector = NULL;
        for (int j = 0; j < state->num_connector; j++) {
            if (state->connectors[j].ptr->state->crtc == crtc) {
                connector = state->connectors[j].ptr;
                break;
            }
        }

        if (connector) {
            struct amdgpu_dm_connector *aconn =
                to_amdgpu_dm_connector(connector);
            // 复制 sink 信息到 stream
            stream->sink = aconn->dc_sink;
            stream->sink->sink_signal = aconn->dc_sink->sink_signal;
        }

        // 5. 验证 stream 是否有效
        if (!dc_validate_stream(adev->dm.dc, stream)) {
            DRM_ERROR("DC validation failed for stream\n");
            kfree(stream);
            ret = -EINVAL;
            goto fail;
        }

        // 6. 保存 stream 到 dm state
        dm_state->streams[i] = stream;
        dm_state->stream_count++;
    }

    return 0;

fail:
    // 清理已创建的 streams
    for (int i = 0; i < state->num_crtc; i++) {
        if (dm_state->streams[i])
            kfree(dm_state->streams[i]);
    }
    return ret;
}
```

---

### 1.8 AMDGPU DM：Atomic Commit 阶段

atomic commit 负责将 stream 提交到 DC 层：

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c
void amdgpu_dm_atomic_commit_tail(struct drm_atomic_state *state) {
    struct dm_atomic_state *dm_state = to_dm_atomic_state(state);
    struct amdgpu_device *adev = state->dev->dev_private;
    struct dc *dc = adev->dm.dc;

    // 1. 准备 surface（framebuffer）
    // ...

    // 2. 准备 stream 更新
    struct dc_stream_update stream_update;
    memset(&stream_update, 0, sizeof(stream_update));

    // 3. 调用 DC 接口提交 stream
    // dc_commit_streams 是 DC 层的入口
    struct dc_stream_state *streams[MAX_STREAMS];
    int stream_count = 0;

    for (int i = 0; i < dm_state->stream_count; i++) {
        if (dm_state->streams[i]) {
            streams[stream_count++] = dm_state->streams[i];

            // 4. 更新 frontbuffer
            struct dc_surface_update surface_upd;
            memset(&surface_upd, 0, sizeof(surface_upd));

            struct dc_plane_state *plane_state =
                dm_state->streams[i]->planes[0];
            if (plane_state) {
                surface_upd.surface = plane_state;
                surface_upd.flip_addr = true;

                dc_update_planes_and_stream(dc,
                    &surface_upd, 1,
                    dm_state->streams[i],
                    &stream_update, 1);
            }
        }
    }

    // 5. 提交 streams 到 DC
    if (stream_count > 0) {
        dc_commit_streams(dc, streams, stream_count);
    }

    // 6. 处理 page flip 事件
    // ...

    // 7. 清理
    for (int i = 0; i < dm_state->stream_count; i++) {
        kfree(dm_state->streams[i]);
    }
}
```

---

### 1.9 DC 层：dc_commit_streams

DC（Display Core）层是 AMD 封装的通用显示处理库，与具体的 DCN 硬件版本无关。

```c
// drivers/gpu/drm/amd/display/dc/core/dc.c
bool dc_commit_streams(struct dc *dc,
                       struct dc_stream_state *streams[],
                       int stream_count) {
    struct dc_state *context;
    enum dc_status result = DC_OK;

    // 1. 验证参数
    if (stream_count > dc->caps.max_streams)
        return false;

    // 2. 创建新的 DC 状态
    context = dc_create_state(dc);
    if (!context)
        return false;

    // 3. 将 stream 添加到状态
    for (int i = 0; i < stream_count; i++) {
        context->streams[i] = streams[i];
        context->stream_count++;
    }

    // 4. 获取当前状态
    struct dc_state *current_context = dc->current_state;

    // 5. 计算资源需求并验证
    if (!dc_validate_plane(dc, context)) {
        dc_release_state(context);
        return false;
    }

    // 6. 应用新状态到硬件
    result = dc_commit_state(dc, context);

    // 7. 更新当前状态指针
    if (result == DC_OK) {
        // 释放旧状态
        dc_release_state(current_context);
        dc->current_state = context;
    } else {
        dc_release_state(context);
    }

    return result == DC_OK;
}
```

---

### 1.10 DC 层：dc_commit_state 硬件应用

`dc_commit_state` 是真正写入硬件寄存器的入口：

```c
// drivers/gpu/drm/amd/display/dc/core/dc.c
static enum dc_status dc_commit_state(struct dc *dc,
                                      struct dc_state *context) {
    enum dc_status result = DC_ERROR_UNEXPECTED;
    struct dc_stream_state *stream;

    // 1. 调用 DCN 的 apply_ctx_for_surface
    if (dc->hwss.apply_ctx_for_surface)
        dc->hwss.apply_ctx_for_surface(dc, context);

    // 2. 启用 stream（包括 timing 编程）
    for (int i = 0; i < context->stream_count; i++) {
        stream = context->streams[i];

        if (dc->hwss.enable_stream) {
            result = dc->hwss.enable_stream(dc, stream);
            if (result != DC_OK) {
                DC_LOG_ERROR("Failed to enable stream %d\n", i);
                return result;
            }
        }

        // 3. 更新 planes
        if (dc->hwss.update_plane) {
            for (int j = 0; j < stream->plane_count; j++) {
                dc->hwss.update_plane(
                    dc, stream->planes[j], stream);
            }
        }

        // 4. 启用 stream 输出
        if (dc->hwss.enable_output) {
            dc->hwss.enable_output(dc, stream);
        }
    }

    // 5. 优化带宽和时钟设置
    if (dc->hwss.optimize_bandwidth)
        dc->hwss.optimize_bandwidth(dc, context);

    return DC_OK;
}
```

`dc->hwss` 是一个函数表（hardware sequencer functions），根据 DCN 版本不同而指向不同的实现。

---

### 1.11 DCN 硬件层：OTG 时序寄存器编程

DCN（Display Core Next）的硬件编程最终由 DCN 版本特定的函数完成。

以 DCN 3.x 为例，`apply_ctx_for_surface` 会最终调用 OTG 的 timing 编程函数：

```c
// drivers/gpu/drm/amd/display/dc/dcn30/dcn30_hwseq.c
void dcn30_apply_ctx_for_surface(struct dc *dc,
                                  struct dc_state *context) {
    // 为每个 stream 编程 OTG 时序
    for (int i = 0; i < context->stream_count; i++) {
        struct dc_stream_state *stream = context->streams[i];
        struct timing_generator *tg = stream->tg;

        if (tg && tg->funcs->program_timing) {
            // 调用 OTG 的 program_timing 函数
            tg->funcs->program_timing(tg, &stream->timing, 0);
        }

        // 编程 OPP（Output Pixel Processor）
        struct output_pixel_processor *opp = stream->opp;
        if (opp && opp->funcs->program_output) {
            opp->funcs->program_output(opp, &stream->timing);
        }
    }
}
```

`program_timing` 函数直接操作 OTG 寄存器：

```c
// drivers/gpu/drm/amd/display/dc/dcn10/dcn10_timing_generator.c
void dcn10_timing_generator_program_timing(
    struct timing_generator *tg,
    const struct dc_crtc_timing *timing,
    int vready_offset) {

    struct dcn10_timing_generator *tgn10 = DCN10TG_FROM_TG(tg);

    // 1. 设置水平时序寄存器
    // H_TOTAL = h_total - 1
    REG_SET(OTG_H_TOTAL, 0,
            OTG_H_TOTAL, timing->h_total - 1);

    // 2. 设置水平同步参数
    // H_SYNC_A_START = h_sync_start - h_addressable
    // H_SYNC_A_END = H_SYNC_A_START + h_sync_width
    REG_SET_2(OTG_H_SYNC_A, 0,
              OTG_H_SYNC_A_START, timing->h_front_porch,
              OTG_H_SYNC_A_END,
              timing->h_front_porch + timing->h_sync_width);

    // 3. 设置垂直时序寄存器
    REG_SET(OTG_V_TOTAL, 0,
            OTG_V_TOTAL, timing->v_total - 1);

    // 4. 设置垂直同步参数
    REG_SET_2(OTG_V_SYNC_A, 0,
              OTG_V_SYNC_A_START, timing->v_front_porch,
              OTG_V_SYNC_A_END,
              timing->v_front_porch + timing->v_sync_width);

    // 5. 设置有效显示区域
    REG_SET_2(OTG_H_BLANK_START_END, 0,
              OTG_H_BLANK_START,
              timing->h_addressable + timing->h_front_porch +
              timing->h_sync_width,
              OTG_H_BLANK_END,
              timing->h_total - timing->h_front_porch -
              timing->h_sync_width - 1);

    REG_SET_2(OTG_V_BLANK_START_END, 0,
              OTG_V_BLANK_START,
              timing->v_addressable + timing->v_front_porch +
              timing->v_sync_width,
              OTG_V_BLANK_END,
              timing->v_total - timing->v_front_porch -
              timing->v_sync_width - 1);

    // 6. 设置像素时钟频率
    // DCN 中的 OTG_PIXEL_RATE 寄存器
    REG_SET(OTG_PIXEL_RATE, 0,
            OTG_PIXEL_RATE, timing->pix_clk_100hz);

    // 7. 设置同步极性
    uint32_t interlace = timing->flags.INTERLACE ? 1 : 0;
    uint32_t hsync_pol = timing->flags.HSYNC_POSITIVE_POLARITY ? 0 : 1;
    uint32_t vsync_pol = timing->flags.VSYNC_POSITIVE_POLARITY ? 0 : 1;

    REG_SET_4(OTG_CONTROL, 0,
              OTG_MASTER_EN, 1,
              OTG_INTERLACE, interlace,
              OTG_HSYNC_INVERT, hsync_pol,
              OTG_VSYNC_INVERT, vsync_pol);

    // 8. 等待 OTG 重新同步（VSYNC）
    // 等待 VBlank 以确保新时序生效
    REG_WAIT(OTG_STATUS, OTG_V_BLANK, 1, 1, 10000);
}
```

---

### 1.12 DC 层：enable_stream 与 output 编程

启用 stream 涉及更多硬件模块的编程：

```c
// drivers/gpu/drm/amd/display/dc/dcn10/dcn10_hwseq.c
void dcn10_enable_stream(struct dc *dc, struct dc_stream_state *stream) {
    struct dc_link *link = stream->sink->link;
    struct display_stream_compressor *dsc = stream->dsc;

    // 1. 设置像素时钟源
    // 选择 DP/DMI 时钟源
    struct clock_source *clk_src = stream->clk_src;
    if (clk_src && clk_src->funcs->program_pix_clk) {
        clk_src->funcs->program_pix_clk(clk_src, &stream->timing);
    }

    // 2. 启用音频（如果支持）
    // ...

    // 3. 启用 DSC（Display Stream Compression）
    if (dsc && stream->timing.flags.DSC) {
        dcn20_dsc_enable(dsc, stream);
    }

    // 4. 设置输出编码器（TMDS 或 DP）
    struct link_encoder *enc = link->link_enc;
    if (enc && enc->funcs->setup) {
        enc->funcs->setup(enc, SIGNAL_TYPE_HDMI_TYPE_A);
    }

    // 5. 启用输出
    if (enc && enc->funcs->enable_tmds_output) {
        enc->funcs->enable_tmds_output(enc,
            link->link_settings.link_rate,
            link->link_settings.lane_count);
    }
}

void dcn10_enable_output(struct dc *dc, struct dc_stream_state *stream) {
    struct dc_link *link = stream->sink->link;

    // 1. 设置 link 训练（DP）或 TMDS 配置（HDMI）
    if (link->public.connector_signal == SIGNAL_TYPE_DISPLAY_PORT) {
        // DP 链路训练
        dp_link_training(link, &link->link_settings);
    }

    // 2. 等待 VSYNC
    // ...

    // 3. 最终激活 stream
    stream->tg->funcs->enable_crtc(stream->tg);
}
```

---

### 1.13 Framebuffer 与 Plane 编程

除了 timing 设置，mode setting 还需要配置 framebuffer 地址和平面参数：

```c
// drivers/gpu/drm/amd/display/dc/dcn10/dcn10_hwseq.c
void dcn10_update_plane(struct dc *dc,
                         struct dc_plane_state *plane_state,
                         struct dc_stream_state *stream) {
    // 1. 获取 DCN 的 hubp（Display Pipe 平面输入）
    struct dcn10_hubp *hubp = plane_state->hubp;

    // 2. 设置 surface 地址（framebuffer 基地址）
    REG_SET_2(DCSURF_ADDR, 0,
              SURFACE_ADDR_HIGH,
              plane_state->address.grph.addr.high_part,
              SURFACE_ADDR_LOW,
              plane_state->address.grph.addr.low_part);

    // 3. 设置 pitch（行跨度）
    REG_SET_2(DCSURF_PITCH, 0,
              PITCH,
              plane_state->plane_size.surface_pitch,
              META_PITCH,
              plane_state->plane_size.meta_pitch);

    // 4. 设置表面格式（tiling 模式）
    REG_SET(DCSURF_CONFIG, 0,
            SURFACE_CONFIG,
            plane_state->tiling_info.gfx9.swizzle);

    // 5. 设置 viewport（显示区域）
    REG_SET_4(DCSURF_VIEWPORT, 0,
              VIEWPORT_X, plane_state->viewport.x,
              VIEWPORT_Y, plane_state->viewport.y,
              VIEWPORT_WIDTH, plane_state->viewport.width,
              VIEWPORT_HEIGHT, plane_state->viewport.height);

    // 6. 设置缩放参数
    REG_SET_3(DCSURF_SCALE, 0,
              SCL_H_RATIO, plane_state->scaling_quality.horizontal.value,
              SCL_V_RATIO, plane_state->scaling_quality.vertical.value,
              SCL_H_PHASE, plane_state->scaling_quality.hphase);

    // 7. 启用平面
    REG_UPDATE(DCSURF_CONTROL, SURFACE_ENABLE, 1);
}
```

---

### 1.14 完整的调用链时序图

```
时间 │
     │  xrandr --output HDMI-A-1 --mode 1920x1080 --rate 60
     │    │
     │    ├─ libXrandr: XRRSetCrtcConfig()
     │    │    │
     │    │    ├─ X Server: ProcRRSetCrtcConfig()
     │    │    │    │
     │    │    │    ├─ xf86CrtcSetMode()
     │    │    │    │    │
     │    │    │    │    ├─ drmModeSetCrtc() [libdrm]
     │    │    │    │    │    │
     ▼    │    │    │    │    ├─ [系统调用] ioctl(DRM_IOCTL_MODE_SETCRTC)
     │    │    │    │    │    │
   内核   │    │    │    │    ├─ drm_mode_setcrtc() [DRM Core]
     │    │    │    │    │    │
     │    │    │    │    │    ├─ drm_mode_set_config_internal()
     │    │    │    │    │    │    │
     │    │    │    │    │    │    ├─ drm_atomic_helper_set_config()
     │    │    │    │    │    │    │    │
     │    │    │    │    │    │    │    ├─ drm_atomic_check_only()
     │    │    │    │    │    │    │    │    │
     │    │    │    │    │    │    │    │    ├─ amdgpu_dm_atomic_check()
     │    │    │    │    │    │    │    │    │    ├─ dc_validate_stream()
     │    │    │    │    │    │    │    │    │    └─ 创建 DC stream
     │    │    │    │    │    │    │    │    │
     │    │    │    │    │    │    │    ├─ drm_atomic_commit()
     │    │    │    │    │    │    │    │    │
     │    │    │    │    │    │    │    │    ├─ amdgpu_dm_atomic_commit_tail()
     │    │    │    │    │    │    │    │    │    │
     │    │    │    │    │    │    │    │    │    ├─ dc_commit_streams()
     │    │    │    │    │    │    │    │    │    │    │
     │    │    │    │    │    │    │    │    │    │    ├─ dc_commit_state()
     │    │    │    │    │    │    │    │    │    │    │    │
     │    │    │    │    │    │    │    │    │    │    │    ├─ dcn30_apply_ctx_for_surface()
     │    │    │    │    │    │    │    │    │    │    │    │    │
     │    │    │    │    │    │    │    │    │    │    │    │    ├─ program_timing()
     │    │    │    │    │    │    │    │    │    │    │    │    │    │
     │    │    │    │    │    │    │    │    │    │    │    │    │    ├─ REG_SET(OTG_H_TOTAL)
     │    │    │    │    │    │    │    │    │    │    │    │    │    ├─ REG_SET(OTG_H_SYNC_A)
     │    │    │    │    │    │    │    │    │    │    │    │    │    ├─ REG_SET(OTG_V_TOTAL)
     │    │    │    │    │    │    │    │    │    │    │    │    │    ├─ REG_SET(OTG_V_SYNC_A)
     │    │    │    │    │    │    │    │    │    │    │    │    │    ├─ REG_SET(OTG_PIXEL_RATE)
     │    │    │    │    │    │    │    │    │    │    │    │    │    └─ REG_SET(OTG_CONTROL)
     │    │    │    │    │    │    │    │    │    │    │    │    │
     │    │    │    │    │    │    │    │    │    │    │    │    ├─ program_output()
     │    │    │    │    │    │    │    │    │    │    │    │    │
     │    │    │    │    │    │    │    │    │    │    │    │    └─ update_plane()
     │    │    │    │    │    │    │    │    │    │    │    │         │
     │    │    │    │    │    │    │    │    │    │    │    │         ├─ DCSURF_ADDR
     │    │    │    │    │    │    │    │    │    │    │    │         ├─ DCSURF_PITCH
     │    │    │    │    │    │    │    │    │    │    │    │         ├─ DCSURF_VIEWPORT
     │    │    │    │    │    │    │    │    │    │    │    │         └─ DCSURF_SCALE
     │    │    │    │    │    │    │    │    │    │    │    │
     │    │    │    │    │    │    │    │    │    │    │    ├─ enable_stream()
     │    │    │    │    │    │    │    │    │    │    │    │    ├─ program_pix_clk()
     │    │    │    │    │    │    │    │    │    │    │    │    ├─ dsc_enable()
     │    │    │    │    │    │    │    │    │    │    │    │    └─ link_enc->setup()
     │    │    │    │    │    │    │    │    │    │    │    │
     │    │    │    │    │    │    │    │    │    │    │    └─ enable_output()
     │    │    │    │    │    │    │    │    │    │    │         └─ dp_link_training()
     │    │    │    │    │    │    │    │    │    │    │
     │    │    │    │    │    │    │    │    │    │    └─ optimize_bandwidth()
     │    │    │    │    │    │    │    │    │    │
     │    │    │    │    │    │    │    │    └─ [硬件生效] VSYNC 到来
     │    │    │    │    │    │    │    │
     ▼    │    │    │    │    │    │    └─ [返回用户态] 设置完成
   完成   │    │    │    │    │    │
          │    │    │    │    │    └─ 屏幕切换到 1920x1080@60Hz
```

---

## 2 实践操作

### 2.1 strace 追踪 xrandr 系统调用

```bash
# 完整追踪 xrandr 的系统调用
strace -f -e trace=ioctl,write xrandr --output HDMI-A-1 --mode 1920x1080 2>&1 | \
    grep -E "DRM_IOCTL|setcrtc|setmode"

# 输出：
# ioctl(4, DRM_IOCTL_MODE_GETRESOURCES, 0x7ffd...) = 0
# ioctl(4, DRM_IOCTL_MODE_GETCONNECTOR, 0x7ffd...) = 0
# ioctl(4, DRM_IOCTL_MODE_GETENCODER, 0x7ffd...) = 0
# ioctl(4, DRM_IOCTL_MODE_GETCRTC, 0x7ffd...) = 0
# ioctl(4, DRM_IOCTL_MODE_SETCRTC, ...) = 0
# ioctl(4, DRM_IOCTL_MODE_GETPROPERTY, 0x7ffd...) = 0
```

### 2.2 ftrace 追踪内核 mode setting 函数

```bash
# 使用 ftrace 追踪内核中的 mode setting 函数调用
mount -t tracefs none /sys/kernel/tracing
cd /sys/kernel/tracing

# 设置追踪函数
echo function > current_tracer
echo "drm_mode_setcrtc" > set_ftrace_filter
echo "drm_mode_set_config_internal" >> set_ftrace_filter
echo "dm_crtc_helper_mode_set" >> set_ftrace_filter
echo "amdgpu_dm_atomic_check" >> set_ftrace_filter
echo "amdgpu_dm_atomic_commit_tail" >> set_ftrace_filter
echo "dc_commit_streams" >> set_ftrace_filter
echo "dc_commit_state" >> set_ftrace_filter
echo "dcn10_timing_generator_program_timing" >> set_ftrace_filter

# 启用追踪
echo 1 > tracing_on

# 在另一个终端执行 mode set
xrandr --output HDMI-A-1 --mode 1920x1080

# 查看追踪结果
cat trace | head -50

# 输出示例：
#  <idle>-0     [000] ..s. 12345.678901: drm_mode_setcrtc <- drm_ioctl
#  <idle>-0     [000] ..s. 12345.678902: drm_mode_set_config_internal <- drm_mode_setcrtc
#  <idle>-0     [000] ..s. 12345.678903: drm_atomic_helper_set_config <- drm_mode_set_config_internal
#  <idle>-0     [000] ..s. 12345.678904: amdgpu_dm_atomic_check <- drm_atomic_helper_set_config
#  <idle>-0     [000] ..s. 12345.678905: amdgpu_dm_atomic_commit_tail <- drm_atomic_helper_commit
#  <idle>-0     [000] ..s. 12345.678906: dc_commit_streams <- amdgpu_dm_atomic_commit_tail
#  <idle>-0     [000] ..s. 12345.678907: dc_commit_state <- dc_commit_streams
#  <idle>-0     [000] ..s. 12345.678908: dcn10_timing_generator_program_timing <- dcn30_apply_ctx

# 清理
echo 0 > tracing_on
echo nop > current_tracer
```

### 2.3 modetest 的 mode setting 演示

```bash
# 1. 列出所有资源
modetest -M amdgpu -c

# 2. 找到 connector ID 和 CRTC ID
# 假设 connector ID = 73, CRTC ID = 90

# 3. 使用 modetest 执行 mode set
# modetest -M <driver> -s <connector_id>@<crtc_id>:<mode>
modetest -M amdgpu -s 73@90:1920x1080

# 4. 带 framebuffer 的 mode set
# 首先创建一个 framebuffer
modetest -M amdgpu -w 73:mode:"1920x1080"
```

### 2.4 通过 /sys/kernel/debug/dri/ 追踪

```bash
# 查看 DRM debug 信息
cat /sys/kernel/debug/dri/0/name

# 查看 connector 状态
cat /sys/kernel/debug/dri/0/HDMI-A-1/state

# 查看 CRTC 状态
cat /sys/kernel/debug/dri/0/CRTC_0/state

# 查看帧缓冲区信息
cat /sys/kernel/debug/dri/0/framebuffer
```

### 2.5 使用 libdrm 的 mode setting C 程序

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/ioctl.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

struct modeset_dev {
    uint32_t conn_id;
    uint32_t crtc_id;
    uint32_t fb_id;
    drmModeModeInfo mode;
    struct modeset_dev *next;
};

static int modeset_find_crtc(int fd, drmModeRes *res,
                             drmModeConnector *conn,
                             struct modeset_dev *dev) {
    // 1. 查找 encoder
    for (int i = 0; i < res->count_encoders; i++) {
        drmModeEncoder *enc = drmModeGetEncoder(fd, res->encoders[i]);
        if (!enc) continue;

        // 检查 encoder 是否与 connector 兼容
        if (enc->encoder_id == conn->encoder_id) {
            // 2. 使用 encoder 当前绑定的 CRTC
            if (enc->crtc_id) {
                dev->crtc_id = enc->crtc_id;
                drmModeFreeEncoder(enc);
                return 0;
            }
        }
        drmModeFreeEncoder(enc);
    }

    // 3. 如果没有找到绑定的 CRTC，遍历所有 CRTC
    for (int i = 0; i < res->count_crtcs; i++) {
        // 检查 CRTC 掩码是否与 conn->possible_crtcs 兼容
        if (conn->possible_crtcs & (1 << i)) {
            dev->crtc_id = res->crtcs[i];
            break;
        }
    }

    return dev->crtc_id ? 0 : -1;
}

static int modeset_create_fb(int fd, struct modeset_dev *dev) {
    struct drm_mode_create_dumb create = {0};
    struct drm_mode_map_dumb map = {0};
    uint32_t handles[4] = {0};
    uint32_t pitches[4] = {0};
    uint32_t offsets[4] = {0};

    // 1. 创建 dumb buffer（简单 framebuffer）
    create.width = dev->mode.hdisplay;
    create.height = dev->mode.vdisplay;
    create.bpp = 32;

    int ret = drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);
    if (ret < 0) {
        fprintf(stderr, "failed to create dumb buffer: %m\n");
        return ret;
    }

    handles[0] = create.handle;
    pitches[0] = create.pitch;
    offsets[0] = 0;

    // 2. 添加 framebuffer 对象
    ret = drmModeAddFB2(fd, dev->mode.hdisplay, dev->mode.vdisplay,
                        DRM_FORMAT_XRGB8888, handles,
                        pitches, offsets, &dev->fb_id, 0);
    if (ret < 0) {
        fprintf(stderr, "failed to add fb: %m\n");
        return ret;
    }

    // 3. 映射 framebuffer 内存
    map.handle = create.handle;
    ret = drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map);
    if (ret < 0) {
        fprintf(stderr, "failed to map dumb buffer: %m\n");
        return ret;
    }

    // 4. 填充 framebuffer 内容（灰色渐变）
    uint32_t *fb_data = mmap(0, create.size, PROT_READ | PROT_WRITE,
                             MAP_SHARED, fd, map.offset);
    if (fb_data == MAP_FAILED) {
        fprintf(stderr, "mmap failed: %m\n");
        return -1;
    }

    for (int y = 0; y < dev->mode.vdisplay; y++) {
        for (int x = 0; x < dev->mode.hdisplay; x++) {
            uint8_t gray = (x * 255 / dev->mode.hdisplay +
                           y * 255 / dev->mode.vdisplay) / 2;
            fb_data[y * (create.pitch / 4) + x] =
                (0xFF << 24) |          // alpha
                (gray << 16) |          // red
                (gray << 8) |           // green
                (gray);                 // blue
        }
    }

    munmap(fb_data, create.size);
    return 0;
}

int main(int argc, char **argv) {
    int fd;
    drmModeRes *res;
    drmModeConnector *conn;
    struct modeset_dev dev;
    int ret;

    // 1. 打开 DRM 设备
    fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);
    if (fd < 0) {
        fprintf(stderr, "cannot open /dev/dri/card0: %m\n");
        return 1;
    }

    // 2. 获取 DRM 资源
    res = drmModeGetResources(fd);
    if (!res) {
        fprintf(stderr, "cannot get DRM resources: %m\n");
        return 1;
    }

    // 3. 找到第一个连接的 connector
    int found = 0;
    for (int i = 0; i < res->count_connectors; i++) {
        conn = drmModeGetConnector(fd, res->connectors[i]);
        if (!conn) continue;

        if (conn->connection == DRM_MODE_CONNECTED && conn->count_modes > 0) {
            found = 1;
            break;
        }
        drmModeFreeConnector(conn);
    }

    if (!found) {
        fprintf(stderr, "no connected connector found\n");
        return 1;
    }

    // 4. 初始化设备
    memset(&dev, 0, sizeof(dev));
    dev.conn_id = conn->connector_id;
    memcpy(&dev.mode, &conn->modes[0], sizeof(drmModeModeInfo));

    printf("Found connector %d, using mode %s\n",
           dev.conn_id, dev.mode.name);

    // 5. 查找 CRTC
    ret = modeset_find_crtc(fd, res, conn, &dev);
    if (ret < 0) {
        fprintf(stderr, "cannot find suitable CRTC\n");
        return 1;
    }

    printf("Using CRTC %d\n", dev.crtc_id);

    // 6. 创建 framebuffer
    ret = modeset_create_fb(fd, &dev);
    if (ret < 0) {
        fprintf(stderr, "cannot create framebuffer\n");
        return 1;
    }

    // 7. 执行 mode set!
    printf("Setting mode %s on connector %d, CRTC %d, FB %d\n",
           dev.mode.name, dev.conn_id, dev.crtc_id, dev.fb_id);

    ret = drmModeSetCrtc(fd, dev.crtc_id, dev.fb_id,
                         0, 0, &dev.conn_id, 1, &dev.mode);
    if (ret < 0) {
        fprintf(stderr, "mode set failed: %m\n");
        return 1;
    }

    printf("Mode set successful! Displaying...\n");
    printf("Press Enter to restore original mode\n");
    getchar();

    // 8. 清理
    drmModeFreeConnector(conn);
    drmModeFreeResources(res);
    close(fd);

    return 0;
}
```

编译运行：
```bash
gcc -o modeset_demo modeset_demo.c -ldrm
sudo ./modeset_demo
```

---

### 2.6 trace-cmd 追踪完整流程

```bash
# 安装 trace-cmd
sudo apt install trace-cmd

# 记录 mode setting 相关的所有函数
sudo trace-cmd record -p function -l drm_mode_setcrtc \
    -l drm_atomic_helper_set_config \
    -l amdgpu_dm_atomic_check \
    -l amdgpu_dm_atomic_commit_tail \
    -l dc_commit_streams \
    -l dc_commit_state \
    -l dcn10_timing_generator_program_timing \
    -l dcn10_update_plane \
    -l dcn10_enable_stream \
    -l dcn10_enable_output \
    sh -c "xrandr --output HDMI-A-1 --mode 1920x1080 && sleep 1"

# 查看追踪结果
sudo trace-cmd report

# 带时间戳的详细输出
sudo trace-cmd report -t | head -30
```

---

### 2.7 通过 eBPF 追踪 mode setting

```c
// modeset_trace.bpf.c - eBPF 程序追踪 mode setting
#include <linux/bpf.h>
#include <bpf_helpers.h>

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1024);
    __type(key, u32);
    __type(value, u64);
} mode_set_count SEC(".maps");

SEC("kprobe/drm_mode_setcrtc")
int trace_mode_setcrtc(struct pt_regs *ctx) {
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    u64 *count, zero = 0;

    count = bpf_map_lookup_elem(&mode_set_count, &pid);
    if (!count) {
        bpf_map_update_elem(&mode_set_count, &pid, &zero, BPF_ANY);
        count = bpf_map_lookup_elem(&mode_set_count, &pid);
    }
    if (count)
        __sync_fetch_and_add(count, 1);

    bpf_printk("drm_mode_setcrtc called by PID %d\n", pid);
    return 0;
}

SEC("kprobe/dc_commit_streams")
int trace_dc_commit(struct pt_regs *ctx) {
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    bpf_printk("dc_commit_streams called by PID %d\n", pid);
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

编译和运行：
```bash
# 使用 bpftrace（更简单的 eBPF 工具）
sudo bpftrace -e '
    kprobe:drm_mode_setcrtc {
        printf("[%s] drm_mode_setcrtc called (PID %d)\n",
               strftime("%H:%M:%S"), pid);
    }
    kprobe:dc_commit_streams {
        printf("[%s] dc_commit_streams (PID %d)\n",
               strftime("%H:%M:%S"), pid);
    }
    kprobe:dcn10_timing_generator_program_timing {
        printf("[%s] Programming OTG timing registers\n",
               strftime("%H:%M:%S"));
    }
' &

# 触发 mode set
xrandr --output HDMI-A-1 --mode 1920x1080

# 停止 bpftrace
kill %1
```

---

## 3 案例分析

### 3.1 Mode Set 失败：CRTC 资源耗尽

**问题描述**：3 台外接显示器同时设置 4K@60Hz 时，第三台显示器无法点亮。

**分析过程**：
```bash
# 查看 CRTC 资源限制
cat /sys/class/drm/card0/resource

# 输出：
# CRTC count: 6
# 可用 CRTC: 0, 1, 2, 3, 4, 5

# 查看当前 CRTC 分配
for crtc in /sys/class/drm/card0/crtc-*; do
    echo "$(basename $crtc): $(cat $crtc/state 2>/dev/null || echo inactive)"
done

# 输出：
# crtc-0: active (3840x2160)
# crtc-1: active (3840x2160)
# crtc-2: active (3840x2160)
# crtc-3: inactive
# crtc-4: inactive
# crtc-5: inactive

# 查看内核日志
dmesg | tail -5
# [drm:amdgpu_dm_atomic_check] No available CRTC for connector HDMI-A-3
# [drm:drm_atomic_helper_set_config] [CRTC:73] fail to set config

# 查看 connector 状态
cat /sys/class/drm/card0-HDMI-A-3/status
# connected

# 查看支持的 CRTC 掩码
cat /sys/class/drm/card0-HDMI-A-3/possible_crtcs
# 0x3F （表示可以使用所有 6 个 CRTC）
```

**根因**：虽然 GPU 有 6 个 CRTC，但显示管线带宽不足以同时驱动 3 个 4K@60Hz 的显示流。DC 层的 bandwidth validation 在 `dc_validate_plane` 中拒绝了第三个 stream。

**解决方案**：
```bash
# 将第三台显示器降到 1080p
xrandr --output HDMI-A-3 --mode 1920x1080 --rate 60

# 或使用 MST 菊花链
# ...

# 验证
dmesg | tail -1
# [drm] Mode set to 1920x1080 on HDMI-A-3 successfully
```

---

### 3.2 自定义 Modeline 设置失败

**问题描述**：用户尝试设置 2560x1440@75Hz 自定义分辨率，但 xrandr 报错。

**分析过程**：
```bash
# 生成 modeline
cvt 2560 1440 75

# Modeline "2560x1440_75.00"  325.50  2560 2768 3040 3488  1440 1443 1448 1500 -hsync +vsync

# 添加 mode
xrandr --newmode "2560x1440_75" 325.50 2560 2768 3040 3488 1440 1443 1448 1500 -hsync +vsync

# 绑定到输出
xrandr --addmode HDMI-A-1 "2560x1440_75"

# 设置 mode（失败）
xrandr --output HDMI-A-1 --mode "2560x1440_75"
# xrandr: Configure crtc 0 failed

# 查看内核日志
dmesg | tail -10
# [drm:amdgpu_dm_connector_mode_valid] Mode 2560x1440: clock 325500 kHz
# [drm:dm_helpers_validate_clock] Failed to validate clock 325500 kHz for TMDS
# [drm:amdgpu_dm_connector_mode_valid] Mode 2560x1440: status=MODE_CLOCK_RANGE
# [drm:drm_mode_setcrtc] Failed to set mode on [CRTC:70:crtc-0] (err=-22)
```

**根因分析**：
```c
// TMDS 单链路最大时钟限制为 340MHz（HDMI 1.4）
// 2560x1440@75Hz 的像素时钟为 325.5MHz，接近极限

// 但实际链路训练后可用带宽可能更低：
// HDMI 1.4 单链路 TMDS 实际可用带宽约 300-330MHz
// 考虑 10% 的控制字符开销，实际数据传输率 ≈ 像素时钟 × 1.25

// 链接带宽计算：
// TMDS 时钟 = 像素时钟 × 1.25（HDMI 编码开销）
// 325.5 MHz × 1.25 = 406.875 MHz TMDS 时钟
// 超出 HDMI 1.4 的 340 MHz TMDS 时钟上限

// 解决方案：
// 1. 降低刷新率到 60Hz：2560x1440@60Hz 像素时钟 241.5MHz
// 2. 使用 DisplayPort 接口（HBR2 可支持 340MHz+）
// 3. 启用 HDMI 2.0（如果硬件支持，TMDS 时钟可达 600MHz）
```

**解决验证**：
```bash
# 方案1：降到 60Hz
cvt 2560 1440 60
xrandr --newmode "2560x1440_60" 241.50 2560 2608 2640 2720 1440 1443 1448 1481 +hsync -vsync
xrandr --addmode HDMI-A-1 "2560x1440_60"
xrandr --output HDMI-A-1 --mode "2560x1440_60"
# 成功

# 验证 TMDS 时钟是否在范围内
cat /sys/kernel/debug/dri/0/HDMI-A-1/status | grep clock
# TMDS clock: 301.875 MHz (在 340MHz 范围内)
```

---

### 3.3 Atomic Commit 超时导致闪屏

**问题描述**：多显示器环境下，拖动窗口时出现间歇性闪屏，dmesg 显示 atomic commit 超时。

**分析过程**：
```bash
# 查看内核日志中的超时信息
dmesg | grep -i "atomic\|timeout\|flip"
# [drm:amdgpu_dm_atomic_commit_tail] *ERROR* [CRTC:70:crtc-0] Atomic commit timeout
# [drm:amdgpu_dm_atomic_commit_tail] *ERROR* Waiting for flips timed out on crtc 1

# 检查帧缓冲区是否被占用
cat /sys/kernel/debug/dri/0/framebuffer | head -20
# 发现多个 FB 被 pin 住未释放
```

**根因分析**：
```c
// Atomic commit tail 流程中等待 VBlank 超时的可能原因：

// 1. GPU 负载过高，渲染速度跟不上 VBlank 周期
//    - dc_commit_state 中等待 MPC 编程完成超时
//    - dcn10_verify_allow_pstate_change_high() 返回 false

// 2. 帧缓冲区被 GPU 引擎占用
//    - amdgpu_bo_pin() 后未及时 unpin
//    - 页面翻转请求排队过多（超过 3 个 pending flip）

// 3. VBlank IRQ 丢失
//    - dcn_irq_ack() 未正确确认中断
//    - 中断处理函数耗时过长

// 内核配置检查：
// CONFIG_DRM_AMD_DC_DCN3_1=y
// CONFIG_HZ=1000  # 确保高精度定时器
```

**解决方案**：
```bash
# 1. 调整 atomic commit 超时时间（默认 1000ms）
echo 2000 > /sys/module/drm/parameters/atomic_timeout_ms

# 2. 减少页面翻转数量（限制 pending flip）
echo N > /sys/module/amdgpu/parameters/atomic_flip_threshold

# 3. 检查 GPU 频率是否过低
cat /sys/class/drm/card0/device/pp_dpm_sclk
# 如果频率偏低，强制高性能：
echo high > /sys/class/drm/card0/device/power_dpm_force_performance_level

# 4. 验证修复
xrandr --output HDMI-A-1 --mode 1920x1080 --rate 60
xrandr --output DP-1 --mode 2560x1440 --rate 60
# 反复拖动窗口测试 10 分钟后无闪屏
```

---

## 4 压力测试与批量验证

### 4.1 Mode Setting 压力测试脚本

```bash
#!/bin/bash
# mode_stress_test.sh - Mode Setting 压力测试
# 测试各种分辨率切换的稳定性和性能

OUTPUT=$1
if [ -z "$OUTPUT" ]; then
    echo "Usage: $0 <output_name>"
    echo "Example: $0 HDMI-A-1"
    exit 1
fi

MODES=(
    "1920x1080"
    "1280x720"
    "1680x1050"
    "1440x900"
    "1366x768"
    "1024x768"
    "640x480"
)

echo "=== Mode Setting Stress Test ==="
echo "Output: $OUTPUT"
echo "Start time: $(date)"
echo ""

PASS=0
FAIL=0
TOTAL=0

# 循环切换模式
for mode in "${MODES[@]}"; do
    for rate in 60 50; do
        TOTAL=$((TOTAL + 1))
        echo -n "Test #$TOTAL: Setting $mode @ ${rate}Hz ... "

        start=$(date +%s%N)
        xrandr --output "$OUTPUT" --mode "$mode" --rate "$rate" 2>/dev/null
        ret=$?
        end=$(date +%s%N)
        elapsed=$(( (end - start) / 1000000 ))

        if [ $ret -eq 0 ]; then
            echo "OK (${elapsed}ms)"
            PASS=$((PASS + 1))
        else
            echo "FAIL (${elapsed}ms)"
            FAIL=$((FAIL + 1))
        fi

        # 检查 dmesg 中是否有错误
        if dmesg | tail -1 | grep -qi "error\|fail\|timeout"; then
            echo "  WARNING: dmesg shows errors after mode set!"
            dmesg | tail -3
        fi

        sleep 1
    done
done

# 恢复原始模式
echo ""
echo "=== Results ==="
echo "Total: $TOTAL | Pass: $PASS | Fail: $FAIL"
echo "End time: $(date)"
```

### 4.2 IGT GPU Tools Mode Setting 测试

```bash
# 安装 IGT GPU Tools
sudo apt install igt-gpu-tools

# 运行 KMS mode setting 测试
sudo kms_mode --device /dev/dri/card0 --run-subtest

# 单测：CRTC 配置测试
sudo kms_crtc_background --device /dev/dri/card0

# Atomic 模式设置测试
sudo kms_atomic --device /dev/dri/card0

# 页面翻转测试
sudo kms_flip --device /dev/dri/card0

# VBlank 测试
sudo kms_vblank --device /dev/dri/card0

# 颜色深度转换测试
sudo kms_color --device /dev/dri/card0

# 强制测试特定输出
sudo kms_force_connector --device /dev/dri/card0

# 查看所有可用的 IGT KMS 测试
ls /usr/lib/x86_64-linux-gnu/igt-gpu-tools/kms_*
```

### 4.3 批量 Modeline 验证

```bash
#!/bin/bash
# batch_modeline_test.sh - 批量验证 Modeline 兼容性

OUTPUT=$1

# 生成多个分辨率的 modeline
for res in "640x480" "800x600" "1024x768" "1280x720" "1280x1024" "1366x768" "1440x900" "1600x900" "1680x1050" "1920x1080"; do
    w=$(echo $res | cut -dx -f1)
    h=$(echo $res | cut -dx -f2)

    for rate in 60 50 30 24; do
        modeline=$(cvt $w $h $rate 2>/dev/null | grep Modeline)
        if [ -z "$modeline" ]; then
            continue
        fi

        name=$(echo "$modeline" | awk '{print $2}' | tr -d '"')
        echo "Testing: $name"

        xrandr --newmode $modeline 2>/dev/null
        xrandr --addmode "$OUTPUT" "$name" 2>/dev/null
        xrandr --output "$OUTPUT" --mode "$name" 2>/dev/null

        if [ $? -eq 0 ]; then
            echo "  PASS: $name"
        else
            echo "  FAIL: $name"
        fi

        xrandr --delmode "$OUTPUT" "$name" 2>/dev/null
        xrandr --rmmode "$name" 2>/dev/null
    done
done
```

---

## 5 相关链接

| 类型 | 链接 | 说明 |
|------|------|------|
| 内核源码 | https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/drm_crtc.c | DRM CRTC 核心实现 |
| 内核源码 | https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/drm_atomic_helper.c | DRM Atomic Helper |
| 内核源码 | https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c | AMDGPU DM 主文件 |
| 内核源码 | https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/core/dc.c | DC 核心实现 |
| 内核源码 | https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/dcn10/dcn10_optc.c | DCN10 OTG 时序生成器 |
| 内核文档 | https://www.kernel.org/doc/html/latest/gpu/drm-kms.html | DRM KMS 文档 |
| 内核文档 | https://www.kernel.org/doc/html/latest/gpu/amdgpu.html | AMDGPU 驱动文档 |
| 工具 | https://cgit.freedesktop.org/xorg/app/xrandr/ | xrandr 源码 |
| 工具 | https://github.com/airlied/modeset | modeset 示例程序 |
| 工具 | https://gitlab.freedesktop.org/drm/igt-gpu-tools | IGT GPU Tools |
| 调试 | https://www.kernel.org/doc/html/latest/trace/ftrace.html | Ftrace 文档 |
| 调试 | https://github.com/iovisor/bpftrace | bpftrace 项目 |
| 协议 | https://vesa.org/vesa-document-library/ | VESA 标准文档 |
| 协议 | https://www.hdmi.org/ | HDMI 规范 |

---

## 今日小结

1. **Mode Setting 完整调用链**：从用户空间 xrandr 到硬件寄存器编程，共经过 5 层——libXrandr/X Server → libdrm → DRM Core → AMDGPU DM → DC → DCN 硬件，每层都有明确的职责划分。

2. **DRM Core 核心 ioctl**：`drm_mode_setcrtc` 是 legacy mode setting 的入口，通过 `drm_crtc_helper_set_config` 调用驱动回调。Atomic path 则通过 `drm_atomic_helper_set_config` 统一管理状态转换。

3. **AMDGPU DM 桥接层**：`dm_crtc_helper_mode_set` 将 DRM mode 转换为 DC timing；`amdgpu_dm_atomic_check` 验证配置可行性；`amdgpu_dm_atomic_commit_tail` 执行最终提交。

4. **DC 核心调度**：`dc_commit_streams` 和 `dc_commit_state` 是 DC 层的关键入口，负责管道的构建和激活。`dc->hwss` 函数指针表根据 DCN 版本动态选择实现。

5. **DCN 寄存器编程**：通过 `REG_SET/REG_WRITE` 宏直接操作 OTG 时序寄存器（H_TOTAL、V_TOTAL、H_SYNC_A、V_SYNC_A、PIXEL_RATE）和 DPP 平面寄存器（DCSURF_ADDR、PITCH、VIEWPORT）。

6. **调试工具组合**：strace 跟踪 ioctl 调用、ftrace 跟踪内核函数调用、bpftrace 跟踪寄存器写入、modetest 验证模式设置、sysfs/debugfs 检查状态，组合使用可覆盖全链路调试需求。

7. **常见失败模式**：CRTC 资源耗尽（`-ENOSPC`）、像素时钟超限（`MODE_CLOCK_RANGE`）、带宽不足（`MODE_BAD`）、atomic commit 超时（`-ETIMEDOUT`），每种失败都有明确的 dmesg 日志定位。

8. **压力测试方法**：通过脚本批量切换分辨率和刷新率，结合 IGT GPU Tools 的 `kms_*` 测试套件，可系统性验证 mode setting 的稳定性和性能。

---

## 扩展思考

1. **Atomic Modesetting 优势**：legacy `drm_mode_setcrtc` 和 atomic `drm_atomic_helper_set_config` 在状态管理上的本质区别是什么？为什么 atomic 能避免中间状态导致的闪屏？

2. **性能瓶颈分析**：在 8K@60Hz 场景下，mode setting 的哪个环节最可能成为性能瓶颈？是 DRM core 的 state 复制、DC 的 bandwidth 计算，还是 DCN 的 register 编程？

3. **多 GPU 场景**：在 PRIME 渲染卸载（render offloading）场景下，mode setting 的 ioctl 调用路径有何变化？discrete GPU 的 framebuffer 如何通过 integrated GPU 输出？

4. **热插拔处理**：显示器热插拔时，`drm_connector_detect` 如何在 `_connector_early_unregister` 和 `_connector_register` 之间协调 mode setting 的重置？

5. **虚拟化场景**：在 GPU 透传（VFIO passthrough）或 virtio-gpu 场景下，guest 的 mode setting 调用链与 bare-metal 有何不同？paravirtualized 驱动如何模拟 CRTC/Encoder/Connector？
