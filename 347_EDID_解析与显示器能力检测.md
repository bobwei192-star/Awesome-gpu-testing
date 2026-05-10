# EDID 解析与显示器能力检测

## 学习目标

- 理解 EDID（Extended Display Identification Data）的规范结构及各数据块的含义
- 掌握 EDID 的读取、解析、缓存和校验机制
- 深入理解 AMDGPU DC 驱动中 EDID 的处理流程和数据流
- 掌握 EDID 相关的调试工具和问题排查方法
- 学会处理 EDID 故障场景，如损坏、缺失、扩展块问题

## 知识详解

### 概念原理

#### EDID 概述

EDID（Extended Display Identification Data）是 VESA 定义的数据格式，用于显示器向显卡报告自身的能力和特性。EDID 存储在显示器的 ROM 中，通过 I2C 总线（DDC 通道）读取，操作系统和显示驱动据此决定最佳的显示模式。

EDID 数据结构通过自顶向下的分层架构，将显示器的基本标识、物理特性、时序支持和扩展功能依次编码。

EDID 的数据结构采用层次化的块（Block）组织方式：

```ascii
┌─────────────────────────────────────────────────────────────┐
│                  EDID 数据结构总览                           │
├─────────────────────────────────────────────────────────────┤
│  Block 0 (Base Block): 128 bytes                            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Header (8 bytes)      │  Manufacturer ID (2 bytes)   │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ Product Code (2 bytes)│  Serial Number (4 bytes)     │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ Week/Year of Manufacture (2 bytes)                   │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ EDID Version/Revision (2 bytes)                      │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ Basic Display Parameters (5 bytes)                   │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ Chromaticity Coordinates (10 bytes)                  │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ Established Timings (3 bytes) │ Standard Timings     │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ Detailed Timing Descriptor #1 (18 bytes)             │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ Detailed Timing Descriptor #2 (18 bytes)             │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ Detailed Timing Descriptor #3 (18 bytes)             │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ Detailed Timing Descriptor #4 (18 bytes)             │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ Extension Count (1 byte)     │  Checksum (1 byte)    │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  Block 1..n (Extension Blocks): 128 bytes each              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ CEA-861 Extension (Tag 0x02)                         │   │
│  │ - Audio Formats                                      │   │
│  │ - Video Capability (VICs)                            │   │
│  │ - HDR Static Metadata (VSDB)                         │   │
│  │ - Speaker Allocation                                 │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ DisplayID Extension (Tag 0x70)                       │   │
│  │ - Detailed timings beyond 4 base blocks              │   │
│  │ - Display device features                            │   │
│  │ - CTA-861 DisplayID integration                      │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ DI-EXT (Display Information Extension, Tag 0x40)     │   │
│  │ - Manufacturer-specific data                         │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

#### Base Block 结构详解

EDID Base Block 的前 128 字节包含以下主要字段：

```ascii
字节偏移 │ 字段名          │ 长度 │ 描述
─────────┼─────────────────┼──────┼──────────────────────────
   00-07 │ Header          │  8B  │ 固定值: 00 FF FF FF FF FF FF 00
   08-09 │ Manufacturer ID │  2B  │ 3 字符 PNP ID 编码
   0A-0B │ Product Code    │  2B  │ 厂商定义产品型号
   0C-0F │ Serial Number   │  4B  │ 序列号（可选）
   10-11 │ Week/Year       │  2B  │ 生产周/年份
   12-13 │ EDID Version    │  2B  │ 版本号: 01 04 (v1.4)
   14-18 │ Basic Params    │  5B  │ 显示类型、信号接口、尺寸等
   19-22 │ Chromaticity    │ 10B  │ 色度坐标 RGGB
   23-25 │ Established     │  3B  │ 标准时序位图
   26-35 │ Standard Timings│  8×2B│ 8 个标准时序
   36-53 │ DT #1           │ 18B  │ 详细时序描述符 1
   54-71 │ DT #2           │ 18B  │ 详细时序描述符 2
   72-89 │ DT #3           │ 18B  │ 详细时序描述符 3
  90-107 │ DT #4           │ 18B  │ 详细时序描述符 4
     108 │ Extension Count │  1B  │ 扩展块数量
     127 │ Checksum        │  1B  │ 校验和: 所有字节和 mod 256 = 0
```

#### Manufacturer ID 编码规则

制造商 ID 使用 3 字符 PNP ID 编码，存储在两个字节中：

```ascii
Bit:  15 14 13 12 11 10  9  8  7  6  5  4  3  2  1  0
     ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
     │  Char 1 (5 bits)  │  Char 2 (5 bits)  │ Char 3 │
     │  0x00='A'...0x19=│  0x00='A'...0x19= │ (5 bits)│
     │      'Z'          │      'Z'          │  same   │
     └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
```

每个字符的编码：'A' = 0x00, 'B' = 0x01, ..., 'Z' = 0x19

示例：十六进制 0x4D10 → 二进制: 0100 1101 0001 0000
- Char 1: 01001 = 9 → 'J'
- Char 2: 01101 = 13 → 'N'
- Char 3: 00010 = 2 → 'C'
→ PNP ID = "JNC"

#### Basic Display Parameters

字节 14-18（基本显示参数）包含关键的显示器能力信息：

```ascii
字节 14: 显示类型与信号接口
  Bit 7: 数字输入 = 1, 模拟输入 = 0
  Bits 6-5: 数字信号类型:
    00 = RGB 4:4:4 + YCrCb 4:4:4
    01 = RGB 4:4:4 only
    10 = RGB + YCrCb 4:4:4 + YCrCb 4:2:2
    保留
  Bit 4-0: 模拟接口时的白电平/同步类型

字节 15: 水平尺寸（cm），最大 255 cm
字节 16: 垂直尺寸（cm），最大 255 cm
  若字节 15/16 = 0，则为投影仪或未知尺寸

字节 17: 显示特性标志
  Bit 7: 电源管理: DPMS 支持
  Bit 6: 预置时序支持: 固定频率
  Bit 5: 标准色彩空间: sRGB
  Bit 4-0: 其他特性

字节 18: 色度坐标信息
  Bit 7: 白点偏移
  Bit 6-5: 时序格式
  Bit 4: 支持 CVT 标准
  Bit 3-0: 保留
```

#### Chromaticity Coordinates（色度坐标）

色度坐标以 10-bit 精度存储显示器的 RGB 和白点的 CIE 1931 xy 坐标：

```ascii
字节 19: 红色 X 高 2-bit (bits 1-0)
        红色 Y 高 2-bit (bits 3-2)
        绿色 X 高 2-bit (bits 5-4)
        绿色 Y 高 2-bit (bits 7-6)
字节 20: 蓝色 X 高 2-bit (bits 1-0)
        蓝色 Y 高 2-bit (bits 3-2)
        白色 X 高 2-bit (bits 5-4)
        白色 Y 高 2-bit (bits 7-6)
字节 21: 红色 X 低 8-bit
字节 22: 红色 Y 低 8-bit
字节 23: 绿色 X 低 8-bit
字节 24: 绿色 Y 低 8-bit
字节 25: 蓝色 X 低 8-bit
字节 26: 蓝色 Y 低 8-bit
字节 27: 白色 X 低 8-bit
字节 28: 白色 Y 低 8-bit
```

坐标值 = (高 2-bit << 8 | 低 8-bit) / 1024

#### Established Timings

字节 23-25 以位图格式表示标准时序的支持情况：

```ascii
字节 23 (Established Timing I):
  Bit 7: 720×400 @ 70Hz
  Bit 6: 720×400 @ 88Hz
  Bit 5: 640×480 @ 60Hz    [VESA STD]
  Bit 4: 640×480 @ 67Hz    [Apple Mac II]
  Bit 3: 640×480 @ 72Hz
  Bit 2: 640×480 @ 75Hz
  Bit 1: 800×600 @ 56Hz
  Bit 0: 800×600 @ 60Hz

字节 24 (Established Timing II):
  Bit 7: 800×600 @ 72Hz
  Bit 6: 800×600 @ 75Hz
  Bit 5: 832×624 @ 75Hz    [Apple Mac]
  Bit 4: 1024×768 @ 87Hz   [Interlaced]
  Bit 3: 1024×768 @ 60Hz
  Bit 2: 1024×768 @ 70Hz
  Bit 1: 1024×768 @ 75Hz
  Bit 0: 1280×1024 @ 75Hz

字节 25 (Manufacturer Timing):
  0x01 = 1152×870 @ 75Hz [Apple Mac]
  0x02 = 1280×1024 @ 60Hz [VESA]
  0x04 = 640×350 @ 85Hz [VESA]
  其他位 = 保留
```

#### Standard Timings

字节 26-35 包含 8 个标准时序条目，每个 2 字节：

```ascii
标准时序条目格式:
字节 N: 水平分辨率 / 8 - 31 (例如 1024/8-31=97=0x61)
字节 N+1: 刷新率与类型
  Bit 7-6: 刷新率类型:
    00 = 厂商保留
    01 = 60Hz
    10 = 75Hz
    11 = 85Hz
  Bit 5-0: 垂直分辨率行数

特殊值:
  0x01 0x01 = 列表结束标记
  0x00 0x00 = 未使用条目
```

#### Detailed Timing Descriptor (DTD)

每个 DTD（详细时序描述符）占用 18 字节，提供完整的显示时序参数：

```ascii
字节偏移 │ 字段              │ 长度 │ 描述
─────────┼───────────────────┼──────┼──────────────────────
   00-01 │ Pixel Clock       │  2B  │ 像素时钟 (单位 10kHz)
      02 │ H_Active (低8位)  │  1B  │ 水平有效像素低 8 位
      03 │ H_Blanking (低8位)│  1B  │ 水平消隐低 8 位
   04-05 │ H_Active/Blanking │  2B  │ 高 4 位 (active/blanking 各 4-bit)
      06 │ V_Active (低8位)  │  1B  │ 垂直有效行低 8 位
      07 │ V_Blanking (低8位)│  1B  │ 垂直消隐低 8 位
   08-09 │ V_Active/Blanking │  2B  │ 高 4 位 (active/blanking 各 4-bit)
      10 │ H_Sync Offset     │  1B  │ 水平同步偏移 (低8位)
      11 │ H_Sync Pulse Width│  1B  │ 水平同步脉宽 (低8位)
   12-13 │ V_Sync Offset/PW │  2B  │ 垂直同步偏移/脉宽
   14-15 │ H/V Border        │  2B  │ 水平/垂直边界
      16 │ Features          │  1B  │ 同步极性、交错、立体等
      17 │ Checksum/Reserved │  1B  │ 仅 Monitor Name/Serial 使用
```

Features 字节（字节 16）的位定义：

```ascii
Bit 7: 交错模式 (Interlaced)
Bit 6: 保留
Bit 5: 数字复合同步 (Digital Composite)
Bit 4: 模拟复合同步 (Analog Composite)
Bit 3: 同步在绿通道 (Sync on Green)
Bit 2: VSync 极性 (1=正极性, 0=负极性)
Bit 1: HSync 极性 (1=正极性, 0=负极性)
Bit 0: 保留
```

当 HSync 极性 = 1 且 VSync 极性 = 1 时，表示该时序使用 RGB 颜色格式。

#### DTD 类型识别

DTD 可以是时序描述符或显示描述符，由前 2 字节（Pixel Clock / Tag）决定：

```ascii
前 2 字节值  │ 类型               │ 描述
────────────┼────────────────────┼────────────────────
  0000h     │ Manufacturer Serial│ 制造商序列号（ASCII）
  0001h-00Fh│ 保留                │ 未使用
  FF10h     │ ASCII String Tag   │ 显示器名称 (Monitor Name)
  FD00h-FDFFh│ Monitor Limits    │ 显示器刷新率/分辨率限制范围
  FC00h-FCFFh│ Text String       │ 显示器型号名（最长 13 字符）
  FB00h-FBFFh│ White Point Data  │ 附加白点信息
  FA00h-FAFFh│ Standard Timing IDs│ 附加标准时序
  >= 0100h  │ Display Timing     │ 实际显示时序描述符
```

Monitor Limits（显示器限制范围）的格式：

```ascii
字节 3: 最小垂直刷新率 (Hz)
字节 4: 最大垂直刷新率 (Hz)
字节 5: 最小水平刷新率 (kHz)
字节 6: 最大水平刷新率 (kHz)
字节 7: 最大像素时钟 (单位 10MHz, 值-255 再 *10)
字节 8: 扩展时序类型:
  0x00 = 默认 (GTF)
  0x02 = 启用 CVT (Coordinated Video Timing)
  0x04 = CVT with reduced blanking
  0x0A = GTF with secondary curve
字节 9-17: 预留 (GTF/CVT 参数)
```

#### CEA-861 扩展块

CEA-861 扩展块（Tag 0x02）为消费电子设备提供增强功能支持：

```ascii
CEA-861 扩展块结构:
┌──────────────────────────────────────────────┐
│  Tag: 0x02 (1 byte)                          │
│  Revision: 0x03 (CEA-861C), 0x04 (D) etc.    │
├──────────────────────────────────────────────┤
│  DTD Offset (1 byte): 指向 DTD 列表的偏移    │
├──────────────────────────────────────────────┤
│  Feature Flag (1 byte):                       │
│  Bit 7: YCbCr 4:4:4, Bit 6: YCbCr 4:2:2     │
│  Bit 5-4: 色彩深度支持                       │
│  Bit 3-0: 保留/其他                          │
├──────────────────────────────────────────────┤
│  Data Block Collection (可变长度):            │
│  ┌────────────────────────────────────────┐   │
│  │ VCDB: Video Capability Data Block     │   │
│  │  - IT/CE 格式标志                     │   │
│  │  - 量化范围 (Q0/Q1)                   │   │
│  ├────────────────────────────────────────┤   │
│  │ VSDB: Vendor Specific Data Block      │   │
│  │  - IEEE OUI (如 HDMI = 0x000C03)      │   │
│  │  - HDMI 特性: TMDS, Deep Color, etc.  │   │
│  │  - 3D 结构支持                        │   │
│  │  - HDR 静态元数据 (ST 2084)           │   │
│  ├────────────────────────────────────────┤   │
│  │ ADB: Audio Data Block                 │   │
│  │  - LPCM, AC3, DTS, DD+ 等格式         │   │
│  │  - 采样率: 32k-192k                   │   │
│  │  - 声道数                             │   │
│  ├────────────────────────────────────────┤   │
│  │ SDB: Speaker Allocation Data Block    │   │
│  │  - 扬声器配置                         │   │
│  ├────────────────────────────────────────┤   │
│  │ VFPDB: Video Format Preference DB     │   │
│  │  - 偏好视频格式列表                   │   │
│  ├────────────────────────────────────────┤   │
│  │ HDR Static Metadata (ST 2084):        │   │
│  │  - Electro-Optical Transfer Function  │   │
│  │  - Static Metadata Descriptor ID      │   │
│  │  - Display Primary 1/2/3 色度坐标     │   │
│  │  - White Point 色度坐标               │   │
│  │  - Max/Min Luminance                  │   │
│  │  - Max FALL / Max CLL                 │   │
│  └────────────────────────────────────────┘   │
├──────────────────────────────────────────────┤
│  DTD List (0-6 DTDs, 18 bytes each)          │
│  Checksum (1 byte)                            │
└──────────────────────────────────────────────┘
```

VSDB (Vendor Specific Data Block) 中最关键的 HDMI 属性：

```ascii
VSDB for HDMI LLC (IEEE OUI 0x000C03):
字节 0: Length (包括 OUI 和 OUI 之后的字节数)
字节 1-3: IEEE OUI (0x00, 0x0C, 0x03)
字节 4: A (Physical Address A)
字节 5: B (Physical Address B)
字节 6: C (Physical Address C) + Flags
  Bit 7: DC_30bit (Deep Color 30-bit)
  Bit 6: DC_36bit (Deep Color 36-bit)
  Bit 5: DC_48bit (Deep Color 48-bit)
  Bit 4: DC_YCC444 (Deep Color YCbCr 4:4:4)
  Bit 3-0: 保留
字节 7: 最大 TMDS 时钟 (MHz / 5)
字节 8: 延迟信息 + 视频延迟
...
字节 N: HDMI 2.1 FRL 能力
  Bit 7: FRL 支持
  Bit 6-4: FRL 最大速率 (0=3-lane 6G, 1=4-lane 6G, 2=4-lane 10G, 3=4-lane 12G, 4=4-lane 6G+DSC, 5=4-lane 10G+DSC, 6=4-lane 12G+DSC)
  Bit 3-0: 保留
```

#### HDR Static Metadata Block

HDR 静态元数据块包含显示器 HDR 能力的详细描述：

```ascii
HDR Static Metadata Data Block:
字节 0: 数据块标签 + 长度
  高 5-bit: Tag = 0x07 (HDR Static Metadata)
  低 3-bit: Length (0x06 = 6 bytes data)

字节 1: Electro-Optical Transfer Function (EOTF)
  Bit 0: Traditional Gamma - SDR Luminance Range
  Bit 1: SMPTE ST 2084 (HDR10)
  Bit 2: Hybrid Log-Gamma (HLG)
  Bit 3-7: 保留

字节 2: Static Metadata Descriptor ID
  0x00 = 未使用
  0x01 = Static Metadata Type 1

字节 3-4: Display Primary 1 (Red) x,y
  高 2-bit x (bits 1-0) + 高 2-bit y (bits 3-2) + 低 8-bit x + 低 8-bit y
  坐标 = (高2bit << 8 | 低8bit) / 50000 (0.00002 精度)

字节 5-6: Display Primary 2 (Green) x,y
字节 7-8: Display Primary 3 (Blue) x,y
字节 9-10: White Point x,y

字节 11-12: Max Luminance (单位: 0.0001 cd/m²)
字节 13-14: Min Luminance (单位: 0.0001 cd/m²)
字节 15-16: Max CLL (Content Light Level, 单位: cd/m²)
字节 17-18: Max FALL (Frame Average Light Level, 单位: cd/m²)
```

#### DisplayID 扩展块

DisplayID 是 VESA 的较新标准，提供更灵活的显示器描述结构：

```ascii
DisplayID 结构:
┌──────────────────────────────────────────┐
│  Header (6 bytes):                       │
│  - 'DISPLAYID' magic                     │
│  - Version (1.0/2.0)                     │
│  - Extension count                       │
├──────────────────────────────────────────┤
│  Data Blocks (可变长度,每个带 header):     │
│  ┌──────────────────────────────────┐    │
│  │ Product Identity Block           │    │
│  │  - Manufacturer PNP ID           │    │
│  │  - Product Code                  │    │
│  │  - Serial Number (string)        │    │
│  │  - Week/Year of Manufacture      │    │
│  ├──────────────────────────────────┤    │
│  │ Display Parameters Block         │    │
│  │  - Display type (LCD/OLED/投影)  │    │
│  │  - Physical size                 │    │
│  │  - Pixel density (DPI)           │    │
│  ├──────────────────────────────────┤    │
│  │ Video Timing Block (Type I-III)  │    │
│  │  - 支持格式列表                  │    │
│  │  - Aspect ratio / refresh        │    │
│  ├──────────────────────────────────┤    │
│  │ Color Characteristics Block      │    │
│  │  - Colorimetry (BT.2020, DCI-P3) │    │
│  │  - Gamut boundaries              │    │
│  ├──────────────────────────────────┤    │
│  │ Display Device Data Block        │    │
│  │  - Feature support flags         │    │
│  └──────────────────────────────────┘    │
└──────────────────────────────────────────┘
```

#### EDID 读取机制：DDC 通道

EDID 通过 DDC（Display Data Channel）读取，DDC 基于 I2C 总线协议：

```ascii
DDC/CI 通信协议栈:
┌─────────────────────────────────────────────┐
│  应用层: EDID 数据块读取 (0x50 / 0x30)      │
├─────────────────────────────────────────────┤
│  传输层: DDC/CI 协议 (VESA DDC/CI v1.1)    │
├─────────────────────────────────────────────┤
│  链路层: I2C 总线 (100kHz standard mode)    │
├─────────────────────────────────────────────┤
│  物理层: DDC 线 (SCL/SDA on DP AUX or       │
│          HDMI DDC pins)                     │
└─────────────────────────────────────────────┘

I2C 读取 EDID 时序:
┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
│START │    │SLA+W│    │ Word│    │ START│
│      │───►│ 0xA0│───►│ 0x00│───►│      │
└──────┘    └──────┘    └──────┘    └──────┘
┌──────┐    ┌────────────────┐    ┌──────┐
│SLA+R │    │ 128 bytes Data │    │STOP  │
│ 0xA1 │───►│  from EDID     │───►│      │
└──────┘    └────────────────┘    └──────┘

I2C 地址: 0x50 (EDID 主块), 0x30 (DisplayID/扩展块)
```

对于 DisplayPort，EDID 通过 AUX 通道读取：

```ascii
DP AUX 读取 EDID 流程:
┌─────────────┐     ┌─────────────────┐     ┌─────────────┐
│ GPU AUX TX  │────►│  AUX Request    │────►│ Monitor AUX │
│             │     │  (Read, Addr:   │     │   RX        │
│ I2C-over-AUX│◄────│   0x50, Len:16)│◄────│             │
│             │     │  AUX Reply with  │     │  I2C-over-  │
│  EDID Cache │     │  EDID data      │     │  AUX engine │
└─────────────┘     └─────────────────┘     └─────────────┘
```

### 实践操作

#### AMDGPU DC EDID 处理流程

在 AMDGPU DC 驱动中，EDID 处理分为以下几个阶段：

```ascii
EDID 处理全流程:
┌──────────────┐
│ HPD 事件触发  │  ── 显示器连接/断开
└──────┬───────┘
       ▼
┌──────────────┐
│ 任务调度     │  ── dm_handle_hpd_work()
└──────┬───────┘
       ▼
┌──────────────────┐
│ EDID 读取 (I2C)   │  ── dc_link_detect() → read_current_link()
└──────┬───────────┘
       ▼
┌──────────────────┐
│ EDID 校验        │  ── Checksum + Block validation
└──────┬───────────┘
       ▼
┌──────────────────────┐
│ EDID 解析与能力提取   │  ── amdgpu_dm_update_connector()
└──────┬───────────────┘
       ▼
┌──────────────────────┐
│ DRM Display Mode List │  ── drm_add_display_info() + drm_add_edid_modes()
└──────┬───────────────┘
       ▼
┌──────────────────────┐
│ 构建 DC 内部时序列表  │  ── dc_sink_create() → link->dc_sink
└──────┬───────────────┘
       ▼
┌──────────────────────┐
│ 显示模式应用         │  ── Atomic Check → Commit
└──────────────────────┘
```

核心数据结构关系：

```ascii
struct drm_connector (DRM Core)
│
├── edid_blob_ptr: struct drm_property_blob *  ── EDID 原始二进制数据
├── display_info: struct drm_display_info      ── 解析后的显示能力
│   ├── width_mm / height_mm                   ── 物理尺寸
│   ├── bpc                                   ── 色彩深度
│   ├── color_formats                         ── 支持的色彩格式
│   ├── max_tmds_clock                        ── 最大 TMDS 时钟
│   ├── hdmi                                   ── HDMI 能力标志
│   ├── non_desktop                           ── 非桌面显示器标志
│   └── rgb_quant_range_selectable            ── 量化范围可选
│
├── monitor_info: struct drm_monitor_info      ── 显示器名称/厂商
├── probed_modes: list_head                    ── 探测到的时序列表
└── modes: list_head                           ── 已确认的时序列表

struct dc_sink (AMDGPU DC)
│
├── edid_caps: struct dc_edid_caps             ── DC 层 EDID 能力
│   ├── edid_buf: uint8_t *                    ── 原始 EDID 数据
│   ├── edid_size: uint32_t                    ── EDID 总大小
│   ├── display_color_depth                    ── 色彩深度能力
│   └── signal_type                            ── 信号类型
│
└── sink_signal: enum signal_type              ── 信号类型
```

EDID 读取与解析的核心函数链：

```c
// dc_link_detect.c - EDID 读取入口
bool dc_link_detect(struct dc_link *link, enum dc_detect_reason reason)
{
    // 1. 读取 EDID
    struct dc_sink *sink = dc_sink_create(link->sink_factory,
        link->link_id, signal_type);
    
    // 2. 在 read_current_link() 中调用
    //    - AUX 或 I2C 通道读取 EDID
    //    - 解析 EDID 到 dc_sink->edid_caps
    result = read_current_link(link, &sink_data);
    
    // 3. 创建 DC sink 对象
    link->dc_sink[id] = sink;
    
    // 4. 通知 DM 层更新
    dm_helpers_dp_update_edid(link, &sink_data);
    
    return result;
}

// amdgpu_dm.c - DRM 层 EDID 处理
static int amdgpu_dm_update_connector(struct amdgpu_display_manager *dm,
    struct drm_connector *connector, struct edid *edid)
{
    // 1. 调用 drm_add_edid_modes() 解析 EDID 并生成 mode list
    drm_connector_update_edid_property(connector, edid);
    drm_add_display_info(edid, &connector->display_info, connector);
    
    // 2. 添加 EDID 中定义的时序
    new_num_modes = drm_add_edid_modes(connector, edid);
    
    // 3. 更新 DC 连接器状态
    dm_dp_create_fake_mst_encoders(dm);
    
    // 4. 处理 HPD 状态变化
    if (connector->force == DRM_FORCE_UNSPECIFIED)
        drm_kms_helper_hotplug_event(dm->ddev);
    
    return new_num_modes;
}
```

EDID 校验实现：

```c
// drivers/gpu/drm/drm_edid.c - EDID 校验
bool drm_edid_is_valid(const struct edid *edid)
{
    // 1. 检查 Header: 00 FF FF FF FF FF FF 00
    if (!drm_edid_block_valid(edid, 0, false))
        return false;
    
    // 2. 检查 Base Block 校验和
    if (!drm_edid_block_valid(edid, 0, true))
        return false;
    
    // 3. 检查所有扩展块
    for (i = 0; i < edid->extensions; i++) {
        if (!drm_edid_block_valid(edid, i + 1, true))
            return false;
    }
    
    return true;
}

static bool drm_edid_block_valid(const struct edid *edid, int block,
    bool check_checksum)
{
    const u8 *raw = (const u8 *)edid + block * EDID_LENGTH;
    u8 csum = 0;
    int i;
    
    // Header 校验
    if (block == 0) {
        for (i = 0; i < EDID_HEADER_SIZE; i++) {
            if (raw[i] != edid_header[i])
                return false;
        }
    }
    
    // 校验和检查
    if (check_checksum) {
        for (i = 0; i < EDID_LENGTH; i++)
            csum += raw[i];
        if (csum != 0) {
            return false;
        }
    }
    
    return true;
}
```

EDID 时序解析：

```c
// drivers/gpu/drm/drm_edid.c - 详细时序描述符解析
static struct drm_display_mode *drm_mode_detailed(struct drm_device *dev,
    const struct edid *edid, const u8 *timing, bool is_extension)
{
    struct drm_display_mode *mode;
    unsigned hactive, hblank, hsync_len, hfront_porch;
    unsigned vactive, vblank, vsync_len, vfront_porch;
    unsigned pixel_clock;
    
    // 1. 解析像素时钟 (10kHz 单位)
    pixel_clock = (timing[1] << 8) | timing[0];
    if (pixel_clock == 0)
        return NULL;
    
    // 2. 解析水平时序参数
    hactive = (timing[4] & 0xF0) << 4 | timing[2];
    hblank = (timing[4] & 0x0F) << 8 | timing[3];
    hsync_len = (timing[11] & 0xF0) << 4 | timing[11] & 0x0F;
    // 注意: 实际 AMDGPU DC 使用的解析方式略有不同
    
    // 3. 解析垂直时序参数
    vactive = (timing[9] & 0xF0) << 4 | timing[5];
    vblank = (timing[9] & 0x0F) << 8 | timing[6];
    vsync_len = (timing[11] & 0x0F);
    
    // 4. 计算前后肩
    hfront_porch = hblank - hsync_len;
    vfront_porch = vblank - vsync_len;
    
    // 5. 创建 DRM display mode
    mode = drm_mode_create(dev);
    mode->clock = pixel_clock / 100;  // 转为 10kHz → MHz
    mode->hdisplay = hactive;
    mode->hsync_start = hactive + hfront_porch;
    mode->hsync_end = hactive + hfront_porch + hsync_len;
    mode->htotal = hactive + hblank;
    mode->vdisplay = vactive;
    mode->vsync_start = vactive + vfront_porch;
    mode->vsync_end = vactive + vfront_porch + vsync_len;
    mode->vtotal = vactive + vblank;
    
    // 6. 设置同步极性
    mode->flags |= (timing[16] & 0x02) ? DRM_MODE_FLAG_PHSYNC : DRM_MODE_FLAG_NHSYNC;
    mode->flags |= (timing[16] & 0x04) ? DRM_MODE_FLAG_PVSYNC : DRM_MODE_FLAG_NVSYNC;
    if (timing[16] & 0x80)
        mode->flags |= DRM_MODE_FLAG_INTERLACE;
    
    drm_mode_set_name(mode);
    
    return mode;
}
```

EDID 在 AMDGPU DC 中的缓存与查找：

```c
// amdgpu_dm.c - EDID 属性管理
void drm_connector_update_edid_property(struct drm_connector *connector,
    const struct edid *edid)
{
    int size;
    
    if (connector->edid_blob_ptr) {
        drm_property_unreference_blob(connector->edid_blob_ptr);
        connector->edid_blob_ptr = NULL;
    }
    
    if (edid) {
        size = EDID_LENGTH * (1 + edid->extensions);
        connector->edid_blob_ptr = drm_property_create_blob(connector->dev,
            size, edid);
        
        // 更新 display_info
        drm_add_display_info(edid, &connector->display_info, connector);
    }
    
    // 通知用户空间 EDID 已更新
    drm_sysfs_hotplug_event(connector->dev);
}

// 读取连接器的 EDID blob
const struct edid *drm_edid_blob_get(struct drm_connector *connector)
{
    if (!connector->edid_blob_ptr)
        return NULL;
    return connector->edid_blob_ptr->data;
}
```

HDR 能力解析（基于 CEA-861 VSDB）：

```c
// drm_edid.c - HDR 元数据解析
void drm_parse_hdr_metadata_block(struct drm_connector *connector,
    const u8 *db)
{
    struct drm_connector_state *state = connector->state;
    struct hdr_output_metadata *hdr_meta;
    int i;
    
    if (!state)
        return;
    
    // 获取 EOTF 支持
    u8 eotf = db[1];
    
    // 检查是否支持 ST 2084 (HDR10)
    if (eotf & BIT(1)) {
        connector->display_info.hdmi.hdr_supported = true;
        
        // 解析静态元数据
        if (db[2] == 0x01) {  // Static Metadata Type 1
            hdr_meta = &state->hdr_output_metadata;
            
            // 显示主色坐标
            for (i = 0; i < 3; i++) {
                u16 x = (db[3 + i*4] << 2) | (db[3 + i*4 + 2] >> 6);
                u16 y = ((db[3 + i*4 + 1] << 2) | (db[3 + i*4 + 2] >> 2)) & 0x3FF;
                // ... 存储到 hdr_meta
            }
            
            // 最大/最小亮度
            u16 max_lum = (db[15] << 8) | db[16];
            u16 min_lum = (db[17] << 8) | db[18];
            
            connector->display_info.hdr.max_luminance = max_lum;
            connector->display_info.hdr.min_luminance = min_lum;
        }
    }
}
```

#### AMDGPU DC 中 EDID 扩展的深度处理

DC 层对 EDID 的处理进一步深入到硬件适配层面：

```c
// dc/dce/dce_link_encoder.c - EDID 读取
bool dce110_link_encoder_dp_aux_read_edid(
    struct link_encoder *enc,
    struct dc_link *link,
    uint8_t *edid_buf,
    uint32_t *edid_size)
{
    // DP AUX I2C-over-AUX 读取配置
    struct aux_request_transaction_data aux_req;
    struct aux_reply_transaction_data aux_rep;
    int retries = 3;
    
    // 通过 AUX 读取 EDID
    for (int block = 0; block <= edid_buf[0x7E]; block++) {
        uint32_t offset = block * EDID_BLOCK_SIZE;
        
        // 发送 I2C read 请求
        aux_req.type = AUX_TRANSACTION_TYPE_I2C;
        aux_req.address = 0x50;  // EDID I2C 地址
        aux_req.length = EDID_BLOCK_SIZE;
        aux_req.data = &edid_buf[offset];
        
        if (dm_helpers_dp_aux_read(link, &aux_req, &aux_rep) !=
            AUX_TRANSACTION_REPLY_I2C_ACK) {
            // 重试或降级处理
            if (--retries == 0)
                return false;
        }
    }
    
    *edid_size = (1 + edid_buf[0x7E]) * EDID_BLOCK_SIZE;
    return true;
}
```

DC 层 EDID 能力缓存结构：

```c
// dc/inc/core_types.h
struct dc_edid_caps {
    // 原始 EDID 数据
    uint8_t *edid_buf;
    uint32_t edid_size;
    
    // 解析后的显示信息
    uint32_t manufacturer_id;
    uint32_t product_id;
    uint32_t serial_number;
    uint8_t week;
    uint8_t year;
    
    // 显示能力
    uint32_t display_color_depth;
    uint32_t supported_color_formats;
    bool hdr_supported;
    bool freesync_supported;
    
    // 时序限制
    uint32_t max_h_resolution;
    uint32_t max_v_resolution;
    uint32_t max_pixel_clock;
    uint32_t min_v_refresh;
    uint32_t max_v_refresh;
    
    // 物理尺寸
    uint32_t width_mm;
    uint32_t height_mm;
    
    // 音频能力
    bool audio_supported;
    uint32_t speaker_config;
};
```

#### EDID 故障场景与处理

EDID 读取可能失败的各种场景以及内核驱动的处理策略：

```ascii
EDID 故障场景树:
┌─────────────────────────────────────────────────────┐
│                   EDID 读取请求                       │
└─────────────────────┬───────────────────────────────┘
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
    ┌──────────┐           ┌──────────┐
    │ I2C ACK  │           │ NAK/NACK │──► 重试 3 次 → 使用 Fallback EDID
    └─────┬────┘           └──────────┘
          │
   ┌──────┴──────┐
   │             │
   ▼             ▼
┌──────┐   ┌──────────┐
│校验通 │   │校验失败   │──► Checksum Error → 记录 dmesg 警告
└──────┘   │(CRC Error)│      → 尝试读取下一个块
           └──────────┘      → 使用已读取的有效块
                              → 严重: 使用 Fallback EDID

Fallback EDID 定义:
// drm_edid.c - 默认回退 EDID
static const struct edid drm_edid_fallback = {
    .header = {0x00, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0x00},
    .mfg_id = {0x01, 0x01},  // 厂商: "AAA"
    .prod_code = {0x00, 0x00},
    .serial = {0x00, 0x00, 0x00, 0x00},
    .mfg_week = 1,
    .mfg_year = 2013,
    .version = 1,
    .revision = 4,
    .input = 0x80,  // 数字输入
    .width_cm = 0,
    .height_cm = 0,
    .established_timings = {
        0x00, 0x00, 0x00  // 无预置时序
    },
    .detailed_timings = {
        // 1024x768 @ 60Hz fallback
        { .pixel_clock = 6500, .h_active = 1024, .h_blank = 320,
          .v_active = 768, .v_blank = 38, .h_sync_offset = 24,
          .h_sync_pulse_width = 136, .v_sync_offset = 3,
          .v_sync_pulse_width = 6, .misc = 0x06 },
        // ... 更多回退时序
    },
    .checksum = 0x00  // 自动计算
};
```

内核日志中的 EDID 错误信息：

```dmesg
[  123.4567] [drm:edid_block_valid] *ERROR* EDID block 0 checksum is 0x8B, expected 0x00
[  123.4568] [drm:edid_block_valid] *ERROR* EDID block 1 checksum is 0x72, expected 0x00
[  123.4569] [drm:drm_edid_is_valid] *ERROR* EDID checksum invalid, using fallback
[  123.5678] [drm:amdgpu_dm_update_connector] *WARN* Connector DP-1: EDID invalid
[  123.5679] [drm:drm_add_edid_modes] *ERROR* EDID block 2 missing, only 2 blocks available of 4
[  124.8901] [drm:amdgpu_dm_hpd_handler] *INFO* HPD received on connector DP-3, re-reading EDID
```

#### Linux 内核 EDID 工具与调试

内核命令行参数 `drm.edid_firmware`：

```bash
# 使用 firmware EDID 覆盖显示器 EDID（调试用）
# 内核启动参数
drm.edid_firmware=HDMI-A-1:edid/1920x1080.bin
drm.edid_firmware=DP-1:edid/4k_hdr.bin,DP-2:edid/1440p.bin

# 多个显示器逐个指定
drm.edid_firmware=VGA-1:edid/1280x1024.bin,HDMI-A-1:forced
# forced = 强制使用默认 fallback EDID

# 内核源码中的示例 EDID 固件路径:
# drivers/gpu/drm/drm_edid_load.c
# 内置 EDID: 1024x768, 1280x1024, 1600x1200, 1920x1080, 2560x1600
```

实用的 EDID 调试命令集合：

```bash
# 1. 读取连接器的 EDID 原始数据
cat /sys/class/drm/card0-HDMI-A-1/edid > edid.bin
cat /sys/class/drm/card0-DP-1/edid > edid_dp.bin
cat /sys/class/drm/card0-eDP-1/edid > edid_edp.bin

# 2. 使用 edid-decode 解析为可读格式
edid-decode edid.bin
edid-decode /sys/class/drm/card0-HDMI-A-1/edid

# 3. 使用 hexdump 查看原始数据
hexdump -C /sys/class/drm/card0-DP-1/edid
xxd /sys/class/drm/card0-DP-1/edid

# 4. 使用 drm_info 查看显示配置
drm_info | grep -A 30 "Connector: DP-1"

# 5. 查看内核中 EDID 解析日志
dmesg | grep -i edid
dmesg | grep -i "display_info\|add modes"
dmesg | grep -i "CEA extension\|HDR\|VSDB"

# 6. 查看 AMDGPU DC 层 EDID 调试日志
echo 0x1f > /sys/module/drm/parameters/debug
echo 0x4 > /sys/module/amdgpu/parameters/amdgpu_dc_debug_mask
dmesg -w | grep -i edid

# 7. 强制重新读取 EDID（HPD 模拟）
# 触发 HPD 模拟：
echo 1 > /sys/kernel/debug/dri/0/amdgpu_hpd_sim
# 或者 echo connector 到触发重新探测
echo detect > /sys/class/drm/card0-DP-1/trigger

# 8. 使用 I2C 工具直接读取
# 安装 i2c-tools
i2cdetect -l                    # 列出所有 I2C 总线
i2cdetect -y <bus_number>       # 检测设备地址
i2cget -y <bus_number> 0x50 0x00  # 读取 EDID 第一个字节
i2cdump -y <bus_number> 0x50    # dump 全部 EDID 内容

# 9. 使用 modetest 查看连接器属性
modetest -M amdgpu -p | grep -A 50 "Connector: DP-1"
modetest -M amdgpu -e          # 查看 encoder

# 10. EDID 校验（Python 脚本）
python3 -c "
import sys
data = open('edid.bin', 'rb').read()
total = sum(data) % 256
print(f'EDID size: {len(data)} bytes')
print(f'Checksum: {hex(total)}')
print(f'Valid: {total == 0}')
print(f'Extension blocks: {data[0x7E]}')
"
```

使用 `edid-decode` 解析 EDID 的完整输出示例：

```bash
$ edid-decode edid.bin
EDID version: 1.4
Manufacturer: DEL Model: a0a5 Serial: 12345
Made in week 1 of 2022
Display is digital, maximum image size is 60 cm x 34 cm
Gamma: 2.20
DPMS levels: Standby Suspend Off
RGB color display
First detailed timing is preferred timing
Color characteristics:
  Red:   0.6406, 0.3486
  Green: 0.2988, 0.5996
  Blue:  0.1474, 0.0644
  White: 0.3125, 0.3291

Established timings supported:
  640x480 @ 60Hz
  800x600 @ 60Hz
  1024x768 @ 60Hz
  1280x1024 @ 60Hz

Standard timings supported:
  1920x1080 @ 60Hz
  2560x1440 @ 60Hz
  3840x2160 @ 30Hz

Detailed timings (preferred):
  Pixel clock: 533250 kHz
  H: 3840 3888 3920 4000 (48 + 32 + 80 = 160)
  V: 2160 2163 2168 2222 (3 + 5 + 54 = 62)
  Hz: 60.00 (vakt)

Detailed timings (second):
  Pixel clock: 241500 kHz
  H: 2560 2608 2640 2720 (48 + 32 + 80 = 160)
  V: 1440 1443 1448 1482 (3 + 5 + 34 = 42)
  Hz: 59.95

Monitor name: DELL S2721QS
Serial number: ABCDEF12345

CEA-861 Extension block (Block 1, revision 3):
  Supports YCbCr 4:4:4 and YCbCr 4:2:2
  Detailed timings (in preferred order):
    Video data block:
      VIC 16: 1920x1080 @ 60Hz (16:9)
      VIC 31: 1920x1080 @ 50Hz (16:9)
      VIC 32: 1920x1080 @ 24Hz (16:9)
      VIC 33: 1920x1080 @ 25Hz (16:9)
      VIC 4: 1280x720 @ 60Hz (16:9)
    Audio data block:
      LPCM: 2 channels, 16/20/24 bit @ 32/44/48/96 kHz
      AC-3: 6 channels, 32/44/48 kHz @ max 640 kbps
      DTS: 6 channels, 44/48 kHz @ max 1536 kbps
    Vendor-specific data block (HDMI):
      IEEE OUI: 0x000C03
      Source physical address: 1.0.0.0
      Supports AI (ACP, ISRC1, ISRC2)
      Maximum TMDS clock: 600 MHz
      Deep Color: 30-bit, 36-bit
      HDMI 2.1: FRL enabled
        FRL max rate: 4-lane 10Gbps
        DSC: supported
  Detailed timing descriptors:
    DTD: 3840x2160 @ 60Hz (HDR10)
    DTD: 3840x2160 @ 30Hz
    DTD: 2560x1440 @ 60Hz

HDR Static Metadata Block (Type 1):
  Electro-Optical Transfer Function:
    Traditional Gamma: supported
    SMPTE ST2084 (HDR10): supported
  Display primaries:
    Primary 1 (Red):  0.6800, 0.3200
    Primary 2 (Green): 0.2650, 0.6900
    Primary 3 (Blue):  0.1500, 0.0600
  White point: 0.3127, 0.3290
  Max luminance: 600.0000 cd/m²
  Min luminance: 0.0500 cd/m²
  Maximum Content Light Level (MaxCLL): 1000 cd/m²
  Maximum Frame-Average Light Level (MaxFALL): 400 cd/m²

DisplayID Extension block (Block 2):
  DisplayID v1.2
  Data blocks:
    Display Parameters:
      Display type: LCD
      Physical size: 600 x 340 mm
      Pixel density: 147 DPI
    Display Device Data Block:
      Feature flags: Panel Self Refresh 2 supported
                     Adaptive-Sync supported
                     Overdrive supported
```

#### EDID 自定义与固件覆盖

在某些情况下，需要覆盖显示器的 EDID。内核提供 `drm.edid_firmware` 参数支持：

```bash
# 内核 boot 参数示例
GRUB_CMDLINE_LINUX="drm.edid_firmware=HDMI-A-1:edid/4k_hdr.bin"
```

创建自定义 EDID 固件：

```c
// 自定义 4K HDR EDID 固件示例
// 文件: /lib/firmware/edid/4k_hdr.bin

static const unsigned char custom_edid[] = {
    0x00, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0x00,  // Header
    0x4D, 0x10,  // Manufacturer: DEL
    0x14, 0x01,  // Product code
    0x01, 0x00, 0x00, 0x00,  // Serial
    0x01, 0x16,  // Week 1, Year 2022
    0x01, 0x04,  // EDID v1.4
    0xA5,  // Digital input, RGB 4:4:4 + YCbCr 4:4:4
    0x3C, 0x22,  // 60cm x 34cm
    0x78,  // Gamma 2.2
    0xEA,  // Features: DPMS, RGB, preferred timing
    // ... 完整的 128 字节 base block ...
    // ... CEA-861 扩展块包含 HDR 元数据 ...
    0x00,  // Checksum (自动计算)
};

// 如果需要在 AMDGPU 驱动中启用固件 EDID
// echo 1 > /sys/module/drm/parameters/edid_firmware_override
```

EDID 固件覆盖的典型使用场景：

```ascii
┌────────────────────────────────────────────────────────────────┐
│ EDID 固件覆盖使用场景                                          │
├────────────────────────────────────────────────────────────────┤
│ 场景 1: 显示器 EDID 损坏但基本功能正常                          │
│   ─ 症状: 1440p 显示器只能检测到 1024x768                      │
│   ─ 方案: 从同型号正常设备提取 EDID 固件覆盖                   │
│   ─ 命令: cat /sys/class/drm/card0-DP-1/edid > /lib/firmware/  │
│           edid/dell_s2721qs.bin                                │
│           drm.edid_firmware=DP-1:edid/dell_s2721qs.bin        │
├────────────────────────────────────────────────────────────────┤
│ 场景 2: 工程样片/开发板需要特定分辨率的 EDID                    │
│   ─ 需求: 4096x2160 @ 60Hz 测试模式                           │
│   ─ 方案: 使用内核内置的 EDID 或自定义                         │
│   ─ 命令: drm.edid_firmware=HDMI-A-1:edid/4096x2160.bin      │
├────────────────────────────────────────────────────────────────┤
│ 场景 3: 显示器 EDID 报告了错误的色彩深度                       │
│   ─ 症状: 10-bit 面板被限制为 8-bit                           │
│   ─ 方案: 修改 EDID 中色彩深度字段并覆盖                      │
│   ─ 手动: 将字节 0x18 bits 重新编码后生成固件                  │
└────────────────────────────────────────────────────────────────┘
```

#### FreeSync EDID 标志检测

FreeSync 能力通常在 EDID 的 DisplayID 或 CEA-861 扩展块中通过 VSDB 宣告：

```c
// amdgpu_dm.c - FreeSync EDID 检测
bool amdgpu_dm_connector_is_freesync_capable(
    struct amdgpu_dm_connector *aconnector)
{
    struct drm_connector *connector = &aconnector->base;
    struct edid *edid;
    
    if (!connector->edid_blob_ptr)
        return false;
    
    edid = (struct edid *)connector->edid_blob_ptr->data;
    
    // 搜索 CEA-861 VSDB 中的 FreeSync 标志
    for (int i = 0; i < edid->extensions; i++) {
        const u8 *cea = (const u8 *)edid + (i + 1) * EDID_LENGTH;
        
        if (cea[0] != 0x02)  // 不是 CEA-861 扩展块
            continue;
        
        // 遍历 data blocks
        int offset = 4;
        while (offset < cea[2] && offset < EDID_LENGTH) {
            u8 tag = cea[offset] >> 5;
            u8 len = cea[offset] & 0x1F;
            
            if (tag == 3) {  // VSDB
                u32 oui = (cea[offset+1] << 16) |
                          (cea[offset+2] << 8) |
                          cea[offset+3];
                
                // AMD OUI: 0x00001A 或 0xAA55
                if (oui == 0x00001A || oui == AMD_FREESYNC_OUI) {
                    // 检查 FreeSync 支持位
                    if (cea[offset+4] & FREESYNC_SUPPORTED) {
                        return true;
                    }
                }
            }
            offset += len + 1;
        }
    }
    
    return false;
}
```

#### E-EDID (Enhanced EDID) 与 DisplayID

新一代显示器越来越多地使用 DisplayID 替代传统 EDID 扩展块：

```c
// drm_displayid.c - DisplayID 解析
struct drm_displayid *drm_get_displayid(struct drm_connector *connector,
    const struct edid *edid)
{
    const u8 *displayid_block;
    
    // DisplayID 位于 CEA-861 扩展块之后
    // 或在单独的 DisplayID 扩展块 (Tag 0x70) 中
    
    for (int i = 0; i < edid->extensions; i++) {
        const u8 *block = (const u8 *)edid + (i + 1) * EDID_LENGTH;
        
        if (block[0] == 0x70) {  // DisplayID Extension
            return parse_displayid_block(block, EDID_LENGTH);
        }
    }
    
    return NULL;
}

// DisplayID 对比 EDID 的优势:
// 1. 每个块类型有唯一的 UUID，不会冲突
// 2. 可变长度数据块，更灵活
// 3. 支持更大数据量（不仅是 128 字节块）
// 4. 更好的扩展性，支持自适应同步、HDR 等新特性
// 5. 专为现代显示器设计的时序描述
```

#### EDID 在 IGT 测试中的应用

IGT（Intel GPU Tools）中的 EDID 测试套件：

```bash
# IGT EDID 相关测试
sudo igt_runner --test edid

# 单项测试
sudo testdisplay -e edid     # EDID 读取测试
sudo kms_edid_feat -f        # EDID 功能解析测试
sudo kms_hdr -d              # HDR EDID 解析测试

# 模拟 EDID 读取错误测试
sudo kms_edid_feat -i        # 无效 EDID 测试
sudo kms_edid_feat -c        # 校验和错误测试
```

#### EDID 自动化测试脚本

```python
#!/usr/bin/env python3
"""
EDID 自动化测试脚本
测试目标: 验证 EDID 读取、解析和校验
"""

import subprocess
import sys
import os
import json

class EDIDTest:
    def __init__(self):
        self.results = {
            'passed': 0,
            'failed': 0,
            'skipped': 0,
            'tests': []
        }
        
    def run_cmd(self, cmd):
        try:
            result = subprocess.run(
                cmd, shell=True, capture_output=True, text=True, timeout=10
            )
            return result.returncode, result.stdout, result.stderr
        except subprocess.TimeoutExpired:
            return -1, '', 'Timeout'
        except Exception as e:
            return -1, '', str(e)
    
    def test_edid_read_all_connectors(self):
        """测试所有连接器的 EDID 是否可读取"""
        print("\n[TEST] 测试所有连接器的 EDID 可读性")
        
        rc, out, err = self.run_cmd(
            "ls /sys/class/drm/*/edid 2>/dev/null"
        )
        
        connectors = out.strip().split('\n')
        if not connectors or connectors == ['']:
            print("  [SKIP] 没有找到 EDID 文件")
            self.results['skipped'] += 1
            return
        
        for edid_path in connectors:
            connector_name = edid_path.split('/')[-2]
            rc, data, err = self.run_cmd(f"cat {edid_path} 2>/dev/null | wc -c")
            
            size = int(data.strip())
            if size > 0 and size % 128 == 0:
                print(f"  [PASS] {connector_name}: EDID size = {size} bytes ({size//128} blocks)")
                self.results['passed'] += 1
            else:
                print(f"  [FAIL] {connector_name}: Invalid EDID size = {size}")
                self.results['failed'] += 1
    
    def test_edid_checksum(self):
        """测试 EDID 校验和"""
        print("\n[TEST] 测试 EDID 校验和")
        
        rc, out, err = self.run_cmd("ls /sys/class/drm/*/edid")
        connectors = out.strip().split('\n')
        
        if not connectors or connectors == ['']:
            print("  [SKIP] 没有找到 EDID 文件")
            self.results['skipped'] += 1
            return
        
        for edid_path in connectors:
            connector = edid_path.split('/')[-2]
            rc, data, err = self.run_cmd(f"cat {edid_path}")
            
            raw = data.encode('latin-1')
            valid = True
            
            # 检查每个块的校验和
            for block in range(len(raw) // 128):
                offset = block * 128
                csum = sum(raw[offset:offset+128]) % 256
                if csum != 0:
                    print(f"  [FAIL] {connector}: Block {block} checksum = {csum} (expected 0)")
                    valid = False
                    self.results['failed'] += 1
            
            if valid:
                print(f"  [PASS] {connector}: All {len(raw)//128} blocks checksum OK")
                self.results['passed'] += 1
    
    def test_edid_parse_with_edid_decode(self):
        """使用 edid-decode 解析 EDID"""
        print("\n[TEST] 使用 edid-decode 解析 EDID")
        
        rc, out, err = self.run_cmd("which edid-decode")
        if rc != 0:
            print("  [SKIP] edid-decode 未安装")
            self.results['skipped'] += 1
            return
        
        rc, out, err = self.run_cmd("ls /sys/class/drm/*/edid")
        connectors = out.strip().split('\n')
        
        for edid_path in connectors[:2]:  # 只测试前两个
            connector = edid_path.split('/')[-2]
            rc, out, err = self.run_cmd(f"edid-decode {edid_path} 2>/dev/null | head -30")
            
            if rc == 0 and 'Manufacturer' in out:
                print(f"  [PASS] {connector}: edid-decode 解析成功")
                for line in out.split('\n')[:5]:
                    print(f"         {line}")
                self.results['passed'] += 1
            else:
                print(f"  [FAIL] {connector}: edid-decode 解析失败")
                if err:
                    print(f"         Error: {err.strip()}")
                self.results['failed'] += 1
    
    def test_hdr_edid_support(self):
        """测试 HDR EDID 元数据"""
        print("\n[TEST] 测试 HDR 支持检测")
        
        # 使用 modetest 获取 HDR 属性
        rc, out, err = self.run_cmd(
            "modetest -M amdgpu -p 2>/dev/null | grep -A 5 'HDR_OUTPUT_METADATA'"
        )
        
        if 'HDR_OUTPUT_METADATA' in out:
            print("  [PASS] 检测到 HDR 输出元数据属性")
            self.results['passed'] += 1
        else:
            print("  [SKIP] 未检测到 HDR 属性（连接器可能不支持）")
            self.results['skipped'] += 1
    
    def test_edid_kernel_messages(self):
        """检查内核日志中的 EDID 错误"""
        print("\n[TEST] 检查内核日志中的 EDID 错误")
        
        rc, out, err = self.run_cmd("dmesg | grep -i 'edid' | tail -20")
        
        if 'checksum' in out.lower() or 'invalid' in out.lower() or 'error' in out.lower():
            print("  [WARN] 检测到 EDID 相关错误:")
            for line in out.split('\n')[:5]:
                print(f"         {line}")
            self.results['failed'] += 1
        elif out:
            print(f"  [PASS] 内核 EDID 日志正常 (共 {len(out.split(chr(10)))} 条)")
            self.results['passed'] += 1
        else:
            print("  [SKIP] 未找到 EDID 相关日志")
            self.results['skipped'] += 1
    
    def run_all(self):
        """运行所有测试"""
        print("=" * 60)
        print("EDID 自动化测试套件")
        print("=" * 60)
        
        self.test_edid_read_all_connectors()
        self.test_edid_checksum()
        self.test_edid_parse_with_edid_decode()
        self.test_hdr_edid_support()
        self.test_edid_kernel_messages()
        
        print("\n" + "=" * 60)
        print("测试结果汇总")
        print(f"  PASS:   {self.results['passed']}")
        print(f"  FAIL:   {self.results['failed']}")
        print(f"  SKIP:   {self.results['skipped']}")
        print(f"  总计:   {self.results['passed'] + self.results['failed'] + self.results['skipped']}")
        print("=" * 60)
        
        return self.results['failed'] == 0


if __name__ == "__main__":
    test = EDIDTest()
    success = test.run_all()
    sys.exit(0 if success else 1)
```

### 案例分析

#### 案例 1：EDID 损坏导致分辨率异常

**问题描述**：用户报告一台 Dell U2719D（2560×1440）在连接 AMD RX 6800 后，最大只能设置 1920×1080。

**排查过程**：

```bash
# 步骤 1: 检查 EDID 原始数据
$ hexdump -C /sys/class/drm/card0-DP-1/edid
00000000  00 ff ff ff ff ff ff 00  10 ac 5e b0 00 00 00 00  |..........^.....|
00000010  01 1e 01 04 b5 3c 22 78  0a ee 91 a3 54 4c 99 26  |.....<"x....TL.&|
...
00000070  00 00 00 fc 00 44 45 4c  4c 20 55 32 37 31 39 44  |.....DELL U2719D|
00000080  f4 5a 80 a0 70 38 2d 40  30 20 35 00 55 50 21 00  |.Z..p8-@0 5.UP!.|
...
000000f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 8a  |................|

# 步骤 2: 检查校验和
$ python3 -c "
data = open('/sys/class/drm/card0-DP-1/edid', 'rb').read()
for i in range(len(data)//128):
    block = data[i*128:(i+1)*128]
    csum = sum(block) % 256
    print(f'Block {i}: checksum = {csum}, valid = {csum == 0}')
"
Block 0: checksum = 0, valid = True
Block 1: checksum = 0x5A, valid = False  <-- 扩展块损坏！

# 步骤 3: 使用 edid-decode 确认
$ edid-decode /sys/class/drm/card0-DP-1/edid
EDID block 1 checksum is 0x5A, expected 0x00
EDID block 1 is corrupt

# 步骤 4: 检查内核日志
$ dmesg | grep -i edid
[ 1234.567] [drm] EDID block 1 checksum is 0x5A, expected 0x00
[ 1234.567] [drm] Falling back to base block only
```

**根因**：EDID 第 1 块的校验和损坏（0x5A ≠ 0x00），驱动回退到 Base Block，缺少了 CEA-861 扩展块中的 2560×1440 时序。

**解决方案**：

```bash
# 方案 1: 使用 edid-decode 修复 EDID
$ edid-decode --fix-checksum edid_dump.bin -o edid_fixed.bin

# 方案 2: 使用固件 EDID 覆盖
$ cp edid_fixed.bin /lib/firmware/edid/dell_u2719d.bin
# 在 /etc/default/grub 中添加:
# GRUB_CMDLINE_LINUX="drm.edid_firmware=DP-1:edid/dell_u2719d.bin"
$ update-grub
$ reboot

# 验证修复效果
$ cat /sys/class/drm/card0-DP-1/modes
2560x1440
1920x1080
...
```

#### 案例 2：通过 EDID 配置 HDR 输出

**问题描述**：LG C2 OLED 电视通过 HDMI 2.1 连接 RX 7900 XTX，HDR 选项显示为灰色不可选。

**排查过程**：

```bash
# 步骤 1: 检查 EDID 是否包含 HDR 元数据
$ edid-decode /sys/class/drm/card0-HDMI-A-1/edid | grep -A 20 "HDR"
HDR Static Metadata Block (Type 1):
  Electro-Optical Transfer Function:
    Traditional Gamma: supported
    SMPTE ST2084 (HDR10): NOT supported  <-- 问题所在！
  Display primaries:
    ...

# 步骤 2: 检查色彩深度
$ edid-decode /sys/class/drm/card0-HDMI-A-1/edid | grep "Deep Color"
Deep Color: 30-bit, 36-bit

# 步骤 3: 检查连接器属性
$ modetest -M amdgpu -p | grep -A 10 "HDMI-A-1"
  HDR_OUTPUT_METADATA: enum {0, 1} = 0
  Colorspace: enum {Default, BT709, ...} = Default
  max_bpc: range [8, 16] = 8
```

**根因**：HDMI 线缆是 HDMI 2.0 版本（18 Gbps），虽然物理连接正常，但 HDMI 2.0 线缆无法传输 HDR 所需的 48 Gbps 带宽。EDID 检测到 TMDS 时钟限制，降级为 SDR 模式。

**解决方案**：

```bash
# 确认线缆规格
$ dmesg | grep -i "tmds"
[ 567.890] [drm] Detected HDMI 2.0 cable, max TMDS clock 600MHz

# 更换 HDMI 2.1 认证线缆后的对比输出
$ edid-decode /sys/class/drm/card0-HDMI-A-1/edid | grep "FRL"
FRL: 4-lane 12Gbps supported  <-- 正确识别 HDMI 2.1 能力
```

#### 案例 3：多显示器 EDID 冲突

**场景**：同时连接三台显示器，其中两台 EDID 报告了相同的 PNP ID，导致驱动混淆。

```bash
# 排查命令
$ for c in /sys/class/drm/card0-*/edid; do
    echo "=== $c ==="
    edid-decode $c 2>/dev/null | grep "Manufacturer\|Product\|Serial"
  done

=== /sys/class/drm/card0-DP-1/edid ===
  Manufacturer: DEL Model: a0a5 Serial: 12345
=== /sys/class/drm/card0-DP-2/edid ===
  Manufacturer: DEL Model: a0a5 Serial: 12345  <-- 相同序列号？
=== /sys/class/drm/card0-HDMI-A-1/edid ===
  Manufacturer: SAM Model: 1234 Serial: 67890
```

**解决方案**：通过 EDID 中的物理地址区别连接器位置，并在驱动中使用 connector ID 而非 EDID 序列号作为唯一标识。

### 相关链接

- VESA EDID v1.4 标准文档: https://vesa.org/vesa-standards/
- CEA-861-G 消费电子视频标准: https://shop.cta.tech/products/cta-861-g
- DisplayID v2.0 标准: https://vesa.org/featured-projects/displayid/
- Linux 内核 EDID 文档: https://docs.kernel.org/gpu/drm-kms.html
- edid-decode 工具主页: https://gitlab.freedesktop.org/emersion/edid-decode
- AMDGPU EDID 调试文档: https://docs.kernel.org/gpu/amdgpu/display/display-debug.html
- IGT GPU Tools EDID 测试: https://gitlab.freedesktop.org/drm/igt-gpu-tools

### 今日小结

今日重点内容回顾：

1. **EDID 数据结构**：EDID 由 128 字节的 Base Block 和若干扩展块组成，包含显示器标识、物理特性、时序支持、色彩能力等信息。

2. **Base Block 核心字段**：Header（固定魔数）、Manufacturer ID（3 字符 PNP 编码）、Basic Display Parameters（信号类型/物理尺寸）、Chromaticity Coordinates（RGGBW 色度坐标）、Established Timings（标准时序位图）、Standard Timings（8 个自定义时序）、Detailed Timing Descriptors（4 个 18 字节描述符）。

3. **扩展块类型**：CEA-861 扩展块（Tag 0x02）包含音频格式/视频 VICs/HDR 元数据/HDMI VSDB；DisplayID 扩展块（Tag 0x70）提供更灵活的描述能力。

4. **EDID 读取机制**：通过 I2C DDC 通道（HDMI 的 DDC 引脚或 DP 的 I2C-over-AUX）读取，内核从 I2C 地址 0x50 读取二进制数据。

5. **AMDGPU DC 处理流程**：HPD 事件触发 → dc_link_detect() 读取 EDID → drm_connector_update_edid_property() 更新 DRM 属性 → drm_add_edid_modes() 生成 mode list → drm_add_display_info() 解析显示能力。

6. **EDID 校验与故障恢复**：校验和检查确保数据完整性，损坏时回退到 Base Block 或使用 Fallback EDID。内核支持 `drm.edid_firmware` 参数覆盖 EDID。

7. **调试工具**：edid-decode（完整解析）、hexdump/xxd（原始查看）、modetest（连接器属性）、i2c-tools（直接读取）、dmesg（内核日志）。

8. **FreeSync/HDR 检测**：通过 CEA-861 VSDB 中厂商特定 OUI（AMD OUI 0x00001A）检测 FreeSync 支持；通过 HDR Static Metadata Data Block 检测 HDR10/HLG 支持。

### 扩展思考

1. **EDID 安全性**：EDID 固件覆盖是否可能被用于恶意篡改显示配置？是否存在 EDID 仿冒攻击（Malicious EDID）的风险？

2. **EDID 与 DisplayID 的未来**：随着 USB-C 和 DP Alt Mode 的普及，DisplayID 是否会完全取代传统 EDID？EDID 的 128 字节块限制在现代高分辨率显示器前是否成为瓶颈？

3. **EDID 在虚拟化场景中的应用**：GPU Passthrough 中，EDID 如何从物理显示器传递到虚拟机？vGPU 方案如何模拟 EDID 以支持无头服务器？

4. **自适应同步的 EDID 协商**：FreeSync Premium Pro 与 G-Sync Compatible 的 EDID 检测方式有何异同？显示器厂商如何通过 EDID 实现两者兼容？

5. **HDR 动态元数据**：除了 EDID 中定义的 HDR10 静态元数据，Dolby Vision 和 HDR10+ 的动态元数据在 EDID 外如何传递（通过 HDMI Venderspecific InfoFrame 或 DP SDP）？