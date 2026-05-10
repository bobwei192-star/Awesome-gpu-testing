# Day 362：libdrm 用户空间库基础

## 目录

1. libdrm 概述与架构定位
2. 核心数据结构
3. 资源枚举 API
4. CRTC / Encoder / Connector / Plane 查询
5. Framebuffer 管理
6. 内存管理（Dumb Buffer / GEM / DMABUF）
7. Atomic API 详解
8. Event 处理机制
9. AMDGPU 驱动专用 libdrm 接口
10. 从 libdrm 到内核 ioctl 的映射
11. 实战：编写 libdrm 测试程序
12. 调试与性能分析
13. 常见问题与陷阱
14. 总结与下一日预告

---

## 1. libdrm 概述与架构定位

### 1.1 什么是 libdrm

libdrm 是 DRM（Direct Rendering Manager）子系统的用户空间库，位于 Mesa/GUI 框架与内核 DRM 驱动之间，提供对 DRM ioctl 的封装。

```
┌──────────────────────────────────────────────┐
│              用户空间应用程序                   │
│  (modetest / xrandr / Weston / Xorg / 游戏)   │
├──────────────────────────────────────────────┤
│              libdrm (libdrm.so)               │
│    drmModeGetConnector / drmModeAddFB / ...   │
├──────────────────────────────────────────────┤
│                DRM ioctl 接口                  │
│  DRM_IOCTL_MODE_GETCONNECTOR / DRM_IOCTL_MODE_ADDFB / ...  │
├──────────────────────────────────────────────┤
│           内核 DRM 核心 (drm.ko)               │
│    drm_mode_getconnector / drm_mode_addfb      │
├──────────────────────────────────────────────┤
│          GPU 驱动 (amdgpu.ko)                  │
│    amdgpu_display_get_connector_info           │
└──────────────────────────────────────────────┘
```

**关键设计原则**：
- libdrm 是对 ioctl 的**薄封装**——每个 libdrm 函数几乎直接对应一个 ioctl 调用
- 不提供高级抽象——原子操作、属性查询等直接反映内核接口
- 保持与内核的**二进制兼容性**——使用 ioctl 版本协商机制

### 1.2 libdrm 的历史版本

| 版本 | 发布时间 | 关键变化 |
|------|----------|----------|
| 2.4.0 | 2007 | 初始版本，基本 KMS API |
| 2.4.30 | 2012 | Atomic API 雏形，property 支持 |
| 2.4.60 | 2014 | 完整的 atomic API（drmModeAtomicCommit） |
| 2.4.80 | 2016 | 增加 syncobj 同步对象支持 |
| 2.4.100 | 2019 | 增加 HDMI 2.1 VRR / ALLM 支持 |
| 2.4.110 | 2022 | 增加 DP 2.0 UHBR 支持，色彩管理 API |
| 2.4.120 | 2025 | 最新的 HDR / 色彩元数据支持 |

### 1.3 libdrm 的构成

libdrm 由多个组件构成：

```
libdrm/
├── xf86drm.c          # 核心封装：drmOpen, drmIoctl, 设备管理
├── xf86drmMode.c      # KMS/Cursor 相关 API
├── xf86drm.h          # 核心头文件（IO 控制、设备信息）
├── drm.h               # 内核 DRM 接口定义（用户空间版本）
├── drm_mode.h          # KMS 模式定义（用户空间版本）
├── libdrm_macros.h     # 可见性/导出宏
├── amdgpu/             # AMDGPU 专用驱动库
│   ├── amdgpu_device.c
│   ├── amdgpu_bo.c
│   ├── amdgpu_cs.c     # Command Submission（命令提交）
│   └── amdgpu_vm.c     # 虚拟内存管理
├── intel/              # Intel 专用驱动库
├── radeon/             # Radeon 专用驱动库
├── nouveau/            # Nouveau 专用驱动库
├── etnaviv/            # Etnaviv 专用驱动库
└── tests/              # 测试程序（modetest, kmstest 等）
```

### 1.4 编译与链接

```bash
# 编译时
gcc -c my_drm_app.c -I/usr/include/libdrm

# 链接时
gcc my_drm_app.o -ldrm -o my_drm_app

# AMDGPU 专用 API 需要额外的库
gcc my_amdgpu_app.o -ldrm -ldrm_amdgpu -o my_amdgpu_app

# pkg-config 方式（推荐）
CFLAGS=$(pkg-config --cflags libdrm)
LIBS=$(pkg-config --libs libdrm)
gcc my_drm_app.c $CFLAGS $LIBS -o my_drm_app
```

---

## 2. 核心数据结构

### 2.1 drmModeRes——显示资源

`drmModeGetResources()` 返回整个 DRM 设备的资源列表：

```c
typedef struct _drmModeRes {
    int count_fbs;             // Framebuffer 数量
    uint32_t *fbs;             // Framebuffer ID 数组
    int count_crtcs;           // CRTC 数量
    uint32_t *crtcs;           // CRTC ID 数组
    int count_connectors;      // Connector 数量
    uint32_t *connectors;      // Connector ID 数组
    int count_encoders;        // Encoder 数量
    uint32_t *encoders;        // Encoder ID 数组
    uint32_t min_width, max_width;    // 支持的宽度范围
    uint32_t min_height, max_height;  // 支持的高度范围
} drmModeRes, *drmModeResPtr;
```

```c
// 使用示例
drmModeRes *res = drmModeGetResources(fd);
if (!res) {
    fprintf(stderr, "Failed to get DRM resources\n");
    return -1;
}

printf("FBs: %d, CRTCs: %d, Connectors: %d, Encoders: %d\n",
       res->count_fbs, res->count_crtcs,
       res->count_connectors, res->count_encoders);

for (int i = 0; i < res->count_crtcs; i++) {
    printf("  CRTC[%d] ID: %u\n", i, res->crtcs[i]);
}

drmModeFreeResources(res);
```

### 2.2 drmModeConnector——连接器

```c
typedef struct _drmModeConnector {
    uint32_t connector_id;            // 连接器 ID
    uint32_t encoder_id;              // 当前绑定的 Encoder ID
    uint32_t connector_type;          // 连接器类型
    uint32_t connector_type_id;       // 相同类型的序号
    drmModeConnection connection;     // 连接状态
    uint32_t mmWidth, mmHeight;       // 物理尺寸（毫米）
    drmModeSubPixel subpixel;         // 子像素排列
    int count_modes;                  // 可用显示模式数量
    drmModeModeInfo *modes;           // 显示模式数组
    int count_props;                  // 属性数量
    uint32_t *props;                  // 属性 ID 数组
    uint64_t *prop_values;            // 属性值数组
    int count_encoders;               // 可用 Encoder 数量
    uint32_t *encoders;               // 可用 Encoder ID 数组
} drmModeConnector, *drmModeConnectorPtr;
```

**connector_type 常量**：

| 类型常量 | 值 | 说明 |
|----------|-----|------|
| DRM_MODE_CONNECTOR_Unknown | 0 | 未知类型 |
| DRM_MODE_CONNECTOR_VGA | 1 | VGA |
| DRM_MODE_CONNECTOR_DVII | 2 | DVI-I |
| DRM_MODE_CONNECTOR_DVID | 3 | DVI-D |
| DRM_MODE_CONNECTOR_DVIA | 4 | DVI-A |
| DRM_MODE_CONNECTOR_Composite | 5 | 复合视频 |
| DRM_MODE_CONNECTOR_SVIDEO | 6 | S-Video |
| DRM_MODE_CONNECTOR_LVDS | 7 | LVDS（笔记本内置） |
| DRM_MODE_CONNECTOR_Component | 8 | 色差分量 |
| DRM_MODE_CONNECTOR_9PinDIN | 9 | 9针 DIN |
| DRM_MODE_CONNECTOR_DisplayPort | 10 | DisplayPort |
| DRM_MODE_CONNECTOR_HDMIA | 11 | HDMI Type A |
| DRM_MODE_CONNECTOR_HDMIB | 12 | HDMI Type B |
| DRM_MODE_CONNECTOR_TV | 13 | TV |
| DRM_MODE_CONNECTOR_eDP | 14 | Embedded DP（笔记本内置） |
| DRM_MODE_CONNECTOR_VIRTUAL | 15 | 虚拟连接器 |
| DRM_MODE_CONNECTOR_DSI | 16 | DSI |
| DRM_MODE_CONNECTOR_DPI | 17 | DPI |
| DRM_MODE_CONNECTOR_WRITEBACK | 18 | 回写连接器 |
| DRM_MODE_CONNECTOR_SPI | 19 | SPI |
| DRM_MODE_CONNECTOR_USB | 20 | USB |

### 2.3 drmModeCrtc——CRTC

```c
typedef struct _drmModeCrtc {
    uint32_t crtc_id;                  // CRTC ID
    uint32_t buffer_id;                // 当前显示的 Framebuffer ID
    uint32_t crtc_x, crtc_y;           // CRTC 在 Framebuffer 中的位置
    uint32_t gamma_size;               // Gamma 表大小
    int mode_valid;                    // 当前模式是否有效
    drmModeModeInfo mode;              // 当前显示模式
} drmModeCrtc, *drmModeCrtcPtr;
```

### 2.4 drmModeEncoder——编码器

```c
typedef struct _drmModeEncoder {
    uint32_t encoder_id;              // Encoder ID
    uint32_t encoder_type;            // 编码器类型（TMDS/LVDS/DP 等）
    uint32_t possible_crtcs;          // 位掩码，表示可绑定的 CRTC
    uint32_t possible_clones;         // 位掩码，表示可克隆的 Encoder
} drmModeEncoder, *drmModeEncoderPtr;
```

**possible_crtcs 详解**：
- 位掩码中的每个 bit 对应一个 CRTC 索引
- Bit 0 = CRTC 0，Bit 1 = CRTC 1，依此类推
- 例如：`possible_crtcs = 0x05`（二进制 0101）表示可绑定到 CRTC 0 和 CRTC 2

### 2.5 drmModeModeInfo——显示模式

```c
typedef struct _drmModeModeInfo {
    uint32_t clock;                    // 像素时钟（kHz）
    uint16_t hdisplay, hsync_start;    // 水平：显示起始、同步起始
    uint16_t hsync_end, htotal;        // 水平：同步结束、总长度
    uint16_t vdisplay, vsync_start;    // 垂直：显示起始、同步起始
    uint16_t vsync_end, vtotal;        // 垂直：同步结束、总长度
    uint32_t vrefresh;                 // 刷新率（Hz）
    uint32_t flags;                    // 标志位（隔行扫描、同步极性等）
    uint32_t type;                     // 模式类型（首选、标准、用户定义等）
    char name[DRM_DISPLAY_MODE_LEN];   // 模式名称，如 "1920x1080"
} drmModeModeInfo, *drmModeModeInfoPtr;
```

**flags 常用值**：
```c
#define DRM_MODE_FLAG_PHSYNC    (1<<0)  // 水平同步极性：正
#define DRM_MODE_FLAG_NHSYNC    (1<<1)  // 水平同步极性：负
#define DRM_MODE_FLAG_PVSYNC    (1<<2)  // 垂直同步极性：正
#define DRM_MODE_FLAG_NVSYNC    (1<<3)  // 垂直同步极性：负
#define DRM_MODE_FLAG_INTERLACE (1<<4)  // 隔行扫描
#define DRM_MODE_FLAG_DBLSCAN   (1<<5)  // 双倍扫描
#define DRM_MODE_FLAG_CSYNC     (1<<6)  // 复合同步
#define DRM_MODE_FLAG_PCSYNC    (1<<7)  // 复合同步：正
#define DRM_MODE_FLAG_NCSYNC    (1<<8)  // 复合同步：负
#define DRM_MODE_FLAG_HSKEW     (1<<9)  // 水平偏移
#define DRM_MODE_FLAG_BCAST     (1<<10) // 广播模式
#define DRM_MODE_FLAG_3D_MASK   (0x1f<<14) // 3D 格式掩码
```

### 2.6 drmModeFB——Framebuffer

```c
typedef struct _drmModeFB {
    uint32_t fb_id;                    // Framebuffer ID
    uint32_t width, height;            // 宽高（像素）
    uint32_t pitch;                    // 行跨度（字节）
    uint32_t bpp;                      // 每像素位数
    uint32_t depth;                    // 色深
    uint32_t handle;                   // GEM handle
} drmModeFB, *drmModeFBPtr;
```

**新版扩展（drmModeFB2）**：
```c
typedef struct _drmModeFB2 {
    uint32_t fb_id;
    uint32_t width, height;
    uint32_t pixel_format;             // 四像素格式码（DRM_FORMAT_XRGB8888 等）
    uint64_t modifier;                 // 显存布局修饰符（tiling/VAR/GFX9 等）
    uint32_t handles[4];               // 每个 plane 的 GEM handle
    uint32_t pitches[4];              // 每个 plane 的行跨度
    uint32_t offsets[4];              // 每个 plane 的偏移
} drmModeFB2, *drmModeFB2Ptr;
```

### 2.7 drmModePlane / drmModePlaneRes——图层

```c
typedef struct _drmModePlane {
    uint32_t count_formats;
    uint32_t *formats;                 // 支持的像素格式列表
    uint32_t plane_id;
    uint32_t crtc_id;                  // 当前绑定的 CRTC
    uint32_t fb_id;                    // 当前使用的 FB
    uint32_t crtc_x, crtc_y;           // CRTC 中的位置
    uint32_t x, y;                     // Framebuffer 中的位置
    uint32_t possible_crtcs;           // 可绑定的 CRTC 位掩码
    uint32_t gamma_size;               // Gamma 表大小
} drmModePlane, *drmModePlanePtr;
```

### 2.8 drmModeProperty——属性

```c
typedef struct _drmModeProperty {
    uint32_t prop_id;                  // 属性 ID
    uint32_t flags;                    // 属性类型标志
    char name[DRM_PROP_NAME_LEN];      // 属性名称
    int count_values;
    uint64_t *values;                  // 属性值（RANGE 类型）
    struct drm_mode_property_enum {
        uint64_t value;
        char name[DRM_PROP_ENUM_NAME_LEN];
    } *enums;                          // 枚举值（ENUM 类型）
    int count_enums;
    int count_blobs;
} drmModeProperty, *drmModePropertyPtr;
```

### 2.9 drmModeAtomicReq——Atomic 请求

```c
// Atomic request 是不透明结构体，通过 API 操作
typedef struct _drmModeAtomicReq drmModeAtomicReq, *drmModeAtomicReqPtr;

// 创建 atomic request
drmModeAtomicReqPtr drmModeAtomicAlloc(void);

// 添加属性修改
int drmModeAtomicAddProperty(drmModeAtomicReqPtr req,
                              uint32_t object_id,
                              uint32_t property_id,
                              uint64_t value);

// 提交 atomic request
int drmModeAtomicCommit(int fd, drmModeAtomicReqPtr req,
                        uint32_t flags, void *user_data);

// 释放 atomic request
void drmModeAtomicFree(drmModeAtomicReqPtr req);
```

---

## 3. 资源枚举 API

### 3.1 drmModeGetResources——获取全局资源

```c
// 函数原型
drmModeResPtr drmModeGetResources(int fd);

// 内核态对应的 ioctl：DRM_IOCTL_MODE_GETRESOURCES
// 返回系统中所有可用的 CRTC、Encoder、Connector、FB 列表

// 典型用法：遍历所有连接器
drmModeRes *res = drmModeGetResources(fd);
for (int i = 0; i < res->count_connectors; i++) {
    drmModeConnector *conn = drmModeGetConnector(fd, res->connectors[i]);
    // 处理连接器
    drmModeFreeConnector(conn);
}
drmModeFreeResources(res);
```

### 3.2 drmModeGetConnector——获取连接器详细信息

```c
drmModeConnectorPtr drmModeGetConnector(int fd, uint32_t connector_id);

// 内核对应：DRM_IOCTL_MODE_GETCONNECTOR
// 返回包括 EDID 属性、可用显示模式、连接状态等完整信息

// 注意：此 ioctl 可能触发 EDID 读取（首次调用时）
// 在热拔插场景中应重新获取以获取最新信息
```

### 3.3 drmModeGetConnectorCurrent——获取连接器当前状态

```c
// 较新版本 libdrm 新增的 API
drmModeConnectorPtr drmModeGetConnectorCurrent(int fd, uint32_t connector_id);

// 与 drmModeGetConnector 的区别：
// - GetConnector：读取 EDID 并返回所有可用模式
// - GetConnectorCurrent：仅返回当前已缓存的状态，不触发 EDID 读取
// - 更适合频繁查询场景（如热拔插监控）
```

### 3.4 drmModeGetEncoder——获取编码器信息

```c
drmModeEncoderPtr drmModeGetEncoder(int fd, uint32_t encoder_id);

// 内核对应：DRM_IOCTL_MODE_GETENCODER
// 获取编码器类型、可能绑定的 CRTC 等信息
```

### 3.5 drmModeGetCrtc——获取 CRTC 状态

```c
drmModeCrtcPtr drmModeGetCrtc(int fd, uint32_t crtc_id);

// 内核对应：DRM_IOCTL_MODE_GETCRTC
// 返回 CRTC 当前模式、显示的 Framebuffer 等
```

### 3.6 drmModeGetPlaneResources / drmModeGetPlane——获取图层信息

```c
drmModePlaneResPtr drmModeGetPlaneResources(int fd);
drmModePlanePtr drmModeGetPlane(int fd, uint32_t plane_id);

// 内核对应：DRM_IOCTL_MODE_GETPLANERESOURCES / DRM_IOCTL_MODE_GETPLANE
// 列出所有可用的硬件图层（overlay / primary / cursor）
```

### 3.7 完整枚举示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

static const char *connector_type_str(uint32_t type) {
    switch (type) {
        case DRM_MODE_CONNECTOR_DisplayPort: return "DP";
        case DRM_MODE_CONNECTOR_HDMIA:      return "HDMI-A";
        case DRM_MODE_CONNECTOR_eDP:        return "eDP";
        case DRM_MODE_CONNECTOR_LVDS:       return "LVDS";
        default:                            return "Other";
    }
}

int main(int argc, char **argv) {
    const char *card = argv[1] ? argv[1] : "/dev/dri/card0";
    int fd = open(card, O_RDWR | O_CLOEXEC);
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

    printf("DRM Resources for %s:\n", card);
    printf("  CRTCs:   %d\n", res->count_crtcs);
    printf("  Encoders: %d\n", res->count_encoders);
    printf("  Connectors: %d\n", res->count_connectors);
    printf("  FBs:     %d\n", res->count_fbs);
    printf("  Size:    %d x %d  to  %d x %d\n\n",
           res->min_width, res->min_height,
           res->max_width, res->max_height);

    // 遍历连接器
    for (int i = 0; i < res->count_connectors; i++) {
        drmModeConnector *conn = drmModeGetConnector(fd, res->connectors[i]);
        if (!conn) continue;

        printf("Connector %d (%s-%d):\n",
               conn->connector_id,
               connector_type_str(conn->connector_type),
               conn->connector_type_id);
        printf("  Status: %s\n",
               conn->connection == DRM_MODE_CONNECTED ? "Connected" :
               conn->connection == DRM_MODE_DISCONNECTED ? "Disconnected" :
               "Unknown");
        printf("  Modes: %d\n", conn->count_modes);
        printf("  Physical: %dmm x %dmm\n",
               conn->mmWidth, conn->mmHeight);

        // 打印前 3 个模式
        for (int m = 0; m < conn->count_modes && m < 3; m++) {
            printf("  Mode[%d]: %s @ %dHz (%s)\n",
                   m, conn->modes[m].name, conn->modes[m].vrefresh,
                   conn->modes[m].type & DRM_MODE_TYPE_PREFERRED ?
                       "preferred" : "normal");
        }

        drmModeFreeConnector(conn);
    }

    drmModeFreeResources(res);
    close(fd);
    return 0;
}
```

**编译与运行**：
```bash
gcc enum_display.c -ldrm -o enum_display
./enum_display /dev/dri/card0
```

**预期输出示例**（AMDGPU 系统）：
```
DRM Resources for /dev/dri/card0:
  CRTCs:   6
  Encoders: 6
  Connectors: 4
  FBs:     0
  Size:    0 x 0  to  16384 x 16384

Connector 74 (DP-1):
  Status: Disconnected
  Modes: 0
  Physical: 0mm x 0mm
Connector 75 (DP-2):
  Status: Connected
  Modes: 17
  Physical: 527mm x 296mm
  Mode[0]: 1920x1080 @ 60Hz (preferred)
  Mode[1]: 1920x1080 @ 50Hz (normal)
  Mode[2]: 1920x1080 @ 59Hz (normal)
Connector 76 (HDMI-A-1):
  Status: Connected
  Modes: 8
  Physical: 1600mm x 900mm
  ...
```

---

## 4. CRTC / Encoder / Connector / Plane 查询

### 4.1 连接器-编码器-CRTC 绑定关系

理解 KMS 中连接器、编码器、CRTC 的绑定关系是关键：

```
Connector (物理接口)
    │
    ▼
Encoder (信号编码器)
    │  possible_crtcs 位掩码决定可绑定到哪些 CRTC
    ▼
CRTC (时序生成器)
    │
    ▼
Plane (图层，关联 Framebuffer)
    │
    ▼
Framebuffer (内存中的像素数据)
```

```c
// 查找连接器对应的 CRTC
static uint32_t find_crtc_for_connector(int fd,
                                          drmModeRes *res,
                                          drmModeConnector *conn) {
    // 方案 1：如果连接器已绑定编码器，检查编码器的 CRTC
    if (conn->encoder_id) {
        drmModeEncoder *enc = drmModeGetEncoder(fd, conn->encoder_id);
        if (enc) {
            // possible_crtcs 位掩码指示可用的 CRTC 索引
            for (int i = 0; i < res->count_crtcs; i++) {
                if (enc->possible_crtcs & (1 << i)) {
                    uint32_t crtc_id = res->crtcs[i];
                    drmModeFreeEncoder(enc);
                    return crtc_id;
                }
            }
            drmModeFreeEncoder(enc);
        }
    }

    // 方案 2：遍历连接器的所有编码器
    for (int i = 0; i < conn->count_encoders; i++) {
        drmModeEncoder *enc = drmModeGetEncoder(fd, conn->encoders[i]);
        if (!enc) continue;
        for (int j = 0; j < res->count_crtcs; j++) {
            if (enc->possible_crtcs & (1 << j)) {
                uint32_t crtc_id = res->crtcs[j];
                drmModeFreeEncoder(enc);
                return crtc_id;
            }
        }
        drmModeFreeEncoder(enc);
    }

    return 0; // 未找到可用 CRTC
}
```

### 4.2 查找最佳的显示模式

```c
// 查找首选模式，若没有首选则返回第一个模式
static drmModeModeInfo *find_best_mode(drmModeConnector *conn) {
    // 首选模式
    for (int i = 0; i < conn->count_modes; i++) {
        if (conn->modes[i].type & DRM_MODE_TYPE_PREFERRED)
            return &conn->modes[i];
    }

    // 如果没有首选模式，返回分辨率最大的模式
    drmModeModeInfo *best = NULL;
    for (int i = 0; i < conn->count_modes; i++) {
        if (!best ||
            (conn->modes[i].hdisplay > best->hdisplay) ||
            (conn->modes[i].hdisplay == best->hdisplay &&
             conn->modes[i].vdisplay > best->vdisplay)) {
            best = &conn->modes[i];
        }
    }
    return best;
}

// 按指定分辨率查找模式
static drmModeModeInfo *find_mode_by_res(drmModeConnector *conn,
                                           int width, int height) {
    for (int i = 0; i < conn->count_modes; i++) {
        if (conn->modes[i].hdisplay == width &&
            conn->modes[i].vdisplay == height) {
            return &conn->modes[i];
        }
    }
    return NULL;
}

// 按刷新率查找模式
static drmModeModeInfo *find_mode_by_refresh(drmModeConnector *conn,
                                               int width, int height,
                                               int refresh_hz) {
    for (int i = 0; i < conn->count_modes; i++) {
        if (conn->modes[i].hdisplay == width &&
            conn->modes[i].vdisplay == height &&
            conn->modes[i].vrefresh == refresh_hz) {
            return &conn->modes[i];
        }
    }
    return NULL;
}
```

### 4.3 查询 Plane 信息

```c
// 遍历所有 Plane
drmModePlaneRes *plane_res = drmModeGetPlaneResources(fd);
if (plane_res) {
    for (int i = 0; i < plane_res->count_planes; i++) {
        drmModePlane *plane = drmModeGetPlane(fd, plane_res->planes[i]);
        if (!plane) continue;

        // 确定图层类型
        const char *type_str = "unknown";
        for (int p = 0; p < plane_res->count_planes; p++) {
            // 通过检查 possible_crtcs 和格式集判断
            // Primary plane 通常支持 DRM_FORMAT_XRGB8888 等基本格式
            // Cursor plane 支持较少格式且尺寸受限
            // Overlay plane 支持额外格式（如 NV12/YUV）
        }

        printf("Plane %d: CRTC=%u FB=%u pos=(%d,%d) crtc_pos=(%d,%d)\n",
               plane->plane_id, plane->crtc_id, plane->fb_id,
               plane->x, plane->y,
               plane->crtc_x, plane->crtc_y);
        printf("  Formats: ");
        for (int f = 0; f < plane->count_formats; f++) {
            char fmt[5] = {0};
            fmt[0] = (plane->formats[f] >> 0) & 0xFF;
            fmt[1] = (plane->formats[f] >> 8) & 0xFF;
            fmt[2] = (plane->formats[f] >> 16) & 0xFF;
            fmt[3] = (plane->formats[f] >> 24) & 0xFF;
            printf("%s ", fmt);
        }
        printf("\n");

        drmModeFreePlane(plane);
    }
    drmModeFreePlaneResources(plane_res);
}
```

**Plane 类型判断**（通过属性 type 而非字段）：
```c
// 获取 plane 的 "type" 属性值来判断
static int get_plane_type(int fd, uint32_t plane_id) {
    drmModeObjectProperties *props =
        drmModeObjectGetProperties(fd, plane_id, DRM_MODE_OBJECT_PLANE);
    if (!props) return -1;

    int type = -1;
    for (int i = 0; i < props->count_props; i++) {
        drmModeProperty *prop = drmModeGetProperty(fd, props->props[i]);
        if (prop && strcmp(prop->name, "type") == 0) {
            type = (int)props->prop_values[i];
        }
        if (prop) drmModeFreeProperty(prop);
    }
    drmModeFreeObjectProperties(props);

    // 内核定义：DRM_PLANE_TYPE_OVERLAY=0, PRIMARY=1, CURSOR=2
    return type;
}
```

---

## 5. Framebuffer 管理

### 5.1 创建 Framebuffer——drmModeAddFB

```c
// Legacy API（仅支持基本格式）
int drmModeAddFB(int fd, uint32_t width, uint32_t height,
                 uint8_t depth, uint8_t bpp, uint32_t pitch,
                 uint32_t bo_handle, uint32_t *buf_id);

// 注意：
// - depth 和 bpp 的组合必须符合内核预定义的表（如 bpp=32, depth=24 对应 XRGB8888）
// - pitch 是行跨度（字节），必须 >= width * (bpp / 8)
// - bo_handle 是 GEM buffer object handle
```

**支持的 depth/bpp 组合**：
```c
// 内核预定义的标准格式
{ .depth = 8,  .bpp = 8  },  // Indexed 8-bit
{ .depth = 15, .bpp = 16 },  // XRGB1555
{ .depth = 16, .bpp = 16 },  // RGB565
{ .depth = 24, .bpp = 32 },  // XRGB8888 最常见
{ .depth = 30, .bpp = 32 },  // XRGB2101010（HDR 支持）
```

### 5.2 创建 Framebuffer——drmModeAddFB2（推荐）

```c
// 新版 API，支持像素格式四码和修饰符
int drmModeAddFB2(int fd, uint32_t width, uint32_t height,
                  uint32_t pixel_format,
                  uint32_t bo_handles[4],
                  uint32_t pitches[4],
                  uint32_t offsets[4],
                  uint32_t *buf_id,
                  uint32_t flags);

// 带修饰符的版本
int drmModeAddFB2WithModifiers(int fd, uint32_t width, uint32_t height,
                               uint32_t pixel_format,
                               uint32_t bo_handles[4],
                               uint32_t pitches[4],
                               uint32_t offsets[4],
                               uint64_t modifier[4],
                               uint32_t *buf_id,
                               uint32_t flags);
```

**常用像素格式四码**：
```c
// include/drm/drm_fourcc.h
#define DRM_FORMAT_XRGB8888  fourcc_code('X', 'R', '2', '4') // 0x34325258
#define DRM_FORMAT_ARGB8888  fourcc_code('A', 'R', '2', '4') // 0x34325241
#define DRM_FORMAT_XRGB2101010 fourcc_code('X', 'R', '3', '0') // 0x30335258
#define DRM_FORMAT_NV12      fourcc_code('N', 'V', '1', '2') // 0x3231564E
#define DRM_FORMAT_P010      fourcc_code('P', '0', '1', '0') // 0x30313050
#define DRM_FORMAT_ARGB16161616F fourcc_code('A', 'R', '6', 'F') // HDR FP16
```

### 5.3 Framebuffer 处理函数族

```c
// 创建 FB
int drmModeAddFB(int fd, uint32_t width, uint32_t height,
                 uint8_t depth, uint8_t bpp, uint32_t pitch,
                 uint32_t bo_handle, uint32_t *buf_id);

int drmModeAddFB2(int fd, uint32_t width, uint32_t height,
                  uint32_t pixel_format,
                  uint32_t bo_handles[4], uint32_t pitches[4],
                  uint32_t offsets[4], uint32_t *buf_id,
                  uint32_t flags);

int drmModeAddFB2WithModifiers(int fd, uint32_t width, uint32_t height,
                               uint32_t pixel_format,
                               uint32_t bo_handles[4], uint32_t pitches[4],
                               uint32_t offsets[4], uint64_t modifier[4],
                               uint32_t *buf_id, uint32_t flags);

// 删除 FB
int drmModeRmFB(int fd, uint32_t buffer_id);

// 标记 FB 脏区域（用于手动更新模式）
int drmModeDirtyFB(int fd, uint32_t buffer_id,
                   drmModeClipPtr clips, uint32_t num_clips);

// 查询 FB 信息（两个版本）
drmModeFBPtr drmModeGetFB(int fd, uint32_t buf_id);     // 旧版
drmModeFB2Ptr drmModeGetFB2(int fd, uint32_t buf_id);   // 新版
```

### 5.4 完整 Framebuffer 创建流程

```c
// 1. 分配 GEM buffer object（dumb buffer）
uint32_t handle;
uint32_t stride;
uint64_t size;
int ret;

struct drm_mode_create_dumb create = {
    .width = 1920,
    .height = 1080,
    .bpp = 32,
    .flags = 0,
};
ret = drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);
if (ret) { perror("CREATE_DUMB"); return -1; }

handle = create.handle;
stride = create.pitch;
size = create.size;
printf("Created dumb buffer: handle=%u stride=%u size=%lu\n",
       handle, stride, size);

// 2. 创建 Framebuffer
uint32_t fb_id;
ret = drmModeAddFB(fd, 1920, 1080, 24, 32, stride, handle, &fb_id);
if (ret) { perror("AddFB"); goto cleanup; }
printf("Created FB: %u\n", fb_id);

// 3. 将 GEM BO 映射到用户空间（用于绘制像素）
struct drm_mode_map_dumb map = { .handle = handle };
ret = drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map);
if (ret) { perror("MAP_DUMB"); goto cleanup; }

uint32_t *fb_ptr = mmap(0, size, PROT_READ | PROT_WRITE,
                        MAP_SHARED, fd, map.offset);
if (fb_ptr == MAP_FAILED) { perror("mmap"); goto cleanup; }

// 4. 绘制像素（填充红色）
for (int y = 0; y < 1080; y++) {
    for (int x = 0; x < 1920; x++) {
        fb_ptr[y * (stride / 4) + x] = 0x00FF0000; // XRGB: R=255
    }
}

// 5. Cleanup
munmap(fb_ptr, size);
drmModeRmFB(fd, fb_id);
struct drm_mode_destroy_dumb destroy = { .handle = handle };
drmIoctl(fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
```

---

## 6. 内存管理（Dumb Buffer / GEM / DMABUF）

### 6.1 Dumb Buffer——最简单的分配方式

Dumb Buffer 是 DRM 提供的最简单的缓存分配方式，适用于不需要 GPU 渲染的简单显示应用（如显示测试工具）。

```c
// 头文件定义位于 <drm/drm.h>
struct drm_mode_create_dumb {
    __u32 height;
    __u32 width;
    __u32 bpp;          // 每像素位数
    __u32 flags;        // 标志位（通常为 0）
    __u32 handle;       // [返回] GEM handle
    __u32 pitch;        // [返回] 行跨度（字节）
    __u64 size;         // [返回] BO 总大小（字节）
};

struct drm_mode_map_dumb {
    __u32 handle;       // GEM handle
    __u32 pad;          // 填充
    __u64 offset;       // [返回] mmap 偏移量
};

struct drm_mode_destroy_dumb {
    __u32 handle;
};

// 对应的 ioctl 命令
#define DRM_IOCTL_MODE_CREATE_DUMB  DRM_IOWR(0xB2, struct drm_mode_create_dumb)
#define DRM_IOCTL_MODE_MAP_DUMB     DRM_IOWR(0xB3, struct drm_mode_map_dumb)
#define DRM_IOCTL_MODE_DESTROY_DUMB DRM_IOW(0xB4, struct drm_mode_destroy_dumb)
```

**Dumb Buffer 的局限性**：
- 不支持 tiling（瓦片化）等 GPU 优化内存布局
- 性能较低，不适合高帧率场景
- 仅适用于小尺寸显示缓冲区
- 部分现代 GPU 可能不支持大尺寸 dumb buffer

### 6.2 GEM——通用图形内存管理

GEM（Graphics Execution Manager）是 DRM 的通用内存管理框架：

```c
// 创建 GEM BO（driver-specific ioctl，不是标准 DRM 命令）
// 对于 AMDGPU，使用 amdgpu_bo_alloc() 替代 dumb buffer

// 通用 GEM 操作
struct drm_gem_open {
    __u32 name;        // 需打开的 GEM 名称
    __u32 handle;      // [返回] GEM handle
    __u64 size;        // [返回] BO 大小
};

struct drm_gem_close {
    __u32 handle;
    __u32 pad;
};

#define DRM_IOCTL_GEM_CLOSE    DRM_IOW(0x09, struct drm_gem_close)
#define DRM_IOCTL_GEM_OPEN     DRM_IOWR(0x0B, struct drm_gem_open)

// GEM FLINK——在不同进程间共享 BO
// 可以通过 drm_gem_flink 创建全局名称，然后其他进程通过 drm_gem_open 打开
struct drm_gem_flink {
    __u32 handle;
    __u32 name;        // [返回] 全局名称
};

#define DRM_IOCTL_GEM_FLINK    DRM_IOWR(0x0A, struct drm_gem_flink)
```

### 6.3 DMABUF——跨设备缓存共享

DMABUF（DMA Buffer）是现代 Linux 上跨设备和跨驱动共享缓存的标准机制：

```c
// 从 GEM handle 获取 DMABUF fd
int drmPrimeHandleToFD(int fd, uint32_t handle,
                        uint32_t flags, int *prime_fd);

// 从 DMABUF fd 获取 GEM handle
int drmPrimeFDToHandle(int fd, int prime_fd, uint32_t *handle);

// DMABUF 的优势：
// 1. 零拷贝——无需通过 CPU 复制数据
// 2. 跨设备共享——GPU 到 GPU、GPU 到 V4L2 等
// 3. 使用 fd 作为安全引用计数机制
// 4. 内核自动处理缓存同步

// 使用示例：将 GPU 渲染结果导出给显示控制器
int prime_fd;
drmPrimeHandleToFD(fd, gem_handle, DRM_CLOEXEC, &prime_fd);

// 另一个进程/驱动可通过 prime_fd 导入
uint32_t imported_handle;
drmPrimeFDToHandle(other_fd, prime_fd, &imported_handle);
close(prime_fd);
```

### 6.4 AMDGPU 专用内存管理

AMDGPU 驱动提供了更复杂的内存管理 API（位于 `libdrm/amdgpu/`）：

```c
#include <amdgpu.h>
#include <amdgpu_drm.h>

// 初始化 AMDGPU 设备
amdgpu_device_handle dev;
uint32_t major, minor;
int ret = amdgpu_device_initialize(fd, &major, &minor, &dev);

// 分配 BO（支持多种堆）
amdgpu_bo_handle bo;
amdgpu_bo_alloc_info alloc_info = {
    .alloc_size = 1920 * 1080 * 4,  // 1920x1080x32bpp
    .phys_alignment = 4096,
    .preferred_heap = AMDGPU_GEM_DOMAIN_VRAM, // 优先使用 VRAM
};
amdgpu_bo_alloc(dev, &alloc_info, &bo);

// 获取 BO 信息（包括 VRAM 地址）
amdgpu_bo_info bo_info = {0};
amdgpu_bo_query_info(bo, &bo_info);

// 将 BO 映射到 CPU
int cpu_access;
amdgpu_bo_cpu_map(bo, (void **)&cpu_ptr);

// 获取 BO 的 GEM handle（用于创建 FB）
uint32_t gem_handle;
amdgpu_bo_export(bo, amdgpu_bo_handle_type_gem_flink, &gem_handle);

// 获取 DMABUF fd
int prime_fd;
amdgpu_bo_export(bo, amdgpu_bo_handle_type_dma_buf_fd, &prime_fd);

// 清理
amdgpu_bo_cpu_unmap(bo);
amdgpu_bo_free(bo);
amdgpu_device_deinitialize(dev);
```

**AMDGPU 内存堆类型**：

| 堆常量 | 说明 |
|--------|------|
| AMDGPU_GEM_DOMAIN_VRAM | 显存（GPU 专用，带宽最高） |
| AMDGPU_GEM_DOMAIN_GTT | GTT 内存（系统内存映射到 GPU 地址空间） |
| AMDGPU_GEM_DOMAIN_CPU | CPU 系统内存 |
| AMDGPU_GEM_DOMAIN_GDS | 全局数据共享（GDS，计算专用） |

---

## 7. Atomic API 详解

### 7.1 Atomic 请求生命周期

```
创建 Atomic Request
    │
    ▼
添加属性修改（可多次调用）
    │  drmModeAtomicAddProperty()
    ▼
验证（可选）
    │  drmModeAtomicCommit(flags=DRM_MODE_ATOMIC_TEST_ONLY)
    ▼
提交
    │  drmModeAtomicCommit(flags=DRM_MODE_ATOMIC_ALLOW_MODESET)
    ▼
处理事件（可选）
    │  drmHandleEvent() — 等待完成事件
```

### 7.2 drmModeAtomicAlloc——创建请求

```c
drmModeAtomicReqPtr drmModeAtomicAlloc(void);
// 返回值：分配的请求对象，失败返回 NULL
// 内部预先分配了约 64 个属性的空间
// 如果超出会自动扩展
```

### 7.3 drmModeAtomicAddProperty——添加属性

```c
int drmModeAtomicAddProperty(drmModeAtomicReqPtr req,
                              uint32_t object_id,
                              uint32_t property_id,
                              uint64_t value);

// 参数：
//   req: Atomic 请求对象
//   object_id: 目标对象 ID（CRTC/Connector/Plane 等）
//   property_id: 属性 ID（通过 drmModeGetProperty 获取）
//   value: 属性值
// 返回值：当前请求中的属性总数，失败返回 -1

// 典型用法：一次 atomic 提交设置多个属性
drmModeAtomicReq *req = drmModeAtomicAlloc();

// 设置 CRTC 模式
drmModeAtomicAddProperty(req, crtc_id, crtc_mode_prop_id, ...);
drmModeAtomicAddProperty(req, crtc_id, crtc_active_prop_id, 1);

// 设置 Connector 的 CRTC 绑定
drmModeAtomicAddProperty(req, conn_id, conn_crtc_id_prop_id, crtc_id);

// 设置 Plane 的 FB 和位置
drmModeAtomicAddProperty(req, plane_id, plane_fb_id_prop, fb_id);
drmModeAtomicAddProperty(req, plane_id, plane_crtc_id_prop, crtc_id);
drmModeAtomicAddProperty(req, plane_id, plane_src_x_prop, 0);
drmModeAtomicAddProperty(req, plane_id, plane_src_y_prop, 0);
drmModeAtomicAddProperty(req, plane_id, plane_src_w_prop, 1920 << 16);
drmModeAtomicAddProperty(req, plane_id, plane_src_h_prop, 1080 << 16);
drmModeAtomicAddProperty(req, plane_id, plane_crtc_x_prop, 0);
drmModeAtomicAddProperty(req, plane_id, plane_crtc_y_prop, 0);
drmModeAtomicAddProperty(req, plane_id, plane_crtc_w_prop, 1920);
drmModeAtomicAddProperty(req, plane_id, plane_crtc_h_prop, 1080);
```

### 7.4 drmModeAtomicCommit——提交请求

```c
int drmModeAtomicCommit(int fd, drmModeAtomicReqPtr req,
                         uint32_t flags, void *user_data);

// flags 参数：
//   DRM_MODE_ATOMIC_TEST_ONLY      (1<<2) — 仅验证，不应用
//   DRM_MODE_ATOMIC_NONBLOCK       (1<<3) — 非阻塞模式
//   DRM_MODE_ATOMIC_ALLOW_MODESET  (1<<4) — 允许模式设置（需要时重建显示管线）
//   DRM_MODE_PAGE_FLIP_EVENT       (1<<5) — 请求 page flip 事件通知
//   DRM_MODE_ATOMIC_FLAGS          (1<<6) — 保留

// 返回值：
//   0: 成功
//   -EINVAL: 参数无效
//   -ENOSPC: 资源不足（如 MST 带宽不足）
//   -EBUSY: 资源忙（如正在 pending 的 flip）
//   -EACCES: 权限不足
//   -ERANGE: 值超出范围

// user_data：用户自定义数据，在 page flip 事件中返回
// 可用于识别哪个提交完成了
```

**Atomic Commit 模式组合**：

| 模式 | 场景 | 说明 |
|------|------|------|
| TEST_ONLY | 验证配置 | 模拟提交检查，不修改硬件状态 |
| ALLOW_MODESET | 模式切换 | 允许完整模式切换（黑屏、重训练链路） |
| ALLOW_MODESET \| PAGE_FLIP_EVENT | 模式切换+事件 | 切换模式并等待完成事件 |
| NONBLOCK | 快速提交 | 不等待垂直消隐，立即提交 |
| PAGE_FLIP_EVENT | 精准翻转 | 下一个 VBlank 时切换，并发送事件 |
| 0 | 小修改 | 仅修改参数（如亮度、色彩），不触发 modeset |

### 7.5 完整 Atomic 模式切换示例

```c
static int atomic_set_mode(int fd, uint32_t crtc_id, uint32_t conn_id,
                            uint32_t plane_id, uint32_t fb_id,
                            drmModeModeInfo *mode) {

    // 1. 获取各对象的属性 ID（此处为简化，假设已获取）
    // 实际使用中需要先通过 drmModeObjectGetProperties 获取

    // 2. 创建 Atomic 请求
    drmModeAtomicReq *req = drmModeAtomicAlloc();
    if (!req) return -1;

    // 3. 先 TEST_ONLY 验证
    drmModeAtomicAddProperty(req, crtc_id, crtc_mode_id, (uint64_t)mode);
    drmModeAtomicAddProperty(req, crtc_id, crtc_active_id, 1);
    drmModeAtomicAddProperty(req, conn_id, conn_crtc_id_id, crtc_id);
    drmModeAtomicAddProperty(req, plane_id, plane_fb_id_id, fb_id);
    drmModeAtomicAddProperty(req, plane_id, plane_crtc_id_id, crtc_id);
    drmModeAtomicAddProperty(req, plane_id, plane_src_w_id, mode->hdisplay << 16);
    drmModeAtomicAddProperty(req, plane_id, plane_src_h_id, mode->vdisplay << 16);
    drmModeAtomicAddProperty(req, plane_id, plane_crtc_w_id, mode->hdisplay);
    drmModeAtomicAddProperty(req, plane_id, plane_crtc_h_id, mode->vdisplay);

    int ret = drmModeAtomicCommit(fd, req, DRM_MODE_ATOMIC_TEST_ONLY, NULL);
    if (ret < 0) {
        fprintf(stderr, "Atomic TEST_ONLY failed: %d\n", ret);
        drmModeAtomicFree(req);
        return ret;
    }
    printf("Atomic test passed, committing...\n");

    // 4. 正式提交（带 page flip 事件）
    ret = drmModeAtomicCommit(fd, req,
               DRM_MODE_ATOMIC_ALLOW_MODESET | DRM_MODE_PAGE_FLIP_EVENT,
               (void *)(uintptr_t)fb_id);
    if (ret < 0) {
        fprintf(stderr, "Atomic commit failed: %d\n", ret);
        drmModeAtomicFree(req);
        return ret;
    }

    drmModeAtomicFree(req);
    return 0;
}
```

### 7.6 Atomic 与 Legacy 对比

| 特性 | Legacy (SetCrtc/SetProperty) | Atomic |
|------|------------------------------|--------|
| 提交方式 | 多个独立 ioctl | 单个 ioctl 提交 |
| 原子性 | 无（中间状态可见） | 全有或全无 |
| 回滚 | 手动处理 | 内核自动回滚 |
| 测试验证 | 无 | TEST_ONLY 标志 |
| 事件 | 单独的 page flip ioctl | 内置于 commit |
| 多对象依赖 | 用户空间处理 | 内核 atomic_check 处理 |
| 性能 | 每属性一次 ioctl | 一次 ioctl 完成所有修改 |
| 推荐 | 仅用于兼容 | 新开发使用 |

---

## 8. Event 处理机制

### 8.1 DRM 事件类型

| 事件类型 | ioctl 命令 | 描述 |
|----------|-----------|------|
| VBlank | DRM_IOCTL_WAIT_VBLANK | 垂直消隐事件 |
| Page Flip | DRM_IOCTL_MODE_PAGE_FLIP | 页面翻转完成通知 |
| Atomic Flip | DRM_MODE_PAGE_FLIP_EVENT 标志 | Atomic 提交翻转事件 |
| DPMS | - | 显示电源状态变化 |
| Hotplug | - | 通过 udev/sysfs 通知 |
| Lease | - | DRM lease 事件 |

### 8.2 drmHandleEvent——事件分发

```c
// 事件数据结构
struct drm_event {
    unsigned long type;      // 事件类型
    unsigned long length;    // 事件数据长度
};

struct drm_event_vblank {
    struct drm_event base;
    uint64_t user_data;      // 用户自定义数据
    uint32_t tv_sec;         // 时间戳：秒
    uint32_t tv_usec;        // 时间戳：微秒
    uint32_t sequence;       // VBlank 序列号
    uint32_t crtc_id;        // 发生事件的 CRTC ID（较新内核）
};

// 事件处理函数
typedef void (*drmEventContextFunc)(int fd, unsigned int sequence,
                                     unsigned int tv_sec,
                                     unsigned int tv_usec,
                                     void *user_data);

typedef struct _drmEventContext {
    int version;                                 // 版本号（填 DRM_EVENT_CONTEXT_VERSION）
    void (*vblank_handler)(int fd, unsigned int sequence,
                           unsigned int tv_sec,
                           unsigned int tv_usec,
                           void *user_data);
    void (*page_flip_handler)(int fd, unsigned int sequence,
                              unsigned int tv_sec,
                              unsigned int tv_usec,
                              unsigned int crtc_id,
                              void *user_data);
    // 新版增加的 atomic 专用 handler
    void (*page_flip_handler2)(int fd, unsigned int sequence,
                               unsigned int tv_sec,
                               unsigned int tv_usec,
                               unsigned int crtc_id,
                               void *user_data);
} drmEventContext, *drmEventContextPtr;
```

### 8.3 drmHandleEvent 使用

```c
#include <poll.h>

static void page_flip_handler(int fd, unsigned int sequence,
                               unsigned int tv_sec,
                               unsigned int tv_usec,
                               unsigned int crtc_id,
                               void *user_data) {
    uint32_t fb_id = (uint32_t)(uintptr_t)user_data;
    printf("Page flip completed: CRTC=%u FB=%u seq=%u\n",
           crtc_id, fb_id, sequence);
}

int main_loop(int fd) {
    drmEventContext evctx = {
        .version = DRM_EVENT_CONTEXT_VERSION,
        .page_flip_handler = page_flip_handler,
    };

    struct pollfd fds = {
        .fd = fd,
        .events = POLLIN,
    };

    while (1) {
        int ret = poll(&fds, 1, 5000); // 5 秒超时
        if (ret < 0) {
            if (errno == EINTR) continue;
            perror("poll");
            break;
        }
        if (ret == 0) {
            printf("Poll timeout\n");
            continue;
        }
        if (fds.revents & POLLIN) {
            // 处理所有待处理事件
            drmHandleEvent(fd, &evctx);
        }
    }
    return 0;
}
```

### 8.4 VBlank 等待

```c
// Legacy VBlank 等待方式
union drm_wait_vblank vbl = {0};
vbl.request.type = DRM_VBLANK_RELATIVE;  // 相对等待
vbl.request.sequence = 1;                 // 等待下一个 VBlank
vbl.request.signal = 0;

int ret = drmIoctl(fd, DRM_IOCTL_WAIT_VBLANK, &vbl);
if (ret == 0) {
    printf("VBlank occurred: seq=%u time=%u.%06u\n",
           vbl.reply.sequence,
           vbl.reply.tval_sec,
           vbl.reply.tval_usec);
}

// VBlank 类型
#define DRM_VBLANK_ABSOLUTE     0x0    // 绝对序列号
#define DRM_VBLANK_RELATIVE     0x1    // 相对当前加 N
#define DRM_VBLANK_EVENT        0x40000000 // 异步事件模式
#define DRM_VBLANK_FLIP         0x80000000 // 同时触发 page flip

// 特定 CRTC 的 VBlank（DRM_VBLANK_SECONDARY = 1 << 5）
// CRTC 索引通过 DRM_VBLANK_HIGH_CRTC_MASK 编码到 type 的高位
vbl.request.type = DRM_VBLANK_RELATIVE | (crtc_index << DRM_VBLANK_HIGH_CRTC_SHIFT);
```

### 8.5 Atomic 结合 Page Flip 事件

```c
// 在 Atomic commit 中请求 page flip 事件
drmModeAtomicReq *req = drmModeAtomicAlloc();
// ... 添加属性 ...

// 使用 PAGE_FLIP_EVENT flag
int ret = drmModeAtomicCommit(fd, req,
    DRM_MODE_ATOMIC_ALLOW_MODESET | DRM_MODE_PAGE_FLIP_EVENT,
    (void *)user_data);

if (ret == 0) {
    // 提交成功，等待事件
    struct pollfd fds = { .fd = fd, .events = POLLIN };
    while (1) {
        poll(&fds, 1, -1);
        if (fds.revents & POLLIN) {
            drmHandleEvent(fd, &evctx);
            break; // 收到事件后退出
        }
    }
}
```

### 8.6 事件处理中的错误场景

```c
// 场景 1：Poll 返回但 drmHandleEvent 没有数据
// 原因：可能是信号或其他 fd 事件干扰
// 处理：检查 poll 返回值继续循环

// 场景 2：Page flip 事件丢失
// 原因：drmHandleEvent 读取的缓冲区不足
// 处理：确保循环读取直到 poll 无数据

// 场景 3：Multiple pending flips
// 只能有一个 pending page flip per CRTC
// 如果之前提交的 flip 未完成，再次提交会返回 -EBUSY
// 处理：等待事件后再提交新的 flip

// 场景 4：Atomic commit 的 USER_DATA 丢失
// 注意：DRM_MODE_PAGE_FLIP_EVENT 的 user_data 在 page_flip_handler 中接收
// 内核在 drm_crtc_commit 中保存并返回该指针
// 确保 user_data 指向的内存在事件处理前有效
```

---

## 9. AMDGPU 驱动专用 libdrm 接口

### 9.1 初始化 AMDGPU 设备

```c
#include <amdgpu.h>
#include <amdgpu_drm.h>

int amdgpu_init(int fd) {
    amdgpu_device_handle dev;
    uint32_t major_version, minor_version;

    // 初始化 AMDGPU 设备
    int r = amdgpu_device_initialize(fd, &major_version, &minor_version, &dev);
    if (r) {
        fprintf(stderr, "amdgpu_device_initialize failed: %d\n", r);
        return r;
    }

    printf("AMDGPU initialized: driver %u.%u, device handle %p\n",
           major_version, minor_version, (void *)dev);

    // 获取设备信息
    amdgpu_device_info dev_info = {0};
    amdgpu_device_info_get(dev, &dev_info);

    // 设备 ID 和修订
    printf("Device: PCI %04x:%04x, Rev %04x\n",
           dev_info.device_id, dev_info.subsys_id, dev_info.revision_id);

    // 显存信息
    uint64_t vram_size, gtt_size;
    amdgpu_device_get_info(dev, &vram_size, &gtt_size);
    printf("VRAM: %llu MB, GTT: %llu MB\n",
           vram_size / (1024*1024), gtt_size / (1024*1024));

    return 0;
}
```

### 9.2 AMDGPU BO 分配与使用

```c
// 分配 VRAM BO（用于扫描输出）
amdgpu_bo_handle scanout_bo;
struct amdgpu_bo_alloc_info scanout_alloc = {
    .alloc_size = 1920 * 1080 * 4,
    .phys_alignment = 256,            // 256 字节对齐
    .preferred_heap = AMDGPU_GEM_DOMAIN_VRAM,
    .flags = 0,                        // 无特殊标志
};

int r = amdgpu_bo_alloc(dev, &scanout_alloc, &scanout_bo);
if (r) {
    fprintf(stderr, "VRAM BO allocation failed: %d\n", r);
    return r;
}

// 申请 CPU 映射（VRAM BO 可能需要通过 GTT 间接访问）
r = amdgpu_bo_cpu_map(scanout_bo, (void **)&cpu_ptr);
if (r) {
    // VRAM 可能不支持直接 CPU 映射
    // 回退：分配 GTT BO 做中间缓冲区
    fprintf(stderr, "VRAM CPU map failed, using GTT fallback\n");

    amdgpu_bo_handle staging_bo;
    struct amdgpu_bo_alloc_info staging_alloc = {
        .alloc_size = 1920 * 1080 * 4,
        .phys_alignment = 256,
        .preferred_heap = AMDGPU_GEM_DOMAIN_GTT,
    };
    amdgpu_bo_alloc(dev, &staging_alloc, &staging_bo);
    amdgpu_bo_cpu_map(staging_bo, (void **)&cpu_ptr);

    // 渲染完成后复制到 VRAM
    // 需要通过 amdgpu_bo_copy() 或 SDMA 引擎
}

// 导出 GEM handle 给 DRM FB 使用
uint32_t gem_handle;
amdgpu_bo_export(scanout_bo, amdgpu_bo_handle_type_gem_flink, &gem_handle);

// 创建 FB
uint32_t fb_id;
drmModeAddFB(fd, 1920, 1080, 24, 32, stride, gem_handle, &fb_id);

// 释放
amdgpu_bo_cpu_unmap(scanout_bo);
amdgpu_bo_free(scanout_bo);
```

### 9.3 AMDGPU Command Submission——命令提交

```c
// AMDGPU 的命令提交接口，用于向 GPU 发送渲染/计算命令

// 创建 IB (Indirect Buffer)
amdgpu_bo_handle ib_bo;
struct amdgpu_bo_alloc_info ib_alloc = {
    .alloc_size = 4096,                     // 4KB IB 空间
    .phys_alignment = 4096,
    .preferred_heap = AMDGPU_GEM_DOMAIN_GTT,
};
amdgpu_bo_alloc(dev, &ib_alloc, &ib_bo);

// 获取 IB 的 GPU 虚拟地址
uint64_t ib_va;
amdgpu_bo_va_op(ib_bo, 0, 4096, 0, AMDGPU_VA_OP_MAP, &ib_va);

// 准备提交（用于 GFX 管线测试）
amdgpu_context_handle context;
amdgpu_cs_ib_info ib_info = {
    .ib_mc_address = ib_va,     // IB 的 GPU 地址
    .size = 256,                 // IB 中的 dword 数量
};
amdgpu_cs_request cs_request = {
    .ip_type = AMDGPU_HW_IP_GFX,
    .ring = 0,
    .number_of_ibs = 1,
    .ibs = &ib_info,
    .fence_info = NULL,
};

amdgpu_cs_submit(context, 0, &cs_request, 1);

// 等待完成
amdgpu_cs_query_fence_status(&fence, AMDGPU_TIMEOUT_INFINITE,
                              &expired, &status);
```

### 9.4 AMDGPU 调试接口

```c
// 查询 GPU 温度
uint32_t temperature;
amdgpu_query_sensor_info(dev, AMDGPU_SENSOR_GFX_TEMPERATURE, &temperature);

// 查询 GPU 使用率
uint32_t gpu_busy_percent;
amdgpu_query_sensor_info(dev, AMDGPU_SENSOR_GPU_LOAD, &gpu_busy_percent);

// 查询显存使用率
uint32_t vram_busy_percent;
amdgpu_query_sensor_info(dev, AMDGPU_SENSOR_MEM_LOAD, &vram_busy_percent);

// 查询显存总量和已用
uint64_t vram_total, vram_used, gtt_total, gtt_used;
amdgpu_query_info(dev, AMDGPU_INFO_VRAM_USAGE, &vram_total, &vram_used);
amdgpu_query_info(dev, AMDGPU_INFO_GTT_USAGE, &gtt_total, &gtt_used);
```

---

## 10. 从 libdrm 到内核 ioctl 的映射

### 10.1 核心映射表

| libdrm 函数 | DRM ioctl 命令 | 说明 |
|------------|----------------|------|
| drmModeGetResources | DRM_IOCTL_MODE_GETRESOURCES | 获取全局资源列表 |
| drmModeGetConnector | DRM_IOCTL_MODE_GETCONNECTOR | 获取连接器信息 |
| drmModeGetEncoder | DRM_IOCTL_MODE_GETENCODER | 获取编码器信息 |
| drmModeGetCrtc | DRM_IOCTL_MODE_GETCRTC | 获取 CRTC 状态 |
| drmModeGetPlane | DRM_IOCTL_MODE_GETPLANE | 获取 Plane 信息 |
| drmModeAddFB | DRM_IOCTL_MODE_ADDFB | 创建 Framebuffer |
| drmModeAddFB2 | DRM_IOCTL_MODE_ADDFB2 | 创建 FB（新版） |
| drmModeRmFB | DRM_IOCTL_MODE_RMFB | 删除 Framebuffer |
| drmModeSetCrtc | DRM_IOCTL_MODE_SETCRTC | 设置 CRTC 模式 |
| drmModeSetPlane | DRM_IOCTL_MODE_SETPLANE | 设置 Plane |
| drmModePageFlip | DRM_IOCTL_MODE_PAGE_FLIP | 页面翻转 |
| drmModeDirtyFB | DRM_IOCTL_MODE_DIRTYFB | 标记 FB 脏区域 |
| drmModeGetProperty | DRM_IOCTL_MODE_GETPROPERTY | 获取属性定义 |
| drmModeObjectGetProperties | DRM_IOCTL_MODE_OBJ_GETPROPERTIES | 获取对象的所有属性值 |
| drmModeSetProperty | DRM_IOCTL_MODE_SETPROPERTY | 设置属性值（Legacy） |
| drmModeAtomicCommit | DRM_IOCTL_MODE_ATOMIC | Atomic 提交 |
| drmModeCreateLease | DRM_IOCTL_MODE_CREATE_LEASE | 创建 DRM Lease |
| drmModeListLeases | DRM_IOCTL_MODE_LIST_LESSEES | 列出 Lease |
| drmModeGetLease | DRM_IOCTL_MODE_GET_LEASE | 获取 Lease 信息 |
| drmModeCursorSet | DRM_IOCTL_MODE_CURSOR | 设置硬件光标 |
| drmModeCursorMove | DRM_IOCTL_MODE_CURSOR | 移动硬件光标（同 ioctl） |
| drmCreateDumbBuffer | DRM_IOCTL_MODE_CREATE_DUMB | 创建 Dumb Buffer |
| drmDestroyDumbBuffer | DRM_IOCTL_MODE_DESTROY_DUMB | 销毁 Dumb Buffer |
| drmMapDumbBuffer | DRM_IOCTL_MODE_MAP_DUMB | 映射 Dumb Buffer |
| drmPrimeHandleToFD | DRM_IOCTL_PRIME_HANDLE_TO_FD | GEM handle → DMABUF fd |
| drmPrimeFDToHandle | DRM_IOCTL_PRIME_FD_TO_HANDLE | DMABUF fd → GEM handle |
| drmWaitVBlank | DRM_IOCTL_WAIT_VBLANK | 等待垂直消隐 |
| drmSetMaster | DRM_IOCTL_SET_MASTER | 获取 DRM Master 权限 |
| drmDropMaster | DRM_IOCTL_DROP_MASTER | 释放 DRM Master 权限 |
| drmModeGetFB | DRM_IOCTL_MODE_GETFB | 获取 FB 信息 |
| drmModeGetFB2 | DRM_IOCTL_MODE_GETFB2 | 获取 FB2 信息 |
| drmSyncobjCreate | DRM_IOCTL_SYNCOBJ_CREATE | 创建同步对象 |
| drmSyncobjWait | DRM_IOCTL_SYNCOBJ_WAIT | 等待同步对象 |
| drmModeCreatePropertyBlob | DRM_IOCTL_MODE_CREATEPROPBLOB | 创建属性 Blob |
| drmModeDestroyPropertyBlob | DRM_IOCTL_MODE_DESTROYPROPBLOB | 销毁属性 Blob |

### 10.2 ioctl 调用流程详解

libdrm 函数如何最终调用内核 ioctl：

```c
// libdrm 内部实现（简化）
int drmModeAddFB(int fd, uint32_t width, uint32_t height,
                 uint8_t depth, uint8_t bpp, uint32_t pitch,
                 uint32_t bo_handle, uint32_t *buf_id)
{
    struct drm_mode_fb_cmd fb = {
        .width = width,
        .height = height,
        .pitch = pitch,
        .bpp = bpp,
        .depth = depth,
        .handle = bo_handle,
    };

    // 实际调用 drmIoctl -> ioctl(fd, DRM_IOCTL_MODE_ADDFB, &fb)
    int ret = drmIoctl(fd, DRM_IOCTL_MODE_ADDFB, &fb);
    if (ret == 0 && buf_id) {
        *buf_id = fb.fb_id;
    }
    return ret;
}

// drmIoctl 封装
int drmIoctl(int fd, unsigned long request, void *arg)
{
    int ret;

    // 循环处理 EINTR/EAGAIN
    do {
        ret = ioctl(fd, request, arg);
    } while (ret < 0 && (errno == EINTR || errno == EAGAIN));

    return ret;
}
```

**内核侧的处理流程**：

```
ioctl(fd, DRM_IOCTL_MODE_GETCONNECTOR, &arg)
    │
    ▼
drm_ioctl()  [drivers/gpu/drm/drm_drv.c]
    │  根据 cmd 号查找 drm_ioctl_desc 表
    ▼
drm_mode_getconnector()  [drivers/gpu/drm/drm_mode_config.c]
    │  验证参数，调用驱动回调
    ▼
connector->funcs->fill_modes()  [amdgpu_dm_connector_funcs]
    │  AMDGPU DC 层：读取 EDID，枚举模式
    ▼
drm_helper_probe_single_connector_modes()
    │  去重、验证、排序显示模式
    ▼
返回到用户空间
```

### 10.3 直接使用 ioctl 的方式（绕过 libdrm）

在某些特殊场景下（如嵌入式系统或最小依赖环境），可以直接通过 ioctl 调用 DRM：

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <drm/drm.h>
#include <drm/drm_mode.h>

int main() {
    int fd = open("/dev/dri/card0", O_RDWR);
    if (fd < 0) { perror("open"); return 1; }

    // 直接调用 ioctl——DRM_IOCTL_MODE_GETRESOURCES
    struct drm_mode_card_res res = {0};
    // 先查询数量
    int ret = ioctl(fd, DRM_IOCTL_MODE_GETRESOURCES, &res);
    if (ret < 0) { perror("GETRESOURCES"); close(fd); return 1; }

    // 分配缓冲区后再次调用
    res.count_connectors = res.count_connectors; // 保持
    res.connector_id_ptr = (uint64_t)(uintptr_t)
        calloc(res.count_connectors, sizeof(uint32_t));
    res.count_encoders = res.count_encoders;
    res.encoder_id_ptr = (uint64_t)(uintptr_t)
        calloc(res.count_encoders, sizeof(uint32_t));
    // ... 填充其他字段 ...

    ret = ioctl(fd, DRM_IOCTL_MODE_GETRESOURCES, &res);
    if (ret < 0) { perror("GETRESOURCES"); goto cleanup; }

    uint32_t *conn_ids = (uint32_t *)(uintptr_t)res.connector_id_ptr;
    for (int i = 0; i < res.count_connectors; i++) {
        printf("Connector ID: %u\n", conn_ids[i]);
    }

cleanup:
    free((void *)(uintptr_t)res.connector_id_ptr);
    free((void *)(uintptr_t)res.encoder_id_ptr);
    free((void *)(uintptr_t)res.crtc_id_ptr);
    free((void *)(uintptr_t)res.fb_id_ptr);
    close(fd);
    return 0;
}
```

**为什么不推荐直接使用 ioctl**：
- 需要手动管理缓冲区分配和释放
- 数据结构版本不兼容风险
- libdrm 提供了更安全的封装和错误处理
- libdrm 跟踪了内核接口变化（如 FB2、Atomic 等）

---

## 11. 实战：编写 libdrm 测试程序

### 11.1 硬件枚举与信息打印

```c
/*
 * drm_info.c —— 打印完整的 DRM 设备树信息
 * 编译：gcc drm_info.c -ldrm -o drm_info
 */
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

static const char *conn_type_name(uint32_t type) {
    switch (type) {
        case DRM_MODE_CONNECTOR_Unknown:      return "Unknown";
        case DRM_MODE_CONNECTOR_VGA:          return "VGA";
        case DRM_MODE_CONNECTOR_DVII:         return "DVI-I";
        case DRM_MODE_CONNECTOR_DVID:         return "DVI-D";
        case DRM_MODE_CONNECTOR_DVIA:         return "DVI-A";
        case DRM_MODE_CONNECTOR_DisplayPort:  return "DP";
        case DRM_MODE_CONNECTOR_HDMIA:        return "HDMI-A";
        case DRM_MODE_CONNECTOR_HDMIB:        return "HDMI-B";
        case DRM_MODE_CONNECTOR_eDP:          return "eDP";
        case DRM_MODE_CONNECTOR_DSI:          return "DSI";
        default:                              return "Other";
    }
}

static const char *enc_type_name(uint32_t type) {
    switch (type) {
        case DRM_MODE_ENCODER_NONE:     return "None";
        case DRM_MODE_ENCODER_DAC:      return "DAC";
        case DRM_MODE_ENCODER_TMDS:     return "TMDS";
        case DRM_MODE_ENCODER_LVDS:     return "LVDS";
        case DRM_MODE_ENCODER_TVDAC:    return "TVDAC";
        case DRM_MODE_ENCODER_VIRTUAL:  return "Virtual";
        case DRM_MODE_ENCODER_DSI:      return "DSI";
        case DRM_MODE_ENCODER_DPMST:    return "DP-MST";
        case DRM_MODE_ENCODER_DPI:      return "DPI";
        default:                        return "Other";
    }
}

static void print_mode(drmModeModeInfo *mode, int index) {
    printf("      Mode[%d]: %s\n", index, mode->name);
    printf("        Clock: %u kHz\n", mode->clock);
    printf("        H: %u-%u-%u-%u\n",
           mode->hdisplay, mode->hsync_start,
           mode->hsync_end, mode->htotal);
    printf("        V: %u-%u-%u-%u\n",
           mode->vdisplay, mode->vsync_start,
           mode->vsync_end, mode->vtotal);
    printf("        Refresh: %u Hz\n", mode->vrefresh);
    printf("        Flags: 0x%x%s%s%s\n",
           mode->flags,
           (mode->flags & DRM_MODE_FLAG_INTERLACE) ? " interlaced" : "",
           (mode->flags & DRM_MODE_FLAG_PHSYNC) ? " +hsync" : "",
           (mode->flags & DRM_MODE_FLAG_NHSYNC) ? " -hsync" : "");
}

int main(int argc, char **argv) {
    const char *card = argv[1] ? argv[1] : "/dev/dri/card0";
    int fd = open(card, O_RDWR | O_CLOEXEC);
    if (fd < 0) { perror("open"); return 1; }

    // 获取 DRM 版本信息
    drmVersionPtr version = drmGetVersion(fd);
    if (version) {
        printf("Driver: %s (version %d.%d.%d)\n",
               version->name, version->version_major,
               version->version_minor, version->version_patchlevel);
        printf("Date:   %s\n", version->date);
        printf("Desc:   %s\n", version->desc);
        drmFreeVersion(version);
    }

    // 获取资源
    drmModeRes *res = drmModeGetResources(fd);
    if (!res) { fprintf(stderr, "Failed to get resources\n"); close(fd); return 1; }

    printf("\n=== DRM Resources ===\n");
    printf("CRTCs: %d, Encoders: %d, Connectors: %d, FBs: %d\n",
           res->count_crtcs, res->count_encoders,
           res->count_connectors, res->count_fbs);

    // 遍历 Connector
    for (int i = 0; i < res->count_connectors; i++) {
        drmModeConnector *conn = drmModeGetConnector(fd, res->connectors[i]);
        if (!conn) continue;

        printf("\n--- Connector %d ---\n", conn->connector_id);
        printf("  Type: %s-%d\n",
               conn_type_name(conn->connector_type),
               conn->connector_type_id);
        printf("  Status: %s\n",
               conn->connection == DRM_MODE_CONNECTED ? "Connected" :
               conn->connection == DRM_MODE_DISCONNECTED ? "Disconnected" :
               "Unknown");
        printf("  Physical: %dmm x %dmm\n",
               conn->mmWidth, conn->mmHeight);
        printf("  Subpixel: %d\n", conn->subpixel);
        printf("  Modes: %d\n", conn->count_modes);

        for (int m = 0; m < conn->count_modes && m < 5; m++)
            print_mode(&conn->modes[m], m);

        printf("  Encoders: ");
        for (int e = 0; e < conn->count_encoders; e++)
            printf("%u ", conn->encoders[e]);
        printf("\n");

        // 查询属性
        drmModeObjectProperties *props =
            drmModeObjectGetProperties(fd, conn->connector_id,
                                       DRM_MODE_OBJECT_CONNECTOR);
        if (props) {
            printf("  Properties: %d\n", props->count_props);
            for (int p = 0; p < props->count_props && p < 10; p++) {
                drmModeProperty *prop =
                    drmModeGetProperty(fd, props->props[p]);
                if (prop) {
                    printf("    [%u] %s = %lu\n",
                           props->props[p], prop->name,
                           props->prop_values[p]);
                    drmModeFreeProperty(prop);
                }
            }
            drmModeFreeObjectProperties(props);
        }

        drmModeFreeConnector(conn);
    }

    // 遍历 CRTC
    for (int i = 0; i < res->count_crtcs; i++) {
        drmModeCrtc *crtc = drmModeGetCrtc(fd, res->crtcs[i]);
        if (!crtc) continue;

        printf("\n--- CRTC %d ---\n", crtc->crtc_id);
        printf("  Buffer ID: %u\n", crtc->buffer_id);
        printf("  Position: (%d, %d)\n", crtc->crtc_x, crtc->crtc_y);
        printf("  Gamma size: %u\n", crtc->gamma_size);
        if (crtc->mode_valid) {
            printf("  Mode: %s @ %uHz\n",
                   crtc->mode.name, crtc->mode.vrefresh);
        }

        drmModeFreeCrtc(crtc);
    }

    // 遍历 Encoder
    for (int i = 0; i < res->count_encoders; i++) {
        drmModeEncoder *enc = drmModeGetEncoder(fd, res->encoders[i]);
        if (!enc) continue;

        printf("\n--- Encoder %d ---\n", enc->encoder_id);
        printf("  Type: %s\n", enc_type_name(enc->encoder_type));
        printf("  Possible CRTCs: 0x%x\n", enc->possible_crtcs);
        printf("  Possible Clones: 0x%x\n", enc->possible_clones);

        drmModeFreeEncoder(enc);
    }

    drmModeFreeResources(res);
    close(fd);
    return 0;
}
```

**运行示例**：
```bash
$ gcc drm_info.c -ldrm -o drm_info
$ ./drm_info /dev/dri/card0

Driver: amdgpu (version 3.57.0)
Date:   2025-01-15
Desc:   AMD GPU driver

=== DRM Resources ===
CRTCs: 6, Encoders: 6, Connectors: 4, FBs: 0

--- Connector 74 ---
  Type: DP-1
  Status: Disconnected
  Modes: 0
  ...

--- Connector 75 ---
  Type: DP-2
  Status: Connected
  Physical: 527mm x 296mm
  Modes: 17
    Mode[0]: 1920x1080
      Clock: 148500 kHz
      H: 1920-2008-2052-2200
      V: 1080-1084-1089-1125
      Refresh: 60 Hz
  ...
```

### 11.2 Atommic 模式设置示例

```c
/*
 * atomic_mode_set.c —— 使用 Atomic API 设置显示模式
 * 编译：gcc atomic_mode_set.c -ldrm -o atomic_mode_set
 * 运行：./atomic_mode_set /dev/dri/card0 [connector_id] [mode_index]
 *
 * 警告：此程序会修改显示输出！确保在测试环境中运行。
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

struct device {
    int fd;
    drmModeRes *res;
    drmModeConnector *conn;
    drmModeEncoder *enc;
    drmModeCrtc *crtc;
    drmModeModeInfo *mode;
    uint32_t crtc_id;
    uint32_t plane_id;
    uint32_t fb_id;
    uint32_t conn_id;
    uint32_t handle;
    uint32_t *fb_ptr;
    uint32_t stride;
    uint64_t fb_size;
};

static int find_plane(int fd, uint32_t crtc_id, uint32_t *plane_id) {
    drmModePlaneRes *plane_res = drmModeGetPlaneResources(fd);
    if (!plane_res) return -1;

    for (int i = 0; i < plane_res->count_planes; i++) {
        drmModePlane *plane = drmModeGetPlane(fd, plane_res->planes[i]);
        if (!plane) continue;

        // 检查此 plane 是否可绑定到目标 CRTC
        // 并且是 primary plane（通过 type 属性判断）
        if (plane->possible_crtcs & (1 << crtc_id)) {
            // 检查 type 属性是否为 PRIMARY
            drmModeObjectProperties *props =
                drmModeObjectGetProperties(fd, plane->plane_id,
                                          DRM_MODE_OBJECT_PLANE);
            int is_primary = 0;
            if (props) {
                for (int p = 0; p < props->count_props; p++) {
                    drmModeProperty *prop =
                        drmModeGetProperty(fd, props->props[p]);
                    if (prop && strcmp(prop->name, "type") == 0) {
                        // DRM_PLANE_TYPE_PRIMARY = 1
                        is_primary = (props->prop_values[p] == 1);
                        drmModeFreeProperty(prop);
                        break;
                    }
                    if (prop) drmModeFreeProperty(prop);
                }
                drmModeFreeObjectProperties(props);
            }

            if (is_primary || plane_res->count_planes == 1) {
                *plane_id = plane->plane_id;
                drmModeFreePlane(plane);
                drmModeFreePlaneResources(plane_res);
                return 0;
            }
        }
        drmModeFreePlane(plane);
    }

    drmModeFreePlaneResources(plane_res);
    return -1;
}

static int create_fb(struct device *dev) {
    // 分配 dumb buffer
    struct drm_mode_create_dumb create = {
        .width = dev->mode->hdisplay,
        .height = dev->mode->vdisplay,
        .bpp = 32,
    };
    int ret = drmIoctl(dev->fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);
    if (ret < 0) { perror("CREATE_DUMB"); return ret; }

    dev->handle = create.handle;
    dev->stride = create.pitch;
    dev->fb_size = create.size;

    // 创建 FB
    ret = drmModeAddFB(dev->fd, dev->mode->hdisplay, dev->mode->vdisplay,
                       24, 32, dev->stride, dev->handle, &dev->fb_id);
    if (ret) { perror("AddFB"); return ret; }

    // 映射到用户空间
    struct drm_mode_map_dumb map = { .handle = dev->handle };
    ret = drmIoctl(dev->fd, DRM_IOCTL_MODE_MAP_DUMB, &map);
    if (ret) { perror("MAP_DUMB"); return ret; }

    dev->fb_ptr = mmap(NULL, dev->fb_size, PROT_READ | PROT_WRITE,
                       MAP_SHARED, dev->fd, map.offset);
    if (dev->fb_ptr == MAP_FAILED) { perror("mmap"); return -1; }

    // 填充渐变颜色
    uint32_t w = dev->mode->hdisplay;
    uint32_t h = dev->mode->vdisplay;
    for (uint32_t y = 0; y < h; y++) {
        for (uint32_t x = 0; x < w; x++) {
            uint8_t r = (x * 255) / w;
            uint8_t g = (y * 255) / h;
            uint8_t b = ((x + y) * 255) / (w + h);
            dev->fb_ptr[y * (dev->stride / 4) + x] =
                (0xFF << 24) | (r << 16) | (g << 8) | b;
        }
    }

    return 0;
}

static int atomic_commit(struct device *dev) {
    drmModeAtomicReq *req = drmModeAtomicAlloc();
    if (!req) return -1;

    // 需要查找属性 ID
    // 注意：实际代码中应通过 drmModeObjectGetProperties 预先查询
    // 此示例使用固定属性名称——实际使用时需要动态查找

    // 简化起见，我们使用 Legacy API 作为后备方案
    int ret = drmModeSetCrtc(dev->fd, dev->crtc_id, dev->fb_id,
                             0, 0, &dev->conn_id, 1, dev->mode);
    if (ret) {
        fprintf(stderr, "drmModeSetCrtc failed: %d\n", ret);
    }

    drmModeAtomicFree(req);
    return ret;
}

int main(int argc, char **argv) {
    const char *card = argv[1] ? argv[1] : "/dev/dri/card0";
    struct device dev = {0};

    dev.fd = open(card, O_RDWR | O_CLOEXEC);
    if (dev.fd < 0) { perror("open"); return 1; }

    // 获取资源
    dev.res = drmModeGetResources(dev.fd);
    if (!dev.res) { fprintf(stderr, "Failed to get resources\n"); return 1; }

    // 查找第一个已连接的连接器
    int conn_idx = 0;
    for (int i = 0; i < dev.res->count_connectors; i++) {
        dev.conn = drmModeGetConnector(dev.fd, dev.res->connectors[i]);
        if (dev.conn && dev.conn->connection == DRM_MODE_CONNECTED) {
            conn_idx = i;
            break;
        }
        if (dev.conn) drmModeFreeConnector(dev.conn);
    }

    if (!dev.conn) {
        fprintf(stderr, "No connected connector found\n");
        return 1;
    }

    dev.conn_id = dev.conn->connector_id;
    printf("Using connector %d (%s-%d)\n",
           dev.conn_id,
           dev.conn->connector_type == DRM_MODE_CONNECTOR_DisplayPort ? "DP" :
           dev.conn->connector_type == DRM_MODE_CONNECTOR_HDMIA ? "HDMI" : "Other",
           dev.conn->connector_type_id);

    // 使用首选模式
    dev.mode = NULL;
    for (int i = 0; i < dev.conn->count_modes; i++) {
        if (dev.conn->modes[i].type & DRM_MODE_TYPE_PREFERRED) {
            dev.mode = &dev.conn->modes[i];
            break;
        }
    }
    if (!dev.mode && dev.conn->count_modes > 0)
        dev.mode = &dev.conn->modes[0];
    if (!dev.mode) {
        fprintf(stderr, "No valid mode\n");
        return 1;
    }
    printf("Using mode: %s @ %uHz\n", dev.mode->name, dev.mode->vrefresh);

    // 查找 Encoder
    dev.enc = drmModeGetEncoder(dev.fd, dev.conn->encoder_id);
    if (!dev.enc) {
        fprintf(stderr, "No encoder\n");
        return 1;
    }

    // 查找 CRTC
    for (int i = 0; i < dev.res->count_crtcs; i++) {
        if (dev.enc->possible_crtcs & (1 << i)) {
            dev.crtc_id = dev.res->crtcs[i];
            break;
        }
    }
    if (!dev.crtc_id) {
        fprintf(stderr, "No compatible CRTC\n");
        return 1;
    }
    printf("Using CRTC: %u\n", dev.crtc_id);

    // 查找 Plane
    if (find_plane(dev.fd, dev.crtc_id, &dev.plane_id) < 0) {
        fprintf(stderr, "No compatible plane\n");
        return 1;
    }
    printf("Using Plane: %u\n", dev.plane_id);

    // 创建 FB
    if (create_fb(&dev) < 0) {
        fprintf(stderr, "Failed to create FB\n");
        return 1;
    }
    printf("Created FB: %u\n", dev.fb_id);

    // Atomic 提交模式设置
    if (atomic_commit(&dev) < 0) {
        fprintf(stderr, "Mode set failed\n");
        return 1;
    }

    printf("Mode set successful!\nPress Enter to exit...\n");
    getchar();

    // 清理
    munmap(dev.fb_ptr, dev.fb_size);
    drmModeRmFB(dev.fd, dev.fb_id);
    struct drm_mode_destroy_dumb destroy = { .handle = dev.handle };
    drmIoctl(dev.fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
    drmModeFreeCrtc(dev.crtc);
    drmModeFreeEncoder(dev.enc);
    drmModeFreeConnector(dev.conn);
    drmModeFreeResources(dev.res);
    close(dev.fd);
    return 0;
}
```

### 11.3 Page Flip 测试程序

```c
/*
 * page_flip.c —— 演示 VBlank 同步的页面翻转
 * 编译：gcc page_flip.c -ldrm -o page_flip
 */
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <poll.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

static int fb_id_1, fb_id_2;
static int current_fb = 0;

static void page_flip_handler(int fd, unsigned int sequence,
                               unsigned int tv_sec,
                               unsigned int tv_usec,
                               unsigned int crtc_id,
                               void *user_data) {
    current_fb = !current_fb;
    printf("Flip completed: seq=%u, fb=%d\n",
           sequence, current_fb ? fb_id_2 : fb_id_1);
}

int main(int argc, char **argv) {
    const char *card = argv[1] ? argv[1] : "/dev/dri/card0";
    int fd = open(card, O_RDWR | O_CLOEXEC);
    if (fd < 0) { perror("open"); return 1; }

    // 获取资源和连接器（与上一示例相同，省略重复代码）
    // ...

    // 创建两个 FB（双缓冲）
    // ...（FB 创建代码略）

    // 设置 CRTC 显示第一个 FB
    drmModeSetCrtc(fd, crtc_id, fb_id_1, 0, 0,
                   &conn_id, 1, mode);

    // 事件处理上下文
    drmEventContext evctx = {
        .version = DRM_EVENT_CONTEXT_VERSION,
        .page_flip_handler = page_flip_handler,
    };

    // 翻转 60 次（约 1 秒 @ 60Hz）
    for (int i = 0; i < 60; i++) {
        uint32_t flip_fb = (i % 2 == 0) ? fb_id_2 : fb_id_1;

        int ret = drmModePageFlip(fd, crtc_id, flip_fb,
                                  DRM_MODE_PAGE_FLIP_EVENT, NULL);
        if (ret < 0) {
            if (ret == -EBUSY) {
                // 等待上次翻转完成
                struct pollfd fds = { .fd = fd, .events = POLLIN };
                poll(&fds, 1, -1);
                drmHandleEvent(fd, &evctx);
                i--; // 重试
                continue;
            }
            perror("PageFlip");
            break;
        }

        // 等待翻转完成事件
        struct pollfd fds = { .fd = fd, .events = POLLIN };
        poll(&fds, 1, -1);
        drmHandleEvent(fd, &evctx);
    }

    close(fd);
    return 0;
}
```

---

## 12. 调试与性能分析

### 12.1 调试工具

| 工具 | 用途 | 示例 |
|------|------|------|
| `modetest` | libdrm 自带的测试工具 | `modetest -M amdgpu -c` |
| `kmstest` | 简化的 KMS 测试 | `kmstest -c` |
| `drm_info` | 第三方 DRM 信息查看器 | `drm_info` |
| `xrandr` | X11 下的显示配置 | `xrandr --props` |
| `wlr-randr` | Wayland 下的显示配置 | `wlr-randr` |
| `igt_drm_fdinfo` | IGT GPU 工具 | 查看 DRM fd 信息 |
| `strace` | 系统调用追踪 | `strace -e ioctl modetest -c` |

**strace 查看 libdrm 的 ioctl 调用**：
```bash
# 追踪 ioctl 调用，过滤 DRM 相关
strace -e ioctl -f modetest -M amdgpu -c 2>&1 | grep DRM_IOCTL

# 更详细的 ioctl 参数查看
strace -e ioctl -v -f modetest -M amdgpu -c 2>&1 | head -100
```

**使用 drm_info 查看完整设备树**：
```bash
# 安装 drm_info（可能需要从源码编译）
git clone https://gitlab.freedesktop.org/emersion/drm_info.git
cd drm_info
meson build && ninja -C build
./build/drm_info
```

**drm_info 输出示例**（截取）：
```
DRM device 0
    Driver: amdgpu (AMD GPU)
    Card: /dev/dri/card0
    CRTCs: 6
        CRTC 0
            Modes: ...
            Gamma size: 256
    Encoders: 6
        Encoder 0
            Type: TMDS
            Possible CRTCs: 0x01
    Connectors: 4
        Connector 0 (DP-1)
            Connected: no
            Modes: 0
        Connector 1 (DP-2)
            Connected: yes
            Modes: 17
            Properties:
                EDID (blob): <hidden>
                link-status (enum): Good
                max bpc (range): 8-16
                ...
    Planes: 6
        Plane 0
            Type: Primary
            Formats: XR24, AR24, XR30, AR30
            Possible CRTCs: 0x01
```

### 12.2 DebugFS 接口

```bash
# DRM 状态转储
cat /sys/kernel/debug/dri/0/state

# AMDGPU DC 状态
cat /sys/kernel/debug/dri/0/amdgpu_dm_visual_confirm

# 查看所有 DRM clients
cat /proc/<pid>/fdinfo/<drm_fd>

# 查看进程的 DRM 内存使用
cat /sys/kernel/debug/dri/0/amdgpu_gem_info
```

### 12.3 性能分析

```c
// 测量 ioctl 调用耗时
#include <time.h>

static double measure_ioctl_time(int fd, int count) {
    struct timespec start, end;

    clock_gettime(CLOCK_MONOTONIC, &start);
    for (int i = 0; i < count; i++) {
        drmModeRes *res = drmModeGetResources(fd);
        if (res) drmModeFreeResources(res);
    }
    clock_gettime(CLOCK_MONOTONIC, &end);

    double elapsed = (end.tv_sec - start.tv_sec) +
                     (end.tv_nsec - start.tv_nsec) / 1e9;
    return elapsed / count * 1000; // 返回毫秒
}

// 测量 Page Flip 延迟
static double measure_flip_latency(int fd, uint32_t crtc_id,
                                    uint32_t fb_id, int iterations) {
    struct timespec start, end;
    double total_latency = 0;

    for (int i = 0; i < iterations; i++) {
        // 提交 flip
        clock_gettime(CLOCK_MONOTONIC, &start);

        int ret = drmModePageFlip(fd, crtc_id, fb_id,
                                  DRM_MODE_PAGE_FLIP_EVENT, NULL);
        if (ret < 0) continue;

        // 等待事件
        struct pollfd fds = { .fd = fd, .events = POLLIN };
        poll(&fds, 1, -1);

        drmEventContext evctx = {
            .version = DRM_EVENT_CONTEXT_VERSION,
            .page_flip_handler = NULL,
        };
        drmHandleEvent(fd, &evctx);

        clock_gettime(CLOCK_MONOTONIC, &end);

        double latency = (end.tv_sec - start.tv_sec) * 1000 +
                         (end.tv_nsec - start.tv_nsec) / 1e6;
        total_latency += latency;
    }

    return total_latency / iterations;
}
```

**性能基准参考**（AMDGPU RX 7900 XTX）：

| 操作 | 平均耗时 | 说明 |
|------|----------|------|
| drmModeGetResources | ~50μs | 仅返回已缓存数据 |
| drmModeGetConnector (首次) | ~5ms | 包含 EDID 读取 |
| drmModeGetConnector (缓存) | ~30μs | 使用已缓存的 EDID |
| drmModeAddFB | ~10μs | 仅创建元数据 |
| drmModeSetCrtc | ~16ms | 含模式切换（VBlank 同步） |
| drmModeAtomicCommit (TEST_ONLY) | ~100μs | 仅验证 |
| drmModeAtomicCommit (modeset) | ~16ms | 显示模式切换 |
| drmModeAtomicCommit (flip) | ~20μs | 仅页面翻转 |
| drmModePageFlip | ~15μs | 页面翻转提交 |
| drmHandleEvent | ~10μs | 事件分发 |

### 12.4 日志与错误诊断

```c
// 启用 libdrm 调试日志
// 方法 1：环境变量
export LIBDRM_DEBUG=1
export LIBDRM_DEBUG_FILE=/tmp/libdrm.log

// 方法 2：代码中启用
#include <xf86drm.h>

// libdrm 调试输出级别
// 0: 无输出（默认）
// 1: 错误信息
// 2: 警告信息
// 3: 调试信息

void drmEnableDebug(int level);  // 某些版本支持

// 常见的错误码解析
static const char *drm_errstr(int err) {
    switch (err) {
        case -EINVAL: return "EINVAL: Invalid argument";
        case -ENOSPC: return "ENOSPC: No space (MST bandwidth)";
        case -EBUSY:  return "EBUSY: Resource busy (pending flip)";
        case -EACCES: return "EACCES: Permission denied (need DRM master)";
        case -ERANGE: return "ERANGE: Value out of range (e.g., max_bpc)";
        case -ENODEV: return "ENODEV: No such device";
        case -ENOMEM: return "ENOMEM: Out of memory";
        case -EOPNOTSUPP: return "EOPNOTSUPP: Operation not supported";
        default:      return "Unknown error";
    }
}
```

---

## 13. 常见问题与陷阱

### 13.1 DRM Master 权限

```c
// 问题：调用某些 ioctl 时返回 -EACCES
// 原因：当前进程不是 DRM Master
// 场景：在 Wayland/Xorg 已经占用 card0 时，其他进程无法成为 Master

// 解决方法 1：使用 renderD 节点（不需要 Master 权限）
int fd = open("/dev/dri/renderD128", O_RDWR);

// 解决方法 2：如果必须使用 card0，先请求 Master
if (drmSetMaster(fd) < 0) {
    // 失败：card0 已被其他进程占用
    // 回退：使用 renderD 节点或停止显示服务器
}

// 解决方法 3：使用 --seat 或 vt 切换
// 在测试环境中，切换到纯控制台（Ctrl+Alt+F2）后运行

// Master 权限检查
static int check_master(int fd) {
    drm_magic_t magic;
    // 尝试获取 magic 以验证是否有权限
    return drmGetMagic(fd, &magic);
}
```

### 13.2 Atomic Commit 失败处理

```c
// 失败的典型原因和排查步骤

int ret = drmModeAtomicCommit(fd, req,
    DRM_MODE_ATOMIC_TEST_ONLY | DRM_MODE_ATOMIC_ALLOW_MODESET,
    NULL);

if (ret < 0) {
    switch (ret) {
        case -EINVAL:
            // 1. 检查属性 ID 是否正确
            // 2. 检查属性值是否在有效范围内
            // 3. 对于 BLOB 属性，检查 blob_id 是否有效
            // 4. 确认所有必要属性都已设置（如 CRTC active + mode）
            break;
        case -ENOSPC:
            // MST 带宽不足
            // 1. 检查 MST link 总带宽
            // 2. 尝试降低分辨率/刷新率/色深
            break;
        case -EBUSY:
            // 前一次 flip 未完成
            // 1. 等待事件处理
            // 2. 或使用 NONBLOCK 标志
            break;
        case -ERANGE:
            // 值超出范围
            // 1. 检查 RANGE 属性的 min/max
            // 2. 对于 max_bpc，检查显示器支持的色深
            break;
    }
}
```

### 13.3 内存泄漏陷阱

```c
// 常见泄漏模式

// 错误：忘记释放资源
drmModeRes *res = drmModeGetResources(fd);
// ... 使用 res ...
// drmModeFreeResources(res);  // 泄漏！

// 错误：在循环中重复获取但未释放
for (int i = 0; i < 1000; i++) {
    drmModeConnector *conn = drmModeGetConnector(fd, conn_id);
    // drmModeFreeConnector(conn);  // 泄漏！
}

// 错误：属性查询未释放
drmModeObjectProperties *props =
    drmModeObjectGetProperties(fd, obj_id, type);
drmModeProperty *prop = drmModeGetProperty(fd, props->props[0]);
// ... 使用 ...
drmModeFreeProperty(prop);
drmModeFreeObjectProperties(props);  // 两个都要释放

// 正确：使用 cleanup 宏（GCC/Clang）
__attribute__((cleanup(cleanup_res))) drmModeRes *res = NULL;

static inline void cleanup_res(drmModeRes **res) {
    if (*res) drmModeFreeResources(*res);
}
```

### 13.4 VBlank 同步问题

```c
// 问题 1：Page flip 事件丢失
// 原因：drmHandleEvent 未在 poll 循环中及时调用
// 解决：确保每次 poll 返回 POLLIN 后立即调用 drmHandleEvent

// 问题 2：多 CRTC 的 VBlank 混淆
// 原因：未指定 CRTC 索引，默认使用 CRTC 0
// 解决：在 atomic commit 中指定 crtc_id

// 问题 3：KMS Thread Safety
// 注意：libdrm 的大多数函数不是线程安全的
// 单个 fd 的 ioctl 操作应在同一线程中串行执行
// 如果必须多线程，为每个线程打开独立的 fd
```

### 13.5 兼容性问题

```c
// 问题 1：旧内核不支持 Atomic API
// 检测：检查 DRM_CAP_ATOMIC
uint64_t atomic_cap = 0;
int ret = drmGetCap(fd, DRM_CAP_ATOMIC, &atomic_cap);
if (ret == 0 && atomic_cap) {
    // 支持 Atomic
} else {
    // 回退到 Legacy API
}

// 问题 2：某些属性可能不存在
// 检测：尝试获取并检查返回值
drmModeProperty *prop = drmModeGetProperty(fd, prop_id);
if (!prop) {
    // 属性不存在，使用默认值或替代方案
}

// 问题 3：不同厂商驱动的行为差异
// AMDGPU：完全支持 Atomic，大量私有属性
// Intel：支持 Atomic，属性集不同
// Nouveau：部分支持 Atomic，某些功能仅 legacy
// vc4/v3d（Raspberry Pi）：基本 Atomic 支持
```

---

## 14. 总结与下一日预告

### 14.1 本日内容总结

Day 362 全面覆盖了 libdrm 用户空间库的核心知识：

1. **架构定位**：libdrm 是用户空间与内核 DRM 驱动之间的薄封装层，每个函数几乎直接映射到一个 ioctl 调用。

2. **核心数据结构**：
   - `drmModeRes`：全局资源列表（CRTC/Encoder/Connector/FB）
   - `drmModeConnector`：连接器，包含连接状态、可用模式、EDID、属性等
   - `drmModeCrtc`：CRTC，包含显示模式、FB 绑定
   - `drmModeEncoder`：编码器，通过 possible_crtcs 位掩码指示可绑定的 CRTC
   - `drmModePlane`：图层，包含 pixel format 支持列表
   - `drmModeModeInfo`：显示模式（timing 参数、刷新率等）
   - `drmModeProperty`：属性定义（RANGE/ENUM/BITMASK/BLOB）
   - `drmModeAtomicReq`：Atomic 请求对象

3. **API 体系**：
   - **资源枚举**：GetResources → GetConnector/GetEncoder/GetCrtc/GetPlane
   - **Framebuffer 管理**：AddFB/AddFB2/RmFB/GetFB
   - **内存管理**：Dumb Buffer（简单显示）、GEM（通用管理）、DMABUF（跨设备共享）、AMDGPU BO（VRAM/GTT）
   - **Atomic API**：Alloc → AddProperty (×N) → Commit (TEST_ONLY + 正式提交) → Free
   - **事件处理**：drmHandleEvent 结合 poll() 处理 VBlank/Page Flip 事件

4. **AMDGPU 扩展**：
   - `amdgpu_device_initialize`：设备初始化
   - `amdgpu_bo_alloc`：VRAM/GTT BO 分配
   - `amdgpu_cs_submit`：命令提交（GFX/Compute）
   - 传感器查询：GPU 温度、负载、显存使用率

5. **从 ioctl 到内核**：每个 libdrm 函数对应一个 `DRM_IOCTL_MODE_*` 命令，经过 `drm_ioctl()` → 对应处理函数 → 驱动回调的完整路径。

### 14.2 关键要点回顾

```
┌─────────────────────────────────────────────────────────┐
│               libdrm 编程黄金法则                          │
├─────────────────────────────────────────────────────────┤
│ 1. 始终使用 drmModeFree* 释放资源                         │
│ 2. 先 TEST_ONLY 再正式 Atomic 提交                        │
│ 3. 检查 DRM_CAP_ATOMIC 做兼容性判断                       │
│ 4. Page Flip 必须等待事件完成再提交下一次                  │
│ 5. 使用 pkg-config 管理编译标志                           │
│ 6. Pure console 下运行测试（避免 Master 冲突）             │
│ 7. Atomic commit 要同时设置 CRTC mode + active           │
│ 8. 所有属性通过 drmModeObjectGetProperties 动态查找        │
└─────────────────────────────────────────────────────────┘
```

### 14.3 下一日预告

**Day 363：Wayland / X11 与 DRM 的交互模型**

Day 363 将讲解：
- Wayland 合成器（Weston / wlroots / KWin）如何使用 libdrm 与 KMS 交互
- Xorg 的 DDX（Device Dependent X）驱动如何通过 libdrm 管理显示
- EGL/GBM 的集成——如何通过 GBM 创建 EGL 表面并用于 KMS 显示
- 典型场景：Wayland 合成器的模式设置、双缓冲翻转、光标管理
- Xorg + glamor 加速的工作原理
- 对比 Wayland 与 X11 在 DRM 使用上的关键差异
- 实战：编写一个简单的 Wayland 风格的合成器原型（使用 GBM + KMS Atomic）

```c
// 预告：Day 363 将分析 Wayland 合成器中的 libdrm 使用模式
// 示例：wlroots 中的 drm_connector 管理
struct wlr_drm_connector {
    struct wlr_output base;
    uint32_t connector_id;
    int32_t width, height;
    uint32_t crtc_id;
    drmModeConnector *conn;
    struct wlr_drm_backend *backend;
    // ...
};
```

---

*【Day 362 完】本日文档共编写 14 个章节，涵盖 libdrm 的完整知识体系：从架构定位、核心数据结构、资源枚举 API，到内存管理、Atomic API、事件处理、AMDGPU 专用接口、ioctl 映射、实战示例、调试方法和常见陷阱。为后续 Wayland/X11 交互模型（Day 363）和测试工具（Day 364）打下坚实基础。*