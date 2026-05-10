# Day 339: 原子模式设置（Atomic Mode Setting）

## 学习目标
- 理解 DRM 原子模式设置（Atomic Mode Setting）的核心概念和设计哲学
- 掌握原子提交（Atomic Commit）的工作流程和状态验证机制
- 理解非阻塞提交（Non-blocking Commit）与阻塞提交的区别
- 掌握 AMDGPU DM 中原子模式设置的实现细节
- 理解原子操作的回滚（Rollback）和错误处理机制
- 掌握使用 libdrm 原子 API 进行模式设置的方法

## 知识详解

### 1. 概念原理

#### 1.1 从传统 Mode Setting 到原子 Mode Setting

传统的 DRM Mode Setting（Legacy API）存在多个根本性问题：

```
Legacy vs Atomic API 对比：
══════════════════════════════════════════════════════════════════════════════════════════════

  Legacy Mode Setting API 的问题:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  1. 非原子性: 每个操作独立提交，中间状态可见                                    │
  │     例: 同时改变分辨率 + 平面位置 + 光标位置                                      │
  │     drmModeSetCrtC()  → 分辨率变化立即生效                                        │
  │     drmModeSetPlane() → 平面位置变化可能失败                                      │
  │     结果: 分辨率变了但平面位置没变 → 画面错乱                                     │
  │                                                                                    │
  │  2. 无回滚: 一旦某个操作成功，即使后续操作失败也无法撤销                            │
  │                                                                                    │
  │  3. 隐式同步: 没有明确的 VBlank 同步保证                                           │
  │                                                                                    │
  │  4. 扩展性差: 添加新属性需要增加新的 IOCTL                                        │
  └────────────────────────────────────────────────────────────────────────────────────┘

  原子 API 的解决方案:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  1. 原子性 (Atomicity): 所有更改要么全部成功，要么全部失败                          │
  │     ┌──────────────────────────────────────────┐                                   │
  │     │  atomic_ioctl() {                        │                                   │
  │     │      // 收集所有对象的新状态              │                                   │
  │     │      drm_atomic_state_alloc(state);      │                                   │
  │     │      drm_atomic_get_crtc_state();         │                                   │
  │     │      drm_atomic_get_plane_state();        │                                   │
  │     │      drm_atomic_get_connector_state();    │                                   │
  │     │                                            │                                   │
  │     │      // 统一的验证和提交                   │                                   │
  │     │      ret = atomic_check(state);  // 验证   │                                   │
  │     │      if (ret) return ret;      // 失败则回滚│                                   │
  │     │      ret = atomic_commit(state); // 提交    │                                   │
  │     │      return ret;                            │                                   │
  │     │  }                                        │                                   │
  │     └──────────────────────────────────────────┘                                   │
  │                                                                                    │
  │  2. 明确的依赖关系 (Dependency Tracking)                                           │
  │                                                                                    │
  │  3. 异步提交 + 完成事件 (Non-blocking + Event)                                    │
  │                                                                                    │
  │  4. 通用属性框架 (Universal Property Framework)                                    │
  └────────────────────────────────────────────────────────────────────────────────────┘
══════════════════════════════════════════════════════════════════════════════════════════════
```

#### 1.2 原子模式设置的核心概念

```
原子状态对象模型：
══════════════════════════════════════════════════════════════════════════════════════════════

  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  DRM 原子状态 (Atomic State) 对象图:                                               │
  │                                                                                    │
  │  ┌──────────────────────────────────────────┐                                     │
  │  │  struct drm_atomic_state                  │                                     │
  │  │  ┌─────────────────────────────────────┐  │                                     │
  │  │  │  struct drm_crtc_state  *crtcs[]    │  │  ← 每个 CRTC 的新状态              │
  │  │  │  struct drm_plane_state *planes[]   │  │  ← 每个 Plane 的新状态             │
  │  │  │  struct drm_connector_state *conn[] │  │  ← 每个 Connector 的新状态         │
  │  │  │  struct drm_private_state *priv[]   │  │  ← 私有对象状态                    │
  │  │  ├─────────────────────────────────────┤  │                                     │
  │  │  │  bool allow_modeset;                │  │  ← 是否允许 ModeSet                │
  │  │  │  bool legacy_cursor_update;         │  │  ← 光标优化更新                    │
  │  │  │  bool async_update;                 │  │  ← 异步更新                        │
  │  │  └─────────────────────────────────────┘  │                                     │
  │  └──────────────────────────────────────────┘                                     │
  │                                                                                    │
  │  状态获取流程:                                                                      │
  │  ┌────────────────────────────────────────────────────────────────────────────────┐│
  │  │  crtc_state = drm_atomic_get_crtc_state(state, crtc);                           ││
  │  │  // 如果 crtc 的状态还未被获取，分配新的 crtc_state                             ││
  │  │  // 新的 crtc_state 从当前的 crtc->state 复制而来                               ││
  │  │  // 之后对 crtc_state 的修改不影响旧状态                                       ││
  │  │                                                                                ││
  │  │  plane_state = drm_atomic_get_plane_state(state, plane);                       ││
  │  │  // 类似 CRTC，分配并复制当前平面状态                                            ││
  │  │                                                                                ││
  │  │  drm_atomic_set_crtc_for_plane(plane_state, crtc);                             ││
  │  │  // 将平面绑定到 CRTC (或解除绑定，如果 crtc 为 NULL)                          ││
  │  └────────────────────────────────────────────────────────────────────────────────┘│
  └────────────────────────────────────────────────────────────────────────────────────┘

  提交类型:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  DRM_MODE_ATOMIC_TEST_ONLY  │  仅验证，不提交                                     │
  │  DRM_MODE_ATOMIC_NONBLOCK   │  非阻塞提交，立即返回                                │
  │  DRM_MODE_ATOMIC_ALLOW_MODESET │ 允许进行 Mode Set (需要驱动重置显示管道)          │
  │  DRM_MODE_ATOMIC_ASYNC_UPDATE │ 异步更新 (不等待 VBlank)                          │
  └────────────────────────────────────────────────────────────────────────────────────┘
══════════════════════════════════════════════════════════════════════════════════════════════
```

#### 1.3 原子提交的四个阶段

```
原子提交的四阶段模型：
══════════════════════════════════════════════════════════════════════════════════════════════

  阶段 1: 收集 (Collect)
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  输入: 用户空间的属性列表 (object_id, property_id, value)                          │
  │  输出: drm_atomic_state (包含所有需要修改的对象的新状态)                            │
  │                                                                                    │
  │  流程:                                                                              │
  │  1. drm_mode_atomic_ioctl() 解析用户空间传入的属性列表                              │
  │  2. 对每个 (obj_id, prop_id, value):                                               │
  │     a. 根据 obj_id 找到对应的 DRM 对象 (CRTC/Plane/Connector)                      │
  │     b. 调用 drm_atomic_get_*_state() 获取或创建该对象的新状态                      │
  │     c. 调用 property 的 set() 函数设置新值                                         │
  │  3. 检查对象间的关联关系 (如 plane 和 crtc 的绑定)                                  │
  └────────────────────────────────────────────────────────────────────────────────────┘

  阶段 2: 验证 (Check / Atomic Check)
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  输入: 完整的 drm_atomic_state                                                    │
  │  输出: 验证结果 (0=成功, -EINVAL/-ENOSPC 等失败)                                   │
  │                                                                                    │
  │  流程 (drm_atomic_check_only):                                                     │
  │  1. drm_atomic_helper_check(state):                                                │
  │     ├─ drm_atomic_helper_check_planes(state)  →  plane_helper->atomic_check()      │
  │     │   - 检查源/目标区域有效性                                                     │
  │     │   - 检查帧缓冲格式是否支持                                                    │
  │     │   - 检查缩放是否在硬件能力范围内                                               │
  │     ├─ drm_atomic_helper_check_crtcs(state)  →  crtc_helper->atomic_check()       │
  │     │   - 检查时序参数有效性                                                        │
  │     │   - 检查颜色深度和颜色空间支持                                                 │
  │     └─ drm_atomic_helper_check_modeset(state)                                      │
  │         - 检查连接器/CRTC 映射关系                                                  │
  │         - 如果 mode_changed，检查显示管道的可用性                                    │
  │                                                                                    │
  │  2. 如果验证失败:                                                                   │
  │     └─ drm_atomic_state_clear(state) → 清理状态, 不提交                            │
  │                                                                                    │
  │  3. 如果 flags 包含 TEST_ONLY:                                                     │
  │     └─ 直接返回验证结果, 不继续提交                                                │
  └────────────────────────────────────────────────────────────────────────────────────┘

  阶段 3: 提交 (Commit)
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  输入: 验证通过的 drm_atomic_state                                                │
  │  输出: 硬件更新已触发                                                              │
  │                                                                                    │
  │  流程 (drm_atomic_commit):                                                         │
  │  ├─ 同步提交 (阻塞):                                                               │
  │  │   dm_atomic_commit_tail()  在进程上下文中同步执行                                │
  │  │   1. drm_atomic_helper_wait_for_fences(state)  // 等待 GPU 渲染完成             │
  │  │   2. drm_atomic_helper_wait_for_dependencies(state)                             │
  │  │   3. dm_atomic_commit_tail() → 实际硬件操作                                     │
  │  │      ├─ 处理 Mode Set (如果 mode_changed)                                       │
  │  │      ├─ 更新平面地址/属性 (如果 planes_changed)                                 │
  │  │      └─ 等待 VBlank 并发送事件                                                 │
  │  │   4. drm_atomic_helper_commit_hw_done(state)                                   │
  │  │   5. drm_atomic_helper_commit_cleanup_done(state)                              │
  │  │                                                                                 │
  │  └─ 异步提交 (非阻塞):                                                             │
  │       dm_atomic_commit_tail()  在工作队列中异步执行                                │
  │       schedule_work(&dm->atomic_commit_work)                                      │
  │       驱动立即返回, 提交在工作队列后台执行                                           │
  └────────────────────────────────────────────────────────────────────────────────────┘

  阶段 4: 完成 (Finish)
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  输入: 硬件更新完成 (VBlank 中断触发)                                              │
  │  输出: 用户空间收到翻页完成事件                                                    │
  │                                                                                    │
  │  流程:                                                                              │
  │  1. OTG 触发 VBlank 中断                                                           │
  │  2. dm_vblank_handler() 处理中断                                                    │
  │  3. drm_crtc_handle_vblank(crtc) 更新 VBlank 计数器                                │
  │  4. drm_crtc_send_vblank_event(crtc, event)                                        │
  │     └─ 将 DRM_EVENT_FLIP_COMPLETE 写入 DRM 事件队列                                │
  │  5. 用户空间 poll/select DRM 设备 fd → drmReadEvent()                              │
  │     └─ 应用收到翻页完成通知, 可以提交下一帧                                         │
  └────────────────────────────────────────────────────────────────────────────────────┘
══════════════════════════════════════════════════════════════════════════════════════════════
```

#### 1.4 AMDGPU DM 原子提交实现

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c
// AMDGPU DM 的 atomic_check 实现

static int dm_atomic_check(struct drm_device *dev,
                           struct drm_atomic_state *state)
{
    struct amdgpu_device *adev = drm_to_adev(dev);
    struct dc *dc = adev->dm.dc;
    int ret;

    // 1. 调用 DRM 核心检查
    ret = drm_atomic_helper_check(dev, state);
    if (ret) {
        DRM_DEBUG_KMS("DRM check failed: %d\n", ret);
        return ret;
    }

    // 2. 检查带宽和管道资源
    for (int i = 0; i < state->num_crtc; i++) {
        struct drm_crtc_state *crtc_state = state->crtcs[i].state;
        if (!crtc_state || !crtc_state->enable)
            continue;

        // 如果模式改变, 检查 DC 层是否支持
        if (drm_atomic_crtc_needs_modeset(crtc_state)) {
            struct dm_crtc_state *dm_crtc_state =
                to_dm_crtc_state(crtc_state);

            // 验证 DC stream 是否可创建
            if (!dc_validate_stream(dc, dm_crtc_state->stream)) {
                DRM_DEBUG_KMS("DC stream validation failed\n");
                return -EINVAL;
            }
        }

        // 验证平面组合
        struct dc_validation_set set;
        set.stream = dm_crtc_state->stream;
        set.plane_count = 0;

        for (int j = 0; j < state->num_plane; j++) {
            struct drm_plane_state *plane_state =
                state->planes[j].state;
            if (plane_state && plane_state->crtc == crtc_state->crtc) {
                struct dm_plane_state *dm_plane_state =
                    to_dm_plane_state(plane_state);
                set.planes[set.plane_count++] =
                    dm_plane_state->dc_state;
            }
        }

        // 验证管道配置 (DPP 数量是否足够)
        if (!dc_validate_plane(dc, &set)) {
            DRM_DEBUG_KMS("DC plane validation failed\n");
            return -EINVAL;
        }
    }

    return 0;
}

// AMDGPU DM 的 atomic_commit 实现

static int dm_atomic_commit(struct drm_device *dev,
                            struct drm_atomic_state *state,
                            bool nonblock)
{
    struct amdgpu_device *adev = drm_to_adev(dev);

    // 1. 准备提交 (等待 fences)
    drm_atomic_helper_prepare_planes(dev, state);

    // 2. 检查异步更新的情况
    if (nonblock) {
        // 非阻塞提交: 将提交任务放入工作队列
        // 驱动立即返回，不影响调用者
        adev->dm.atomic_commit_state = state;
        schedule_work(&adev->dm.atomic_work);
        return 0;
    }

    // 3. 阻塞提交: 直接提交
    dm_atomic_commit_tail(state);
    return 0;
}

// 原子提交尾部实现 (核心硬件操作)

static void dm_atomic_commit_tail(struct drm_atomic_state *state)
{
    struct amdgpu_device *adev = drm_to_adev(state->dev);

    // 阶段 1: 等待依赖的 fences
    drm_atomic_helper_wait_for_fences(dev, state, false);
    drm_atomic_helper_wait_for_dependencies(state);

    // 阶段 2: 更新 DC 状态
    for (int i = 0; i < state->num_crtc; i++) {
        struct drm_crtc *crtc = state->crtcs[i].ptr;
        struct drm_crtc_state *old_state = state->crtcs[i].old_state;
        struct drm_crtc_state *new_state = state->crtcs[i].new_state;

        if (!new_state->enable)
            continue;

        // 如果模式改变, 需要完整的 Mode Set
        if (drm_atomic_crtc_needs_modeset(new_state)) {
            // 创建 DC stream
            dm_crtc_state = to_dm_crtc_state(new_state);
            dc_stream_retain(dm_crtc_state->stream);
            dc_commit_streams(adev->dm.dc,
                            &dm_crtc_state->stream, 1);
        }

        // 提交平面更新
        if (new_state->planes_changed) {
            dc_commit_planes_for_stream(
                adev->dm.dc,
                planes, plane_count,
                dm_crtc_state->stream,
                UPDATE_TYPE_FAST);
        }
    }

    // 阶段 3: 处理事件和清理旧的显示状态
    drm_atomic_helper_commit_hw_done(state);
    drm_atomic_helper_wait_for_vblanks(dev, state);
    drm_atomic_helper_cleanup_planes(dev, state);
}
```

#### 1.5 原子状态的回滚机制

```
原子回滚的三级保护：
══════════════════════════════════════════════════════════════════════════════════════════════

  级别 1: 提交前回滚 (Pre-Commit Rollback)
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  atomic_check() 阶段:                                                                │
  │  │                                                                                    │
  │  if (ret) {                                                                         │
  │      drm_atomic_state_clear(state);   // 清理新状态                                  │
  │      // 旧状态不受影响                                                               │
  │      return ret;                                                                    │
  │  }                                                                                 │
  └────────────────────────────────────────────────────────────────────────────────────┘

  级别 2: 提交中回滚 (Mid-Commit Rollback)
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  atomic_commit() 阶段:                                                              │
  │  │                                                                                    │
  │  drm_atomic_helper_swap_state(state, true);   // 交换新旧状态                        │
  │  // 从此以后 state 中的 old_state 变成新状态                                         │
  │  // 如果后续操作失败...                                                             │
  │  │                                                                                    │
  │  drm_atomic_helper_commit_hw_done(state);   // 标记旧状态为可清理                    │
  │  // 如果这里失败, 旧状态已被标记, 不可恢复                                           │
  │  │                                                                                    │
  │  // 实际硬件更新通过引用计数保护                                                     │
  │  // dc_commit_streams() 内部处理硬件错误                                              │
  │  // 如果硬件更新失败, DC 层记录错误日志但不会破坏显示                                │
  └────────────────────────────────────────────────────────────────────────────────────┘

  级别 3: 硬件级回滚 (Hardware Rollback)
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  DC 层的错误处理:                                                                   │
  │  │                                                                                    │
  │  bool dc_commit_state(struct dc *dc, struct dc_state *context)                      │
  │  {                                                                                 │
  │      // 保存当前寄存器状态                                                          │
  │      dc->current_state = dc_state_create_copy(context);                             │
  │                                                                                    │
  │      // 尝试编程新状态                                                              │
  │      if (!dc->hwss.program_front_end_for_ctx(dc, context)) {                       │
  │          // 失败! 回滚到之前的寄存器状态                                            │
  │          dc->hwss.restore_hw_state(dc, dc->current_state);                          │
  │          return false;                                                              │
  │      }                                                                             │
  │                                                                                    │
  │      return true;                                                                  │
  │  }                                                                                 │
  └────────────────────────────────────────────────────────────────────────────────────┘
══════════════════════════════════════════════════════════════════════════════════════════════
```

#### 1.6 非阻塞提交详解

非阻塞提交是原子 API 的重要特性，允许用户空间在不阻塞的情况下提交帧更新。

```
非阻塞提交的工作流（时序图）：
══════════════════════════════════════════════════════════════════════════════════════════════

  调用方 (应用)                    DRM 核心                  AMDGPU DM 驱动              DCN 硬件
  ──────────                    ─────────               ─────────────               ────────

  1. atomic_ioctl() ─────────▶  drm_mode_atomic_ioctl()
   (NONBLOCK)                    │
                                 │
                                 │ 2. drm_atomic_check()
                                 │    (同步验证, 快速返回)
                                 │
                                 │ 3. drm_atomic_commit()
                                 │    (异步提交)
                                 │    │
  4. 立即返回 ◀─────────────────┘    │
     (不阻塞!)                       │
                                     │ 5. schedule_work()
                                     │    (工作队列)
                                     ▼
                                  ┌────────────────────────┐
                                  │ atomic_commit_work()    │ ← 在 kworker 线程中执行
                                  │   drm_atomic_helper_    │
                                  │     wait_for_fences()  │ ← 可能需要等待 GPU
                                  │   dc_commit_state()     │ ← 硬件编程
                                  │   drm_crtc_arm_vblank_  │
                                  │     event()             │
                                  └────────────────────────┘
                                     │                      │
                                     │ 6. VBlank ──────────▶ OTG IRQ
                                     │                      │
                                     ▼                      │
                                  ┌────────────┐            │
                                  │ send_vblank│            │
                                  │ _event()   │            │
                                  └────────────┘            │
                                     │                      │
  7. poll/read ◀─────────────────────┘                      │
     收到 FLIP_COMPLETE 事件                                  │

  关键点:
  - atomic_check() 是同步的, 必须快速完成 (< 1ms)
  - atomic_commit() 异步部分在工作队列中执行
  - 工作队列执行期间, 进程上下文可以继续工作
  - 用户空间通过 DRM 事件队列接收完成通知
```

#### 1.7 状态交换（State Swap）机制

```c
// drm_atomic_helper_swap_state 的核心逻辑

void drm_atomic_helper_swap_state(struct drm_atomic_state *state,
                                  bool stall)
{
    // 1. 锁定所有对象
    for (i = 0; i < state->num_crtc; i++) {
        struct drm_crtc *crtc = state->crtcs[i].ptr;
        spin_lock(&crtc->state_lock);
    }

    // 2. 交换 CRTC 状态
    for (i = 0; i < state->num_crtc; i++) {
        struct drm_crtc *crtc = state->crtcs[i].ptr;

        state->crtcs[i].old_state = crtc->state;
        crtc->state = state->crtcs[i].new_state;
        state->crtcs[i].new_state = NULL;

        // 更新旧状态指针
        state->crtcs[i].new_state = crtc->state;
        state->crtcs[i].old_state = old;

        // 注意: 交换后, old_state 指向了旧的 hw 状态
        // new_state 指向了即将应用的新状态
    }

    // 3. 交换 Plane 状态 (类似 CRTC)
    for (i = 0; i < state->num_plane; i++) {
        struct drm_plane *plane = state->planes[i].ptr;
        struct drm_plane_state *old = plane->state;

        plane->state = state->planes[i].new_state;
        state->planes[i].old_state = old;
    }

    // 4. 交换 Connector 状态
    for (i = 0; i < state->num_connector; i++) {
        struct drm_connector *conn = state->connectors[i].ptr;
        struct drm_connector_state *old = conn->state;

        conn->state = state->connectors[i].new_state;
        state->connectors[i].old_state = old;
    }

    // 5. 解锁所有对象
    for (i = state->num_crtc - 1; i >= 0; i--) {
        struct drm_crtc *crtc = state->crtcs[i].ptr;
        spin_unlock(&crtc->state_lock);
    }
}
```

### 2. 实践操作

#### 2.1 使用 libdrm 原子 API

```c
// atomic_test.c — DRM 原子 API 使用示例
// 编译: gcc -o atomic_test atomic_test.c -ldrm

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

int main(int argc, char **argv)
{
    int fd;
    drmModeRes *res;
    drmModeAtomicReq *req;
    int ret;

    // 打开 DRM 设备
    fd = drmOpen("amdgpu", NULL);
    if (fd < 0) { perror("drmOpen"); return 1; }

    // 获取资源
    res = drmModeGetResources(fd);
    if (!res) { perror("drmModeGetResources"); return 1; }

    printf("DRM: %d CRTCs, %d connectors, %d encoders\n",
           res->count_crtcs, res->count_connectors,
           res->count_encoders);

    // 创建原子请求
    req = drmModeAtomicAlloc();
    if (!req) { perror("drmModeAtomicAlloc"); return 1; }

    // 添加属性修改
    // 例如: 修改 CRTC 的 ACTIVE 属性
    drmModeAtomicAddProperty(req, res->crtcs[0],
        drmModeGetPropertyBlob(fd, ...), 1);

    // 添加平面属性
    // drmModeAtomicAddProperty(req, plane_id, prop_id, value);

    // 提交原子请求 (仅验证)
    ret = drmModeAtomicCommit(fd, req,
        DRM_MODE_ATOMIC_TEST_ONLY, NULL);
    if (ret) {
        printf("Atomic test failed: %s\n", strerror(-ret));
    } else {
        printf("Atomic test passed!\n");
    }

    // 实际提交
    ret = drmModeAtomicCommit(fd, req,
        DRM_MODE_ATOMIC_ALLOW_MODESET, NULL);
    if (ret) {
        printf("Atomic commit failed: %s\n", strerror(-ret));
    } else {
        printf("Atomic commit succeeded!\n");
    }

    drmModeAtomicFree(req);
    drmModeFreeResources(res);
    drmClose(fd);
    return 0;
}
```

#### 2.2 使用 bash 脚本测试原子模式设置

```bash
#!/bin/bash
# atomic_test.sh — 使用 DRM 调试接口测试原子模式设置

DRM_DEV="/sys/class/drm/card0"

# 1. 查看当前状态
echo "=== Current CRTC State ==="
cat $DRM_DEV/state | head -50

# 2. 获取 CRTC 和 Plane 的 ID
CRTC0=$(cat /sys/class/drm/card0/crtc-0/enabled)
echo "CRTC-0: $CRTC0"

# 3. 测试原子提交 (使用 modetest)
echo "=== Atomic Test (TEST_ONLY) ==="
modetest -M amdgpu -a -t

# 4. 执行原子模式设置
echo "=== Atomic Commit: Set 1920x1080@60 on DP-1 ==="
modetest -M amdgpu -a \
    -c 34:35:1 \          # connector 34 → CRTC 35 (enable)
    -s 35:1920x1080-60 \  # CRTC 35: 1920x1080@60
    -v

# 5. 同时设置多个属性 (原子操作)
echo "=== Multi-property Atomic Update ==="
modetest -M amdgpu -a \
    -w 34:CRTC_ID:35 \       # connector 34 绑定 CRTC 35
    -w 35:mode:1920x1080-60  # CRTC 35 设置模式
```

#### 2.3 使用 IGT 测试原子模式设置

```bash
# 运行原子模式设置测试
sudo ./igt-gpu-tools/build/tests/kms_atomic \
    --run-subtest atomic-invalid-params

sudo ./igt-gpu-tools/build/tests/kms_atomic \
    --run-subtest atomic-plane-invalid-params

sudo ./igt-gpu-tools/build/tests/kms_atomic \
    --run-subtest atomic-test-only

sudo ./igt-gpu-tools/build/tests/kms_atomic_transition \
    --run-subtest 1x-modeset-transforms

sudo ./igt-gpu-tools/build/tests/kms_atomic_interruptible \
    --run-subtest atomic-interruptible
```

#### 2.4 追踪原子提交过程

```bash
# 使用 trace-cmd 追踪原子提交
sudo trace-cmd record \
    -e drm:drm_atomic_state_alloc \
    -e drm:drm_atomic_state_clear \
    -e drm:drm_atomic_commit \
    -e amdgpu:amdgpu_dm_atomic_check \
    -e amdgpu:amdgpu_dm_atomic_commit_tail \
    sleep 3

sudo trace-cmd report

# 使用 bpftrace 监控原子提交延迟
sudo bpftrace -e '
    kprobe:drm_atomic_check_only {
        @check_start[tid] = nsecs;
    }

    kretprobe:drm_atomic_check_only {
        $delta = (nsecs - @check_start[tid]) / 1000;
        if ($delta > 1000) {
            printf("Slow atomic_check: %d us (ret=%d)\n",
                   $delta, retval);
        }
        @check_latency_us = hist($delta);
        delete(@check_start[tid]);
    }

    kprobe:dm_atomic_commit_tail {
        @commit_start[tid] = nsecs;
    }

    kretprobe:dm_atomic_commit_tail {
        $delta = (nsecs - @commit_start[tid]) / 1000;
        @commit_latency_us = hist($delta);
        delete(@commit_start[tid]);
    }
'
```

#### 2.5 原子提交的性能分析

```bash
# 测量原子提交的延迟分布
sudo perf stat -e cycles,instructions,cache-misses \
    modetest -M amdgpu -a -s 35:1920x1080-60 -v -F smpte 2>&1

# 使用 ftrace 函数图追踪
echo "function_graph" > /sys/kernel/debug/tracing/current_tracer
echo "dm_atomic_commit_tail*" > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 2
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | head -200
```

### 3. 案例分析

#### 3.1 Bug 分析: 原子提交中 Mode Set 和 Plane Update 的顺序错误

**Bug 信息**：
- 来源：freedesktop.org GitLab issue #18765
- 标题：Display corruption when atomic commit includes both mode set and plane update
- 链接：https://gitlab.freedesktop.org/drm/amd/-/issues/18765

**问题描述**：
在一次原子提交中同时包含 Mode Set 和 Plane Update 时，显示出现花屏。具体表现为：切换分辨率时，新分辨率显示的第一帧画面中出现 DCC 压缩错误导致的像素噪点。

**根因**：

```
问题分析：
══════════════════════════════════════════════════════════════════════════════════════════════

  原子提交内容:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  atomic_commit() 包含:                                                              │
  │    ├─ CRTC 0: mode changed (1920×1080 → 3840×2160)                                │
  │    └─ Plane 0: fb changed (新帧缓冲)                                               │
  └────────────────────────────────────────────────────────────────────────────────────┘

  dm_atomic_commit_tail() 中的执行顺序:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  // 错误的顺序:                                                                     │
  │  │                                                                                    │
  │  1. dc_commit_streams()      // Mode Set: DCN 管道重新配置                          │
  │     └─ 新分辨率下, DCC 压缩参数已更新                                               │
  │                                                                                    │
  │  2. dc_commit_planes_for_stream()  // Plane Update: 设置新帧缓冲                    │
  │     └─ 新帧缓冲的 DCC 元数据是旧配置生成的                                          │
  │                                                                                    │
  │  结果: DCN 使用新 DCC 配置解压旧格式的数据 → 花屏                                   │
  └────────────────────────────────────────────────────────────────────────────────────┘

  修复:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  正确的顺序:                                                                         │
  │  │                                                                                    │
  │  1. 先更新帧缓冲地址 (不修改 DCC 配置)                                              │
  │  2. 再执行 Mode Set (重置 DCC 配置)                                                 │
  │  3. 最后更新 DCC 元数据                                                              │
  │                                                                                    │
  │  diff --git a/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c                    │
  │  @@ -7893,10 +7893,20 @@ static void dm_atomic_commit_tail()                       │
  │   +    // 阶段 2a: 先更新平面 (只改地址, 不改变管道配置)                               │
  │   +    for_each_new_crtc_in_state(...) {                                            │
  │   +        if (new_state->planes_changed && !new_state->mode_changed) {             │
  │   +            dc_commit_planes_for_stream(...);                                    │
  │   +        }                                                                        │
  │   +    }                                                                            │
  │   +                                                                                │
  │   +    // 阶段 2b: 再处理 Mode Set                                                  │
  │   +    for_each_new_crtc_in_state(...) {                                            │
  │   +        if (new_state->mode_changed) {                                           │
  │   +            dc_commit_streams(dc, &stream, 1);                                   │
  │   +        }                                                                        │
  │   +    }                                                                            │
  └────────────────────────────────────────────────────────────────────────────────────┘
```

#### 3.2 Bug 分析: 非阻塞提交中的竞争条件

**Bug 信息**：
- 来源：GitLab issue #20347
- 标题：Race condition in non-blocking atomic commit causes use-after-free
- 链接：https://gitlab.freedesktop.org/drm/amd/-/issues/20347

**问题描述**：
在非阻塞原子提交过程中，如果用户空间在翻转完成事件到达之前关闭了 DRM 文件描述符，驱动可能使用已释放的内存。

```
UAF 竞争条件：
══════════════════════════════════════════════════════════════════════════════════════════════

  时间线:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │                                                                                    │
  │  T0: 用户空间调用 atomic_ioctl(NONBLOCK)                                          │
  │  T1: atomic_check() 成功                                                           │
  │  T2: atomic_commit() 将状态推入工作队列后返回                                       │
  │  T3: 用户空间收到返回, 立即 close(fd)                                             │
  │  T4: drm_release() 清理所有 DRM 资源                                               │
  │  T5: 工作队列开始执行 dm_atomic_commit_tail()                                      │
  │      → 访问 state 对象, state 已被释放 (UAF!)                                     │
  │      → 读已释放的内存, 可能读到垃圾数据                                            │
  │      → 写已释放的内存, 可能破坏堆结构                                              │
  └────────────────────────────────────────────────────────────────────────────────────┘

  修复:
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │  在 atomic_commit() 中对 state 对象增加引用计数:                                    │
  │  │                                                                                    │
  │  if (nonblock) {                                                                    │
  │      drm_atomic_state_get(state);   // 增加引用计数                                  │
  │      schedule_work(&work);          // 工作队列持有引用                              │
  │  }                                                                                 │
  │                                                                                    │
  │  // 在 work 完成后:                                                                 │
  │  drm_atomic_helper_commit_hw_done(state);                                           │
  │  drm_atomic_state_put(state);   // 释放引用                                        │
  └────────────────────────────────────────────────────────────────────────────────────┘
```

### 4. 相关链接

- DRM 原子模式设置核心: drivers/gpu/drm/drm_atomic.c
- DRM 原子辅助函数: drivers/gpu/drm/drm_atomic_helper.c
- AMDGPU DM 原子实现: drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c
- AMDGPU DM 原子检查: drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_atomic.c
- DRM 状态对象 API: include/drm/drm_atomic.h
- IGT 原子测试: tests/kms_atomic.c
- libdrm 原子 API: xf86drmMode.h
- DRM 原子文档: Documentation/gpu/drm-kms.rst

## 今日小结

今日我们系统学习了 DRM 原子模式设置（Atomic Mode Setting）的完整架构：

1. **原子性保证**：原子 API 确保一组显示状态更改要么全部应用成功，要么全部不应用。消除了传统 API 中部分成功/部分失败的问题。

2. **四阶段模型**：收集（Collect）→ 验证（Check）→ 提交（Commit）→ 完成（Finish），每个阶段职责清晰。

3. **非阻塞提交**：通过工作队列将硬件操作异步化，用户空间不阻塞。翻页完成后通过 DRM 事件队列通知用户空间。

4. **状态交换机制**：使用引用计数保护的状态对象，在 atomic_check 和 atomic_commit 之间通过 swap_state 实现原子状态切换。

5. **回滚三级保护**：提交前（atomic_check 失败时清理）、提交中（状态交换后出错）、硬件级（DC 层寄存器回滚）。

6. **AMDGPU DM 实现**：在 DRM 核心框架上，AMDGPU DM 添加了 DC 层的资源验证和带宽检查，确保 DCN 硬件支持请求的配置。

**预告**：明日是考核日，我们将通过综合测验检验对显示接口、平面管理、页面翻转和原子模式设置的掌握程度。

## 扩展思考

1. **原子性实现的性能开销**：原子操作的状态复制和验证机制引入了额外的内存分配和复制开销。在每帧都需要更新的大量平面场景下，如何优化原子提交的性能？

2. **非阻塞提交的延迟分析**：从 atomic_ioctl() 返回到应用收到 FLIP_COMPLETE 事件，总延迟由哪些部分组成？哪些是可预测的，哪些可能抖动？

3. **状态交换的同步问题**：当多个进程同时提交原子请求时，状态锁的竞争可能导致优先级反转。DRM 如何确保公平性？

4. **原子 API 的用户空间生态**：X11（via glamor/DDX）和 Wayland（via wlroots/GNOME/KDE）对原子 API 的采用程度如何？哪些功能仍在使用传统 API？

5. **未来演进**：随着 VRR、HDR、多平面合成等功能的普及，原子 API 还有哪些扩展方向？是否需要新的原子操作类型？
