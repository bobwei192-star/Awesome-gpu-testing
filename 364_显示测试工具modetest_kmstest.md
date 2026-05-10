# Day 364：显示测试工具：modetest / kmstest

## 概述

显示测试工具是 GPU 驱动开发中不可或缺的调试验证手段。本章将深入分析 libdrm 套件中的 modetest 和 kmstest 两个核心测试工具，通过源码解读、使用方法和实践案例，掌握 KMS 用户空间接口的硬件验证技能。

### 本章内容结构

| 章节 | 内容 | 难度 |
|------|------|------|
| 1 | modetest 工具概述与编译 | ★☆☆☆☆ |
| 2 | modetest 命令行详解 | ★★☆☆☆ |
| 3 | modetest 源码架构分析 | ★★★★☆ |
| 4 | modetest 核心功能实现 | ★★★★☆ |
| 5 | kmstest 工具详解 | ★★★☆☆ |
| 6 | kms-steal 与 kms-lease 工具 | ★★★★☆ |
| 7 | 自定义测试脚本开发 | ★★★☆☆ |
| 8 | 压力测试与稳定性验证 | ★★★★☆ |
| 9 | 自动化测试框架设计 | ★★★★★ |
| 10 | 本章总结与实践 | ★★★☆☆ |

---

### 1. modetest 工具概述与编译

#### 1.1 工具定位

modetest 是 libdrm 官方提供的 KMS 测试工具，源代码位于 `libdrm/tests/modetest/` 目录。它是验证显示控制器功能的首选工具，支持：

- 枚举显示资源（CRTC、Encoder、Connector、Plane、FB）
- 测试模式设置（Mode Setting）
- 测试页面翻转（Page Flip）
- 测试 Overlay/Cursor Plane
- 测试 Atomic 模式设置
- 测试 VBlank 事件

#### 1.2 编译方法

```bash
# 1. 获取 libdrm 源码
git clone https://gitlab.freedesktop.org/mesa/drm.git
cd drm

# 2. 配置编译选项
# --enable-install-test-programs: 安装测试程序
# --enable-cairomm: 启用 Cairo 支持（modetest 图形输出）
meson setup builddir \
    -Dinstall-test-programs=true \
    -Dcairo=true \
    -Dman-pages=false

# 3. 编译
ninja -C builddir

# 4. 安装（可选）
sudo ninja -C builddir install

# 5. 验证编译结果
ls builddir/tests/modetest/modetest
file builddir/tests/modetest/modetest

# 6. 查看依赖库
ldd builddir/tests/modetest/modetest
```

#### 1.3 基本使用方法

```bash
# 查看帮助
modetest -h

# 列出所有显示资源（无需 root，只读操作）
modetest -M amdgpu -c   # 列出连接器
modetest -M amdgpu -e   # 列出编码器
modetest -M amdgpu -p   # 列出 plane
modetest -M amdgpu -f   # 列出 framebuffer

# 组合查看
modetest -M amdgpu       # 显示所有资源（简略模式）
modetest -M amdgpu -v    # 显示所有资源（详细模式）

# 测试模式设置（需 root）
modetest -M amdgpu -s 32@0:1920x1080@60
# 32 = connector ID, 0 = CRTC index, 1920x1080@60 = 分辨率

# 测试页面翻转（需 root）
modetest -M amdgpu -s 32@0:1920x1080@60 -v
# -v 启用 page flip 模式

# 测试 Overlay Plane
modetest -M amdgpu -s 32@0:1920x1080@60 -w 32@0:1920x1080@60
# -w 在指定 plane 上显示测试图案
```

#### 1.4 输出解读

```bash
$ modetest -M amdgpu -c

# 输出示例与解读：
Connectors:
id      encoder status          type    size (mm)       modes   encoders
32      30      connected       DP-1    600x340          12      30
  modes:
        index   mode (name) mode (vrefresh) hdisp hsync_start hsync_end htotal vdisp vsync_start vsync_end vtotal clock (kHz)
        0       3840x2160 (60.00 Hz) 3840 3888 3920 4000 2160 2163 2168 2222 533250
        1       2560x1440 (59.95 Hz) 2560 2608 2640 2720 1440 1443 1448 1481 241500
        2       1920x1080 (60.00 Hz) 1920 2008 2052 2200 1080 1084 1089 1125 148500
  props:
        1 EDID:
                flags: immutable blob
                blobs:
                value: 00ffff00ffffffff001e6d...
        2 DPMS:
                flags: enum
                enums: On=0 Standby=1 Suspend=2 Off=3
                value: 0
        3 link-status:
                flags: enum
                enums: Good=0 Bad=1
                value: 0
        4 non-desktop:
                flags: immutable range
                range: 0-1
                value: 0
```

### 2. modetest 命令行详解

#### 2.1 命令行参数完整列表

```bash
modetest [options]

# 通用选项
  -M, --master <driver>    指定 DRM 驱动 (如 amdgpu, i915, rockchip)
  -D, --device <file>      指定 DRM 设备文件 (默认 /dev/dri/card0)
  -h, --help               显示帮助信息
  
# 查询选项（只读，不需要 root）
  -c, --connector          列出所有连接器及其属性
  -e, --encoder            列出所有编码器
  -f, --fb                 列出所有 framebuffer
  -p, --plane              列出所有 plane
  -v, --verbose            详细输出模式

# 测试选项（需要 root）
  -s, --set <connector_id>@<crtc_id>:<mode>[-<vrefresh>][@<format>]
                            设置显示模式
  -w, --set-underline <connector_id>@<crtc_id>:<w>x<h>[+<x>+<y>][@<format>]
                            设置 overlay/underline plane
  -C, --cursor <w>x<h>     测试硬件光标
  -a, --atomic             使用 Atomic API（替代 Legacy API）
  -n, --count <count>      页面翻转次数
  -t, --time <seconds>     测试持续时间

# 高级选项
  -F, --format <format>    指定 pixel format (ARGB8888, XRGB8888, NV12...)
      --dump-blob <prop_id> 导出 blob 属性
      --set-prop <prop_id>=<value>
                            设置属性值
```

#### 2.2 常用测试场景

```bash
# 场景 1: 基本模式设置测试
# 验证单个显示器能否正常输出指定分辨率
modetest -M amdgpu -s 32@0:1920x1080 -a
# -a 使用 Atomic API，兼容性更好

# 场景 2: 多显示器测试
# 两个显示器同时输出
modetest -M amdgpu -s 32@0:1920x1080 -s 33@1:1920x1080 -a

# 场景 3: Overlay 测试
# 在主平面之上叠加一个测试图案
modetest -M amdgpu \
  -s 32@0:1920x1080 \
  -w 32@1:640x480+100+100 -a

# 场景 4: Cursor 测试
# 启用硬件光标并移动
modetest -M amdgpu \
  -s 32@0:1920x1080 -C 64x64 -a

# 场景 5: Page Flip 性能测试
# 统计页面翻转延迟
modetest -M amdgpu \
  -s 32@0:1920x1080 -n 1000 -v -a

# 场景 6: VBlank 测试
# 验证 VBlank 中断是否正常工作
modetest -M amdgpu \
  -s 32@0:1920x1080 -n 100 -v -a 2>&1 | grep VBlank

# 场景 7: 分辨率切换测试
# 在不同分辨率之间切换，验证模式设置稳定性
for mode in "1920x1080" "1280x720" "1024x768"; do
  echo "Testing mode: $mode"
  modetest -M amdgpu -s 32@0:${mode} -n 10 -a
  sleep 2
done

# 场景 8: 热插拔模拟测试
# 配合 chvt 切换 VT，模拟显示状态变化
modetest -M amdgpu -c          # 连接前状态
# ... 物理连接显示器 ...
modetest -M amdgpu -c          # 连接后状态
modetest -M amdgpu -s 32@0:1920x1080 -a  # 设置输出
```

#### 2.3 输出日志分析

```bash
# 保存测试日志
modetest -M amdgpu -v -a 2>&1 | tee modetest.log

# 关键日志解读：
# [CRTC:31] 表示操作的是 CRTC ID 31
# [CONNECTOR:32:DP-1] 表示操作的是 DP-1 接口
# [PLANE:34] 表示操作的是 Plane ID 34
# mode_set 表示进入了模式设置流程
# page_flip 表示执行了页面翻转

# 提取性能数据（页面翻转延迟）
grep "page_flip" modetest.log | \
  awk '{print $NF}' | \
  sort -n | \
  awk 'BEGIN{min=999;max=0;sum=0;n=0} \
       {if($1<min)min=$1;if($1>max)max=$1;sum+=$1;n++} \
       END{printf "Min: %.2fms  Max: %.2fms  Avg: %.2fms\n",min/1000,max/1000,sum/n/1000}'
```

### 3. modetest 源码架构分析

#### 3.1 源码文件结构

```
libdrm/tests/modetest/
├── meson.build              # Meson 编译配置
├── modetest.c               # 主入口 + 核心测试逻辑
├── modetest.h               # 头文件定义
├── buffers.c                # Buffer 分配管理
├── buffers.h                # Buffer 接口声明
└── android/                 # Android 平台适配
    └── ...
```

#### 3.2 主入口分析

```c
// modetest.c — 主入口函数
// 完整的命令行参数解析和测试调度

int main(int argc, char **argv)
{
    struct modetest_state state = {0};
    int ret;
    
    // 1. 解析命令行参数
    ret = parse_command_line(&state, argc, argv);
    if (ret)
        return ret;
    
    // 2. 打开 DRM 设备
    // 设备文件默认为 /dev/dri/card0
    // 可通过 -D 参数指定
    state.fd = open_device(state.device, state.master);
    if (state.fd < 0)
        return -errno;
    
    // 3. 获取 DRM 资源（只读操作）
    state.resources = drmModeGetResources(state.fd);
    if (!state.resources) {
        fprintf(stderr, "drmModeGetResources failed: %s\n",
                strerror(errno));
        return -errno;
    }
    
    // 4. 根据命令行参数执行不同操作
    if (state.list_connectors)
        print_connectors(&state);
    if (state.list_encoders)
        print_encoders(&state);
    if (state.list_planes)
        print_planes(&state);
    if (state.list_fbs)
        print_fbs(&state);
    
    // 5. 执行模式设置测试
    if (state.set_mode) {
        ret = set_mode(&state);
        if (ret)
            fprintf(stderr, "Mode setting failed: %d\n", ret);
    }
    
    // 6. 清理
    drmModeFreeResources(state.resources);
    close(state.fd);
    
    return ret;
}
```

#### 3.3 设备打开流程

```c
// modetest.c — DRM 设备打开
// 包含 master/slave 模式选择

static int open_device(const char *device, int master)
{
    int fd, ret;
    drmVersionPtr version;
    char buf[256];
    
    // 1. 打开设备文件
    // O_RDWR: 读写访问
    // O_CLOEXEC: 执行 exec 时自动关闭
    fd = open(device, O_RDWR | O_CLOEXEC);
    if (fd < 0) {
        fprintf(stderr, "Cannot open '%s': %s\n",
                device, strerror(errno));
        return -errno;
    }
    
    // 2. 获取驱动版本信息（验证设备是否为 DRM 设备）
    version = drmGetVersion(fd);
    if (!version) {
        fprintf(stderr, "Cannot get version for '%s': %s\n",
                device, strerror(errno));
        close(fd);
        return -errno;
    }
    
    printf("Driver: %s (version %d.%d.%d)\n",
           version->name,
           version->version_major,
           version->version_minor,
           version->version_patchlevel);
    
    // 打印驱动描述信息
    printf("Description: %s\n", version->desc);
    
    drmFreeVersion(version);
    
    // 3. 尝试获取 DRM Master 权限
    // 如果已经有其他进程是 master，会返回 EBUSY
    // 此时只能进行只读查询操作
    if (master) {
        ret = drmSetMaster(fd);
        if (ret) {
            fprintf(stderr, "Cannot set master: %s\n",
                    strerror(errno));
            fprintf(stderr, "Running in slave mode (read-only)\n");
        }
    }
    
    return fd;
}
```

#### 3.4 资源枚举实现

```c
// modetest.c — 打印连接器信息
// 展示 KMS 资源枚举的标准模式

static void print_connectors(struct modetest_state *state)
{
    int i, j;
    
    printf("Connectors:\n");
    printf("id\tencoder\tstatus\t\ttype\tsize (mm)\tmodes\tencoders\n");
    
    for (i = 0; i < state->resources->count_connectors; i++) {
        // 1. 获取连接器信息
        drmModeConnector *conn = drmModeGetConnector(
            state->fd, state->resources->connectors[i]);
        
        if (!conn) {
            fprintf(stderr, "Cannot get connector %d: %s\n",
                    state->resources->connectors[i],
                    strerror(errno));
            continue;
        }
        
        // 2. 打印连接器基本信息
        printf("%d\t%d\t%s\t\t%s\t%dx%d\t\t%d\t",
               conn->connector_id,
               conn->encoder_id,
               connector_status_str(conn->connection),
               connector_type_str(conn->connector_type),
               conn->mmWidth, conn->mmHeight,
               conn->count_modes);
        
        // 3. 打印关联的编码器列表
        for (j = 0; j < conn->count_encoders; j++)
            printf("%d ", conn->encoders[j]);
        printf("\n");
        
        // 4. 打印所有支持的显示模式
        printf("  modes:\n");
        printf("\tindex mode (name) mode (vrefresh) hdisp "
               "hsync_start hsync_end htotal vdisp "
               "vsync_start vsync_end vtotal clock(kHz)\n");
        
        for (j = 0; j < conn->count_modes; j++) {
            drmModeModeInfo *mode = &conn->modes[j];
            printf("\t%d %s (%d Hz) %d %d %d %d %d %d %d %d %d\n",
                   j,
                   mode->name,
                   mode->vrefresh ? mode->vrefresh : 60,
                   mode->hdisplay, mode->hsync_start,
                   mode->hsync_end, mode->htotal,
                   mode->vdisplay, mode->vsync_start,
                   mode->vsync_end, mode->vtotal,
                   mode->clock);
        }
        
        // 5. 打印连接器属性
        print_properties(state->fd, conn->connector_id,
                         DRM_MODE_OBJECT_CONNECTOR);
        
        drmModeFreeConnector(conn);
    }
}
```

#### 3.5 模式设置实现

```c
// modetest.c — 模式设置核心实现
// 支持 Legacy 和 Atomic 两种路径

static int set_mode(struct modetest_state *state)
{
    uint32_t connector_id = state->connector_id;
    uint32_t crtc_id = state->crtc_id;
    drmModeModeInfo *mode = &state->mode;
    int ret;
    
    // 1. 创建 framebuffer
    struct bo *bo = create_bo_for_mode(state, mode);
    if (!bo) {
        fprintf(stderr, "Failed to create BO\n");
        return -1;
    }
    
    uint32_t fb_id;
    ret = add_fb(state, bo, &fb_id);
    if (ret) {
        fprintf(stderr, "Failed to add FB: %d\n", ret);
        destroy_bo(bo);
        return ret;
    }
    
    // 2. 选择 API 路径
    if (state->use_atomic) {
        // Atomic 路径
        ret = set_mode_atomic(state, connector_id,
                              crtc_id, fb_id, mode);
    } else {
        // Legacy 路径
        ret = set_mode_legacy(state, connector_id,
                              crtc_id, fb_id, mode);
    }
    
    // 3. 清理
    if (fb_id)
        drmModeRmFB(state->fd, fb_id);
    destroy_bo(bo);
    
    return ret;
}

// Legacy 模式设置
static int set_mode_legacy(struct modetest_state *state,
                            uint32_t connector_id,
                            uint32_t crtc_id,
                            uint32_t fb_id,
                            drmModeModeInfo *mode)
{
    // drmModeSetCrtc 是传统的模式设置 API
    // 参数说明：
    //   fd: DRM 设备文件描述符
    //   crtc_id: 目标 CRTC
    //   fb_id: framebuffer ID
    //   x, y: 显示偏移（通常为 0）
    //   connectors: 连接器 ID 数组
    //   connector_count: 连接器数量
    //   mode: 显示模式（NULL 表示关闭 CRTC）
    int ret = drmModeSetCrtc(state->fd, crtc_id, fb_id,
                             0, 0,
                             &connector_id, 1,
                             mode);
    
    if (ret) {
        fprintf(stderr, "drmModeSetCrtc failed: %s\n",
                strerror(errno));
    }
    
    return ret;
}

// Atomic 模式设置
static int set_mode_atomic(struct modetest_state *state,
                            uint32_t connector_id,
                            uint32_t crtc_id,
                            uint32_t fb_id,
                            drmModeModeInfo *mode)
{
    drmModeAtomicReq *req = drmModeAtomicAlloc();
    uint32_t blob_id;
    int ret;
    
    if (!req)
        return -ENOMEM;
    
    // 1. 创建一个 blob 属性包含 mode 信息
    // drmModeCreatePropertyBlob 将 mode 结构体序列化为 blob
    ret = drmModeCreatePropertyBlob(state->fd, mode,
                                     sizeof(*mode),
                                     &blob_id);
    if (ret) {
        fprintf(stderr, "Create blob failed: %d\n", ret);
        drmModeAtomicFree(req);
        return ret;
    }
    
    // 2. 设置连接器属性
    // CRTC_ID: 连接器关联到哪个 CRTC
    uint32_t prop_id = get_prop_id(state->fd, connector_id,
                                    DRM_MODE_OBJECT_CONNECTOR,
                                    "CRTC_ID");
    drmModeAtomicAddProperty(req, connector_id, prop_id,
                             crtc_id);
    
    // 3. 设置 CRTC 属性
    // ACTIVE: 使能 CRTC
    // MODE_ID: 引用之前创建的 blob
    prop_id = get_prop_id(state->fd, crtc_id,
                           DRM_MODE_OBJECT_CRTC,
                           "ACTIVE");
    drmModeAtomicAddProperty(req, crtc_id, prop_id, 1);
    
    prop_id = get_prop_id(state->fd, crtc_id,
                           DRM_MODE_OBJECT_CRTC,
                           "MODE_ID");
    drmModeAtomicAddProperty(req, crtc_id, prop_id,
                             blob_id);
    
    // 4. 设置主平面属性
    // 找到关联的 primary plane 并设置 FB_ID
    drmModePlaneRes *plane_res = drmModeGetPlaneResources(
        state->fd);
    if (plane_res) {
        for (int i = 0; i < plane_res->count_planes; i++) {
            drmModePlane *plane = drmModeGetPlane(
                state->fd, plane_res->planes[i]);
            if (plane && plane->crtc_id == crtc_id) {
                // 是关联的 primary plane
                prop_id = get_prop_id(state->fd,
                    plane->plane_id,
                    DRM_MODE_OBJECT_PLANE, "FB_ID");
                if (prop_id)
                    drmModeAtomicAddProperty(req,
                        plane->plane_id, prop_id, fb_id);
                
                prop_id = get_prop_id(state->fd,
                    plane->plane_id,
                    DRM_MODE_OBJECT_PLANE, "CRTC_ID");
                if (prop_id)
                    drmModeAtomicAddProperty(req,
                        plane->plane_id, prop_id, crtc_id);
                
                drmModeFreePlane(plane);
                break;
            }
            drmModeFreePlane(plane);
        }
        drmModeFreePlaneResources(plane_res);
    }
    
    // 5. 提交 Atomic 请求
    // TEST_ONLY 标志可以用于验证配置是否合法
    if (state->test_only) {
        ret = drmModeAtomicCommit(state->fd, req,
            DRM_MODE_ATOMIC_TEST_ONLY |
            DRM_MODE_ATOMIC_ALLOW_MODESET,
            NULL);
        printf("TEST_ONLY: %s\n", ret ? "FAIL" : "PASS");
    }
    
    // 正式提交
    ret = drmModeAtomicCommit(state->fd, req,
        DRM_MODE_ATOMIC_ALLOW_MODESET, NULL);
    
    drmModeDestroyPropertyBlob(state->fd, blob_id);
    drmModeAtomicFree(req);
    
    return ret;
}
```

### 4. modetest 核心功能实现

#### 4.1 Buffer 分配

```c
// buffers.c — Buffer 分配管理
// 支持多种 buffer 类型：dumb、GBM、DMABUF

struct bo {
    uint32_t width;
    uint32_t height;
    uint32_t depth;
    uint32_t bpp;         // bits per pixel
    uint32_t format;      // DRM_FORMAT_*
    uint32_t handles[4];  // 每个 plane 的 GEM handle
    uint32_t pitches[4];  // 每行字节数
    uint32_t offsets[4];  // 每个 plane 的偏移
    uint64_t modifier;    // DRM format modifier
    int fd;               // DMABUF FD（如果有）
    void *map;            // CPU 映射地址
    size_t size;          // buffer 总大小
};

// 创建 dumb buffer (CPU 可访问，用于测试图案)
static struct bo *create_dumb_bo(int fd,
                                  uint32_t width,
                                  uint32_t height,
                                  uint32_t format)
{
    struct bo *bo = calloc(1, sizeof(*bo));
    struct drm_mode_create_dumb create = {0};
    struct drm_mode_map_dumb map = {0};
    int ret;
    
    bo->width = width;
    bo->height = height;
    bo->format = format;
    bo->bpp = 32;
    
    // 1. 创建 dumb buffer
    // 内核分配连续的物理内存
    create.width = width;
    create.height = height;
    create.bpp = 32;
    
    ret = drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);
    if (ret) {
        fprintf(stderr, "CREATE_DUMB failed: %s\n",
                strerror(errno));
        free(bo);
        return NULL;
    }
    
    bo->handles[0] = create.handle;
    bo->pitches[0] = create.pitch;
    bo->size = create.size;
    
    // 2. 映射到 CPU 地址空间（用于填充像素数据）
    map.handle = create.handle;
    ret = drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map);
    if (ret) {
        fprintf(stderr, "MAP_DUMB failed: %s\n",
                strerror(errno));
        drmModeDestroyDumb(fd, bo->handles[0]);
        free(bo);
        return NULL;
    }
    
    // mmap 到用户空间
    bo->map = mmap(NULL, bo->size,
                   PROT_READ | PROT_WRITE,
                   MAP_SHARED,
                   fd, map.offset);
    
    if (bo->map == MAP_FAILED) {
        fprintf(stderr, "mmap failed: %s\n",
                strerror(errno));
        drmModeDestroyDumb(fd, bo->handles[0]);
        free(bo);
        return NULL;
    }
    
    return bo;
}

// 填充测试图案（彩色渐变条）
static void fill_test_pattern(struct bo *bo)
{
    uint32_t *pixels = bo->map;
    int width = bo->width;
    int height = bo->height;
    int stride = bo->pitches[0] / 4;
    
    // 生成垂直渐变条
    // 每个竖条宽度为 width/8
    static const uint32_t colors[] = {
        0xFFFFFFFF,  // 白
        0xFFFF0000,  // 红
        0xFF00FF00,  // 绿
        0xFF0000FF,  // 蓝
        0xFFFFFF00,  // 黄
        0xFFFF00FF,  // 紫
        0xFF00FFFF,  // 青
        0xFF000000,  // 黑
    };
    
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            int bar = (x * 8) / width;
            pixels[y * stride + x] = colors[bar];
        }
    }
}
```

#### 4.2 Page Flip 实现

```c
// modetest.c — 页面翻转测试
// 展示 Page Flip 事件的完整处理流程

struct flip_state {
    struct modetest_state *state;
    struct bo **bos;          // 双缓冲 BO 数组
    int current_bo;           // 当前显示的 BO 索引
    int flip_count;           // 已完成的翻转次数
    int total_flips;          // 目标翻转次数
    struct timespec start;    // 测试开始时间
};

// Page Flip 事件回调函数
// 每个 flip 完成后内核通过 DRM 事件通知用户空间
static void page_flip_handler(int fd, unsigned int sequence,
                               unsigned int tv_sec,
                               unsigned int tv_usec,
                               void *user_data)
{
    struct flip_state *flip = user_data;
    struct timespec now;
    double elapsed;
    
    // 1. 记录翻转完成事件
    flip->flip_count++;
    clock_gettime(CLOCK_MONOTONIC, &now);
    
    elapsed = (now.tv_sec - flip->start.tv_sec) +
              (now.tv_nsec - flip->start.tv_nsec) / 1e9;
    
    printf("Flip %d: seq=%u time=%.6fs\n",
           flip->flip_count, sequence, elapsed);
    
    // 2. 检查是否达到目标次数
    if (flip->flip_count >= flip->total_flips) {
        printf("Flip test completed: %d flips in %.3fs (%.1f FPS)\n",
               flip->flip_count, elapsed,
               flip->flip_count / elapsed);
        return;
    }
    
    // 3. 准备下一次翻转
    // 切换到另一个 BO，实现双缓冲翻转
    int next_bo = 1 - flip->current_bo;
    struct bo *bo = flip->bos[next_bo];
    
    // 更新主平面的 FB_ID
    drmModeAtomicReq *req = drmModeAtomicAlloc();
    drmModeAtomicAddProperty(req,
        flip->state->plane_id,
        get_prop_id(flip->state->fd,
                     flip->state->plane_id,
                     DRM_MODE_OBJECT_PLANE,
                     "FB_ID"),
        bo->fb_id);
    
    drmModeAtomicCommit(flip->state->fd, req,
        DRM_MODE_ATOMIC_NONBLOCK |
        DRM_MODE_PAGE_FLIP_EVENT,
        flip);
    
    drmModeAtomicFree(req);
    flip->current_bo = next_bo;
}

// 初始化 Page Flip 测试
static int start_flip_test(struct modetest_state *state)
{
    struct flip_state *flip = calloc(1, sizeof(*flip));
    
    flip->state = state;
    flip->total_flips = state->flip_count;
    flip->current_bo = 0;
    flip->flip_count = 0;
    
    // 创建两个 BO 实现双缓冲
    flip->bos = calloc(2, sizeof(struct bo *));
    flip->bos[0] = create_bo_for_mode(state, &state->mode);
    flip->bos[1] = create_bo_for_mode(state, &state->mode);
    
    if (!flip->bos[0] || !flip->bos[1]) {
        fprintf(stderr, "Failed to create flip BOs\n");
        return -1;
    }
    
    // 添加 FB
    for (int i = 0; i < 2; i++) {
        if (add_fb(state, flip->bos[i],
                   &flip->bos[i]->fb_id)) {
            fprintf(stderr, "Failed to add FB %d\n", i);
            return -1;
        }
    }
    
    // 记录开始时间
    clock_gettime(CLOCK_MONOTONIC, &flip->start);
    
    // 第一次 Atomic 提交（不等待事件，通过参数传 user_data）
    drmModeAtomicReq *req = drmModeAtomicAlloc();
    drmModeAtomicAddProperty(req, state->plane_id,
        get_prop_id(state->fd, state->plane_id,
                     DRM_MODE_OBJECT_PLANE, "FB_ID"),
        flip->bos[0]->fb_id);
    
    drmModeAtomicCommit(state->fd, req,
        DRM_MODE_ATOMIC_NONBLOCK |
        DRM_MODE_PAGE_FLIP_EVENT,
        flip);
    
    drmModeAtomicFree(req);
    
    // 事件处理循环
    while (flip->flip_count < flip->total_flips) {
        struct pollfd pfd = {
            .fd = state->fd,
            .events = POLLIN,
        };
        
        // 等待 DRM 事件
        int ret = poll(&pfd, 1, 5000);
        if (ret < 0) {
            perror("poll");
            break;
        }
        
        if (pfd.revents & POLLIN) {
            // 处理 DRM 事件
            // 内核会调用 page_flip_handler 回调
            drmHandleEvent(state->fd,
                &drm_event_ctx);
        }
    }
    
    return 0;
}
```

#### 4.3 属性查询与打印

```c
// modetest.c — 属性打印通用函数
// DRM 属性是用户空间与内核通信的关键接口

static void print_properties(int fd, uint32_t obj_id,
                              uint32_t obj_type)
{
    drmModeObjectProperties *props =
        drmModeObjectGetProperties(fd, obj_id, obj_type);
    
    if (!props) {
        printf("  No properties found\n");
        return;
    }
    
    printf("  props:\n");
    
    for (uint32_t i = 0; i < props->count_props; i++) {
        drmModeProperty *prop =
            drmModeGetProperty(fd, props->props[i]);
        
        if (!prop) continue;
        
        printf("    %d %s:\n", prop->prop_id, prop->name);
        
        // 根据属性类型打印不同的值格式
        switch (prop->flags & DRM_MODE_PROP_TYPE) {
        case DRM_MODE_PROP_RANGE:
            // 范围属性：打印当前值和取值范围
            printf("      flags: range\n");
            if (prop->count_values >= 2) {
                printf("      range: %llu-%llu\n",
                       prop->values[0], prop->values[1]);
            }
            printf("      value: %llu\n",
                   props->prop_values[i]);
            break;
            
        case DRM_MODE_PROP_ENUM:
            // 枚举属性：打印所有枚举值及其含义
            printf("      flags: enum\n");
            printf("      enums:\n");
            for (int j = 0; j < prop->count_enums; j++) {
                printf("        %s=%llu\n",
                       prop->enums[j].name,
                       prop->enums[j].value);
            }
            printf("      value: %llu (%s)\n",
                   props->prop_values[i],
                   value_to_enum_name(prop,
                       props->prop_values[i]));
            break;
            
        case DRM_MODE_PROP_BLOB:
            // Blob 属性：打印 blob 大小和二进制数据
            printf("      flags: immutable blob\n");
            printf("      blobs:\n");
            // EDID、MODE_ID 等大块数据
            {
                drmModePropertyBlobPtr blob =
                    drmModeGetPropertyBlob(fd,
                        props->prop_values[i]);
                if (blob) {
                    printf("      value: %d bytes\n",
                           blob->length);
                    if (strcmp(prop->name, "EDID") == 0) {
                        printf("      EDID data:\n");
                        hex_dump(blob->data,
                                 min(128, blob->length));
                    }
                    drmModeFreePropertyBlob(blob);
                }
            }
            break;
            
        case DRM_MODE_PROP_BITMASK:
            // 位掩码属性：打印每一位的含义
            printf("      flags: bitmask\n");
            for (int j = 0; j < prop->count_enums; j++) {
                if (props->prop_values[i] &
                    (1ULL << prop->enums[j].value)) {
                    printf("        %s\n",
                           prop->enums[j].name);
                }
            }
            break;
            
        case DRM_MODE_PROP_OBJECT:
            // 对象引用属性：打印引用的对象 ID
            printf("      flags: object\n");
            printf("      value: %llu (type=%d)\n",
                   props->prop_values[i],
                   prop->values[0]);
            break;
            
        case DRM_MODE_PROP_SIGNED_RANGE:
            printf("      flags: signed range\n");
            printf("      value: %lld\n",
                   (int64_t)props->prop_values[i]);
            break;
        }
        
        // 打印通用标志
        if (prop->flags & DRM_MODE_PROP_ATOMIC)
            printf("      atomic: true\n");
        if (prop->flags & DRM_MODE_PROP_IMMUTABLE)
            printf("      immutable: true\n");
        if (prop->flags & DRM_MODE_PROP_PENDING)
            printf("      pending: true\n");
        
        drmModeFreeProperty(prop);
    }
    
    drmModeFreeObjectProperties(props);
}
```

### 5. kmstest 工具详解

#### 5.1 kmstest 概述

kmstest 是 libdrm 提供的另一个测试工具，相比 modetest，它更侧重于：

- 简化的命令行接口
- 预设测试图案（渐变、棋盘格、彩色条纹）
- 自动化的多输出测试
- 更详细的性能统计

```bash
# kmstest 基本用法
kmstest -h

# 列出所有输出
kmstest -M amdgpu -c

# 在所有输出上显示测试图案
kmstest -M amdgpu -r

# 指定输出测试（connector_id 方式）
kmstest -M amdgpu -c 32

# 测试 Page Flip 性能
kmstest -M amdgpu -r -f 1000

# 使用特定测试图案
kmstest -M amdgpu -r -t gradient     # 渐变
kmstest -M amdgpu -r -t checker      # 棋盘格
kmstest -M amdgpu -r -t stripes      # 彩色条纹
kmstest -M amdgpu -r -t solid        # 纯色
```

#### 5.2 kmstest 源码结构

```
libdrm/tests/kmstest/
├── meson.build
├── kmstest.c           # 主入口
├── kmstest.h           # 头文件
├── pattern.c           # 测试图案生成
└── pattern.h           # 图案接口
```

#### 5.3 测试图案生成

```c
// pattern.c — 测试图案生成器
// 生成多种测试图案用于显示验证

// 渐变图案
void fill_gradient(uint32_t *buf, int width, int height,
                    int stride)
{
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            uint8_t r = (x * 255) / width;
            uint8_t g = (y * 255) / height;
            uint8_t b = ((x + y) * 255) / (width + height);
            buf[y * stride + x] = (0xFF << 24) |
                                   (r << 16) |
                                   (g << 8) |
                                   b;
        }
    }
}

// 棋盘格图案（用于对齐检测）
void fill_checkerboard(uint32_t *buf, int width, int height,
                        int stride, int cell_size)
{
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            int cell_x = x / cell_size;
            int cell_y = y / cell_size;
            
            if ((cell_x + cell_y) % 2 == 0) {
                buf[y * stride + x] = 0xFFFFFFFF;  // 白色
            } else {
                buf[y * stride + x] = 0xFF000000;  // 黑色
            }
        }
    }
}

// 彩色条纹（用于色彩还原测试）
void fill_stripes(uint32_t *buf, int width, int height,
                   int stride)
{
    static const uint32_t colors[] = {
        0xFFFFFFFF,  // 白
        0xFFFFFF00,  // 黄
        0xFFFF00FF,  // 紫
        0xFFFF0000,  // 红
        0xFF00FFFF,  // 青
        0xFF00FF00,  // 绿
        0xFF0000FF,  // 蓝
        0xFF000000,  // 黑
    };
    
    int num_stripes = sizeof(colors) / sizeof(colors[0]);
    int stripe_width = (width + num_stripes - 1) / num_stripes;
    
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            int stripe = x / stripe_width;
            buf[y * stride + x] = colors[stripe % num_stripes];
        }
    }
}

// 分辨率测试图（用于清晰度评估）
void fill_resolution_test(uint32_t *buf, int width, int height,
                           int stride)
{
    // 背景色
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            buf[y * stride + x] = 0xFF808080;  // 中性灰
        }
    }
    
    // 绘制同心圆（检测几何失真）
    int cx = width / 2, cy = height / 2;
    int max_r = min(width, height) / 2 - 20;
    
    for (int r = max_r; r > 0; r -= 20) {
        for (int angle = 0; angle < 360; angle++) {
            int x = cx + r * cos(angle * M_PI / 180);
            int y = cy + r * sin(angle * M_PI / 180);
            if (x >= 0 && x < width && y >= 0 && y < height) {
                buf[y * stride + x] = 0xFF000000;
            }
        }
    }
    
    // 绘制水平和垂直细线（检测分辨率）
    for (int i = 0; i < 20; i++) {
        int x = (i * width) / 20;
        int y = (i * height) / 20;
        
        // 垂直线
        for (int ly = 0; ly < height; ly += 2)
            buf[ly * stride + x] = 0xFF000000;
        
        // 水平线
        for (int lx = 0; lx < width; lx += 2)
            buf[y * stride + lx] = 0xFF000000;
    }
}
```

#### 5.4 kmstest 性能统计

```c
// kmstest.c — 性能统计实现
// 精确测量 flip 延迟和帧率

struct perf_stats {
    int total_flips;
    int failed_flips;
    struct timespec start_time;
    struct timespec last_flip_time;
    
    // 延迟统计
    double min_latency_us;
    double max_latency_us;
    double total_latency_us;
    
    // Flip 间隔统计
    double min_interval_us;
    double max_interval_us;
    double total_interval_us;
};

static void update_perf_stats(struct perf_stats *stats,
                               struct timespec *now)
{
    double elapsed, interval;
    
    // 计算总耗时
    elapsed = (now->tv_sec - stats->start_time.tv_sec) * 1e6 +
              (now->tv_nsec - stats->start_time.tv_nsec) / 1e3;
    
    // 计算两次 flip 间隔
    if (stats->total_flips > 0) {
        interval = (now->tv_sec - stats->last_flip_time.tv_sec) *
                   1e6 +
                   (now->tv_nsec -
                    stats->last_flip_time.tv_nsec) / 1e3;
        
        stats->min_interval_us = min(stats->min_interval_us,
                                      interval);
        stats->max_interval_us = max(stats->max_interval_us,
                                      interval);
        stats->total_interval_us += interval;
    }
    
    stats->last_flip_time = *now;
    stats->total_flips++;
}

static void print_perf_stats(struct perf_stats *stats)
{
    double avg_interval = stats->total_interval_us /
                          (stats->total_flips - 1);
    
    printf("\n=== Performance Statistics ===\n");
    printf("Total flips:     %d\n", stats->total_flips);
    printf("Failed flips:    %d\n", stats->failed_flips);
    printf("Success rate:    %.2f%%\n",
           (stats->total_flips - stats->failed_flips) * 100.0 /
           stats->total_flips);
    printf("\nFlip Interval:\n");
    printf("  Min:    %.2f us (%.1f FPS)\n",
           stats->min_interval_us,
           1e6 / stats->min_interval_us);
    printf("  Max:    %.2f us (%.1f FPS)\n",
           stats->max_interval_us,
           1e6 / stats->max_interval_us);
    printf("  Avg:    %.2f us (%.1f FPS)\n",
           avg_interval, 1e6 / avg_interval);
}
```

### 6. kms-steal 与 kms-lease 工具

#### 6.1 kms-steal 工具

kms-steal 用于测试 DRM Master 的抢夺行为，模拟多进程场景。

```bash
# 基本用法
kms-steal -M amdgpu -t 10

# 参数说明
# -t: 测试持续时间（秒）
# -i: 抢夺间隔（毫秒）
# -D: DRM 设备文件路径

# 源码位置
# libdrm/tests/kms-steal/
```

```c
// kms-steal.c — DRM Master 抢夺测试
// 验证多个进程交替获取 Master 权限的稳定性

static void *steal_thread(void *arg)
{
    struct steal_test *test = arg;
    int fd;
    
    // 1. 打开 DRM 设备
    fd = open(test->device, O_RDWR | O_CLOEXEC);
    if (fd < 0) {
        fprintf(stderr, "Thread: open failed: %s\n",
                strerror(errno));
        return NULL;
    }
    
    // 2. 循环抢夺 Master
    for (int i = 0; i < test->iterations; i++) {
        // 尝试成为 Master
        int ret = drmSetMaster(fd);
        if (ret == 0) {
            printf("Thread %p: became master\n", pthread_self());
            
            // 持有 Master 一段时间
            usleep(test->hold_time_us);
            
            // 释放 Master
            drmDropMaster(fd);
            printf("Thread %p: dropped master\n", pthread_self());
        } else if (errno == EBUSY) {
            printf("Thread %p: master busy, retrying\n",
                   pthread_self());
        }
        
        usleep(test->interval_us);
    }
    
    close(fd);
    return NULL;
}

// 多线程抢夺测试
static void run_steal_test(struct steal_test *test)
{
    pthread_t threads[4];
    
    printf("Starting steal test: %d threads, %d iterations\n",
           test->num_threads, test->iterations);
    
    for (int i = 0; i < test->num_threads; i++) {
        pthread_create(&threads[i], NULL,
                       steal_thread, test);
    }
    
    for (int i = 0; i < test->num_threads; i++) {
        pthread_join(threads[i], NULL);
    }
    
    printf("Steal test completed\n");
}
```

#### 6.2 kms-lease 工具

kms-lease 用于测试 DRM Leasing 功能，主要用于 VR/AR 设备验证。

```bash
# 列出当前 leases
kms-lease -M amdgpu -l

# 创建一个 lease（将 connector 32 租出）
kms-lease -M amdgpu -c 32

# 租约的 receiver 端使用方式
# 从 lease FD 创建新的 DRM 环境
LEASE_FD=$(kms-lease -M amdgpu -c 32 2>&1 | \
           grep "lease fd" | awk '{print $NF}')
export LEASE_FD

# 在 receiver 进程中使用 lease FD
./vr_application --drm-fd $LEASE_FD
```

```c
// kms-lease.c — DRM Leasing 测试工具

struct lease_test {
    int fd;
    uint32_t *objects;
    int num_objects;
    int *lease_fds;
    int num_leases;
};

// 创建租约
static int create_lease(struct lease_test *test)
{
    int lease_fd;
    int ret;
    
    // drmModeCreateLease 创建新的 DRM FD
    // 该 FD 只能访问指定的 CRTC/Connector/Plane
    ret = drmModeCreateLease(test->fd,
                              test->objects,
                              test->num_objects,
                              0, // flags (保留)
                              &lease_fd);
    
    if (ret) {
        fprintf(stderr, "Create lease failed: %s\n",
                strerror(ret == -EACCES ? "Not master" :
                         ret == -EBUSY  ? "Already leased" :
                         ret == -EINVAL ? "Invalid combo" :
                         strerror(errno)));
        return ret;
    }
    
    printf("Lease created: fd=%d\n", lease_fd);
    printf("  Objects: ");
    for (int i = 0; i < test->num_objects; i++) {
        printf("%d ", test->objects[i]);
    }
    printf("\n");
    
    // 在租约 FD 上可以直接使用 DRM API
    // 但只能访问租约中的对象
    drmModeRes *res = drmModeGetResources(lease_fd);
    if (res) {
        printf("  Leased CRTCs: %d\n", res->count_crtcs);
        printf("  Leased connectors: %d\n",
               res->count_connectors);
        drmModeFreeResources(res);
    }
    
    return lease_fd;
}

// 撤销租约
static void revoke_lease(struct lease_test *test, int lease_fd)
{
    // 关闭 lease FD 即可撤销租约
    close(lease_fd);
    printf("Lease revoked\n");
    
    // 内核会自动恢复原始状态
    // 但通常需要重新设置显示模式
}

// 列出当前活跃的租约
static void list_leases(int fd)
{
    drmModeLessee *lessees;
    int ret;
    
    ret = drmModeListLessees(fd, &lessees, 0);
    if (ret < 0) {
        printf("No active leases\n");
        return;
    }
    
    printf("Active leases: %d\n", ret);
    for (int i = 0; i < ret; i++) {
        printf("  Lessee %d: id=%d\n", i, lessees[i]);
    }
    
    free(lessees);
}
```

### 7. 自定义测试脚本开发

#### 7.1 Shell 测试脚本

```bash
#!/bin/bash
# kms_test_suite.sh — KMS 功能测试套件
# 用于验证显示控制器的基本功能和稳定性

DRM_DEVICE=${1:-"/dev/dri/card0"}
DRIVER=${2:-"amdgpu"}
MODETEST="modetest -M $DRIVER -D $DRM_DEVICE"
LOG_DIR="kms_test_logs_$(date +%Y%m%d_%H%M%S)"
PASS=0
FAIL=0

mkdir -p $LOG_DIR

log_result() {
    local test_name=$1
    local result=$2
    local detail=$3
    
    if [ "$result" = "PASS" ]; then
        echo "[PASS] $test_name"
        PASS=$((PASS + 1))
    else
        echo "[FAIL] $test_name: $detail"
        FAIL=$((FAIL + 1))
    fi
    echo "[$result] $test_name - $detail" >> $LOG_DIR/results.log
}

# Test 1: 设备访问测试
test_device_access() {
    echo "=== Test 1: Device Access ===" >&2
    
    # 检查设备文件存在
    if [ ! -e "$DRM_DEVICE" ]; then
        log_result "Device file exists" "FAIL" "$DRM_DEVICE not found"
        return 1
    fi
    
    # 检查权限
    if [ ! -r "$DRM_DEVICE" ] || [ ! -w "$DRM_DEVICE" ]; then
        log_result "Device permissions" "FAIL" "Need R/W access"
        return 1
    fi
    
    # 测试打开设备
    $MODETEST -c > /dev/null 2>&1
    if [ $? -ne 0 ]; then
        log_result "Device open" "FAIL" "Cannot open DRM device"
        return 1
    fi
    
    log_result "Device access" "PASS" ""
    return 0
}

# Test 2: 资源枚举测试
test_resource_enum() {
    echo "=== Test 2: Resource Enumeration ===" >&2
    
    # 枚举连接器
    $MODETEST -c > $LOG_DIR/connectors.log 2>&1
    local conn_count=$(grep -c "connected" $LOG_DIR/connectors.log)
    
    if [ $conn_count -eq 0 ]; then
        log_result "Connector enumeration" "WARN" "No connected connectors found"
    else
        log_result "Connector enumeration" "PASS" "Found $conn_count connected connectors"
    fi
    
    # 枚举 Plane
    $MODETEST -p > $LOG_DIR/planes.log 2>&1
    local plane_count=$(grep -c "plane" $LOG_DIR/planes.log)
    
    if [ $plane_count -lt 2 ]; then
        log_result "Plane enumeration" "WARN" "Less than 2 planes: $plane_count"
    else
        log_result "Plane enumeration" "PASS" "Found $plane_count planes"
    fi
    
    # 保存资源信息供后续测试使用
    grep "connected" $LOG_DIR/connectors.log | \
        awk '{print $1}' > $LOG_DIR/active_connectors.txt
    
    return 0
}

# Test 3: 模式设置测试
test_mode_set() {
    echo "=== Test 3: Mode Setting ===" >&2
    
    local connectors=$(cat $LOG_DIR/active_connectors.txt)
    if [ -z "$connectors" ]; then
        log_result "Mode setting" "SKIP" "No active connectors"
        return 1
    fi
    
    # 对每个活跃连接器测试模式设置
    for conn_id in $connectors; do
        # 测试 1080p
        echo "  Testing connector $conn_id @ 1920x1080" >&2
        if $MODETEST -s ${conn_id}@0:1920x1080 -n 5 -a \
            > $LOG_DIR/modeset_${conn_id}_1080p.log 2>&1; then
            log_result "Mode set $conn_id @1080p" "PASS" ""
        else
            log_result "Mode set $conn_id @1080p" "FAIL" \
                "See $LOG_DIR/modeset_${conn_id}_1080p.log"
        fi
        
        # 测试 4K（如果支持）
        echo "  Testing connector $conn_id @ 3840x2160" >&2
        if $MODETEST -s ${conn_id}@0:3840x2160 -n 5 -a \
            > $LOG_DIR/modeset_${conn_id}_4k.log 2>&1; then
            log_result "Mode set $conn_id @4K" "PASS" ""
        else
            log_result "Mode set $conn_id @4K" "SKIP" "Not supported"
        fi
    done
}

# Test 4: Page Flip 性能测试
test_page_flip() {
    echo "=== Test 4: Page Flip Performance ===" >&2
    
    local connectors=$(cat $LOG_DIR/active_connectors.txt)
    
    for conn_id in $connectors; do
        # 翻转 200 次统计性能
        echo "  Testing flip on connector $conn_id" >&2
        $MODETEST -s ${conn_id}@0:1920x1080 -n 200 -v -a \
            > $LOG_DIR/flip_${conn_id}.log 2>&1
        
        # 提取翻转延迟数据
        local flip_times=$(grep "Flip" $LOG_DIR/flip_${conn_id}.log | \
            wc -l)
        
        if [ $flip_times -ge 100 ]; then
            # 计算 FPS
            local first_time=$(grep "Flip 1" \
                $LOG_DIR/flip_${conn_id}.log | \
                awk '{print $NF}' | sed 's/s//')
            local last_time=$(grep "Flip $flip_times" \
                $LOG_DIR/flip_${conn_id}.log 2>/dev/null | \
                awk '{print $NF}' | sed 's/s//')
            
            if [ -n "$first_time" ] && [ -n "$last_time" ]; then
                local duration=$(echo "$last_time - $first_time" | bc)
                local fps=$(echo "$flip_times / $duration" | bc)
                log_result "Flip $conn_id" "PASS" \
                    "~${fps} FPS over ${duration}s"
            fi
        else
            log_result "Flip $conn_id" "FAIL" \
                "Only $flip_times flips completed"
        fi
    done
}

# Test 5: Overlay Plane 测试
test_overlay() {
    echo "=== Test 5: Overlay Plane ===" >&2
    
    local connectors=$(cat $LOG_DIR/active_connectors.txt)
    
    for conn_id in $connectors; do
        echo "  Testing overlay on connector $conn_id" >&2
        
        # 在主平面 + overlay 上同时显示
        $MODETEST -M $DRIVER \
            -s ${conn_id}@0:1920x1080 \
            -w ${conn_id}@1:640x480+100+100 \
            -n 50 -a \
            > $LOG_DIR/overlay_${conn_id}.log 2>&1
        
        if [ $? -eq 0 ]; then
            log_result "Overlay $conn_id" "PASS" "640x480 offset +100+100"
        else
            log_result "Overlay $conn_id" "FAIL" \
                "See $LOG_DIR/overlay_${conn_id}.log"
        fi
    done
}

# Test 6: 分辨率切换稳定性测试
test_resolution_switch() {
    echo "=== Test 6: Resolution Switch Stability ===" >&2
    
    local connectors=$(cat $LOG_DIR/active_connectors.txt)
    local modes="1920x1080 1280x720 1024x768 800x600 640x480"
    
    for conn_id in $connectors; do
        local switch_count=0
        
        echo "  Testing resolution switching on connector $conn_id" >&2
        
        for mode in $modes; do
            echo "    Switching to $mode" >&2
            if $MODETEST -s ${conn_id}@0:${mode} -n 3 -a \
                > $LOG_DIR/switch_${conn_id}_${mode}.log 2>&1; then
                switch_count=$((switch_count + 1))
            fi
            sleep 1  # 等待模式切换稳定
        done
        
        log_result "Resolution switch $conn_id" "PASS" \
            "$switch_count/$((echo $modes | wc -w)) modes supported"
    done
}

# 主测试流程
main() {
    echo "KMS Test Suite - $(date)"
    echo "Device: $DRM_DEVICE"
    echo "Driver: $DRIVER"
    echo "Logs: $LOG_DIR"
    echo "========================================="
    
    test_device_access
    test_resource_enum
    test_mode_set
    test_page_flip
    test_overlay
    test_resolution_switch
    
    echo "========================================="
    echo "Results: PASS=$PASS FAIL=$FAIL"
    echo "Logs saved to: $LOG_DIR"
}

main
```

#### 7.2 C 语言测试程序

```c
// kms_custom_test.c — 自定义 KMS 测试程序
// 完整的 Atomic modesetting 测试框架

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <poll.h>
#include <sys/mman.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

#define MAX_CONNECTORS 16
#define MAX_CRTCS 8
#define MAX_PLANES 16

struct kms_device {
    int fd;
    drmModeRes *resources;
    drmModePlaneRes *plane_resources;
};

struct kms_output {
    uint32_t connector_id;
    uint32_t crtc_id;
    uint32_t plane_id;
    drmModeModeInfo mode;
    uint32_t fb_id;
    void *fb_data;
    size_t fb_size;
    uint32_t fb_handle;
    uint32_t fb_pitch;
};

// 初始化 KMS 设备
static int kms_open(struct kms_device *dev, const char *path)
{
    dev->fd = open(path, O_RDWR | O_CLOEXEC);
    if (dev->fd < 0) {
        perror("open");
        return -errno;
    }
    
    // 获取 Master 权限
    if (drmSetMaster(dev->fd)) {
        fprintf(stderr, "Cannot set master: %s\n",
                strerror(errno));
        fprintf(stderr, "Run as root or stop X11/Wayland\n");
        close(dev->fd);
        return -errno;
    }
    
    // 获取资源
    dev->resources = drmModeGetResources(dev->fd);
    if (!dev->resources) {
        fprintf(stderr, "Cannot get resources: %s\n",
                strerror(errno));
        close(dev->fd);
        return -errno;
    }
    
    // 启用 Atomic
    if (drmSetClientCap(dev->fd,
                         DRM_CLIENT_CAP_ATOMIC, 1)) {
        fprintf(stderr, "Atomic not supported\n");
    }
    
    // 获取 Plane 资源
    dev->plane_resources = drmModeGetPlaneResources(dev->fd);
    
    return 0;
}

// 查找有 connected 连接器的输出
static int kms_find_outputs(struct kms_device *dev,
                             struct kms_output *outputs,
                             int max_outputs)
{
    int count = 0;
    int found_crtc_mask = 0;
    
    for (int i = 0; i < dev->resources->count_connectors
         && count < max_outputs; i++) {
        
        drmModeConnector *conn = drmModeGetConnector(
            dev->fd, dev->resources->connectors[i]);
        
        if (!conn || conn->connection != DRM_MODE_CONNECTED) {
            drmModeFreeConnector(conn);
            continue;
        }
        
        struct kms_output *out = &outputs[count];
        memset(out, 0, sizeof(*out));
        out->connector_id = conn->connector_id;
        out->mode = conn->modes[0];  // 使用首选模式
        
        // 查找可用的 CRTC
        for (int j = 0; j < conn->count_encoders; j++) {
            drmModeEncoder *enc = drmModeGetEncoder(
                dev->fd, conn->encoders[j]);
            
            if (!enc) continue;
            
            for (int k = 0; k < dev->resources->count_crtcs; k++) {
                if ((enc->possible_crtcs & (1 << k)) &&
                    !(found_crtc_mask & (1 << k))) {
                    
                    out->crtc_id = dev->resources->crtcs[k];
                    found_crtc_mask |= (1 << k);
                    
                    // 查找关联的 primary plane
                    if (dev->plane_resources) {
                        for (int p = 0; p <
                             dev->plane_resources->count_planes; p++) {
                            
                            drmModePlane *plane = drmModeGetPlane(
                                dev->fd,
                                dev->plane_resources->planes[p]);
                            
                            if (plane &&
                                (plane->possible_crtcs & (1 << k))) {
                                
                                // 检查是否为 primary plane
                                drmModeObjectProperties *props =
                                    drmModeObjectGetProperties(
                                        dev->fd, plane->plane_id,
                                        DRM_MODE_OBJECT_PLANE);
                                
                                if (props) {
                                    for (int q = 0; q <
                                         props->count_props; q++) {
                                        drmModeProperty *prop =
                                            drmModeGetProperty(
                                                dev->fd,
                                                props->props[q]);
                                        if (prop &&
                                            strcmp(prop->name,
                                                   "type") == 0 &&
                                            props->prop_values[q]
                                            == 0) {  // Primary
                                            out->plane_id =
                                                plane->plane_id;
                                        }
                                        if (prop)
                                            drmModeFreeProperty(prop);
                                    }
                                    drmModeFreeObjectProperties(props);
                                }
                                drmModeFreePlane(plane);
                                break;
                            }
                            if (plane) drmModeFreePlane(plane);
                        }
                    }
                    
                    drmModeFreeEncoder(enc);
                    goto found;
                }
            }
            drmModeFreeEncoder(enc);
        }
        
found:
        if (out->crtc_id) {
            count++;
            printf("Output %d: conn=%d crtc=%d plane=%d %s\n",
                   count, out->connector_id, out->crtc_id,
                   out->plane_id, out->mode.name);
        }
        
        drmModeFreeConnector(conn);
    }
    
    return count;
}

// 创建 framebuffer
static int kms_create_fb(struct kms_device *dev,
                          struct kms_output *out)
{
    struct drm_mode_create_dumb create = {0};
    struct drm_mode_map_dumb map = {0};
    uint32_t handles[4] = {0};
    uint32_t pitches[4] = {0};
    uint32_t offsets[4] = {0};
    int ret;
    
    // 创建 dumb buffer
    create.width = out->mode.hdisplay;
    create.height = out->mode.vdisplay;
    create.bpp = 32;
    
    ret = drmIoctl(dev->fd, DRM_IOCTL_MODE_CREATE_DUMB,
                    &create);
    if (ret) {
        fprintf(stderr, "CREATE_DUMB failed: %s\n",
                strerror(errno));
        return ret;
    }
    
    out->fb_handle = create.handle;
    out->fb_pitch = create.pitch;
    out->fb_size = create.size;
    
    // 添加 FB
    handles[0] = create.handle;
    pitches[0] = create.pitch;
    
    ret = drmModeAddFB2(dev->fd,
                         out->mode.hdisplay,
                         out->mode.vdisplay,
                         DRM_FORMAT_XRGB8888,
                         handles, pitches, offsets,
                         &out->fb_id, 0);
    if (ret) {
        fprintf(stderr, "AddFB2 failed: %s\n",
                strerror(errno));
        return ret;
    }
    
    // 映射到 CPU
    map.handle = create.handle;
    ret = drmIoctl(dev->fd, DRM_IOCTL_MODE_MAP_DUMB, &map);
    if (ret) {
        fprintf(stderr, "MAP_DUMB failed: %s\n",
                strerror(errno));
        return ret;
    }
    
    out->fb_data = mmap(NULL, out->fb_size,
                         PROT_READ | PROT_WRITE,
                         MAP_SHARED,
                         dev->fd, map.offset);
    
    if (out->fb_data == MAP_FAILED) {
        fprintf(stderr, "mmap failed: %s\n",
                strerror(errno));
        return -errno;
    }
    
    // 填充测试图案
    uint32_t *pixels = out->fb_data;
    int stride = out->fb_pitch / 4;
    
    // 绘制彩色竖条（8条标准色）
    for (int y = 0; y < out->mode.vdisplay; y++) {
        for (int x = 0; x < out->mode.hdisplay; x++) {
            int bar = (x * 8) / out->mode.hdisplay;
            uint32_t color;
            switch (bar) {
            case 0: color = 0xFFFFFFFF; break; // 白色
            case 1: color = 0xFFFF00FF; break; // 黄色
            case 2: color = 0xFF00FFFF; break; // 青色
            case 3: color = 0xFF00FF00; break; // 绿色
            case 4: color = 0xFFFF00FF; break; // 品红
            case 5: color = 0xFFFF0000; break; // 红色
            case 6: color = 0xFF0000FF; break; // 蓝色
            case 7: color = 0xFF000000; break; // 黑色
            }
            pixels[y * stride + x] = color;
        }
    }
    
    return 0;
}

//kms_set_mode — 使用 Atomic API 设置显示模式
int kms_set_mode(struct kms_device *dev,
                 struct kms_output *out)
{
    drmModeAtomicReq *req = drmModeAtomicAlloc();
    if (!req) {
        fprintf(stderr, "AtomicAlloc failed\n");
        return -ENOMEM;
    }
    
    uint32_t blob_id;
    int ret;
    
    // 创建模式 Blob
    ret = drmModeCreatePropertyBlob(dev->fd,
                                     &out->mode,
                                     sizeof(out->mode),
                                     &blob_id);
    if (ret) {
        fprintf(stderr, "CreatePropertyBlob failed: %s\n",
                strerror(errno));
        drmModeAtomicFree(req);
        return ret;
    }
    
    // 添加 CRTC 属性
    drmModeAtomicAddProperty(req, out->crtc->crtc_id,
                              dev->props[PROP_MODE_ID],
                              blob_id);
    drmModeAtomicAddProperty(req, out->crtc->crtc_id,
                              dev->props[PROP_ACTIVE], 1);
    
    // 添加 Plane 属性
    drmModeAtomicAddProperty(req, out->plane->plane_id,
                              dev->props[PROP_FB_ID],
                              out->fb_id);
    drmModeAtomicAddProperty(req, out->plane->plane_id,
                              dev->props[PROP_CRTC_ID],
                              out->crtc->crtc_id);
    drmModeAtomicAddProperty(req, out->plane->plane_id,
                              dev->props[PROP_SRC_X], 0);
    drmModeAtomicAddProperty(req, out->plane->plane_id,
                              dev->props[PROP_SRC_Y], 0);
    drmModeAtomicAddProperty(req, out->plane->plane_id,
                              dev->props[PROP_SRC_W],
                              out->mode.hdisplay << 16);
    drmModeAtomicAddProperty(req, out->plane->plane_id,
                              dev->props[PROP_SRC_H],
                              out->mode.vdisplay << 16);
    drmModeAtomicAddProperty(req, out->plane->plane_id,
                              dev->props[PROP_CRTC_X], 0);
    drmModeAtomicAddProperty(req, out->plane->plane_id,
                              dev->props[PROP_CRTC_Y], 0);
    drmModeAtomicAddProperty(req, out->plane->plane_id,
                              dev->props[PROP_CRTC_W],
                              out->mode.hdisplay);
    drmModeAtomicAddProperty(req, out->plane->plane_id,
                              dev->props[PROP_CRTC_H],
                              out->mode.vdisplay);
    
    // 添加 Connector 属性
    drmModeAtomicAddProperty(req, out->conn->connector_id,
                              dev->props[PROP_CRTC_ID],
                              out->crtc->crtc_id);
    
    // TEST_ONLY 验证
    ret = drmModeAtomicCommit(dev->fd, req,
                               DRM_MODE_ATOMIC_TEST_ONLY |
                               DRM_MODE_ATOMIC_ALLOW_MODESET,
                               NULL);
    if (ret) {
        fprintf(stderr, "Atomic TEST_ONLY failed: %s\n",
                strerror(errno));
        drmModeAtomicFree(req);
        return ret;
    }
    
    // 实际提交
    ret = drmModeAtomicCommit(dev->fd, req,
                               DRM_MODE_ATOMIC_ALLOW_MODESET,
                               NULL);
    if (ret) {
        fprintf(stderr, "Atomic Commit failed: %s\n",
                strerror(errno));
        drmModeAtomicFree(req);
        return ret;
    }
    
    drmModeAtomicFree(req);
    return 0;
}

// kms_cleanup — 释放资源
void kms_cleanup(struct kms_device *dev,
                  struct kms_output *out, int count)
{
    for (int i = 0; i < count; i++) {
        if (out[i].fb_data)
            munmap(out[i].fb_data, out[i].fb_size);
        if (out[i].fb_id)
            drmModeRmFB(dev->fd, out[i].fb_id);
        if (out[i].fb_handle)
            drmIoctl(dev->fd, DRM_IOCTL_MODE_DESTROY_DUMB,
                     &(struct drm_mode_destroy_dumb){
                         .handle = out[i].fb_handle});
    }
    for (int i = 0; i < dev->num_planes; i++)
        drmModeFreePlane(dev->planes[i]);
    for (int i = 0; i < dev->num_crtcs; i++)
        drmModeFreeCrtc(dev->crtcs[i]);
    for (int i = 0; i < dev->num_connectors; i++)
        drmModeFreeConnector(dev->connectors[i]);
    free(dev->planes);
    free(dev->crtcs);
    free(dev->connectors);
    free(dev->encoders);
    free(out);
    close(dev->fd);
}

// main — 主测试入口
int main(int argc, char *argv[])
{
    const char *device = "/dev/dri/card0";
    const char *driver = "amdgpu";
    
    int opt;
    while ((opt = getopt(argc, argv, "M:D:")) != -1) {
        switch (opt) {
        case 'M': driver = optarg; break;
        case 'D': device = optarg; break;
        default:
            fprintf(stderr, "Usage: %s [-M driver] [-D device]\n",
                    argv[0]);
            return 1;
        }
    }
    
    struct kms_device dev = {0};
    if (kms_open(&dev, device) < 0)
        return 1;
    
    int num_outputs;
    struct kms_output *outputs;
    if (kms_find_outputs(&dev, &outputs, &num_outputs) < 0) {
        kms_cleanup(&dev, NULL, 0);
        return 1;
    }
    
    printf("Found %d active outputs\n", num_outputs);
    
    for (int i = 0; i < num_outputs; i++) {
        printf("\n=== Testing output %d: %s ===\n",
               i, outputs[i].conn->connector_type_name);
        
        if (kms_create_fb(&dev, &outputs[i]) < 0) {
            fprintf(stderr, "Failed to create FB for output %d\n", i);
            continue;
        }
        
        if (kms_set_mode(&dev, &outputs[i]) < 0) {
            fprintf(stderr, "Failed to set mode for output %d\n", i);
            continue;
        }
        
        printf("Mode set: %dx%d@%dHz\n",
               outputs[i].mode.hdisplay,
               outputs[i].mode.vdisplay,
               outputs[i].mode.vrefresh);
        
        sleep(3);
    }
    
    kms_cleanup(&dev, outputs, num_outputs);
    printf("\nTest completed successfully\n");
    return 0;
}
```

---

### 8. 压力测试与稳定性验证

#### 8.1 测试目标

压力测试旨在验证显示控制器在极端条件下的稳定性：

| 测试类型 | 目标 | 持续时间 |
|---------|------|---------|
| 长时间页面翻转 | 检测内存泄漏和 FIFO 溢出 | ≥ 24 小时 |
| 分辨率切换 | 验证 PLL 锁定和时序重配 | ≥ 1000 次 |
| 热插拔模拟 | 验证 HPD 中断处理 | ≥ 500 次 |
| 多显示器 | 验证带宽分配和时钟同步 | ≥ 8 小时 |
| 电源状态切换 | 验证 S3/S4 恢复 | ≥ 200 次 |

#### 8.2 长时间翻转测试脚本

```bash
#!/bin/bash
# stress_flip.sh — 长时间页面翻转压力测试

DRM_DEVICE=${1:-"/dev/dri/card0"}
DRIVER=${2:-"amdgpu"}
DURATION=${3:-86400}  # 默认 24 小时
LOG_FILE="stress_flip_$(date +%Y%m%d_%H%M%S).log"

echo "Starting flip stress test for ${DURATION}s" | tee $LOG_FILE

# 获取可用输出
CONNECTORS=$(modetest -M $DRIVER -D $DRM_DEVICE -c | 
             awk '/connected/ {print $1}')

start_time=$(date +%s)
end_time=$((start_time + DURATION))
flip_count=0
error_count=0

while [ $(date +%s) -lt $end_time ]; do
    for conn in $CONNECTORS; do
        # 使用 Atomic 模式翻转
        modetest -M $DRIVER -D $DRM_DEVICE \
            -a -s ${conn}@0:1920x1080 \
            -v 60 >> $LOG_FILE 2>&1
        
        if [ $? -eq 0 ]; then
            ((flip_count++))
        else
            ((error_count++))
            echo "ERROR at flip $flip_count: connector $conn" | 
                tee -a $LOG_FILE
        fi
        
        # 每秒报告
        elapsed=$(( $(date +%s) - start_time ))
        if [ $((flip_count % 60)) -eq 0 ]; then
            echo "[${elapsed}s] Flips: $flip_count, Errors: $error_count" \
                | tee -a $LOG_FILE
        fi
    done
done

echo "=== Stress Test Complete ===" | tee -a $LOG_FILE
echo "Total flips: $flip_count" | tee -a $LOG_FILE
echo "Total errors: $error_count" | tee -a $LOG_FILE
echo "Error rate: $(echo "scale=4; $error_count * 100 / $flip_count" | bc)%" \
    | tee -a $LOG_FILE
```

#### 8.3 分辨率切换测试

```bash
#!/bin/bash
# stress_resolution.sh — 分辨率切换压力测试

DRM_DEVICE=${1:-"/dev/dri/card0"}
DRIVER=${2:-"amdgpu"}
ITERATIONS=${3:-1000}
REPORT_FILE="resolution_test_$(date +%Y%m%d_%H%M%S).csv"

echo "iteration,connector,resolution,result,duration_ms" > $REPORT_FILE

# 收集所有支持的分辨率
declare -A RES_MAP
while read -r conn res; do
    RES_MAP[$conn]+="$res "
done < <(modetest -M $DRIVER -D $DRM_DEVICE -c |
         grep -E '^\s+\d+' | awk '{print $2, $1}')

for ((i=1; i<=ITERATIONS; i++)); do
    for conn in "${!RES_MAP[@]}"; do
        for res in ${RES_MAP[$conn]}; do
            start=$(date +%s%N)
            
            modetest -M $DRIVER -D $DRM_DEVICE \
                -a -s ${conn}@0:${res} > /dev/null 2>&1
            
            end=$(date +%s%N)
            duration_ms=$(( (end - start) / 1000000 ))
            
            if [ $? -eq 0 ]; then
                echo "$i,$conn,$res,PASS,$duration_ms" >> $REPORT_FILE
            else
                echo "$i,$conn,$res,FAIL,$duration_ms" >> $REPORT_FILE
            fi
        done
    done
    
    if [ $((i % 100)) -eq 0 ]; then
        echo "Completed $i/$ITERATIONS iterations"
    fi
done

echo "Resolution switching test complete"
echo "Report saved to $REPORT_FILE"

# 生成统计
echo "=== Summary ==="
awk -F',' 'NR>1 {res[$4]++} END {for (r in res) print r": "res[r]}' $REPORT_FILE
```

#### 8.4 多显示器带宽测试

```c
// 多显示器带宽压力测试
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <xf86drm.h>
#include <xf86drmMode.h>
#include <drm_fourcc.h>
#include <sys/time.h>

struct bandwidth_stats {
    long long total_bytes;
    long long total_pixels;
    double elapsed_sec;
    double fps;
    double gbps;
};

static struct bandwidth_stats stats = {0};

// 计算单条连接的带宽需求
double calc_connector_bandwidth(drmModeModeInfo *mode,
                                int bpp)
{
    // Pixel clock = H_total × V_total × refresh × 1.05 (blanking)
    double pixel_clock = mode->htotal * mode->vtotal *
                         mode->vrefresh * 1.05;
    // Data rate = pixel_clock × bpp
    return pixel_clock * bpp;  // bps
}

// 测试所有输出同时翻转
int stress_multi_display(int fd, drmModeRes *res,
                         drmModeConnector **conns,
                         int num_conns)
{
    struct timeval start, now;
    gettimeofday(&start, NULL);
    
    int frame_count = 0;
    const int TARGET_FRAMES = 600;  // ~10 秒 @ 60Hz
    
    while (frame_count < TARGET_FRAMES) {
        for (int i = 0; i < num_conns; i++) {
            if (conns[i]->connection != DRM_MODE_CONNECTED)
                continue;
            
            // 为每个输出创建 FB
            uint32_t handles[4] = {0}, pitches[4] = {0};
            uint32_t offsets[4] = {0}, fb_id;
            struct drm_mode_create_dumb create = {
                .width = conns[i]->modes[0].hdisplay,
                .height = conns[i]->modes[0].vdisplay,
                .bpp = 32
            };
            
            int ret = drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB,
                                &create);
            if (ret) continue;
            
            handles[0] = create.handle;
            pitches[0] = create.pitch;
            
            ret = drmModeAddFB2(fd, create.width, create.height,
                                 DRM_FORMAT_XRGB8888,
                                 handles, pitches, offsets,
                                 &fb_id, 0);
            if (ret) {
                drmIoctl(fd, DRM_IOCTL_MODE_DESTROY_DUMB,
                         &(struct drm_mode_destroy_dumb){
                             .handle = create.handle});
                continue;
            }
            
            // 原子提交
            drmModeAtomicReq *req = drmModeAtomicAlloc();
            if (req) {
                uint32_t blob_id;
                drmModeCreatePropertyBlob(fd, &conns[i]->modes[0],
                                           sizeof(conns[i]->modes[0]),
                                           &blob_id);
                
                drmModeAtomicAddProperty(req,
                    res->crtcs[i], /* MODE_ID */ 0, blob_id);
                drmModeAtomicAddProperty(req,
                    res->crtcs[i], /* ACTIVE */ 0, 1);
                
                drmModeAtomicCommit(fd, req,
                    DRM_MODE_ATOMIC_ALLOW_MODESET, NULL);
                drmModeAtomicFree(req);
            }
            
            drmModeRmFB(fd, fb_id);
            stats.total_pixels += create.width * create.height;
            stats.total_bytes += create.width * create.height * 4;
        }
        
        frame_count++;
    }
    
    gettimeofday(&now, NULL);
    stats.elapsed_sec = (now.tv_sec - start.tv_sec) +
                         (now.tv_usec - start.tv_usec) / 1e6;
    stats.fps = frame_count / stats.elapsed_sec;
    stats.gbps = (stats.total_bytes * 8) / stats.elapsed_sec / 1e9;
    
    return 0;
}

int main(int argc, char *argv[])
{
    const char *device = "/dev/dri/card0";
    int fd = open(device, O_RDWR);
    if (fd < 0) {
        fprintf(stderr, "Cannot open %s: %s\n",
                device, strerror(errno));
        return 1;
    }
    
    drmModeRes *res = drmModeGetResources(fd);
    if (!res) {
        fprintf(stderr, "Cannot get resources\n");
        close(fd);
        return 1;
    }
    
    drmModeConnector **conns = calloc(res->count_connectors,
                                       sizeof(drmModeConnector*));
    int num_connected = 0;
    
    for (int i = 0; i < res->count_connectors; i++) {
        conns[i] = drmModeGetConnector(fd, res->connectors[i]);
        if (conns[i] && conns[i]->connection == DRM_MODE_CONNECTED) {
            num_connected++;
            
            double bw = calc_connector_bandwidth(&conns[i]->modes[0],
                                                   32);
            printf("Connector %d (%s): %dx%d @ %dHz, BW=%.2f Gbps\n",
                   i,
                   conns[i]->connector_type_name,
                   conns[i]->modes[0].hdisplay,
                   conns[i]->modes[0].vdisplay,
                   conns[i]->modes[0].vrefresh,
                   bw / 1e9);
        }
    }
    
    printf("\nStarting multi-display stress test...\n");
    stress_multi_display(fd, res, conns, res->count_connectors);
    
    printf("\n=== Bandwidth Test Results ===\n");
    printf("Total pixels rendered: %lld\n", stats.total_pixels);
    printf("Total data transferred: %.2f GB\n",
           stats.total_bytes / 1e9);
    printf("Elapsed time: %.2f s\n", stats.elapsed_sec);
    printf("Average FPS: %.2f\n", stats.fps);
    printf("Effective bandwidth: %.2f Gbps\n", stats.gbps);
    
    for (int i = 0; i < res->count_connectors; i++)
        if (conns[i]) drmModeFreeConnector(conns[i]);
    free(conns);
    drmModeFreeResources(res);
    close(fd);
    
    return 0;
}
```

#### 8.5 电源状态切换测试

```bash
#!/bin/bash
# stress_power.sh — 电源状态切换测试

DRM_DEVICE=${1:-"/dev/dri/card0"}
ITERATIONS=${2:-200}

echo "Power state switching test: $ITERATIONS iterations"

for ((i=1; i<=ITERATIONS; i++)); do
    # S3 (Suspend to RAM)
    echo "Iteration $i: Entering S3..."
    echo mem > /sys/power/state
    
    # 恢复后等待
    sleep 2
    
    # 验证显示状态
    if modetest -D $DRM_DEVICE -c > /dev/null 2>&1; then
        echo "  Display recovered OK"
        
        # 验证模式设置
        CONN=$(modetest -D $DRM_DEVICE -c | 
               grep connected | head -1 | awk '{print $1}')
        if [ -n "$CONN" ]; then
            modetest -D $DRM_DEVICE -a -s ${CONN}@0:1920x1080 \
                > /dev/null 2>&1 && \
                echo "  Mode set OK" || \
                echo "  Mode set FAIL"
        fi
    else
        echo "  Display recovery FAILED"
    fi
    
    sleep 1
done

echo "Power state test complete"
```

#### 8.6 稳定性监控脚本

```bash
#!/bin/bash
# monitor_display.sh — 实时显示控制器监控

DRM_DEVICE=${1:-"/dev/dri/card0"}
DRIVER=${2:-"amdgpu"}
INTERVAL=${3:-5}

echo "Starting display controller monitor..."
echo "Device: $DRM_DEVICE, Interval: ${INTERVAL}s"
echo ""

while true; do
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    
    # 获取 CRTC 状态
    echo "[$timestamp] CRTC Status:"
    modetest -M $DRIVER -D $DRM_DEVICE -e 2>/dev/null | 
        while read line; do
            if echo $line | grep -qE '^\s+\d+'; then
                crtc_id=$(echo $line | awk '{print $1}')
                echo "  CRTC[$crtc_id]: $(echo $line | 
                    awk '{print $2, $3, $4, $5}')"
            fi
        done
    
    # 获取 VBlank 计数
    echo "  VBlank counts:"
    for crtc in $(modetest -M $DRIVER -D $DRM_DEVICE -e 2>/dev/null |
                  grep -E '^\s+\d+' | awk '{print $1}'); do
        vblank=$(cat /sys/kernel/debug/dri/0/crtc-${crtc}/vblank_count \
                 2>/dev/null || echo "N/A")
        echo "    CRTC[$crtc]: $vblank"
    done
    
    # 获取内存使用
    echo "  Memory usage:"
    cat /sys/kernel/debug/dri/0/gem_stats 2>/dev/null | 
        grep -E 'total|object' | head -3 | 
        while read line; do echo "    $line"; done
    
    echo ""
    sleep $INTERVAL
done
```

---

### 9. 自动化测试框架设计

#### 9.1 框架架构

自动化测试框架整合 modetest、kmstest 和自定义测试工具，提供统一的测试执行和报告生成能力。

```
kms_test_framework/
├── core/
│   ├── runner.py        # 测试执行引擎
│   ├── reporter.py      # 报告生成器
│   └── config.py        # 配置管理
├── tests/
│   ├── test_modeset.py  # 模式设置测试
│   ├── test_flip.py     # 页面翻转测试
│   ├── test_plane.py    # Plane 测试
│   ├── test_hotplug.py  # 热插拔测试
│   ├── test_stress.py   # 压力测试
│   └── test_power.py    # 电源管理测试
├── tools/
│   ├── modetest_wrapper.py  # modetest 封装
│   └── analyzer.py          # 日志分析器
└── results/
    └── reports/          # 测试报告输出
```

#### 9.2 测试执行引擎

```python
#!/usr/bin/env python3
# runner.py — 测试执行引擎

import subprocess
import threading
import queue
import time
import json
import os
import re
from datetime import datetime
from dataclasses import dataclass, field, asdict
from typing import List, Optional

@dataclass
class TestCase:
    """测试用例定义"""
    name: str
    category: str
    command: List[str]
    timeout: int = 60
    expected_return: int = 0
    expected_patterns: List[str] = field(default_factory=list)
    env: dict = field(default_factory=dict)

@dataclass
class TestResult:
    """测试结果"""
    name: str
    category: str
    passed: bool
    return_code: int
    stdout: str
    stderr: str
    duration_ms: float
    timestamp: str
    errors: List[str] = field(default_factory=list)

class TestRunner:
    """多线程测试执行器"""
    
    def __init__(self, max_workers=4):
        self.max_workers = max_workers
        self.results = []
        self.lock = threading.Lock()
        
    def run_single(self, test: TestCase) -> TestResult:
        """执行单个测试用例"""
        start = time.time()
        errors = []
        
        try:
            proc = subprocess.run(
                test.command,
                capture_output=True,
                text=True,
                timeout=test.timeout,
                env={**os.environ, **test.env}
            )
            
            duration = (time.time() - start) * 1000
            
            passed = True
            if proc.returncode != test.expected_return:
                passed = False
                errors.append(
                    f"Return code {proc.returncode} != "
                    f"{test.expected_return}"
                )
            
            for pattern in test.expected_patterns:
                if not re.search(pattern, proc.stdout, re.MULTILINE):
                    passed = False
                    errors.append(f"Pattern not found: {pattern}")
            
            return TestResult(
                name=test.name,
                category=test.category,
                passed=passed,
                return_code=proc.returncode,
                stdout=proc.stdout,
                stderr=proc.stderr,
                duration_ms=duration,
                timestamp=datetime.now().isoformat(),
                errors=errors
            )
            
        except subprocess.TimeoutExpired:
            duration = (time.time() - start) * 1000
            return TestResult(
                name=test.name,
                category=test.category,
                passed=False,
                return_code=-1,
                stdout="",
                stderr=f"Timeout after {test.timeout}s",
                duration_ms=duration,
                timestamp=datetime.now().isoformat(),
                errors=["Timeout"]
            )
    
    def run_batch(self, tests: List[TestCase],
                  parallel: bool = True):
        """批量执行测试"""
        if parallel:
            threads = []
            sem = threading.Semaphore(self.max_workers)
            
            def worker(test):
                sem.acquire()
                result = self.run_single(test)
                with self.lock:
                    self.results.append(result)
                sem.release()
            
            for test in tests:
                t = threading.Thread(target=worker, args=(test,))
                threads.append(t)
                t.start()
            
            for t in threads:
                t.join()
        else:
            for test in tests:
                result = self.run_single(test)
                self.results.append(result)
        
        return self.results
    
    def get_summary(self) -> dict:
        """获取测试摘要"""
        total = len(self.results)
        passed = sum(1 for r in self.results if r.passed)
        failed = total - passed
        
        by_category = {}
        for r in self.results:
            by_category.setdefault(r.category, []).append(r)
        
        return {
            "total": total,
            "passed": passed,
            "failed": failed,
            "pass_rate": f"{(passed/total*100):.1f}%" if total else "N/A",
            "avg_duration_ms": (
                sum(r.duration_ms for r in self.results) / total
                if total else 0
            ),
            "by_category": {
                cat: {
                    "total": len(results),
                    "passed": sum(1 for r in results if r.passed)
                }
                for cat, results in by_category.items()
            }
        }

# 测试用例定义
def create_kms_tests(driver="amdgpu", device="/dev/dri/card0"):
    """创建 KMS 测试用例集"""
    M = ["modetest", "-M", driver, "-D", device]
    
    return [
        # 资源枚举测试
        TestCase(
            name="resource_enumeration",
            category="enumeration",
            command=M + ["-e"],
            expected_patterns=[
                r"CRTCs", r"Encoders", r"Connectors"
            ]
        ),
        
        # Framebuffer 测试
        TestCase(
            name="framebuffer_list",
            category="enumeration",
            command=M + ["-f"],
            expected_patterns=[r"framebuffers"]
        ),
        
        # Plane 枚举测试
        TestCase(
            name="plane_enumeration",
            category="enumeration",
            command=M + ["-p"],
            expected_patterns=[r"Planes", r"type"]
        ),
        
        # 连接器详情测试
        TestCase(
            name="connector_details",
            category="enumeration",
            command=M + ["-c"],
            expected_patterns=[
                r"connected", r"modes", r"Properties"
            ]
        ),
        
        # 模式设置测试 (使用 Atomic API)
        TestCase(
            name="atomic_modeset",
            category="modeset",
            command=M + ["-a", "-s", "43@0:1920x1080"],
            timeout=30,
            expected_patterns=[r"Setting"]
        ),
        
        # 页面翻转测试
        TestCase(
            name="page_flip",
            category="flip",
            command=M + ["-a", "-s", "43@0:1920x1080",
                         "-v", "10"],
            timeout=30,
            expected_patterns=[r"freq"]
        ),
        
        # VBlank 测试
        TestCase(
            name="vblank_test",
            category="vblank",
            command=M + ["-a", "-s", "43@0:1920x1080",
                         "-v", "5"],
            timeout=20,
            expected_patterns=[r"vblank"]
        ),
        
        # kmstest 模式测试
        TestCase(
            name="kmstest_modeset",
            category="kmstest",
            command=["kmstest", "-M", driver,
                     "-D", device, "-r"],
            timeout=30,
            expected_patterns=[r"Mode"]
        ),
    ]

# 压力测试用例
def create_stress_tests(driver="amdgpu",
                         device="/dev/dri/card0"):
    """创建压力测试用例"""
    return [
        TestCase(
            name="long_flip_stress",
            category="stress",
            command=[
                "bash", "stress_flip.sh",
                device, driver, "60"  # 60 秒
            ],
            timeout=120,
            expected_patterns=[r"Stress Test Complete"]
        ),
        TestCase(
            name="resolution_switch",
            category="stress",
            command=[
                "bash", "stress_resolution.sh",
                device, driver, "50"  # 50 次
            ],
            timeout=300,
            expected_patterns=[r"Summary"]
        ),
    ]

if __name__ == "__main__":
    import sys
    
    driver = sys.argv[1] if len(sys.argv) > 1 else "amdgpu"
    device = sys.argv[2] if len(sys.argv) > 2 else "/dev/dri/card0"
    
    runner = TestRunner(max_workers=2)
    
    # 执行基本测试
    print("=== KMS 基本功能测试 ===")
    basic_tests = create_kms_tests(driver, device)
    results = runner.run_batch(basic_tests, parallel=False)
    
    summary = runner.get_summary()
    print(f"\n基本测试完成: {summary['passed']}/{summary['total']} 通过")
    
    # 输出详细结果
    for r in results:
        status = "PASS" if r.passed else "FAIL"
        print(f"  [{status}] {r.name} ({r.duration_ms:.0f}ms)")
        if not r.passed:
            for err in r.errors:
                print(f"    - {err}")
    
    # 保存 JSON 报告
    report = {
        "timestamp": datetime.now().isoformat(),
        "driver": driver,
        "device": device,
        "summary": summary,
        "results": [asdict(r) for r in results]
    }
    
    report_file = f"kms_test_report_{datetime.now():%Y%m%d_%H%M%S}.json"
    with open(report_file, "w") as f:
        json.dump(report, f, indent=2)
    print(f"\n报告已保存: {report_file}")
```

#### 9.3 结果报告生成器

```python
#!/usr/bin/env python3
# reporter.py — 测试报告生成器

import json
import os
from datetime import datetime
from typing import List, Dict

class ReportGenerator:
    """HTML/JSON 报告生成器"""
    
    HTML_TEMPLATE = """
    <!DOCTYPE html>
    <html>
    <head>
        <title>KMS 测试报告</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 20px; }
            .pass { color: green; }
            .fail { color: red; }
            table { border-collapse: collapse; width: 100%%; }
            th, td { border: 1px solid #ddd; padding: 8px; 
                     text-align: left; }
            th { background-color: #f2f2f2; }
            .summary { margin: 20px 0; }
            .progress-bar { height: 20px; 
                           background-color: #e0e0e0; }
            .progress-fill { height: 100%%; 
                           background-color: #4CAF50; }
        </style>
    </head>
    <body>
        <h1>KMS 测试报告</h1>
        <p>生成时间: {timestamp}</p>
        
        <div class="summary">
            <h2>测试摘要</h2>
            <p>总计: {total} | 
               通过: <span class="pass">{passed}</span> | 
               失败: <span class="fail">{failed}</span> | 
               通过率: {pass_rate}</p>
            <div class="progress-bar">
                <div class="progress-fill" 
                     style="width: {pass_rate}"></div>
            </div>
        </div>
        
        <h2>按分类统计</h2>
        <table>
            <tr><th>分类</th><th>总数</th><th>通过</th>
                <th>通过率</th></tr>
            {category_rows}
        </table>
        
        <h2>详细结果</h2>
        <table>
            <tr><th>测试名称</th><th>分类</th><th>状态</th>
                <th>耗时(ms)</th><th>错误</th></tr>
            {detail_rows}
        </table>
        
        <h2>失败测试详细日志</h2>
        {failure_logs}
    </body>
    </html>
    """
    
    def __init__(self, report_dir="reports"):
        self.report_dir = report_dir
        os.makedirs(report_dir, exist_ok=True)
    
    def generate_html(self, report_data: dict) -> str:
        """生成 HTML 报告"""
        summary = report_data["summary"]
        results = report_data["results"]
        
        # 分类统计行
        category_rows = ""
        for cat, stat in summary["by_category"].items():
            rate = (stat["passed"]/stat["total"]*100) if stat["total"] else 0
            category_rows += (
                f"<tr><td>{cat}</td><td>{stat['total']}</td>"
                f"<td>{stat['passed']}</td>"
                f"<td>{rate:.1f}%</td></tr>"
            )
        
        # 详细结果行
        detail_rows = ""
        failures = []
        for r in results:
            status_class = "pass" if r["passed"] else "fail"
            status_text = "PASS" if r["passed"] else "FAIL"
            errors = "; ".join(r.get("errors", []))
            
            detail_rows += (
                f"<tr><td>{r['name']}</td><td>{r['category']}</td>"
                f"<td class='{status_class}'>{status_text}</td>"
                f"<td>{r['duration_ms']:.0f}</td>"
                f"<td>{errors}</td></tr>"
            )
            
            if not r["passed"]:
                failures.append(r)
        
        # 失败日志
        failure_logs = ""
        for r in failures:
            failure_logs += f"""
            <h3>{r['name']}</h3>
            <pre style="background:#f5f5f5;padding:10px;
                 overflow-x:auto;">
STDOUT:
{r.get('stdout', 'N/A')[:2000]}

STDERR:
{r.get('stderr', 'N/A')[:1000]}
            </pre>
            """
        
        pass_rate = summary["pass_rate"]
        
        return self.HTML_TEMPLATE.format(
            timestamp=report_data.get("timestamp", ""),
            total=summary["total"],
            passed=summary["passed"],
            failed=summary["failed"],
            pass_rate=pass_rate,
            category_rows=category_rows,
            detail_rows=detail_rows,
            failure_logs=failure_logs
        )
    
    def save_report(self, report_data: dict,
                    formats: List[str] = ["html", "json"]):
        """保存报告"""
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        
        saved = []
        
        if "json" in formats:
            path = os.path.join(self.report_dir,
                                f"kms_report_{timestamp}.json")
            with open(path, "w") as f:
                json.dump(report_data, f, indent=2)
            saved.append(path)
        
        if "html" in formats:
            html = self.generate_html(report_data)
            path = os.path.join(self.report_dir,
                                f"kms_report_{timestamp}.html")
            with open(path, "w") as f:
                f.write(html)
            saved.append(path)
        
        return saved

# 趋势分析
class TrendAnalyzer:
    """跨版本测试趋势分析"""
    
    def __init__(self, report_dir="reports"):
        self.report_dir = report_dir
    
    def analyze_trends(self) -> dict:
        """分析历史趋势"""
        trend_files = sorted([
            f for f in os.listdir(self.report_dir)
            if f.endswith(".json")
        ])
        
        trends = {
            "dates": [],
            "pass_rates": [],
            "avg_durations": []
        }
        
        for fname in trend_files[-30:]:  # 最近 30 次
            path = os.path.join(self.report_dir, fname)
            with open(path) as f:
                data = json.load(f)
            
            trends["dates"].append(
                data["timestamp"][:10])
            trends["pass_rates"].append(
                float(data["summary"]["pass_rate"].rstrip("%")))
            trends["avg_durations"].append(
                data["summary"]["avg_duration_ms"])
        
        return trends
```

#### 9.4 CI/CD 集成

```yaml
# .gitlab-ci.yml — KMS 测试 CI 配置
kms_test:
  stage: test
  script:
    # 环境准备
    - modprobe amdgpu
    - echo 0 > /sys/class/vtconsole/vtcon0/bind
    - echo 0 > /sys/class/vtconsole/vtcon1/bind
    - echo efi-framebuffer.0 > /sys/bus/platform/drivers/efi-framebuffer/unbind
    
    # 执行测试
    - python3 runner.py amdgpu /dev/dri/card0
    
    # 生成报告
    - python3 reporter.py --format html
  artifacts:
    paths:
      - reports/
    expire_in: 30 days
  only:
    - master
    - merge_requests
```

```yaml
# Jenkinsfile — Jenkins 流水线
pipeline {
    agent { label 'kms-test-node' }
    
    parameters {
        string(name: 'DRIVER', defaultValue: 'amdgpu')
        string(name: 'DEVICE', defaultValue: '/dev/dri/card0')
        choice(name: 'TEST_LEVEL',
               choices: ['basic', 'full', 'stress'])
    }
    
    stages {
        stage('Environment Setup') {
            steps {
                sh '''
                    modprobe ${DRIVER}
                    # 释放 VT 控制
                    for vt in /sys/class/vtconsole/vtcon*; do
                        echo 0 > $vt/bind 2>/dev/null || true
                    done
                '''
            }
        }
        
        stage('Run KMS Tests') {
            steps {
                script {
                    def testScript = ''
                    switch(params.TEST_LEVEL) {
                        case 'basic':
                            testScript = 'runner.py --level basic'
                            break
                        case 'full':
                            testScript = 'runner.py --level full'
                            break
                        case 'stress':
                            testScript = '''
                                runner.py --level basic &&
                                bash stress_flip.sh 3600 &&
                                bash stress_resolution.sh 500
                            '''
                            break
                    }
                    
                    sh "python3 ${testScript} ${DRIVER} ${DEVICE}"
                }
            }
        }
        
        stage('Generate Report') {
            steps {
                sh 'python3 reporter.py --format html --format json'
            }
        }
        
        stage('Archive Results') {
            steps {
                archiveArtifacts artifacts: 'reports/*',
                                 fingerprint: true
            }
        }
    }
    
    post {
        always {
            junit 'reports/*.xml'
            cleanWs()
        }
        failure {
            slackSend(
                channel: '#kms-test',
                message: "KMS test FAILED: ${env.BUILD_URL}"
            )
        }
    }
}
```

#### 9.5 日志分析工具

```python
#!/usr/bin/env python3
# analyzer.py — KMS 测试日志分析器

import re
from collections import defaultdict

class KMSLogAnalyzer:
    """modetest/kmstest 日志分析"""
    
    def __init__(self):
        self.metrics = defaultdict(list)
        self.errors = []
        self.warnings = []
    
    def parse_modetest_output(self, log_text: str):
        """解析 modetest 输出"""
        # 提取翻转频率
        for match in re.finditer(
            r'freq:\s+([\d.]+)\s+Hz', log_text):
            self.metrics['flip_freq'].append(
                float(match.group(1)))
        
        # 提取 VBlank 信息
        for match in re.finditer(
            r'vblank:\s+(\d+)', log_text):
            self.metrics['vblank_count'].append(
                int(match.group(1)))
        
        # 提取分辨率
        for match in re.finditer(
            r'(\d+)x(\d+)(?:@(\d+))?', log_text):
            self.metrics['resolutions'].append({
                'width': int(match.group(1)),
                'height': int(match.group(2)),
                'refresh': int(match.group(3)) 
                    if match.group(3) else 60
            })
        
        # 检测错误
        for match in re.finditer(
            r'(error|failed|timeout|critical)',
            log_text, re.IGNORECASE):
            self.errors.append({
                'line': match.group(),
                'context': log_text[
                    max(0, match.start()-50):
                    match.end()+50
                ]
            })
    
    def parse_kmstest_output(self, log_text: str):
        """解析 kmstest 输出"""
        # 提取每个输出的测试结果
        for match in re.finditer(
            r'Output\s+(\d+):\s+(\w+)', log_text):
            self.metrics['output_results'].append({
                'output': int(match.group(1)),
                'status': match.group(2)
            })
        
        # 提取性能指标
        for match in re.finditer(
            r'(\d+)\s+fps,\s+avg\s+([\d.]+)\s*ms',
            log_text):
            self.metrics['performance'].append({
                'fps': int(match.group(1)),
                'avg_latency': float(match.group(2))
            })
    
    def get_report(self) -> dict:
        """生成分析报告"""
        report = {
            'summary': {},
            'metrics': {},
            'errors': self.errors[:10],
            'warnings': self.warnings[:10]
        }
        
        if self.metrics.get('flip_freq'):
            freqs = self.metrics['flip_freq']
            report['metrics']['flip_frequency'] = {
                'avg': sum(freqs) / len(freqs),
                'min': min(freqs),
                'max': max(freqs),
                'samples': len(freqs)
            }
        
        if self.metrics.get('performance'):
            perfs = self.metrics['performance']
            report['metrics']['performance'] = {
                'avg_fps': sum(p['fps'] for p in perfs) / len(perfs),
                'avg_latency_ms': sum(
                    p['avg_latency'] for p in perfs
                ) / len(perfs)
            }
        
        if self.metrics.get('resolutions'):
            report['metrics']['modes_tested'] = len(
                self.metrics['resolutions'])
        
        report['summary'] = {
            'total_errors': len(self.errors),
            'total_warnings': len(self.warnings),
            'metrics_collected': len(self.metrics)
        }
        
        return report
    
    def compare_baseline(self, baseline: dict,
                          current: dict) -> dict:
        """与基线比较"""
        deltas = {}
        
        for metric in ['flip_frequency', 'performance']:
            if metric in baseline and metric in current:
                b = baseline[metric]
                c = current[metric]
                deltas[metric] = {
                    'baseline': b,
                    'current': c,
                    'delta_pct': (
                        (c['avg'] - b['avg']) / b['avg'] * 100
                    ) if isinstance(c, dict) and 'avg' in c else 0
                }
        
        return deltas
```

---

### 10. 本章总结与实践

#### 10.1 核心知识点回顾

| 工具 | 用途 | 关键命令/API | 适用场景 |
|------|------|-------------|---------|
| modetest | 通用 KMS 功能验证 | `-c` 列举连接器、`-s` 模式设置、`-v` 翻转、`-a` Atomic | 日常开发调试、回归测试 |
| kmstest | 多输出模式测试 | `-r` 分辨率测试、`-p` 性能测试、`-o` 单输出 | 多显示器兼容性验证 |
| kms-steal | Master 权限竞争 | multi-thread set/drop master | 多进程场景验证 |
| kms-lease | DRM 租约管理 | `drmModeCreateLease`、`drmModeListLessees` | VR/AR 设备测试 |
| 自研测试框架 | 自动化 CI 测试 | Python 封装、多线程执行、HTML 报告 | 持续集成、质量门禁 |

#### 10.2 核心技能掌握

完成本章学习后，应掌握以下能力：

1. **工具使用能力**
   - 熟练使用 modetest 完成日常 KMS 调试
   - 理解 kmstest 的测试模式生成原理
   - 掌握 kms-steal/kms-lease 的竞争测试方法

2. **源代码分析能力**
   - 理解 modetest 的源码架构设计
   - 掌握 Atomic API 的完整调用流程
   - 熟悉 dumb buffer 的创建、映射和使用

3. **测试脚本开发能力**
   - 使用 Shell 编写自动化测试套件
   - 使用 C 语言开发自定义测试框架
   - 使用 Python 构建完整的 CI 测试系统

4. **问题定位能力**
   - 通过 modetest 输出快速定位 KMS 问题
   - 分析页面翻转失败的根本原因
   - 识别 Atomic 校验失败的属性冲突

#### 10.3 常见问题排查指南

```bash
# Q1: modetest 无法打开 DRM 设备
# 原因: 权限不足或 VT 占用
sudo chmod 666 /dev/dri/card0
# 或者切换到文本终端 Ctrl+Alt+F2

# Q2: Atomic 提交失败
# 原因: 属性值冲突或不支持
modetest -M amdgpu -a -s 43@0:1920x1080 \
    --atomic-test-only  # 先验证

# Q3: 页面翻转无事件
# 原因: 未设置 CLIENT_CAP_ATOMIC
cat /sys/module/drm/parameters/debug
# 应包含 0x06 (atomic + vblank)

# Q4: kmstest 输出不正确
# 原因: DRM_MODE_CONNECTOR_Unknown
modetest -c | grep connector  # 检查类型
# 某些 DP 转 HDMI 适配器可能识别异常

# Q5: kms-steal 死锁
# 原因: 未释放 Master 权限
# 检查 /sys/kernel/debug/dri/0/name
# 确保退出前调用 drmDropMaster
```

#### 10.4 实践任务清单

| 任务 | 难度 | 预期用时 | 验证方法 |
|------|------|---------|---------|
| 编译 modetest 并枚举所有显示资源 | ★☆☆☆☆ | 30min | 输出包含 CRTC/Encoder/Connector/Plane |
| 使用 Atomic API 设置 4K 分辨率 | ★★☆☆☆ | 1h | modetest -c 显示新分辨率生效 |
| 编写页面翻转测试，统计 FPS | ★★★☆☆ | 2h | 输出包含翻转频率统计数据 |
| 完成 kmstest 所有 4 种测试模式 | ★★★☆☆ | 1h | 每种模式显示正确图案 |
| 实现 kms-steal 多进程竞争测试 | ★★★★☆ | 3h | 验证 drmSetMaster 返回 EBUSY |
| 构建 Python 自动化测试框架 | ★★★★★ | 4h | 生成 HTML 报告，通过率 ≥ 90% |
| 完成 24 小时压力测试 | ★★★☆☆ | 24h | 错误率 < 0.01% |

#### 10.5 Day 365 预告

**[考核] KMS 用户空间接口综合实践**

明天的考核将综合运用本章和之前学到的知识，完成以下任务：

1. **理论考核**：KMS 核心概念、API 调用流程、Atomic 状态验证
2. **代码分析**：分析提供的一段 DRM 驱动代码，找出问题
3. **实践编程**：编写完整的 KMS 测试程序，涵盖模式设置、页面翻转、多输出管理
4. **故障排查**：根据提供的异常 log，定位并描述问题根因

> **准备建议**：确保掌握 modetest 源码中的 Atomic 提交流程、dumb buffer 管理、事件处理循环三个核心模块。

---

*本章完，继续下一章：Day 365 - [考核] KMS 用户空间接口综合实践*