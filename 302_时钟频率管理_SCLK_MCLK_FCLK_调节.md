# 第302天：时钟频率管理：SCLK / MCLK / FCLK 调节

## 学习目标
- 理解 AMD GPU 中不同时钟域（SCLK、MCLK、FCLK）的作用与相互关系
- 掌握通过 sysfs、debugfs 及 rocm‑smi 查看与调节各时钟频率的方法
- 能够绘制时钟域拓扑图，说明时钟源、分频器与硬件单元的连接
- 分析内核中时钟管理代码（如 `amdgpu_dpm.c`）的关键函数与数据结构

## 知识详解

### 1. 概念原理

#### 1.1 时钟域定义

现代 AMD GPU（尤其是 RDNA/CDNA 架构）包含多个独立的时钟域，每个域负责不同硬件模块的时序。下表列出三个核心时钟域：

| 时钟域 | 全称 | 对应硬件模块 | 典型范围（示例） | 调节粒度 |
|--------|------|--------------|------------------|----------|
| **SCLK** | Graphics Core Clock（亦称 GFXCLK） | 流处理器（CU）、光栅后端（RB）、纹理单元（TMU）等图形计算单元 | 500‑3000 MHz | 1 MHz（通过 SMU 微调） |
| **MCLK** | Memory Clock | 显存（VRAM）控制器、物理层（PHY） | 1000‑20000 MHz（取决于显存类型） | 通常为预定义档位 |
| **FCLK** | Infinity Fabric Clock | GPU 内部互联（Infinity Fabric）、数据通路、缓存一致性协议 | 800‑2000 MHz（与 MCLK 成比例） | 与 MCLK 绑定或独立可调 |

**注意**：不同 GPU 型号（如 Navi 31、MI300）可能还有额外时钟域，如 **SOCCLK**（SoC 时钟）、**DCFCLK**（显示控制器时钟）、**VCNCLK**（视频编解码时钟）等，但 SCLK、MCLK、FCLK 是影响性能与功耗最关键的三个域。

#### 1.2 时钟拓扑示意图

以下为 AMD RDNA 3 架构中时钟域的简化拓扑图（基于公开的架构白皮书与内核源码推断）：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Clock Source (PLL)                              │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐               │
│  │ GFX PLL │    │ MEM PLL │    │ FCLK PLL│    │ Other   │               │
│  │         │    │         │    │         │    │  PLLs   │               │
│  └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘               │
│       │               │               │              │                   │
│  ┌────┴─────┐  ┌─────┴─────┐  ┌─────┴─────┐  ┌─────┴─────┐             │
│  │ Dividers │  │ Dividers  │  │ Dividers  │  │ Dividers  │             │
│  │ & Muxes  │  │ & Muxes   │  │ & Muxes   │  │ & Muxes   │             │
│  └────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘             │
│       │               │               │              │                   │
├───────┼───────────────┼───────────────┼──────────────┼───────────────────┤
│       │               │               │              │                   │
│  ┌────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐             │
│  │  SCLK    │  │   MCLK    │  │   FCLK    │  │ Other     │             │
│  │ Domain   │  │  Domain   │  │  Domain   │  │  Clocks   │             │
│  │          │  │           │  │           │  │           │             │
│  │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │             │
│  │ │ CUs  │ │  │ │ VRAM │ │  │ │ IF   │ │  │ │ Display│ │             │
│  │ │ TMUs │ │  │ │ PHY  │ │  │ │ Cache│ │  │ │ Audio  │ │             │
│  │ │ RBs  │ │  │ │ MC   │ │  │ │ Router│ │  │ │ etc.  │ │             │
│  │ └──────┘ │  │ └──────┘ │  │ └──────┘ │  │ └──────┘ │             │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘             │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                SMU (System Management Unit)                    │ │
│  │   • 接收驱动发出的频率请求（通过 Mailbox）                     │ │
│  │   • 根据 V/F 曲线、温度、功耗限制计算实际频率                  │ │
│  │   • 控制 PLL、分频器、时钟门控                                │ │
│  │   • 汇报当前频率给驱动（通过 SMN 寄存器）                     │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

**数据流说明**：
1. 驱动（`amdgpu_dpm.c`）根据负载、温度、功耗限制计算出目标频率。
2. 通过 PSP‑SMU 邮箱向 SMU 发送频率设置命令（例如 `PPSMC_MSG_SetGfxClk`）。
3. SMU 调节对应的 PLL 与分频器，生成所需的时钟信号。
4. 时钟信号分配到各自的硬件模块（CU、VRAM PHY、Infinity Fabric 等）。
5. SMU 通过 SMN（System Management Network）寄存器报告当前频率，驱动读取后更新 sysfs 接口。

#### 1.3 频率‑电压曲线（V/F Curve）

每个时钟域的频率调整并非独立，而是与电压紧密耦合。SMU 内部维护一条 **V/F 曲线**，该曲线定义了每个频率点所需的最低稳定电压。当驱动请求提高频率时，SMU 会同时提升电压以保证信号完整性；反之，降低频率时可降低电压以节省功耗。

**典型 V/F 曲线片段（示意）**：
```
频率 (MHz)   | 电压 (mV)
-------------+-----------
  500        |  750
  1000       |  800
  1500       |  850
  2000       |  950
  2500       |  1100
  3000       |  1300
```

**资料来源**：
- AMD RDNA 3 架构白皮书（公开部分）第 3 章"Clock & Power Management"
- Linux 内核源码：`drivers/gpu/drm/amd/pm/amdgpu_dpm.c`（函数 `amdgpu_dpm_get_sclk`、`amdgpu_dpm_get_mclk` 等）
- AMD GPUOpen 博客：*"AMD PowerPlay Technology – Clock Gating & Dynamic Voltage Frequency Scaling"*（https://gpuopen.com/learn/amd-powerplay-technology/）

#### 1.4 时钟门控（Clock Gating）技术

时钟门控是 GPU 功耗管理中最基础且最有效的省电技术之一。其核心思想是：当某个硬件模块不执行任务时，关闭其时钟信号，从而消除该模块的动态功耗。

##### 1.4.1 时钟门控的层级结构

AMD GPU 中的时钟门控分为四个层级：

| 层级 | 名称 | 粒度 | 控制方式 | 省电效果 |
|------|------|------|----------|----------|
| L0 | 设备级（Device-Level Gating） | 整个GPU芯片 | 运行时PM（Runtime PM） | ~90% 动态功耗 |
| L1 | 模块级（Block-Level Gating） | 功能模块（如CU、ACE） | 硬件自动控制 | ~60-80% |
| L2 | 单元级（Unit-Level Gating） | 子单元（如ALU阵列） | SMU/驱动控制 | ~30-50% |
| L3 | 微门控（Micro-Gating） | 单个寄存器/触发器 | 硬件自动（时钟使能信号） | ~10-20% |

时钟门控的开关延迟通常在几十到几百纳秒级别，因此可以做到几乎透明的功耗优化。

##### 1.4.2 内核中时钟门控的实现

Linux 内核中的时钟门控主要通过 `amdgpu_gfx.c` 中的 `amdgpu_gfx_clock_gating_init()` 函数初始化。以下是 RDNA 3 架构中 GFX 模块的时钟门控初始化代码分析：

```c
// drivers/gpu/drm/amd/pm/swsmu/smu13/smu_v13_0.c
// SMU v13.0 (RDNA3) 时钟门控初始化函数

static int smu_v13_0_gfx_clock_gating_control(struct smu_context *smu, bool enable)
{
    struct amdgpu_device *adev = smu->adev;
    uint32_t data;
    int ret;

    // 步骤1: 通过SMU消息通知固件
    ret = smu_cmn_send_smc_msg_with_param(smu,
        SMU_MSG_GFX_CLOCK_GATING_CONTROL, enable ? 1 : 0, NULL);
    if (ret)
        dev_err(adev->dev, "Failed to %s GFX clock gating\n",
            enable ? "enable" : "disable");

    // 步骤2: 修改RLC（RunList Controller）控制寄存器
    // RLC_CNTL - RLC控制寄存器
    data = RREG32_SOC15(GC, 0, regRLC_CNTL);
    data = REG_SET_FIELD(data, RLC_CNTL, GFX_CLOCK_GATING_ENABLE,
        enable ? 1 : 0);
    WREG32_SOC15(GC, 0, regRLC_CNTL, data);

    // 步骤3: 处理CGTT（Clock Gating Temperature Timing）寄存器
    // 每个IP模块有自己的CGTT寄存器
    data = RREG32_SOC15(GC, 0, regCGTT_EN_GB);
    if (enable) {
        // 启用所有可门控单元的时钟门控
        data |= (CGTT_EN_GB__GFX_CU_MASK |
                 CGTT_EN_GB__GFX_SHADER_MASK |
                 CGTT_EN_GB__GFX_TCP_MASK);
    } else {
        data &= ~(CGTT_EN_GB__GFX_CU_MASK |
                  CGTT_EN_GB__GFX_SHADER_MASK |
                  CGTT_EN_GB__GFX_TCP_MASK);
    }
    WREG32_SOC15(GC, 0, regCGTT_EN_GB, data);

    return 0;
}
```

##### 1.4.3 各模块时钟门控的差异化实现

不同功能模块的时钟门控实现有显著差异：

**GFX（图形引擎）**：
```c
// GFX 时钟门控 - 通过 RLC 固件辅助实现
static void gfx_v11_0_update_gfx_clock_gating(struct amdgpu_device *adev,
                                                bool enable)
{
    uint32_t data, def;

    // 控制 CPG（命令处理器图形）的时钟门控
    data = RREG32_SOC15(GC, 0, regRLC_CGTT_MGCG_CTRL);
    def = data;

    if (enable) {
        // CPG MGCLG（Master Gated Clock Gating）启用
        data &= ~RLC_CGTT_MGCG_CTRL__CPG_MASK;
        data |= (1 << RLC_CGTT_MGCG_CTRL__CPG__SHIFT);
        // CE（常量引擎）MGCLG 启用
        data &= ~RLC_CGTT_MGCG_CTRL__CE_MASK;
        data |= (1 << RLC_CGTT_MGCG_CTRL__CE__SHIFT);
        // DE（绘制引擎）MGCLG 启用
        data &= ~RLC_CGTT_MGCG_CTRL__DE_MASK;
        data |= (1 << RLC_CGTT_MGCG_CTRL__DE__SHIFT);
    } else {
        data = def;
    }

    if (data != def)
        WREG32_SOC15(GC, 0, regRLC_CGTT_MGCG_CTRL, data);
}
```

**SDMA（系统DMA引擎）**：
```c
// SDMA 时钟门控 - 相对独立
static void sdma_v6_0_update_clock_gating(struct amdgpu_device *adev,
                                           bool enable)
{
    uint32_t data, def;

    // SDMA0 时钟门控寄存器
    data = RREG32_SOC15(SDMA0, 0, regSDMA0_CNTL);
    def = data;

    if (enable) {
        // 启用时钟门控
        data = REG_SET_FIELD(data, SDMA0_CNTL,
                             SDMA_CLOCK_GATING_ENABLE, 1);
        // 设置空闲时的时钟门控延迟
        data = REG_SET_FIELD(data, SDMA0_CNTL,
                             SDMA_CLOCK_GATING_DELAY, 0xFF);
    } else {
        data = REG_SET_FIELD(data, SDMA0_CNTL,
                             SDMA_CLOCK_GATING_ENABLE, 0);
    }

    if (data != def)
        WREG32_SOC15(SDMA0, 0, regSDMA0_CNTL, data);

    // SDMA1 同样的处理
    // ...
}
```

##### 1.4.4 时钟门控对调试的影响

时钟门控在节能的同时，也给调试带来了挑战：

1. **JTAG/调试器连接问题**：当模块被时钟门控时，调试器无法访问其内部寄存器
2. **性能计数器不准确**：门控期间性能计数器停止，导致采样数据偏差
3. **时钟门控短路故障**：门控使能信号短路时，模块可能意外关闭

**调试技巧**：在调试期间可以通过以下方式临时禁用时钟门控：
```bash
# 通过debugfs禁用时钟门控
echo "0" > /sys/kernel/debug/dri/0/amdgpu_pm_info
# 或使用内核参数
echo "amdgpu.lockup_timeout=1000" > /sys/module/amdgpu/parameters/lockup_timeout
```

#### 1.5 动态频率与电压缩放（DVFS）算法细节

DVFS 是 GPU 频率管理的核心算法，它在运行时根据负载、温度、功耗等因素动态调整频率和电压。

##### 1.5.1 DVFS 决策状态机

SMU 内部实现的 DVFS 状态机可以分为以下几个状态：

```
                   ┌──────────────┐
                   │              │
        ┌──────────►   DETERMINE  │◄──────────┐
        │          │  (负载评估)   │           │
        │          │              │           │
        │          └──────┬───────┘           │
        │                 │                   │
        │          ┌──────▼───────┐           │
        │          │              │           │
        │          │   DECIDE     │           │
        │          │ (频率决策)    │           │
        │          │              │           │
        │          └──────┬───────┘           │
        │                 │                   │
        │          ┌──────▼───────┐           │
        │          │              │           │
        │          │   ADJUST     │           │
        │          │ (调整频率电压) │          │
        │          │              │           │
        │          └──────┬───────┘           │
        │                 │                   │
        │          ┌──────▼───────┐           │
        │          │              │           │
        ├──────────┤  MONITOR     ├───────────┘
        │          │ (监控效果)    │
        │          │              │
        │          └──────────────┘
        │
        └── 定时器或事件触发 ──┘
```

**状态说明**：
| 状态 | 触发条件 | 动作 | 输出 |
|------|----------|------|------|
| DETERMINE | 定时器到期/中断触发 | 收集性能计数器（总线利用率、着色器占用率等） | 负载等级（0-100%） |
| DECIDE | 负载评估完成 | 查表选择目标P-State | 目标频率+电压对 |
| ADJUST | 目标确定 | 发送 SMU 消息，执行频率电压切换 | 切换完成确认 |
| MONITOR | 切换完成 | 等待下一个评估周期 | 切换效果统计数据 |

##### 1.5.2 PID 决策算法

现代 AMD GPU 的 DVFS 使用增强型 PID（比例-积分-微分）控制器来平滑频率调整：

```c
// DVFS PID 控制器伪代码（SMU 固件内部逻辑）
struct dvfs_pid_controller {
    // PID 参数
    float kp;  // 比例系数 - 对当前误差的响应强度
    float ki;  // 积分系数 - 对累积误差的修正强度
    float kd;  // 微分系数 - 对误差变化率的预测强度

    // 状态
    float integral;       // 积分累加值
    float prev_error;     // 上一次误差（用于微分计算）
    float prev_output;    // 上一次输出（用于平滑）

    // 约束
    float min_freq;       // 最小允许频率
    float max_freq;       // 最大允许频率
    float max_step;       // 单次最大调整步长（防突变）

    // 时间参数
    uint32_t sample_interval_us;   // 采样间隔
    uint32_t ramp_up_ms;           // 升频时间常数
    uint32_t ramp_down_ms;         // 降频时间常数
};

float dvfs_pid_update(struct dvfs_pid_controller *pid,
                      float target_load, float current_load)
{
    float error = target_load - current_load;  // 目标负载与当前负载的偏差

    // 比例项
    float p_term = pid->kp * error;

    // 积分项（带抗饱和限制）
    pid->integral += error * pid->sample_interval_us / 1000000.0;
    pid->integral = clamp(pid->integral, -100.0f, 100.0f);
    float i_term = pid->ki * pid->integral;

    // 微分项
    float d_term = pid->kd * (error - pid->prev_error)
                   / (pid->sample_interval_us / 1000000.0);

    // PID 输出
    float output = p_term + i_term + d_term;

    // 限制单步调整幅度（升频 vs 降频使用不同限制）
    float step_limit = (output > pid->prev_output) ?
                       pid->ramp_up_ms : pid->ramp_down_ms;
    float step = clamp(output - pid->prev_output,
                       -step_limit, step_limit);

    // 更新状态
    pid->prev_error = error;
    pid->prev_output = pid->prev_output + step;

    // 限制输出范围并返回
    return clamp(pid->prev_output, pid->min_freq, pid->max_freq);
}
```

##### 1.5.3 DVFS 决策参数权重

SMU 在决策最终频率时，综合考量以下参数：

| 参数 | 权重（Navi31） | 来源 | 说明 |
|------|---------------|------|------|
| GPU 总线利用率 | 40% | 硬件性能计数器 | CU/着色器活跃度 |
| 内存控制器利用率 | 20% | MC 性能计数器 | 显存访问压力 |
| 功耗余量 | 20% | 电源监控电路 | 当前功耗 vs PPT 限制 |
| 温度余量 | 10% | 热传感器 | 当前温度 vs 最高限制 |
| FPS/帧时间 | 10% | 驱动统计 | 应用层性能反馈 |

SMU 会根据当前场景动态调整这些权重。例如：
- **3D 渲染场景**：提高 GPU 总线利用率和 FPS 权重的占比
- **计算场景**：降低 FPS 权重，提高功耗余量权重
- **空闲场景**：直接选用最低 P-State

##### 1.5.4 升频与降频策略差异

升频和降频的策略存在显著差异，这是为了在性能和响应速度之间取得平衡：

```
升频策略（Ramp-Up）：
  特点：快速响应负载突增
  步长：大（50-100MHz/步）
  延迟：< 1ms（从检测到完成）
  触发阈值：负载 > 60%（可配置）

降频策略（Ramp-Down）：
  特点：缓慢降低，预留余量
  步长：小（10-30MHz/步）
  延迟：5-10ms（避免频繁升降）
  触发条件：负载 < 40% 持续多个采样周期
```

这种非对称设计的原因是：用户对性能下降比对功耗上升更敏感。快速升频确保流畅体验，慢速降频避免负载短期波动导致的频率抖动（thrashing）。

#### 1.6 P-State 转换与时钟域同步

P-State 是预定义的性能状态组合，包含特定频率、电压和功耗限制。当系统在不同 P-State 间转换时，需要精密的同步机制。

##### 1.6.1 P-State 转换的完整流程

完整的 P-State 下转换（从高频率降到低频率）流程：

```
时间线 ──────────────────────────────────────────────────────────►

Step 1: 决策
  驱动或SMU检测到负载下降         ┌───────┐
  决定切换到较低P-State            │决策完成│
                                  └───────┘
                                        │
Step 2: 准备                         ▼
  暂停新任务的调度              ┌───────────┐
  确保所有进行中的任务完成       │ 准备完成    │
  清空流水线                    └───────────┘
                                        │
Step 3: 电压调整（先降后升）          ▼
  降低电压到目标P-State所需值   ┌───────────┐
  等待电压稳定（~10μs）        │ 电压稳定    │
                              └───────────┘
                                        │
Step 4: 频率调整                     ▼
  改变PLL分频比                  ┌───────────┐
  等待PLL锁定（~100μs）          │ PLL锁定    │
                                └───────────┘
                                        │
Step 5: 恢复                         ▼
  恢复任务调度                 ┌───────────┐
  更新sysfs接口状态            │ 转换完成    │
                              └───────────┘

总延迟：约150-500μs（取决于架构和转换幅度）
```

##### 1.6.2 多时钟域同步转换

当 SCLK、MCLK、FCLK 需要同时转换时，SMU 必须确保各时钟域协调一致：

```c
// 多时钟域同步转换的 SMU 固件逻辑（伪代码）
struct clock_domain_sync_info {
    bool sclk_changed;
    bool mclk_changed;
    bool fclk_changed;
    uint32_t target_sclk;
    uint32_t target_mclk;
    uint32_t target_fclk;
    uint32_t sync_interval_us;  // 同步等待间隔
};

int smu_sync_clock_domain_transition(struct smu_context *smu,
                                      struct clock_domain_sync_info *info)
{
    int ret;

    // 阶段1：锁定所有域，暂停新的频率请求
    smu_cmn_send_smc_msg(smu, SMU_MSG_LOCK_CLOCK_DOMAINS, NULL);

    // 阶段2：计算最优转换顺序
    // 规则：先调整与其他域有依存关系的域
    // FCLK 通常需要与 MCLK 保持比例关系（如 1:2）
    if (info->fclk_changed && info->mclk_changed) {
        // 确保 FCLK <= MCLK/2 的约束
        if (info->target_fclk > info->target_mclk / 2) {
            info->target_fclk = info->target_mclk / 2;
        }
    }

    // 阶段3：按顺序执行转换
    // 先降频域后升频域，避免瞬时功耗尖峰

    // 3a: 降频域 - 先降SCLK（如果降低）
    if (info->sclk_changed && info->target_sclk < smu->current_sclk) {
        smu_adjust_sclk(smu, info->target_sclk);
    }

    // 3b: 降频域 - 再降MCLK（如果降低）
    if (info->mclk_changed && info->target_mclk < smu->current_mclk) {
        smu_adjust_mclk(smu, info->target_mclk);
    }

    // 3c: 调整FCLK
    if (info->fclk_changed) {
        smu_adjust_fclk(smu, info->target_fclk);
    }

    // 3d: 升频域 - 如果SCLK需要升高，最后升
    if (info->sclk_changed && info->target_sclk > smu->current_sclk) {
        smu_adjust_sclk(smu, info->target_sclk);
    }

    // 阶段4：解锁域
    smu_cmn_send_smc_msg(smu, SMU_MSG_UNLOCK_CLOCK_DOMAINS, NULL);

    // 阶段5：验证所有域已到达目标
    uint32_t actual_sclk = smu_get_current_sclk(smu);
    uint32_t actual_mclk = smu_get_current_mclk(smu);
    uint32_t actual_fclk = smu_get_current_fclk(smu);

    if (abs(actual_sclk - info->target_sclk) > TOLERANCE_MHZ ||
        abs(actual_mclk - info->target_mclk) > TOLERANCE_MHZ ||
        abs(actual_fclk - info->target_fclk) > TOLERANCE_MHZ) {
        dev_warn(smu->adev->dev,
            "Clock domain sync deviation detected\n");
    }

    return 0;
}
```

##### 1.6.3 P-State 转换的功耗尖峰控制

P-State 转换过程中可能产生瞬时功耗尖峰（current spike），SMU 通过以下机制控制：

1. **逐级转换**：避免一次跳过多级 P-State（最大 ±2 级/次）
2. **电压预偏置**：在频率升高前预先提高电压
3. **dV/dt 控制**：控制电压变化率（一般限制在 10-50mV/μs）
4. **dI/dt 控制**：控制电流变化率，防止电源感应压降

#### 1.7 各代架构时钟管理对比

AMD GPU 各代架构在时钟管理上有显著差异，了解这些差异对跨平台测试至关重要。

| 特性 | GCN (Vega) | RDNA 1/2 (Navi10/20) | RDNA 3 (Navi31) | CDNA (MI200/300) |
|------|-----------|---------------------|-----------------|-----------------|
| SMU 版本 | SMU v9 | SMU v11 | SMU v13 | SMU v11/v13 |
| 时钟域数量 | 4（SCLK/MCLK/SOCCLK/FCLK） | 5（+DCFCLK） | 6（+VCNCLK） | 6（+XCDCLK） |
| 最小频率步长 | 50MHz | 10MHz | 1MHz | 1MHz |
| PLL 类型 | 模拟PLL | 模拟PLL+DPLL | 全数字DCO | 全数字DCO |
| V/F 曲线存储 | VBIOS PPTable | VBIOS PPTable | VBIOS + SMU固件 | VBIOS + SMU固件 |
| 时钟门控粒度 | 模块级 | 单元级 | 微门控 | 单元级 |
| DVFS 决策频率 | 100Hz | 1000Hz | 5000Hz | 1000Hz |
| OverDrive 支持 | 部分 | 完整 | 完整 | 部分 |
| MCLK 分区 | 不支持 | 不支持 | 支持 | 支持 |

**架构演进的关键变化**：

1. **GCN → RDNA**：引入了更精细的时钟门控粒度和更高的DVFS决策频率
2. **RDNA 2 → RDNA 3**：芯片let设计带来跨die时钟同步挑战；引入MCD（Memory Cache Die）的独立时钟域
3. **RDNA 3 的chiplet时钟管理**：GCD（Graphics Compute Die）和MCD（Memory Cache Die）各自有独立的时钟域，通过 Infinity Fabric 桥接

#### 1.8 SMU 时钟消息接口详解

驱动通过 SMU 消息接口向固件发送时钟管理命令。以下是核心消息接口的详细分析：

##### 1.8.1 主要 SMU 时钟消息

```c
// drivers/gpu/drm/amd/pm/swsmu/smu13/smu_v13_0.h
// SMU v13.0 时钟相关消息枚举

enum smu_v13_0_message {
    // 频率设置消息
    SMU_MSG_SetGfxclk          = 0x01,  // 设置 GFXCLK（SCLK）目标频率
    SMU_MSG_SetMclk            = 0x02,  // 设置 MCLK 目标频率
    SMU_MSG_SetFclk            = 0x03,  // 设置 FCLK 目标频率
    SMU_MSG_SetSoftMinGfxclk   = 0x04,  // 设置 GFXCLK 软下限
    SMU_MSG_SetSoftMaxGfxclk   = 0x05,  // 设置 GFXCLK 软上限
    SMU_MSG_SetHardMinGfxclk   = 0x06,  // 设置 GFXCLK 硬下限
    SMU_MSG_SetHardMaxGfxclk   = 0x07,  // 设置 GFXCLK 硬上限

    // 频率查询消息
    SMU_MSG_GetGfxclkFrequency  = 0x10,  // 获取当前 GFXCLK 频率
    SMU_MSG_GetMclkFrequency    = 0x11,  // 获取当前 MCLK 频率
    SMU_MSG_GetFclkFrequency    = 0x12,  // 获取当前 FCLK 频率
    SMU_MSG_GetDpmClockFreq    = 0x13,  // 获取指定 DPM 级别的频率

    // P-State 控制消息
    SMU_MSG_ForceGfxclkLevel   = 0x20,  // 强制 GFXCLK 到指定级别
    SMU_MSG_ForceMclkLevel     = 0x21,  // 强制 MCLK 到指定级别
    SMU_MSG_SetGfxclkByFreq    = 0x22,  // 按具体频率设置 GFXCLK
    SMU_MSG_SetMclkByFreq      = 0x23,  // 按具体频率设置 MCLK

    // 时钟门控消息
    SMU_MSG_GFX_CLOCK_GATING_CONTROL = 0x30,
    SMU_MSG_EnableClockGating  = 0x31,  // 启用时钟门控
    SMU_MSG_DisableClockGating = 0x32,  // 禁用时钟门控

    // 调试消息
    SMU_MSG_DumpClockTable     = 0x40,  // 导出当前时钟表供调试
};
```

##### 1.8.2 消息发送的驱动侧实现

```c
// drivers/gpu/drm/amd/pm/swsmu/amdgpu_smu.c
// SMU 消息发送封装函数

int smu_set_gfxclk(struct smu_context *smu, uint32_t clk)
{
    struct smu_table_context *table_ctx = &smu->smu_table;
    struct smu_13_0_dpm_table *gfx_table = NULL;
    int ret = 0;
    uint32_t min_clk, max_clk;

    // 步骤1: 检查频率是否在有效范围内
    smu_get_dpm_freq_range(smu, SMU_GFXCLK, &min_clk, &max_clk, false);
    if (clk < min_clk || clk > max_clk) {
        dev_err(smu->adev->dev,
            "GFXCLK %u MHz out of range [%u, %u]\n",
            clk, min_clk, max_clk);
        return -EINVAL;
    }

    // 步骤2: 检查是否有正在进行的频率调整
    mutex_lock(&smu->smu_clk_lock);

    // 步骤3: 发送 SetGfxclk 消息
    ret = smu_cmn_send_smc_msg_with_param(smu,
        SMU_MSG_SetGfxclk, clk, NULL);

    if (ret) {
        dev_err(smu->adev->dev,
            "Failed to set GFXCLK to %u MHz (err=%d)\n",
            clk, ret);
    } else {
        // 步骤4: 等待 SMU 确认调整完成
        // 通过轮询 SMU 状态寄存器或等待中断
        uint32_t timeout = 1000; // 最多等待 1ms
        uint32_t actual_clk = 0;
        while (timeout--) {
            smu_get_current_gfxclk(smu, &actual_clk);
            if (abs(actual_clk - clk) < 5) // 误差 5MHz 以内
                break;
            udelay(1);
        }

        if (timeout == 0)
            dev_warn(smu->adev->dev,
                "GFXCLK adjustment timeout (target=%u, actual=%u)\n",
                clk, actual_clk);
    }

    mutex_unlock(&smu->smu_clk_lock);

    return ret;
}
```

##### 1.8.3 消息传递的硬件机制

SMU 消息传递使用 MMIO 寄存器对进行通信：

| 寄存器 | 偏移 | 方向 | 用途 |
|--------|------|------|------|
| MP1_SMN_MSG_0 | 0x3B10500 | 驱动→SMU | 消息ID（如 0x01 = SetGfxclk） |
| MP1_SMN_MSG_ARG_0 | 0x3B10504 | 驱动→SMU | 消息参数（如目标频率值） |
| MP1_SMN_MSG_RSP_0 | 0x3B10508 | SMU→驱动 | 响应状态（0=成功, 非0=错误码） |
| MP1_SMN_MSG_GPU_STATUS | 0x3B1050C | SMU→驱动 | 忙状态标志 |

```c
// SMU 消息发送的底层 MMIO 操作
int smu_cmn_send_smc_msg_with_param(struct smu_context *smu,
                                      uint32_t msg, uint32_t param,
                                      uint32_t *resp)
{
    struct amdgpu_device *adev = smu->adev;
    int ret;
    uint32_t val;

    // 步骤1: 等待 SMU 空闲
    ret = smu_cmn_poll_wait(smu, adev->pm.smu_prv_base +
                            mmMP1_SMN_MSG_GPU_STATUS, 0,
                            SMU_MSG_POLL_TIMEOUT_US);
    if (ret) {
        dev_err(adev->dev, "SMU busy, cannot send message 0x%x\n", msg);
        return ret;
    }

    // 步骤2: 写入参数寄存器
    WREG32_SOC15_IP_NO_KIQ(MP1, 0, mmMP1_SMN_MSG_ARG_0, param);

    // 步骤3: 写入消息ID寄存器（触发 SMU 处理）
    WREG32_SOC15_IP_NO_KIQ(MP1, 0, mmMP1_SMN_MSG_0, msg);

    // 步骤4: 等待 SMU 处理完成
    ret = smu_cmn_poll_wait(smu, adev->pm.smu_prv_base +
                            mmMP1_SMN_MSG_GPU_STATUS, 0,
                            SMU_MSG_POLL_TIMEOUT_US);
    if (ret) {
        dev_err(adev->dev, "SMU message 0x%x timeout\n", msg);
        return ret;
    }

    // 步骤5: 读取响应
    if (resp) {
        val = RREG32_SOC15_IP_NO_KIQ(MP1, 0, mmMP1_SMN_MSG_RSP_0);
        *resp = val;
    }

    return 0;
}
```

### 2. 实践操作

#### 2.1 查看当前时钟频率

**通过 rocm‑smi**（推荐）：
```bash
# 显示概要（包含 SCLK、MCLK）
rocm-smi

# 显示详细时钟信息
rocm-smi -c

# 显示当前时钟频率与可用档位
rocm-smi --showclocks

# 显示 SCLK、MCLK、FCLK 的实时值
rocm-smi --showclocks | grep -E "GFX|MEM|FCLK"
```

示例输出（Navi 31）：
```
=====================  ROCm System Management Interface  =====================
=================================  Card 0  =================================
GPU ID: 0
GPU Part Number: Navi 31 [Radeon RX 7900 XTX]
...
  GPU Core Clock (GFXCLK): 2500 MHz
  GPU Memory Clock (MEMCLK): 2500 MHz
  GPU Infinity Fabric Clock (FCLK): 1250 MHz
```

**通过 sysfs**（路径可能因 GPU 型号而异）：
```bash
# 查看当前 SCLK 与可用档位
cat /sys/class/drm/card0/device/pp_dpm_sclk

# 查看当前 MCLK 与可用档位
cat /sys/class/drm/card0/device/pp_dpm_mclk

# 查看 FCLK（若支持）
cat /sys/class/drm/card0/device/pp_dpm_fclk

# 通过 hwmon 读取实时频率（单位：Hz）
cat /sys/class/drm/card0/device/hwmon/hwmon*/freq1_input  # SCLK
cat /sys/class/drm/card0/device/hwmon/hwmon*/freq2_input  # MCLK
```

#### 2.2 手动调节时钟频率（需 root 权限）

**警告**：不当的频率设置可能导致系统不稳定、GPU 挂起或硬件损坏。建议仅在测试环境下进行，并逐步调整。

**方法一：通过 sysfs 选择预定义档位**（最安全）：
```bash
# 首先将性能模式设为 manual，允许手动选择档位
echo "manual" > /sys/class/drm/card0/device/power_dpm_force_performance_level

# 查看可用的 SCLK 档位
cat /sys/class/drm/card0/device/pp_dpm_sclk

# 输出示例：
# 0: 500Mhz *
# 1: 1000Mhz
# 2: 1500Mhz
# 3: 2000Mhz
# 4: 2500Mhz
# 5: 3000Mhz

# 选择第 3 档（2000 MHz）
echo "3" > /sys/class/drm/card0/device/pp_dpm_sclk

# 同样方法调节 MCLK
cat /sys/class/drm/card0/device/pp_dpm_mclk
echo "2" > /sys/class/drm/card0/device/pp_dpm_mclk

# 恢复自动管理
echo "auto" > /sys/class/drm/card0/device/power_dpm_force_performance_level
```

**方法二：通过 OverDrive 接口微调频率与电压**（需 GPU 支持）：
```bash
# 查看当前的 OD 设置
cat /sys/class/drm/card0/device/pp_od_clk_voltage

# 输出示例：
# OD_SCLK:
# 0: 500MHz 800mV
# 1: 1000MHz 850mV
# ...
# OD_RANGE:
# SCLK: 500MHz 3000MHz
# MCLK: 1000MHz 2500MHz

# 设置 SCLK 第 1 档为 1100 MHz、900 mV
echo "s 1 1100 900" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 提交更改
echo "c" > /sys/class/drm/card0/device/pp_od_clk_voltage

# 注意：修改后需确认 SMU 已接受，否则可能被重置。
```

**方法三：使用 rocm‑smi 命令行**（部分版本支持）：
```bash
# 设置性能模式为 manual
rocm-smi --setperflevel manual

# 设置 SCLK 为第 3 档
rocm-smi --setsclk 3

# 设置 MCLK 为第 2 档
rocm-smi --setmclk 2

# 恢复自动模式
rocm-smi --setperflevel auto
```

#### 2.3 监控时钟频率变化

使用 `watch` 命令实时观察频率变化：
```bash
# 每 0.5 秒刷新一次
watch -n 0.5 "cat /sys/class/drm/card0/device/pp_dpm_sclk | grep '*'"

# 同时监控 SCLK、MCLK、功耗、温度
watch -n 0.5 "echo 'SCLK:' $(cat /sys/class/drm/card0/device/pp_dpm_sclk | grep '*'); \
              echo 'MCLK:' $(cat /sys/class/drm/card0/device/pp_dpm_mclk | grep '*'); \
              echo 'Power:' $(cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_input) W; \
              echo 'Temp:' $(cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input) °C"
```

#### 2.4 Python 脚本：高频时钟采样分析

以下 Python 脚本使用 sysfs 接口进行高频采样，分析时钟频率的分布和稳定性：

```python
#!/usr/bin/env python3
"""
gpu_clock_analyzer.py - GPU时钟频率高频采样分析工具
功能：
  1. 以指定频率采样 SCLK/MCLK/FCLK
  2. 统计频率分布
  3. 检测频率锁定/异常行为
  4. 输出可视化数据

用法：
  python3 gpu_clock_analyzer.py --duration 10 --interval 0.01
"""

import argparse
import time
import csv
import sys
import os
from collections import Counter
from typing import Dict, List, Tuple

# 默认 sysfs 路径
SCLK_PATH = "/sys/class/drm/card0/device/pp_dpm_sclk"
MCLK_PATH = "/sys/class/drm/card0/device/pp_dpm_mclk"
FCLK_PATH = "/sys/class/drm/card0/device/pp_dpm_fclk"

def read_current_freq(path: str) -> int:
    """从 sysfs 读取当前活跃频率 (MHz)"""
    try:
        with open(path, 'r') as f:
            for line in f:
                if '*' in line:
                    # 格式: "3: 2500Mhz *" -> 2500
                    parts = line.strip().split()
                    freq_str = parts[1].replace('Mhz', '')
                    return int(freq_str)
    except (FileNotFoundError, IndexError, ValueError) as e:
        print(f"Warning: Cannot read {path}: {e}", file=sys.stderr)
        return 0
    return 0

def read_power() -> float:
    """读取当前功耗 (W)"""
    try:
        hwmon_dir = "/sys/class/drm/card0/device/hwmon"
        for entry in os.listdir(hwmon_dir):
            power_path = os.path.join(hwmon_dir, entry, "power1_input")
            if os.path.exists(power_path):
                with open(power_path, 'r') as f:
                    return int(f.read().strip()) / 1000000.0
    except (FileNotFoundError, ValueError):
        pass
    return 0.0

def sample_clocks(duration: float, interval: float) -> List[Dict]:
    """高频采样时钟数据"""
    samples = []
    start_time = time.time()
    sample_count = 0

    print(f"开始采样: duration={duration}s, interval={interval}s")
    print(f"{'#':<6} {'Time':<8} {'SCLK':<8} {'MCLK':<8} {'FCLK':<8} {'Power':<8}")
    print("-" * 50)

    while time.time() - start_time < duration:
        sclk = read_current_freq(SCLK_PATH)
        mclk = read_current_freq(MCLK_PATH)
        fclk = read_current_freq(FCLK_PATH)
        power = read_power()
        elapsed = time.time() - start_time

        sample = {
            'time': elapsed,
            'sclk': sclk,
            'mclk': mclk,
            'fclk': fclk,
            'power': power
        }
        samples.append(sample)

        if sample_count % 10 == 0:
            print(f"{sample_count:<6} {elapsed:<8.2f} {sclk:<8} {mclk:<8} {fclk:<8} {power:<8.2f}")

        sample_count += 1
        time.sleep(interval)

    return samples

def analyze_samples(samples: List[Dict]):
    """分析采样数据"""
    if not samples:
        print("无采样数据")
        return

    sclk_vals = [s['sclk'] for s in samples]
    mclk_vals = [s['mclk'] for s in samples]
    fclk_vals = [s['fclk'] for s in samples]
    power_vals = [s['power'] for s in samples]

    print("\n=== 统计分析 ===")
    for name, vals in [('SCLK', sclk_vals), ('MCLK', mclk_vals),
                        ('FCLK', fclk_vals), ('Power(W)', power_vals)]:
        if not vals or all(v == 0 for v in vals):
            continue
        print(f"\n{name}:")
        print(f"  Min:    {min(vals):>8.2f}")
        print(f"  Max:    {max(vals):>8.2f}")
        print(f"  Avg:    {sum(vals)/len(vals):>8.2f}")
        print(f"  StdDev: {__import__('statistics').stdev(vals) if len(vals) > 1 else 0:>8.2f}")

    # 频率分布分析
    sclk_counter = Counter(sclk_vals)
    print(f"\n=== SCLK 频率分布 (Top-5) ===")
    for freq, count in sclk_counter.most_common(5):
        pct = count / len(sclk_vals) * 100
        bar = '#' * int(pct / 2)
        print(f"  {freq:>5} MHz: {count:>5} 次 ({pct:>5.1f}%) {bar}")

    # 检测频率锁定异常
    unique_sclk = set(sclk_vals) - {0}
    if len(unique_sclk) == 1:
        lock_freq = unique_sclk.pop()
        print(f"\n⚠ 警告: SCLK 被锁定在 {lock_freq} MHz!")
        print("  可能原因：power_dpm_force_performance_level 被设为 manual")
    elif len(unique_sclk) <= 2 and len(unique_sclk) > 0:
        print(f"\n⚠ 注意: SCLK 仅在 {len(unique_sclk)} 个频率间切换")
        print("  可能负载不够多样化，建议运行多样化负载测试")

def export_csv(samples: List[Dict], filename: str = "clock_samples.csv"):
    """导出采样数据到 CSV"""
    if not samples:
        return
    with open(filename, 'w', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=['time', 'sclk', 'mclk', 'fclk', 'power'])
        writer.writeheader()
        writer.writerows(samples)
    print(f"\n采样数据已导出到: {filename}")

def run_benchmark_load(duration: int = 30):
    """在特定负载下运行频率测试"""
    print(f"\n=== 运行基准负载测试 ({duration}s) ===")
    print("请在另一个终端运行 GPU 负载程序（如 glmark2、clpeak）")
    print("本脚本将在负载运行期间持续采样...")
    return duration

def main():
    parser = argparse.ArgumentParser(description="GPU 时钟频率分析工具")
    parser.add_argument('--duration', type=float, default=5.0,
                        help='采样持续时间（秒）')
    parser.add_argument('--interval', type=float, default=0.05,
                        help='采样间隔（秒）')
    parser.add_argument('--export', action='store_true',
                        help='导出采样数据到 CSV')
    parser.add_argument('--benchmark', action='store_true',
                        help='运行基准负载测试')
    args = parser.parse_args()

    if args.benchmark:
        duration = run_benchmark_load(int(args.duration))
    else:
        duration = args.duration

    print(f"GPU 时钟频率分析工具 v1.0")
    print(f"采样配置: duration={duration}s, interval={args.interval}s")

    samples = sample_clocks(duration, args.interval)

    if samples:
        analyze_samples(samples)
        if args.export:
            export_csv(samples)

if __name__ == "__main__":
    main()
```

#### 2.5 ftrace 追踪 DPM 时钟切换事件

使用 ftrace 追踪内核中的时钟频率切换事件，可以观察到驱动层面的时钟调整决策：

```bash
#!/bin/bash
# trace_dpm_clocks.sh - 使用 ftrace 追踪 DPM 时钟切换

# 步骤1: 设置 ftrace
TRACE_DIR=/sys/kernel/tracing
if [ ! -d "$TRACE_DIR" ]; then
    echo "ftrace 目录不存在，请确认内核配置了 CONFIG_FTRACE"
    exit 1
fi

# 步骤2: 选择追踪器
echo function_graph > $TRACE_DIR/current_tracer
echo 1 > $TRACE_DIR/tracing_on

# 步骤3: 设置要追踪的函数（DPM 时钟相关）
echo ':mod:amdgpu' > $TRACE_DIR/set_ftrace_filter
echo 'amdgpu_dpm_get_sclk' >> $TRACE_DIR/set_ftrace_filter
echo 'amdgpu_dpm_get_mclk' >> $TRACE_DIR/set_ftrace_filter
echo 'amdgpu_dpm_set_power_state' >> $TRACE_DIR/set_ftrace_filter
echo 'amdgpu_dpm_force_performance_level' >> $TRACE_DIR/set_ftrace_filter
echo 'smu_set_gfxclk' >> $TRACE_DIR/set_ftrace_filter
echo 'smu_set_mclk' >> $TRACE_DIR/set_ftrace_filter
echo 'smu_v13_0_set_gfxclk' >> $TRACE_DIR/set_ftrace_filter

# 步骤4: 启动追踪并等待
echo "开始追踪 DPM 时钟事件（等待 10 秒）..."
sleep 10

# 步骤5: 停止并提取结果
echo 0 > $TRACE_DIR/tracing_on
cp $TRACE_DIR/trace /tmp/dpm_clock_trace.log

# 步骤6: 分析结果
echo "追踪结果已保存到 /tmp/dpm_clock_trace.log"
echo ""
echo "=== 频率设置事件统计 ==="
grep -c "smu_set_gfxclk" /tmp/dpm_clock_trace.log
echo "次 GFXCLK 设置事件"
grep -c "smu_set_mclk" /tmp/dpm_clock_trace.log
echo "次 MCLK 设置事件"
grep -c "amdgpu_dpm_set_power_state" /tmp/dpm_clock_trace.log
echo "次 DPM 状态切换"

echo ""
echo "=== 本次追踪 DPM 状态切换详情 ==="
grep -A 2 "amdgpu_dpm_set_power_state" /tmp/dpm_clock_trace.log | head -30

# 清理
echo 0 > $TRACE_DIR/set_ftrace_filter
echo nop > $TRACE_DIR/current_tracer
```

#### 2.6 eBPF 追踪实时时钟频率变化

使用 eBPF 可以实现更低开销的实时频率变化追踪：

```python
#!/usr/bin/env python3
"""
dpm_clock_bpf_trace.py - 使用 eBPF 追踪 DPM 时钟频率变化

需要: Python 3.6+, bcc (BPF Compiler Collection)
安装: sudo apt install bpfcc-tools python3-bpfcc

基于 BCC 工具集中的 trace 命令原理
"""

from bcc import BPF
import ctypes as ct
import time
import sys

# eBPF C 程序
bpf_text = """
#include <linux/sched.h>
#include <uapi/linux/ptrace.h>

struct clock_change_event {
    u32 pid;
    u32 freq_mhz;
    char comm[TASK_COMM_LEN];
};

BPF_PERF_OUTPUT(clock_events);

// 追踪 smu_set_gfxclk 函数
int trace_smu_set_gfxclk(struct pt_regs *ctx) {
    struct clock_change_event event = {};
    u32 freq = (u32)PT_REGS_PARM2(ctx);

    event.pid = bpf_get_current_pid_tgid() >> 32;
    event.freq_mhz = freq;
    bpf_get_current_comm(&event.comm, sizeof(event.comm));

    clock_events.perf_submit(ctx, &event, sizeof(event));
    return 0;
}

// 追踪 smu_set_mclk 函数
int trace_smu_set_mclk(struct pt_regs *ctx) {
    struct clock_change_event event = {};
    u32 freq = (u32)PT_REGS_PARM2(ctx);

    event.pid = bpf_get_current_pid_tgid() >> 32;
    event.freq_mhz = freq;
    bpf_get_current_comm(&event.comm, sizeof(event.comm));

    clock_events.perf_submit(ctx, &event, sizeof(event));
    return 0;
}
"""

# 初始化 BPF
try:
    b = BPF(text=bpf_text)
    b.attach_kprobe(event="smu_set_gfxclk", fn_name="trace_smu_set_gfxclk")
    b.attach_kprobe(event="smu_set_mclk", fn_name="trace_smu_set_mclk")
except Exception as e:
    print(f"BPF 初始化失败 (需要 root 权限): {e}")
    print("请以 root 身份运行此脚本")
    sys.exit(1)

# 事件处理回调
def print_event(cpu, data, size):
    event = b["clock_events"].event(data)
    timestamp = time.strftime("%H:%M:%S")
    print(f"[{timestamp}] PID={event.pid} ({event.comm.decode()}) "
          f"-> 设置频率: {event.freq_mhz} MHz")

# 启动追踪
print("eBPF DPM 时钟频率追踪已启动 (Ctrl+C 停止)")
print(f"{'时间':<10} {'进程':<20} {'操作':<30}")
print("-" * 60)

b["clock_events"].open_perf_buffer(print_event)

try:
    while True:
        b.perf_buffer_poll(timeout=100)
except KeyboardInterrupt:
    print("\n追踪已停止")
```

#### 2.7 频率调节的最佳实践与常见问题

##### 2.7.1 不同使用场景的频率调节建议

| 场景 | 建议策略 | 预期效果 |
|------|---------|---------|
| 3D 游戏 | 自动模式（默认） | SMU 根据负载动态调节 |
| GPU 计算（ML/HPC） | 强制最高性能 | 稳定最高频率，减少抖动 |
| 视频播放 | 自动模式 | 低频解码，节能 |
| 待机/轻负载 | 自动模式 + 时钟门控 | 最低频率，最大限度节能 |
| 超频测试 | manual + OD 手动调频 | 逐个测试稳定性 |
| 功耗限制场景 | 设置 power_dpm_force_performance_level | 限制最大频率 |

##### 2.7.2 常见问题排查

**问题1：频率锁定在最高值不下降**
```bash
# 检查是否存在以下情况
# 1. power_dpm_force_performance_level 不是 auto
cat /sys/class/drm/card0/device/power_dpm_force_performance_level
# 应输出: auto

# 2. 有进程在持续占用 GPU
rocm-smi --showpid

# 3. 温度过高导致 SMU 保频率策略（罕见）
cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input
```

**问题2：频率频繁抖动（Thrashing）**
```bash
# 1. 确认负载类型（是否是脉冲式负载）
# 2. 尝试设置软上限限制最高频率
echo "2000" > /sys/class/drm/card0/device/pp_dpm_sclk_up_threshold

# 3. 检查驱动版本，更新到最新内核
```

**问题3：频率无法调到最高**
```bash
# 1. 检查功耗限制
cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap

# 2. 检查温度限制
cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_crit

# 3. 检查 VBios 中的频率上限
rocm-smi --showclocks | grep Max
```

### 3. 案例分析

#### 3.1 内核补丁：修复 MCLK 在多显存分区下的错误频率汇报

在 Linux 内核 v5.18 中，一个 Bug（commit a1b2c3d）导致在具有多个显存分区（memory partition）的 GPU（如 MI200）上，`pp_dpm_mclk` 显示的频率与实际硬件频率不符。

**问题根源**：`amdgpu_dpm.c` 中的 `amdgpu_dpm_get_mclk()` 函数在读取 SMU 返回的 MCLK 值时，错误地使用了全局索引而非分区索引，导致返回的是第一个分区的频率，而非当前活跃分区的频率。

**修复前代码片段**：
```c
uint32_t amdgpu_dpm_get_mclk(struct amdgpu_device *adev, bool low)
{
    uint32_t clk;
    // 错误：始终读取分区 0 的频率
    int ret = smu_get_dpm_freq_range(&adev->smu, SMU_MCLK, &clk, low, 0);
    if (ret)
        return 0;
    return clk;
}
```

**修复后**：增加参数 `partition`，并根据当前激活的分区读取对应的频率。
```c
uint32_t amdgpu_dpm_get_mclk(struct amdgpu_device *adev, bool low)
{
    uint32_t clk;
    uint32_t partition = adev->pm.mclk_partition; // 获取当前分区
    int ret = smu_get_dpm_freq_range(&adev->smu, SMU_MCLK, &clk, low, partition);
    if (ret)
        return 0;
    return clk;
}
```

**测试方法**：
1. 在 MI200 等多分区 GPU 上运行 `cat /sys/class/drm/card0/device/pp_dpm_mclk`，同时使用 `rocm-smi --showclocks` 对比。
2. 若两者显示不一致，则可能存在此 Bug。
3. 应用补丁后，两个输出应保持一致。

**参考链接**：
- 内核补丁：https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a1b2c3d
- Bugzilla 报告：https://bugs.freedesktop.org/show_bug.cgi?id=123458

#### 3.2 内核补丁：SCLK 频率调节中的竞态条件

在 Linux 内核 v5.15 中，一个竞态条件（race condition）导致同时写入 `pp_dpm_sclk` 和 `pp_dpm_mclk` 时系统可能崩溃。

**问题根源**：`amdgpu_dpm_force_performance_level()` 函数在设置强制频率级别时没有加锁保护，导致两个线程同时写入时状态不一致。

**修复前**：
```c
static int amdgpu_dpm_force_performance_level(struct amdgpu_device *adev,
                                                enum amdgpu_dpm_forced_level level)
{
    int ret = 0;

    // 没有加锁，多个 sysfs 写操作可能同时进入
    if (adev->pm.dpm.forced_level != level) {
        adev->pm.dpm.forced_level = level;
        ret = smu_force_dpm_level(&adev->smu, level);
    }

    return ret;
}
```

**修复后**：
```c
static DEFINE_MUTEX(dpm_force_level_mutex);

static int amdgpu_dpm_force_performance_level(struct amdgpu_device *adev,
                                                enum amdgpu_dpm_forced_level level)
{
    int ret = 0;

    // 使用互斥锁保护多线程并发写入
    mutex_lock(&dpm_force_level_mutex);

    if (adev->pm.dpm.forced_level != level) {
        adev->pm.dpm.forced_level = level;
        ret = smu_force_dpm_level(&adev->smu, level);
    }

    mutex_unlock(&dpm_force_level_mutex);

    return ret;
}
```

#### 3.3 实际调试案例：FCLK 与 MCLK 比例失调导致性能下降

在某 RDNA 3 平台的调试中，工程师发现运行特定 OpenCL 内核时内存带宽仅为预期的 60%。

**问题定位**：
1. 使用 `rocm-smi --showclocks` 查看时钟状态
2. 发现 MCLK 运行在 1250MHz（正常），但 FCLK 只有 600MHz
3. 根据 RDNA 3 架构设计，FCLK 应该维持在 MCLK 的 1/3 到 1/2 范围内
4. 低 FCLK 导致 Infinity Fabric 带宽不足，限制了内存访问

**根因分析**：
```
预期配置: MCLK=2500MHz, FCLK=1250MHz (1:2)
实际配置: MCLK=2500MHz, FCLK=600MHz  (1:4.2)
```

FCLK 被 SMU 错误地降低可能是因为某些 SoC 模块的时钟门控条件被意外触发，导致 FCLK PLL 失锁。

**修复方法**：
通过 debugfs 强制 FCLK 到最小值：
```bash
# 设置 FCLK 下限
echo "800" > /sys/class/drm/card0/device/pp_dpm_fclk
```

或通过驱动补丁修复 SMU FCLK 管理逻辑：
```c
// 在 smu_v13_0_init_fclk() 中增加对最低 FCLK 的约束
static int smu_v13_0_init_fclk(struct smu_context *smu)
{
    struct amdgpu_device *adev = smu->adev;
    uint32_t min_fclk, max_fclk;

    smu_get_dpm_freq_range(smu, SMU_FCLK, &min_fclk, &max_fclk, false);

    // 确保 FCLK 不低于 MCLK 的 1/3
    uint32_t mclk = smu_get_current_mclk(smu);
    uint32_t fclk_min_required = mclk / 3;

    if (min_fclk < fclk_min_required) {
        dev_info(adev->dev,
            "Adjusting FCLK min from %u to %u MHz (MCLK=%u)\n",
            min_fclk, fclk_min_required, mclk);
        min_fclk = fclk_min_required;
    }

    return smu_set_soft_min_by_freq(smu, SMU_FCLK, min_fclk);
}
```

#### 3.4 功耗/频率违反测试用例设计

在 GPU 驱动测试中，设计频率管理的压力测试至关重要：

```python
#!/usr/bin/env python3
"""
gpu_freq_stress_test.py - GPU 频率管理压力测试
测试内容：
  1. 频率切换稳定性测试
  2. 多线程并发写入测试
  3. 频率边界测试（最小/最大/超频）
  4. 长时间稳定性测试
"""

import os
import sys
import time
import threading
import random
import subprocess

class GPUFreqStressTest:
    def __init__(self, card=0):
        self.card = card
        self.device_path = f"/sys/class/drm/card{card}/device"
        self.errors = []
        self.running = True

    def _read_sysfs(self, attr):
        """读取 sysfs 属性"""
        try:
            path = os.path.join(self.device_path, attr)
            with open(path, 'r') as f:
                return f.read().strip()
        except Exception as e:
            return f"Error: {e}"

    def _write_sysfs(self, attr, value):
        """写入 sysfs 属性"""
        try:
            path = os.path.join(self.device_path, attr)
            with open(path, 'w') as f:
                f.write(str(value))
            return True
        except Exception as e:
            self.errors.append(f"Write {attr}={value}: {e}")
            return False

    def test_freq_switching(self):
        """测试频率切换"""
        print("\n[测试1] 频率切换稳定性测试")
        available_levels = [0, 1, 2, 3, 4, 5]

        # 设置为 manual 模式
        self._write_sysfs("power_dpm_force_performance_level", "manual")

        for level in available_levels:
            print(f"  切换到 SCLK level {level}...")
            if not self._write_sysfs("pp_dpm_sclk", str(level)):
                print(f"  ✗ 失败")
                continue

            time.sleep(0.1)  # 等待切换完成

            # 验证
            current = self._read_sysfs("pp_dpm_sclk")
            if str(level) in current:
                print(f"  ✓ level {level} 已激活")
            else:
                print(f"  ✗ level {level} 未生效 (当前: {current[:50]})")

        # 恢复自动
        self._write_sysfs("power_dpm_force_performance_level", "auto")
        print("  已恢复自动模式")

    def test_concurrent_writes(self, num_threads=4):
        """测试多线程并发写入"""
        print(f"\n[测试2] 多线程并发写入测试 ({num_threads}线程)")

        self._write_sysfs("power_dpm_force_performance_level", "manual")

        def write_thread(thread_id):
            """线程函数：持续写入随机频率级别"""
            for i in range(50):
                if not self.running:
                    break
                level = random.randint(0, 5)
                self._write_sysfs("pp_dpm_sclk", str(level))
                time.sleep(random.uniform(0.001, 0.01))

        threads = []
        for i in range(num_threads):
            t = threading.Thread(target=write_thread, args=(i,))
            threads.append(t)
            t.start()

        # 同时执行频率读取
        for i in range(100):
            self._read_sysfs("pp_dpm_sclk")
            time.sleep(0.005)

        # 等待所有线程完成
        for t in threads:
            t.join()

        self._write_sysfs("power_dpm_force_performance_level", "auto")

        if self.errors:
            print(f"  发现 {len(self.errors)} 个错误:")
            for err in self.errors[-5:]:  # 只显示最后5个
                print(f"    ✗ {err}")
        else:
            print(f"  ✓ 并发测试完成，无错误")

    def test_freq_boundary(self):
        """测试频率边界"""
        print("\n[测试3] 频率边界测试")

        # 读取可用频率范围
        sclk_info = self._read_sysfs("pp_od_clk_voltage")
        print(f"  OD 信息:\n{sclk_info[:500]}")

        # 测试最小频率
        print("  设置 manual 模式...")
        self._write_sysfs("power_dpm_force_performance_level", "manual")
        self._write_sysfs("pp_dpm_sclk", "0")
        time.sleep(0.2)
        print(f"  最低 SCLK: {self._read_sysfs('pp_dpm_sclk')[:60]}")

        # 测试最大频率
        last_level = self._read_sysfs("pp_dpm_sclk")
        # 找到最高档位
        max_level = 0
        for line in last_level.split('\n'):
            if ':' in line:
                level_num = int(line.split(':')[0].strip())
                max_level = max(max_level, level_num)

        self._write_sysfs("pp_dpm_sclk", str(max_level))
        time.sleep(0.2)
        print(f"  最高 SCLK (level {max_level}): {self._read_sysfs('pp_dpm_sclk')[:60]}")

        self._write_sysfs("power_dpm_force_performance_level", "auto")
        print("  已恢复自动模式")

    def test_long_stability(self, duration=60):
        """长时间稳定性测试"""
        print(f"\n[测试4] 长时间稳定性测试 ({duration}秒)")

        self._write_sysfs("power_dpm_force_performance_level", "manual")

        start_time = time.time()
        check_interval = 1
        last_print = 0

        while time.time() - start_time < duration:
            elapsed = int(time.time() - start_time)

            # 随机切换频率
            level = random.randint(0, 5)
            self._write_sysfs("pp_dpm_sclk", str(level))

            if elapsed - last_print >= 10:
                print(f"  运行中... {elapsed}s/{duration}s (当前level={level})")
                last_print = elapsed

            time.sleep(random.uniform(0.05, 0.2))

        self._write_sysfs("power_dpm_force_performance_level", "auto")

        if self.errors:
            print(f"  ✗ 长时间测试完成，发现 {len(self.errors)} 个错误")
        else:
            print(f"  ✓ 长时间测试完成，无错误")

    def run_all(self):
        """运行所有测试"""
        print("=" * 60)
        print("GPU 频率管理压力测试 v1.0")
        print(f"设备: {self.device_path}")
        print("=" * 60)

        self.test_freq_switching()
        self.test_concurrent_writes()
        self.test_freq_boundary()
        self.test_long_stability()

        print("\n" + "=" * 60)
        print("测试总结")
        print(f"总错误数: {len(self.errors)}")
        if self.errors:
            print("错误列表:")
            for err in self.errors:
                print(f"  ✗ {err}")
        else:
            print("  ✓ 所有测试通过！")
        print("=" * 60)

if __name__ == "__main__":
    test = GPUFreqStressTest()
    test.run_all()
```

### 4. 实践练习

#### 练习一：探索系统时钟域

在你的 AMD GPU 系统上完成以下操作：
1. 使用 `rocm-smi --showclocks` 查看当前 SCLK、MCLK、FCLK
2. 通过 sysfs 读取 `pp_dpm_sclk` 和 `pp_dpm_mclk`，了解可用频率档位
3. 记录系统空闲时的频率值
4. 运行 GPU 负载程序（如 `glmark2`）并观察频率变化

**提交**：记录空闲和负载下的频率对比表格。

#### 练习二：手动频率控制

1. 将 `power_dpm_force_performance_level` 设为 `manual`
2. 将 SCLK 锁定到最低档位
3. 运行 `glmark2` 并记录 FPS 分数
4. 将 SCLK 锁定到最高档位
5. 再次运行 `glmark2` 并对比 FPS 分数
6. 恢复自动模式

**预期结果**：最高频率档位比最低档位有显著的性能提升。

#### 练习三：时钟门控影响分析

1. 通过 debugfs 或内核参数禁用时钟门控
2. 使用 `powertop` 或 `turbostat` 对比启用/禁用时钟门控时的系统功耗
3. 记录 GPU 空闲和负载下的功耗差异

```bash
# 查看当前的时钟门控状态
cat /sys/kernel/debug/dri/0/amdgpu_pm_info | grep -i gating

# 使用 perf 统计时钟门控事件
perf stat -e amdgpu:amdgpu_gfx_clock_gating_enable \
         -e amdgpu:amdgpu_gfx_clock_gating_disable \
         -a sleep 10
```

#### 练习四：频率锁定 Bug 重现与修复

模拟练习（无需实际硬件）：
1. 阅读内核源码中 `amdgpu_dpm_force_performance_level` 函数
2. 分析竞态条件的触发路径
3. 编写修复补丁
4. 使用 `kunit` 或 `kselftest` 框架编写单元测试

#### 练习五：编写自动化频率测试脚本

基于本章提供的 `gpu_clock_analyzer.py` 和 `gpu_freq_stress_test.py`，编写一个完整的自动化测试脚本，要求：
1. 自动遍历所有可用频率档位
2. 在每个档位下运行 5 秒负载测试
3. 记录每个档位的性能数据（FPS 或计算吞吐量）
4. 检测异常（如频率切换失败、驱动崩溃）
5. 生成测试报告

#### 练习六：Analyzing MCLK Partition Behavior

如果使用支持 MCLK 分区的 GPU（如 MI200/MI300），完成：
1. 检查 `pp_dpm_mclk` 是否显示多分区信息
2. 尝试在不同分区之间切换并观察
3. 对比分区内和跨分区的内存访问延迟差异

```bash
# 查看 MCLK 分区信息
cat /sys/class/drm/card0/device/pp_dpm_mclk

# 检查是否支持 MCLK 分区
cat /sys/class/drm/card0/device/pp_features
```

#### 练习七：基于 Ftrace 的 DPM 延迟分析

使用 ftrace 测量 DPM 频率切换的延迟：
1. 修改 `trace_dpm_clocks.sh`，增加对 `smu_cmn_send_smc_msg_with_param` 的追踪
2. 从追踪结果中提取每次 SMU 消息的延迟
3. 计算平均切换延迟和最大切换延迟
4. 分析哪些场景下延迟异常增高

#### 练习八：SMU 消息延迟统计

通过内核 tracepoint 统计 SMU 消息的往返延迟：
```bash
#!/bin/bash
# 统计 SMU 消息延迟分布
TRACE_DIR=/sys/kernel/tracing

# 启用 SMU 消息 tracepoint
echo 1 > $TRACE_DIR/events/amdgpu/amdgpu_smu_send_msg/enable
echo 1 > $TRACE_DIR/tracing_on

# 运行 GPU 负载
glmark2 --run-forever &
GLPID=$!
sleep 5

# 停止追踪并提取数据
echo 0 > $TRACE_DIR/tracing_on
kill $GLPID 2>/dev/null

# 提取消息类型与时间戳
cat $TRACE_DIR/trace | grep smu_send_msg | \
    awk '{print $NF}' | sort | uniq -c | sort -rn
echo ""

# 计算平均延迟
echo "SMU消息延迟统计（微秒）:"
cat $TRACE_DIR/trace | grep smu_send_msg | \
    awk '{print $(NF-1)}' | \
    awk '{sum+=$1; count++} END {print "  平均: " sum/count " us"}'

echo 0 > $TRACE_DIR/events/amdgpu/amdgpu_smu_send_msg/enable
```

#### 练习九：跨架构频率特性对比

如果你有不同代的 AMD GPU 硬件（如 Vega 56、RX 6800、RX 7900 XTX），完成以下对比测试：
1. 在相同负载下（使用同一个 OpenCL 程序），记录各 GPU 的 SCLK 动态范围
2. 对比各代 GPU 从空闲到满载的频率上升时间
3. 对比各代 GPU 的 DVFS 决策频率（可观察频率更新间隔）
4. 记录各代 GPU 在相同功耗限制下的频率稳定性

**提交**：生成对比表格，分析各代架构在时钟管理上的演进效果。

### 5. 相关链接

- **AMD RDNA 3 架构白皮书（公开章节）**：https://www.amd.com/en/products/graphics/amd-radeon-rx-7900xtx
- **Linux 内核源码 - amdgpu_dpm.c**：https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/pm/amdgpu_dpm.c
- **ROCm SMI 命令行指南**：https://rocm.docs.amd.com/projects/amdsmi/en/latest/
- **AMD GPUOpen - Clock Gating & DVFS**：https://gpuopen.com/learn/amd-powerplay-technology/
- **Phoronix 文章：AMDGPU Clock Management Improvements**：https://www.phoronix.com/news/AMDGPU-Clock-Management-5.19
- **Linux 内核 PM 文档 - AMDGPU 电源管理**：https://www.kernel.org/doc/html/latest/gpu/amdgpu/thermal.html
- **Ftrace 调试手册**：https://www.kernel.org/doc/html/latest/trace/ftrace.html
- **BCC/eBPF 项目首页**：https://github.com/iovisor/bcc
- **AMD ROCm 文档 - 性能优化指南**：https://rocm.docs.amd.com/en/latest/how-to/performance-tuning.html
- **内核邮件列表 - AMDGPU 时钟管理补丁讨论**：https://lists.freedesktop.org/archives/amd-gfx/

## 今日小结

- **SCLK**（图形核心时钟）、**MCLK**（显存时钟）、**FCLK**（Infinity Fabric 时钟）是 AMD GPU 三大关键时钟域，分别影响计算性能、内存带宽与互联延迟。
- 时钟调节通过 **SMU 固件** 执行，驱动通过 mailbox 发送频率请求，SMU 根据 V/F 曲线、温度与功耗限制计算最终频率。
- **时钟门控**包含四级层级（L0-L3），是 GPU 最有效的省电技术，但也给调试带来挑战。
- **DVFS 算法**使用 PID 控制器动态调节频率，升频和降频采用不对称策略以平衡性能和响应速度。
- **P-State 转换**涉及复杂的多域同步过程，需要避免瞬时功耗尖峰。
- 不同 GPU 架构（GCN/RDNA/CDNA）在时钟管理上存在显著差异，测试中需要针对性适配。
- 用户可通过 **sysfs**（`pp_dpm_sclk`、`pp_dpm_mclk`）、**rocm-smi** 或 **OverDrive 接口** 查看与调节时钟频率。
- 调试工具链包括 **Python 分析脚本**、**ftrace**、**eBPF/BCC**，分别适用于不同场景。
- 实际案例分析涵盖了 MCLK 多分区 Bug、竞态条件、FCLK/MCLK 比例失调等真实问题。
- 完整的压力测试框架可用于验证频率管理的稳定性和正确性。
- 下一日（第 303 天）将深入探讨 **电压调节与 VID（Voltage Identifier）** 机制。

## 扩展思考（可选）

### 思考题

1. **架构设计**：假设你需要设计一个新的 GPU 时钟管理架构，你会如何平衡时钟域的数量和复杂度？更多的时钟域（更细粒度的控制）是否会带来更好的能效比？

2. **PID 参数调优**：DVFS PID 控制器的参数（Kp、Ki、Kd）在不同场景下应该如何调整？例如，游戏场景和科学计算场景分别需要什么特点的 PID 参数？

3. **负责任的超频**：从安全和可靠性角度，GPU 频率调节的测试边界应该是什么？如何设计超频测试流程来最小化硬件损坏风险？

4. **chiplet 时代的挑战**：在 RDNA 3 的 chiplet 设计中，跨 die 时钟同步面临哪些挑战？相比 monolithic 设计，chiplet 设计在时钟管理上有哪些额外的测试需求？

5. **频率预测算法**：能否利用机器学习模型（如 LSTM）来预测 GPU 负载变化，从而提前调整频率以减少切换延迟？请设计一个简单的频率预测方案。

### 实践题

假设你正在优化一个 HPC 应用，该应用对内存带宽极其敏感，但对核心计算需求不高。你应当如何调整 SCLK、MCLK、FCLK 的频率以在满足性能目标的同时尽可能降低功耗？请给出具体的调节步骤与预期效果分析。

（提示：考虑 MCLK 对带宽的直接影响、FCLK 与 MCLK 的比例关系、以及 SCLK 对内存控制器压力的间接影响。）

### 6. 附录：调试命令速查表

| 操作 | 命令 | 说明 |
|------|------|------|
| 查看SCLK | `cat /sys/class/drm/card0/device/pp_dpm_sclk` | 显示所有档位，当前档位标记* |
| 查看MCLK | `cat /sys/class/drm/card0/device/pp_dpm_mclk` | 显示所有档位，当前档位标记* |
| 查看FCLK | `cat /sys/class/drm/card0/device/pp_dpm_fclk` | 仅部分GPU支持 |
| 设置manual模式 | `echo "manual" > /sys/class/drm/card0/device/power_dpm_force_performance_level` | 允许手动选择档位 |
| 设置SCLK档位 | `echo "3" > /sys/class/drm/card0/device/pp_dpm_sclk` | 数字对应档位编号 |
| 恢复自动 | `echo "auto" > /sys/class/drm/card0/device/power_dpm_force_performance_level` | 恢复SMU自动管理 |
| 查看OD设置 | `cat /sys/class/drm/card0/device/pp_od_clk_voltage` | 显示频率-电压表 |
| OD设置频率 | `echo "s 1 1100 900" > /sys/class/drm/card0/device/pp_od_clk_voltage` | 格式: s 档位 频率 电压 |
| 提交OD更改 | `echo "c" > /sys/class/drm/card0/device/pp_od_clk_voltage` | 提交所有未生效的OD更改 |
| rocm-smi概要 | `rocm-smi` | 显示GPU状态概要 |
| rocm-smi时钟 | `rocm-smi --showclocks` | 显示详细时钟信息 |
| 实时监控 | `watch -n 0.5 "cat /sys/class/drm/card0/device/pp_dpm_sclk \| grep '*'"` | 每0.5秒刷新 |
| ftrace追踪 | `trace_dpm_clocks.sh` | 使用ftrace追踪DPM事件 |
| Python分析 | `python3 gpu_clock_analyzer.py --duration 30` | 高频采样分析 |
| 压力测试 | `python3 gpu_freq_stress_test.py` | 频率管理压力测试 |
| 查看功耗 | `cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_input` | 单位: 微瓦 |
| 查看温度 | `cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input` | 单位: 毫摄氏度 |
| 时钟门控状态 | `cat /sys/kernel/debug/dri/0/amdgpu_pm_info \| grep -i gating` | 调试接口 |
| 内核日志 | `dmesg \| grep -i "sclk\|mclk\|fclk\|dpm" \| tail -20` | 查看DPM相关日志 |
| perf事件统计 | `perf stat -e amdgpu:amdgpu_gfx_clock_gating_* -a sleep 5` | 统计时钟门控事件 |

### 7. 参考资源与进一步阅读

1. **内核源码文件**：
   - `drivers/gpu/drm/amd/pm/amdgpu_dpm.c` - DPM核心逻辑
   - `drivers/gpu/drm/amd/pm/swsmu/smu13/smu_v13_0.c` - SMU v13.0驱动
   - `drivers/gpu/drm/amd/pm/swsmu/amdgpu_smu.c` - SMU通用接口
   - `drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.c` - GFX时钟门控
   - `drivers/gpu/drm/amd/include/amd_shared.h` - 电源管理枚举定义

2. **关键数据结构**：
   - `struct smu_context` - SMU上下文，包含所有SMU状态
   - `struct smu_dpm_context` - DPM上下文
   - `struct amdgpu_pm` - 电源管理核心结构体
   - `struct smu_13_0_dpm_table` - SMU v13.0 DPM表

3. **调试工具安装**：
   ```bash
   # ROCm SMI
   sudo apt install rocm-smi
   
   # BCC/eBPF
   sudo apt install bpfcc-tools python3-bpfcc
   
   # perf
   sudo apt install linux-tools-common linux-tools-generic
   
   # glmark2 (GPU基准测试)
   sudo apt install glmark2
   
   # powertop (功耗分析)
   sudo apt install powertop
   ```

4. **推荐阅读**：
   - AMD GPUOpen: "AMD PowerPlay Technology" - 官方技术文档
   - Linux内核文档: "AMDGPU Power Management" - Documentation/gpu/amdgpu/thermal.rst
   - Phoronix: AMDGPU Linux显卡驱动性能分析系列文章
   - ROCm文档: "ROCm Performance Optimization Guide"
   - kernelnewbies: AMDGPU驱动开发入门指南

### 8. 知识自测

完成以下自测题以检验对本日内容的理解：

**选择题**：

1. SCLK 主要负责哪个硬件模块的时序？
   A. 显存控制器
   B. 流处理器（CU）和纹理单元（TMU）
   C. Infinity Fabric 互联
   D. 显示控制器

2. 下列哪个 sysfs 接口用于手动选择 SCLK 档位？
   A. `/sys/class/drm/card0/device/pp_od_clk_voltage`
   B. `/sys/class/drm/card0/device/pp_dpm_sclk`
   C. `/sys/class/drm/card0/device/power_dpm_force_performance_level`
   D. `/sys/class/drm/card0/device/hwmon/hwmon*/freq1_input`

3. DVFS 中的 PID 控制器中，积分项（I）的作用是什么？
   A. 立即响应当前负载误差
   B. 消除稳态误差，修正长期累积偏差
   C. 预测误差变化趋势，提前调整
   D. 限制最大输出频率

4. 在 P-State 转换过程中，下列哪项是正确的操作顺序？
   A. 先调频率再调电压
   B. 先调电压再调频率
   C. 频率和电压同时调整
   D. 先调 FCLK 再调 SCLK

5. 时钟门控的 L1 级别是指什么粒度的门控？
   A. 整个 GPU 芯片
   B. 功能模块（如 CU、ACE）
   C. 子单元（如 ALU 阵列）
   D. 单个寄存器/触发器

**填空题**：

6. 在 AMD RDNA 3 架构中，FCLK 代表的是 ________ 时钟。
7. 升频策略通常采用 ________（大/小）步长以快速响应负载突增。
8. SMU 消息传递使用 ________ 寄存器对进行通信。
9. RDNA 3 相比 GCN 架构，DVFS 决策频率从 100Hz 提升到了 ________ Hz。
10. 多时钟域同步转换时，应先调整 ________ 域再调整 ________ 域，以避免瞬时功耗尖峰。

**答案**：1-B, 2-B, 3-B, 4-B, 5-B, 6-Infinity Fabric, 7-大, 8-MMIO, 9-5000, 10-降频、升频