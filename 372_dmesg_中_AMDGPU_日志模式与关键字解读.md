# dmesg 中 AMDGPU 日志模式与关键字解读

## 学习目标

- 掌握 AMDGPU 驱动的日志输出机制和日志级别
- 学会解读 dmesg 中 AMDGPU 的关键日志信息
- 能够通过日志快速定位 GPU 驱动问题的根因
- 掌握日志过滤和分析工具的使用方法

## 知识详解

### 概念原理

#### AMDGPU 日志架构

```ascii
AMDGPU 日志系统:

┌──────────────────────────────────────────────────────────────┐
│  内核日志框架                                                    │
│                                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ pr_info  │  │ pr_warn  │  │ pr_err   │  │ pr_debug     │  │
│  │          │  │          │  │          │  │ (需要 DEBUG)  │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘  │
│       │             │             │               │           │
│  ┌────▼─────────────▼─────────────▼───────────────▼────────┐  │
│  │                printk 环形缓冲区                         │  │
│  │                (/dev/kmsg, dmesg)                        │  │
│  └────────────────────────┬─────────────────────────────────┘  │
│                           │                                     │
│  ┌────────────────────────▼─────────────────────────────────┐  │
│  │  AMDGPU 驱动的日志包装函数                                 │  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │ DRM_DEBUG_DRIVER(...)     // drm 调试信息            │  │  │
│  │  │ DRM_INFO(...)             // 重要信息                │  │  │
│  │  │ DRM_WARN(...)             // 警告                    │  │  │
│  │  │ DRM_ERROR(...)            // 错误                    │  │  │
│  │  │ DRM_DEBUG_KMS(...)        // KMS 调试               │  │  │
│  │  │ DRM_DEBUG_ATOMIC(...)     // 原子操作调试            │  │  │
│  │  │ DRM_DEBUG_VBL(...)        // VBlank 调试             │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └────────────────────────┬─────────────────────────────────┘  │
│                           │                                     │
│  ┌────────────────────────▼─────────────────────────────────┐  │
│  │  AMDGPU 各子系统的日志输出                               │  │
│  │                                                          │  │
│  │  [drm] DC: Display Core 初始化/配置                      │  │
│  │  [drm] DM: Display Manager 状态                          │  │
│  │  [drm] GEM: 内存对象管理                                  │  │
│  │  [drm] VCE: 视频编解码                                    │  │
│  │  [drm] SMU: 电源管理                                      │  │
│  │  [drm] PCIE: 链路状态                                    │  │
│  │  [drm] RAS: 可靠性管理                                   │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

#### 日志级别和过滤

```ascii
内核日志级别:

┌──────────────────────────────────────────────────────────────┐
│ 级别    │ 值   │ 函数宏          │ 含义      │ 默认显示       │
│ ───────┼──────┼────────────────┼──────────┼─────────────   │
│ KERN_EMERG   │ 0 │ pr_emerg     │ 紧急      │ 控制台         │
│ KERN_ALERT   │ 1 │ pr_alert     │ 警报      │ 控制台         │
│ KERN_CRIT    │ 2 │ pr_crit      │ 严重      │ 控制台         │
│ KERN_ERR     │ 3 │ pr_err       │ 错误      │ 控制台         │
│ KERN_WARNING │ 4 │ pr_warn      │ 警告      │ 控制台         │
│ KERN_NOTICE  │ 5 │ pr_notice    │ 通知      │ 控制台         │
│ KERN_INFO    │ 6 │ pr_info      │ 信息      │ 控制台         │
│ KERN_DEBUG   │ 7 │ pr_debug     │ 调试      │ 需手动启用     │
└──────────────────────────────────────────────────────────────┘

AMDGPU 驱动额外调试级别:
┌──────────────────────────────────────────────────────────────┐
│ 调试级别掩码 (drv->debug_mask):                               │
│                                                               │
│ 位    │ 值        │ 宏                       │ 含义           │
│ ─────┼──────────┼─────────────────────────┼────────────────    │
│ 0    │ 0x00001  │ DRM_UT_CORE             │ DRM 核心         │
│ 1    │ 0x00002  │ DRM_UT_DRIVER           │ 驱动通用         │
│ 2    │ 0x00004  │ DRM_UT_KMS              │ KMS 模式设置     │
│ 3    │ 0x00008  │ DRM_UT_PRIM             │ 图元             │
│ 4    │ 0x00010  │ DRM_UT_ATOMIC           │ 原子操作         │
│ 5    │ 0x00020  │ DRM_UT_VBL              │ VBlank           │
│ 6    │ 0x00040  │ DRM_UT_STATE            │ 状态检查         │
│ 7    │ 0x00080  │ DRM_UT_LEASE            │ DRM Lease        │
│ 8    │ 0x00100  │ DRM_UT_DP               │ DisplayPort      │
│ 9    │ 0x00200  │ DRM_UT_HDMI             │ HDMI             │
│ 10   │ 0x00400  │ DRM_UT_EDID             │ EDID             │
│ ...  │ ...      │ ...                     │ ...              │
└──────────────────────────────────────────────────────────────┘
```

### 实践操作

#### 关键日志关键字解读

```bash
# 1. 驱动加载日志
dmesg | grep amdgpu

# 输出示例及解读:

# [    0.123] [drm] amdgpu kernel module initialized
# 解释: 模块初始化成功

# [    0.456] [drm] amdgpu: Runtime PM enabled
# 解释: 运行时电源管理已使能

# [    0.789] [drm] GPU posting...
# 解释: 开始 GPU POST (上电自检)

# [    1.012] [drm] initializing kernel modesetting
# 解释: 开始 KMS 初始化

# [    1.234] [drm] register mmio base: 0xF000000000
# 解释: BAR0 MMIO 基地址

# [    1.456] [drm] register mmio size: 2097152
# 解释: BAR0 大小 2MB

# [    1.678] [drm] add ip block number 0 <psp>
# 解释: 注册 PSP IP 块

# [    2.000] [drm] BIOS CRC: 0x12345678 (valid)
# 解释: VBIOS CRC 验证通过

# [    2.500] [drm] VCN decode is enabled in VM mode
# 解释: VCN 解码器启用

# [    3.000] [drm] Display Core initialized
# 解释: DC 显示核心初始化完成

# [    3.500] [drm] DMUB hardware initialized
# 解释: DMUB 显示微控制器初始化

# [    4.000] [drm] fb0: amdgpudrmfb frame buffer device
# 解释: 帧缓冲设备创建

# [    4.500] [drm] amdgpu: psp cmd failed
# 解释: 致命错误: PSP 命令失败!
```

#### 常见错误日志解读

```bash
# 1. GPU Hang 日志
dmesg | grep -i "hang\|timeout\|reset\|recovery"

# [drm] GPU hang detected on GFX pipe
# [drm] GRBM_STATUS=0xA0000002
# [drm] CP_STATUS=0x80000000
# [drm] GFX is busy, no progress for 2 seconds
# [drm] Starting GPU recovery...
# [drm] GPU reset begin
# [drm] BIF_MEM_STATUS=0x00000000
# [drm] GPU reset succeeded, trying to resume

# 解读:
# - "GPU hang detected"  → GPU 卡住
# - "GRBM_STATUS=0xA0000002" → GUI_ACTIVE + CP_BUSY
# - "Starting GPU recovery" → 开始 GPU 重置
# - "GPU reset succeeded"  → 恢复成功

# 2. 显存错误日志
dmesg | grep -i "page fault\|VM fault\|GPU fault"

# [drm] GPU page fault at address 0x00000000deadbeef
# [drm] VM fault on GPU 0x00000000deadbeef
# [drm] Invalid page table entry at 0x...

# 解读:
# - "page fault" → GPU 访问无效地址
# - "VM fault"   → 虚拟内存错误
# - 常见原因: 用户态驱动 bug, 显存损坏

# 3. 电源管理日志
dmesg | grep -i "power\|thermal\|throttle\|clock"

# [drm] SMU: firmware version 85.0.0
# [drm] SMU: power table loaded
# [drm] Thermal alert: GPU temp = 95°C
# [drm] SMU: throttling GFX clock

# 解读:
# - "power table loaded" → 电源参数表加载成功
# - "Thermal alert"     → 温度告警
# - "throttling"        → 降频

# 4. PCIE 链路日志
dmesg | grep -i "pcie\|link\|bandwidth"

# [drm] PCIE link: max Gen4 x16, current Gen4 x16
# [drm] PCIE link: bandwidth 32.0 GT/s
# [drm] PCIE link degraded: Gen3 instead of Gen4

# 解读:
# - "link degraded" → PCIE 链路降级
# - "bandwidth"     → 当前带宽

# 5. 显示相关日志
dmesg | grep -i "display\|connector\|HDMI\|DP\|mode"

# [drm] Connector DP-1: get supported mode
# [drm] DisplayPort: link training successful
# [drm] Atomic commit: setting mode 3840x2160

# 解读:
# - "link training" → DP 链路训练
# - "Atomic commit" → 原子模式设置
```

#### 启用调试日志

```bash
# 1. 设置 DRM 调试级别
# 启用所有调试输出
echo 0xFFFFFFFF > /sys/module/drm/parameters/debug

# 启用特定类别
echo 0x04 > /sys/module/drm/parameters/debug  # KMS
echo 0x10 > /sys/module/drm/parameters/debug  # Atomic
echo 0x20 > /sys/module/drm/parameters/debug  # VBlank

# 2. 内核命令行参数
# 在 GRUB 中添加:
# drm.debug=0x1f log_buf_len=8M
# drm.debug=0x04 表示 KMS 调试

# 3. AMDGPU 模块参数
# modinfo amdgpu | grep debug
# amdgpu_debug: 启用 AMDGPU 调试 (int)
# amdgpu_emu_mode: 仿真模式 (int)

# 4. kmsg 实时监控
sudo cat /dev/kmsg | grep amdgpu

# 5. 使用 trace event
# 列出 AMDGPU trace events
sudo cat /sys/kernel/debug/tracing/available_events | grep amdgpu

# 启用特定 trace
echo 1 > /sys/kernel/debug/tracing/events/amdgpu/amdgpu_vm_update/enable
```

#### 日志过滤和分析工具

```bash
#!/bin/bash
# amdgpu_log_analyzer.sh - AMDGPU 日志分析脚本

# 用法: ./amdgpu_log_analyzer.sh [log_file]

LOG_FILE="${1:-/dev/stdin}"

echo "AMDGPU 日志分析报告"
echo "===================="
echo ""

# 1. 基本统计
echo "--- 基本统计 ---"
grep -c "\[drm\]" "$LOG_FILE" | awk '{print "  总 DRM 日志行数: " $1}'
grep -c "error\|ERROR\|fail\|FAIL" "$LOG_FILE" | awk '{print "  错误数: " $1}'
grep -c "WARN" "$LOG_FILE" | awk '{print "  警告数: " $1}'

# 2. 加载状态
echo ""
echo "--- 加载状态 ---"
if grep -q "amdgpu kernel module initialized" "$LOG_FILE"; then
    echo "  模块加载: ✓ 成功"
else
    echo "  模块加载: ✗ 失败"
fi

if grep -q "GPU posting" "$LOG_FILE"; then
    echo "  GPU POST: ✓ 完成"
fi

if grep -q "Display Core initialized" "$LOG_FILE"; then
    echo "  显示核心: ✓ 初始化完成"
fi

if grep -q "amdgpu_device_init" "$LOG_FILE"; then
    echo "  设备初始化: ✗ 失败"
fi

# 3. PCIE 链路检查
echo ""
echo "--- PCIE 链路 ---"
PCI_LINK=$(grep "PCIE link" "$LOG_FILE" | tail -1)
if [ -n "$PCI_LINK" ]; then
    echo "  $PCI_LINK"
    if echo "$PCI_LINK" | grep -q "degraded"; then
        echo "  ⚠ 链路降级!"
    fi
else
    echo "  未检测到"
fi

# 4. 温度告警
echo ""
echo "--- 温度事件 ---"
THRM_ALERTS=$(grep -c "Thermal alert" "$LOG_FILE")
if [ "$THRM_ALERTS" -gt 0 ]; then
    echo "  ⚠ 温度告警次数: $THRM_ALERTS"
    grep "Thermal alert" "$LOG_FILE" | tail -3
else
    echo "  无温度告警"
fi

# 5. GPU Reset
echo ""
echo "--- GPU Reset ---"
RESETS=$(grep -c "GPU reset" "$LOG_FILE")
if [ "$RESETS" -gt 0 ]; then
    echo "  ⚠ GPU 重置次数: $RESETS"
    grep "GPU reset" "$LOG_FILE"
fi

# 6. 固件版本
echo ""
echo "--- 固件版本 ---"
grep "firmware version\|Firmware" "$LOG_FILE"

# 7. 超时检测
echo ""
echo "--- 超时检测 ---"
TIMEOUTS=$(grep -c "timeout" "$LOG_FILE")
if [ "$TIMEOUTS" -gt 0 ]; then
    echo "  ⚠ 超时次数: $TIMEOUTS"
    grep "timeout" "$LOG_FILE"
else
    echo "  无超时事件"
fi

echo ""
echo "--- 分析完成 ---"
```

#### Python 日志分析

```python
#!/usr/bin/env python3
# amdgpu_dmesg_parser.py - AMDGPU dmesg 解析器

import re
import sys
from dataclasses import dataclass
from typing import List, Optional
from datetime import timedelta


@dataclass
class LogEntry:
    timestamp: float
    level: str
    component: str
    message: str
    raw: str


class AMDGPULogParser:
    """AMDGPU dmesg 日志解析器"""
    
    PATTERNS = {
        "init": re.compile(r"amdgpu kernel module initialized"),
        "gpu_post": re.compile(r"GPU posting"),
        "dc_init": re.compile(r"Display Core initialized"),
        "dmub_init": re.compile(r"DMUB hardware initialized"),
        "gpu_hang": re.compile(r"GPU hang detected"),
        "gpu_reset": re.compile(r"GPU reset (begin|succeeded|failed)"),
        "page_fault": re.compile(r"GPU page fault|VM fault"),
        "thermal": re.compile(r"Thermal alert"),
        "throttle": re.compile(r"throttling"),
        "link_degraded": re.compile(r"link degraded"),
        "link_status": re.compile(r"PCIE link"),
        "fw_version": re.compile(r"firmware version"),
        "timeout": re.compile(r"timeout"),
        "error": re.compile(r"ERROR|error"),
    }
    
    def __init__(self, log_lines: List[str]):
        self.entries = self._parse(log_lines)
    
    def _parse(self, lines: List[str]) -> List[LogEntry]:
        entries = []
        pattern = re.compile(
            r'\[(?P<time>\s*[\d.]+)\]\s+(?P<msg>.+)'
        )
        
        for line in lines:
            match = pattern.search(line)
            if match and ("amdgpu" in line or "drm" in line):
                try:
                    ts = float(match.group("time").strip())
                except ValueError:
                    ts = 0.0
                
                msg = match.group("msg")
                entry = LogEntry(
                    timestamp=ts,
                    level=self._detect_level(msg),
                    component=self._detect_component(msg),
                    message=msg,
                    raw=line.strip()
                )
                entries.append(entry)
        
        return entries
    
    def _detect_level(self, msg: str) -> str:
        if "ERROR" in msg or "error" in msg or "fail" in msg.lower():
            return "ERROR"
        if "WARN" in msg or "warning" in msg.lower():
            return "WARN"
        if "timeout" in msg.lower():
            return "WARN"
        return "INFO"
    
    def _detect_component(self, msg: str) -> str:
        if "PSP" in msg or "psp" in msg:
            return "PSP"
        if "SMU" in msg or "smu" in msg:
            return "SMU"
        if "DC" in msg or "Display" in msg:
            return "DC"
        if "DMUB" in msg:
            return "DMUB"
        if "PCIE" in msg or "pcie" in msg:
            return "PCIE"
        if "VCN" in msg:
            return "VCN"
        if "PSP" in msg:
            return "PSP"
        if "hang" in msg or "reset" in msg:
            return "GFX"
        return "GENERAL"
    
    def search(self, pattern_name: str) -> List[LogEntry]:
        """按模式搜索日志"""
        if pattern_name not in self.PATTERNS:
            return []
        
        regex = self.PATTERNS[pattern_name]
        return [e for e in self.entries if regex.search(e.message)]
    
    def get_errors(self) -> List[LogEntry]:
        """获取所有错误日志"""
        return [e for e in self.entries if e.level == "ERROR"]
    
    def get_timeline(self) -> str:
        """生成时间线"""
        lines = []
        for e in self.entries:
            lines.append(f"[{e.timestamp:8.3f}] [{e.level:5s}] {e.message}")
        return "\n".join(lines)
    
    def get_summary(self) -> str:
        """生成摘要"""
        errors = self.get_errors()
        hangs = self.search("gpu_hang")
        resets = self.search("gpu_reset")
        thermals = self.search("thermal")
        
        parts = []
        parts.append(f"总日志条目: {len(self.entries)}")
        parts.append(f"错误: {len(errors)}")
        parts.append(f"GPU Hang: {len(hangs)}")
        parts.append(f"GPU Reset: {len(resets)}")
        parts.append(f"温度告警: {len(thermals)}")
        
        if errors:
            parts.append("\n错误列表:")
            for e in errors[:10]:
                parts.append(f"  [{e.timestamp}] {e.component}: {e.message[:80]}")
        
        return "\n".join(parts)


# 使用示例
if __name__ == "__main__":
    # 从文件或标准输入读取
    if len(sys.argv) > 1:
        with open(sys.argv[1]) as f:
            lines = f.readlines()
    else:
        import subprocess
        result = subprocess.run(["dmesg"], capture_output=True, text=True)
        lines = result.stdout.split("\n")
    
    parser = AMDGPULogParser(lines)
    
    print("=" * 60)
    print("AMDGPU dmesg 分析报告")
    print("=" * 60)
    print()
    print(parser.get_summary())
    print()
    
    # 检查特定问题
    hangs = parser.search("gpu_hang")
    if hangs:
        print(f"\n⚠ GPU Hang 事件 ({len(hangs)}):")
        for h in hangs:
            print(f"  {h.raw}")
    
    thermals = parser.search("thermal")
    if thermals:
        print(f"\n⚠ 温度告警 ({len(thermals)}):")
        for t in thermals:
            print(f"  {t.raw}")
```

### 案例分析

#### 案例 1：通过日志定位 GPU 初始化失败

```bash
# 错误日志
dmesg | grep amdgpu

# [    0.123] [drm] amdgpu kernel module initialized
# [    0.456] [drm] GPU posting...
# [    0.789] [drm] initializing kernel modesetting
# [    1.012] [drm] register mmio base: 0xF000000000
# [    1.456] [drm] psp_cmd_submit: PSP command 0x1234 timeout
# [    1.457] [drm] amdgpu: psp init failed
# [    1.458] [drm] amdgpu_device_init failed (-110)

# 分析:
# - PSP 命令超时 (-110 = ETIMEDOUT)
# - 可能原因: 固件文件损坏, PSP 硬件故障, PCIE 通信问题
# - 排查方向: 检查固件文件, 重新安装固件包

# 解决方案:
sudo update-initramfs -u
sudo apt install --reinstall amdgpu-firmware
reboot
```

#### 案例 2：GPU 重置循环分析

```bash
# 日志显示 GPU 反复重置
dmesg | grep -E "hang|reset|recovery" | tail -20

# [ 100.123] [drm] GPU hang detected on GFX pipe
# [ 100.456] [drm] GPU reset begin
# [ 100.789] [drm] GPU reset succeeded, trying to resume
# [ 120.123] [drm] GPU hang detected on GFX pipe  ← 再次 hang
# [ 120.456] [drm] GPU reset begin
# [ 120.789] [drm] GPU reset succeeded, trying to resume
# [ 140.123] [drm] GPU hang detected on GFX pipe  ← 循环

# 分析:
# - GPU 每次恢复后约 20 秒再次 hang
# - 可能是某个特定操作触发了 hang
# - 排查方向: 找出触发的用户程序, 检查显存错误

# 启用更多调试信息
echo 0xFFFFFFFF > /sys/module/drm/parameters/debug

# 检查是否有 VM page fault
dmesg | grep -i "page fault\|VM fault"
```

### 相关链接

- Linux 内核日志: https://docs.kernel.org/core-api/printk-basics.html
- DRM 调试: https://docs.kernel.org/gpu/drm-kms-helpers.html
- AMDGPU 调试参数: https://docs.kernel.org/gpu/amdgpu/amdgpu-params.html
- dmesg 手册: man dmesg

### 今日小结

1. **日志系统**：AMDGPU 使用内核 printk 系统和 DRM 日志宏输出信息，支持 8 级日志和按位控制的调试级别。

2. **关键日志阶段**：模块初始化、GPU POST、KMS 初始化、IP 块加载、显示核心初始化、帧缓冲创建是驱动加载的六个关键阶段。

3. **常见错误模式**：GPU Hang (GRBM_STATUS 分析)、VM Page Fault (地址验证)、电源管理告警 (温度/降频)、PCIE 链路降级。

4. **调试方法**：设置 drm.debug 参数、使用 kmsg 实时监控、trace event 跟踪、Python 脚本批量分析。

5. **故障定位**：通过关键字（timeout、fail、error、degraded、throttle）快速定位问题类型和方向。

### 扩展思考

1. **日志性能开销**：启用所有调试日志（drm.debug=0xFFFFFFFF）对系统性能的影响有多大？在生产环境中有哪些安全的调试级别？

2. **日志持久化**：在 GPU 完全锁定或系统崩溃的情况下，内核日志环形缓冲区中的信息能否保存？pstore/ramoops 如何帮助保留崩溃日志？

3. **自动化日志分析**：如何构建一个自动化的 AMDGPU 日志告警系统，根据日志模式自动分类和升级问题？