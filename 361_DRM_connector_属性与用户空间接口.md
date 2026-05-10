# 第361天：DRM connector 属性与用户空间接口

## 学习目标
- 理解 DRM connector 属性的体系结构：标准属性 vs 驱动私有属性
- 掌握 drm_property 数据结构的实现与注册机制
- 分析 AMDGPU DC 驱动中自定义 connector 属性的添加流程
- 掌握用户空间通过 libdrm / ioctl 枚举和修改属性的完整路径

## 知识详解

### 1. DRM 属性（Property）体系概述

DRM 属性是内核与用户空间之间传递显示配置信息的通用机制。属性以键值对的形式附着在 DRM 对象（CRTC、Connector、Plane）上，用户空间可以通过 ioctl 读取或修改它们。

```
┌─────────────────────────────────────────────────────────────┐
│                    用户空间应用程序                           │
│  (xrandr / modetest / Wayland compositor / KDE / GNOME)     │
└──────────────────────────┬──────────────────────────────────┘
                           │ DRM_IOCTL_MODE_GETPROPERTY
                           │ DRM_IOCTL_MODE_SETPROPERTY
                           │ DRM_IOCTL_MODE_GETCONNECTOR
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  DRM Core (drm_mode_object)                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  drm_property (id, name, flags, values[], enum_list)  │  │
│  └───────────────────────────────────────────────────────┘  │
│                          │ 关联                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  drm_connector ──→ properties[]                        │  │
│  │  drm_crtc      ──→ properties[]                        │  │
│  │  drm_plane     ──→ properties[]                        │  │
│  └───────────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           │ 驱动回调
                           ▼
┌─────────────────────────────────────────────────────────────┐
│               AMDGPU DC (amdgpu_dm_connector)               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  标准属性: EDID, DPMS, link-status, broadcast-rgb    │  │
│  │  私有属性: VR range, adaptive_sync, colorspace,      │  │
│  │           scaling mode, underscan, audio format ...   │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

#### 1.1 drm_property 数据结构

属性在 DRM 核心中用 `struct drm_property` 表示，定义于 [include/drm/drm_property.h](https://elixir.bootlin.com/linux/v6.6/source/include/drm/drm_property.h)：

```c
struct drm_property {
    struct list_head head;        // 全局属性链表节点
    struct drm_mode_object base;  // 模式对象基类（含 id）
    uint64_t flags;               // 属性标志位
    char name[DRM_PROP_NAME_LEN]; // 属性名称字符串
    uint32_t num_values;          // 值的数量 (range: 2=min/max)
    uint64_t *values;             // 值数组 (range使用前两个)
    struct drm_property_enum_list *enum_list; // 枚举列表 (enum类型)
    int num_enum;                 // 枚举项数量
};

struct drm_mode_object {
    uint32_t id;                  // 全局唯一对象 ID
    uint32_t type;                // 对象类型 (connector/crtc/plane)
    ...
};
```

属性类型由 `flags` 字段区分：

| flags 取值 | 含义 | 值个数 |
|-----------|------|--------|
| `DRM_MODE_PROP_RANGE` | 范围型（整数区间） | 2 (min, max) |
| `DRM_MODE_PROP_ENUM` | 枚举型（命名选项列表） | enum count |
| `DRM_MODE_PROP_BITMASK` | 位掩码型 | enum count |
| `DRM_MODE_PROP_BLOB` | 二进制大对象（如 EDID） | 0 |
| `DRM_MODE_PROP_OBJECT` | 引用其他 drm_mode_object | 0 |
| `DRM_MODE_PROP_SIGNED_RANGE` | 有符号范围 | 2 |
| `DRM_MODE_PROP_ATOMIC` | 仅可用于 atomic 提交 | - |

属性创建 API（[drivers/gpu/drm/drm_property.c](https://elixir.bootlin.com/linux/v6.6/source/drivers/gpu/drm/drm_property.c)）：

```c
// 创建范围属性
struct drm_property *drm_property_create_range(
    struct drm_device *dev, u32 flags,
    const char *name, uint64_t min, uint64_t max);

// 创建带可选值的范围属性
struct drm_property *drm_property_create_signed_range(
    struct drm_device *dev, u32 flags,
    const char *name, int64_t min, int64_t max);

// 创建枚举属性
struct drm_property *drm_property_create_enum(
    struct drm_device *dev, u32 flags,
    const char *name,
    const struct drm_prop_enum_list *list, int len);

// 创建位掩码属性（底层与 enum 相同但语义不同）
struct drm_property *drm_property_create_bitmask(
    struct drm_device *dev, u32 flags,
    const char *name,
    const struct drm_prop_enum_list *list, int len);

// 创建 blob 属性（用于传递任意大小的二进制数据）
struct drm_property *drm_property_create_blob(
    struct drm_device *dev, u32 flags,
    const char *name);

// 创建对象引用属性
struct drm_property *drm_property_create_object(
    struct drm_device *dev, u32 flags,
    const char *name, uint32_t type);

// 将属性附加到模式对象 (connector/crtc/plane)
void drm_object_attach_property(
    struct drm_mode_object *obj,
    struct drm_property *property,
    uint64_t init_val);
```

#### 1.2 属性值的存储与读取

属性值在 `drm_mode_object` 中通过 `properties` 指针数组和 `property_values` 值数组存储：

```c
struct drm_mode_object {
    uint32_t id;
    uint32_t type;
    struct drm_object_properties *properties; // properties[i] 指向 drm_property
    ...
};

struct drm_object_properties {
    int count;                         // 属性数量
    struct drm_property *properties[DRM_OBJECT_MAX_PROPERTY]; // 指针数组 (max 24)
    uint64_t values[DRM_OBJECT_MAX_PROPERTY];                 // 对应的值
};
```

用户空间通过以下 ioctl 访问属性：

```
DRM_IOCTL_MODE_GETPROPERTY   → 获取某个属性的定义（名称、类型、枚举值等）
DRM_IOCTL_MODE_GETCONNECTOR  → 获取 connector 当前状态，包括关联属性和值
DRM_IOCTL_MODE_SETPROPERTY   → 设置属性值（legacy 接口）
```

### 2. 标准 Connector 属性

DRM 核心定义了一组标准 connector 属性，所有驱动共享。这些属性在 [drivers/gpu/drm/drm_connector.c](https://elixir.bootlin.com/linux/v6.6/source/drivers/gpu/drm/drm_connector.c) 中的 `drm_connector_attach_dpms()` 和 `drm_connector_create_standard_properties()` 函数中注册。

#### 2.1 DPMS 属性

DPMS（Display Power Management Signaling）是最基础的 connector 属性之一，控制显示器的电源状态。

```c
// drivers/gpu/drm/drm_connector.c
int drm_connector_init(struct drm_device *dev,
                       struct drm_connector *connector,
                       const struct drm_connector_funcs *funcs,
                       int connector_type)
{
    ...
    drm_object_attach_property(&connector->base,
        dev->mode_config.dpms_property, 0);
    ...
}

// DPMS 枚举值 (include/uapi/drm/drm_mode.h)
#define DRM_MODE_DPMS_ON         0  // 正常显示
#define DRM_MODE_DPMS_STANDBY    1  // 待机
#define DRM_MODE_DPMS_SUSPEND    2  // 挂起
#define DRM_MODE_DPMS_OFF        3  // 关闭
```

在 atomic 模式下，DPMS 被抽象为 CRTC 的 active 状态 + connector 的 crtc 绑定组合，但 legacy 接口仍然保留 DPMS 属性。

#### 2.2 EDID 属性

EDID 是 blob 类型的属性，保存显示器的 EDID 二进制数据：

```c
// drivers/gpu/drm/drm_connector.c
int drm_connector_create_standard_properties(struct drm_device *dev)
{
    dev->mode_config.edid_property =
        drm_property_create_blob(dev, 0, "EDID");
    ...
}
```

当驱动通过 `drm_add_edid_modes()` 解析 EDID 后，EDID blob 通过 `drm_connector_update_edid_property()` 更新：

```c
int drm_connector_update_edid_property(struct drm_connector *connector,
                                       const struct edid *edid)
{
    int ret;
    struct drm_property_blob *blob;

    if (edid) {
        // 创建包含原始 EDID 数据的 blob
        blob = drm_property_create_blob(connector->dev,
                     edid->extensions * EDID_LENGTH + EDID_LENGTH,
                     edid);
        ...
    }
    // 将 blob ID 写入 connector 的 EDID 属性
    drm_object_property_set_value(&connector->base,
        dev->mode_config.edid_property,
        blob ? blob->base.id : 0);
    ...
}
```

用户空间读取 EDID 属性即可获得显示器 EDID 的原始字节流，这是显示器能力检测的首要信息来源。

#### 2.3 非标准但通用的属性

以下属性虽然不是 DRM 核心强制注册，但被大多数驱动（包括 AMDGPU）广泛实现，可视为事实标准：

| 属性名 | 类型 | 用途 |
|--------|------|------|
| `scaling mode` | ENUM | None / Full / Center / Full aspect |
| `broadcast rgb` | ENUM | Automatic / Full / Limited 16:235 |
| `link-status` | ENUM | Good / Bad（DP 链路重训练） |
| `non-desktop` | RANGE(0/1) | 是否为非桌面显示器（VR HMD） |
| `content type` | ENUM | No Data / Graphics / Photo / Cinema / Game |
| `hdr_output_metadata` | BLOB | HDR 静态元数据 (HDR10 / HLG) |
| `Colorspace` | ENUM | Default / BT.709 / BT.2020 / SMPTE_ST_2084 |
| `max bpc` | RANGE | 最大每通道色深 (8/10/12/16) |
| `vrr_capable` | RANGE(0/1) | 是否支持 VRR（Variable Refresh Rate） |
| `adaptive_sync` | ENUM | 自适应同步开关 |

#### 2.4 link-status 属性的特殊作用

`link-status` 属性在 DP 链路训练失败时特别重要。当 DP 链路因噪声或电缆问题降级时，驱动将 `link-status` 设置为 `BAD`，用户空间检测到后可以触发链路重训练：

```c
// drivers/gpu/drm/drm_dp_helper.c 中的典型处理
// 当 DP 链路训练失败或发生链路退化时:
connector->state->link_status = DRM_LINK_STATUS_BAD;
```

用户空间需要做：
1. 检测到 `link-status` 变为 BAD
2. 触发一个 atomic commit 尝试修复
3. 驱动在 `atomic_check` 中执行链路重训练

### 3. AMDGPU DC 驱动的 Connector 属性

AMDGPU 驱动在 DC（Display Core）层实现了大量私有 connector 属性。这些属性主要在 [drivers/gpu/drm/amd/amdgpu/amdgpu_dm.c](https://elixir.bootlin.com/linux/v6.6/source/drivers/gpu/drm/amd/amdgpu/amdgpu_dm.c) 中的 `amdgpu_dm_connector_late_register()` 和 `amdgpu_dm_connector_init()` 函数中注册。

#### 3.1 AMDGPU 私有属性总览

```
属性名称                    类型          默认值         说明
─────────────────────────────────────────────────────────────
underscan                 ENUM           OFF          消隐/过扫描
underscan hborder         RANGE(0-128)   0            水平边距宽
underscan vborder         RANGE(0-128)   0            垂直边距宽
audio                     ENUM           AUTO         音频输出模式
dce audio                 RANGE(0-1)     1            DCE 音频开关
freesync_capability       RANGE(0-1)     0            FreeSync 能力
freesync_vid_min         RANGE(10-250)  40           FreeSync 最小刷新率 VID
freesync_vid_max         RANGE(240-250) 0            FreeSync 最大刷新率 VID
vr_range                  BLOB           -            VRR 频率范围 (min/max)
adaptive_sync             ENUM           OFF          自适应同步开关
max_bpc                   RANGE(8-16)    16           最大色深
force_audio               ENUM           OFF          音频强制模式
colorspace                ENUM           Default      色彩空间
output_csc                RANGE          0            输出色彩空间转换矩阵
content_type              ENUM           No Data      内容类型
hdmi_20bpc                RANGE(0-1)     0            HDMI 2.0 10bpc 支持
```

#### 3.2 属性注册流程分析

AMDGPU 的属性注册入口在 `amdgpu_dm_connector_init()`：

```c
// 简化自 drivers/gpu/drm/amd/amdgpu/amdgpu_dm.c
static int amdgpu_dm_connector_init(struct amdgpu_display_manager *dm,
                                    struct amdgpu_dm_connector *aconnector,
                                    int link_index,
                                    struct amdgpu_dm_connector *aconnector_alt)
{
    struct drm_connector *connector = &aconnector->base;
    struct drm_connector_funcs *funcs = &dm_connector_funcs;
    int res;

    // 1. 初始化基础 connector
    res = drm_connector_init(dm->ddev, connector, funcs,
                             connector_type);
    if (res)
        return res;

    // 2. 注册标准属性
    drm_connector_attach_dpms(connector);
    drm_connector_attach_edid_property(connector);
    drm_connector_attach_content_type_property(connector);
    drm_connector_attach_vrr_capable_property(connector);
    drm_connector_attach_max_bpc_property(connector, 8, 16);

    // 3. 注册私有属性
    // 过扫描 (underscan)
    if (connector->connector_type == DRM_MODE_CONNECTOR_HDMIA ||
        connector->connector_type == DRM_MODE_CONNECTOR_HDMIB) {
        amdgpu_dm_connector_add_underscan_properties(connector);
    }

    // 音频
    amdgpu_dm_connector_add_audio_properties(connector);

    // FreeSync / VRR
    amdgpu_dm_connector_add_freesync_properties(connector);

    // 色彩空间
    amdgpu_dm_connector_add_colorspace_properties(connector);

    // 自适应同步
    drm_connector_attach_adaptive_sync_property(connector);

    return 0;
}
```

#### 3.3 FreeSync / VRR 属性注册详解

FreeSync 是 AMDGPU 的关键特性，其属性注册实现：

```c
// drivers/gpu/drm/amd/amdgpu/amdgpu_dm.c
static void amdgpu_dm_connector_add_freesync_properties(
        struct drm_connector *connector)
{
    struct drm_device *dev = connector->dev;

    // freesync_capability: 0/1 表示是否支持 FreeSync
    drm_object_attach_property(&connector->base,
        dev->mode_config.freesync_capability_property, 0);

    // freesync_vid_min: VID 最小刷新率
    drm_object_attach_property(&connector->base,
        dev->mode_config.freesync_vid_min_property, 40);

    // freesync_vid_max: VID 最大刷新率
    drm_object_attach_property(&connector->base,
        dev->mode_config.freesync_vid_max_property, 0);
}
```

这些属性的值在显示器 EDID 解析阶段被填充。当 `amdgpu_dm_update_connector_after_detect()` 读取到显示器的 VRR 能力范围后，会更新这些属性：

```c
static void amdgpu_dm_set_freesync_properties(
        struct drm_connector *connector,
        struct dc_sink *dc_sink)
{
    struct drm_device *dev = connector->dev;
    uint64_t min_refresh = drm_connector->display_info.monitor_range.min_vfreq;
    uint64_t max_refresh = drm_connector->display_info.monitor_range.max_vfreq;
    bool vrr_capable = (min_refresh != max_refresh) && (max_refresh > min_refresh);

    // 更新 VRR 能力
    drm_object_property_set_value(&connector->base,
        dev->mode_config.vrr_capable_property, vrr_capable);

    // 更新 FreeSync 范围
    drm_object_property_set_value(&connector->base,
        dev->mode_config.freesync_vid_min_property, min_refresh);
    drm_object_property_set_value(&connector->base,
        dev->mode_config.freesync_vid_max_property, max_refresh);
}
```

#### 3.4 underscan 属性分析

underscan（过扫描）是 HDMI 连接中常用的属性，用于调整显示画面边距以适应电视机的过扫描区域：

```c
static void amdgpu_dm_connector_add_underscan_properties(
        struct drm_connector *connector)
{
    struct drm_device *dev = connector->dev;

    // underscan 枚举: "off" / "on" / "auto"
    static const struct drm_prop_enum_list underscan_list[] = {
        { UNDERSCAN_OFF, "off" },
        { UNDERSCAN_ON,  "on" },
        { UNDERSCAN_AUTO, "auto" },
    };

    struct drm_property *prop;

    prop = drm_property_create_enum(dev, 0,
                   "underscan", underscan_list,
                   ARRAY_SIZE(underscan_list));
    drm_object_attach_property(&connector->base, prop, UNDERSCAN_OFF);

    // underscan hborder: 水平边距 (0-128)
    prop = drm_property_create_range(dev, 0,
                   "underscan hborder", 0, 128);
    drm_object_attach_property(&connector->base, prop, 0);

    // underscan vborder: 垂直边距 (0-128)
    prop = drm_property_create_range(dev, 0,
                   "underscan vborder", 0, 128);
    drm_object_attach_property(&connector->base, prop, 0);
}
```

当用户设置 `underscan` 为 "on" 时，DC 层会在显示管线中插入 border 缩放到实际的显示区域，具体实现在 `dc/dcnXX/dcnXX_opp.c` 中的 OPP 缩放处理。

### 4. Atomic 属性与 Legacy 属性的对比

从 Linux 4.0+ 引入 atomic 模式设置后，属性系统有了重大演进。

#### 4.1 Legacy 属性接口

传统属性通过单独的 setproperty ioctl 操作：

```
struct drm_mode_connector_set_property {
    __u32 connector_id;    // 目标 connector ID
    __u64 value;          // 要设置的值
};

// ioctl: DRM_IOCTL_MODE_SETPROPERTY
int drm_mode_connector_property_set_ioctl(
    struct drm_device *dev,
    void *data,
    struct drm_file *file_priv)
{
    struct drm_mode_connector_set_property *prop = data;
    struct drm_connector *connector;
    struct drm_property *property;

    // 查找 connector 和 property
    connector = drm_connector_find(dev, prop->connector_id);
    property = drm_property_find(dev, prop->prop_id);

    // 调用驱动回调
    if (connector->funcs->set_property)
        connector->funcs->set_property(connector, property,
                                        prop->value);
    else
        drm_object_property_set_value(&connector->base,
                                       property, prop->value);
    return 0;
}
```

legacy 接口的问题：
1. 每次 ioctl 只能修改一个属性
2. 无法实现原子切换（多个属性的同时修改）
3. 不支持回滚（rollback）
4. 可能导致闪烁（中间状态可见）

#### 4.2 Atomic 属性接口

Atomic API 将所有属性打包在一个提交中：

```c
struct drm_mode_atomic {
    __u32 flags;              // 标志位
    __u32 count_objs;         // 对象数量
    __u64 objs_ptr;           // 对象 ID 数组指针
    __u64 count_props_ptr;    // 每个对象的属性数量数组
    __u64 props_ptr;          // 属性 ID 数组指针
    __u64 prop_values_ptr;    // 属性值数组指针
    __u64 reserved;           // 保留
    __u64 user_data;          // 用户数据（用于 event）
};

// ioctl: DRM_IOCTL_MODE_ATOMIC
```

atomic 提交的基本流程：

```
用户空间:
  1. 构造 atomic 请求: drmModeAtomicAlloc()
  2. 添加属性修改: drmModeAtomicAddProperty()
  3. 提交: drmModeAtomicCommit()

内核:
  1. drm_atomic_commit()
  2. ├─ lock_all() / acquire_modeset_locks()
  3. ├─ atomic_check()  → 驱动验证配置是否可行
  4. ├─ swap_state()    → 交换新旧状态（内存中）
  5. └─ atomic_commit() → 硬件编程（异步或同步）
```

在 atomic 模式中，大多数 connector 属性都被映射到 `drm_connector_state` 中的一个字段：

```c
struct drm_connector_state {
    struct drm_connector *connector;
    struct drm_crtc *crtc;           // 绑定的 CRTC
    struct drm_encoder *encoder;     // 绑定的 encoder
    struct drm_writeback_job *writeback_job;
    struct drm_blob *hdr_metadata;
    struct drm_mode_mode *mode;

    // atomic-specific fields
    uint8_t max_bpc;                  // 最大色深
    enum drm_connector_picture_aspect_ratio picture_aspect_ratio;
    enum drm_connector_content_type content_type;
    unsigned int best_encoder;
    ...

    // driver-specific state (embedded via container_of)
    // AMDGPU extends with:
    //   bool underscan_enable;
    //   unsigned int underscan_hborder;
    //   unsigned int underscan_vborder;
    //   enum amdgpu_audio audio;
    //   enum amdgpu_colorspace colorspace;
};
```

#### 4.3 AMDGPU 的 atomic 属性处理

AMDGPU DC 驱动在 `atomic_check` 和 `atomic_commit` 阶段处理 connector 属性的变化：

```c
// amdgpu_dm.c - atomic_check implementation
static int dm_connector_atomic_check(struct drm_connector *connector,
                                     struct drm_atomic_state *state)
{
    struct drm_connector_state *new_state =
        drm_atomic_get_new_connector_state(state, connector);
    struct drm_connector_state *old_state =
        drm_atomic_get_old_connector_state(state, connector);
    struct drm_crtc *crtc = new_state->crtc;
    struct drm_crtc_state *crtc_state;

    // 如果没有绑定 CRTC，跳过
    if (!crtc)
        return 0;

    // 检查 max_bpc 变化是否需要修改 CRTC 时钟
    if (new_state->max_bpc != old_state->max_bpc) {
        crtc_state = drm_atomic_get_crtc_state(state, crtc);
        if (IS_ERR(crtc_state))
            return PTR_ERR(crtc_state);

        // 标记模式可能需要改变（色深变化影响像素时钟）
        crtc_state->mode_changed = true;
    }

    // 检查色彩空间变化
    if (new_state->colorspace != old_state->colorspace) {
        crtc_state = drm_atomic_get_crtc_state(state, crtc);
        if (IS_ERR(crtc_state))
            return PTR_ERR(crtc_state);
        crtc_state->mode_changed = true;
    }

    return 0;
}
```

### 5. drm_connector 属性与 DC 管线的映射

用户空间设置的 connector 属性需要最终传递到 DC 层的显示管线。AMDGPU 通过 `dm_connector_state` 结构体关联 DRM connector state 和 DC 状态：

```c
struct dm_connector_state {
    struct drm_connector_state base;  // 继承 DRM connector state

    // DC 相关内容
    enum amdgpu_underscan_mode underscan_mode;
    uint16_t underscan_hborder;
    uint16_t underscan_vborder;
    enum amdgpu_audio audio_mode;
    bool force_audio;
    enum amdgpu_colorspace colorspace;
    uint8_t max_bpc;

    // FreeSync
    bool freesync_enabled;
    uint8_t vrr_min_refresh;
    uint8_t vrr_max_refresh;

    // 硬件相关信息
    struct dc_stream_state *dc_stream;
    bool abm_level;
};
```

属性值从 DRM 对象到 DC 硬件的传递路径：

```
用户空间设置属性
    │
    ▼
DRM ioctl (atomic / legacy)
    │
    ▼
drm_connector_funcs->atomic_set_property  (DM实现)
    │  将 DRM 属性值复制到 dm_connector_state
    ▼
atomic_check 验证
    │
    ▼
atomic_commit
    │  调用 DC 接口更新流状态
    ▼
dc_stream_update
    │  将属性值写入 DC 流结构体
    ▼
dcnXX_opp_set_underscan / dcnXX_set_audio / ...
    │  硬件寄存器编程
    ▼
显示器输出效果变化
```

具体每个属性的 DC 映射路径如下：

#### 5.1 underscan 属性 → DC OPP 设置

```c
// amdgpu_dm.c - 在 atomic_commit 中
static void dm_update_underscan(struct dc_stream_update *stream_update,
                                struct dm_connector_state *dm_state)
{
    if (dm_state->underscan_mode == UNDERSCAN_OFF) {
        stream_update->scaling_info.scaling_mode = SCALING_MODE_IDENTITY;
    } else if (dm_state->underscan_mode == UNDERSCAN_ON) {
        // 启用边框缩小
        stream_update->scaling_info.scaling_mode =
            SCALING_MODE_CENTER_TIMING;
        stream_update->scaling_info.h_border = dm_state->underscan_hborder;
        stream_update->scaling_info.v_border = dm_state->underscan_vborder;
    }
    // UNDERSCAN_AUTO: 根据 EDID 的 overscan 标志自动决定
}
```

DC 层收到 `SCALING_MODE_CENTER_TIMING` 后，在 [drivers/gpu/drm/amd/display/dc/core/dc_resource.c](https://elixir.bootlin.com/linux/v6.6/source/drivers/gpu/drm/amd/display/dc/core/dc_resource.c) 中执行实际的缩放计算。

#### 5.2 max_bpc 属性 → DC 色彩深度设置

```c
// amdgpu_dm.c
static enum dc_color_depth amdgpu_max_bpc_to_dc(uint8_t max_bpc)
{
    switch (max_bpc) {
    case 8:  return COLOR_DEPTH_888;
    case 10: return COLOR_DEPTH_101010;
    case 12: return COLOR_DEPTH_121212;
    case 16: return COLOR_DEPTH_161616;
    default: return COLOR_DEPTH_888;
    }
}

// atomic_commit 中调用
if (dm_new_state->max_bpc != dm_old_state->max_bpc) {
    stream_update->output_color_depth =
        amdgpu_max_bpc_to_dc(dm_new_state->max_bpc);
}
```

### 6. 用户空间通过 libdrm 操作属性

#### 6.1 libdrm API 概览

[libdrm](https://gitlab.freedesktop.org/mesa/drm) 是用户空间访问 DRM 接口的标准库，提供以下属性相关 API：

```c
#include <xf86drm.h>
#include <xf86drmMode.h>

// 获取 connector 的所有属性
drmModeObjectPropertiesPtr drmModeObjectGetProperties(
    int fd, uint32_t id, uint32_t type);

// 释放属性列表
void drmModeFreeObjectProperties(
    drmModeObjectPropertiesPtr props);

// 根据 ID 获取属性信息
drmModePropertyPtr drmModeGetProperty(
    int fd, uint32_t propertyId);

// 释放属性信息
void drmModeFreeProperty(drmModePropertyPtr prop);

// 传统接口：设置属性
int drmModeConnectorSetProperty(
    int fd, uint32_t connector_id,
    uint32_t property_id, uint64_t value);

// Atomic 接口：分配 atomic 请求
drmModeAtomicReqPtr drmModeAtomicAlloc(void);

// Atomic 接口：添加属性修改
int drmModeAtomicAddProperty(
    drmModeAtomicReqPtr req,
    uint32_t obj_id, uint32_t property_id,
    uint64_t value);

// Atomic 接口：提交
int drmModeAtomicCommit(
    int fd, drmModeAtomicReqPtr req,
    uint32_t flags, void *user_data);
```

#### 6.2 枚举 connector 属性的示例代码

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

static void print_properties(int fd, uint32_t connector_id)
{
    drmModeObjectProperties *props;
    drmModeProperty *prop;
    int i;

    // 获取 connector 上的所有属性
    props = drmModeObjectGetProperties(
        fd, connector_id,
        DRM_MODE_OBJECT_CONNECTOR);
    if (!props) {
        fprintf(stderr, "Cannot get properties for connector %d\n",
                connector_id);
        return;
    }

    printf("Connector %d has %d properties:\n",
           connector_id, props->count_props);

    for (i = 0; i < props->count_props; i++) {
        prop = drmModeGetProperty(fd, props->props[i]);
        if (!prop)
            continue;

        printf("  [%d] %s (id=%d, flags=0x%x) = ",
               i, prop->name, prop->prop_id, prop->flags);

        // 根据属性类型打印值
        if (prop->flags & DRM_MODE_PROP_RANGE) {
            printf("%llu [range: %llu, %llu]",
                   (unsigned long long)props->prop_values[i],
                   (unsigned long long)prop->values[0],
                   (unsigned long long)prop->values[1]);
        } else if (prop->flags & DRM_MODE_PROP_ENUM) {
            int j;
            for (j = 0; j < prop->count_enums; j++) {
                if (prop->enums[j].value == props->prop_values[i]) {
                    printf("%s", prop->enums[j].name);
                    break;
                }
            }
        } else if (prop->flags & DRM_MODE_PROP_BLOB) {
            printf("blob(id=%llu)",
                   (unsigned long long)props->prop_values[i]);
        } else if (prop->flags & DRM_MODE_PROP_BITMASK) {
            printf("0x%llx",
                   (unsigned long long)props->prop_values[i]);
        } else {
            printf("%llu",
                   (unsigned long long)props->prop_values[i]);
        }
        printf("\n");

        drmModeFreeProperty(prop);
    }

    drmModeFreeObjectProperties(props);
}

int main(void)
{
    int fd, i;
    drmModeRes *res;
    drmModeConnector *connector;

    fd = open("/dev/dri/card0", O_RDWR);
    if (fd < 0) {
        perror("open card0");
        return 1;
    }

    res = drmModeGetResources(fd);
    if (!res) {
        fprintf(stderr, "Cannot get DRM resources\n");
        close(fd);
        return 1;
    }

    for (i = 0; i < res->count_connectors; i++) {
        print_properties(fd, res->connectors[i]);
    }

    drmModeFreeResources(res);
    close(fd);
    return 0;
}
```

#### 6.3 Atomic 方式设置 VRR 属性

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

static int set_vrr_adaptive_sync(int fd, uint32_t crtc_id,
                                 uint32_t connector_id, bool enable)
{
    drmModeAtomicReq *req = drmModeAtomicAlloc();
    int ret;

    if (!req)
        return -1;

    // 查找 "adaptive_sync" 属性 ID
    drmModeObjectProperties *props = drmModeObjectGetProperties(
        fd, connector_id, DRM_MODE_OBJECT_CONNECTOR);
    if (!props) {
        drmModeAtomicFree(req);
        return -1;
    }

    uint32_t adaptive_sync_id = 0;
    for (int i = 0; i < props->count_props; i++) {
        drmModeProperty *p = drmModeGetProperty(fd, props->props[i]);
        if (p && p->flags & DRM_MODE_PROP_ENUM &&
            strcmp(p->name, "adaptive_sync") == 0) {
            adaptive_sync_id = p->prop_id;
            drmModeFreeProperty(p);
            break;
        }
        if (p) drmModeFreeProperty(p);
    }

    if (!adaptive_sync_id) {
        fprintf(stderr, "adaptive_sync property not found\n");
        drmModeFreeObjectProperties(props);
        drmModeAtomicFree(req);
        return -1;
    }

    // 添加属性修改
    drmModeAtomicAddProperty(req, crtc_id,
        DRM_MODE_ATOMIC_ACTIVE_PROP, 1);

    drmModeAtomicAddProperty(req, connector_id,
        adaptive_sync_id, enable ? 1 : 0);

    // 提交 atomic 请求
    ret = drmModeAtomicCommit(fd, req,
        DRM_MODE_ATOMIC_ALLOW_MODESET, NULL);
    if (ret)
        fprintf(stderr, "Atomic commit failed: %s\n", strerror(-ret));

    drmModeAtomicFree(req);
    drmModeFreeObjectProperties(props);
    return ret;
}
```

### 7. Connector 属性的内核实现细节

#### 7.1 drm_property 的创建与注册

以下是从 DRM 核心到 AMDGPU 驱动的完整属性注册链：

```
drm_property_create_*(dev, flags, name, ...)
    │
    ▼
drm_property_create(dev, flags, name, num_values)
    │  ├─ kzalloc(sizeof(struct drm_property))
    │  ├─ drm_mode_object_get(dev, &prop->base, DRM_MODE_OBJECT_PROPERTY)
    │  ├─ prop->flags = flags
    │  ├─ strscpy(prop->name, name, DRM_PROP_NAME_LEN)
    │  └─ list_add_tail(&prop->head, &dev->mode_config.property_list)
    ▼
drm_object_attach_property(obj, prop, init_val)
    │  ├─ WARN_ON(obj->properties->count >= DRM_OBJECT_MAX_PROPERTY) // 24
    │  ├─ obj->properties->properties[count] = prop
    │  └─ obj->properties->values[count] = init_val
```

`DRM_OBJECT_MAX_PROPERTY` 的限制（当前为 24）意味着每个对象最多只能附加 24 个属性。AMDGPU 的 connector 已经接近这个上限。

#### 7.2 属性的 get/set 回调机制

对于 atomic 驱动，属性通过 `atomic_get_property` 和 `atomic_set_property` 回调实现读写：

```c
// AMDGPU DM 的实现
static const struct drm_connector_funcs dm_connector_funcs = {
    .dpms = drm_atomic_helper_connector_dpms,
    .detect = dm_connector_detect,
    .fill_modes = drm_helper_probe_single_connector_modes,
    .destroy = dm_connector_destroy,
    .reset = dm_connector_reset,           // 重置状态
    .atomic_duplicate_state = dm_connector_duplicate_state,
    .atomic_destroy_state = dm_connector_destroy_state,
    .atomic_set_property = dm_connector_atomic_set_property,
    .atomic_get_property = dm_connector_atomic_get_property,
    .late_register = amdgpu_dm_connector_late_register,
    .early_unregister = amdgpu_dm_connector_early_unregister,
};
```

`atomic_set_property` 的核心实现：

```c
// amdgpu_dm.c
static int dm_connector_atomic_set_property(
    struct drm_connector *connector,
    struct drm_connector_state *state,
    struct drm_property *property,
    uint64_t val)
{
    struct dm_connector_state *dm_state = to_dm_connector_state(state);
    struct drm_device *dev = connector->dev;

    // 1. 检查是否是 DRM 核心标准属性
    if (property == dev->mode_config.max_bpc_property) {
        dm_state->max_bpc = val;
        return 0;
    }
    if (property == dev->mode_config.vrr_capable_property) {
        dm_state->freesync_enabled = val;
        return 0;
    }
    if (property == dev->mode_config.content_type_property) {
        state->content_type = val;
        return 0;
    }

    // 2. 检查 AMDGPU 私有属性
    if (property == dev->mode_config.underscan_property) {
        dm_state->underscan_mode = val;
        return 0;
    }
    if (property == dev->mode_config.underscan_hborder_property) {
        dm_state->underscan_hborder = val;
        return 0;
    }
    if (property == dev->mode_config.underscan_vborder_property) {
        dm_state->underscan_vborder = val;
        return 0;
    }
    if (property == dev->mode_config.audio_property) {
        dm_state->audio_mode = val;
        return 0;
    }
    if (property == dev->mode_config.colorspace_property) {
        dm_state->colorspace = val;
        return 0;
    }

    // 3. 未识别的属性，由 DRM 核心处理
    return -EINVAL;
}
```

`atomic_get_property` 是反向操作，将 `dm_connector_state` 中的值读取到 `val` 中。

### 8. 调试 Connector 属性的方法

#### 8.1 使用 modetest 列出所有属性

```bash
# 列出所有 connectors 和它们的属性
modetest -c -M amdgpu

# 输出示例:
Connectors:
id      encoder status          name            size (mm)       modes   encoders
5       4       connected       HDMI-A-1        600x340         12      4
  modes:
        index name refresh (Hz) hdisp hss hse htot vdisp vss vse vtot
  ...
  properties:
        1 EDID:
                flags: immutable blob
                blobs:
                value:
                        00ffffffffffff001e6df...
        2 DPMS:
                flags: enum
                enums: On=0 Standby=1 Suspend=2 Off=3
                value: 0
        14 link-status:
                flags: enum
                enums: Good=0 Bad=1
                value: 0
        20 broadcast rgb:
                flags: enum
                enums: Automatic=0 Full=1 Limited 16:235=2
                value: 0
        22 scaling mode:
                flags: enum
                enums: None=0 Full=1 Center=2 Full aspect=3
                value: 0
        28 max bpc:
                flags: range
                range: 8-16
                value: 16
        32 underscan:
                flags: enum
                enums: off=0 on=1 auto=2
                value: 0
        34 underscan hborder:
                flags: range
                range: 0-128
                value: 0
        35 underscan vborder:
                flags: range
                range: 0-128
                value: 0
        36 audio:
                flags: enum
                enums: auto=0 on=1 off=2
                value: 0
        42 vrr_capable:
                flags: range
                value: 1
        43 colorspace:
                flags: enum
                enums: Default=0 BT709_YCC=1 ...
                value: 0
```

#### 8.2 使用 xrandr 查看 connector 属性

```bash
# 查看所有属性（包括驱动私有属性）
xrandr --props

# 输出示例:
Screen 0: minimum 320 x 200, current 1920 x 1080, maximum 16384 x 16384
HDMI-A-0 connected 1920x1080+0+0 (normal left inverted right x axis y axis) 600mm x 340mm
    EDID:
        00ffffffffffff001e6df...
    max bpc: 16
        range: (8, 16)
    underscan: off
        supported: off, on, auto
    underscan hborder: 0
        range: (0, 128)
    underscan vborder: 0
        range: (0, 128)
    audio: auto
        supported: auto, on, off
    ...
```

```bash
# 设置属性值
xrandr --output HDMI-A-0 --set "max bpc" 10
xrandr --output HDMI-A-0 --set "underscan" on
xrandr --output HDMI-A-0 --set "underscan hborder" 40
xrandr --output HDMI-A-0 --set "underscan vborder" 30
```

#### 8.3 使用 debugfs 查看内核中的属性状态

AMDGPU 驱动在 debugfs 中导出 connector 属性信息：

```bash
# 查看所有 DRM 对象属性
cat /sys/kernel/debug/dri/0/state

# 查看 AMDGPU 特定 connector 信息
cat /sys/kernel/debug/dri/0/connector/connector5
```

#### 8.4 使用 strace 跟踪属性设置操作

```bash
# 跟踪 xrandr 设置属性时的系统调用
strace -e ioctl xrandr --output HDMI-A-0 --set "max bpc" 10 2>&1 | \
    grep -E "DRM_IOCTL|MODE_GETPROPERTY|MODE_SETPROPERTY|MODE_ATOMIC"
```

### 9. 典型问题场景分析

#### 9.1 场景：HDMI 显示器输出范围错误

**问题描述**：HDMI 显示器画面暗部细节丢失，黑色不黑而发灰，或者白色过曝。

**原因**：HDMI 有两种 RGB 输出范围——Full Range (0-255) 和 Limited Range (16-235)。电视通常使用 Limited Range，显示器通常使用 Full Range。驱动可能错误地检测了输出范围。

**解决方案**：通过 `broadcast rgb` 属性手动设置：

```bash
# 检查当前设置
xrandr --props | grep "broadcast rgb"

# 设置为 Full Range
xrandr --output HDMI-A-0 --set "broadcast rgb" Full

# 或设置为 Limited Range
xrandr --output HDMI-A-0 --set "broadcast rgb" "Limited 16:235"
```

**内核中的实现**：`broadcast rgb` 属性通过 [drivers/gpu/drm/drm_connector.c](https://elixir.bootlin.com/linux/v6.6/source/drivers/gpu/drm/drm_connector.c) 中的 `drm_connector_attach_broadcast_rgb_property()` 注册，AMDGPU DC 在 `dc_link_set_preferred_training_settings()` 中根据该属性调整输出色彩量化范围。

#### 9.2 场景：FreeSync / VRR 不工作

**问题描述**：显示器支持 FreeSync，但游戏中看不到可变刷新率效果。

**检查步骤**：

```bash
# 1. 确认显示器支持 VRR
xrandr --props | grep -E "vrr_capable|freesync"
# vrr_capable: 1 表示支持

# 2. 启用 adaptive_sync（某些系统需要）
xrandr --output DisplayPort-0 --set "adaptive_sync" 1

# 3. 确认驱动已启用 VRR
cat /sys/kernel/debug/dri/0/connector/connector5/vrr_range
```

**内核处理**：VRR 启用需要 connector 属性和 CRTC 属性的协同配置。VRR 的实际生效由 DC 层的 `dc_stream_set_vrr()` 控制，该函数根据 `dm_connector_state` 中保存的 VRR 范围和 enable 标志来配置 DCN 的 timing generator。

#### 9.3 场景：max_bpc 设置后不生效

**问题描述**：通过 `max_bpc` 属性设置为 10bpc 或 12bpc，但显示器仍报告为 8bpc。

**原因分析**：

```c
// 在 dm_connector_atomic_check 中，max_bpc 变化会标记 mode_changed
// 但真正生效需要完整的 modeset 提交
```

**正确的设置方式**：需要带有 ALLOW_MODESET 标志的 atomic 提交：

```bash
# 使用 xrandr 需要 --mode 配合
xrandr --output HDMI-A-0 --set "max bpc" 10 --mode 1920x1080

# 或者使用 wlr-randr (Wayland)
wlr-randr --output HDMI-A-0 --custom-mode 1920x1080@60 --max-bpc 10
```

如果仍然不生效，可能的原因：
1. 显示器 EDID 报告的最大色深小于设定值
2. 当前分辨率/刷新率组合不支持更高的色深（带宽不足）
3. 显示链路（HDMI/DP 版本）限制

### 10. Connector 属性与 DRM 对象类型的关系

Connector 属性只是 DRM 对象属性系统的一部分。其他 DRM 对象（CRTC、Plane）也有各自的属性：

```
DRM Object Types:
┌──────────────────────────────────────────────────┐
│ DRM_MODE_OBJECT_CRTC = 0xCCCCCCCC               │
│   - mode (blob) / active (range)                │
│   - vrr_enabled (range) / vrr_min/max (range)   │
│   - background_color (range)                    │
│   - CTM (blob - color transformation matrix)    │
│   - GAMMA_LUT (blob) / DEGAMMA_LUT (blob)       │
│   - OUT_FENCE_PTR (range)                       │
├──────────────────────────────────────────────────┤
│ DRM_MODE_OBJECT_CONNECTOR = 0xCCCCCCCB           │
│   - EDID (blob) / DPMS (enum)                   │
│   - link-status (enum) / content-type (enum)    │
│   - scaling mode (enum) / max bpc (range)       │
│   - broadcast rgb (enum) / underscan (enum)     │
│   - audio (enum) / colorspace (enum)            │
│   - vrr_capable (range) / adaptive_sync (enum)  │
│   - HDR_OUTPUT_METADATA (blob)                  │
├──────────────────────────────────────────────────┤
│ DRM_MODE_OBJECT_PLANE = 0xCCCCCCCC              │
│   - type (enum): primary / overlay / cursor     │
│   - FB_ID (object) / CRTC_ID (object)           │
│   - SRC_X/Y/W/H (range) / CRTC_X/Y/W/H (range)  │
│   - rotation (bitmask) / alpha (range)          │
│   - blend_mode (enum) / color_encoding (enum)   │
│   - FB_DAMAGE_CLIPS (blob)                      │
│   - IN_FORMATS (blob - immutable)               │
│   - zpos (range)                                │
└──────────────────────────────────────────────────┘
```

### 11. 属性变更的完整时序图

```
用户空间                        DRM 核心                   AMDGPU DC驱动           硬件
    │                             │                          │                    │
    │  DRM_IOCTL_MODE_ATOMIC      │                          │                    │
    │────────────────────────────>│                          │                    │
    │                             │                          │                    │
    │                    drm_atomic_commit()                │                    │
    │                    ┌─ 锁定所有模式锁                  │                    │
    │                    └─ 调用 atomic_check()             │                    │
    │                             │                          │                    │
    │                             │  dm_connector_atomic_check()                │
    │                             │─────────────────────────>│                    │
    │                             │                          │                    │
    │                             │  ◄── 验证通过/失败 ─────│                    │
    │                             │                          │                    │
    │                    swap_state() 交换新旧状态           │                    │
    │                             │                          │                    │
    │                    atomic_commit() 异步/同步提交       │                    │
    │                             │                          │                    │
    │                             │  dm_update_connectors()  │                    │
    │                             │─────────────────────────>│                    │
    │                             │                          │                    │
    │                             │  比较新旧 dm_connector_state            │
    │                             │  收集变化的属性:                        │
    │                             │  ├─ mode_changed? → modeset            │
    │                             │  ├─ max_bpc? → update_color_depth      │
    │                             │  ├─ underscan? → update_scaling        │
    │                             │  ├─ colorspace? → set_output_csc       │
    │                             │  └─ vrr? → set_vrr_range              │
    │                             │                          │                    │
    │                             │  dc_stream_update()                     │
    │                             │─────────────────────────────────────────>│
    │                             │                          │                    │
    │                             │                          │  REG_WRITE 设置硬件寄存器
    │                             │                          │  ──► DCN timing
    │                             │                          │  ──► AFMT color
    │                             │                          │  ──► OPP scaling
    │                             │                          │                    │
    │  ◄── commit 完成 ──────────│                          │                    │
    │                             │                          │                    │
    │  DRM_IOCTL_MODE_GETCONNECTOR │                          │                    │
    │────────────────────────────>│                          │                    │
    │                             │  读取 drm_connector_state 中保存的属性值     │
    │  ◄── 返回 connector 状态 ──│                          │                    │
```

### 12. AMDGPU 私有属性的特殊情况处理

#### 12.1 audio 属性

HDMI/DP 音频输出控制属性：

```c
// 枚举值
#define AMDGPU_AUDIO_AUTO  0  // 自动（根据 EDID 判断）
#define AMDGPU_AUDIO_ON    1  // 强制开启
#define AMDGPU_AUDIO_OFF   2  // 强制关闭

// 典型使用场景:
//   xrandr --output HDMI-A-0 --set "audio" on    # 强制开启音频
//   xrandr --output HDMI-A-0 --set "audio" off   # 强制关闭音频
//   xrandr --output HDMI-A-0 --set "audio" auto  # 恢复自动模式
```

AMDGPU DC 会根据该属性值决定是否向显示器发送音频数据包。

#### 12.2 force_audio 属性

强制音频输出模式的特殊属性：

```c
// 枚举值
#define AMDGPU_FORCE_AUDIO_OFF   0  // 不强制
#define AMDGPU_FORCE_AUDIO_RGB   1  // 强制 RGB 下输出音频
#define AMDGPU_FORCE_AUDIO_YUV   2  // 强制 YUV 下输出音频
```

#### 12.3 colorspace 属性

控制输出色彩空间：

```c
// 枚举值 (具体定义参考 AMDGPU 头文件)
enum amdgpu_dm_colorspace {
    AMDGPU_DM_COLORSPACE_DEFAULT       = 0,
    AMDGPU_DM_COLORSPACE_BT709_YCC     = 1,  // BT.709 YCbCr
    AMDGPU_DM_COLORSPACE_BT2020_RGB    = 2,  // BT.2020 RGB
    AMDGPU_DM_COLORSPACE_BT2020_YCC    = 3,  // BT.2020 YCbCr
    AMDGPU_DM_COLORSPACE_SMPTE_ST_2084 = 4,  // ST.2084 (PQ)
};
```

当 colorspace 设置为 BT.2020 RGB 时，DC 层的 `dcnXX_set_output_csc()` 会配置 output CSC 矩阵将驱动内部色彩空间转换到 BT.2020 色彩空间。

### 13. Connector 属性的测试方法

#### 13.1 使用 Python ctypes 测试属性枚举

```python
#!/usr/bin/env python3
"""DRM connector 属性枚举测试脚本"""

import ctypes
import struct
import fcntl
import os

# DRM ioctl 定义 (从 drm.h 和 drm_mode.h 获取)
DRM_IOCTL_BASE = 0x64
DRM_IOWR = lambda nr, size: (2 << 30) | (ord('d') << 8) | nr | (size << 16)
DRM_IOW  = lambda nr, size: (1 << 30) | (ord('d') << 8) | nr | (size << 16)

DRM_IOCTL_MODE_GETRESOURCES = DRM_IOWR(0xA0, 160)  # 大小视结构体而定
DRM_IOCTL_MODE_GETCONNECTOR = DRM_IOWR(0xA7, 80)
DRM_IOCTL_MODE_GETPROPERTY  = DRM_IOWR(0xAA, 64)
DRM_IOCTL_MODE_GETPROPBLOB  = DRM_IOWR(0xAC, 16)

DRM_MODE_OBJECT_CONNECTOR = 0xCCCCCCCB

class drm_mode_get_connector(ctypes.Structure):
    _fields_ = [
        ("connector_id", ctypes.c_uint32),
        ("encoder_id", ctypes.c_uint32),
        ("crtc_id", ctypes.c_uint32),
        ("connector_type", ctypes.c_uint32),
        ("connector_type_id", ctypes.c_uint32),
        ("connection", ctypes.c_uint32),
        ("mm_width", ctypes.c_uint32),
        ("mm_height", ctypes.c_uint32),
        ("subpixel", ctypes.c_uint32),
        ("pad", ctypes.c_uint32),
        ("modes_ptr", ctypes.c_uint64),
        ("count_modes", ctypes.c_uint32),
        ("props_ptr", ctypes.c_uint64),
        ("count_props", ctypes.c_uint32),
        ("encoders_ptr", ctypes.c_uint64),
        ("count_encoders", ctypes.c_uint32),
    ]

def list_connector_properties(fd, connector_id):
    """列出指定 connector 的所有属性"""
    conn = drm_mode_get_connector()
    conn.connector_id = connector_id

    # 第一次调用获取属性数量
    fcntl.ioctl(fd, DRM_IOCTL_MODE_GETCONNECTOR, conn)
    count = conn.count_props

    if count == 0:
        print(f"  No properties for connector {connector_id}")
        return

    # 分配数组并第二次调用获取属性 ID 和值
    prop_ids = (ctypes.c_uint32 * count)()
    prop_vals = (ctypes.c_uint64 * count)()
    conn.props_ptr = ctypes.addressof(prop_ids)
    conn.count_props = count

    fcntl.ioctl(fd, DRM_IOCTL_MODE_GETCONNECTOR, conn)

    for i in range(count):
        pid = prop_ids[i]
        val = prop_vals[i]

        # 获取属性详细信息
        buf = bytearray(64)
        print(f"  Property {pid}: value={val}")


def main():
    fd = os.open("/dev/dri/card0", os.O_RDWR)
    try:
        list_connector_properties(fd, 5)  # 假设 connector 5
    finally:
        os.close(fd)

if __name__ == "__main__":
    main()
```

#### 13.2 使用 IGT 测试属性

IGT（Intel GPU Tools）也支持 AMDGPU，可以使用其属性测试框架：

```bash
# 运行 kms_properties 测试
kms_properties --device /dev/dri/card0

# 列出所有 connector 属性及其取值
kms_properties --show-properties

# 验证某个属性是否可设置
kms_properties --test-connector-property "max bpc" 10

# 使用 kms_atomic 进行 atomic 属性验证
kms_atomic --test-connector
```

### 14. Connector 属性的演进与新增流程

#### 14.1 新增一个 connector 属性的步骤

如果 AMDGPU 驱动需要新增一个自定义 connector 属性，流程如下：

```
Step 1: 在 amdgpu_dm.h 中定义枚举/常量
    enum amdgpu_new_feature {
        AMDGPU_NEW_FEATURE_OFF = 0,
        AMDGPU_NEW_FEATURE_ON  = 1,
    };

Step 2: 在 struct dm_connector_state 中添加字段
    struct dm_connector_state {
        ...
        enum amdgpu_new_feature new_feature;
    };

Step 3: 在 dm_connector_reset 中初始化
    dm_state->new_feature = AMDGPU_NEW_FEATURE_OFF;

Step 4: 在 dp_connector_atomic_set/get_property 中添加处理
    if (property == dev->mode_config.new_feature_property) {
        dm_state->new_feature = val;
        return 0;
    }

Step 5: 在 amdgpu_dm_connector_init 中注册属性并附加
    prop = drm_property_create_enum(dev, 0, "new feature",
                    new_feature_list, ARRAY_SIZE(new_feature_list));
    drm_object_attach_property(&connector->base, prop,
                    AMDGPU_NEW_FEATURE_OFF);

Step 6: 在 DC 层实现属性值到硬件配置的转换
    在 stream_update 或 commit 阶段读取 dm_state->new_feature
    并调用 DC 的接口配置硬件

Step 7: 更新文档和用户空间工具
    - 更新 modetest 的已知属性列表
    - 更新 AMDGPU 驱动文档
```

#### 14.2 历史演进时间线

| 内核版本 | 属性相关变更 |
|---------|-------------|
| Linux 3.1 | 引入 `DRM_IOCTL_MODE_GETPROPERTY` 和 `DRM_IOCTL_MODE_SETPROPERTY` |
| Linux 3.9 | AMDGPU 引入 `underscan` 和 `audio` 私有属性 |
| Linux 4.0 | 引入 Atomic Mode Setting（Daniele Vetter 主导） |
| Linux 4.7 | AMDGPU 开始支持 atomic（预准备阶段） |
| Linux 4.10 | AMDGPU DC 支持通过 atomic 接口操作属性 |
| Linux 4.15 | AMDGPU 默认启用 atomic 支持 |
| Linux 5.0 | 引入 `VRR_CAPABLE` 标准属性 |
| Linux 5.6 | 引入 `HDR_OUTPUT_METADATA` blob 属性 |
| Linux 5.11 | 引入 `adaptive_sync` 枚举属性 |
| Linux 6.0 | AMDGPU 增加 `colorspace` 属性支持 |
| Linux 6.3 | 改进 max_bpc 属性的 atomic 处理逻辑 |
| Linux 6.6 | AMDGPU 增加 content_type 属性支持 |

### 15. 相关链接

- **DRM property 核心实现**: [drivers/gpu/drm/drm_property.c](https://elixir.bootlin.com/linux/v6.6/source/drivers/gpu/drm/drm_property.c)
- **Connector 标准属性注册**: [drivers/gpu/drm/drm_connector.c](https://elixir.bootlin.com/linux/v6.6/source/drivers/gpu/drm/drm_connector.c)
- **DRM 对象属性头文件**: [include/drm/drm_property.h](https://elixir.bootlin.com/linux/v6.6/source/include/drm/drm_property.h)
- **UAPI 属性常量定义**: [include/uapi/drm/drm_mode.h](https://elixir.bootlin.com/linux/v6.6/source/include/uapi/drm/drm_mode.h)
- **AMDGPU DM connector 实现**: [drivers/gpu/drm/amd/amdgpu/amdgpu_dm.c](https://elixir.bootlin.com/linux/v6.6/source/drivers/gpu/drm/amd/amdgpu/amdgpu_dm.c)
- **AMDGPU DM connector 头文件**: [drivers/gpu/drm/amd/amdgpu/amdgpu_dm.h](https://elixir.bootlin.com/linux/v6.6/source/drivers/gpu/drm/amd/amdgpu/amdgpu_dm.h)
- **libdrm 用户空间 API**: [libdrm 源码](https://gitlab.freedesktop.org/mesa/drm)
- **IGT kms_properties 测试**: [IGT kms_properties](https://gitlab.freedesktop.org/drm/igt-gpu-tools/-/tree/master/tests/kms_properties.c)
- **AMD FreeSync 技术文档**: [GPUOpen FreeSync](https://www.amd.com/en/technologies/freesync)
- **DRM Atomic Mode Setting 文档**: [kernel.org](https://docs.kernel.org/gpu/drm-kms.html)

## 今日小结

Day 361 深入分析了 DRM connector 属性的完整体系：

1. **属性框架**: `drm_property` 数据结构支持 RANGE、ENUM、BITMASK、BLOB、OBJECT 等类型，每种类型适用于不同的配置场景。属性通过 `drm_object_attach_property()` 附加到 connector 对象上。

2. **标准属性**: DPMS、EDID 由 DRM 核心自动注册；link-status、broadcast rgb、scaling mode、max bpc、content type 等由驱动注册但广泛使用。

3. **AMDGPU 私有属性**: AMDGPU DC 驱动注册了大量专有属性——underscan（过扫描）、audio（音频输出控制）、colorspace（色彩空间选择）、VRR/FreeSync 相关属性等。这些属性通过 `dm_connector_state` 结构体映射到 DC 层的流配置。

4. **Atomic vs Legacy**: Atomic API 支持在单个提交中同时修改多个属性并实现原子切换，解决了 legacy 接口的时序问题。AMDGPU 驱动完全支持 atomic，所有属性通过 `atomic_set_property` / `atomic_get_property` 回调处理。

5. **用户空间接口**: 通过 libdrm 的 `drmModeObjectGetProperties` / `drmModeAtomicCommit` 或命令行工具（xrandr / modetest / wlr-randr）访问。

**下一日预告**: Day 362 将讲解 libdrm 用户空间库基础——包括 API 架构、核心数据结构、与内核 ioctl 的映射关系，以及 libdrm 在 AMDGPU 测试中的应用。

## 扩展思考
- AMDGPU 驱动 connector 属性数量已接近 DRM 核心的 24 个上限（`DRM_OBJECT_MAX_PROPERTY`），如果新增更多私有属性，应该使用什么方案绕过这个限制？
- Connector 属性与 Plane 属性在 atomic check 中可能出现依赖关系（如 max_bpc 影响 plane 的 pixel format）。AMDGPU 如何正确处理这种跨对象的属性依赖验证？
