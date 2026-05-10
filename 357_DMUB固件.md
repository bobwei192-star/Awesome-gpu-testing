# Day 357: DMUB（Display Manager Microcontroller Unit B）固件

## 1. DMUB 概述

DMUB (Display Manager Microcontroller Unit B) 是 AMDGPU DC 子系统中的一个关键组件，它是一个集成在 GPU 显示引擎中的微控制器，专门用于处理显示相关的实时任务。DMUB 的引入使得许多需要精确时序控制的操作可以从主 CPU 卸载到专用的微控制器上执行，从而降低延迟、减少 CPU 开销，并提高电源效率。

### 1.1 DMUB 的核心职责

```c
// DMUB 主要功能领域
enum dmub_function_area {
    DMUB_FUNC_PSR,             // Panel Self Refresh 控制
    DMUB_FUNC_ABM,             // Adaptive Backlight Modulation
    DMUB_FUNC_HPD,             // 热插拔检测 (DPIA)
    DMUB_FUNC_TIMING,          // 时序控制
    DMUB_FUNC_POWER,           // 电源状态管理
    DMUB_FUNC_PLANE_UPDATE,    // 平面更新
    DMUB_FUNC_VRR,             // Variable Refresh Rate
    DMUB_FUNC_MST,             // Multi-Stream Transport
    DMUB_FUNC_TRACE,           // 调试追踪
    DMUB_FUNC_MAX
};

// DMUB 存在的意义：卸载 CPU 负担
struct dmub_offload_benefits {
    // 实时性：微控制器响应时间 < 1us，远快于主 CPU (通常 > 10us)
    uint32_t latency_us;

    // CPU 节省：每个功能卸载减少的主 CPU 占用
    uint32_t cpu_cycles_saved_per_frame;

    // 功耗降低：DMCUB 在低功耗模式下仅需微瓦级功率
    uint32_t power_saved_mw;

    // 精确度：硬件定时器精度达到纳秒级
    uint32_t timer_precision_ns;
};
```

### 1.2 DMUB 硬件架构概览

```
+---------------------------------------------------------------+
|                        GPU Die                                  |
|                                                                |
|  +------------------+    +----------------------------------+   |
|  |  主 CPU (Host)   |    |  显示引擎 (Display Core)          |   |
|  |                  |    |                                  |   |
|  |  amdgpu_dm.ko   |<-->|  +--------+  +----------------+  |   |
|  |  (Linux 驱动)    |    |  | DCN    |  | DMCUB (ARC EM) |  |   |
|  |                  |    |  | Regs   |  |                |  |   |
|  |  dmub_srv APIs   |<-->|  +--------+  | - PSR         |  |   |
|  |                  |    |              | - ABM         |  |   |
|  +------------------+    |              | - HPD         |  |   |
|                          |              | - Timing      |  |   |
|  +------------------+    |              +----------------+  |   |
|  | System Memory     |    |                                  |   |
|  |                  |    |  +----------------------------+   |   |
|  | dmub_fw.bin     |<---+->| Ring Buffer (DRAM/SRAM)     |   |   |
|  | (固件二进制)     |    |  | - flush_queue (Host->DMCUB) |   |   |
|  |                  |    |  | - notify_queue (DMCUB->Host)|   |   |
|  +------------------+    |  +----------------------------+   |   |
|                          +----------------------------------+   |
+---------------------------------------------------------------+
```

### 1.3 DMUB 使用的处理器架构

```c
// DMCUB 处理器信息
struct dmub_processor_info {
    // 处理器架构：ARC EM (Synopsys DesignWare ARC EM)
    const char *architecture;

    // 指令集：ARCompact (16/32位混合指令)
    uint32_t instruction_set;

    // 时钟频率：典型值 200-400 MHz
    uint32_t clock_freq_mhz;

    // 本地内存：通常 64-128 KB SRAM
    uint32_t local_memory_bytes;

    // 固件大小限制：受本地内存限制
    uint32_t max_firmware_size_bytes;

    // 支持的中断数量
    uint32_t interrupt_count;

    // 通用寄存器数量
    uint32_t general_purpose_regs;
};
```

## 2. dmub_srv 结构体详解

`dmub_srv` 是驱动与 DMCUB 通信的核心结构体，它封装了固件管理、队列控制、状态跟踪等功能。

### 2.1 dmub_srv 完整定义

```c
// DMUB 服务结构体 - 驱动与 DMCUB 通信的核心
struct dmub_srv {
    // 固件相关信息
    const struct dmub_fw_info *fw;          // 固件元数据
    uint32_t fw_version;                     // 固件版本号
    size_t fw_size;                          // 固件大小
    const uint8_t *fw_data;                  // 固件二进制数据指针

    // 内存区域配置
    struct dmub_region fb_base;             // 帧缓冲基地址 (GPU 地址空间)
    struct dmub_region fb_offset;           // 帧缓冲偏移量

    // 状态管理
    enum dmub_status status;                // DMUB 当前状态
    bool hw_init;                            // 硬件初始化标志
    bool sw_init;                            // 软件初始化标志
    bool feature_active[DMUB_FUNC_MAX];     // 各功能激活状态

    // 环形缓冲区 (Ring Buffer)
    struct dmub_rb flush_queue;             // 主机 -> DMCUB 队列
    struct dmub_rb notify_queue;            // DMCUB -> 主机队列

    // ABM 缓存
    struct dmub_abm_cache abm_cache;

    // 注册的回调函数
    struct dmub_srv_callbacks {
        void (*dmcub_hw_int)(struct dmub_srv *srv);
        bool (*is_gpu_idle)(struct dmub_srv *srv);
        void (*process_pending)(struct dmub_srv *srv);
    } callbacks;

    // 调试和追踪
    struct dmub_trace_buf trace_buf;
    bool trace_enabled;

    // 内部状态
    void *priv;                              // 驱动私有数据
    struct device *dev;                      // 内核设备结构体
};
```

### 2.2 dmub_srv 初始化和启动流程

```c
// DMUB 初始化阶段枚举
enum dmub_init_phase {
    DMUB_INIT_PHASE_ALLOC,      // 分配内存资源
    DMUB_INIT_PHASE_HW_INIT,    // 硬件初始化 (寄存器配置)
    DMUB_INIT_PHASE_FW_LOAD,    // 固件加载到 SRAM
    DMUB_INIT_PHASE_FW_START,   // 启动 DMCUB 执行固件
    DMUB_INIT_PHASE_READY,      // DMUB 就绪，可以接收命令
    DMUB_INIT_PHASE_COMPLETE,   // 初始化完成
};

// 初始化流程函数
int dmub_srv_init(struct dmub_srv *srv)
{
    int ret;

    // Phase 1: 内存分配
    ret = dmub_allocate_memory(srv);
    if (ret)
        return ret;

    // Phase 2: 硬件初始化
    ret = dmub_hw_init(srv);
    if (ret)
        goto error_hw;

    // Phase 3: 加载固件到 SRAM
    ret = dmub_load_firmware(srv);
    if (ret)
        goto error_fw;

    // Phase 4: 启动 DMCUB
    ret = dmub_start_firmware(srv);
    if (ret)
        goto error_start;

    // Phase 5: 等待 DMUB 就绪
    ret = dmub_wait_ready(srv, 1000);  // 超时 1 秒
    if (ret)
        goto error_ready;

    // Phase 6: 初始化完成
    srv->status = DMUB_STATUS_OK;
    return 0;

error_ready:
    dmub_stop_firmware(srv);
error_start:
    dmub_unload_firmware(srv);
error_fw:
    dmub_hw_fini(srv);
error_hw:
    dmub_free_memory(srv);
    return ret;
}

// 等待 DMUB 就绪
int dmub_wait_ready(struct dmub_srv *srv, uint32_t timeout_ms)
{
    uint32_t wait_us = timeout_ms * 1000;
    uint32_t sleep_us = 10;

    while (wait_us > 0) {
        // 检查 DMCUB 的状态寄存器
        if (dmub_is_ready(srv))
            return 0;

        udelay(sleep_us);
        wait_us -= sleep_us;
    }

    DRM_ERROR("DMUB failed to become ready within %d ms\n",
              timeout_ms);
    return -ETIMEDOUT;
}

// 检查 DMUB 是否就绪
bool dmub_is_ready(struct dmub_srv *srv)
{
    uint32_t status_reg;

    // 读取 DMCUB 状态寄存器
    status_reg = dmub_read_reg(srv, DMUB_REG_STATUS);

    // 检查就绪标志位
    return (status_reg & DMUB_STATUS_READY_BIT) != 0;
}
```

### 2.3 固件加载与版本管理

```c
// 固件版本信息
struct dmub_fw_info {
    uint32_t version_major;      // 主版本号
    uint32_t version_minor;      // 次版本号
    uint32_t version_revision;   // 修订号
    uint32_t version_build;      // 构建号
    char build_date[16];         // 构建日期
    char build_time[16];         // 构建时间

    // 固件支持的硬件版本
    uint32_t hw_version_min;     // 最小支持硬件版本
    uint32_t hw_version_max;     // 最大支持硬件版本
};

// 固件加载过程
int dmub_load_firmware(struct dmub_srv *srv)
{
    const struct firmware *fw = NULL;
    struct dmub_fw_meta *meta;
    int ret;

    // 1. 使用 request_firmware 从文件系统加载固件
    ret = request_firmware(&fw, "amdgpu/dmub.bin", srv->dev);
    if (ret) {
        DRM_ERROR("Failed to load DMUB firmware\n");
        return ret;
    }

    // 2. 解析固件头部，验证元数据
    meta = (struct dmub_fw_meta *)fw->data;
    if (meta->magic != DMUB_FW_MAGIC) {
        DRM_ERROR("Invalid DMUB firmware magic\n");
        ret = -EINVAL;
        goto release_fw;
    }

    // 3. 检查固件版本与硬件兼容性
    if (!dmub_check_hw_compatibility(srv, meta)) {
        DRM_ERROR("DMUB firmware incompatible with HW\n");
        ret = -EINVAL;
        goto release_fw;
    }

    // 4. 保存固件信息
    srv->fw_version = meta->version;
    srv->fw_size = fw->size;
    srv->fw_data = fw->data;

    // 5. 将固件复制到 DMCUB SRAM
    ret = dmub_copy_to_sram(srv,
                            DMUB_SRAM_BASE,
                            fw->data,
                            fw->size);
    if (ret) {
        DRM_ERROR("Failed to copy DMUB firmware to SRAM\n");
        goto release_fw;
    }

    DRM_INFO("DMUB firmware loaded: v%u.%u.%u.%u\n",
             meta->version_major, meta->version_minor,
             meta->version_revision, meta->version_build);

release_fw:
    release_firmware(fw);
    return ret;
}

// 固件版本兼容性检查
bool dmub_check_hw_compatibility(struct dmub_srv *srv,
                                  struct dmub_fw_meta *meta)
{
    uint32_t hw_version = dmub_get_hw_version(srv);

    if (hw_version < meta->hw_version_min ||
        hw_version > meta->hw_version_max) {
        DRM_WARN("DMUB firmware HW version mismatch: "
                 "HW=%u, FW supports %u-%u\n",
                 hw_version,
                 meta->hw_version_min,
                 meta->hw_version_max);
        return false;
    }

    return true;
}
```

## 3. 环形缓冲区 (Ring Buffer) 通信机制

DMUB 使用无锁环形缓冲区 (lock-free ring buffer) 在主机 CPU 和 DMCUB 之间传递消息。这是整个通信架构的核心。

### 3.1 dmub_rb 结构体

```c
// 环形缓冲区结构体
struct dmub_rb {
    // 缓冲区基地址 (GPU 可见内存)
    struct dmub_region base_address;

    // 缓冲区大小 (必须是 2 的幂)
    uint32_t capacity;

    // 写指针 (生产者写入位置)
    volatile uint32_t wrpt;

    // 读指针 (消费者读取位置)
    volatile uint32_t rdpt;

    // 缓冲区数据存储 (通过 base_address 映射)
    struct dmub_msg *buffer;

    // 写指针的 GPU 地址 (DMCUB 写入 notify_queue)
    uint64_t wrpt_gpu_addr;

    // 读指针的 GPU 地址 (DMCUB 读取 flush_queue)
    uint64_t rdpt_gpu_addr;
};

// 环形缓冲区中的消息结构
struct dmub_msg {
    // 消息头部
    struct {
        uint32_t type    : 8;   // 消息类型
        uint32_t length  : 8;   // 数据长度 (字节)
        uint32_t seq_num : 8;   // 序列号
        uint32_t flags   : 8;   // 标志位
    } header;

    // 消息数据
    uint32_t data[DMUB_MSG_MAX_DATA_DWORDS];
};

// 消息类型定义
enum dmub_msg_type {
    DMUB_MSG_TYPE_CMD     = 0x01,  // 命令消息
    DMUB_MSG_TYPE_RESP    = 0x02,  // 响应消息
    DMUB_MSG_TYPE_EVENT   = 0x03,  // 事件通知
    DMUB_MSG_TYPE_TRACE   = 0x04,  // 调试追踪
    DMUB_MSG_TYPE_ERROR   = 0xFF,  // 错误消息
};

// 消息标志位
#define DMUB_MSG_FLAG_NONE      0x00
#define DMUB_MSG_FLAG_URGENT    0x01  // 紧急消息，DMCUB 应优先处理
#define DMUB_MSG_FLAG_RESP_REQ  0x02  // 需要 DMCUB 回复
#define DMUB_MSG_FLAG_DELAYED   0x04  // 延迟处理 (非实时)
```

### 3.2 无锁队列操作原理

```c
// 主机发送命令到 DMCUB (写 flush_queue)
// 主机是 flush_queue 的生产者，DMCUB 是消费者
int dmub_send_command(struct dmub_srv *srv,
                       struct dmub_msg *msg)
{
    struct dmub_rb *flush_q = &srv->flush_queue;
    uint32_t wrpt, rdpt;
    uint32_t next_wrpt;
    uint32_t msg_size;
    int retry = 0;

    // 计算消息大小 (DWORD 对齐)
    msg_size = sizeof(struct dmub_msg_header) +
               ALIGN(msg->header.length, 4);

    do {
        // 读取当前写指针和读指针
        wrpt = READ_ONCE(flush_q->wrpt);
        rdpt = READ_ONCE(flush_q->rdpt);

        // 计算下一个写指针位置
        next_wrpt = (wrpt + msg_size) & (flush_q->capacity - 1);

        // 检查缓冲区是否有足够空间
        // 空: wrpt == rdpt
        // 满: (wrpt + msg_size) & (capacity - 1) == rdpt
        if (next_wrpt == rdpt) {
            // 队列满，等待 DMCUB 消费
            if (retry++ > DMUB_SEND_RETRY_MAX) {
                DRM_ERROR("DMUB flush queue full, "
                          "msg type=%d\n", msg->header.type);
                return -EAGAIN;
            }
            udelay(1);
            continue;
        }

        // 写入消息到缓冲区
        memcpy(&flush_q->buffer[wrpt], msg, msg_size);

        // 内存屏障：确保数据写入完成后再更新写指针
        wmb();

        // 更新写指针 (原子操作)
        WRITE_ONCE(flush_q->wrpt, next_wrpt);

        // 通知 DMCUB 有新消息 (通过 doorbell 寄存器)
        dmub_ring_doorbell(srv);

        return 0;

    } while (retry <= DMUB_SEND_RETRY_MAX);

    return -EAGAIN;
}

// 主机接收 DMCUB 通知 (读 notify_queue)
// 主机是 notify_queue 的消费者，DMCUB 是生产者
int dmub_receive_notification(struct dmub_srv *srv,
                               struct dmub_msg *msg)
{
    struct dmub_rb *notify_q = &srv->notify_queue;
    uint32_t wrpt, rdpt;
    uint32_t msg_size;

    // 读取读指针和写指针
    rdpt = READ_ONCE(notify_q->rdpt);
    wrpt = READ_ONCE(notify_q->wrpt);

    // 检查是否有新消息
    if (rdpt == wrpt)
        return 0;  // 无新消息

    // 读取消息
    msg_size = sizeof(struct dmub_msg_header) +
               ALIGN(msg->header.length, 4);

    // 从缓冲区复制消息
    memcpy(msg, &notify_q->buffer[rdpt], msg_size);

    // 内存屏障：确保数据读取完成后更新读指针
    rmb();

    // 更新读指针
    uint32_t next_rdpt = (rdpt + msg_size) &
                         (notify_q->capacity - 1);
    WRITE_ONCE(notify_q->rdpt, next_rdpt);

    return msg_size;
}
```

### 3.3 Doorbell 寄存器机制

```c
// Doorbell 寄存器 - 用于主机通知 DMCUB 处理消息
// DMCUB 在空闲时睡眠等待 doorbell，收到后处理 flush_queue

// Doorbell 寄存器地址
#define mmDMUB_DOORBELL_OFFSET         0x3A00
#define mmDMUB_DOORBELL_DATA           0x3A04

// Doorbell 类型
enum dmub_doorbell_type {
    DOORBELL_TYPE_FLUSH = 0,    // flush_queue 有新消息
    DOORBELL_TYPE_WAKE  = 1,    // 唤醒 DMCUB (从睡眠中)
    DOORBELL_TYPE_RESET = 2,    // 重置 DMCUB
    DOORBELL_TYPE_TRACE = 3,    // 追踪数据可用
};

// Doorbell 配置
struct dmub_doorbell_config {
    uint32_t doorbell_offset;    // Doorbell 寄存器偏移
    uint32_t doorbell_data;      // Doorbell 数据值
    uint32_t doorbell_ack;       // DMCUB 确认值
};

// 触发 Doorbell
void dmub_ring_doorbell(struct dmub_srv *srv)
{
    struct dmub_doorbell_config *db = &srv->doorbell_config;

    // 写入 doorbell 寄存器触发 DMCUB 中断
    dmub_write_reg(srv,
                   mmDMUB_DOORBELL_OFFSET,
                   db->doorbell_offset);

    // 写入 doorbell 数据
    dmub_write_reg(srv,
                   mmDMUB_DOORBELL_DATA,
                   db->doorbell_data | DOORBELL_TYPE_FLUSH);

    // 可选：等待 DMCUB 确认
    uint32_t timeout = 100;
    while (timeout--) {
        uint32_t ack = dmub_read_reg(srv, mmDMUB_DOORBELL_DATA);
        if (ack & DMUB_DOORBELL_ACK_MASK)
            break;
        udelay(1);
    }
}
```

### 3.4 完整通信流程

```
主机 (Host)                              DMCUB
   |                                       |
   | 1. 准备命令消息                       |
   |    (填充 msg 结构)                     |
   |                                       |
   | 2. 写入 flush_queue                   |
   |    (复制 msg 到缓冲区)                |
   |                                       |
   | 3. 内存屏障 (wmb)                     |
   |    (确保数据可见)                      |
   |                                       |
   | 4. 更新 wrpt                          |
   |    (原子写)                            |
   |                                       |
   | 5. 触发 Doorbell                     |
   |    (写寄存器)        +--------------->| 6. 收到 Doorbell 中断
   |                                       | 7. 读取 flush_queue
   |                                       |    (从 wrpt/rdpt 判断)
   |                                       | 8. 处理命令
   |                                       |    (PSR/ABM/时序等)
   |                                       | 9. 写入 notify_queue
   |                                       |    (添加响应/事件)
   | 11. 轮询 rdpt 变化  |<--------------+| 10. 更新 wrpt (notify)
   |    (或等待中断)      |               |    触发主机中断
   | 12. 读取 notify_queue                |
   | 13. 处理响应                         |
   |                                       |
```

## 4. DMUB 命令系统

DMUB 定义了一系列标准命令，用于主机向 DMCUB 发送控制请求。

### 4.1 命令枚举

```c
// DMUB 命令类型 (完整列表)
enum dmub_cmd_type {
    // 时序控制命令
    DMUB_CMD_SET_TIMING              = 0x01,
    DMUB_CMD_GET_TIMING              = 0x02,

    // 平面更新命令
    DMUB_CMD_UPDATE_PLANE            = 0x03,
    DMUB_CMD_DISABLE_PLANE           = 0x04,

    // 电源管理命令
    DMUB_CMD_SET_POWER_STATE         = 0x05,
    DMUB_CMD_QUERY_POWER_STATE       = 0x06,

    // ABM (Adaptive Backlight) 命令
    DMUB_CMD_ABM_INIT                = 0x10,
    DMUB_CMD_ABM_SET_LEVEL           = 0x11,
    DMUB_CMD_ABM_GET_LEVEL           = 0x12,
    DMUB_CMD_ABM_SET_PARAM           = 0x13,
    DMUB_CMD_ABM_PAUSE               = 0x14,
    DMUB_CMD_ABM_RESUME              = 0x15,

    // PSR (Panel Self Refresh) 命令
    DMUB_CMD_PSR_SET_VERSION         = 0x20,
    DMUB_CMD_PSR_ENTER               = 0x21,
    DMUB_CMD_PSR_EXIT                = 0x22,
    DMUB_CMD_PSR_GET_STATE           = 0x23,
    DMUB_CMD_PSR_SET_LEVEL           = 0x24,
    DMUB_CMD_PSR_FORCE_STATIC        = 0x25,

    // VRR (Variable Refresh Rate) 命令
    DMUB_CMD_VRR_UPDATE              = 0x30,
    DMUB_CMD_VRR_SET_RANGE           = 0x31,
    DMUB_CMD_VRR_GET_STATUS          = 0x32,

    // HPD 命令 (DPIA)
    DMUB_CMD_QUERY_HPD_STATE         = 0x40,
    DMUB_CMD_SET_HPD_FILTER          = 0x41,
    DMUB_CMD_GET_HPD_STATUS          = 0x42,

    // MST 命令
    DMUB_CMD_MST_SET_CONFIG          = 0x50,
    DMUB_CMD_MST_SEND_MSG            = 0x51,
    DMUB_CMD_MST_RECV_MSG            = 0x52,

    // GPIO 命令
    DMUB_CMD_GPIO_CONTROL            = 0x60,

    // 调试命令
    DMUB_CMD_TRACE_START             = 0xF0,
    DMUB_CMD_TRACE_STOP              = 0xF1,
    DMUB_CMD_TRACE_READ              = 0xF2,
    DMUB_CMD_GET_FW_VERSION          = 0xF3,
    DMUB_CMD_SET_DEBUG_MODE          = 0xF4,
};
```

### 4.2 命令数据结构

```c
// 通用命令头部
struct dmub_cmd_header {
    uint8_t type;           // 命令类型 (enum dmub_cmd_type)
    uint8_t sub_type;       // 子类型
    uint8_t payload_size;   // 负载大小 (字节)
    uint8_t reserved;
};

// SET_TIMING 命令 - 配置显示器时序
struct dmub_cmd_set_timing {
    struct dmub_cmd_header header;

    struct timing_params {
        uint32_t h_total;
        uint32_t h_addressable;
        uint32_t h_sync_start;
        uint32_t h_sync_width;
        uint32_t v_total;
        uint32_t v_addressable;
        uint32_t v_sync_start;
        uint32_t v_sync_width;
        uint32_t pix_clk_100hz;
        uint32_t h_border_left;
        uint32_t h_border_right;
        uint32_t v_border_top;
        uint32_t v_border_bottom;
        uint8_t  color_depth;
        uint8_t  flags;
        uint8_t  otg_inst;  // OTG 实例编号
        uint8_t  reserved;
    } timing;

    uint32_t crtc_timing_params;
};

// SET_POWER_STATE 命令 - 控制电源状态
struct dmub_cmd_set_power_state {
    struct dmub_cmd_header header;

    uint8_t  power_state;    // POWER_STATE_D0/D1/D2/D3
    uint8_t  display_count;  // 活跃显示器数量
    uint8_t  psr_allowed;    // 是否允许 PSR
    uint8_t  abm_allowed;    // 是否允许 ABM

    uint32_t min_dispclk;    // 最低 DISPCLK 需求
    uint32_t min_dcfclk;     // 最低 DCFCLK 需求
};

// PSR_ENTER 命令 - 进入 Panel Self Refresh
struct dmub_cmd_psr_enter {
    struct dmub_cmd_header header;

    uint8_t  psr_level;      // PSR 级别 (PSR1/PSR2 SU/SD)
    uint8_t  pipe_mask;      // 参与的管道位掩码
    uint16_t idle_time_ms;   // 空闲超时时间
    uint32_t frame_delay;    // 帧延迟计数
    uint32_t phy_link_rate;  // 物理链路速率 (保持连接)
};

// PSR_EXIT 命令 - 退出 Panel Self Refresh
struct dmub_cmd_psr_exit {
    struct dmub_cmd_header header;

    uint32_t exit_reason;    // 退出原因
    uint8_t  skip_frame;     // 退出时是否跳过一帧
    uint8_t  force_full_update; // 强制全帧更新
    uint16_t reserved;
};

// 退出 PSR 的原因
enum dmub_psr_exit_reason {
    PSR_EXIT_REASON_HOST_TRIGGERED    = 0,
    PSR_EXIT_REASON_SYNC_VSYNC       = 1,
    PSR_EXIT_REASON_CURSOR_UPDATE    = 2,
    PSR_EXIT_REASON_PLANE_UPDATE     = 3,
    PSR_EXIT_REASON_HPD_IRQ          = 4,
    PSR_EXIT_REASON_MULTI_DISPLAY    = 5,
    PSR_EXIT_REASON_AUDIO_ACTIVE     = 6,
    PSR_EXIT_REASON_VRR_ACTIVE       = 7,
};

// ABM_SET_LEVEL 命令 - 设置自适应背光级别
struct dmub_cmd_abm_set_level {
    struct dmub_cmd_header header;

    uint8_t  level;           // 背光级别 (0-255)
    uint8_t  ramp_up;         // 是否渐变调整
    uint8_t  panel_inst;      // 面板实例
    uint8_t  reserved;

    uint32_t transition_ms;   // 渐变时间 (毫秒)
    uint8_t  ambient_lux;     // 环境光传感器读数
    uint8_t  backlight_min;   // 最小背光值
    uint8_t  backlight_max;   // 最大背光值
    uint8_t  flags;
};

// VRR_UPDATE 命令 - 更新可变刷新率设置
struct dmub_cmd_vrr_update {
    struct dmub_cmd_header header;

    uint32_t min_refresh_rate;    // 最低刷新率 (mHz)
    uint32_t max_refresh_rate;    // 最高刷新率 (mHz)
    uint32_t current_refresh_rate; // 当前刷新率 (mHz)
    uint8_t  vrr_enabled;
    uint8_t  vrr_active;
    uint8_t  freesync_supported;
    uint8_t  reserved;
};

// UPDATE_PLANE 命令 - 更新显示平面
struct dmub_cmd_update_plane {
    struct dmub_cmd_header header;

    uint8_t  pipe_idx;           // 管道索引
    uint8_t  plane_type;         // 平面类型 (OVL/DRR/CUR)
    uint8_t  enable;             // 启用/禁用
    uint8_t  reserved;

    struct {
        uint32_t src_x;
        uint32_t src_y;
        uint32_t src_w;
        uint32_t src_h;
        uint32_t dst_x;
        uint32_t dst_y;
        uint32_t dst_w;
        uint32_t dst_h;
    } addr;

    uint32_t format;             // 像素格式
    uint32_t pitch;              // 行跨度
    uint64_t addr_gpu;           // GPU 地址
};
```

### 4.3 命令发送辅助函数

```c
// 通用的命令发送辅助函数
int dmub_send_command_sync(struct dmub_srv *srv,
                            struct dmub_msg *msg,
                            uint32_t timeout_ms)
{
    int ret;

    // 1. 发送命令
    ret = dmub_send_command(srv, msg);
    if (ret)
        return ret;

    // 2. 等待 DMCUB 处理完成 (通过 notify_queue 或轮询)
    uint32_t elapsed = 0;
    while (elapsed < timeout_ms) {
        // 检查是否收到响应
        struct dmub_msg response;
        int resp_size = dmub_receive_notification(srv, &response);

        if (resp_size > 0) {
            // 验证响应匹配
            if (response.header.seq_num == msg->header.seq_num)
                return 0;
        }

        // 等待一小段时间
        udelay(100);
        elapsed++;
    }

    return -ETIMEDOUT;
}

// PSR 进入辅助函数
int dmub_psr_enter(struct dmub_srv *srv,
                    uint8_t psr_level,
                    uint8_t pipe_mask)
{
    struct dmub_msg msg = {0};
    struct dmub_cmd_psr_enter *cmd =
        (struct dmub_cmd_psr_enter *)msg.data;

    // 填充消息头部
    msg.header.type = DMUB_MSG_TYPE_CMD;
    msg.header.length = sizeof(struct dmub_cmd_psr_enter);
    msg.header.seq_num = srv->next_seq_num++;
    msg.header.flags = DMUB_MSG_FLAG_RESP_REQ;

    // 填充命令数据
    cmd->header.type = DMUB_CMD_PSR_ENTER;
    cmd->header.payload_size = sizeof(struct dmub_cmd_psr_enter) -
                               sizeof(struct dmub_cmd_header);
    cmd->psr_level = psr_level;
    cmd->pipe_mask = pipe_mask;
    cmd->idle_time_ms = 100;  // 100ms 空闲后进入 PSR

    // 发送并等待响应
    return dmub_send_command_sync(srv, &msg, 100);
}

// PSR 退出辅助函数
int dmub_psr_exit(struct dmub_srv *srv,
                   enum dmub_psr_exit_reason reason)
{
    struct dmub_msg msg = {0};
    struct dmub_cmd_psr_exit *cmd =
        (struct dmub_cmd_psr_exit *)msg.data;

    msg.header.type = DMUB_MSG_TYPE_CMD;
    msg.header.length = sizeof(struct dmub_cmd_psr_exit);
    msg.header.seq_num = srv->next_seq_num++;
    msg.header.flags = DMUB_MSG_FLAG_RESP_REQ | DMUB_MSG_FLAG_URGENT;

    cmd->header.type = DMUB_CMD_PSR_EXIT;
    cmd->exit_reason = reason;
    cmd->skip_frame = false;
    cmd->force_full_update = true;

    return dmub_send_command_sync(srv, &msg, 50);
}

// ABM 级别设置辅助函数
int dmub_abm_set_level(struct dmub_srv *srv,
                        uint8_t level,
                        uint8_t panel_inst)
{
    struct dmub_msg msg = {0};
    struct dmub_cmd_abm_set_level *cmd =
        (struct dmub_cmd_abm_set_level *)msg.data;

    msg.header.type = DMUB_MSG_TYPE_CMD;
    msg.header.length = sizeof(struct dmub_cmd_abm_set_level);
    msg.header.seq_num = srv->next_seq_num++;

    cmd->header.type = DMUB_CMD_ABM_SET_LEVEL;
    cmd->level = level;
    cmd->panel_inst = panel_inst;
    cmd->ramp_up = true;
    cmd->transition_ms = 500;  // 500ms 渐变时间

    return dmub_send_command_sync(srv, &msg, 200);
}
```

## 5. DMUB 与 PSR 集成

PSR (Panel Self Refresh) 是 DMUB 最重要的功能之一。DMCUB 负责精确控制 PSR 的进入和退出时序。

### 5.1 PSR 状态机在 DMUB 中的实现

```c
// DMCUB 内部的 PSR 状态机
enum psr_state {
    PSR_STATE_INACTIVE,      // 未激活 PSR
    PSR_STATE_IDLE,          // 空闲，等待进入条件
    PSR_STATE_ENTERING,      // 正在进入 PSR
    PSR_STATE_ACTIVE,        // PSR 激活中
    PSR_STATE_EXITING,       // 正在退出 PSR
    PSR_STATE_ERROR,         // 错误状态
};

// DMCUB 固件中的 PSR 控制结构
struct dmub_psr_context {
    enum psr_state state;           // 当前状态
    uint32_t idle_frame_count;      // 空闲帧计数
    uint32_t idle_threshold;        // 进入 PSR 的空闲阈值
    uint32_t last_vblank_time;      // 上次 VBlank 时间
    uint32_t frame_stable_count;    // 帧稳定计数

    // PSR 进入条件检查
    bool cursor_update_pending;     // 光标更新待处理
    bool plane_update_pending;      // 平面更新待处理
    bool audio_active;              // 音频活跃中
    bool vrr_active;                // VRR 活跃中

    // 退出原因追踪
    uint32_t exit_count;            // 退出次数统计
    enum dmub_psr_exit_reason last_exit_reason;

    // 定时器
    uint32_t entry_timer_ms;        // 进入延迟定时器
    uint32_t exit_timer_us;         // 退出超时定时器
};
```

### 5.2 PSR 进入/退出流程

```c
// 主机端 PSR 控制函数 (在 amdgpu_dm_psr.c 中)

// 尝试进入 PSR
int amdgpu_dm_psr_enable(struct amdgpu_device *adev,
                          struct dmub_srv *srv,
                          struct dc_link *link,
                          bool enable)
{
    int ret;

    if (!link->psr_settings.psr_feature_enabled)
        return -EOPNOTSUPP;

    if (enable) {
        // 检查 PSR 进入条件
        if (link->audio_stream_active) {
            DRM_DEBUG("PSR blocked: audio active\n");
            return -EBUSY;
        }

        if (link->psr_settings.psr_active) {
            DRM_DEBUG("PSR already active\n");
            return 0;
        }

        // 发送 PSR_ENTER 命令到 DMCUB
        ret = dmub_psr_enter(srv,
                             link->psr_settings.psr_level,
                             1 << link->psr_settings.pipe_idx);
        if (ret == 0) {
            link->psr_settings.psr_active = true;
            link->psr_settings.psr_entry_count++;

            DRM_DEBUG("PSR entered (level=%d, count=%d)\n",
                      link->psr_settings.psr_level,
                      link->psr_settings.psr_entry_count);
        }
    } else {
        // 退出 PSR
        ret = dmub_psr_exit(srv, PSR_EXIT_REASON_HOST_TRIGGERED);
        if (ret == 0) {
            link->psr_settings.psr_active = false;
            DRM_DEBUG("PSR exited\n");
        }
    }

    return ret;
}

// 平面更新时强制退出 PSR
int amdgpu_dm_psr_exit_plane_update(struct amdgpu_device *adev,
                                     struct dc_link *link)
{
    if (!link->psr_settings.psr_active)
        return 0;

    // 平面更新需要立即退出 PSR
    return dmub_psr_exit(link->dmub_srv,
                         PSR_EXIT_REASON_PLANE_UPDATE);
}

// 光标移动触发 PSR 退出
int amdgpu_dm_psr_exit_cursor_move(struct amdgpu_device *adev,
                                    struct dc_link *link)
{
    if (!link->psr_settings.psr_active)
        return 0;

    // 光标更新同样需要退出 PSR
    return dmub_psr_exit(link->dmub_srv,
                         PSR_EXIT_REASON_CURSOR_UPDATE);
}
```

### 5.3 PSR 调试接口

```c
// PSR 调试统计信息
struct dmub_psr_debug_stats {
    // 基本计数
    uint32_t entry_count;        // PSR 进入总次数
    uint32_t exit_count;         // PSR 退出总次数
    uint32_t total_psr_time_ms;  // 累计 PSR 时间 (ms)

    // 最近状态
    bool     psr_active;
    bool     psr_ready;

    // 退出原因分布
    uint32_t exit_reasons[8];    // 每种退出原因的次数

    // 性能数据
    uint32_t avg_entry_time_us;  // 平均进入时间
    uint32_t avg_exit_time_us;   // 平均退出时间
    uint32_t max_entry_time_us;  // 最大进入时间
    uint32_t max_exit_time_us;   // 最大退出时间

    // 错误计数
    uint32_t entry_failures;
    uint32_t exit_failures;
    uint32_t crc_errors;
    uint32_t timeout_errors;
};

// 读取 PSR 调试信息 (通过 DMCUB 命令)
int dmub_psr_get_debug_stats(struct dmub_srv *srv,
                              struct dmub_psr_debug_stats *stats)
{
    struct dmub_msg msg = {0};
    struct dmub_cmd_header *cmd =
        (struct dmub_cmd_header *)msg.data;

    msg.header.type = DMUB_MSG_TYPE_CMD;
    msg.header.length = sizeof(struct dmub_cmd_header);
    msg.header.seq_num = srv->next_seq_num++;
    msg.header.flags = DMUB_MSG_FLAG_RESP_REQ;

    cmd->type = DMUB_CMD_PSR_GET_STATE;

    // 发送命令并读取响应
    int ret = dmub_send_command(srv, &msg);
    if (ret)
        return ret;

    // 从 notify_queue 读取统计信息
    struct dmub_msg response;
    ret = dmub_receive_notification(srv, &response);
    if (ret > 0) {
        memcpy(stats, response.data, sizeof(*stats));
        return 0;
    }

    return -EIO;
}

// debugfs 接口: 显示 PSR 状态
static int dmub_psr_debugfs_show(struct seq_file *m, void *data)
{
    struct amdgpu_device *adev = m->private;
    struct dmub_psr_debug_stats stats;

    if (dmub_psr_get_debug_stats(adev->dmub_srv, &stats))
        return -EIO;

    seq_printf(m, "PSR Status:\n");
    seq_printf(m, "  Active:       %s\n",
               stats.psr_active ? "yes" : "no");
    seq_printf(m, "  Ready:        %s\n",
               stats.psr_ready ? "yes" : "no");
    seq_printf(m, "  Entry Count:  %u\n", stats.entry_count);
    seq_printf(m, "  Exit Count:   %u\n", stats.exit_count);
    seq_printf(m, "  Total PSR:    %u ms\n",
               stats.total_psr_time_ms);
    seq_printf(m, "\n");
    seq_printf(m, "Exit Reasons:\n");
    seq_printf(m, "  Host Triggered:  %u\n",
               stats.exit_reasons[PSR_EXIT_REASON_HOST_TRIGGERED]);
    seq_printf(m, "  VBlank Sync:    %u\n",
               stats.exit_reasons[PSR_EXIT_REASON_SYNC_VSYNC]);
    seq_printf(m, "  Cursor Update:  %u\n",
               stats.exit_reasons[PSR_EXIT_REASON_CURSOR_UPDATE]);
    seq_printf(m, "  Plane Update:   %u\n",
               stats.exit_reasons[PSR_EXIT_REASON_PLANE_UPDATE]);
    seq_printf(m, "  HPD IRQ:        %u\n",
               stats.exit_reasons[PSR_EXIT_REASON_HPD_IRQ]);
    seq_printf(m, "  Audio Active:   %u\n",
               stats.exit_reasons[PSR_EXIT_REASON_AUDIO_ACTIVE]);
    seq_printf(m, "\n");
    seq_printf(m, "Performance:\n");
    seq_printf(m, "  Avg Entry:   %u us\n",
               stats.avg_entry_time_us);
    seq_printf(m, "  Avg Exit:    %u us\n",
               stats.avg_exit_time_us);
    seq_printf(m, "  Max Entry:   %u us\n",
               stats.max_entry_time_us);
    seq_printf(m, "  Max Exit:    %u us\n",
               stats.max_exit_time_us);
    seq_printf(m, "\n");
    seq_printf(m, "Errors:\n");
    seq_printf(m, "  Entry Failures: %u\n",
               stats.entry_failures);
    seq_printf(m, "  Exit Failures:  %u\n",
               stats.exit_failures);
    seq_printf(m, "  CRC Errors:     %u\n",
               stats.crc_errors);
    seq_printf(m, "  Timeouts:       %u\n",
               stats.timeout_errors);

    return 0;
}
```

## 6. DMUB 与 ABM 集成

ABM (Adaptive Backlight Modulation) 是 DMUB 的另一核心功能，DMCUB 根据画面内容实时调整背光亮度。

### 6.1 ABM 控制结构

```c
// ABM 配置参数
struct dmub_abm_config {
    // ABM 级别 (0=禁用, 1-4=不同强度)
    uint8_t level;

    // 背光控制范围
    uint8_t backlight_min;       // 最小背光 (0-255)
    uint8_t backlight_max;       // 最大背光 (0-255)
    uint8_t backlight_default;   // 默认背光 (0-255)

    // 环境光补偿
    bool ambient_light_compensation;
    uint8_t ambient_light_threshold;

    // 渐变配置
    uint16_t ramp_up_ms;         // 亮度增加渐变时间
    uint16_t ramp_down_ms;       // 亮度减少渐变时间

    // 帧分析参数
    uint16_t frame_sample_interval; // 帧采样间隔
    uint8_t  histogram_bins;        // 直方图分箱数
    uint8_t  black_level_threshold; // 黑色级别阈值
};

// ABM 缓存 (在 dmub_srv 中)
struct dmub_abm_cache {
    // 缓存 ABM 配置以避免重复发送
    struct dmub_abm_config config;

    // 上次设置的时间戳
    unsigned long last_update_jiffies;

    // 缓存是否有效
    bool valid;
};

// ABM 统计分析
struct dmub_abm_stats {
    uint32_t total_adjustments;      // 总调整次数
    uint32_t level_changes;          // 级别变更次数
    uint32_t frame_analyzed;         // 已分析帧数

    uint32_t avg_brightness;         // 平均亮度
    uint32_t min_brightness;         // 最小亮度
    uint32_t max_brightness;         // 最大亮度

    uint32_t ramp_up_count;          // 渐变增加次数
    uint32_t ramp_down_count;        // 渐变减少次数

    uint32_t ambient_low_count;      // 环境光过低次数
    uint32_t ambient_high_count;     // 环境光过高次数
};
```

### 6.2 ABM 命令发送与回调

```c
// DM 层 ABM 控制函数
void amdgpu_dm_update_abm_level(struct amdgpu_device *adev,
                                 struct dc_link *link,
                                 uint8_t level)
{
    struct dmub_srv *srv = adev->dmub_srv;

    // 检查缓存，避免重复发送相同命令
    if (srv->abm_cache.valid &&
        srv->abm_cache.config.level == level) {
        return;  // 无变化，跳过
    }

    // 限制有效范围
    level = min(level, (uint8_t)DMUB_ABM_MAX_LEVEL);

    // 发送 ABM_SET_LEVEL 命令到 DMCUB
    int ret = dmub_abm_set_level(srv, level, 0);
    if (ret == 0) {
        // 更新缓存
        srv->abm_cache.config.level = level;
        srv->abm_cache.last_update_jiffies = jiffies;
        srv->abm_cache.valid = true;

        DRM_DEBUG("ABM level updated to %d\n", level);
    } else {
        DRM_WARN("Failed to update ABM level to %d: %d\n",
                 level, ret);
    }
}

// 初始化 ABM
int dmub_abm_init(struct dmub_srv *srv,
                   struct dmub_abm_config *config)
{
    struct dmub_msg msg = {0};
    struct dmub_cmd_abm_init *cmd =
        (struct dmub_cmd_abm_init *)msg.data;

    msg.header.type = DMUB_MSG_TYPE_CMD;
    msg.header.length = sizeof(struct dmub_cmd_abm_init);
    msg.header.seq_num = srv->next_seq_num++;
    msg.header.flags = DMUB_MSG_FLAG_RESP_REQ;

    cmd->header.type = DMUB_CMD_ABM_INIT;
    cmd->config = *config;

    int ret = dmub_send_command_sync(srv, &msg, 500);

    if (ret == 0) {
        // 缓存初始化配置
        srv->abm_cache.config = *config;
        srv->abm_cache.valid = true;
        srv->feature_active[DMUB_FUNC_ABM] = true;
    }

    return ret;
}

// ABM 暂停/恢复 (在系统挂起/恢复时使用)
int dmub_abm_pause(struct dmub_srv *srv)
{
    struct dmub_msg msg = {0};
    struct dmub_cmd_header *cmd =
        (struct dmub_cmd_header *)msg.data;

    msg.header.type = DMUB_MSG_TYPE_CMD;
    msg.header.length = sizeof(struct dmub_cmd_header);
    msg.header.seq_num = srv->next_seq_num++;

    cmd->type = DMUB_CMD_ABM_PAUSE;

    return dmub_send_command_sync(srv, &msg, 200);
}

int dmub_abm_resume(struct dmub_srv *srv)
{
    struct dmub_msg msg = {0};
    struct dmub_cmd_header *cmd =
        (struct dmub_cmd_header *)msg.data;

    msg.header.type = DMUB_MSG_TYPE_CMD;
    msg.header.length = sizeof(struct dmub_cmd_header);
    msg.header.seq_num = srv->next_seq_num++;

    cmd->type = DMUB_CMD_ABM_RESUME;

    return dmub_send_command_sync(srv, &msg, 200);
}
```

## 7. DMUB 状态管理

### 7.1 DMUB 状态枚举

```c
// DMUB 状态枚举
enum dmub_status {
    DMUB_STATUS_OK          = 0,  // 正常运行
    DMUB_STATUS_INIT        = 1,  // 初始化中
    DMUB_STATUS_ERROR       = 2,  // 错误状态
    DMUB_STATUS_SUSPENDED   = 3,  // 已挂起
    DMUB_STATUS_HW_ERROR    = 4,  // 硬件错误
    DMUB_STATUS_FW_ERROR    = 5,  // 固件错误
    DMUB_STATUS_TIMEOUT     = 6,  // 超时
    DMUB_STATUS_QUEUE_FULL  = 7,  // 队列满
    DMUB_STATUS_QUEUE_EMPTY = 8,  // 队列空
    DMUB_STATUS_NOT_READY   = 9,  // 未就绪
};

// DMUB 错误信息
struct dmub_error_info {
    uint32_t error_code;          // 错误码
    uint32_t error_line;          // 出错行号 (固件内)
    char     error_file[32];      // 出错文件名 (固件内)
    uint32_t fw_pc;               // 固件程序计数器
    uint32_t fw_lr;               // 固件链接寄存器
    uint32_t fw_sp;               // 固件栈指针
    uint32_t timestamp_ms;        // 错误时间戳
};

// 获取 DMCUB 错误信息
int dmub_get_error_info(struct dmub_srv *srv,
                         struct dmub_error_info *err_info)
{
    // 从 DMCUB 的错误寄存器读取
    err_info->error_code = dmub_read_reg(srv, DMUB_REG_ERROR_CODE);
    err_info->fw_pc = dmub_read_reg(srv, DMUB_REG_FW_PC);
    err_info->fw_lr = dmub_read_reg(srv, DMUB_REG_FW_LR);
    err_info->fw_sp = dmub_read_reg(srv, DMUB_REG_FW_SP);
    err_info->timestamp_ms = jiffies_to_msecs(jiffies);

    // 从专用 SRAM 区域读取错误字符串
    dmub_read_from_sram(srv,
                        DMUB_SRAM_ERROR_INFO,
                        err_info->error_file,
                        sizeof(err_info->error_file));

    return 0;
}
```

### 7.2 系统挂起/恢复处理

```c
// DMUB 系统挂起
int dmub_srv_suspend(struct dmub_srv *srv)
{
    int ret;

    if (srv->status == DMUB_STATUS_SUSPENDED)
        return 0;

    DRM_DEBUG("DMUB suspending...\n");

    // 1. 暂停 ABM
    if (srv->feature_active[DMUB_FUNC_ABM]) {
        dmub_abm_pause(srv);
    }

    // 2. 强制退出 PSR (如果活跃)
    for (int i = 0; i < srv->link_count; i++) {
        if (srv->links[i]->psr_settings.psr_active) {
            dmub_psr_exit(srv,
                          PSR_EXIT_REASON_HOST_TRIGGERED);
        }
    }

    // 3. 等待 DMCUB 处理完所有待处理命令
    ret = dmub_wait_for_idle(srv, 100);
    if (ret) {
        DRM_WARN("DMUB not idle before suspend\n");
    }

    // 4. 停止 DMCUB
    dmub_stop_firmware(srv);

    // 5. 保存 DMCUB 状态 (如果需要恢复)
    ret = dmub_save_state(srv);
    if (ret) {
        DRM_WARN("Failed to save DMUB state: %d\n", ret);
    }

    srv->status = DMUB_STATUS_SUSPENDED;
    DRM_DEBUG("DMUB suspended\n");

    return 0;
}

// DMUB 系统恢复
int dmub_srv_resume(struct dmub_srv *srv)
{
    int ret;

    if (srv->status != DMUB_STATUS_SUSPENDED)
        return 0;

    DRM_DEBUG("DMUB resuming...\n");

    // 1. 恢复 DMCUB 状态 (如果是 S3 恢复)
    ret = dmub_restore_state(srv);
    if (ret) {
        // 如果状态恢复失败，重新初始化
        DRM_WARN("Failed to restore DMUB state, reinitializing\n");
        ret = dmub_srv_init(srv);
        if (ret) {
            DRM_ERROR("Failed to reinitialize DMUB: %d\n", ret);
            return ret;
        }
    }

    // 2. 重新启动固件
    dmub_start_firmware(srv);

    // 3. 等待 DMCUB 就绪
    ret = dmub_wait_ready(srv, 1000);
    if (ret) {
        DRM_ERROR("DMUB not ready after resume\n");
        return ret;
    }

    // 4. 重新初始化 ABM (如果之前活跃)
    if (srv->feature_active[DMUB_FUNC_ABM]) {
        dmub_abm_init(srv, &srv->abm_cache.config);
    }

    // 5. 重新启用 PSR (如果显示器支持)
    for (int i = 0; i < srv->link_count; i++) {
        if (srv->links[i]->psr_settings.psr_feature_enabled) {
            amdgpu_dm_psr_enable(srv->adev,
                                 srv,
                                 srv->links[i],
                                 true);
        }
    }

    srv->status = DMUB_STATUS_OK;
    DRM_DEBUG("DMUB resumed\n");

    return 0;
}

// 等待 DMCUB 空闲
int dmub_wait_for_idle(struct dmub_srv *srv, uint32_t timeout_ms)
{
    uint32_t elapsed = 0;

    while (elapsed < timeout_ms) {
        // 检查 flush_queue 是否全部处理完毕
        uint32_t wrpt = READ_ONCE(srv->flush_queue.wrpt);
        uint32_t rdpt = READ_ONCE(srv->flush_queue.rdpt);

        if (wrpt == rdpt)
            return 0;  // 队列已空

        // 检查 DMCUB 状态寄存器
        uint32_t status = dmub_read_reg(srv, DMUB_REG_STATUS);
        if (status & DMUB_STATUS_IDLE_BIT)
            return 0;

        msleep(1);
        elapsed++;
    }

    return -ETIMEDOUT;
}
```

## 8. DMUB 调试与追踪

### 8.1 追踪缓冲区

```c
// DMUB 追踪缓冲区 (用于调试)
struct dmub_trace_buf {
    // 追踪缓冲区地址 (GPU 可见内存)
    struct dmub_region buf_addr;

    // 缓冲区大小
    uint32_t buf_size;

    // 写指针 (DMCUB 写入)
    volatile uint32_t wrpt;

    // 读指针 (主机读取)
    volatile uint32_t rdpt;

    // 追踪条目格式
    struct trace_entry {
        uint32_t timestamp;     // 时间戳 (DMCUB 时钟周期)
        uint16_t event_id;      // 事件 ID
        uint16_t line;          // 源码行号
        uint32_t data[4];       // 附加数据
        char     file[16];      // 源文件名
    } *entries;
};

// 追踪事件 ID 定义
enum dmub_trace_event {
    TRACE_EVENT_PSR_ENTER       = 0x100,
    TRACE_EVENT_PSR_EXIT        = 0x101,
    TRACE_EVENT_PSR_IDLE        = 0x102,
    TRACE_EVENT_ABM_UPDATE      = 0x200,
    TRACE_EVENT_ABM_LEVEL_CHG   = 0x201,
    TRACE_EVENT_VRR_UPDATE      = 0x300,
    TRACE_EVENT_VRR_FLIP        = 0x301,
    TRACE_EVENT_HPD_RX          = 0x400,
    TRACE_EVENT_TIMING_SET      = 0x500,
    TRACE_EVENT_POWER_STATE     = 0x600,
    TRACE_EVENT_ERROR           = 0xFFF,
};

// 启动追踪
int dmub_trace_start(struct dmub_srv *srv)
{
    struct dmub_msg msg = {0};
    struct dmub_cmd_header *cmd =
        (struct dmub_cmd_header *)msg.data;

    msg.header.type = DMUB_MSG_TYPE_CMD;
    msg.header.length = sizeof(struct dmub_cmd_header);
    msg.header.seq_num = srv->next_seq_num++;

    cmd->type = DMUB_CMD_TRACE_START;

    // 分配追踪缓冲区
    if (!srv->trace_buf.buf_addr.base_address) {
        // 从 GPU 内存分配 64KB 追踪缓冲区
        int ret = dmub_alloc_trace_buffer(srv, 65536);
        if (ret)
            return ret;
    }

    srv->trace_enabled = true;

    return dmub_send_command_sync(srv, &msg, 100);
}

// 停止追踪
int dmub_trace_stop(struct dmub_srv *srv)
{
    struct dmub_msg msg = {0};
    struct dmub_cmd_header *cmd =
        (struct dmub_cmd_header *)msg.data;

    msg.header.type = DMUB_MSG_TYPE_CMD;
    msg.header.length = sizeof(struct dmub_cmd_header);
    msg.header.seq_num = srv->next_seq_num++;

    cmd->type = DMUB_CMD_TRACE_STOP;

    srv->trace_enabled = false;

    return dmub_send_command_sync(srv, &msg, 100);
}

// 读取追踪条目
int dmub_trace_read(struct dmub_srv *srv,
                     struct dmub_trace_buf *buf)
{
    uint32_t rdpt = READ_ONCE(buf->rdpt);
    uint32_t wrpt = READ_ONCE(buf->wrpt);

    if (rdpt == wrpt)
        return 0;  // 无新条目

    int count = 0;
    while (rdpt != wrpt) {
        struct trace_entry *entry = &buf->entries[rdpt];

        // 打印追踪条目
        DRM_DEBUG("[DMUB TRACE] event=0x%04X file=%s line=%u "
                  "ts=%u data=[0x%08X 0x%08X 0x%08X 0x%08X]\n",
                  entry->event_id, entry->file, entry->line,
                  entry->timestamp,
                  entry->data[0], entry->data[1],
                  entry->data[2], entry->data[3]);

        rdpt = (rdpt + 1) & (buf->buf_size /
               sizeof(struct trace_entry) - 1);
        count++;
    }

    // 更新读指针
    WRITE_ONCE(buf->rdpt, rdpt);

    return count;
}
```

### 8.2 debugfs 接口

```c
// DMUB debugfs 文件定义
static const struct drm_debugfs_reg32 dmub_debugfs_regs[] = {
    {"DMUB_STATUS",    DMUB_REG_STATUS},
    {"DMUB_CTRL",      DMUB_REG_CONTROL},
    {"DMUB_SCRATCH0",  DMUB_REG_SCRATCH0},
    {"DMUB_SCRATCH1",  DMUB_REG_SCRATCH1},
    {"DMUB_SCRATCH2",  DMUB_REG_SCRATCH2},
    {"DMUB_SCRATCH3",  DMUB_REG_SCRATCH3},
    {"DMUB_FW_PC",     DMUB_REG_FW_PC},
    {"DMUB_FW_LR",     DMUB_REG_FW_LR},
    {"DMUB_FW_SP",     DMUB_REG_FW_SP},
    {"DMUB_ERROR_CODE",DMUB_REG_ERROR_CODE},
    {"DMUB_DOORBELL",  DMUB_REG_DOORBELL},
};

// debugfs 读取 DMUB 状态
static int dmub_debugfs_status(struct seq_file *m, void *data)
{
    struct amdgpu_device *adev = m->private;
    struct dmub_srv *srv = adev->dmub_srv;

    if (!srv) {
        seq_puts(m, "DMUB not initialized\n");
        return 0;
    }

    seq_printf(m, "DMUB Status:\n");
    seq_printf(m, "  FW Version:   %u.%u.%u\n",
               (srv->fw_version >> 16) & 0xFF,
               (srv->fw_version >> 8) & 0xFF,
               srv->fw_version & 0xFF);
    seq_printf(m, "  Status:       %d\n", srv->status);
    seq_printf(m, "  HW Init:      %s\n",
               srv->hw_init ? "yes" : "no");
    seq_printf(m, "  SW Init:      %s\n",
               srv->sw_init ? "yes" : "no");
    seq_printf(m, "\n");

    // 各功能状态
    seq_printf(m, "Features:\n");
    seq_printf(m, "  PSR:    %s\n",
               srv->feature_active[DMUB_FUNC_PSR] ?
               "active" : "inactive");
    seq_printf(m, "  ABM:    %s\n",
               srv->feature_active[DMUB_FUNC_ABM] ?
               "active" : "inactive");
    seq_printf(m, "  HPD:    %s\n",
               srv->feature_active[DMUB_FUNC_HPD] ?
               "active" : "inactive");
    seq_printf(m, "  VRR:    %s\n",
               srv->feature_active[DMUB_FUNC_VRR] ?
               "active" : "inactive");
    seq_printf(m, "\n");

    // 队列状态
    seq_printf(m, "Queues:\n");
    seq_printf(m, "  Flush Queue:  wrpt=%u rdpt=%u used=%u\n",
               srv->flush_queue.wrpt,
               srv->flush_queue.rdpt,
               (srv->flush_queue.wrpt - srv->flush_queue.rdpt) &
               (srv->flush_queue.capacity - 1));
    seq_printf(m, "  Notify Queue: wrpt=%u rdpt=%u used=%u\n",
               srv->notify_queue.wrpt,
               srv->notify_queue.rdpt,
               (srv->notify_queue.wrpt - srv->notify_queue.rdpt) &
               (srv->notify_queue.capacity - 1));

    // 追踪状态
    seq_printf(m, "\nTrace: %s\n",
               srv->trace_enabled ? "enabled" : "disabled");

    // 读取并显示错误信息
    struct dmub_error_info err;
    if (dmub_get_error_info(srv, &err) == 0 && err.error_code) {
        seq_printf(m, "\nLast Error: code=0x%08X pc=0x%08X\n",
                   err.error_code, err.fw_pc);
    }

    return 0;
}

// DMUB 寄存器 dump
static int dmub_debugfs_regs(struct seq_file *m, void *data)
{
    struct amdgpu_device *adev = m->private;

    seq_printf(m, "%-16s %-10s\n", "Register", "Value");
    seq_printf(m, "%-16s %-10s\n", "--------", "-----");

    for (int i = 0; i < ARRAY_SIZE(dmub_debugfs_regs); i++) {
        uint32_t val = RREG32(dmub_debugfs_regs[i].reg_offset);
        seq_printf(m, "%-16s 0x%08X\n",
                   dmub_debugfs_regs[i].name, val);
    }

    return 0;
}

// DMUB debugfs 初始化
void dmub_debugfs_init(struct amdgpu_device *adev)
{
    struct drm_minor *minor = adev->ddev->primary;

    // 创建 DMUB debugfs 目录
    struct dentry *dmub_dir = debugfs_create_dir("amdgpu_dm_dmub",
                                                  minor->debugfs_root);

    // 注册 debugfs 文件
    debugfs_create_file("status", 0444,
                        dmub_dir, adev,
                        &dmub_debugfs_status_fops);
    debugfs_create_file("regs", 0444,
                        dmub_dir, adev,
                        &dmub_debugfs_regs_fops);
    debugfs_create_file("trace", 0644,
                        dmub_dir, adev,
                        &dmub_debugfs_trace_fops);
    debugfs_create_file("fw_version", 0444,
                        dmub_dir, adev,
                        &dmub_debugfs_fw_version_fops);

    // 创建 PSR debugfs 文件
    struct dentry *psr_dir = debugfs_create_dir("amdgpu_dm_psr",
                                                 minor->debugfs_root);
    debugfs_create_file("state", 0444,
                        psr_dir, adev,
                        &dmub_psr_debugfs_show_fops);
    debugfs_create_file("stats", 0444,
                        psr_dir, adev,
                        &dmub_psr_stats_fops);
    debugfs_create_file("set_level", 0644,
                        psr_dir, adev,
                        &dmub_psr_set_level_fops);
}
```

## 9. DMUB 固件更新与兼容性

### 9.1 固件版本管理

```c
// DMUB 固件版本兼容性矩阵
struct dmub_fw_compatibility {
    // 固件二进制文件名
    const char *fw_filename;

    // 支持的硬件 ID 列表
    uint32_t hw_id_list[8];
    uint32_t hw_id_count;

    // 最低驱动版本要求
    uint32_t min_driver_version;

    // 固件特性位图
    uint64_t features_bitmap;
};

// 固件特性位图定义
#define DMUB_FW_FEATURE_PSR1        (1ULL << 0)
#define DMUB_FW_FEATURE_PSR2        (1ULL << 1)
#define DMUB_FW_FEATURE_ABM         (1ULL << 2)
#define DMUB_FW_FEATURE_VRR         (1ULL << 3)
#define DMUB_FW_FEATURE_HPD_DPIA    (1ULL << 4)
#define DMUB_FW_FEATURE_MST         (1ULL << 5)
#define DMUB_FW_FEATURE_TRACE       (1ULL << 6)

// 固件版本检查函数
bool dmub_fw_check_version(struct dmub_srv *srv,
                            uint32_t required_version)
{
    if (srv->fw_version < required_version) {
        DRM_WARN("DMUB FW version %u < required %u, "
                 "some features may be unavailable\n",
                 srv->fw_version, required_version);
        return false;
    }
    return true;
}

// 固件特性检查函数
bool dmub_fw_has_feature(struct dmub_srv *srv,
                          uint64_t feature_bit)
{
    return (srv->fw->features_bitmap & feature_bit) != 0;
}
```

## 10. 自动化测试与调试工具

### 10.1 DMUB 压力测试工具

```python
"""
DMUB 固件通信压力测试
测试 DMCUB 在高负载下的响应能力
"""
import subprocess
import time
import threading
import argparse
from collections import defaultdict

class DMUBStressTester:
    def __init__(self, duration=300, rate=100):
        self.duration = duration
        self.target_rate = rate
        self.sent_count = 0
        self.error_count = 0
        self.latencies = []
        self.running = False

    def _send_psr_command(self, action):
        """通过 debugfs 发送 PSR 命令"""
        cmd = (f"echo {action} > "
               f"/sys/kernel/debug/dri/0/amdgpu_dm_psr/state")
        start = time.time()
        result = subprocess.run(
            cmd, shell=True, capture_output=True, text=True
        )
        latency = (time.time() - start) * 1000
        return result.returncode == 0, latency

    def _send_abm_command(self, level):
        """通过 debugfs 发送 ABM 命令"""
        cmd = (f"echo {level} > "
               f"/sys/kernel/debug/dri/0/amdgpu_dm_abm/level")
        start = time.time()
        result = subprocess.run(
            cmd, shell=True, capture_output=True, text=True
        )
        latency = (time.time() - start) * 1000
        return result.returncode == 0, latency

    def _worker_psr(self):
        """PSR 命令发送线程"""
        actions = ['enable', 'disable']
        while self.running:
            import random
            action = random.choice(actions)
            success, latency = self._send_psr_command(action)
            self.sent_count += 1
            if success:
                self.latencies.append(latency)
            else:
                self.error_count += 1

    def _worker_abm(self):
        """ABM 命令发送线程"""
        while self.running:
            import random
            level = random.randint(0, 4)
            success, latency = self._send_abm_command(level)
            self.sent_count += 1
            if success:
                self.latencies.append(latency)
            else:
                self.error_count += 1

    def run(self):
        self.running = True
        threads = []

        # 启动多个工作线程模拟并发
        for i in range(4):
            t = threading.Thread(
                target=self._worker_psr if i % 2 == 0
                else self._worker_abm
            )
            threads.append(t)
            t.start()

        time.sleep(self.duration)
        self.running = False

        for t in threads:
            t.join()

        self._report()

    def _report(self):
        print("DMUB Stress Test Report")
        print("=" * 40)
        print(f"Duration:      {self.duration}s")
        print(f"Total Commands: {self.sent_count}")
        print(f"Errors:         {self.error_count}")
        print(f"Success Rate:   "
              f"{(self.sent_count - self.error_count) / "
              f"self.sent_count * 100:.1f}%")

        if self.latencies:
            avg = sum(self.latencies) / len(self.latencies)
            print(f"Avg Latency:    {avg:.2f}ms")
            print(f"Max Latency:    {max(self.latencies):.2f}ms")
            print(f"Min Latency:    {min(self.latencies):.2f}ms")

        if self.error_count > 0:
            print("\nWARNING: Errors detected!")

### 10.2 DMUB 追踪分析脚本

```python
"""
DMUB 追踪数据分析工具
分析 DMCUB 固件输出的追踪事件序列
"""
import sys
from collections import Counter

class DMUBTraceAnalyzer:
    def __init__(self, trace_file):
        self.trace_file = trace_file
        self.events = []
        self.load()

    def load(self):
        with open(self.trace_file, 'r') as f:
            for line in f:
                if 'DMUB TRACE' in line:
                    self._parse_line(line)

    def _parse_line(self, line):
        import re
        match = re.search(
            r'event=0x([0-9A-F]+)\s+'
            r'file=(\S+)\s+'
            r'line=(\d+)\s+'
            r'ts=(\d+)',
            line
        )
        if match:
            self.events.append({
                'event_id': int(match.group(1), 16),
                'file': match.group(2),
                'line': int(match.group(3)),
                'timestamp': int(match.group(4)),
            })

    def analyze_psr_transitions(self):
        """分析 PSR 状态转换"""
        psr_entries = 0
        psr_exits = 0
        psr_durations = []
        last_entry_ts = None

        for event in self.events:
            if event['event_id'] == 0x100:  # PSR_ENTER
                psr_entries += 1
                last_entry_ts = event['timestamp']
            elif event['event_id'] == 0x101:  # PSR_EXIT
                psr_exits += 1
                if last_entry_ts:
                    duration = event['timestamp'] - last_entry_ts
                    psr_durations.append(duration)
                    last_entry_ts = None

        print("PSR Transition Analysis:")
        print(f"  Entries: {psr_entries}")
        print(f"  Exits:   {psr_exits}")
        if psr_durations:
            avg = sum(psr_durations) / len(psr_durations)
            print(f"  Avg Duration: {avg:.0f} cycles")
            print(f"  Min Duration: {min(psr_durations)} cycles")
            print(f"  Max Duration: {max(psr_durations)} cycles")

    def analyze_error_events(self):
        """分析错误事件"""
        errors = [e for e in self.events
                  if e['event_id'] == 0xFFF]
        if errors:
            print(f"Error Events: {len(errors)}")
            for err in errors:
                print(f"  [t={err['timestamp']}] {err['file']}:"
                      f"{err['line']}")
        else:
            print("No error events found")

    def event_distribution(self):
        """事件分布统计"""
        counter = Counter(e['event_id'] for e in self.events)
        print("Event Distribution:")
        for event_id, count in counter.most_common():
            name = {
                0x100: 'PSR_ENTER', 0x101: 'PSR_EXIT',
                0x200: 'ABM_UPDATE', 0x300: 'VRR_UPDATE',
                0x400: 'HPD_RX', 0x500: 'TIMING_SET',
            }.get(event_id, f'UNKNOWN(0x{event_id:03X})')
            print(f"  0x{event_id:03X} ({name}): {count}")

if __name__ == '__main__':
    analyzer = DMUBTraceAnalyzer(sys.argv[1])
    analyzer.analyze_psr_transitions()
    analyzer.analyze_error_events()
    analyzer.event_distribution()
```

## 11. 总结

本章深入介绍了 DMUB (Display Manager Microcontroller Unit B) 固件的架构与实现：

1. **DMUB 概述**：DMCUB 是集成在 GPU 显示引擎中的 ARC EM 微控制器，负责处理 PSR、ABM、HPD 等实时显示任务，将关键操作从主 CPU 卸载以降低延迟和功耗。

2. **dmub_srv 结构体**：驱动与 DMCUB 通信的核心数据结构，封装了固件管理、队列控制、状态跟踪、回调函数等功能。

3. **环形缓冲区通信**：使用无锁 ring buffer (flush_queue 和 notify_queue) 实现主机与 DMCUB 之间的高效双向通信，通过 doorbell 寄存器触发中断通知。

4. **命令系统**：定义了完整的命令枚举，包括时序控制 (SET_TIMING)、平面更新 (UPDATE_PLANE)、电源管理 (SET_POWER_STATE)、ABM、PSR、VRR、HPD、MST 等各类命令及其数据结构。

5. **PSR 集成**：DMCUB 固件内部实现 PSR 状态机，精确控制 PSR 进入/退出时序，支持多种退出原因追踪和调试统计。

6. **ABM 集成**：DMCUB 负责实时分析画面内容并动态调整背光亮度，支持渐变过渡和环境光补偿。

7. **状态管理**：完整的 DMUB 状态枚举和错误信息报告机制，以及系统挂起/恢复时的正确处理流程。

8. **调试与追踪**：固件内置追踪缓冲区，支持事件级调试，配合 debugfs 接口提供丰富的诊断信息。

9. **固件版本管理**：兼容性矩阵和特性位图机制，确保固件与硬件和驱动的匹配。

作为测试开发工程师，理解 DMUB 的工作原理对于调试 PSR 进入/退出问题、ABM 亮度异常、DPIA HPD 响应延迟等复杂问题至关重要。通过本章提供的压力测试和追踪分析工具，可以系统性地验证 DMUB 固件的稳定性和性能。