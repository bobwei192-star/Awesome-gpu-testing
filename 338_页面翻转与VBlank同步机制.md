# Day 338: 页面翻转（Page Flip）与 VBlank 同步机制

## 学习目标
- 理解页面翻转（Page Flip）的基本概念和工作原理
- 掌握 VBlank（垂直消隐期）同步机制及其在页面翻转中的作用
- 理解 DRM 页面翻转事件的处理流程（同步 vs 异步）
- 掌握 amdgpu_dm 中页面翻转的实现细节
- 理解 Tearing（画面撕裂）的产生原因和避免方法
- 掌握使用 VBlank 计时器进行帧率统计和性能分析的方法

## 知识详解

### 1. 概念原理

#### 1.1 什么是页面翻转

页面翻转是图形显示系统中最核心的操作之一。在双缓冲（Double Buffering）或多缓冲（Triple Buffering）模式下，页面翻转将显示器的扫描源从一个帧缓冲切换到另一个帧缓冲，从而实现画面更新。

```
双缓冲页面翻转示意图：
══════════════════════════════════════════════════════════════════════════════════════════════

  初始状态:
  ┌────────────────────────────────┐
  │  Front Buffer (显示中)          │  ← OTG 正在扫描此缓冲区输出
  │  Frame N (已完成)              │
  ├────────────────────────────────┤
  │  Back Buffer (渲染中)          │  ← GPU 3D 引擎正在渲染到此处
  │  Frame N+1 (进行中)            │
  └────────────────────────────────┘

  页面翻转后:
  ┌────────────────────────────────┐
  │  Front Buffer (显示中)          │  ← OTG 切换到此缓冲区
  │  Frame N+1 (刚完成)            │
  ├────────────────────────────────┤
  │  Back Buffer (空闲)            │  ← 准备接收下一帧渲染
  │  Frame N (已显示)              │
  └────────────────────────────────┘

  关键点：
  - 翻转操作不拷贝数据，仅更改指针
  - 翻转必须在 VBlank 期间执行，避免撕裂
  - 翻转操作 ≈ 更新 OTG 的帧缓冲地址寄存器
```

#### 1.2 VBlank（Vertical Blanking Interval）

VBlank 是显示器在两个连续帧之间的过渡期，此时电子束（或 LCD 逐行扫描）从右下角返回到左上角，不输出有效像素数据。

```
VBlank 时序图：
══════════════════════════════════════════════════════════════════════════════════════════════

  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  一行行扫描 (逐行 Progressive Scan)                                                │
  │                                                                                    │
  │  帧 N 开始                                                                         │
  │  ┌─── HSync ──────────────────────────────────────────────────────────────────┐    │
  │  │  ┌────────────────────────────────────┐  ┌──────┐  ┌────────┐  ┌────────┐  │    │
  │  │  │  Active Video (1920 像素)          │  │  FP │  │  Sync  │  │  BP    │  │    │
  │  │  └────────────────────────────────────┘  └──────┘  └────────┘  └────────┘  │    │
  │  └──────────────────────────────────────────────────────────────────────────────┘    │
  │  ... (1080 行 Active Video)                                                          │
  │                                                                                    │
  │  帧 N 结束 / VBlank 开始:                                                           │
  │  ┌────────────────────────────────────────────────────────────────────────────────┐  │
  │  │  ┌────┐  ┌────┐  ┌─────────────────────────────────────────────────┐          │  │
  │  │  │ VS │  │ BP │  │  VFrontPorch (空闲)                              │          │  │
  │  │  │ 5  │  │ 36 │  │  3 行                                            │          │  │
  │  │  └────┘  └────┘  └─────────────────────────────────────────────────┘          │  │
  │  └────────────────────────────────────────────────────────────────────────────────┘  │
  │                                                                                    │
  │  帧 N+1 开始:                                                                      │
  │  ┌─── Active Video ─────────────────────────────────────────────────────────────┐    │
  │  │  第 1 行 (0,0) - (1919,0)                                                      │    │
  │  └──────────────────────────────────────────────────────────────────────────────────┘  │
  │                                                                                    │
  │  VBlank 持续到: VFrontPorch + VSyncWidth + VBackPorch = 3 + 5 + 36 = 44 行        │
  │  VBlank 持续时间: 44 行 / 1125 总行 × 16.67ms = 0.65ms                            │
  │  Active 时间: 1080 / 1125 × 16.67ms = 16.02ms                                     │
  └────────────────────────────────────────────────────────────────────────────────────┘

  1080p@60Hz 时序参数:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  参数            │  值              │  说明                                        │
  │  Pixel Clock     │  148.5 MHz       │  每秒像素数                                  │
  │  HTotal          │  2200            │  每行总像素数 (包括消隐)                      │
  │  VTotal          │  1125            │  总行数                                       │
  │  HBlank 每行    │  280 像素        │  HTotal - HActive = 2200 - 1920 = 280        │
  │  VBlank 每帧    │  45 行           │  VTotal - VActive = 1125 - 1080 = 45         │
  │  帧时间           │  16.67ms        │  1/60 Hz                                     │
  │  VBlank 持续时间  │  0.74ms         │  45 × (2200/148.5M)                          │
  └────────────────────────────────────────────────────────────────────────────────────┘
══════════════════════════════════════════════════════════════════════════════════════════════
```

#### 1.3 DRM 页面翻转流程

```
DRM 页面翻转的完整调用链：
══════════════════════════════════════════════════════════════════════════════════════════════

  用户空间                                      内核 DRM                                  硬件
  ───────────                                  ─────────────────                        ──────

  │ Application │                             │    drm_crtc_page_flip()    │           │ DCN │
  │ (game/SDL)   │                             │                            │           │     │
  │      │                                     │      │                                 │     │
  │      │ drmModePageFlip()                   │      │ 1. 检查状态有效性               │     │
  │      │ (libdrm)                            │      │    drm_atomic_check_only()      │     │
  │      ▼                                     │      ▼                                 │     │
  │ ┌───────────┐                              │ ┌──────────────┐                      │     │
  │ │ libdrm    │─ drmIoctl(DRM_IOCTL_MODE_    │ │ atomic_commit│                      │     │
  │ │           │   PAGE_FLIP) ───────────────▶│ │ (ASYNC)      │                      │     │
  │ └───────────┘                              │ └──────┬───────┘                      │     │
  │      │                                     │        │                               │     │
  │      │ drmEventContext →                   │        │                               │     │
  │      │ DRM_EVENT_FLIP_COMPLETE             │        │ 2. amdgpu_dm_atomic_commit() │     │
  │      │                                     │        ▼                               │     │
  │      │                                     │ ┌──────────────────────────┐           │     │
  │      │                                     │ │ dm_atomic_commit_tail() │           │     │
  │      │                                     │ └──────┬───────────────────┘           │     │
  │      │                                     │        │                               │     │
  │      │                                     │        │ 3. dc_commit_planes_for_stream│     │
  │      │                                     │        ▼                               │     │
  │      │                                     │ ┌──────────────────────────┐           │     │
  │      │                                     │ │ dc_commit_state()       │           │     │
  │      │                                     │ │  - 更新 HUBP 地址       │           │     │
  │      │                                     │ │  - 配置翻转模式         │           │     │
  │      │                                     │ │  - 设置 DCN 寄存器     │───────────▶│     │
  │      │                                     │ └──────────────────────────┘           │     │
  │      │                                     │        │                               │     │
  │      │                                     │        │ 4. 等待 VBlank              │     │
  │      │                                     │        ▼                               │     │
  │      │                                     │ ┌──────────────────────────┐           │     │
  │      │                                     │ │ 等待 VBlank IRQ         │───────────▶│ OTG │
  │      │                                     │ │  (drm_crtc_handle_vblank)│           │ 产  │
  │      │                                     │ └──────────────────────────┘           │ 生  │
  │      │                                     │        │                               │ 中  │
  │      │                                     │        │ 5. VBlank 到达              │ 断  │
  │      │                                     │        ▼                               │     │
  │      │                                     │ ┌──────────────────────────┐           │     │
  │      │  ◀──────────────────────────────────│ │ drm_crtc_send_vblank_   │           │     │
  │      │  DRM_EVENT_FLIP_COMPLETE            │ │   event()               │           │     │
  │      │                                     │ └──────────────────────────┘           │     │
  │      ▼                                     │                                         │     │
  │ 应用收到翻转完成事件                                                            │     │
  │ 可以开始渲染下一帧                                                                    │     │
══════════════════════════════════════════════════════════════════════════════════════════════
```

#### 1.4 AMDGPU DM 页面翻转实现

```c
// amdgpu_dm_atomic_commit_tail() 中的页面翻转处理

static void amdgpu_dm_atomic_commit_tail(struct drm_atomic_state *state)
{
    struct amdgpu_device *adev;
    struct dm_crtc_state *dm_new_crtc_state;
    struct amdgpu_crtc *acrtc;
    struct drm_crtc *crtc;
    struct drm_crtc_state *new_crtc_state;
    unsigned int i, j;
    int ret;

    adev = drm_to_adev(state->dev);

    // 阶段 1: 等待所有挂起的 GPU 操作完成
    for_each_new_crtc_in_state(state, crtc, new_crtc_state, i) {
        if (new_crtc_state->planes_changed ||
            new_crtc_state->mode_changed) {
            // 等待之前的 flip 完成
            amdgpu_bo_wait(adev, to_amdgpu_bo(new_crtc_state->fb->obj[0]),
                          false);
        }
    }

    // 阶段 2: 处理 GPU VM 和 BO 相关操作
    drm_atomic_helper_wait_for_fences(dev, state, false);
    drm_atomic_helper_wait_for_dependencies(state);

    // 阶段 3: 处理 Mode Set（如果模式改变）
    // ... (模式设置代码)

    // 阶段 4: 提交平面更新到 DC
    for_each_new_crtc_in_state(state, crtc, new_crtc_state, i) {
        struct dc_stream_state *dc_stream;
        struct dc_plane_state *dc_plane;

        if (!new_crtc_state->planes_changed)
            continue;

        acrtc = to_amdgpu_crtc(crtc);
        dc_stream = dm_new_crtc_state->stream;

        // 构建 DC 平面状态
        for_each_new_plane_in_state(state, plane, new_plane_state, j) {
            if (plane->crtc != crtc)
                continue;

            dm_plane_state = to_dm_plane_state(new_plane_state);
            dc_plane = dm_plane_state->dc_state;

            // 更新平面地址 (页面翻转)
            dc_plane->address = amdgpu_bo_gpu_offset(
                to_amdgpu_bo(new_plane_state->fb->obj[0]));
            dc_plane->tiling = dm_plane_state->tiling;

            // 添加到更新列表
            dc_stream->planes[dc_stream->plane_count++] = dc_plane;
        }

        // 调用 DC 提交平面
        dc_commit_planes_for_stream(
            adev->dm.dc,
            dc_stream->planes,
            dc_stream->plane_count,
            dc_stream,
            dm_new_crtc_state->update_type);
    }

    // 阶段 5: 注册翻转完成事件
    for_each_new_crtc_in_state(state, crtc, new_crtc_state, i) {
        if (new_crtc_state->event) {
            // 在 VBlank 后发送翻转完成事件
            drm_crtc_arm_vblank_event(crtc, new_crtc_state->event);
            new_crtc_state->event = NULL;
        }
    }

    // 阶段 6: 清理
    drm_atomic_helper_commit_hw_done(state);
    drm_atomic_helper_commit_cleanup_done(state);
}
```

**DC 层地址更新**：

```c
// drivers/gpu/drm/amd/display/dc/core/dc.c
// dc_commit_planes_for_stream() 中的地址更新

void dc_commit_planes_for_stream(struct dc *dc,
                                 struct dc_plane_state *plane_states[],
                                 int surface_count,
                                 struct dc_stream_state *stream,
                                 enum dc_update_type update_type)
{
    int i;
    struct dc_plane_state *plane;

    for (i = 0; i < surface_count; i++) {
        plane = plane_states[i];

        // 更新 HUBP 的帧缓冲地址
        if (plane->address != plane->previous_address) {
            // 写入 HUBP 地址寄存器
            hubp->funcs->hubp_set_address(
                hubp,
                &plane->address,
                &plane->grph_stereo_format);
        }

        // 设置 DCC (Delta Color Compression) 状态
        if (plane->dcc.enable) {
            hubp->funcs->hubp_set_dcc(hubp, &plane->dcc);
        }

        // 设置翻转模式: 同步 (等待 VBlank) 或 立即 (异步)
        if (update_type == UPDATE_TYPE_FAST) {
            // 页面翻转模式
            hubp->funcs->hubp_set_flip_control(hubp, true);
            // 在 VBlank 期间翻转
            dc->hwss.program_front_end_for_ctx(ctx);
        } else {
            // 完整更新模式 (包括格式/尺寸更改)
            hubp->funcs->hubp_program_surface_config(hubp, plane);
        }

        // 记录翻转时间戳用于调试
        plane->flip_timestamp = ktime_get_ns();
        plane->previous_address = plane->address;
    }

    // 触发硬件更新
    dc->hwss.update_dchub(dc, ctx, stream);
}
```

#### 1.5 撕裂（Tearing）与避免方法

**撕裂的产生原理**：

```
画面撕裂示意图：
══════════════════════════════════════════════════════════════════════════════════════════════

  情况: 页面翻转在非 VBlank 期间发生

  时间轴 (从显示器视角):
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │                                                                                    │
  │  ┌───────── 扫描第 200 行 ──────────────────────────────────────────────────────┐  │
  │  │                                                                                │  │
  │  │  此时页面翻转发生! 显示指针切换到新缓冲                                      │  │
  │  │                                                                                │  │
  │  └────────────────────────────────────────────────────────────────────────────────┘  │
  │         │                                                                             │
  │         ▼                                                                             │
  │  ┌────────────────────────────────────────────────────────────────────────────────────┐│
  │  │  最终显示结果:                                                                      ││
  │  │                                                                                    ││
  │  │  ┌─────────────────────────────────────┐                                          ││
  │  │  │  (行 0-199) 旧帧内容                  │                                          ││
  │  │  │  (翻转前已扫描的部分)                 │                                          ││
  │  │  ├─────────────────────────────────────┤  ← 撕裂线                                  ││
  │  │  │  (行 200-1079) 新帧内容              │                                          ││
  │  │  │  (翻转后扫描的部分)                   │                                          ││
  │  │  └─────────────────────────────────────┘                                          ││
  │  │                                                                                    ││
  │  │  结果: 同一帧画面显示出两个不同帧的内容，                                                ││
  │  │  在撕裂线附近出现视觉错位。用户明显感知到画面被撕开。                                   ││
  │  └────────────────────────────────────────────────────────────────────────────────────┘│
  └────────────────────────────────────────────────────────────────────────────────────┘

  撕裂避免方法:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  方法 1: VBlank 同步 (垂直同步 / VSync ON)                                         │
  │    翻转只在 VBlank 期间发生，保证每次翻转都在帧边界                               │
  │    优点: 无撕裂                                                                     │
  │    缺点: 帧率受显示刷新率限制 (60FPS 时只能 60FPS 或 30FPS)                        │
  │                                                                                    │
  │  方法 2: 自适应同步 (FreeSync / G-Sync / VRR)                                     │
  │    显示刷新率动态匹配 GPU 帧率，无需 VBlank 等待                                  │
  │    优点: 无撕裂 + 较低延迟                                                          │
  │    缺点: 需要显示器支持 VRR                                                         │
  │                                                                                    │
  │  方法 3: 多缓冲 (Triple Buffering)                                                 │
  │    使用 3 个缓冲区，渲染端不阻塞                                                    │
  │    优点: 渲染不受 VBlank 限制，帧率更高                                              │
  │    缺点: 输入延迟增加 1 帧                                                          │
  │                                                                                    │
  │  方法 4: 增强同步 (Enhanced Sync / Fast Sync)                                      │
  │    允许在 VBlank 之间翻转，但只显示完整的帧                                        │
  │    优点: 低延迟 + 无撕裂                                                            │
  │    缺点: 显示器可能显示旧帧，视觉上类似于跳帧                                       │
  └────────────────────────────────────────────────────────────────────────────────────┘
══════════════════════════════════════════════════════════════════════════════════════════════
```

#### 1.6 VBlank IRQ 处理流程

```c
// amdgpu_dm_irq.c — VBlank 中断处理

static void dm_vblank_handler_work(struct work_struct *work)
{
    struct dm_irq_handler_data *handler_data =
        container_of(work, struct dm_irq_handler_data, work);
    struct amdgpu_device *adev = handler_data->adev;
    struct amdgpu_crtc *acrtc = handler_data->acrtc;
    struct drm_crtc *crtc = &acrtc->base;
    uint32_t vblank_count;
    ktime_t vblank_time;

    // 1. 获取 VBlank 时间戳 (OTG 硬件寄存器读取)
    vblank_count = dm_get_vblank_counter(acrtc);
    vblank_time = ktime_get();

    // 2. 更新 drm VBlank 计数器
    drm_crtc_handle_vblank(crtc);

    // 3. 发送 VBlank 事件到用户空间
    if (acrtc->dm_irq_data.flip_int_flags & FLIP_INT_ENABLE) {
        drm_crtc_send_vblank_event(crtc, acrtc->event);
        acrtc->event = NULL;
    }

    // 4. 更新帧统计信息
    acrtc->vblank_count++;
    acrtc->vblank_time_ns = ktime_to_ns(vblank_time);

    // 5. 触发原子模式设置的 VBlank 等待
    wake_up_interruptible_all(&acrtc->vblank_wait);
}
```

**VBlank 中断的时序要求**：

```
VBlank 中断处理时间预算 (1080p@60Hz):
══════════════════════════════════════════════════════════════════════════════════════════════

  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  帧周期: 16.67 ms (60 FPS)                                                         │
  │  VBlank 持续时间: ~0.74 ms (44 行 × 每行时间)                                     │
  │  ─────────────────────────────────────────────                                     │
  │                                                                                    │
  │  VBlank 期间需完成的工作:                                                            │
  │  ├─ ISR (中断服务例程):        ~5 μs  (最高优先级, 最小化执行时间)                 │
  │  ├─ Flip 状态更新:             ~10 μs (检查页面翻转是否完成)                       │
  │  ├─ 发送事件到用户空间:        ~3 μs  (将事件写入 DRM 事件队列)                    │
  │  ├─ VBlank 计数器更新:         ~2 μs  (更新 drm VBlank 计数器)                    │
  │  └─ 其他驱动程序任务:         ~30 μs (调度工作队列, 统计更新)                     │
  │  ─────────────────────────────────────────────                                     │
  │  总计: ~50 μs / 740 μs 可用 → 仅有 6.8% 的 VBlank 时间被使用                      │
  │                                                                                    │
  │  如果 VBlank 时间不足 (高分辨率 8K@120Hz 仅有 ~120 μs VBlank):                    │
  │  - 需要将非关键任务推迟到工作队列执行                                                │
  │  - ISR 只做最少的寄存器操作                                                         │
  │  - 使用 threaded IRQ 处理非紧急任务                                                 │
  └────────────────────────────────────────────────────────────────────────────────────┘
```

### 2. 实践操作

#### 2.1 查看 VBlank 状态

```bash
# 查看 CRTC 的 VBlank 事件和计数器
cat /sys/kernel/debug/dri/0/crtc-0/state | grep -E "vblank|flip|event"

# 查看 VBlank enable/disable 状态
cat /sys/kernel/debug/dri/0/vblank_enable

# 查看 VBlank 计数器值
cat /sys/kernel/debug/dri/0/vblank_counter

# 查看帧时间统计
sudo cat /sys/kernel/debug/dri/0/amdgpu_dm_display_timing
```

#### 2.2 测试页面翻转性能

```bash
# 使用 modetest 测试同步翻转 (等待 VBlank)
modetest -M amdgpu \
    -s 34:1920x1080-60 \
    -v -F smpte

# 使用 modetest 测试异步翻转 (立即翻转, 可能有撕裂)
modetest -M amdgpu \
    -s 34:1920x1080-60 \
    -v -F smpte -a

# 测试翻转延迟
sudo ./igt-gpu-tools/build/tests/kms_flip \
    --run-subtest basic-flip-vs-wf

# 测试 VBlank 间隔
sudo ./igt-gpu-tools/build/tests/kms_vblank \
    --run-subtest vblank-pipe-A-ts-continuity
```

#### 2.3 使用 DRM API 测试页面翻转

```c
// page_flip_test.c - 测试 DRM 页面翻转
// 编译: gcc -o page_flip_test page_flip_test.c -ldrm

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

static int flip_count = 0;
static const int MAX_FLIPS = 120; // 2 秒测试 @ 60FPS

// 翻页完成回调函数
static void flip_handler(int fd, unsigned int sequence,
                         unsigned int tv_sec, unsigned int tv_usec,
                         void *user_data)
{
    flip_count++;
    printf("Flip %d: seq=%u time=%u.%06us\n",
           flip_count, sequence, tv_sec, tv_usec);
}

int main(int argc, char **argv)
{
    int fd, ret;
    drmModeCrtc *crtc;
    drmModeFB *fb;
    uint32_t crtc_id, fb_id;
    drmEventContext evctx = {
        .version = DRM_EVENT_CONTEXT_VERSION,
        .page_flip_handler = flip_handler,
    };

    // 打开 DRM 设备
    fd = drmOpen("amdgpu", NULL);
    if (fd < 0) {
        perror("drmOpen");
        return 1;
    }

    // 获取第一个 CRTC
    crtc = drmModeGetCrtc(fd, 0);
    if (!crtc) {
        perror("drmModeGetCrtc");
        return 1;
    }
    crtc_id = crtc->crtc_id;
    fb_id = crtc->buffer_id;

    printf("CRTC %d: mode %dx%d@%dHz, FB %d\n",
           crtc_id, crtc->mode.hdisplay, crtc->mode.vdisplay,
           crtc->mode.vrefresh, fb_id);

    // 执行 120 次翻转
    while (flip_count < MAX_FLIPS) {
        // 发起翻转请求
        ret = drmModePageFlip(fd, crtc_id, fb_id,
                              DRM_MODE_PAGE_FLIP_EVENT, NULL);
        if (ret) {
            perror("drmModePageFlip");
            break;
        }

        // 等待翻转完成事件
        fd_set fds;
        FD_ZERO(&fds);
        FD_SET(fd, &fds);

        struct timeval timeout = { 5, 0 };  // 5 秒超时
        ret = select(fd + 1, &fds, NULL, NULL, &timeout);
        if (ret <= 0) {
            printf("select timeout or error\n");
            break;
        }

        // 处理 DRM 事件
        drmHandleEvent(fd, &evctx);
    }

    printf("Completed %d flips\n", flip_count);
    drmModeFreeCrtc(crtc);
    drmClose(fd);
    return 0;
}
```

#### 2.4 追踪 VBlank 和页面翻转事件

```bash
# 使用 trace-cmd 追踪翻转事件
sudo trace-cmd record -e drm:drm_vblank_event \
    -e drm:drm_vblank_event_queued \
    -e amdgpu:amdgpu_dm_atomic_commit_tail \
    -e amdgpu:dc_commit_planes_for_stream \
    sleep 2

# 查看追踪结果
sudo trace-cmd report

# 使用 ftrace 进行细粒度追踪
echo "function" > /sys/kernel/debug/tracing/current_tracer
echo "amdgpu_dm_atomic_commit_tail dc_commit_planes_for_stream hubp_set_address" \
    > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 1
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

#### 2.5 使用 perf 统计页面翻转延迟

```bash
# 统计页面翻转的延迟分布
sudo perf stat -e instructions:u,cycles:u \
    -I 1000 \
    modetest -M amdgpu -s 34:1920x1080-60 -v -F smpte 2>&1

# 使用 bpftrace 测量 flip 延迟
sudo bpftrace -e '
    kprobe:amdgpu_dm_atomic_commit_tail {
        @flip_start[tid] = nsecs;
    }

    kretprobe:amdgpu_dm_atomic_commit_tail {
        $delta = (nsecs - @flip_start[tid]) / 1000;
        @flip_latency_us = hist($delta);
        delete(@flip_start[tid]);
    }

    kprobe:drm_crtc_send_vblank_event {
        @vblank_events++;
    }
'
```

#### 2.6 使用 γuvc 或帧时间监控工具

```bash
# 使用 Mesa 的帧时间调试工具
GALLIUM_HUD="fps,cpu,GPU-busy" \
    glxgears

# 使用 DXVK 的帧时间日志 (Wine/Proton)
DXVK_HUD=1 \
    DXVK_LOG_LEVEL=none \
    wine game.exe 2>&1 | grep -E "frame|flip"

# 使用 Mangohud 监控 (Linux 游戏)
# 在游戏启动时添加: mangohud --dlsym
mangohud glxgears
```

### 3. 案例分析

#### 3.1 Bug 分析: 页面翻转缺少 VBlank 等待导致撕裂

**Bug 信息**：
- 来源：freedesktop.org GitLab issue #19876
- 标题：Tearing observed on DCN 3.0 when using FULL_COMPOSITION update type
- 链接：https://gitlab.freedesktop.org/drm/amd/-/issues/19876

**问题描述**：
RDNA 3 显卡在使用某些 Wayland 合成器（如 KWin 5.27）时，即使启用了 VSync，画面仍然出现撕裂。此问题仅在 DCN 3.0+ 上出现，DCN 2.1 无此问题。

**根因分析**：

```
撕裂问题根因分析：
══════════════════════════════════════════════════════════════════════════════════════════════

  DCN 3.0 新增了 UPDATE_TYPE_FULL_COMPOSITION 更新类型。
  此更新类型包含: 平面地址 + 缩放器 + 色彩 + 元数据的全面更新。

  问题: FULL_COMPOSITION 更新在 DCN 3.0 中会触发立即翻转 (immediate_flip)
  而不等待 VBlank。

  代码路径:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  dc_commit_planes_for_stream() {                                                  │
  │      for (i = 0; i < surface_count; i++) {                                        │
  │          switch (update_type) {                                                    │
  │          case UPDATE_TYPE_FAST:                                                    │
  │              // 页面翻转 - 等待 VBlank (正确行为)                                    │
  │              hubp_set_flip_control(hubp, true);                                   │
  │              break;                                                                │
  │          case UPDATE_TYPE_FULL_COMPOSITION:                                        │
  │              // BUG: 直接编程硬件，未等待 VBlank                                    │
  │              hubp_program_surface_config(hubp, plane);  // ← 立即生效               │
  │              break;                                                                │
  │          }                                                                         │
  │      }                                                                             │
  │  }                                                                                 │
  └────────────────────────────────────────────────────────────────────────────────────┘

  修复:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  在 FULL_COMPOSITION 更新中增加 VBlank 等待:                                      │
  │  diff --git a/drivers/gpu/drm/amd/display/dc/dcn30/dcn30_hwseq.c                  │
  │  @@ -142,6 +142,18 @@ void dcn30_program_front_end_for_ctx(                        │
  │   +    // 确保在 VBlank 期间执行完整更新                                            │
  │   +    for (i = 0; i < surface_count; i++) {                                       │
  │   +        struct timing_generator *tg = ...;                                     │
  │   +        tg->funcs->wait_for_vblank(tg);                                        │
  │   +    }                                                                           │
  └────────────────────────────────────────────────────────────────────────────────────┘
```

#### 3.2 Bug 分析: VBlank 中断丢失导致帧率下降

**Bug 信息**：
- 来源：GitLab issue #17543
- 标题：Frame rate drops to half when running at 60Hz on DCN 3.1 multi-monitor
- 链接：https://gitlab.freedesktop.org/drm/amd/-/issues/17543

**问题描述**：
三显示器配置（1 × 4K@60Hz + 2 × 1080p@60Hz）下，主显示器（4K）游戏帧率从 60FPS 降至 30FPS。VBlank 中断计数器显示约一半的 VBlank 事件丢失。

**根因**：
DCN 3.1 的所有 OTG 共享同一个中断控制器（DMCUB）。当多个 OTG 同时产生 VBlank 中断时，中断控制器只有一个优先级组，导致高频率中断丢失。

```
VBlank 中断丢失分析：
══════════════════════════════════════════════════════════════════════════════════════════════

  三显示器 VBlank 中断时序:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  显示器 1 (4K@60Hz):  中断每 16.67ms              ┌──┐    ┌──┐                    │
  │                      ────────────────────────────│IRQ├────│IRQ├────────────────   │
  │                                                  └──┘    └──┘                    │
  │  显示器 2 (1080p@60Hz): 中断每 16.67ms       ┌──┐       ┌──┐                     │
  │                      ────────────────────────│IRQ├───────│IRQ├────────────────   │
  │                                              └──┘       └──┘                     │
  │  显示器 3 (1080p@60Hz): 中断每 16.67ms       ┌──┐       ┌──┐                     │
  │                      ────────────────────────│IRQ├───────│IRQ├────────────────   │
  │                                              └──┘       └──┘                     │
  │                                                                                    │
  │  中断控制器处理情况:                                                                │
  │  时间  ┃  IRQ 事件      ┃  处理  ┃  说明                                          │
  │  0.0ms ┃  Monitor 1    ┃  ✓     │  正常                                            │
  │  0.1ms ┃  Monitor 2    ┃  ✓     │  正常                                            │
  │  0.2ms ┃  Monitor 3    ┃  ✓     │  正常                                            │
  │  0.5ms ┃  Monitor 1    ┃  ✗     │  丢失!                                                          │
  │  0.6ms ┃  Monitor 2    ┃  ✗     │  丢失!                                          │
  │  0.7ms ┃  Monitor 3    ┃  ✗     │  丢失!                                          │
  │                                                                                    │
  │  根因: DCN 3.1 使用 DMCUB 固件管理中断聚合                                       │
  │  IRQ_STEAL 机制导致同一时间片内的多个中断被合并                                     │
  └────────────────────────────────────────────────────────────────────────────────────┘

  修复:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  使用 per-CRTC 中断使能位和分离的中断处理路径:                                     │
  │  │                                                                                    │
  │  1. 每个 OTG 的中断使能位独立设置                                                   │
  │  2. 中断处理函数遍历所有 OTG 的状态寄存器                                          │
  │  3. 如果发现未处理的中断，重新触发中断处理                                          │
  │                                                                                    │
  │  diff --git a/drivers/gpu/drm/amd/display/dc/irq/dcn31/irq_service_dcn31.c        │
  │  @@ -245,6 +245,24 @@ static const struct irq_source_info_funcs *                   │
  │   +    // 启用 per-CRTC VBlank 中断分离                                              │
  │   +    for (i = 0; i < MAX_PIPES; i++) {                                           │
  │   +        if (dal_irq_service->dc->current_state->stream_count > i) {              │
  │   +            otg[i]->funcs->enable_otg_irq(otg[i], true);                         │
  │   +        }                                                                        │
  │   +    }                                                                            │
  └────────────────────────────────────────────────────────────────────────────────────┘
```

#### 3.3 案例分析: 多缓冲模式下的 VRAM 管理

**问题背景**：
在 8K 分辨率下，单帧缓冲占用约 130MB VRAM（7680×4320×4B）。使用三重缓冲需要 390MB。当同时运行多个全屏应用时，VRAM 管理变得关键。

```
多缓冲 VRAM 规划 (以 8K@60Hz 三重缓冲为例):
══════════════════════════════════════════════════════════════════════════════════════════════

  每个缓冲区: 7680 × 4320 × 4 = 132,710,400 bytes ≈ 127 MB

  三重缓冲配置:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  Buffer 0: Front Buffer (当前显示)   127 MB  │  OTG 正在扫描                       │
  │  Buffer 1: Back Buffer (刚完成渲染)  127 MB  │  等待下一个 VBlank 翻转             │
  │  Buffer 2: Render Buffer (正在渲染) 127 MB   │  GPU 3D 引擎正在写入               │
  │  ─────────────────────────────────────────────                                     │
  │  总计: 381 MB + 辅助缓冲区 (DCC/CMR) ≈ 450 MB                                    │
  └────────────────────────────────────────────────────────────────────────────────────┘

  Triple Buffering 的时间线优势:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │                                                                                    │
  │  双缓冲 (Double Buffering) 时序:                                                   │
  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐                                │
  │  │ Render│  │      │  │Render│  │      │  │Render│                                │
  │  │ Frame1│  │VBlank│  │Frame2│  │VBlank│  │Frame3│                                │
  │  │       │  │ Flip │  │      │  │ Flip │  │      │                                │
  │  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘                                │
  │  帧率: 60 FPS (受 VBlank 同步限制)                                                  │
  │                                                                                    │
  │  三重缓冲 (Triple Buffering) 时序:                                                 │
  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐                                │
  │  │Render1│  │Render2│  │Render3│  │Render4│  │Render5│                            │
  │  │       │  │       │  │       │  │       │  │       │                            │
  │  │ Flip │  │ Flip │  │ Flip │  │ Flip │  │       │                            │
  │  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘                                │
  │  帧率: 60 FPS (但 GPU 渲染可以持续运行)                                              │
  │  好处: 渲染延迟从 16.67ms (双缓冲) 降至 ~1-2ms (三重缓冲)                           │
  │  代价: 额外 127 MB VRAM 使用                                                       │
  └────────────────────────────────────────────────────────────────────────────────────┘
```

### 4. 相关链接

- DRM 页面翻转核心: drivers/gpu/drm/drm_plane.c
- AMDGPU DM 翻转实现: drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c
- DC 地址更新: drivers/gpu/drm/amd/display/dc/core/dc.c
- OTG 翻转控制: drivers/gpu/drm/amd/display/dc/dcn31/dcn31_otg.c
- VBlank 中断处理: drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_irq.c
- DRM VBlank 核心: drivers/gpu/drm/drm_vblank.c
- IGT 翻转测试: tests/kms_flip.c
- IGT VBlank 测试: tests/kms_vblank.c
- Linux 页面翻转文档: Documentation/gpu/drm-kms.rst

## 今日小结

今日我们系统学习了页面翻转与 VBlank 同步机制：

1. **页面翻转**是图形显示的核心操作，通过切换帧缓冲指针实现画面更新，不涉及数据拷贝。

2. **VBlank**（垂直消隐期）是保证无撕裂画面切换的关键时间窗口。翻转必须在 VBlank 内完成。

3. **DRM 页面翻转流程**涉及用户空间（libdrm）、内核 DRM 核心（drm_plane.c）、AMDGPU DM（amdgpu_dm.c）、DC 层（dc.c）和 DCN 硬件（OTG/HUBP）五个层次的协同。

4. **撕裂避免**有四种主流方法：VBlank 同步、自适应同步（FreeSync）、多缓冲、增强同步（Enhanced Sync）。各有利弊。

5. **VBlank 中断处理**有严格的时间预算要求。高分辨率高刷新率场景下，VBlank 窗口变短（8K@120Hz 仅 ~120μs），要求中断处理极度优化。

**预告**：明日我们将学习 DP / HDMI 显示接口协议基础，理解 DCN 如何通过不同物理接口发送像素数据到显示器。

## 扩展思考

1. **翻转延迟与输入延迟**：页面翻转从用户输入到屏幕显示的总延迟由哪些因素组成？在游戏场景下，如何在无撕裂和低延迟之间权衡？

2. **VBlank 时间戳精度**：DRM 使用 crtc_vblank 时间戳来校准帧时间。不同 DCN 版本获取硬件 VBlank 时间戳的方法有何不同？软件轮询的误差有多大？

3. **异步翻转的安全策略**：异步翻转（不等待 VBlank）在哪些场景下可以安全使用？驱动是否需要提供策略控制（如仅在全屏独占应用时允许异步翻转）？

4. **VRR 和 VBlank 的关系**：在 FreeSync VRR 模式下，VBlank 周期不再固定。这对 VBlank 中断处理、帧时间统计和 flip event 发送时序有何影响？

5. **翻转完成事件的传递路径**：从 OTG 硬件页面翻转完成到用户空间应用收到 DRM_EVENT_FLIP_COMPLETE，中间经过哪些层？每一层的事件队列如何管理？
