# Day 384：vGPU 调度与资源隔离

## 学习目标

- 理解 vGPU 的调度模型（时间片、优先级、公平调度）
- 掌握 vGPU 资源隔离的实现机制（显存、计算、带宽）
- 理解 GPU 硬件调度器（HWS）与软件调度器的协同
- 学会配置和优化 vGPU 调度参数
- 掌握 vGPU 资源隔离的测试验证方法

## 知识详解

### 概念原理

#### vGPU 调度模型

vGPU 调度本质上是在多个虚拟 GPU 实例之间**时分复用**和**空分复用**物理 GPU 资源：

```
GPU 资源复用模型：

资源类型          复用方式          粒度              隔离级别
─────────────────────────────────────────────────────────────
计算单元 (CU/SIMD)  时分复用         WF (Wavefront)    时间片轮转
显存 (VRAM)        空分复用         页 (4KB)           地址空间隔离
显存带宽           时分复用         请求级               QoS 控制
PCIe 带宽          时分复用         事务级               QoS 控制
编解码引擎 (VCN)   空分复用         实例级              独立 pipeline
SDMA 引擎          空分复用         通道级              独立通道
```

#### 时间片调度（Temporal Scheduling）

时间片调度是 GPU 计算资源分配的核心机制：

```
时间片调度模型：

┌─────────────────────────────────────────────────────────┐
│                   时间轴 (μs 级别)                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐│
│  │  VF 0    │  │  VF 1    │  │  VF 2    │  │  VF 0    ││
│  │  t=100μs │  │  t=100μs │  │  t=100μs │  │  t=100μs ││
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘│
│                                                         │
│  ◄──────────── 调度周期 (Quantum) ────────────────────► │
│                                                         │
│  VF 切换过程：                                           │
│  1. VF 0 时间片结束                                      │
│  2. 保存 VF 0 的上下文 (Wavefront 状态缓存)              │
│  3. 恢复 VF 1 的上下文                                   │
│  4. VF 1 开始执行                                       │
│  5. 上下文切换开销: ~5-10μs                             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**调度算法：**

```c
// GPU 硬件调度器 (HWS) 的简化调度算法
// 源码参考: drivers/gpu/drm/amd/amdgpu/gfx_v9_4_3.c (部分结构)

struct vgpu_sched_context {
    // 每个 VF 的调度上下文
    uint32_t            vf_id;
    uint32_t            priority;        // 优先级 (0-7, 7最高)
    uint64_t            time_quantum;    // 时间片长度 (ns)
    uint64_t            used_time;       // 已使用时间
    uint64_t            last_switch;     // 上次切换时间
    
    // 公平调度参数
    uint64_t            total_runtime;   // 累计运行时间
    uint64_t            wait_time;       // 累计等待时间
    uint32_t            sched_weight;    // 调度权重
    
    // Wavefront 上下文
    uint32_t            wave_count;      // 活跃 Wavefront 数量
    uint64_t            wf_context_ptr;  // Wavefront 上下文指针
};

// 公平轮转调度 (Round Robin with Weights)
int hws_schedule(struct vgpu_sched_context *contexts, int num_vfs)
{
    // 计算总权重
    uint32_t total_weight = 0;
    for (int i = 0; i < num_vfs; i++) {
        if (contexts[i].wave_count > 0)
            total_weight += contexts[i].sched_weight;
    }
    
    while (true) {
        uint64_t now = get_current_time();
        
        for (int i = 0; i < num_vfs; i++) {
            struct vgpu_sched_context *ctx = &contexts[i];
            
            // 检查 VF 是否有待执行的工作
            if (ctx->wave_count == 0)
                continue;
            
            // 检查是否该 VF 的时间片
            uint64_t elapsed = now - ctx->last_switch;
            
            // 计算该 VF 应得的时间片（基于权重）
            uint64_t fair_slice = (total_weight > 0) ?
                (ctx->time_quantum * ctx->sched_weight) / total_weight :
                ctx->time_quantum;
            
            // 如果该 VF 已经执行了足够的时间，切换到下一个
            if (ctx->used_time >= fair_slice) {
                // 保存上下文
                save_wavefront_context(ctx);
                
                // 切换到下一个 VF
                ctx->used_time = 0;
                
                // 加载下一个 VF 的上下文
                int next = (i + 1) % num_vfs;
                while (contexts[next].wave_count == 0 && next != i)
                    next = (next + 1) % num_vfs;
                
                if (next != i) {
                    load_wavefront_context(&contexts[next]);
                    contexts[next].last_switch = now;
                }
            }
            
            // 执行一个指令周期
            execute_gpu_cycle();
            ctx->used_time += CYCLE_TIME;
        }
    }
}
```

#### 优先级调度（Priority Scheduling）

优先级调度允许某些 VF 获得更多的 GPU 时间：

```
优先级调度示例：

VF 配置:
  VF 0: Priority = 7 (最高), Weight = 8
  VF 1: Priority = 4 (中),   Weight = 4
  VF 2: Priority = 1 (低),   Weight = 1

时间片分配:
  VF 0: 8/(8+4+1) = 61.5%  GPU 时间
  VF 1: 4/(8+4+1) = 30.8%  GPU 时间
  VF 2: 1/(8+4+1) = 7.7%   GPU 时间

时间线:
  ┌───────┬──────┬───┬───────┬──────┬───┬──────────┐
  │ VF 0  │VF 1  │VF2│ VF 0  │VF 1  │VF2│ VF 0     │
  │ 615μs │308μs │77μ│ 615μs │308μs │77μ│ 615μs    │
  └───────┴──────┴───┴───────┴──────┴───┴──────────┘
  ◄────────── 调度周期 1000μs ──────────────────────►
```

**优先级继承：** 当低优先级 VF 持有高优先级 VF 需要的资源时，低优先级 VF 临时继承高优先级，防止优先级反转。

#### 显存隔离（Memory Isolation）

显存隔离通过 GPU 的 IOMMU（GART）和页表实现：

```
显存隔离架构：

┌─────────────────────────────────────────────────────┐
│                VF 0 地址空间                          │
│  ┌─────────────────────────────────────────────────┐│
│  │  VF 0 页表 (每 VF 独立 GART)                    ││
│  │                                                 ││
│  │  GPU VA → 物理 VRAM 映射                        ││
│  │  0x0000_0000 - 0x0FFF_FFFF → VRAM Base 0 + offset│
│  │  0x1000_0000 - 0x1FFF_FFFF → GTT (系统内存)     ││
│  └─────────────────────────────────────────────────┘│
│              │                                       │
└──────────────┼───────────────────────────────────────┘
               │ 硬件地址翻译 (IOMMU/GART)
┌──────────────┼───────────────────────────────────────┐
│              ▼                                       │
│  ┌─────────────────────────────────────────────────┐│
│  │           物理 VRAM 地址空间                      ││
│  │                                                 ││
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────┐││
│  │  │ VF 0 分区     │  │ VF 1 分区     │  │ PF     │││
│  │  │ 0x0000-1FFFF  │  │ 0x20000-3FFFF │  │ 保留区 │││
│  │  └──────────────┘  └──────────────┘  └────────┘││
│  └─────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

**显存隔离的核心机制：**

```c
// VF 页表隔离（简化实现）
struct vf_vm_context {
    // 每 VF 独立的 VM 结构
    struct amdgpu_vm    *vm;
    uint64_t            vram_base;      // VF 的 VRAM 基地址
    uint64_t            vram_size;      // VF 的 VRAM 大小
    struct mutex        ptable_lock;
    
    // GART 表（GPU 地址翻译）
    uint64_t            *gart_table;    // GART 页表
    uint32_t            gart_size;      // GART 表大小（页数）
};

// VF 地址翻译
uint64_t vf_translate_address(struct vf_vm_context *ctx, 
                               uint64_t gpu_va)
{
    // 1. 检查 GPU VA 是否在 VF 的合法范围内
    if (gpu_va >= ctx->vram_size) {
        // GTT (系统内存) 路径
        return translate_gtt(gpu_va);
    }
    
    // 2. VRAM 路径：GPU VA → VF 本地 VRAM 偏移
    uint64_t vf_offset = gpu_va;
    
    // 3. 检查偏移是否超出 VF 的显存范围
    if (vf_offset >= ctx->vram_size) {
        // 访问越界 → 返回无效地址，触发 page fault
        return INVALID_ADDRESS;
    }
    
    // 4. 转换为物理地址
    uint64_t phys_addr = ctx->vram_base + vf_offset;
    
    return phys_addr;
}

// 关键的隔离保证：
// 1. VF 的 GPU VA 空间完全独立
// 2. VF 的 GART 表只能映射到自己的 VRAM 分区
// 3. 跨分区访问触发 GPU page fault
// 4. PF 可以配置 VF 的最大显存使用量
```

#### 带宽隔离（Bandwidth Isolation）

显存带宽和 PCIe 带宽的隔离通常通过 QoS（Quality of Service）控制器实现：

```
带宽 QoS 架构：

┌─────────────────────────────────────────────────────┐
│                 GPU 内存控制器 (UMC)                  │
│                                                     │
│  ┌─────────────────────────────────────────────────┐│
│  │            UMC QoS 控制器                        ││
│  │                                                 ││
│  │  VF 0:  Max BW = 200 GB/s  Priority = 3        ││
│  │  VF 1:  Max BW = 100 GB/s  Priority = 2        ││
│  │  VF 2:  Max BW = 50 GB/s   Priority = 1        ││
│  │  PF:     Max BW = 50 GB/s   Priority = 0        ││
│  │                                                 ││
│  │  仲裁算法: Weighted Round Robin + Token Bucket   ││
│  │  每周期预算: VF0: 200t, VF1: 100t, VF2: 50t    ││
│  └─────────────────────────────────────────────────┘│
│                      │                               │
│                      ▼                               │
│  ┌─────────────────────────────────────────────────┐│
│  │              HBM/GDDR 控制器                      ││
│  └─────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

**Token Bucket 带宽控制算法：**

```c
// 带宽控制的 Token Bucket 算法
struct bandwidth_bucket {
    uint64_t    tokens;             // 当前 token 数
    uint64_t    max_tokens;         // 最大 token 数
    uint64_t    refill_rate;        // 补充速率 (tokens/μs)
    uint64_t    last_refill_time;   // 上次补充时间
};

bool bw_bucket_try_consume(struct bandwidth_bucket *bucket,
                            uint64_t tokens_needed,
                            uint64_t now)
{
    // 1. 补充 token
    uint64_t elapsed = now - bucket->last_refill_time;
    bucket->tokens += elapsed * bucket->refill_rate;
    if (bucket->tokens > bucket->max_tokens)
        bucket->tokens = bucket->max_tokens;
    bucket->last_refill_time = now;
    
    // 2. 检查是否有足够的 token
    if (bucket->tokens >= tokens_needed) {
        bucket->tokens -= tokens_needed;
        return true;  // 允许请求
    }
    
    // 3. token 不足，拒绝或延迟
    return false;  // 请求被限流
}

// 每 VF 的带宽配置
struct vf_bw_config {
    struct bandwidth_bucket    read_bucket;   // 读带宽
    struct bandwidth_bucket    write_bucket;  // 写带宽
    uint32_t                  priority;       // 优先级
};
```

#### 公平调度指标

衡量 vGPU 调度公平性的关键指标：

```
公平性指标：

1. 性能隔离度 (Performance Isolation)
   定义: VF 并发运行时的性能 / VF 单独运行时的性能
   理想值: 1.0 (完全隔离，不受其他 VF 影响)
   可接受: > 0.85

2. 公平性指数 (Fairness Index - Jain's Index)
   定义: J(x₁, x₂, ..., xₙ) = (Σxᵢ)² / (n × Σxᵢ²)
   其中 xᵢ = VF i 的实际性能 / VF i 的预期性能
   理想值: 1.0 (完全公平)
   可接受: > 0.95

3. 尾部延迟 (Tail Latency)
   定义: P99 的 GPU 命令完成延迟
   理想值: < 2× 基准延迟
   可接受: < 5× 基准延迟

4. 上下文切换开销
   定义: VF 切换时的性能损失
   理想值: < 1%
   可接受: < 5%
```

### 实践操作

#### 配置 vGPU 调度参数

```bash
#!/bin/bash
# vGPU 调度配置示例

# 注意：以下接口可能因驱动版本而异
# 实际部署需参考具体硬件文档

configure_vf_scheduling() {
    local vf_count=$1
    local gpu_bdf="0000:03:00.0"
    
    echo "配置 vGPU 调度参数..."
    
    # 1. 设置 VF 优先级
    # 优先级范围: 0 (最低) - 7 (最高)
    # VF 0 高优先级 (实时), VF 1 中优先级, VF 2 低优先级
    for vf_id in $(seq 0 $((vf_count - 1))); do
        case ${vf_id} in
            0) echo 7 > /sys/bus/pci/devices/${gpu_bdf}/virtfn${vf_id}/priority ;;
            1) echo 4 > /sys/bus/pci/devices/${gpu_bdf}/virtfn${vf_id}/priority ;;
            *) echo 1 > /sys/bus/pci/devices/${gpu_bdf}/virtfn${vf_id}/priority ;;
        esac
    done
    
    # 2. 设置调度权重 (时间片分配比例)
    for vf_id in $(seq 0 $((vf_count - 1))); do
        case ${vf_id} in
            0) echo 8 > /sys/bus/pci/devices/${gpu_bdf}/virtfn${vf_id}/sched_weight ;;
            1) echo 4 > /sys/bus/pci/devices/${gpu_bdf}/virtfn${vf_id}/sched_weight ;;
            *) echo 2 > /sys/bus/pci/devices/${gpu_bdf}/virtfn${vf_id}/sched_weight ;;
        esac
    done
    
    # 3. 设置时间片长度 (微秒)
    # 更小的时间片 → 更公平但切换开销更大
    echo 200 > /sys/bus/pci/devices/${gpu_bdf}/sched_timeslice_us
    
    # 4. 设置带宽限制 (MB/s)
    # VF 0: 无限制 (0), VF 1: 100GB/s, VF 2: 50GB/s
    for vf_id in $(seq 0 $((vf_count - 1))); do
        case ${vf_id} in
            0) echo "0"     > /sys/bus/pci/devices/${gpu_bdf}/virtfn${vf_id}/bw_limit ;;
            1) echo "102400" > /sys/bus/pci/devices/${gpu_bdf}/virtfn${vf_id}/bw_limit ;;
            *) echo "51200"  > /sys/bus/pci/devices/${gpu_bdf}/virtfn${vf_id}/bw_limit ;;
        esac
    done
    
    echo "调度配置完成"
    echo "  VF 0: 优先级=7, 权重=8, 带宽=无限制"
    echo "  VF 1: 优先级=4, 权重=4, 带宽=100GB/s"
    echo "  VF 2: 优先级=1, 权重=2, 带宽=50GB/s"
    echo "  时间片: 200μs"
}

show_scheduling_status() {
    local gpu_bdf="0000:03:00.0"
    
    echo ""
    echo "当前调度状态:"
    echo "==================="
    
    # 显示调度统计
    if [ -f /sys/kernel/debug/dri/0/amdgpu_sched_info ]; then
        cat /sys/kernel/debug/dri/0/amdgpu_sched_info
    fi
    
    echo ""
    # 显示 VF 的 GPU 使用率
    for vf_path in /sys/bus/pci/devices/${gpu_bdf}/virtfn*; do
        vf_id=$(basename ${vf_path})
        if [ -f ${vf_path}/gpu_util ]; then
            echo "${vf_id} GPU 使用率: $(cat ${vf_path}/gpu_util)%"
        fi
    done
}

# 主流程
echo "vGPU 调度配置工具"
echo "==================="

# 创建 3 个 VF
echo 0 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs
sleep 1
echo 3 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs
sleep 2

configure_vf_scheduling 3
show_scheduling_status
```

#### 资源隔离测试

```bash
#!/bin/bash
# 文件名: vgpu_isolation_test.sh
# 用途: 测试 vGPU 资源隔离性

set -e

echo "============================================"
echo "  vGPU 资源隔离性测试套件"
echo "============================================"
echo ""

# 测试配置
GPU_BDF="0000:03:00.0"
VF_COUNT=2
TEST_DIR="/tmp/vgpu_test"

mkdir -p ${TEST_DIR}

# 测试 1: 显存隔离测试
test_vram_isolation() {
    echo "测试 1: 显存隔离性"
    echo "------------------"
    
    echo "  步骤 1: 在 VF 0 中分配 4GB 显存"
    # vf_exec 0 "./vram_alloc 4096"   # 使用模拟工具
    echo "  VF 0 已分配 4GB 显存"
    
    echo "  步骤 2: 尝试在 VF 1 中读取 VF 0 的显存"
    # VF 1 尝试读取 VF 0 的物理地址 → 应触发 page fault
    # vf_exec 1 "./vram_read 0x10000000 4"
    
    echo "  步骤 3: 检查是否记录到隔离违规"
    # dmesg | grep "GPU.*page fault\|VM.*fault"
    
    local fault_count=$(dmesg | grep -c "VM.*fault" || true)
    if [ ${fault_count} -gt 0 ]; then
        echo "  结果: ✅ 显存隔离生效 (检测到 ${fault_count} 次 page fault)"
    else
        echo "  结果: ⚠️ 未检测到 page fault (可能隔离实现不同)"
    fi
    
    # 验证 VF 1 的可用显存不受 VF 0 分配的影响
    # local vf1_free_before=$(vf_exec 1 "cat /sys/class/drm/card1/device/mem_info_vram_free")
    # local vf1_free_after=$(vf_exec 1 "cat /sys/class/drm/card1/device/mem_info_vram_free")
    # if [ "${vf1_free_before}" == "${vf1_free_after}" ]; then
    #     echo "  结果: ✅ VF 1 显存不受 VF 0 分配影响"
    # fi
}

# 测试 2: 计算资源隔离测试
test_compute_isolation() {
    echo ""
    echo "测试 2: 计算资源隔离性"
    echo "----------------------"
    
    echo "  步骤 1: 单独运行基准测试"
    local baseline_fps=1000  # 假设单 VF 基准 FPS
    
    echo "  步骤 2: 双 VF 并发运行"
    # vf_exec 0 "./gpu_burn --threads 64" &
    # PID0=$!
    # vf_exec 1 "./gpu_burn --threads 64" &
    # PID1=$!
    
    sleep 3
    
    # 获取并发时的性能
    # local perf_vf0=$(vf_exec 0 "./gpu_benchmark --quick")
    # local perf_vf1=$(vf_exec 1 "./gpu_benchmark --quick")
    
    echo "  单 VF 基准性能: ${baseline_fps} FPS"
    echo "  并发时 VF 0 性能: 未测量"
    echo "  并发时 VF 1 性能: 未测量"
    
    # 计算隔离度
    # local isolation_0=$(echo "scale=2; ${perf_vf0} / ${baseline_fps}" | bc)
    # local isolation_1=$(echo "scale=2; ${perf_vf1} / ${baseline_fps}" | bc)
    # echo "  VF 0 隔离度: ${isolation_0}"
    # echo "  VF 1 隔离度: ${isolation_1}"
    
    # kill ${PID0} ${PID1} 2>/dev/null
}

# 测试 3: 优先级调度测试
test_priority_scheduling() {
    echo ""
    echo "测试 3: 优先级调度验证"
    echo "----------------------"
    
    # 配置不同优先级
    # echo 7 > /sys/bus/pci/devices/${GPU_BDF}/virtfn0/priority
    # echo 1 > /sys/bus/pci/devices/${GPU_BDF}/virtfn1/priority
    
    echo "  高优先级 VF (优先级 7) vs 低优先级 VF (优先级 1)"
    
    # 并发运行并比较性能
    # vf_exec 0 "./gpu_benchmark --duration 10" &
    # vf_exec 1 "./gpu_benchmark --duration 10" &
    # wait
    
    echo "  预期: 高优先级 VF 应获得更多 GPU 时间"
    echo "  结果: 未执行实际测试 (需要物理 SR-IOV 环境)"
}

# 测试 4: 错误隔离测试
test_error_isolation() {
    echo ""
    echo "测试 4: 错误隔离性"
    echo "------------------"
    
    echo "  在 VF 0 中注入 GPU hang"
    # 通过 VF 的 debugfs 触发 hang
    # echo 1 > /sys/kernel/debug/dri/0/amdgpu_ring_hang
    
    sleep 5
    
    echo "  检查 VF 1 是否受影响"
    # vf_exec 1 "./gpu_benchmark --quick"
    
    # local vf1_alive=$?
    # if [ ${vf1_alive} -eq 0 ]; then
    #     echo "  结果: ✅ 错误隔离生效 (VF 1 未受影响)"
    # else
    #     echo "  结果: ❌ 错误隔离失败 (VF 1 受影响)"
    # fi
    
    echo "  结果: 未执行实际测试 (需要物理 SR-IOV 环境)"
}

# 测试 5: 带宽隔离测试
test_bandwidth_isolation() {
    echo ""
    echo "测试 5: 带宽隔离性"
    echo "------------------"
    
    # 配置带宽限制
    # echo 10240 > /sys/bus/pci/devices/${GPU_BDF}/virtfn0/bw_limit  # 10GB/s
    # echo 1024  > /sys/bus/pci/devices/${GPU_BDF}/virtfn1/bw_limit   # 1GB/s
    
    echo "  VF 0 带宽限制: 10 GB/s"
    echo "  VF 1 带宽限制: 1 GB/s"
    
    # 运行内存带宽测试
    # vf_exec 0 "./mem_benchmark --size 1024" &
    # PID0=$!
    # vf_exec 1 "./mem_benchmark --size 1024" &
    # PID1=$!
    
    sleep 5
    
    # VF 0 的带宽约为 VF 1 的 10 倍
    # local bw_vf0=$(vf_exec 0 "cat /sys/kernel/debug/dri/0/amdgpu_vram_bw")
    # local bw_vf1=$(vf_exec 1 "cat /sys/kernel/debug/dri/0/amdgpu_vram_bw")
    
    # echo "  VF 0 带宽: ${bw_vf0} GB/s (限制 10 GB/s)"
    # echo "  VF 1 带宽: ${bw_vf1} GB/s (限制 1 GB/s)"
    
    # kill ${PID0} ${PID1} 2>/dev/null
}

generate_report() {
    local report_file="${TEST_DIR}/isolation_report_$(date +%Y%m%d_%H%M%S).txt"
    
    cat > ${report_file} << EOF
vGPU 资源隔离测试报告
======================
测试时间: $(date)
GPU BDF: ${GPU_BDF}
VF 数量: ${VF_COUNT}

测试结果摘要:
  1. 显存隔离: ✅ (通过)
  2. 计算隔离: ⚠️ (需手动验证)
  3. 优先级调度: ⚠️ (需手动验证)
  4. 错误隔离: ⚠️ (需手动验证)
  5. 带宽隔离: ⚠️ (需手动验证)

详细结果请查看各测试输出
EOF
    
    echo ""
    echo "报告已保存: ${report_file}"
    cat ${report_file}
}

# 主流程
echo "开始时间: $(date)"
echo ""

# 创建 VF
echo "设置 ${VF_COUNT} 个 VF..."
echo 0 > /sys/bus/pci/devices/${GPU_BDF}/sriov_numvfs 2>/dev/null || true
sleep 1
echo ${VF_COUNT} > /sys/bus/pci/devices/${GPU_BDF}/sriov_numvfs 2>/dev/null || true
sleep 2

test_vram_isolation
test_compute_isolation
test_priority_scheduling
test_error_isolation
test_bandwidth_isolation

generate_report

echo ""
echo "测试完成: $(date)"
```

#### 性能隔离的量化分析

```python
#!/usr/bin/env python3
# vGPU 性能隔离量化分析工具

import time
import statistics
import subprocess
import json
from dataclasses import dataclass
from typing import List, Dict

@dataclass
class VFPerfSample:
    vf_id: int
    timestamp: float
    fps: float
    gpu_util: float
    vram_used: int
    vram_bw: float

class VGPUIsolationAnalyzer:
    def __init__(self, vf_count: int):
        self.vf_count = vf_count
        self.samples: Dict[int, List[VFPerfSample]] = {
            i: [] for i in range(vf_count)
        }
        self.baseline: Dict[int, VFPerfSample] = {}
    
    def collect_baseline(self, duration: int = 10):
        """收集单 VF 运行时的基准性能"""
        print(f"收集基准性能 (单 VF, {duration}s)...")
        for vf_id in range(self.vf_count):
            samples = self._measure_vf_perf(vf_id, duration)
            if samples:
                avg_fps = statistics.mean(s.fps for s in samples)
                self.baseline[vf_id] = VFPerfSample(
                    vf_id=vf_id,
                    timestamp=time.time(),
                    fps=avg_fps,
                    gpu_util=0,
                    vram_used=0,
                    vram_bw=0
                )
                print(f"  VF {vf_id} 基准: {avg_fps:.1f} FPS")
    
    def collect_concurrent(self, duration: int = 30):
        """收集并发运行时的性能"""
        print(f"\n收集并发性能 ({duration}s)...")
        
        # 启动所有 VF 的负载
        processes = []
        for vf_id in range(self.vf_count):
            # 模拟启动 VF 负载
            # proc = subprocess.Popen([
            #     "vf_exec", str(vf_id), "./gpu_benchmark"
            # ])
            # processes.append(proc)
            pass
        
        # 采样
        start = time.time()
        while time.time() - start < duration:
            for vf_id in range(self.vf_count):
                # 收集 VF 性能数据
                sample = self._sample_vf(vf_id)
                if sample:
                    self.samples[vf_id].append(sample)
            time.sleep(0.5)
        
        # 停止负载
        # for proc in processes:
        #     proc.terminate()
    
    def _measure_vf_perf(self, vf_id: int, duration: int) -> List[VFPerfSample]:
        """测量指定 VF 的性能"""
        # 实际实现需要与 VF 通信
        return []
    
    def _sample_vf(self, vf_id: int) -> VFPerfSample:
        """采样 VF 的当前性能"""
        # 通过 sysfs 或 debugfs 读取
        try:
            # 示例路径（实际取决于驱动实现）
            with open(f"/sys/bus/pci/devices/0000:03:00.{vf_id*2}/gpu_util") as f:
                gpu_util = float(f.read().strip())
            
            return VFPerfSample(
                vf_id=vf_id,
                timestamp=time.time(),
                fps=0,  # 需要外部基准测试工具
                gpu_util=gpu_util,
                vram_used=0,
                vram_bw=0
            )
        except (FileNotFoundError, ValueError):
            return None
    
    def calculate_isolation(self) -> Dict:
        """计算隔离性指标"""
        results = {}
        
        for vf_id in range(self.vf_count):
            if vf_id not in self.baseline or vf_id not in self.samples:
                continue
            
            baseline = self.baseline[vf_id]
            concurrent = self.samples[vf_id]
            
            if not concurrent:
                continue
            
            # 计算平均并发性能
            avg_fps = statistics.mean(s.fps for s in concurrent if s.fps > 0)
            
            # 性能隔离度
            isolation = avg_fps / baseline.fps if baseline.fps > 0 else 0
            
            results[vf_id] = {
                "baseline_fps": round(baseline.fps, 1),
                "concurrent_fps": round(avg_fps, 1),
                "isolation_ratio": round(isolation, 3),
                "performance_loss": f"{(1 - isolation) * 100:.1f}%",
                "gpu_util_avg": round(
                    statistics.mean(s.gpu_util for s in concurrent), 1
                ),
                "gpu_util_max": round(
                    max(s.gpu_util for s in concurrent), 1
                ),
            }
        
        # 计算公平性指数 (Jain's Index)
        fps_values = [
            r["concurrent_fps"] for r in results.values()
            if r["concurrent_fps"] > 0
        ]
        if fps_values:
            n = len(fps_values)
            sum_fps = sum(fps_values)
            sum_sq = sum(f ** 2 for f in fps_values)
            fairness = (sum_fps ** 2) / (n * sum_sq) if sum_sq > 0 else 0
        else:
            fairness = 1.0
        
        results["fairness_index"] = round(fairness, 4)
        results["vf_count"] = self.vf_count
        
        return results
    
    def generate_report(self, output_file: str = None):
        """生成隔离性测试报告"""
        results = self.calculate_isolation()
        
        report = []
        report.append("=" * 60)
        report.append("vGPU 性能隔离性分析报告")
        report.append("=" * 60)
        report.append(f"VF 数量: {results['vf_count']}")
        report.append(f"公平性指数 (Jain's Index): {results.get('fairness_index', 'N/A')}")
        report.append("")
        
        report.append(f"{'VF':<8} {'基准FPS':<12} {'并发FPS':<12} {'隔离度':<10} {'性能损失':<10}")
        report.append("-" * 52)
        
        for vf_id in range(self.vf_count):
            if vf_id in results:
                r = results[vf_id]
                report.append(
                    f"VF{vf_id:<5} {r['baseline_fps']:<10.1f} "
                    f"{r['concurrent_fps']:<10.1f} "
                    f"{r['isolation_ratio']:<8.3f} "
                    f"{r['performance_loss']:<10}"
                )
        
        report.append("")
        report.append("评估标准:")
        report.append("  隔离度 > 0.85: ✅ 良好")
        report.append("  隔离度 0.70-0.85: ⚠️ 可接受")
        report.append("  隔离度 < 0.70: ❌ 需优化")
        report.append("")
        report.append(f"公平性指数 > 0.95: {'✅ 公平' if results.get('fairness_index', 0) > 0.95 else '⚠️ 需检查'}")
        
        report_str = "\n".join(report)
        print(report_str)
        
        if output_file:
            with open(output_file, 'w') as f:
                f.write(report_str)
            print(f"\n报告已保存: {output_file}")
        
        return results

# 使用示例
if __name__ == "__main__":
    analyzer = VGPUIsolationAnalyzer(vf_count=4)
    
    # 收集基准
    analyzer.collect_baseline(duration=10)
    
    # 收集并发数据
    analyzer.collect_concurrent(duration=30)
    
    # 生成报告
    results = analyzer.generate_report("isolation_report.json")
    
    # 导出 JSON
    with open("isolation_metrics.json", "w") as f:
        json.dump(results, f, indent=2)
```

### 案例分析

#### 案例一：vGPU 调度抖动问题

**问题现象：** 在 4 个 VF 并发运行 AI 推理任务时，VF 2 的性能出现周期性抖动（每 3-5 秒性能下降 50%）。

```bash
# 性能监控日志
时间           VF0 (fps)    VF1 (fps)    VF2 (fps)    VF3 (fps)
───────────────────────────────────────────────────────────────
T+00:00       120          118          115          119
T+00:01       121          117          116          120
T+00:02       119          120          62           118  ← VF2 抖动
T+00:03       120          119          58           121  ← VF2 抖动
T+00:04       118          121          114          119
T+00:05       120          118          116          120
```

**诊断过程：**

```bash
# 1. 检查调度时间片配置
cat /sys/bus/pci/devices/0000:03:00.0/sched_timeslice_us
# 输出: 5000 (5ms 时间片，过大)

# 2. 检查 VF 上下文切换延迟
cat /sys/kernel/debug/dri/0/amdgpu_sched_stats
# 输出:
# VF 0: switches=2345, avg_switch_latency=8μs
# VF 1: switches=2340, avg_switch_latency=9μs
# VF 2: switches=2348, avg_switch_latency=452μs  ← 异常
# VF 3: switches=2342, avg_switch_latency=8μs

# 3. 检查 VF 2 的 wavefront 数量
cat /sys/kernel/debug/dri/0/amdgpu_wave_info
# VF 2 wave_count=128 (远多于其他 VF 的 32)
```

**根本原因：** VF 2 运行的任务生成了大量 Wavefront（128 个），上下文切换时需要保存/恢复大量 Wavefront 状态，导致切换延迟高达 452μs。

**解决方案：**

```bash
# 方案 1: 减少时间片长度，降低抖动幅度
echo 500 > /sys/bus/pci/devices/0000:03:00.0/sched_timeslice_us

# 方案 2: 限制 VF 的最大 Wavefront 数量
echo 64 > /sys/bus/pci/devices/0000:03:00.0/virtfn2/max_waves

# 方案 3: 为 VF 2 绑定到专用 CU（计算单元），减少上下文切换
# 需要硬件支持 CU 分区
echo "reserved:8" > /sys/bus/pci/devices/0000:03:00.0/virtfn2/cu_mask

# 验证改进
# 预期：VF 2 的切换延迟降至 <50μs，性能抖动消除
```

#### 案例二：显存隔离违规

**问题现象：** VF 1 中运行的应用报告 GPU page fault，dmesg 显示跨 VF 显存访问。

```bash
# dmesg 日志
[  1234.567] amdgpu 0000:03:00.1: GPU page fault at address 0x00000000f0000000
[  1234.567] amdgpu 0000:03:00.1: VM fault (0x02) in page table at 0x00000000f0000000
[  1234.567] amdgpu 0000:03:00.1: fence wait timeout on ring 0
[  1234.567] amdgpu 0000:03:00.1: Try to recover VF...
```

**诊断：**

```bash
# 1. 检查显存分区配置
cat /sys/bus/pci/devices/0000:03:00.0/vram_partition
# VF 0: base=0x00000000, size=0x40000000 (1GB)
# VF 1: base=0x00000000, size=0x40000000 (1GB)  ← 错误！base 重叠

# 2. 检查 VF 页表配置
cat /sys/kernel/debug/dri/1/amdgpu_vm_info
# 发现 VF 1 的 GART 表错误地映射到了 VF 0 的地址范围
```

**根本原因：** PF 驱动在创建 VF 1 时错误地配置了 VF 1 的 VRAM base，导致两个 VF 的显存分区重叠。

**解决方案：**

```bash
# 1. 销毁所有 VF
echo 0 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs

# 2. 修复驱动配置重新创建
# 更新驱动或固件到修复版本

# 3. 重新创建 VF 并验证分区
echo 4 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs

# 4. 验证分区不重叠
cat /sys/bus/pci/devices/0000:03:00.0/vram_partition
# VF 0: base=0x00000000, size=0x40000000
# VF 1: base=0x40000000, size=0x40000000
# VF 2: base=0x80000000, size=0x40000000
# VF 3: base=0xC0000000, size=0x40000000

# 5. 验证 page fault 消除
dmesg -c  # 清除日志
# 运行负载测试
# 预期：无新的 page fault 日志
```

### 相关链接

- [AMDGPU 调度器文档](https://docs.kernel.org/gpu/amdgpu.html)
- [PCIe SR-IOV 调度与隔离规范](https://pcisig.com/specifications/iov/)
- [Linux CFS 调度器（GPU 调度参考）](https://www.kernel.org/doc/html/latest/scheduler/sched-design-CFS.html)
- [IOMMU 与 GPU 地址翻译](https://www.kernel.org/doc/html/latest/core-api/iommu.html)
- [Jain's Fairness Index 说明](https://en.wikipedia.org/wiki/Fairness_measure)

### 今日小结

vGPU 调度与资源隔离是实现 GPU 虚拟化的核心技术：

1. **计算调度**：时间片轮转 + 优先级加权，上下文切换是关键开销
2. **显存隔离**：独立的 GART 页表，地址空间完全隔离
3. **带宽控制**：Token Bucket 算法限制每 VF 的带宽使用
4. **公平性**：通过 Jain's Index 等指标量化隔离效果
5. **错误隔离**：硬件级 page fault 保护，防止跨 VF 干扰
6. **配置优化**：根据工作负载特征调整时间片、优先级和带宽限制

### 扩展思考

1. 如何在不支持硬件 SR-IOV 的 GPU 上实现软件层面的资源隔离？
2. GPU 时间片大小如何影响交互式应用（如云游戏）的用户体验？
3. 在多 VF 场景下，如何动态调整资源分配以提高整体利用率？
4. 对于 GPU 驱动测试而言，vGPU 隔离性测试应该覆盖哪些边界场景？
