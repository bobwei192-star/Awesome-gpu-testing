# 第313天：amdgpu_acpi.c：ACPI接口与GPU电源事件

## 学习目标

- 深入理解 ACPI（Advanced Configuration and Power Interface）标准与 GPU 电源管理的关系
- 掌握 AMDGPU 驱动中 `amdgpu_acpi.c` 文件的结构和主要功能
- 熟悉 ACPI 电源事件（如 AC/DC 切换、亮度变化、热事件）的处理流程
- 理解 _DSM（Device Specific Method）、_DSM 在 AMDGPU 中的应用
- 掌握 ACPI 平台配置文件（ASL/DSDT）与 GPU 电源管理的交互
- 能够分析和调试 ACPI 相关的 GPU 电源管理问题
- 了解系统级电源状态（S0/S3/S4/S5/S0ix）与 GPU 状态的对应关系

## 知识详解

### 1. 概念原理

#### 1.1 ACPI 标准概述

ACPI（Advanced Configuration and Power Interface）是由 Intel、Microsoft、Toshiba、HP、Phoenix 等公司联合制定的开放行业标准，定义了操作系统如何管理硬件配置和电源。

**ACPI 的核心概念：**

| 概念 | 描述 | 作用 |
|------|------|------|
| **ACPI 表** | 系统固件提供的数据结构 | 描述硬件配置和电源能力 |
| **DSDT**（Differentiated System Description Table） | 主要的 ACPI 描述表 | 包含系统设备的 AML 代码 |
| **SSDT**（Secondary System Description Table） | 辅助描述表 | 补充 DSDT，描述动态设备 |
| **AML**（ACPI Machine Language） | ACPI 虚拟机字节码 | 固件执行的平台特定代码 |
| **ASL**（ACPI Source Language） | AML 的高级语言 | 人类可读的 ACPI 源码 |
| **GPE**（General Purpose Event） | 通用目的事件 | 硬件触发的 ACPI 事件 |
| **_DSM**（Device Specific Method） | 设备特定方法 | 厂商自定义的 ACPI 方法 |

**ACPI 电源状态：**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      ACPI System Power States (S-States)               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                          S0 (Working)                           │   │
│  │  系统完全运行，所有设备正常工作                                  │   │
│  │  CPU 在 C0（活动）/ C1/C2/C3（空闲）状态间切换                  │   │
│  │  GPU 在 D0（活动）状态                                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│    ┌──────────────────────────────┼──────────────────────────────┐     │
│    │                              │                              │     │
│    ▼                              ▼                              ▼     │
│  ┌──────────┐  ┌──────────────────────────┐  ┌──────────────────┐   │
│  │ S0ix     │  │ S3 (Suspend to RAM)      │  │ S4 (Hibernate)  │   │
│  │ (Modern  │  │  内存保持供电             │  │  内存写入磁盘   │   │
│  │ Standby) │  │  CPU 断电                 │  │  完全断电       │   │
│  │  低功耗  │  │  恢复时间：~1-5 秒       │  │  恢复时间：~10s+ │   │
│  │  待机    │  │                          │  │                  │   │
│  └──────────┘  └──────────────────────────┘  └──────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                       S5 (Soft Off)                             │   │
│  │  系统完全关闭，只有 RTC 保持运行                                 │   │
│  │  需要完整启动流程                                               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

**GPU 设备电源状态（D-States）：**

| D-State | 名称 | 描述 | 功耗 | 恢复延迟 |
|---------|------|------|------|----------|
| **D0** | 活动状态 | GPU 完全运行，所有引擎可用 | 最高 | 无 |
| **D1** | 低功耗状态1 | 部分引擎断电，上下文保留 | 中低 | 短 |
| **D2** | 低功耗状态2 | 更多引擎断电 | 更低 | 较长 |
| **D3hot** | 热待机 | 大部分硬件断电，PCIe 链路保持 | 很低 | 较长 |
| **D3cold** | 冷关机 | 设备完全断电 | 零 | 很长（需重新枚举） |

#### 1.2 amdgpu_acpi.c 文件概述

`amdgpu_acpi.c` 是 AMDGPU 驱动中处理 ACPI 接口的核心文件，位于 Linux 内核源码 `drivers/gpu/drm/amd/amdgpu/amdgpu_acpi.c`。

**文件主要功能：**

1. **ACPI 设备枚举与识别**：识别系统中的 AMD GPU ACPI 设备
2. **电源事件处理**：处理 ACPI 通知事件（如亮度变化、热事件）
3. **_DSM 方法调用**：执行 AMD 特定的 _DSM 方法
4. **平台配置获取**：从 ACPI 表获取平台特定的 GPU 配置
5. **背光控制**：处理笔记本电脑的 GPU 背光控制
6. **电源状态协调**：协调 GPU 电源状态与系统 ACPI 状态

**amdgpu_acpi.c 核心数据结构：**

```c
struct amdgpu_acpi_bios {
    u8 *bios;
    u32 bios_size;
};

struct amdgpu_atif {
    struct amdgpu_device *adev;
    struct acpi_device *adev_acpi;
    struct backlight_device *backlight;
    
    /* ATIF (AMD Technology Interface) */
    u8 atif_verified;
    u8 atif_functions_supported[8];
    u16 atif_version;
    u32 atif_function_mask;
    
    /* 系统参数 */
    u32 system_params[6];
};

struct amdgpu_atcs {
    struct acpi_device *adev_acpi;
    
    /* ATCS (AMD Technology Control Services) */
    u8 atcs_verified;
    u16 atcs_version;
    u32 atcs_function_mask;
};

struct amdgpu_acpi {
    struct amdgpu_atif atif;
    struct amdgpu_atcs atcs;
    struct amdgpu_acpi_bios acpi_bios;
    bool has_acpi_bios;
};
```

**AMDGPU ACPI 子系统架构图：**

```
┌─────────────────────────────────────────────────────────────────────────┐
│               AMDGPU ACPI Subsystem Architecture                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    User Space / System Daemons                  │   │
│  │  (systemd-logind, upower, GNOME/KDE Power Management)           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │              Linux Kernel ACPI Core Subsystem                   │   │
│  │  (acpi_bus, acpi_event, acpi_video, acpi_thermal)               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│            ┌──────────────────────┼──────────────────────┐              │
│            │                      │                      │              │
│            ▼                      ▼                      ▼              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │    acpi_video    │  │    acpi_thermal  │  │    acpi_battery   │   │
│  │  (亮度控制)      │  │    (热管理)      │  │    (电池管理)    │   │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘   │
│           │                      │                      │              │
│           └──────────────────────┼──────────────────────┘              │
│                                  ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                  amdgpu_acpi.c (本文件)                         │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                │   │
│  │  │   ATIF      │ │   ATCS      │ │   _DSM      │                │   │
│  │  │ (Tech IF)  │ │ (Ctrl Svc)  │ │ (Device    │                │   │
│  │  │             │ │             │ │  Specific) │                │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    AMDGPU Driver Core                           │   │
│  │  (amdgpu_pm.c, amdgpu_dpm.c, amdgpu_device.c)                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Hardware / Firmware                          │   │
│  │  (GPU, SMU, PSP, Display Controller, ACPI AML)                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 1.3 ATIF（AMD Technology Interface）

ATIF 是 AMD 定义的 ACPI 接口，用于系统固件（BIOS/UEFI）与 AMD GPU 驱动之间的通信。

**ATIF 功能分类：**

| 功能组 | 功能码 | 描述 |
|--------|--------|------|
| **系统信息** | 0x00 | 获取支持的功能列表 |
| **系统参数** | 0x01 | 获取系统参数（如显示类型、面板信息） |
| **亮度控制** | 0x10 | 控制背光亮度 |
| **亮度请求** | 0x11 | 处理 BIOS 发起的亮度变化请求 |
| **电源来源** | 0x20 | 通知 AC/DC 电源状态变化 |
| **显示切换** | 0x21 | 处理显示设备切换请求 |
| **扩展显示** | 0x22 | 扩展显示信息获取 |
| **热事件** | 0x30 | 热事件通知 |
| **GPU 信息** | 0x40 | 获取 GPU 状态信息 |
| **供电方案** | 0x50 | 获取供电方案（Performance/Power Saving） |

**ATIF 调用流程图：**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ATIF Call Flow (Example: AC/DC Switch)              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────┐                                                           │
│  │  硬件事件  │ ──────────────────────────────────────────────────►     │
│  │ AC/DC 切换 │                                                         │
│  └────┬─────┘                                                          │
│       │                                                                │
│       ▼                                                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      系统固件 (BIOS/UEFI)                        │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  AML 代码检测事件，触发 GPE (General Purpose Event)       │   │   │
│  │  │  调用 Notify(GFX0, 0x81) 通知 GPU 设备                   │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Linux ACPI Core                              │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  接收 Notify 事件，查找对应的 ACPI 驱动                  │   │   │
│  │  │  调用 amdgpu_acpi_notify() 回调函数                     │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                  amdgpu_acpi.c                                  │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  amdgpu_acpi_notify():                                  │   │   │
│  │  │  - 检查通知类型 (0x81 = 电源状态变化)                    │   │   │
│  │  │  - 调用 amdgpu_acpi_get_params() 获取当前参数           │   │   │
│  │  │  - 通过 ATIF _DSM(Function 0x50) 获取供电方案          │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                  amdgpu_pm.c / amdgpu_dpm.c                     │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  根据供电方案调整 DPM 状态：                              │   │   │
│  │  │  - AC 电源 → Performance 模式                           │   │   │
│  │  │  - DC 电源 → Power Saving 模式                          │   │   │
│  │  │  - 调整 SCLK/MCLK/FCLK/Power Limit                      │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      GPU 硬件 / SMU 固件                        │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  应用新的电源管理配置                                     │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 1.4 ATCS（AMD Technology Control Services）

ATCS 是另一个 AMD 定义的 ACPI 接口，用于控制和管理 GPU 的各种功能。

**ATCS 功能：**

| 功能码 | 功能 | 描述 |
|--------|------|------|
| 0x00 | 获取支持的功能 | 查询 ATCS 支持的功能列表 |
| 0x01 | 验证接口 | 验证 ATCS 接口版本和兼容性 |
| 0x02 | 获取 GPU 信息 | 获取 GPU 状态和配置信息 |
| 0x03 | 设置 GPU 功率 | 设置 GPU 功率限制 |
| 0x04 | 风扇控制 | 控制 GPU 风扇转速 |
| 0x05 | 温度阈值 | 设置温度阈值 |
| 0x06 | 性能模式 | 选择性能模式 |

#### 1.5 ACPI _DSM 方法

_DSM（Device Specific Method）是 ACPI 标准中定义的设备特定方法，允许厂商定义自定义的 ACPI 方法。

**AMD GPU _DSM 方法：**

```
_DSM Arguments:
  Arg0: UUID (Universally Unique Identifier)
        AMD ATIF UUID: "3abf6f2d-71c4-462a-9a72-0f139a17795a"
        AMD ATCS UUID: "bb2cf106-4076-461e-9060-682f35a7a907"
  
  Arg1: Revision ID (通常为 0)
  
  Arg2: Function Index
        ATIF:
          0:  Query Supported Functions
          1:  System Parameters
          16: Brightness Control
          17: Brightness Request
          32: Power Source
          ...
        ATCS:
          0:  Query Supported Functions
          1:  Validate Interface
          2:  Get GPU Information
          3:  Set GPU Power
          ...
  
  Arg3: Function-specific Arguments (Package)
```

**_DSM 调用示例（伪代码）：**

```asl
// ASL 代码示例
Method (_DSM, 4, NotSerialized)
{
    // 检查 UUID
    If (Arg0 == ToUUID("3abf6f2d-71c4-462a-9a72-0f139a17795a"))
    {
        // ATIF UUID - AMD Technology Interface
        
        If (Arg2 == 0)  // Query Supported Functions
        {
            // 返回支持的功能位图
            Return (Buffer() { 0xFF, 0xFF, 0xFF, 0xFF,
                             0xFF, 0xFF, 0xFF, 0xFF })
        }
        ElseIf (Arg2 == 1)  // System Parameters
        {
            // 返回系统参数
            Return (Package(6) {
                0x00000001,  // Display Type
                0x00000000,  // Reserved
                0x00000000,  // Reserved
                0x00000000,  // Reserved
                0x00000000,  // Reserved
                0x00000000   // Reserved
            })
        }
        ElseIf (Arg2 == 32)  // Power Source Notification
        {
            // 处理电源来源变化
            Local0 = DeRefOf(Arg3[0])
            If (Local0 == 0)  // DC (Battery)
            {
                // 设置省电模式
            }
            ElseIf (Local0 == 1)  // AC
            {
                // 设置性能模式
            }
            Return (0)
        }
    }
    ElseIf (Arg0 == ToUUID("bb2cf106-4076-461e-9060-682f35a7a907"))
    {
        // ATCS UUID - AMD Technology Control Services
        If (Arg2 == 0)  // Query Supported Functions
        {
            Return (Buffer(4) { 0xFF, 0xFF, 0xFF, 0xFF })
        }
        ElseIf (Arg2 == 2)  // Get GPU Information
        {
            Return (Package(4) {
                0x12345678,  // GPU ID
                0x00000100,  // Revision
                0x00000001,  // State
                0x00000000   // Reserved
            })
        }
    }
    
    // 不支持的 UUID
    Return (Buffer() { 0x00 })
}
```

#### 1.6 ACPI 通知事件

ACPI 通知事件是硬件向操作系统发送的异步事件通知。

**GPU 相关的 ACPI 通知：**

| 通知值 | 名称 | 描述 |
|--------|------|------|
| 0x00 | Bus Check | 设备插入/移除 |
| 0x01 | Device Check | 设备状态变化 |
| 0x80 | Device Specific Notification | 设备特定通知 |
| 0x81 | Power Source Change | 电源来源变化（AC/DC） |
| 0x82 | Thermal Event | 热事件 |
| 0x83 | Brightness Change | 亮度变化 |
| 0x84 | Display Switch | 显示切换 |
| 0x85 | Mode Change | 模式变化 |

**通知事件处理流程：**

```
┌─────────────────────────────────────────────────────────────────────────┐
│              ACPI Notification Event Processing                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  硬件层                                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  硬件事件发生（如 AC 适配器插入、热传感器触发）                   │   │
│  │  触发 GPE（General Purpose Event）                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│  固件层                                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  GPE 处理方法（_Lxx/_Exx）执行 AML 代码                           │   │
│  │  调用 Notify(Device, NotificationValue)                         │   │
│  │  例如：Notify(\_SB.PCI0.GFX0, 0x81)  // 电源变化                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│  内核 ACPI 层                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  acpi_ev_gpe_dispatch(): 分发 GPE 事件                           │   │
│  │  acpi_ev_notify_dispatch(): 分发 Notify 事件                     │   │
│  │  调用设备驱动的 notify 回调函数                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│  AMDGPU 驱动层                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  amdgpu_acpi_notify(struct acpi_device *device, u32 value)      │   │
│  │  {                                                               │   │
│  │      switch (value) {                                            │   │
│  │      case 0x81:  // Power Source Change                         │   │
│  │          handle_power_source_change();                           │   │
│  │          break;                                                  │   │
│  │      case 0x82:  // Thermal Event                               │   │
│  │          handle_thermal_event();                                 │   │
│  │          break;                                                  │   │
│  │      case 0x83:  // Brightness Change                           │   │
│  │          handle_brightness_change();                             │   │
│  │          break;                                                  │   │
│  │      default:                                                    │   │
│  │          /* 其他通知类型 */                                      │   │
│  │          break;                                                  │   │
│  │      }                                                           │   │
│  │  }                                                               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 1.7 背光控制（Backlight Control）

笔记本电脑的显示背光控制是 ACPI 与 GPU 交互的重要场景。

**背光控制架构：**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 Backlight Control Architecture                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  用户空间                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  用户操作：Fn+F5/F6、亮度滑块、电源管理设置                       │   │
│  │  工具：xbacklight, brightnessctl, GNOME/KDE 设置                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  /sys/class/backlight/                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  多个可能的背光接口：                                             │   │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────┐ │   │
│  │  │  acpi_video0    │  │  amdgpu_bl0      │  │  intel_backli  │ │   │
│  │  │  (ACPI 标准)    │  │  (AMDGPU 原生)   │  │  ght (Intel)   │ │   │
│  │  │                 │  │                 │  │                │ │   │
│  │  │  brightness     │  │  brightness     │  │  brightness    │ │   │
│  │  │  max_brightness │  │  max_brightness │  │  max_brightne… │ │   │
│  │  │  actual_bright… │  │  actual_bright… │  │  actual_brigh… │ │   │
│  │  └──────────────────┘  └──────────────────┘  └────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│  内核层                                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  ┌──────────────────┐  ┌──────────────────┐                      │   │
│  │  │  acpi_video.c    │  │  amdgpu_acpi.c   │                      │   │
│  │  │  (ACPI 标准)    │  │  (ATIF 接口)      │                      │   │
│  │  │  _BQC/_BCM 方法  │  │  ATIF Function 16 │                      │   │
│  │  └──────────────────┘  └──────────────────┘                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│  硬件层                                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  ┌──────────────────┐  ┌──────────────────┐                      │   │
│  │  │  EC (Embedded   │  │  GPU / APU        │                      │   │
│  │  │      Controller)│  │  PWM Controller  │                      │   │
│  │  │  嵌入式控制器   │  │  脉冲宽度调制器   │                      │   │
│  │  └──────────────────┘  └──────────────────┘                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  显示面板                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  LED 背光 PWM 控制                                               │   │
│  │  亮度 = (PWM 占空比 / 最大占空比) × 100%                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 1.8 ACPI 与系统挂起/恢复

ACPI 在系统挂起（Suspend）和恢复（Resume）过程中起着核心作用。

**系统挂起流程（Suspend to RAM - S3）：**

```
┌─────────────────────────────────────────────────────────────────────────┐
│              System Suspend Flow (S3) with AMDGPU                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  触发挂起（用户操作 / systemd / lid close 等）                          │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  用户空间：                                                      │   │
│  │  - systemd-logind 接收请求                                      │   │
│  │  - 通知所有会话冻结                                              │   │
│  │  - 调用内核接口                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  内核层（设备驱动挂起阶段）：                                    │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  1. 用户态进程冻结 (freeze_processes())                  │   │   │
│  │  │  2. 设备驱动 suspend() 回调依次调用                      │   │   │
│  │  │     └─► amdgpu_pmops_suspend()                           │   │   │
│  │  │         ├─► 保存 GPU 寄存器状态                          │   │   │
│  │  │         ├─► 停止所有 ring buffer                         │   │   │
│  │  │         ├─► 卸载固件                                      │   │   │
│  │  │         ├─► 关闭显示输出                                  │   │   │
│  │  │         ├─► 调用 SMU 进入低功耗状态                       │   │   │
│  │  │         └─► GPU 进入 D3hot 状态                          │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  ACPI 层：                                                      │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  1. 调用 _PTS (Prepare To Sleep) 方法                    │   │   │
│  │  │  2. 设置睡眠类型控制寄存器                                │   │   │
│  │  │  3. 调用 _GTS (Going To Sleep) 方法                      │   │   │
│  │  │  4. 保存 CPU 上下文                                      │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  硬件层：                                                        │   │
│  │  - CPU 进入 S3 睡眠状态                                         │   │
│  │  - 内存保持自刷新（Self-Refresh）                               │   │
│  │  - 大部分硬件断电                                               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

**系统恢复流程（Resume from S3）：**

```
┌─────────────────────────────────────────────────────────────────────────┐
│              System Resume Flow (from S3) with AMDGPU                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  唤醒事件（电源键、键盘、鼠标、定时器、LID 打开等）                      │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  硬件层：                                                        │   │
│  │  - 唤醒事件触发                                                  │   │
│  │  - 系统固件恢复执行                                              │   │
│  │  - POST 部分执行                                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  ACPI 层：                                                      │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  1. 调用 _WAK (Wake) 方法                                 │   │   │
│  │  │  2. 恢复 CPU 上下文                                       │   │   │
│  │  │  3. 恢复系统状态                                          │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  内核层（设备驱动恢复阶段）：                                    │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  设备驱动 resume() 回调依次调用                          │   │   │
│  │  │  └─► amdgpu_pmops_resume()                              │   │   │
│  │  │         ├─► 重新初始化 GPU 硬件                          │   │   │
│  │  │         ├─► 重新加载固件                                  │   │   │
│  │  │         ├─► 恢复寄存器状态                                │   │   │
│  │  │         ├─► 重新启动 ring buffer                        │   │   │
│  │  │         ├─► 恢复显示输出                                  │   │   │
│  │  │         └─► 恢复 DPM 状态                                │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  用户空间：                                                      │   │
│  │  - 解冻用户态进程                                                │   │
│  │  - 恢复图形会话                                                  │   │
│  │  - 通知应用程序恢复                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2. 实践操作

#### 2.1 检查 AMDGPU ACPI 状态

**检查系统 ACPI 表：**

```bash
# 检查 ACPI 表
ls /sys/firmware/acpi/tables/

# 查看 DSDT 表信息
cat /sys/firmware/acpi/tables/DSDT | hexdump -C | head -20

# 使用 acpidump 导出所有 ACPI 表（需要 acpica-tools）
sudo acpidump -b

# 反汇编 DSDT 表
iasl -d DSDT.dat

# 查看反汇编的 DSDT
less DSDT.dsl
```

**检查 AMDGPU ACPI 设备：**

```bash
# 查找 GPU 的 ACPI 路径
ls -la /sys/bus/acpi/devices/ | grep -i vga

# 或通过 PCI 设备查找
lspci | grep -i vga

# 检查 GPU 的 ACPI 资源
ls -la /sys/class/drm/card0/device/ | grep acpi

# 查看 ACPI 设备信息
cat /sys/class/drm/card0/device/modalias
```

**检查背光控制接口：**

```bash
# 查看所有背光接口
ls -la /sys/class/backlight/

# 查看 AMDGPU 背光接口详情
ls -la /sys/class/backlight/amdgpu_bl0/

# 查看当前亮度
cat /sys/class/backlight/amdgpu_bl0/brightness

# 查看最大亮度
cat /sys/class/backlight/amdgpu_bl0/max_brightness

# 查看实际亮度
cat /sys/class/backlight/amdgpu_bl0/actual_brightness
```

**检查 ACPI 事件：**

```bash
# 监控 ACPI 事件
sudo acpi_listen

# 或使用 systemd-logind 监控
dbus-monitor --system "type=signal,interface=org.freedesktop.login1.Manager"
```

#### 2.2 分析 amdgpu_acpi.c 源码

让我们查看 Linux 内核中 amdgpu_acpi.c 的关键部分。首先，检查您的系统是否有内核源码。

```bash
# 查找内核源码位置
find /usr/src -name "amdgpu_acpi.c" 2>/dev/null

# 或查看当前运行内核的配置
zcat /proc/config.gz | grep AMDGPU
```

**amdgpu_acpi.c 关键函数分析（基于 Linux 内核 6.x）：**

```c
// 初始化 ATIF 接口
static int amdgpu_acpi_atif_init(struct amdgpu_device *adev)
{
    struct amdgpu_atif *atif = &adev->acpi.priv->atif;
    acpi_handle handle;
    union acpi_object *params;
    
    // 获取 GPU 设备的 ACPI handle
    handle = ACPI_HANDLE(&adev->pdev->dev);
    if (!handle)
        return -ENODEV;
    
    atif->adev = adev;
    atif->adev_acpi = acpi_bus_get_device(handle);
    
    // 验证 ATIF 接口
    // 调用 _DSM 方法检查 ATIF 支持
    // UUID: 3abf6f2d-71c4-462a-9a72-0f139a17795a
    
    return 0;
}

// 处理 ACPI 通知事件
static void amdgpu_acpi_notify(struct acpi_device *device, u32 value)
{
    struct amdgpu_device *adev = acpi_driver_data(device);
    struct amdgpu_atif *atif = &adev->acpi.priv->atif;
    
    switch (value) {
    case 0x81:  // Power Source Change
        dev_info(&adev->pdev->dev, "ACPI: Power source change notification\n");
        // 重新获取系统参数
        amdgpu_acpi_get_params(adev);
        // 通知电源管理子系统
        schedule_work(&atif->atif_notify_work);
        break;
        
    case 0x82:  // Thermal Event
        dev_info(&adev->pdev->dev, "ACPI: Thermal event notification\n");
        // 处理热事件
        break;
        
    case 0x83:  // Brightness Change
        dev_info(&adev->pdev->dev, "ACPI: Brightness change notification\n");
        // 处理亮度变化
        break;
        
    case 0x84:  // Display Switch
        dev_info(&adev->pdev->dev, "ACPI: Display switch notification\n");
        // 处理显示切换
        break;
        
    default:
        dev_dbg(&adev->pdev->dev, "ACPI: Unknown notification: 0x%x\n", value);
        break;
    }
}

// 获取系统参数
static int amdgpu_acpi_get_params(struct amdgpu_device *adev)
{
    struct amdgpu_atif *atif = &adev->acpi.priv->atif;
    union acpi_object *params;
    acpi_handle handle;
    
    handle = ACPI_HANDLE(&adev->pdev->dev);
    if (!handle)
        return -ENODEV;
    
    // 调用 _DSM Function 1 获取系统参数
    // ...
    
    return 0;
}

// 电源来源变化工作处理
static void atif_notify_work_handler(struct work_struct *work)
{
    struct amdgpu_atif *atif = container_of(work, struct amdgpu_atif, atif_notify_work);
    struct amdgpu_device *adev = atif->adev;
    
    // 获取电源来源
    // AC: 1, DC: 0
    
    // 根据电源来源调整 DPM 状态
    // ...
}

// ACPI 驱动注册
static struct acpi_driver amdgpu_acpi_driver = {
    .name = "amdgpu_acpi",
    .class = "display",
    .ids = amdgpu_acpi_device_ids,
    .ops = {
        .add = amdgpu_acpi_add,
        .remove = amdgpu_acpi_remove,
        .notify = amdgpu_acpi_notify,
    },
};
```

#### 2.3 监控 ACPI 电源事件

**创建 ACPI 事件监控脚本：**

```bash
#!/bin/bash
# acpi_event_monitor.sh
# 监控 GPU 相关的 ACPI 事件

LOG_FILE="acpi_events.log"

echo "========================================"
echo "  ACPI Event Monitor Started"
echo "========================================"
echo "Log file: $LOG_FILE"
echo "Press Ctrl+C to stop"
echo "========================================"
echo ""

# 初始化日志文件
echo "=== ACPI Event Monitor Start: $(date) ===" > "$LOG_FILE"

# 函数：记录事件
log_event() {
    local timestamp=$(date "+%Y-%m-%d %H:%M:%S")
    echo "[$timestamp] $1" | tee -a "$LOG_FILE"
}

# 函数：获取当前电源状态
get_power_source() {
    if [ -f /sys/class/power_supply/AC/online ]; then
        local ac_online=$(cat /sys/class/power_supply/AC/online 2>/dev/null)
        if [ "$ac_online" = "1" ]; then
            echo "AC Power (Plugged In)"
        else
            echo "Battery Power"
        fi
    else
        echo "Unknown"
    fi
}

# 函数：获取 GPU 温度
get_gpu_temp() {
    local hwmon_path=$(find /sys/class/drm/card0/device/hwmon/ -name "hwmon*" | head -1)
    if [ -n "$hwmon_path" ] && [ -f "$hwmon_path/temp1_input" ]; then
        local temp=$(cat "$hwmon_path/temp1_input")
        echo $((temp / 1000))
    else
        echo "N/A"
    fi
}

# 函数：获取 GPU 频率
get_gpu_clk() {
    local clk_path="/sys/class/drm/card0/device/pp_dpm_sclk"
    if [ -f "$clk_path" ]; then
        cat "$clk_path" | grep "\*" | awk '{print $2}'
    else
        echo "N/A"
    fi
}

# 函数：获取当前亮度
get_brightness() {
    if [ -d "/sys/class/backlight/amdgpu_bl0" ]; then
        local current=$(cat /sys/class/backlight/amdgpu_bl0/brightness)
        local max=$(cat /sys/class/backlight/amdgpu_bl0/max_brightness)
        local percent=$((current * 100 / max))
        echo "${current}/${max} (${percent}%)"
    else
        echo "N/A"
    fi
}

# 函数：打印当前状态
print_status() {
    log_event "=== Current System Status ==="
    log_event "Power Source: $(get_power_source)"
    log_event "GPU Temperature: $(get_gpu_temp)°C"
    log_event "GPU Clock: $(get_gpu_clk)"
    log_event "Brightness: $(get_brightness)"
    log_event "=============================="
}

# 初始状态
print_status

# 后台监控 acpi_listen
# 使用 named pipe 进行进程间通信
FIFO_FILE="/tmp/acpi_monitor_fifo"
mkfifo "$FIFO_FILE" 2>/dev/null

# 启动 acpi_listen 并写入 FIFO
acpi_listen > "$FIFO_FILE" &
ACPI_PID=$!

# 清理函数
cleanup() {
    log_event "=== ACPI Event Monitor Stopped ==="
    kill $ACPI_PID 2>/dev/null
    rm -f "$FIFO_FILE"
    exit 0
}

trap cleanup SIGINT SIGTERM

# 读取 FIFO 并处理事件
while read -r line; do
    log_event "ACPI Event: $line"
    
    # 解析事件
    # 格式通常是: "device_class notification_type data"
    
    # 检查电源变化事件
    if echo "$line" | grep -iq "ac_adapter\|power_supply"; then
        log_event ">>> Power Source Changed!"
        sleep 1  # 等待状态稳定
        print_status
    fi
    
    # 检查亮度变化事件
    if echo "$line" | grep -iq "video\|brightness"; then
        log_event ">>> Brightness Changed!"
        log_event "New Brightness: $(get_brightness)"
    fi
    
    # 检查热事件
    if echo "$line" | grep -iq "thermal"; then
        log_event ">>> Thermal Event Detected!"
        log_event "GPU Temperature: $(get_gpu_temp)°C"
    fi
    
    # 检查 LID 事件
    if echo "$line" | grep -iq "button/lid"; then
        log_event ">>> Lid Event Detected!"
    fi
    
done < "$FIFO_FILE"
```

**使用方法：**

```bash
# 给予执行权限
chmod +x acpi_event_monitor.sh

# 运行监控脚本
sudo ./acpi_event_monitor.sh

# 在另一个终端测试事件：
# 1. 插拔 AC 适配器
# 2. 调整屏幕亮度
# 3. 查看日志文件
tail -f acpi_events.log
```

#### 2.4 测试系统挂起/恢复

**挂起/恢复测试脚本：**

```bash
#!/bin/bash
# suspend_resume_test.sh
# 测试系统挂起/恢复流程

LOG_FILE="suspend_resume_test.log"
TEST_COUNT=3
SUSPEND_DURATION=10

echo "========================================"
echo "  Suspend/Resume Test Suite"
echo "========================================"
echo "Log file: $LOG_FILE"
echo "Test count: $TEST_COUNT"
echo "Suspend duration: $SUSPEND_DURATION seconds"
echo "========================================"
echo ""

# 检查是否为 root
if [ "$EUID" -ne 0 ]; then
    echo "ERROR: This script must be run as root"
    exit 1
fi

# 初始化日志
echo "=== Suspend/Resume Test Start: $(date) ===" > "$LOG_FILE"

# 函数：记录日志
log() {
    local timestamp=$(date "+%Y-%m-%d %H:%M:%S")
    echo "[$timestamp] $1" | tee -a "$LOG_FILE"
}

# 函数：获取 GPU 状态
get_gpu_status() {
    echo "--- GPU Status ---"
    
    # 设备信息
    if [ -f /sys/class/drm/card0/device/vendor ]; then
        echo "Vendor: $(cat /sys/class/drm/card0/device/vendor)"
        echo "Device: $(cat /sys/class/drm/card0/device/device)"
    fi
    
    # DPM 状态
    if [ -f /sys/class/drm/card0/device/power_dpm_state ]; then
        echo "Power DPM State: $(cat /sys/class/drm/card0/device/power_dpm_state)"
    fi
    
    # SCLK
    if [ -f /sys/class/drm/card0/device/pp_dpm_sclk ]; then
        echo "SCLK States:"
        cat /sys/class/drm/card0/device/pp_dpm_sclk
    fi
    
    # MCLK
    if [ -f /sys/class/drm/card0/device/pp_dpm_mclk ]; then
        echo "MCLK States:"
        cat /sys/class/drm/card0/device/pp_dpm_mclk
    fi
}

# 函数：检查 AMDGPU 驱动状态
check_amdgpu_driver() {
    echo "--- AMDGPU Driver Status ---"
    
    # 检查模块是否加载
    if lsmod | grep -q amdgpu; then
        echo "amdgpu module: LOADED"
    else
        echo "amdgpu module: NOT LOADED"
    fi
    
    # 检查 DRM 设备
    if [ -d /sys/class/drm/card0 ]; then
        echo "DRM card0: PRESENT"
    else
        echo "DRM card0: MISSING"
    fi
    
    # 检查渲染节点
    if [ -c /dev/dri/renderD128 ]; then
        echo "Render node: PRESENT"
    else
        echo "Render node: MISSING"
    fi
}

# 函数：执行挂起
do_suspend() {
    local method=$1
    log "Suspending system using $method..."
    
    # 根据方法选择挂起方式
    case $method in
        "mem")
            # S3 - Suspend to RAM
            echo mem > /sys/power/state
            ;;
        "freeze")
            # S0ix - Freeze (suspend-to-idle)
            echo freeze > /sys/power/state
            ;;
        "disk")
            # S4 - Hibernate
            echo disk > /sys/power/state
            ;;
    esac
    
    log "System resumed from $method"
}

# 主测试循环
for i in $(seq 1 $TEST_COUNT); do
    log "========================================"
    log "Test Cycle $i / $TEST_COUNT"
    log "========================================"
    
    # 挂起前检查
    log "Pre-suspend check..."
    get_gpu_status >> "$LOG_FILE"
    check_amdgpu_driver >> "$LOG_FILE"
    
    # 执行挂起
    # 首先测试 freeze (S0ix)
    if [ $((i % 2)) -eq 1 ]; then
        do_suspend "freeze"
    else
        do_suspend "mem"
    fi
    
    # 等待恢复稳定
    sleep 5
    
    # 挂起后检查
    log "Post-resume check..."
    get_gpu_status >> "$LOG_FILE"
    check_amdgpu_driver >> "$LOG_FILE"
    
    # 检查 dmesg 错误
    log "Checking dmesg for AMDGPU errors..."
    dmesg | grep -i amdgpu | tail -20 >> "$LOG_FILE"
    
    # 检查是否有错误
    if dmesg | grep -i "amdgpu.*error\|amdgpu.*fail" | tail -5; then
        log "WARNING: AMDGPU errors detected in dmesg!"
    else
        log "No AMDGPU errors detected"
    fi
    
    # 测试间隔
    if [ $i -lt $TEST_COUNT ]; then
        log "Waiting 10 seconds before next test..."
        sleep 10
    fi
done

log "========================================"
log "  All Tests Completed"
log "========================================"
log "Full log: $LOG_FILE"
```

**使用方法：**

```bash
# 给予执行权限
chmod +x suspend_resume_test.sh

# 运行测试（需要 root 权限）
sudo ./suspend_resume_test.sh

# 查看详细日志
cat suspend_resume_test.log
```

#### 2.5 调试 ACPI 问题

**常用的 ACPI 调试技巧：**

```bash
# 1. 启用 ACPI 调试
# 添加内核启动参数：acpi.debug_layer=0x2 acpi.debug_level=0x4
# 或临时设置：
sudo mount -t debugfs none /sys/kernel/debug
echo 0x2 | sudo tee /sys/module/acpi/parameters/debug_layer
echo 0x4 | sudo tee /sys/module/acpi/parameters/debug_level

# 2. 查看 ACPI 设备信息
ls -la /sys/bus/acpi/devices/

# 3. 查看 _DSM 方法支持
# 使用 acpiextract 提取 _DSM
sudo acpidump -b
iasl -e *.dat -d DSDT.dat

# 4. 检查 GPU 设备的 ACPI 路径
ls -la /sys/class/drm/card0/device/firmware_node/

# 5. 检查电源管理配置
cat /sys/power/state
cat /sys/power/mem_sleep

# 6. 检查 ACPI 中断
cat /proc/interrupts | grep acpi

# 7. 查看 dmesg 中的 ACPI 相关日志
dmesg | grep -i acpi | tail -50
dmesg | grep -i amdgpu.*acpi | tail -20

# 8. 使用 acpidbg 调试（需要内核配置 CONFIG_ACPI_DEBUGGER）
# sudo acpidbg

# 9. 检查 EC（Embedded Controller）事件
cat /proc/interrupts | grep EC
```

**内核启动参数调优：**

```
# 在 GRUB 配置中添加（/etc/default/grub）
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash acpi_osi=Linux acpi_backlight=vendor"

# 常用 ACPI 参数：
# acpi_osi=Linux          - 告诉 BIOS 我们是 Linux
# acpi_backlight=vendor   - 使用厂商特定的背光控制
# acpi_backlight=video    - 使用标准 ACPI 视频控制
# acpi_backlight=native   - 使用 GPU 原生控制
# acpi=noirq              - 禁用 ACPI IRQ 路由
# pci=noacpi              - 禁用 PCI ACPI
# acpi=off                - 完全禁用 ACPI（仅用于调试）
```

### 3. 案例分析

#### 3.1 案例一：笔记本电脑 AC/DC 切换时 GPU 性能下降问题

**问题描述：**
某款 AMD 笔记本电脑，在拔掉 AC 适配器切换到电池供电时，GPU 性能显著下降，即使插回 AC 适配器后性能也无法恢复。

**问题分析：**

1. **现象观察：**
   - AC 供电时：SCLK 可达 2000MHz
   - 切换到电池：SCLK 限制在 800MHz
   - 切回 AC：SCLK 仍保持在 800MHz

2. **初步排查：**
   ```bash
   # 查看当前 DPM 状态
   cat /sys/class/drm/card0/device/power_dpm_state
   
   # 查看 SCLK 状态
   cat /sys/class/drm/card0/device/pp_dpm_sclk
   
   # 查看 dmesg
   dmesg | grep -i amdgpu.*power
   ```

3. **ACPI 事件监控：**
   ```bash
   # 监控 ACPI 事件
   sudo acpi_listen
   
   # 插拔 AC 适配器时观察：
   # ac_adapter ACPI0003:00 00000080 00000000
   ```

4. **检查 ATIF 系统参数：**
   发现问题：ACPI 通知发送后，驱动没有正确重新获取系统参数。

**根因分析：**

查看内核源码 `amdgpu_acpi.c` 中的 `amdgpu_acpi_notify` 函数：

```c
static void amdgpu_acpi_notify(struct acpi_device *device, u32 value)
{
    struct amdgpu_device *adev = acpi_driver_data(device);
    
    switch (value) {
    case 0x81:  // Power Source Change
        // 问题：这里只打印日志，没有调用参数更新
        dev_info(&adev->pdev->dev, "ACPI: Power source change\n");
        // 缺少：amdgpu_acpi_get_params(adev);
        break;
    // ...
    }
}
```

**修复方案：**

内核补丁（类似修复）：

```diff
diff --git a/drivers/gpu/drm/amd/amdgpu/amdgpu_acpi.c b/drivers/gpu/drm/amd/amdgpu/amdgpu_acpi.c
index abc123..def456 100644
--- a/drivers/gpu/drm/amd/amdgpu/amdgpu_acpi.c
+++ b/drivers/gpu/drm/amd/amdgpu/amdgpu_acpi.c
@@ -500,6 +500,12 @@ static void amdgpu_acpi_notify(struct acpi_device *device, u32 value)
        case 0x81:  /* Power Source Change */
                dev_info(&adev->pdev->dev,
                        "ACPI: Power source change notification\n");
+
+               /* Re-fetch system params to get updated power profile */
+               if (amdgpu_acpi_get_params(adev) == 0) {
+                       /* Notify PM subsystem */
+                       schedule_work(&atif->atif_notify_work);
+               }
                break;
 
        case 0x82:  /* Thermal Event */
```

**临时解决方案（用户态）：**

```bash
#!/bin/bash
# power_source_fix.sh
# 临时修复：手动触发电源配置文件更新

fix_power_profile() {
    # 强制重新加载 DPM 配置
    echo "performance" > /sys/class/drm/card0/device/power_dpm_force_performance_level
    sleep 1
    echo "auto" > /sys/class/drm/card0/device/power_dpm_force_performance_level
    
    # 或直接设置 SCLK 范围
    echo "s 0 30" > /sys/class/drm/card0/device/pp_od_clk_voltage
    cat /sys/class/drm/card0/device/pp_od_clk_voltage
}

# 监控电源变化
udevadm monitor --property | while read -r line; do
    if echo "$line" | grep -q "POWER_SUPPLY_ONLINE"; then
        sleep 2
        fix_power_profile
        echo "[$(date)] Fixed power profile after AC/DC switch"
    fi
done
```

**资料来源：**
- Linux 内核补丁：https://lore.kernel.org/dri-devel/
- AMDGPU 驱动 Bug 报告：https://gitlab.freedesktop.org/drm/amd/-/issues/
- 类似问题：https://gitlab.freedesktop.org/drm/amd/-/issues/1234

#### 3.2 案例二：系统挂起后 GPU 无法恢复

**问题描述：**
某款搭载 AMD RDNA 2 GPU 的笔记本电脑，执行挂起（Suspend to RAM）后，恢复时屏幕黑屏，GPU 无法正常工作。

**问题分析：**

1. **现象：**
   - 挂起正常
   - 唤醒时电源指示灯亮
   - 屏幕无显示
   - 可以 SSH 登录（说明系统内核仍在运行）

2. **SSH 登录检查：**
   ```bash
   # 检查 GPU 状态
   cat /sys/class/drm/card0/device/device
   
   # 检查 dmesg
   dmesg | grep -i amdgpu | tail -50
   
   # 可能看到的错误：
   # [drm:amdgpu_device_ip_resume_phase2 [amdgpu]] *ERROR* resume of IP block <smu> failed -62
   # [drm:amdgpu_device_resume [amdgpu]] *ERROR* amdgpu_device_ip_resume failed (-62).
   ```

3. **检查 ACPI 表：**
   发现问题：DSDT 表中的 _WAK 方法没有正确处理 GPU 恢复。

**根因分析：**

查看系统 DSDT 表中的 _WAK 方法：

```asl
Method (_WAK, 1, NotSerialized)
{
    // 问题：_WAK 方法中缺少 GPU 相关的恢复代码
    // 或者顺序不正确
    
    // 应该在恢复 GPU 之前完成某些 EC 操作
    If (Arg0 == 0x03)  // S3 resume
    {
        // 缺少：等待 EC 就绪
        // 缺少：调用 GPU 的 _PS0 方法
    }
    
    Return (Package(2) { 0x00, 0x00 })
}
```

**解决方案：**

1. **内核参数临时修复：**
   ```
   # 在 GRUB 中添加
   GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amdgpu.runpm=0"
   ```

2. **DSDT 覆盖（DSDT Override）：**
   ```bash
   # 1. 提取 DSDT
   sudo acpidump -b
   iasl -d DSDT.dat
   
   # 2. 编辑 DSDT.dsl，修复 _WAK 方法
   
   # 3. 重新编译
   iasl -tc DSDT.dsl
   
   # 4. 创建 initramfs 覆盖
   mkdir -p kernel/firmware/acpi
   cp DSDT.aml kernel/firmware/acpi/
   
   # 5. 打包
   find kernel | cpio -H newc --create > acpi_override.cpio
   
   # 6. 配置 GRUB 使用
   # /boot/grub/grub.cfg 中添加：
   # initrd /acpi_override.cpio /initrd.img
   ```

3. **运行时电源管理禁用：**
   ```bash
   # 临时禁用 Runtime PM
   echo on > /sys/bus/pci/devices/0000:01:00.0/power/control
   
   # 永久禁用（modprobe 配置）
   echo "options amdgpu runpm=0" | sudo tee /etc/modprobe.d/amdgpu.conf
   ```

**资料来源：**
- AMDGPU 挂起/恢复 Bug：https://gitlab.freedesktop.org/drm/amd/-/issues/1500
- DSDT 覆盖指南：https://www.kernel.org/doc/html/latest/admin-guide/acpi/initrd_table_override.html
- Linux 内核文档：`Documentation/power/runtime_pm.txt`

#### 3.3 案例三：背光控制失效问题

**问题描述：**
某款 AMD APU 笔记本电脑，升级内核后屏幕亮度调节功能失效，Fn 快捷键和系统设置都无法调整亮度。

**问题分析：**

1. **检查背光接口：**
   ```bash
   ls -la /sys/class/backlight/
   
   # 可能的情况：
   # 情况1：没有任何背光接口
   # 情况2：有多个接口（acpi_video0 + amdgpu_bl0）
   # 情况3：接口存在但写入无效
   ```

2. **检查内核日志：**
   ```bash
   dmesg | grep -i backlight
   dmesg | grep -i amdgpu.*acpi
   
   # 可能看到：
   # amdgpu 0000:04:00.0: ATIF: VRAM brightness is not supported
   # amdgpu 0000:04:00.0: Backlight control is disabled
   ```

3. **检查 ACPI _DSM 支持：**
   ```bash
   # 使用 acpidump + iasl 检查 DSDT 中的 _DSM
   # 查看 ATIF UUID 的 Function 16（亮度控制）是否被声明为支持
   ```

**根因分析：**

查看 `amdgpu_acpi.c` 中的背光检测逻辑：

```c
static int amdgpu_acpi_atif_probe_backlight(struct amdgpu_device *adev)
{
    struct amdgpu_atif *atif = &adev->acpi.priv->atif;
    
    // 检查 ATIF 功能 16（亮度控制）是否支持
    if (!(atif->atif_functions_supported[0] & (1 << 16))) {
        dev_info(&adev->pdev->dev,
                "ATIF: Brightness control not supported by BIOS\n");
        return -ENODEV;
    }
    
    // 继续初始化背光...
}
```

**问题：**
- 旧版本 BIOS 的 DSDT 表中，ATIF 功能位图错误地标记亮度控制为不支持
- 或者 ATIF _DSM 方法实现有 bug

**解决方案：**

1. **强制使用 acpi_video 接口：**
   ```
   # 内核参数
   acpi_backlight=video
   ```

2. **强制使用 native 接口：**
   ```
   # 内核参数
   acpi_backlight=native
   ```

3. **禁用 ATIF 背光检测：**
   ```bash
   # 编写内核模块参数
   echo "options amdgpu abmlevel=0" | sudo tee /etc/modprobe.d/amdgpu.conf
   ```

4. **用户态脚本临时修复：**
   ```bash
   #!
