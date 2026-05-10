# Day 359: 音频透传：HDMI/DP 音频驱动集成

## 1. 音频透传概述

音频透传 (Audio Passthrough) 是指将音频数据从源设备（GPU）通过显示接口（HDMI/DP）传输到显示设备（显示器/音响）的过程。在 AMDGPU DC 子系统中，音频驱动集成涵盖了从 HDA (High Definition Audio) 控制器到显示引擎的完整链路。

### 1.1 音频架构层次

```
用户空间 (PulseAudio/ALSA/ PipeWire)
    │
    ▼
HDA 驱动层 (snd-hda-intel / snd-hda-codec)
    │  ── HDA 总线通信
    ▼
AMDGPU 音频驱动 (amdgpu_audio)
    │  ── GPU 编解码器控制
    ▼
DC 音频层 (DC Audio)
    │  ── 显示引擎流配置
    ▼
显示引擎
    │  ── 打包成 TMDS/FRL 流
    ▼
物理层 (HDMI/DP 发送器)
    │
    ▼
显示设备 (扬声器/音响)
```

### 1.2 音频流类型

```c
// 音频流类型
enum audio_stream_type {
    AUDIO_STREAM_PCM,           // 未压缩 PCM (2ch ~ 8ch)
    AUDIO_STREAM_AC3,           // Dolby Digital (AC-3)
    AUDIO_STREAM_EAC3,          // Dolby Digital Plus (E-AC-3)
    AUDIO_STREAM_TRUEHD,        // Dolby TrueHD (无损失)
    AUDIO_STREAM_DTS,           // DTS Core
    AUDIO_STREAM_DTS_HD,        // DTS-HD Master Audio
    AUDIO_STREAM_AAC,           // AAC
    AUDIO_STREAM_MPEG,          // MPEG Audio
    AUDIO_STREAM_DSD,           // Direct Stream Digital
    AUDIO_STREAM_MAT,           // MLP (Meridian Lossless Packing)
};

// 音频格式描述
struct audio_format {
    enum audio_stream_type stream_type;  // 流类型
    uint32_t sample_rate;                // 采样率 (Hz)
    uint32_t sample_size;                // 采样位深 (bits)
    uint32_t channels;                   // 声道数
    bool     is_passthrough;             // 是否透传模式
    uint32_t bitrate;                    // 比特率 (kbps, 压缩格式)
};
```

### 1.3 HDMI vs DP 音频对比

```
HDMI 音频特性:
  - 强制包含音频 (除非仅用于 EDID 读取)
  - 支持 ARC/eARC (音频回传通道)
  - 通过 TMDS 或 FRL 传输
  - 封装在 Audio Data Island Packet 中

DP 音频特性:
  - 可选音频 (显示数据通道中嵌入)
  - 通过 Main Link 传输
  - 使用 Secondary Data Packet (SDP) 封装
  - 天然支持 MST 多流音频

共同限制:
  - 最大 8 声道 (目前消费级标准)
  - 采样率范围: 32kHz ~ 192kHz
  - 位深: 16bit, 20bit, 24bit
```

## 2. 核心数据结构

### 2.1 音频端点与编解码器

```c
// 音频编解码器 (Codec) 结构
struct audio_codec {
    uint32_t codec_id;             // 编解码器 ID
    uint32_t vendor_id;            // 厂商 ID (0x1002 = AMD)
    uint32_t revision_id;          // 修订版本
    uint32_t subsystem_id;         // 子系统 ID

    // HDA 编解码器功能
    struct {
        uint32_t afg_id;           // Audio Function Group ID
        uint32_t widget_count;     // Widget 数量
        uint32_t default_pcm;      // 默认 PCM 引脚
        uint32_t digital_pcm;      // 数字 (HDMI/DP) PCM 引脚
    } hda;

    // 支持的音频格式 (位图)
    uint64_t pcm_supported;        // PCM 格式支持
    uint64_t non_pcm_supported;    // 非 PCM (压缩) 格式支持

    // 支持的最大参数
    struct {
        uint32_t max_channels;     // 最大声道数
        uint32_t max_sample_rate;  // 最大采样率
        uint32_t max_bits_per_sample; // 最大位深
    } caps;
};

// 音频端点 (Pin) 结构
struct audio_pin {
    uint32_t pin_id;               // Pin ID
    uint32_t hda_pin;              // HDA Pin Widget ID
    uint32_t hda_dev_entry;        // HDA 设备项

    // 连到哪个显示引擎
    uint32_t dig_id;               // DIG 引擎编号
    uint32_t afmt_id;              // AFMT 模块编号

    // 连接类型
    enum connector_type conn_type; // DP/HDMI

    // 是否 MST 端点
    bool is_mst_pin;
    uint32_t mst_stream_id;        // MST 流 ID

    // 当前活跃的音频流
    struct audio_stream *active_stream;
    struct list_head stream_list;  // 等待队列
};
```

### 2.2 音频流结构

```c
// 音频流
struct audio_stream {
    // 流标识
    uint32_t stream_id;
    uint32_t hda_stream;           // HDA Stream ID

    // 音频格式
    struct audio_format format;
    struct audio_format hw_format; // 硬件实际配置

    // 状态管理
    enum {
        AUDIO_STREAM_IDLE,         // 空闲
        AUDIO_STREAM_PREPARED,     // 已准备
        AUDIO_STREAM_RUNNING,      // 运行中
        AUDIO_STREAM_PAUSED,       // 暂停
        AUDIO_STREAM_STOPPED,      // 已停止
    } state;

    // 通道映射
    struct channel_map {
        uint8_t channels[8];       // FL, FR, FC, LFE, RL, RR, RLC, RRC
        bool     lfe_present;      // 是否有低音炮
    } ch_map;

    // 音量/静音控制
    struct {
        int32_t  volume[8];        // 各声道音量 (0..100)
        bool     mute;             // 静音
    } control;

    // 统计
    struct {
        uint64_t frames_played;    // 播放帧数
        uint64_t underruns;        // 下溢次数
        uint64_t errors;           // 错误计数
    } stats;

    // 关联的链路
    struct dc_link *link;
    struct audio_widget *widget;
};
```

### 2.3 DC 音频子系统

```c
// DC 音频子系统
struct dc_audio_subsystem {
    // 注册的编解码器
    struct audio_codec codec;
    struct audio_codec *mst_codecs[MAX_MST_PINS];

    // 音频端点列表
    struct audio_pin pins[MAX_PINS];
    uint32_t pin_count;

    // 活跃流列表
    struct audio_stream streams[MAX_AUDIO_STREAMS];
    uint32_t stream_count;

    // HDA 控制器接口
    struct hda_controller_ops *hda_ops;

    // 音频功能标志
    union {
        struct {
            uint32_t hdmi_audio   : 1;  // HDMI 音频支持
            uint32_t dp_audio     : 1;  // DP 音频支持
            uint32_t mst_audio    : 1;  // MST 音频支持
            uint32_t hbr_audio    : 1;  // HBR 音频支持
            uint32_t arc          : 1;  // ARC 支持
            uint32_t earc         : 1;  // eARC 支持
            uint32_t reserved     : 26;
        } flags;
        uint32_t raw;
    } caps;

    // 锁定
    struct mutex lock;
};
```

## 3. HDA (High Definition Audio) 集成

AMDGPU 使用内置的 HDA 编解码器与系统音频框架集成。

### 3.1 HDA 控制器接口

```c
// HDA 控制器操作
struct hda_controller_ops {
    // 启动/停止 HDA 流
    int (*stream_start)(struct audio_stream *stream);
    int (*stream_stop)(struct audio_stream *stream);
    int (*stream_reset)(struct audio_stream *stream);

    // 设置 HDA 流格式
    int (*stream_set_format)(struct audio_stream *stream,
                             struct audio_format *fmt);

    // 设置 HDA 通道映射
    int (*stream_set_channel_map)(struct audio_stream *stream,
                                  struct channel_map *map);

    // 控制音量/静音
    int (*set_volume)(struct audio_pin *pin,
                      int *volumes, int ch_count);
    int (*set_mute)(struct audio_pin *pin, bool mute);

    // 读取/写入编解码器寄存器
    int (*codec_read)(uint32_t codec_addr,
                      uint32_t reg, uint32_t *val);
    int (*codec_write)(uint32_t codec_addr,
                       uint32_t reg, uint32_t val);

    // 查询编解码器能力
    int (*codec_query_caps)(struct audio_codec *codec);
};

// HDA Stream Descriptor (SD) 寄存器
struct hda_sd_regs {
    uint32_t ctl;       // Control (0x00)
    uint32_t sts;       // Status (0x04)
    uint32_t lpib;      // Link Position In Buffer (0x08)
    uint32_t cbl;       // Cyclic Buffer Length (0x0C)
    uint32_t lvi;       // Last Valid Index (0x10)
    uint32_t fifod;     // FIFOD (0x14)
    uint32_t fmt;       // Format (0x18)
    uint32_t bdpl;      // Buffer Descriptor Pointer Low (0x1C)
    uint32_t bdpu;      // Buffer Descriptor Pointer Up (0x20)
    uint32_t reserved[6];
};

// HDA 流格式寄存器编码
#define HDA_FMT_CHANNEL_MASK   0xF000  // 声道数编码
#define HDA_FMT_BITS_MASK      0x0F00  // 位深编码
#define HDA_FMT_DIV_MASK       0x00F0  // 分频
#define HDA_FMT_MULT_MASK      0x000F  // 倍频

#define HDA_FMT_CHANNEL(n)     (((n) - 1) << 12)  // 声道 (1-16)
#define HDA_FMT_BITS(b)        ((b) << 8)          // 位深
#define HDA_FMT_RATE(r)        ((r) << 0)          // 采样率

// HDA 流启动流程
static int hda_audio_stream_start(struct audio_stream *stream)
{
    struct hda_sd_regs *sd = stream->hda_stream_regs;

    // 1. 停止流 (如果运行中)
    sd->ctl &= ~HDA_SD_CTL_RUN;
    udelay(50);

    // 2. 重置流
    sd->ctl |= HDA_SD_CTL_RESET;
    udelay(50);
    sd->ctl &= ~HDA_SD_CTL_RESET;
    udelay(50);

    // 3. 配置流格式
    sd->fmt = encode_hda_format(&stream->hw_format);

    // 4. 设置缓冲区描述符
    sd->cbl = stream->buf_desc.byte_count;
    sd->lvi = stream->buf_desc.num_entries - 1;
    sd->bdpl = lower_32_bits(stream->buf_desc.dma_addr);
    sd->bdpu = upper_32_bits(stream->buf_desc.dma_addr);

    // 5. 传输方向
    sd->ctl |= HDA_SD_CTL_DIR_OUT;  // 输出 (播放)

    // 6. 设置中断
    sd->ctl |= HDA_SD_CTL_IOC;     // Interrupt on Completion

    // 7. 启动流
    wmb();  // 确保所有配置写入生效
    sd->ctl |= HDA_SD_CTL_RUN | HDA_SD_CTL_DMA_START;

    DRM_DEBUG("HDA stream %u started: fmt=0x%08X\n",
              stream->hda_stream, sd->fmt);

    return 0;
}
```

### 3.2 编解码器探测与初始化

```c
// AMDGPU HDA 编解码器探测
static int amdgpu_audio_codec_probe(struct amdgpu_device *adev)
{
    struct dc_audio_subsystem *audio = adev->dm.audio;
    int ret;

    DRM_INFO("Probing AMDGPU audio codec\n");

    // 1. 读取编解码器 ID
    ret = audio->hda_ops->codec_read(HDA_CODEC_ADDR_PRIMARY,
                                      HDA_REG_VENDOR_ID,
                                      &audio->codec.vendor_id);
    if (ret) {
        DRM_ERROR("Failed to read codec vendor ID\n");
        return ret;
    }

    ret = audio->hda_ops->codec_read(HDA_CODEC_ADDR_PRIMARY,
                                      HDA_REG_REVISION_ID,
                                      &audio->codec.revision_id);
    if (ret) {
        DRM_ERROR("Failed to read codec revision ID\n");
        return ret;
    }

    DRM_INFO("AMD GPU audio codec: vendor=0x%04X rev=0x%04X\n",
             audio->codec.vendor_id >> 16,
             audio->codec.revision_id);

    // 2. 查询编解码器能力
    ret = audio->hda_ops->codec_query_caps(&audio->codec);
    if (ret) {
        DRM_ERROR("Failed to query codec caps\n");
        return ret;
    }

    DRM_INFO("  Max channels:   %u\n",
             audio->codec.caps.max_channels);
    DRM_INFO("  Max sample rate: %u Hz\n",
             audio->codec.caps.max_sample_rate);
    DRM_INFO("  Max bit depth:   %u bit\n",
             audio->codec.caps.max_bits_per_sample);

    // 3. 枚举音频端点 (Pin Widgets)
    ret = amdgpu_audio_enum_pins(adev);
    if (ret) {
        DRM_ERROR("Failed to enumerate audio pins\n");
        return ret;
    }

    DRM_INFO("Found %u audio pins\n", audio->pin_count);

    return 0;
}

// 枚举音频 Pin Widget
static int amdgpu_audio_enum_pins(struct amdgpu_device *adev)
{
    struct dc_audio_subsystem *audio = adev->dm.audio;
    struct audio_codec *codec = &audio->codec;
    uint32_t widget_id;
    uint32_t widget_type;
    int pin_count = 0;

    // 遍历所有 Widget
    for (widget_id = 0; widget_id < codec->hda.widget_count; widget_id++) {
        uint32_t reg_val;

        // 读取 Widget 类型
        audio->hda_ops->codec_read(HDA_CODEC_ADDR_PRIMARY,
                                    HDA_REG_WIDGET_CAPS(widget_id),
                                    &reg_val);

        widget_type = (reg_val >> 20) & 0x0F;

        // Pin Widget 类型为 0x04 (Audio Output) 或 0x07 (Pin Complex)
        if (widget_type == 0x07) {  // Pin Complex
            struct audio_pin *pin = &audio->pins[pin_count];

            pin->pin_id = pin_count;
            pin->hda_pin = widget_id;

            // 读取 Pin Capabilities
            audio->hda_ops->codec_read(HDA_CODEC_ADDR_PRIMARY,
                                        HDA_REG_PIN_CAPS(widget_id),
                                        &reg_val);

            // 判断连接器类型 (DP 或 HDMI)
            uint32_t conn_type = (reg_val >> 4) & 0x0F;
            if (conn_type == 0x00)
                pin->conn_type = CONNECTOR_TYPE_HDMI;
            else if (conn_type == 0x01)
                pin->conn_type = CONNECTOR_TYPE_DP;
            else
                continue;  // 跳过非显示音频 Pin

            DRM_DEBUG("Pin %d: Widget=0x%02X Type=%s\n",
                      pin_count, widget_id,
                      pin->conn_type == CONNECTOR_TYPE_HDMI ?
                          "HDMI" : "DP");

            // 读取默认配置 (Default Configuration)
            audio->hda_ops->codec_read(HDA_CODEC_ADDR_PRIMARY,
                                        HDA_REG_PIN_CONFIG(widget_id),
                                        &pin->hda_dev_entry);

            pin_count++;
        }
    }

    audio->pin_count = pin_count;
    return 0;
}
```

### 3.3 HDA 音频流生命周期

```c
// 音频流完整生命周期管理

// 1. 分配音频流
static struct audio_stream *
amdgpu_audio_stream_alloc(struct amdgpu_device *adev,
                           struct audio_pin *pin)
{
    struct dc_audio_subsystem *audio = adev->dm.audio;
    struct audio_stream *stream = NULL;

    mutex_lock(&audio->lock);

    // 查找空闲流
    for (int i = 0; i < MAX_AUDIO_STREAMS; i++) {
        if (audio->streams[i].state == AUDIO_STREAM_IDLE) {
            stream = &audio->streams[i];
            break;
        }
    }

    if (!stream) {
        DRM_ERROR("No available audio streams\n");
        mutex_unlock(&audio->lock);
        return NULL;
    }

    // 初始化流
    memset(stream, 0, sizeof(*stream));
    stream->stream_id = stream - audio->streams;
    stream->state = AUDIO_STREAM_IDLE;
    stream->hda_stream = pin->hda_dev_entry;
    stream->link = pin_get_link(pin);

    // Pin 关联到流
    pin->active_stream = stream;

    mutex_unlock(&audio->lock);

    DRM_DEBUG("Allocated audio stream %d on pin %d\n",
              stream->stream_id, pin->pin_id);

    return stream;
}

// 2. 准备音频流 (设置格式和缓冲区)
static int
amdgpu_audio_stream_prepare(struct audio_stream *stream,
                             struct audio_format *format)
{
    int ret;

    DRM_DEBUG("Preparing stream %d: %uch %uHz %ubit\n",
              stream->stream_id,
              format->channels, format->sample_rate,
              format->sample_size);

    // 保存格式
    stream->format = *format;

    // 1. 协商硬件格式 (考虑显示引擎限制)
    ret = negotiate_hw_format(stream, format);
    if (ret)
        return ret;

    // 2. 分配 DMA 缓冲区
    ret = alloc_audio_dma_buf(stream);
    if (ret) {
        DRM_ERROR("Failed to alloc DMA buffer for stream %d\n",
                  stream->stream_id);
        return ret;
    }

    // 3. 设置 HDA 流格式
    ret = stream->link->audio->hda_ops->stream_set_format(
        stream, &stream->hw_format);
    if (ret)
        goto err_free_dma;

    // 4. 设置通道映射
    struct channel_map default_map = {
        .channels = {0, 1, 2, 3, 4, 5, 6, 7},
        .lfe_present = true,
    };
    ret = stream->link->audio->hda_ops->stream_set_channel_map(
        stream, &default_map);
    if (ret)
        goto err_free_dma;

    stream->state = AUDIO_STREAM_PREPARED;

    return 0;

err_free_dma:
    free_audio_dma_buf(stream);
    return ret;
}

// 3. 启动音频流
static int
amdgpu_audio_stream_start(struct audio_stream *stream)
{
    int ret;

    DRM_DEBUG("Starting audio stream %d\n", stream->stream_id);

    // 1. 配置显示引擎音频信息 (InfoFrame/SDP)
    ret = configure_display_audio(stream);
    if (ret) {
        DRM_ERROR("Failed to configure display audio\n");
        return ret;
    }

    // 2. 启动 HDA 流
    ret = stream->link->audio->hda_ops->stream_start(stream);
    if (ret) {
        DRM_ERROR("Failed to start HDA stream\n");
        return ret;
    }

    stream->state = AUDIO_STREAM_RUNNING;

    DRM_DEBUG("Audio stream %d started successfully\n",
              stream->stream_id);

    return 0;
}

// 4. 停止并释放音频流
static int
amdgpu_audio_stream_stop(struct audio_stream *stream)
{
    struct dc_audio_subsystem *audio;

    DRM_DEBUG("Stopping audio stream %d\n", stream->stream_id);

    // 1. 停止 HDA 流
    audio = stream->link->audio;
    audio->hda_ops->stream_stop(stream);

    // 2. 清除显示引擎音频配置
    clear_display_audio(stream);

    // 3. 释放 DMA 缓冲区
    free_audio_dma_buf(stream);

    // 4. 清除 Pin 关联
    struct audio_pin *pin = find_pin_by_stream(stream);
    if (pin)
        pin->active_stream = NULL;

    // 5. 重置流状态
    stream->state = AUDIO_STREAM_IDLE;
    stream->format = (struct audio_format){0};

    return 0;
}
```

## 4. 显示引擎音频配置

音频流需要经过显示引擎打包后才能传输。不同的接口（HDMI/DP）使用不同的数据包格式。

### 4.1 通用音频配置接口

```c
// 显示引擎音频配置
struct display_audio_config {
    // 音频编码格式
    enum audio_stream_type coding_type;

    // 声道数及映射
    uint8_t channel_count;
    uint8_t channel_allocation;  // CA 值 (Channel Allocation)

    // 采样率编码
    uint8_t sample_rate_code;    // 3-bit 编码

    // 位深
    uint8_t sample_size_code;    // 2-bit 编码

    // 原始 PCM 标志
    bool is_pcm;

    // InfoFrame / SDP 数据
    union {
        uint8_t hdmi_aud_infoframe[HDMI_INFOFRAME_SIZE];
        uint8_t dp_audio_sdp[DP_SDP_SIZE];
    } packet;

    // 音频 CTS/N 值 (HDMI 时钟恢复)
    struct {
        uint32_t n;              // N 值
        uint32_t cts;            // CTS 值
    } hdmi_ncts;
};

// 音频 CTS/N 计算表
static const struct hdmi_audio_n_cts {
    uint32_t sample_rate;
    uint32_t tmds_clock_khz;
    uint32_t n;
    uint32_t cts;
} hdmi_n_cts_table[] = {
    // 32kHz
    { 32000,   25175,   4576,  28125  },
    { 32000,   27000,   4096,  25200  },
    { 32000,   27027,   4096,  25200  },  // 27.027MHz
    { 32000,   54000,   4096,  50400  },
    { 32000,   54054,   4096,  50400  },  // 54.054MHz
    { 32000,   74250,   4096,  69300  },
    { 32000,  148500,   4096,  138600 },

    // 44.1kHz
    { 44100,   25175,   7007,  31250  },
    { 44100,   27000,   6272,  28000  },
    { 44100,   27027,   6272,  28000  },
    { 44100,   54000,   6272,  56000  },
    { 44100,   54054,   6272,  56000  },
    { 44100,   74250,   4704,  39200  },
    { 44100,  148500,   4704,  78400  },

    // 48kHz
    { 48000,   25175,   6144,  31250  },
    { 48000,   27000,   6144,  27000  },
    { 48000,   27027,   6144,  27027  },
    { 48000,   54000,   6144,  54000  },
    { 48000,   54054,   6144,  54054  },
    { 48000,   74250,   6144,  74250  },
    { 48000,  148500,   6144, 148500  },

    // 176.4kHz (HBR)
    { 176400,  27000,   6272,  7000   },
    { 176400,  54000,   6272,  14000  },
    { 176400,  74250,   4704,  9800   },
    { 176400, 148500,   4704,  19600  },

    // 192kHz (HBR)
    { 192000,  27000,   6144,  6750   },
    { 192000,  54000,   6144,  13500  },
    { 192000,  74250,   6144,  18563  },
    { 192000, 148500,   6144,  37125  },
};

// 计算 HDMI CTS/N 值
static int hdmi_calc_n_cts(uint32_t sample_rate,
                            uint32_t tmds_clock_khz,
                            uint32_t *out_n,
                            uint32_t *out_cts)
{
    for (int i = 0; i < ARRAY_SIZE(hdmi_n_cts_table); i++) {
        if (hdmi_n_cts_table[i].sample_rate == sample_rate &&
            hdmi_n_cts_table[i].tmds_clock_khz == tmds_clock_khz) {
            *out_n = hdmi_n_cts_table[i].n;
            *out_cts = hdmi_n_cts_table[i].cts;
            return 0;
        }
    }

    DRM_ERROR("No N/CTS found for %uHz @ %u.%03uMHz\n",
              sample_rate,
              tmds_clock_khz / 1000,
              tmds_clock_khz % 1000);

    return -EINVAL;
}
```

### 4.2 HDMI 音频 InfoFrame

```c
// HDMI Audio InfoFrame 结构
struct hdmi_audio_infoframe {
    uint8_t header[3];           // 0x84, 0x01, 0x0A

    // Byte 3: CT (Coding Type)
    union {
        struct {
            uint8_t ct     : 4;  // Coding Type
            uint8_t res1   : 3;
            uint8_t cc     : 3;  // Channel Count
        } __packed;
        uint8_t raw;
    } byte3;

    // Byte 4: CA / LFEPBL
    union {
        struct {
            uint8_t ca     : 7;  // Channel Allocation
            uint8_t lfe    : 1;  // LFE Playback Level
        } __packed;
        uint8_t raw;
    } byte4;

    // Byte 5: LFEPBL / LS / DM_INH
    union {
        struct {
            uint8_t dm_inh : 1;  // Downmix Inhibit
            uint8_t lsv    : 3;  // Level Shift Value
            uint8_t res2   : 4;
        } __packed;
        uint8_t raw;
    } byte5;

    // Byte 6-10: 厂商特定 / 保留
    uint8_t vendor_specific[5];

    // Byte 11-13: 保留 (End Byte = 0x00)
    uint8_t reserved[3];
} __packed;

// 构建 HDMI Audio InfoFrame
static void build_hdmi_audio_infoframe(
    struct hdmi_audio_infoframe *frame,
    struct display_audio_config *config)
{
    // 1. Header
    frame->header[0] = 0x84;  // InfoFrame Type: Audio
    frame->header[1] = 0x01;  // Version
    frame->header[2] = 0x0A;  // Length (10 bytes)

    // 2. Coding Type & Channel Count
    frame->byte3.ct = config->coding_type;
    frame->byte3.cc = config->channel_count - 1;

    // 3. Channel Allocation
    frame->byte4.ca = calc_hdmi_channel_allocation(
        config->channel_count);
    frame->byte4.lfe = 0;  // 0dB

    // 4. Downmix Inhibit & Level Shift
    frame->byte5.dm_inh = 0;  // 允许 Downmix
    frame->byte5.lsv = 0;     // 无电平偏移

    // 5. 保留清零
    memset(frame->vendor_specific, 0, sizeof(frame->vendor_specific));
    memset(frame->reserved, 0, sizeof(frame->reserved));
}

// 计算 HDMI Channel Allocation 值
static uint8_t calc_hdmi_channel_allocation(uint8_t channels)
{
    // CA 值根据声道数计算
    // 0x00 = FL/FR (2.0)
    // 0x01 = FL/FR/LFE (2.1)
    // 0x03 = FL/FR/LFE/FC (3.1)
    // 0x07 = FL/FR/FC/LFE/RL/RR (5.1)
    // 0x0B = FL/FR/FC/LFE/RL/RR/RLC/RRC (7.1)
    // 0x13 = FL/FR/FC/LFE/RL/RR/FLC/FRC (7.1 替代)

    static const uint8_t ca_map[] = {
        [2] = 0x00,  // 2.0
        [3] = 0x01,  // 3.0 (2.1)
        [4] = 0x03,  // 4.0 (3.1)
        [6] = 0x07,  // 6.0 (5.1)
        [8] = 0x0B,  // 8.0 (7.1)
    };

    if (channels > ARRAY_SIZE(ca_map))
        return 0x00;

    return ca_map[channels];
}
```

### 4.3 DP 音频 SDP

```c
// DP Audio SDP (Secondary Data Packet)
struct dp_audio_sdp {
    // Header (2 bytes)
    struct {
        uint8_t hb0;  // SDP Type = 0x02 (Audio)
        uint8_t hb1;  // 保留
    } header;

    // Sub-header (4 bytes)
    struct {
        uint8_t sb0;  // Audio Stream Type
        uint8_t sb1;  // Number of Channels - 1
        uint8_t sb2;  // Sample Rate Code
        uint8_t sb3;  // Sample Size Code
    } subheader;

    // Params (32 bytes)
    union {
        struct {
            // Audio InfoFrame Data
            uint8_t aud_infoframe_data[10];

            // Reserved
            uint8_t reserved[22];
        };
        uint8_t raw[32];
    } params;

    // 总大小: 2 + 4 + 32 = 38 bytes
    // 实际传输中 padding 到 64 bytes
} __packed;

// 构建 DP Audio SDP
static void build_dp_audio_sdp(struct dp_audio_sdp *sdp,
                                struct display_audio_config *config)
{
    // Header
    sdp->header.hb0 = 0x02;  // Audio Stream SDP
    sdp->header.hb1 = 0x00;

    // Sub-header
    sdp->subheader.sb0 = config->coding_type;
    sdp->subheader.sb1 = config->channel_count - 1;
    sdp->subheader.sb2 = config->sample_rate_code;
    sdp->subheader.sb3 = config->sample_size_code;

    // Audio InfoFrame (部分字段)
    sdp->params.aud_infoframe_data[0] =
        config->channel_allocation;
    sdp->params.aud_infoframe_data[1] =
        (config->sample_size_code << 4) | 0x01;
    sdp->params.aud_infoframe_data[2] =
        config->sample_rate_code;
}

// DP 音频流类型编码
enum dp_audio_stream_type {
    DP_AUDIO_STREAM_PCM     = 0x00,
    DP_AUDIO_STREAM_AC3     = 0x01,
    DP_AUDIO_STREAM_DTS     = 0x02,
    DP_AUDIO_STREAM_AAC     = 0x03,
    DP_AUDIO_STREAM_EAC3    = 0x04,
    DP_AUDIO_STREAM_TRUEHD  = 0x05,
    DP_AUDIO_STREAM_DTS_HD  = 0x06,
    DP_AUDIO_STREAM_MAT     = 0x07,
    DP_AUDIO_STREAM_DSD     = 0x08,
};

// DP 采样率编码
static const struct {
    uint32_t rate_hz;
    uint8_t  code;
} dp_sample_rate_map[] = {
    { 32000,  0x01 },
    { 44100,  0x02 },
    { 48000,  0x03 },
    { 88200,  0x04 },
    { 96000,  0x05 },
    { 176400, 0x06 },
    { 192000, 0x07 },
    { 0,      0x00 },
};

// DP 位深编码
static const struct {
    uint32_t bits;
    uint8_t  code;
} dp_sample_size_map[] = {
    { 16, 0x01 },
    { 20, 0x02 },
    { 24, 0x03 },
    { 0,  0x00 },
};
```

### 4.4 显示引擎寄存器配置

```c
// AFMT (Audio Format) 模块寄存器
struct afmt_regs {
    uint32_t afmt_info0;     // Audio InfoFrame 0
    uint32_t afmt_info1;     // Audio InfoFrame 1
    uint32_t afmt_ctl;       // Audio Control
    uint32_t afmt_cntl;      // Audio Channel Control
    uint32_t afmt_608_hdmi;  // Audio 608/HDMI
    uint32_t afmt_609_hdmi;  // Audio 609/HDMI
    uint32_t afmt_audpacket; // Audio Packet Control
    uint32_t afmt_vbi;       // Audio VBI
};

// 配置 AFMT 模块
static void configure_afmt(struct afmt_regs *afmt,
                            struct display_audio_config *cfg)
{
    // 1. 配置音频控制寄存器
    uint32_t ctl = 0;
    ctl |= REG_SET(AFMT_CTL__PACKET_TYPE, cfg->coding_type);
    ctl |= REG_SET(AFMT_CTL__SAMPLE_RATE, cfg->sample_rate_code);
    REG_WRITE(&afmt->afmt_ctl, ctl);

    // 2. 配置音频通道控制
    uint32_t cntl = 0;
    cntl |= REG_SET(AFMT_CNTL__CHANNEL_COUNT,
                    cfg->channel_count - 1);
    cntl |= REG_SET(AFMT_CNTL__CHANNEL_ALLOCATION,
                    cfg->channel_allocation);
    cntl |= REG_SET(AFMT_CNTL__SAMPLE_SIZE, cfg->sample_size_code);
    REG_WRITE(&afmt->afmt_cntl, cntl);

    // 3. 配置音频数据包
    uint32_t pkt = REG_SET(AFMT_AUDPACKET__AUDIO_PACKET_ENABLE, 1);
    if (cfg->is_pcm)
        pkt |= AFMT_AUDPACKET__PCM_ONLY;
    REG_WRITE(&afmt->afmt_audpacket, pkt);

    DRM_DEBUG("AFMT configured: ctl=0x%08X cntl=0x%08X\n",
              ctl, cntl);
}

// 配置显示引擎音频 (入口函数)
static int configure_display_audio(struct audio_stream *stream)
{
    struct dc_link *link = stream->link;
    struct display_audio_config cfg = {0};
    int ret;

    // 1. 构建音频配置
    cfg.channel_count = stream->hw_format.channels;
    cfg.sample_rate_code = encode_sample_rate(
        stream->hw_format.sample_rate);
    cfg.sample_size_code = encode_sample_size(
        stream->hw_format.sample_size);
    cfg.coding_type = stream->hw_format.stream_type;
    cfg.is_pcm = (stream->hw_format.stream_type ==
                  AUDIO_STREAM_PCM);
    cfg.channel_allocation = calc_hdmi_channel_allocation(
        cfg.channel_count);

    // 2. 根据连接类型配置
    if (link->connector_signal & SIGNAL_TYPE_HDMI) {
        // HDMI: 配置 InfoFrame + CTS/N
        struct hdmi_audio_infoframe *frame =
            (struct hdmi_audio_infoframe *)cfg.packet.hdmi_aud_infoframe;

        build_hdmi_audio_infoframe(frame, &cfg);

        // 计算 CTS/N
        ret = hdmi_calc_n_cts(
            stream->hw_format.sample_rate,
            link->cur_link_settings.tmds_clock_khz / 1000,
            &cfg.hdmi_ncts.n,
            &cfg.hdmi_ncts.cts);
        if (ret)
            return ret;

        // 写入 HDMI 寄存器
        write_hdmi_audio_infoframe(link, frame);
        write_hdmi_ncts(link,
                         cfg.hdmi_ncts.n,
                         cfg.hdmi_ncts.cts);

    } else if (link->connector_signal & SIGNAL_TYPE_DISPLAY_PORT) {
        // DP: 配置 Audio SDP
        struct dp_audio_sdp sdp;

        build_dp_audio_sdp(&sdp, &cfg);

        // 写入 DP 寄存器
        write_dp_audio_sdp(link, &sdp);
    }

    // 3. 配置 AFMT 模块
    struct afmt_regs *afmt = get_afmt_regs(link, stream->afmt_id);
    configure_afmt(afmt, &cfg);

    return 0;
}

// 清除显示引擎音频配置
static void clear_display_audio(struct audio_stream *stream)
{
    struct afmt_regs *afmt = get_afmt_regs(
        stream->link, stream->afmt_id);

    // 禁用音频数据包
    REG_WRITE(&afmt->afmt_audpacket, 0);

    // 清除 InfoFrame / SDP
    if (stream->link->connector_signal & SIGNAL_TYPE_HDMI) {
        clear_hdmi_audio_infoframe(stream->link);
    } else {
        clear_dp_audio_sdp(stream->link);
    }

    DRM_DEBUG("Display audio cleared for stream %d\n",
              stream->stream_id);
}
```

## 5. MST 音频处理

MST (Multi-Stream Transport) 允许在单个 DP 链路上传输多个音视频流。

### 5.1 MST 音频架构

```c
// MST 音频端点
struct mst_audio_pin {
    struct audio_pin base;       // 基础 Pin
    uint32_t vc_payload_id;      // VC Payload ID
    uint32_t stream_index;       // MST 流索引

    // MST 特有的属性
    struct {
        bool is_virtual;         // 虚拟 Pin (非物理)
        bool stream_active;      // 流是否活跃
        uint32_t peer_gpu;       // 对端 GPU (多 GPU 场景)
    } mst;
};

// MST 音频流映射
struct mst_audio_stream_map {
    struct dc_link *link;                // 物理链路
    struct mst_audio_pin *mst_pins[MAX_MST_PINS];
    uint32_t mst_pin_count;

    // VCPI (Virtual Channel Payload Identifier) 映射
    struct {
        uint32_t vcpi;
        struct mst_audio_pin *pin;
    } vcpi_map[MAX_MST_PINS];
};

// MST 音频初始化
static int mst_audio_init(struct dc_link *link)
{
    struct mst_audio_stream_map *map;
    uint8_t dpcd_mstm_ctrl;

    // 1. 检查是否支持 MST
    dpcd_read(link, DPCD_ADDR_MSTM_CTRL,
              &dpcd_mstm_ctrl, 1);

    if (!(dpcd_mstm_ctrl & DPCD_MSTM_ENABLE)) {
        DRM_DEBUG("MST not enabled on link %d\n",
                  link->link_id);
        return 0;
    }

    // 2. 分配 MST 音频映射
    map = kzalloc(sizeof(*map), GFP_KERNEL);
    if (!map)
        return -ENOMEM;

    map->link = link;
    link->mst_audio_map = map;

    // 3. 注册 MST 音频端点
    for (int i = 0; i < link->mst_stream_count; i++) {
        struct mst_audio_pin *mst_pin;

        mst_pin = kzalloc(sizeof(*mst_pin), GFP_KERNEL);
        if (!mst_pin)
            continue;

        mst_pin->base.pin_id = link->pin_count + i;
        mst_pin->base.conn_type = CONNECTOR_TYPE_DP;
        mst_pin->base.is_mst_pin = true;
        mst_pin->base.dig_id = link->dig_id;
        mst_pin->stream_index = i;
        mst_pin->vc_payload_id = link->vcpi[i];

        map->mst_pins[map->mst_pin_count++] = mst_pin;
        map->vcpi_map[mst_pin->vc_payload_id].vcpi =
            mst_pin->vc_payload_id;
        map->vcpi_map[mst_pin->vc_payload_id].pin = mst_pin;

        DRM_DEBUG("MST audio pin %d created (VCPI=%u)\n",
                  mst_pin->base.pin_id,
                  mst_pin->vc_payload_id);
    }

    return 0;
}
```

### 5.2 MST 音频流分配

```c
// MST 音频流分配 (VC Payload 分配时)
static int mst_audio_assign_stream(struct dc_link *link,
                                    uint32_t vcpi)
{
    struct mst_audio_stream_map *map = link->mst_audio_map;
    struct mst_audio_pin *mst_pin = NULL;

    // 1. 根据 VCPI 找到对应的 MST Pin
    for (int i = 0; i < map->mst_pin_count; i++) {
        if (map->mst_pins[i]->vc_payload_id == vcpi) {
            mst_pin = map->mst_pins[i];
            break;
        }
    }

    if (!mst_pin) {
        DRM_ERROR("No MST pin found for VCPI %u\n", vcpi);
        return -EINVAL;
    }

    // 2. 创建音频流
    struct audio_stream *stream = amdgpu_audio_stream_alloc(
        link->adev, &mst_pin->base);
    if (!stream)
        return -ENOMEM;

    // 3. MST 特有的配置
    stream->mst_vcpi = vcpi;
    mst_pin->mst.stream_active = true;

    // 4. 配置 MST 音频路由
    configure_mst_audio_routing(link, vcpi, stream);

    DRM_DEBUG("MST audio stream assigned: VCPI=%u stream=%d\n",
              vcpi, stream->stream_id);

    return 0;
}

// MST 音频流释放 (VC Payload 释放时)
static int mst_audio_free_stream(struct dc_link *link,
                                  uint32_t vcpi)
{
    struct mst_audio_stream_map *map = link->mst_audio_map;
    struct audio_stream *stream = NULL;

    // 查找对应流
    for (int i = 0; i < link->adev->dm.audio->stream_count; i++) {
        if (link->adev->dm.audio->streams[i].mst_vcpi == vcpi) {
            stream = &link->adev->dm.audio->streams[i];
            break;
        }
    }

    if (!stream) {
        DRM_DEBUG("No stream found for VCPI %u\n", vcpi);
        return 0;
    }

    // 停止并释放
    amdgpu_audio_stream_stop(stream);

    // 清除 MST 标记
    if (stream->pin) {
        struct mst_audio_pin *mpin =
            container_of(stream->pin, struct mst_audio_pin, base);
        mpin->mst.stream_active = false;
    }

    return 0;
}

// 配置 MST 音频路由
static void configure_mst_audio_routing(struct dc_link *link,
                                         uint32_t vcpi,
                                         struct audio_stream *stream)
{
    // MST 音频路由通过 DIG 的 AFMT 模块配置
    // 每个 MST 流拥有独立的音频 SDP

    struct display_audio_config cfg = {0};

    // 基本配置
    cfg.channel_count = 2;
    cfg.coding_type = AUDIO_STREAM_PCM;
    cfg.is_pcm = true;

    // 对于 MST，SDP 中包含 VCPI 标识
    struct dp_audio_sdp sdp;
    build_dp_audio_sdp(&sdp, &cfg);

    // MST 特有的字段: 音频流标识
    sdp.params.aud_infoframe_data[3] = vcpi;

    // 写入 DP 辅助通道或 MAIN_LINK 寄存器
    write_dp_audio_sdp_mst(link, vcpi, &sdp);
}
```

## 6. 音频格式协商

音频格式协商是驱动根据源端能力和显示端 EDID 能力确定最终音频格式的过程。

### 6.1 EDID 音频能力解析

```c
// EDID 音频数据块 (CEA-861 短音频描述符)
struct cea_audio_data_block {
    uint8_t length;                // 数据块长度
    uint8_t format_count;          // 格式数量

    struct short_audio_desc {
        uint8_t format_code  : 7;  // Audio Format Code (1=PCM, 2=AC3, ...)
        uint8_t max_channel  : 3;  // 最大声道数 - 1
        uint8_t sample_rate  : 3;  // 支持的采样率 (位图)
        uint8_t sample_size  : 2;  // 支持的位深 (位图, 仅 PCM)
        uint8_t max_bitrate;       // 最大比特率 (非 PCM)
    } __packed formats[];
};

// 解析 EDID 音频数据块
static int parse_edid_audio_blocks(struct dc_sink *sink,
                                    struct audio_codec *codec)
{
    struct edid *edid = sink->edid_caps.edid_buf;
    int audio_formats = 0;

    DRM_DEBUG("Parsing EDID audio data blocks\n");

    // 遍历 EDID 扩展块
    for (int blk = 0; blk < edid->extensions; blk++) {
        struct cea_audio_data_block *audio_blk;
        uint8_t *cea_data;

        cea_data = get_cea_data_block(edid, blk,
                                       CEA_DATA_BLOCK_AUDIO);
        if (!cea_data)
            continue;

        audio_blk = (struct cea_audio_data_block *)cea_data;

        // 遍历短音频描述符
        for (int i = 0;
             i < audio_blk->format_count &&
             audio_formats < MAX_AUDIO_FORMATS;
             i++) {

            struct short_audio_desc *desc =
                &audio_blk->formats[i];

            uint8_t fmt_code = desc->format_code;
            uint8_t channels = (desc->max_channel & 0x07) + 1;
            uint8_t rates = desc->sample_rate & 0x07;
            uint8_t sizes = desc->sample_size & 0x03;

            DRM_DEBUG("  Format %d: code=%u ch=%u rates=0x%X\n",
                      audio_formats, fmt_code, channels, rates);

            // 记录到编解码器能力
            codec->pcm_supported |= (fmt_code == 1) ?
                (rates << 16) | (sizes << 8) | channels : 0;
            codec->non_pcm_supported |= (fmt_code > 1) ?
                (1 << (fmt_code - 2)) : 0;

            audio_formats++;
        }
    }

    DRM_DEBUG("Found %d audio formats in EDID\n", audio_formats);
    return audio_formats;
}

// 音频格式协商
static int negotiate_hw_format(struct audio_stream *stream,
                                struct audio_format *request)
{
    struct dc_link *link = stream->link;
    struct audio_codec *codec = &link->adev->dm.audio->codec;
    struct dc_sink *sink = link->local_sink;

    DRM_DEBUG("Negotiating audio format for stream %d\n",
              stream->stream_id);

    // 1. 检查 sink 是否存在
    if (!sink || !sink->edid_caps.is_valid) {
        DRM_WARN("No valid sink, using default format\n");
        stream->hw_format = *request;
        return 0;
    }

    // 2. 声道数限制
    stream->hw_format.channels = min(
        request->channels,
        (uint32_t)codec->caps.max_channels);

    // 根据 EDID 限制声道数
    uint8_t edid_max_ch = (codec->pcm_supported & 0xFF);
    stream->hw_format.channels = min(
        stream->hw_format.channels,
        (uint32_t)edid_max_ch);

    // 3. 采样率限制
    // 寻找最接近的向下兼容采样率
    static const uint32_t valid_rates[] = {
        192000, 176400, 96000, 88200, 48000, 44100, 32000
    };

    stream->hw_format.sample_rate = request->sample_rate;
    for (int i = 0; i < ARRAY_SIZE(valid_rates); i++) {
        if (request->sample_rate >= valid_rates[i] &&
            (codec->caps.max_sample_rate >= valid_rates[i])) {
            stream->hw_format.sample_rate = valid_rates[i];
            break;
        }
    }

    // 4. 位深限制
    stream->hw_format.sample_size = min(
        request->sample_size,
        (uint32_t)codec->caps.max_bits_per_sample);

    // 如果 PPS (Pixel Precise SDP) 支持，可以向上取整
    stream->hw_format.sample_size = min(
        stream->hw_format.sample_size, (uint32_t)24);

    // 5. 检查透传格式支持
    if (!request->is_passthrough) {
        stream->hw_format.stream_type = AUDIO_STREAM_PCM;
        stream->hw_format.is_passthrough = false;
    } else {
        // 检查非 PCM 格式支持
        uint32_t ncpcm_bit = 1 << (request->stream_type - 2);
        if (codec->non_pcm_supported & ncpcm_bit) {
            stream->hw_format = *request;
            stream->hw_format.is_passthrough = true;
        } else {
            DRM_WARN("Non-PCM format %d not supported, "
                     "falling back to PCM\n",
                     request->stream_type);
            stream->hw_format.stream_type = AUDIO_STREAM_PCM;
            stream->hw_format.is_passthrough = false;
        }
    }

    DRM_DEBUG("Negotiated format: %uch %uHz %ubit %s\n",
              stream->hw_format.channels,
              stream->hw_format.sample_rate,
              stream->hw_format.sample_size,
              stream->hw_format.is_passthrough ?
                  "passthrough" : "PCM");

    return 0;
}
```

### 6.2 音频格式编码/解码

```c
// 采样率编码 (HDMI/DP 寄存器格式)
static uint8_t encode_sample_rate(uint32_t rate_hz)
{
    for (int i = 0; i < ARRAY_SIZE(dp_sample_rate_map); i++) {
        if (dp_sample_rate_map[i].rate_hz == rate_hz)
            return dp_sample_rate_map[i].code;
    }
    return 0x03;  // 默认 48kHz
}

// 位深编码
static uint8_t encode_sample_size(uint32_t bits)
{
    for (int i = 0; i < ARRAY_SIZE(dp_sample_size_map); i++) {
        if (dp_sample_size_map[i].bits == bits)
            return dp_sample_size_map[i].code;
    }
    return 0x01;  // 默认 16bit
}

// 解码采样率
static uint32_t decode_sample_rate(uint8_t code)
{
    for (int i = 0; i < ARRAY_SIZE(dp_sample_rate_map); i++) {
        if (dp_sample_rate_map[i].code == code)
            return dp_sample_rate_map[i].rate_hz;
    }
    return 48000;
}

// 音频格式能力集
struct audio_caps {
    uint32_t rates;     // 采样率位图
    uint32_t sizes;     // 位深位图
    uint32_t channels;  // 声道数
};

// 合并两个能力集 (取交集)
static struct audio_caps audio_caps_intersect(
    struct audio_caps *a, struct audio_caps *b)
{
    struct audio_caps result = {
        .rates    = a->rates & b->rates,
        .sizes    = a->sizes & b->sizes,
        .channels = min(a->channels, b->channels),
    };
    return result;
}

// 能力集转可读字符串
static void audio_caps_to_string(struct audio_caps *caps,
                                  char *buf, size_t buf_size)
{
    char rates_str[64] = {0};
    char sizes_str[32] = {0};

    static const struct {
        uint32_t bit;
        const char *name;
    } rate_names[] = {
        { 0x01, "32kHz"  },
        { 0x02, "44.1kHz"},
        { 0x04, "48kHz"  },
        { 0x08, "88.2kHz"},
        { 0x10, "96kHz"  },
        { 0x20, "176.4kHz"},
        { 0x40, "192kHz" },
    };

    for (int i = 0; i < ARRAY_SIZE(rate_names); i++) {
        if (caps->rates & rate_names[i].bit) {
            if (strlen(rates_str) > 0)
                strcat(rates_str, "/");
            strcat(rates_str, rate_names[i].name);
        }
    }

    if (caps->sizes & 0x01) strcat(sizes_str, "16 ");
    if (caps->sizes & 0x02) strcat(sizes_str, "20 ");
    if (caps->sizes & 0x01) strcat(sizes_str, "24 ");

    snprintf(buf, buf_size, "%uch %s %sbit",
             caps->channels, rates_str, sizes_str);
}
```

## 7. 音频 DMA 与缓冲区管理

### 7.1 音频 DMA 缓冲区

```c
// 音频 DMA 缓冲区描述符
struct audio_dma_buf {
    // DMA 缓冲区参数
    uint32_t buf_size;          // 总大小 (bytes)
    uint32_t period_size;       // 周期大小 (bytes)
    uint32_t period_count;      // 周期数

    // DMA 地址
    dma_addr_t dma_addr;        // DMA 总线地址
    void       *cpu_addr;       // CPU 虚拟地址

    // 缓冲区描述符表 (BDL - Buffer Descriptor List)
    struct hda_bdl_entry {
        dma_addr_t addr;        // 段地址
        uint32_t   length;      // 段长度
        uint32_t   ioc;         // Interrupt on Completion
        uint32_t   reserved;
    } __packed *bdl;
    uint32_t bdl_entries;       // BDL 条目数
};

// 分配音频 DMA 缓冲区
static int alloc_audio_dma_buf(struct audio_stream *stream)
{
    struct device *dev = stream->link->adev->dev;
    struct audio_dma_buf *buf;
    int ret;

    buf = kzalloc(sizeof(*buf), GFP_KERNEL);
    if (!buf)
        return -ENOMEM;

    // 计算缓冲区大小
    // period_size = 帧大小 * 每周期帧数
    // 例如: 48kHz, 2ch, 16bit -> 帧大小 = 4 bytes
    //       period_size = 4 * 1024 = 4096 bytes
    uint32_t frame_size = (stream->hw_format.sample_size / 8) *
                           stream->hw_format.channels;
    buf->period_size = frame_size * 1024;  // 1024 帧/周期
    buf->period_count = 4;                  // 4 个周期
    buf->buf_size = buf->period_size *
                    buf->period_count;

    // 分配 DMA 缓冲区
    buf->cpu_addr = dma_alloc_coherent(dev,
                                        buf->buf_size,
                                        &buf->dma_addr,
                                        GFP_KERNEL);
    if (!buf->cpu_addr) {
        DRM_ERROR("Failed to alloc DMA buffer (%u bytes)\n",
                  buf->buf_size);
        ret = -ENOMEM;
        goto err_free_buf;
    }

    // 分配 BDL (每个周期一个条目)
    buf->bdl_entries = buf->period_count;
    buf->bdl = dma_alloc_coherent(dev,
                                   buf->bdl_entries *
                                       sizeof(struct hda_bdl_entry),
                                   (dma_addr_t *)&buf->dma_addr,
                                   GFP_KERNEL);
    if (!buf->bdl) {
        ret = -ENOMEM;
        goto err_free_dma;
    }

    // 填充 BDL 条目
    for (int i = 0; i < buf->bdl_entries; i++) {
        buf->bdl[i].addr = buf->dma_addr + i * buf->period_size;
        buf->bdl[i].length = buf->period_size;
        buf->bdl[i].ioc = 1;  // 周期完成后触发中断
    }

    stream->buf_desc = *buf;

    DRM_DEBUG("Audio DMA buffer: size=%u period=%u count=%u\n",
              buf->buf_size, buf->period_size, buf->period_count);
    DRM_DEBUG("  DMA addr: 0x%llx CPU addr: %p\n",
              buf->dma_addr, buf->cpu_addr);

    return 0;

err_free_dma:
    dma_free_coherent(dev, buf->buf_size,
                      buf->cpu_addr, buf->dma_addr);
err_free_buf:
    kfree(buf);
    return ret;
}

// 释放音频 DMA 缓冲区
static void free_audio_dma_buf(struct audio_stream *stream)
{
    struct device *dev = stream->link->adev->dev;

    // 释放 BDL
    if (stream->buf_desc.bdl) {
        dma_free_coherent(dev,
                           stream->buf_desc.bdl_entries *
                               sizeof(struct hda_bdl_entry),
                           stream->buf_desc.bdl,
                           stream->buf_desc.dma_addr);
    }

    // 释放数据缓冲区
    if (stream->buf_desc.cpu_addr) {
        dma_free_coherent(dev,
                           stream->buf_desc.buf_size,
                           stream->buf_desc.cpu_addr,
                           stream->buf_desc.dma_addr);
    }

    memset(&stream->buf_desc, 0, sizeof(stream->buf_desc));
}

// HDA BDL (Buffer Descriptor List) 中断处理
static irqreturn_t hda_audio_interrupt(int irq, void *dev_id)
{
    struct audio_stream *stream = dev_id;
    uint32_t sd_sts;

    // 读取流状态寄存器
    sd_sts = stream->hda_stream_regs->sts;

    if (sd_sts & HDA_SD_STS_IOC) {
        // 周期完成中断
        stream->stats.frames_played +=
            stream->buf_desc.period_size /
            ((stream->hw_format.sample_size / 8) *
             stream->hw_format.channels);

        // 清除 IOC 位
        stream->hda_stream_regs->sts = HDA_SD_STS_IOC;
    }

    if (sd_sts & HDA_SD_STS_FIFO_ERR) {
        // FIFO 错误
        stream->stats.underruns++;
        DRM_WARN("Audio FIFO underrun on stream %d (total: %lu)\n",
                 stream->stream_id, stream->stats.underruns);

        // 清除错误标志
        stream->hda_stream_regs->sts = HDA_SD_STS_FIFO_ERR;
    }

    if (sd_sts & HDA_SD_STS_DESC_ERR) {
        // 描述符错误
        stream->stats.errors++;
        DRM_ERR("Audio descriptor error on stream %d\n",
                stream->stream_id);

        stream->hda_stream_regs->sts = HDA_SD_STS_DESC_ERR;
    }

    return IRQ_HANDLED;
}
```

### 7.2 音频时钟恢复

```c
// HDMI 音频时钟恢复
// HDMI 使用 CTS (Cycle Time Stamp) 和 N 值来恢复音频时钟

struct hdmi_audio_clk {
    uint32_t n;                 // N 值 (音频时钟分频)
    uint32_t cts;               // CTS (恢复计时)

    // 推导值
    uint32_t audio_clk_hz;      // 恢复的音频时钟频率
    uint32_t jitter_ps;         // 时钟抖动 (皮秒)
};

// 计算音频时钟质量
static struct hdmi_audio_clk
calc_hdmi_audio_clk_quality(uint32_t sample_rate,
                             uint32_t tmds_clk_hz)
{
    struct hdmi_audio_clk clk = {0};
    uint32_t n, cts;
    int ret;

    // 使用标准 N/CTS 表
    ret = hdmi_calc_n_cts(sample_rate,
                           tmds_clk_hz / 1000,
                           &n, &cts);
    if (ret) {
        // 回退到计算模式
        // N = 128 * sample_rate / 1000 (近似)
        n = 128 * sample_rate / 1000;
        cts = tmds_clk_hz * n / (128 * sample_rate);
    }

    clk.n = n;
    clk.cts = cts;

    // 恢复的音频时钟
    clk.audio_clk_hz = (tmds_clk_hz * n) / cts;

    // 估算抖动 (基于 CTS 舍入误差)
    uint64_t ideal_clk = (uint64_t)tmds_clk_hz * n;
    uint64_t actual_clk = (uint64_t)cts * 128 * sample_rate;
    int64_t error_ppm = ((int64_t)ideal_clk - (int64_t)actual_clk) *
                         1000000 / (int64_t)actual_clk;

    clk.jitter_ps = abs(error_ppm) * 1000 /
                     (sample_rate / 1000);

    return clk;
}

// 写入 HDMI 音频时钟寄存器
static void write_hdmi_ncts(struct dc_link *link,
                             uint32_t n, uint32_t cts)
{
    uint32_t reg_val;

    // 写入 N 值
    reg_val = REG_SET(HDMI_AUDIO_N__N, n);
    REG_WRITE(link->hdmi_regs + HDMI_AUDIO_N, reg_val);

    // 写入 CTS 值
    reg_val = REG_SET(HDMI_AUDIO_CTS__CTS, cts);
    REG_WRITE(link->hdmi_regs + HDMI_AUDIO_CTS, reg_val);

    DRM_DEBUG("HDMI audio clock: N=%u CTS=%u\n", n, cts);
}
```

## 8. DP 音频特有的处理

### 8.1 DP 音频训练

```c
// DP 音频训练 (Audio Training)
// DP 需要在链路训练完成后额外进行音频训练

#define DP_AUDIO_TRAINING_PATTERN     0x01  // 音频训练码
#define DP_AUDIO_TRAINING_DONE        0x02

// DP 音频训练序列
static int dp_audio_training(struct dc_link *link)
{
    uint8_t audio_test_pattern;
    int retry = 5;

    DRM_DEBUG("Starting DP audio training on link %d\n",
              link->link_id);

    // 1. 写入音频训练模式
    dpcd_write(link, DPCD_ADDR_AUDIO_TEST_PATTERN,
               &audio_test_pattern, 1);

    // 2. 发送训练码
    uint32_t train_pattern = DP_AUDIO_TRAINING_PATTERN;
    for (int i = 0; i < link->cur_link_settings.lane_count; i++) {
        // 在每个 Lane 上发送音频训练序列
        write_audio_training_pattern(link, i, train_pattern);
    }

    // 3. 等待训练完成
    for (int i = 0; i < retry; i++) {
        uint8_t status;
        dpcd_read(link, DPCD_ADDR_AUDIO_TEST_PATTERN,
                  &status, 1);

        if (status & DP_AUDIO_TRAINING_DONE) {
            DRM_DEBUG("DP audio training complete\n");
            return 0;
        }

        msleep(10);
    }

    DRM_WARN("DP audio training timeout\n");
    return -ETIMEDOUT;
}

// DP 音频 SDP 写入寄存器
static void write_dp_audio_sdp(struct dc_link *link,
                                struct dp_audio_sdp *sdp)
{
    void __iomem *dp_regs = link->dp_regs;
    uint32_t *sdp_words = (uint32_t *)sdp;

    // 将 38 字节的 SDP 按 32bit 写入寄存器
    for (int i = 0; i < sizeof(*sdp) / 4; i++) {
        REG_WRITE(dp_regs + DP_SDP_WRITE_AUDIO(i),
                  sdp_words[i]);
    }

    // 触发 SDP 发送
    REG_WRITE(dp_regs + DP_DP_SDP_AUDIO_SEND, 1);

    DRM_DEBUG("DP audio SDP written\n");
}

// DP 音频测试模式
static int dp_audio_set_test_pattern(struct dc_link *link,
                                      uint8_t pattern_type)
{
    uint8_t test_pattern;

    switch (pattern_type) {
    case 0:
        test_pattern = 0x00;  // 正常模式
        break;
    case 1:
        test_pattern = 0x01;  // 正弦波
        break;
    case 2:
        test_pattern = 0x02;  // 白噪声
        break;
    case 3:
        test_pattern = 0x03;  // 渐进正弦波
        break;
    default:
        return -EINVAL;
    }

    return dpcd_write(link, DPCD_ADDR_AUDIO_TEST_PATTERN,
                       &test_pattern, 1);
}

### 8.2 DP 音频通道映射

DP 音频需要额外的通道映射配置，不同于 HDMI 的 CA 值。

```c
// DP 音频通道映射
// DP 支持 Speaker Mapping Data Block (SMDB)

struct dp_speaker_mapping {
    uint8_t front_left       : 1;
    uint8_t front_right      : 1;
    uint8_t front_center     : 1;
    uint8_t lfe              : 1;
    uint8_t surround_left    : 1;
    uint8_t surround_right   : 1;
    uint8_t surround_back_l  : 1;
    uint8_t surround_back_r  : 1;
};

// 写入 DP 扬声器映射
static int dp_write_speaker_mapping(struct dc_link *link,
                                     struct dp_speaker_mapping *map)
{
    uint8_t smdb[3] = {0};

    // Speaker Mapping Data Block
    smdb[0] = 0x09;  // Tag=9, Length=2
    smdb[1] = *(uint8_t *)map;

    return dpcd_write(link, DPCD_ADDR_AUDIO_SPEAKER_MAPPING,
                       smdb, 3);
}

// DP 音频格式设置 (DPCD)
static int dp_audio_set_format(struct dc_link *link,
                                struct audio_format *fmt)
{
    uint8_t audio_cfg[3] = {0};

    // 配置音频格式
    audio_cfg[0] = fmt->channels - 1;           // 声道数
    audio_cfg[1] = encode_sample_rate(fmt->sample_rate);  // 采样率
    audio_cfg[2] = encode_sample_size(fmt->sample_size);  // 位深

    return dpcd_write(link, DPCD_ADDR_AUDIO_FORMAT,
                       audio_cfg, 3);
}
```

## 9. 音频测试与调试

### 9.1 DebugFS 接口

```c
// 音频调试信息 (DebugFS)
struct audio_debug_info {
    // 编解码器
    uint32_t codec_vendor;
    uint32_t codec_revision;

    // Pin 信息
    uint32_t pin_count;
    struct {
        uint32_t pin_id;
        uint32_t hda_widget;
        uint32_t conn_type;   // 0=HDMI, 1=DP
        bool     active;
        uint32_t ch_map;
        uint32_t sample_rate;
        uint32_t sample_size;
        uint64_t frames_played;
        uint64_t underruns;
    } pins[MAX_PINS];

    // 流统计
    uint32_t active_streams;
    uint32_t total_streams;
    uint32_t format_errors;
};

// DebugFS 读取音频状态
static int amdgpu_debugfs_audio_info_show(struct seq_file *m,
                                           void *unused)
{
    struct amdgpu_device *adev = m->private;
    struct dc_audio_subsystem *audio = adev->dm.audio;

    seq_printf(m, "AMDGPU Audio Subsystem\n");
    seq_printf(m, "=====================\n\n");

    // 1. 编解码器信息
    seq_printf(m, "Codec Vendor:  0x%04X\n",
               audio->codec.vendor_id >> 16);
    seq_printf(m, "Codec Revision: 0x%04X\n",
               audio->codec.revision_id);
    seq_printf(m, "Max Channels:   %u\n",
               audio->codec.caps.max_channels);
    seq_printf(m, "Max Sample Rate: %u Hz\n",
               audio->codec.caps.max_sample_rate);
    seq_printf(m, "Max Bit Depth:   %u bit\n\n",
               audio->codec.caps.max_bits_per_sample);

    // 2. 音频能力
    seq_printf(m, "Capabilities:\n");
    seq_printf(m, "  HDMI Audio:  %s\n",
               audio->caps.flags.hdmi_audio ? "Yes" : "No");
    seq_printf(m, "  DP Audio:    %s\n",
               audio->caps.flags.dp_audio ? "Yes" : "No");
    seq_printf(m, "  MST Audio:   %s\n",
               audio->caps.flags.mst_audio ? "Yes" : "No");
    seq_printf(m, "  HBR Audio:   %s\n\n",
               audio->caps.flags.hbr_audio ? "Yes" : "No");

    // 3. Pin 状态
    seq_printf(m, "Audio Pins (%u):\n", audio->pin_count);
    for (int i = 0; i < audio->pin_count; i++) {
        struct audio_pin *pin = &audio->pins[i];
        struct audio_stream *s = pin->active_stream;

        seq_printf(m, "  Pin %d: Widget=0x%02X %s\n",
                   pin->pin_id, pin->hda_pin,
                   pin->conn_type == CONNECTOR_TYPE_HDMI ?
                       "HDMI" : "DP");

        if (s && s->state == AUDIO_STREAM_RUNNING) {
            seq_printf(m, "    Stream: %d [RUNNING]\n",
                       s->stream_id);
            seq_printf(m, "    Format: %uch %uHz %ubit %s\n",
                       s->hw_format.channels,
                       s->hw_format.sample_rate,
                       s->hw_format.sample_size,
                       s->hw_format.is_passthrough ?
                           "Passthrough" : "PCM");
            seq_printf(m, "    Frames: %lu  Underruns: %lu\n",
                       s->stats.frames_played,
                       s->stats.underruns);
        } else {
            seq_printf(m, "    Status: IDLE\n");
        }
    }

    return 0;
}

// DebugFS 音频强制测试命令
static int amdgpu_debugfs_audio_test(struct seq_file *m,
                                      void *unused)
{
    struct amdgpu_device *adev = m->private;

    seq_printf(m, "Audio Test Commands:\n");
    seq_printf(m, "  echo test-tone <pin_id> <freq> > audio\n");
    seq_printf(m, "  echo silence <pin_id> > audio\n");
    seq_printf(m, "  echo format-list <pin_id> > audio\n");
    seq_printf(m, "  echo enable-mst-audio <link_id> > audio\n");
    seq_printf(m, "  echo clear-stats > audio\n");

    return 0;
}

// 音频测试命令处理
static int amdgpu_debugfs_audio_write(struct file *file,
                                       const char __user *buf,
                                       size_t size, loff_t *pos)
{
    struct amdgpu_device *adev = file->f_inode->i_private;
    char cmd[64];
    int ret;

    if (size >= sizeof(cmd))
        return -EINVAL;

    if (copy_from_user(cmd, buf, size))
        return -EFAULT;
    cmd[size] = '\0';

    // 解析命令
    if (strncmp(cmd, "test-tone", 9) == 0) {
        uint32_t pin_id, freq;
        sscanf(cmd + 10, "%u %u", &pin_id, &freq);
        ret = audio_play_test_tone(adev, pin_id, freq);
        return ret ? ret : size;
    }
    else if (strncmp(cmd, "clear-stats", 11) == 0) {
        audio_clear_stats(adev);
        return size;
    }

    DRM_ERROR("Unknown audio test command: %s\n", cmd);
    return -EINVAL;
}
```

### 9.2 ALSA 测试 (用户空间)

```bash
#!/bin/bash
# ============================================
# HDMI/DP 音频测试脚本
# ============================================

# 配置
CARD=0       # 音频设备卡号 (通常 HDMI/DP 在 card 0 或 card 1)
DEVICE=3     # 设备编号 (可通过 aplay -l 查看)
SAMPLE_RATE=48000
CHANNELS=2
FORMAT=S16_LE
DURATION=5   # 测试时长 (秒)

echo "========================================="
echo "HDMI/DP 音频测试工具"
echo "========================================="

# 1. 列出音频设备
echo ""
echo "[1] 列出音频设备..."
aplay -l | grep -E "HDMI|DP|HDA"

# 2. 播放测试音 (正弦波)
echo ""
echo "[2] 播放测试音 (48000Hz 正弦波)..."

# 生成测试音频 (使用 sox/ffmpeg 或直接通过 speaker-test)
if command -v speaker-test &> /dev/null; then
    echo "  使用 speaker-test..."
    speaker-test -c $CHANNELS -r $SAMPLE_RATE \
                 -D hw:$CARD,$DEVICE -l 1 -t sine -f 440
else
    echo "  speaker-test 未安装, 跳过"
fi

# 3. 播放 WAV 文件
echo ""
echo "[3] 播放测试 WAV 文件..."
if [ -f /usr/share/sounds/alsa/test.wav ]; then
    aplay -D hw:$CARD,$DEVICE /usr/share/sounds/alsa/test.wav
else
    echo "  测试文件不存在, 生成测试音频..."
    if command -v ffmpeg &> /dev/null; then
        ffmpeg -f lavfi -i "sine=frequency=440:duration=3" \
               -ac $CHANNELS -ar $SAMPLE_RATE \
               -f wav - | aplay -D hw:$CARD,$DEVICE
    fi
fi

# 4. 多声道测试
echo ""
echo "[4] 多声道测试 (7.1声道)..."
speaker-test -c 8 -r $SAMPLE_RATE \
             -D hw:$CARD,$DEVICE -l 1 -t sine -f 1000

# 5. 采样率扫描测试
echo ""
echo "[5] 采样率扫描测试..."
for RATE in 32000 44100 48000 88200 96000 176400 192000; do
    echo "  测试采样率: ${RATE}Hz"
    if command -v speaker-test &> /dev/null; then
        speaker-test -c 2 -r $RATE \
                     -D hw:$CARD,$DEVICE -l 1 \
                     -t sine -f 1000 &>/dev/null
        if [ $? -eq 0 ]; then
            echo "    -> 支持"
        else
            echo "    -> 不支持"
        fi
    fi
done

# 6. EDID 音频能力查询
echo ""
echo "[6] EDID 音频能力..."
for card in /sys/class/drm/card*/HDMI*/edid; do
    if [ -f "$card" ]; then
        echo "  $card:"
        cat "$card" | edid-decode 2>/dev/null | grep -A 30 "Audio Data Block" || \
            echo "    edid-decode 未安装"
    fi
done

# 7. 检查音频状态
echo ""
echo "[7] 系统音频状态检查..."
echo "  ALSA 设备列表:"
for dev in /proc/asound/card*/eld#*; do
    if [ -f "$dev" ]; then
        echo "    $dev"
        head -5 "$dev"
    fi
done

echo ""
echo "========================================="
echo "测试完成"
echo "========================================="
```

### 9.3 音频格式协商测试 (Python)

```python
#!/usr/bin/env python3
"""
音频格式协商测试工具
测试 HDMI/DP 音频在不同格式下的驱动行为
"""

import os
import sys
import time
import struct
import subprocess
from enum import IntEnum


class AudioFormat(IntEnum):
    """音频编码格式"""
    PCM = 1
    AC3 = 2
    EAC3 = 3
    TRUEHD = 4
    DTS = 5
    DTS_HD = 6
    AAC = 7
    DSD = 8


class AudioTestSuite:
    """音频测试套件"""

    def __init__(self, card=0, device=3):
        self.card = card
        self.device = device
        self.debugfs = "/sys/kernel/debug/dri/0/amdgpu_audio_info"

    def check_audio_caps(self):
        """检查驱动报告的音频能力"""
        try:
            with open(self.debugfs, "r") as f:
                content = f.read()
                print("=== 驱动音频能力 ===")
                print(content)

                # 检查关键能力
                if "HBR Audio: Yes" in content:
                    print("[PASS] HBR 音频支持")
                else:
                    print("[INFO] HBR 音频不支持")

                return content
        except FileNotFoundError:
            print(f"[WARN] DebugFS 不可用: {self.debugfs}")
            return None

    def test_pcm_playback(self, channels=2, rate=48000,
                          bits=16, duration=3):
        """测试 PCM 播放"""
        print(f"\n=== PCM 测试: {channels}ch {rate}Hz {bits}bit ===")

        cmd = [
            "speaker-test", "-c", str(channels),
            "-r", str(rate), "-F", str(bits),
            "-D", f"hw:{self.card},{self.device}",
            "-l", "1", "-t", "sine", "-f", "1000"
        ]

        try:
            result = subprocess.run(
                cmd, capture_output=True, text=True, timeout=duration
            )
            if result.returncode == 0:
                print(f"  [PASS] PCM {channels}ch {rate}Hz")
                return True
            else:
                print(f"  [FAIL] {result.stderr}")
                return False
        except subprocess.TimeoutExpired:
            print(f"  [PASS] PCM {channels}ch {rate}Hz (播放完成)")
            return True
        except FileNotFoundError:
            print("  [SKIP] speaker-test 未安装")
            return None

    def test_format_scan(self):
        """扫描所有支持的音频格式"""
        print("\n=== 格式扫描测试 ===")

        formats = [
            (2, 32000, 16), (2, 44100, 16), (2, 48000, 16),
            (2, 88200, 24), (2, 96000, 24), (2, 176400, 24),
            (2, 192000, 24), (6, 48000, 16), (6, 96000, 24),
            (8, 48000, 16), (8, 96000, 24),
        ]

        results = {"pass": 0, "fail": 0, "skip": 0}

        for channels, rate, bits in formats:
            result = self.test_pcm_playback(channels, rate, bits)
            if result is True:
                results["pass"] += 1
            elif result is False:
                results["fail"] += 1
            else:
                results["skip"] += 1
            time.sleep(0.5)

        print(f"\n=== 格式扫描结果 ===")
        print(f"  PASS: {results['pass']}")
        print(f"  FAIL: {results['fail']}")
        print(f"  SKIP: {results['skip']}")

        return results["fail"] == 0

    def test_multi_channel_routing(self):
        """多声道路由测试"""
        print("\n=== 多声道路由测试 ===")

        # 测试每个声道的独立输出
        layouts = [
            ("2.0", 2, [0, 1]),
            ("5.1", 6, [0, 1, 4, 5, 2, 3]),
            ("7.1", 8, [0, 1, 4, 5, 2, 3, 6, 7]),
        ]

        for name, channels, layout in layouts:
            print(f"  测试声道布局: {name} (通道: {layout})")
            cmd = [
                "speaker-test", "-c", str(channels),
                "-D", f"hw:{self.card},{self.device}",
                "-l", "1", "-t", "sine", "-f", "1000"
            ]
            try:
                subprocess.run(cmd, capture_output=True,
                               timeout=5)
                print(f"    [PASS] {name}")
            except subprocess.TimeoutExpired:
                print(f"    [PASS] {name} (播放完成)")
            except Exception as e:
                print(f"    [FAIL] {e}")

    def test_passthrough(self):
        """透传格式测试"""
        print("\n=== 透传格式测试 ===")

        # 尝试播放压缩格式 (需要接收设备支持)
        passthrough_tests = [
            ("AC3", "ac3"),
            ("DTS", "dts"),
            ("EAC3", "eac3"),
        ]

        for name, codec in passthrough_tests:
            print(f"  测试 {name} 透传...")
            # 使用 aplay 播放压缩流
            # 通常需要特殊的测试文件
            test_file = f"/tmp/test_{codec}.bin"

            if os.path.exists(test_file):
                cmd = [
                    "aplay", "-D",
                    f"hw:{self.card},{self.device}",
                    "--format", "raw", "-c", "2",
                    "-r", "48000", "-f", "S16_LE",
                    test_file
                ]
                try:
                    subprocess.run(cmd, capture_output=True,
                                   timeout=3)
                    print(f"    [PASS] {name} 透传")
                except Exception as e:
                    print(f"    [FAIL] {e}")
            else:
                print(f"    [SKIP] 测试文件不存在: {test_file}")


def main():
    test = AudioTestSuite(card=0, device=3)

    print("AMDGPU 音频测试套件")
    print("=" * 40)

    # 1. 检查驱动能力
    test.check_audio_caps()

    # 2. 基础 PCM 测试
    test.test_pcm_playback(2, 48000, 16)

    # 3. 格式扫描
    test.test_format_scan()

    # 4. 多声道测试
    test.test_multi_channel_routing()

    # 5. 透传测试
    test.test_passthrough()


if __name__ == "__main__":
    main()
```

## 10. 常见音频问题与故障排除

### 10.1 无音频输出排查

```
无音频输出检查清单:

1. 检查 HDMI/DP 连接
   $ xrandr --verbose | grep Audio
   - HDMI: 检查 "audio" 属性是否为 "on"

2. 检查 ALSA 设备
   $ aplay -l
   - 确认看到 "HDMI" 或 "DP" 设备
   - 注意 card 和 device 编号

3. 检查 ELD (EDID Like Data)
   $ cat /proc/asound/card0/eld#0
   - 确认 monitor_name 正确
   - 确认 audio_para 包含有效格式

4. 检查音频驱动加载
   $ lsmod | grep snd_hda_intel
   $ dmesg | grep -i "audio\|HDA\|HDMI"
   - 检查编解码器初始化日志

5. 测试强制输出
   $ aplay -D plughw:0,3 /usr/share/sounds/alsa/test.wav
   - 使用 plughw 绕过格式转换
```

### 10.2 常见错误码

```c
// 音频错误码
#define AUDIO_ERR_BASE      -8192

enum audio_error_code {
    // 编解码器错误
    AUDIO_ERR_CODEC_INIT        = AUDIO_ERR_BASE - 1,
    AUDIO_ERR_CODEC_READ        = AUDIO_ERR_BASE - 2,
    AUDIO_ERR_CODEC_WRITE       = AUDIO_ERR_BASE - 3,
    AUDIO_ERR_CODEC_TIMEOUT     = AUDIO_ERR_BASE - 4,

    // 流错误
    AUDIO_ERR_STREAM_ALLOC      = AUDIO_ERR_BASE - 10,
    AUDIO_ERR_STREAM_FORMAT     = AUDIO_ERR_BASE - 11,
    AUDIO_ERR_STREAM_DMA        = AUDIO_ERR_BASE - 12,
    AUDIO_ERR_STREAM_UNDERRUN   = AUDIO_ERR_BASE - 13,

    // 显示引擎错误
    AUDIO_ERR_AFMT_CONFIG       = AUDIO_ERR_BASE - 20,
    AUDIO_ERR_INFOFRAME         = AUDIO_ERR_BASE - 21,
    AUDIO_ERR_SDP               = AUDIO_ERR_BASE - 22,
    AUDIO_ERR_NCTS              = AUDIO_ERR_BASE - 23,

    // 时钟恢复错误
    AUDIO_ERR_CLOCK_RECOVERY    = AUDIO_ERR_BASE - 30,
    AUDIO_ERR_CTS_NOT_FOUND     = AUDIO_ERR_BASE - 31,
    AUDIO_ERR_AUDIO_JITTER      = AUDIO_ERR_BASE - 32,
};

// 音频错误处理
static void audio_handle_error(struct audio_stream *stream,
                                enum audio_error_code err)
{
    DRM_WARN("Audio error on stream %d: code=%d\n",
             stream->stream_id, err);

    switch (err) {
    case AUDIO_ERR_STREAM_UNDERRUN:
        // 尝试恢复: 重置流指针
        hda_stream_reset(stream);
        break;

    case AUDIO_ERR_CTS_NOT_FOUND:
        // 使用备用 CTS 计算
        hdmi_calc_n_cts_fallback(stream);
        break;

    case AUDIO_ERR_INFOFRAME:
        // 重新发送 InfoFrame
        resend_audio_infoframe(stream);
        break;

    case AUDIO_ERR_CODEC_TIMEOUT:
        // 重置编解码器
        audio_codec_reset(stream->link->adev);
        break;

    default:
        // 通用恢复: 停止后重新启动
        amdgpu_audio_stream_stop(stream);
        break;
    }
}
```

### 10.3 驱动级调试技巧

```c
// 1. 启用音频详细日志
// 内核参数: amdgpu_audio_debug=1
// 或运行时:
// echo 0x1F > /sys/module/amdgpu/parameters/debug_mask

#define AUDIO_DEBUG_INFO   0x01  // 基本信息
#define AUDIO_DEBUG_FLOW   0x02  // 流控制
#define AUDIO_DEBUG_REG    0x04  // 寄存器操作
#define AUDIO_DEBUG_PKT    0x08  // 数据包详细
#define AUDIO_DEBUG_ERR    0x10  // 错误追踪

// 2. 音频寄存器转储
static void dump_audio_registers(struct dc_link *link)
{
    if (!(amdgpu_audio_debug & AUDIO_DEBUG_REG))
        return;

    DRM_INFO("Audio registers for link %d:\n", link->link_id);

    if (link->connector_signal & SIGNAL_TYPE_HDMI) {
        // HDMI 寄存器
        DRM_INFO("  AFMT_CTL:    0x%08X\n",
                 REG_READ(link->afmt_regs + AFMT_CTL));
        DRM_INFO("  AFMT_CNTL:   0x%08X\n",
                 REG_READ(link->afmt_regs + AFMT_CNTL));
        DRM_INFO("  HDMI_AUDIO_N:  0x%08X\n",
                 REG_READ(link->hdmi_regs + HDMI_AUDIO_N));
        DRM_INFO("  HDMI_AUDIO_CTS:0x%08X\n",
                 REG_READ(link->hdmi_regs + HDMI_AUDIO_CTS));
    } else {
        // DP 寄存器
        DRM_INFO("  DP_AUDIO_SDP_CTL: 0x%08X\n",
                 REG_READ(link->dp_regs + DP_AUDIO_SDP_CTL));
        DRM_INFO("  DP_VID_AUDIO_CTL: 0x%08X\n",
                 REG_READ(link->dp_regs + DP_VID_AUDIO_CTL));
    }
}

// 3. 音频流统计转储
static void dump_audio_stats(struct amdgpu_device *adev)
{
    struct dc_audio_subsystem *audio = adev->dm.audio;

    DRM_INFO("Audio stream statistics:\n");

    for (int i = 0; i < audio->stream_count; i++) {
        struct audio_stream *s = &audio->streams[i];

        if (s->state != AUDIO_STREAM_IDLE) {
            DRM_INFO("  Stream %d: state=%d frames=%lu "
                     "underruns=%lu errors=%lu\n",
                     s->stream_id, s->state,
                     s->stats.frames_played,
                     s->stats.underruns,
                     s->stats.errors);
        }
    }
}

// 4. DPCD 音频状态检查
static int check_dpcd_audio_status(struct dc_link *link)
{
    uint8_t status[4];

    // 读取 DPCD 音频状态
    dpcd_read(link, DPCD_ADDR_AUDIO_STATUS, status, 4);

    DRM_INFO("DPCD Audio Status for link %d:\n", link->link_id);
    DRM_INFO("  Link Audio Capable:    %s\n",
             (status[0] & 0x01) ? "Yes" : "No");
    DRM_INFO("  Audio Stream Active:   %s\n",
             (status[0] & 0x02) ? "Yes" : "No");
    DRM_INFO("  Mute Status:           %s\n",
             (status[1] & 0x01) ? "Muted" : "Unmuted");
    DRM_INFO("  Sample Rate:           %u Hz\n",
             decode_sample_rate(status[2] & 0x0F));
    DRM_INFO("  Channel Count:         %u\n",
             (status[3] & 0x0F) + 1);

    return 0;
}
```

## 11. 总结

Day 359 深入讲解了 AMDGPU DC 子系统的音频透传机制，涵盖了从 HDA 编解码器到显示引擎 Audio InfoFrame/SDP 的完整链路。

### 关键知识点总结

```
架构理解
  ├── 音频路径: ALSA → HDA Codec → DC Audio → AFMT → HDMI/DP TX
  ├── 两种接口: HDMI (InfoFrame) 和 DP (SDP) 封装差异
  └── MST 支持: 每个 VC Payload 独立音频流

数据流管理
  ├── 编解码器探测: Widget 枚举和 Pin 发现
  ├── 流生命周期: Alloc → Prepare → Start → Stop → Free
  ├── DMA 缓冲区: BDL 循环缓冲区管理
  └── 格式协商: 请求格式 → EDID 能力 → 降级 → 最终格式

时钟与同步
  ├── HDMI: N/CTS 时钟恢复 (精确度关键)
  ├── DP: 基于 Main Link Symbol 的时钟恢复
  └── 时基: 128 × fs 参考时钟

调试手段
  ├── DebugFS: 实时音频状态查询
  ├── DPCD: DP 音频状态寄存器
  ├── ALSA: speaker-test / aplay / arecord
  └── dmesg: 音频详细日志 (debug_mask)
```

### 测试重点

1. **基本功能测试**: 各采样率/声道/位深的 PCM 播放
2. **多声道路由**: 5.1/7.1 声道映射正确性
3. **采样率切换**: 32kHz~192kHz 无缝切换
4. **HBR 测试**: 176.4kHz/192kHz 高带宽音频
5. **透传测试**: AC3/DTS 压缩格式透传
6. **MST 音频**: 多流独立音频输出
7. **热插拔**: 音频设备拔插时的恢复

### 下一步

Day 360 将进行 DC 子系统进阶综合分析考核，综合运用 Day 356~359 的知识进行实战分析。
```