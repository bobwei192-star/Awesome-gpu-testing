# Day 392：AMDGPU 驱动参数调优实践

## 学习目标

- 掌握 amdgpu 内核模块参数体系
- 理解每个参数对性能和功能的影响
- 学会根据不同工作负载调优驱动参数
- 掌握参数持久化配置和运行时调整方法

## 知识详解

### 概念原理

#### AMDGPU 模块参数分类

amdgpu 驱动的所有模块参数分为以下几类：

```
amdgpu 模块参数体系:

┌─────────────────────────────────────────────────────────────┐
│                    amdgpu 模块参数                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 调试类 (Debug)                                          │
│     ┌─────────────────────────────────────────────────────┐ │
│     │ amdgpu_locked_rotation_vram                         │ │
│     │ amdgpu_lbpw                                        │ │
│     │ amdgpu_benchmark                                   │ │
│     │ amdgpu_testing                                     │ │
│     └─────────────────────────────────────────────────────┘ │
│                                                             │
│  2. 电源管理类 (Power Management)                            │
│     ┌─────────────────────────────────────────────────────┐ │
│     │ amdgpu_powerplay                                   │ │
│     │ amdgpu_dpm                                        │ │
│     │ amdgpu_dc                                         │ │
│     │ amdgpu_sched_jobs                                 │ │
│     │ amdgpu_sched_hang_limit                           │ │
│     └─────────────────────────────────────────────────────┘ │
│                                                             │
│  3. 内存管理类 (Memory Management)                           │
│     ┌─────────────────────────────────────────────────────┐ │
│     │ amdgpu_vram_limit                                  │ │
│     │ amdgpu_gtt_size                                   │ │
│     │ amdgpu_moverate                                   │ │
│     │ amdgpu_vis_vram_limit                             │ │
│     │ amdgpu_sg_display                                 │ │
│     └─────────────────────────────────────────────────────┘ │
│                                                             │
│  4. 调度类 (Scheduling)                                     │
│     ┌─────────────────────────────────────────────────────┐ │
│     │ amdgpu_sched_jobs                                  │ │
│     │ amdgpu_sched_hang_limit                            │ │
│     │ amdgpu_sched_timeout                               │ │
│     │ amdgpu_ring_mux                                    │ │
│     └─────────────────────────────────────────────────────┘ │
│                                                             │
│  5. 虚拟化类 (Virtualization)                                │
│     ┌─────────────────────────────────────────────────────┐ │
│     │ amdgpu_virt_mode                                   │ │
│     │ amdgpu_ignore_bad_page                             │ │
│     └─────────────────────────────────────────────────────┘ │
│                                                             │
│  6. 特性开关类 (Feature Flags)                               │
│     ┌─────────────────────────────────────────────────────┐ │
│     │ amdgpu_ucode_version                               │ │
│     │ amdgpu_gpu_recovery                                │ │
│     │ amdgpu_emu_mode                                    │ │
│     │ amdgpu_prim_buf_discard_enable                     │ │
│     └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

#### 完整参数清单

```c
// 内核源码: drivers/gpu/drm/amd/amdgpu/amdgpu_drv.c

// 参数定义表
module_param_named(testing, amdgpu_testing, int, 0644);
MODULE_PARM_DESC(testing, "测试模式 (-1=自动, 0=禁用, 1=启用)");

module_param_named(benchmark, amdgpu_benchmarking, int, 0444);
MODULE_PARM_DESC(benchmark, "启动时基准测试");

module_param_named(vram_limit, amdgpu_vram_limit, ulong, 0444);
MODULE_PARM_DESC(vram_limit, "VRAM 限制 (MB, 0=无限制)");

module_param_named(vis_vram_limit, amdgpu_vis_vram_limit, ulong, 0444);
MODULE_PARM_DESC(vis_vram_limit, "可视 VRAM 限制 (MB)");

module_param_named(gtt_size, amdgpu_gtt_size, ulong, 0444);
MODULE_PARM_DESC(gtt_size, "GTT 大小 (MB)");

module_param_named(moverate, amdgpu_moverate, int, 0444);
MODULE_PARM_DESC(moverate, "DMA 迁移速率 (MB/s)");

module_param_named(emu_mode, amdgpu_emu_mode, int, 0444);
MODULE_PARM_DESC(emu_mode, "模拟器模式");

module_param_named(gpu_recovery, amdgpu_gpu_recovery, int, 0444);
MODULE_PARM_DESC(gpu_recovery, "GPU 恢复机制 (0=禁用, 1=启用, 2=强制)");

module_param_named(sched_jobs, amdgpu_sched_jobs, int, 0444);
MODULE_PARM_DESC(sched_jobs, "调度器中最大作业数");

module_param_named(sched_hang_limit, amdgpu_sched_hang_limit, int, 0444);
MODULE_PARM_DESC(sched_hang_limit, "调度器挂起限制");

module_param_named(sched_timeout, amdgpu_sched_timeout, int, 0444);
MODULE_PARM_DESC(sched_timeout, "调度器超时 (ms)");

module_param_named(hw_i2c, amdgpu_hw_i2c, int, 0444);
MODULE_PARM_DESC(hw_i2c, "硬件 I2C (0=软件, 1=硬件)");

module_param_named(pcie_gen_cap, amdgpu_pcie_gen_cap, int, 0444);
MODULE_PARM_DESC(pcie_gen_cap, "PCIe 代际限制 (0=自动)");

module_param_named(pcie_lane_cap, amdgpu_pcie_lane_cap, int, 0444);
MODULE_PARM_DESC(pcie_lane_cap, "PCIe 通道限制 (0=自动)");

module_param_named(ip_block_mask, amdgpu_ip_block_mask, ulong, 0444);
MODULE_PARM_DESC(ip_block_mask, "IP 块掩码 (bitmask)");

module_param_named(virtual_display, amdgpu_virtual_display, int, 0444);
MODULE_PARM_DESC(virtual_display, "虚拟显示输出");

module_param_named(compute_preempt, amdgpu_compute_preempt, int, 0644);
MODULE_PARM_DESC(compute_preempt, "计算抢占模式");

module_param_named(prim_buf_discard_enable, amdgpu_prim_buf_discard_enable, int, 0444);
MODULE_PARM_DESC(prim_buf_discard_enable, "原始缓冲区丢弃启用");

module_param_named(num_kernel_pages, amdgpu_num_kernel_pages, uint, 0444);
MODULE_PARM_DESC(num_kernel_pages, "内核分页数量");

module_param_named(si_support, amdgpu_si_support, int, 0444);
MODULE_PARM_DESC(si_support, "SI 芯片支持");

module_param_named(cik_support, amdgpu_cik_support, int, 0444);
MODULE_PARM_DESC(cik_support, "CIK 芯片支持");

module_param_named(ignore_bad_page, amdgpu_ignore_bad_page, int, 0644);
MODULE_PARM_DESC(ignore_bad_page, "忽略坏页");

module_param_named(lockup_timeout, amdgpu_lockup_timeout, int, 0444);
MODULE_PARM_DESC(lockup_timeout, "锁定时超时 (ms)");

module_param_named(max_shared_memory, amdgpu_max_shared_memory, ulong, 0444);
MODULE_PARM_DESC(max_shared_memory, "最大共享内存 (KB)");

module_param_named(ras_mask, amdgpu_ras_mask, ulong, 0444);
MODULE_PARM_DESC(ras_mask, "RAS 特性掩码");

module_param_named(ras_enable, amdgpu_ras_enable, int, 0444);
MODULE_PARM_DESC(ras_enable, "RAS 启用");

module_param_named(smartshift_enable, amdgpu_smartshift_enable, int, 0644);
MODULE_PARM_DESC(smartshift_enable, "SmartShift 启用");

module_param_named(apu_prefer_single_disp, amdgpu_apu_prefer_single_disp, int, 0444);
MODULE_PARM_DESC(apu_prefer_single_disp, "APU 单显示偏好");

module_param_named(freesync_video, amdgpu_freesync_video, int, 0644);
MODULE_PARM_DESC(freesync_video, "FreeSync 视频模式");
```

#### 关键参数详解

##### amdgpu_gpu_recovery

```c
// 内核源码: drivers/gpu/drm/amd/amdgpu/amdgpu_device.c

int amdgpu_gpu_recovery = 1;  // 默认启用

/*
 * GPU 恢复机制:
 * 0 = 禁用 GPU 恢复
 *      - GPU 挂起时直接上报错误
 *      - 用于调试: 方便捕获完整 crash dump
 *      - 不适用于生产环境
 * 
 * 1 = 启用 GPU 恢复 (默认)
 *      - 检测到 GPU 挂起时尝试恢复
 *      - 恢复流程: 暂停→重置→恢复
 *      - 应用层可能感知到短暂停顿
 * 
 * 2 = 强制 GPU 恢复
 *      - 即使 GPU 没有挂起也尝试恢复
 *      - 用于测试恢复路径
 */

// 恢复流程
static int amdgpu_device_gpu_recover(struct amdgpu_device *adev) {
    // 阶段 1: 暂停所有引擎
    amdgpu_device_ip_suspend(adev);
    
    // 阶段 2: 硬件重置
    amdgpu_asic_reset(adev);
    
    // 阶段 3: 恢复 IP 块
    amdgpu_device_ip_resume_phase1(adev);
    amdgpu_device_ip_resume_phase2(adev);
    
    // 阶段 4: 恢复 GPU 调度器
    drm_sched_resubmit_jobs();
    
    return 0;
}
```

##### amdgpu_sched_jobs / amdgpu_sched_timeout

```
调度参数对性能的影响:

┌───────────────┬───────────┬──────────┬──────────────────┐
│    参数       │  典型值    │  效果    │    适用场景       │
├───────────────┼───────────┼──────────┼──────────────────┤
│ sched_jobs    │ 16-1024   │ 高=更多  │ 计算密集型: 256   │
│ (最大作业数)  │           │ 排队作业  │ 图形密集型: 64    │
├───────────────┼───────────┼──────────┼──────────────────┤
│ sched_hang    │ 0-10      │ 高=更多  │ 开发测试: 10      │
│ (挂起限制)    │           │ 容忍度   │ 生产环境: 2       │
├───────────────┼───────────┼──────────┼──────────────────┤
│ sched_timeout │ 100-10000 │ 超时后   │ AI训练: 10000     │
│ (ms)          │           │ 触发恢复 │ 游戏: 1000        │
└───────────────┴───────────┴──────────┴──────────────────┘

调度队列模型:

┌─────────────────────────────────────────────────────┐
│                  GPU 调度器                           │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ GFX 队列 │  │ Compute  │  │ SDMA 队列│           │
│  │ (环 0)   │  │ (环 1)   │  │ (环 2)   │           │
│  ├──────────┤  ├──────────┤  ├──────────┤           │
│  │jobs=64   │  │jobs=256  │  │jobs=128  │           │
│  │timeout=  │  │timeout=  │  │timeout=  │           │
│  │1000ms    │  │10000ms   │  │5000ms    │           │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
│       │             │             │                   │
│       └──────┬──────┴──────┬──────┘                   │
│              │             │                           │
│              ▼             ▼                           │
│       ┌────────────┐ ┌────────────┐                   │
│       │ HW Ring 0  │ │ HW Ring 1 │                   │
│       │ (Graphics) │ │ (Compute)  │                   │
│       └────────────┘ └────────────┘                   │
└─────────────────────────────────────────────────────┘
```

##### amdgpu_vram_limit / 内存限制

```
内存限制机制:

┌─────────────────────────────────────────────────────┐
│                  内存分配路径                          │
├─────────────────────────────────────────────────────┤
│                                                      │
│  应用程序 ──→ amdgpu_gem_create ──→ amdgpu_bo_create │
│                    │                                  │
│                    ▼                                  │
│            ┌─────────────────┐                        │
│            │ 检查 VRAM 限制  │                        │
│            │                 │                        │
│            │ amdgpu_vram_   │                        │
│            │ limit > total? │                        │
│            └───────┬─────────┘                        │
│                    │                                  │
│            ┌───────┴───────┐                          │
│            ▼               ▼                          │
│     ┌────────────┐  ┌────────────┐                    │
│     │ VRAM 不足  │  │ 分配 VRAM  │                    │
│     │ → GTT 回退 │  │ (正常)     │                    │
│     └────────────┘  └────────────┘                    │
│                                                      │
│  用途:                                                │
│  - 限制单个 GPU 的显存使用                             │
│  - 多 GPU 环境中的显存分配控制                          │
│  - 测试内存压力场景                                    │
│                                                      │
└─────────────────────────────────────────────────────┘
```

##### amdgpu_ip_block_mask

```
IP 块掩码定义:

bit 0:  IP 块 0  (GMC)
bit 1:  IP 块 1  (GFX)
bit 2:  IP 块 2  (SDMA)
bit 3:  IP 块 3  (PSP)
bit 4:  IP 块 4  (SMU)
bit 5:  IP 块 5  (DCE)
bit 6:  IP 块 6  (VCN)
bit 7:  IP 块 7  (HDP)
bit 8:  IP 块 8  (JPEG)
bit 9:  IP 块 9  (NBIO)
bit 10: IP 块 10 (IH)
bit 11: IP 块 11 (MP0)
bit 12: IP 块 12 (MP1)
...

用法:
  amdgpu_ip_block_mask=0x0000000F  → 只启用前 4 个 IP 块
  amdgpu_ip_block_mask=0xFFFFFFDF  → 禁用 DCE (bit 5)
  amdgpu_ip_block_mask=0xFFFFFFBF  → 禁用 VCN (bit 6)

调试用途:
  - 隔离特定 IP 块的问题
  - 在无显示环境下禁用 DCE
  - 测试无视频编解码器 (VCN) 的系统
```

##### amdgpu_pcie_gen_cap / amdgpu_pcie_lane_cap

```
PCIe 代际/通道限制:

┌─────────────────────────────────────────────────────┐
│  PCIe 代际和通道限制                                  │
├─────────────────────────────────────────────────────┤
│                                                      │
│  amdgpu_pcie_gen_cap:                                │
│    0 = 自动 (由 BIOS 和硬件协商)                      │
│    1 = 限制到 Gen1 (2.5 GT/s)                        │
│    2 = 限制到 Gen2 (5.0 GT/s)                        │
│    3 = 限制到 Gen3 (8.0 GT/s)                        │
│    4 = 限制到 Gen4 (16.0 GT/s)                       │
│    5 = 限制到 Gen5 (32.0 GT/s)                       │
│                                                      │
│  amdgpu_pcie_lane_cap:                               │
│    0 = 自动                                           │
│    1 = 限制到 x1                                      │
│    2 = 限制到 x2                                      │
│    4 = 限制到 x4                                      │
│    8 = 限制到 x8                                      │
│    16 = 限制到 x16                                    │
│                                                      │
│  用途:                                                │
│  - 测试不同 PCIe 带宽对性能的影响                       │
│  - 模拟 PCIe 链路问题                                 │
│  - 节能 (减少通道数)                                  │
│  - 兼容性测试                                          │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### 实践操作

#### 参数查看和调整工具

```bash
#!/bin/bash
# amdgpu 参数管理工具

echo "=============================================="
echo "  AMDGPU 驱动参数管理"
echo "=============================================="
echo ""

# 获取当前参数值
get_module_param() {
    local param_name=$1
    local param_file="/sys/module/amdgpu/parameters/${param_name}"
    
    if [ -f "${param_file}" ]; then
        echo "$(cat ${param_file})"
    else
        echo "N/A"
    fi
}

show_all_params() {
    echo "[1/6] 所有 amdgpu 模块参数"
    echo "-------------------------"
    
    for param_file in /sys/module/amdgpu/parameters/*; do
        param_name=$(basename ${param_file})
        param_val=$(cat ${param_file} 2>/dev/null || echo "N/A")
        printf "  %-35s = %s\n" "${param_name}" "${param_val}"
    done
}

show_power_params() {
    echo ""
    echo "[2/6] 电源管理参数"
    echo "-----------------"
    
    for gpu_dir in /sys/class/drm/card?; do
        if [ -e "${gpu_dir}" ]; then
            echo "  GPU ${gpu_dir##*/}:"
            
            # 电源状态
            if [ -f "${gpu_dir}/device/power_dpm_force_performance_level" ]; then
                echo "    性能等级: $(cat ${gpu_dir}/device/power_dpm_force_performance_level)"
            fi
            
            # DPM 状态
            if [ -f "${gpu_dir}/device/pp_dpm_sclk" ]; then
                echo "    实际 DPM 状态 (sclk):"
                cat "${gpu_dir}/device/pp_dpm_sclk" | head -5 | while read line; do
                    echo "      ${line}"
                done
            fi
            
            # 功耗上限
            if [ -f "${gpu_dir}/device/ppt_capability" ]; then
                echo "    功耗上限: $(cat ${gpu_dir}/device/ppt_capability) W"
            fi
        fi
    done
}

show_memory_params() {
    echo ""
    echo "[3/6] 内存管理参数"
    echo "-----------------"
    
    echo "  模块参数:"
    echo "    vram_limit:     $(get_module_param vram_limit) MB"
    echo "    vis_vram_limit: $(get_module_param vis_vram_limit) MB"
    echo "    gtt_size:       $(get_module_param gtt_size) MB"
    echo "    moverate:       $(get_module_param moverate) MB/s"
    echo ""
    
    for gpu_dir in /sys/class/drm/card?; do
        if [ -e "${gpu_dir}" ]; then
            echo "  GPU ${gpu_dir##*/} 内存状态:"
            
            for mem_info in mem_info_vram_total mem_info_vram_used \
                            mem_info_gtt_total mem_info_gtt_used \
                            mem_info_visible_vram_total mem_info_visible_vram_used; do
                if [ -f "${gpu_dir}/device/${mem_info}" ]; then
                    val=$(cat "${gpu_dir}/device/${mem_info}")
                    printf "    %-35s = %d MB\n" "${mem_info}:" "$((val / 1048576))"
                fi
            done
            echo ""
        fi
    done
}

show_scheduler_params() {
    echo ""
    echo "[4/6] 调度器参数"
    echo "--------------"
    
    echo "  模块参数:"
    echo "    sched_jobs:      $(get_module_param sched_jobs)"
    echo "    sched_hang_limit: $(get_module_param sched_hang_limit)"
    echo "    sched_timeout:   $(get_module_param sched_timeout) ms"
    
    echo ""
    echo "  DRM 调度器状态:"
    for gpu_dir in /sys/class/drm/card?; do
        if [ -d "${gpu_dir}/device/drm/card${gpu_dir##*/card}" ]; then
            # 查看调度器队列
            for ring in ${gpu_dir}/device/drm/card${gpu_dir##*/card}/gpu_sched/*; do
                if [ -d "${ring}" ]; then
                    ring_name=$(basename ${ring})
                    echo "    ${ring_name}:"
                    
                    for attr in "num_jobs" "hung" "total_entities" "sched_running"; do
                        val_file="${ring}/${attr}"
                        if [ -f "${val_file}" ]; then
                            echo "      ${attr} = $(cat ${val_file})"
                        fi
                    done
                fi
            done
        fi
    done
}

show_feature_params() {
    echo ""
    echo "[5/6] 特性开关参数"
    echo "-----------------"
    
    echo "  GPU 恢复:     $(get_module_param gpu_recovery)"
    echo "  RAS 启用:     $(get_module_param ras_enable)"
    echo "  RAS 掩码:     0x$(printf '%x' $(get_module_param ras_mask))"
    echo "  HW I2C:       $(get_module_param hw_i2c)"
    echo "  FreeSync:     $(get_module_param freesync_video)"
    echo "  SI 支持:      $(get_module_param si_support)"
    echo "  CIK 支持:     $(get_module_param cik_support)"
    echo "  IP 块掩码:    0x$(printf '%lx' $(get_module_param ip_block_mask))"
}

show_pcie_params() {
    echo ""
    echo "[6/6] PCIe 参数"
    echo "--------------"
    
    echo "  模块参数:"
    echo "    pcie_gen_cap:  $(get_module_param pcie_gen_cap)"
    echo "    pcie_lane_cap: $(get_module_param pcie_lane_cap)"
    
    echo ""
    echo "  实际 PCIe 状态:"
    for gpu_dir in /sys/class/drm/card?; do
        if [ -e "${gpu_dir}" ]; then
            bdf=$(basename $(readlink -f ${gpu_dir}/device) 2>/dev/null || echo "N/A")
            
            if [ -f "${gpu_dir}/device/current_link_speed" ]; then
                speed=$(cat ${gpu_dir}/device/current_link_speed)
                width=$(cat ${gpu_dir}/device/current_link_width)
                echo "  ${gpu_dir##*/}: ${bdf}: ${speed} x${width}"
            fi
            
            if [ -f "${gpu_dir}/device/max_link_speed" ]; then
                max_speed=$(cat ${gpu_dir}/device/max_link_speed)
                max_width=$(cat ${gpu_dir}/device/max_link_width)
                echo "           最大: ${max_speed} x${max_width}"
            fi
        fi
    done
}

# 主流程
case "$1" in
    all)       show_all_params ;;
    power)     show_power_params ;;
    memory)    show_memory_params ;;
    sched)     show_scheduler_params ;;
    feature)   show_feature_params ;;
    pcie)      show_pcie_params ;;
    *)
        echo "用法: $0 {all|power|memory|sched|feature|pcie}"
        echo ""
        echo "示例:"
        echo "  $0 all      - 显示所有参数"
        echo "  $0 power    - 电源管理参数"
        echo "  $0 memory   - 内存管理参数"
        echo "  $0 sched    - 调度器参数"
        echo "  $0 feature  - 特性开关参数"
        echo "  $0 pcie     - PCIe 参数"
        ;;
esac
```

#### 按工作负载调优配置

```bash
#!/bin/bash
# 根据工作负载类型应用 amdgpu 参数

# 工作负载类型
WORKLOAD_GAMING=1
WORKLOAD_COMPUTE=2
WORKLOAD_AI_TRAINING=3
WORKLOAD_DESKTOP=4
WORKLOAD_DEBUG=5

# 配置名
declare -A PROFILE_NAMES
PROFILE_NAMES[$WORKLOAD_GAMING]="游戏优化"
PROFILE_NAMES[$WORKLOAD_COMPUTE]="计算密集"
PROFILE_NAMES[$WORKLOAD_AI_TRAINING]="AI 训练"
PROFILE_NAMES[$WORKLOAD_DESKTOP]="桌面办公"
PROFILE_NAMES[$WORKLOAD_DEBUG]="调试模式"

apply_gaming_profile() {
    echo "  应用游戏优化配置..."
    
    # 低延迟渲染
    echo low > /sys/class/drm/card0/device/power_dpm_force_performance_level
    
    # 启用 FreeSync
    echo Y > /sys/module/amdgpu/parameters/freesync_video 2>/dev/null
    
    # 减少调度超时 (快速检测卡顿)
    echo 1000 > /sys/module/amdgpu/parameters/sched_timeout
    
    # 减少队列深度 (降低输入延迟)
    echo 64 > /sys/module/amdgpu/parameters/sched_jobs
    
    # 小缓冲区快速移动
    echo 500 > /sys/module/amdgpu/parameters/moverate
    
    echo "  ✅ 游戏优化已应用"
}

apply_compute_profile() {
    echo "  应用计算优化配置..."
    
    # 最大性能
    echo high > /sys/class/drm/card0/device/power_dpm_force_performance_level
    
    # 长超时 (计算任务运行时间长)
    echo 10000 > /sys/module/amdgpu/parameters/sched_timeout
    
    # 大队列深度 (批量提交)
    echo 256 > /sys/module/amdgpu/parameters/sched_jobs
    
    # 高迁移速率
    echo 2000 > /sys/module/amdgpu/parameters/moverate
    
    # 禁用 GPU 恢复 (计算任务不支持恢复)
    echo 0 > /sys/module/amdgpu/parameters/gpu_recovery
    
    echo "  ✅ 计算优化已应用"
}

apply_ai_training_profile() {
    echo "  应用 AI 训练优化配置..."
    
    # 固定最高性能
    echo high > /sys/class/drm/card0/device/power_dpm_force_performance_level
    
    # 非常长的超时 (训练任务可能数小时)
    echo 30000 > /sys/module/amdgpu/parameters/sched_timeout
    
    # 最大队列深度
    if [ $(getconf _NPROCESSORS_ONLN) -gt 16 ]; then
        echo 512 > /sys/module/amdgpu/parameters/sched_jobs
    else
        echo 256 > /sys/module/amdgpu/parameters/sched_jobs
    fi
    
    # 最大迁移速率
    echo 4000 > /sys/module/amdgpu/parameters/moverate
    
    # 启用 RAS (检测显存错误)
    echo 1 > /sys/module/amdgpu/parameters/ras_enable
    
    # 分配指定显存大小 (如果有约束)
    # echo 32768 > /sys/module/amdgpu/parameters/vram_limit
    
    echo "  ✅ AI 训练优化已应用"
}

apply_desktop_profile() {
    echo "  应用桌面办公优化配置..."
    
    # 自动电源管理 (平衡模式)
    echo auto > /sys/class/drm/card0/device/power_dpm_force_performance_level
    
    # 启用 GPU 恢复
    echo 1 > /sys/module/amdgpu/parameters/gpu_recovery
    
    # 标准超时
    echo 2000 > /sys/module/amdgpu/parameters/sched_timeout
    
    # 中等队列
    echo 64 > /sys/module/amdgpu/parameters/sched_jobs
    
    # 启用 FreeSync
    echo Y > /sys/module/amdgpu/parameters/freesync_video 2>/dev/null
    
    echo "  ✅ 桌面办公优化已应用"
}

apply_debug_profile() {
    echo "  应用调试优化配置..."
    
    # 禁用 GPU 恢复 (捕获完整信息)
    echo 0 > /sys/module/amdgpu/parameters/gpu_recovery
    
    # 短超时 (快速发现问题)
    echo 500 > /sys/module/amdgpu/parameters/sched_timeout
    
    # 小队列
    echo 32 > /sys/module/amdgpu/parameters/sched_jobs
    
    # 禁用电源管理 (稳定状态)
    echo high > /sys/class/drm/card0/device/power_dpm_force_performance_level
    
    # 启用 RAS
    echo 1 > /sys/module/amdgpu/parameters/ras_enable
    
    # 完整 RAS 掩码
    echo 0xFFFFFFFF > /sys/module/amdgpu/parameters/ras_mask
    
    echo "  ✅ 调试模式已应用 (禁用 GPU 恢复)"
}

apply_profile() {
    local profile=$1
    
    echo ""
    echo "应用配置: ${PROFILE_NAMES[$profile]}"
    echo "======================================"
    
    # 检查 GPU 是否存在
    if [ ! -d /sys/class/drm/card0 ]; then
        echo "  ❌ 未找到 GPU 设备"
        return 1
    fi
    
    # 记录当前配置 (备份)
    echo "  备份当前配置..."
    mkdir -p /tmp/amdgpu_profile_backup
    for param in sched_jobs sched_timeout sched_hang_limit \
                 gpu_recovery vram_limit moverate ras_enable ras_mask; do
        if [ -f "/sys/module/amdgpu/parameters/${param}" ]; then
            cp "/sys/module/amdgpu/parameters/${param}" \
               "/tmp/amdgpu_profile_backup/${param}" 2>/dev/null
        fi
    done
    
    case $profile in
        $WORKLOAD_GAMING)
            apply_gaming_profile
            ;;
        $WORKLOAD_COMPUTE)
            apply_compute_profile
            ;;
        $WORKLOAD_AI_TRAINING)
            apply_ai_training_profile
            ;;
        $WORKLOAD_DESKTOP)
            apply_desktop_profile
            ;;
        $WORKLOAD_DEBUG)
            apply_debug_profile
            ;;
    esac
    
    echo ""
    echo "  当前参数:"
    for param in sched_jobs sched_timeout gpu_recovery moverate; do
        if [ -f "/sys/module/amdgpu/parameters/${param}" ]; then
            echo "    ${param} = $(cat /sys/module/amdgpu/parameters/${param})"
        fi
    done
    
    echo ""
    echo "配置完成: ${PROFILE_NAMES[$profile]}"
}

restore_profile() {
    echo "恢复备份配置..."
    if [ -d /tmp/amdgpu_profile_backup ]; then
        for param_file in /tmp/amdgpu_profile_backup/*; do
            if [ -f "${param_file}" ]; then
                param_name=$(basename ${param_file})
                param_val=$(cat ${param_file})
                dest="/sys/module/amdgpu/parameters/${param_name}"
                if [ -f "${dest}" ]; then
                    echo "${param_val}" > "${dest}" 2>/dev/null || \
                        echo "    ❌ 无法恢复 ${param_name} (只读参数)"
                fi
            fi
        done
        echo "  ✅ 配置已恢复"
    else
        echo "  ❌ 无备份配置"
    fi
}

# 主流程
echo ""
echo "AMDGPU 工作负载配置工具"
echo "========================"
echo ""

case "$1" in
    gaming)     apply_profile $WORKLOAD_GAMING ;;
    compute)    apply_profile $WORKLOAD_COMPUTE ;;
    ai)         apply_profile $WORKLOAD_AI_TRAINING ;;
    desktop)    apply_profile $WORKLOAD_DESKTOP ;;
    debug)      apply_profile $WORKLOAD_DEBUG ;;
    restore)    restore_profile ;;
    *)
        echo "用法: $0 {gaming|compute|ai|desktop|debug|restore}"
        echo ""
        echo "  配置说明:"
        echo "    gaming  - 低延迟渲染, FreeSync 优化"
        echo "    compute - 计算密集型, 长超时, 大队列"
        echo "    ai      - AI 训练优化, RAS 监控, 高性能"
        echo "    desktop - 平衡模式, 省电"
        echo "    debug   - 禁用 GPU 恢复, 短超时, 完整日志"
        echo "    restore - 恢复上次配置"
        ;;
esac
```

#### 持久化配置 (modprobe.d)

```bash
# 在 /etc/modprobe.d/amdgpu.conf 中配置

# 基本配置
options amdgpu gpu_recovery=1
options amdgpu sched_jobs=64
options amdgpu sched_hang_limit=2

# AI 训练服务器配置
# options amdgpu gpu_recovery=0
# options amdgpu ras_enable=1
# options amdgpu sched_timeout=10000
# options amdgpu sched_jobs=256

# 调试配置
# options amdgpu gpu_recovery=0
# options amdgpu sched_timeout=500
# options amdgpu ip_block_mask=0xFFFFFFDF  # 禁用 DCE

# 内存限制
# options amdgpu vram_limit=8192  # 8GB 显存限制

# PCIe 限制测试
# options amdgpu pcie_gen_cap=3  # 限制到 Gen3
# options amdgpu pcie_lane_cap=8  # 限制到 x8

# 旧版本 GPU 支持
# options amdgpu si_support=1
# options amdgpu cik_support=1
```

#### 运行时参数调整测试

```python
#!/usr/bin/env python3
# AMDGPU 参数调优测试框架

import os
import time
import json
import subprocess
from typing import Dict, List, Tuple

class AMDGPUParamTuner:
    """AMDGPU 参数调优测试"""
    
    def __init__(self):
        self.sysfs_base = "/sys/module/amdgpu/parameters"
        self.results = {}
    
    def _read_param(self, name: str) -> str:
        """读取模块参数"""
        path = f"{self.sysfs_base}/{name}"
        try:
            with open(path, 'r') as f:
                return f.read().strip()
        except:
            return "N/A"
    
    def _write_param(self, name: str, value: str) -> bool:
        """写入模块参数 (需要 root)"""
        path = f"{self.sysfs_base}/{name}"
        try:
            with open(path, 'w') as f:
                f.write(value)
            return True
        except PermissionError:
            print(f"  ⚠️ 无权限写入 {name} (需要 root)")
            return False
        except:
            return False
    
    def _run_benchmark(self, bench_type: str = "compute") -> Dict:
        """运行性能基准测试"""
        result = {
            "throughput": 0.0,
            "latency": 0.0,
            "power": 0.0,
        }
        
        # 使用 rocminfo 获取 GPU 信息
        try:
            rocminfo = subprocess.run(
                ["rocminfo"], capture_output=True, text=True, timeout=30
            )
        except:
            pass
        
        # 使用性能计数器 (实际测试需要 ROCM 工具)
        try:
            # 模拟性能数据
            result = {
                "throughput": 100.0,  # GB/s
                "latency": 500.0,     # ns
                "power": 250.0,       # W
            }
        except:
            pass
        
        return result
    
    def test_sched_jobs_impact(self) -> List[Dict]:
        """测试 sched_jobs 参数对性能的影响"""
        print("\nsched_jobs 参数调优测试")
        print("=" * 50)
        
        test_values = [16, 32, 64, 128, 256, 512]
        results = []
        
        original = int(self._read_param("sched_jobs"))
        
        for val in test_values:
            print(f"  测试 sched_jobs={val}...")
            
            if self._write_param("sched_jobs", str(val)):
                time.sleep(1)  # 等待生效
                
                bench = self._run_benchmark()
                bench["param_value"] = val
                results.append(bench)
                
                print(f"    throughput={bench['throughput']:.1f}, "
                      f"latency={bench['latency']:.1f}")
        
        # 恢复原始值
        self._write_param("sched_jobs", str(original))
        
        return results
    
    def test_gpu_recovery_mode(self) -> Dict:
        """测试 GPU 恢复模式"""
        print("\nGPU 恢复模式测试")
        print("=" * 50)
        
        modes = {0: "禁用", 1: "启用", 2: "强制"}
        results = {}
        
        original = int(self._read_param("gpu_recovery"))
        
        for mode, desc in modes.items():
            print(f"  测试 gpu_recovery={mode} ({desc})...")
            
            if self._write_param("gpu_recovery", str(mode)):
                time.sleep(1)
                
                # 模拟 GPU 挂起和恢复
                # 实际测试需要注入 GPU 错误
                
                results[mode] = {
                    "mode": mode,
                    "description": desc,
                    "status": "ready"
                }
        
        self._write_param("gpu_recovery", str(original))
        
        return results
    
    def test_vram_limit_impact(self) -> List[Dict]:
        """测试 VRAM 限制对应用的影响"""
        print("\nVRAM 限制测试")
        print("=" * 50)
        
        # 检测实际显存
        try:
            vram_total = 0
            for gpu_dir in os.listdir("/sys/class/drm"):
                if gpu_dir.startswith("card"):
                    vram_file = f"/sys/class/drm/{gpu_dir}/device/mem_info_vram_total"
                    if os.path.exists(vram_file):
                        with open(vram_file) as f:
                            vram_total = int(f.read()) // (1024 * 1024)  # MB
                        break
        except:
            vram_total = 16384  # 默认 16GB
        
        test_limits = [
            0,                      # 无限制
            vram_total // 2,        # 50%
            vram_total // 4,        # 25%
        ]
        
        results = []
        original = int(self._read_param("vram_limit"))
        
        for limit in test_limits:
            print(f"  测试 vram_limit={limit} MB...")
            
            if self._write_param("vram_limit", str(limit)):
                time.sleep(2)
                
                # 尝试分配显存
                try:
                    alloc_test = subprocess.run(
                        ["rocm-smi", "--showmeminfo"],
                        capture_output=True, text=True, timeout=10
                    )
                except:
                    pass
                
                results.append({
                    "limit_mb": limit,
                    "status": "applied"
                })
        
        self._write_param("vram_limit", str(original))
        
        return results
    
    def test_pcie_limit_impact(self) -> List[Dict]:
        """测试 PCIe 限制对性能的影响"""
        print("\nPCIe 限制测试")
        print("=" * 50)
        
        gen_values = [0, 1, 2, 3, 4]
        lane_values = [0, 1, 4, 8, 16]
        
        results = []
        original_gen = int(self._read_param("pcie_gen_cap"))
        original_lane = int(self._read_param("pcie_lane_cap"))
        
        for gen in gen_values:
            self._write_param("pcie_gen_cap", str(gen))
            
            for lane in lane_values:
                self._write_param("pcie_lane_cap", str(lane))
                
                print(f"  PCIe Gen{gen} x{lane}...")
                time.sleep(3)  # 等待链路重新协商
                
                # 获取实际协商速度
                try:
                    for gpu_dir in os.listdir("/sys/class/drm"):
                        if gpu_dir.startswith("card"):
                            speed_file = f"/sys/class/drm/{gpu_dir}/device/current_link_speed"
                            width_file = f"/sys/class/drm/{gpu_dir}/device/current_link_width"
                            if os.path.exists(speed_file) and os.path.exists(width_file):
                                with open(speed_file) as f:
                                    speed = f.read().strip()
                                with open(width_file) as f:
                                    width = f.read().strip()
                                print(f"    协商: {speed} x{width}")
                    except:
                        pass
                    
                    results.append({
                        "gen": gen,
                        "lane": lane,
                        "negotiated_speed": speed,
                        "negotiated_width": width,
                    })
                    
                except:
                    pass
        
        self._write_param("pcie_gen_cap", str(original_gen))
        self._write_param("pcie_lane_cap", str(original_lane))
        
        return results
    
    def tune_for_workload(self, workload: str) -> Dict:
        """自动调优特定工作负载"""
        print(f"\n自动调优: {workload}")
        print("=" * 50)
        
        configs = {
            "gaming": {
                "power_dpm_force_performance_level": "low",
                "sched_timeout": "1000",
                "sched_jobs": "64",
                "freesync_video": "Y",
            },
            "compute": {
                "power_dpm_force_performance_level": "high",
                "sched_timeout": "10000",
                "sched_jobs": "256",
                "gpu_recovery": "0",
            },
            "ai_training": {
                "power_dpm_force_performance_level": "high",
                "sched_timeout": "30000",
                "sched_jobs": "512",
                "gpu_recovery": "0",
                "ras_enable": "1",
            },
            "desktop": {
                "power_dpm_force_performance_level": "auto",
                "sched_timeout": "2000",
                "sched_jobs": "64",
                "gpu_recovery": "1",
            },
        }
        
        if workload not in configs:
            print(f"  未知工作负载: {workload}")
            return {}
        
        applied = {}
        config = configs[workload]
        
        for param, value in config.items():
            # 检查是模块参数还是电源管理参数
            if param.startswith("power_"):
                # 电源管理参数 (per GPU)
                for gpu_dir in os.listdir("/sys/class/drm"):
                    if gpu_dir.startswith("card"):
                        path = f"/sys/class/drm/{gpu_dir}/device/{param}"
                        try:
                            with open(path, 'w') as f:
                                f.write(value)
                            applied[param] = value
                        except:
                            pass
            else:
                # 模块参数
                if self._write_param(param, value):
                    applied[param] = value
        
        self.results[workload] = applied
        return applied
    
    def generate_report(self, output_file: str = "amdgpu_tuning_report.json"):
        """生成调优报告"""
        
        report = {
            "system_info": self._get_system_info(),
            "tests": self.results,
            "recommendations": self._generate_recommendations()
        }
        
        with open(output_file, 'w') as f:
            json.dump(report, f, indent=2)
        
        print(f"\n报告已生成: {output_file}")
        
        return report
    
    def _get_system_info(self) -> Dict:
        """获取系统信息"""
        info = {}
        
        # GPU 信息
        gpus = []
        for gpu_dir in os.listdir("/sys/class/drm"):
            if gpu_dir.startswith("card"):
                try:
                    model = ""
                    vram = 0
                    
                    model_file = f"/sys/class/drm/{gpu_dir}/device/product_name"
                    vram_file = f"/sys/class/drm/{gpu_dir}/device/mem_info_vram_total"
                    
                    if os.path.exists(model_file):
                        with open(model_file) as f:
                            model = f.read().strip()
                    if os.path.exists(vram_file):
                        with open(vram_file) as f:
                            vram = int(f.read()) // (1024 * 1024)
                    
                    gpus.append({"name": model, "vram_mb": vram})
                except:
                    pass
        
        info["gpus"] = gpus
        
        # 当前参数
        params = {}
        if os.path.exists(self.sysfs_base):
            for param_file in os.listdir(self.sysfs_base):
                val = self._read_param(param_file)
                if val != "N/A":
                    params[param_file] = val
        
        info["current_params"] = params
        
        return info
    
    def _generate_recommendations(self) -> List[str]:
        """生成推荐配置"""
        recs = []
        
        # 基于 GPU 数量的建议
        gpu_count = len(self._get_system_info().get("gpus", []))
        if gpu_count > 1:
            recs.append("多 GPU 环境建议启用 XGMI/P2P 以获得最佳性能")
            recs.append("考虑使用 amdgpu_vram_limit 控制单个 GPU 的显存分配")
        
        # 基于参数的建议
        current_jobs = int(self._read_param("sched_jobs"))
        if current_jobs < 64:
            recs.append("sched_jobs 过低，建议至少设置为 64 以提高吞吐量")
        
        current_recovery = int(self._read_param("gpu_recovery"))
        if current_recovery == 0:
            recs.append("GPU 恢复已禁用，生产环境建议启用 (gpu_recovery=1)")
        
        return recs

# 命令行接口
if __name__ == "__main__":
    tuner = AMDGPUParamTuner()
    
    import sys
    if len(sys.argv) > 1:
        cmd = sys.argv[1]
        if cmd == "tune" and len(sys.argv) > 2:
            tuner.tune_for_workload(sys.argv[2])
        elif cmd == "test-sched":
            tuner.test_sched_jobs_impact()
        elif cmd == "test-pcie":
            tuner.test_pcie_limit_impact()
        elif cmd == "report":
            tuner.generate_report()
        else:
            print(f"用法: {sys.argv[0]} {{tune|test-sched|test-pcie|report}}")
    else:
        # 默认执行完整测试
        tuner.test_sched_jobs_impact()
        tuner.test_pcie_limit_impact()
        tuner.generate_report()
```

### 案例分析

#### 案例一：GPU 频繁超时恢复

**问题现象：**
```
AI 训练任务运行时，dmesg 频繁出现:
[  123.456] amdgpu: RING gfx timeout (last fence seq=0x1234)
[  124.567] amdgpu: Failed to suspend process 0x5678
[  125.678] amdgpu: GPU recovery begins
[  130.123] amdgpu: GPU recovery succeeded

每 10-15 分钟触发一次恢复，训练效率极低
```

**诊断过程：**
```bash
# 1. 查看当前调度器参数
cat /sys/module/amdgpu/parameters/sched_timeout
# 1000 (ms) - 默认 1 秒超时

cat /sys/module/amdgpu/parameters/sched_hang_limit
# 2 - 默认 2 次挂起触发恢复

# 2. 检查任务执行时间
# 计算任务实际执行时间长，频繁超时是因为训练迭代超过 1 秒
# 而默认 sched_timeout=1000ms

# 3. 检查 GPU 负载
cat /sys/class/drm/card0/device/gpu_busy_percent
# 98% - GPU 接近满载
```

**解决方案：**
```bash
# 方案 1: 增加调度超时
echo 10000 > /sys/module/amdgpu/parameters/sched_timeout
# 从 1 秒增加到 10 秒

# 方案 2: 增加挂起限制
echo 5 > /sys/module/amdgpu/parameters/sched_hang_limit

# 方案 3: 禁用 GPU 恢复 (如果确认不是真正的 GPU 故障)
echo 0 > /sys/module/amdgpu/parameters/gpu_recovery

# 持久化配置
cat > /etc/modprobe.d/amdgpu-ai.conf << 'EOF'
options amdgpu sched_timeout=30000
options amdgpu sched_jobs=256
options amdgpu sched_hang_limit=10
options amdgpu gpu_recovery=0
EOF

# 应用后效果
# 超时消失，训练效率恢复到 100%
```

#### 案例二：PCIe 带宽限制导致性能下降

**问题现象：**
```
4 GPU 系统，某个 GPU 性能明显低于其他
ROCM 带宽测试显示:
  GPU 0: 31.5 GB/s (预期 Gen4 x16)
  GPU 1: 31.2 GB/s
  GPU 2: 32.0 GB/s
  GPU 3: 8.1 GB/s  ← 性能只有 25%
```

**诊断过程：**
```bash
# 1. 检查 PCIe 链路状态
cat /sys/class/drm/card3/device/current_link_speed
# 2.5 GT/s (Gen1)  ← 降速了!

cat /sys/class/drm/card3/device/current_link_width
# x16 (通道数正常)

# 2. 检查最大支持
cat /sys/class/drm/card3/device/max_link_speed
# 16.0 GT/s (Gen4)

# 3. 检查 PCIe 错误计数
cat /sys/bus/pci/devices/0000:06:00.0/aer_dev_correctable
# Correctable Error Count: 8743  ← 大量可纠正错误
```

**解决方案：**
```bash
# 方案 1: 检查 PCIe 连接器
# 重新插拔 GPU，清洁金手指

# 方案 2: 检查温度
cat /sys/class/drm/card3/device/hwmon/hwmon*/temp1_input
# 85000 (85°C) - 检查是否温度过高导致降速

# 方案 3: 强制 PCIe 协商 (临时)
echo 4 > /sys/module/amdgpu/parameters/pcie_gen_cap
echo 16 > /sys/module/amdgpu/parameters/pcie_lane_cap
# 需要重新绑定驱动或重启

# 方案 4: 检查 amdgpu 模块参数
cat /sys/module/amdgpu/parameters/pcie_gen_cap
# 如果被设置为 1 或 2，改为 0 (自动)

# 验证修复
cat /sys/class/drm/card3/device/current_link_speed
# 16.0 GT/s (Gen4) - 恢复正常
```

### 相关链接

- [AMDGPU 内核参数文档](https://www.kernel.org/doc/html/latest/gpu/amdgpu.html)
- [AMDGPU 驱动参数源码](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/amdgpu/amdgpu_drv.c)
- [ROCm 性能调优指南](https://rocm.docs.amd.com/en/latest/how-to/tuning-guides/)
- [Linux modprobe.d 文档](https://man7.org/linux/man-pages/man5/modprobe.d.5.html)

### 今日小结

AMDGPU 驱动参数调优是 GPU 测试开发的关键技能：

1. **参数分类**: 调试/电源/内存/调度/特性和 PCIe 六大类
2. **关键参数**: gpu_recovery (恢复), sched_jobs/timeout (调度), vram_limit (内存), pcie_gen_cap (PCIe)
3. **工作负载适配**: 游戏（低延迟）、计算（高吞吐）、AI 训练（长超时+RAS）
4. **调优方法**: 运行时 sysfs 调整、持久化 modprobe.d 配置、性能基准测试
5. **调试技巧**: 禁用恢复捕获完整信息、IP 块掩码隔离问题、PCIe 限制模拟故障
6. **持久化配置**: /etc/modprobe.d/amdgpu.conf 保存优化配置

### 扩展思考

1. 如何设计一个自动化参数调优系统，能在不同工作负载间动态切换参数？
2. 在虚拟化环境中（SR-IOV VF），哪些模块参数对 VF 有效，哪些只对 PF 生效？
3. 如何评估特定参数修改对功耗和性能的综合影响？
4. amdgpu_ip_block_mask 参数在驱动调试中有哪些高级用法？
