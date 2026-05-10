# 第330天：[考核] KMS 架构与 Mode Setting 流程图

## 学习目标
- 综合运用 Days 326-329 所学知识，独立绘制完整的 KMS 架构与 Mode Setting 流程图
- 深入理解从用户空间到硬件寄存器的全链路数据流和关键函数调用关系
- 掌握 DRM/KMS 核心数据结构的关联关系与生命周期
- 能够通过实际调试工具验证 Mode Setting 流程中的每个环节
- 完成综合测验题，检验对 KMS 架构的掌握程度

## 考核内容

### 第一部分：KMS 整体架构图（40分）

请根据以下要求，绘制完整的 KMS 架构分层图。以下为参考答案：

```
                        KMS (Kernel Mode Setting) 完整架构
                        ═══════════════════════════════════

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                               User Space                                              │
│                                                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │   xrandr     │  │   wayland    │  │   modetest   │  │   custom     │             │
│  │   (X11)      │  │   compositor │  │   (libdrm)   │  │   app(KMS)   │             │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│         │                  │                  │                  │                    │
│         └──────────────────┴──────────────────┴──────────────────┘                    │
│                                    │                                                   │
│                          ┌─────────┴─────────┐                                        │
│                          │     libdrm (libdrm_amdgpu.so)      │                        │
│                          │  drmModeGetResources, drmModeSetCrtc│                        │
│                          │  drmModeAddFB, drmModePageFlip     │                        │
│                          │  drmModeAtomicCommit               │                        │
│                          └─────────┬─────────────────────────┘                        │
│                                    │                                                   │
│                          System Call: ioctl()                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                              Kernel Space                                              │
│                                                                                       │
│  ┌──────────────────────────────────────────────────────────────────────────────┐    │
│  │                         DRM Subsystem (drivers/gpu/drm)                      │    │
│  │                                                                              │    │
│  │  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐     │    │
│  │  │   DRM Core         │  │   DRM KMS Helper   │  │   DRM Atomic      │     │    │
│  │  │                    │  │                    │  │   Helper          │     │    │
│  │  │  drm_mode_setcrtc()│  │  drm_crtc_helper_  │  │  drm_atomic_      │     │    │
│  │  │  drm_mode_addfb()  │  │  set_config()      │  │  helper_set_config│     │    │
│  │  │  drm_mode_pageflip │  │  drm_encoder_      │  │  drm_atomic_      │     │    │
│  │  │  drm_ioctl()       │  │  helper_funcs       │  │  check_commit()   │     │    │
│  │  │  drm_mode_config   │  │  drm_crtc_funcs     │  │  drm_atomic_state │     │    │
│  │  │  drm_framebuffer   │  │  drm_plane_helper   │  │  drm_connector_   │     │    │
│  │  └────────┬───────────┘  └────────┬───────────┘  │  atomic_check     │     │    │
│  │           │                       │               └────────┬──────────┘     │    │
│  │           └───────────────────────┼────────────────────────┘                 │    │
│  │                                   │                                           │    │
│  │  ┌────────────────────────────────┴──────────────────────────────────────┐   │    │
│  │  │                   DRM Device Driver Callbacks                          │   │    │
│  │  │  struct drm_driver ──> amdgpu_driver_funcs                             │   │    │
│  │  │  struct drm_mode_config_funcs ──> amdgpu_mode_funcs                    │   │    │
│  │  │  struct drm_crtc_funcs ──> amdgpu_dm_crtc_funcs                        │   │    │
│  │  │  struct drm_connector_funcs ──> amdgpu_dm_connector_funcs              │   │    │
│  │  │  struct drm_plane_funcs ──> dm_plane_funcs                             │   │    │
│  │  │  struct drm_encoder_funcs ──> amdgpu_dm_encoder_funcs                  │   │    │
│  │  └────────────────────────────────────────────────────────────────────────┘   │    │
│  │                                   │                                           │    │
│  │  ┌────────────────────────────────┴──────────────────────────────────────┐   │    │
│  │  │                   AMDGPU DM (Display Manager)                         │   │    │
│  │  │  amdgpu_dm.c / amdgpu_dm_crtc.c / amdgpu_dm_plane.c                   │   │    │
│  │  │                                                                       │   │    │
│  │  │  dm_crtc_helper_mode_set()    ──  DRM mode → DC timing                │   │    │
│  │  │  amdgpu_dm_atomic_check()     ──  Validate & create DC state          │   │    │
│  │  │  amdgpu_dm_atomic_commit_tail() ──  Commit to DC                      │   │    │
│  │  │  dm_update_planes_state()     ──  Plane state management              │   │    │
│  │  │  dm_crtc_duplicate_state()    ──  State duplication for atomic        │   │    │
│  │  └────────────────────────────────┬──────────────────────────────────────┘   │    │
│  │                                   │                                           │    │
│  │  ┌────────────────────────────────┴──────────────────────────────────────┐   │    │
│  │  │              DC (Display Core)                                        │   │    │
│  │  │  drivers/gpu/drm/amd/display/dc/                                      │   │    │
│  │  │                                                                       │   │    │
│  │  │  dc_commit_streams()          ──  Commit stream(s) to HW              │   │    │
│  │  │  dc_commit_state()            ──  Apply full state to HW              │   │    │
│  │  │  dc_validate_plane()          ──  Validate plane configuration        │   │    │
│  │  │  dc_stream_create()           ──  Create DC stream from timing        │   │    │
│  │  │  resource_build_scaling_params()  ──  Build MPO/Scaling params        │   │    │
│  │  │                                                                       │   │    │
│  │  │  ┌──────────────────────────────────────────────────────────────┐    │   │    │
│  │  │  │            dc->hwss (Hardware Sequencer Function Table)       │    │   │    │
│  │  │  │  dcn20_hwseq.c / dcn30_hwseq.c / dcn31_hwseq.c               │    │   │    │
│  │  │  │                                                                  │    │   │    │
│  │  │  │  .apply_ctx_for_surface()     ──  Program surface to HW         │    │   │    │
│  │  │  │  .program_front_end()         ──  Program DPP/MPC               │    │   │    │
│  │  │  │  .program_timing()            ──  Program OTG timing            │    │   │    │
│  │  │  │  .enable_display_power_gating()  ──  Power gate control         │    │   │    │
│  │  │  └──────────────────────────────────────────────────────────────┘    │   │    │
│  │  └──────────────────────────────────────────────────────────────────┘   │    │
│  │                                   │                                           │    │
│  │  ┌────────────────────────────────┴──────────────────────────────────────┐   │    │
│  │  │                   DCN Hardware (Display Core Next)                     │   │    │
│  │  │                                                                       │   │    │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐             │   │    │
│  │  │  │   OTG    │  │   DPP    │  │   MPC    │  │   OPP    │             │   │    │
│  │  │  │ Timing   │  │ Surface  │  │ Blending │  │ Output   │             │   │    │
│  │  │  │ Generator│  │ Pipeline │  │ & Mix    │  │ Processing│             │   │    │
│  │  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘             │   │    │
│  │  │       │              │              │              │                    │   │    │
│  │  │       └──────────────┴──────────────┴──────────────┘                    │   │    │
│  │  │                              │                                           │   │    │
│  │  │  ┌───────────────────────────┴────────────────────────────────────┐    │   │    │
│  │  │  │      DIO (Display I/O) / HUBP / DCHUB / DCSURF / DMUB          │    │   │    │
│  │  │  │      MMHUBBUB / DCHUBBUB                                       │    │   │    │
│  │  │  └────────────────────────────────────────────────────────────────┘    │   │    │
│  │  └──────────────────────────────────────────────────────────────────┘   │    │
│  └──────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                       │
│  ┌──────────────────────────────────────────────────────────────────────────────┐    │
│  │                   AMDGPU KMS 核心数据结构关系图                               │    │
│  │                                                                              │    │
│  │  ┌──────────────┐       ┌──────────────────┐                                │    │
│  │  │ drm_device    │──────>│ drm_mode_config   │                                │    │
│  │  │ (struct)      │       │ (struct)          │                                │    │
│  │  │  .dev_private─>│       │  .crtc_list       │                                │    │
│  │  │  .mode_config │       │  .connector_list   │                                │    │
│  │  └──────────────┘       │  .encoder_list     │                                │    │
│  │                          │  .plane_list       │                                │    │
│  │                          │  .fb_list          │                                │    │
│  │                          │  .property_list    │                                │    │
│  │                          └────────┬─────────┘                                  │    │
│  │                                   │                                             │    │
│  │            ┌──────────────────────┼──────────────────────┐                      │    │
│  │            │                      │                      │                      │    │
│  │  ┌─────────┴─────────┐  ┌────────┴────────┐  ┌─────────┴─────────┐            │    │
│  │  │   drm_crtc        │  │  drm_connector   │  │   drm_encoder    │            │    │
│  │  │  .primary          │  │  .connector_type  │  │  .encoder_type   │            │    │
│  │  │  .cursor           │  │  .connector_id   │  │  .possible_crtcs │            │    │
│  │  │  .mode             │  │  .display_info   │  │  .crtc           │            │    │
│  │  │  .state (atomic)   │  │  .state (atomic) │  │  .bridge         │            │    │
│  │  │  .funcs            │  │  .funcs          │  │  .funcs          │            │    │
│  │  │  .helper_private   │  │  .helper_private │  │  .helper_private │            │    │
│  │  └────────┬──────────┘  └────────┬─────────┘  └────────┬─────────┘            │    │
│  │           │                      │                      │                       │    │
│  │           │              drm_connector                 │                       │    │
│  │           │              .encoder_ids[]                │                       │    │
│  │           │                      │                      │                       │    │
│  │           └──────────────────────┼──────────────────────┘                       │    │
│  │                                  │ (connector <-> encoder binding)                 │    │
│  │                                  │                                              │    │
│  │  ┌───────────────────────────────┴──────────────────────────────────────┐     │    │
│  │  │                         drm_framebuffer                              │     │    │
│  │  │  .width / .height / .pitches[4] / .offsets[4] / .modifier            │     │    │
│  │  │  .format (drm_format_info) / .funcs                                  │     │    │
│  │  └───────────────────────────────┬──────────────────────────────────────┘     │    │
│  │                                  │                                              │    │
│  │  ┌───────────────────────────────┴──────────────────────────────────────┐     │    │
│  │  │                         drm_plane                                    │     │    │
│  │  │  .type (PRIMARY/OVERLAY/CURSOR) / .possible_crtcs                    │     │    │
│  │  │  .format_types[] / .state (drm_plane_state)                          │     │    │
│  │  │  .funcs (drm_plane_funcs) / .helper_private                          │     │    │
│  │  └───────────────────────────────┬──────────────────────────────────────┘     │    │
│  │                                  │                                              │    │
│  │  ┌───────────────────────────────┴──────────────────────────────────────┐     │    │
│  │  │                    drm_property (属性系统)                            │     │    │
│  │  │  .name / .type / .values[] / .enum_values[] / .blob                  │     │    │
│  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │     │    │
│  │  │  │ CRTC_ID  │ │ FB_ID    │ │ IN_FENCE │ │ OUT_FENCE│ │ alpha    │   │     │    │
│  │  │  │ CONNEC-  │ │ MODE_ID  │ │ CRTC_X/Y │ │ CRTC_W/H │ │ SRC_X/Y  │   │     │    │
│  │  │  │ TOR_ID   │ │          │ │ CRTC_W/H  │ │          │ │ SRC_W/H  │   │     │    │
│  │  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘   │     │    │
│  │  └──────────────────────────────────────────────────────────────────────┘     │    │
│  └──────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘

                        KMS 模式设置完整调用链
                        ═══════════════════════════

                         Legacy Path                          Atomic Path
                      (drmModeSetCrtc)                    (drmModeAtomicCommit)
                            │                                      │
                            ▼                                      ▼
                    ┌───────────────┐                    ┌───────────────┐
                    │ drm_mode_set- │                    │ drm_mode_atom-│
                    │ crtc ioctl    │                    │ ic_ioctl      │
                    └───────┬───────┘                    └───────┬───────┘
                            │                                      │
                            ▼                                      ▼
                    ┌───────────────┐                    ┌───────────────┐
                    │ drm_mode_set- │                    │ drm_atomic_   │
                    │ crtc()        │                    │ helper_set_   │
                    └───────┬───────┘                    │ config()      │
                            │                           └───────┬───────┘
                            ▼                                      │
                    ┌───────────────┐                              │
                    │ drm_crtc_     │                              │
                    │ helper_set_   │                              │
                    │ config()      │                              │
                    └───────┬───────┘                              │
                            │                                      │
                            ▼                                      ▼
                    ┌───────────────┐                    ┌───────────────┐
                    │ dm_crtc_helper│                    │ amdgpu_dm_    │
                    │ _mode_set()   │                    │ atomic_check() │
                    └───────┬───────┘                    └───────┬───────┘
                            │                                      │
                            ▼                                      ▼
                    ┌───────────────┐                    ┌───────────────┐
                    │ dc_stream_    │                    │ amdgpu_dm_    │
                    │ create()      │                    │ atomic_commit │
                    └───────┬───────┘                    │ _tail()       │
                            │                           └───────┬───────┘
                            ▼                                      │
                    ┌───────────────┐                              │
                    │ dc_commit_    │                              │
                    │ streams()     │                              │
                    └───────┬───────┘                              │
                            │                                      │
                            ▼                                      ▼
                    ┌───────────────┐                    ┌───────────────┐
                    │ dc_commit_    │                    │ dc_commit_    │
                    │ state()       │                    │ state()       │
                    └───────┬───────┘                    └───────┬───────┘
                            │                                      │
                            └──────────────────┬───────────────────┘
                                               │
                                               ▼
                                    ┌───────────────────┐
                                    │  dc->hwss.apply_  │
                                    │  ctx_for_surface  │
                                    └────────┬──────────┘
                                             │
                            ┌────────────────┼────────────────┐
                            │                │                │
                            ▼                ▼                ▼
                    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
                    │ OTG Timing   │ │ DPP Surface  │ │ MPC Blending │
                    │ Program      │ │ Program      │ │ Program      │
                    │ (H_TOTAL,    │ │ (DCSURF_ADDR,│ │ (MPC_CSC,    │
                    │  V_TOTAL,    │ │  PITCH,      │ │  MPC_OCSC,   │
                    │  HSYNC,      │ │  VIEWPORT,   │ │  MPC_OGAM)   │
                    │  VSYNC...)   │ │  SCALE...)   │ │              │
                    └──────────────┘ └──────────────┘ └──────────────┘
```

### 第二部分：关键概念辨析（20分）

#### 2.1 填空：完成以下关键概念的定义

```
1. KMS (Kernel Mode Setting) 是指 _________________________________________
   _________________________________________________，与 UMS 的区别在于 _______
   _________________________________________________。

2. CRTC (Cathode Ray Tube Controller) 在 DRM 中实际表示 ___________________
   _________________________________________________，主要负责 _______________
   _________________________________________________。

3. Atomic Modesetting 是指 _________________________________________________
   _________________________________________________，其核心优势包括 ________
   _________________________________________________。

4. VBlank (Vertical Blanking Interval) 是指 _______________________________
   _________________________________________________，在 KMS 中的主要用途包括 _
   _________________________________________________。

5. drm_display_mode 结构体中，clock 字段表示 __________________ 单位是 _____，
   hdisplay/vdisplay 表示 _________________________________________________，
   hsync_start/hsync_end 表示 _____________________________________________。
```

#### 2.2 判断对错并说明理由

```
1. [  ] 一个 CRTC 可以同时驱动多个 Connector 输出不同的画面。
   理由：_________________________________________________

2. [  ] Plane 必须绑定到 CRTC 才能生效，且一个 Plane 同一时间只能绑定到一个 CRTC。
   理由：_________________________________________________

3. [  ] drmModeSetCrtc() 是一次完整的 atomic 操作。
   理由：_________________________________________________

4. [  ] 在 AMDGPU 驱动中，dc_commit_state() 是直接操作 DCN 硬件寄存器的函数。
   理由：_________________________________________________

5. [  ] 通过 strace 追踪 xrandr 可以看到完整的 libdrm 到内核的 ioctl 调用序列。
   理由：_________________________________________________
```

### 第三部分：Mode Setting 流程追踪（20分）

#### 3.1 以下是一个通过 strace 捕获的 xrandr --output HDMI-A-1 --mode 1920x1080 操作的部分输出，请补充注释说明每个 ioctl 的作用

```c
// strace -e ioctl xrandr --output HDMI-A-1 --mode 1920x1080 2>&1 | head -30

ioctl(3, DRM_IOCTL_MODE_GETRESOURCES, 0x7fff...)  // 作用：_______________
// _________________________________________________

ioctl(3, DRM_IOCTL_MODE_GETCRTC, {crtc_id=0, ...})  // 作用：_______________
// _________________________________________________

ioctl(3, DRM_IOCTL_MODE_GETENCODER, {encoder_id=0, ...})  // 作用：___________
// _________________________________________________

ioctl(3, DRM_IOCTL_MODE_GETCONNECTOR, {connector_id=19, ...})  // 作用：______
// _________________________________________________

ioctl(3, DRM_IOCTL_MODE_GETPROPERTY, ...)  // 作用：_______________________
// _________________________________________________

ioctl(3, DRM_IOCTL_MODE_SETCRTC, {crtc_id=0, fb_id=42, ...})  // 作用：______
// _________________________________________________
```

#### 3.2 请按正确顺序排列以下 Mode Setting 函数调用（填写序号）

```
(  ) dc_commit_state()
(  ) drm_mode_setcrtc()
(  ) dm_crtc_helper_mode_set()
(  ) drm_crtc_helper_set_config()
(  ) dcn10_timing_generator_program_timing()
(  ) dc_commit_streams()
(  ) dc_stream_create()
(  ) amdgpu_dm_atomic_commit_tail()
(  ) drm_atomic_helper_set_config()
(  ) amdgpu_dm_atomic_check()
```

### 第四部分：架构图示题（10分）

请根据以下场景绘制对应的组件连接图：

**场景**：一台笔记本电脑，使用 AMD RDNA 3 集成显卡，连接了两个显示器：
- 内置 eDP 面板（1920x1080@60Hz）
- 外接 DP 显示器（2560x1440@144Hz）

请绘制：
1. DRM 核心组件（CRTC/Encoder/Connector/Plane）的连接关系
2. AMDGPU DM 映射到 DC 的 stream 和 sink 关系
3. DCN 硬件管线（OTG/DPP/OPP/DIO）的分配情况

```
参考答案框架（请完成图）：

                    ┌───────────────┐
                    │  GPU (RDNA 3) │
                    └───────┬───────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
    ┌─────┴─────┐     ┌────┴────┐      ┌────┴────┐
    │  CRTC_1   │     │ CRTC_2  │      │ CRTC_3  │
    │ (eDP)     │     │ (DP)    │      │ (备用的) │
    └─────┬─────┘     └────┬────┘      └─────────┘
          │                 │
    ┌─────┴─────┐     ┌────┴────┐
    │ Plane_1   │     │Plane_2  │
    │(Primary)  │     │(Primary)│
    └─────┬─────┘     └────┬────┘
          │                 │
    ┌─────┴─────┐     ┌────┴────┐
    │ Encoder_1 │     │Encoder_2│
    │ (eDP)     │     │ (DP)    │
    └─────┬─────┘     └────┬────┘
          │                 │
    ┌─────┴─────┐     ┌────┴────┐
    │Connector_1│     │Connector│
    │ (eDP-1)   │     │ (DP-1)  │
    │ 1920x1080 │     │2560x1440│
    │ @60Hz     │     │@144Hz   │
    └───────────┘     └─────────┘

    DC 层映射：
    ┌──────────────┐  ┌──────────────┐
    │ stream_1     │  │ stream_2     │
    │ ──> sink_1   │  │ ──> sink_2   │
    │ (eDP panel)  │  │ (DP monitor) │
    │ timing:      │  │ timing:      │
    │ 148.5MHz     │  │ 497.75MHz    │
    └──────────────┘  └──────────────┘

    DCN 管线分配：
    ┌─────┐ ┌─────┐  ┌─────┐ ┌─────┐
    │OTG0 │ │DPP0 │  │OTG1 │ │DPP1 │
    │     │ │     │  │     │ │     │
    │eDP  │ │eDP  │  │DP   │ │DP   │
    │tim- │ │surf-│  │tim- │ │surf-│
    │ing  │ │ace  │  │ing  │ │ace  │
    └─────┘ └─────┘  └─────┘ └─────┘
```

### 第五部分：调试工具实践（10分）

请写出以下调试场景下应使用的命令：

#### 5.1 追踪 xrandr 设置分辨率的完整 ioctl 调用序列

```bash
# 命令：
```

#### 5.2 在 AMDGPU 驱动中追踪 mode setting 相关的内核函数调用

```bash
# 1. 使用 ftrace 追踪：
```

```bash
# 2. 使用 bpftrace 追踪 OTG 寄存器编程：
```

#### 5.3 查看当前 CRTC 的 mode 设置及 VBlank 状态

```bash
# 通过 sysfs/debugfs：
```

#### 5.4 验证 framebuffer 格式和 modifier

```bash
# 通过 modetest：
```

#### 5.5 测试 vblank 事件响应

```bash
# 使用 modetest 的 vblank 测试：
```

### 第六部分：案例分析（20分）

#### 6.1 CRTC 资源分配失败

**场景**：用户连接了 3 台 4K 显示器到 AMD GPU，第三台显示器无法点亮。dmesg 输出如下：
```
[drm:amdgpu_dm_atomic_check] No more crtc available
[drm:drm_mode_setcrtc] Failed to set mode on [CRTC:70:crtc-0] (err=-28)
```

**问题**：
1. 错误码 -28 对应的含义是什么？___________________________
2. 为什么会出现 "No more crtc available" 错误？___________________________
3. AMD GPU 通常有多少个 CRTC？不同 ASIC 有何差异？___________________________
4. 在不更换硬件的前提下，有哪些解决方案？___________________________

#### 6.2 自定义分辨率设置失败

**场景**：用户尝试设置 3840x2160@75Hz 分辨率，dmesg 显示：
```
[drm:amdgpu_dm_connector_mode_valid] Mode 3840x2160: clock 712500 kHz
[drm:dm_helpers_validate_clock] Failed to validate clock 712500 kHz for TMDS
[drm:amdgpu_dm_connector_mode_valid] Mode 3840x2160: status=MODE_CLOCK_RANGE
```

**问题**：
1. 该分辨率无法设置的根本原因是什么？___________________________
2. 像素时钟 712.5MHz 与 TMDS 时钟的关系是什么？___________________________
3. 为什么 3840x2160@60Hz 可以正常工作？___________________________
4. 如果显示器支持 DSC，是否有可能实现该分辨率？___________________________

#### 6.3 Atomic Commit 超时

**场景**：多任务负载下，系统间歇性出现屏幕闪烁，dmesg 显示：
```
[drm:amdgpu_dm_atomic_commit_tail] *ERROR* [CRTC:70:crtc-0] Atomic commit timeout
[drm:amdgpu_dm_atomic_commit_tail] *ERROR* Waiting for flips timed out
```

**问题**：
1. Atomic commit timeout 的默认超时时间是多少？___________________________
2. 列出可能导致该超时的 3 个原因：___________________________
3. 如何通过调试手段定位具体原因？___________________________
4. 如何临时绕过此问题？___________________________

---

## 参考答案

### 第一部分参考答案

架构图中已展示完整的 KMS 分层架构：
- 用户层：xrandr/wayland/modetest → libdrm → ioctl
- 内核层：DRM Core → DRM Helper → AMDGPU DM → DC → DCN HW
- 数据流向：Mode 配置 → CRTC/Encoder/Connector 绑定 → FB 关联 → Plane 配置
- 数据结构：drm_device → drm_mode_config → drm_crtc/drm_connector/drm_encoder/drm_plane

### 第二部分参考答案

#### 2.1 填空

1. **KMS**：由内核直接管理显示模式设置，而非由用户空间 X 服务器管理。与 UMS 的区别在于内核拥有模式设置的最终控制权，支持多进程/多显示服务器的原子性操作，消除了 X 服务器崩溃导致显示异常的风险。

2. **CRTC**：在 DRM 中实际表示数字显示控制器（Display Controller），主要负责从 framebuffer 读取像素数据，生成显示时序信号（H-sync/V-sync/DE/Clock），并驱动 Encoder 将数据发送到显示设备。

3. **Atomic Modesetting**：一种将所有显示状态变更（CRTC/Connector/Encoder/Plane 配置）合并为单个原子操作提交的机制。核心优势包括：无中间状态（避免闪屏）、自动依赖解析、支持 TEST_ONLY 验证、支持 fences 同步、状态回滚能力。

4. **VBlank**：在显示扫描完成一帧后，电子枪从右下角回扫到左上角的时间间隔。在 KMS 中的主要用途包括：页面翻转同步（避免撕裂）、帧率计数、vblank 同步渲染、vblank 事件监听。

5. **drm_display_mode**：`clock` 字段表示像素时钟频率（Pixel Clock），单位是 kHz。`hdisplay/vdisplay` 表示图像的可见分辨率（有效像素）。`hsync_start/hsync_end` 表示行同步信号的起始和结束位置（包含 front porch、sync、back porch）。

#### 2.2 判断对错

1. **[X] 错误**。一个 CRTC 只能驱动一个 Connector 输出画面（或通过 MST 分发同一画面到多个显示器）。要让不同 Connector 显示不同内容，需要多个 CRTC。
   *理由：CRTC 是独立的显示时序生成器，每个 CRTC 产生一个视频流。要驱动两个不同画面的显示器，需要两个 CRTC。*

2. **[O] 正确**。Plane 必须绑定到 CRTC 才能参与最终的画面合成。一个 Plane 同一时间只能绑定到一个 CRTC，但一个 CRTC 可以绑定多个 Plane（primary + overlay + cursor）。
   *理由：每个 Plane 对应一个硬件层（DPP/MPC 输入），这些层最终在 MPC 中合成为一个流输出到单个 CRTC 对应的 OTG。*

3. **[X] 错误**。`drmModeSetCrtc()` 是 legacy mode setting 操作，不是 atomic 操作。它内部实现为一次性的 ioctl，但不具备 atomic 的 TEST_ONLY、状态回滚、多对象协调等特性。
   *理由：Legacy ioctl 可能产生中间状态（先解绑再重新绑定），而 atomic commit 确保要么全部成功、要么全部回滚。*

4. **[X] 错误**。`dc_commit_state()` 是 DC 层的核心调度函数，它负责协调状态转换，但实际硬件寄存器的编程是通过 `dc->hwss`（Hardware Sequencer Function Table）中的函数（如 `dcn10_timing_generator_program_timing()`）来完成的。
   *理由：DC 层抽象了硬件差异，`dc->hwss` 根据 DCN 版本（DCN 2.0/3.0/3.1）选择不同的实现。*

5. **[O] 正确**。strace 可以捕获所有 ioctl 系统调用，包括 libdrm 发起的 `DRM_IOCTL_MODE_*` 系列 ioctl。通过 strace 可以完整观察到从 xrandr 到内核的 ioctl 序列。
   *理由：strace 工作在系统调用层，对用户空间完全透明，可以看到所有 ioctl 的编号、参数和返回值。*

### 第三部分参考答案

#### 3.1 ioctl 注释

```c
ioctl(3, DRM_IOCTL_MODE_GETRESOURCES, 0x7fff...)
// 作用：获取 DRM 设备的全部资源列表，包括所有 CRTC ID、Encoder ID、
//       Connector ID、FB ID 的数量和 ID 数组。

ioctl(3, DRM_IOCTL_MODE_GETCRTC, {crtc_id=0, ...})
// 作用：查询指定 CRTC 的当前状态，包括当前 mode、绑定的 fb_id、
//       (x,y) 偏移位置、是否开启等。

ioctl(3, DRM_IOCTL_MODE_GETENCODER, {encoder_id=0, ...})
// 作用：查询指定 Encoder 的信息，包括 encoder_type、possible_crtcs 位图、
//       绑定的 crtc_id、可能的 connector 列表。

ioctl(3, DRM_IOCTL_MODE_GETCONNECTOR, {connector_id=19, ...})
// 作用：查询指定 Connector 的详细信息，包括连接状态、物理尺寸、支持的
//       所有 mode 列表、属性、EDID 数据、dpms 状态等。

ioctl(3, DRM_IOCTL_MODE_GETPROPERTY, ...)
// 作用：查询 DRM 属性的定义，包括属性名、类型（range/enum/bitmask/blob）、
//       可取值列表、当前值等。

ioctl(3, DRM_IOCTL_MODE_SETCRTC, {crtc_id=0, fb_id=42, ...})
// 作用：设置 CRTC 的模式配置，包括绑定的 fb_id、显示的起始位置 (x,y)、
//       配置的 mode（timing 参数）。这是 legacy mode setting 的核心 ioctl。
```

#### 3.2 函数调用顺序

```
Legacy 路径：
( 2  ) drm_mode_setcrtc()              ← ioctl 处理入口
( 3  ) drm_crtc_helper_set_config()     ← Helper 封装
( 4  ) dm_crtc_helper_mode_set()        ← AMDGPU DM 转换
( 5  ) dc_stream_create()                ← DC stream 创建
( 6  ) dc_commit_streams()              ← DC stream 提交
( 7  ) dc_commit_state()                ← DC 状态应用
( 9  ) dcn10_timing_generator_program_timing()  ← OTG 寄存器编程

Atomic 路径：
( 1  ) drm_atomic_helper_set_config()   ← Atomic 入口
( 8  ) amdgpu_dm_atomic_check()         ← Atomic check 阶段
( 8  ) amdgpu_dm_atomic_commit_tail()   ← Atomic commit 阶段
( 7  ) dc_commit_state()                ← 状态应用（两条路径交汇）
( 9  ) dcn10_timing_generator_program_timing()  ← 硬件编程
```

注：`dc_commit_streams()` 和 `amdgpu_dm_atomic_commit_tail()` 在不同路径中位于不同的调用层次，但最终都汇入 `dc_commit_state()`。

### 第四部分参考答案

参见上方完整架构图中的双显示器场景说明：
- CRTC_1 + Plane_1(Primary) + Encoder_1(eDP) + Connector_1(eDP-1) = 内置面板
- CRTC_2 + Plane_2(Primary) + Encoder_2(DP) + Connector_2(DP-1) = 外接显示器
- DC 层分别创建 stream_1 → sink_1 和 stream_2 → sink_2
- DCN 管线分配 OTG0/DPP0 给 eDP，OTG1/DPP1 给 DP
- 每个 stream 有自己的 timing 参数（像素时钟不同）

### 第五部分参考答案

```bash
# 5.1 追踪 xrandr ioctl
strace -e ioctl xrandr --output HDMI-A-1 --mode 1920x1080 2>&1 | grep DRM_IOCTL

# 5.2a ftrace 追踪
echo function > /sys/kernel/tracing/current_tracer
echo amdgpu_dm_atomic_commit_tail dc_commit_streams dc_commit_state > /sys/kernel/tracing/set_ftrace_filter
echo ':mod:amdgpu' > /sys/kernel/tracing/set_ftrace_filter
echo 1 > /sys/kernel/tracing/tracing_on
xrandr --output HDMI-A-1 --mode 1920x1080
cat /sys/kernel/tracing/trace

# 5.2b bpftrace 追踪 OTG 寄存器
bpftrace -e 'kprobe:dcn10_timing_generator_program_timing { printf("OTG programmed: crtc=%d\n", arg0); }'

# 5.3 查看 CRTC mode 和 VBlank 状态
cat /sys/kernel/debug/dri/0/crtc-0/state
cat /sys/kernel/debug/dri/0/crtc-0/vblank_count
cat /sys/kernel/debug/dri/0/crtc-0/vblank_enable

# 5.4 验证 framebuffer 格式
modetest -M amdgpu -f

# 5.5 vblank 测试
modetest -M amdgpu -v
```

### 第六部分参考答案

#### 6.1 CRTC 资源分配失败

1. 错误码 -28 对应 `-ENOSPC`（No space left on device），在此上下文中表示没有可用的 CRTC 资源。

2. AMD GPU 的 CRTC 数量有限（通常 Navi 系列有 6 个，APU 有 3-4 个）。当连接的显示器超过 CRTC 数量时，额外的显示器无法获取 CRTC 资源，导致 mode setting 失败。

3. 不同 ASIC 的 CRTC 数量：
   - RDNA 1/2/3 (Navi 10/21/31)：6 个 CRTC
   - RDNA 3.5 (Strix Point)：4 个 CRTC
   - RDNA 2 APU (Rembrandt)：4 个 CRTC
   - GCN/GFX8 以前：6 个 CRTC
   - Vega 10/20：6 个 CRTC

4. 解决方案：
   - 关闭一个显示器释放 CRTC
   - 使用 MST Hub（DisplayPort Multi-Stream Transport）在单 CRTC 下驱动多台显示器（需 hardware 支持 MST daisy chain）
   - 降低分辨率（不影响 CRTC 数量，但某些情况下可以减少带宽需求）
   - 使用支持 DSC 的显示器以降低接口带宽

#### 6.2 自定义分辨率设置失败

1. 根本原因：3840x2160@75Hz 的像素时钟 712.5MHz 超出了 HDMI 1.4 的 TMDS 时钟上限（340MHz）。即使使用 HDMI 2.0（上限 600MHz），712.5MHz 仍然超出规格。

2. TMDS 时钟与像素时钟的关系：TMDS 时钟 = 像素时钟 × (color_depth / 8)
   - 8bit 色深：TMDS 时钟 = 712.5MHz（HDMI 1.4 每通道编码效率 1:1）
   - 实际 HDMI 1.4 TMDS 传输时钟 = 像素时钟 × 1.25（包含 TMDS 字符开销）
   - 712.5MHz × 1.25 = 890.625MHz（远超极限）

3. 3840x2160@60Hz 可以正常工作是因为其像素时钟约 594MHz（实际 533.25MHz-594MHz 根据 CVT-RB），在 HDMI 2.0 的 TMDS 600MHz 上限范围内。444.8MHz(TMDS) × 1.25 ≈ 556MHz，仍在 600MHz 内。

4. 如果显示器支持 DSC（Display Stream Compression），可以压缩数据量从而在较低的链路速率下传输更高的分辨率。DSC 可以压缩 2:1-3:1 的比率。因此开启 DSC 后，3840x2160@75Hz 可能在 HDMI 2.0 上实现（压缩后带宽减半）。

#### 6.3 Atomic Commit 超时

1. 默认超时时间：1000ms（1 秒），可调参数：
   ```
   /sys/module/drm/parameters/atomic_timeout_ms
   ```

2. 可能原因：
   - GPU 负载过高：渲染命令堆积，DC 等待 GPU 空闲超时
   - framebuffer 被 pin 住：缓冲区被其他 GPU 引擎占用，无法完成翻转
   - VBlank IRQ 丢失：中断处理不及时，atomic commit 等待 vblank 事件超时
   - DCN 硬件卡死：OTG 状态机异常，register 编程无响应

3. 定位方法：
   - 检查 dmesg 完整日志：`dmesg | grep -i "atomic\|timeout\|flip"`
   - 检查 framebuffer 状态：`cat /sys/kernel/debug/dri/0/framebuffer`
   - 检查 GPU 频率：`cat /sys/class/drm/card0/device/pp_dpm_sclk`
   - 使用 ftrace 追踪 atomic commit 流程：`echo amdgpu_dm_atomic_commit_tail > /sys/kernel/tracing/set_ftrace_filter`
   - 使用 bpftrace 追踪 register 写入时间：`kretprobe:dcn10_update_plane { printf("took %d us\n", elapsed); }`

4. 临时绕过方案：
   - 增加 atomic 超时时间：`echo 2000 > /sys/module/drm/parameters/atomic_timeout_ms`
   - 降低 GPU 负载：关闭不必要的图形应用
   - 强制 GPU 进入高性能模式：`echo high > /sys/class/drm/card0/device/power_dpm_force_performance_level`
   - 降低刷新率或分辨率以减少带宽需求

---

## 考核评分标准

| 部分 | 分值 | 评分要素 |
|------|------|----------|
| 第一部分：KMS 架构图 | 40分 | 架构完整性（10分）、组件连接准确性（10分）、函数调用链正确性（10分）、数据结构关系正确性（10分） |
| 第二部分：概念辨析 | 20分 | 填空准确度（10分）、判断对错及理由充分性（10分） |
| 第三部分：流程追踪 | 20分 | ioctl 注释理解（10分）、函数调用顺序（10分） |
| 第四部分：架构图示 | 10分 | 组件分配正确性（5分）、连接关系正确性（5分） |
| 第五部分：调试工具 | 10分 | 命令准确性（5分）、参数正确性（5分） |
| 第六部分：案例分析 | 20分 | 根因分析深度（10分）、解决方案可行性（10分） |
| **总分** | **120分** | ≥90 分为优秀，≥70 分为合格 |

## 扩展学习资源

- Linux 内核文档: https://www.kernel.org/doc/html/latest/gpu/drm-kms.html
- AMDGPU 显示驱动源码: drivers/gpu/drm/amd/display/
- DRM 调试指南: https://www.kernel.org/doc/html/latest/gpu/drm-uapi.html
- IGT GPU Tools: https://gitlab.freedesktop.org/drm/igt-gpu-tools
- AMD ROCm 文档: https://rocm.docs.amd.com/
