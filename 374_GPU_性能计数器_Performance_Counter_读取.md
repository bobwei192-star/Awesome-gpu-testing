# GPU 性能计数器（Performance Counter）读取

## 学习目标

- 理解 GPU 性能计数器的架构和工作原理
- 掌握 AMDGPU 性能计数器的类型和读取方法
- 学会使用 umr 和 perf 工具采集性能数据
- 能够通过性能计数器诊断 GPU 性能瓶颈

## 知识详解

### 概念原理

#### 性能计数器架构

```ascii
AMDGPU 性能计数器架构:

┌──────────────────────────────────────────────────────────────┐
│  GPU 性能计数器系统                                             │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  IP 级计数器:                                           │  │
│  │                                                         │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐      │  │
│  │  │  GFX    │ │  Memory │ │  Video  │ │  Display│      │  │
│  │  │  Pipe   │ │  Contr  │ │  Engine │ │  Engine │      │  │
│  │  │  12 cnt │ │  8 cnt  │ │  4 cnt  │ │  4 cnt  │      │  │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘      │  │
│  │                                                         │  │
│  │  每个 IP 块有专用的硬件计数器寄存器                          │  │
│  │  可配置为不同的事件源                                      │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  CP (Command Processor) 统计:                          │  │
│  │  - ME (微引擎) 统计: 命令处理周期                         │  │
│  │  - PFP (前端点) 统计: 取指/解码                          │  │
│  │  - CE (常量引擎) 统计: 常量读取                           │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  SPI (Shader Processor Input) 统计:                    │  │
│  │  - 线程发射率                                           │  │
│  │  - Wave 分配/等待                                       │  │
│  │  - LDS (本地数据共享) 冲突                               │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  内存层级计数器:                                        │  │
│  │  - L1/L2 Cache 命中/未命中                              │  │
│  │  - VRAM 读/写带宽                                       │  │
│  │  - PCIe 带宽                                            │  │
│  │  - 内存控制器利用率                                      │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

#### 计数器类型

```ascii
GFX IP 性能计数器 (Nav31 / RDNA3):

┌──────────────────────────────────────────────────────────────┐
│  计数器组        │ 事件数 │ 典型事件                           │
│ ────────────────┼───────┼───────────────────────────────    │
│  GFX Core        │ 256   │ gfx_BUSY, gfx_VALID, gfx_IDLE    │
│  Cache/L1       │ 64    │ l1_HIT, l1_MISS, l1_READ, l1_WRITE│
│  L2 Cache        │ 128   │ l2_HIT, l2_MISS, l2_READ_BW      │
│  Memory (VRAM)   │ 64    │ vram_READ, vram_WRITE, vram_BW   │
│  PCIe            │ 32    │ pcie_TX, pcie_RX, pcie_BW        │
│  VCN             │ 64    │ enc_BUSY, dec_BUSY, bitrate      │
│  SDMA            │ 32    │ sdma_BUSY, sdma_BW               │
│  CP              │ 32    │ cp_BUSY, cp_STALLED              │
│  SPI             │ 32    │ wave_ISSUE, wave_WAIT, lds_CONF  │
│  RLC             │ 16    │ rlc_BUSY, rlc_POWER              │
└──────────────────────────────────────────────────────────────┘

计数器数据路径:

┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ 硬件事件源     │───→│ 计数器寄存器  │───→│ 中断/采样     │
│ (Event Mux)   │    │ (32-bit 累加) │    │ (定时触发)    │
└──────────────┘    └──────────────┘    └──────────────┘
       │                    │                    │
       │ 事件选择            │ 计数值              │ 时间窗口
       ▼                    ▼                    ▼
┌──────────────────────────────────────────────────────────────┐
│  umr --perf / sysfs 接口                                     │
│                                                               │
│  计数器配置: { event_id, inst, slice, counter_slot }         │
│  读出计数值 ⟶ 除以采样时间 ⟶ 每秒事件数                       │
└──────────────────────────────────────────────────────────────┘
```

#### 关键性能指标

```ascii
典型 GPU 性能指标:

┌──────────────────────────────────────────────────────────────┐
│  指标              │ 公式/来源           │ 含义                │
│ ─────────────────┼───────────────────┼─────────────────    │
│  GFX 利用率 (%)    │ gfx_BUSY / total   │ GPU 核心活跃比例     │
│  像素填充率        │ gfx_VALID / time   │ 每秒渲染像素数       │
│  显存带宽 (GB/s)   │ vram_BW / time     │ VRAM 实际吞吐量      │
│  L2 命中率 (%)     │ l2_HIT/(HIT+MISS)  │ L2 Cache 效率        │
│  PCIe 带宽 (GB/s)  │ pcie_BW / time     │ PCIe 数据传输速率    │
│  Wave 发射率       │ wave_ISSUE / time   │ 着色器启动率         │
│  EU 利用率 (%)     │ wave_ACTIVE/total   │ ALU 单元利用率       │
│  内存延迟 (ns)     │ vram_LATENCY       │ 显存访问延迟          │
│  CP 停顿 (%)       │ cp_STALLED/total   │ 命令处理器等待时间    │
└──────────────────────────────────────────────────────────────┘

性能瓶颈诊断:

┌──────────────────────────────────────────────────────────────┐
│  症状                  │ 计数器模式           │ 可能原因       │
│ ─────────────────────┼───────────────────┼──────────────   │
│  GFX_BUSY 低, gfx_VALID 低 │ 计算/内存均空闲    │ CPU 绑定      │
│  GFX_BUSY 高, l2_MISS 高   │ 计算密集+缓存未命中 │ 内存带宽瓶颈   │
│  GFX_BUSY 高, vram_BW 高   │ 计算密集+高带宽    │ 计算+内存混合  │
│  GFX_BUSY 低, l2_HIT 低    │ GPU 等待数据       │ PCIe 带宽瓶颈  │
│  cp_STALLED 高             │ 命令处理停顿       │ DrawCall 过多  │
│  wave_WAIT 高              │ 线程等待 LDS       │ 局部内存冲突    │
└──────────────────────────────────────────────────────────────┘
```

### 实践操作

#### 使用 umr 读取性能计数器

```bash
# 1. 列出可用的性能计数器
sudo umr --perf list

# 输出:
# Available performance counters:
# ┌──────────────────────┬────────────────────────────────────┐
# │ Counter Name         │ Description                        │
# ├──────────────────────┼────────────────────────────────────┤
# │ gfx_BUSY             │ GFX pipe busy clock cycles         │
# │ gfx_VALID            │ GFX pipe valid pixel throughput    │
# │ gfx_IDLE             │ GFX pipe idle cycles               │
# │ gfx_CACHE_HIT        │ GFX data cache hit rate            │
# │ gfx_CACHE_MISS       │ GFX data cache miss rate           │
# │ l1_HIT               │ L1 cache hit counter               │
# │ l1_MISS              │ L1 cache miss counter              │
# │ l2_HIT               │ L2 cache hit counter               │
# │ l2_MISS              │ L2 cache miss counter              │
# │ vram_READ            │ VRAM read transactions             │
# │ vram_WRITE           │ VRAM write transactions            │
# │ vram_BW              │ VRAM bandwidth in bytes            │
# │ pcie_TX              │ PCIe transmit bytes                │
# │ pcie_RX              │ PCIe receive bytes                 │
# │ wave_ISSUE           │ Wavefronts issued                  │
# │ wave_WAIT            │ Wavefronts waiting for LDS         │
# │ cp_BUSY              │ Command processor busy             │
# │ cp_STALLED           │ Command processor stalled          │
# │ sdma_BUSY            │ SDMA engine busy                   │
# │ enc_BUSY             │ Video encoder busy                 │
# │ dec_BUSY             │ Video decoder busy                 │
# └──────────────────────┴────────────────────────────────────┘

# 2. 启用计数器
sudo umr --perf enable gfx_BUSY,gfx_VALID,l2_HIT,l2_MISS

# 3. 采样计数器 (采样 2 秒)
sudo umr --perf sample 2000

# 输出:
# Sampling for 2000ms...
# ┌────────────────┬────────────┬──────────────┬────────────────┐
# │ Counter        │ Value      │ Rate/s       │ %              │
# ├────────────────┼────────────┼──────────────┼────────────────┤
# │ gfx_BUSY       │ 1234567890 │ 617283945    │ 61.7%          │
# │ gfx_VALID      │ 987654321  │ 493827160    │ 49.4 MPixels/s │
# │ l2_HIT         │ 5000000000 │ 2500000000   │ 83.3%          │
# │ l2_MISS        │ 1000000000 │ 500000000    │ 16.7%          │
# └────────────────┴────────────┴──────────────┴────────────────┘

# 4. 连续监控
sudo umr --perf enable gfx_BUSY,gfx_VALID --monitor 1
```

#### 使用自定义计数器配置

```bash
# 1. 创建计数器配置文件
# perf_config.json
sudo tee /tmp/perf_config.json << 'EOF'
{
    "counters": [
        {
            "name": "gfx_pipe_0",
            "ip": "GFX",
            "instance": 0,
            "event_id": 0x01,
            "counter_slot": 0
        },
        {
            "name": "gfx_pipe_1",
            "ip": "GFX",
            "instance": 1,
            "event_id": 0x01,
            "counter_slot": 1
        },
        {
            "name": "l2_cache_miss",
            "ip": "L2_CACHE",
            "instance": 0,
            "event_id": 0x03,
            "counter_slot": 2
        },
        {
            "name": "vram_bandwidth",
            "ip": "UMC",
            "instance": 0,
            "event_id": 0x10,
            "counter_slot": 3
        }
    ],
    "sampling": {
        "interval_ms": 1000,
        "total_ms": 10000
    }
}
EOF

# 2. 使用配置文件采样
sudo umr --perf config /tmp/perf_config.json

# 3. 保存结果到文件
sudo umr --perf config /tmp/perf_config.json --output perf_result.csv

# CSV 输出格式:
# timestamp,gfx_pipe_0,gfx_pipe_1,l2_cache_miss,vram_bandwidth
# 1000,123456,789012,3456,987654321
# 2000,234567,890123,4567,876543210
# ...
```

#### 批量性能分析脚本

```bash
#!/bin/bash
# gpu_perf_analyzer.sh - GPU 性能分析脚本

echo "GPU 性能分析工具"
echo "================="

# 1. 基本性能概览
echo ""
echo "--- 基本性能概览 ---"

# 使用 umr 快速采样
sudo umr --perf enable gfx_BUSY,gfx_VALID,l2_HIT,l2_MISS,vram_BW,pcie_TX
sudo umr --perf sample 3000

# 2. GFX 管线利用率
echo ""
echo "--- GFX 管线利用率 (5 秒采样) ---"
sudo umr --perf enable gfx_BUSY,cp_BUSY,cp_STALLED,wave_ISSUE
sudo umr --perf sample 5000

# 3. 内存系统分析
echo ""
echo "--- 内存系统分析 (5 秒采样) ---"
sudo umr --perf enable l1_HIT,l1_MISS,l2_HIT,l2_MISS,vram_READ,vram_WRITE
sudo umr --perf sample 5000

# 4. PCIe 带宽分析
echo ""
echo "--- PCIe 带宽分析 (3 秒采样) ---"
sudo umr --perf enable pcie_TX,pcie_RX
sudo umr --perf sample 3000

echo ""
echo "分析完成"
```

#### 使用 perf 工具

```bash
# 1. 使用 Linux perf 工具监控 GPU
sudo perf stat -e amdgpu:gfx_busy_cycle -I 1000

# 2. 查看可用的 AMDGPU trace events
sudo cat /sys/kernel/debug/tracing/available_events | grep amdgpu | head -20

# 输出:
# amdgpu:amdgpu_vm_flush
# amdgpu:amdgpu_cs_ioctl
# amdgpu:amdgpu_sched_run_job
# amdgpu:amdgpu_gem_object_create
# amdgpu:amdgpu_gem_object_free
# amdgpu:amdgpu_fence_emit

# 3. 启用 trace event 监控 GFX 使用
echo 1 > /sys/kernel/debug/tracing/events/amdgpu/enable
cat /sys/kernel/debug/tracing/trace_pipe

# 4. 特定事件的 trace
echo 1 > /sys/kernel/debug/tracing/events/amdgpu/amdgpu_sched_run_job/enable
cat /sys/kernel/debug/tracing/trace_pipe | head -20
```

#### Python 性能数据采集器

```python
#!/usr/bin/env python3
# gpu_perf_collector.py - GPU 性能数据采集器

import json
import time
import signal
import subprocess
from datetime import datetime
from pathlib import Path
from typing import Dict, List, Optional


class GPUPerfCollector:
    """GPU 性能数据采集器"""
    
    def __init__(self, gpu_index: int = 0):
        self.gpu_index = gpu_index
        self.running = False
        self.samples: List[Dict] = []
    
    def read_sysfs_gpu_busy(self) -> Optional[float]:
        """读取 GPU 利用率"""
        path = f"/sys/class/drm/card{self.gpu_index}/device/gpu_busy_percent"
        try:
            with open(path) as f:
                return float(f.read().strip())
        except (FileNotFoundError, ValueError):
            return None
    
    def read_memory_stats(self) -> Dict:
        """读取内存统计"""
        base = f"/sys/class/drm/card{self.gpu_index}/device"
        stats = {}
        
        keys = [
            "mem_info_vram_total", "mem_info_vram_used",
            "mem_info_gtt_total", "mem_info_gtt_used",
            "mem_info_vis_vram_total", "mem_info_vis_vram_used",
        ]
        
        for key in keys:
            path = f"{base}/{key}"
            try:
                with open(path) as f:
                    stats[key] = int(f.read().strip())
            except FileNotFoundError:
                pass
        
        return stats
    
    def read_power_stats(self) -> Dict:
        """读取功耗统计"""
        base = f"/sys/class/drm/card{self.gpu_index}/device/hwmon"
        stats = {}
        
        try:
            hwmon_dir = Path(base)
            if not hwmon_dir.exists():
                return stats
            
            for hwmon in hwmon_dir.iterdir():
                power_path = hwmon / "power1_average"
                temp_path = hwmon / "temp1_input"
                
                if power_path.exists():
                    with open(power_path) as f:
                        stats["power_uW"] = int(f.read().strip())
                
                if temp_path.exists():
                    with open(temp_path) as f:
                        stats["temp_mC"] = int(f.read().strip())
        except (PermissionError, FileNotFoundError):
            pass
        
        return stats
    
    def read_clocks(self) -> Dict:
        """读取时钟频率"""
        base = f"/sys/class/drm/card{self.gpu_index}/device"
        clocks = {}
        
        clock_files = {
            "gfx_clock": "pp_dpm_sclk",
            "mem_clock": "pp_dpm_mclk",
        }
        
        for name, file in clock_files.items():
            path = f"{base}/{file}"
            try:
                with open(path) as f:
                    clocks[name] = f.read().strip()
            except FileNotFoundError:
                pass
        
        return clocks
    
    def collect_sample(self) -> Dict:
        """采集一次样本"""
        sample = {
            "timestamp": datetime.now().isoformat(),
            "gpu_busy_percent": self.read_sysfs_gpu_busy(),
            "memory": self.read_memory_stats(),
            "power": self.read_power_stats(),
            "clocks": self.read_clocks(),
        }
        
        return sample
    
    def collect_umr_counters(self, counters: List[str],
                              duration_ms: int = 2000) -> Dict:
        """使用 umr 采集性能计数器"""
        counter_str = ",".join(counters)
        
        try:
            result = subprocess.run(
                ["sudo", "umr", "--perf", "enable", counter_str,
                 "--perf", "sample", str(duration_ms)],
                capture_output=True, text=True, timeout=duration_ms // 1000 + 5
            )
            return self._parse_umr_output(result.stdout)
        except (subprocess.TimeoutError, subprocess.CalledProcessError) as e:
            return {"error": str(e)}
    
    def _parse_umr_output(self, output: str) -> Dict:
        """解析 umr 输出"""
        data = {}
        for line in output.split("\n"):
            parts = line.split("│")
            if len(parts) >= 4:
                name = parts[1].strip()
                value = parts[2].strip()
                rate = parts[3].strip()
                if name and value:
                    data[name] = {
                        "value": self._parse_num(value),
                        "rate": rate
                    }
        return data
    
    def _parse_num(self, s: str) -> int:
        """解析数值"""
        s = s.replace(",", "").strip()
        try:
            return int(s)
        except ValueError:
            return 0
    
    def start_monitoring(self, interval: float = 1.0):
        """开始持续监控"""
        self.running = True
        self.samples = []
        
        def signal_handler(signum, frame):
            self.running = False
        
        signal.signal(signal.SIGINT, signal_handler)
        
        print(f"{'时间':20s} {'GPU%':8s} {'VRAM':12s} {'功率':10s} {'温度':8s}")
        print("-" * 60)
        
        try:
            while self.running:
                sample = self.collect_sample()
                self.samples.append(sample)
                
                gpu_pct = sample.get("gpu_busy_percent", 0)
                vram = "N/A"
                if "memory" in sample and "mem_info_vram_used" in sample["memory"]:
                    vram = f"{sample['memory']['mem_info_vram_used'] // (1024*1024)}MB"
                power = "N/A"
                if "power" in sample and "power_uW" in sample["power"]:
                    power = f"{sample['power']['power_uW'] / 1000000:.1f}W"
                temp = "N/A"
                if "power" in sample and "temp_mC" in sample["power"]:
                    temp = f"{sample['power']['temp_mC'] / 1000:.0f}°C"
                
                now = datetime.now().strftime("%H:%M:%S")
                print(f"{now:20s} {gpu_pct:>6.1f}%  {vram:>10s}  {power:>8s}  {temp:>6s}")
                
                time.sleep(interval)
        except KeyboardInterrupt:
            pass
        
        print(f"\n采集完成: {len(self.samples)} 个样本")
    
    def generate_report(self) -> str:
        """生成报告"""
        if not self.samples:
            return "No data collected"
        
        gpu_busy = [s.get("gpu_busy_percent", 0) or 0 for s in self.samples]
        vram_used = []
        for s in self.samples:
            if "memory" in s and "mem_info_vram_used" in s["memory"]:
                vram_used.append(s["memory"]["mem_info_vram_used"])
        
        report = []
        report.append("GPU 性能报告")
        report.append("=" * 40)
        report.append(f"采样次数: {len(self.samples)}")
        report.append(f"采样间隔: 1 秒")
        
        if gpu_busy:
            report.append(f"\nGPU 利用率:")
            report.append(f"  平均: {sum(gpu_busy)/len(gpu_busy):.1f}%")
            report.append(f"  峰值: {max(gpu_busy):.1f}%")
            report.append(f"  最低: {min(gpu_busy):.1f}%")
        
        if vram_used:
            avg_vram = sum(vram_used) / len(vram_used) / (1024*1024)
            max_vram = max(vram_used) / (1024*1024)
            report.append(f"\nVRAM 使用:")
            report.append(f"  平均: {avg_vram:.0f}MB")
            report.append(f"  峰值: {max_vram:.0f}MB")
        
        power_samples = [s.get("power", {}).get("power_uW", 0) or 0
                        for s in self.samples]
        if any(power_samples):
            avg_power = sum(power_samples) / len(power_samples) / 1000000
            max_power = max(power_samples) / 1000000
            report.append(f"\n功耗:")
            report.append(f"  平均: {avg_power:.1f}W")
            report.append(f"  峰值: {max_power:.1f}W")
        
        return "\n".join(report)
    
    def export_csv(self, path: str):
        """导出 CSV"""
        if not self.samples:
            return
        
        with open(path, "w") as f:
            f.write("timestamp,gpu_busy,vram_used,power, temperature\n")
            for s in self.samples:
                ts = s["timestamp"]
                busy = s.get("gpu_busy_percent", "")
                vram = ""
                if "memory" in s and "mem_info_vram_used" in s["memory"]:
                    vram = s["memory"]["mem_info_vram_used"]
                power = ""
                temp = ""
                if "power" in s:
                    power = s["power"].get("power_uW", "")
                    temp = s["power"].get("temp_mC", "")
                f.write(f"{ts},{busy},{vram},{power},{temp}\n")


# 性能诊断函数
def diagnose_performance(collector: GPUPerfCollector):
    """诊断性能瓶颈"""
    
    # 采集 umr 计数器
    counters = ["gfx_BUSY", "gfx_VALID", "l2_HIT", "l2_MISS",
                "vram_BW", "cp_STALLED"]
    umr_data = collector.collect_umr_counters(counters, 5000)
    
    diag = []
    
    if "gfx_BUSY" in umr_data:
        gfx_busy = umr_data["gfx_BUSY"]["value"]
        if gfx_busy < 50:
            diag.append("检测: GPU 利用率 < 50%")
            diag.append("诊断: 可能受 CPU 绑定 (CPU 是瓶颈)")
        elif gfx_busy > 90:
            diag.append("检测: GPU 利用率 > 90%")
            
            if "l2_MISS" in umr_data:
                l2_miss = umr_data["l2_MISS"]["value"]
                l2_hit = umr_data["l2_HIT"]["value"]
                total = l2_hit + l2_miss
                if total > 0:
                    miss_rate = l2_miss / total * 100
                    if miss_rate > 20:
                        diag.append(f"检测: L2 Cache 未命中率 {miss_rate:.1f}%")
                        diag.append("诊断: 内存带宽可能是瓶颈")
            
            if "cp_STALLED" in umr_data:
                cp_stalled = umr_data["cp_STALLED"]["value"]
                if cp_stalled > 1000000:
                    diag.append("检测: CP 停顿频繁")
                    diag.append("诊断: Draw Call 可能过多, 考虑批处理优化")
    
    return "\n".join(diag)


# 主程序
if __name__ == "__main__":
    import sys
    
    collector = GPUPerfCollector()
    
    if len(sys.argv) > 1 and sys.argv[1] == "--monitor":
        interval = float(sys.argv[2]) if len(sys.argv) > 2 else 1.0
        collector.start_monitoring(interval)
        print()
        print(collector.generate_report())
    
    elif len(sys.argv) > 1 and sys.argv[1] == "--diagnose":
        print("运行性能诊断...")
        diag = diagnose_performance(collector)
        print(diag)
    
    elif len(sys.argv) > 1 and sys.argv[1] == "--export":
        if len(sys.argv) > 2:
            collector.start_monitoring(0.5)
            collector.export_csv(sys.argv[2])
            print(f"数据已导出到 {sys.argv[2]}")
    
    else:
        sample = collector.collect_sample()
        print(json.dumps(sample, indent=2, default=str))
```

### 案例分析

#### 案例 1：L2 Cache 未命中导致性能下降

```bash
# 场景: 3D 渲染性能只有预期的 60%

# 性能数据:
sudo umr --perf enable gfx_BUSY,gfx_VALID,l2_HIT,l2_MISS,vram_BW
sudo umr --perf sample 3000

# 输出:
# gfx_BUSY:      92%
# gfx_VALID:     180 MPixels/s (预期 300 MPixels/s)
# l2_HIT:        3500000000
# l2_MISS:       1500000000  ★ 未命中率 30%!
# vram_BW:       350 GB/s (接近带宽上限)

# 分析:
# - GPU 非常忙碌 (92%)
# - 但像素填充率只有 60%
# - L2 未命中率 30% (正常应 < 10%)
# - VRAM 带宽接近饱和

# 诊断: 内存带宽瓶颈
# - 渲染分辨率过高 (4K)
# - 纹理数据超过 L2 Cache 大小
# - 多次访问未缓存的显存区域

# 优化方案:
# 1. 减少纹理大小: 使用 mipmap
# 2. 启用 DCC 压缩
# 3. 降低渲染分辨率
# 4. 检查纹理格式 (使用 BCn 压缩)
```

#### 案例 2：Draw Call 过多导致 CP 停顿

```bash
# 场景: CPU 帧率限制

# 性能数据:
sudo umr --perf enable gfx_BUSY,cp_BUSY,cp_STALLED
sudo umr --perf sample 3000

# 输出:
# gfx_BUSY:      45%
# cp_BUSY:       95%
# cp_STALLED:    8000000 ★ 非常高

# 分析:
# - GPU 利用率只有 45%
# - CP (命令处理器) 非常忙碌
# - CP 停顿计数极高

# 诊断: Draw Call 过多导致 CPU 瓶颈
# - 每个 Draw Call 需要 CP 处理
# - 当 Draw Call 过多时 CP 成为瓶颈
# - GPU 核心空闲等待命令

# 排查:
# 使用 GPU 调试工具查看 Draw Call 数量
sudo umr --perf enable cp_STALLED --monitor 1

# 解决方案:
# 1. 合�� Draw Call (批处理)
# 2. 使用实例化渲染 (Instancing)
# 3. 减少状态切换
# 4. 使用间接绘制 (Indirect Draw)
```

### 相关链接

- AMDGPU 性能计数器文档: https://docs.kernel.org/gpu/amdgpu/perf-counters.html
- umr 工具文档: https://gitlab.freedesktop.org/tomstdenis/umr
- Linux perf 工具: https://perf.wiki.kernel.org/
- AMD ROCm Profiler: https://rocm.docs.amd.com/en/latest/

### 今日小结

1. **计数器架构**：GPU 性能计数器分布在各个 IP 块（GFX、Cache、Memory、PCIE 等），每个 IP 块有独立的硬件计数器寄存器。

2. **关键指标**：GFX 利用率、像素填充率、Cache 命中率、显存带宽、PCIE 带宽是诊断性能瓶颈的核心指标。

3. **采集工具**：umr --perf 提供灵活的计数器配置和采样功能；sysfs 提供实时 GPU 利用率和内存统计。

4. **瓶颈诊断**：不同的计数器组合可以诊断不同类型的性能瓶颈（CPU 绑定、内存带宽、Draw Call 限制等）。

5. **分析方法**：同时监控多个相关计数器、对比实际值与预期值、分析计数器间的关联性可以发现深层性能问题。

### 扩展思考

1. **多代 GPU 差异**：RDNA2、RDNA3、CDNA 等不同架构的性能计数器有何差异？计数器事件 ID 如何对应到具体的硬件单元？

2. **计数器开销**：启用大量性能计数器是否会引入性能开销？如何平衡监控精度和性能影响？

3. **AI 辅助分析**：如何利用机器学习自动分析性能计数器数据，识别性能异常和瓶颈模式？