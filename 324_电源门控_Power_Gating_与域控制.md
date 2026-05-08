# 电源门控（Power Gating）与域控制

## 1. 概述

### 1.1 什么是电源门控（Power Gating）

电源门控（Power Gating, PG）是一种通过物理切断空闲功能模块的电源供应来消除静态功耗（泄漏功耗）的低功耗技术。与时钟门控（Clock Gating）仅停止时钟翻转不同，电源门控将模块的电源轨完全断开，使泄漏电流降至接近零的水平。

在现代GPU中，随着制程节点不断微缩（从28nm到3nm），泄漏功耗占总功耗的比例从约10%上升到超过40%。电源门控技术因此成为先进GPU低功耗设计中不可或缺的核心技术。

### 1.2 PG与CG的核心区别

| 特性 | 时钟门控（CG） | 电源门控（PG） |
|------|---------------|---------------|
| 功耗消除 | 仅消除动态功耗 | 消除动态+静态（泄漏）功耗 |
| 状态保留 | 寄存器状态完全保留 | 状态丢失，需要保存/恢复 |
| 唤醒延迟 | 1-10个时钟周期 | 10-1000μs（含PLL锁定） |
| 实现复杂度 | 低（只需门控时钟树） | 高（需电源开关+隔离单元） |
| 面积开销 | 小（~1%门数） | 大（~5-10%门数+电源网格） |
| 适用场景 | 短时空闲（ns-μs级） | 长时间空闲（μs-ms级） |
| 功耗节省幅度 | 30-70%（动态功耗） | 90-99%（总功耗） |

PG的功耗节省幅度远大于CG，但代价是更高的唤醒延迟和实现复杂度。因此在现代GPU中，两者通常配合使用形成层次化的功耗管理策略。

### 1.3 电源门控的关键挑战

实现有效的电源门控面临以下核心挑战：

1. **状态保存与恢复（SSR）**：门控前需要将关键状态保存到始终保持供电的保存寄存器或SRAM中，恢复时再重新载入
2. **浪涌电流控制**：电源域重新开启时的瞬时大电流可能引起电源网格电压跌落（IR Drop）
3. **隔离单元**：门控域的输出需要隔离逻辑防止浮空输入传播到活动域
4. **唤醒延迟隐藏**：从PG状态恢复的延迟需要通过预测或延迟隐藏技术来避免性能损失
5. **电源开关尺寸**：开关晶体管的尺寸需要在导通电阻和泄漏之间权衡

### 1.4 演进历史

| 时代 | 制程 | PG能力 | 代表架构 |
|------|------|--------|---------|
| 2011-2013 | 28nm | GFX粗略门控（CC6状态） | GCN 1.0 (Tahiti) |
| 2013-2015 | 28nm | GFX域PG + MC域PG | GCN 2.0-3.0 (Hawaii) |
| 2016-2018 | 14/16nm | GFXOFF + 多域独立PG | GCN 4.0-5.0 (Polaris, Vega) |
| 2019-2020 | 7nm | GFXOFF v2 + 域级精细PG | RDNA 1 (Navi 10) |
| 2020-2022 | 7nm | GFXOFF v3 + VCN独立PG | RDNA 2 (Navi 21) |
| 2023-2024 | 5/6nm | 芯片级PG + 每CU独立PG | RDNA 3 (Navi 31) |

## 2. 电源门控硬件基础

### 2.1 电源开关单元

电源门控的核心是电源开关单元（Power Switch），它控制电源轨与功能模块之间的连接。

#### 2.1.1 Header型开关

Header开关连接在电源轨（VDD）和模块虚拟电源轨（VVDD）之间，使用PMOS晶体管实现：

```verilog
// Header型电源开关单元
module header_power_switch (
    input  wire vdd,        // 全局电源轨
    input  wire sleep_en,    // 休眠使能（高电平=断开）
    output wire vvdd         // 虚拟电源轨（输出到门控域）
);

    // PMOS电源开关阵列（多指并联降低导通电阻）
    parameter FP = 32;        // 并行因子
    parameter W = 8;          // 单指宽度（μm）
    parameter RON = 0.5;      // 目标导通电阻（Ω）

    wire [FP-1:0] gate;
    genvar i;

    // 去耦电容建模
    reg [31:0] decap_count;

    // 开关控制逻辑
    assign gate = {FP{sleep_en}};

    // PMOS开关阵列
    for (i = 0; i < FP; i = i + 1) begin : switch_array
        pmos_switch #(.W(W)) u_switch (
            .drain  (vdd),
            .source (vvdd),
            .gate   (gate[i])
        );
    end

    // 浪涌电流限制：分级开启
    reg [3:0] wake_phase;
    reg [FP-1:0] phased_gate;

    always @(posedge wake_req or negedge clk) begin
        if (!clk) begin
            wake_phase <= 0;
            phased_gate <= {FP{1'b1}};
        end else if (wake_req && wake_phase < 8) begin
            // 分8级开启，每级开启4个开关
            for (int j = 0; j < FP/8; j++) begin
                phased_gate[wake_phase*(FP/8) + j] <= 1'b0;
            end
            wake_phase <= wake_phase + 1;
        end
    end

endmodule
```

#### 2.1.2 Footer型开关

Footer开关连接在模块虚拟地（VVSS）和全局地（VSS）之间，使用NMOS晶体管实现：

```verilog
// Footer型电源开关单元
module footer_power_switch (
    input  wire vss,         // 全局地轨
    input  wire sleep_en,    // 休眠使能（高电平=断开）
    output wire vvss         // 虚拟地轨（输出到门控域）
);

    // NMOS电源开关阵列
    parameter FP = 48;
    parameter W = 6;

    wire [FP-1:0] gate;

    // 开关控制
    assign gate = {FP{sleep_en}};

    // NMOS开关阵列
    genvar i;
    for (i = 0; i < FP; i = i + 1) begin : switch_array
        nmos_switch #(.W(W)) u_switch (
            .drain  (vss),
            .source (vvss),
            .gate   (gate[i])
        );
    end

endmodule
```

#### 2.1.3 Header vs Footer对比

| 特性 | Header（PMOS） | Footer（NMOS） |
|------|---------------|---------------|
| 导通电阻 | 较高（~2x NMOS） | 较低 |
| 面积效率 | 较大 | 较小 |
| 体效应 | 较小 | 较大 |
| 地弹噪声 | 较小 | 较大 |
| 适用场景 | 高性能逻辑 | 低泄漏优化 |

在实际GPU芯片中，通常采用Header+Footer混合方式，关键路径使用Header型以保证性能，非关键路径使用Footer型以节省面积。

### 2.2 隔离单元（Isolation Cells）

当电源域被门控后，其输出信号变为浮空状态（X态）。如果不加隔离，X态会传播到相邻的活动域，导致逻辑错误甚至漏电增加。隔离单元在PG域关闭时将其输出钳位到固定电平：

```verilog
// 隔离单元 - AND型（低电平钳位）
module isolation_cell_and (
    input  wire data_in,      // 来自PG域的数据
    input  wire iso_en,       // 隔离使能（高电平=隔离）
    output wire data_out      // 输出到活动域
);

    // 隔离使能有效时，输出钳位到0
    assign data_out = iso_en ? 1'b0 : data_in;

endmodule

// 隔离单元 - OR型（高电平钳位）
module isolation_cell_or (
    input  wire data_in,
    input  wire iso_en,
    output wire data_out
);

    assign data_out = iso_en ? 1'b1 : data_in;

endmodule

// 隔离单元 - 锁存型（保持最后值）
module isolation_cell_latch (
    input  wire data_in,
    input  wire iso_en,
    input  wire clk,
    output wire data_out
);

    reg hold_value;

    always @(posedge clk or posedge iso_en) begin
        if (iso_en)
            hold_value <= hold_value;  // 保持最后值
        else
            hold_value <= data_in;
    end

    assign data_out = iso_en ? hold_value : data_in;

endmodule
```

隔离单元的使用原则：
- **AND型**：用于低电平有效的控制信号或数据总线（默认输出0最安全）
- **OR型**：用于高电平有效的控制信号（如时钟使能）
- **锁存型**：用于需要保存最后逻辑值的关键控制信号（如状态机输出）

### 2.3 状态保存与恢复（SSR）

PG域关闭前必须将关键状态保存到始终保持供电的记忆单元中。SSR是PG实现中最复杂和最影响性能的环节：

```verilog
// 状态保存与恢复控制器
module ssr_controller (
    input  wire clk,
    input  wire rst_n,
    input  wire pg_request,      // PG请求
    input  wire wake_request,    // 唤醒请求
    output reg  state_saved,     // 状态已保存
    output reg  state_restored,  // 状态已恢复
    output reg  pg_ready,        // PG就绪
    output reg  save_busy,       // 保存忙
    output reg  restore_busy,    // 恢复忙
    output reg  [2:0] ssr_state  // SSR状态机
);

    // 状态编码
    localparam SSR_IDLE      = 3'd0;
    localparam SSR_SAVE_REQ  = 3'd1;
    localparam SSR_SAVING    = 3'd2;
    localparam SSR_SAVE_DONE = 3'd3;
    localparam SSR_PG        = 3'd4;
    localparam SSR_RESTORE   = 3'd5;
    localparam SSR_WAIT_LOCK = 3'd6;
    localparam SSR_ACTIVE    = 3'd7;

    // SSHB（状态保存硬件桥）接口信号
    reg  sshb_clk;
    reg  sshb_scan_en;
    reg  sshb_restore;
    wire sshb_shift_done;
    wire [15:0] scan_chain_len;

    // 本地保存寄存器阵列（始终保持供电）
    reg [31:0] save_regs [0:127];

    // 需要保存的状态寄存器列表
    reg [6:0] save_addr;
    reg [31:0] state_data;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            ssr_state     <= SSR_IDLE;
            state_saved   <= 1'b0;
            state_restored <= 1'b0;
            pg_ready      <= 1'b0;
            save_busy     <= 1'b0;
            restore_busy  <= 1'b0;
            save_addr     <= 7'd0;
        end else begin
            case (ssr_state)
                SSR_IDLE: begin
                    state_saved   <= 1'b0;
                    state_restored <= 1'b0;
                    pg_ready      <= 1'b0;
                    save_addr     <= 7'd0;

                    if (pg_request) begin
                        ssr_state <= SSR_SAVE_REQ;
                        save_busy <= 1'b1;
                    end
                end

                SSR_SAVE_REQ: begin
                    // 发起SSHB扫描保存操作
                    sshb_scan_en <= 1'b1;
                    ssr_state    <= SSR_SAVING;
                end

                SSR_SAVING: begin
                    // 逐个保存关键状态寄存器
                    if (save_addr < 128) begin
                        save_regs[save_addr] <= state_data;
                        save_addr <= save_addr + 1;
                    end else begin
                        sshb_scan_en  <= 1'b0;
                        state_saved   <= 1'b1;
                        save_busy     <= 1'b0;
                        ssr_state     <= SSR_SAVE_DONE;
                    end
                end

                SSR_SAVE_DONE: begin
                    // 延迟几拍确保所有保存操作完成
                    pg_ready  <= 1'b1;
                    ssr_state <= SSR_PG;
                end

                SSR_PG: begin
                    // 等待唤醒请求
                    pg_ready <= 1'b0;
                    if (wake_request) begin
                        ssr_state    <= SSR_RESTORE;
                        restore_busy <= 1'b1;
                    end
                end

                SSR_RESTORE: begin
                    // 发起SSHB扫描恢复操作
                    sshb_restore <= 1'b1;

                    // 从保存寄存器恢复
                    for (int i = 0; i < 128; i++) begin
                        restore_state(i, save_regs[i]);
                    end

                    state_restored <= 1'b1;
                    restore_busy   <= 1'b0;
                    sshb_restore   <= 1'b0;
                    ssr_state      <= SSR_WAIT_LOCK;
                end

                SSR_WAIT_LOCK: begin
                    // 等待PLL重新锁定和时钟稳定
                    if (pll_locked && clk_stable) begin
                        ssr_state <= SSR_ACTIVE;
                    end
                end

                SSR_ACTIVE: begin
                    // 正常操作状态
                    state_saved   <= 1'b0;
                    state_restored <= 1'b0;
                    ssr_state     <= SSR_IDLE;
                end
            endcase
        end
    end

endmodule
```

SSR的关键性能指标：
- **保存延迟**：扫描链长度 × 扫描时钟周期，典型值 1-10μs
- **恢复延迟**：与保存延迟相同 + PLL锁定时间（1-5μs）
- **状态存储开销**：每100万门约需4-8KB保存存储器

### 2.4 电源域唤醒序列

完整的电源域唤醒序列比单纯的CG唤醒复杂得多：

```
时间线：  t0          t1          t2          t3          t4          t5          t6
         |           |           |           |           |           |           |
事件：  唤醒请求   隔离撤销   电源开启   时钟稳定   状态恢复    PLL锁定    正常操作
        │           │           │           │           │           │           │
        ▼           ▼           ▼           ▼           ▼           ▼           ▼
状态：   PG状态   释放隔离   开关导通   时钟门控   扫描恢复   频率锁定   完全活动
延迟：   0ns       10ns       100ns      500ns      2μs        5μs        5.5μs
```

各阶段延迟明细：

| 阶段 | 延迟范围 | 影响因素 | 优化技术 |
|------|---------|---------|---------|
| 隔离撤销 | 5-20ns | RC延迟、隔离单元尺寸 | 提前预测撤销 |
| 电源开启 | 50-500ns | 开关尺寸、去耦电容 | 分级开启+提前开启 |
| 时钟稳定 | 100-500ns | PLL类型、时钟树RC | 快速重锁定PLL |
| 状态恢复 | 0.5-5μs | 状态量、扫描时钟频率 | 并行恢复、压缩状态 |
| PLL锁定 | 1-5μs | PLL带宽、参考时钟 | 注入锁定、FLL替代 |

## 3. AMDGPU电源域架构

### 3.1 电源域层次结构

AMDGPU SoC将芯片划分为多个独立的电源域（Power Domain），每个域可以通过SMU固件独立控制：

```
                     ┌─────────────────────────────────────┐
                     │          SMU固件（始终供电）           │
                     │   PG序列控制 / 电源轨管理 / 温度监控    │
                     └──────────┬──────────────────────────┘
                                │  PMBUS / I2C
          ┌─────────────────────┼─────────────────────┐
          │         │           │           │         │
          ▼         ▼           ▼           ▼         ▼
   ┌──────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────┐
   │  GFX域   │ │ VCN域  │ │ SDMA域 │ │ DISP域 │ │  其他域   │
   │ (最高功耗)│ │ (视频) │ │  (DMA) │ │ (显示) │ │(PCIe,IO) │
   └──────────┘ └────────┘ └────────┘ └────────┘ └──────────┘
        │            │          │          │           │
        ▼            ▼          ▼          ▼           ▼
   ┌──────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────┐
   │电源开关阵列│ │电源开关 │ │电源开关 │ │电源开关 │ │ 电源开关  │
   │ VDD_GFX  │ │VDD_VCN │ │VDD_SDMA│ │VDD_DISP│ │  VDD_IO  │
   └──────────┘ └────────┘ └────────┘ └────────┘ └──────────┘
```

每个电源域的属性：

| 域 | 供电电压 | 典型峰值功耗 | 门控深度 | 主要门控状态 |
|----|---------|-------------|---------|-------------|
| GFX | VDD_GFX (0.8-1.2V) | 150-300W | 最深 | GFXOFF, DeepSleep |
| VCN | VDD_VCN (0.7-0.9V) | 5-15W | 深 | D3Hot, D3Cold |
| SDMA | VDD_SOC (0.7-0.9V) | 2-5W | 中 | SDMA_SLEEP |
| DISP | VDD_DISP (0.7-0.9V) | 1-3W | 浅 | Display OFF |
| PCIe | VDD_IO (1.8V) | 1-2W | 固定 | ASPM L1/L2 |

### 3.2 GFX电源域详解

GFX域是GPU中最大的电源域，包含Shader阵列、光栅化单元、渲染后端等所有图形相关逻辑。GFX域的PG控制最为复杂：

```c
// GFX电源域状态枚举
enum gfx_power_state {
    GFX_POWER_STATE_0_FULL    = 0,  // 全速运行，所有单元供电
    GFX_POWER_STATE_1_LS      = 1,  // 轻度休眠（Light Sleep），部分单元CG
    GFX_POWER_STATE_2_DS      = 2,  // 深度休眠（Deep Sleep），大部分单元CG
    GFX_POWER_STATE_3_CG      = 3,  // 时钟门控，动态功耗消除
    GFX_POWER_STATE_4_PG      = 4,  // 电源门控，泄漏消除
    GFX_POWER_STATE_5_BACO    = 5,  // BACO（Bus Alive Chip Off）
};

// GFX电源域控制结构体
struct gfx_power_domain {
    enum gfx_power_state   current_state;
    enum gfx_power_state   target_state;
    uint64_t               state_residency_ns[6];
    uint64_t               last_transition_ns;

    /* 电源开关控制 */
    uint32_t               header_switch_ctrl;
    uint32_t               footer_switch_ctrl;
    uint32_t               iso_ctrl;

    /* 状态保存区域 */
    void                  *ssr_save_area;
    uint32_t               ssr_save_size;
    uint32_t               ssr_save_done;

    /* 唤醒统计 */
    atomic64_t             wakeup_count;
    uint64_t               avg_wakeup_latency_ns;
    uint64_t               max_wakeup_latency_ns;

    /* 电源测点 */
    uint32_t               voltage_monitor;
    uint32_t               current_monitor;
    uint32_t               inrush_limit;

    /* 依赖域 */
    uint32_t               dep_domain_mask;
    bool                   dep_domains_ready[8];
};
```

#### 3.2.1 GFXOFF状态机

GFXOFF是AMDGPU中最重要的PG状态，当GPU无图形/计算负载时自动进入：

```c
// GFXOFF状态机实现
enum gfxoff_state {
    GFXOFF_STATE_DISABLED    = 0,  // GFXOFF被禁用（调试/兼容性）
    GFXOFF_STATE_ALLOWED     = 1,  // 允许进入GFXOFF
    GFXOFF_STATE_REQUEST     = 2,  // 正在请求进入GFXOFF
    GFXOFF_STATE_ENTERING    = 3,  // 正在进入GFXOFF（SSR序列）
    GFXOFF_STATE_ACTIVE      = 4,  // GFXOFF激活（电源关闭）
    GFXOFF_STATE_EXITING     = 5,  // 正在退出GFXOFF（恢复序列）
    GFXOFF_STATE_ERROR       = 6,  // GFXOFF错误状态
};

struct gfxoff_manager {
    enum gfxoff_state       state;
    spinlock_t              lock;

    /* GFXOFF计数器 */
    atomic_t                gfxoff_count;
    atomic_t                gfxoff_allow_count;
    atomic_t                gfxoff_deny_count;

    /* 延迟统计 */
    uint64_t                entry_latency_ns;
    uint64_t                exit_latency_ns;
    uint64_t                total_gfxoff_time_ns;

    /* 阻止GFXOFF的原因掩码 */
    uint32_t                deny_reasons;
    /* 位0: MMIO访问进行中 */
    /* 位1: 寄存器编程中 */
    /* 位2: 中断处理中 */
    /* 位3: DMA进行中 */
    /* 位4: 帧缓冲器访问中 */
    /* 位5: 电源状态转换中 */
    /* 位6: PerfMon采样中 */

    /* RLC/SMU通信 */
    uint32_t                rlc_gfxoff_ctrl;
    uint32_t                smu_msg_gfxoff;
};

// GFXOFF入口序列
static int gfx_enter_gfxoff(struct gfxoff_manager *mgr,
                             struct amdgpu_device *adev)
{
    unsigned long flags;
    int ret = 0;

    spin_lock_irqsave(&mgr->lock, flags);

    /* 检查是否允许进入GFXOFF */
    if (mgr->deny_reasons != 0) {
        spin_unlock_irqrestore(&mgr->lock, flags);
        return -EBUSY;
    }

    if (mgr->state != GFXOFF_STATE_ALLOWED) {
        spin_unlock_irqrestore(&mgr->lock, flags);
        return -EINVAL;
    }

    mgr->state = GFXOFF_STATE_ENTERING;
    spin_unlock_irqrestore(&mgr->lock, flags);

    /* 步骤1: 触发RLC进入GFXOFF */
    WREG32_SOC15(GC, 0, regRLC_GCR_CNTL, 0x1);

    /* 步骤2: 等待RLC确认 */
    ret = amdgpu_rlc_wait_for_idle(adev);
    if (ret) {
        dev_err(adev->dev, "RLC idle wait timeout during GFXOFF entry\n");
        mgr->state = GFXOFF_STATE_ERROR;
        return ret;
    }

    /* 步骤3: 发送SMU消息请求电源门控 */
    ret = smu_send_message(adev, SMU_MSG_GFXOFF, 0);
    if (ret) {
        dev_err(adev->dev, "SMU GFXOFF message failed\n");
        mgr->state = GFXOFF_STATE_ERROR;
        return ret;
    }

    /* 步骤4: 等待SMU确认电源已关闭 */
    ret = smu_wait_for_response(adev, SMU_MSG_GFXOFF,
                                 GFXOFF_TIMEOUT_MS);
    if (ret) {
        dev_err(adev->dev, "SMU GFXOFF response timeout\n");
        mgr->state = GFXOFF_STATE_ERROR;
        return ret;
    }

    mgr->state = GFXOFF_STATE_ACTIVE;
    atomic_inc(&mgr->gfxoff_count);

    return 0;
}

// GFXOFF出口序列
static int gfx_exit_gfxoff(struct gfxoff_manager *mgr,
                            struct amdgpu_device *adev)
{
    unsigned long flags;
    int ret = 0;
    uint64_t start_ns;

    spin_lock_irqsave(&mgr->lock, flags);

    if (mgr->state != GFXOFF_STATE_ACTIVE &&
        mgr->state != GFXOFF_STATE_ENTERING) {
        spin_unlock_irqrestore(&mgr->lock, flags);
        return 0;  // 不需要退出
    }

    mgr->state = GFXOFF_STATE_EXITING;
    start_ns = ktime_get_ns();
    spin_unlock_irqrestore(&mgr->lock, flags);

    /* 步骤1: 发送SMU消息请求电源恢复 */
    ret = smu_send_message(adev, SMU_MSG_GFXOFF_EXIT, 0);
    if (ret) {
        mgr->state = GFXOFF_STATE_ERROR;
        return ret;
    }

    /* 步骤2: 等待电源稳定 */
    udelay(100);

    /* 步骤3: 恢复RLC状态 */
    WREG32_SOC15(GC, 0, regRLC_GCR_CNTL, 0x0);

    /* 步骤4: 等待RLC完成恢复 */
    ret = amdgpu_rlc_wait_for_active(adev);
    if (ret) {
        dev_err(adev->dev, "RLC active wait timeout during GFXOFF exit\n");
        mgr->state = GFXOFF_STATE_ERROR;
        return ret;
    }

    mgr->exit_latency_ns = ktime_get_ns() - start_ns;
    mgr->state = GFXOFF_STATE_ALLOWED;

    /* 更新最大延迟统计 */
    if (mgr->exit_latency_ns > mgr->max_wakeup_latency_ns)
        mgr->max_wakeup_latency_ns = mgr->exit_latency_ns;

    return 0;
}
```

### 3.3 VCN电源域

VCN（Video Core Next）域负责视频编解码，其PG控制独立于GFX域：

```c
// VCN电源域状态与配置
struct vcn_power_domain {
    bool                    pg_enabled;
    enum vcn_power_state {
        VCN_POWER_ON        = 0,  // 全供电
        VCN_D3HOT           = 1,  // D3Hot - 主电源关闭，辅助供电保留
        VCN_D3COLD          = 2,  // D3Cold - 完全断电
    } state;

    /* 电源控制寄存器 */
    uint32_t                vcn_pg_ctrl;
    uint32_t                vcn_pg_status;
    uint32_t                vcn_powergate_delay;
    uint32_t                vcn_ungate_delay;

    /* UVD/VCE状态保存区 */
    void                   *uvd_save_area;
    uint32_t                uvd_save_size;

    /* 解码会话计数 */
    atomic_t                decode_session_count;
    atomic_t                encode_session_count;

    /* 空闲超时 */
    uint32_t                idle_timeout_ms;
    struct timer_list       idle_timer;
};

// VCN电源门控控制
static int vcn_powergate(struct vcn_power_domain *vcn_pg,
                          struct amdgpu_device *adev, bool gate)
{
    int ret = 0;

    if (gate) {
        /* --- 电源门控序列 --- */

        /* 1. 等待VCN空闲 */
        ret = vcn_wait_for_idle(adev);
        if (ret) {
            dev_warn(adev->dev, "VCN not idle, skipping PG\n");
            return -EBUSY;
        }

        /* 2. 保存UVD状态 */
        vcn_save_uvd_state(adev, vcn_pg->uvd_save_area);

        /* 3. 禁用VCN时钟门控（准备PG） */
        WREG32_SOC15(VCN, 0, regVCN_CG_CTRL, 0);

        /* 4. 等待时钟停止 */
        udelay(10);

        /* 5. 发送SMU PG消息 */
        ret = smu_send_message(adev, SMU_MSG_POWERGATE_VCN, 1);
        if (ret) {
            dev_err(adev->dev, "VCN powergate SMU msg failed\n");
            return ret;
        }

        /* 6. 等待SMU确认电源已关闭 */
        ret = smu_wait_for_response(adev, SMU_MSG_POWERGATE_VCN,
                                     VCN_PG_TIMEOUT_MS);
        if (ret) {
            dev_err(adev->dev, "VCN powergate timeout\n");
            vcn_pg->state = VCN_POWER_ON;  // 回滚
            return ret;
        }

        vcn_pg->state = VCN_D3HOT;

    } else {
        /* --- 电源恢复序列 --- */

        /* 1. 发送SMU恢复供电消息 */
        ret = smu_send_message(adev, SMU_MSG_POWERGATE_VCN, 0);
        if (ret) {
            dev_err(adev->dev, "VCN ungate SMU msg failed\n");
            return ret;
        }

        /* 2. 等待电源稳定 */
        mdelay(1);

        /* 3. 恢复UVD状态 */
        vcn_restore_uvd_state(adev, vcn_pg->uvd_save_area);

        /* 4. 重新初始化VCN时钟门控 */
        VCN_CG_CTRL_DEFAULT = RREG32_SOC15(VCN, 0, regVCN_CG_CTRL);
        WREG32_SOC15(VCN, 0, regVCN_CG_INIT, 1);
        WREG32_SOC15(VCN, 0, regVCN_CG_CTRL, VCN_CG_CTRL_DEFAULT);

        vcn_pg->state = VCN_POWER_ON;
    }

    return ret;
}

// VCN空闲超时处理函数
static void vcn_idle_timeout_handler(struct timer_list *t)
{
    struct vcn_power_domain *vcn_pg = from_timer(vcn_pg, t, idle_timer);

    if (atomic_read(&vcn_pg->decode_session_count) == 0 &&
        atomic_read(&vcn_pg->encode_session_count) == 0) {
        vcn_powergate(vcn_pg, container_of(vcn_pg,
                      struct amdgpu_device, vcn_pg), true);
    }
}
```

### 3.4 SDMA电源域

SDMA（System DMA）引擎执行独立于GFX的数据传输操作，其PG策略需要考虑传输模式：

```c
// SDMA电源域控制
struct sdma_power_domain {
    bool                    pg_capable;
    bool                    pg_enabled;

    /* SDMA实例数 */
    uint32_t                instance_count;

    /* 每个实例的状态 */
    struct sdma_instance {
        bool                 active;
        enum sdma_pg_state {
            SDMA_PG_ON      = 0,
            SDMA_PG_SLEEP   = 1,
            SDMA_PG_OFF     = 2,
        } state;

        /* 空闲检测 */
        uint64_t             last_activity_ns;
        uint32_t             idle_threshold_us;

        /* 当前传输 */
        struct dma_fence    *running_fence;
    } instances[8];

    /* 全局PG控制 */
    uint32_t                sdma_pg_ctrl;
    uint32_t                sdma_pg_status;
};

// SDMA实例空闲检测与PG控制
static int sdma_instance_powergate(struct sdma_instance *inst,
                                    struct amdgpu_device *adev,
                                    uint32_t inst_idx)
{
    uint64_t idle_time_ns;
    uint32_t ctrl_val;

    if (inst->state == SDMA_PG_OFF)
        return 0;

    /* 检查空闲时间 */
    if (inst->running_fence &&
        !dma_fence_is_signaled(inst->running_fence)) {
        return -EBUSY;  /* 传输进行中 */
    }

    idle_time_ns = ktime_get_ns() - inst->last_activity_ns;
    if (idle_time_ns < (uint64_t)inst->idle_threshold_us * 1000)
        return -EAGAIN;  /* 空闲时间不足 */

    /* 进入休眠状态 */
    ctrl_val = RREG32_SOC15(SDMA0, inst_idx, regSDMA_PG_CTRL);
    ctrl_val |= SDMA_PG_CTRL_SLEEP_EN;
    WREG32_SOC15(SDMA0, inst_idx, regSDMA_PG_CTRL, ctrl_val);

    inst->state = SDMA_PG_SLEEP;

    /* 如果空闲时间超过长阈值，完全关闭电源 */
    if (idle_time_ns > (uint64_t)inst->idle_threshold_us * 10000) {
        smu_send_message(adev,
                         SMU_MSG_POWERGATE_SDMA + inst_idx, 1);
        inst->state = SDMA_PG_OFF;
    }

    return 0;
}
```

### 3.5 Display电源域

Display域控制显示输出接口（DP/HDMI/eDP），其PG策略需要考虑热插拔和模式切换：

```c
// Display电源域状态
struct display_power_domain {
    bool                    pg_enabled;

    /* 显示模式 */
    bool                    dpms_off;
    uint32_t                active_outputs;

    /* 电源状态 */
    enum disp_power_state {
        DISP_POWER_ON       = 0,  // 全供电
        DISP_POWER_STANDBY  = 1,  // 待机（仅保留AUX）
        DISP_POWER_OFF      = 2,  // 完全断电
    } state;

    /* PHY电源控制 */
    uint32_t                phy_power_ctrl;
    uint32_t                phy_power_status;

    /* DMCU（显示微控制器）状态 */
    struct dmcu_state      *dmcu;

    /* 唤醒事件 */
    struct work_struct      hotplug_work;
};

// Display电源门控入口
static int display_enter_powergate(struct display_power_domain *disp_pg,
                                    struct amdgpu_device *adev)
{
    int i;

    /* 检查是否有活动显示器 */
    if (disp_pg->active_outputs > 0)
        return -EBUSY;

    if (disp_pg->dmcu && disp_pg->dmcu->fw_state != DMCU_FW_IDLE)
        return -EBUSY;

    /* 步骤1: 关闭显示引擎时钟 */
    for (i = 0; i < adev->mode_info.num_crtc; i++) {
        WREG32(regCRTC_MASTER_UPDATE_LOCK + i, 1);
        WREG32(regCRTC_DISP_CTRL + i, 0);
        WREG32(regCRTC_MASTER_UPDATE_LOCK + i, 0);
    }

    /* 步骤2: 关闭PHY */
    for (i = 0; i < adev->mode_info.num_phy; i++) {
        disp_pg->phy_power_ctrl = RREG32(regDCIO_PHY_POWERGT +
                                          i * 4);
        WREG32(regDCIO_PHY_POWERGT + i * 4, 0);
    }

    /* 步骤3: 发送SMU消息 */
    smu_send_message(adev, SMU_MSG_POWERGATE_DISP, 1);
    smu_wait_for_response(adev, SMU_MSG_POWERGATE_DISP,
                           DISP_PG_TIMEOUT_MS);

    disp_pg->state = DISP_POWER_OFF;
    return 0;
}

// Display唤醒（热插拔检测）
static void display_wake_on_hotplug(struct work_struct *work)
{
    struct display_power_domain *disp_pg =
        container_of(work, struct display_power_domain, hotplug_work);
    struct amdgpu_device *adev = container_of(disp_pg,
        struct amdgpu_device, disp_pg);

    /* 检查是否DPMS OFF状态 */
    if (!disp_pg->dpms_off)
        return;

    /* 唤醒显示域 */
    smu_send_message(adev, SMU_MSG_POWERGATE_DISP, 0);
    smu_wait_for_response(adev, SMU_MSG_POWERGATE_DISP,
                           DISP_PG_TIMEOUT_MS);

    /* 重新初始化PHY */
    display_init_phy(adev);

    /* 恢复显示模式 */
    display_restore_mode(adev);

    disp_pg->state = DISP_POWER_ON;
}
```

### 3.6 电源域依赖关系管理

不同电源域之间存在功能依赖关系，例如VCN解码时需要SDMA进行数据传输，GFX计算时需要VCN输出显示帧。域控制必须正确处理这些依赖：

```c
// 电源域依赖关系表
struct power_domain_dependency {
    uint32_t            domain_id;
    uint32_t            num_deps;
    uint32_t            dep_domains[8];
    enum dep_type {
        DEP_TYPE_HARD  = 0,  // 硬依赖：必须先唤醒
        DEP_TYPE_SOFT  = 1,  // 软依赖：建议先唤醒
        DEP_TYPE_ASYNC = 2,  // 异步依赖：可并行唤醒
    } dep_types[8];
};

static const struct power_domain_dependency domain_deps[] = {
    {
        .domain_id  = POWER_DOMAIN_GFX,
        .num_deps   = 2,
        .dep_domains = {POWER_DOMAIN_SDMA0, POWER_DOMAIN_SDMA1},
        .dep_types  = {DEP_TYPE_SOFT, DEP_TYPE_SOFT},
    },
    {
        .domain_id  = POWER_DOMAIN_VCN,
        .num_deps   = 3,
        .dep_domains = {POWER_DOMAIN_SDMA0, POWER_DOMAIN_SDMA1,
                        POWER_DOMAIN_DISP},
        .dep_types  = {DEP_TYPE_HARD, DEP_TYPE_SOFT,
                        DEP_TYPE_ASYNC},
    },
    {
        .domain_id  = POWER_DOMAIN_DISP,
        .num_deps   = 1,
        .dep_domains = {POWER_DOMAIN_GFX},
        .dep_types  = {DEP_TYPE_ASYNC},
    },
};

// 依赖唤醒管理器
struct dep_wake_manager {
    uint32_t                pending_domains;
    uint32_t                completed_domains;
    struct completion       wake_done;

    /* 唤醒延迟追踪 */
    uint64_t                wake_start_ns;
    uint64_t                total_wake_latency_ns;
    uint32_t                num_serial_steps;
};

// 带依赖管理的唤醒函数
static int power_domain_wake_with_deps(
    struct amdgpu_device *adev,
    uint32_t             domain_id,
    struct dep_wake_manager *mgr)
{
    const struct power_domain_dependency *dep;
    int ret = 0;
    uint32_t i;

    dep = &domain_deps[domain_id];

    /* 标记本域为待唤醒 */
    mgr->pending_domains |= (1 << domain_id);
    mgr->wake_start_ns = ktime_get_ns();

    /* 按依赖类型处理前置域 */
    for (i = 0; i < dep->num_deps; i++) {
        uint32_t dep_domain = dep->dep_domains[i];

        if (mgr->completed_domains & (1 << dep_domain))
            continue;  /* 已完成 */

        switch (dep->dep_types[i]) {
        case DEP_TYPE_HARD:
            /* 硬依赖：串行等待前置域唤醒 */
            ret = _power_domain_wake(adev, dep_domain);
            if (ret) {
                dev_err(adev->dev,
                        "Failed to wake dep domain %u\n",
                        dep_domain);
                return ret;
            }
            mgr->completed_domains |= (1 << dep_domain);
            mgr->num_serial_steps++;
            break;

        case DEP_TYPE_SOFT:
            /* 软依赖：异步发出请求但不等待 */
            _power_domain_wake_async(adev, dep_domain);
            mgr->completed_domains |= (1 << dep_domain);
            break;

        case DEP_TYPE_ASYNC:
            /* 异步依赖：与目标域并行唤醒 */
            break;
        }
    }

    /* 唤醒目标域 */
    ret = _power_domain_wake(adev, domain_id);
    if (ret)
        return ret;

    mgr->completed_domains |= (1 << domain_id);
    mgr->total_wake_latency_ns = ktime_get_ns() -
                                  mgr->wake_start_ns;

    complete_all(&mgr->wake_done);
    return 0;
}
```

## 4. SMU固件与PG控制

### 4.1 SMU电源管理架构

SMU（System Management Unit）是AMDGPU中负责电源管理的专用微控制器。它运行独立的固件，通过专用的消息接口与驱动通信：

```c
// SMU电源门控消息定义
enum smu_pg_message {
    SMU_MSG_GFXOFF              = 0x01,  // 进入GFXOFF
    SMU_MSG_GFXOFF_EXIT         = 0x02,  // 退出GFXOFF
    SMU_MSG_POWERGATE_VCN       = 0x10,  // VCN电源门控
    SMU_MSG_POWERGATE_SDMA      = 0x11,  // SDMA电源门控
    SMU_MSG_POWERGATE_DISP      = 0x12,  // Display电源门控
    SMU_MSG_POWERGATE_JPEG      = 0x13,  // JPEG电源门控
    SMU_MSG_POWERGATE_GFX       = 0x14,  // GFX完全门控
    SMU_MSG_SET_PG_DOMAIN_MASK  = 0x20,  // 设置门控域掩码
    SMU_MSG_GET_PG_STATUS       = 0x21,  // 查询门控状态
    SMU_MSG_SET_SSR_CONFIG      = 0x30,  // 配置SSR参数
    SMU_MSG_PG_SWITCH_CTRL      = 0x31,  // 电源开关控制
    SMU_MSG_ISO_CTRL            = 0x32,  // 隔离单元控制
};

// SMU消息传输结构
struct smu_msg_pg {
    uint32_t            msg_id;
    uint32_t            arg;
    uint32_t            reserved;

    /* 响应 */
    int32_t             result;
    uint32_t            status;
    uint32_t            latency_us;
};

// SMU PG消息发送函数
static int smu_send_pg_message(struct amdgpu_device *adev,
                                enum smu_pg_message msg,
                                uint32_t arg)
{
    struct smu_context *smu = &adev->smu;
    struct smu_msg_pg pg_msg;
    int ret;
    uint32_t timeout_us;

    memset(&pg_msg, 0, sizeof(pg_msg));
    pg_msg.msg_id = msg;
    pg_msg.arg    = arg;

    mutex_lock(&smu->pg_msg_lock);

    /* 写入消息到SMU共享内存 */
    smu_write_shared_memory(smu, &pg_msg, sizeof(pg_msg));

    /* 触发SMU中断 */
    WREG32_FIELD(smu->mmio_base + smu->msg_trigger_offset,
                 SMU_MSG_TRIGGER, 1);

    /* 计算超时（不同类型的PG操作延迟不同） */
    switch (msg) {
    case SMU_MSG_GFXOFF:
        timeout_us = 1000;    // 1ms
        break;
    case SMU_MSG_POWERGATE_VCN:
        timeout_us = 10000;   // 10ms
        break;
    case SMU_MSG_POWERGATE_DISP:
        timeout_us = 50000;   // 50ms（含PHY训练）
        break;
    default:
        timeout_us = 5000;    // 5ms
    }

    /* 等待SMU响应 */
    ret = read_poll_timeout(
        RREG32_FIELD(smu->mmio_base + smu->resp_offset,
                     SMU_RESP_READY),
        val, val == 1, 10, timeout_us);

    if (ret) {
        dev_err(adev->dev,
                "SMU PG msg 0x%02X timeout after %uus\n",
                msg, timeout_us);
        mutex_unlock(&smu->pg_msg_lock);
        return -ETIMEDOUT;
    }

    /* 读取响应 */
    smu_read_shared_memory(smu, &pg_msg, sizeof(pg_msg));

    mutex_unlock(&smu->pg_msg_lock);

    return pg_msg.result;
}
```

### 4.2 SMU PG序列器

SMU固件内部实现了硬件序列器（HW Sequencer）来精确控制电源门控的时间序列：

```c
// SMU PG硬件序列器配置
struct smu_pg_sequencer {
    /* 序列阶段配置 */
    struct pg_phase_config {
        uint32_t            phase_id;
        uint32_t            delay_us;
        uint32_t            ctrl_mask;
        uint32_t            ctrl_value;
        bool                wait_for_ack;
    } phases[16];

    uint32_t                num_phases;
    uint32_t                current_phase;

    /* 定时器配置 */
    uint32_t                phase_timer_us;
    uint32_t                total_sequence_us;

    /* 状态 */
    enum seq_state {
        SEQ_IDLE,
        SEQ_RUNNING,
        SEQ_COMPLETE,
        SEQ_ERROR,
    } state;

    /* 错误恢复 */
    uint32_t                max_retries;
    uint32_t                retry_count;
    struct pg_phase_config  recovery_phases[8];
};

// 配置GFXOFF入口序列
static int smu_configure_gfxoff_entry_seq(
    struct smu_pg_sequencer *seq)
{
    /* 定义GFXOFF入口的8个阶段 */
    struct pg_phase_config gfxoff_entry[] = {
        {
            .phase_id      = 0,
            .delay_us      = 0,
            .ctrl_mask     = GFX_CLK_GATE_MASK,
            .ctrl_value    = GFX_CLK_GATE_EN,
            .wait_for_ack  = true,
            .desc          = "门控GFX时钟",
        },
        {
            .phase_id      = 1,
            .delay_us      = 1,
            .ctrl_mask     = GFX_ISO_MASK,
            .ctrl_value    = GFX_ISO_EN,
            .wait_for_ack  = true,
            .desc          = "使能隔离单元",
        },
        {
            .phase_id      = 2,
            .delay_us      = 10,
            .ctrl_mask     = GFX_HEADER_SW_MASK,
            .ctrl_value    = GFX_HEADER_SW_OFF,
            .wait_for_ack  = false,
            .desc          = "关闭Header开关（分级）",
        },
        {
            .phase_id      = 3,
            .delay_us      = 5,
            .ctrl_mask     = GFX_FOOTER_SW_MASK,
            .ctrl_value    = GFX_FOOTER_SW_OFF,
            .wait_for_ack  = false,
            .desc          = "关闭Footer开关",
        },
        {
            .phase_id      = 4,
            .delay_us      = 50,
            .ctrl_mask     = GFX_RETENTION_MASK,
            .ctrl_value    = GFX_RETENTION_EN,
            .wait_for_ack  = true,
            .desc          = "使能保持寄存器",
        },
        {
            .phase_id      = 5,
            .delay_us      = 0,
            .ctrl_mask     = GFX_PWR_RAIL_MASK,
            .ctrl_value    = GFX_PWR_RAIL_OFF,
            .wait_for_ack  = true,
            .desc          = "关闭电源轨",
        },
        {
            .phase_id      = 6,
            .delay_us      = 100,
            .ctrl_mask     = 0,
            .ctrl_value    = 0,
            .wait_for_ack  = false,
            .desc          = "等待电压稳定",
        },
        {
            .phase_id      = 7,
            .delay_us      = 0,
            .ctrl_mask     = GFX_PG_STATUS_MASK,
            .ctrl_value    = GFX_PG_STATUS_OFF,
            .wait_for_ack  = true,
            .desc          = "确认PG完成",
        },
    };

    seq->num_phases = ARRAY_SIZE(gfxoff_entry);
    memcpy(seq->phases, gfxoff_entry,
           sizeof(gfxoff_entry));

    /* 计算总序列时间 */
    seq->total_sequence_us = 0;
    for (int i = 0; i < seq->num_phases; i++)
        seq->total_sequence_us += seq->phases[i].delay_us;

    return 0;
}

// 执行PG序列
static int smu_execute_pg_sequence(struct smu_pg_sequencer *seq,
                                    void __iomem *mmio_base)
{
    int ret = 0;

    seq->state = SEQ_RUNNING;

    for (seq->current_phase = 0;
         seq->current_phase < seq->num_phases;
         seq->current_phase++) {

        struct pg_phase_config *phase =
            &seq->phases[seq->current_phase];

        /* 写入控制寄存器 */
        if (phase->ctrl_mask) {
            uint32_t val = RREG32(mmio_base + phase->ctrl_mask);
            val = (val & ~phase->ctrl_mask) | phase->ctrl_value;
            WREG32(mmio_base + phase->ctrl_mask, val);
        }

        /* 等待固定延迟 */
        if (phase->delay_us > 0)
            udelay(phase->delay_us);

        /* 等待硬件确认 */
        if (phase->wait_for_ack) {
            ret = read_poll_timeout(
                RREG32(mmio_base + phase->ctrl_mask),
                val, (val & phase->ctrl_mask) == phase->ctrl_value,
                1, 1000);
            if (ret) {
                dev_err(dev, "PG phase %d timeout\n",
                        phase->phase_id);
                seq->state = SEQ_ERROR;
                goto recover;
            }
        }
    }

    seq->state = SEQ_COMPLETE;
    return 0;

recover:
    /* 错误恢复：反向执行恢复序列 */
    for (int i = seq->current_phase; i > 0; i--) {
        /* 恢复操作... */
    }
    return ret;
}
```

### 4.3 电源轨控制

SMU通过PMBUS/I2C接口控制外部电压调节器（Voltage Regulator, VR）：

```c
// VR控制器接口
struct vr_controller {
    /* PMBUS/I2C接口 */
    struct i2c_adapter      *i2c_bus;
    uint8_t                 vr_addr;
    uint32_t                i2c_speed_khz;

    /* VR型号参数 */
    enum vr_type {
        VR_TYPE_INTEL_IMVP9,
        VR_TYPE_AMD_SVI3,
        VR_TYPE_PMBUS_COMPLIANT,
    } type;

    /* 电源轨参数 */
    struct power_rail {
        const char          *name;
        uint32_t            voltage_mv;
        uint32_t            current_a;
        uint32_t            max_current_a;
        bool                enabled;
    } rails[VR_MAX_RAILS];

    /* 控制接口 */
    int (*set_voltage)(struct vr_controller *vr,
                       uint32_t rail_id, uint32_t mv);
    int (*set_current_limit)(struct vr_controller *vr,
                              uint32_t rail_id, uint32_t ma);
    int (*read_voltage)(struct vr_controller *vr,
                        uint32_t rail_id, uint32_t *mv);
    int (*read_current)(struct vr_controller *vr,
                        uint32_t rail_id, uint32_t *ma);
    int (*enable_rail)(struct vr_controller *vr,
                       uint32_t rail_id, bool enable);
};

// SVI3协议电压设置
static int svi3_set_voltage(struct vr_controller *vr,
                             uint32_t rail_id, uint32_t mv)
{
    uint8_t buf[3];
    int ret;

    /* SVI3协议包格式 */
    buf[0] = SVI3_CMD_SET_VOLTAGE | rail_id;
    buf[1] = (mv >> 4) & 0xFF;       // 电压高8位
    buf[2] = ((mv & 0xF) << 4) |      // 电压低4位 + VBOOT
             SVI3_VBOOT_DISABLED;

    ret = i2c_master_send(vr->i2c_bus, vr->vr_addr,
                          buf, 3);
    if (ret < 0)
        return ret;

    vr->rails[rail_id].voltage_mv = mv;
    return 0;
}

// 电源轨使能控制（PG核心操作）
static int power_rail_enable(struct vr_controller *vr,
                              uint32_t rail_id, bool enable)
{
    uint8_t buf[2];
    int ret;

    if (enable) {
        /* 开启电源轨：设置软启动斜率 */
        buf[0] = SVI3_CMD_ENABLE_RAIL | rail_id;
        buf[1] = SVI3_SLEW_RATE_10MV_US;

        ret = i2c_master_send(vr->i2c_bus, vr->vr_addr, buf, 2);
        if (ret < 0)
            return ret;

        /* 等待输出电压稳定 */
        mdelay(VR_SETTLE_TIME_MS);
    } else {
        /* 关闭电源轨：先降低电压再关闭 */
        svi3_set_voltage(vr, rail_id, VR_SAFE_SHUTDOWN_MV);
        udelay(VR_SHUTDOWN_DELAY_US);

        buf[0] = SVI3_CMD_DISABLE_RAIL | rail_id;
        buf[1] = 0;

        ret = i2c_master_send(vr->i2c_bus, vr->vr_addr, buf, 2);
        if (ret < 0)
            return ret;
    }

    vr->rails[rail_id].enabled = enable;

    dev_info(vr->dev,
             "Power rail %s (ID %u) %s\n",
             vr->rails[rail_id].name, rail_id,
             enable ? "ON" : "OFF");

    return 0;
}
```

## 5. RDNA3多芯片电源管理

### 5.1 GCD/MCD独立门控

RDNA3架构采用chiplet设计，包含一个GCD（Graphics Compute Die）和多个MCD（Memory Cache Die）。每个die可以独立进行电源门控：

```c
// RDNA3 Chiplet电源域
struct chiplet_power_domain {
    /* GCD电源域 */
    struct {
        bool                pg_enabled;
        uint32_t            voltage_mv;
        uint32_t            freq_mhz;
        enum gcd_pg_state {
            GCD_PG_ON,
            GCD_PG_LIGHT_SLEEP,
            GCD_PG_DEEP_SLEEP,
            GCD_PG_OFF,
        } state;

        /* GCD内部的CU级门控 */
        uint32_t            active_cu_count;
        uint32_t            total_cu_count;
        uint32_t            cu_pg_mask;      // 位图：每CU是否门控
    } gcd;

    /* MCD电源域（最多8个） */
    struct mcd_domain {
        bool                present;
        bool                pg_enabled;

        enum mcd_pg_state {
            MCD_PG_ON,
            MCD_PG_SELF_REFRESH,
            MCD_PG_OFF,
        } state;

        /* 缓存保持 */
        bool                cache_retention;
        uint32_t            self_refresh_delay_us;

        /* 电源轨 */
        uint32_t            vdd_mcd_mv;
    } mcds[8];
};

// RDNA3芯片间PG协调
static int rdna3_chiplet_pg_coordinate(
    struct chiplet_power_domain *chiplet,
    struct amdgpu_device *adev)
{
    uint32_t i;
    int ret;

    /* 1. 检查所有MCD的空闲状态 */
    bool all_mcd_idle = true;
    for (i = 0; i < 8; i++) {
        if (!chiplet->mcds[i].present)
            continue;

        if (chiplet->mcds[i].state != MCD_PG_OFF &&
            chiplet->mcds[i].state != MCD_PG_SELF_REFRESH) {
            all_mcd_idle = false;
        }
    }

    /* 2. 如果所有MCD空闲，可尝试GCD深度休眠 */
    if (all_mcd_idle &&
        chiplet->gcd.active_cu_count == 0 &&
        chiplet->gcd.state != GCD_PG_OFF) {

        ret = smu_send_message(adev,
                                SMU_MSG_GCD_DEEP_SLEEP, 1);
        if (ret == 0) {
            chiplet->gcd.state = GCD_PG_DEEP_SLEEP;
        }
    }

    /* 3. MCD自刷新控制 */
    for (i = 0; i < 8; i++) {
        if (!chiplet->mcds[i].present)
            continue;

        if (chiplet->mcds[i].state == MCD_PG_ON &&
            chiplet->gcd.state == GCD_PG_DEEP_SLEEP) {

            /* 进入自刷新模式 */
            smu_send_message(adev,
                SMU_MSG_MCD_SELF_REFRESH + i, 1);
            chiplet->mcds[i].state = MCD_PG_SELF_REFRESH;
        }
    }

    return 0;
}

// CU（计算单元）级精细电源门控
struct cu_power_gate {
    uint32_t                cu_bitmap;          // 每CU位图
    uint32_t                active_cu_count;
    uint32_t                cu_pg_threshold;    // 门控阈值
    uint32_t                cu_ungate_latency_ns;

    /* SA（着色器阵列）级控制 */
    struct sa_pg_state {
        uint32_t            sa_id;
        uint32_t            cu_mask;            // 本SA的CU位图
        uint32_t            pg_cu_mask;         // 已门控CU
        uint32_t            active_wave_count;
    } sas[4];

    /* 工作负载感知调度器接口 */
    bool                    (*should_gate_cu)(uint32_t cu_id,
                                              uint32_t idle_cycles);
};

// RDNA3 CU级PG决策
static int rdna3_cu_pg_decision(struct cu_power_gate *cu_pg,
                                 uint32_t sa_id)
{
    struct sa_pg_state *sa = &cu_pg->sas[sa_id];
    uint32_t gatable_cus;
    uint32_t cu_id;

    /* 找出空闲CU（无wave在运行） */
    gatable_cus = sa->cu_mask & ~cu_pg->cu_bitmap;

    /* 检查门控阈值：至少需要连续空闲N个周期 */
    for_each_set_bit(cu_id, &gatable_cus, 32) {
        if (cu_pg->should_gate_cu(cu_id, 1000)) {
            /* 门控此CU */
            sa->pg_cu_mask |= (1 << cu_id);

            /* 发送SMU消息门控单个CU */
            smu_send_message(cu_pg->smu,
                SMU_MSG_POWERGATE_CU, cu_id);
        }
    }

    /* 如果整个SA都空，门控整个SA（更深层休眠） */
    if (sa->pg_cu_mask == sa->cu_mask &&
        sa->active_wave_count == 0) {

        smu_send_message(cu_pg->smu,
            SMU_MSG_POWERGATE_SA, sa_id);
    }

    return 0;
}
```

### 5.2 Chiplet间唤醒广播

当一个die需要访问另一个die的资源时，需要触发跨die唤醒：

```c
// 跨die唤醒广播
struct cross_die_wakeup {
    /* 唤醒广播掩码 */
    uint32_t                broadcast_mask;

    /* 各die唤醒状态 */
    struct die_wake_state {
        bool                wake_requested;
        uint64_t            wake_start_ns;
        uint64_t            wake_complete_ns;
        uint32_t            wake_latency_ns;
    } dies[4];

    /* Infinity Fabric接口 */
    struct ifi_wake_ctrl {
        uint32_t            ifi_wake_reg;
        uint32_t            ifi_ack_reg;
        uint32_t            ifi_timeout_us;
    } ifi;

    /* 完成回调 */
    void (*wake_complete_cb)(uint32_t die_mask);
};

// GCD唤醒MCD序列
static int gcd_wake_mcd(struct cross_die_wakeup *cdw,
                         uint32_t mcd_id)
{
    int ret;
    uint64_t start_ns = ktime_get_ns();

    cdw->dies[mcd_id].wake_requested = true;
    cdw->dies[mcd_id].wake_start_ns = start_ns;

    /* 通过Infinity Fabric发送唤醒信号 */
    WREG32(cdw->ifi.ifi_wake_reg + mcd_id * 4,
           CROSS_DIE_WAKE_REQ);

    /* 等待MCD确认 */
    ret = read_poll_timeout(
        RREG32(cdw->ifi.ifi_ack_reg + mcd_id * 4),
        val, val & CROSS_DIE_WAKE_ACK,
        1, cdw->ifi.ifi_timeout_us);

    if (ret) {
        dev_err(dev, "MCD %u wake timeout\n", mcd_id);
        return -ETIMEDOUT;
    }

    cdw->dies[mcd_id].wake_complete_ns = ktime_get_ns();
    cdw->dies[mcd_id].wake_latency_ns =
        cdw->dies[mcd_id].wake_complete_ns -
        cdw->dies[mcd_id].wake_start_ns;

    /* 广播唤醒完成 */
    cdw->broadcast_mask |= (1 << mcd_id);
    if (cdw->wake_complete_cb)
        cdw->wake_complete_cb(cdw->broadcast_mask);

    return 0;
}
```

## 6. PG性能分析与优化

### 6.1 唤醒延迟分解

将PG唤醒延迟分解到各个子阶段，以识别瓶颈：

```c
// PG唤醒延迟分析器
struct pg_latency_analyzer {
    /* 各阶段延迟统计（ns） */
    struct phase_latency {
        const char      *name;
        uint64_t        min_ns;
        uint64_t        max_ns;
        uint64_t        avg_ns;
        uint64_t        total_ns;
        uint32_t        sample_count;
    } phases[PG_PHASE_MAX];

    /* 当前采样数据 */
    uint64_t            sample_start_ns;
    uint32_t            current_phase_idx;

    /* 唤醒源分类 */
    struct wake_source_stats {
        enum wake_source {
            WAKE_CMD_BUFFER,     // 命令缓冲区提交
            WAKE_MMIO_ACCESS,    // MMIO访问
            WAKE_INTERRUPT,      // 中断
            WAKE_DMA_COMPLETE,   // DMA完成
            WAKE_TIMER,          // 定时器
            WAKE_VBLANK,         // VBlank
        } source;
        uint32_t        count;
        uint64_t        total_latency_ns;
    } wake_sources[6];
};

// 延迟采样函数
static void pg_record_latency_sample(
    struct pg_latency_analyzer *analyzer,
    enum pg_phase phase,
    uint64_t latency_ns)
{
    struct phase_latency *pl = &analyzer->phases[phase];

    pl->sample_count++;

    if (latency_ns < pl->min_ns || pl->sample_count == 1)
        pl->min_ns = latency_ns;

    if (latency_ns > pl->max_ns)
        pl->max_ns = latency_ns;

    pl->total_ns += latency_ns;
    pl->avg_ns = pl->total_ns / pl->sample_count;
}

// 标记PG阶段开始
static void pg_phase_start(struct pg_latency_analyzer *analyzer,
                            enum pg_phase phase)
{
    analyzer->current_phase_idx = phase;
    analyzer->sample_start_ns = ktime_get_ns();
}

// 标记PG阶段结束并记录延迟
static void pg_phase_end(struct pg_latency_analyzer *analyzer)
{
    uint64_t latency_ns;

    latency_ns = ktime_get_ns() - analyzer->sample_start_ns;
    pg_record_latency_sample(analyzer,
                             analyzer->current_phase_idx,
                             latency_ns);
}

### 6.2 PG盈亏平衡分析

PG的进入和退出都有显著延迟和功耗开销。盈亏平衡点是最小空闲时间，只有超过此值时PG才有利：

```c
// PG盈亏计算
struct pg_break_even {
    /* 进入开销 */
    uint64_t            entry_energy_nj;    // 进入序列消耗能量
    uint64_t            entry_latency_ns;   // 进入序列延迟

    /* 退出开销 */
    uint64_t            exit_energy_nj;     // 退出序列消耗能量
    uint64_t            exit_latency_ns;    // 退出序列延迟

    /* 功耗参数 */
    uint32_t            active_power_mw;    // 活动功耗
    uint32_t            idle_power_mw;      // 空闲（CG）功耗
    uint32_t            leakage_power_mw;   // 泄漏（PG关断）功耗

    /* 计算结果 */
    uint64_t            break_even_ns;      // 盈亏平衡时间（ns）
    uint64_t            energy_saved_pj;    // 节省能量（pJ）
};

// 计算PG盈亏平衡点
static struct pg_break_even pg_calculate_break_even(
    uint32_t active_power_mw,
    uint32_t idle_power_mw,
    uint32_t leakage_power_mw,
    uint64_t entry_latency_ns,
    uint64_t exit_latency_ns,
    uint64_t entry_energy_nj,
    uint64_t exit_energy_nj)
{
    struct pg_break_even be;

    be.active_power_mw   = active_power_mw;
    be.idle_power_mw     = idle_power_mw;
    be.leakage_power_mw  = leakage_power_mw;
    be.entry_latency_ns  = entry_latency_ns;
    be.exit_latency_ns   = exit_latency_ns;
    be.entry_energy_nj   = entry_energy_nj;
    be.exit_energy_nj    = exit_energy_nj;

    /* 不进入PG时每ns消耗的额外泄漏能量 */
    /* idle功耗 - leakage功耗 = 额外节省 */
    uint64_t save_power_mw = idle_power_mw - leakage_power_mw;

    if (save_power_mw == 0) {
        be.break_even_ns = UINT64_MAX;  // 永远不值得PG
        return be;
    }

    /* PG总开销 = 进入能量 + 退出能量 */
    uint64_t total_overhead_nj = entry_energy_nj + exit_energy_nj;

    /* 盈亏平衡时间 = 开销 / 节省功率 */
    /* save_power_mw = mJ/s = pJ/ns */
    be.break_even_ns = (total_overhead_nj * 1000000) / save_power_mw;

    return be;
}

// RDNA3各域PG盈亏平衡点
static const struct pg_break_even rdna3_pg_breakeven[] = {
    {
        .domain         = "GFX (GFXOFF)",
        .entry_latency_ns  = 5000,     // 5μs
        .exit_latency_ns   = 10000,    // 10μs
        .entry_energy_nj   = 500,      // 500nJ
        .exit_energy_nj    = 1000,     // 1000nJ
        .idle_power_mw     = 15000,    // 15W idle
        .leakage_power_mw  = 500,      // 0.5W leakage in PG
        .break_even_ns     = 100000,   // ~100μs
    },
    {
        .domain         = "VCN",
        .entry_latency_ns  = 10000,    // 10μs
        .exit_latency_ns   = 50000,    // 50μs
        .entry_energy_nj   = 200,
        .exit_energy_nj    = 500,
        .idle_power_mw     = 2000,
        .leakage_power_mw  = 50,
        .break_even_ns     = 350000,   // ~350μs
    },
};
```

### 6.3 自适应PG策略

根据工作负载特征动态调整PG门控策略：

```c
// 自适应PG策略控制器
struct adaptive_pg_controller {
    /* 历史工作负载特征 */
    struct workload_history {
        uint64_t        avg_idle_duration_ns;
        uint64_t        min_idle_duration_ns;
        uint64_t        max_idle_duration_ns;
        uint32_t        idle_event_count;
        uint32_t        false_wake_count;   // 唤醒后很快又空闲
    } history;

    /* 策略参数 */
    struct pg_policy_params {
        uint32_t        min_idle_threshold_ns;  // 最小空闲阈值
        uint32_t        max_idle_threshold_ns;  // 最大空闲阈值
        uint32_t        current_threshold_ns;
        float           hysteresis_factor;     // 磁滞因子
        bool            aggressive_mode;       // 激进模式
    } params;

    /* 策略决策 */
    enum pg_policy {
        PG_POLICY_CONSERVATIVE,   // 保守：长空闲才PG
        PG_POLICY_BALANCED,       // 平衡：盈亏平衡点
        PG_POLICY_AGGRESSIVE,     // 激进：短空闲也PG
    } policy;
};

// 自适应阈值更新
static void adaptive_pg_update_threshold(
    struct adaptive_pg_controller *ctrl)
{
    uint64_t avg_idle = ctrl->history.avg_idle_duration_ns;
    uint32_t false_rate;

    /* 计算误唤醒率 */
    false_rate = (ctrl->history.idle_event_count > 0) ?
        (ctrl->history.false_wake_count * 100 /
         ctrl->history.idle_event_count) : 0;

    /* 根据误唤醒率调整策略 */
    if (false_rate > 30) {
        /* 误唤醒率过高，增加阈值 */
        ctrl->policy = PG_POLICY_CONSERVATIVE;
        ctrl->params.current_threshold_ns =
            min(ctrl->params.current_threshold_ns * 2,
                ctrl->params.max_idle_threshold_ns);
    } else if (false_rate < 5 && avg_idle > 0) {
        /* 误唤醒率低，尝试减小阈值 */
        ctrl->policy = PG_POLICY_AGGRESSIVE;
        ctrl->params.current_threshold_ns =
            max(ctrl->params.current_threshold_ns / 2,
                ctrl->params.min_idle_threshold_ns);
    } else {
        ctrl->policy = PG_POLICY_BALANCED;
    }

    /* 应用磁滞避免抖动 */
    ctrl->params.current_threshold_ns =
        ctrl->params.current_threshold_ns *
        ctrl->params.hysteresis_factor;
}

## 7. sysfs接口与调试

### 7.1 PG状态查询

```c
// sysfs：电源门控状态显示
static ssize_t pg_state_show(struct device *dev,
                              struct device_attribute *attr,
                              char *buf)
{
    struct amdgpu_device *adev = dev_get_drvdata(dev);
    int size = 0;

    size += scnprintf(buf + size, PAGE_SIZE - size,
        "=== Power Gating States ===\n\n");

    /* GFX域 */
    size += scnprintf(buf + size, PAGE_SIZE - size,
        "GFX Domain:\n");
    size += scnprintf(buf + size, PAGE_SIZE - size,
        "  State:        %s\n",
        gfxoff_state_str(adev->gfxoff_mgr.state));
    size += scnprintf(buf + size, PAGE_SIZE - size,
        "  GFXOFF count: %d\n",
        atomic_read(&adev->gfxoff_mgr.gfxoff_count));
    size += scnprintf(buf + size, PAGE_SIZE - size,
        "  Entry latency: %llu ns\n",
        adev->gfxoff_mgr.entry_latency_ns);
    size += scnprintf(buf + size, PAGE_SIZE - size,
        "  Exit latency:  %llu ns\n",
        adev->gfxoff_mgr.exit_latency_ns);
    size += scnprintf(buf + size, PAGE_SIZE - size,
        "  Total GFXOFF:  %llu ns\n",
        adev->gfxoff_mgr.total_gfxoff_time_ns);
    size += scnprintf(buf + size, PAGE_SIZE - size,
        "  Deny reasons:  0x%08X\n\n",
        adev->gfxoff_mgr.deny_reasons);

    /* VCN域 */
    size += scnprintf(buf + size, PAGE_SIZE - size,
        "VCN Domain:\n");
    size += scnprintf(buf + size, PAGE_SIZE - size,
        "  State:        %s\n",
        vcn_pg_state_str(adev->vcn_pg.state));
    size += scnprintf(buf + size, PAGE_SIZE - size,
        "  Decode sessions: %d\n",
        atomic_read(&adev->vcn_pg.decode_session_count));
    size += scnprintf(buf + size, PAGE_SIZE - size,
        "  Encode sessions: %d\n\n",
        atomic_read(&adev->vcn_pg.encode_session_count));

    /* SDMA域 */
    size += scnprintf(buf + size, PAGE_SIZE - size,
        "SDMA Domain:\n");
    for (int i = 0; i < adev->sdma_pg.instance_count; i++) {
        size += scnprintf(buf + size, PAGE_SIZE - size,
            "  Instance %d: %s\n",
            i, sdma_pg_state_str(
                adev->sdma_pg.instances[i].state));
    }

    return size;
}

// sysfs：PG控制接口
static ssize_t pg_control_store(struct device *dev,
                                 struct device_attribute *attr,
                                 const char *buf, size_t count)
{
    struct amdgpu_device *adev = dev_get_drvdata(dev);
    char cmd[32];
    uint32_t domain, action;

    if (sscanf(buf, "%31s %u %u", cmd, &domain, &action) < 3)
        return -EINVAL;

    if (strcmp(cmd, "pg") == 0) {
        switch (domain) {
        case 0: /* GFX */
            if (action)
                gfx_enter_gfxoff(&adev->gfxoff_mgr, adev);
            else
                gfx_exit_gfxoff(&adev->gfxoff_mgr, adev);
            break;
        case 1: /* VCN */
            vcn_powergate(&adev->vcn_pg, adev, !!action);
            break;
        case 2: /* SDMA0 */
            sdma_instance_powergate(&adev->sdma_pg.instances[0],
                                     adev, 0);
            break;
        default:
            return -EINVAL;
        }
    }

    return count;
}
```

### 7.2 调试寄存器转储

```c
// PG调试寄存器转储
static void dump_pg_registers(struct amdgpu_device *adev,
                               struct seq_file *m)
{
    uint32_t data;

    seq_printf(m, "=== Power Gating Registers ===\n\n");

    /* GFX PG控制寄存器 */
    data = RREG32_SOC15(GC, 0, regGFX_PG_CTRL);
    seq_printf(m, "GFX_PG_CTRL    (0x%05X): 0x%08X\n",
               regGFX_PG_CTRL, data);
    seq_printf(m, "  PG_EN:       %d\n",
               (data >> GFX_PG_CTRL__PG_EN_SHIFT) & 1);
    seq_printf(m, "  GFXOFF_ST:   %d\n",
               (data >> GFX_PG_CTRL__GFXOFF_ST_SHIFT) & 0x7);

    /* GFX电源状态寄存器 */
    data = RREG32_SOC15(GC, 0, regGFX_PG_STATUS);
    seq_printf(m, "GFX_PG_STATUS  (0x%05X): 0x%08X\n",
               regGFX_PG_STATUS, data);
    seq_printf(m, "  POWER_GOOD:  %d\n",
               (data >> GFX_PG_STATUS__POWER_GOOD_SHIFT) & 1);
    seq_printf(m, "  ISO_ON:      %d\n",
               (data >> GFX_PG_STATUS__ISO_ON_SHIFT) & 1);
    seq_printf(m, "  CLK_OFF:     %d\n",
               (data >> GFX_PG_STATUS__CLK_OFF_SHIFT) & 1);

    /* VCN PG寄存器 */
    data = RREG32_SOC15(VCN, 0, regVCN_PG_CTRL);
    seq_printf(m, "\nVCN_PG_CTRL    (0x%05X): 0x%08X\n",
               regVCN_PG_CTRL, data);

    /* SDMA PG寄存器 */
    data = RREG32_SOC15(SDMA0, 0, regSDMA_PG_CTRL);
    seq_printf(m, "SDMA_PG_CTRL   (0x%05X): 0x%08X\n",
               regSDMA_PG_CTRL, data);

    /* 电源轨监控 */
    seq_printf(m, "\n=== Power Rails ===\n");
    seq_printf(m, "VDD_GFX:  %u mV / %u A\n",
               adev->vr_ctrl.rails[RAIL_GFX].voltage_mv,
               adev->vr_ctrl.rails[RAIL_GFX].current_a);
    seq_printf(m, "VDD_VCN:  %u mV / %u A\n",
               adev->vr_ctrl.rails[RAIL_VCN].voltage_mv,
               adev->vr_ctrl.rails[RAIL_VCN].current_a);
    seq_printf(m, "VDD_SOC:  %u mV / %u A\n",
               adev->vr_ctrl.rails[RAIL_SOC].voltage_mv,
               adev->vr_ctrl.rails[RAIL_SOC].current_a);

    /* PG序列延迟 */
    seq_printf(m, "\n=== PG Latency Stats ===\n");
    for (int i = 0; i < PG_PHASE_MAX; i++) {
        struct phase_latency *pl = &adev->pg_latency.phases[i];
        if (pl->sample_count > 0) {
            seq_printf(m, "Phase %d (%s): %llu/%llu/%llu ns (min/avg/max) [%u samples]\n",
                       i, pl->name,
                       pl->min_ns, pl->avg_ns, pl->max_ns,
                       pl->sample_count);
        }
    }

    /* 浪涌电流统计 */
    seq_printf(m, "\n=== Inrush Current Events ===\n");
    seq_printf(m, "Peak inrush:  %u mA\n",
               adev->pg_stats.peak_inrush_ma);
    seq_printf(m, "Avg inrush:   %u mA\n",
               adev->pg_stats.avg_inrush_ma);
    seq_printf(m, "Events:       %u\n",
               adev->pg_stats.inrush_events);
}

## 8. NVIDIA GPU电源门控对比

### 8.1 NVIDIA PG架构

NVIDIA从Turing架构开始引入精细化的电源门控机制。与AMDGPU类似，NVIDIA将GPU划分为多个电源域：

| 特性 | AMD RDNA3 | NVIDIA Ada Lovelace |
|------|-----------|-------------------|
| 门控粒度 | CU级 + SA级 + 域级 | SM级 + GPC级 + 域级 |
| 控制固件 | SMU (专用Cortex-M) | GSP (RISC-V) |
| 电源域数 | 5+ 域 (GFX/VCN/SDMA/DISP/IO) | 6+ 域 (GRAPHICS/MEDIA/DISPLAY/MEM/IO) |
| 深度休眠 | GFXOFF (10μs唤醒) | CG_IDLE (8μs唤醒) |
| Chiplet PG | GCD/MCD独立门控 | N/A (单芯片) |
| 状态保存 | SSHB扫描链 | ROP (Register Obfuscation Pipeline) |
| 接口协议 | SVI3 (AMD定制) | IMVP9 (Intel VR规范) |
| 精细门控 | FGCG (每CU独立) | SM_GATING (每SM独立) |

### 8.2 NVIDIA的时钟门控 vs 电源门控层次

NVIDIA的低功耗层次（从浅到深）：
1. **CG_IDLE**：类似AMD RLC_CGCG，时钟门控
2. **CG_SLEEP**：类似AMD CGLS，更深层时钟门控
3. **PG_IDLE**：类似AMD GFXOFF，电源门控（~5μs唤醒）
4. **PG_DEEP**：类似AMD BACO，更深层电源门控（~50μs唤醒）

### 8.3 关键差异

1. **GSP固件架构**：NVIDIA的GSP（GPU System Processor）基于RISC-V，比AMD的SMU（基于Cortex-M）具有更开放的指令集
2. **VR接口**：AMD使用SVI3（与VR通过I2C通信），NVIDIA使用IMVP9（Intel定义的移动VR规范）
3. **状态保存机制**：AMD使用SSHB扫描链（硬件自动扫描），NVIDIA使用ROP（寄存器混淆管道，增加了安全保护）

## 9. 总结

电源门控（Power Gating）是现代GPU低功耗设计中最有效的技术之一，能够消除高达99%的泄漏功耗。本文档详细介绍了：

1. **硬件基础**：电源开关（Header/Footer）、隔离单元、SSR状态保存恢复机制
2. **AMDGPU电源域架构**：GFX/VCN/SDMA/DISP域的独立PG控制和GFXOFF状态机
3. **SMU固件控制**：PG消息协议、硬件序列器、电源轨控制（SVI3协议）
4. **多芯片管理**：RDNA3 GCD/MCD独立门控和跨die唤醒
5. **性能分析**：延迟分解、盈亏平衡点计算、自适应策略
6. **调试接口**：sysfs状态查询、寄存器转储

PG与CG配合形成完整的电源管理层次：CG处理短时空闲（ns级），PG处理长时间空闲（μs级），两者协同实现GPU在各种工作负载下的最优能效。
