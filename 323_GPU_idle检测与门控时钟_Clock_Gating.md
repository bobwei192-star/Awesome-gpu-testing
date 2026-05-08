# GPU Idle检测与门控时钟 (Clock Gating) 深度分析

## 1. 概述

### 1.1 什么是Clock Gating

时钟门控（Clock Gating, CG）是GPU功耗管理中最重要的节能技术之一。其核心思想是：当某个功能单元（如着色器、纹理单元、内存控制器等）没有工作任务时，关闭其时钟信号，从而消除该单元的动态功耗。

在现代GPU中，Clock Gating是降低功耗的第一道防线，其重要性体现在：

- **响应速度**：纳秒级开关，比DVFS（微秒级）和Power Gating（毫秒级）快得多
- **零开销恢复**：从门控状态恢复到完全运行状态只需1-2个时钟周期
- **透明性**：对软件和驱动完全透明，完全由硬件自动管理
- **节能效果**：在典型工作负载下可降低30-50%的动态功耗

### 1.2 Clock Gating的分类

根据门控粒度和控制方式，Clock Gating可分为以下几个层次：

| 层次 | 粒度 | 控制方式 | 响应时间 | 节能效果 |
|------|------|---------|---------|---------|
| 寄存器级CG | 单个寄存器 | 本地门控逻辑 | 1周期 | 5-10% |
| 功能块级CG | ALU/纹理单元 | 硬件自动 | 2-5周期 | 15-25% |
| 引擎级CG | GFX/MEM/VCN | 硬件+固件协同 | 10-100周期 | 30-50% |
| 粗粒度CG | 整个IP模块 | SMU/驱动控制 | 1-10μs | 50-80% |
| 深度睡眠CG | SoC级 | 固件管理 | 10-100μs | 90%+ |

### 1.3 Idle检测原理

Idle检测是Clock Gating的前置条件，其基本原理是：

```
硬件活动监控器
    ↓ 持续监测
引擎活动状态 (active/idle)
    ↓ 满足门控条件
门控使能信号 (cg_enable)
    ↓
时钟控制单元 (Clock Controller)
    ↓
门控执行 (Clock Off/On)
```

Idle检测需要平衡两个关键指标：
- **检测准确率**：避免误判导致性能损失
- **检测延迟**：及时进入低功耗状态

### 1.4 演进历史

AMD GPU的Clock Gating技术演进：

| GPU架构 | 代际 | CG特性 | 改进点 |
|---------|------|--------|-------|
| TeraScale | HD 2000-6000 | 基础CG | 功能块级门控 |
| GCN 1-3 | HD 7000-R9 300 | 引擎级CG | 引入MGCG(Multi-level CG) |
| GCN 4-5 (Vega) | RX 400-VII | Fine-Grained CG | 每CU独立门控 |
| RDNA1 | RX 5000 | 增强FGCG | 改进门控延迟 |
| RDNA2 | RX 6000 | 自适应CG | 基于工作负载动态调整 |
| RDNA3 | RX 7000 | Chiplet CG | GCD/MCD独立门控 |

## 2. Clock Gating硬件实现基础

### 2.1 基本门控单元

Clock Gating的基本硬件单元是门控时钟单元（Gated Clock Cell）：

```
           ┌──────────────┐
 时钟 ────→│              │───→ 门控时钟输出
           │   AND/OR     │
 使能 ────→│   门控逻辑    │
           └──────────────┘
```

实现方式主要有两种：

**锁存器型（Latch-based CG）**：
- 使用透明锁存器消除毛刺
- 面积稍大，但时序更安全
- 适用于高频率场景

```verilog
module latch_cg_cell (
    input  wire clk,
    input  wire enable,
    input  wire scan_mode,
    output wire gated_clk
);
    wire latch_en = clk || !enable;
    reg  enable_latched;
    
    always_latch begin
        if (scan_mode)
            enable_latched = 1'b1;
        else if (latch_en)
            enable_latched = enable;
    end
    
    assign gated_clk = clk & enable_latched;
endmodule
```

**集成型（Integrated CG Cell）**：
- 标准单元库提供的专用CG单元
- 面积最优，时序特性好
- 综合工具自动插入

### 2.2 时钟网络架构

GPU的时钟网络采用H树或多级缓冲树结构：

```
       ┌── PLL ──→ Global Clock
       │
PLL ───┼── PLL ──→ GFX Clock Domain
       │           ├── CU Array (每个CU可独立门控)
       │           ├── L1 Cache (按bank门控)
       │           └── Geometry Processor
       │
       └── PLL ──→ Memory Clock Domain
                    ├── Memory Controller
                    ├── Infinity Fabric (IF)
                    └── L2 Cache
```

时钟域划分的关键原则：
- 每个时钟域具有独立的门控控制
- 相关功能单元共享同一时钟域
- 跨时钟域同步使用异步FIFO

### 2.3 门控使能条件

Clock Gating的使能条件由硬件活动检测器生成：

```c
/* 硬件活动检测器状态 */
struct activity_detector {
    /* 空闲计数器 */
    uint32_t idle_cycles;        /* 连续空闲周期数 */
    uint32_t idle_threshold;     /* 门控触发阈值 */
    
    /* 活动检测 */
    uint32_t active_flags;       /* 活动位图 */
    uint32_t quiesce_mask;       /* 静默掩码 */
    
    /* 门控状态 */
    enum cg_state {
        CG_STATE_ACTIVE,         /* 完全运行 */
        CG_STATE_IDLE_DETECT,    /* 空闲检测中 */
        CG_STATE_CG_PENDING,     /* 门控等待 */
        CG_STATE_CLOCK_OFF,      /* 时钟关闭 */
        CG_STATE_WAKEUP          /* 唤醒中 */
    } state;
};

/* 门控条件检查 */
static int check_gate_condition(struct activity_detector *det)
{
    /* 条件1: 连续空闲超过阈值 */
    if (det->idle_cycles < det->idle_threshold)
        return 0;
    
    /* 条件2: 所有活动标志清零 */
    if (det->active_flags & ~det->quiesce_mask)
        return 0;
    
    /* 条件3: 没有pending的内存请求 */
    if (has_pending_memory_requests())
        return 0;
    
    /* 条件4: 没有in-flight的纹理采样 */
    if (has_pending_texture_requests())
        return 0;
    
    return 1; /* 满足门控条件 */
}
```

### 2.4 唤醒机制

当门控的单元需要重新工作时，必须快速唤醒：

```
         Clock Gated          Wakeup Request
             │                      │
             ▼                      ▼
   ┌─────────────────┐    ┌─────────────────┐
   │    Clock Off     │───→│   Wakeup Start   │
   │   (零功耗)       │    │   (开启PLL)      │
   └─────────────────┘    └─────────────────┘
                                  │
                                  ▼
   ┌─────────────────┐    ┌─────────────────┐
   │   Fully Active   │←───│  Clock Stable   │
   │   (全速运行)     │    │  (等待锁相环)    │
   └─────────────────┘    └─────────────────┘
```

唤醒延迟的组成：
- PLL重锁时间：1-5μs（如果PLL也被关闭）
- 时钟树稳定时间：10-100ns
- 流水线刷新时间：1-10周期

```verilog
module wakeup_controller (
    input  wire clk,
    input  wire wakeup_req,
    input  wire pll_locked,
    output reg  clk_enable,
    output reg  wakeup_done
);
    typedef enum reg [1:0] {
        SLEEP = 2'b00,
        WAKEUP_PLL = 2'b01,
        WAKEUP_STABLE = 2'b10,
        ACTIVE = 2'b11
    } state_t;
    
    reg [1:0] state;
    reg [3:0] stable_counter;
    
    always_ff @(posedge clk or posedge wakeup_req) begin
        if (wakeup_req) begin
            state <= WAKEUP_PLL;
            clk_enable <= 0;
            wakeup_done <= 0;
        end else begin
            case (state)
                SLEEP: begin
                    clk_enable <= 0;
                    wakeup_done <= 0;
                end
                
                WAKEUP_PLL: begin
                    if (pll_locked) begin
                        state <= WAKEUP_STABLE;
                        stable_counter <= 8;
                    end
                end
                
                WAKEUP_STABLE: begin
                    stable_counter <= stable_counter - 1;
                    if (stable_counter == 0) begin
                        clk_enable <= 1;
                        state <= ACTIVE;
                        wakeup_done <= 1;
                    end
                end
                
                ACTIVE: begin
                    clk_enable <= 1;
                    wakeup_done <= 1;
                end
            endcase
        end
    end
endmodule
```

## 3. AMDGPU驱动中的Clock Gating实现

### 3.1 驱动CG架构

AMDGPU驱动中的Clock Gating管理分为三层：

```
用户态 (sysfs)
    │
    ▼
内核驱动层 (amdgpu_dpm.c/amdgpu_cg.c)
    ├── CG掩码管理 (cg_mask)
    ├── 状态查询 (cg_flags)
    └── 电源状态转换 (cg_ps_transition)
    │
    ▼
SMU固件层 (Power Management Firmware)
    ├── CG策略决策
    ├── Idle检测管理
    └── 状态转换控制
    │
    ▼
硬件层 (GPU)
    ├── 门控单元 (CG Cells)
    ├── 活动检测器
    └── 时钟控制器
```

### 3.2 CG数据结构

```c
/* amdgpu_cg.h - Clock Gating管理结构 */

/* 时钟门控标志位定义 */
#define AMD_CG_SUPPORT_GFX_MGCG       (1 << 0)
#define AMD_CG_SUPPORT_GFX_MGLS       (1 << 1)
#define AMD_CG_SUPPORT_GFX_CGCG       (1 << 2)
#define AMD_CG_SUPPORT_GFX_CGLS       (1 << 3)
#define AMD_CG_SUPPORT_GFX_FGCG       (1 << 4)
#define AMD_CG_SUPPORT_MC_MGCG        (1 << 5)
#define AMD_CG_SUPPORT_MC_LS          (1 << 6)
#define AMD_CG_SUPPORT_SDMA_MGCG      (1 << 7)
#define AMD_CG_SUPPORT_SDMA_LS        (1 << 8)
#define AMD_CG_SUPPORT_VCE_MGCG       (1 << 9)
#define AMD_CG_SUPPORT_VCE_LS         (1 << 10)
#define AMD_CG_SUPPORT_UVD_MGCG       (1 << 11)
#define AMD_CG_SUPPORT_UVD_LS         (1 << 12)
#define AMD_CG_SUPPORT_VCN_MGCG       (1 << 13)
#define AMD_CG_SUPPORT_VCN_LS         (1 << 14)
#define AMD_CG_SUPPORT_JPEG_MGCG      (1 << 15)
#define AMD_CG_SUPPORT_HDP_MGCG       (1 << 16)
#define AMD_CG_SUPPORT_HDP_LS         (1 << 17)
#define AMD_CG_SUPPORT_IH_MGCG        (1 << 18)
#define AMD_CG_SUPPORT_IH_LS          (1 << 19)
#define AMD_CG_SUPPORT_ATHUB_MGCG     (1 << 20)
#define AMD_CG_SUPPORT_ATHUB_LS       (1 << 21)
#define AMD_CG_SUPPORT_DF_MGCG        (1 << 22)

/* CG状态结构 */
struct amdgpu_cg_state {
    /* GFX引擎CG掩码 */
    uint32_t gfx_cg_mask;
    uint32_t gfx_cg_flags;
    
    /* 内存控制器CG */
    uint32_t mc_cg_mask;
    uint32_t mc_ls_mask;
    
    /* SDMA引擎CG */
    uint32_t sdma_cg_mask;
    uint32_t sdma_ls_mask;
    
    /* 多媒体引擎CG */
    uint32_t vcn_cg_mask;
    uint32_t jpeg_cg_mask;
    
    /* 其他引擎CG */
    uint32_t hdp_cg_mask;
    uint32_t ih_cg_mask;
    uint32_t athub_cg_mask;
    uint32_t df_cg_mask;
};

/* CG控制函数表 */
struct amdgpu_cg_funcs {
    /* 初始化CG */
    int (*init_cg)(struct amdgpu_device *adev);
    
    /* 设置CG掩码 */
    int (*set_cg_mask)(struct amdgpu_device *adev,
                       uint32_t mask, bool enable);
    
    /* 电源状态转换时的CG处理 */
    int (*cg_ps_transition)(struct amdgpu_device *adev,
                            bool enable);
    
    /* 更新CG状态 */
    int (*update_cg)(struct amdgpu_device *adev,
                     enum amdgpu_cg_block block,
                     enum amdgpu_cg_action action);
    
    /* 获取当前CG状态 */
    uint32_t (*get_cg_state)(struct amdgpu_device *adev,
                             enum amdgpu_cg_block block);
    
    /* 调试接口 */
    void (*dump_cg_status)(struct amdgpu_device *adev,
                           struct seq_file *m);
};
```

### 3.3 GFX引擎Clock Gating

GFX引擎是最复杂的CG管理单元，包含多个子模块：

```
GFX Engine Clock Domains
├── CP (Command Processor)
│   ├── ME (Micro Engine)
│   ├── PFP (Prefetch/Parser)
│   ├── CE (Constant Engine)
│   └── DE (Draw Engine)
├── SPI (Shader Processor Input)
├── SC (Scan Converter)
├── CB (Color Backend)
├── DB (Depth Backend)
├── TA/TD (Texture Address/Data)
├── PA (Primitive Assembly)
├── VGT (Vertex Group Tesselator)
├── IA (Input Assembly)
└── CU (Compute Unit) Array
    ├── CU0 ~ CUn (每个CU独立门控)
    ├── LDS (Local Data Share)
    └── GDS (Global Data Share)
```

每个CU的独立Clock Gating：

```c
/* CU级CG控制 */
struct cu_cg_control {
    uint32_t cu_mask;          /* CU使能位图 */
    uint32_t cu_cg_enable;     /* CU门控使能 */
    uint32_t idle_counter;     /* 空闲计数器 */
    uint32_t wakeup_latency;   /* 唤醒延迟(周期数) */
};

/* RDNA3 GFX CG寄存器偏移 */
#define regCGTS_CU0_CTRL         0x1A00
#define regCGTS_CU1_CTRL         0x1A01
#define regCGTS_CU2_CTRL         0x1A02
/* ... 每个CU独立的CG控制寄存器 */

#define CGTS_CU_CTRL__CG_ENABLE         (1 << 0)
#define CGTS_CU_CTRL__LS_ENABLE         (1 << 1)
#define CGTS_CU_CTRL__IDLE_THRESHOLD    (0xFF << 8)
#define CGTS_CU_CTRL__WAKEUP_DELAY      (0x3 << 16)

/* GFX引擎CG整体控制 */
#define regRLC_CGCG_CTRL          0x1B00
#define regRLC_CGLS_CTRL          0x1B01
#define regRLC_MGCG_CTRL          0x1B02

/* CGCG (Coarse-Grained Clock Gating) */
#define RLC_CGCG_CTRL__CG_ENABLE          (1 << 0)
#define RLC_CGCG_CTRL__IDLE_THRESHOLD     (0xFFFF << 16)

/* CGLS (Clock Gating Light Sleep) */
#define RLC_CGLS_CTRL__LS_ENABLE          (1 << 0)
#define RLC_CGLS_CTRL__LS_IDLE_THRESHOLD  (0xFFFF << 16)

/* MGCG (Multi-level Clock Gating) */
#define RLC_MGCG_CTRL__MGCG_ENABLE        (1 << 0)
#define RLC_MGCG_CTRL__FGCG_ENABLE        (1 << 1)
#define RLC_MGCG_CTRL__SPLL_SLEEP         (1 << 2)
```

GFX CG初始化流程：

```c
/* GFX Clock Gating初始化 */
static int gfx_v11_init_cg(struct amdgpu_device *adev)
{
    uint32_t data;
    int ret;

    /* 1. 使能MGCG (Multi-level Clock Gating) */
    if (adev->cg_flags & AMD_CG_SUPPORT_GFX_MGCG) {
        data = RREG32_SOC15(GC, 0, regRLC_MGCG_CTRL);
        data |= RLC_MGCG_CTRL__MGCG_ENABLE;
        WREG32_SOC15(GC, 0, regRLC_MGCG_CTRL, data);
    }

    /* 2. 使能CGCG (Coarse-Grained CG) */
    if (adev->cg_flags & AMD_CG_SUPPORT_GFX_CGCG) {
        data = RREG32_SOC15(GC, 0, regRLC_CGCG_CTRL);
        data &= ~RLC_CGCG_CTRL__IDLE_THRESHOLD_MASK;
        data |= (0xFF << 16); /* 设置空闲阈值 */
        data |= RLC_CGCG_CTRL__CG_ENABLE;
        WREG32_SOC15(GC, 0, regRLC_CGCG_CTRL, data);
    }

    /* 3. 使能CGLS (Clock Gating Light Sleep) */
    if (adev->cg_flags & AMD_CG_SUPPORT_GFX_CGLS) {
        data = RREG32_SOC15(GC, 0, regRLC_CGLS_CTRL);
        data |= RLC_CGLS_CTRL__LS_ENABLE;
        data &= ~RLC_CGLS_CTRL__LS_IDLE_THRESHOLD_MASK;
        data |= (0x10 << 16); /* Light Sleep空闲阈值 */
        WREG32_SOC15(GC, 0, regRLC_CGLS_CTRL, data);
    }

    /* 4. 使能FGCG (Fine-Grained Clock Gating) */
    if (adev->cg_flags & AMD_CG_SUPPORT_GFX_FGCG) {
        /* 每个CU独立配置 */
        for (int i = 0; i < adev->gfx.cu_count; i++) {
            data = RREG32_SOC15(GC, 0,
                                regCGTS_CU0_CTRL + i);
            data |= CGTS_CU_CTRL__CG_ENABLE;
            data |= CGTS_CU_CTRL__LS_ENABLE;
            data &= ~CGTS_CU_CTRL__IDLE_THRESHOLD_MASK;
            data |= (0x40 << 8); /* 64周期空闲阈值 */
            WREG32_SOC15(GC, 0,
                         regCGTS_CU0_CTRL + i, data);
        }
    }

    return 0;
}

/* 电源状态转换时的CG处理 */
static int gfx_v11_cg_ps_transition(struct amdgpu_device *adev,
                                     bool enable)
{
    uint32_t data;

    if (enable) {
        /* 进入电源状态: 使能所有CG */
        gfx_v11_init_cg(adev);
    } else {
        /* 退出电源状态: 禁用CG */
        /* 1. 禁用FGCG */
        for (int i = 0; i < adev->gfx.cu_count; i++) {
            data = RREG32_SOC15(GC, 0,
                                regCGTS_CU0_CTRL + i);
            data &= ~CGTS_CU_CTRL__CG_ENABLE;
            data &= ~CGTS_CU_CTRL__LS_ENABLE;
            WREG32_SOC15(GC, 0,
                         regCGTS_CU0_CTRL + i, data);
        }

        /* 2. 禁用CGCG */
        data = RREG32_SOC15(GC, 0, regRLC_CGCG_CTRL);
        data &= ~RLC_CGCG_CTRL__CG_ENABLE;
        WREG32_SOC15(GC, 0, regRLC_CGCG_CTRL, data);

        /* 3. 禁用MGCG */
        data = RREG32_SOC15(GC, 0, regRLC_MGCG_CTRL);
        data &= ~RLC_MGCG_CTRL__MGCG_ENABLE;
        WREG32_SOC15(GC, 0, regRLC_MGCG_CTRL, data);
    }

    return 0;
}
```

### 3.4 内存控制器Clock Gating

内存控制器的CG管理直接影响显存带宽和延迟：

```c
/* 内存控制器CG寄存器 */
#define regMC_CG_CTRL            0x2000
#define regMC_LS_CTRL            0x2001
#define regMC_SEQ_CG            0x2002

/* MC_CG_CTRL位域 */
#define MC_CG_CTRL__MCLK_CG_ENABLE       (1 << 0)
#define MC_CG_CTRL__PHY_CG_ENABLE        (1 << 1)
#define MC_CG_CTRL__DRAM_CG_ENABLE       (1 << 2)
#define MC_CG_CTRL__CTLR_CG_ENABLE       (1 << 3)
#define MC_CG_CTRL__IDLE_MASK            (0xF << 16)

/* 内存控制器CG初始化 */
static int mc_v11_init_cg(struct amdgpu_device *adev)
{
    uint32_t data;

    /* MCLK时钟门控 */
    if (adev->cg_flags & AMD_CG_SUPPORT_MC_MGCG) {
        data = RREG32_SOC15(MC, 0, regMC_CG_CTRL);
        
        /* 使能MCLK门控 */
        data |= MC_CG_CTRL__MCLK_CG_ENABLE;
        
        /* 使能PHY门控 */
        data |= MC_CG_CTRL__PHY_CG_ENABLE;
        
        /* 设置空闲检测掩码 */
        data &= ~MC_CG_CTRL__IDLE_MASK;
        data |= (0x8 << 16); /* 8周期空闲检测 */
        
        WREG32_SOC15(MC, 0, regMC_CG_CTRL, data);
    }

    /* 内存Light Sleep */
    if (adev->cg_flags & AMD_CG_SUPPORT_MC_LS) {
        data = RREG32_SOC15(MC, 0, regMC_LS_CTRL);
        data |= (1 << 0); /* LS使能 */
        data |= (0x20 << 8); /* 32周期进入LS */
        WREG32_SOC15(MC, 0, regMC_LS_CTRL, data);
    }

    /* 内存序列器CG */
    data = RREG32_SOC15(MC, 0, regMC_SEQ_CG);
    data |= (1 << 0) | (1 << 1) | (1 << 2);
    WREG32_SOC15(MC, 0, regMC_SEQ_CG, data);

    return 0;
}
```

### 3.5 多媒体引擎Clock Gating

VCN (Video Core Next) 引擎的CG管理：

```c
/* VCN CG寄存器 */
#define regVCN_CG_CTRL          0x3000
#define regUVD_CG_CTRL          0x3001
#define regJPEG_CG_CTRL         0x3002

#define VCN_CG_CTRL__DEC_CG_ENABLE      (1 << 0)
#define VCN_CG_CTRL__ENC_CG_ENABLE      (1 << 1)
#define VCN_CG_CTRL__DIV_CG_ENABLE      (1 << 2)
#define VCN_CG_CTRL__SYS_CG_ENABLE      (1 << 3)
#define VCN_CG_CTRL__IDLE_THRESHOLD     (0xFF << 16)

/* VCN Clock Gating初始化 */
static int vcn_v11_init_cg(struct amdgpu_device *adev)
{
    uint32_t data;

    if (!(adev->cg_flags & AMD_CG_SUPPORT_VCN_MGCG))
        return 0;

    /* VCN解码器CG */
    data = RREG32_SOC15(VCN, 0, regVCN_CG_CTRL);
    data |= VCN_CG_CTRL__DEC_CG_ENABLE;
    data |= VCN_CG_CTRL__ENC_CG_ENABLE;
    data |= VCN_CG_CTRL__SYS_CG_ENABLE;
    data &= ~VCN_CG_CTRL__IDLE_THRESHOLD_MASK;
    data |= (0x20 << 16);
    WREG32_SOC15(VCN, 0, regVCN_CG_CTRL, data);

    /* JPEG CG */
    data = RREG32_SOC15(VCN, 0, regJPEG_CG_CTRL);
    data |= (1 << 0) | (1 << 1) | (1 << 2);
    data |= (0x10 << 16);
    WREG32_SOC15(VCN, 0, regJPEG_CG_CTRL, data);

    return 0;
}

/* VCN CG电源状态转换 */
static int vcn_v11_cg_ps_transition(struct amdgpu_device *adev,
                                     bool enable)
{
    uint32_t data;

    if (enable) {
        /* 进入VCN电源状态 */
        vcn_v11_init_cg(adev);
    } else {
        /* 退出VCN电源状态 - 确保完全唤醒 */
        data = RREG32_SOC15(VCN, 0, regVCN_CG_CTRL);
        data &= ~(VCN_CG_CTRL__DEC_CG_ENABLE |
                  VCN_CG_CTRL__ENC_CG_ENABLE |
                  VCN_CG_CTRL__SYS_CG_ENABLE);
        WREG32_SOC15(VCN, 0, regVCN_CG_CTRL, data);
    }

    return 0;
}
```

### 3.6 SDMA引擎Clock Gating

```c
/* SDMA CG寄存器 */
#define regSDMA_CG_CTRL         0x4000

#define SDMA_CG_CTRL__INST_CG_ENABLE    (1 << 0)
#define SDMA_CG_CTRL__FIFO_CG_ENABLE    (1 << 1)
#define SDMA_CG_CTRL__SRBM_CG_ENABLE    (1 << 2)
#define SDMA_CG_CTRL__IDLE_MASK         (0xFF << 16)

/* SDMA CG初始化 */
static int sdma_v11_init_cg(struct amdgpu_device *adev)
{
    uint32_t data;

    if (!(adev->cg_flags & AMD_CG_SUPPORT_SDMA_MGCG))
        return 0;

    /* 为每个SDMA实例配置CG */
    for (int i = 0; i < adev->sdma.num_instances; i++) {
        data = RREG32_SOC15(SDMA, i, regSDMA_CG_CTRL);
        
        data |= SDMA_CG_CTRL__INST_CG_ENABLE;
        data |= SDMA_CG_CTRL__FIFO_CG_ENABLE;
        data |= SDMA_CG_CTRL__SRBM_CG_ENABLE;
        data &= ~SDMA_CG_CTRL__IDLE_MASK;
        
        /* SDMA0使用不同的空闲阈值 */
        data |= (i == 0) ? (0x20 << 16) : (0x40 << 16);
        
        WREG32_SOC15(SDMA, i, regSDMA_CG_CTRL, data);
    }

    return 0;
}

/* SDMA Light Sleep */
static int sdma_v11_init_ls(struct amdgpu_device *adev)
{
    uint32_t data;

    if (!(adev->cg_flags & AMD_CG_SUPPORT_SDMA_LS))
        return 0;

    for (int i = 0; i < adev->sdma.num_instances; i++) {
        data = RREG32_SOC15(SDMA, i, regSDMA_CG_CTRL);
        data |= (1 << 24); /* LS使能 */
        data |= (0x80 << 8); /* LS空闲阈值 */
        WREG32_SOC15(SDMA, i, regSDMA_CG_CTRL, data);
    }

    return 0;
}
```

### 3.7 HDP与IH引擎Clock Gating

```c
/* HDP (Host Data Path) CG */
#define regHDP_CG_CTRL          0x5000

#define HDP_CG_CTRL__HDP_IF_CG_ENABLE   (1 << 0)
#define HDP_CG_CTRL__HDP_MEM_CG_ENABLE  (1 << 1)
#define HDP_CG_CTRL__HDP_REG_CG_ENABLE  (1 << 2)

static int hdp_v11_init_cg(struct amdgpu_device *adev)
{
    uint32_t data;

    if (!(adev->cg_flags & AMD_CG_SUPPORT_HDP_MGCG))
        return 0;

    data = RREG32_SOC15(HDP, 0, regHDP_CG_CTRL);
    data |= HDP_CG_CTRL__HDP_IF_CG_ENABLE;
    data |= HDP_CG_CTRL__HDP_MEM_CG_ENABLE;
    data |= HDP_CG_CTRL__HDP_REG_CG_ENABLE;
    WREG32_SOC15(HDP, 0, regHDP_CG_CTRL, data);

    return 0;
}

/* IH (Interrupt Handler) CG */
#define regIH_CG_CTRL           0x6000

#define IH_CG_CTRL__IH_CG_ENABLE        (1 << 0)
#define IH_CG_CTRL__IH_LS_ENABLE        (1 << 1)
#define IH_CG_CTRL__IDLE_THRESHOLD      (0xFFF << 16)

static int ih_v11_init_cg(struct amdgpu_device *adev)
{
    uint32_t data;

    if (!(adev->cg_flags & AMD_CG_SUPPORT_IH_MGCG))
        return 0;

    data = RREG32_SOC15(IH, 0, regIH_CG_CTRL);
    data |= IH_CG_CTRL__IH_CG_ENABLE;
    data |= IH_CG_CTRL__IH_LS_ENABLE;
    data &= ~IH_CG_CTRL__IDLE_THRESHOLD_MASK;
    data |= (0x100 << 16); /* 256周期空闲 */
    WREG32_SOC15(IH, 0, regIH_CG_CTRL, data);

    return 0;
}
```

### 3.8 CG状态转换的完整流程

```c
/* 完整的CG电源状态转换 */
int amdgpu_cg_ps_transition(struct amdgpu_device *adev,
                            enum amdgpu_power_state old_state,
                            enum amdgpu_power_state new_state)
{
    int ret = 0;

    mutex_lock(&adev->cg_lock);

    /* 1. 通知RLC (RunList Controller) 状态变更 */
    gfx_v11_rlc_stop(adev);

    /* 2. 逐模块进行CG状态转换 */
    
    /* GFX引擎 */
    if (adev->cg_funcs->gfx_cg_ps_transition) {
        ret = adev->cg_funcs->gfx_cg_ps_transition(
            adev, new_state == POWER_STATE_ACTIVE);
        if (ret)
            goto done;
    }

    /* 内存控制器 */
    if (adev->cg_funcs->mc_cg_ps_transition) {
        ret = adev->cg_funcs->mc_cg_ps_transition(
            adev, new_state == POWER_STATE_ACTIVE);
        if (ret)
            goto done;
    }

    /* VCN引擎 */
    if (adev->cg_funcs->vcn_cg_ps_transition) {
        ret = adev->cg_funcs->vcn_cg_ps_transition(
            adev, new_state == POWER_STATE_ACTIVE);
        if (ret)
            goto done;
    }

    /* SDMA引擎 */
    if (adev->cg_funcs->sdma_cg_ps_transition) {
        ret = adev->cg_funcs->sdma_cg_ps_transition(
            adev, new_state == POWER_STATE_ACTIVE);
        if (ret)
            goto done;
    }

    /* HDP引擎 */
    if (adev->cg_funcs->hdp_cg_ps_transition) {
        ret = adev->cg_funcs->hdp_cg_ps_transition(
            adev, new_state == POWER_STATE_ACTIVE);
        if (ret)
            goto done;
    }

    /* 3. 重新启动RLC */
    gfx_v11_rlc_start(adev);

    /* 4. 更新电源状态标志 */
    adev->cg_state.power_state = new_state;

done:
    mutex_unlock(&adev->cg_lock);
    return ret;
}
```

## 4. Idle检测机制深度分析

### 4.1 基于计数器的Idle检测

这是最基本的Idle检测方法：

```c
/* 硬件Idle检测计数器 */
struct hw_idle_counter {
    /* 配置参数 */
    uint32_t threshold;        /* 门控触发阈值 */
    uint32_t hysteresis;       /* 迟滞值(防止抖动) */
    
    /* 运行时状态 */
    uint32_t idle_cycles;      /* 当前空闲周期计数 */
    uint32_t active_cycles;    /* 当前活动周期计数 */
    bool prev_idle;            /* 上一周期是否空闲 */
    bool current_idle;         /* 当前是否空闲 */
    
    /* 统计信息 */
    uint64_t total_idle_cycles;
    uint64_t total_active_cycles;
    uint32_t gate_count;       /* 门控触发次数 */
    uint32_t wakeup_count;     /* 唤醒次数 */
};

/* Idle状态机 */
enum idle_state {
    IDLE_STATE_ACTIVE,         /* 活跃运行 */
    IDLE_STATE_WAITING,        /* 等待确认空闲 */
    IDLE_STATE_IDLE,           /* 已确认空闲 */
    IDLE_STATE_GATING,         /* 正在门控 */
    IDLE_STATE_GATED,          /* 已门控 */
    IDLE_STATE_WAKING          /* 正在唤醒 */
};

/* Idle检测状态机实现 */
static enum idle_state idle_detection_fsm(
    struct hw_idle_counter *counter,
    bool has_activity)
{
    switch (counter->state) {
    case IDLE_STATE_ACTIVE:
        if (!has_activity) {
            counter->idle_cycles = 0;
            counter->state = IDLE_STATE_WAITING;
        }
        break;

    case IDLE_STATE_WAITING:
        if (has_activity) {
            counter->state = IDLE_STATE_ACTIVE;
        } else {
            counter->idle_cycles++;
            if (counter->idle_cycles >= counter->threshold) {
                counter->state = IDLE_STATE_IDLE;
            }
        }
        break;

    case IDLE_STATE_IDLE:
        counter->state = IDLE_STATE_GATING;
        break;

    case IDLE_STATE_GATING:
        /* 等待流水线排空 */
        if (is_pipeline_empty()) {
            counter->state = IDLE_STATE_GATED;
            counter->gate_count++;
        }
        break;

    case IDLE_STATE_GATED:
        if (has_activity) {
            counter->state = IDLE_STATE_WAKING;
        }
        break;

    case IDLE_STATE_WAKING:
        /* 等待时钟稳定 */
        if (is_clock_stable()) {
            counter->state = IDLE_STATE_ACTIVE;
            counter->wakeup_count++;
        }
        break;
    }

    return counter->state;
}
```

### 4.2 基于预测的Idle检测

利用历史模式预测未来的空闲状态：

```c
/* 预测式Idle检测 */
struct predictive_idle_detector {
    /* 历史模式记录 */
    uint8_t history_buffer[IDLE_HISTORY_SIZE];
    int history_index;
    
    /* 模式匹配 */
    uint8_t current_pattern;
    uint8_t predicted_pattern;
    int match_length;
    
    /* 统计模型 */
    float idle_probability;      /* 空闲概率 */
    float idle_duration_mean;    /* 平均空闲时长 */
    float idle_duration_variance;/* 空闲时长方差 */
    
    /* 决策参数 */
    float gate_threshold;        /* 门控概率阈值 */
    float min_gate_duration;     /* 最小有效门控时长 */
};

/* 模式更新 */
static void update_idle_pattern(
    struct predictive_idle_detector *pid,
    bool is_idle)
{
    /* 移位寄存器风格的模式记录 */
    pid->history_buffer[pid->history_index] = is_idle ? 1 : 0;
    pid->history_index = (pid->history_index + 1) %
                         IDLE_HISTORY_SIZE;

    /* 更新统计信息 */
    float alpha = 0.1f;
    pid->idle_probability = (1 - alpha) * pid->idle_probability +
                            alpha * (is_idle ? 1.0f : 0.0f);
}

/* 预测下一次空闲 */
static bool predict_next_idle(
    struct predictive_idle_detector *pid)
{
    /* 使用简单马尔可夫模型 */
    float trans_idle_to_idle = 0.8f;  /* P(idle|idle) */
    float trans_active_to_idle = 0.2f;/* P(idle|active) */

    bool last_was_idle =
        pid->history_buffer[(pid->history_index - 1) %
                            IDLE_HISTORY_SIZE];

    float pred_prob = last_was_idle ?
                      trans_idle_to_idle :
                      trans_active_to_idle;

    /* 结合历史平均 */
    pred_prob = 0.7f * pred_prob +
                0.3f * pid->idle_probability;

    return pred_prob > pid->gate_threshold;
}
```

### 4.3 基于事件的Idle检测

利用GPU内部事件信号进行更精确的Idle判断：

```c
/* 事件驱动Idle检测 */
struct event_driven_idle {
    /* 事件源 */
    struct {
        bool draw_call_active;      /* Draw Call进行中 */
        bool compute_task_active;   /* Compute任务进行中 */
        bool dma_transfer_active;   /* DMA传输进行中 */
        bool page_fault_active;     /* 缺页处理中 */
        bool fence_signal_pending;  /* Fence等待中 */
    } events;
    
    /* 依赖图 - 各模块间的依赖关系 */
    struct module_dependency {
        enum cg_module src;
        enum cg_module dst;
        bool active;
    } dependencies[MAX_DEPENDENCIES];
    
    /* 模块活跃传播 */
    uint32_t active_modules_bitmap;
};

/* 事件传播 - 当一个模块活跃时，依赖它的模块也不能门控 */
static void propagate_activity(
    struct event_driven_idle *edi,
    enum cg_module module,
    bool active)
{
    if (active) {
        edi->active_modules_bitmap |= (1 << module);
        
        /* 传播到所有依赖模块 */
        for (int i = 0; i < edi->num_dependencies; i++) {
            if (edi->dependencies[i].src == module) {
                edi->active_modules_bitmap |=
                    (1 << edi->dependencies[i].dst);
            }
        }
    } else {
        /* 不能直接清除，需要检查是否还有其它依赖源 */
        bool has_other_source = false;
        
        for (int i = 0; i < edi->num_dependencies; i++) {
            if (edi->dependencies[i].dst == module &&
                edi->active_modules_bitmap &
                (1 << edi->dependencies[i].src)) {
                has_other_source = true;
                break;
            }
        }
        
        if (!has_other_source) {
            edi->active_modules_bitmap &= ~(1 << module);
        }
    }
}
```

### 4.4 多级Idle检测策略

不同引擎需要不同的Idle检测策略：

| 引擎 | 检测方法 | 空闲阈值 | 特点 |
|------|---------|---------|------|
| GFX CU | 计数器+预测 | 64-256周期 | 需要快速响应Draw Call |
| GFX CP | 事件驱动 | 立即门控 | 命令流中断即门控 |
| MC | 计数器+排队 | 8-32周期 | 避免影响内存延迟 |
| SDMA | 事件驱动 | 32-128周期 | 配合DMA传输模式 |
| VCN | 帧边界检测 | 帧间空闲 | 视频帧边界自然空闲 |
| JPEG | 帧边界检测 | 帧间空闲 | 图片解码间隔 |
| HDP | 事件驱动 | 16-64周期 | PCIe传输中断即门控 |
| IH | 等待队列 | 256-1024周期 | 中断稀疏性高 |

```c
/* 多级Idle检测配置 */
struct idle_detection_config {
    enum cg_module module;
    enum idle_detect_method {
        IDLE_DETECT_COUNTER,       /* 计数器 */
        IDLE_DETECT_PREDICTIVE,    /* 预测 */
        IDLE_DETECT_EVENT_DRIVEN,  /* 事件驱动 */
        IDLE_DETECT_HYBRID        /* 混合 */
    } method;
    
    union {
        struct {
            uint32_t threshold;
            uint32_t hysteresis;
        } counter;
        
        struct {
            float gate_probability;
            int pattern_length;
        } predictive;
        
        struct {
            uint32_t event_mask;
            uint32_t quiesce_mask;
        } event_driven;
    } config;
    
    /* 最优策略选择 */
    int (*select_strategy)(void *context,
                           struct workload_stats *stats);
};

/* 动态策略选择 */
static int select_idle_strategy(
    struct idle_detection_config *cfg,
    struct workload_stats *stats)
{
    /* 根据工作负载特征选择最优检测方法 */
    
    /* 高频率切换负载 -> 预测方法 */
    if (stats->switch_frequency > 1000) {
        cfg->method = IDLE_DETECT_PREDICTIVE;
        cfg->config.predictive.gate_probability = 0.7f;
        return 0;
    }
    
    /* 持续计算负载 -> 计数器方法 */
    if (stats->compute_utilization > 0.8f) {
        cfg->method = IDLE_DETECT_COUNTER;
        cfg->config.counter.threshold = 256;
        cfg->config.counter.hysteresis = 32;
        return 0;
    }
    
    /* IO密集负载 -> 事件驱动 */
    if (stats->io_wait_ratio > 0.3f) {
        cfg->method = IDLE_DETECT_EVENT_DRIVEN;
        return 0;
    }
    
    /* 默认使用混合方法 */
    cfg->method = IDLE_DETECT_HYBRID;
    return 0;
}
```

## 5. Fine-Grained Clock Gating (FGCG)

### 5.1 FGCG的基本原理

FGCG是RDNA/RDNA2/RDNA3引入的高级Clock Gating技术，比传统MGCG具有更细的粒度：

```
传统MGCG:
┌─────────────────────────────────────┐
│          GFX Engine                 │
│  ┌──────┐ ┌──────┐ ┌──────┐       │
│  │ CU0  │ │ CU1  │ │ CU2  │       │
│  └──────┘ └──────┘ └──────┘       │
│  ┌──────┐ ┌──────┐ ┌──────┐       │
│  │  TA   │ │  TD  │ │  SC  │       │
│  └──────┘ └──────┘ └──────┘       │
│        一个门控信号控制整个引擎        │
└─────────────────────────────────────┘

FGCG:
┌─────────────────────────────────────┐
│          GFX Engine                 │
│  ┌──────┐ ┌──────┐ ┌──────┐       │
│  │CU0 CG│ │CU1 CG│ │CU2 CG│       │  ← 每CU独立门控
│  └──────┘ └──────┘ └──────┘       │
│  ┌──────┐ ┌──────┐ ┌──────┐       │
│  │TA CG │ │TD CG │ │SC CG │       │  ← 每功能块独立门控
│  └──────┘ └──────┘ └──────┘       │
│  ┌──────┐ ┌──────┐                 │
│  │L1 CG │ │L1 CG │                 │  ← L1 Cache按bank门控
│  └──────┘ └──────┘                 │
│          多个独立门控域             │
└─────────────────────────────────────┘
```

### 5.2 FGCG的硬件实现

```verilog
// FGCG控制单元
module fgcg_controller #(
    parameter NUM_CU = 12,
    parameter NUM_TA = 4,
    parameter NUM_TD = 4,
    parameter NUM_BANKS = 16
) (
    input  wire clk,
    input  wire rst_n,
    
    // 每个CU的活动信号
    input  wire [NUM_CU-1:0] cu_active,
    // 每个TA的活动信号
    input  wire [NUM_TA-1:0] ta_active,
    // 每个TD的活动信号
    input  wire [NUM_TD-1:0] td_active,
    // L2 cache bank活动
    input  wire [NUM_BANKS-1:0] bank_active,
    
    // 门控使能
    output wire [NUM_CU-1:0] cu_clk_en,
    output wire [NUM_TA-1:0] ta_clk_en,
    output wire [NUM_TD-1:0] td_clk_en,
    output wire [NUM_BANKS-1:0] bank_clk_en,
    
    // 配置接口
    input  wire [7:0] config_idle_threshold,
    input  wire [2:0] config_wakeup_delay,
    
    // 状态输出
    output wire [31:0] status_gate_count,
    output wire [31:0] status_wakeup_count
);

    // 每CU独立门控逻辑
    genvar i;
    generate
        for (i = 0; i < NUM_CU; i++) begin : cu_gate_gen
            cu_gate_control u_cu_gate (
                .clk(clk),
                .rst_n(rst_n),
                .active(cu_active[i]),
                .idle_threshold(config_idle_threshold),
                .wakeup_delay(config_wakeup_delay),
                .clk_en(cu_clk_en[i]),
                .gate_count(),
                .wakeup_count()
            );
        end
    endgenerate
    
    // TA/TD类似...
    
endmodule

// 单个CU的门控控制
module cu_gate_control (
    input  wire clk,
    input  wire rst_n,
    input  wire active,
    input  wire [7:0] idle_threshold,
    input  wire [2:0] wakeup_delay,
    output reg clk_en,
    output reg [31:0] gate_count,
    output reg [31:0] wakeup_count
);
    
    reg [7:0] idle_counter;
    reg [2:0] wakeup_counter;
    reg gated;
    
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            clk_en <= 1'b1;
            gated <= 1'b0;
            idle_counter <= 0;
            gate_count <= 0;
            wakeup_count <= 0;
        end else begin
            if (gated) begin
                // 已门控状态
                clk_en <= 1'b0;
                
                if (active) begin
                    // 需要唤醒
                    gated <= 1'b0;
                    wakeup_counter <= wakeup_delay;
                end
            end else if (wakeup_counter > 0) begin
                // 唤醒中
                wakeup_counter <= wakeup_counter - 1;
                if (wakeup_counter == 1) begin
                    clk_en <= 1'b1;
                    wakeup_count <= wakeup_count + 1;
                end
            end else begin
                // 活动状态
                clk_en <= 1'b1;
                
                if (!active) begin
                    if (idle_counter < idle_threshold) begin
                        idle_counter <= idle_counter + 1;
                    end else begin
                        gated <= 1'b1;
                        gate_count <= gate_count + 1;
                    end
                end else begin
                    idle_counter <= 0;
                end
            end
        end
    end
    
endmodule
```

### 5.3 FGCG在驱动中的配置

```c
/* FGCG配置结构 */
struct fgcg_config {
    /* 每CU的独立配置 */
    struct {
        bool enable;
        uint32_t idle_threshold;
        uint32_t wakeup_delay;
        bool ls_enable;         /* Light Sleep */
    } cu_config[MAX_CU_COUNT];
    
    /* TA/TD配置 */
    struct {
        bool enable;
        uint32_t idle_threshold;
    } ta_config[MAX_TA_COUNT];
    
    /* L1 Cache Bank配置 */
    struct {
        bool enable;
        uint32_t idle_threshold;
    } bank_config[MAX_BANK_COUNT];
    
    /* 全局FGCG控制 */
    bool global_enable;
    uint32_t master_idle_threshold;
    uint32_t debounce_cycles;
};

/* FGCG初始化 */
static int gfx_v11_fgcg_init(struct amdgpu_device *adev,
                              struct fgcg_config *config)
{
    uint32_t data;
    int ret = 0;

    /* 1. 使能FGCG全局控制 */
    data = RREG32_SOC15(GC, 0, regRLC_MGCG_CTRL);
    data |= RLC_MGCG_CTRL__FGCG_ENABLE;
    WREG32_SOC15(GC, 0, regRLC_MGCG_CTRL, data);

    /* 2. 配置每CU FGCG参数 */
    for (int i = 0; i < adev->gfx.cu_count; i++) {
        if (!config->cu_config[i].enable)
            continue;

        data = RREG32_SOC15(GC, 0,
                            regCGTS_CU0_CTRL + i);
        
        data &= ~CGTS_CU_CTRL__CG_ENABLE;
        data &= ~CGTS_CU_CTRL__LS_ENABLE;
        data &= ~CGTS_CU_CTRL__IDLE_THRESHOLD_MASK;
        data &= ~CGTS_CU_CTRL__WAKEUP_DELAY_MASK;
        
        if (config->cu_config[i].enable)
            data |= CGTS_CU_CTRL__CG_ENABLE;
        
        if (config->cu_config[i].ls_enable)
            data |= CGTS_CU_CTRL__LS_ENABLE;
        
        data |= (config->cu_config[i].idle_threshold << 8);
        data |= (config->cu_config[i].wakeup_delay << 16);
        
        WREG32_SOC15(GC, 0, regCGTS_CU0_CTRL + i, data);
    }

    /* 3. 配置TA/TD FGCG */
    for (int i = 0; i < adev->gfx.ta_count; i++) {
        if (!config->ta_config[i].enable)
            continue;
        
        /* TA/TD CG寄存器配置 */
        data = RREG32_SOC15(GC, 0,
                            regCGTS_TA0_CTRL + i);
        data |= (1 << 0); /* TA/TD CG使能 */
        data |= (config->ta_config[i].idle_threshold << 8);
        WREG32_SOC15(GC, 0,
                     regCGTS_TA0_CTRL + i, data);
    }

    /* 4. 配置L1 Cache Bank FGCG */
    for (int i = 0; i < adev->gfx.l1_bank_count; i++) {
        if (!config->bank_config[i].enable)
            continue;
        
        data = RREG32_SOC15(GC, 0,
                            regCGTS_GL1_CTRL + i);
        data |= (1 << 0);
        data |= (config->bank_config[i].idle_threshold << 8);
        WREG32_SOC15(GC, 0,
                     regCGTS_GL1_CTRL + i, data);
    }

    return 0;
}

/* 动态FGCG调优 */
static int fgcg_tune_parameters(
    struct amdgpu_device *adev,
    struct workload_stats *stats)
{
    struct fgcg_config *config = &adev->fgcg_config;
    
    /* 根据工作负载密度调整空闲阈值 */
    if (stats->warp_occupancy > 0.8f) {
        /* 高占用率: 提高阈值减少误门控 */
        for (int i = 0; i < adev->gfx.cu_count; i++) {
            config->cu_config[i].idle_threshold = 256;
        }
    } else if (stats->warp_occupancy < 0.3f) {
        /* 低占用率: 降低阈值快速节能 */
        for (int i = 0; i < adev->gfx.cu_count; i++) {
            config->cu_config[i].idle_threshold = 32;
        }
    }
    
    /* 根据Warp调度效率调整唤醒延迟 */
    if (stats->wavefront_efficiency > 0.9f) {
        for (int i = 0; i < adev->gfx.cu_count; i++) {
            config->cu_config[i].wakeup_delay = 1;
        }
    }
    
    /* 应用配置 */
    return gfx_v11_fgcg_init(adev, config);
}
```

### 5.4 FGCG与工作负载的交互

```c
/* FGCG对性能影响的建模 */
struct fgcg_perf_model {
    /* 门控开销 */
    float gate_latency_ns;      /* 门控延迟(ns) */
    float wakeup_latency_ns;    /* 唤醒延迟(ns) */
    float gate_overhead_energy; /* 门控能量开销(pJ) */
    
    /* 节能收益 */
    float power_save_per_cycle;  /* 每周期节能(pW) */
    float avg_gate_duration;     /* 平均门控时长(周期) */
    
    /* 净收益计算 */
    float break_even_cycles;     /* 盈亏平衡周期数 */
};

/* 计算FGCG的盈亏平衡点 */
static float calculate_fgcg_breakeven(
    struct fgcg_perf_model *model)
{
    /* 门控的总开销 = 门控延迟 + 唤醒延迟 */
    float total_overhead_ns =
        model->gate_latency_ns +
        model->wakeup_latency_ns;
    
    /* 门控的能量开销 */
    float overhead_energy =
        model->gate_overhead_energy;
    
    /* 每纳秒节能 */
    float save_per_ns =
        model->power_save_per_cycle /
        model->avg_gate_duration;
    
    /* 盈亏平衡周期数 */
    model->break_even_cycles =
        overhead_energy / model->power_save_per_cycle +
        total_overhead_ns / save_per_ns;
    
    return model->break_even_cycles;
}

/* 自适应FGCG使能决策 */
static bool should_enable_fgcg(
    struct amdgpu_device *adev,
    struct workload_stats *stats)
{
    struct fgcg_perf_model model;
    
    /* 获取当前工作负载特征 */
    float avg_idle_length = stats->avg_idle_cycles;
    float idle_frequency = stats->idle_frequency;
    float perf_sensitivity = stats->perf_sensitivity;
    
    /* 计算盈亏平衡点 */
    float breakeven = calculate_fgcg_breakeven(&model);
    
    /* 决策逻辑 */
    if (avg_idle_length < breakeven) {
        /* 空闲太短，FGCG得不偿失 */
        return false;
    }
    
    if (idle_frequency > 10000 && perf_sensitivity > 0.9f) {
        /* 频繁空闲+性能敏感 -> 关闭FGCG避免抖动 */
        return false;
    }
    
    if (avg_idle_length > breakeven * 10) {
        /* 空闲很长 -> FGCG收益大 */
        return true;
    }
    
    /* 默认使能 */
    return adev->fgcg_config.global_enable;
}
```

## 6. Clock Gating与Power Gating的协同

### 6.1 CG-PG层次结构

Clock Gating和Power Gating组成多层次节能体系：

```
节能层次                    功耗    恢复时间    控制粒度
                               ↓        ↓          ↓
Level 0: 全速运行              100%     -          -
Level 1: 寄存器CG             90%     1周期       寄存器级
Level 2: 功能块CG             75%     2-5周期     功能单元级
Level 3: 引擎级CG             60%     10-100周期   IP模块级
Level 4: FGCG                 50%     1-10周期     CU/TA级
Level 5: 内存LS (Light Sleep) 40%     0.5-1μs     PHY级
Level 6: Power Gating         10%     10-100μs    IP模块级
Level 7: BACO                 5%      1-10ms      GPU芯片级
```

### 6.2 状态转换策略

```c
/* CG-PG协同状态转换 */
struct cg_pg_coordinator {
    /* 当前层次 */
    int current_level;
    int target_level;
    
    /* 状态转换时间戳 */
    ktime_t last_level_up;
    ktime_t last_level_down;
    
    /* 驻留时间统计 */
    uint64_t level_residency[8];
    ktime_t level_entry_time[8];
    
    /* 转换策略 */
    struct {
        int up_threshold_ms;    /* 升级到PG的时间阈值 */
        int down_threshold_us;  /* 降级到CG的时间阈值 */
        bool prefer_deep_sleep; /* 偏好深度睡眠 */
    } strategy;
};

/* 层次转换决策 */
static int decide_cg_pg_transition(
    struct cg_pg_coordinator *coord,
    struct workload_stats *stats)
{
    ktime_t now = ktime_get();
    int suggested_level = coord->current_level;

    /* 如果空闲时间超过升级阈值 */
    if (stats->continuous_idle_ms >
        coord->strategy.up_threshold_ms) {
        
        if (coord->current_level < 7) {
            suggested_level = coord->current_level + 1;
        }
        
        /* 如果偏好深度睡眠，直接跳到PG层次 */
        if (coord->strategy.prefer_deep_sleep &&
            suggested_level < 6 &&
            stats->continuous_idle_ms >
            coord->strategy.up_threshold_ms * 10) {
            suggested_level = 6;
        }
    }

    /* 如果有新活动 */
    if (stats->has_new_activity) {
        /* 从PG唤醒到CG */
        if (coord->current_level >= 6) {
            suggested_level = 3;
        }
        
        /* 从深度CG唤醒到轻度CG */
        if (coord->current_level >= 4) {
            suggested_level = 2;
        }
    }

    /* 记录驻留时间 */
    if (suggested_level != coord->current_level) {
        ktime_t entry = coord->level_entry_time[
                        coord->current_level];
        coord->level_residency[coord->current_level] +=
            ktime_to_us(ktime_sub(now, entry));
        
        coord->level_entry_time[suggested_level] = now;
        coord->current_level = suggested_level;
    }

    return suggested_level;
}
```

### 6.3 协同实现代码

```c
/* CG-PG协同管理 */
static int cg_pg_cooperative_control(
    struct amdgpu_device *adev,
    int target_level)
{
    int ret = 0;

    mutex_lock(&adev->cg_pg_lock);

    switch (target_level) {
    case 0: /* 全速运行 - 禁用所有CG和PG */
        amdgpu_cg_disable_all(adev);
        amdgpu_pg_disable_all(adev);
        break;

    case 1: /* 仅寄存器CG */
        amdgpu_cg_enable_reg(adev);
        amdgpu_pg_disable_all(adev);
        break;

    case 2: /* 功能块CG */
        amdgpu_cg_enable_func_block(adev);
        amdgpu_pg_disable_all(adev);
        break;

    case 3: /* 引擎级CG + FGCG */
        amdgpu_cg_enable_all(adev);
        amdgpu_fgcg_enable(adev);
        amdgpu_pg_disable_all(adev);
        break;

    case 4: /* 增强CG */
        amdgpu_cg_enable_all(adev);
        amdgpu_fgcg_enable(adev);
        amdgpu_cg_enable_light_sleep(adev);
        amdgpu_pg_disable_all(adev);
        break;

    case 5: /* 内存LS */
        amdgpu_cg_enable_all(adev);
        amdgpu_fgcg_enable(adev);
        amdgpu_memory_ls_enable(adev);
        amdgpu_pg_disable_all(adev);
        break;

    case 6: /* Power Gating */
        amdgpu_cg_enable_all(adev);
        amdgpu_fgcg_enable(adev);
        amdgpu_pg_enable_selected(adev);
        break;

    case 7: /* BACO - 芯片级电源门控 */
        amdgpu_cg_enable_all(adev);
        amdgpu_pg_enable_all(adev);
        amdgpu_baco_entry(adev);
        break;

    default:
        ret = -EINVAL;
        break;
    }

    if (ret == 0)
        adev->cg_pg_level = target_level;

    mutex_unlock(&adev->cg_pg_lock);
    return ret;
}
```

## 7. sysfs接口与调试

### 7.1 CG状态查询接口

```c
/* sysfs CG状态显示 */
static ssize_t cg_state_show(struct device *dev,
                              struct device_attribute *attr,
                              char *buf)
{
    struct amdgpu_device *adev = dev_get_drvdata(dev);
    int offset = 0;

    offset += scnprintf(buf + offset, PAGE_SIZE - offset,
                "=== Clock Gating Status ===\n\n");

    /* GFX CG状态 */
    offset += scnprintf(buf + offset, PAGE_SIZE - offset,
                "GFX Engine:\n");
    offset += scnprintf(buf + offset, PAGE_SIZE - offset,
                "  MGCG: %s\n",
                adev->cg_flags & AMD_CG_SUPPORT_GFX_MGCG ?
                "enabled" : "disabled");
    offset += scnprintf(buf + offset, PAGE_SIZE - offset,
                "  CGCG: %s\n",
                adev->cg_flags & AMD_CG_SUPPORT_GFX_CGCG ?
                "enabled" : "disabled");
    offset += scnprintf(buf + offset, PAGE_SIZE - offset,
                "  CGLS: %s\n",
                adev->cg_flags & AMD_CG_SUPPORT_GFX_CGLS ?
                "enabled" : "disabled");
    offset += scnprintf(buf + offset, PAGE_SIZE - offset,
                "  FGCG: %s\n",
                adev->cg_flags & AMD_CG_SUPPORT_GFX_FGCG ?
                "enabled" : "disabled");

    /* 内存控制器CG */
    offset += scnprintf(buf + offset, PAGE_SIZE - offset,
                "Memory Controller:\n");
    offset += scnprintf(buf + offset, PAGE_SIZE - offset,
                "  MCLK CG: %s\n",
                adev->cg_flags & AMD_CG_SUPPORT_MC_MGCG ?
                "enabled" : "disabled");
    offset += scnprintf(buf + offset, PAGE_SIZE - offset,
                "  Light Sleep: %s\n",
                adev->cg_flags & AMD_CG_SUPPORT_MC_LS ?
                "enabled" : "disabled");

    /* VCN引擎CG */
    offset += scnprintf(buf + offset, PAGE_SIZE - offset,
                "VCN Engine:\n");
    offset += scnprintf(buf + offset, PAGE_SIZE - offset,
                "  Decoder CG: %s\n",
                adev->cg_flags & AMD_CG_SUPPORT_VCN_MGCG ?
                "enabled" : "disabled");

    /* SDMA引擎CG */
    offset += scnprintf(buf + offset, PAGE_SIZE - offset,
                "SDMA Engine:\n");
    offset += scnprintf(buf + offset, PAGE_SIZE - offset,
                "  SDMA0 CG: %s\n",
                adev->cg_flags & AMD_CG_SUPPORT_SDMA_MGCG ?
                "enabled" : "disabled");

    /* CG-PG协同层次 */
    offset += scnprintf(buf + offset, PAGE_SIZE - offset,
                "\nCG-PG Level: %d\n",
                adev->cg_pg_level);

    return offset;
}

/* sysfs CG控制接口 */
static ssize_t cg_control_store(struct device *dev,
                                 struct device_attribute *attr,
                                 const char *buf, size_t count)
{
    struct amdgpu_device *adev = dev_get_drvdata(dev);
    char cmd[32];
    uint32_t mask;
    bool enable;

    if (sscanf(buf, "%31s %x %d", cmd, &mask, &enable) != 3)
        return -EINVAL;

    if (strcmp(cmd, "cg") == 0) {
        if (enable)
            adev->cg_flags |= mask;
        else
            adev->cg_flags &= ~mask;
        
        /* 重新初始化CG */
        amdgpu_cg_init(adev);
    }

    return count;
}

static DEVICE_ATTR(cg_state, 0444, cg_state_show, NULL);
static DEVICE_ATTR(cg_control, 0200, NULL, cg_control_store);
```

### 7.2 调试寄存器转储

```c
/* CG调试寄存器转储 */
static void dump_cg_registers(struct amdgpu_device *adev,
                               struct seq_file *m)
{
    uint32_t data;

    seq_printf(m, "=== Clock Gating Registers ===\n\n");

    /* RLC CG控制寄存器 */
    data = RREG32_SOC15(GC, 0, regRLC_CGCG_CTRL);
    seq_printf(m, "RLC_CGCG_CTRL (0x%05X): 0x%08X\n",
               regRLC_CGCG_CTRL, data);

    /* RLC CGLS控制寄存器 */
    data = RREG32_SOC15(GC, 0, regRLC_CGLS_CTRL);
    seq_printf(m, "RLC_CGLS_CTRL (0x%05X): 0x%08X\n",
               regRLC_CGLS_CTRL, data);

    /* RLC MGCG控制寄存器 */
    data = RREG32_SOC15(GC, 0, regRLC_MGCG_CTRL);
    seq_printf(m, "RLC_MGCG_CTRL (0x%05X): 0x%08X\n",
               regRLC_MGCG_CTRL, data);

    /* 内存控制器CG寄存器 */
    data = RREG32_SOC15(GC, 0, regMC_CG_CTRL);
    seq_printf(m, "MC_CG_CTRL    (0x%05X): 0x%08X\n",
               regMC_CG_CTRL, data);

    /* VCN CG寄存器 */
    data = RREG32_SOC15(VCN, 0, regVCN_CG_CTRL);
    seq_printf(m, "VCN_CG_CTRL   (0x%05X): 0x%08X\n",
               regVCN_CG_CTRL, data);

    /* SDMA CG寄存器 */
    data = RREG32_SOC15(SDMA0, 0, regSDMA_CG_CTRL);
    seq_printf(m, "SDMA_CG_CTRL  (0x%05X): 0x%08X\n",
               regSDMA_CG_CTRL, data);

    /* HDP CG寄存器 */
    data = RREG32_SOC15(HDP, 0, regHDP_HOST_PATH_CNTL);
    seq_printf(m, "HDP_CG_CTRL   (0x%05X): 0x%08X\n",
               regHDP_HOST_PATH_CNTL, data);

    seq_printf(m, "\n=== CG Residency Stats ===\n");
    seq_printf(m, "CGCG residency:    %llu ns\n",
               amdgpu_cg_get_residency(adev, AMD_CG_BLOCK_GFX_CGCG));
    seq_printf(m, "CGLS residency:    %llu ns\n",
               amdgpu_cg_get_residency(adev, AMD_CG_BLOCK_GFX_CGLS));
    seq_printf(m, "FGCG residency:    %llu ns\n",
               amdgpu_cg_get_residency(adev, AMD_CG_BLOCK_GFX_FGCG));
    seq_printf(m, "MC_CG residency:   %llu ns\n",
               amdgpu_cg_get_residency(adev, AMD_CG_BLOCK_MC));
    seq_printf(m, "VCN_CG residency:  %llu ns\n",
               amdgpu_cg_get_residency(adev, AMD_CG_BLOCK_VCN));

    /* 唤醒统计 */
    seq_printf(m, "\n=== Wakeup Events ===\n");
    seq_printf(m, "Total wakeups:     %llu\n",
               atomic64_read(&adev->cg_stats.wakeup_count));
    seq_printf(m, "Wakeup latency:    %lu ns (avg)\n",
               adev->cg_stats.avg_wakeup_latency);
    seq_printf(m, "False wakeups:     %llu\n",
               atomic64_read(&adev->cg_stats.false_wakeup_count));
}
