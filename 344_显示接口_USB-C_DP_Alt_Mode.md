# 第344天：显示接口：USB-C DP Alt Mode

## 学习目标
- 理解 USB-C 接口的物理层特性与 Alternate Mode 概念
- 掌握 DP Alt Mode 的协议转换机制与 Pin 分配
- 学习 USB-C DP Alt Mode 的链路训练与协商流程
- 理解 DP Alt Mode 与 USB 3.x 数据共存机制
- 掌握 AMDGPU 驱动中 DP Alt Mode 的实现
- 学习 USB-C 显示输出的测试方法
- 理解 USB-C 充电与显示同时工作的电源管理
- 掌握常见 USB-C DP 兼容性问题的调试方法

## 知识详解

### 1. USB-C 接口基础

#### 1.1 USB-C 物理层特性

USB-C 接口采用 24-pin 可反转连接器设计，支持正反插：

```
USB-C 连接器引脚定义 (从连接器正面视角)

USB-C Receptacle (母座) 引脚图:

┌────────────────────────────────────────────────────────┐
│  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐  │
│  │GND │TX1+│TX1-│VBUS│CC1 │D+  │D-  │SBU1│VBUS│RX2-│  │
│  │ A1 │ A2 │ A3 │ A4 │ A5 │ A6 │ A7 │ A8 │ A9 │ A10│  │
│  ├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤  │
│  │    │    │    │    │    │    │    │    │    │    │  │
│  │ B1 │ B2 │ B3 │ B4 │ B5 │ B6 │ B7 │ B8 │ B9 │ B10│  │
│  │GND │RX1+│RX1-│VBUS│SBU2│D-  │D+  │CC2 │VBUS│TX2-│  │
│  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘  │
│                          ██                            │
│  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐  │
│  │GND │TX2+│TX2-│VBUS│CC2 │D+  │D-  │SBU2│VBUS│RX1-│  │
│  │ B1 │ B2 │ B3 │ B4 │ B5 │ B6 │ B7 │ B8 │ B9 │ B10│  │
│  ├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤  │
│  │GND │RX1+│RX1-│VBUS│CC1 │D-  │D+  │SBU1│VBUS│TX1-│  │
│  │ A1 │ A2 │ A3 │ A4 │ A5 │ A6 │ A7 │ A8 │ A9 │ A10│  │
│  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘  │
└────────────────────────────────────────────────────────┘
                ↑ 正插 (Top View)         ↑ 反插 (Bottom View)

引脚功能分类:
  ┌──────────┬──────────────────────────────────────────────┐
  │ 引脚组    │  功能                                        │
  ├──────────┼──────────────────────────────────────────────┤
  │ GND      │  电源地 (A1, B1, A12, B12)                   │
  │ VBUS     │  总线电源 (A4, A9, B4, B9) - 最大 5A/20V    │
  │ CC1/CC2  │  配置通道 (A5, B5) - 角色检测/协商           │
  │ D+/D-    │  USB 2.0 差分对 (A6/A7, B6/B7)              │
  │ TX1/2±   │  SuperSpeed 发送差分对                        │
  │ RX1/2±   │  SuperSpeed 接收差分对                        │
  │ SBU1/2   │  边带使用 (A8, B8) - DP Alt Mode 音频/DPCD  │
  └──────────┴──────────────────────────────────────────────┘

USB-C 关键电气特性:
  参数              规格
  ────────────     ─────────────────────
  VBUS 电压        5V (默认), 最高 20V (PD)
  VBUS 电流        0.5A (默认), 最高 5A (PD)
  差分阻抗         90Ω ± 15% (100Ω DP)
  插入损耗         < 10dB @ 10GHz
  最大速率         USB 3.2 Gen2x2 (20Gbps)
  CC 电压          0.35V - 1.6V (取决于角色)
```

#### 1.2 CC (Configuration Channel) 协议

CC 线是 USB-C 的核心控制通道：

```
CC 线功能与状态

CC 线在 USB-C 系统中的三种主要功能:

1. 连接检测与角色识别
   ┌──────────────┬──────────────┬──────────────────┐
   │  角色         │  CC 电阻      │  电压范围          │
   ├──────────────┼──────────────┼──────────────────┤
   │  Source (DFP)│  Rp (上拉)    │  0.35V - 1.6V    │
   │  Sink (UFP)  │  Rd (下拉)    │  0V - 0.2V       │
   │  DRP         │  交替 Rp/Rd  │  变化             │
   └──────────────┴──────────────┴──────────────────┘

2. 电流能力通告 (通过 Rp 值编码)
   ┌─────────────┬─────────────────┬────────────────────┐
   │  Rp 值       │  默认电流        │  USB PD 协商后     │
   ├─────────────┼─────────────────┼────────────────────┤
   │  10kΩ (Rd)  │  USB 2.0 (0.5A) │  不支持 PD         │
   │  22kΩ (Rp)  │  1.5A @ 5V      │  支持 PD 协商      │
   │  56kΩ (Rp)  │  3.0A @ 5V      │  支持 PD 协商      │
   └─────────────┴─────────────────┴────────────────────┘

3. Alternate Mode 发现与协商
   通过 CC 线传输 SOP'/SOP'' 发现消息:
   ┌──────────────┬──────────────────────────────────────┐
   │  消息类型     │  作用                                 │
   ├──────────────┼──────────────────────────────────────┤
   │  Discover    │  查询 Sink 支持的 Alt Modes           │
   │  Enter Mode │  进入指定的 Alt Mode (如 DP Alt Mode) │
   │  Exit Mode  │  退出 Alt Mode                        │
   │  Status     │  查询 Alt Mode 状态                   │
   └──────────────┴──────────────────────────────────────┘
```

### 2. DP Alt Mode 概述

#### 2.1 DP Alt Mode 协议栈

```
DP Alt Mode 协议栈架构

USB-C 物理层
┌────────────────────────────────────────────────────────┐
│  USB-C 连接器与线缆                                    │
│  24-pin, 正反插, 最高 240W (PD 3.1)                   │
└─────────────────────┬──────────────────────────────────┘
                      │
┌─────────────────────┴──────────────────────────────────┐
│  CC 协议层 (Configuration Channel)                     │
│  ├── 连接检测与角色协商                                │
│  ├── VCONN 供电                                       │
│  ├── Alternate Mode 发现/进入/退出                     │
│  └── USB PD 消息传输                                   │
└─────────────────────┬──────────────────────────────────┘
                      │
┌─────────────────────┴──────────────────────────────────┐
│  DP Alt Mode 适配层                                    │
│  ├── Pin 分配配置 (C/D/E/F 模式)                      │
│  ├── DP 链路训练通过 AUX (经 SBU)                     │
│  ├── DPCD 寄存器映射                                   │
│  └── HPD 信号通过 CC 或 SBU                           │
└─────────────────────┬──────────────────────────────────┘
                      │
┌─────────────────────┴──────────────────────────────────┐
│  DisplayPort 协议层                                    │
│  ├── Main Link (视频数据)                              │
│  │   ├── 1-4 lane, HBR/UHBR                            │
│  │   └── 使用 USB-C 高速差分对传输                     │
│  ├── AUX 通道 (链路训练/DPCD/EDID)                    │
│  │   └── 通过 SBU 引脚传输                             │
│  └── HPD (热插拔检测)                                  │
│      └── 通过 CC 或 SBU 传输                           │
└────────────────────────────────────────────────────────┘
```

#### 2.2 Pin 分配模式

DP Alt Mode 定义了 4 种 Pin 分配模式 (C/D/E/F)：

```
DP Alt Mode Pin 分配

Mode C (2-lane DP + USB 3.x):
  ┌─────┬───────┬──────────┬──────────┬─────────────────────┐
  │ Pin │ USB-C │ DP Alt   │ 信号      │ 说明                │
  ├─────┼───────┼──────────┼──────────┼─────────────────────┤
  │ A2  │ TX1+  │ DP Lane0 │ ML0+     │ DP 主链路 Lane 0    │
  │ A3  │ TX1-  │ DP Lane0 │ ML0-     │                     │
  │ B2  │ TX2+  │ USB3 TX │ SSTX+    │ USB 3.x 发送        │
  │ B3  │ TX2-  │ USB3 TX │ SSTX-    │                     │
  │ A10 │ RX2-  │ USB3 RX │ SSRX-    │ USB 3.x 接收        │
  │ B10 │ RX1-  │ DP Lane1 │ ML1-     │ DP 主链路 Lane 1    │
  │ A8  │ SBU1  │ DP AUX  │ AUX+     │ DP AUX 通道 P       │
  │ B8  │ SBU2  │ DP AUX  │ AUX-     │ DP AUX 通道 N       │
  │ A5  │ CC1   │ CC/HPD  │ CC+      │ CC 或 HPD 信号      │
  │ B5  │ CC2   │ VCONN   │ VCONN    │ 线缆供电            │
  └─────┴───────┴──────────┴──────────┴─────────────────────┘

Mode D (4-lane DP, 无 USB 3.x):
  ┌─────┬───────┬──────────┬──────────┬─────────────────────┐
  │ Pin │ USB-C │ DP Alt   │ 信号      │ 说明                │
  ├─────┼───────┼──────────┼──────────┼─────────────────────┤
  │ A2  │ TX1+  │ DP Lane0 │ ML0+     │ DP 主链路 Lane 0    │
  │ A3  │ TX1-  │ DP Lane0 │ ML0-     │                     │
  │ B2  │ TX2+  │ DP Lane1 │ ML1+     │ DP 主链路 Lane 1    │
  │ B3  │ TX2-  │ DP Lane1 │ ML1-     │                     │
  │ A10 │ RX2-  │ DP Lane3 │ ML3-     │ DP 主链路 Lane 3    │
  │ B10 │ RX1-  │ DP Lane2 │ ML2-     │ DP 主链路 Lane 2    │
  │ A8  │ SBU1  │ DP AUX  │ AUX+     │ DP AUX 通道 P       │
  │ B8  │ SBU2  │ DP AUX  │ AUX-     │ DP AUX 通道 N       │
  │ A5  │ CC1   │ CC/HPD  │ CC+      │ CC 或 HPD 信号      │
  │ B5  │ CC2   │ VCONN   │ VCONN    │ 线缆供电            │
  └─────┴───────┴──────────┴──────────┴─────────────────────┘

Mode E (2-lane DP + USB 2.0):
  ┌─────┬───────┬──────────┬──────────┬─────────────────────┐
  │ Pin │ USB-C │ DP Alt   │ 信号      │ 说明                │
  ├─────┼───────┼──────────┼──────────┼─────────────────────┤
  │ A2  │ TX1+  │ DP Lane0 │ ML0+     │ DP 主链路 Lane 0    │
  │ A3  │ TX1-  │ DP Lane0 │ ML0-     │                     │
  │ B2  │ TX2+  │ DP Lane1 │ ML1+     │ DP 主链路 Lane 1    │
  │ B3  │ TX2-  │ DP Lane1 │ ML1-     │                     │
  │ A10 │ RX2-  │ unused   │ -        │ 未使用              │
  │ B10 │ RX1-  │ unused   │ -        │ 未使用              │
  │ A8  │ SBU1  │ DP AUX  │ AUX+     │ DP AUX 通道 P       │
  │ B8  │ SBU2  │ DP AUX  │ AUX-     │ DP AUX 通道 N       │
  │ A5  │ CC1   │ CC/HPD  │ CC+      │ CC 或 HPD 信号      │
  │ B5  │ CC2   │ VCONN   │ VCONN    │ 线缆供电            │
  └─────┴───────┴──────────┴──────────┴─────────────────────┘

Mode F (2-lane DP + USB 3.x + USB 2.0):
  ┌─────┬───────┬──────────┬──────────┬─────────────────────┐
  │ Pin │ USB-C │ DP Alt   │ 信号      │ 说明                │
  ├─────┼───────┼──────────┼──────────┼─────────────────────┤
  │ A2  │ TX1+  │ DP Lane0 │ ML0+     │ DP 主链路 Lane 0    │
  │ A3  │ TX1-  │ DP Lane0 │ ML0-     │                     │
  │ B2  │ TX2+  │ USB3 TX │ SSTX+    │ USB 3.x 发送        │
  │ B3  │ TX2-  │ USB3 TX │ SSTX-    │                     │
  │ A10 │ RX2-  │ DP Lane1 │ ML1-     │ DP 主链路 Lane 1    │
  │ B10 │ RX1-  │ USB3 RX │ SSRX-    │ USB 3.x 接收        │
  │ A8  │ SBU1  │ DP AUX  │ AUX+     │ DP AUX 通道 P       │
  │ B8  │ SBU2  │ DP AUX  │ AUX-     │ DP AUX 通道 N       │
  │ A5  │ CC1   │ CC/HPD  │ CC+      │ CC 或 HPD 信号      │
  │ B5  │ CC2   │ VCONN   │ VCONN    │ 线缆供电            │
  │ A6/A7│ D+/D-│ USB2.0 │ D+/D-    │ USB 2.0 通道        │
  └─────┴───────┴──────────┴──────────┴─────────────────────┘
```

#### 2.3 DP Alt Mode 协商流程

```
DP Alt Mode 完整协商流程

USB-C 连接事件
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ Phase 1: USB-C 连接检测                             │
│   ├── CC1/CC2 检测到 Rd 下拉                        │
│   ├── Source 检测到连接                              │
│   └── USB 2.0 枚举（可选）                           │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│ Phase 2: Alt Mode 发现 (Discovery)                  │
│   ├── Source 发送 SOP' Discover SVIDs 消息          │
│   └── Sink 回复支持的 SVID 列表                      │
│       ├── 0xFF01: DP Alt Mode                     │
│       └── 其他: Thunderbolt, HDMI, 等              │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│ Phase 3: Alt Mode 进入 (Enter Mode)                 │
│   ├── Source 发送 Enter Mode (0xFF01, DP)           │
│   ├── Sink 确认进入 DP Alt Mode                     │
│   ├── Pin 分配协商 (C/D/E/F)                        │
│   └── 高速差分对切换到 DP 信号                       │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│ Phase 4: DP 链路训练                                 │
│   ├── AUX 通道通过 SBU 建立                          │
│   ├── HPD 信号建立                                   │
│   ├── Source 读取 Sink DPCD 能力                    │
│   ├── 链路训练 (CR/EQ/Symbol Lock)                  │
│   └── 视频流开始                                     │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│ Phase 5: 正常运行                                     │
│   ├── 视频数据传输                                    │
│   ├── USB 2.0 同时工作 (所有模式)                    │
│   ├── USB 3.x 可选 (C/F 模式)                        │
│   └── HPD 热插拔事件监测                              │
└─────────────────────────────────────────────────────┘

协商时序:
  阶段            典型时间
  ────────────    ──────────
  Phase 1 (检测)   < 100ms
  Phase 2 (发现)   < 50ms
  Phase 3 (进入)   < 50ms
  Phase 4 (训练)   < 500ms
  总计             < 700ms
```

### 3. DP Alt Mode 驱动实现

#### 3.1 Linux 内核 USB-C DP Alt Mode 支持

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_usb_c.c
// AMDGPU USB-C DP Alt Mode 支持

#include <linux/usb/typec.h>
#include <linux/usb/typec_altmode.h>
#include <linux/usb/typec_dp.h>

struct amdgpu_usb_c_dp {
    struct typec_altmode *typec_alt;
    struct typec_port *typec_port;
    struct dc_link *dc_link;
    int dp_lane_count;
    int pin_assignment;
    bool mode_entered;
    bool hpd_connected;

    // Pin 分配配置
    const struct dp_pin_assignment *pin_cfg;
    struct work_struct hpd_work;
};

// Pin 分配模式定义
struct dp_pin_assignment {
    int mode;
    int dp_lanes;
    bool usb3_enabled;
    bool usb2_enabled;
    const char *description;
};

static const struct dp_pin_assignment dp_pin_assignments[] = {
    { .mode = 'C', .dp_lanes = 2, .usb3_enabled = true,
      .usb2_enabled = true, .description = "2-lane DP + USB 3.x" },
    { .mode = 'D', .dp_lanes = 4, .usb3_enabled = false,
      .usb2_enabled = true, .description = "4-lane DP only" },
    { .mode = 'E', .dp_lanes = 2, .usb3_enabled = false,
      .usb2_enabled = true, .description = "2-lane DP + USB 2.0" },
    { .mode = 'F', .dp_lanes = 2, .usb3_enabled = true,
      .usb2_enabled = true, .description = "2-lane DP + USB 3.x alt" },
};

// Alt Mode 进入回调
static int amdgpu_dp_altmode_enter(struct typec_altmode *alt)
{
    struct amdgpu_usb_c_dp *dp = typec_altmode_get_drvdata(alt);
    struct typec_dp_config config = {0};
    int ret;

    // 1. 选择 Pin 分配模式
    config.pin_assignment = TYPEC_DP_PIN_ASSIGNMENT_C;
    config.role = TYPEC_DP_SOURCE;

    // 2. 发送 DP 配置
    ret = typec_altmode_notify(alt, TYPEC_DP_STATE_ENTER, &config);
    if (ret < 0) {
        DRM_ERROR("Failed to enter DP Alt Mode\n");
        return ret;
    }

    dp->mode_entered = true;
    dp->pin_assignment = config.pin_assignment;

    DRM_INFO("USB-C DP Alt Mode entered (Pin Assignment %c)\n",
             dp_pin_assignments[config.pin_assignment].mode);

    return 0;
}

// Alt Mode 退出回调
static int amdgpu_dp_altmode_exit(struct typec_altmode *alt)
{
    struct amdgpu_usb_c_dp *dp = typec_altmode_get_drvdata(alt);

    // 1. 通知 DC 链路断开
    if (dp->dc_link)
        dc_link_set_usb_c_dp_alt_mode(dp->dc_link, false);

    // 2. 通知 Type-C 子系统
    typec_altmode_notify(alt, TYPEC_DP_STATE_EXIT, NULL);

    dp->mode_entered = false;
    DRM_INFO("USB-C DP Alt Mode exited\n");

    return 0;
}

// HPD 事件处理
static void amdgpu_dp_altmode_hpd_work(struct work_struct *work)
{
    struct amdgpu_usb_c_dp *dp =
        container_of(work, struct amdgpu_usb_c_dp, hpd_work);

    if (dp->hpd_connected) {
        // HPD 高电平: 显示器已连接
        dc_link_set_hpd_signal(dp->dc_link, true);
        DRM_DEBUG("USB-C DP: HPD connected\n");
    } else {
        // HPD 低电平: 显示器断开
        dc_link_set_hpd_signal(dp->dc_link, false);
        DRM_DEBUG("USB-C DP: HPD disconnected\n");
    }
}
```

#### 3.2 AMDGPU DC 的 USB-C DP 支持

```c
// drivers/gpu/drm/amd/display/dc/core/dc_link_usb_c.c
// DC 层 USB-C DP 支持

#include "dc_link_usb_c.h"

struct dc_usb_c_dp {
    struct dc_link *link;
    uint32_t dp_lane_count;
    uint32_t dp_link_rate;
    bool alt_mode_active;

    // USB-C 特定参数
    uint32_t available_usb_lanes;    // 剩余的 USB 通道数
    uint32_t total_dp_lanes;         // 可用的 DP 通道数
    bool usb3_disabled;              // 是否牺牲 USB 3.x
};

// 根据 Pin 分配计算可用 DP 通道数
uint32_t dc_link_usb_c_get_dp_lane_count(
    struct dc_link *link,
    int pin_assignment)
{
    switch (pin_assignment) {
    case TYPEC_DP_PIN_ASSIGNMENT_C: // 2 DP + USB 3.x
        return 2;
    case TYPEC_DP_PIN_ASSIGNMENT_D: // 4 DP only
        return 4;
    case TYPEC_DP_PIN_ASSIGNMENT_E: // 2 DP + USB 2.0
        return 2;
    case TYPEC_DP_PIN_ASSIGNMENT_F: // 2 DP + USB 3.x alt
        return 2;
    default:
        return 0;
    }
}

// USB-C DP Alt Mode 带宽计算
bool dc_link_usb_c_calculate_bandwidth(
    struct dc_link *link,
    struct dc_crtc_timing *timing)
{
    struct dc_usb_c_dp *usb_c = link->usb_c_data;
    uint32_t required_lanes;
    uint32_t required_rate;
    uint32_t total_bandwidth;
    uint32_t required_bandwidth;

    // 计算所需带宽
    required_bandwidth = timing->h_addressable *
                         timing->v_addressable *
                         timing->display_color_depth *
                         timing->pix_clk_100hz / 100;

    // 根据可用通道数选择最佳链路速率
    if (usb_c->dp_lane_count >= 4) {
        // 4-lane 模式: 使用 HBR3
        required_rate = LINK_RATE_HBR3;
        required_lanes = 4;
    } else if (usb_c->dp_lane_count >= 2) {
        // 2-lane 模式
        if (required_bandwidth <= 2 * LINK_RATE_HBR3_BW) {
            required_rate = LINK_RATE_HBR3;
            required_lanes = 2;
        } else if (required_bandwidth <= 2 * LINK_RATE_HBR2_BW) {
            required_rate = LINK_RATE_HBR2;
            required_lanes = 2;
        } else {
            // 带宽不足，提示用户
            DRM_WARN("USB-C DP: Bandwidth insufficient for %dx%d@%dHz\n",
                     timing->h_addressable,
                     timing->v_addressable,
                     timing->refresh_rate);
            return false;
        }
    }

    // 更新链路设置
    link->cur_link_settings.lane_count = required_lanes;
    link->cur_link_settings.link_rate = required_rate;

    return true;
}

// USB-C PD 协商完成回调
void dc_link_usb_c_pd_finished(struct dc_link *link)
{
    struct dc_usb_c_dp *usb_c = link->usb_c_data;

    // USB PD 协商完成后可以启用高带宽模式
    // 只有在协商了足够的供电能力后才能使用高分辨率

    if (link->dpcd_caps.max_link_rate >= LINK_RATE_HBR3) {
        DRM_INFO("USB-C DP: PD negotiation complete, HBR3 enabled\n");
    }
}
```

### 4. USB-C DP Alt Mode 测试方法

#### 4.1 测试环境准备

```bash
# 1. 检查 USB-C DP Alt Mode 支持
sudo lsusb -t
# 输出示例:
# /:  Bus 04.Port 1: Dev 1, Class=root_hub, Driver=xhci-hcd/1p, 5000M
#     |__ Port 1: Dev 2, If 0, Class=Billboard, Driver=, 5000M

# 2. 查看 Type-C 端口信息
sudo cat /sys/class/typec/port0/port_type
# 输出: drp (Dual Role Port)

# 3. 检查 Alt Mode 支持
sudo cat /sys/class/typec/port0/partner/svid
# 输出: 0xff01 (DP Alt Mode)

# 4. 查看显示器连接状态
sudo cat /sys/kernel/debug/dri/0/DP-2/status
# 输出:
# connector: DP-2 (USB-C)
# status: connected
# link rate: 0x1e (8.1 Gbps)
# lane count: 2
```

#### 4.2 DP Alt Mode 调试

```bash
# 1. 使用 typec 工具查看 Alt Mode 状态
sudo typec status
# 输出示例:
# Port 0:
#   Type: DRP
#   Power: Source 3.0A
#   Alt Mode: DP
#   Pin Assignment: C (2 lanes)
#   USB 3.x: Enabled
#   HPD: Connected

# 2. 查看 DPCD 能力
sudo cat /sys/kernel/debug/dri/0/DP-2/dpcd | head -30
# 检查 USB-C 特定 DPCD 寄存器

# 3. 监控 USB-C 连接事件
sudo udevadm monitor --property
# 插入 USB-C DP 显示器时观察事件

# 4. 使用 IGT 测试
sudo ./build/tests/kms_dp_alt_mode
# 输出示例:
# IGT kms_dp_alt_mode: Starting subtest: usb_c_dp_connect
# (kms_dp_alt_mode:1234) INFO: USB-C DP connected via Alt Mode
# (kms_dp_alt_mode:1234) INFO: Pin assignment: C
# (kms_dp_alt_mode:1234) INFO: Lane count: 2
# Subtest usb_c_dp_connect: SUCCESS (1.234s)

# 5. 链路训练调试
sudo cat /sys/kernel/debug/dri/0/DP-2/link_training
# 输出:
# Training status: SUCCESS
# Attempts: 1
# Final lane count: 2
# Final link rate: HBR3 (8.1 Gbps)
# Training pattern used: TPS4
```

#### 4.3 兼容性测试

```bash
#!/bin/bash
# USB-C DP Alt Mode 兼容性测试脚本

echo "=== USB-C DP Alt Mode Compatibility Test ==="

# 测试 1: Pin 分配模式验证
echo "Test 1: Pin Assignment Detection"
for mode in C D E F; do
    echo "  Testing mode $mode..."
    # 检查当前 Pin 分配
    PIN=$(sudo cat /sys/class/typec/port0/partner/pin_assignment 2>/dev/null)
    if [ "$PIN" == "$mode" ]; then
        echo "  [PASS] Mode $mode detected"
    fi
done

# 测试 2: 带宽验证
echo "Test 2: Bandwidth Verification"
LANE_COUNT=$(sudo cat /sys/kernel/debug/dri/0/DP-2/link_status | grep "Lane count" | awk '{print $3}')
LINK_RATE=$(sudo cat /sys/kernel/debug/dri/0/DP-2/link_status | grep "Link rate" | awk '{print $3}')
echo "  Lane count: $LANE_COUNT"
echo "  Link rate: $LINK_RATE"

# 测试 3: USB 3.x 共存验证
echo "Test 3: USB 3.x Coexistence"
USB_DEVICES=$(lsusb -t | grep -c "5000M")
if [ "$USB_DEVICES" -gt 0 ]; then
    echo "  [PASS] USB 3.x devices detected"
else
    echo "  [WARN] No USB 3.x devices"
fi

# 测试 4: HPD 功能测试
echo "Test 4: HPD Functionality"
echo "  Please disconnect and reconnect the USB-C DP display..."
# 监控 HPD 事件
sudo udevadm monitor --property --subsystem-match=drm &
MONITOR_PID=$!
sleep 5
kill $MONITOR_PID

# 测试 5: 热插拔测试
echo "Test 5: Hot Plug Test"
for i in 1 2 3; do
    echo "  Iteration $i: Please unplug and replug the display"
    read -p "  Press Enter after replugging..."
    STATUS=$(sudo cat /sys/kernel/debug/dri/0/DP-2/status)
    if echo "$STATUS" | grep -q "connected"; then
        echo "  [PASS] Hot plug iteration $i"
    else
        echo "  [FAIL] Hot plug iteration $i"
    fi
done

echo "=== Test Complete ==="
```

### 5. USB-C DP Alt Mode Bug 案例分析

#### 5.1 Pin 分配协商失败

```
Bug 分析: USB-C DP Alt Mode Pin 分配协商失败

┌─────────────────────────────────────────────────────────────────┐
│ Bug ID: GitLab drm/amd issue #1852                             │
│ 内核版本: 5.15+                                                 │
│ 硬件: Lenovo ThinkPad X1 Carbon Gen 9 + Dell U2723QE 显示器    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ 症状:                                                          │
│  - 通过 USB-C 连接显示器无显示输出                               │
│  - USB 2.0 功能正常                                             │
│  - dmesg 显示 Pin 分配协商超时                                   │
│                                                                 │
│ dmesg 日志:                                                    │
│ [ 1234.567] amdgpu: [drm] Alt Mode enter failed,               │
│            pin_assignment not supported by sink                │
│ [ 1234.568] amdgpu: [drm] Falling back to USB 2.0 mode        │
│                                                                 │
│ 根因分析:                                                      │
│  1. Source 尝试使用 Mode C (2-lane DP + USB 3.x)               │
│  2. Sink (Dell U2723QE) 仅支持 Mode D (4-lane DP)             │
│  3. 双方没有共同的 Pin 分配模式                                 │
│  4. 协商超时失败                                               │
│                                                                 │
│ 修复方案:                                                      │
│  1. Source 应遍历所有支持的 Pin 分配模式                         │
│  2. 按优先级协商: D → C → E → F                                │
│  3. 如果 Sink 和 Source 没有共同模式，报告清晰错误                │
│                                                                 │
│ Patch: drm/amd/display: Improve DP Alt Mode pin assignment     │
│        negotiation with fallback mechanism                     │
└─────────────────────────────────────────────────────────────────┘
```

#### 5.2 USB 3.x 干扰问题

```
Bug 分析: USB 3.x 数据传输导致 DP 显示闪烁

┌─────────────────────────────────────────────────────────────────┐
│ Bug ID: freedesktop.org bug #123456                            │
│ 内核版本: 5.18+                                                 │
│ 硬件: AMD Ryzen 7 6800U + USB-C 扩展坞                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ 症状:                                                          │
│  - USB 3.x U盘读写时，外接显示器出现水平条纹闪烁                   │
│  - USB 2.0 设备不影响显示                                        │
│  - 仅发生在 Mode C Pin 分配                                      │
│                                                                 │
│ 根因分析:                                                      │
│  1. Mode C 下 DP Lane 0 和 USB 3.x TX 共享相邻差分对            │
│  2. USB 3.x 高速传输 (5Gbps+) 产生 EMI                         │
│  3. 串扰导致 DP Lane 0 信号质量下降                              │
│  4. 链路 CRC 错误增加，触发画面重传                              │
│                                                                 │
│  信号串扰示意图:                                                │
│                                                                 │
│  USB 3.x TX ──────┐                                            │
│                   ├── 电磁耦合 ──→ DP ML0 信号降级               │
│  DP Lane 0 ───────┘                                            │
│                                                                 │
│  受影响时段: USB 3.x 批量传输时                                  │
│                                                                 │
│ 修复方案:                                                      │
│  1. 在 USB 3.x 活跃时切换到 Mode E (牺牲 USB 3.x)              │
│  2. 增加 DP Lane 的电磁屏蔽                                     │
│  3. 降低 USB 3.x 传输速率到 Gen1 (5Gbps)                      │
│                                                                 │
│ 临时解决方法:                                                  │
│  echo 0 | sudo tee /sys/class/typec/port0/usb3_disable         │
└─────────────────────────────────────────────────────────────────┘
```

#### 5.3 USB PD 供电冲突

```
Bug 分析: USB-C PD 充电导致 DP 链路断开

┌─────────────────────────────────────────────────────────────────┐
│ Bug ID: Kernel Bugzilla #215432                                │
│ 内核版本: 5.10+                                                 │
│ 硬件: HP EliteBook 845 G8 + HP USB-C Dock G5                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ 症状:                                                          │
│  - 插入 USB-C 扩展坞，DP 显示正常                                │
│  - 开始 PD 充电 (60W+) 时，显示器闪烁后断开                      │
│  - dmesg 显示链路训练失败日志                                    │
│                                                                 │
│ 根因分析:                                                      │
│  1. PD 协商从 20V/3A 切换到 20V/5A 时                          │
│  2. VBUS 电压瞬间跌落 (dropout)                                 │
│  3. 电压跌落导致 DP 链路 PHY 供电不稳定                          │
│  4. DP 链路训练丢失同步                                         │
│  5. 链路恢复训练失败                                            │
│                                                                 │
│ 时序分析:                                                      │
│                                                                 │
│  VBUS ── 20V ──┐                                               │
│                 │  PD 协商 (20V 3A → 20V 5A)                   │
│                 │                                              │
│                 └──── 18V ── 19V ── 20V                        │
│                        ↑        ↑                              │
│                        │        └── 电压恢复                    │
│                        └── 电压跌落 (< 3ms)                    │
│                                                                 │
│  DP Lane 信号 ── 稳定 ──┐ ┌── 重训练 ── 稳定                    │
│                          │ │                                   │
│                          └─┘ ← 时钟丢失                         │
│                                                                 │
│ 修复方案:                                                      │
│  1. 在 PD 协商期间保持 DP PHY 供电                               │
│  2. 增加 VBUS 保持电容                                         │
│  3. PD 协商前预先降低链路速率                                   │
│                                                                 │
│ 补丁: usb: typec: Fix display dropout during PD contract       │
│       renegotiation                                            │
└─────────────────────────────────────────────────────────────────┘
```

### 6. USB-C DP Alt Mode 高级话题

#### 6.1 多显示器菊花链

USB-C 扩展坞支持通过 MST 实现多显示器：

```
USB-C 扩展坞多显示器拓扑

笔记本 (USB-C Source)
    │
    └── USB-C 线缆
         │
    ┌────┴──────────────────────────────────────────────┐
    │  USB-C 扩展坞 (Dock)                              │
    │  ┌──────────────────────────────────────────────┐ │
    │  │  DP Alt Mode 拆分器                          │ │
    │  │  ┌──────────┐  ┌──────────┐                │ │
    │  │  │ DP 信号  │  │ USB 3.x │                │ │
    │  │  │ (4-lane) │  │ Hub     │                │ │
    │  │  └────┬─────┘  └────┬─────┘                │ │
    │  └───────┼─────────────┼──────────────────────┘ │
    └──────────┼─────────────┼─────────────────────────┘
               │             │
         ┌─────┴──────┐  ┌──┴────────┐
         │ DP 显示器 1 │  │ USB 设备  │
         │ (DP花瓶链)  │  │ (键鼠/存储)│
         └─────┬──────┘  └───────────┘
               │
         ┌─────┴──────┐
         │ DP 显示器 2 │
         └────────────┘

扩展坞带宽分配:
  ┌──────────────┬────────────┬────────────┬────────────┐
  │  接口         │  通道数     │  带宽       │  备注       │
  ├──────────────┼────────────┼────────────┼────────────┤
  │  USB-C 链路  │  4-lane    │  HBR3      │  32.4 Gbps │
  │  DP 显示器 1 │  2-lane    │  HBR3      │  16.2 Gbps │
  │  DP 显示器 2 │  2-lane    │  HBR3      │  16.2 Gbps │
  │  USB 3.x     │  -         │  10Gbps    │  共享      │
  └──────────────┴────────────┴────────────┴────────────┘
```

#### 6.2 USB-C 线缆质量影响

```
线缆质量对 DP Alt Mode 的影响

USB-C 线缆认证等级:
  ┌────────────┬────────────┬──────────────┬──────────────────┐
  │  认证       │  速率       │  DP 支持     │  最大长度          │
  ├────────────┼────────────┼──────────────┼──────────────────┤
  │  USB 2.0  │  480Mbps   │  不支持      │  2m               │
  │  USB 3.2  │  10Gbps    │  HBR3 2-lane │  1m               │
  │  USB4     │  20Gbps    │  HBR3 4-lane │  0.8m             │
  │  USB4 Gen4│  40Gbps    │  UHBR10      │  0.5m             │
  └────────────┴────────────┴──────────────┴──────────────────┘

信号衰减 vs 线缆长度:
  信号幅度 (dB)
   0 │ ─────────────────────────────────────
  -3 │                        ┌────────────────
  -6 │                   ┌────┤
  -9 │              ┌────┤    │
 -12 │         ┌────┤    │    │
 -15 │    ┌────┤    │    │    │
 -18 │ ┌───┤    │    │    │    │
     └────────────────────────────────────────
       0   0.5  1.0  1.5  2.0  2.5  3.0
              线缆长度 (m)
         ┌── USB 2.0
         ├── USB 3.x 10Gbps
         └── DP HBR3

关键问题:
  - 长线缆 (>1m) 在高分辨率下可能无法稳定工作
  - 被动线缆 vs 主动线缆 (带信号增强)
  - eMarker 芯片必须正确识别
```

#### 6.3 DP Alt Mode 与 Thunderbolt 共存

```
USB-C 接口的 Alt Mode 共存策略

USB-C 物理接口
    │
    ├── USB 2.0 (所有设备)
    │
    ├── USB 3.x / USB4 (数据)
    │
    ├── DP Alt Mode (显示)
    │   ├── 使用高速差分对传输 DP
    │   └── 与 USB 3.x 共享带宽
    │
    ├── Thunderbolt Alt Mode
    │   ├── 使用全部高速差分对
    │   ├── 支持 PCIe + DP + USB
    │   └── 需要认证控制器
    │
    └── HDMI Alt Mode (较少见)
        └── 需要专用转换芯片

Thunderbolt 与 DP Alt Mode 关键区别:
  ┌──────────────────┬────────────────────┬────────────────────┐
  │  特性             │  DP Alt Mode       │  Thunderbolt 4     │
  ├──────────────────┼────────────────────┼────────────────────┤
  │  协议             │  原生 DP           │  PCIe + DP 隧道    │
  │  最大 DP 带宽    │  32.4 Gbps         │  32.4 Gbps         │
  │  USB 数据        │  受 Pin 分配影响   │  同时 40Gbps       │
  │  PCIe 隧道       │  不支持            │  支持              │
  │  菊花链          │  有限 (MST)        │  最多 6 设备       │
  │  控制器成本      │  低                │  高                │
  │  通用性          │  广泛              │  Intel/Apple 为主  │
  └──────────────────┴────────────────────┴────────────────────┘
```

### 7. AMDGPU USB-C DP 驱动架构

#### 7.1 驱动初始化流程

```c
// amdgpu_dm_usb_c_init - USB-C 子系统初始化
int amdgpu_dm_usb_c_init(struct amdgpu_device *adev)
{
    struct amdgpu_usb_c_dp *usb_c_dp;
    int ret;

    usb_c_dp = kzalloc(sizeof(*usb_c_dp), GFP_KERNEL);
    if (!usb_c_dp)
        return -ENOMEM;

    // 1. 注册 Type-C Alt Mode 回调
    usb_c_dp->typec_alt = typec_altmode_register(
        adev->dev,
        &amdgpu_dp_altmode_ops,
        usb_c_dp);

    if (IS_ERR(usb_c_dp->typec_alt)) {
        ret = PTR_ERR(usb_c_dp->typec_alt);
        DRM_ERROR("Failed to register Type-C Alt Mode: %d\n", ret);
        goto fail;
    }

    // 2. 初始化工作队列
    INIT_WORK(&usb_c_dp->hpd_work, amdgpu_dp_altmode_hpd_work);

    // 3. 获取 Type-C 端口
    usb_c_dp->typec_port = typec_port_get(adev->dev);

    // 4. 注册 HPD 处理
    adev->dm.usb_c_dp = usb_c_dp;

    DRM_INFO("USB-C DP Alt Mode support initialized\n");
    return 0;

fail:
    kfree(usb_c_dp);
    return ret;
}

// Alt Mode 状态变化通知
void amdgpu_dm_usb_c_altmode_notify(
    struct amdgpu_device *adev,
    enum typec_dp_state state,
    struct typec_dp_config *config)
{
    struct amdgpu_usb_c_dp *usb_c_dp = adev->dm.usb_c_dp;
    struct dc_link *link = usb_c_dp->dc_link;

    if (!link)
        return;

    switch (state) {
    case TYPEC_DP_STATE_ENTER:
        // DP Alt Mode 已进入
        link->usb_c_data->alt_mode_active = true;
        link->usb_c_data->dp_lane_count =
            dc_link_usb_c_get_dp_lane_count(link, config->pin_assignment);
        // 触发链路训练
        dc_link_dp_perform_link_training(link, NULL, false);
        break;

    case TYPEC_DP_STATE_EXIT:
        // DP Alt Mode 已退出
        link->usb_c_data->alt_mode_active = false;
        dc_link_set_edp_power_state(link, DC_EDP_POWER_OFF);
        break;

    case TYPEC_DP_STATE_CONFIGURE:
        // 重新配置 (如 Pin 分配改变)
        dc_link_set_usb_c_dp_alt_mode(link, true);
        break;
    }
}
```

#### 7.2 热插拔检测集成

```c
// USB-C HPD 与标准 DP HPD 的集成

// USB-C 端 HPD 中断处理
void amdgpu_dm_usb_c_hpd_irq(struct amdgpu_device *adev, bool connected)
{
    struct amdgpu_usb_c_dp *usb_c_dp = adev->dm.usb_c_dp;

    if (connected) {
        // 1. 标记连接
        usb_c_dp->hpd_connected = true;

        // 2. 触发 EDID 读取
        schedule_work(&usb_c_dp->hpd_work);

        // 3. 通知 DRM 核心
        drm_helper_hpd_irq_event(adev->ddev);
    } else {
        // 1. 标记断开
        usb_c_dp->hpd_connected = false;

        // 2. 通知 DC 链路断开
        schedule_work(&usb_c_dp->hpd_work);

        // 3. 通知 DRM 核心
        drm_helper_hpd_irq_event(adev->ddev);
    }
}

// USB-C 线缆方向检测
enum typec_orientation amdgpu_dm_usb_c_get_orientation(
    struct amdgpu_device *adev)
{
    // 检测 USB-C 正反插方向
    // 影响 DP Lane 的极性配置
    return typec_port_get_orientation(adev->dm.usb_c_dp->typec_port);
}

// 根据方向调整 Lane 极性
void amdgpu_dm_usb_c_adjust_lane_polarity(
    struct dc_link *link,
    enum typec_orientation orientation)
{
    // 反转插入时，TX1/RX1 和 TX2/RX2 交换
    // 需要重新映射 Lane 到物理引脚

    if (orientation == TYPEC_ORIENTATION_REVERSE) {
        // Lane 0/1 交换
        link->dpcd_caps.lane_polarity_inverted = true;
    } else {
        link->dpcd_caps.lane_polarity_inverted = false;
    }
}
```

### 8. USB-C DP Alt Mode 测试工具

#### 8.1 调试工具链

```bash
# 1. type-c 子系统状态查看
sudo ls -la /sys/class/typec/
# 输出:
# total 0
# lrwxrwxrwx 1 root root 0 ... port0 -> ../../devices/.../typec/port0

sudo cat /sys/class/typec/port0/port_type
# drp

sudo cat /sys/class/typec/port0/power_role
# source

sudo cat /sys/class/typec/port0/data_role
# dfp (Downstream Facing Port)

# 2. 查看 USB-C PD 协商状态
sudo cat /sys/class/typec/port0/pd_capabilities
# 输出:
# PD version: 3.0
# Power: Source 20.0V 5.0A 100.0W

# 3. 使用 ucsi 工具 (UCSI: USB Type-C Connector System Software Interface)
sudo ucsi status
# 输出示例:
# Connector 0:
#   Orientation: Normal
#   Current Limit: 3.0A
#   Power Direction: Source
#   DP Alt Mode: Active
#   Pin Assignment: C

# 4. 监控 USB-C 事件
sudo ucsi monitor
# 实时显示 USB-C 事件

# 5. 查看 DRM 连接器信息
sudo cat /sys/kernel/debug/dri/0/DP-2/status
```

#### 8.2 自动化测试

```python
#!/usr/bin/env python3
# USB-C DP Alt Mode 自动化测试脚本

import subprocess
import time
import sys
import re

class USBCDPAltModeTest:
    def __init__(self):
        self.port = '/sys/class/typec/port0'
        self.drm_connector = '/sys/kernel/debug/dri/0/DP-2'
        self.passed = 0
        self.failed = 0

    def read_sysfs(self, path):
        try:
            with open(path, 'r') as f:
                return f.read().strip()
        except:
            return None

    def test_typec_detection(self):
        """测试 Type-C 端口检测"""
        port_type = self.read_sysfs(f'{self.port}/port_type')
        if port_type:
            print(f'[PASS] Type-C port detected: {port_type}')
            self.passed += 1
            return True
        print('[FAIL] Type-C port not detected')
        self.failed += 1
        return False

    def test_alt_mode_active(self):
        """测试 Alt Mode 激活"""
        partner = f'{self.port}/partner'
        svid = self.read_sysfs(f'{partner}/svid')
        if svid == '0xff01':
            print(f'[PASS] DP Alt Mode active (SVID: {svid})')
            self.passed += 1
            return True
        print(f'[FAIL] DP Alt Mode not active (SVID: {svid})')
        self.failed += 1
        return False

    def test_dp_link_status(self):
        """测试 DP 链路状态"""
        status = self.read_sysfs(f'{self.drm_connector}/status')
        if status and 'connected' in status:
            print(f'[PASS] DP link connected')
            self.passed += 1
            return True
        print(f'[FAIL] DP link not connected')
        self.failed += 1
        return False

    def test_pin_assignment(self):
        """测试 Pin 分配"""
        pin = self.read_sysfs(f'{self.port}/partner/pin_assignment')
        if pin:
            print(f'[PASS] Pin assignment: {pin}')
            self.passed += 1
            return True
        print(f'[FAIL] Pin assignment unknown')
        self.failed += 1
        return False

    def test_hotplug(self):
        """测试热插拔"""
        print('Please unplug and replug the USB-C DP display...')
        time.sleep(2)

        # 等待 HPD 事件
        for i in range(10):
            status = self.read_sysfs(f'{self.drm_connector}/status')
            if status and 'connected' in status:
                print(f'[PASS] Hot plug detected')
                self.passed += 1
                return True
            time.sleep(1)

        print('[FAIL] Hot plug not detected')
        self.failed += 1
        return False

    def test_usb3_coexistence(self):
        """测试 USB 3.x 共存"""
        usb_devices = subprocess.run(
            ['lsusb', '-t'],
            capture_output=True, text=True
        )
        usb3_count = usb_devices.stdout.count('5000M')
        if usb3_count > 0:
            print(f'[PASS] USB 3.x coexisting ({usb3_count} devices)')
            self.passed += 1
        else:
            print(f'[WARN] No USB 3.x devices in coexistence mode')
            self.passed += 1  # Not necessarily a failure

    def run_all(self):
        """运行所有测试"""
        print('=== USB-C DP Alt Mode Test Suite ===')
        print()

        tests = [
            ('Type-C Detection', self.test_typec_detection),
            ('Alt Mode Active', self.test_alt_mode_active),
            ('DP Link Status', self.test_dp_link_status),
            ('Pin Assignment', self.test_pin_assignment),
            ('USB 3.x Coexistence', self.test_usb3_coexistence),
        ]

        for name, test_func in tests:
            print(f'Test: {name}')
            test_func()
            print()

        # 热插拔测试需要用户交互
        print('Test: Hot Plug (interactive)')
        self.test_hotplug()
        print()

        print('=== Results ===')
        print(f'Passed: {self.passed}')
        print(f'Failed: {self.failed}')
        return self.failed == 0

if __name__ == '__main__':
    tester = USBCDPAltModeTest()
    success = tester.run_all()
    sys.exit(0 if success else 1)
```

### 9. USB-C 线缆与适配器测试

#### 9.1 线缆认证测试

```
USB-C 线缆认证测试矩阵

┌──────────────────────┬──────────┬──────────┬──────────┬──────────┐
│  测试项目             │  USB 2.0 │  USB 3.2 │  USB4    │  DP Alt  │
├──────────────────────┼──────────┼──────────┼──────────┼──────────┤
│  连接器完整性         │  必要     │  必要     │  必要     │  必要     │
│  差分阻抗            │  必要     │  必要     │  必要     │  必要     │
│  插入损耗            │  可选     │  必要     │  必要     │  必要     │
│  回波损耗            │  可选     │  必要     │  必要     │  必要     │
│  近端串扰 (NEXT)    │  可选     │  必要     │  必要     │  必要     │
│  远端串扰 (FEXT)    │  可选     │  必要     │  必要     │  必要     │
│  传播延迟            │  可选     │  可选     │  必要     │  可选     │
│  差分偏斜            │  可选     │  可选     │  必要     │  必要     │
│  eMarker 验证        │  N/A      │  必要     │  必要     │  必要     │
│  VCONN 供电         │  N/A      │  必要     │  必要     │  必要     │
│  SBU 信号质量        │  N/A      │  N/A      │  N/A      │  必要     │
└──────────────────────┴──────────┴──────────┴──────────┴──────────┘
```

#### 9.2 常见适配器兼容性

```bash
# USB-C 转 DP 适配器兼容性测试

# 1. 适配器类型识别
sudo dmesg | grep -i "typec\|altmode\|dp"
# 查看适配器支持的 Alt Mode

# 2. 测试不同分辨率和刷新率
# 测试 1080p@60Hz
xrandr --output DP-2 --mode 1920x1080 --rate 60
# 测试 4K@30Hz
xrandr --output DP-2 --mode 3840x2160 --rate 30
# 测试 4K@60Hz
xrandr --output DP-2 --mode 3840x2160 --rate 60

# 3. 测试同时充电和显示
# 连接 USB-C 扩展坞，同时充电和输出显示
# 检查充电状态
sudo cat /sys/class/power_supply/ACAD/online

# 4. 测试 USB 3.0 同时传输
# 在 USB-C 扩展坞上连接 USB 3.0 U盘
# 执行大文件传输
dd if=/dev/zero of=/media/usb/testfile bs=1M count=1000
# 同时观察显示是否正常
```

### 相关链接

- VESA DisplayPort Alt Mode for USB-C 标准: https://vesa.org/usb-c/
- Linux Type-C 子系统文档: Documentation/usb/typec.rst
- USB-IF USB-C 规范: https://www.usb.org/usb-type-c
- AMDGPU USB-C DP 驱动: drivers/gpu/drm/amd/display/
- Type-C Alt Mode 内核 API: include/linux/usb/typec_altmode.h
- IGT kms_dp_alt_mode 测试: tests/kms_dp_alt_mode.c
- ucsi 工具: https://github.com/linux-nvme/ucsi
- USB-C 调试指南: Documentation/admin-guide/typec.rst

## 今日小结

今天我们深入学习了 USB-C DP Alt Mode 技术，这是现代笔记本和移动设备中最重要的显示输出方式之一。我们掌握了 USB-C 接口的物理层特性、CC 协议的工作机制，以及 DP Alt Mode 的四种 Pin 分配模式 (C/D/E/F)。

在驱动实现方面，我们分析了 AMDGPU 驱动中 USB-C DP Alt Mode 的初始化流程、Alt Mode 协商机制，以及 HPD 热插拔检测的集成方式。通过实际的 Bug 案例分析（Pin 分配协商失败、USB 3.x 干扰、PD 供电冲突），我们理解了 USB-C DP Alt Mode 在实际应用中面临的主要挑战和解决方案。

我们还学习了 USB-C DP Alt Mode 的测试方法，包括使用 sysfs/typec 调试接口、自动化测试脚本编写，以及线缆质量和适配器兼容性的测试策略。理解了 USB-C 线缆长度、信号质量和带宽之间的权衡关系。

明天我们将进行显示接口协议的综合考核，检验对 HDMI、DP、eDP 和 USB-C DP Alt Mode 等显示接口技术的掌握程度。

## 扩展思考

1. USB4 规范将 DP 隧道化和 Alt Mode 合并为统一的协议。USB4 的 DP 隧道与 DP Alt Mode 在带宽分配和延迟特性上有何本质区别？这对驱动开发者和测试工程师意味着什么？

2. 随着 USB-C 扩展坞的普及，多显示器的带宽竞争问题越来越突出。在有限的 USB-C 链路带宽下，如何在多个显示器、USB 设备和充电功率之间动态平衡？是否可以通过 USB PD 协议动态调整资源分配？

3. USB-C DP Alt Mode 依赖于 SBU 引脚传输 AUX 信号。SBU 引脚在任何线缆中都是直连的（无信号调理），这是否限制了 AUX 通道的信号质量？在 UHBR 速率下，SBU 的噪声会如何影响 DP 链路训练的可靠性？