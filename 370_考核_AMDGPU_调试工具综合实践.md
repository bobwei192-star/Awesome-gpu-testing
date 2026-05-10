# [考核] AMDGPU 调试工具综合实践

## 考核说明

- 本考核为 AMDGPU 调试工具的综合实践，总分 100 分
- 考核时间为 120 分钟
- 所有操作需在测试环境或虚拟机中完成
- 提交方式：将操作命令和输出截图整理为报告

## 第一部分：基础知识（30 分）

### 一、选择题（每题 2 分，共 10 分）

1. amdgpu_regs debugfs 接口使用的寄存器访问方式是：
   A. SMN 间接访问
   B. MMIO 直接访问
   C. PCIE 配置空间访问
   D. ACPI 访问

2. SMN 寄存器访问相比 MMIO 的主要特点是：
   A. 访问速度更快
   B. 可以访问完整的 32-bit GPU 地址空间
   C. 不需要锁保护
   D. 支持字节级访问

3. umr 工具中用于捕获着色器波形的参数是：
   A. --perf
   B. --waves
   C. --trace
   D. --scan

4. PCIE 配置空间中用于读取链路宽度和速度的偏移是：
   A. 0x00
   B. 0x10
   C. 0x42
   D. 0x60

5. 以下哪个不是 umr 工具的功能：
   A. 寄存器读写
   B. 波形反汇编
   C. 内核模块编译
   D. 性能计数器读取

### 二、判断题（每题 2 分，共 10 分）

1. [ ] MMIO 寄存器访问使用 ioremap 映射的虚拟地址，不需要锁保护。
2. [ ] SMN 总线延迟约为 MMIO 的 10-20 倍。
3. [ ] umr 的波形转储功能可以显示每个着色器波形的 VGPR 状态。
4. [ ] PCIE 配置空间的 AER 能力寄存器用于电源管理。
5. [ ] amdgpu_regs 只能读取单个寄存器，不能批量读取。

### 三、简答题（每题 5 分，共 10 分）

1. 简述 MMIO、SMN、PCIE Config Space 三种寄存器访问方式的应用场景选择和性能差异。

2. 说明 umr 工具的 --perf 参数如何用于 GPU 性能分析，列出至少 3 个可读的性能计数器及其含义。

## 第二部分：实践操作（40 分）

### 四、寄存器读取实践（15 分）

在测试环境中完成以下操作并记录结果：

1. （5 分）使用 amdgpu_regs debugfs 接口读取 DCN OTG 的以下寄存器，并说明每个值的含义：
   - OTG_MASTER_EN (偏移 0x1A000)
   - OTG_H_TOTAL (偏移 0x1A008)
   - OTG_V_TOTAL (偏移 0x1A00C)

2. （5 分）使用 amdgpu_smn_regs 读取 SMU 温度传感器寄存器（偏移 0x13A00000），并记录温度值。

3. （5 分）使用 amdgpu_pcie_cap 读取 PCIE 链路状态（偏移 0x42），解析出当前链路宽度和速度。

### 五、umr 工具使用（15 分）

完成以下 umr 操作并记录结果：

1. （5 分）使用 umr -L 列出系统 GPU 和 IP 块信息。
2. （5 分）使用 umr -S dcn 扫描 DCN 块的寄存器并记录 OTG_MASTER_EN 的值。
3. （5 分）使用 umr --perf 读取 GFX 利用率和像素填充率。

### 六、问题排查实践（10 分）

场景：系统启动后显示器无信号输出，但 GPU 驱动已成功加载。

1. （5 分）设计一个使用 amdgpu_regs 的排查步骤，至少 3 个检查点。
2. （5 分）设计一个使用 umr 的排查步骤，至少 2 个检查点。

## 第三部分：综合案例分析（30 分）

### 七、GPU Hang 分析（15 分）

场景：运行 3D 应用时 GPU 无响应，内核日志显示 "GPU hang detected"。

```bash
dmesg 输出:
[ 1234.567] [drm] GPU hang detected on GFX pipe
[ 1234.567] [drm] GRBM_STATUS=0xA0000002
[ 1234.567] [drm] CP_STATUS=0x80000000
[ 1234.567] [drm] GFX is busy, no progress for 2 seconds
[ 1234.567] [drm] Starting GPU recovery...
```

问题：
1. （5 分）根据 GRBM_STATUS=0xA0000002，分析 GPU hang 的可能原因（参考 GRBM_STATUS 的位定义）。
2. （5 分）使用 umr 的波形转储功能，写出捕获当前波形的命令，并说明如何判断波形是否卡住。
3. （5 分）设计一个 umr --perf 计数器组合，用于监控 GFX pipe 的活跃情况和内存访问模式。

### 八、性能分析（15 分）

场景：3D 渲染性能低于预期（预期 60fps，实际 35fps）。

```bash
# 使用 umr 采集的性能数据
sudo umr --perf enable gfx_BUSY,gfx_VALID,gfx_CACHE_HIT,gfx_MEM_READ
sudo umr --perf sample 2000

输出:
  gfx_BUSY:      62%
  gfx_VALID:     350 MPixels/s
  gfx_CACHE_HIT: 45%
  gfx_MEM_READ:  1250 MB/s
```

问题：
1. （5 分）根据数据，判断性能瓶颈类型（计算密集型/内存带宽受限/着色器效率问题）。
2. （5 分）编写一个批量寄存器读取脚本（bash 或 Python），自动检查 PCIE 链路状态、GPU 温度和 GFX 时钟频率，并将结果写入日志文件。
3. （5 分）分析为什么 gfx_BUSY 只有 62% 而不是 100%，可能的原因是什么？

## 附加题（20 分）

9. （10 分）设计一个自动化 GPU 健康检查脚本，要求：
   - 检查 GPU 温度是否在安全范围内（< 95°C）
   - 检查 PCIE 链路是否降级
   - 检查 GFX 利用率和内存带宽
   - 记录到日志文件并支持报警
   可以使用 amdgpu_regs、umr 或 sysfs 接口。

10. （10 分）阅读 AMDGPU 源码后，回答问题：
  在 amdgpu_device.c 中，RREG32 和 WREG32 宏的实际实现是什么？它们如何保证访问的原子性和正确性？在虚拟化环境中，这些函数有什么特殊处理？