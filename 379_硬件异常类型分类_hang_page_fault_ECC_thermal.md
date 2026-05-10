# 硬件异常类型分类：hang / page fault / ECC / thermal

## 学习目标

- 理解 GPU 硬件异常的四种主要类型及其原理
- 掌握每种异常类型的识别方法、诊断工具和排查步骤
- 学会通过日志特征区分不同类型的异常
- 能够在多异常并发时确定根因

## 知识详解

### 概念原理

#### GPU 异常分类总览

```ascii
GPU 硬件异常分类:

┌──────────────────────────────────────────────────────────────┐
│  GPU 硬件异常                                                  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 1. Hang (挂起/死锁)                                     │  │
│  │                                                         │  │
│  │  ┌──────────────────────┐  ┌──────────────────────┐    │  │
│  │  │ GPU Core Hang        │  │ Memory Hang          │    │  │
│  │  │ - 着色器死循环         │  │ - 内存控制器死锁      │    │  │
│  │  │ - CP/PFP 卡住         │  │ - VRAM 地址死锁       │    │  │
│  │  │ - Wave 无法前进       │  │ - Page Table Walk Hang│    │  │
│  │  └──────────────────────┘  └──────────────────────┘    │  │
│  │  ┌──────────────────────┐  ┌──────────────────────┐    │  │
│  │  │ Command Processor    │  │ Display Engine       │    │  │
│  │  │ - Ring 无前进         │  │ - DCN 管线挂起        │    │  │
│  │  │ - IB 解析卡住         │  │ - OTG 无 VBlank      │    │  │
│  │  │ - DMA 超时            │  │ - DSC 编码锁定       │    │  │
│  │  └──────────────────────┘  └──────────────────────┘    │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 2. Page Fault (页错误)                                  │  │
│  │                                                         │  │
│  │  ┌──────────────────────┐  ┌──────────────────────┐    │  │
│  │  │ GPU VM Page Fault    │  │ Invalid Address      │    │  │
│  │  │ - 无效的 GPU VA      │  │ - 地址未映射          │    │  │
│  │  │ - 权限错误             │  │ - 地址已释放          │    │  │
│  │  │ - 页表条目无效         │  │ - 越界访问            │    │  │
│  │  └──────────────────────┘  └──────────────────────┘    │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 3. ECC (Error Correcting Code) 错误                     │  │
│  │                                                         │  │
│  │  ┌──────────────────────┐  ┌──────────────────────┐    │  │
│  │  │ CE (可纠正)          │  │ UE (不可纠正)         │    │  │
│  │  │ - 单比特翻转          │  │ - 多比特翻转          │    │  │
│  │  │ - 硬件自动纠正        │  │ - 数据已损坏          │    │  │
│  │  │ - 计数+继续运行        │  │ - 需要恢复或隔离      │    │  │
│  │  └──────────────────────┘  └──────────────────────┘    │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 4. Thermal (热异常)                                     │  │
│  │                                                         │  │
│  │  ┌──────────────────────┐  ┌──────────────────────┐    │  │
│  │  │ 过温                  │  │ 温控降频              │    │  │
│  │  │ - 温度 > 阈值         │  │ - throttling 激活     │    │  │
│  │  │ - 可能热关断           │  │ - 性能下降            │    │  │
│  │  │ - 硬件损坏风险         │  │ - 温度恢复正常后解除   │    │  │
│  │  └──────────────────────┘  └──────────────────────┘    │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

#### 各异常类型的日志特征

```ascii
异常类型日志特征对比:

┌──────────────────────────────────────────────────────────────┐
│  Hang 日志特征:                                                │
│  ──────────────────────────────────────────────────────────  │
│  ● "GPU hang detected on GFX pipe"                           │
│  ● "GFX is busy, no progress for N seconds"                   │
│  ● "GRBM_STATUS=0xA0000002" (GUI_ACTIVE|CP_BUSY)              │
│  ● "CP_STATUS=0x80000000" (CP 卡住)                           │
│  ● "ring GFX timeout" / "ring SDMA timeout"                  │
│  ● 通常周期性出现 (看门狗超时)                                  │
│  ● GPU 锁定, 无显示更新                                        │
│                                                               │
│  Page Fault 日志特征:                                          │
│  ──────────────────────────────────────────────────────────  │
│  ● "GPU page fault at address 0xDEADBEEF"                    │
│  ● "VM fault on GPU 0x..."                                   │
│  ● "Invalid page table entry"                                 │
│  ● "Permission fault" (只读页写访问)                           │
│  ● 通常有完整的错误地址和页表信息                                │
│  ● 可能触发 hang (如果 VM 错误导致 GPU 无法继续)                │
│                                                               │
│  ECC 错误日志特征:                                             │
│  ──────────────────────────────────────────────────────────  │
│  ● "RAS: Corrected ECC error at 0x..."                        │
│  ● "RAS: Uncorrectable ECC error at 0x..."                   │
│  ● "UMC CE count: N" / "UMC UE count: N"                     │
│  ● 包含 ECC 校正子 (syndrome)                                  │
│  ● CE: 仅计数, GPU 继续运行                                    │
│  ● UE: 需要恢复                                               │
│                                                               │
│  Thermal 异常日志特征:                                          │
│  ──────────────────────────────────────────────────────────  │
│  ● "Thermal alert: GPU temp = 95°C"                          │
│  ● "Throttling GFX clock" / "Throttling memory clock"        │
│  ● "Temperature limit reached"                               │
│  ● "Critical temperature threshold exceeded"                   │
│  ● "GPU overheat, shutting down"                              │
│  ● 温度值会持续显示                                             │
└──────────────────────────────────────────────────────────────┘
```

#### 异常关联分析

```ascii
异常之间的关联关系:

┌──────────────────────────────────────────────────────────────┐
│  常见异常链:                                                    │
│                                                               │
│  Chain 1: 热→降频→性能→页错误→Hang                                  │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐          │
│  │过热   │─→│降频   │─→│超时   │─→│页错误  │─→│Hang   │          │
│  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘          │
│                                                               │
│  Chain 2: ECC→数据损坏→应用崩溃→驱动恢复                          │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐                      │
│  │ECC UE│─→│损坏数据│─→│应用崩溃 │─→│GPU恢复│                      │
│  └──────┘  └──────┘  └──────┘  └──────┘                      │
│                                                               │
│  Chain 3: 页错误→页表损坏→VM hang→GPU 重置                        │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐                      │
│  │页错误  │─→│页表损坏│─→│VM hang│─→│GPU重置│                      │
│  └──────┘  └──────┘  └──────┘  └──────┘                      │
│                                                               │
│  混合异常诊断:                                                   │
│                                                               │
│  当 Hang 和 ECC 同时出现时:                                     │
│  1. 先确定时间顺序 (哪个先发生)                                   │
│  2. 如果是 ECC→Hang: 内存损坏导致数据处理错误                    │
│  3. 如果是 Hang→ECC: GPU 卡住导致内存访问超时, 可能误报 ECC      │
│                                                               │
│  当 Hang 和 Thermal 同时出现时:                                 │
│  1. 如果是 Thermal→Hang: 过热导致降频, 渲染超时触发看门狗        │
│  2. 如果是 Hang→Thermal: GPU 卡在全速运行, 功耗/温度上升         │
│                                                               │
│  当 Page Fault 和 Hang 同时出现时:                              │
│  1. Page Fault→Hang: VM 错误导致 GPU 无法继续                    │
│  2. 通常 page fault 日志在 hang 日志之前                          │
└──────────────────────────────────────────────────────────────┘
```

### 实践操作

#### Hang 异常诊断

```bash
# 1. 检测 GPU hang
dmesg | grep -E "hang|timeout|no progress"

# 输出:
# [ 1234.567] [drm] GPU hang detected on GFX pipe
# [ 1234.567] [drm] GRBM_STATUS=0xA0000002
# [ 1234.567] [drm] CP_STATUS=0x80000000
# [ 1234.567] [drm] GFX is busy, no progress for 2 seconds

# 2. GRBM_STATUS 寄存器解读
# 位域含义:
# bit0:  GUI_ACTIVE (GUI 正在处理)
# bit1:  CP_BUSY (命令处理器忙碌)
# bit2:  CP_COHERENCY (CP 一致性)
# bit4:  RLC_BUSY (RLC 忙碌)
# bit5:  GFX_CLK (GFX 时钟)
# bit29: GUI_IDLE (GUI 空闲)

# 0xA0000002:
# bit1  = 1: CP_BUSY
# bit29 = 1: NOT GUI_IDLE (= GUI_ACTIVE)
# bit30 = 1: 保留
# bit31 = 1: 保留
# 解读: CP 忙碌 + GUI 活跃 = 渲染未完成

# 3. 查看 CP 状态
cat /sys/kernel/debug/dri/0/amdgpu_ring_gfx

# 输出:
# ring GFX: busy, last_seq=1234, last_fence=1240
# ring GFX: active IBs: 3
# ring GFX: pending IBs: 5

# 4. 检查 hang 的 wave 状态
sudo umr --waves --verbose
```

#### Page Fault 诊断

```bash
# 1. 检测 page fault
dmesg | grep -i "page fault\|VM fault\|invalid address"

# 输出:
# [ 1234.567] [drm] GPU page fault at address 0x0000000012345678
# [ 1234.567] [drm] VM fault on GPU 0x0000000012345678
# [ 1234.567] [drm] Invalid page table entry at VA 0x12345678
# [ 1234.567] [drm] Page walk: PTE=0x0000000000000000 (invalid)
# [ 1234.567] [drm] VM fault flags: READ | WRITE | INVALID
# [ 1234.567] [drm] Fault from process PID 5678 (game)

# 2. 分析 page fault
# 关键信息:
# - 地址: 0x12345678 (GPU 虚拟地址)
# - 原因: PTE 无效 (页表条目不存在)
# - 来源: PID 5678 (进程名 game)
# - 操作: READ | WRITE

# 3. 查看进程的虚拟地址映射
cat /proc/5678/maps | grep "dri"

# 4. 检查 GPU 页表
# (需要 debugfs)
cat /sys/kernel/debug/dri/0/amdgpu_vm_pd 2>/dev/null

# 5. 判断 page fault 类型
# 无效地址: 写入未分配的内存区域
#   可能原因: 用户态驱动 bug, 数据竞争
# 权限错误: 写入只读页面
#   可能原因: 共享内存使用不当
# 已释放地址: 使用已释放的 BO
#   可能原因: use-after-free bug
```

#### ECC 错误诊断

```bash
# 1. 检测 ECC 错误
dmesg | grep -i "ECC\|corrected\|uncorrectable\|ras"

# 输出:
# [ 1234.567] [drm] RAS: UMC CE error at 0x00200000 (corrected)
# [ 1234.567] [drm] RAS: UMC CE syndrome: 0x1234
# [ 1234.567] [drm] RAS: UMC Ch1 CE count: 15
# [ 1234.567] [drm] RAS: UMC Ch1 UE count: 0

# 2. 查看所有 ECC 计数器
cat /sys/class/drm/card0/device/ras_ecc_cnt

# 3. CE 错误分析
# - 如果 CE 错误分散在所有通道: 可能是辐射/背景噪声
# - 如果 CE 错误集中在单一通道: 可能是硬件缺陷
# - 如果 CE 错误集中在同一地址: 可能是行锤 (row hammer)

# 4. UE 错误分析
# - UE 通常意味着硬件损伤
# - 记录地址并检查地址使用情况
# - 如果 UE 地址在内核保留区域: 需要 GPU 重置
# - 如果 UE 地址在应用内存: 应用会崩溃

# 5. ECC 趋势分析
# CE 增长率判断:
# < 1/小时: 正常
# 1-10/小时: 轻度关注
# 10-100/小时: 需要监控
# > 100/小时: 需要更换
```

#### Thermal 异常诊断

```bash
# 1. 检测温度异常
dmesg | grep -i "thermal\|temperature\|throttl\|overheat"

# 输出:
# [ 1234.567] [drm] Thermal alert: GPU temp = 95°C
# [ 1234.567] [drm] Throttling: GFX clock reduced to 1200MHz
# [ 1234.567] [drm] Thermal alarm: critical temp 105°C!

# 2. 查看当前温度
cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input
# 输出: 95000 (即 95°C)

# 3. 查看温度阈值
cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_crit
# 输出: 105000 (临界温度 105°C)

cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_emergency
# 输出: 110000 (紧急温度 110°C)

# 4. 查看降频状态
cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap_max
cat /sys/class/drm/card0/device/pp_dpm_sclk

# 5. 温度历史监控
watch -n 1 cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input
```

#### 综合异常诊断脚本

```python
#!/usr/bin/env python3
# gpu_exception_diagnose.py - GPU 异常诊断工具

import re
import sys
import subprocess
from typing import List, Dict, Optional
from collections import Counter
from dataclasses import dataclass


@dataclass
class DiagnosedException:
    """诊断的异常"""
    type: str           # hang / page_fault / ecc / thermal
    severity: str       # info / warning / error / critical
    message: str
    timestamp: float
    details: Dict


class GPUExceptionDiagnoser:
    """GPU 异常诊断器"""
    
    PATTERNS = {
        "hang": re.compile(
            r'GPU hang detected|no progress|ring.*timeout|'
            r'GRBM_STATUS|CP_STATUS'
        ),
        "page_fault": re.compile(
            r'page fault|VM fault|Invalid page table|'
            r'invalid address|Permission fault'
        ),
        "ecc_ce": re.compile(
            r'Corrected ECC error|CE error|corrected error'
        ),
        "ecc_ue": re.compile(
            r'Uncorrectable ECC error|UE error|uncorrectable error'
        ),
        "thermal": re.compile(
            r'Thermal alert|throttling|overheat|'
            r'temperature.*limit|critical temp'
        ),
    }
    
    def __init__(self, log_lines: List[str]):
        self.lines = log_lines
        self.exceptions: List[DiagnosedException] = []
        self._parse()
    
    def _parse(self):
        """解析所有异常"""
        # 先提取时间戳和消息
        entries = []
        for line in self.lines:
            match = re.match(r'\[(\s*[\d.]+)\]\s*(.*)', line)
            if match:
                try:
                    ts = float(match.group(1).strip())
                    msg = match.group(2)
                    entries.append((ts, msg))
                except ValueError:
                    continue
        
        # 分类异常
        for ts, msg in entries:
            exc = None
            
            for exc_type, pattern in self.PATTERNS.items():
                if pattern.search(msg):
                    severity = self._determine_severity(exc_type, msg)
                    details = self._extract_details(exc_type, msg)
                    
                    exc = DiagnosedException(
                        type=exc_type,
                        severity=severity,
                        message=msg[:100],
                        timestamp=ts,
                        details=details,
                    )
                    self.exceptions.append(exc)
    
    def _determine_severity(self, exc_type: str, msg: str) -> str:
        severity_map = {
            "hang": "error",
            "page_fault": "warning",
            "ecc_ce": "info",
            "ecc_ue": "error",
            "thermal": "warning",
        }
        return severity_map.get(exc_type, "info")
    
    def _extract_details(self, exc_type: str, msg: str) -> Dict:
        details = {}
        
        if exc_type == "hang":
            grbm = re.search(r'GRBM_STATUS=(0x[0-9a-fA-F]+)', msg)
            if grbm:
                details["grbm_status"] = grbm.group(1)
        
        elif exc_type == "page_fault":
            addr = re.search(r'at address (0x[0-9a-fA-F]+)', msg)
            if addr:
                details["address"] = addr.group(1)
            pid = re.search(r'PID (\d+)', msg)
            if pid:
                details["pid"] = pid.group(1)
        
        elif exc_type in ("ecc_ce", "ecc_ue"):
            addr = re.search(r'at (0x[0-9a-fA-F]+)', msg)
            if addr:
                details["address"] = addr.group(1)
        
        elif exc_type == "thermal":
            temp = re.search(r'(\d+)°C', msg)
            if temp:
                details["temperature"] = temp.group(1)
        
        return details
    
    def analyze(self) -> Dict:
        """综合分析"""
        if not self.exceptions:
            return {"status": "no_exceptions", "message": "No exceptions found"}
        
        # 按类型统计
        type_counts = Counter(e.type for e in self.exceptions)
        severity_counts = Counter(e.severity for e in self.exceptions)
        
        # 按时间排序
        sorted_excs = sorted(self.exceptions, key=lambda e: e.timestamp)
        
        # 检测异常链
        chains = self._detect_chains(sorted_excs)
        
        return {
            "status": "has_exceptions",
            "total": len(self.exceptions),
            "by_type": dict(type_counts),
            "by_severity": dict(severity_counts),
            "chains": chains,
            "time_range": {
                "start": sorted_excs[0].timestamp,
                "end": sorted_excs[-1].timestamp,
            },
            "critical": [e for e in self.exceptions
                        if e.severity in ("error", "critical")],
        }
    
    def _detect_chains(
        self, excs: List[DiagnosedException], 
        window: float = 5.0
    ) -> List[Dict]:
        """检测异常链"""
        chains = []
        i = 0
        
        while i < len(excs):
            chain = [excs[i]]
            j = i + 1
            while j < len(excs) and excs[j].timestamp - excs[i].timestamp < window:
                chain.append(excs[j])
                j += 1
            
            if len(chain) > 1:
                chains.append({
                    "start_time": chain[0].timestamp,
                    "end_time": chain[-1].timestamp,
                    "sequence": [e.type for e in chain],
                    "severity": max(e.severity for e in chain),
                    "description": " → ".join(e.type for e in chain),
                })
            
            i = j
        
        return chains
    
    def generate_report(self) -> str:
        """生成诊断报告"""
        analysis = self.analyze()
        
        report = []
        report.append("GPU 异常诊断报告")
        report.append("=" * 60)
        
        if analysis["status"] == "no_exceptions":
            report.append("未检测到异常")
            return "\n".join(report)
        
        report.append(f"异常总数: {analysis['total']}")
        report.append(f"异常类型: {analysis['by_type']}")
        report.append(f"严重级别: {analysis['by_severity']}")
        report.append(f"时间范围: {analysis['time_range']['start']}s - "
                      f"{analysis['time_range']['end']}s")
        
        # 异常链
        if analysis["chains"]:
            report.append(f"\n异常链检测 ({len(analysis['chains'])}):")
            for chain in analysis["chains"]:
                report.append(f"  [{chain['start_time']:.3f}] {chain['description']} "
                              f"({chain['severity']})")
        
        # 严重异常
        if analysis["critical"]:
            report.append(f"\n严重异常 ({len(analysis['critical'])}):")
            for exc in analysis["critical"]:
                report.append(f"  [{exc.timestamp:.3f}] [{exc.type}] {exc.message}")
        
        # 诊断建议
        report.append(f"\n诊断建议:")
        has_ue = any(e.type == "ecc_ue" for e in self.exceptions)
        has_hang = any(e.type == "hang" for e in self.exceptions)
        has_thermal = any(e.type == "thermal" for e in self.exceptions)
        
        if has_ue:
            report.append("  ⚠ UE 错误检测到硬件故障, 建议 RMA")
        if has_hang and has_thermal:
            report.append("  ⚠ Hang 和 Thermal 同时出现, 检查散热系统")
        if has_hang and not has_thermal:
            report.append("  ⚠ Hang 但温度正常, 检查驱动/应用")
        
        return "\n".join(report)


# 命令行诊断
def diagnose_from_dmesg():
    """从 dmesg 诊断"""
    result = subprocess.run(["dmesg"], capture_output=True, text=True)
    lines = result.stdout.split("\n")
    diagnoser = GPUExceptionDiagnoser(lines)
    return diagnoser.generate_report()


if __name__ == "__main__":
    if len(sys.argv) > 1:
        with open(sys.argv[1]) as f:
            lines = f.readlines()
        diagnoser = GPUExceptionDiagnoser(lines)
        print(diagnoser.generate_report())
    else:
        print(diagnose_from_dmesg())
```

### 案例分析

#### 案例 1：Thermal → Throttling → Hang 链式故障

```bash
# 完整日志时间线
dmesg

# [ 500.000] [drm] Thermal alert: GPU temp = 92°C
# [ 500.001] [drm] Throttling: GFX clock 2500→1800MHz
# [ 505.000] [drm] Thermal alert: GPU temp = 98°C
# [ 505.001] [drm] Throttling: GFX clock 1800→1200MHz
# [ 510.000] [drm] GPU hang detected on GFX pipe
# [ 510.001] [drm] GFX has been busy but no progress
# [ 510.002] [drm] Starting GPU recovery

# 分析:
# 异常链: Thermal → Throttling → Hang
# 1. 温度上升到 92°C → 触发降频到 1800MHz
# 2. 温度继续上升到 98°C → 进一步降频到 1200MHz
# 3. 渲染帧无法在时间内完成 → 看门狗触发 hang

# 根因: 散热不足
# - 检查风扇是否正常
# - 检查散热器是否积灰
# - 检查环境温度

# 解决方案:
# 1. 清理散热器
# 2. 提高风扇转速
# 3. 降低功耗限制
```

#### 案例 2：Page Fault → Hang → Recovery

```bash
# 日志
dmesg

# [ 800.000] [drm] GPU page fault at address 0x0000000012340000
# [ 800.001] [drm] VM fault from PID 1234 (game)
# [ 800.002] [drm] Invalid page table entry
# [ 800.003] [drm] VM fault flags: WRITE | INVALID
# [ 802.000] [drm] GPU hang detected on GFX pipe
# [ 802.001] [drm] Starting GPU recovery

# 分析:
# 异常链: Page Fault → Hang
# 1. 应用写入无效 GPU 地址 → Page Fault
# 2. GPU 无法处理该错误 → 管线卡住 → Hang

# 根因: 应用 bug (野指针或 use-after-free)
# - 地址 0x12340000 不在有效映射中
# - 可能是在释放 BO 后继续写入

# 排查:
# 1. 使用 valgrind 或 address sanitizer 检查应用
# 2. 检查驱动版本是否匹配
# 3. 确认是否使用了已弃用的 API

# 解决方案:
# 1. 更新应用程序
# 2. 或在驱动层面加强参数验证
```

### 相关链接

- AMDGPU 错误处理文档: https://docs.kernel.org/gpu/amdgpu/amdgpu-ras.html
- Linux 内核级错误检测: https://docs.kernel.org/dev-tools/kcov.html
- GRBM_STATUS 寄存器定义: AMDGPU 编程指南
- PCIe AER 文档: https://docs.kernel.org/PCI/pcieaer-howto.html

### 今日小结

1. **四种异常类型**：Hang（挂起）、Page Fault（页错误）、ECC（纠错码错误）、Thermal（热异常）是 GPU 最常见的硬件异常类型。

2. **日志特征**：每种异常类型有独特的日志关键字和模式，通过 dmesg 可以快速区分。

3. **异常关联**：不同类型的异常经常形成链式关系（如 Thermal→Throttling→Hang），诊断时应找出链头（根因）。

4. **诊断工具**：GRBM_STATUS 寄存器解读 Hang 类型、Page Fault 地址分析、ECC 计数趋势监控、温度阈值检查是各类型的核心诊断方法。

5. **根因分析**：Hang → 检查驱动和应用；Page Fault → 检查应用内存管理；ECC → 检查硬件健康；Thermal → 检查散热系统。

### 扩展思考

1. **异常预测**：能否通过 CE 错误增长趋势、温度变化率、Page Fault 频率等指标预测即将发生的严重异常？如何设置合理的预警阈值？

2. **自动化异常分类**：如何使用机器学习对 GPU 异常日志进行自动分类？不同架构（RDNA3/CDNA3）的异常模式有何差异？

3. **多 GPU 系统的异常关联**：在 MI250 等 多 GPU 系统中，一个 GPU 的 hang 会影响其他 GPU 吗？XGMI 互连中的异常如何传播？