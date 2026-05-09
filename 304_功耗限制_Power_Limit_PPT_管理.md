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

---

## 附录A：PPT管理内核代码深度分析

### A.1 amdgpu_dpm_set_power_limit() 函数全解析

```c
// drivers/gpu/drm/amd/pm/amdgpu_dpm.c
int amdgpu_dpm_set_power_limit(struct amdgpu_device *adev, uint32_t limit)
{
    int ret = 0;

    if (!adev->pm.dpm_enabled)
        return -EINVAL;

    mutex_lock(&adev->pm.mutex);

    switch (adev->asic_type) {
    case CHIP_NAVI10:
    case CHIP_NAVI12:
    case CHIP_NAVI14:
    case CHIP_NAVI21:
    case CHIP_NAVI31:
    case CHIP_NAVI32:
    case CHIP_NAVI33:
        if (adev->pm.funcs && adev->pm.funcs->set_power_limit)
            ret = adev->pm.funcs->set_power_limit(adev, limit);
        break;
    default:
        dev_err(adev->dev, "unsupported ASIC for set_power_limit\n");
        ret = -EOPNOTSUPP;
        break;
    }

    mutex_unlock(&adev->pm.mutex);
    return ret;
}
```

**调用链分析**：
```
用户态写入 power1_cap
    → amdgpu_hwmon_set_power_cap() [drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c]
        → amdgpu_dpm_set_power_limit() [drivers/gpu/drm/amd/pm/amdgpu_dpm.c]
            → adev->pm.funcs->set_power_limit() [具体 SMU 实现]
                → smu_v13_0_set_power_limit() [RDNA 3]
                    → smu_cmn_send_smc_msg_with_param(SMU_MSG_SetPowerLimit, limit)
```

### A.2 SMU PPT 处理流程（RDNA 3 示例）

```c
// drivers/gpu/drm/amd/pm/swsmu/smu_v13_0.c
static int smu_v13_0_set_power_limit(struct smu_context *smu, uint32_t limit)
{
    struct amdgpu_device *adev = smu->adev;
    uint32_t ppt_limit;
    int ret;

    // 将限制值从微瓦转换为 SMU 内部单位
    ppt_limit = limit_to_smu_power(smu, limit);

    // 检查限制是否在允许范围内
    if (ppt_limit < smu->ppt_limit_min || ppt_limit > smu->ppt_limit_max) {
        dev_err(adev->dev, "Power limit %u out of range [%u, %u]\n",
                ppt_limit, smu->ppt_limit_min, smu->ppt_limit_max);
        return -EINVAL;
    }

    // 发送 SMU 消息
    ret = smu_cmn_send_smc_msg_with_param(smu, SMU_MSG_SetPowerLimit, ppt_limit);
    if (ret) {
        dev_err(adev->dev, "Failed to set power limit: %d\n", ret);
        return ret;
    }

    // 更新缓存
    smu->current_power_limit = ppt_limit;

    dev_dbg(adev->dev, "Power limit set to %u (SMU unit: %u)\n", limit, ppt_limit);
    return 0;
}
```

### A.3 PPT 双时间窗口算法详解

SMU 内部实现了一个双时间窗口功耗检测算法，用于平衡瞬时保护和持续限制：

```c
// SMU 固件伪代码
struct ppt_window {
    uint32_t window_size_ms;    // 时间窗口大小
    uint32_t power_sum;          // 窗口内功耗累积
    uint32_t sample_count;       // 采样计数
    uint32_t threshold;          // 阈值（PPT）
};

struct ppt_controller {
    struct ppt_window fast_window;  // 快速窗口（~1ms）
    struct ppt_window slow_window;  // 慢速窗口（~10ms）
    uint32_t current_power;         // 当前瞬时功耗
    bool throttling_active;         // 是否正在降频
};
```

**算法工作流程**：
1. **快速窗口（Fast Window）**：~1ms 窗口，用于检测瞬时功耗尖峰。当瞬时功耗超过快速阈值时，立即触发轻度降频。
2. **慢速窗口（Slow Window）**：~10ms 窗口，用于检测持续过载。当慢速窗口平均功耗超过 PPT 时，逐步降低频率直到平均功耗降至限制以内。
3. **恢复阶段**：当慢速窗口平均功耗低于 PPT 的 90% 时，逐步恢复频率。

### A.4 PPT 与 TDC/EDC 的协同工作

```
功耗/电流
  ^
  |    EDC 线 ──•────────────────────•  (瞬时电流峰值限制)
  |             ╲                    ╲
  |              ╲    TDC 线 ────•───────•  (持续电流限制)
  |               ╲              ╲       ╲
  |                ╲    PPT 线 ──────•───────•  (封装功耗限制)
  |                 ╲              ╲       ╲
  |                  ╲              ╲       ╲
  +──────────────────•──────────────•───────•───→ 时间
                    触发降频点      恢复点    安全点
```

**三层保护机制**：
- **EDC 触发** → 立即大幅降频（~50%），持续~100μs
- **TDC 触发** → 中度降频（~20%），持续~1ms
- **PPT 触发** → 轻度降频（~5-10%），持续到功耗回归

### A.5 power1_cap sysfs 属性实现

```c
// drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c
static ssize_t amdgpu_hwmon_set_power_cap(struct device *dev,
                                           struct device_attribute *attr,
                                           const char *buf, size_t count)
{
    struct amdgpu_device *adev;
    struct drm_minor *minor;
    unsigned long limit;
    int ret;

    minor = to_drm_minor(dev);
    if (!minor)
        return -ENODEV;

    adev = drm_to_adev(minor->dev);
    if (!adev)
        return -ENODEV;

    ret = kstrtoul(buf, 10, &limit);
    if (ret)
        return ret;

    // 限制值检查（微瓦）
    if (limit < adev->pm.power_cap_min || limit > adev->pm.power_cap_max) {
        dev_err(adev->dev,
                "Power cap %lu out of range [%llu, %llu]\n",
                limit, adev->pm.power_cap_min, adev->pm.power_cap_max);
        return -EINVAL;
    }

    ret = amdgpu_dpm_set_power_limit(adev, limit);
    if (ret)
        return ret;

    return count;
}
```

## 附录B：多GPU系统PPT协同管理框架

### B.1 框架设计

```python
#!/usr/bin/env python3
"""
multi_gpu_ppt_manager.py
多 GPU PPT 协同管理框架
适用于数据中心和计算集群环境
"""

import os
import time
import threading
import logging
from dataclasses import dataclass
from typing import Dict, List, Optional, Tuple
from enum import Enum

logging.basicConfig(level=logging.INFO,
                   format='%(asctime)s [%(levelname)s] %(message)s')
logger = logging.getLogger(__name__)


class WorkloadType(Enum):
    COMPUTE_INTENSIVE = "compute_intensive"
    MEMORY_INTENSIVE = "memory_intensive"
    MIXED = "mixed"
    IDLE = "idle"


@dataclass
class GPUMetrics:
    gpu_id: int
    power_usage_w: float = 0.0
    power_limit_w: float = 0.0
    temperature_c: float = 0.0
    gpu_util_pct: float = 0.0
    mem_util_pct: float = 0.0
    mem_bw_util_pct: float = 0.0
    workload_type: WorkloadType = WorkloadType.IDLE
    task_priority: int = 0


@dataclass
class PPTConfig:
    min_limit_w: float
    max_limit_w: float
    default_limit_w: float
    current_limit_w: float
    boost_limit_w: float  # 临时提升限制


class GPUPowerManager:
    def __init__(self, cluster_power_budget_w: float):
        self.cluster_budget = cluster_power_budget_w
        self.gpus: Dict[int, GPUPowerManager] = {}
        self.monitoring_thread: Optional[threading.Thread] = None
        self.running = False

    def discover_gpus(self) -> List[int]:
        """发现系统中的 AMD GPU 设备"""
        gpu_ids = []
        hwmon_base = "/sys/class/drm/"
        for entry in os.listdir(hwmon_base):
            if entry.startswith("card"):
                device_path = os.path.join(hwmon_base, entry, "device")
                if os.path.exists(device_path):
                    hwmon_path = os.path.join(device_path, "hwmon")
                    if os.path.exists(hwmon_path):
                        for hwmon_entry in os.listdir(hwmon_path):
                            cap_path = os.path.join(hwmon_path, hwmon_entry, "power1_cap")
                            if os.path.exists(cap_path):
                                card_id = int(entry.replace("card", ""))
                                gpu_ids.append(card_id)
        return sorted(gpu_ids)

    def read_gpu_metrics(self, gpu_id: int) -> GPUMetrics:
        """读取单个 GPU 的监控指标"""
        metrics = GPUMetrics(gpu_id=gpu_id)
        hwmon_base = f"/sys/class/drm/card{gpu_id}/device/hwmon"

        for hwmon_entry in os.listdir(hwmon_base):
            hwmon_path = os.path.join(hwmon_base, hwmon_entry)

            # 读取功耗
            power_path = os.path.join(hwmon_path, "power1_input")
            if os.path.exists(power_path):
                with open(power_path, 'r') as f:
                    metrics.power_usage_w = int(f.read().strip()) / 1_000_000

            # 读取功耗限制
            cap_path = os.path.join(hwmon_path, "power1_cap")
            if os.path.exists(cap_path):
                with open(cap_path, 'r') as f:
                    metrics.power_limit_w = int(f.read().strip()) / 1_000_000

            # 读取温度
            temp_path = os.path.join(hwmon_path, "temp1_input")
            if os.path.exists(temp_path):
                with open(temp_path, 'r') as f:
                    metrics.temperature_c = int(f.read().strip()) / 1000

        return metrics

    def classify_workload(self, gpu_id: int) -> WorkloadType:
        """基于利用率指标分类工作负载类型"""
        metrics = self.read_gpu_metrics(gpu_id)
        gpu_util_path = f"/sys/class/drm/card{gpu_id}/device/gpu_busy_percent"
        mem_util_path = f"/sys/class/drm/card{gpu_id}/device/mem_busy_percent"

        gpu_util = 0
        mem_util = 0

        if os.path.exists(gpu_util_path):
            with open(gpu_util_path, 'r') as f:
                gpu_util = int(f.read().strip())

        if os.path.exists(mem_util_path):
            with open(mem_util_path, 'r') as f:
                mem_util = int(f.read().strip())

        if gpu_util < 5 and mem_util < 5:
            return WorkloadType.IDLE
        elif gpu_util > 70 and mem_util < 30:
            return WorkloadType.COMPUTE_INTENSIVE
        elif mem_util > 70 and gpu_util < 30:
            return WorkloadType.MEMORY_INTENSIVE
        else:
            return WorkloadType.MIXED

    def calculate_optimal_ppt(self, cluster_budget: float,
                               gpu_metrics_list: List[Tuple[int, GPUMetrics, WorkloadType]]) -> Dict[int, float]:
        """基于优先级和负载类型计算最优 PPT 分配"""
        # 计算总需求
        total_priority_weight = sum(
            m.task_priority for _, m, _ in gpu_metrics_list
        )

        # 基础分配（每个 GPU 至少获得最小 PPT）
        base_ppt = 100.0  # 最小基础 PPT (W)
        results: Dict[int, float] = {}

        for gpu_id, metrics, wl_type in gpu_metrics_list:
            if wl_type == WorkloadType.IDLE:
                results[gpu_id] = min(base_ppt, metrics.power_limit_w)
            else:
                # 按优先级分配剩余预算
                priority_ratio = metrics.task_priority / total_priority_weight
                allocated = base_ppt + (cluster_budget - base_ppt * len(gpu_metrics_list)) * priority_ratio
                results[gpu_id] = min(allocated, metrics.power_limit_w * 1.2)

        return results

    def apply_ppt_limits(self, ppt_allocation: Dict[int, float]):
        """应用 PPT 限制到各 GPU"""
        for gpu_id, limit_w in ppt_allocation.items():
            hwmon_base = f"/sys/class/drm/card{gpu_id}/device/hwmon"
            for hwmon_entry in os.listdir(hwmon_base):
                cap_path = os.path.join(hwmon_base, hwmon_entry, "power1_cap")
                if os.path.exists(cap_path):
                    limit_uw = int(limit_w * 1_000_000)
                    try:
                        with open(cap_path, 'w') as f:
                            f.write(str(limit_uw))
                        logger.info(f"GPU {gpu_id}: PPT 设置为 {limit_w:.1f}W")
                    except PermissionError:
                        logger.error(f"GPU {gpu_id}: 权限不足，需要 root 权限")

    def temperature_aware_throttle(self, gpu_id: int, threshold_c: float = 85.0):
        """温度感知降频：温度超过阈值时自动降低 PPT"""
        metrics = self.read_gpu_metrics(gpu_id)
        if metrics.temperature_c > threshold_c:
            reduction_factor = 1.0 - (metrics.temperature_c - threshold_c) / 30.0
            new_limit = metrics.power_limit_w * max(reduction_factor, 0.6)
            hwmon_base = f"/sys/class/drm/card{gpu_id}/device/hwmon"
            for hwmon_entry in os.listdir(hwmon_base):
                cap_path = os.path.join(hwmon_base, hwmon_entry, "power1_cap")
                if os.path.exists(cap_path):
                    with open(cap_path, 'w') as f:
                        f.write(str(int(new_limit * 1_000_000)))
            logger.warning(f"GPU {gpu_id}: 温度 {metrics.temperature_c:.1f}°C 超过阈值，"
                          f"PPT 降至 {new_limit:.1f}W")


# ===== 使用示例 =====
if __name__ == "__main__":
    manager = GPUPowerManager(cluster_power_budget_w=1200.0)

    # 发现 GPU
    gpu_ids = manager.discover_gpus()
    logger.info(f"发现 GPU: {gpu_ids}")

    # 分类工作负载
    for gpu_id in gpu_ids:
        wl_type = manager.classify_workload(gpu_id)
        logger.info(f"GPU {gpu_id}: 工作负载类型 = {wl_type.value}")

    # 计算并应用 PPT
    metrics_list = []
    for gpu_id in gpu_ids:
        metrics = manager.read_gpu_metrics(gpu_id)
        metrics.task_priority = 1 if gpu_id == 0 else 2
        wl_type = manager.classify_workload(gpu_id)
        metrics_list.append((gpu_id, metrics, wl_type))

    allocation = manager.calculate_optimal_ppt(1200.0, metrics_list)
    manager.apply_ppt_limits(allocation)
```

### B.2 动态 PPT 调整策略伪代码

```python
def dynamic_ppt_adjustment_loop(manager: GPUPowerManager, interval_s: int = 5):
    """动态 PPT 调整主循环"""
    while True:
        total_cluster_power = 0.0
        gpu_data = []

        for gpu_id in manager.discover_gpus():
            metrics = manager.read_gpu_metrics(gpu_id)
            wl_type = manager.classify_workload(gpu_id)
            total_cluster_power += metrics.power_usage_w
            gpu_data.append((gpu_id, metrics, wl_type))

            # 温度感知降频
            if metrics.temperature_c > 85.0:
                manager.temperature_aware_throttle(gpu_id)

        # 检查集群总功耗
        if total_cluster_power > manager.cluster_budget:
            # 计算需要削减的功率
            excess = total_cluster_power - manager.cluster_budget
            reduction_factor = 1.0 - excess / total_cluster_power

            for gpu_id, metrics, wl_type in gpu_data:
                if wl_type != WorkloadType.IDLE:
                    new_limit = metrics.power_limit_w * reduction_factor
                    # 确保不低于最小值
                    new_limit = max(new_limit, 100.0)
                    manager.apply_ppt_limits({gpu_id: new_limit})

        time.sleep(interval_s)
```

## 附录C：PPT高级调试技巧与工具

### C.1 使用 ftrace 追踪 PPT 设置

```bash
#!/bin/bash
# ppt_ftrace_debug.sh
# 使用 ftrace 追踪 PPT 相关内核函数

# 设置 ftrace
FTRACE_DIR="/sys/kernel/debug/tracing"

# 清除之前的追踪
echo nop > $FTRACE_DIR/current_tracer
echo > $FTRACE_DIR/trace

# 设置追踪函数
echo function > $FTRACE_DIR/current_tracer
echo amdgpu_dpm_set_power_limit > $FTRACE_DIR/set_ftrace_filter
echo amdgpu_hwmon_set_power_cap >> $FTRACE_DIR/set_ftrace_filter
echo smu_v13_0_set_power_limit >> $FTRACE_DIR/set_ftrace_filter

# 启用函数追踪
echo 1 > $FTRACE_DIR/tracing_on

# 设置 PPT 以触发追踪
echo 300000000 > /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap

sleep 1

# 停止追踪
echo 0 > $FTRACE_DIR/tracing_on

# 查看追踪结果
cat $FTRACE_DIR/trace
```

**输出示例**：
```
# tracer: function
#
# entries-in-buffer/entries-written: 42/42   #P:16
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||   TIMESTAMP  FUNCTION
#              | |       |   ||||      |         |
    kworker/2:1-89    [002] ....  1234.567891: amdgpu_hwmon_set_power_cap <-dev_attr_show
    kworker/2:1-89    [002] ....  1234.567892: amdgpu_dpm_set_power_limit <-amdgpu_hwmon_set_power_cap
    kworker/2:1-89    [002] ....  1234.567893: smu_v13_0_set_power_limit <-amdgpu_dpm_set_power_limit
    kworker/2:1-89    [002] ....  1234.567894: smu_cmn_send_smc_msg_with_param <-smu_v13_0_set_power_limit
```

### C.2 使用 eBPF 监控 PPT 变化

```python
#!/usr/bin/env python3
"""
ppt_tracer_bpf.py
使用 eBPF/BCC 监控 PPT 变化
"""

from bcc import BPF
import ctypes as ct

# eBPF 程序
bpf_source = """
#include <linux/ptrace.h>

BPF_HASH(ppt_events, u64, u64);

int trace_ppt_set(struct pt_regs *ctx)
{
    u64 limit = 0;
    u64 *count;

    // 读取参数（第二个参数是限制值）
    if (ctx->di)
        limit = ctx->si;

    count = ppt_events.lookup(&limit);
    if (count) {
        (*count)++;
    } else {
        u64 one = 1;
        ppt_events.update(&limit, &one);
    }

    return 0;
}
"""

# 加载 eBPF 程序
b = BPF(text=bpf_source)
b.attach_kprobe(event="amdgpu_dpm_set_power_limit", fn_name="trace_ppt_set")

print("正在监控 PPT 设置事件...")
print("按 Ctrl+C 退出")

try:
    while True:
        time.sleep(2)
        ppt_events = b["ppt_events"]
        for key, value in sorted(ppt_events.items(),
                                  key=lambda kv: kv[1].value,
                                  reverse=True)[:10]:
            limit_w = key.value / 1000000
            print(f"PPT={limit_w:.0f}W: {value.value}次")
        print("---")
except KeyboardInterrupt:
    pass
```

### C.3 PPT 压力测试工具

```bash
#!/bin/bash
# ppt_stress_test.sh
# PPT 压力测试：验证不同功耗限制下的系统稳定性

set -e

GPU_HWMON_PATH="/sys/class/drm/card0/device/hwmon/hwmon*"
TEST_DURATION=300  # 每个测试点 5 分钟
COOLDOWN_DURATION=60  # 冷却时间

echo "============================================"
echo "  GPU PPT 压力测试套件 v1.0"
echo "============================================"

# 获取 GPU 信息
gpu_name=$(cat /sys/class/drm/card0/device/vendor 2>/dev/null || echo "Unknown")
echo "GPU: $gpu_name"

# 获取 PPT 范围
ppt_min=$(cat $GPU_HWMON_PATH/power1_cap_min 2>/dev/null || echo "0")
ppt_max=$(cat $GPU_HWMON_PATH/power1_cap_max 2>/dev/null || echo "0")
ppt_default=$(cat $GPU_HWMON_PATH/power1_cap_default 2>/dev/null || echo "0")
echo "PPT 范围: $((ppt_min/1000000))W - $((ppt_max/1000000))W"
echo "默认 PPT: $((ppt_default/1000000))W"
echo ""

# 测试点：从 60% 到 110% 的默认 PPT
test_points=$(python3 -c "
d=$ppt_default
for p in [0.6, 0.7, 0.8, 0.9, 1.0, 1.1]:
    print(int(d * p))
")

for ppt in $test_points; do
    ppt_w=$((ppt/1000000))
    echo "--------------------------------------------"
    echo "测试 PPT = ${ppt_w}W"

    # 设置 PPT
    echo $ppt > $GPU_HWMON_PATH/power1_cap
    echo "  已设置 PPT"

    # 等待稳定
    sleep 5

    # 运行压力测试
    echo "  运行压力测试 ${TEST_DURATION}s..."
    timeout $TEST_DURATION glmark2 --run-forever > /dev/null 2>&1 &
    STRESS_PID=$!

    # 监控数据
    echo "  时间(s),功耗(W),频率(MHz),温度(C)" > ppt_stress_${ppt_w}w.csv
    for i in $(seq 1 $TEST_DURATION); do
        power=$(cat $GPU_HWMON_PATH/power1_input)
        power_w=$((power/1000000))

        freq=$(cat /sys/class/drm/card0/device/pp_dpm_sclk 2>/dev/null | head -1 | grep -oP '\d+Mhz' || echo "N/A")

        temp=$(cat $GPU_HWMON_PATH/temp1_input 2>/dev/null || echo "0")
        temp_c=$((temp/1000))

        echo "$i,$power_w,$freq,$temp_c" >> ppt_stress_${ppt_w}w.csv
        sleep 1
    done

    # 停止压力测试
    kill $STRESS_PID 2>/dev/null || true
    wait $STRESS_PID 2>/dev/null || true

    # 冷却
    echo "  冷却 ${COOLDOWN_DURATION}s..."
    sleep $COOLDOWN_DURATION
done

# 恢复默认 PPT
echo $ppt_default > $GPU_HWMON_PATH/power1_cap
echo "已恢复默认 PPT"

# 生成报告
echo ""
echo "============================================"
echo "  测试完成，生成报告..."
echo "============================================"

for csv in ppt_stress_*.csv; do
    ppt_w=$(echo $csv | grep -oP '\d+')
    echo ""
    echo "PPT=${ppt_w}W 统计:"
    awk -F',' 'NR>1 {
        sum_p+=$2; sum_f+=$3; sum_t+=$4;
        if($2>max_p) max_p=$2;
        if(NR==2) {min_p=$2; min_f=$3; min_t=$4}
        if($2<min_p) min_p=$2;
        n++}
        END {
            printf "  平均功耗: %.1fW (最小: %.1fW, 最大: %.1fW)\n", sum_p/n, min_p, max_p
            printf "  平均温度: %.1f°C\n", sum_t/n
        }' $csv
done
```

### C.4 常见 PPT 问题排查

| 问题 | 症状 | 可能原因 | 排查方法 |
|------|------|---------|---------|
| PPT 设置不生效 | 写入 power1_cap 后读取仍是旧值 | root 权限不足 / SMU 固件不支持 | `dmesg \| grep -i smu` 检查错误 |
| 写入超出范围 | write error: Invalid argument | 值超出 [min, max] 范围 | 先读取 power1_cap_min/max |
| PPT 自动恢复 | 设置后几秒自动回到默认值 | SMU 固件强制覆盖 / 热保护触发 | 检查 temp1_input 是否接近阈值 |
| 多 GPU 设置混乱 | 修改 card0 影响 card1 | 内核 Bug（见案例3.1） | `dmesg \| grep amdgpu` 检查设备映射 |
| 性能不提升 | 提高 PPT 后性能无变化 | 已达到 V/F 曲线极限 / 温度限制 | 检查是否触发 thermal throttling |

## 附录D：功耗测量硬件原理

### D.1 GPU 功耗测量方法

AMD GPU 使用多种方法测量功耗：

| 方法 | 精度 | 延迟 | 实现方式 | 适用架构 |
|------|------|------|---------|---------|
| **电流检测电阻** | ±1% | ~100μs | VRM 输出串联精密电阻 | RDNA 1/2/3 |
| **IMON 总线** | ±2% | ~1ms | VRM 控制器 IMON 引脚 | RDNA 2/3 |
| **SMU 估算** | ±5% | ~10ms | 基于活动计数器和 V/F 模型 | 所有架构 |
| **VDDCR 监测** | ±3% | ~500μs | 电压调节器数字接口 | RDNA 3+ |

### D.2 power1_input 数据解析

```bash
# power1_input 输出原始值（微瓦）
$ cat /sys/class/drm/card0/device/hwmon/hwmon6/power1_input
285000000

# 转换为瓦特
$ echo "scale=1; $(cat /sys/class/drm/card0/device/hwmon/hwmon6/power1_input) / 1000000" | bc
285.0
```

### D.3 外部功耗测量对照验证

```python
#!/usr/bin/env python3
"""
external_power_validation.py
使用外部功率计验证 GPU 功耗读数
支持：Yokogawa WT310、Chroma 66202、PMBus VRM 读数
"""

import subprocess
import time
import csv
from datetime import datetime


class ExternalPowerMeter:
    def __init__(self, interface: str = "usb"):
        self.interface = interface

    def read_power_w(self) -> float:
        """从外部功率计读取功耗"""
        # 示例：通过串口读取 Yokogawa WT310
        try:
            result = subprocess.run(
                ["python3", "-c", """
import serial
ser = serial.Serial('/dev/ttyUSB0', 9600, timeout=2)
ser.write(b'FETC:POW?\\n')
response = ser.readline().decode().strip()
print(response)
ser.close()
"""],
                capture_output=True, text=True, timeout=5
            )
            return float(result.stdout.strip())
        except Exception as e:
            print(f"功率计读取失败: {e}")
            return 0.0


class GPUCompareTest:
    def __init__(self, gpu_id: int = 0):
        self.gpu_id = gpu_id
        self.hwmon_path = f"/sys/class/drm/card{gpu_id}/device/hwmon"
        self.meter = ExternalPowerMeter()

    def get_gpu_power_w(self) -> float:
        """读取 GPU 报告功耗"""
        for entry in os.listdir(self.hwmon_path):
            power_path = os.path.join(self.hwmon_path, entry, "power1_input")
            if os.path.exists(power_path):
                with open(power_path) as f:
                    return int(f.read().strip()) / 1_000_000
        return 0.0

    def run_comparison(self, duration_s: int = 60, interval_s: int = 1):
        """运行对比测试"""
        records = []

        print(f"开始功耗对比测试（{duration_s}秒）")
        print(f"{'时间':20s} {'GPU报告(W)':12s} {'功率计(W)':12s} {'误差(%)':10s}")

        for i in range(duration_s):
            timestamp = datetime.now().isoformat()
            gpu_power = self.get_gpu_power_w()
            ext_power = self.meter.read_power_w()

            if gpu_power > 0 and ext_power > 0:
                error_pct = (gpu_power - ext_power) / ext_power * 100
                records.append({
                    "time": timestamp,
                    "gpu_power": gpu_power,
                    "ext_power": ext_power,
                    "error_pct": error_pct
                })
                print(f"{timestamp:20s} {gpu_power:<12.1f} {ext_power:<12.1f} {error_pct:<+10.2f}")

            time.sleep(interval_s)

        # 统计分析
        if records:
            errors = [r["error_pct"] for r in records]
            avg_error = sum(errors) / len(errors)
            max_error = max(abs(e) for e in errors)
            print(f"\n统计结果：")
            print(f"  平均误差: {avg_error:+.2f}%")
            print(f"  最大绝对误差: {max_error:.2f}%")

            # 保存到 CSV
            with open("power_comparison.csv", "w", newline="") as f:
                writer = csv.DictWriter(f, fieldnames=records[0].keys())
                writer.writeheader()
                writer.writerows(records)
            print(f"数据已保存至 power_comparison.csv")
```

## 附录E：知识自测

### E.1 选择题（每题 5 分）

**1. PPT（Package Power Limit）在 AMD GPU 电源管理中的主要作用是？**
A) 控制 GPU 核心的最大运行频率
B) 限制 GPU 封装的最高持续功耗，防止过热和硬件损坏
C) 管理显存的数据传输速率
D) 调节风扇转速以降低噪音

**2. 在 RDNA 3 架构中，以下哪个组件负责执行 PPT 限制？**
A) Linux 内核调度器
B) AMDGPU 内核驱动
C) SMU（System Management Unit）固件
D) ROCm SMI 用户空间工具

**3. 以下哪个 sysfs 属性用于读取当前 GPU 功耗？**
A) power1_cap
B) power1_input
C) power1_max
D) temp1_input

**4. 当 GPU 实测功耗超过 PPT 限制时，SMU 首先采取的行动是？**
A) 立即关闭 GPU
B) 触发降频（Power Throttling），降低 SCLK/MCLK
C) 发送中断请求给 CPU
D) 增加风扇转速到最大值

**5. 在多 GPU 系统中设置 PPT 时，以下哪项是正确的？**
A) 所有 GPU 共享同一个 PPT 限制
B) 每个 GPU 有独立的 PPT 限制，需通过设备实例识别分别设置
C) 只能通过内核模块参数设置
D) 多 GPU 系统不支持 PPT 设置

**6. 以下关于 TDC 的描述正确的是？**
A) TDC 是封装功耗限制
B) TDC 是热设计电流限制，防止 VRM 过流
C) TDC 与 PPT 完全相同
D) TDC 是显存频率限制

**7. 使用 rocm-smi 设置功耗限制的正确命令是？**
A) rocm-smi --setfan 250
B) rocm-smi --setpowerlimit 250
C) rocm-smi --setsclk 250
D) rocm-smi --setvoltage 250

**8. PPT 双时间窗口算法中，快速窗口的主要作用是？**
A) 记录长期功耗趋势
B) 检测瞬时功耗尖峰，快速响应
C) 计算平均性能指标
D) 管理显存带宽分配

**9. 以下哪个操作能在不重启系统的情况下恢复 PPT 到默认值？**
A) 重新安装驱动
B) 写入 power1_cap_default 的值到 power1_cap
C) 重启 GPU
D) 断开显示器连接

**10. 在数据中心场景中，动态 PPT 调整策略应考虑的因素包括？（多选）**
A) GPU 当前负载类型
B) 集群整体功耗预算
C) 任务优先级
D) GPU 温度

### E.2 填空题（每题 3 分）

1. AMD GPU 的功耗限制通过 SMU 固件和 _____ 驱动协同实现。
2. power1_cap 的单位是 _____，1W 等于 _____ µW。
3. PPT 与 _____ 曲线紧密耦合，降低电压可在相同 PPT 下达到更高频率。
4. EDC（Electrical Design Current）限制的是 _____ 电流峰值。
5. 在 RDNA 3 中，设置 PPT 的 SMU 消息是 _____。
6. 双时间窗口算法包含 _____ 窗口和 _____ 窗口。

### E.3 简答题（每题 10 分）

1. 解释 PPT、TDP、TDC 和 EDC 四种功耗/电流限制的区别与联系。
2. 描述在 Linux 系统中通过 sysfs 设置 GPU PPT 的完整步骤和验证方法。
3. 分析为什么降低 GPU 电压可以在不降低 PPT 的情况下提高性能。

## 附录F：知识自测参考答案

### F.1 选择题答案

1. B → PPT 的核心作用是限制 GPU 封装的最高持续功耗，确保在散热能力范围内运行
2. C → SMU（System Management Unit）固件是实际执行 PPT 限制的组件
3. B → power1_input 用于读取当前功耗，power1_cap 用于设置/读取限制
4. B → 触发降频（Power Throttling）是 SMU 的首个响应动作
5. B → 多 GPU 系统中每个 GPU 有独立的 PPT 限制
6. B → TDC（Thermal Design Current）是热设计电流限制
7. B → rocm-smi --setpowerlimit 用于设置功耗限制
8. B → 快速窗口（~1ms）用于检测瞬时功耗尖峰
9. B → 将 power1_cap_default 的值写入 power1_cap 即可恢复默认
10. A, B, C, D → 动态 PPT 调整应综合考虑负载类型、集群预算、任务优先级和温度

### F.2 填空题答案

1. amdgpu（或 Linux 内核）
2. 微瓦（µW），1,000,000
3. 频率-电压（V/F）
4. 瞬时
5. SMU_MSG_SetPowerLimit
6. 快速（Fast），慢速（Slow）

### F.3 简答题参考答案要点

**1. PPT、TDP、TDC、EDC 的区别与联系**：
- PPT（Package Power Limit）：GPU 封装的最大持续功耗限制，是用户可配置的主要限制
- TDP（Thermal Design Power）：散热系统设计基准值，通常作为 PPT 的默认值
- TDC（Thermal Design Current）：最大持续电流限制，保护 VRM 免受过热
- EDC（Electrical Design Current）：最大瞬时电流限制，保护 VRM 免受过流损坏
- 四者形成分层保护：EDC（瞬时）→ TDC（持续）→ PPT（功耗）→ TDP（散热设计）

**2. 设置 GPU PPT 的完整步骤**：
- 步骤1：查找 hwmon 路径 `ls /sys/class/drm/card0/device/hwmon/`
- 步骤2：读取当前限制 `cat power1_cap`（记录原始值）
- 步骤3：读取范围 `cat power1_cap_min` `cat power1_cap_max`
- 步骤4：写入新值 `echo 250000000 > power1_cap`
- 步骤5：验证 `cat power1_cap`
- 步骤6：运行负载测试并监控 `cat power1_input`
- 步骤7：测试完成后恢复原始值

**3. 降低电压提高性能的原理**：
- 功耗公式：P = CV²f（C=电容, V=电压, f=频率）
- 电压对功耗呈平方影响，降低 10% 电压可降低 ~19% 功耗
- PPT 固定时，功耗降低释放了"功耗预算"
- 释放的预算可用于提升频率（V/F 曲线上的更高点）
- 关键约束：需确保稳定性，过低电压导致计算错误

## 附录G：实践项目——命令行PPT优化工具

```bash
#!/bin/bash
# pptsense.sh
# 交互式 PPT 优化工具 - 智能功耗限制助手
# 用法: sudo ./pptsense.sh [gpu_id]

set -e

GPU_ID=${1:-0}
HWMON_PATH="/sys/class/drm/card${GPU_ID}/device/hwmon"

# 查找 hwmon 入口
find_hwmon() {
    for entry in "$HWMON_PATH"/hwmon*; do
        if [ -f "$entry/power1_cap" ]; then
            echo "$entry"
            return 0
        fi
    done
    echo ""
    return 1
}

HW=$(find_hwmon)
if [ -z "$HW" ]; then
    echo "错误：找不到 GPU $GPU_ID 的 hwmon 接口"
    exit 1
fi

echo "╔══════════════════════════════════════════════════╗"
echo "║         PPT Sense v1.0 - 智能功耗限制助手        ║"
echo "╠══════════════════════════════════════════════════╣"
echo "║  GPU ID: $GPU_ID"
echo "║  设备路径: $HW"
echo "╚══════════════════════════════════════════════════╝"
echo ""

# 显示当前状态
ppt_current=$(cat "$HW/power1_cap")
ppt_min=$(cat "$HW/power1_cap_min" 2>/dev/null || echo "0")
ppt_max=$(cat "$HW/power1_cap_max" 2>/dev/null || echo "0")
ppt_default=$(cat "$HW/power1_cap_default" 2>/dev/null || echo "$ppt_current")

echo "当前 PPT: $((ppt_current/1000000)) W"
echo "默认 PPT: $((ppt_default/1000000)) W"
echo "范围: $((ppt_min/1000000)) - $((ppt_max/1000000)) W"
echo ""

# 菜单
echo "选择操作："
echo "  1) 设置绝对功耗限制"
echo "  2) 设置百分比（相对默认值）"
echo "  3) 运行能效扫描测试"
echo "  4) 恢复默认设置"
echo "  5) 查看实时功耗监控"
echo "  0) 退出"
read -p "选择 [0-5]: " choice

case $choice in
    1)
        read -p "输入功耗限制 (W): " limit_w
        limit_uw=$((limit_w * 1000000))
        if [ "$limit_uw" -ge "$ppt_min" ] && [ "$limit_uw" -le "$ppt_max" ]; then
            echo "$limit_uw" > "$HW/power1_cap"
            echo "PPT 已设置为 ${limit_w}W"
        else
            echo "错误：值超出范围 [$((ppt_min/1000000))W, $((ppt_max/1000000))W]"
        fi
        ;;
    2)
        read -p "输入百分比 (50-110): " pct
        limit_uw=$((ppt_default * pct / 100))
        limit_w=$((limit_uw/1000000))
        echo "设置 PPT 为 ${pct}% = ${limit_w}W"
        if [ "$limit_uw" -ge "$ppt_min" ] && [ "$limit_uw" -le "$ppt_max" ]; then
            echo "$limit_uw" > "$HW/power1_cap"
            echo "已完成"
        else
            echo "错误：计算结果超出范围"
        fi
        ;;
    3)
        echo "运行能效扫描（将测试 60%/80%/100% PPT）..."
        for pct in 60 80 100; do
            test_uw=$((ppt_default * pct / 100))
            echo "$test_uw" > "$HW/power1_cap"
            sleep 2
            power=$(cat "$HW/power1_input")
            echo "  ${pct}% PPT: 当前功耗 $((power/1000000))W"
            sleep 3
        done
        echo "$ppt_default" > "$HW/power1_cap"
        echo "扫描完成，已恢复默认"
        ;;
    4)
        echo "$ppt_default" > "$HW/power1_cap"
        echo "PPT 已恢复默认值 $((ppt_default/1000000))W"
        ;;
    5)
        echo "实时监控（按 Ctrl+C 退出）:"
        echo "  时间        功耗(W)  PPT(W) 温度(C) 频率(MHz)"
        while true; do
            power=$(cat "$HW/power1_input" 2>/dev/null)
            cap=$(cat "$HW/power1_cap" 2>/dev/null)
            temp=$(cat "$HW/temp1_input" 2>/dev/null)
            freq=$(cat "/sys/class/drm/card${GPU_ID}/device/pp_dpm_sclk" 2>/dev/null | head -1 | grep -oP 'Mhz \K[0-9]+' || echo "N/A")
            echo "$(date +%H:%M:%S)  $((power/1000000))W  $((cap/1000000))W  $((temp/1000))C  ${freq}MHz"
            sleep 1
        done
        ;;
    0)
        echo "退出"
        exit 0
        ;;
    *)
        echo "无效选择"
        exit 1
        ;;
esac
```

## 附录H：跨厂商功耗管理对比

### H.1 AMD vs NVIDIA vs Intel 功耗管理对比

| 特性 | AMD | NVIDIA | Intel Arc |
|------|-----|--------|-----------|
| 功耗限制接口 | sysfs (power1_cap) | nvidia-smi -pl | sysfs (power1_cap) |
| 驱动模块 | amdgpu | nvidia | i915 / xe |
| SMU/微控制器 | SMU (RDNA架构) | PMU (Falton uC) | GuC / HuC |
| 用户态工具 | rocm-smi | nvidia-smi | intel_gpu_top |
| 双窗口算法 | 支持（1ms+10ms） | 支持（瞬态+持续） | 类似机制 |
| TDP 配置方式 | VBIOS PPT Table | VBIOS Power Table | VBIOS |
| OverDrive 支持 | 是（pp_od_clk_voltage） | 是（nvidia-smi -ac） | 部分 |
| 多GPU管理 | rocm-smi 遍历 | nvidia-smi 遍历 | sysfs 遍历 |
| 功耗测量精度 | ±1-5%（取决于方法） | ±1-3% | ±2-5% |
| 动态调整频率 | FRL (Fuzzy Rule Logic) | GPU Boost 4.0 | Dynamic Tuning |

### H.2 NVIDIA 对应接口参考

```bash
# NVIDIA GPU 功耗管理命令对应

# 查看功耗限制
nvidia-smi -q -d POWER

# 设置功耗限制（需要 root）
sudo nvidia-smi -pl 250

# 查看当前功耗
nvidia-smi --query-gpu=power.draw --format=csv

# 查看功耗范围
nvidia-smi -q -d POWER | grep -i "Power Limit"

# 持久化设置
nvidia-persistenced
```

### H.3 Intel Arc 对应接口参考

```bash
# Intel Arc GPU 功耗管理

# 通过 sysfs 查看功耗
cat /sys/class/drm/card1/device/hwmon/hwmon*/power1_input

# 设置功耗限制（i915 驱动）
echo 200000000 > /sys/class/drm/card1/device/hwmon/hwmon*/power1_cap

# 使用 intel_gpu_top 监控
sudo intel_gpu_top -l

# Xe 驱动（新一代）
# 类似的 sysfs 接口
```

## 附录I：参考文献与延伸阅读

### 内核源码参考
- `drivers/gpu/drm/amd/pm/amdgpu_dpm.c` - DPM 核心逻辑
- `drivers/gpu/drm/amd/pm/swsmu/smu_v13_0.c` - RDNA 3 SMU 实现
- `drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c` - PM sysfs 接口
- `drivers/gpu/drm/amd/pm/swsmu/smu_cmn.c` - SMU 通信通用层
- `include/linux/hwmon.h` - hwmon 子系统 API

### 技术文档
- AMD RDNA 3 架构白皮书 - Power Delivery 章节
- AMD GPUOpen - PowerPlay Technology: https://gpuopen.com/learn/amd-powerplay-technology/
- Linux 内核 hwmon 文档: https://www.kernel.org/doc/html/latest/hwmon/
- ROCm SMI 文档: https://rocm.docs.amd.com/projects/amdsmi/en/latest/

### 测试工具
- Phoronix Test Suite: https://www.phoronix-test-suite.com/
- glmark2: https://github.com/glmark2/glmark2
- GpuTest: https://www.geeks3d.com/gputest/
- Unigine Heaven/Superposition: https://benchmark.unigine.com/

### 进阶阅读
- Linux 电源管理子系统: Documentation/power/
- ACPI 功耗管理规范: https://uefi.org/specifications
- PCIe 功耗管理（ASPM）: PCI Express Base Specification
- PMBus 协议: https://pmbus.org/

---

## 附录J：PPT 调试命令速查表

| 命令 | 功能 | 示例 |
|------|------|------|
| `cat power1_cap` | 查看当前 PPT 限制 | `cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap` |
| `cat power1_input` | 查看当前功耗 | `cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_input` |
| `cat power1_cap_max` | 查看最大允许 PPT | `cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap_max` |
| `cat power1_cap_min` | 查看最小允许 PPT | `cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap_min` |
| `cat power1_cap_default` | 查看默认 PPT | `cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap_default` |
| `echo N > power1_cap` | 设置 PPT（N 微瓦） | `echo 250000000 > /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap` |
| `rocm-smi --showpower` | 查看功耗信息 | `rocm-smi --showpower` |
| `rocm-smi --setpowerlimit N` | 设置功耗限制 | `rocm-smi --setpowerlimit 250` |
| `cat pp_od_clk_voltage` | 查看 OverDrive 设置 | `cat /sys/class/drm/card0/device/pp_od_clk_voltage` |
| `cat amdgpu_pm_info` | 查看 PM 详细信息 | `sudo cat /sys/kernel/debug/dri/0/amdgpu_pm_info` |
| `dmesg \| grep -i power` | 查看电源管理日志 | `dmesg \| grep -i "power\|ppt\|smu"` |
| `watch -n 1 cat power1_input` | 实时监控功耗 | `watch -n 1 "cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_input"` |

## 附录K：PPT 相关术语表

| 术语 | 全称 | 中文 | 说明 |
|------|------|------|------|
| PPT | Package Power Limit | 封装功耗限制 | GPU 芯片封装的最大持续功耗 |
| TDP | Thermal Design Power | 热设计功耗 | 散热系统设计的功耗基准 |
| TDC | Thermal Design Current | 热设计电流 | VRM 最大持续电流限制 |
| EDC | Electrical Design Current | 电气设计电流 | VRM 最大瞬时电流限制 |
| SMU | System Management Unit | 系统管理单元 | GPU 内部固件微控制器 |
| VRM | Voltage Regulator Module | 电压调节模块 | 为 GPU 供电的电路 |
| V/F Curve | Voltage-Frequency Curve | 电压-频率曲线 | 频率与对应最低电压的关系曲线 |
| DPM | Dynamic Power Management | 动态电源管理 | GPU 实时功耗调节机制 |
| P-State | Performance State | 性能状态 | 频率+电压+功耗的组合状态 |
| OverDrive | OverDrive | 超频接口 | AMD GPU 超频调压用户接口 |
| Throttling | Thermal Throttling | 降频 | 温度/功耗超限时降低频率的保护机制 |
| PID | Proportional-Integral-Derivative | 比例-积分-微分 | 控制算法，用于 DVFS 调节 |
| FRL | Fuzzy Rule Logic | 模糊规则逻辑 | AMD 自适应频率调节算法 |
| IMON | Current Monitor | 电流监控 | VRM 控制器提供的电流监测信号 |
| SVI3 | Serial Voltage Interface 3 | 串行电压接口 v3 | AMD SMU-VRM 通信协议 |

## 附录L：PPT 性能基准测试数据收集与分析

### L.1 自动化基准测试脚本

```bash
#!/bin/bash
# ppt_benchmark_suite.sh
# PPT 性能基准测试套件 - 自动化收集不同 PPT 下的性能数据

set -e

GPU_ID=${1:-0}
HWMON_PATH=$(find /sys/class/drm/card${GPU_ID}/device/hwmon -name "hwmon*" | head -1)
RESULTS_DIR="ppt_benchmark_$(date +%Y%m%d_%H%M%S)"
BENCHMARK_CMD=${2:-"glmark2 --run-forever"}

mkdir -p "$RESULTS_DIR"
echo "结果保存至: $RESULTS_DIR"

# 获取 PPT 范围
PPT_DEFAULT=$(cat "$HWMON_PATH/power1_cap_default" 2>/dev/null)
PPT_MIN=$(cat "$HWMON_PATH/power1_cap_min" 2>/dev/null)
PPT_MAX=$(cat "$HWMON_PATH/power1_cap_max" 2>/dev/null)

echo "默认 PPT: $((PPT_DEFAULT/1000000))W"
echo "PPT 范围: $((PPT_MIN/1000000)) - $((PPT_MAX/1000000))W"

# 测试点
TEST_POINTS=(60 70 80 90 100 110)
BENCHMARK_DURATION=30
COOLDOWN=30

echo "基准测试配置:"
echo "  测试点: ${TEST_POINTS[*]}%"
echo "  每点测试时长: ${BENCHMARK_DURATION}s"
echo "  冷却时间: ${COOLDOWN}s"

echo "PPT_PCT,PPT_W,AVG_POWER_W,MAX_POWER_W,AVG_FREQ_MHZ,AVG_TEMP_C,PERF_SCORE" \
    > "$RESULTS_DIR/summary.csv"

for pct in "${TEST_POINTS[@]}"; do
    TEST_PPT=$((PPT_DEFAULT * pct / 100))
    TEST_PPT_W=$((TEST_PPT / 1000000))

    echo ""
    echo "=========================================="
    echo "测试: $pct% PPT = ${TEST_PPT_W}W"

    # 设置 PPT
    echo "$TEST_PPT" > "$HWMON_PATH/power1_cap"
    sleep 3

    # 运行基准测试
    echo "  运行基准测试 ${BENCHMARK_DURATION}s..."
    timeout $BENCHMARK_DURATION $BENCHMARK_CMD > /dev/null 2>&1 &
    BENCH_PID=$!

    # 收集数据
    DATA_FILE="$RESULTS_DIR/data_${pct}pct.csv"
    echo "time,power_w,freq_mhz,temp_c" > "$DATA_FILE"

    for i in $(seq 1 $BENCHMARK_DURATION); do
        power=$(cat "$HWMON_PATH/power1_input" 2>/dev/null || echo "0")
        power_w=$((power/1000000))
        temp=$(cat "$HWMON_PATH/temp1_input" 2>/dev/null || echo "0")
        temp_c=$((temp/1000))
        freq=$(cat "/sys/class/drm/card${GPU_ID}/device/pp_dpm_sclk" 2>/dev/null | \
               grep -oP '\d+Mhz' | head -1 | tr -d 'Mhz' || echo "0")
        echo "$i,$power_w,$freq,$temp_c" >> "$DATA_FILE"
        sleep 1
    done

    kill $BENCH_PID 2>/dev/null || true
    wait $BENCH_PID 2>/dev/null || true

    # 分析数据
    awk -F',' 'NR>1 {
        sum_p+=$2; sum_f+=$3; sum_t+=$4;
        if($2>max_p) max_p=$2;
        n++
    } END {
        printf "%d,%d,%.1f,%.1f,%.0f,%.1f,%d\n",
            '$pct', '$TEST_PPT_W',
            sum_p/n, max_p, sum_f/n, sum_t/n, n
    }' "$DATA_FILE" >> "$RESULTS_DIR/summary.csv"

    # 冷却
    echo "  冷却 ${COOLDOWN}s..."
    sleep $COOLDOWN
done

# 恢复默认 PPT
echo "$PPT_DEFAULT" > "$HWMON_PATH/power1_cap"
echo "已恢复默认 PPT"

# 生成 HTML 报告
echo "<html><body><h2>PPT Benchmark Report</h2>
<table border=1><tr>
<th>PPT%</th><th>PPT(W)</th><th>Avg Power(W)</th>
<th>Max Power(W)</th><th>Avg Freq(MHz)</th><th>Avg Temp(C)</th>
</tr>" > "$RESULTS_DIR/report.html"

awk -F',' 'NR>1 {
    printf "<tr><td>%s%%</td><td>%s</td><td>%s</td><td>%s</td><td>%s</td><td>%s</td></tr>\n",
        $1, $2, $3, $4, $5, $6
}' "$RESULTS_DIR/summary.csv" >> "$RESULTS_DIR/report.html"

echo "</table></body></html>" >> "$RESULTS_DIR/report.html"
echo "HTML 报告: $RESULTS_DIR/report.html"
echo "基准测试完成！"
```

### L.2 数据分析与可视化

```python
#!/usr/bin/env python3
"""
ppt_benchmark_analyzer.py
PPT 基准测试数据分析与可视化
"""

import csv
import sys
import matplotlib.pyplot as plt
import numpy as np
from pathlib import Path

class PPTBenchmarkAnalyzer:
    def __init__(self, summary_csv: str):
        self.data = []
        with open(summary_csv, 'r') as f:
            reader = csv.DictReader(f)
            for row in reader:
                self.data.append(row)

    def plot_power_perf_tradeoff(self):
        """绘制功耗-性能权衡曲线"""
        ppt_pct = [int(d['PPT_PCT']) for d in self.data]
        ppt_w = [int(d['PPT_W']) for d in self.data]
        avg_power = [float(d['AVG_POWER_W']) for d in self.data]
        max_power = [float(d['MAX_POWER_W']) for d in self.data]

        fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 5))

        # 左图：PPT vs 功耗
        ax1.plot(ppt_w, avg_power, 'b-o', label='平均功耗', linewidth=2)
        ax1.plot(ppt_w, max_power, 'r--s', label='最高功耗', linewidth=2)
        ax1.plot(ppt_w, ppt_w, 'k:', label='PPT 限制线', alpha=0.5)
        ax1.set_xlabel('PPT 限制 (W)')
        ax1.set_ylabel('实际功耗 (W)')
        ax1.set_title('PPT vs 实际功耗')
        ax1.legend()
        ax1.grid(True, alpha=0.3)

        # 右图：能效比
        perf_scores = [float(d['PERF_SCORE']) for d in self.data]
        efficiency = [p / max(pw, 1) for p, pw in zip(perf_scores, avg_power)]

        ax2.bar([str(w) for w in ppt_w], efficiency, color='green', alpha=0.7)
        ax2.set_xlabel('PPT 限制 (W)')
        ax2.set_ylabel('能效比 (Score/W)')
        ax2.set_title('不同 PPT 下的能效比')
        ax2.grid(True, alpha=0.3, axis='y')

        plt.tight_layout()
        plt.savefig('ppt_efficiency_analysis.png', dpi=150)
        print(f"图表已保存: ppt_efficiency_analysis.png")

    def find_optimal_ppt(self):
        """寻找最优 PPT 设置点"""
        best_efficiency = 0
        best_ppt = 0

        for d in self.data:
            perf = float(d['PERF_SCORE'])
            power = float(d['AVG_POWER_W'])
            efficiency = perf / max(power, 1)

            if efficiency > best_efficiency:
                best_efficiency = efficiency
                best_ppt = int(d['PPT_PCT'])

        print(f"最优能效 PPT: {best_ppt}%")
        print(f"最高能效比: {best_efficiency:.2f} Score/W")
        return best_ppt

    def generate_report(self):
        """生成分析报告"""
        print("=" * 50)
        print("PPT 基准测试分析报告")
        print("=" * 50)
        print(f"{'PPT%':<8} {'PPT(W)':<10} {'平均功耗':<10} {'最高功耗':<10} {'能效比':<10}")
        print("-" * 50)

        for d in self.data:
            perf = float(d['PERF_SCORE'])
            power = float(d['AVG_POWER_W'])
            efficiency = perf / max(power, 1)
            print(f"{d['PPT_PCT']:<8} {d['PPT_W']:<10} "
                  f"{d['AVG_POWER_W']:<10} {d['MAX_POWER_W']:<10} "
                  f"{efficiency:<10.2f}")

        optimal = self.find_optimal_ppt()
        print(f"\n推荐: 设置 PPT 为 {optimal}% 可获得最佳能效比")


if __name__ == "__main__":
    analyzer = PPTBenchmarkAnalyzer("ppt_benchmark_*/summary.csv")
    analyzer.generate_report()
    analyzer.plot_power_perf_tradeoff()
```

---

## 今日小结补充

通过第 304 天的学习，我们深入理解了：
- PPT 作为 AMD GPU 核心功耗限制机制，通过 SMU 固件实现实时监控与执行
- 双时间窗口算法（快速窗口 ~1ms + 慢速窗口 ~10ms）平衡瞬时保护和持续限制
- PPT 与 V/F 曲线、TDC、EDC 的协同工作原理
- 通过 sysfs (power1_cap)、rocm-smi、OverDrive、内核参数四种设置方法
- 多 GPU 系统中 PPT 管理的注意事项（设备实例识别、独立设置）
- 实际测试方法论：功耗限制验证、性能-功耗曲线扫描、数据驱动分析
- 动态 PPT 调整策略设计（负载感知、温度感知、优先级感知）

下一日（第 305 天）将是 **考核日**，主题为 **"GPU 频率与功耗控制实践"**，综合应用第 302-304 天所学知识。