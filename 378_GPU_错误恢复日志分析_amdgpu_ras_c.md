# GPU 错误恢复日志分析：amdgpu_ras.c

## 学习目标

- 理解 amdgpu_ras.c 的核心实现和错误处理流程
- 掌握 GPU 错误恢复的代码路径和关键函数
- 学会分析 GPU 恢复日志并进行故障诊断
- 能够根据错误恢复日志确定根因和修复方向

## 知识详解

### 概念原理

#### amdgpu_ras.c 核心架构

```ascii
amdgpu_ras.c 架构:

┌──────────────────────────────────────────────────────────────┐
│  amdgpu_ras.c - RAS 核心管理层                                 │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 初始化/去初始化                                          │  │
│  │                                                         │  │
│  │  amdgpu_ras_init()           - RAS 子系统初始化          │  │
│  │  amdgpu_ras_fini()           - RAS 子系统去初始化         │  │
│  │  amdgpu_ras_recovery_init()  - 错误恢复初始化             │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 错误检测和上报                                           │  │
│  │                                                         │  │
│  │  amdgpu_ras_query_error()    - 查询错误状态              │  │
│  │  amdgpu_ras_log_error()      - 记录错误到日志             │  │
│  │  amdgpu_ras_save_bad_pages() - 保存坏页面信息            │  │
│  │  amdgpu_ras_reset_gpu()      - 触发 GPU 重置             │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 错误恢复                                                │  │
│  │                                                         │  │
│  │  amdgpu_ras_do_recovery()    - 执行恢复流程              │  │
│  │  amdgpu_ras_recovery_fini()  - 恢复完成清理              │  │
│  │  amdgpu_ras_handle_ras_err() - 处理 RAS 错误            │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ sysfs 接口                                              │  │
│  │                                                         │  │
│  │  amdgpu_ras_sysfs_init()     - 创建 sysfs 文件           │  │
│  │  amdgpu_ras_sysfs_fini()     - 移除 sysfs 文件           │  │
│  │  ras_ecc_cnt_show()          - 显示 ECC 计数             │  │
│  │  ras_inject_store()          - 错误注入接口              │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ EEPROM 管理                                             │  │
│  │                                                         │  │
│  │  amdgpu_ras_eeprom_init()    - EEPROM 初始化             │  │
│  │  amdgpu_ras_eeprom_write()   - 写入错误记录              │  │
│  │  amdgpu_ras_eeprom_read()    - 读取错误记录              │  │
│  │  amdgpu_ras_eeprom_table()   - 管理 EEPROM 表            │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

#### 错误恢复流程

```ascii
GPU 错误恢复流程 (amdgpu_ras_do_recovery):

┌──────────────────────────────────────────────────────────────┐
│  1. 错误检测阶段                                               │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  a. 硬件触发 RAS 中断                                    │  │
│  │  b. amdgpu_ras_interrupt_handler() 读取错误寄存器        │  │
│  │  c. 识别错误源 (UMC/SDMA/GFX/...)                       │  │
│  │  d. 清除中断标志                                         │  │
│  │  e. 保存错误上下文                                       │  │
│  └────────────────────────────────────────────────────────┘  │
│       │                                                       │
│       ▼                                                       │
│  2. 错误分类阶段                                               │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  a. CE (Correctable): 计数更新, 继续运行                 │  │
│  │  b. UE (Uncorrectable): 进入恢复流程                     │  │
│  │  c. Fatal: 立即触发 GPU reset                           │  │
│  └────────────────────────────────────────────────────────┘  │
│       │                                                       │
│       ▼                                                       │
│  3. 恢复决策阶段                                               │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  amdgpu_ras_do_recovery():                              │  │
│  │                                                         │  │
│  │  a. 检查是否可以恢复                                      │  │
│  │  b. 检查恢复队列是否已满                                   │  │
│  │  c. 确定恢复策略 (soft reset / hard reset)               │  │
│  │  d. 将恢复请求加入工作队列                                 │  │
│  └────────────────────────────────────────────────────────┘  │
│       │                                                       │
│       ▼                                                       │
│  4. GPU 重置阶段                                               │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  amdgpu_device_gpu_recover():                           │  │
│  │                                                         │  │
│  │  a. 暂停所有 GPU 调度器                                   │  │
│  │  b. 等待所有 in-flight 操作完成                           │  │
│  │  c. 保存关键寄存器状态                                    │  │
│  │  d. 执行 GPU soft reset                                  │  │
│  │  e. 如果 soft reset 失败, 尝试 hard reset                 │  │
│  │  f. 恢复寄存器状态                                        │  │
│  │  g. 恢复调度器                                            │  │
│  └────────────────────────────────────────────────────────┘  │
│       │                                                       │
│       ▼                                                       │
│  5. 恢复后处理阶段                                             │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  a. 验证重置是否成功                                     │  │
│  │  b. 记录恢复日志                                         │  │
│  │  c. 恢复失败计数统计                                      │  │
│  │  d. 如果连续失败, 升级恢复策略                             │  │
│  │  e. 通知用户空间 (uevent)                                │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

#### 关键数据结构

```c
// RAS 错误源描述
struct amdgpu_ras_err_handler_data {
    struct amdgpu_ras_err_node *nodes;  // 错误节点数组
    int count;                           // 错误节点数
    int max_count;                       // 最大节点数
};

// RAS 错误节点
struct amdgpu_ras_err_node {
    struct list_head node;               // 链表节点
    struct amdgpu_ras_err_info err_info; // 错误信息
};

// RAS 错误信息
struct amdgpu_ras_err_info {
    uint64_t address;                    // 错误地址
    uint32_t timestamp;                  // 时间戳
    uint32_t error_type;                 // 错误类型
    uint32_t ip_blk;                     // 所属 IP 块
    uint32_t channel_id;                 // 通道 ID
    uint32_t syndrome;                   // ECC 校正子
};

// RAS 管理器
struct amdgpu_ras_manager {
    struct amdgpu_device *adev;          // GPU 设备
    struct mutex recovery_lock;          // 恢复锁
    struct work_struct recovery_work;    // 恢复工作项
    atomic_t recovery_count;             // 恢复计数
    unsigned long features;              // RAS 特性位图
    struct ras_err_handler_data *eh_data;// 错误处理数据
};
```

### 实践操作

#### 分析 GPU 恢复日志

```bash
# 1. 查看完整的 GPU 恢复日志
dmesg | grep -E "ras|RAS|recover|reset|hang"

# 典型恢复日志序列 (按时间顺序):
#
# [ 100.000] [drm] RAS: UMC UE error at 0x00100000
# [ 100.001] [drm] RAS: Uncorrectable error detected
# [ 100.002] [drm] amdgpu_ras_do_recovery: starting recovery
# [ 100.003] [drm] amdgpu_device_gpu_recover: begin
# [ 100.004] [drm] Freezing GPU scheduler
# [ 100.050] [drm] Waiting for GPU idle...
# [ 100.200] [drm] GPU idle confirmed
# [ 100.201] [drm] Saving GPU register state
# [ 100.500] [drm] Executing GPU soft reset...
# [ 100.800] [drm] GPU soft reset succeeded
# [ 100.801] [drm] Restoring GPU register state
# [ 101.000] [drm] Resuming GPU scheduler
# [ 101.001] [drm] amdgpu_device_gpu_recover: succeeded
# [ 101.002] [drm] RAS recovery completed

# 2. 恢复失败日志
#
# [ 200.000] [drm] RAS: Fatal error detected
# [ 200.001] [drm] amdgpu_device_gpu_recover: begin
# [ 200.002] [drm] Freezing GPU scheduler
# [ 200.050] [drm] GPU idle timeout! (10s elapsed)
# [ 210.000] [drm] Force stopping GPU scheduler
# [ 210.001] [drm] Executing GPU soft reset...
# [ 215.000] [drm] GPU soft reset TIMEOUT!
# [ 215.001] [drm] Attempting hard reset...
# [ 215.500] [drm] GPU hard reset initiated
# [ 216.000] [drm] GPU hard reset succeeded
# [ 216.001] [drm] amdgpu_device_gpu_recover: succeeded (hard reset)
```

#### 恢复日志分析工具

```python
#!/usr/bin/env python3
# gpu_recovery_log_analyzer.py - GPU 恢复日志分析器

import re
import sys
from typing import List, Dict, Optional
from datetime import datetime, timedelta
from collections import Counter


class RecoveryLogEntry:
    """恢复日志条目"""
    
    def __init__(self, timestamp: float, level: str, message: str):
        self.timestamp = timestamp
        self.level = level
        self.message = message


class RecoveryEvent:
    """恢复事件"""
    
    def __init__(self, start_time: float):
        self.start_time = start_time
        self.end_time: Optional[float] = None
        self.entries: List[RecoveryLogEntry] = []
        self.trigger_type: str = "unknown"
        self.outcome: str = "unknown"
        self.reset_type: str = "unknown"
        self.error_info: Dict = {}
    
    @property
    def duration(self) -> Optional[float]:
        if self.start_time and self.end_time:
            return self.end_time - self.start_time
        return None


class RecoveryLogAnalyzer:
    """GPU 恢复日志分析器"""
    
    PATTERNS = {
        "recovery_start": re.compile(
            r'amdgpu_device_gpu_recover:\s*begin'
        ),
        "recovery_end": re.compile(
            r'amdgpu_device_gpu_recover:\s*(succeeded|failed)'
        ),
        "ras_error": re.compile(
            r'RAS:\s*(\w+)\s*error\s*at\s*(0x[0-9a-fA-F]+)'
        ),
        "ras_type": re.compile(
            r'RAS:\s*(Correctable|Uncorrectable|Fatal)\s*error'
        ),
        "soft_reset": re.compile(
            r'GPU soft reset\s*(succeeded|TIMEOUT|failed)'
        ),
        "hard_reset": re.compile(
            r'GPU hard reset\s*(succeeded|failed|initiated)'
        ),
        "scheduler": re.compile(
            r'GPU scheduler\s*(Freezing|Resuming|timeout)'
        ),
        "gpu_idle": re.compile(
            r'GPU idle\s*(confirmed|timeout)'
        ),
        "hang": re.compile(
            r'GPU hang\s*detected'
        ),
    }
    
    def __init__(self, log_lines: List[str]):
        self.lines = log_lines
        self.events: List[RecoveryEvent] = []
        self._parse()
    
    def _parse(self):
        """解析日志"""
        current_event: Optional[RecoveryEvent] = None
        
        for line in self.lines:
            match = re.match(
                r'\[(\s*[\d.]+)\]\s*\[?(drm)\]?\s*(.*)',
                line
            )
            if not match:
                continue
            
            try:
                ts = float(match.group(1).strip())
            except ValueError:
                continue
            
            msg = match.group(3).strip()
            level = self._detect_level(msg)
            entry = RecoveryLogEntry(ts, level, msg)
            
            # 检测恢复开始
            if self.PATTERNS["recovery_start"].search(msg):
                if current_event:
                    current_event.end_time = ts
                    self.events.append(current_event)
                current_event = RecoveryEvent(ts)
            
            if current_event:
                current_event.entries.append(entry)
                
                # 检测触发类型
                ras_error = self.PATTERNS["ras_error"].search(msg)
                if ras_error:
                    current_event.trigger_type = "ras"
                    current_event.error_info["error_source"] = ras_error.group(1)
                    current_event.error_info["error_address"] = ras_error.group(2)
                
                ras_type = self.PATTERNS["ras_type"].search(msg)
                if ras_type:
                    current_event.error_info["error_severity"] = ras_type.group(1)
                
                # 检测重置类型
                soft = self.PATTERNS["soft_reset"].search(msg)
                if soft:
                    current_event.reset_type = f"soft ({soft.group(1)})"
                
                hard = self.PATTERNS["hard_reset"].search(msg)
                if hard:
                    if "initiated" in msg:
                        current_event.reset_type = "hard"
                    else:
                        current_event.reset_type = f"hard ({hard.group(1)})"
                
                # 检测结果
                end = self.PATTERNS["recovery_end"].search(msg)
                if end:
                    current_event.outcome = end.group(1)
                    current_event.end_time = ts
            
            # 检测没有开始标记的恢复
            if not current_event and self.PATTERNS["hang"].search(msg):
                current_event = RecoveryEvent(ts)
                current_event.trigger_type = "hang"
        
        # 保存最后一个事件
        if current_event:
            self.events.append(current_event)
    
    def _detect_level(self, msg: str) -> str:
        if "ERROR" in msg or "failed" in msg or "Fatal" in msg:
            return "ERROR"
        if "timeout" in msg.lower():
            return "WARN"
        return "INFO"
    
    def analyze(self) -> Dict:
        """分析恢复事件"""
        if not self.events:
            return {"events": [], "summary": {"total": 0}}
        
        durations = [e.duration for e in self.events if e.duration]
        outcomes = Counter(e.outcome for e in self.events)
        triggers = Counter(e.trigger_type for e in self.events)
        reset_types = Counter(e.reset_type for e in self.events)
        
        succeeded = sum(1 for e in self.events if e.outcome == "succeeded")
        failed = sum(1 for e in self.events if e.outcome == "failed")
        
        analysis = {
            "events": [],
            "summary": {
                "total": len(self.events),
                "succeeded": succeeded,
                "failed": failed,
                "success_rate": (succeeded / len(self.events) * 100
                                if self.events else 0),
                "avg_duration": (sum(durations) / len(durations)
                                if durations else 0),
                "max_duration": max(durations) if durations else 0,
                "min_duration": min(durations) if durations else 0,
                "trigger_types": dict(triggers),
                "outcomes": dict(outcomes),
                "reset_types": dict(reset_types),
            }
        }
        
        # 每个事件的详细信息
        for i, event in enumerate(self.events):
            event_info = {
                "index": i + 1,
                "start_time": event.start_time,
                "end_time": event.end_time,
                "duration": event.duration,
                "trigger_type": event.trigger_type,
                "outcome": event.outcome,
                "reset_type": event.reset_type,
                "error_info": event.error_info,
                "log_count": len(event.entries),
                "errors": [e.message for e in event.entries
                          if e.level == "ERROR"],
                "warnings": [e.message for e in event.entries
                            if e.level == "WARN"],
            }
            analysis["events"].append(event_info)
        
        return analysis
    
    def generate_report(self) -> str:
        """生成报告"""
        analysis = self.analyze()
        summary = analysis["summary"]
        
        report = []
        report.append("GPU 恢复日志分析报告")
        report.append("=" * 60)
        report.append(f"恢复事件总数: {summary['total']}")
        report.append(f"成功: {summary['succeeded']}")
        report.append(f"失败: {summary['failed']}")
        report.append(f"成功率: {summary['success_rate']:.1f}%")
        
        if summary.get("avg_duration"):
            report.append(f"\n恢复耗时:")
            report.append(f"  平均: {summary['avg_duration']:.3f}s")
            report.append(f"  最长: {summary['max_duration']:.3f}s")
            report.append(f"  最短: {summary['min_duration']:.3f}s")
        
        report.append(f"\n触发类型分布:")
        for t, c in summary.get("trigger_types", {}).items():
            report.append(f"  {t}: {c} 次")
        
        report.append(f"\n重置类型:")
        for t, c in summary.get("reset_types", {}).items():
            report.append(f"  {t}: {c} 次")
        
        report.append(f"\n详细事件列表:")
        for event in analysis["events"]:
            report.append(f"\n  事件 #{event['index']}:")
            report.append(f"    触发: {event['trigger_type']}")
            report.append(f"    结果: {event['outcome']}")
            report.append(f"    耗时: {event['duration']:.3f}s" if event['duration'] else "    耗时: N/A")
            report.append(f"    重置类型: {event['reset_type']}")
            if event['error_info']:
                report.append(f"    错误信息: {event['error_info']}")
            if event['errors']:
                report.append(f"    错误日志:")
                for err in event['errors'][:3]:
                    report.append(f"      {err}")
            if event['warnings']:
                report.append(f"    警告:")
                for w in event['warnings'][:3]:
                    report.append(f"      {w}")
        
        # 建议
        report.append(f"\n分析建议:")
        if summary['failed'] > 0:
            report.append(f"  ⚠ 存在失败的恢复事件")
        if summary.get('total', 0) > 10:
            report.append(f"  ⚠ GPU 频繁恢复 ({summary['total']} 次)")
        if any(e.get("reset_type", "").startswith("hard") for e in analysis["events"]):
            report.append(f"  ⚠ 使用了 hard reset (可能指示严重问题)")
        if summary.get('success_rate', 100) < 80:
            report.append(f"  ⚠ 恢复成功率较低")
        
        return "\n".join(report)


# 主程序
if __name__ == "__main__":
    if len(sys.argv) > 1:
        with open(sys.argv[1]) as f:
            lines = f.readlines()
    else:
        import subprocess
        result = subprocess.run(["dmesg"], capture_output=True, text=True)
        lines = result.stdout.split("\n")
    
    analyzer = RecoveryLogAnalyzer(lines)
    print(analyzer.generate_report())
```

#### 恢复计数器监控

```bash
#!/bin/bash
# gpu_recovery_monitor.sh - GPU 恢复监控

echo "GPU 恢复监控"
echo "============="
echo ""

# 1. 查看恢复计数
if [ -f /sys/class/drm/card0/device/ras_recovery_cnt ]; then
    echo "恢复次数: $(cat /sys/class/drm/card0/device/ras_recovery_cnt)"
else
    echo "恢复计数器不可用"
fi

# 2. 查看恢复历史
dmesg | grep -c "amdgpu_device_gpu_recover"
echo ""

# 3. 最近恢复时间
echo "最近恢复事件:"
dmesg | grep "amdgpu_device_gpu_recover" | tail -5

echo ""

# 4. 检查恢复失败
echo "恢复失败:"
FAILED=$(dmesg | grep -c "amdgpu_device_gpu_recover.*failed")
echo "  失败次数: $FAILED"

# 5. 计算成功率
TOTAL=$(dmesg | grep -c "amdgpu_device_gpu_recover")
if [ "$TOTAL" -gt 0 ]; then
    SUCCESS=$((TOTAL - FAILED))
    RATE=$(echo "scale=2; $SUCCESS * 100 / $TOTAL" | bc)
    echo "  总恢复: $TOTAL"
    echo "  成功率: $RATE%"
fi
```

### 案例分析

#### 案例 1：UE 错误触发的 GPU 恢复

```bash
# 完整恢复日志
dmesg

# [ 1000.000] [drm] RAS: UMC UE error at 0x00200000
# [ 1000.001] [drm] RAS: Uncorrectable error detected
# [ 1000.002] [drm] amdgpu_ras_do_recovery: start
# [ 1000.003] [drm] amdgpu_device_gpu_recover: begin
# [ 1000.004] [drm] Freezing GPU scheduler
# [ 1000.005] [drm] Waiting for GPU idle...
# [ 1000.200] [drm] GPU idle confirmed
# [ 1000.201] [drm] Saving GPU register state
# [ 1000.500] [drm] Executing GPU soft reset...
# [ 1000.800] [drm] GPU soft reset succeeded
# [ 1000.801] [drm] Restoring GPU register state
# [ 1001.000] [drm] Resuming GPU scheduler
# [ 1001.001] [drm] amdgpu_device_gpu_recover: succeeded
# [ 1001.002] [drm] RAS recovery completed (1.002s)

# 分析:
# - 触发: UMC UE 错误 (不可纠正)
# - 恢复类型: soft reset
# - 总耗时: 1.002 秒
# - 恢复成功

# 解读:
# 1. UE 错误导致数据损坏
# 2. 驱动自动触发 GPU 恢复
# 3. soft reset 成功
# 4. 应用可能丢失部分渲染帧

# 后续行动:
# 1. 记录错误地址到 RMA 报告
# 2. 检查 CE 错误计数 (是否在此之前有增长)
```

#### 案例 2：级联恢复失败

```bash
# 日志: 连续恢复失败升级

# [ 2000.000] [drm] GPU hang detected on GFX pipe
# [ 2000.001] [drm] GRBM_STATUS=0xA0000002
# [ 2000.002] [drm] amdgpu_device_gpu_recover: begin
# [ 2000.003] [drm] Freezing GPU scheduler
# [ 2000.050] [drm] GPU idle timeout! (10s wait)
# [ 2000.051] [drm] Force stopping GPU scheduler
# [ 2000.052] [drm] Executing soft reset...
# [ 2005.000] [drm] Soft reset TIMEOUT! (5s wait)
# [ 2005.001] [drm] Attempting hard reset...
# [ 2005.500] [drm] Hard reset initiated
# [ 2006.000] [drm] Hard reset succeeded
# [ 2006.001] [drm] Restoring GPU state
# [ 2007.000] [drm] Resuming scheduler
# [ 2007.001] [drm] amdgpu_device_gpu_recover: succeeded

# 分析:
# - GPU 挂起 (hang) 触发恢复
# - 软重置失败 (GPU 无法 idle)
# - 升级到硬重置
# - 总耗时: ~7 秒

# 关键日志行:
# "GPU idle timeout" → GPU 完全锁死
# "Soft reset TIMEOUT" → 软重置不起作用
# "Attempting hard reset" → 升级到硬重置

# 根因分析:
# 1. GPU 硬锁死 (比软锁死更严重)
# 2. 可能原因: 电源崩溃、热关断、PCIe 链路断开
# 3. 需要检查硬件状态
```

### 相关链接

- AMDGPU RAS 代码: drivers/gpu/drm/amd/amdgpu/amdgpu_ras.c
- AMDGPU 恢复机制: https://docs.kernel.org/gpu/amdgpu/amdgpu-recovery.html
- Linux 内核错误恢复模式: https://docs.kernel.org/gpu/amdgpu/amdgpu-ras.html

### 今日小结

1. **核心代码**：amdgpu_ras.c 实现 RAS 子���统的核心管理功能，包括错误检测、日志记录、恢复触发和 sysfs 接口。

2. **恢复流程**：四个阶段（错误检测→错误分类→恢复决策→GPU 重置）执行 GPU 错误恢复，支持 soft reset 和 hard reset 两种级别。

3. **日志分析**：通过监控恢复日志的关键时间点（触发类型、恢复耗时、重置类型、结果）可以判断 GPU 健康状况。

4. **恢复升级**：当 soft reset 失败时自动升级到 hard reset，连续失败则需要人工干预。

5. **监控指标**：恢复成功率、平均恢复耗时、触发类型分布、重置类型分布是评估 GPU 稳定性的核心指标。

### 扩展思考

1. **恢复策略优化**：不同的错误类型应该使用不同的恢复策略。例如，UE 错误后是否需要完全重置 GPU 还是可以只重置内存控制器？如何设计最优的恢复策略树？

2. **恢复期间的用户体验**：GPU 恢复期间（1-7 秒）应用会卡死。如何设计恢复流程以最小化对用户的影响？支持恢复期间保持显示输出？

3. **机器学习预测恢复**：能否通过学习历史恢复日志，预测哪些错误最终会导致恢复失败？在错误发生前预防性地迁移 GPU 工作负载？