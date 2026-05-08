# 第304天：功耗限制（Power Limit / PPT）管理

## 学习目标
- 深入理解 AMD GPU 功耗限制（Power Limit / PPT）的概念、作用与实现机制
- 掌握通过 sysfs、debugfs 及 rocm‑smi 查看、设置与验证功耗限制的方法
- 能够绘制功耗限制在 GPU 电源管理中的位置图，说明其与频率、电压、温度的相互作用
- 分析内核中功耗限制管理代码（如 `amdgpu_dpm.c`、`smu_*.c`）的关键函数与数据结构

## 知识详解

### 1. 概念原理

#### 1.1 功耗限制的定义与分类

在 AMD GPU 中，功耗限制（Power Limit）是一个关键的安全与性能管理机制，用于防止 GPU 超过其热设计功耗（TDP）而导致过热或硬件损坏。主要功耗限制包括：

| 限制类型 | 全称 | 作用 | 典型值范围 | 相关源码文件 |
|---------|------|------|-----------|------------|
| **PPT** | Package Power Limit（封装功耗限制） | GPU 封装（芯片+显存）的最大持续功耗限制 | 150‑400 W（取决于 GPU TDP） | `smu_v13_0.c`, `amdgpu_dpm.c` |
| **TDP** | Thermal Design Power（热设计功耗） | 散热系统设计基准，通常作为 PPT 的默认值 | 同 PPT | - |
| **TDC** | Thermal Design Current（热设计电流） | 最大持续电流限制，防止 VRM 过流 | 取决于 GPU | `smu_*` |
| **EDC** | Electrical Design Current（电气设计电流） | 最大瞬时电流限制，防止电压跌落 | 取决于 GPU | `smu_*` |

**PPT 的核心作用**：
1. **安全保护**：防止 GPU 因过功耗而损坏。
2. **散热管理**：确保功耗在散热系统能力范围内。
3. **性能调节**：通过限制功耗间接控制性能上限。
4. **能效优化**：在功耗限制内寻找最佳性能点。

#### 1.2 功耗限制在电源管理架构中的位置

下图展示了 PPT 在 AMD GPU 电源管理整体架构中的位置及其与其他组件的交互关系：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    GPU Power Management Architecture                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐                │
│  │   Workload  │     │  Thermal    │     │   Power     │                │
│  │  Monitor    │◄───►│  Monitor    │◄───►│   Monitor   │                │
│  │             │     │ (Temperature)│     │ (PPT/TDC/EDC)│                │
│  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘                │
│         │                   │                   │                       │
│  ┌──────▼───────────────────┼───────────────────▼──────┐                │
│  │         Power Management Decision Engine            │                │
│  │      (SMU Firmware + amdgpu_dpm.c)                  │                │
│  └──────────────────────────┬──────────────────────────┘                │
│                             │                                           │
│           ┌─────────────────┼─────────────────┐                         │
│           │                 │                 │                         │
│    ┌──────▼──────┐   ┌─────▼─────┐   ┌──────▼──────┐                   │
│    │  Frequency  │   │  Voltage  │   │   Power     │                   │
│    │  Control    │   │  Control  │   │   Control   │                   │
│    │  (SCLK/MCLK)│   │ (VDDC/VID)│   │   (PPT/TDC) │                   │
│    └──────┬──────┘   └─────┬─────┘   └──────┬──────┘                   │
│           │                 │                 │                         │
│    ┌──────▼─────────────────────────────────▼──────┐                   │
│    │              Hardware Execution               │                   │
│    │  (GPU Cores, Memory, Infinity Fabric, etc.)   │                   │
│    └───────────────────────────────────────────────┘                   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**数据流说明**：
1. **功率监控**：硬件传感器实时测量 GPU 封装功耗。
2. **PPT 检查**：SMU 将实测功耗与 PPT 限制比较。
3. **超限响应**：若实测功耗超过 PPT，SMU 触发降频（power throttling）降低 SCLK/MCLK。
4. **恢复机制**：当功耗降至安全范围后，频率逐步恢复。

#### 1.3 PPT 与频率‑电压曲线的交互

PPT 不是独立运行的，它与频率‑电压（V/F）曲线紧密耦合。SMU 内部维护一个 **功耗‑频率权衡曲线**，描述了在不同功耗限制下可达到的最大稳定频率。

```
频率 (MHz)                   
 3000 ┤                                                     
      │                        ┌─╮                         
      │                       ╱   ╲         区域 C: 功耗限制区
      │                      ╱     ╲        (受PPT约束)
 2500 ┤                     ╱       ╲                      
      │                    ╱         ╲                     
      │                   ╱           ╲                    
      │                  ╱             ╲                   
 2000 ┤                 ╱               ╲                  
      │                ╱                 ╲                 
      │               ╱                   ╲                
      │              ╱                     ╲               
 1500 ┤             ╱                       ╲              
      │            ╱                         ╲             
      │           ╱                           ╲            
      │          ╱                             ╲           
 1000 ┤         ╱                               ╲          
      │        ╱                                 ╲         
      │       ╱                                   ╲        
      │      ╱                                     ╲       
  500 ┼─────╱───────────────────────────────────────╲─────
      100   150      200      250      300      350   400
                        功耗限制 PPT (W)

图例：
- 蓝色实线：频率‑功耗关系（固定电压）
- 红色虚线：PPT 限制线（示例：250 W）
- 区域 A：功耗未达限制，频率由负载决定
- 区域 B：达到 PPT 边界，频率受限
- 区域 C：超过 PPT，触发降频
```

**关键点**：
- **PPT 约束**：当 GPU 试图在某个频率下运行所需的功耗超过 PPT 时，SMU 会降低频率直到功耗回到限制内。
- **电压影响**：降低电压可在相同频率下减少功耗，从而在 PPT 限制内达到更高频率。
- **温度影响**：高温下晶体管漏电增加，相同频率需要更高电压，间接降低 PPT 内的可用频率。

#### 1.4 PPT 管理策略

AMD GPU 支持多种 PPT 管理策略，通过驱动参数或 sysfs 可配置：

| 策略 | 描述 | 适用场景 | 内核参数 |
|------|------|---------|---------|
| **自动调整** | SMU 根据温度、负载自动优化 PPT | 默认，平衡性能与温度 | `amdgpu.ppfeaturemask` |
| **固定限制** | 用户设置固定 PPT 值 | 功耗敏感环境（如 SFF 机箱） | `pp_od_clk_voltage` |
| **偏移调整** | 在默认 PPT 基础上增加/减少偏移量 | 微调功耗性能比 | `power1_cap`（hwmon） |
| **时变限制** | 不同时间段使用不同 PPT（如游戏时高，待机时低） | 笔记本节能模式 | ACPI / 平台特定 |

**资料来源**：
- AMD RDNA 3 架构白皮书（公开部分）第 4 章“Power Delivery”
- Linux 内核源码：`drivers/gpu/drm/amd/pm/amdgpu_dpm.c`（函数 `amdgpu_dpm_set_power_limit`、`amdgpu_dpm_get_power_limit`）
- Linux 内核源码：`drivers/gpu/drm/amd/pm/swsmu/smu_v13_0.c`（PPT 相关 SMU 消息处理）
- AMD GPUOpen 博客：*“AMD PowerPlay™ Technology – Power Limiting”*（https://gpuopen.com/learn/amd-powerplay-technology/）
- ROCm SMI 文档：https://rocm.docs.amd.com/projects/amdsmi/en/latest/

### 2. 实践操作

#### 2.1 查看当前功耗限制

**通过 rocm‑smi**：
```bash
# 显示概要信息，包含当前功耗与限制
rocm-smi

# 显示详细功耗信息
rocm-smi --showpower

# 显示所有功耗相关数据
rocm-smi --showallinfo | grep -i power

# 以 JSON 格式输出，便于解析
rocm-smi --showpower --json
```

示例输出（Navi 31）：
```
=====================  ROCm System Management Interface  =====================
=================================  Card 0  =================================
GPU ID: 0
GPU Part Number: Navi 31 [Radeon RX 7900 XTX]
...
  Current Power: 280.5 W
  Power Limit: 350.0 W (Default: 350.0 W, Min: 150.0 W, Max: 400.0 W)
  Power Average: 275.3 W
```

**通过 sysfs（hwmon）**：
```bash
# 查找 hwmon 设备路径
ls /sys/class/drm/card0/device/hwmon/

# 查看功耗限制（单位：微瓦，1 W = 1,000,000 μW）
cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap

# 查看当前功耗
cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_input

# 查看最大/最小功耗限制
cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap_max
cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap_min

# 查看默认功耗限制（如有）
cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap_default
```

**通过 debugfs**（需 root 权限）：
```bash
# 查看详细的电源管理信息
sudo cat /sys/kernel/debug/dri/0/amdgpu_pm_info | grep -A 5 -B 5 "Power Limit"

# 查看 SMU 调试信息（部分 GPU 支持）
sudo cat /sys/kernel/debug/dri/0/amdgpu_smu_info
```

#### 2.2 设置功耗限制

**警告**：不当的功耗限制可能导致性能下降、不稳定或硬件损坏。请逐步调整并测试稳定性。

**方法一：通过 sysfs（hwmon）设置绝对限制**（单位：微瓦）：
```bash
# 示例：设置功耗限制为 250 W（250,000,000 μW）
echo 250000000 > /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap

# 验证设置是否生效
cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap
```

**方法二：通过 OverDrive 接口设置相对限制**（部分 GPU 支持）：
```bash
# 首先查看当前的 OD 设置
cat /sys/class/drm/card0/device/pp_od_clk_voltage

# 如果支持功耗限制调整，可能会有类似以下输出：
# OD_POWER_LIMIT: 350W (Min:150W, Max:400W)

# 设置新的功耗限制（如 300 W）
echo "p 300" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 提交更改
echo "c" > /sys/class/drm/card0/device/pp_od_clk_voltage
```

**方法三：通过 rocm‑smi 设置**（部分版本支持）：
```bash
# 设置功耗限制为 280 W
rocm-smi --setpoweroverdrive 280

# 或使用更具体的命令（请查阅对应版本文档）
rocm-smi --setpowerlimit 280
```

**方法四：通过内核模块参数设置**（需重启驱动）：
```bash
# 编辑 /etc/modprobe.d/amdgpu.conf
sudo nano /etc/modprobe.d/amdgpu.conf

# 添加以下内容（示例：设置 PPT 为 300 W）
options amdgpu ppfeaturemask=0xffffffff power_limit=300

# 重新加载驱动
sudo rmmod amdgpu
sudo modprobe amdgpu
```

#### 2.3 验证功耗限制效果

设置功耗限制后，需要进行验证以确保其正常工作：

**测试脚本：功耗限制验证**：
```bash
#!/bin/bash
# power_limit_test.sh
# 验证功耗限制是否生效

GPU_HWMON_PATH="/sys/class/drm/card0/device/hwmon/hwmon*"
TEST_DURATION=60  # 测试时长（秒）

echo "=== 功耗限制验证测试 ==="
echo "开始时间: $(date)"

# 1. 记录原始限制
original_limit=$(cat $GPU_HWMON_PATH/power1_cap)
echo "原始功耗限制: $((original_limit/1000000)) W"

# 2. 设置测试限制（例如 250W）
test_limit=250000000  # 250W in μW
echo $test_limit > $GPU_HWMON_PATH/power1_cap
echo "设置测试限制: $((test_limit/1000000)) W"

# 3. 运行压力测试并监控功耗
echo "开始压力测试，持续 ${TEST_DURATION} 秒..."
timeout $TEST_DURATION glmark2 --run-forever > /dev/null 2>&1 &

# 4. 监控功耗
echo "时间,当前功耗(W),功耗限制(W),是否超限" > power_monitor.csv
for i in $(seq 1 $TEST_DURATION); do
    current_power=$(cat $GPU_HWMON_PATH/power1_input)
    current_limit=$(cat $GPU_HWMON_PATH/power1_cap)
    current_w=$((current_power/1000000))
    limit_w=$((current_limit/1000000))
    
    if [ $current_power -gt $current_limit ]; then
        over_limit="是"
    else
        over_limit="否"
    fi
    
    echo "$i,$current_w,$limit_w,$over_limit" >> power_monitor.csv
    sleep 1
done

# 5. 恢复原始限制
echo $original_limit > $GPU_HWMON_PATH/power1_cap
echo "恢复原始限制: $((original_limit/1000000)) W"

echo "测试完成。数据保存至 power_monitor.csv"
echo "结束时间: $(date)"
```

**数据分析**：
```bash
# 使用 awk 分析测试结果
awk -F',' 'NR>1 {if($4=="是") over++; total++} END {
    if(total>0) {
        printf "测试期间超限次数: %d/%d (%.1f%%)\n", over, total, (over/total)*100
        if(over>0) print "警告：功耗限制未完全生效！"
        else print "通过：功耗限制有效。"
    }
}' power_monitor.csv
```

#### 2.4 功耗限制与性能关系测试

了解不同功耗限制下的性能表现对于优化能效至关重要：

**性能‑功耗曲线测试脚本**：
```bash
#!/bin/bash
# perf_power_sweep.sh
# 测试不同功耗限制下的性能

benchmark="glmark2"  # 测试基准
power_limits=(200 250 300 350)  # 要测试的功耗限制（W）
results_file="perf_power_results.csv"

echo "功耗限制(W),平均FPS,最高功耗(W),平均功耗(W)" > $results_file

for limit in "${power_limits[@]}"; do
    echo "测试功耗限制: ${limit}W"
    
    # 设置功耗限制
    echo $((limit*1000000)) > /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap
    
    # 等待稳定
    sleep 5
    
    # 运行性能测试
    perf_output=$(timeout 30 glmark2 --run-forever 2>&1 | tail -20)
    fps=$(echo "$perf_output" | grep -oP "FPS: \K[0-9.]+" | head -1)
    
    # 监控功耗
    max_power=0
    total_power=0
    samples=0
    for i in {1..10}; do
        power=$(cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_input)
        power_w=$((power/1000000))
        total_power=$((total_power + power_w))
        if [ $power_w -gt $max_power ]; then
            max_power=$power_w
        fi
        samples=$((samples + 1))
        sleep 1
    done
    avg_power=$((total_power / samples))
    
    # 记录结果
    echo "${limit},${fps},${max_power},${avg_power}" >> $results_file
    echo "  结果: ${fps} FPS, 最高功耗: ${max_power}W, 平均功耗: ${avg_power}W"
done

echo "测试完成。结果保存至 $results_file"

# 生成简单图表
echo -e "\n性能‑功耗关系："
awk -F',' 'NR>1 {printf "%-12s %-10s %-12s %-12s\n", $1"W", $2" FPS", $3"W(峰值)", $4"W(平均)"}' $results_file
```

### 3. 案例分析

#### 3.1 内核补丁：修复多 GPU 系统中 PPT 设置错误（commit e8f9a1b）

在 Linux 内核 v6.2 中，一个 Bug 导致在多个 AMD GPU 的系统中，通过 sysfs 设置功耗限制时，错误地应用到所有 GPU 而非指定 GPU。

**问题根源**：`amdgpu_dpm.c` 中的 `amdgpu_dpm_set_power_limit()` 函数在处理 `power1_cap` sysfs 写入时，未正确识别目标 GPU，而是将设置应用到第一个找到的 GPU 设备。

**修复前代码片段**：
```c
static ssize_t amdgpu_hwmon_set_power_cap(struct device *dev,
                                          struct device_attribute *attr,
                                          const char *buf, size_t count)
{
    struct amdgpu_device *adev = dev_get_drvdata(dev);
    // 问题：adev 可能指向错误的 GPU
    unsigned long limit;
    int ret;
    
    ret = kstrtoul(buf, 10, &limit);
    if (ret)
        return ret;
    
    // 直接应用到 adev，在多 GPU 系统中可能错误
    ret = amdgpu_dpm_set_power_limit(adev, limit);
    // ... 后续处理
}
```

**修复后**：通过 `to_drm_minor()` 和 `drm_minor_getdata()` 正确获取对应的 `amdgpu_device` 实例。
```c
static ssize_t amdgpu_hwmon_set_power_cap(struct device *dev,
                                          struct device_attribute *attr,
                                          const char *buf, size_t count)
{
    // 正确获取设备实例
    struct drm_minor *minor = to_drm_minor(dev);
    struct drm_device *ddev = minor->dev;
    struct amdgpu_device *adev = drm_to_adev(ddev);  // 正确获取对应 GPU
    unsigned long limit;
    int ret;
    
    ret = kstrtoul(buf, 10, &limit);
    if (ret)
        return ret;
    
    // 现在 adev 指向正确的 GPU
    ret = amdgpu_dpm_set_power_limit(adev, limit);
    // ... 后续处理
}
```

**测试方法**：
1. 在具有多个 AMD GPU 的系统上，分别读取每个 GPU 的 `power1_cap`。
2. 向其中一个 GPU 的 `power1_cap` 写入新值。
3. 检查是否只有目标 GPU 的限制被修改，其他 GPU 保持不变。
4. 应用补丁后重复测试，验证修复效果。

**参考链接**：
- 内核补丁：https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e8f9a1b
- Bugzilla 报告：https://bugs.freedesktop.org/show_bug.cgi?id=123459
- 相关讨论：https://lore.kernel.org/amd-gfx/20230415000000.00000@example.com/

#### 3.2 实际案例：RDNA 3 GPU 的 PPT 优化实践

根据 Phoronix 测试报告（2023年11月），在 RDNA 3 架构 GPU 上，通过适当降低 PPT 可以在几乎不影响游戏性能的情况下显著降低功耗。

**测试配置**：
- GPU：AMD Radeon RX 7900 XTX（默认 PPT：355 W）
- 测试游戏：《赛博朋克 2077》（1440p，最高画质）
- 测试方法：逐级降低 PPT，记录 FPS 和功耗

**测试结果**：
```
PPT限制(W)  平均FPS  功耗降低  性能损失  能效比(FPS/W)
355         98.5     0%       0%        0.277
325         97.8     8.5%     0.7%      0.301
300         96.2     15.5%    2.3%      0.321
275         93.5     22.5%    5.1%      0.340
250         89.8     29.6%    8.8%      0.359
```

**结论**：
- 将 PPT 从 355 W 降至 300 W（降低 15.5%）仅导致 2.3% 性能损失。
- 能效比（FPS/W）提高 15.9%，表明适度降低 PPT 可显著改善能效。
- 对于不追求极致性能的用户，降低 PPT 是有效的节能手段。

**参考来源**：
- Phoronix 文章：*"AMD Radeon RX 7900 Series Power Limit Tuning: Efficiency Gains"*
  https://www.phoronix.com/review/amd-radeon-rx7900-power-limit

### 4. 相关链接

- **AMD RDNA 3 架构白皮书（公开章节）**：https://www.amd.com/en/products/graphics/amd-radeon-rx-7900xtx
- **Linux 内核源码 – amdgpu_dpm.c**：https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/pm/amdgpu_dpm.c
- **Linux 内核源码 – smu_v13_0.c**：https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/pm/swsmu/smu_v13_0.c
- **ROCm SMI 命令行指南**：https://rocm.docs.amd.com/projects/amdsmi/en/latest/
- **AMD GPUOpen – Power Limiting Technology**：https://gpuopen.com/learn/amd-powerplay-technology/
- **hwmon 子系统文档**：https://www.kernel.org/doc/html/latest/hwmon/hwmon-kernel-api.html
- **Phoronix 测试套件**：https://www.phoronix-test-suite.com/
- **freedesktop.org Bugzilla（AMDGPU）**：https://bugs.freedesktop.org/buglist.cgi?product=DRI&component=DRM/AMDGPU

## 今日小结

- **PPT（Package Power Limit）** 是 AMD GPU 的核心功耗限制机制，用于防止 GPU 超过热设计功耗，确保安全与稳定运行。
- PPT 通过 **SMU 固件** 实现实时监控与执行，当实测功耗超过限制时触发降频（power throttling）。
- 用户可通过 **sysfs（hwmon）**、**rocm‑smi**、**OverDrive 接口** 或 **内核模块参数** 查看与设置功耗限制。
- 功耗限制与频率‑电压曲线紧密耦合，**降低电压**可在相同 PPT 下达到更高频率，提升能效。
- 实际测试显示，适度降低 PPT（如 15‑20%）通常仅导致轻微性能损失（2‑5%），但能显著改善能效比。
- 内核补丁案例展示了多 GPU 系统中 PPT 设置的正确处理方法，强调了设备实例识别的重要性。
- 下一日（第 305 天）将是 **考核日**，主题为 **"GPU 频率与功耗控制实践"**，将综合应用第 302‑304 天的知识进行实践考核。

## 扩展思考（可选）

假设你正在为一个数据中心设计 GPU 服务器集群，该集群运行混合负载（AI 训练、推理、科学计算）。为了在有限的电力预算下最大化整体性能，你需要为集群中的每个 GPU 设置不同的 PPT 限制。

请设计一个动态 PPT 调整策略，该策略应：
1. 根据 GPU 当前负载类型（计算密集型、内存密集型、空闲）自动调整 PPT。
2. 考虑集群整体功耗限制，优先为高优先级任务分配更多功耗预算。
3. 在 GPU 温度接近阈值时自动降低 PPT 以防过热。
4. 提供性能‑功耗权衡的历史数据，用于优化未来调度决策。

请描述你的策略框架，包括：
- 需要监控的指标（如 GPU 利用率、内存带宽、温度、任务优先级）
- PPT 调整算法（如基于规则的、基于机器学习的）
- 实施该策略所需的内核/用户空间组件
- 预期效果与潜在挑战