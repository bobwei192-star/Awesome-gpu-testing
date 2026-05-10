# GPU 内存使用追踪：amdgpu_gem_info / fdinfo

## 学习目标

- 理解 GEM（Graphics Execution Manager）对象模型和 GPU 内存管理架构
- 掌握通过 sysfs/debugfs 接口追踪 GPU 内存使用的方法
- 学会使用 fdinfo 监控进程级 GPU 内存占用
- 能够识别和排查 GPU 内存泄漏问题

## 知识详解

### 概念原理

#### GPU 内存架构

```ascii
AMDGPU 内存管理架构:

┌──────────────────────────────────────────────────────────────┐
│  系统内存 (System RAM)                                        │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  CPU 可访问区域                                          │  │
│  │                                                         │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐   │  │
│  │  │ 用户空间 BO  │  │ 内核空间 BO  │  │ GART 表     │   │  │
│  │  │ (显存回退)    │  │ (驱动使用)    │  │ (页表)       │   │  │
│  │  └──────────────┘  └──────────────┘  └─────────────┘   │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  VRAM (通过 PCIe BAR 映射)                               │  │
│  │                                                         │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐   │  │
│  │  │ 显存 BO      │  │ 帧缓冲        │  │ GART 窗口   │   │  │
│  │  │ (纹理/顶点)   │  │ (显示扫描)    │  │ (显存访问)   │   │  │
│  │  └──────────────┘  └──────────────┘  └─────────────┘   │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
├───────────────────────────────────────────────────────────────┤
│  GPU 内存 (VRAM)                                              │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  VRAM 地址空间                                           │  │
│  │                                                         │  │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌─────┐  │  │
│  │  │ FB   │ │ GART │ │ BO   │ │ BO   │ │ BO   │ │ ... │  │  │
│  │  │      │ │ 表   │ │ #1   │ │ #2   │ │ #3   │ │     │  │  │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └─────┘  │  │
│  │                                                         │  │
│  │  0x0      FB_SIZE ...                                   │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  GTT (Graphics Translation Table)                       │  │
│  │  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐    │  │
│  │  │ PTE0 │ PTE1 │ PTE2 │ PTE3 │ PTE4 │ ...  │ PTEn │    │  │
│  │  ├──────┼──────┼──────┼──────┼──────┼──────┼──────┤    │  │
│  │  │VRAM  │SysP  │VRAM  │SysP  │SysP  │ ...  │VRAM  │    │  │
│  │  └──────┴──────┴──────┴──────┴──────┴──────┴──────┘    │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

#### GEM BO（Buffer Object）模型

```ascii
GEM Buffer Object 生命周期:

┌──────────────────────────────────────────────────────────────┐
│  创建                                    销毁                  │
│    │                                       ▲                  │
│    ▼                                       │                  │
│  ┌─────────────────────────────────────────┴────┐             │
│  │          GEM BO 对象                          │             │
│  │                                               │             │
│  │  ┌────────────────────────────────────┐       │             │
│  │  │ 元数据:                             │       │             │
│  │  │  - size: 64MB (对象大小)            │       │             │
│  │  │  - flags: VRAM|CPU_ACCESS          │       │             │
│  │  │  - placement: VRAM (当前位置)       │       │             │
│  │  │  - refcount: 3 (引用计数)           │       │             │
│  │  │  - fpriv_list: (进程列表)           │       │             │
│  │  │  - ttm: (TTM 对象)                 │       │             │
│  │  └────────────────────────────────────┘       │             │
│  │                                               │             │
│  │  ┌────────────────────────────────────┐       │             │
│  │  │ 内存页面 (pages):                    │       │             │
│  │  │  ┌────┬────┬────┬────┬────┬────┐   │       │             │
│  │  │  │ 0  │ 1  │ 2  │ 3  │ .. │ N  │   │       │             │
│  │  │  └────┴────┴────┴────┴────┴────┘   │       │             │
│  │  │ 每个页面 4KB (或 2MB 大页)           │       │             │
│  │  └────────────────────────────────────┘       │             │
│  └─────────────────────────────────────────────────┘             │
│                                                                  │
│  状态转移:                                                        │
│  ┌─────────┐  分配   ┌─────────┐  迁移   ┌─────────┐  换出   ┌──┐│
│  │ 未分配   │───────→│ VRAM    │───────→│ GTT     │───────→│系 ││
│  │          │         │ (显存)  │         │ (系统)  │         │统 ││
│  └─────────┘         └─────────┘         └─────────┘         │内 ││
│       ▲                                                     │存 ││
│       └─────────────────────────────────────────────────────┘──┘│
└──────────────────────────────────────────────────────────────┘
```

#### VRAM 分配和管理

```ascii
VRAM 分配器 (TTM/VRAM Manager):

┌──────────────────────────────────────────────────────────────┐
│  VRAM 区域划分:                                               │
│                                                               │
│  地址              │ 大小      │ 用途                          │
│  ─────────────────┼─────────┼────────────────────────────    │
│  0x00000000       │ 16MB     │ 帧缓冲 (FB)                    │
│  0x01000000       │ 4MB      │ GART 页表                      │
│  0x01400000       │ 2MB      │ 命令缓冲区 ring                │
│  0x01600000       │ 128MB    │ 纹理/顶点数据 BO               │
│  ...              │ ...      │ ...                           │
│  0x7FC00000       │ 4MB      │ 固件日志缓冲区                  │
│  └────────────────┴──────────┘                               │
│                                                               │
│  分配器类型:                                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ TTM (Translation Table Manager)                        │  │
│  │  - 通用内存管理器                                       │  │
│  │  - 支持多种内存类型 (VRAM, GTT, SYSTEM)                  │  │
│  │  - 支持内存迁移 (eviction)                              │  │
│  │  - 支持交换 (swap)                                      │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ amdgpu_vm (Virtual Memory)                             │  │
│  │  - GPU 虚拟地址映射                                     │  │
│  │  - 页表管理 (PTE/PMD/PUD)                              │  │
│  │  - 支持 48-bit 虚拟地址空间                              │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### 实践操作

#### 查看 GPU 内存使用概览

```bash
# 1. 查看 VRAM 总大小和使用情况
cat /sys/class/drm/card0/device/mem_info_vram_total
# 17179869184  (16 GB)

cat /sys/class/drm/card0/device/mem_info_vram_used
# 4294967296   (4 GB)

cat /sys/class/drm/card0/device/mem_info_vis_vram_total
# 8589934592   (8 GB, 可见 VRAM)

cat /sys/class/drm/card0/device/mem_info_vis_vram_used
# 2147483648   (2 GB, 可见 VRAM 使用)

# 2. 查看 GTT 使用情况
cat /sys/class/drm/card0/device/mem_info_gtt_total
# 8589934592   (8 GB)

cat /sys/class/drm/card0/device/mem_info_gtt_used
# 1073741824   (1 GB)

# 3. 查看内存使用百分比
#!/bin/bash
# gpu_mem_usage.sh - GPU 内存使用监控

TOTAL=$(cat /sys/class/drm/card0/device/mem_info_vram_total)
USED=$(cat /sys/class/drm/card0/device/mem_info_vram_used)
PERCENT=$((USED * 100 / TOTAL))

echo "VRAM: $((USED / 1024 / 1024))MB / $((TOTAL / 1024 / 1024))MB ($PERCENT%)"

VIS_TOTAL=$(cat /sys/class/drm/card0/device/mem_info_vis_vram_total)
VIS_USED=$(cat /sys/class/drm/card0/device/mem_info_vis_vram_used)
echo "Visible VRAM: $((VIS_USED / 1024 / 1024))MB / $((VIS_TOTAL / 1024 / 1024))MB"

GTT_TOTAL=$(cat /sys/class/drm/card0/device/mem_info_gtt_total)
GTT_USED=$(cat /sys/class/drm/card0/device/mem_info_gtt_used)
echo "GTT: $((GTT_USED / 1024 / 1024))MB / $((GTT_TOTAL / 1024 / 1024))MB"

# 输出:
# VRAM: 4096MB / 16384MB (25%)
# Visible VRAM: 2048MB / 8192MB
# GTT: 1024MB / 8192MB
```

#### 进程级内存追踪 (fdinfo)

```bash
# 1. 查看进程文件描述符
ls /proc/<PID>/fdinfo/

# 2. 查看 DRM 文件描述符的 fdinfo
cat /proc/<PID>/fdinfo/<DRM_FD>

# 输出:
# pos:    0
# flags:  02000002
# mnt_id: 29
# drm-driver: amdgpu
# drm-pdev:   0000:03:00.0
# drm-client-id:       42
# drm-engine-gfx:      123456789 ns
# drm-engine-sdma:     2345678 ns
# drm-engine-video:    0 ns
# drm-engine-copy:     0 ns
# drm-memory-vram:     1073741824
# drm-memory-gtt:      268435456
# drm-memory-cpu:      134217728

# 解释:
# drm-engine-*: GPU 各引擎累计时间 (纳秒)
# drm-memory-vram: VRAM 使用量 (字节) = 1GB
# drm-memory-gtt: GTT 使用量 (字节) = 256MB
# drm-memory-cpu: CPU 内存使用 (字节) = 128MB

# 3. 监控所有进程的 GPU 内存使用
#!/bin/bash
# gpu_mem_per_process.sh

echo "PID    NAME       VRAM      GTT       CPU       ENG_GFX"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

for pid in /proc/[0-9]*; do
    PID=$(basename "$pid")
    NAME=$(cat "$pid/comm" 2>/dev/null)
    
    for fd in "$pid/fdinfo"/*; do
        if grep -q "drm-driver: amdgpu" "$fd" 2>/dev/null; then
            VRAM=$(grep "drm-memory-vram" "$fd" 2>/dev/null | cut -d: -f2 | tr -d ' ')
            GTT=$(grep "drm-memory-gtt" "$fd" 2>/dev/null | cut -d: -f2 | tr -d ' ')
            CPU=$(grep "drm-memory-cpu" "$fd" 2>/dev/null | cut -d: -f2 | tr -d ' ')
            ENG=$(grep "drm-engine-gfx" "$fd" 2>/dev/null | cut -d: -f2 | tr -d ' ')
            
            VRAM_MB=$((VRAM / 1024 / 1024 2>/dev/null || echo 0))
            GTT_MB=$((GTT / 1024 / 1024 2>/dev/null || echo 0))
            CPU_MB=$((CPU / 1024 / 1024 2>/dev/null || echo 0))
            ENG_MS=$((ENG / 1000000 2>/dev/null || echo 0))
            
            printf "%-6s %-10s %-8s %-8s %-8s %s\n" "$PID" "${NAME:0:10}" "${VRAM_MB}MB" "${GTT_MB}MB" "${CPU_MB}MB" "${ENG_MS}ms"
        fi
    done
done
```

#### amdgpu_gem_info debugfs 接口

```bash
# 1. 查看所有 GEM BO 信息
cat /sys/kernel/debug/dri/0/amdgpu_gem_info

# 输出示例:
# VRAM: 4096MB 8192MB used (25%)
# GTT: 1024MB 8192MB used (12%)
#
# GEM BO 列表:
# ┌──────┬──────────┬──────────┬─────────┬────────┬────────┬───────┐
# │ 地址  │ 大小     │ 类型     │ 标志    │ 引用   │ 进程   │ 名称  │
# ├──────┼──────────┼──────────┼─────────┼────────┼────────┼───────┤
# │ 0x100 │ 67108864 │ VRAM     │ R       │ 3      │ 1234   │ fb    │
# │ 0x500 │ 16777216 │ GTT      │ RW      │ 2      │ 5678   │ tex0  │
# │ 0x900 │ 33554432 │ VRAM     │ RW      │ 1      │ 5678   │ vb0   │
# │ ...   │          │          │         │        │        │       │
# └──────┴──────────┴──────────┴─────────┴────────┴────────┴───────┘

# 2. 查看特定进程的 GEM 对象
cat /sys/kernel/debug/dri/0/amdgpu_gem_info | grep "5678"

# 3. 查看 GEM 对象详细信息
# (每个对象的详细内容)
cat /sys/kernel/debug/dri/0/amdgpu_gem_info

# 输出详细格式:
# BO 0x0000000012345678
#   Size: 67108864 (64MB)
#   Flags: 0x0000000d (VRAM|CPU_ACCESS|NO_CPU_ACCESS)
#   Placement: VRAM
#   Domain: VRAM
#   Fence: 0xffff888123456789
#   Shared: 2
#   Process: 1234 (Xorg)
#   Imported: no
#   Exported: no
```

#### 使用 Python 监控 GPU 内存

```python
#!/usr/bin/env python3
# gpu_mem_monitor.py - GPU 内存监控工具

import os
import time
import json
import subprocess
from pathlib import Path
from datetime import datetime
from collections import defaultdict


class GPUMemoryMonitor:
    """GPU 内存监控器"""
    
    DRM_PATH = "/sys/class/drm"
    DEBUGFS_PATH = "/sys/kernel/debug/dri"
    
    def __init__(self, card="card0"):
        self.card = card
        self.device_path = f"{self.DRM_PATH}/{card}/device"
        self.stats_history = []
    
    def read_sysfs(self, name: str) -> int:
        """读取 sysfs 文件中的数值"""
        path = f"{self.device_path}/{name}"
        try:
            with open(path) as f:
                return int(f.read().strip())
        except (FileNotFoundError, ValueError):
            return 0
    
    def get_memory_stats(self) -> dict:
        """获取内存统计"""
        stats = {
            "timestamp": datetime.now().isoformat(),
            "vram": {
                "total": self.read_sysfs("mem_info_vram_total"),
                "used": self.read_sysfs("mem_info_vram_used"),
            },
            "vis_vram": {
                "total": self.read_sysfs("mem_info_vis_vram_total"),
                "used": self.read_sysfs("mem_info_vis_vram_used"),
            },
            "gtt": {
                "total": self.read_sysfs("mem_info_gtt_total"),
                "used": self.read_sysfs("mem_info_gtt_used"),
            },
        }
        
        # 计算百分比
        for key in ["vram", "vis_vram", "gtt"]:
            t = stats[key]["total"]
            u = stats[key]["used"]
            stats[key]["percent"] = (u / t * 100) if t > 0 else 0
        
        return stats
    
    def get_process_memory(self) -> dict:
        """获取各进程内存使用"""
        processes = {}
        
        for proc in Path("/proc").iterdir():
            if not proc.name.isdigit():
                continue
            
            pid = proc.name
            try:
                comm = (proc / "comm").read_text().strip()
            except (PermissionError, FileNotFoundError):
                continue
            
            for fdinfo in (proc / "fdinfo").iterdir():
                try:
                    content = fdinfo.read_text()
                except (PermissionError, FileNotFoundError):
                    continue
                
                if "drm-driver: amdgpu" not in content:
                    continue
                
                vram = self._parse_fdinfo_value(content, "drm-memory-vram")
                gtt = self._parse_fdinfo_value(content, "drm-memory-gtt")
                cpu = self._parse_fdinfo_value(content, "drm-memory-cpu")
                eng = self._parse_fdinfo_value(content, "drm-engine-gfx")
                
                processes[pid] = {
                    "name": comm,
                    "vram": vram,
                    "gtt": gtt,
                    "cpu": cpu,
                    "engine_time_ns": eng,
                }
        
        return processes
    
    def _parse_fdinfo_value(self, content: str, key: str) -> int:
        """解析 fdinfo 中的值"""
        for line in content.split("\n"):
            if line.startswith(key):
                try:
                    return int(line.split(":")[1].strip())
                except (IndexError, ValueError):
                    return 0
        return 0
    
    def get_gem_info(self) -> list:
        """获取 GEM BO 信息"""
        path = f"{self.DEBUGFS_PATH}/0/amdgpu_gem_info"
        try:
            with open(path) as f:
                return self._parse_gem_info(f.read())
        except FileNotFoundError:
            return []
    
    def _parse_gem_info(self, content: str) -> list:
        """解析 GEM 信息"""
        objects = []
        current = {}
        
        for line in content.split("\n"):
            if line.startswith("BO "):
                if current:
                    objects.append(current)
                current = {"address": line.split()[1]}
            elif ":" in line and current:
                key, value = line.split(":", 1)
                current[key.strip()] = value.strip()
        
        if current:
            objects.append(current)
        
        return objects
    
    def monitor(self, interval: float = 1.0, count: int = 10):
        """实时监控"""
        print(f"{'时间':20s} {'VRAM使用':12s} {'VRAM%':8s} {'GTT使用':12s} {'GTT%':8s}")
        print("-" * 60)
        
        for _ in range(count):
            stats = self.get_memory_stats()
            vram_used = stats["vram"]["used"] // (1024*1024)
            vram_pct = stats["vram"]["percent"]
            gtt_used = stats["gtt"]["used"] // (1024*1024)
            gtt_pct = stats["gtt"]["percent"]
            
            print(f"{datetime.now().strftime('%H:%M:%S'):20s} "
                  f"{vram_used:6d}MB     {vram_pct:5.1f}%    "
                  f"{gtt_used:6d}MB     {gtt_pct:5.1f}%")
            
            self.stats_history.append(stats)
            time.sleep(interval)
    
    def report(self) -> str:
        """生成报告"""
        if not self.stats_history:
            return "No data collected"
        
        vram_used = [s["vram"]["used"] for s in self.stats_history]
        gtt_used = [s["gtt"]["used"] for s in self.stats_history]
        
        report = []
        report.append("GPU 内存使用报告")
        report.append("=" * 50)
        report.append(f"采样次数: {len(self.stats_history)}")
        report.append(f"VRAM:")
        report.append(f"  平均: {sum(vram_used)/len(vram_used)/1024/1024:.0f}MB")
        report.append(f"  峰值: {max(vram_used)/1024/1024:.0f}MB")
        report.append(f"  最小: {min(vram_used)/1024/1024:.0f}MB")
        report.append(f"GTT:")
        report.append(f"  平均: {sum(gtt_used)/len(gtt_used)/1024/1024:.0f}MB")
        report.append(f"  峰值: {max(gtt_used)/1024/1024:.0f}MB")
        
        return "\n".join(report)


# 使用示例
if __name__ == "__main__":
    import sys
    
    monitor = GPUMemoryMonitor()
    
    if len(sys.argv) > 1 and sys.argv[1] == "--monitor":
        interval = float(sys.argv[2]) if len(sys.argv) > 2 else 1.0
        count = int(sys.argv[3]) if len(sys.argv) > 3 else 10
        monitor.monitor(interval, count)
        print()
        print(monitor.report())
    
    elif len(sys.argv) > 1 and sys.argv[1] == "--process":
        procs = monitor.get_process_memory()
        print(f"{'PID':>6s} {'NAME':15s} {'VRAM':>10s} {'GTT':>10s} {'GFX_ENG':>10s}")
        print("-" * 55)
        for pid, info in sorted(procs.items(), key=lambda x: x[1]["vram"], reverse=True)[:20]:
            vram_mb = info["vram"] // (1024*1024)
            gtt_mb = info["gtt"] // (1024*1024)
            eng_ms = info["engine_time_ns"] // 1000000
            print(f"{pid:>6s} {info['name'][:15]:15s} {vram_mb:8d}MB {gtt_mb:8d}MB {eng_ms:8d}ms")
    
    else:
        stats = monitor.get_memory_stats()
        print(json.dumps(stats, indent=2))
```

#### VRAM 压力测试

```bash
#!/bin/bash
# vram_stress_test.sh - VRAM 压力测试

# 使用 glmark2 或 vkpeak 测试
# 监控 VRAM 使用情况

echo "VRAM 压力测试"
echo "=============="

# 记录初始状态
echo "初始 VRAM 使用:"
cat /sys/class/drm/card0/device/mem_info_vram_used | awk '{printf "  %d MB\n", $1/1024/1024}'

# 运行压力测试
echo ""
echo "运行 glmark2..."
glmark2 --benchmark build:duration=30 &

# 监控 VRAM 变化
for i in {1..30}; do
    USED=$(cat /sys/class/drm/card0/device/mem_info_vram_used)
    printf "\r  时间: %02ds  VRAM: %d MB" $i $(($USED/1024/1024))
    sleep 1
done

echo ""
echo ""
echo "测试完成后的 VRAM:"
cat /sys/class/drm/card0/device/mem_info_vram_used | awk '{printf "  %d MB\n", $1/1024/1024}'

# 检查是否有内存泄漏
echo ""
echo "等待 5 秒后检查..."
sleep 5

LEAK_CHECK=$(cat /sys/class/drm/card0/device/mem_info_vram_used)
INITIAL=$(cat /sys/class/drm/card0/device/mem_info_vram_used)

if [ $LEAK_CHECK -gt $((INITIAL * 11 / 10)) ]; then
    echo "⚠ 可能存在内存泄漏！"
    echo "  初始: $(($INITIAL/1024/1024))MB"
    echo "  当前: $(($LEAK_CHECK/1024/1024))MB"
else
    echo "✓ 内存使用正常"
fi
```

### 案例分析

#### 案例 1：VRAM 内存泄漏排查

```bash
# 症状: 运行游戏后 VRAM 使用持续增长
watch -n 1 cat /sys/class/drm/card0/device/mem_info_vram_used

# VRAM 使用每帧增长 64MB
# 时间   VRAM 使用
# 10:00  4096MB
# 10:01  4160MB  (+64MB)
# 10:02  4224MB  (+64MB)
# ...

# 排查步骤:

# 1. 找出占用 VRAM 的进程
for pid in /proc/[0-9]*; do
    PID=$(basename "$pid")
    for fd in "$pid/fdinfo"/*; do
        if grep -q "drm-memory-vram" "$fd" 2>/dev/null; then
            VRAM=$(grep "drm-memory-vram" "$fd" 2>/dev/null | cut -d: -f2)
            if [ "${VRAM:-0}" -gt 1073741824 ]; then  # > 1GB
                echo "PID $PID ($(cat $pid/comm)): $((VRAM/1024/1024))MB"
            fi
        fi
    done
done

# 2. 查看 GEM BO 列表找出未释放的对象
cat /sys/kernel/debug/dri/0/amdgpu_gem_info | grep -A5 "5678"  # 可疑进程 PID

# 3. 确认泄漏对象
# 找到持续增长的 BO
# 每次执行都有新 BO 创建但未释放

# 解决方案:
# - 更新 GPU 驱动
# - 更新有问题的应用程序
# - 设置 VRAM 使用限制
echo 8589934592 > /sys/class/drm/card0/device/mem_info_vram_total  # 限制为 8GB
```

#### 案例 2：多进程 GPU 内存竞争

```bash
# 症状: 系统中有多个 GPU 应用性能下降

# 检查各进程内存分配
cat /sys/kernel/debug/dri/0/amdgpu_gem_info | tail -30

# 输出:
# PID 1234 (Xorg):        512MB VRAM
# PID 5678 (Chrome):      2048MB VRAM
# PID 9012 (Game):        4096MB VRAM
# PID 3456 (AI):          6144MB VRAM
# ─────────────────────────────
# 总 VRAM: 12760MB / 16384MB

# VRAM 已使用 78%
# 当接近 100% 时, 触发内存迁移 (eviction) 导致性能下降

# 解决方案:
# 1. 关闭不必要的应用
# 2. 调整内存优先级
echo high > /sys/class/drm/card0/device/mem_info_vram_priority
# 3. 使用 GTT 作为回退
```

### 相关链接

- DRM GEM 文档: https://docs.kernel.org/gpu/drm-mm.html
- AMDGPU 内存管理: https://docs.kernel.org/gpu/amdgpu/amdgpu-memory.html
- fdinfo DRM 接口: https://docs.kernel.org/gpu/drm-usage-stats.html
- DRM 内存统计文档: Documentation/gpu/drm-usage-stats.rst

### 今日小结

1. **GEM BO 模型**：GPU 内存管理的基本单位是 Buffer Object (BO)，支持 VRAM、GTT、CPU 三种内存类型。

2. **内存追踪接口**：sysfs 提供全局内存统计（mem_info_vram_used）、debugfs 提供 GEM BO 详细信息（amdgpu_gem_info）、fdinfo 提供进程级内存使用。

3. **进程级监控**：通过 proc fdinfo 中的 drm-memory-* 字段可以追踪每个进程的 GPU 内存使用情况。

4. **内存压力检测**：VRAM 使用超过 90% 时性能会下降，需要监控并处理内存竞争。

5. **泄漏排查**：通过比较创建和销毁的 BO 数量、监控 VRAM 使用趋势可以定位内存泄漏。

### 扩展思考

1. **内存过度分配**：如果多个进程申请的总 VRAM 超过物理容量，TTM 如何处理？eviction 机制如何选择被换出的 BO？

2. **HMM (Heterogeneous Memory Management)**：AMDGPU 如何利用 HMM 实现 GPU 和 CPU 的统一内存访问？这如何影响内存追踪？

3. **显存压缩**：DCC (Delta Color Compression) 和显存压缩技术如何影响实际显存使用的统计？如何计算真实有效显存使用量？