# 第351天：光标（Cursor）与硬件覆盖层

## 学习目标

- 理解 DRM 中 Cursor Plane 和 Overlay Plane 的概念及其与 Primary Plane 的区别
- 掌握 AMDGPU 中光标硬件的实现原理和编程接口
- 学习硬件覆盖层（Hardware Overlay）在视频播放和合成场景中的应用
- 掌握光标测试方法：IGT kms_cursor_crc、手动测试和性能基准
- 深入理解 DRM Plane 的 Z-order、alpha 合成和像素格式
- 了解 Overlay 在节能和性能优化中的作用

## 知识详解

### 一、DRM Plane 体系概述

在 DRM/KMS 架构中，显示内容通过 Plane（平面）进行管理和合成。每个 CRTC 可以关联多个 Plane，硬件将这些 Plane 合成为最终的显示画面。

#### 1.1 Plane 的三种类型

```
DRM Plane 类型
┌─────────────────────────────────────────────────────────────────┐
│  DRM_PLANE_TYPE_PRIMARY（主平面）                                │
│  ├─ 每个 CRTC 必备一个主平面                                     │
│  ├─ 通常用于显示主帧缓冲区的桌面内容                             │
│  ├─ 对应 AMDGPU DCN 中的 DPP（DCN Pipe Plane）                   │
│  └─ 不支持色彩键（Color Key）                                    │
├─────────────────────────────────────────────────────────────────┤
│  DRM_PLANE_TYPE_CURSOR（光标平面）                               │
│  ├─ 每个 CRTC 可选一个光标平面                                   │
│  ├─ 专用于显示鼠标指针图像                                       │
│  ├─ 硬件特性：小尺寸、独立位置/透明度、无需翻转 vsync           │
│  ├─ 对应 AMDGPU DCN 中的 MPC（Multiple Pipe Combined）光标路径  │
│  └─ 典型限制：最大 256×256 像素，硬编码 ARGB8888 格式           │
├─────────────────────────────────────────────────────────────────┤
│  DRM_PLANE_TYPE_OVERLAY（覆盖平面）                              │
│  ├─ 每个 CRTC 可选多个覆盖平面                                   │
│  ├─ 用于显示视频帧、OSD、字幕等覆盖内容                          │
│  ├─ 硬件特性：支持 YUV 格式、缩放、色彩键                       │
│  ├─ 对应 AMDGPU DCN 中的额外 DPP 管道                           │
│  └─ 优势：无需合成器进行 GPU 合成，减少功耗和延迟               │
└─────────────────────────────────────────────────────────────────┘
```

#### 1.2 Plane 合成架构

```
Plane 硬件合成流程
                              ┌─────────────────┐
                              │  用户空间缓冲区  │
                              │  (DMA-BUF)      │
                              └────────┬────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
                    ▼                  ▼                  ▼
            ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
            │ Primary Plane │  │ Overlay Plane│  │ Cursor Plane │
            │ (DPP0)        │  │ (DPP1)       │  │ (MPC 路径)   │
            │ Desktop/Render│  │ Video Frame  │  │ Pointer Icon │
            │ ARGB8888      │  │ NV12/YUYV    │  │ ARGB8888     │
            └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
                   │                  │                  │
                   └──────────────────┼──────────────────┘
                                      │
                                      ▼
                            ┌──────────────────┐
                            │  MPC (Multiple   │
                            │  Pipe Combine)   │
                            │  Alpha Blending  │
                            │  Z-order:        │
                            │  Cursor > Overlay│
                            │  > Primary       │
                            └────────┬─────────┘
                                     │
                                     ▼
                            ┌──────────────────┐
                            │  OPTC (Output    │
                            │  Timing Ctrl)    │
                            │  → Display       │
                            └──────────────────┘
```

#### 1.3 Plane 的核心数据结构

DRM 内核中 Plane 的核心数据结构是 `drm_plane`：

```c
// include/drm/drm_plane.h
struct drm_plane {
    struct drm_device *dev;
    struct list_head head;          // 全局 plane 列表

    uint32_t possible_crtcs;        // 位掩码，此 plane 可关联的 CRTC
    uint32_t format_count;          // 支持的像素格式数量
    const uint32_t *format_types;   // 支持的像素格式数组
    uint64_t *modifiers;            // 支持的 modifier（DCC 等）

    struct drm_crtc *crtc;          // 当前关联的 CRTC
    struct drm_framebuffer *fb;     // 当前关联的 framebuffer

    struct drm_plane_state *state;  // 当前状态（atomic）
    struct drm_plane_state *fb_damage_clips;  // 损坏区域

    const struct drm_plane_funcs *funcs;
    struct drm_object_properties properties;
    enum drm_plane_type type;       // PRIMARY / CURSOR / OVERLAY
};

// include/drm/drm_plane.h
struct drm_plane_state {
    struct drm_plane *plane;
    struct drm_crtc *crtc;         // 关联 CRTC
    struct drm_framebuffer *fb;    // 关联 Framebuffer

    int32_t crtc_x, crtc_y;        // 在 CRTC 中的位置
    uint32_t crtc_w, crtc_h;       // CRTC 中的尺寸
    uint32_t src_x, src_y;         // 源缓冲区位置（16.16 定点）
    uint32_t src_w, src_h;         // 源缓冲区尺寸（16.16 定点）

    struct drm_rect src;           // 上述的 rect 形式
    struct drm_rect dst;           // 目标位置 rect

    uint16_t alpha;                // 全局 alpha（0=透明, 0xffff=不透明）
    uint16_t pixel_blend_mode;     // 混合模式：None/Premultiplied/Coverage

    bool visible;                  // 此 plane 是否可见
    bool fb_changed;               // Framebuffer 是否变化
    bool crtc_changed;             // CRTC 是否变化
};
```

### 二、光标（Cursor）平面详解

#### 2.1 光标 Plane 的硬件特性

光标平面是 DRM 中最小但最重要的平面之一。其硬件特性决定了它的设计约束：

```
AMDGPU DCN 光标硬件特性
┌─────────────────────────────────────────────────────────────────┐
│  属性                  │  典型值                     │  说明     │
├─────────────────────────────────────────────────────────────────┤
│  最大光标尺寸           │  256×256 像素               │ DCN 限制  │
│  最小光标尺寸           │  32×32 像素                 │           │
│  对齐要求               │  64 字节对齐                │ 缓存行对齐 │
│  支持的像素格式         │  ARGB8888（32-bit）          │ 仅限此格式 │
│  光标颜色模式           │  32-bit RGBA（预乘 alpha）   │           │
│  光标位置精度           │  屏幕像素坐标（整数）        │           │
│  光标热键               │  硬件不支持热键              │ 用户空间计算│
│  透明度支持             │  每像素 alpha               │           │
│  gamma/LUT 校正         │  与主平面共享 gamma          │           │
│  翻转同步               │  无需 vsync 同步             │ 立即生效   │
│  每 CRTC 数量           │  1 个                       │           │
│  硬件光标层 Z-order     │  最高（覆盖所有其他平面）     │           │
└─────────────────────────────────────────────────────────────────┘
```

#### 2.2 DRM 光标操作接口

DRM 提供三种方式操作光标：

```c
// 1. Legacy SET_CURSOR IOCTL（传统接口）
// drivers/gpu/drm/drm_ioctl.c
DRM_IOCTL_DEF(DRM_IOCTL_MODE_CURSOR, drm_mode_cursor_ioctl, ...)
DRM_IOCTL_DEF(DRM_IOCTL_MODE_CURSOR2, drm_mode_cursor2_ioctl, ...)

// 入参结构
struct drm_mode_cursor2 {
    __u32 flags;           // CRTC_BG_NO_BO / BO / MOVE
    __u32 crtc_id;
    __s32 x;               // 屏幕 X 坐标
    __s32 y;               // 屏幕 Y 坐标
    __u32 width;           // 光标图像宽度
    __u32 height;          // 光标图像高度
    __u32 handle;          // GEM buffer handle（BO 标志时有效）
    __s32 hot_x;           // 热键 X 偏移
    __s32 hot_y;           // 热键 Y 偏移
    __u32 pad64;
};

// 2. Atomic 接口（推荐方式）
// 通过 drmModeAtomicAddProperty 设置 CURSOR 平面的属性：
// - FB_ID: 光标图像 buffer
// - CRTC_X, CRTC_Y: 光标位置
// - SRC_W, SRC_H: 光标尺寸

// 3. DRM_CLIENT_CAP_CURSOR_PLANE_COMBINED 接口
// 将 CURSOR plane 作为普通 plane 管理（简化用户空间处理）
```

#### 2.3 AMDGPU 光标实现源码分析

AMDGPU 的光标实现主要在 `dm_plane_helpers.c` 和 `dcnXX_plane.c` 中：

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/dm_plane_helpers.c
// AMDGPU 光标更新入口

int dm_plane_helper_commit(struct drm_plane *plane,
                           struct drm_plane_state *state)
{
    struct amdgpu_device *adev = drm_to_adev(plane->dev);
    struct dm_crtc_state *dm_state = to_dm_crtc_state(state->crtc->state);
    struct dc_plane_state *dc_plane;
    struct dc_cursor_attributes cursor_attr;
    struct dc_cursor_position cursor_pos;

    if (plane->type != DRM_PLANE_TYPE_CURSOR)
        return 0;  // 仅处理光标平面

    // 填充光标属性
    if (state->fb) {
        struct amdgpu_framebuffer *afb = to_amdgpu_fb(state->fb);

        cursor_attr.pitch = afb->base.pitches[0];
        cursor_attr.width = state->crtc_w;
        cursor_attr.height = state->crtc_h;
        cursor_attr.color_format = CURSOR_FORMAT_ARGB8888;

        // 获取 GPU 物理地址
        amdgpu_bo_va_for_early_clear(afb->obj, &cursor_attr.address);

        // 设置热点偏移（由用户空间传递）
        cursor_attr.hot_x = state->hot_x;
        cursor_attr.hot_y = state->hot_y;

        // 更新硬件光标
        dc_stream_set_cursor_attributes(
            dm_state->stream, &cursor_attr);
    }

    // 更新光标位置
    cursor_pos.x = state->crtc_x;
    cursor_pos.y = state->crtc_y;
    cursor_pos.enable = state->visible;
    cursor_pos.mirror = false;

    dc_stream_set_cursor_position(
        dm_state->stream, &cursor_pos);

    return 0;
}
```

DCN 硬件层的光标设置：

```c
// drivers/gpu/drm/amd/display/dc/dcn20/dcn20_plane.c
// DCN 硬件光标编程

void dcn20_set_cursor_attributes(
    struct dc_plane *plane,
    struct dc_cursor_attributes *attr)
{
    struct dcn20_plane *dcn20_plane = TO_DCN20_PLANE(plane);
    uint64_t address = attr->address;
    uint32_t pitch = attr->pitch;
    uint32_t width = attr->width;
    uint32_t height = attr->height;

    // 编程寄存器
    // CURSOR0_ADDR_HI: 光标 buffer 高 32 位地址
    // CURSOR0_ADDR_LO: 光标 buffer 低 32 位地址
    // CURSOR0_SIZE: 光标尺寸（宽/高）
    // CURSOR0_CONTROL: 光标格式和启用
    // CURSOR0_HOTSPOT: 热点偏移

    REG_SET(CURSOR0_ADDR_HI, 0,
            CURSOR_ADDR_HIGH, address >> 32);
    REG_SET(CURSOR0_ADDR_LO, 0,
            CURSOR_ADDR_LOW, address & 0xFFFFFFFF);

    REG_SET_2(CURSOR0_SIZE, 0,
              CURSOR_WIDTH, width,
              CURSOR_HEIGHT, height);

    REG_SET_2(CURSOR0_HOTSPOT, 0,
              HOTSPOT_X, attr->hot_x,
              HOTSPOT_Y, attr->hot_y);

    REG_UPDATE(CURSOR0_CONTROL,
               CURSOR_ENABLE, 1,
               CURSOR_FORMAT, CURSOR_FORMAT_ARGB8888,
               CURSOR_MODE, CURSOR_MODE_NORMAL);

    // 如果是预乘 alpha 模式
    if (attr->pre_multiplied_alpha) {
        REG_UPDATE(CURSOR0_CONTROL,
                   CURSOR_PRE_MULTIPLIED_ALPHA, 1);
    }
}

void dcn20_set_cursor_position(
    struct dc_plane *plane,
    struct dc_cursor_position *pos)
{
    struct dcn20_plane *dcn20_plane = TO_DCN20_PLANE(plane);

    if (pos->enable) {
        // 编程光标屏幕位置
        REG_SET_2(CURSOR0_POSITION, 0,
                  CURSOR_X, pos->x,
                  CURSOR_Y, pos->y);

        // 启用光标
        REG_UPDATE(CURSOR0_CONTROL,
                   CURSOR_ENABLE, 1);
    } else {
        // 禁用光标
        REG_UPDATE(CURSOR0_CONTROL,
                   CURSOR_ENABLE, 0);
    }
}
```

#### 2.4 光标测试方法

**使用 IGT kms_cursor_crc 测试：**

```bash
# 1. 列出所有子测试
$ sudo igt/kms_cursor_crc --list-subtests

# 输出：
# test_cursor_size_64x64
# test_cursor_size_128x128
# test_cursor_size_256x256
# test_cursor_alpha
# test_cursor_opacity
# test_cursor_movement
# test_cursor_hotspot
# test_cursor_rapid_movement

# 2. 运行基本光标测试
$ sudo igt/kms_cursor_crc --run-subtest "test_cursor_size_64x64"

# 3. 运行所有光标测试
$ sudo igt/kms_cursor_crc

# 4. 测试光标 alpha 合成
$ sudo igt/kms_cursor_crc --run-subtest "test_cursor_alpha"

# 5. 测试光标快速移动（用于检测撕裂/抖动）
$ sudo igt/kms_cursor_crc --run-subtest "test_cursor_rapid_movement"

# 6. 测试光标热点
$ sudo igt/kms_cursor_crc --run-subtest "test_cursor_hotspot"
```

**手动光标测试：**

```bash
# 1. 使用 modetest 设置光标
# 首先创建光标图片（32x32 ARGB）
$ cat > create_cursor.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

// 创建一个简单的箭头光标图案
void create_arrow_cursor(uint32_t *pixels, int size)
{
    memset(pixels, 0, size * size * 4);

    // 绘制一个指向左上角的箭头
    for (int y = 0; y < size; y++) {
        for (int x = 0; x < size; x++) {
            if (x == 0 && y < size/2) {
                // 箭头主体：蓝色不透明
                pixels[y * size + x] = 0xFF0000FF;
            } else if (y == 0 && x < size/2) {
                pixels[y * size + x] = 0xFF0000FF;
            } else if (x + y < size/2) {
                // 斜线填充
                pixels[y * size + x] = 0xFF0000FF;
            } else if (x < 4 && y < size) {
                // 细条
                pixels[y * size + x] = 0xFF0000FF;
            }
        }
    }
}

int main(int argc, char **argv)
{
    int fd;
    drmModeRes *res;
    struct drm_mode_create_dumb create = {0};
    struct drm_mode_map_dumb map = {0};
    uint32_t *pixels;
    uint32_t fb_id;
    int cursor_size = 64;
    int crtc_id, conn_id;

    if (argc < 3) {
        printf("Usage: %s <crtc_id> <conn_id>\n", argv[0]);
        return 1;
    }

    crtc_id = atoi(argv[1]);
    conn_id = atoi(argv[2]);

    fd = drmOpen("amdgpu", NULL);

    // 创建 dumb buffer 作为光标图像
    create.width = cursor_size;
    create.height = cursor_size;
    create.bpp = 32;
    drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);

    // 创建 framebuffer
    drmModeAddFB(fd, cursor_size, cursor_size, 32, 32,
                 create.pitch, create.handle, &fb_id);

    // 映射 buffer
    map.handle = create.handle;
    drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map);

    pixels = mmap(0, create.size, PROT_READ | PROT_WRITE,
                  MAP_SHARED, fd, map.offset);

    // 绘制光标图案
    create_arrow_cursor(pixels, cursor_size);
    munmap(pixels, create.size);

    // 设置光标 (SET_CURSOR2 IOCTL)
    struct drm_mode_cursor2 cursor = {
        .flags = DRM_MODE_CURSOR_BO | DRM_MODE_CURSOR_MOVE,
        .crtc_id = crtc_id,
        .x = 100,
        .y = 100,
        .width = cursor_size,
        .height = cursor_size,
        .handle = create.handle,
        .hot_x = 1,
        .hot_y = 1,
    };
    drmIoctl(fd, DRM_IOCTL_MODE_CURSOR2, &cursor);

    printf("Cursor set on CRTC %d, position (100,100)\n", crtc_id);

    // 移动光标
    for (int i = 0; i < 10; i++) {
        cursor.x = 100 + i * 20;
        cursor.y = 100 + i * 20;
        cursor.flags = DRM_MODE_CURSOR_MOVE;
        drmIoctl(fd, DRM_IOCTL_MODE_CURSOR2, &cursor);
        usleep(100000);
    }

    sleep(3);

    // 清除光标
    cursor.flags = 0;
    cursor.handle = 0;
    drmIoctl(fd, DRM_IOCTL_MODE_CURSOR2, &cursor);

    drmModeRmFB(fd, fb_id);
    drmClose(fd);
    return 0;
}
EOF

$ gcc -o create_cursor create_cursor.c -ldrm
$ ./create_cursor <crtc_id> <conn_id>

# 2. 通过 sysfs 查看光标状态
$ cat /sys/kernel/debug/dri/0/crtc-0/cursor_position
# 输出示例：x=500, y=300, enabled=1

# 3. 测试光标跨显示器
$ xrandr --output DP-1 --set "CursorSize" 128
$ xrandr --output DP-1 --set "CursorHotspot" 1,1

# 4. 查看光标性能
$ cat > cursor_perf_test.sh << 'EOF'
#!/bin/bash
# 光标性能测试

CRTC_ID=$1
DURATION=${2:-5}

echo "光标性能测试 (CRTC $CRTC_ID, ${DURATION}s)"

# 创建测试程序
cat > /tmp/cursor_move.c << 'TESTEOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

int main(int argc, char **argv)
{
    int fd, crtc_id, duration;
    struct drm_mode_cursor2 cursor;
    int x, y, dx = 2, dy = 1;
    int moves = 0;

    if (argc < 3) return 1;
    crtc_id = atoi(argv[1]);
    duration = atoi(argv[2]);

    fd = drmOpen("amdgpu", NULL);
    if (fd < 0) return 1;

    cursor.crtc_id = crtc_id;
    cursor.flags = DRM_MODE_CURSOR_MOVE;
    cursor.handle = 0;
    cursor.width = 64;
    cursor.height = 64;
    cursor.hot_x = 0;
    cursor.hot_y = 0;

    long start = time(NULL);
    while (time(NULL) - start < duration) {
        // 正弦波轨迹
        x = 100 + (int)(500 * (0.5 + 0.5 * sin(moves * 0.05)));
        y = 100 + (int)(300 * (0.5 + 0.5 * cos(moves * 0.07)));
        cursor.x = x;
        cursor.y = y;
        drmIoctl(fd, DRM_IOCTL_MODE_CURSOR2, &cursor);
        moves++;
        usleep(1000);  // 1ms
    }

    double rate = moves / (double)duration;
    printf("光标移动次数: %d\n", moves);
    printf("平均移动速率: %.0f 次/秒\n", rate);
    printf("平均延迟: %.2f us\n", 1000000.0 / rate);

    drmClose(fd);
    return 0;
}
TESTEOF

gcc -o /tmp/cursor_move /tmp/cursor_move.c -ldrm

echo "--- 测试固定光标位置 ---"
# 先在固定位置测试背景性能
timeout $DURATION /tmp/cursor_move $CRTC_ID $DURATION

echo "--- 测试结果 ---"
echo "记录 VBlank 同步状态:"
cat /sys/kernel/debug/dri/0/crtc-${CRTC_ID}/vblank_count | head -3
EOF

$ chmod +x cursor_perf_test.sh
$ ./cursor_perf_test.sh 64
```

**光标 CRC 校验测试：**

```bash
# 使用 IGT 的 CRC 验证光标像素正确性
$ sudo igt/kms_cursor_crc --run-subtest "test_cursor_size_64x64"

# kms_cursor_crc 原理：
# 1. 创建已知图案的光标
# 2. 捕获显示管线的 CRC
# 3. 与预期 CRC 值比较
# 4. 如果不匹配，表明光标合成有问题

# 手动 CRC 捕获
$ cat /sys/kernel/debug/dri/0/crtc-0/crc/data
# 启用 CRC 捕获
$ echo "auto" > /sys/kernel/debug/dri/0/crtc-0/crc/control
$ cat /sys/kernel/debug/dri/0/crtc-0/crc/data | head -5
```

### 三、覆盖（Overlay）平面详解

#### 3.1 Overlay Plane 与硬件合成

Overlay plane 允许将独立的图层直接合成到显示画面中，无需用户空间的合成器进行 GPU 合成。

```
硬件合成 vs 软件合成

软件合成（无 Overlay）：
┌────────┐   ┌────────┐
│ Desktop│   │ Video  │
│(ARGB)  │   │(NV12)  │
└───┬────┘   └───┬────┘
    └─────┬──────┘
          │ GPU 合成（耗费带宽和电能）
          ▼
    ┌──────────┐
    │ 合成器    │
    │(Mutter/  │
    │ KWin)    │
    └────┬─────┘
         │ 输出到 Primary Plane
         ▼
    ┌──────────┐
    │ 主平面    │
    │ ARGB     │
    └────┬─────┘
         │
         ▼
     显示器

硬件合成（使用 Overlay）：
┌────────┐             ┌────────┐
│ Desktop│             │ Video  │
│(ARGB)  │             │(NV12)  │
└───┬────┘             └───┬────┘
    │                      │
    ▼                      ▼
┌──────────┐         ┌──────────┐
│主平面     │         │Overlay   │
│(DPP0)    │         │(DPP1)    │
└────┬─────┘         └────┬─────┘
     │                    │
     └────────┬───────────┘
              │ 硬件合成（MPC）
              │ 零额外功耗
              ▼
         ┌──────────┐
         │  MPC     │
         │ alpha    │
         │ blending │
         └────┬─────┘
              │
              ▼
          显示器
```

#### 3.2 AMDGPU Overlay Plane 能力

```c
// drivers/gpu/drm/amd/amdgpu/dce_virtual.c
// AMDGPU 驱动中 Overlay Plane 的注册

static const struct drm_plane_funcs amdgpu_overlay_plane_funcs = {
    .update_plane = drm_atomic_helper_update_plane,
    .disable_plane = drm_atomic_helper_disable_plane,
    .destroy = drm_plane_cleanup,
    .reset = drm_atomic_helper_plane_reset,
    .atomic_duplicate_state = drm_atomic_helper_plane_duplicate_state,
    .atomic_destroy_state = drm_atomic_helper_plane_destroy_state,
};

// AMDGPU 中 Overlay Plane 的格式列表
static const uint32_t amdgpu_overlay_formats[] = {
    DRM_FORMAT_ARGB8888,
    DRM_FORMAT_XRGB8888,
    DRM_FORMAT_NV12,         // YUV 420 半平面
    DRM_FORMAT_P010,         // 10-bit YUV 420
    DRM_FORMAT_YUYV,         // YUV 422 交错
    DRM_FORMAT_UYVY,         // YUV 422 交错
    DRM_FORMAT_VYUY,         // YUV 422 交错
    DRM_FORMAT_YVYU,         // YUV 422 交错
};

// AMDGPU 中 Overlay Plane 的属性
static const struct drm_prop_enum_list amdgpu_overlay_properties[] = {
    { "zpos", 0, 0 },           // Z-order (0=最底, 255=最顶)
    { "alpha", 0, 0 },          // 全局 alpha
    { "pixel_blend_mode", 0, 0 }, // None / Premultiplied / Coverage
    { "color_encoding", 0, 0 },  // BT.601 / BT.709
    { "color_range", 0, 0 },     // Limited / Full
    { "scaling_filter", 0, 0 },  // Nearest / Bilinear / Bicubic
};
```

#### 3.3 Overlay 的 Z-order 管理

Overlay plane 的 Z-order 决定了多个 plane 的叠放顺序：

```
Z-order 示例（3 个 Overlay + 1 个 Primary）：

Z=3 (最顶层):   ┌────────────────────────┐
                 │  字幕 Overlay          │
                 │  (NV12, alpha=0.8)    │
                 └────────────────────────┘
Z=2:             ┌────────────────────────┐
                 │  视频 Overlay          │
                 │  (NV12, 1920x1080)    │
                 └────────────────────────┘
Z=1:             ┌────┐
                 │OSD │
                 │    │
                 └────┘
Z=0 (最底层):   ┌────────────────────────┐
                 │  桌面 Primary         │
                 │  (ARGB8888)          │
                 └────────────────────────┘

每个 Overlay 可以独立设置：
- 位置（crtc_x, crtc_y）
- 尺寸（crtc_w, crtc_h）
- Alpha（全局不透明度）
- 缩放（src ↔ dst 之间的尺寸变换）
- 裁剪（src 矩形区域）
```

#### 3.4 Overlay 与视频播放

Overlay 在视频播放中的典型应用：

```bash
# 1. 使用 mpv 启用 Overlay 模式
$ mpv --vo=gpu --gpu-api=vulkan --gpu-context=waylandvk \
      --hwdec=vaapi --hwdec-codecs=all \
      --vo=gpu-next --target-colorspace-hint=yes \
      video.mp4

# 2. 验证 Overlay 是否启用
# 检查 DRM plane 状态
$ cat /sys/kernel/debug/dri/0/state | grep -A20 "plane"
# 如果 overlay plane 的 fb_id 非零，表示 overlay 正在使用

# 3. 查看 overlay 缓冲区信息
$ cat /sys/kernel/debug/dri/0/state | grep "NV12\|P010\|YUYV"

# 4. 使用 IGT 验证 overlay
$ sudo igt/kms_plane --run-subtest "test_overlay_format_nv12"
$ sudo igt/kms_plane --run-subtest "test_overlay_scaling"
```

#### 3.5 Overlay 性能基准测试

```bash
# overlay 性能基准测试脚本
$ cat > overlay_benchmark.sh << 'EOF'
#!/bin/bash
# Overlay Plane 性能基准测试

CRTC_ID=$1
OVERLAY_FORMAT=${2:-NV12}
OVERLAY_W=${3:-1920}
OVERLAY_H=${4:-1080}

echo "Overlay 性能基准测试"
echo "===================="
echo "CRTC: $CRTC_ID"
echo "Format: $OVERLAY_FORMAT"
echo "Size: ${OVERLAY_W}x${OVERLAY_H}"

# 1. 创建测试用 overlay buffer
cat > /tmp/overlay_bench.c << 'TESTEOF'
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

int main(int argc, char **argv)
{
    int fd, crtc_id, conn_id;
    int overlay_w = 1920, overlay_h = 1080;
    drmModeAtomicReq *req;
    drmModePlane *plane_overlay = NULL;
    drmModePlaneRes *plane_res;
    uint32_t overlay_fb_id;
    uint32_t overlay_handle;

    if (argc < 3) {
        printf("Usage: %s <crtc_id> <conn_id> [w] [h]\n", argv[0]);
        return 1;
    }

    crtc_id = atoi(argv[1]);
    conn_id = atoi(argv[2]);
    if (argc > 3) overlay_w = atoi(argv[3]);
    if (argc > 4) overlay_h = atoi(argv[4]);

    fd = drmOpen("amdgpu", NULL);
    if (fd < 0) { perror("drmOpen"); return 1; }

    // 查找 OVERLAY plane
    plane_res = drmModeGetPlaneResources(fd);
    for (int i = 0; i < plane_res->count_planes; i++) {
        drmModePlane *p = drmModeGetPlane(fd, plane_res->planes[i]);
        if (p->possible_crtcs & (1 << crtc_id)) {
            // 检查 plane 类型
            drmModeObjectProperties *props =
                drmModeObjectGetProperties(fd, p->plane_id,
                                            DRM_MODE_OBJECT_PLANE);
            for (int j = 0; j < props->count_props; j++) {
                drmModePropertyRes *prop =
                    drmModeGetProperty(fd, props->props[j]);
                if (prop && strcmp(prop->name, "type") == 0) {
                    uint64_t type_val = props->prop_values[j];
                    if (type_val == DRM_PLANE_TYPE_OVERLAY) {
                        plane_overlay = p;
                        printf("Found OVERLAY plane: id=%d\n",
                               p->plane_id);
                    }
                }
                drmModeFreeProperty(prop);
            }
            drmModeFreeObjectProperties(props);
        }
        if (plane_overlay) break;
        drmModeFreePlane(p);
    }

    if (!plane_overlay) {
        printf("No OVERLAY plane found\n");
        return 1;
    }

    // 创建 dumb buffer 作为 overlay 帧
    struct drm_mode_create_dumb create = {0};
    create.width = overlay_w;
    create.height = overlay_h;
    create.bpp = 32;  // 对于 NV12 需要特殊处理
    drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);

    // 创建 framebuffer
    drmModeAddFB(fd, overlay_w, overlay_h, 32, 32,
                 create.pitch, create.handle, &overlay_fb_id);

    // 使用 atomic 设置 overlay
    req = drmModeAtomicAlloc();
    drmModeAtomicAddProperty(req, plane_overlay->plane_id,
                             DRM_MODE_OBJECT_PLANE,
                             plane_overlay->props[0],  // FB_ID
                             overlay_fb_id);
    drmModeAtomicAddProperty(req, plane_overlay->plane_id,
                             DRM_MODE_OBJECT_PLANE,
                             plane_overlay->props[1],  // CRTC_ID
                             crtc_id);
    drmModeAtomicAddProperty(req, plane_overlay->plane_id,
                             DRM_MODE_OBJECT_PLANE,
                             plane_overlay->props[2],  // SRC_X
                             0);
    // ... 更多属性设置

    int ret = drmModeAtomicCommit(fd, req,
                                   DRM_MODE_ATOMIC_ALLOW_MODESET,
                                   NULL);
    if (ret == 0)
        printf("Overlay enabled successfully\n");
    else
        printf("Overlay atomic commit failed: %s\n", strerror(-ret));

    drmModeAtomicFree(req);

    // 性能测量
    printf("\nPerforming overlay flip benchmark...\n");
    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);

    int flips = 500;
    for (int i = 0; i < flips; i++) {
        req = drmModeAtomicAlloc();
        drmModeAtomicAddProperty(req, plane_overlay->plane_id,
                                 DRM_MODE_OBJECT_PLANE,
                                 plane_overlay->props[0],
                                 overlay_fb_id);
        drmModeAtomicCommit(fd, req,
                             DRM_MODE_ATOMIC_NONBLOCK, NULL);
        drmModeAtomicFree(req);
    }

    clock_gettime(CLOCK_MONOTONIC, &end);
    double elapsed = (end.tv_sec - start.tv_sec) +
                     (end.tv_nsec - start.tv_nsec) / 1e9;
    printf("Overlay flip benchmark: %d flips in %.2f seconds\n",
           flips, elapsed);
    printf("Average flip rate: %.0f fps\n", flips / elapsed);

    // 清理
    drmModeRmFB(fd, overlay_fb_id);
    drmModeFreePlaneResources(plane_res);
    drmClose(fd);
    return 0;
}
TESTEOF

gcc -o /tmp/overlay_bench /tmp/overlay_bench.c -ldrm -lm

echo "执行 Overlay 基准测试..."
timeout 10 /tmp/overlay_bench $CRTC_ID 0 $OVERLAY_W $OVERLAY_H
EOF

$ chmod +x overlay_benchmark.sh
$ ./overlay_benchmark.sh 64
```

### 四、光标与 Overlay 的综合测试

#### 4.1 多 Plane 交互测试

测试光标和 Overlay 同时存在时的合成正确性：

```bash
$ cat > multi_plane_test.sh << 'EOF'
#!/bin/bash
# 多 Plane 交互测试

CRTC_ID=$1
CONN_ID=$2

echo "多 Plane 交互测试"
echo "================="

# 1. 启动 Primary plane (桌面)
echo "1. 设置 Primary plane..."
modetest -M amdgpu -s ${CRTC_ID}@${CONN_ID}:1920x1080 &
PRIMARY_PID=$!
sleep 2

# 2. 添加 Overlay plane
echo "2. 添加 Overlay plane..."
# 使用 IGT 的 kms_plane 测试 overlay
sudo igt/kms_plane --run-subtest "test_overlay_format_nv12" &
OVERLAY_PID=$!
sleep 2

# 3. 添加 Cursor plane
echo "3. 添加 Cursor plane..."
sudo igt/kms_cursor_crc --run-subtest "test_cursor_size_64x64" &
CURSOR_PID=$!
sleep 2

# 4. 检查所有 plane 状态
echo "4. 检查 plane 状态..."
STATE=$(cat /sys/kernel/debug/dri/0/state)
echo "$STATE" | grep -E "plane|fb_id|crtc_id" | head -20

# 5. 检查是否有任何错误
echo "5. 检查内核日志..."
dmesg | tail -20 | grep -i "cursor\|overlay\|plane" || echo "无相关错误"

wait
echo "测试完成"
EOF

$ sudo ./multi_plane_test.sh 64 0
```

#### 4.2 光标可见性测试

```c
// cursor_visibility.c - 测试光标在不同场景下的可见性
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

typedef struct {
    int fd;
    uint32_t crtc_id;
    uint32_t conn_id;
    drmModeModeInfo *mode;
} display_t;

// 测试场景 1：光标覆盖在 Overlay 上
int test_cursor_overlay_zorder(display_t *disp)
{
    printf("Test: Cursor Z-order vs Overlay\n");

    // 同时启用光标和 overlay
    // 预期：光标始终在 overlay 之上

    return 0;
}

// 测试场景 2：光标跨显示器边界
int test_cursor_boundary(display_t *disp)
{
    printf("Test: Cursor at display boundary\n");

    struct drm_mode_cursor2 cursor;
    cursor.crtc_id = disp->crtc_id;

    // 测试位置：显示器左上角
    cursor.flags = DRM_MODE_CURSOR_MOVE;
    cursor.x = 0;
    cursor.y = 0;
    drmIoctl(disp->fd, DRM_IOCTL_MODE_CURSOR2, &cursor);
    printf("  Position (0,0) - top-left\n");
    usleep(500000);

    // 测试位置：显示器右下角
    cursor.x = disp->mode->hdisplay - 64;
    cursor.y = disp->mode->vdisplay - 64;
    drmIoctl(disp->fd, DRM_IOCTL_MODE_CURSOR2, &cursor);
    printf("  Position (%d,%d) - bottom-right\n",
           cursor.x, cursor.y);
    usleep(500000);

    // 测试位置：完全超出屏幕
    cursor.x = -100;
    cursor.y = -100;
    drmIoctl(disp->fd, DRM_IOCTL_MODE_CURSOR2, &cursor);
    printf("  Position (-100,-100) - outside (should hide)\n");
    usleep(500000);

    printf("  Cursor boundary test completed\n");
    return 0;
}

// 测试场景 3：光标 alpha 值变化
int test_cursor_alpha(display_t *disp)
{
    printf("Test: Cursor alpha blending\n");

    // 创建不同 alpha 值的光标图案
    // 全透明 → 半透明 → 不透明

    return 0;
}

// 测试场景 4：光标热键偏移
int test_cursor_hotspot(display_t *disp)
{
    printf("Test: Cursor hotspot\n");

    struct drm_mode_cursor2 cursor;
    cursor.crtc_id = disp->crtc_id;
    cursor.flags = DRM_MODE_CURSOR_BO | DRM_MODE_CURSOR_MOVE;
    cursor.width = 64;
    cursor.height = 64;
    cursor.x = 500;
    cursor.y = 300;

    // 测试热键 (0,0)：左上角
    cursor.hot_x = 0;
    cursor.hot_y = 0;
    drmIoctl(disp->fd, DRM_IOCTL_MODE_CURSOR2, &cursor);
    printf("  Hotspot (0,0) - top-left\n");
    usleep(1000000);

    // 测试热键 (32,32)：中心
    cursor.hot_x = 32;
    cursor.hot_y = 32;
    drmIoctl(disp->fd, DRM_IOCTL_MODE_CURSOR2, &cursor);
    printf("  Hotspot (32,32) - center\n");
    usleep(1000000);

    // 测试热键 (63,63)：右下角
    cursor.hot_x = 63;
    cursor.hot_y = 63;
    drmIoctl(disp->fd, DRM_IOCTL_MODE_CURSOR2, &cursor);
    printf("  Hotspot (63,63) - bottom-right\n");
    usleep(1000000);

    printf("  Hotspot test completed\n");
    return 0;
}

int main(int argc, char **argv)
{
    display_t disp;

    if (argc < 3) {
        printf("Usage: %s <crtc_id> <conn_id>\n", argv[0]);
        return 1;
    }

    disp.fd = drmOpen("amdgpu", NULL);
    disp.crtc_id = atoi(argv[1]);
    disp.conn_id = atoi(argv[2]);

    // 获取当前模式
    drmModeConnector *conn = drmModeGetConnector(disp.fd, disp.conn_id);
    if (conn && conn->count_modes > 0) {
        disp.mode = &conn->modes[0];
    } else {
        printf("Cannot get display mode\n");
        return 1;
    }

    printf("=== Cursor Visibility Test Suite ===\n\n");

    test_cursor_boundary(&disp);
    test_cursor_hotspot(&disp);

    drmModeFreeConnector(conn);
    drmClose(disp.fd);

    printf("\n=== All tests completed ===\n");
    return 0;
}
```

### 五、光标与 Overlay 的常见问题

#### 5.1 光标常见问题

```
光标问题分类：
┌─────────────────────────────────────────────────────────────────┐
│  问题类型           │  症状                       │  常见原因    │
├─────────────────────────────────────────────────────────────────┤
│  光标缺失           │  鼠标指针完全不可见          │ 光标 buffer  │
│                     │                              │ 未正确映射   │
├─────────────────────────────────────────────────────────────────┤
│  光标残影           │  移动后残留前一个位置的图像   │ 光标 buffer  │
│                     │                              │ 未清空       │
├─────────────────────────────────────────────────────────────────┤
│  光标撕裂           │  移动时光标出现水平撕裂       │ 光标更新与   │
│                     │                              │ vblank 不同步 │
├─────────────────────────────────────────────────────────────────┤
│  光标闪烁           │  光标间歇性消失和出现         │ 光标 buffer  │
│                     │                              │ 被回收或解映射│
├─────────────────────────────────────────────────────────────────┤
│  光标尺寸异常       │  光标显示为色块或拉伸变形     │ 像素格式     │
│                     │                              │ 不匹配       │
├─────────────────────────────────────────────────────────────────┤
│  光标颜色错误       │  光标颜色与预期不符           │ alpha 预乘   │
│                     │                              │ 处理错误     │
├─────────────────────────────────────────────────────────────────┤
│  光标热点偏移       │  点击位置与视觉位置不符       │ 热键设置     │
│                     │                              │ 不正确       │
├─────────────────────────────────────────────────────────────────┤
│  多显示器光标问题   │  光标跨显示器时位置跳跃        │ CRTC 坐标    │
│                     │                              │ 转换错误     │
└─────────────────────────────────────────────────────────────────┘
```

#### 5.2 Overlay 常见问题

```
Overlay 问题分类：
┌─────────────────────────────────────────────────────────────────┐
│  问题类型           │  症状                       │  常见原因    │
├─────────────────────────────────────────────────────────────────┤
│  Overlay 不显示     │  Overlay 内容不可见           │ Z-order 错误 │
│                     │                              │ 或被遮盖     │
├─────────────────────────────────────────────────────────────────┤
│  Overlay 合成错误   │  Overlay 与背景合成后颜色错误  │ alpha 混合   │
│                     │                              │ 模式不匹配   │
├─────────────────────────────────────────────────────────────────┤
│  Overlay 性能问题   │  使用 Overlay 后帧率下降       │ 格式转换     │
│                     │                              │ 开销过大     │
├─────────────────────────────────────────────────────────────────┤
│  Overlay 格式不匹配 │  YUV 视频在 Overlay 上颜色异常  │ 色彩空间     │
│                     │                              │ 转换错误     │
├─────────────────────────────────────────────────────────────────┤
│  Overlay 缩放问题   │  视频缩放后出现锯齿或模糊       │ 缩放滤波器   │
│                     │                              │ 选择不当     │
├─────────────────────────────────────────────────────────────────┤
│  Overlay 同步问题   │  Overlay 与主画面不同步         │ Flip 同步    │
│                     │                              │ 机制问题     │
├─────────────────────────────────────────────────────────────────┤
│  Overlay 资源耗尽   │  无法创建新的 Overlay plane     │ 硬件限制     │
│                     │                              │ 或管道不足   │
└─────────────────────────────────────────────────────────────────┘
```

#### 5.3 调试技术

```bash
# 1. 查看所有 plane 状态
$ cat /sys/kernel/debug/dri/0/state | grep -A30 "plane"

# 2. 查看光标特定状态
$ for crtc in /sys/kernel/debug/dri/0/crtc-*; do
    echo "=== $(basename $crtc) ==="
    cat $crtc/cursor_position 2>/dev/null || echo "cursor_position N/A"
    cat $crtc/cursor_size 2>/dev/null || echo "cursor_size N/A"
  done

# 3. 查看 overlay 格式支持
$ modetest -M amdgpu -p | grep -A5 "OVERLAY"

# 4. 实时监控 plane 更新
$ sudo bpftrace -e '
kprobe:dm_plane_helper_commit {
    printf("Plane commit: plane_id=%d type=%d fb=%p\n",
           arg0, arg1, arg2);
}
kprobe:dcn20_set_cursor_attributes {
    printf("Cursor attr: w=%d h=%d addr=%llx\n",
           arg1->width, arg1->height, arg1->address);
}
kprobe:dcn20_set_cursor_position {
    printf("Cursor pos: x=%d y=%d enable=%d\n",
           arg1->x, arg1->y, arg1->enable);
}
'

# 5. GL compositor cursor overlay 验证
$ sudo apt install mesa-utils
$ export KMS_DRI_PRIME=1
$ modetest -M amdgpu -P
# 输出中将列出所有 plane 及其支持格式
```

### 六、光标速度与响应测试

光标响应速度是用户体验的关键指标，直接反映驱动中光标路径的延迟：

```bash
$ cat > cursor_latency_test.c << 'EOF'
// cursor_latency_test.c - 精确测量光标更新延迟
// 使用方法：在光标移动的同时用高速相机拍摄屏幕
// 或者使用光电传感器检测屏幕亮度变化
//
// 原理：创建一个全白光标在黑色背景上移动
// 同时记录时间戳，通过外部测量工具检测实际显示延迟

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

#define CURSOR_SIZE 64

int main(int argc, char **argv)
{
    int fd;
    struct drm_mode_cursor2 cursor;
    struct drm_mode_create_dumb create = {0};
    struct drm_mode_map_dumb map = {0};
    uint32_t *pixels;
    uint32_t fb_id;
    uint32_t crtc_id;
    struct timespec ts;
    FILE *log;
    int test_duration = 10;
    int iterations = 0;

    if (argc < 2) {
        printf("Usage: %s <crtc_id> [duration]\n", argv[0]);
        return 1;
    }

    crtc_id = atoi(argv[1]);
    if (argc > 2) test_duration = atoi(argv[2]);

    fd = drmOpen("amdgpu", NULL);
    log = fopen("cursor_latency.log", "w");

    // 创建光标 buffer
    create.width = CURSOR_SIZE;
    create.height = CURSOR_SIZE;
    create.bpp = 32;
    drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);
    drmModeAddFB(fd, CURSOR_SIZE, CURSOR_SIZE, 32, 32,
                 create.pitch, create.handle, &fb_id);

    map.handle = create.handle;
    drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map);
    pixels = mmap(0, create.size, PROT_READ | PROT_WRITE,
                  MAP_SHARED, fd, map.offset);

    // 填充白色光标（用于外部传感器检测）
    for (int y = 0; y < CURSOR_SIZE; y++)
        for (int x = 0; x < CURSOR_SIZE; x++)
            pixels[y * CURSOR_SIZE + x] = 0xFFFFFFFF;

    cursor.crtc_id = crtc_id;
    cursor.flags = DRM_MODE_CURSOR_BO;
    cursor.width = CURSOR_SIZE;
    cursor.height = CURSOR_SIZE;
    cursor.handle = create.handle;
    cursor.hot_x = 0;
    cursor.hot_y = 0;

    // 设置光标图像
    drmIoctl(fd, DRM_IOCTL_MODE_CURSOR2, &cursor);

    printf("开始光标延迟测试 (%d秒)...\n", test_duration);
    fprintf(log, "timestamp_ns,x,y\n");

    time_t start = time(NULL);
    while (time(NULL) - start < test_duration) {
        // 产生已知轨迹
        int x = 100 + (iterations * 3) % 1500;
        int y = 100 + (iterations * 2) % 800;

        cursor.flags = DRM_MODE_CURSOR_MOVE;
        cursor.x = x;
        cursor.y = y;

        clock_gettime(CLOCK_MONOTONIC, &ts);
        drmIoctl(fd, DRM_IOCTL_MODE_CURSOR2, &cursor);

        fprintf(log, "%ld,%d,%d\n",
                ts.tv_sec * 1000000000L + ts.tv_nsec, x, y);

        iterations++;
        usleep(1000);  // 1ms 间隔
    }

    // 清除光标
    cursor.flags = 0;
    cursor.handle = 0;
    drmIoctl(fd, DRM_IOCTL_MODE_CURSOR2, &cursor);

    munmap(pixels, create.size);
    drmModeRmFB(fd, fb_id);
    fclose(log);
    drmClose(fd);

    printf("测试完成: %d 次光标更新\n", iterations);
    printf("平均更新速率: %.0f 次/秒\n",
           iterations / (double)test_duration);
    printf("日志文件: cursor_latency.log\n");

    return 0;
}
EOF

$ gcc -o cursor_latency_test cursor_latency_test.c -ldrm
$ sudo ./cursor_latency_test 64 10
```

### 七、DRM Plane 调试接口汇总

```bash
# Plane 调试接口一览

# 1. 列出所有 planes
$ modetest -M amdgpu -p

# 示例输出：
# Plane 31 (id=31)
#     CRTCs: 64 65
#     formats: ARGB8888 XRGB8888 NV12 P010 YUYV UYVY
#     objects: plane
#         type: OVERLAY
#         zpos: 1
#         alpha: 65535
#         pixel_blend_mode: 1

# 2. 查看 plane 属性
$ modetest -M amdgpu -P

# 3. 通过 sysfs 查看 plane 状态
$ cat /sys/kernel/debug/dri/0/state | grep -B2 "plane"

# 4. 查看每个 plane 的 framebuffer
$ for plane in /sys/kernel/debug/dri/0/plane-*; do
    echo "=== $(basename $plane) ==="
    cat $plane/fb_id 2>/dev/null
  done

# 5. 实时监控 plane 变化
$ watch -n 0.5 'cat /sys/kernel/debug/dri/0/state | grep -E "plane|fb_id"'

# 6. IGT plane 测试
$ sudo igt/kms_plane --list-subtests
$ sudo igt/kms_plane --run-subtest "test_plane_position"
$ sudo igt/kms_plane --run-subtest "test_plane_alpha"
$ sudo igt/kms_plane --run-subtest "test_plane_zorder"
$ sudo igt/kms_plane --run-subtest "test_plane_scaling"
$ sudo igt/kms_plane --run-subtest "test_plane_pixel_format"

# 7. IGT cursor 测试
$ sudo igt/kms_cursor_crc --list-subtests
$ sudo igt/kms_cursor_crc --run-subtest "test_cursor_size_256x256"
```

### 八、硬件 Overlay 与合成器协作

现代 Wayland 合成器利用 Overlay 平面实现高效合成：

```
Mutter (GNOME) 合成策略：
┌─────────────────────────────────────────────────────────────────┐
│  Mutter 渲染管线                                                 │
│  1. 应用程序渲染到各自的 offscreen buffer                        │
│  2. 合成器使用 GPU 将所有窗口合成为一个最终图像                   │
│  3. 最终图像通过 Primary plane 输出                              │
│                                                                  │
│  如果启用 direct scanout:                                         │
│  - 全屏应用程序（如游戏）可直接使用 Primary plane                │
│  - 视频播放窗口可直接使用 Overlay plane                          │
│  - 合成器略过 GPU 合成步骤                                       │
│  - 大幅降低功耗和延迟                                             │
└─────────────────────────────────────────────────────────────────┘

Direct Scanout 流程：
┌─────────────────────────────────────────────────────────────────┐
│  应用程序全屏                                                      │
│      │                                                            │
│      ▼                                                            │
│  合成器检测到全屏窗口                                               │
│      │                                                            │
│      ├── 普通窗口? → 使用 Primary plane 合成                       │
│      │                                                             │
│      ├── 全屏视频? → Overlay plane: NV12 视频帧                   │
│      │                  Primary plane: 其他 UI 元素               │
│      │                                                             │
│      ├── 全屏游戏? → Primary plane: 游戏帧缓冲区                  │
│      │                  Cursor plane: 游戏内光标                  │
│      │                                                             │
│      └── 多个全屏窗口? → 多个 Overlay planes                    │
│                           每个 plane 对应一个全屏窗口             │
│                           硬件完成合成                             │
└─────────────────────────────────────────────────────────────────┘
```

### 九、实践项目：光标性能分析工具

```bash
$ cat > cursor_analyzer.py << 'EOF'
#!/usr/bin/env python3
"""
光标性能分析工具
分析 cursor_latency.log 中的数据
"""

import sys
import math
from collections import defaultdict

def analyze_cursor_log(filename):
    """分析光标延迟测试日志"""
    samples = []
    frame_intervals = []
    positions = []
    errors = []

    with open(filename) as f:
        for line in f:
            if line.startswith('timestamp_ns'):
                continue
            parts = line.strip().split(',')
            if len(parts) == 3:
                ts, x, y = int(parts[0]), int(parts[1]), int(parts[2])
                samples.append((ts, x, y))
                positions.append((x, y))

    print("=" * 60)
    print("  光标性能分析报告")
    print("=" * 60)
    print(f"总采样数: {len(samples)}")

    if len(samples) < 2:
        print("数据不足，无法分析")
        return

    # 计算帧间隔
    for i in range(1, len(samples)):
        interval = (samples[i][0] - samples[i-1][0]) / 1000000  # ms
        frame_intervals.append(interval)

    avg_interval = sum(frame_intervals) / len(frame_intervals)
    min_interval = min(frame_intervals)
    max_interval = max(frame_intervals)

    print(f"更新间隔统计:")
    print(f"  平均: {avg_interval:.3f} ms ({1000/avg_interval:.1f} Hz)")
    print(f"  最小: {min_interval:.3f} ms ({1000/min_interval:.1f} Hz)")
    print(f"  最大: {max_interval:.3f} ms ({1000/max_interval:.1f} Hz)")

    # 计算标准差
    variance = sum((i - avg_interval)**2 for i in frame_intervals) / len(frame_intervals)
    stddev = math.sqrt(variance)
    print(f"  标准差: {stddev:.3f} ms")

    # 抖动分析（jitter）
    jitter = stddev / avg_interval * 100
    print(f"  抖动率: {jitter:.2f}%")
    if jitter < 5:
        print("  抖动评级: 优秀")
    elif jitter < 10:
        print("  抖动评级: 良好")
    elif jitter < 20:
        print("  抖动评级: 一般")
    else:
        print("  抖动评级: 较差")

    # 位置分析
    if len(positions) > 1:
        dx_total = sum(abs(positions[i][0] - positions[i-1][0])
                       for i in range(1, len(positions)))
        dy_total = sum(abs(positions[i][1] - positions[i-1][1])
                       for i in range(1, len(positions)))
        total_distance = math.sqrt(dx_total**2 + dy_total**2)
        print(f"\n光标移动统计:")
        print(f"  总水平移动: {dx_total} 像素")
        print(f"  总垂直移动: {dy_total} 像素")
        print(f"  总距离: {total_distance:.0f} 像素")

        # 检测跳跃（位置突变）
        for i in range(1, len(positions)):
            jump = math.sqrt((positions[i][0] - positions[i-1][0])**2 +
                            (positions[i][1] - positions[i-1][1])**2)
            if jump > 50:  # 单步跳跃 > 50 像素视为异常
                time_ms = samples[i][0] / 1000000
                errors.append((time_ms, jump, positions[i]))

        if errors:
            print(f"\n检测到 {len(errors)} 次位置跳跃 (>50px):")
            for time_ms, jump, pos in errors[:5]:
                print(f"  T={time_ms:.1f}ms: 跳跃 {jump:.0f}px 到 ({pos[0]},{pos[1]})")
        else:
            print("  无异常位置跳跃")

    # 更新率稳定性分析
    print(f"\n更新率分布:")
    intervals_hist = defaultdict(int)
    for interval in frame_intervals:
        bucket = round(interval, 1)
        intervals_hist[bucket] += 1

    for interval in sorted(intervals_hist.keys()):
        count = intervals_hist[interval]
        bar = '#' * min(count, 50)
        freq_ms = interval
        freq_hz = 1000 / interval if interval > 0 else 0
        print(f"  {freq_ms:.1f}ms ({freq_hz:.0f}Hz): {bar} ({count})")

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("Usage: python3 cursor_analyzer.py <cursor_latency.log>")
        sys.exit(1)
    analyze_cursor_log(sys.argv[1])
EOF

$ python3 cursor_analyzer.py cursor_latency.log
```

## 相关链接

- DRM Plane 文档: https://dri.freedesktop.org/docs/drm/gpu/drm-kms.html#drm-plane
- AMDGPU DCN 光标寄存器: https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/dcn20/dcn20_plane.c
- IGT kms_cursor_crc 测试: https://gitlab.freedesktop.org/drm/igt-gpu-tools
- IGT kms_plane 测试: https://gitlab.freedesktop.org/drm/igt-gpu-tools
- DRM Plane 属性文档: https://dri.freedesktop.org/docs/drm/gpu/drm-kms.html#plane-composition-properties
- AMDGPU DM Plane: https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/amdgpu_dm/dm_plane_helpers.c
- Wayland direct scanout: https://wayland.freedesktop.org/docs/html/
- MPC 合成文档: https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/dc/inc/hw/mpc.h
- DRM atomic mode setting: https://dri.freedesktop.org/docs/drm/gpu/drm-kms.html#atomic-mode-setting

## 今日小结

本日深入学习了 DRM Plane 体系中的光标（Cursor）和覆盖（Overlay）平面。

核心收获：
1. **DRM Plane 体系**：PRIMARY / CURSOR / OVERLAY 三种平面类型，各自的用途和限制
2. **光标实现**：从用户空间的 SET_CURSOR2 IOCTL 到内核 DRM 层再到 AMDGPU DCN 硬件寄存器编程的完整路径
3. **硬件覆盖层**：Overlay 如何实现零额外功耗的硬件合成，以及 YUV 格式视频的显示
4. **Z-order 和 Alpha 合成**：多个 plane 的叠放顺序和透明度混合机制
5. **测试方法**：IGT kms_cursor_crc、modetest 手动测试和自定义性能分析工具
6. **常见问题**：光标残影、撕裂、Overlay 合成错误等问题的诊断

第 352 天将学习显示旋转（Rotation）与缩放（Scaling）的驱动实现，这是 Plane 处理中与几何变换相关的另一个重要主题。

## 扩展思考

1. **硬件光标 vs 软件光标**：在虚拟化环境（如 QEMU/KVM）中，硬件光标可能不可用或需要特殊处理。思考如何检测和回退到软件光标模式，以及这对测试自动化的影响。

2. **多 Plane 的效率边界**：虽然硬件 Overlay 可以减少合成开销，但每个额外的 plane 都会增加显示管线的带宽需求。思考在什么情况下使用多个 Overlay 反而会降低性能，以及如何通过测试确定最佳 plane 数量。

3. **Cursor Plane 的替代方案**：在现代 Wayland 合成器中，有的选择只使用 Primary plane 并通过 GPU 合成光标而不是使用专用的 Cursor plane。思考这种设计在功能和功耗方面的权衡。
