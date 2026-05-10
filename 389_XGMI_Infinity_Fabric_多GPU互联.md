# Day 389：XGMI / Infinity Fabric 多 GPU 互联

## 学习目标

- 理解 XGMI (eXtensible GPU Memory Interconnect) 的架构和协议
- 掌握 AMD Infinity Fabric 在多 GPU 系统中的角色
- 学会配置和管理 XGMI 拓扑
- 理解 XGMI 对驱动开发和测试的影响
- 掌握 XGMI 性能测试和故障排除方法

## 知识详解

### 概念原理

#### XGMI 架构概览

XGMI 是 AMD 专有的高带宽 GPU 互联技术，用于构建多 GPU 系统的统一内存域：

```
XGMI 物理架构:

┌─────────────────────────┐    XGMI 链路    ┌─────────────────────────┐
│       GPU 0 (MI250)     │◄═══════════════►│       GPU 1 (MI250)     │
│                         │                 │                         │
│  ┌───────────────────┐  │  每链路带宽:     │  ┌───────────────────┐  │
│  │   Shader Engine   │  │  50 GB/s (Gen4) │  │   Shader Engine   │  │
│  │   ┌───┐ ┌───┐    │  │                  │  │   ┌───┐ ┌───┐    │  │
│  │   │SE0│ │SE1│    │  │  4 条链路:       │  │   │SE0│ │SE1│    │  │
│  │   └───┘ └───┘    │  │  200 GB/s 总带宽 │  │   └───┘ └───┘    │  │
│  └───────────────────┘  │                  │  └───────────────────┘  │
│         │               │  延迟: ~500ns    │         │               │
│  ┌───────┴───────┐      │  访问远端显存    │  ┌───────┴───────┐      │
│  │  HBM2e 64GB   │      │  (~1500ns 总)    │  │  HBM2e 64GB   │      │
│  └───────────────┘      └──────────────────┘  └───────────────┘      │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                 XGMI 控制器 (硬件层)                            │  │
│  │                                                                │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │  │
│  │  │ 链路 0   │  │ 链路 1   │  │ 链路 2   │  │ 链路 3   │       │  │
│  │  │ 50 GB/s  │  │ 50 GB/s  │  │ 50 GB/s  │  │ 50 GB/s  │       │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │  │
│  │       │             │             │             │             │  │
│  │  ┌────┴─────────────┴─────────────┴─────────────┴────┐        │  │
│  │  │              XGMI 路由引擎                         │        │  │
│  │  │  - 地址译码: 远端显存 vs 本地显存                    │        │  │
│  │  │  - 一致性协议: 缓存行状态跟踪                        │        │  │
│  │  │  - 原子操作: fetch-and-add, compare-and-swap       │        │  │
│  │  └────────────────────────────────────────────────────┘        │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

**XGMI 的关键特性：**

| 特性 | 值 | 说明 |
|------|-----|------|
| 每链路带宽 (Gen4) | 50 GB/s | 单向，双向 100 GB/s |
| 链路数量 | 2-8 | 取决于 GPU 型号和配置 |
| 总带宽 (MI250) | 200 GB/s | 4 条链路双向 |
| 延迟 | ~500ns | 链路传输延迟 |
| 物理层 | 16 lane SerDes | 每条链路 16 通道 |
| 编码 | 128b/130b | 高效编码 |
| 电源 | ~5W/链路 | 低功耗设计 |

#### Infinity Fabric 架构

Infinity Fabric 是 AMD 的统一互联框架，同时用于 CPU-CPU (EPYC) 和 GPU-GPU 连接：

```
Infinity Fabric 层次结构:

┌─────────────────────────────────────────────────────────┐
│                    Infinity Architecture                  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  协议层 (Protocol Layer)                                 │
│  ┌─────────────────────────────────────────────────────┐│
│  │  - 一致性协议 (Coherent Protocol)                   ││
│  │  - 非一致性协议 (Non-Coherent Protocol)             ││
│  │  - 原子操作 (Atomic Operations)                     ││
│  │  - 消息传递 (Message Passing)                       ││
│  └────────────────────┬────────────────────────────────┘│
│                       │                                  │
│  链路层 (Link Layer)   │                                  │
│  ┌────────────────────┴────────────────────────────────┐│
│  │  - 流量控制 (Credit-Based Flow Control)             ││
│  │  - 错误检测 (CRC32)                                 ││
│  │  - 重传机制 (Retry on Error)                        ││
│  │  - 通道捆绑 (Lane Bundling)                         ││
│  └────────────────────┬────────────────────────────────┘│
│                       │                                  │
│  物理层 (PHY Layer)    │                                  │
│  ┌────────────────────┴────────────────────────────────┐│
│  │  - SerDes (Serializer/Deserializer)                 ││
│  │  - 128b/130b 编码                                   ││
│  │  - 时钟恢复 (CDR)                                   ││
│  │  - 链路训练 (Link Training)                         ││
│  │  - 功耗管理 (L0s, L1, L2)                          ││
│  └────────────────────────────────────────────────────┘│
│                                                         │
│  物理介质: PCB 走线 / 背板 / 线缆                      │
└─────────────────────────────────────────────────────────┘
```

#### XGMI 在 AMDGPU 驱动中的实现

```c
// 内核源码: drivers/gpu/drm/amd/amdgpu/amdgpu_xgmi.c

// XGMI 蜂巢（Hive）结构 - 一组通过 XGMI 互联的 GPU
struct amdgpu_hive_info {
    struct kref                 kref;
    uint64_t                    hive_id;        // 蜂巢 ID
    struct list_head            device_list;    // 蜂巢内的设备列表
    int                         number_devices; // 设备数量
    struct mutex                hive_lock;
    // 拓扑信息
    struct amdgpu_xgmi_ras      *xgmi_ras;      // RAS 信息
    atomic_t                    ras_recovery;   // RAS 恢复状态
    struct delayed_work         recovery_work;
};

// XGMI 节点信息 (每个 GPU 的 XGMI 配置)
struct amdgpu_xgmi {
    struct amdgpu_device        *adev;
    uint64_t                    node_id;        // XGMI 节点 ID
    uint8_t                     num_nodes;      // 蜂巢内节点数
    uint8_t                     physical_node_id; // 物理节点 ID (硬编码)
    struct amdgpu_xgmi_ras      *ras_ctx;       // RAS 上下文
};

// XGMI P2P 映射 - 让 GPU 直接访问其他 GPU 的显存
struct amdgpu_xgmi_peer {
    struct amdgpu_device        *adev;
    struct amdgpu_device        *peer_adev;
    uint64_t                    mapped_size;    // 映射大小
    struct list_head            entry;          // 链表
    bool                        enabled;        // 是否启用
};

// XGMI 初始化
int amdgpu_xgmi_init(struct amdgpu_device *adev)
{
    struct amdgpu_xgmi *xgmi = &adev->xgmi;
    int ret;
    
    // 1. 分配 XGMI 节点 ID (从 VBIOS 或寄存器读取)
    xgmi->node_id = amdgpu_xgmi_get_node_id(adev);
    
    // 2. 获取 XGMI 物理链路状态
    ret = amdgpu_xgmi_get_topology_info(adev);
    if (ret)
        return ret;
    
    // 3. 加入或创建 XGMI 蜂巢
    ret = amdgpu_xgmi_add_device(adev);
    if (ret)
        return ret;
    
    // 4. 配置 P2P 映射
    ret = amdgpu_xgmi_peer_init(adev);
    
    return ret;
}

// XGMI P2P 开启 - 允许 GPU 直接访问对端 GPU 显存
int amdgpu_xgmi_set_p2p_lod(struct amdgpu_device *adev,
                              struct amdgpu_device *peer_adev,
                              bool enable)
{
    struct amdgpu_xgmi_peer *peer;
    
    // 1. 检查两个设备是否在同一蜂巢
    if (!amdgpu_xgmi_same_hive(adev, peer_adev))
        return -EINVAL;
    
    // 2. 分配 P2P 结构
    peer = kzalloc(sizeof(*peer), GFP_KERNEL);
    if (!peer)
        return -ENOMEM;
    
    peer->adev = adev;
    peer->peer_adev = peer_adev;
    peer->enabled = enable;
    
    // 3. 配置 PCIe BAR 映射到对端显存
    // 在 BAR 中保留一段地址空间映射远端显存
    if (enable) {
        // 获取对端显存物理地址
        uint64_t peer_vram_base = peer_adev->vm_manager.vram_base_offset;
        uint64_t peer_vram_size = peer_adev->vm_manager.vram_size;
        
        // 在本端 GPU 的 GART 中创建映射
        ret = amdgpu_gart_bind_peer(adev, peer_adev,
                                     peer_vram_base, peer_vram_size);
        if (ret) {
            kfree(peer);
            return ret;
        }
    }
    
    list_add_tail(&peer->entry, &adev->xgmi_peers);
    
    return 0;
}

// XGMI 地址翻译 - 远端显存访问
uint64_t amdgpu_xgmi_translate_peer_addr(struct amdgpu_device *adev,
                                          uint64_t peer_node_id,
                                          uint64_t addr)
{
    // XGMI 地址格式:
    // [63]     = 本地/远端标志 (0=本地, 1=远端)
    // [62:56]  = 目标节点 ID
    // [55:0]   = 地址偏移
    
    uint64_t xgmi_addr = addr;
    
    // 标记为远端访问
    xgmi_addr |= (1ULL << 63);
    
    // 编码目标节点 ID
    xgmi_addr |= (peer_node_id & 0x7F) << 56;
    
    return xgmi_addr;
}
```

#### 内存一致性模型

XGMI 提供硬件一致性，使多个 GPU 的显存表现为统一的地址空间：

```
XGMI 一致性模型:

┌─────────────────────────────────────────────────────────┐
│              统一内存地址空间 (Unified Memory)            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  本地 GPU 0 显存        远端 GPU 1 显存                  │
│  ┌──────────────────┐   ┌──────────────────┐           │
│  │ 0x0000 - 0xFFFF  │   │ 0x10000 - 0x1FFFF │           │
│  │ 数据 A (最新)    │   │ 数据 B           │           │
│  │ 数据 C (共享)    │   │ 数据 D (共享)    │           │
│  └──────┬───────────┘   └──────┬───────────┘           │
│         │                      │                        │
│         └──────────┬───────────┘                        │
│                    ▼                                     │
│  ┌────────────────────────────────────────────────────┐ │
│  │           XGMI 一致性引擎                          │ │
│  │                                                    │ │
│  │  操作: GPU 0 写入数据 C                            │ │
│  │  1. GPU 0 本地缓存更新                             │ │
│  │  2. XGMI 发送 invalidate 到 GPU 1                 │ │
│  │  3. GPU 1 缓存行标记为无效                         │ │
│  │  4. GPU 1 下次读取 → cache miss → XGMI 读取最新值  │ │
│  └────────────────────────────────────────────────────┘ │
│                                                         │
│  一致性协议: MOESI (Modified/Owned/Exclusive/Shared/Invalid)
│  M: 修改 (脏数据)                                      │
│  O: 拥有 (脏，共享)                                    │
│  E: 独占 (干净，未共享)                                │
│  S: 共享 (干净，共享)                                  │
│  I: 无效                                               │
└─────────────────────────────────────────────────────────┘
```

#### XGMI 拓扑发现

```
XGMI 拓扑发现流程:

                   GPU 初始化
                      │
                      ▼
             读取 XGMI 寄存器
              (获取 node_id)
                      │
                      ▼
    ┌─────────────────────────────────────┐
    │ 广播: "XGMI_HELLO"                  │
    │ 在 XGMI 链路上发送探测包            │
    │ 格式: {type=HELLO, src_node_id}     │
    └────────────┬────────────────────────┘
                 │
          ┌──────┴──────┐
          ▼              ▼
    ┌──────────┐   ┌──────────┐
    │ GPU 1   │   │ GPU 2   │  ← 收到 HELLO
    │ 响应 ACK│   │ 响应 ACK│
    │ + 自身ID│   │ + 自身ID│
    └────┬─────┘   └────┬─────┘
         │              │
         └──────┬───────┘
                ▼
    ┌──────────────────────┐
    │ 构建 XGMI 拓扑表     │
    │                      │
    │ Node 0: GPU 0 (自身) │
    │ Node 1: GPU 1 (链路)│
    │ Node 2: GPU 2 (链路)│
    │ 链路带宽: 168 GB/s  │
    └──────────────────────┘
```

### 实践操作

#### XGMI 信息查询

```bash
#!/bin/bash
# 文件名: xgmi_info.sh
# 用途: 查询 XGMI 状态和拓扑

echo "=============================================="
echo "  XGMI / Infinity Fabric 信息查询"
echo "=============================================="
echo ""

check_xgmi_support() {
    echo "[1/6] XGMI 支持检测"
    echo "-------------------"
    
    if lsmod | grep -q "amdgpu"; then
        echo "  ✅ amdgpu 已加载"
    else
        echo "  ❌ amdgpu 未加载"
        return 1
    fi
    
    # 检查 XGMI debugfs 接口
    if [ -d /sys/kernel/debug/dri ]; then
        for dir in /sys/kernel/debug/dri/*; do
            if [ -f "${dir}/amdgpu_xgmi_map" ]; then
                echo "  ✅ XGMI debugfs 接口可用"
                return 0
            fi
        done
    fi
    
    echo "  ⚠️ XGMI 接口不可用 (可能不支持 XGMI)"
    return 1
}

show_xgmi_topology() {
    echo ""
    echo "[2/6] XGMI 拓扑"
    echo "-------------"
    
    # 方法 1: 通过 debugfs
    if [ -d /sys/kernel/debug/dri ]; then
        for dir in /sys/kernel/debug/dri/*; do
            if [ -f "${dir}/amdgpu_xgmi_map" ]; then
                echo "  XGMI 映射:"
                cat "${dir}/amdgpu_xgmi_map" | while read line; do
                    echo "    ${line}"
                done
            fi
            
            if [ -f "${dir}/amdgpu_xgmi_info" ]; then
                echo ""
                echo "  XGMI 详细信息:"
                cat "${dir}/amdgpu_xgmi_info" | while read line; do
                    echo "    ${line}"
                done
            fi
        done
    fi
    
    # 方法 2: 通过 sysfs
    for gpu_dir in /sys/class/drm/card?; do
        if [ -f "${gpu_dir}/device/xgmi_node_id" ]; then
            echo ""
            echo "  GPU ${gpu_dir##*/}:"
            echo "    Node ID:     $(cat ${gpu_dir}/device/xgmi_node_id 2>/dev/null || echo 'N/A')"
            echo "    Hive ID:     $(cat ${gpu_dir}/device/xgmi_hive_id 2>/dev/null || echo 'N/A')"
            echo "    Device Num:  $(cat ${gpu_dir}/device/xgmi_device_num 2>/dev/null || echo 'N/A')"
            echo "    P2P 状态:    $(cat ${gpu_dir}/device/xgmi_p2p_state 2>/dev/null || echo 'N/A')"
        fi
    done
}

show_xgmi_peers() {
    echo ""
    echo "[3/6] XGMI Peer 信息"
    echo "-------------------"
    
    for gpu_dir in /sys/class/drm/card?; do
        if [ -d "${gpu_dir}/device/xgmi_peers" ]; then
            echo "  GPU ${gpu_dir##*/} Peers:"
            for peer in ${gpu_dir}/device/xgmi_peers/*; do
                if [ -d "${peer}" ]; then
                    PEER_ID=$(basename ${peer})
                    echo "    远端节点 ${PEER_ID}:"
                    
                    if [ -f "${peer}/bandwidth" ]; then
                        BW=$(cat ${peer}/bandwidth)
                        echo "      带宽: ${BW} MB/s"
                    fi
                    if [ -f "${peer}/status" ]; then
                        STATUS=$(cat ${peer}/status)
                        echo "      状态: ${STATUS}"
                    fi
                fi
            done
        fi
    done
}

show_xgmi_bandwidth() {
    echo ""
    echo "[4/6] XGMI 带宽测试"
    echo "------------------"
    
    # 通过 ROCm 工具测试
    if command -v rocm-bandwidth-test &> /dev/null; then
        echo "  P2P 带宽测试 (XGMI):"
        rocm-bandwidth-test -t p2p 2>/dev/null | grep -E "GPU|Bandwidth|b/w" | while read line; do
            echo "    ${line}"
        done
    elif command -v rocprof &> /dev/null; then
        echo "  ROCProfiler 带宽测试..."
        rocprof --stats -i /tmp/prof_input.txt -- ./xgmi_bw_test 2>/dev/null || \
        echo "  ⚠️ 需要 xgmi_bw_test 工具"
    else
        echo "  ⚠️ 无带宽测试工具"
    fi
}

show_xgmi_ras() {
    echo ""
    echo "[5/6] XGMI RAS 状态"
    echo "------------------"
    
    for gpu_dir in /sys/class/drm/card?; do
        # 检查 XGMI RAS 接口 (如果存在)
        ras_base="${gpu_dir}/device/ras"
        if [ -d "${ras_base}" ]; then
            # 查找 XGMI 相关的 RAS 计数器
            for ras_entry in $(find ${ras_base} -name "*xgmi*" -o -name "*xgmi*" 2>/dev/null); do
                echo "  ${ras_entry}: $(cat ${ras_entry} 2>/dev/null)"
            done
        fi
    done
    
    # 检查 XGMI 错误计数
    if [ -f /sys/kernel/debug/dri/0/amdgpu_ras_eeprom ]; then
        echo ""
        echo "  RAS EEPROM (XGMI 错误记录):"
        cat /sys/kernel/debug/dri/0/amdgpu_ras_eeprom 2>/dev/null | grep -i "xgmi" | while read line; do
            echo "    ${line}"
        done
    fi
}

show_xgmi_debug() {
    echo ""
    echo "[6/6] XGMI 调试信息"
    echo "------------------"
    
    # 寄存器转储
    for gpu_dir in /sys/kernel/debug/dri/*; do
        if [ -f "${gpu_dir}/amdgpu_regs_xgmi" ]; then
            echo "  XGMI 寄存器:"
            cat "${gpu_dir}/amdgpu_regs_xgmi" 2>/dev/null | head -30 | while read line; do
                echo "    ${line}"
            done
        fi
    done
    
    # 链路状态
    for gpu_dir in /sys/class/drm/card?; do
        if [ -f "${gpu_dir}/device/xgmi_link_status" ]; then
            echo "  XGMI 链路状态:"
            cat "${gpu_dir}/device/xgmi_link_status" | while read line; do
                echo "    ${line}"
            done
        fi
    done
    
    # XGMI 电源状态
    if [ -f /sys/kernel/debug/dri/0/amdgpu_pm_info ]; then
        echo ""
        echo "  XGMI 电源信息:"
        grep -i "xgmi" /sys/kernel/debug/dri/0/amdgpu_pm_info 2>/dev/null | while read line; do
            echo "    ${line}"
        done
    fi
}

# 主流程
check_xgmi_support
show_xgmi_topology
show_xgmi_peers
show_xgmi_bandwidth
show_xgmi_ras
show_xgmi_debug

echo ""
echo "=============================================="
echo "  查询完成"
echo "=============================================="
```

#### XGMI 带宽和延迟基准测试

```python
#!/usr/bin/env python3
# XGMI 性能基准测试工具

import os
import sys
import time
import struct
import ctypes
import numpy as np
from enum import IntEnum
from typing import List, Dict, Tuple

# XGMI 性能测试配置
XGMI_LINK_BW_GBS = 50  # 每链路 50 GB/s
XGMI_LATENCY_NS = 500   # 链路延迟 500ns
PAGE_SIZE = 4096

class XGMITestType(IntEnum):
    LATENCY = 0      # 延迟测试
    BANDWIDTH = 1    # 带宽测试
    STRESS = 2       # 压力测试
    ATOMIC = 3       # 原子操作测试

class XGMIBenchmark:
    def __init__(self, gpu_count: int = None):
        self.gpu_count = self._detect_xgmi_gpus() if gpu_count is None else gpu_count
        self.results = {}
    
    def _detect_xgmi_gpus(self) -> int:
        """检测 XGMI 互联的 GPU 数量"""
        count = 0
        try:
            for entry in os.listdir("/sys/class/drm"):
                if entry.startswith("card") and entry != "card0":
                    xgmi_path = f"/sys/class/drm/{entry}/device/xgmi_node_id"
                    if os.path.exists(xgmi_path):
                        count += 1
        except FileNotFoundError:
            pass
        return max(count, 1)
    
    def _allocate_pinned_memory(self, size_mb: int) -> Tuple[int, int]:
        """分配固定内存 (huge pages)"""
        # 使用 mmap 分配 huge pages
        import mmap
        size = size_mb * 1024 * 1024
        
        # 尝试分配 2MB huge pages
        try:
            buf = mmap.mmap(
                -1, size,
                mmap.MAP_PRIVATE | mmap.MAP_ANONYMOUS | mmap.MAP_HUGETLB,
                mmap.PROT_READ | mmap.PROT_WRITE
            )
            addr = ctypes.addressof(ctypes.c_char.from_buffer(buf))
            return addr, size
        except:
            # 回退到普通内存
            buf = bytearray(size)
            addr = ctypes.addressof(ctypes.c_char.from_buffer(buf))
            return addr, size
    
    def measure_latency(self, src_gpu: int, dst_gpu: int,
                         iterations: int = 10000) -> Dict:
        """测量 XGMI P2P 延迟"""
        print(f"测量 GPU {src_gpu} → GPU {dst_gpu} P2P 延迟...")
        
        latencies = []
        msg_sizes = [4, 64, 256, 1024, 4096]  # bytes
        
        for size in msg_sizes:
            times = []
            
            for _ in range(iterations):
                # 使用 RCCL 或 ROCclr 执行 P2P 延迟测试
                # 实际实现需要调用 ROCm Runtime API
                # 这里模拟延迟数据
                
                base_latency = XGMI_LATENCY_NS  # 500ns 链路延迟
                size_overhead = (size / 50)  # ~20ns per byte
                variation = np.random.normal(0, 50)  # 随机抖动
                
                total_latency = base_latency + size_overhead + variation
                times.append(total_latency)
            
            lats = {
                "min": min(times),
                "max": max(times),
                "avg": np.mean(times),
                "p50": np.percentile(times, 50),
                "p99": np.percentile(times, 99),
                "p999": np.percentile(times, 99.9),
            }
            
            latencies.append({
                "size_bytes": size,
                **lats
            })
            
            print(f"  {size:>6}B: avg={lats['avg']:.0f}ns, "
                  f"p99={lats['p99']:.0f}ns, p999={lats['p999']:.0f}ns")
        
        return {"type": "latency", "results": latencies}
    
    def measure_bandwidth(self, src_gpu: int, dst_gpu: int,
                           duration: int = 5) -> Dict:
        """测量 XGMI P2P 带宽"""
        print(f"测量 GPU {src_gpu} → GPU {dst_gpu} P2P 带宽...")
        
        sizes = [
            1 * 1024 * 1024,      # 1 MB
            8 * 1024 * 1024,      # 8 MB
            64 * 1024 * 1024,     # 64 MB
            256 * 1024 * 1024,    # 256 MB
            1024 * 1024 * 1024,   # 1 GB
        ]
        
        results = []
        
        for size in sizes:
            # 分配内存
            _, _ = self._allocate_pinned_memory(size // (1024 * 1024))
            
            # 执行带宽测试
            # 实际实现需要调用 HIP/ROCclr API
            # 使用 rocm-bandwidth-test 或自定义 kernel
            
            # 模拟带宽数据 (受链路数量影响)
            num_links = min(4, self.gpu_count)  # 最多 4 条链路
            theoretical_bw = XGMI_LINK_BW_GBS * num_links
            
            # 实际带宽 ~85% 理论值
            actual_bw = theoretical_bw * 0.85
            
            # 小消息效率较低
            efficiency = min(1.0, size / (256 * 1024 * 1024))
            if efficiency < 1.0:
                actual_bw *= (0.5 + efficiency * 0.5)
            
            results.append({
                "size_mb": size // (1024 * 1024),
                "bandwidth_gbs": round(actual_bw, 2),
                "efficiency": round(actual_bw / theoretical_bw, 3),
            })
            
            print(f"  {size // (1024*1024):>6}MB: {actual_bw:.1f} GB/s "
                  f"(efficiency: {results[-1]['efficiency']:.1%})")
        
        return {"type": "bandwidth", "results": results}
    
    def measure_atomic_perf(self, src_gpu: int, dst_gpu: int,
                             iterations: int = 100000) -> Dict:
        """测量 XGMI 原子操作性能"""
        print(f"测量 GPU {src_gpu} → GPU {dst_gpu} 原子操作...")
        
        ops = ["fetch_add", "compare_swap", "and", "or", "xor"]
        results = []
        
        for op in ops:
            times = []
            
            for _ in range(iterations):
                # XGMI 原子操作延迟 ~ 2x 普通 P2P 延迟
                base_latency = XGMI_LATENCY_NS * 2
                variation = np.random.normal(0, 100)
                times.append(base_latency + variation)
            
            results.append({
                "operation": op,
                "avg_latency_ns": np.mean(times),
                "ops_per_sec": 1e9 / np.mean(times),
            })
            
            print(f"  {op:>15}: {results[-1]['avg_latency_ns']:.0f}ns, "
                  f"{results[-1]['ops_per_sec']:.0f} ops/s")
        
        return {"type": "atomic", "results": results}
    
    def test_all_pairs(self) -> Dict:
        """测试所有 GPU 对的 P2P 性能"""
        results = {}
        
        print("\nXGMI 全对测试:")
        print("=" * 60)
        
        for i in range(self.gpu_count):
            for j in range(i + 1, self.gpu_count):
                print(f"\nGPU {i} ↔ GPU {j}:")
                print("-" * 40)
                
                latency = self.measure_latency(i, j, iterations=1000)
                bandwidth = self.measure_bandwidth(i, j)
                atomic = self.measure_atomic_perf(i, j)
                
                results[f"{i}_{j}"] = {
                    "latency": latency,
                    "bandwidth": bandwidth,
                    "atomic": atomic,
                }
        
        return results
    
    def run_stress_test(self, duration: int = 60) -> Dict:
        """XGMI 压力测试"""
        print(f"\nXGMI 压力测试 (持续时间: {duration}s)")
        print("=" * 60)
        
        # 持续 All-Reduce 操作
        import threading
        
        class StressWorker(threading.Thread):
            def __init__(self, gpu_id, duration, stats):
                super().__init__()
                self.gpu_id = gpu_id
                self.duration = duration
                self.stats = stats
            
            def run(self):
                end_time = time.time() + self.duration
                ops = 0
                errors = 0
                
                while time.time() < end_time:
                    try:
                        # 模拟 P2P 读写
                        _ = self._p2p_read_write()
                        ops += 1
                    except:
                        errors += 1
                    
                    # 每秒记录
                    if ops % 1000 == 0:
                        self.stats[self.gpu_id] = {
                            "ops": ops,
                            "errors": errors,
                            "throughput": ops / (time.time() - (end_time - self.duration) + 0.001)
                        }
            
            def _p2p_read_write(self):
                """模拟 P2P 读写"""
                # 实际调用 HIP API
                return True
        
        # 启动所有 GPU 的负载线程
        threads = []
        stats = {}
        
        for gpu_id in range(self.gpu_count):
            worker = StressWorker(gpu_id, duration, stats)
            threads.append(worker)
            worker.start()
        
        # 等待完成
        for t in threads:
            t.join()
        
        return {
            "type": "stress",
            "duration": duration,
            "stats": stats
        }
    
    def generate_report(self, output_file: str = "xgmi_benchmark.json"):
        """生成测试报告"""
        import json
        
        report = {
            "xgmi_config": {
                "gpu_count": self.gpu_count,
                "link_bw_gbs": XGMI_LINK_BW_GBS,
                "link_latency_ns": XGMI_LATENCY_NS,
            },
            "tests": {}
        }
        
        print("\nXGMI 性能测试报告")
        print("=" * 60)
        print(f"GPU 数量: {self.gpu_count}")
        print(f"XGMI 链路带宽: {XGMI_LINK_BW_GBS} GB/s (每条)")
        print(f"XGMI 链路延迟: {XGMI_LATENCY_NS} ns")
        print()
        
        # 全对测试
        all_pairs = self.test_all_pairs()
        report["tests"]["all_pairs"] = all_pairs
        
        # 压力测试
        stress = self.run_stress_test(duration=30)
        report["tests"]["stress"] = stress
        
        # 保存报告
        with open(output_file, "w") as f:
            json.dump(report, f, indent=2)
        
        print(f"\n报告已保存: {output_file}")
        
        return report

# 命令行接口
if __name__ == "__main__":
    benchmark = XGMIBenchmark()
    
    if len(sys.argv) > 1:
        if sys.argv[1] == "all-pairs":
            benchmark.test_all_pairs()
        elif sys.argv[1] == "stress":
            duration = int(sys.argv[2]) if len(sys.argv) > 2 else 60
            benchmark.run_stress_test(duration)
        else:
            print(f"用法: {sys.argv[0]} [all-pairs|stress] [duration]")
    else:
        benchmark.generate_report()
```

#### XGMI 故障排除

```bash
#!/bin/bash
# XGMI 故障诊断工具

echo "XGMI 故障诊断"
echo "========================"
echo ""

# 1. 检查 XGMI 链路状态
check_link_status() {
    echo "[1/4] XGMI 链路状态"
    echo "-------------------"
    
    for gpu_dir in /sys/class/drm/card?; do
        if [ -f "${gpu_dir}/device/xgmi_link_status" ]; then
            echo "  ${gpu_dir##*/}:"
            cat "${gpu_dir}/device/xgmi_link_status" | while read line; do
                echo "    ${line}"
            done
            echo ""
        fi
    done
}

# 2. 检查 XGMI 错误计数
check_ras_errors() {
    echo "[2/4] XGMI RAS 错误检查"
    echo "-----------------------"
    
    # 通过 dmesg 检查 XGMI 相关错误
    echo "  dmesg XGMI 错误:"
    dmesg | grep -i "xgmi" | grep -i "error\|fail\|timeout\|recover" | tail -20 | while read line; do
        echo "    ${line}"
    done
    
    echo ""
    echo "  RAS 错误计数:"
    for dir in /sys/kernel/debug/dri/*; do
        if [ -f "${dir}/amdgpu_ras_errors" ]; then
            cat "${dir}/amdgpu_ras_errors" 2>/dev/null | grep -i "xgmi" | while read line; do
                echo "    ${line}"
            done
        fi
    done
    
    echo ""
    echo "  XGMI 重试计数:"
    for gpu_dir in /sys/class/drm/card?; do
        if [ -f "${gpu_dir}/device/xgmi_retry_count" ]; then
            echo "    ${gpu_dir##*/}: $(cat ${gpu_dir}/device/xgmi_retry_count) 次重试"
        fi
    done
}

# 3. 检查 XGMI 电源和时钟
check_power_clocks() {
    echo "[3/4] XGMI 电源和时钟"
    echo "---------------------"
    
    for gpu_dir in /sys/class/drm/card?; do
        if [ -f "${gpu_dir}/device/pp_dpm_xgmi" ]; then
            echo "  ${gpu_dir##*/} XGMI 时钟:"
            cat "${gpu_dir}/device/pp_dpm_xgmi" | while read line; do
                echo "    ${line}"
            done
            echo ""
        fi
    done
    
    # XGMI 电压
    for gpu_dir in /sys/class/drm/card?; do
        if [ -f "${gpu_dir}/device/pp_od_clk_voltage" ]; then
            echo "  ${gpu_dir##*/} XGMI 电压表:"
            grep "XGMI" "${gpu_dir}/device/pp_od_clk_voltage" 2>/dev/null | while read line; do
                echo "    ${line}"
            done
        fi
    done
}

# 4. XGMI 链路复位
reset_link() {
    echo "[4/4] XGMI 链路复位"
    echo "-------------------"
    
    local gpu_id=$1
    
    if [ -z "${gpu_id}" ]; then
        echo "  用法: $0 reset <gpu_id>"
        return
    fi
    
    echo "  复位 GPU ${gpu_id} 的 XGMI 链路..."
    
    # 方法 1: 通过 sysfs
    if [ -f "/sys/class/drm/card${gpu_id}/device/xgmi_reset" ]; then
        echo 1 > "/sys/class/drm/card${gpu_id}/device/xgmi_reset"
        echo "  ✅ 复位命令已发送"
    fi
    
    # 方法 2: 通过 debugfs
    if [ -f "/sys/kernel/debug/dri/${gpu_id}/amdgpu_xgmi_reset" ]; then
        echo 1 > "/sys/kernel/debug/dri/${gpu_id}/amdgpu_xgmi_reset"
        echo "  ✅ Debugfs 复位已触发"
    fi
    
    # 检查复位结果
    sleep 2
    if [ -f "/sys/class/drm/card${gpu_id}/device/xgmi_link_status" ]; then
        echo "  复位后链路状态:"
        cat "/sys/class/drm/card${gpu_id}/device/xgmi_link_status"
    fi
}

# 主流程
case "$1" in
    status)    check_link_status ;;
    errors)    check_ras_errors ;;
    power)     check_power_clocks ;;
    reset)     reset_link "$2" ;;
    *)
        echo "用法: $0 {status|errors|power|reset}"
        echo ""
        echo "  status - 检查 XGMI 链路状态"
        echo "  errors - 检查 XGMI RAS 错误"
        echo "  power  - 检查 XGMI 电源和时钟"
        echo "  reset  - 复位 XGMI 链路 (需要 GPU ID)"
        echo ""
        echo "示例:"
        echo "  $0 status        # 查看所有 XGMI 链路状态"
        echo "  $0 errors        # 查看 XGMI 错误信息"  
        echo "  $0 reset 0       # 复位 GPU 0 的 XGMI 链路"
        ;;
esac
```

### 案例分析

#### 案例一：XGMI 链路降速导致多 GPU 性能下降

**问题现象：**
```
4x MI250 系统，预期 All-Reduce 带宽 160 GB/s，实际只有 80 GB/s
ROCm 带宽测试结果显示:
  GPU 0-1: 45 GB/s (预期 100 GB/s)
  GPU 0-2: 42 GB/s (预期 100 GB/s)
  GPU 0-3: 38 GB/s (预期 100 GB/s)
  所有 P2P 对都只有 ~50% 性能
```

**诊断过程：**
```bash
# 1. 检查 XGMI 链路状态
cat /sys/class/drm/card0/device/xgmi_link_status
# 输出:
# Link 0: Width x16, Speed 8.0 GT/s (Gen4)  ← 应该是 16 GT/s (Gen5)
# Link 1: Width x16, Speed 8.0 GT/s (Gen4)  ← 降速了
# Link 2: Width x16, Speed 8.0 GT/s (Gen4)
# Link 3: Width x16, Speed 8.0 GT/s (Gen4)

# 2. 检查 dmesg 错误
dmesg | grep -i "xgmi"
# [    5.123] amdgpu: XGMI: link 0 training failed, retrying
# [    5.456] amdgpu: XGMI: link 0 retry success, limited to Gen4
# [    5.789] amdgpu: XGMI: all links trained at Gen4 speed

# 3. 检查 XGMI 重试计数
cat /sys/class/drm/card0/device/xgmi_retry_count
# 输出: 5  (多次重试，说明链路质量不佳)
```

**根本原因：** XGMI 链路训练时失败，回退到 Gen4 速率。这是由于 PCB 走线过长或信号完整性问题导致。

```bash
# 链路训练过程:
# 1. 启动: 所有链路尝试 Gen5 速率
# 2. 链路 0: 训练失败 (CRC 错误率 > 阈值)
# 3. 重试: 使用 Gen5 再次训练
# 4. 再失败: 回退到 Gen4
# 5. 其他链路: 为了保持一致，全部回退到 Gen4

# 性能影响:
# Gen5 x16: 100 GB/s per link (双向)
# Gen4 x16:  50 GB/s per link (双向)
# 总性能损失: 50%
```

**解决方案：**
```bash
# 方案 1: 硬件修复
# - 检查 XGMI 连接器是否松动
# - 清洁 PCB 连接器触点
# - 确保散热良好 (温度过高会影响信号质量)
# - 减少 XGMI 走线中的串扰

# 方案 2: 强制链路速率 (如果需要稳定运行)
# 通过 sysfs 强制 Gen5 重新训练
echo "force_gen5" > /sys/class/drm/card0/device/xgmi_link_speed
echo 1 > /sys/class/drm/card0/device/xgmi_reset

# 方案 3: 软件优化补偿
# 减少跨 XGMI 通信量
export RCCL_BUFFER_SIZE=16M  # 减小缓冲区大小
export RCCL_ALGO=Tree        # 使用 Tree AllReduce (减少通信量)

# 方案 4: 检查并调整 XGMI 电压
cat /sys/class/drm/card0/device/pp_od_clk_voltage | grep XGMI
# 如果支持，适当提高 XGMI 电压改善链路质量
echo "s XGMI 1.0 850" > /sys/class/drm/card0/device/pp_od_clk_voltage
```

#### 案例二：XGMI 一致性导致的内存访问异常

**问题现象：**
```python
# GPU 0 写入数据
# GPU 1 读取同一地址，但读取到旧数据
# 代码示例 (伪代码)

# GPU 0:
xgmi_store(peer_addr, value=42)
# 预期: GPU 1 能立即看到 42
# 实际: GPU 1 可能读到之前的值

# GPU 1:
result = xgmi_load(peer_addr)
print(result)  # 可能打印旧值 (如 0)，而不是 42
```

**诊断过程：**
```bash
# 1. 检查 XGMI 一致性配置
cat /sys/class/drm/card0/device/xgmi_coherency
# 输出: non-coherent (非一致性模式)
# 问题: 非一致性模式下需要手动 flush

# 2. 检查 dmesg 一致性警告
dmesg | grep -i "coherency\|coherent"
# [    3.456] amdgpu: XGMI running in non-coherent mode
# [    3.456] amdgpu: Please use atomic operations for synchronization
```

**根本原因：** XGMI 配置为非一致性模式，需要显式的同步操作。

**解决方案：**
```python
# 方案 1: 使用 XGMI 原子操作确保可见性
import ctypes

def xgmi_atomic_store(addr, value):
    """使用原子操作确保数据可见"""
    # XGMI atomic fetch-and-store
    # 确保写入操作通过 XGMI 一致性引擎传播
    ctypes.memmove(addr, value, 8)  # 8 bytes
    # 触发 XGMI 写入屏障
    xgmi_memory_barrier()

def xgmi_atomic_load(addr):
    """带一致性的读取"""
    # XGMI atomic load with acquire semantics
    xgmi_memory_barrier()
    result = ctypes.memmove(ctypes.c_uint64(), addr, 8)
    return result

# 方案 2: 使用 __atomic 内置函数
# C/C++:
# __atomic_store_n(addr, value, __ATOMIC_RELEASE);
# __atomic_load_n(addr, __ATOMIC_ACQUIRE);

# 方案 3: 启用硬件一致性 (如果支持)
echo "coherent" > /sys/class/drm/card0/device/xgmi_coherency
# 注意: 启用一致性可能带来性能开销

# 方案 4: 使用 hipEventSynchronize 确保跨 GPU 一致性
# hipEvent_t event;
# hipEventCreate(&event);
# hipEventRecord(event, stream_gpu0);
# hipStreamWaitEvent(stream_gpu1, event, 0);
```

### 相关链接

- [AMD Infinity Architecture 白皮书](https://www.amd.com/en/technologies/infinity-architecture)
- [AMD XGMI 技术概述](https://www.amd.com/system/files/documents/amd-xgmi-white-paper.pdf)
- [AMDGOU XGMI 驱动源码](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/amdgpu/amdgpu_xgmi.c)
- [ROCm 多 GPU 编程](https://rocm.docs.amd.com/en/latest/how-to/multi-gpu.html)
- [RCCL 集合通信库](https://github.com/ROCmSoftwarePlatform/rccl)

### 今日小结

XGMI / Infinity Fabric 是 AMD 多 GPU 互联的核心技术：

1. **物理层**: 多条 SerDes 链路，每链路 50-100 GB/s，低延迟
2. **拓扑**: GPU 通过 XGMI 组成蜂巢（Hive），统一内存地址空间
3. **一致性**: MOESI 协议，硬件维护缓存一致性
4. **驱动支持**: amdgpu_xgmi.c 管理拓扑发现、P2P 映射、RAS 监控
5. **性能测试**: 延迟 ~500ns，带宽可达 200 GB/s (4 链路)
6. **故障排除**: 链路降速、一致性模式、RAS 错误计数
7. **测试重点**: P2P 带宽、原子操作延迟、链路训练稳定性

### 扩展思考

1. XGMI 和 NVLink 在架构设计上的主要区别是什么？对驱动开发有何影响？
2. 在 XGMI 非一致性模式下，驱动如何确保跨 GPU 同步的正确性？
3. 多 GPU 系统的 XGMI 链路故障（如一链路断开）对运行中的工作负载有何影响？
4. 对于 GPU 驱动测试，XGMI 场景需要覆盖哪些特殊的测试用例（如链路重训练、P2P 映射失败恢复）？
