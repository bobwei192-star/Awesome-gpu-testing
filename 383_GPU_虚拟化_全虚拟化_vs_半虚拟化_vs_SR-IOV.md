# Day 383：GPU 虚拟化：全虚拟化 vs 半虚拟化 vs SR-IOV

## 学习目标

- 理解 GPU 虚拟化的三大技术路径：全虚拟化、半虚拟化、SR-IOV
- 掌握每种虚拟化方案的架构原理、性能特征和适用场景
- 理解 API 转发（如 Mesa 的 VirGL）与设备直通的区别
- 学会根据业务需求选择合适的 GPU 虚拟化方案
- 了解 GPU 虚拟化的发展趋势和前沿技术

## 知识详解

### 概念原理

#### GPU 虚拟化的核心矛盾

GPU 虚拟化面临的本质挑战可以概括为**三个矛盾**：

```
矛盾一：性能 vs 共享
  ┌─────────────────────────┐
  │  全虚拟化               │
  │  (软模拟，性能差)        │
  ├─────────────────────────┤
  │  半虚拟化/API 转发       │  ← 性能与共享的平衡点
  │  (性能较好，API 受限)    │
  ├─────────────────────────┤
  │  设备直通/SR-IOV        │
  │  (性能高，共享度低)      │
  └─────────────────────────┘
  
矛盾二：隔离性 vs 密度
  ┌─────────────────────────┐
  │  每 VF 独立硬件资源      │
  │  (隔离好，密度低)        │  ← SR-IOV
  ├─────────────────────────┤
  │  软件层隔离              │
  │  (隔离较弱，密度高)      │  ← API 转发
  └─────────────────────────┘

矛盾三：功能完整 vs 驱动兼容
  ┌─────────────────────────┐
  │  完整 GPU API            │
  │  (需要原生驱动)          │  ← 设备直通
  ├─────────────────────────┤
  │  有限的 API 子集         │
  │  (无需原生驱动)          │  ← API 转发
  └─────────────────────────┘
```

#### 三大虚拟化方案总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                     GPU 虚拟化技术全景图                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    1. 全虚拟化 (Full Virtualization)          │  │
│  │                                                              │  │
│  │  VM Guest          VM Guest          VM Guest                │  │
│  │  ┌────────┐       ┌────────┐       ┌────────┐               │  │
│  │  │ 原生驱动 │       │ 原生驱动 │       │ 原生驱动 │               │  │
│  │  └────┬───┘       └────┬───┘       └────┬───┘               │  │
│  │       │                │                │                     │  │
│  │  ┌────┴────────────────┴────────────────┴────────┐          │  │
│  │  │          Hypervisor (软件模拟 GPU)              │          │  │
│  │  │          - Trap 所有 GPU 访问                  │          │  │
│  │  │          - 软件渲染/位图传输                    │          │  │
│  │  │          - 性能极差 (1-5% 原生)                │          │  │
│  │  └───────────────────────┬────────────────────────┘          │  │
│  │                          │                                    │  │
│  │                    ┌─────┴─────┐                              │  │
│  │                    │  物理 GPU  │                              │  │
│  │                    └───────────┘                              │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │             2. 半虚拟化/API 转发 (Paravirtualization)         │  │
│  │                                                              │  │
│  │  VM Guest          VM Guest          VM Guest                │  │
│  │  ┌────────┐       ┌────────┐       ┌────────┐               │  │
│  │  │ 前端驱动 │       │ 前端驱动 │       │ 前端驱动 │               │  │
│  │  │(VirGL)  │       │(VirGL)  │       │(VirGL)  │               │  │
│  │  └────┬───┘       └────┬───┘       └────┬───┘               │  │
│  │       │                │                │                     │  │
│  │  ┌────┴────────────────┴────────────────┴────────┐          │  │
│  │  │        Hypervisor (API 转发层)                 │          │  │
│  │  │        - 拦截 OpenGL/Vulkan API 调用          │          │  │
│  │  │        - 转发到 Host GPU 执行                 │          │  │
│  │  │        - 性能约 60-90% 原生                   │          │  │
│  │  └───────────────────────┬────────────────────────┘          │  │
│  │                          │                                    │  │
│  │                    ┌─────┴─────┐                              │  │
│  │                    │  物理 GPU  │                              │  │
│  │                    └───────────┘                              │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              3. SR-IOV (硬件辅助虚拟化)                      │  │
│  │                                                              │  │
│  │  VM Guest          VM Guest          VM Guest                │  │
│  │  ┌────────┐       ┌────────┐       ┌────────┐               │  │
│  │  │ VF 驱动 │       │ VF 驱动 │       │ VF 驱动 │               │  │
│  │  └────┬───┘       └────┬───┘       └────┬───┘               │  │
│  │       │                │                │                     │  │
│  │  ┌────┴────────────────┴────────────────┴────────┐          │  │
│  │  │    PCIe SR-IOV (硬件级虚拟化)                  │          │  │
│  │  │    - 每个 VF 有独立 BAR/DMA/中断               │          │  │
│  │  │    - 硬件资源分区                             │          │  │
│  │  │    - 性能约 90-95% 原生                       │          │  │
│  │  └───────────────────────┬────────────────────────┘          │  │
│  │                          │                                    │  │
│  │  ┌──────────────┬────────┴────────┬──────────────┐          │  │
│  │  │   VF 0 (PF)   │   VF 1         │   VF 2       │          │  │
│  │  │   完整功能     │   子集功能      │   子集功能    │          │  │
│  │  └──────────────┴─────────────────┴──────────────┘          │  │
│  │                    ┌───────────┐                              │  │
│  │                    │ 硬件调度器 │                              │  │
│  │                    └───────────┘                              │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1. 全虚拟化（Full Virtualization）

**架构原理：**

全虚拟化通过 Hypervisor 的软件模拟层（QEMU/VMWare）让 Guest OS 以为自己在直接访问 GPU 硬件。Hypervisor 捕获所有 GPU 寄存器的读写和 DMA 操作，并通过软件模拟来实现。

```c
// QEMU 中 GPU 全虚拟化的简化示意
// 虚拟 GPU 寄存器读写陷入处理

// Guest 尝试写入 GPU 寄存器
void gpu_mmio_write(void *opaque, hwaddr addr, uint64_t val, unsigned size)
{
    VirtGPUState *s = opaque;
    
    switch (addr) {
    case VGPU_REG_COMMAND:
        // 模拟 GPU 命令处理
        virt_gpu_process_cmd(s, val);
        break;
    case VGPU_REG_STATUS:
        // Guest 写入状态寄存器 → 忽略（只读寄存器）
        break;
    case VGPU_REG_FRAMEBUFFER:
        // 帧缓冲写入 → 复制到 Host 显示
        virt_gpu_update_fb(s, addr, val);
        break;
    default:
        // 其他 MMIO 模拟
        break;
    }
}

// 主要性能瓶颈：
// 1. 每个 MMIO 访问都需要 VM-Exit → 30μs 延迟
// 2. 软件渲染没有硬件加速
// 3. DMA 操作需要多次拷贝
```

**全虚拟化的性能瓶颈分析：**

```
GPU 全虚拟化延迟分布：

操作                        延迟 (μs)        占比
─────────────────────────────────────────────
MMIO 模拟 (每次 VM-Exit)     30-50           5%
命令缓冲区模拟                200-500         15%
帧缓冲传输 (每次屏幕更新)     1000-5000       40%
DMA 操作模拟                 500-2000        20%
状态同步                     200-500         10%
其他                         100-300         10%
─────────────────────────────────────────────
总计 (每帧)                  ~10-50ms        100%
```

**真实案例：** QEMU 的 virtio-gpu 和 VMWare 的 SVGA II 是全虚拟化 GPU 的代表。它们主要用于：
- 虚拟机的 2D 显示（控制台、桌面环境）
- 基本的 3D 加速（通过软件渲染）
- 不适合高性能计算、3D 游戏或 AI 训练

#### 2. 半虚拟化 / API 转发（Paravirtualization）

**架构原理：**

半虚拟化不在硬件层面模拟 GPU，而是在 API 层面进行转发。Guest 中的应用程序调用图形 API（OpenGL/Vulkan），这些调用通过前端驱动（VirGL/VirtIO-GPU）传递给 Host 的后端渲染器，由物理 GPU 执行。

```
VirGL (Virtual GPU for OpenGL) 架构详解：

┌─────────────────────────────────────────────┐
│ VM Guest                                      │
│                                               │
│  应用程序 (应用)                                │
│     │                                          │
│     ▼ (OpenGL/Vulkan API 调用)                  │
│  ┌─────────────────────┐                      │
│  │  Mesa 3D (Guest)    │                      │
│  │  - 状态追踪器         │                      │
│  │  - 命令编码器         │                      │
│  └─────────┬───────────┘                      │
│            │                                   │
│  ┌─────────┴───────────┐                      │
│  │  VirGL 前端驱动      │                      │
│  │  - virtio-gpu venc   │                      │
│  │  - 命令缓冲 (CMD)    │                      │
│  │  - 数据缓冲 (DATA)   │                      │
│  └─────────┬───────────┘                      │
│            │                                   │
└────────────┼───────────────────────────────────┘
             │ virtio 通道 (共享内存)
┌────────────┼───────────────────────────────────┐
│ Host        ▼                                   │
│  ┌───────────────────────────┐                  │
│  │  VirGL 后端 (Host)        │                  │
│  │  - 命令解码器              │                  │
│  │  - 状态还原                │                  │
│  │  - 渲染控制               │                  │
│  └─────────┬─────────────────┘                  │
│            │                                    │
│  ┌─────────┴─────────────────┐                  │
│  │  Host Mesa 3D             │                  │
│  │  (使用物理 GPU 渲染)      │                  │
│  └─────────┬─────────────────┘                  │
│            │                                    │
│  ┌─────────┴─────────────────┐                  │
│  │  AMDGPU/NVIDIA 驱动       │                  │
│  └─────────┬─────────────────┘                  │
│            │                                    │
└────────────┼────────────────────────────────────┘
             │
             ▼
      ┌──────────────┐
      │   物理 GPU    │
      └──────────────┘
```

**VirGL 命令流示例：**

```c
// Guest 侧：GL 命令编码
void virgl_gl_draw_arrays(struct virgl_context *ctx, 
                           GLenum mode, GLint first, GLsizei count)
{
    struct virgl_cmd_draw_arrays cmd = {
        .header.type = VIRGL_CMD_DRAW_ARRAYS,
        .header.len = sizeof(cmd) / 4,
        .mode = mode,
        .first = first,
        .count = count,
    };
    
    // 编码命令到 virtio 命令缓冲区
    virgl_encoder_write(ctx->cbuf, &cmd, sizeof(cmd));
    
    // 通知 Host 执行
    virtio_gpu_notify(ctx->virtio_dev);
}

// Host 侧：GL 命令解码和执行
int virgl_renderer_decode_cmd(struct virgl_renderer *renderer,
                               const uint32_t *buf, int len)
{
    struct virgl_cmd_header *hdr = (struct virgl_cmd_header *)buf;
    
    switch (hdr->type) {
    case VIRGL_CMD_DRAW_ARRAYS: {
        struct virgl_cmd_draw_arrays *cmd = 
            (struct virgl_cmd_draw_arrays *)buf;
        
        // 恢复 GPU 状态
        virgl_renderer_restore_state(renderer);
        
        // 执行实际的 GL 调用
        glDrawArrays(cmd->mode, cmd->first, cmd->count);
        break;
    }
    // ... 其他命令处理
    }
}
```

**API 转发的性能特征：**

```
性能开销分布：

操作类型                    每帧调用次数    单次开销 (μs)    总开销 (μs)
─────────────────────────────────────────────────────────────────────
状态绑定 (bind texture)     1000            2                2000
绘制调用 (draw call)        500             5                2500
状态查询 (get)              100             3                300
缓冲区更新 (buffer data)    50              50               2500
帧提交 (swap buffers)       1               500              500
─────────────────────────────────────────────────────────────────────
每帧 API 转发总开销                           ~7800 μs (7.8ms)

如果目标帧率 60fps (16.6ms 每帧):
- API 转发开销: 7.8ms (47%)
- GPU 渲染时间: 8.8ms (53%)
- 整体性能约为原生的 53-60%
```

#### 3. SR-IOV（硬件辅助虚拟化）

SR-IOV 在前两天的文档中已详细介绍（Day 381-382），这里重点对比其与其他方案的区别。

**SR-IOV 的独特优势：**

```
性能特征对比（以 AMDGPU 为例）：

指标               全虚拟化      API 转发      SR-IOV      原生
─────────────────────────────────────────────────────────────────
GPU 计算           1-5%        60-80%       90-95%       100%
3D 渲染            1-5%        50-70%       90-95%       100%
显存访问           通过模拟      通过转发      直接访问      直接访问
延迟               +10-50ms    +5-10ms      +0.5-2ms     0
PCIe 带宽利用率    30-50%      60-70%       90-95%       100%
API 兼容性         完全         部分(Vulkan)  完全          完全
多 VM 共享         支持         支持          支持(有限)    不支持
─────────────────────────────────────────────────────────────────
```

#### GPU 虚拟化新技术趋势

**MDEV（Mediated Device）模式：**

NVIDIA 的 vGPU 和 Intel 的 GVT-g 采用了一种称为"mediated passthrough"的混合方案：

```
MDEV 架构：

┌──────────────────────────────────────────────────┐
│                  Hypervisor                       │
│                                                   │
│  VM Guest 1               VM Guest 2              │
│  ┌────────────────┐      ┌────────────────┐      │
│  │ 原生驱动 (部分)  │      │ 原生驱动 (部分)  │      │
│  │ - 计算直接访问   │      │ - 计算直接访问   │      │
│  │ - 显示/管理陷入  │      │ - 显示/管理陷入  │      │
│  └───────┬────────┘      └───────┬────────┘      │
│          │                       │                │
│  ┌───────┴───────────────────────┴────────┐      │
│  │      MDEV 中介层 (Mediation Layer)      │      │
│  │      - 仲裁显示引擎访问                  │      │
│  │      - 虚拟化帧缓冲                      │      │
│  │      - 调度 GPU 时间片                  │      │
│  └───────────────────┬────────────────────┘      │
│                      │                            │
└──────────────────────┼────────────────────────────┘
                       │
              ┌────────┴────────┐
              │    物理 GPU      │
              │  - 计算直通       │
              │  - 显示陷入      │
              └─────────────────┘
```

#### 技术选型决策框架

```
GPU 虚拟化方案选择决策树：

业务需求
│
├─ 是否需要完整的 GPU 计算/渲染性能 (95%+)？
│   ├─ 是 → 需要多租户共享？
│   │   ├─ 是 → SR-IOV（硬件支持时）
│   │   │         或 MDEV (NVIDIA vGPU)
│   │   └─ 否 → 设备直通 (Passthrough)
│   └─ 否 → 需要完整 API 兼容性？
│       ├─ 是 → API 转发 (VirGL)
│       └─ 否 → 全虚拟化 (virtio-gpu)
│
├─ 主要应用场景？
│   ├─ AI 训练/推理 → SR-IOV 或设备直通
│   ├─ 3D 渲染/设计 → API 转发或 SR-IOV
│   ├─ 云游戏 → SR-IOV 或 MDEV
│   ├─ 虚拟桌面 (VDI) → API 转发或全虚拟化
│   └─ 显示终端 → 全虚拟化
│
├─ VM 密度要求？
│   ├─ 高 (16+ VM/GPU) → API 转发或全虚拟化
│   ├─ 中 (4-8 VM/GPU) → SR-IOV 或 MDEV
│   └─ 低 (1-2 VM/GPU) → 设备直通
│
└─ 安全隔离要求？
    ├─ 高 → SR-IOV 或设备直通
    ├─ 中 → MDEV 或 API 转发
    └─ 低 → 全虚拟化
```

### 实践操作

#### 1. 方案一：QEMU + virtio-gpu（全虚拟化）

```bash
# 创建带有 virtio-gpu 的虚拟机
sudo qemu-system-x86_64 \
    -name "gpu-vm-fullvirt" \
    -m 4096 \
    -smp 4 \
    -vga virtio \
    -display gtk,gl=on \
    -device virtio-gpu-pci \
    -drive file=./ubuntu.qcow2,format=qcow2 \
    -netdev user,id=net0 \
    -device virtio-net-pci,netdev=net0

# 在 Guest 中检查 GPU
lspci | grep "VGA"
# 输出: 00:02.0 VGA compatible controller: Red Hat, Inc. Virtio GPU

# 安装 Guest 驱动
sudo apt install mesa-utils
glxinfo | grep "OpenGL renderer"
# 输出: OpenGL renderer: virgl (Software Renderer)
# 或 (如果 Host 支持): virgl (AMD Radeon ...)

# 验证性能
glxgears
# 预期: ~60-200 FPS (纯软件渲染，性能有限)
```

#### 2. 方案二：QEMU + VirGL（API 转发）

```bash
# 1. 在 Host 上安装 VirGL 支持
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients \
                 bridge-utils virt-manager \
                 libegl1-mesa libgles2-mesa \
                 virgl-server

# 2. 创建支持 VirGL 的 VM
sudo qemu-system-x86_64 \
    -name "gpu-vm-virgl" \
    -machine accel=kvm \
    -m 8192 \
    -smp 8 \
    -vga none \
    -device virtio-vga-gl \
    -display gtk,gl=on \
    -device virtio-gpu-pci,gl=on \
    -object memory-backend-memfd,id=mem,size=8192M \
    -numa node,memdev=mem \
    -drive file=./ubuntu.qcow2,format=qcow2 \
    -netdev user,id=net0 \
    -device virtio-net-pci,netdev=net0

# 3. 在 Guest 中验证 VirGL
sudo apt install mesa-utils
glxinfo | grep -E "OpenGL renderer|OpenGL version"

# 输出示例：
# OpenGL renderer: virgl (AMD RADV NAVI31)
# OpenGL version: 4.5 (Compatibility Profile) Mesa 24.0.0

# 4. 运行 3D 基准测试
glxgears
# 预期: ~1000-3000 FPS (硬件加速)

# 测试 Vulkan 支持（如果 Guest Mesa 支持）
vulkaninfo | grep "deviceName"
# 输出: deviceName = virgl (AMD RADV NAVI31)
```

#### 3. 方案三：VFIO Passthrough（设备直通）

```bash
# 1. 确认硬件支持 IOMMU
dmesg | grep "AMD-Vi\|Intel-IOMMU"

# 2. 找到 GPU 的 BDF 和 IOMMU 分组
lspci | grep AMD
# 03:00.0 VGA compatible controller: Advanced Micro Devices, Inc.

# 检查 IOMMU 分组
ls -l /sys/kernel/iommu_groups/ | grep "0000:03:00"

# 3. 解绑 GPU 驱动并绑定到 vfio-pci
GPU_BDF="0000:03:00.0"
GPU_ID=$(lspci -n -s ${GPU_BDF} | cut -d' ' -f3 | tr ':' ' ')

# 解绑
echo ${GPU_BDF} > /sys/bus/pci/devices/${GPU_BDF}/driver/unbind

# 绑定到 vfio-pci
echo "vfio-pci" > /sys/bus/pci/devices/${GPU_BDF}/driver_override
echo ${GPU_BDF} > /sys/bus/pci/drivers/vfio-pci/bind

# 或者通过模块参数自动绑定
echo "options vfio-pci ids=${GPU_ID}" > /etc/modprobe.d/vfio.conf

# 4. 启动 VM 直通 GPU
sudo qemu-system-x86_64 \
    -name "gpu-vm-passthrough" \
    -machine accel=kvm \
    -m 16384 \
    -smp 16 \
    -vga none \
    -nographic \
    -device vfio-pci,host=03:00.0 \
    -drive file=./ubuntu.qcow2,format=qcow2

# 注意：设备直通后 Host 无法再使用该 GPU
# 如果只有一张 GPU，Host 需要无头模式或使用其他显示设备
```

### 对比表格

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                      三种 GPU 虚拟化方案全方位对比                                   │
├────────────┬──────────────────┬──────────────────┬──────────────────┬────────────────┤
│  维度       │  全虚拟化         │  API 转发         │  SR-IOV          │  设备直通       │
├────────────┼──────────────────┼──────────────────┼──────────────────┼────────────────┤
│ 实现方式    │ 软件模拟 GPU      │ API 拦截转发      │ 硬件虚拟化 VF    │ 整个设备直通    │
│ Guest 驱动  │ 原生/虚拟驱动     │ 专用前端驱动      │ 轻量 VF 驱动     │ 原生驱动        │
│ 性能(计算)  │ 1-5%             │ 60-80%           │ 90-95%           │ 95-100%        │
│ 性能(图形)  │ 1-5%             │ 50-70%           │ 90-95%           │ 95-100%        │
│ 性能(显存)  │ 软件模拟          │ 通过命令转发      │ 直接访问          │ 直接访问        │
│ CPU 开销    │ 高 (VM-Exit 多)   │ 中 (API 编码)     │ 低 (硬件处理)    │ 低(少量陷入)    │
│ API 兼容性  │ 完全              │ 部分              │ 完全              │ 完全            │
│ VM 密度     │ 高 (不限)         │ 高 (不限)         │ 中 (受限资源)    │ 低 (1:1)        │
│ 安全性      │ 高 (完全隔离)     │ 中 (共享 Host)    │ 高 (硬件隔离)    │ 高 (独占设备)    │
│ 热迁移      │ 支持              │ 支持              │ 有限              │ 不支持           │
│ VMM 软件    │ QEMU/KVM         │ QEMU/KVM         │ KVM/Libvirt      │ KVM/Libvirt     │
│ 典型方案    │ virtio-gpu        │ VirGL            │ AMD SR-IOV       │ VFIO             │
│ 硬件需求    │ 无                │ 无                │ 支持 SR-IOV      │ IOMMU            │
│ 适用场景    │ VDI、控制台        │ 云游戏、桌面      │ AI训练、高性能    │ 性能敏感型工作   │
│ 优点        │ 兼容性好          │ 性能/密度均衡     │ 高性能隔离        │ 原生性能         │
│ 缺点        │ 性能极差          │ API 覆盖不全      │ 硬件依赖          │ 不能共享         │
└────────────┴──────────────────┴──────────────────┴──────────────────┴────────────────┘
```

### 案例分析

#### 案例一：云游戏平台方案选型

**需求：** 一个云游戏平台需要在一台服务器上同时运行 8 个游戏实例，每个实例需要独立的 3D 渲染能力，目标 1080p@60fps。

**方案对比分析：**

```python
#!/usr/bin/env python3
# 云游戏平台 GPU 虚拟化方案评估

class GPU_Virt_Evaluator:
    def __init__(self):
        self.scenarios = {}
    
    def add_scenario(self, name, req_gpu_perf, req_vram_gb, vm_count):
        self.scenarios[name] = {
            'gpu_perf': req_gpu_perf,
            'vram': req_vram_gb * vm_count,
            'vms': vm_count
        }
    
    def evaluate_full_virt(self, scenario):
        """全虚拟化方案评估"""
        # 每 VM 性能 = 原生 × 5% × VRAM 带宽因子
        perf_per_vm = scenario['gpu_perf'] * 0.03
        return {
            '方案': '全虚拟化 (virtio-gpu)',
            '每 VM 性能': f"{perf_per_vm:.1f} FPS",
            '是否达标': '否 (性能不足)',
            '问题': '软件渲染无法达到 60fps 要求',
        }
    
    def evaluate_api_forward(self, scenario):
        """API 转发方案评估"""
        # 每 VM 性能损失约 40%
        perf_per_vm = scenario['gpu_perf'] * 0.60 / scenario['vms']
        vram_needed = scenario['vram']
        return {
            '方案': 'API 转发 (VirGL)',
            '每 VM 性能': f"{perf_per_vm:.1f} FPS",
            '总显存需求': f"{vram_needed} GB",
            '是否达标': '是' if perf_per_vm >= 60 else '否',
            '备注': f"需要 GPU 原生性能 ≥ {60 * scenario['vms'] / 0.6:.0f} FPS"
        }
    
    def evaluate_sriov(self, scenario):
        """SR-IOV 方案评估"""
        # 每 VF 性能接近原生，但有硬件调度开销
        perf_per_vm = scenario['gpu_perf'] * 0.92 / scenario['vms']
        vram_per_vm = scenario['vram'] / scenario['vms']
        return {
            '方案': 'SR-IOV',
            '每 VM 性能': f"{perf_per_vm:.1f} FPS",
            '每 VM 显存': f"{vram_per_vm:.0f} GB",
            '是否达标': '是' if perf_per_vm >= 60 else '否',
            '限制': f"GPU 需支持 {scenario['vms']} 个 VF"
        }
    
    def evaluate_passthrough(self, scenario):
        """设备直通方案评估"""
        return {
            '方案': '设备直通 (VFIO)',
            '每 VM 性能': f"{scenario['gpu_perf']:.0f} FPS",
            '是否达标': '是',
            '限制': f"需要 {scenario['vms']} 张 GPU，成本过高"
        }

# 评估：8 个 1080p@60fps 游戏实例
evaluator = GPU_Virt_Evaluator()
evaluator.add_scenario('云游戏', req_gpu_perf=960,  # 原生 GPU 960fps@1080p
                        req_vram_gb=4, vms=8)

for name, scenario in evaluator.scenarios.items():
    print(f"\n=== {name} 场景 ===")
    for eval_func in [evaluator.evaluate_full_virt,
                       evaluator.evaluate_api_forward,
                       evaluator.evaluate_sriov,
                       evaluator.evaluate_passthrough]:
        result = eval_func(scenario)
        print(f"\n{result['方案']}:")
        for k, v in result.items():
            if k != '方案':
                print(f"  {k}: {v}")
```

**评估结论：**
- 全虚拟化：完全无法满足需求
- API 转发：需要原生 GPU 达到 ~800fps，高端 GPU 可满足
- SR-IOV：需要 GPU 支持 8 个 VF 且有足够显存，最推荐
- 设备直通：需要 8 张 GPU，成本不可接受

**最终推荐：** SR-IOV（如硬件支持）或 API 转发（折衷方案）

#### 案例二：AI 训练平台 GPU 共享

**需求：** 4 个 AI 团队共享一张 80GB 显存的 GPU，每个团队需要独立环境运行 PyTorch 训练任务。

```bash
#!/bin/bash
# AI 训练平台 GPU 共享方案

echo "AI 训练平台 GPU 共享方案对比"
echo "============================"

# 场景参数
GPU_NAME="AMD Instinct MI250"
VRAM_TOTAL=131072  # 128GB in MB
TEAMS=4
VRAM_PER_TEAM=$((VRAM_TOTAL / (TEAMS + 1)))  # 保留部分给 Host

echo "GPU: ${GPU_NAME}"
echo "总显存: $((VRAM_TOTAL / 1024))GB"
echo "团队数: ${TEAMS}"
echo "每团队可用: $((VRAM_PER_TEAM / 1024))GB"
echo ""

echo "方案一: API 转发 (VirGL)"
echo "  优点: 不需要硬件 SR-IOV 支持"
echo "  缺点: PyTorch 可能不兼容 VirGL"
echo "  性能: 计算任务 60-80%"
echo "  适用: 仅推理场景"
echo ""

echo "方案二: SR-IOV"
echo "  优点: 完全兼容 PyTorch/ROCm"
echo "  缺点: 需要 GPU 支持 SR-IOV"
echo "  性能: 90-95%"
echo "  适用: 训练 + 推理"
echo "  需检查: GPU SR-IOV 能力"
lspci -vvv | grep -A 20 "SR-IOV" | head -5
echo ""

echo "方案三: 设备直通"
echo "  优点: 完全原生性能"
echo "  缺点: 每团队需独立 GPU"
echo "  成本: ${TEAMS}x GPU"
echo ""

echo "方案四: 容器化共享 (非虚拟化)"
echo "  优点: 零虚拟化开销"
echo "  缺点: 隔离性弱"
echo "  工具: nvidia-docker / rocm-docker"
echo "  如果使用 MPS (NVIDIA) 或 ROCm MPS:"
echo "   - 显存共享"
echo "   - 计算资源时分复用"
echo "   - 隔离性较差"
```

### 相关链接

- [QEMU VirtIO-GPU 文档](https://www.qemu.org/docs/master/system/devices/virtio-gpu.html)
- [VirGL 项目](https://virgil3d.github.io/)
- [AMDGPU SR-IOV 文档](https://docs.amd.com/bundle/AMD-Instinct-SR-IOV-Guide)
- [VFIO/设备直通文档](https://www.kernel.org/doc/html/latest/driver-api/vfio.html)
- [NVIDIA vGPU/MDEV 文档](https://docs.nvidia.com/grid/)

### 今日小结

GPU 虚拟化的三大方案各有适用场景：

1. **全虚拟化**：性能最低但兼容性最好，适合基本的 2D 显示需求
2. **API 转发**：性能和共享度均衡，适合云游戏和虚拟桌面
3. **SR-IOV**：性能最接近原生，适合高性能计算和 AI 训练
4. **设备直通**：性能最佳但无法共享，适合单 VM 高性能场景

选择方案时需综合考虑：性能需求、VM 密度、API 兼容性、硬件支持和安全隔离要求。

### 扩展思考

1. 未来 GPU 虚拟化的发展方向是什么？是否会出现统一的虚拟化标准？
2. 对于 GPU 驱动测试而言，哪种虚拟化方案最适合构建测试环境？
3. 如何在虚拟化环境中准确测量和评估 GPU 性能损失？
4. 容器化（Docker/ROCm）与虚拟化方案在 GPU 共享上有何本质区别？各有什么优劣？
