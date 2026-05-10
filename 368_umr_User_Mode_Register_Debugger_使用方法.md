# umr（User Mode Register Debugger）使用方法

## 学习目标

- 理解 umr 工具的定位和功能范围，掌握其与 debugfs 的关系和差异
- 掌握 umr 的安装和基本使用方法，包括寄存器读写、波形转储等核心功能
- 学习使用 umr 进行 GPU 寄存器遍历、反汇编和性能分析
- 掌握 umr 在驱动开发和调试中的高级用法
- 了解 umr 的架构设计和扩展机制

## 知识详解

### 概念原理

#### umr 工具概述

umr (User Mode Register Debugger) 是 AMD 提供的用户空间 GPU 调试工具，提供比 debugfs 更丰富的功能：

```ascii
umr 工具架构:

┌──────────────────────────────────────────────────────────────┐
│  用户空间                                                      │
│                                                               │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  umr 命令行工具                                            │ │
│  │                                                           │ │
│  │  ┌─────────┐ ┌──────────┐ ┌─────────┐ ┌──────────────┐ │ │
│  │  │ 寄存器   │ │ 波形     │ │ 反汇编   │ │ 性能计数器    │ │ │
│  │  │ 读/写   │ │ 转储     │ │ 引擎     │ │ 读取          │ │ │
│  │  └────┬────┘ └────┬─────┘ └────┬────┘ └──────┬───────┘ │ │
│  └───────┼───────────┼────────────┼──────────────┼──────────┘ │
│          │           │            │              │            │
│  ┌───────▼───────────▼────────────▼──────────────▼──────────┐ │
│  │  umr 内核模块 (amdgpu_umr.ko)                            │ │
│  │  - 提供用户空间到硬件的安全访问通道                         │ │
│  │  - 寄存器同步/异步访问                                     │ │
│  │  - 完整 MMIO/SMN 空间访问                                  │ │
│  │  - 支持 SR-IOV VF 调试                                     │ │
│  └───────────────────────┬──────────────────────────────────┘ │
│                          │                                    │
├──────────────────────────┼────────────────────────────────────┤
│  内核空间                 │                                    │
│                          │                                    │
│  ┌──────────────────────▼──────────────────────────────────┐  │
│  │  AMDGPU 驱动 / KMD                                     │  │
│  │  - RREG32/WREG32 (MMIO)                                │  │
│  │  - amdgpu_smn_rreg/wreg (SMN)                          │  │
│  │  - pci_read/write_config (PCIE)                        │  │
│  └──────────────────────┬──────────────────────────────────┘  │
│                         │                                      │
├─────────────────────────┼──────────────────────────────────────┤
│  硬件层                  │                                      │
│  ┌──────────────────────▼──────────────────────────────────┐  │
│  │  GPU ASIC                                                │  │
│  │  - 寄存器文件 (Register File)                            │  │
│  │  - 性能计数器 (Performance Counter)                      │  │
│  │  - 波形快照 (Wave Snapshot)                              │  │
│  └─────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘

umr vs debugfs 对比:
┌──────────────────────────────────────────────────────────────┐
│ 特性            │ debugfs amdgpu_regs     │ umr               │
├──────────────────┼──────────────────────┼─────────────────────┤
│ 寄存器读取       │ ✅ 基本 MMIO/SMN      │ ✅ 高级 (含描述)     │
│ 波形转储         │ ❌                    │ ✅ 完整 Wave state  │
│ 反汇编           │ ❌                    │ ✅ GCN/RDNA 指令    │
│ 性能计数器       │ ❌                    │ ✅ 完整支持          │
│ 寄存器描述       │ 无                   │ ✅ 从头文件自动生成  │
│ 块/实例遍历      │ 手动                  │ ✅ 自动              │
│ SR-IOV VF 调试   | ❌                    │ ✅ 支持              │
│ 图形界面         │ 无                   │ ✅ Tcl/Tk GUI       │
│ 安装复杂度       │ 内置 (内核自带)        │ 需单独编译安装       │
└──────────────────────────────────────────────────────────────┘
```

### 实践操作

#### 安装 umr

```bash
# 1. 克隆 umr 源码
git clone https://gitlab.freedesktop.org/tomstdenis/umr.git
cd umr

# 2. 安装依赖
# Debian/Ubuntu
sudo apt-get install build-essential libpciaccess-dev \
    libkmod-dev libdrm-dev python3 python3-pip \
    flex bison tcl-dev tk-dev

# Fedora/RHEL
sudo dnf install gcc make pciaccess-devel libkmod-devel \
    libdrm-devel python3 flex bison tcl-devel tk-devel

# 3. 编译 umr
make clean
make -j$(nproc)

# 4. 安装
sudo make install

# 5. 安装 umr 内核模块 (可选，用于更完整的硬件访问)
# umr 默认通过 dev_mem / sysfs 访问，但某些功能需要内核模块
cd kernel
make
sudo insmod umr_kmod.ko

# 6. 验证安装
umr --version
# 输出: umr version: 3.0.0

# 7. 列出支持的 GPU
umr -l
# 输出:
# Supported ASICs:
#   - gfx1030 (Sienna Cichlid / RX 6700 XT)
#   - gfx1100 (Navi 31 / RX 7900 XTX)
#   - gfx1102 (Navi 33 / RX 7600)
#   - ...
```

#### 基本命令

```bash
# 1. 列出系统 GPU 和 IP 块
sudo umr -L

# 输出:
# GPU 0: gfx1100 [Navi 31]
#   GFX: gfx1100, REV: 0, 8 SE, 4 RB
#   SDMA: sdma6, 2 engines
#   DCN: dcn3.2, 5 pipes
#   VCN: vcn4.0, decode/encode
#   SMU: smu13, firmware v.85.0.0
#   MMHUB: mmhub4.0

# 2. 读取单个寄存器 (按名称或偏移)
# 按名称读取
sudo umr -r mmOTG_MASTER_EN

# 输出:
# GPU0: mmOTG_MASTER_EN[0] = 0x00000001

# 按偏移读取
sudo umr -r 0x1A000

# 3. 读取指定块的寄存器
sudo umr -r mmOTG_MASTER_EN -b dcn

# 输出块下所有实例:
# GPU0 dcn0: mmOTG_MASTER_EN[0] = 0x00000001
# GPU0 dcn0: mmOTG_MASTER_EN[1] = 0x00000000

# 4. 写入寄存器 (危险！)
sudo umr -w mmOTG_MASTER_EN=0

# 5. 批量寄存器读取 (扫描块)
sudo umr -s dcn

# 6. 查看寄存器描述
sudo umr -d mmOTG_MASTER_EN

# 输出:
# mmOTG_MASTER_EN - OTG Master Enable Register
#   Offset: 0x1A000
#   Type: read-write
#   Fields:
#     [1:0]   OTG_MASTER_EN: 0 = disabled, 1 = enabled
#     [31:2]  Reserved
#   Default: 0x00000000
```

#### 寄存器遍历

```bash
# 1. 扫描指定 IP 块的所有寄存器
sudo umr -S dcn

# 输出 (部分):
# GPU0 dcn: Register Summary
# ┌──────────────────┬──────────┬────────────┬──────────────────┐
# │ Register         │ Offset   │ Value     │ Description       │
# ├──────────────────┼──────────┼────────────┼──────────────────┤
# │ OTG_MASTER_EN    │ 0x1A000  │ 0x00000001 │ OTG 主使能        │
# │ OTG_CONTROL      │ 0x1A004  │ 0x00000000 │ OTG 控制          │
# │ OTG_H_TOTAL      │ 0x1A008  │ 0x00001B58 │ 水平总像素        │
# │ OTG_V_TOTAL      │ 0x1A00C  │ 0x00000438 │ 垂直总行数        │
# │ OTG_H_SYNC_A     │ 0x1A010  │ 0x00000058 │ 水平同步开始      │
# │ OTG_H_SYNC_B     │ 0x1A014  │ 0x000000C8 │ 水平同步结束      │
# │ OTG_V_SYNC_A     │ 0x1A018  │ 0x0000000A │ 垂直同步开始      │
# │ OTG_V_SYNC_B     │ 0x1A01C  │ 0x00000014 │ 垂直同步结束      │
# │ OTG_PIXEL_RATE   │ 0x1A020  │ 0x00000000 │ 像素时钟          │
# │ OTG_V_UPDATE     │ 0x1A284  │ 0x00001000 │ VUpdate 位置      │
# │ OTG_INTERLACE    │ 0x1A288  │ 0x00000000 │ 隔行扫描          │
# │ ... (more)       │          │            │                  │
# └──────────────────┴──────────┴────────────┴──────────────────┘

# 2. 按实例遍历 (多 pipe 场景)
sudo umr -S dcn --instance 0

# 3. 差异扫描 (比较两个块的值)
sudo umr -S dcn --diff dcn_snapshot_before.bin

# 4. 保存/加载寄存器快照
sudo umr -S dcn --snap dcn_snapshot.bin
sudo umr -S dcn --load dcn_snapshot.bin  # 写入快照 (危险!)
```

#### 波形转储与分析

```bash
# 1. 捕获当前 GPU 着色器波形
sudo umr --waves

# 输出:
# GPU0: Wave Snapshot
# ┌──────┬─────┬──────┬────────┬────────┬────────┬──────────┐
# │ SE   │ SH  │ CU   │ WaveID │ Status │ PC     │ Insts    │
# ├──────┼─────┼──────┼────────┼────────┼────────┼──────────┤
# │ 0    │ 0   │ 0    │ 0      │ ACTIVE │ 0x1400 │ 124      │
# │ 0    │ 0   │ 0    │ 1      │ ACTIVE │ 0x1A00 │ 87       │
# │ 0    │ 0   │ 1    │ 0      │ STALLED│ 0x800  │ 23       │
# │ 0    │ 0   │ 1    │ 1      │ IDLE   │ 0x0    │ 0        │
# │ ...  │     │      │        │        │        │          │
# │ 3    │ 1   │ 7    │ 3      │ ACTIVE │ 0x2C00 │ 245      │
# └──────┴─────┴──────┴────────┴────────┴────────┴──────────┘
# Total waves: 12 / 40 (30% utilization)

# 2. 详细波形状态
sudo umr --waves --verbose

# 显示每个波形的:
# - PC (程序计数器)
# - 执行掩码 (EXEC)
# - VGPR 状态
# - SGPR 状态
# - LDS 使用量
# - 指令计数

# 3. 波形反汇编 (PC 地址指向的指令)
sudo umr --waves --disassemble

# 输出:
# Wave 0: PC = 0x1400
#   0x1400: S_LOAD_DWORDX4 s[8:11], s[0:1], 0x10
#   0x1404: V_ADD_CO_U32 v1, vcc, s8, v0
#   0x1408: V_ADD_CO_U32 v2, vcc, s9, v1
#   0x140C: S_WAITCNT 49279
#   0x1410: V_ADD_F32 v3, v2, v1
#   0x1414: V_MUL_F32 v4, v3, s12
#   0x1418: EXPORT v4, 0, row, 0

# 4. 持续波形监控
sudo umr --waves --monitor 100

# 每秒采样一次波形状态，持续 100 次
# 用于发现瞬时 GPU 瓶颈

# 5. 波形触发捕获 (条件断点)
sudo umr --waves --trigger "PC == 0x1400" --capture
```

#### 性能计数器

```bash
# 1. 列出可用性能计数器
sudo umr --perf list

# 输出:
# GPU0 Performance Counters:
# ┌──────────────────┬──────────────────────────────────────────┐
# │ Counter          │ Description                              │
# ├──────────────────┼──────────────────────────────────────────┤
# │ gfx_VALID        │ 有效像素数                                │
# │ gfx_PRIM         │ 处理的图元数                              │
# │ gfx_CLOCK        │ GFX 时钟周期                              │
# │ gfx_BUSY         │ GFX 忙碌时间                              │
# │ gfx_ACTIVE       │ GFX 活跃周期                              │
# │ gfx_VERT         │ 处理的顶点数                              │
# │ gfx_RASTER       │ 光栅化像素数                              │
# │ gfx_CACHE_HIT    │ L1/L2 Cache 命中率                        │
# │ gfx_MEM_READ     │ 内存读取事务数                            │
# │ gfx_MEM_WRITE    │ 内存写入事务数                            │
# │ dcn_PIXEL_CLK    │ 显示像素时钟                              │
# │ dcn_VBLANK       │ VBlank 周期                               │
# │ dcn_FRAME        │ 帧计数器                                  │
# │ pcie_TX          │ PCIE 发送带宽                             │
# │ pcie_RX          │ PCIE 接收带宽                             │
# │ mem_READ_BW      │ 显存读取带宽                              │
# │ mem_WRITE_BW     │ 显存写入带宽                              │
# └──────────────────┴──────────────────────────────────────────┘

# 2. 启用并读取性能计数器
# 采样时间 1 秒
sudo umr --perf enable gfx_VALID,gfx_PRIM,gfx_CLOCK,gfx_BUSY
sudo umr --perf sample 1000

# 输出:
# GPU0 Performance Counters (1000ms sample):
#   gfx_VALID:    12456789 pixels
#   gfx_PRIM:     34567 primitives
#   gfx_CLOCK:    2500000000 cycles @ 2.5 GHz
#   gfx_BUSY:     85.3% (2132ms busy / 2500ms total)
#   ---
#   Derived metrics:
#   Pixel fillrate:  498.3 MPixels/sec
#   Primitive rate:  34.6 MPrims/sec
#   Utilization:     85.3%

# 3. 自定义计数器组合
sudo umr --perf enable --config counter_config.json
# counter_config.json:
# {
#   "counters": ["gfx_VALID", "gfx_BUSY"],
#   "sample_ms": 500,
#   "events": [
#     {"name": "vertex_load", "counter": "gfx_VERT", "threshold": 1000000},
#     {"name": "cache_miss", "counter": "gfx_CACHE_HIT", "threshold": 50}
#   ]
# }

# 4. 持续性能监控
sudo umr --perf enable gfx_BUSY --monitor
# 持续输出每 100ms 的 GFX 利用率
# 输出:
#  [100ms] gfx_BUSY = 45.2%
#  [200ms] gfx_BUSY = 67.8%
#  [300ms] gfx_BUSY = 91.2%
#  [400ms] gfx_BUSY = 88.5%
#  ...
```

#### I/O 波形跟踪

```bash
# 1. 跟踪 GPU I/O 操作
sudo umr --trace --io

# 输出:
# GPU0 I/O Trace:
# ┌───────┬──────────┬────────┬──────────┬──────────────┐
# │ Time  │ Type     │ Offset │ Value    │ Description   │
# ├───────┼──────────┼────────┼──────────┼──────────────┤
# │ 0.001 │ MMIO RD  │ 0x8000 │ 0x000001 │ GRBM_STATUS   │
# │ 0.001 │ MMIO RD  │ 0x8004 │ 0x000000 │ GRBM_STATUS2  │
# │ 0.002 │ MMIO WR  │ 0x1A004│ 0x000001 │ OTG_CONTROL   │
# │ 0.003 │ MMIO WR  │ 0x1A008│ 0x001B58 │ OTG_H_TOTAL   │
# │ 0.003 │ PCIE CFG │ 0x0010 │ 0x000000 │ BAR0          │
# │ ...   │          │        │          │              │
# └───────┴──────────┴────────┴──────────┴──────────────┘

# 2. 过滤器 (跟踪特定范围)
sudo umr --trace --io --filter "dcn,0x1A000-0x1B000"

# 3. 跟踪并保存为 pcap 格式 (Wireshark 分析)
sudo umr --trace --io --pcap gpu_trace.pcap

# 4. 跟踪 DMA 操作
sudo umr --trace --dma

# 输出:
# GPU0 DMA Trace:
# ┌───────┬──────────┬──────────┬──────────┬──────────┐
# │ Time  │ Engine   │ Src Addr │ Dst Addr │ Size     │
# ├───────┼──────────┼──────────┼──────────┼──────────┤
# │ 0.001 │ SDMA0    │ 0x100000 │ 0x200000 │ 0x1000   │
# │ 0.002 │ SDMA0    │ 0x200000 │ 0x300000 │ 0x2000   │
# └───────┴──────────┴──────────┴──────────┴──────────┘
```

#### 高级寄存器操作

```bash
# 1. 按块读取寄存器
sudo umr -B dcn -r mmOTG_MASTER_EN,mmOTG_H_TOTAL,mmOTG_V_TOTAL

# 2. 十六进制批量读取
sudo umr -x 0x1A000 16

# 输出:
# 0x0001A000: 01 00 00 00 00 00 00 00 58 1B 00 00 38 04 00 00
# 0x0001A010: 58 00 00 00 C8 00 00 00 0A 00 00 00 14 00 00 00

# 3. 寄存器位域操作
# 读取 OTG_MASTER_EN 的位域 [1:0]
sudo umr -r mmOTG_MASTER_EN --field "OTG_MASTER_EN"

# 输出: OTG_MASTER_EN[1:0] = 0x1

# 4. 导出寄存器头文件 (用于文档或代码生成)
sudo umr --export-header navi31_regs.h

# 5. 交互模式
sudo umr -i

# umr> help
# umr> list gpu
# umr> read dcn mmOTG_MASTER_EN
# umr> read dcn mmOTG_H_TOTAL
# umr> write dcn mmOTG_CONTROL 0x00000001
# umr> perf start gfx_BUSY
# umr> perf stop
# umr> waves
# umr> quit

# 6. 启动 Tcl/Tk GUI
sudo umr -g

# 在 GUI 中可以:
# - 树形浏览所有寄存器
# - 实时更新寄存器值
# - 搜索寄存器名称/偏移
# - 编辑和写入寄存器
# - 查看寄存器描述
```

#### GPU 寄存器配置脚本

```bash
#!/bin/bash
# umr_config.sh - 使用 umr 配置 GPU 示例

# 1. 获取当前显示模式
get_display_mode() {
    echo "当前显示模式:"
    sudo umr -B dcn -r mmOTG_H_TOTAL,mmOTG_V_TOTAL
    
    # 解码
    local h_total=$(sudo umr -r mmOTG_H_TOTAL 2>/dev/null | awk '{print $NF}')
    local v_total=$(sudo umr -r mmOTG_V_TOTAL 2>/dev/null | awk '{print $NF}')
    echo "分辨率约: $((h_total - 160)) x $((v_total - 40))"
}

# 2. 检查 GPU 温度
get_gpu_temp() {
    echo "GPU 温度:"
    sudo umr -r mmSMU_TEMPERATURE 2>/dev/null || \
        sudo umr -x 0x13A0000C 1
}

# 3. 检查 PCIE 链路状态
get_pcie_link() {
    echo "PCIE 链路:"
    local link_status=$(sudo umr -x 0x00000042 1 2>/dev/null | \
        awk '{print $2}')
    local width=$(( (link_status >> 4) & 0xF ))
    local speed=$(( link_status & 0xF ))
    echo "  宽度: x${width}"
    echo "  速度: Gen${speed}"
}

# 4. GPU 频率
get_gpu_clocks() {
    echo "GPU 时钟:"
    sudo umr --perf enable gfx_CLOCK --sample 100
}

# 主程序
case "$1" in
    display)
        get_display_mode
        ;;
    temp)
        get_gpu_temp
        ;;
    pcie)
        get_pcie_link
        ;;
    clocks)
        get_gpu_clocks
        ;;
    all)
        get_display_mode
        echo "---"
        get_gpu_temp
        echo "---"
        get_pcie_link
        echo "---"
        get_gpu_clocks
        ;;
    *)
        echo "用法: $0 {display|temp|pcie|clocks|all}"
        ;;
esac
```

#### umr Python 绑定

```python
#!/usr/bin/env python3
# umr_python_bindings.py - umr Python 接口示例

import subprocess
import json
import re


class UMRDebugger:
    """umr 工具 Python 封装"""
    
    def __init__(self, gpu_index=0):
        self.gpu_index = gpu_index
        self._check_umr()
    
    def _check_umr(self):
        """检查 umr 是否可用"""
        result = subprocess.run(
            ["which", "umr"],
            capture_output=True, text=True
        )
        if result.returncode != 0:
            raise RuntimeError("umr not found in PATH")
    
    def _run_umr(self, args):
        """运行 umr 命令"""
        cmd = ["sudo", "umr"] + args
        result = subprocess.run(cmd, capture_output=True, text=True)
        if result.returncode != 0:
            raise RuntimeError(f"umr error: {result.stderr}")
        return result.stdout
    
    def read_register(self, reg_name, block=None):
        """读取寄存器值"""
        args = ["-r", reg_name]
        if block:
            args += ["-b", block]
        output = self._run_umr(args)
        
        # 解析输出: "GPU0: mmREG_NAME[0] = 0x12345678"
        match = re.search(r'=\s*(0x[0-9A-Fa-f]+)', output)
        if match:
            return int(match.group(1), 16)
        return None
    
    def write_register(self, reg_name, value):
        """写入寄存器值"""
        args = ["-w", f"{reg_name}={value}"]
        self._run_umr(args)
    
    def read_offset(self, offset, count=1):
        """按偏移读取寄存器"""
        args = ["-x", f"0x{offset:X}", str(count)]
        output = self._run_umr(args)
        return output
    
    def scan_block(self, block_name):
        """扫描 IP 块的所有寄存器"""
        args = ["-S", block_name]
        return self._run_umr(args)
    
    def get_waves(self, verbose=False):
        """获取波形状态"""
        args = ["--waves"]
        if verbose:
            args.append("--verbose")
        return self._run_umr(args)
    
    def get_perf_counters(self, counter_names, sample_ms=1000):
        """获取性能计数器"""
        counters = ",".join(counter_names)
        args = ["--perf", "enable", counters]
        self._run_umr(args)
        
        args = ["--perf", "sample", str(sample_ms)]
        return self._run_umr(args)
    
    def get_gpu_temperature(self):
        """获取 GPU 温度"""
        value = self.read_register("mmSMU_TEMPERATURE")
        if value is not None:
            # 温度通常编码为 value & 0xFF 度
            temp_c = value & 0xFF
            return temp_c
        return None
    
    def get_display_timing(self):
        """获取当前显示时序"""
        h_total = self.read_register("mmOTG_H_TOTAL", "dcn")
        v_total = self.read_register("mmOTG_V_TOTAL", "dcn")
        
        if h_total and v_total:
            return {
                "h_total": h_total,
                "v_total": v_total,
                "approx_width": (h_total & 0xFFFF) - 160,
                "approx_height": (v_total & 0xFFFF) - 40
            }
        return None


# 使用示例
if __name__ == "__main__":
    try:
        umr = UMRDebugger()
        
        print("=== GPU 状态报告 ===")
        
        # 温度
        temp = umr.get_gpu_temperature()
        if temp:
            print(f"GPU 温度: {temp}°C")
        
        # 显示时序
        timing = umr.get_display_timing()
        if timing:
            print(f"显示分辨率: ~{timing['approx_width']}x{timing['approx_height']}")
        
        # 波形状态
        print("\n波形状态:")
        print(umr.get_waves())
        
        # 性能计数器
        print("\nGFX 利用率:")
        print(umr.get_perf_counters(["gfx_BUSY"], 500))
        
    except Exception as e:
        print(f"错误: {e}")
```

### 案例分析

#### 案例 1：使用 umr 定位着色器 Hang 问题

**问题描述**：运行 3D 应用时 GPU 无响应，触发 GPU hang 检测，驱动重置。

**排查过程**：

```bash
# 步骤 1: 检查 GPU 状态
sudo umr -r mmGRBM_STATUS

# 输出: mmGRBM_STATUS = 0x80000002
# bit[31] = 1: GUI_ACTIVE
# bit[1]  = 1: CP_BUSY (命令处理器忙)
# 但长时间没有进展

# 步骤 2: 捕获波形
sudo umr --waves --verbose

# 发现波形卡在特定的着色器:
# SE0 SH0 CU2 Wave3:
#   Status: STALLED (等待内存)
#   PC: 0x4A00 (循环体内)
#   VGPR: 全为 0x7FFFFFFF (NaN 传染)

# 步骤 3: 反汇编卡住的波形
sudo umr --waves --disassemble

# 0x4A00: V_MUL_F32 v4, v0, v0
# 0x4A04: V_ADD_F32 v5, v4, 1.0
# 0x4A08: V_RCP_F32 v6, v5      ← 除以零！
# 0x4A0C: V_MUL_F32 v7, v5, v6

# 步骤 4: 检查常量缓冲区
sudo umr -r sq_register

# 结论: 着色器 `1/(x^2 + 1)` 中 x=0 导致 RCP 异常
# 修复: 添加保护 if (x > EPSILON) 避免除以零
```

#### 案例 2：性能计数器分析着色器优化

```bash
# 步骤 1: 基准性能
sudo umr --perf enable gfx_BUSY,gfx_VALID,gfx_CLOCK
sudo umr --perf sample 2000

# 输出: gfx_BUSY = 62%, pixel rate = 350 MP/s

# 步骤 2: 分析瓶颈计数器
sudo umr --perf enable gfx_CACHE_HIT,gfx_MEM_READ,gfx_MEM_WRITE
sudo umr --perf sample 2000

# 输出:
# gfx_CACHE_HIT: 45% (L2 命中率低)
# gfx_MEM_READ:  1250 MB/s
# gfx_MEM_WRITE: 800 MB/s

# 步骤 3: 优化后 (改为本地读取,缓存友好)
# 重新测试:
sudo umr --perf sample 2000

# 输出:
# gfx_BUSY = 78% (提升 16%)
# gfx_CACHE_HIT = 82% (大幅提升)
# pixel rate = 510 MP/s (提升 45%)
```

### 相关链接

- umr 官方仓库: https://gitlab.freedesktop.org/tomstdenis/umr
- AMDGPU 调试指南: https://docs.kernel.org/gpu/amdgpu/amdgpu-debugging.html
- ROCm 调试工具: https://rocm.docs.amd.com/en/latest/
- GPU 性能分析: https://gpuopen.com/performance/

### 今日小结

1. **umr 工具定位**：相比 debugfs，umr 提供更丰富的功能，包括寄存器描述查询、波形转储、反汇编、性能计数器和交互模式。

2. **核心功能**：寄存器读/写（按名称或偏移）、块扫描、波形快照捕获、着色器反汇编、性能计数器分析、I/O 跟踪。

3. **GPU 诊断**：umr 可以快速诊断 GPU hang、性能瓶颈、显示异常和 PCIE 链路问题。

4. **脚本化**：umr 支持命令行批处理和 Python 绑定，适合集成到自动化测试框架中。

5. **安全注意事项**：寄存器写入可能导致 GPU 崩溃或硬件损坏，需要 root 权限和谨慎操作。

### 扩展思考

1. **umr vs perftest/rocprof**：在 GPU 性能分析场景中，umr 的硬件级计数器与 ROCm 的分析工具有何不同？什么场景选择哪种？

2. **SR-IOV 调试**：umr 如何支持虚拟化环境中的 VF 调试？PF 和 VF 的寄存器访问权限有什么不同？

3. **波形级调试的局限**：波形转储只能捕获瞬时状态，如何结合指令跟踪和性能计数器做全流程性能分析？

4. **umr 的未来**：随着 GFX IP 版本升级，umr 如何保持对新硬件的支持？社区是如何维护寄存器描述数据库的？