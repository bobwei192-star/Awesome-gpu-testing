# 第318天：GPU 功耗剖析工具：rocm-smi / amdgpu_top

## 内容简介
本课程深入讲解 AMD GPU 的两大核心功耗剖析工具：ROCm-SMI（Radeon Open Compute System Management Interface）和 amdgpu_top。涵盖工具架构、安装配置、命令行使用、API 编程接口、性能监控、功耗分析、故障排查等内容，帮助学员全面掌握 GPU 功耗监测和性能剖析技能。

## 学习目标
1. 理解 ROCm-SMI 和 amdgpu_top 的架构设计与工作原理
2. 掌握 rocm-smi 的完整命令行操作和常用监控场景
3. 学会使用 amdgpu_top 进行实时 GPU 性能剖析
4. 理解工具输出的各项指标含义及其背后硬件/驱动数据来源
5. 能够编写脚本和程序利用工具的 API 进行自定义监控
6. 掌握多 GPU 环境下的功耗监测和管理方法

## 第一节：ROCm-SMI 概述

### 1.1 ROCm-SMI 是什么

ROCm-SMI（Radeon Open Compute System Management Interface）是 AMD 官方提供的 GPU 系统管理命令行工具，类似于 NVIDIA 的 nvidia-smi。它通过封装 AMDGPU 内核驱动的 sysfs 接口和 KFD（Kernel Fusion Driver）接口，提供统一、易用的 GPU 管理能力。

```
┌────────────────────────────────────────────────────────────┐
│                    用户/管理员                              │
└────────────────────────┬───────────────────────────────────┘
                         │
┌────────────────────────▼───────────────────────────────────┐
│                    rocm-smi CLI                             │
│   rocm-smi --showpower --showtemp --showclock ...           │
└────────────┬───────────────────────────────┬────────────────┘
             │                               │
┌────────────▼──────────────┐  ┌─────────────▼──────────────┐
│     Python 封装            │  │     C/C++ 底层库            │
│   rocm_smi_lib.py         │  │   librocm_smi64.so          │
└────────────┬──────────────┘  └─────────────┬──────────────┘
             │                               │
┌────────────▼───────────────────────────────▼────────────────┐
│                  AMDGPU 内核驱动                              │
│    /sys/class/drm/card*/device/  +  /dev/kfd                 │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                   AMD GPU 硬件                                │
│     GFX Core  /  HBM/Memory  /  SMU  /  VCN  /  DCN         │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 架构组件

| 组件 | 描述 | 技术栈 |
|------|------|--------|
| rocm-smi CLI | 命令行接口工具 | Python/C++ |
| librocm_smi64.so | 核心库，封装 sysfs 访问 | C/C++ |
| rocm_smi_lib.py | Python 封装库 | Python CFFI |
| config 文件 | 输出格式和监控配置 | INI/JSON |
| 单元测试 | 工具验证测试套件 | Python unittest |

### 1.3 安装方式

**方式 1：通过 ROCm 包管理器安装**
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install rocm-smi-lib

# RHEL/CentOS
sudo yum install rocm-smi-lib

# Arch Linux
sudo pacman -S rocm-smi-lib
```

**方式 2：从源码构建**
```bash
git clone https://github.com/RadeonOpenCompute/rocm_smi_lib.git
cd rocm_smi_lib
mkdir build && cd build
cmake ..
make -j$(nproc)
sudo make install
```

**方式 3：pip 安装（仅 Python 绑定）**
```bash
pip install rocm-smi-lib
```

### 1.4 验证安装
```bash
# 检查版本
rocm-smi --version

# 检查可用 GPU
rocm-smi --showallinfo

# 检查帮助信息
rocm-smi --help
```

## 第二节：ROCm-SMI 命令行详解

### 2.1 基本用法和输出格式

```bash
# 默认输出（简洁模式）
rocm-smi

# 完整信息输出
rocm-smi --showallinfo

# JSON 格式输出
rocm-smi --json --showallinfo

# CSV 格式输出
rocm-smi --csv --showpower --showtemp
```

**默认输出示例：**
```
======================= ROCm System Management Interface =======================
GPU  Temp (Edge)  Temp (Junction)  Power    Power Cap  VRAM%  GPU%  SCLK  MCLK
0    45.0°C       57.0°C           125.0W   250.0W      35%    45%   1500  1000
1    42.0°C       53.0°C           98.0W    250.0W      20%    30%   1200  1000
================================================================================
```

### 2.2 功耗和电源管理相关选项

```bash
# 显示功耗信息
rocm-smi --showpower

# 输出示例：
GPU  Power (Average)  Power Cap  Power Cap Max  Power Cap Min
0    125.0W            250.0W     300.0W         50.0W
1    98.0W             250.0W     300.0W         50.0W

# 显示 GPU 使用率
rocm-smi --showuse

# 显示电压
rocm-smi --showvoltage

# 显示时钟频率
rocm-smi --showclock

# 查看所有与功耗相关的信息
rocm-smi --showpower --showvoltage --showclock --showtemp
```

### 2.3 温度监控

```bash
# 显示温度
rocm-smi --showtemp

# 温度输出示例：
GPU  Temperature (Edge)  Temperature (Junction)  Temperature (Memory)  Temperature (HBM 0)
0    45.0°C              57.0°C                  62.0°C                N/A
1    42.0°C              53.0°C                  58.0°C                N/A

# 显示极限温度
rocm-smi --showtemp --showtemp极限

# 持续监控温度
rocm-smi --showtemp --loop --loopdelay 2
```

### 2.4 风扇控制

```bash
# 显示风扇速度
rocm-smi --showfan

GPU  Fan Speed (%)
0    35%
1    28%

# 设置风扇速度（手动模式）
rocm-smi --setfan 50 -i 0

# 重置风扇控制为自动
rocm-smi --resetfans -i 0

# 获取风扇转速（RPM）
rocm-smi --showfanrpm
```

### 2.5 性能等级控制

```bash
# 显示当前性能等级
rocm-smi --showperflevel

GPU  Perf Level
0    auto
1    auto

# 设置性能等级
rocm-smi --setperflevel low -i 0    # 低功耗
rocm-smi --setperflevel high -i 0   # 高性能
rocm-smi --setperflevel manual -i 0 # 手动
rocm-smi --setperflevel auto -i 0   # 自动

# 查看支持的性能等级
rocm-smi --showperflevel --list Supported
```

### 2.6 OverDrive 超频/降频控制

```bash
# 显示当前 OD 配置
rocm-smi --showod

GPU  SCLK Clock  MCLK Clock  VDDC
0    500-2500    1000        800mV
1    500-2500    1000        800mV

# 设置 SCLK 频率
rocm-smi --setsclk 0,1800 -i 0  # 设置 GPU 0 的 SCLK 为 1800MHz

# 设置 MCLK 频率
rocm-smi --setmclk 0,1200 -i 0  # 设置 GPU 0 的 MCLK 为 1200MHz

# 设置电压（VDDC）
rocm-smi --setvolt 0,900 -i 0   # 设置 GPU 0 的电压为 900mV

# 重置 OD 配置
rocm-smi --resetod -i 0
```

### 2.7 功耗配置文件管理

```bash
# 显示当前功耗配置文件
rocm-smi --showprofile

GPU  Profile
0    CUSTOM (0)
1    CUSTOM (0)

# 列出所有可用配置
rocm-smi --showprofile --list Profiles

# 设置配置文件
rocm-smi --setprofile 0    # CUSTOM - 自定义
rocm-smi --setprofile 1    # VR - 虚拟现实
rocm-smi --setprofile 2    # POWER_SAVING - 省电
rocm-smi --setprofile 3    # VIDEO - 视频
rocm-smi --setprofile 4    # COMPUTE - 计算
rocm-smi --setprofile 5    # 3D_FULL_SCREEN - 3D 全屏
```

### 2.8 内存和显存信息

```bash
# 显示显存使用
rocm-smi --showmeminfo

GPU  VRAM Total  VRAM Used  VRAM Free  VRAM%  Vis_VRAM Total  Vis_VRAM Used
0    16GB        5.6GB      10.4GB     35%    16GB            5.6GB
1    16GB        3.2GB      12.8GB     20%    16GB            3.2GB

# 显示显存类型和带宽
rocm-smi --showmeminfo vram

# 显示内存温度
rocm-smi --showtemp --showmemtemp
```

### 2.9 进程信息

```bash
# 显示运行在 GPU 上的进程
rocm-smi --showprocessinfo

GPU  PID   Name          VRAM Used  GPU%  Type
0    12345  python3       2.1GB      45%   Compute
0    12346  chrome        0.5GB      5%    Graphics
1    23456  python3       1.8GB      30%   Compute

# 显示特定 GPU 上的进程
rocm-smi --showprocessinfo -i 0

# 持续监控进程
rocm-smi --showprocessinfo --loop
```

### 2.10 多 GPU 和显示管理

```bash
# 指定 GPU 范围
rocm-smi --showpower -i 0,1,2    # 指定多个 GPU
rocm-smi --showtemp -i 0-3       # 指定范围

# 显示所有 GPU 摘要
rocm-smi --showallinfo

# 显示拓扑信息
rocm-smi --showtopo

# 显示 PCIe 信息
rocm-smi --showpcie

GPU  PCIe Speed  PCIe Width  PCIe Generation
0    16 GT/s     16          Gen4
1    16 GT/s     8           Gen4
```

## 第三节：ROCm-SMI Python API

### 3.1 基本 Python 封装

```python
#!/usr/bin/env python3
# rocm_smi_basic.py - ROCm-SMI Python API 基础使用

import pyrocm_smi as rsmi
import json

def init_rsmi():
    """初始化 ROCm-SMI 库"""
    try:
        rsmi.rsmi_init(0)
        return True
    except rsmi.rsmi_status_t.RSMI_STATUS_INIT_ERROR:
        print("Failed to initialize ROCm-SMI")
        return False

def shutdown_rsmi():
    """关闭 ROCm-SMI 库"""
    rsmi.rsmi_shut_down()

def get_device_count():
    """获取 GPU 设备数量"""
    count = rsmi.rsmi_num_monitor_devices()
    return count

def get_device_name(device_id):
    """获取 GPU 名称"""
    try:
        name = rsmi.rsmi_dev_name_get(device_id)
        return name
    except Exception as e:
        return f"Unknown: {e}"

def get_device_temp(device_id, sensor=0):
    """获取 GPU 温度（sensor: 0=edge, 1=junction, 2=memory）"""
    try:
        temp = rsmi.rsmi_dev_temp_metric_get(device_id, sensor, rsmi.RSMI_TEMP_CURRENT)
        return temp / 1000.0  # 转换为摄氏度
    except Exception as e:
        return None

def get_device_power(device_id):
    """获取 GPU 当前功耗（瓦特）"""
    try:
        power = rsmi.rsmi_dev_power_ave_get(device_id, 0)
        return power / 1000000.0  # 微瓦 -> 瓦
    except Exception as e:
        return None

def get_device_power_cap(device_id):
    """获取 GPU 功耗上限"""
    try:
        power_cap = rsmi.rsmi_dev_power_cap_get(device_id, 0)
        return power_cap / 1000000.0
    except Exception as e:
        return None

def get_device_usage(device_id):
    """获取 GPU 使用率"""
    try:
        busy = rsmi.rsmi_dev_busy_percent_get(device_id)
        return busy
    except Exception as e:
        return None

def get_device_clocks(device_id):
    """获取 GPU 时钟频率"""
    clocks = {}
    try:
        # GFX 时钟
        gfx_clk = rsmi.rsmi_dev_gpu_clk_freq_get(device_id, 0)  # 0 = GFXCLK
        clocks['gfx'] = gfx_clk / 1000000.0
    except:
        clocks['gfx'] = None
    
    try:
        # 显存时钟
        mem_clk = rsmi.rsmi_dev_gpu_clk_freq_get(device_id, 1)  # 1 = MEMCLK
        clocks['mem'] = mem_clk / 1000000.0
    except:
        clocks['mem'] = None
    
    return clocks

def get_device_vram_usage(device_id):
    """获取显存使用信息"""
    try:
        vram = rsmi.rsmi_dev_memory_usage_get(device_id, rsmi.RSMI_MEM_TYPE_VRAM)
        total = vram[0] / (1024**3)  # 转换为 GB
        used = vram[1] / (1024**3)
        return {'total_gb': total, 'used_gb': used, 'percent': (used/total)*100 if total > 0 else 0}
    except Exception as e:
        return None

def get_fan_speed(device_id):
    """获取风扇速度"""
    try:
        fan_speed = rsmi.rsmi_dev_fan_speed_get(device_id, 0)
        return fan_speed
    except Exception as e:
        return None

def get_device_pcie_info(device_id):
    """获取 PCIe 信息"""
    info = {}
    try:
        pcie_bandwidth = rsmi.rsmi_dev_pcie_bandwidth_get(device_id)
        info['lanes'] = pcie_bandwidth[1]
        info['generation'] = pcie_bandwidth[0]
    except:
        pass
    return info


def monitor_all():
    """监控所有 GPU 并输出 JSON"""
    if not init_rsmi():
        return
    
    count = get_device_count()
    print(f"Detected {count} GPU(s)\n")
    
    result = []
    for i in range(count):
        info = {
            'device_id': i,
            'name': get_device_name(i),
            'temperature_c': get_device_temp(i),
            'junction_temp_c': get_device_temp(i, 1),
            'power_w': get_device_power(i),
            'power_cap_w': get_device_power_cap(i),
            'usage_percent': get_device_usage(i),
            'clocks_mhz': get_device_clocks(i),
            'vram': get_device_vram_usage(i),
            'fan_speed': get_fan_speed(i),
            'pcie': get_device_pcie_info(i),
        }
        result.append(info)
        
        print(f"=== GPU {i} ===")
        print(f"  Name: {info['name']}")
        print(f"  Temperature: {info['temperature_c']:.1f}°C (edge) / {info['junction_temp_c']:.1f}°C (junction)")
        print(f"  Power: {info['power_w']:.1f}W / {info['power_cap_w']:.1f}W")
        print(f"  Usage: {info['usage_percent']}%")
        print(f"  Clocks: GFX={info['clocks_mhz']['gfx']} MHz, MEM={info['clocks_mhz']['mem']} MHz")
        if info['vram']:
            print(f"  VRAM: {info['vram']['used_gb']:.1f}GB / {info['vram']['total_gb']:.1f}GB ({info['vram']['percent']:.1f}%)")
        print(f"  Fan: {info['fan_speed']}%")
        print()
    
    print("--- JSON Output ---")
    print(json.dumps(result, indent=2))
    
    shutdown_rsmi()

if __name__ == "__main__":
    monitor_all()
```

### 3.2 持续监控类

```python
#!/usr/bin/env python3
# rsmi_monitor.py - 持续 GPU 监控

import pyrocm_smi as rsmi
import time
import csv
import signal
import sys
from datetime import datetime

class RSMI_Monitor:
    """ROCm-SMI 持续监控器"""
    
    def __init__(self, interval=2.0, output_file=None):
        self.interval = interval
        self.output_file = output_file
        self.running = True
        self.csv_writer = None
        self.csv_file = None
        
        # 初始化
        rsmi.rsmi_init(0)
        self.gpu_count = rsmi.rsmi_num_monitor_devices()
        
        signal.signal(signal.SIGINT, self.signal_handler)
        signal.signal(signal.SIGTERM, self.signal_handler)
    
    def signal_handler(self, signum, frame):
        print("\nStopping monitor...")
        self.running = False
    
    def setup_csv(self):
        if self.output_file:
            self.csv_file = open(self.output_file, 'w', newline='')
            self.csv_writer = csv.writer(self.csv_file)
            header = ['timestamp', 'gpu_id']
            header.extend(['temp_edge', 'temp_junction', 'power_w', 'usage_pct', 'gfx_clk', 'mem_clk', 'fan_pct', 'vram_used_gb'])
            self.csv_writer.writerow(header)
    
    def close_csv(self):
        if self.csv_file:
            self.csv_file.close()
    
    def read_gpu(self, device_id):
        """读取单 GPU 所有指标"""
        data = {
            'timestamp': datetime.now().isoformat(),
            'gpu_id': device_id,
        }
        
        try:
            temp = rsmi.rsmi_dev_temp_metric_get(device_id, 0, rsmi.RSMI_TEMP_CURRENT)
            data['temp_edge'] = temp / 1000.0
        except:
            data['temp_edge'] = None
        
        try:
            temp_j = rsmi.rsmi_dev_temp_metric_get(device_id, 1, rsmi.RSMI_TEMP_CURRENT)
            data['temp_junction'] = temp_j / 1000.0
        except:
            data['temp_junction'] = None
        
        try:
            power = rsmi.rsmi_dev_power_ave_get(device_id, 0)
            data['power_w'] = power / 1000000.0
        except:
            data['power_w'] = None
        
        try:
            busy = rsmi.rsmi_dev_busy_percent_get(device_id)
            data['usage_pct'] = busy
        except:
            data['usage_pct'] = None
        
        try:
            gfx_clk = rsmi.rsmi_dev_gpu_clk_freq_get(device_id, 0)
            data['gfx_clk'] = gfx_clk / 1000000.0
        except:
            data['gfx_clk'] = None
        
        try:
            mem_clk = rsmi.rsmi_dev_gpu_clk_freq_get(device_id, 1)
            data['mem_clk'] = mem_clk / 1000000.0
        except:
            data['mem_clk'] = None
        
        try:
            fan = rsmi.rsmi_dev_fan_speed_get(device_id, 0)
            data['fan_pct'] = fan
        except:
            data['fan_pct'] = None
        
        try:
            vram = rsmi.rsmi_dev_memory_usage_get(device_id, rsmi.RSMI_MEM_TYPE_VRAM)
            data['vram_used_gb'] = vram[1] / (1024**3)
        except:
            data['vram_used_gb'] = None
        
        return data
    
    def print_status(self, data_list):
        """打印状态行"""
        print(f"[{data_list[0]['timestamp']}] ", end='')
        
        for d in data_list:
            gpu_id = d['gpu_id']
            temp = d['temp_edge'] or 0
            power = d['power_w'] or 0
            usage = d['usage_pct'] or 0
            gfx = d['gfx_clk'] or 0
            
            print(f"GPU{gpu_id}: {temp:.0f}°C | {power:.0f}W | {usage}% | {gfx:.0f}MHz | ", end='')
        
        print()
    
    def run(self):
        print(f"Starting RSMI Monitor ({self.gpu_count} GPUs, interval={self.interval}s)")
        print("Press Ctrl+C to stop\n")
        
        self.setup_csv()
        
        while self.running:
            all_data = []
            
            for gpu_id in range(self.gpu_count):
                data = self.read_gpu(gpu_id)
                all_data.append(data)
                
                if self.csv_writer:
                    row = [data['timestamp'], data['gpu_id'],
                           data['temp_edge'], data['temp_junction'],
                           data['power_w'], data['usage_pct'],
                           data['gfx_clk'], data['mem_clk'],
                           data['fan_pct'], data['vram_used_gb']]
                    self.csv_writer.writerow(row)
            
            self.print_status(all_data)
            
            if self.output_file:
                self.csv_file.flush()
            
            time.sleep(self.interval)
        
        self.close_csv()
        rsmi.rsmi_shut_down()
        print("Monitor stopped.")


if __name__ == "__main__":
    import argparse
    
    parser = argparse.ArgumentParser(description="ROCm-SMI GPU Monitor")
    parser.add_argument('-i', '--interval', type=float, default=2.0, help='Polling interval (seconds)')
    parser.add_argument('-o', '--output', type=str, help='Output CSV file')
    parser.add_argument('-p', '--performance', action='store_true', help='Show performance data')
    
    args = parser.parse_args()
    
    monitor = RSMI_Monitor(interval=args.interval, output_file=args.output)
    monitor.run()
```

### 3.4 ROCm-SMI 异常处理

```python
#!/usr/bin/env python3
# rsmi_error_handling.py - ROCm-SMI 错误处理示例

import pyrocm_smi as rsmi

class RSMIError(Exception):
    """ROCm-SMI 基本异常"""
    pass

class RSMIInitError(RSMIError):
    """初始化失败"""
    pass

class RSMIQueryError(RSMIError):
    """查询失败"""
    pass

def error_code_to_str(status):
    """将错误码转换为可读信息"""
    error_map = {
        0: "RSMI_STATUS_SUCCESS",
        1: "RSMI_STATUS_INVALID_ARGS",
        2: "RSMI_STATUS_BUSY",
        3: "RSMI_STATUS_NO_DATA",
        4: "RSMI_STATUS_PERMISSION",
        5: "RSMI_STATUS_INSUFFICIENT_SIZE",
        6: "RSMI_STATUS_UNEXPECTED_SIZE",
        7: "RSMI_STATUS_NOT_YET_IMPLEMENTED",
        8: "RSMI_STATUS_UNSUPPORTED",
        9: "RSMI_STATUS_FILE_ERROR",
        10: "RSMI_STATUS_SYSFS_ERROR",
        11: "RSMI_STATUS_INIT_ERROR",
        12: "RSMI_STATUS_NOT_INIT",
        13: "RSMI_STATUS_API_NOT_SUPPORTED",
        14: "RSMI_STATUS_OUT_OF_RESOURCES",
        15: "RSMI_STATUS_INTERNAL_EXCEPTION",
        16: "RSMI_STATUS_POWER_AND_THERMAL_FATAL",
        17: "RSMI_STATUS_RANGE_IS_NOT_SET",
        18: "RSMI_STATUS_LOAD_MODULE",
        19: "RSMI_STATUS_TIMEOUT",
        20: "RSMI_STATUS_IN_ACTIVE_USE",
        21: "RSMI_STATUS_NOT_FOUND",
        22: "RSMI_STATUS_RETRY",
        23: "RSMI_STATUS_NO_DEVICE",
        24: "RSMI_STATUS_ALREADY_SET",
        25: "RSMI_STATUS_ADDRESS_ALREADY_MAPPED",
    }
    return error_map.get(status, f"UNKNOWN_ERROR ({status})")


def safe_rsmi_query(func, *args, **kwargs):
    """安全执行 RSMI 查询"""
    try:
        result = func(*args, **kwargs)
        return result
    except rsmi.rsmi_status_t as e:
        err_code = e.value if hasattr(e, 'value') else e
        err_str = error_code_to_str(err_code)
        raise RSMIQueryError(f"Query failed: {err_str}") from e
    except Exception as e:
        raise RSMIQueryError(f"Unexpected error: {e}") from e


class SafeRSMI:
    """安全的 RSMI 封装"""
    
    def __init__(self):
        self.initialized = False
    
    def __enter__(self):
        try:
            rsmi.rsmi_init(0)
            self.initialized = True
        except Exception as e:
            raise RSMIInitError(f"Cannot initialize ROCm-SMI: {e}")
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        if self.initialized:
            rsmi.rsmi_shut_down()
    
    def get_device_info(self, device_id):
        info = {}
        
        try:
            info['name'] = safe_rsmi_query(rsmi.rsmi_dev_name_get, device_id)
        except RSMIQueryError:
            info['name'] = 'Unknown'
        
        # 尝试各种查询，失败则记录 None
        query_tasks = [
            ('temp_edge', lambda: safe_rsmi_query(rsmi.rsmi_dev_temp_metric_get, device_id, 0, rsmi.RSMI_TEMP_CURRENT) / 1000.0),
            ('temp_junction', lambda: safe_rsmi_query(rsmi.rsmi_dev_temp_metric_get, device_id, 1, rsmi.RSMI_TEMP_CURRENT) / 1000.0),
            ('power_ave', lambda: safe_rsmi_query(rsmi.rsmi_dev_power_ave_get, device_id, 0) / 1000000.0),
            ('power_cap', lambda: safe_rsmi_query(rsmi.rsmi_dev_power_cap_get, device_id, 0) / 1000000.0),
            ('busy_percent', lambda: safe_rsmi_query(rsmi.rsmi_dev_busy_percent_get, device_id)),
            ('gfx_clk', lambda: safe_rsmi_query(rsmi.rsmi_dev_gpu_clk_freq_get, device_id, 0) / 1000000.0),
            ('mem_clk', lambda: safe_rsmi_query(rsmi.rsmi_dev_gpu_clk_freq_get, device_id, 1) / 1000000.0),
            ('fan_speed', lambda: safe_rsmi_query(rsmi.rsmi_dev_fan_speed_get, device_id, 0)),
        ]
        
        for name, query in query_tasks:
            try:
                info[name] = query()
            except RSMIQueryError:
                info[name] = None
        
        return info


# 使用示例
if __name__ == "__main__":
    try:
        with SafeRSMI() as rsmi_client:
            for gpu_id in range(4):
                info = rsmi_client.get_device_info(gpu_id)
                print(f"GPU {gpu_id} ({info['name']}):")
                print(f"  Temp: {info.get('temp_edge', 'N/A')}°C")
                print(f"  Power: {info.get('power_ave', 'N/A')}W")
                print(f"  Busy: {info.get('busy_percent', 'N/A')}%")
                print()
    except RSMIInitError as e:
        print(f"Error: {e}")
        print("Is ROCm installed? Check: ldconfig -p | grep rocm")
```

## 第四节：amdgpu_top 工具

### 4.1 amdgpu_top 简介

`amdgpu_top` 是一个类似于 `htop`/`nvtop` 的实时 GPU 监控工具，专门针对 AMD GPU 设计，提供：

- 实时 GPU 使用率和时钟频率显示
- 温度、功耗、风扇转速监控
- 每个进程的 GPU 使用情况
- 彩色图形化界面
- 多 GPU 支持

```
┌─────────────────────────────────────────────────────────────────────┐
│                         amdgpu_top v1.2.3                            │
├─────────────────────────────────────────────────────────────────────┤
│ GPU0: AMD Radeon RX 7900 XTX                                        │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ GFX  ████████████████████████████░░░░ 78%  2450MHz              │ │
│ │ MEM  ██████████░░░░░░░░░░░░░░░░░░░░░░ 32%  1200MHz             │ │
│ │ PCIe ██████░░░░░░░░░░░░░░░░░░░░░░░░░░ 18%  Gen4 x16            │ │
│ │ Enc  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  0%                      │ │
│ │ Dec  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  0%                      │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│  Temp: 52°C  Power: 185W/355W  Fan: 35%  VRAM: 8.5/24.0GB          │
├─────────────────────────────────────────────────────────────────────┤
│ PID    NAME            GPU%  MEM%  TYPE                              │
│ 12345  python3          45%  12%   Compute                           │
│ 23456  Xorg              5%   2%   Graphics                          │
│ 34567  chrome            3%   1%   Graphics                          │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 安装 amdgpu_top

```bash
# 方法 1：从包管理器安装
sudo apt install amdgpu-top    # Debian/Ubuntu
sudo yum install amdgpu-top    # RHEL/CentOS

# 方法 2：从源码构建
git clone https://github.com/Umio-Yasuno/amdgpu_top.git
cd amdgpu_top
cargo build --release
sudo cp target/release/amdgpu_top /usr/local/bin/

# 方法 3：使用预编译二进制
# 从 GitHub Releases 下载
wget https://github.com/Umio-Yasuno/amdgpu_top/releases/latest/download/amdgpu_top-x86_64-unknown-linux-gnu.tar.gz
tar xzf amdgpu_top-x86_64-unknown-linux-gnu.tar.gz
sudo cp amdgpu_top /usr/local/bin/
```

### 4.3 基本使用

```bash
# 启动 amdgpu_top（默认显示所有 GPU）
amdgpu_top

# 指定刷新间隔
amdgpu_top -d 2  # 2 秒刷新一次

# 选择特定 GPU
amdgpu_top -g 0  # 只显示 GPU 0

# 显示 JSON 输出（适合脚本处理）
amdgpu_top -J

# CSV 输出模式
amdgpu_top -C

# 设置刷新次数后退出
amdgpu_top -n 10  # 刷新 10 次后退出

# 不显示 TUI，只打印一次状态
amdgpu_top -O
```

### 4.4 amdgpu_top 的命令行选项详解

```bash
# 帮助信息
amdgpu_top --help

# 显示版本
amdgpu_top -V

# 选项分类：

# 显示控制选项
amdgpu_top --no-color        # 禁用颜色
amdgpu_top --no-header       # 不显示标题
amdgpu_top --no-processes    # 不显示进程列表

# 输出格式选项
amdgpu_top -J --pretty       # JSON 格式（格式化）
amdgpu_top -C --csv          # CSV 格式
amdgpu_top -R --raw          # 原始格式

# 过滤选项
amdgpu_top -p 12345          # 只显示 PID 12345 的进程
amdgpu_top -u username       # 只显示特定用户的进程

# 性能选项
amdgpu_top --profile         # 性能分析模式
amdgpu_top --no-sort         # 不排序进程
```

### 4.5 JSON 输出解析

```bash
# 获取 JSON 格式数据
amdgpu_top -J | python3 -m json.tool

# 示例输出：
{
  "gpus": [
    {
      "index": 0,
      "name": "AMD Radeon RX 7900 XTX",
      "driver": "amdgpu",
      "temperature": {
        "edge": 45.0,
        "junction": 57.0,
        "memory": 62.0
      },
      "power": {
        "average": 125.0,
        "cap": 250.0
      },
      "clocks": {
        "gfx": 2450,
        "mem": 1200,
        "soc": 850
      },
      "usage": {
        "gfx": 78.0,
        "mem": 32.0,
        "pcie": 18.0
      },
      "fan": {
        "speed": 35,
        "rpm": 1200
      },
      "memory": {
        "total": 24564,
        "used": 8512,
        "free": 16052
      },
      "processes": [
        {
          "pid": 12345,
          "name": "python3",
          "gpu_usage": 45.0,
          "memory_usage": 3072,
          "type": "Compute"
        }
      ]
    }
  ]
}
```

### 4.6 用 Python 解析 amdgpu_top JSON 输出

```python
#!/usr/bin/env python3
# parse_amdgpu_top.py - 解析 amdgpu_top JSON 输出

import subprocess
import json
import time

def get_amdgpu_top_data(gpu_index=None):
    """获取 amdgpu_top 数据"""
    cmd = ['amdgpu_top', '-J']
    if gpu_index is not None:
        cmd.extend(['-g', str(gpu_index)])
    
    try:
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=5)
        if result.returncode == 0:
            return json.loads(result.stdout)
        else:
            print(f"Error: {result.stderr}")
            return None
    except subprocess.TimeoutExpired:
        print("Timeout getting amdgpu_top data")
        return None
    except FileNotFoundError:
        print("amdgpu_top not found. Install it first.")
        return None

def collect_timeseries(duration=60, interval=2):
    """收集时间序列数据"""
    data_points = []
    start_time = time.time()
    
    print(f"Collecting data for {duration}s at {interval}s intervals...")
    
    while time.time() - start_time < duration:
        data = get_amdgpu_top_data()
        if data and 'gpus' in data:
            timestamp = time.time() - start_time
            for gpu in data['gpus']:
                dp = {
                    'time': timestamp,
                    'gpu': gpu['index'],
                    'temp_edge': gpu.get('temperature', {}).get('edge'),
                    'power_avg': gpu.get('power', {}).get('average'),
                    'gfx_usage': gpu.get('usage', {}).get('gfx'),
                    'gfx_clock': gpu.get('clocks', {}).get('gfx'),
                    'vram_used': gpu.get('memory', {}).get('used'),
                }
                data_points.append(dp)
        
        time.sleep(interval)
    
    return data_points

def analyze_power_profile(data_points):
    """分析功耗特性"""
    if not data_points:
        return
    
    powers = [dp['power_avg'] for dp in data_points if dp['power_avg'] is not None]
    temps = [dp['temp_edge'] for dp in data_points if dp['temp_edge'] is not None]
    gfx_usage = [dp['gfx_usage'] for dp in data_points if dp['gfx_usage'] is not None]
    
    print("=== Power Profile Analysis ===")
    print(f"Duration: {data_points[-1]['time']:.1f}s")
    print(f"Data points: {len(data_points)}")
    
    if powers:
        print(f"\nPower (W):")
        print(f"  Avg: {sum(powers)/len(powers):.1f}")
        print(f"  Min: {min(powers):.1f}")
        print(f"  Max: {max(powers):.1f}")
        print(f"  Total: {sum(powers) * (data_points[1]['time'] - data_points[0]['time'] if len(data_points) > 1 else 1) / 3600:.3f} Wh")
    
    if temps:
        print(f"\nTemperature (°C):")
        print(f"  Avg: {sum(temps)/len(temps):.1f}")
        print(f"  Min: {min(temps):.1f}")
        print(f"  Max: {max(temps):.1f}")
    
    if gfx_usage:
        print(f"\nGPU Usage (%):")
        print(f"  Avg: {sum(gfx_usage)/len(gfx_usage):.1f}")
        print(f"  Min: {min(gfx_usage):.1f}")
        print(f"  Max: {max(gfx_usage):.1f}")
    
    # 计算能效
    if powers and gfx_usage:
        avg_power = sum(powers)/len(powers)
        avg_usage = sum(gfx_usage)/len(gfx_usage)
        if avg_power > 0:
            efficiency = avg_usage / avg_power
            print(f"\nPower Efficiency:")
            print(f"  Performance/Watt: {efficiency:.2f} %/W")
    
    print()

def detect_anomalies(data_points):
    """检测异常"""
    if len(data_points) < 5:
        return
    
    print("=== Anomaly Detection ===")
    
    # 温度异常检测
    temps = [dp['temp_edge'] for dp in data_points if dp['temp_edge'] is not None]
    if temps:
        avg_temp = sum(temps) / len(temps)
        max_temp = max(temps)
        min_temp = min(temps)
        
        if max_temp - min_temp > 30:
            print(f"Warning: Large temperature variation ({min_temp:.0f}°C to {max_temp:.0f}°C)")
        
        if max_temp > 95:
            print(f"CRITICAL: GPU temperature reached {max_temp:.0f}°C (near throttling threshold)")
    
    # 功耗突增检测
    if len(powers := [dp['power_avg'] for dp in data_points if dp['power_avg'] is not None]) > 2:
        for i in range(2, len(powers)):
            delta = powers[i] - powers[i-2]
            if delta > 50:
                print(f"Anomaly: Power spike of {delta:.0f}W at t={data_points[i]['time']:.1f}s")
    
    print()


if __name__ == "__main__":
    import argparse
    
    parser = argparse.ArgumentParser()
    parser.add_argument('--duration', type=int, default=30, help='Monitoring duration (seconds)')
    parser.add_argument('--interval', type=float, default=2.0, help='Sampling interval')
    parser.add_argument('--output', type=str, help='Output JSON file')
    
    args = parser.parse_args()
    
    data = collect_timeseries(duration=args.duration, interval=args.interval)
    
    analyze_power_profile(data)
    detect_anomalies(data)
    
    if args.output:
        with open(args.output, 'w') as f:
            json.dump(data, f, indent=2)
        print(f"Data saved to {args.output}")
```

## 第五节：rocprof 与性能剖析

### 5.1 rocprof 简介

`rocprof` 是 ROCm 提供的性能分析工具，可以对 GPU kernel 执行进行profiling：

```bash
# 安装 rocprof
sudo apt install rocprofiler

# 基本用法：对 OpenCL 程序进行 profiling
rocprof --stats ./my_opencl_application

# 对 HIP 程序进行 profiling
rocprof --hip-trace ./my_hip_application

# 输出结果
ls -la *.csv
# results.csv      - kernel 级别统计
# results.hip.csv  - HIP API trace
```

### 5.2 使用 rocprof 分析功耗

```bash
# 启用功耗监控计数器
rocprof --pmc --power-profile ./application

# 使用自定义计数器配置
cat > power_counters.txt << 'EOF'
# rocprof 计数器配置
pmc: GRBM_COUNT, GRBM_GUI_ACTIVE, TCC_HIT[0], TCC_MISS[0]
pmc: POWER_AVERAGE
pmc: TEMP_EDGE
EOF

rocprof --cntr power_counters.txt ./application
```

### 5.3 rocprof 输出分析

```python
#!/usr/bin/env python3
# analyze_rocprof.py - rocprof 结果分析

import csv
import sys
from collections import defaultdict

def parse_rocprof_csv(filename):
    """解析 rocprof 输出的 CSV 文件"""
    kernels = []
    
    with open(filename, 'r') as f:
        reader = csv.DictReader(f)
        for row in reader:
            kernels.append(row)
    
    return kernels

def analyze_kernels(kernels):
    """分析 kernel 性能数据"""
    print("=== Kernel Performance Analysis ===")
    print(f"Total kernels: {len(kernels)}")
    print()
    
    # 按 kernel 名称分组
    by_name = defaultdict(list)
    for k in kernels:
        by_name[k.get('KernelName', 'unknown')].append(k)
    
    # 统计每个 kernel 的执行情况
    for name, instances in sorted(by_name.items(), key=lambda x: len(x[1]), reverse=True)[:10]:
        durations = [float(inst.get('DurationNs', 0)) / 1000000 for inst in instances if inst.get('DurationNs')]
        
        if durations:
            avg_dur = sum(durations) / len(durations)
            total_dur = sum(durations)
            calls = len(durations)
            
            print(f"  {name}:")
            print(f"    Calls: {calls}")
            print(f"    Avg duration: {avg_dur:.3f} ms")
            print(f"    Total: {total_dur:.3f} ms")
            print()
    
    # 计算总体统计
    all_durations = [float(k.get('DurationNs', 0)) / 1000000 for k in kernels if k.get('DurationNs')]
    if all_durations:
        print(f"--- Overall ---")
        print(f"Total GPU time: {sum(all_durations):.3f} ms")
        print(f"Average kernel: {sum(all_durations)/len(all_durations):.3f} ms")
        print(f"Min kernel: {min(all_durations):.3f} ms")
        print(f"Max kernel: {max(all_durations):.3f} ms")

def correlate_power(kernel_csv, power_csv):
    """关联 kernel 执行与功耗数据"""
    kernels = parse_rocprof_csv(kernel_csv)
    power_data = parse_rocprof_csv(power_csv)
    
    print("=== Power-Kernel Correlation ===")
    
    # 假设我们有时间戳对齐的数据
    # 简单关联：计算 kernel 执行期间的功耗
    total_power_energy = 0
    total_duration = 0
    
    for k in kernels:
        duration_s = float(k.get('DurationNs', 0)) / 1e9
        power_w = float(k.get('PowerAverage', 0)) / 1e6 if k.get('PowerAverage') else 0
        
        if duration_s > 0:
            energy_j = power_w * duration_s
            total_power_energy += energy_j
            total_duration += duration_s
    
    if total_duration > 0:
        avg_power = total_power_energy / total_duration
        print(f"  Average power during kernel execution: {avg_power:.2f} W")
        print(f"  Total GPU time: {total_duration:.3f} s")
        print(f"  Total energy: {total_power_energy:.3f} J")


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python analyze_rocprof.py <results.csv>")
        sys.exit(1)
    
    kernels = parse_rocprof_csv(sys.argv[1])
    analyze_kernels(kernels)
```

## 第六节：功率监控综合实战

### 6.1 实时功率日志采集系统

```bash
#!/bin/bash
# gpu_power_logger.sh - GPU 功率日志采集系统

LOG_DIR="/var/log/gpu_power"
INTERVAL=2
RETENTION_DAYS=30

# 创建日志目录
sudo mkdir -p $LOG_DIR
sudo chmod 755 $LOG_DIR

# 获取当前日期
DATE=$(date +%Y%m%d)
LOG_FILE="$LOG_DIR/gpu_power_$DATE.csv"

# CSV 头
if [ ! -f "$LOG_FILE" ]; then
    echo "timestamp,gpu_id,name,temp_edge,temp_junction,power_avg,power_cap,gpu_busy,gfx_clk,mem_clk,fan_speed,vram_used_gb,vram_total_gb,pcie_speed,pcie_width" | sudo tee -a "$LOG_FILE"
fi

# 获取 GPU 数量
GPU_COUNT=$(rocm-smi --json --showallinfo 2>/dev/null | python3 -c "import json,sys; d=json.load(sys.stdin); print(len(d))" 2>/dev/null || echo 1)

echo "Starting GPU power logging to $LOG_FILE"
echo "Interval: ${INTERVAL}s, GPUs: $GPU_COUNT"
echo "Press Ctrl+C to stop"

# 清理旧日志
find $LOG_DIR -name "gpu_power_*.csv" -mtime +$RETENTION_DAYS -delete 2>/dev/null

cleanup() {
    echo ""
    echo "Logging stopped."
    echo "Log file: $LOG_FILE"
    echo "Total lines: $(wc -l < $LOG_FILE)"
    
    # 生成摘要
    echo ""
    echo "=== Daily Summary ==="
    awk -F',' 'NR>1 {
        power[$2]+=$6; count[$2]++; 
        if($6>max[$2]) max[$2]=$6;
        if(min[$2]==""||$6<min[$2]) min[$2]=$6;
    } END {
        for(gpu in power) {
            avg=power[gpu]/count[gpu];
            printf "GPU %s: avg=%.1fW max=%.1fW min=%sW samples=%d\n", gpu, avg, max[gpu], min[gpu], count[gpu];
        }
    }' "$LOG_FILE"
    
    exit 0
}

trap cleanup SIGINT SIGTERM

# 主循环
while true; do
    TIMESTAMP=$(date +%Y-%m-%dT%H:%M:%S)
    
    for gpu_id in $(seq 0 $((GPU_COUNT - 1))); do
        # 使用 rocm-smi JSON 输出获取单卡信息
        JSON_DATA=$(rocm-smi --json --showallinfo --showtemp --showpower --showclock --showfan --showmeminfo -i $gpu_id 2>/dev/null)
        
        if [ -n "$JSON_DATA" ]; then
            python3 -c "
import json, sys
data = json.load(sys.stdin)
if isinstance(data, dict) and str($gpu_id) in data:
    d = data[str($gpu_id)]
    name = d.get('name', 'unknown')
    temp_e = d.get('temperature', {}).get('edge_slow', 0)
    temp_j = d.get('temperature', {}).get('junction_slow', 0)
    power = d.get('power', {}).get('average_slow', 0)
    cap = d.get('power', {}).get('cap_slow', 0)
    busy = d.get('gpu_busy_percent', 0)
    gfx = d.get('clocks', {}).get('gfx_mhz', 0)
    mem = d.get('clocks', {}).get('mem_mhz', 0)
    fan = d.get('fan_speed', '0').replace('%','')
    vram_u = d.get('vram_usage', {}).get('used_gb', 0)
    vram_t = d.get('vram_usage', {}).get('total_gb', 0)
    pcie_s = d.get('pcie_info', {}).get('speed', '')
    pcie_w = d.get('pcie_info', {}).get('lanes', '')
    print(f'$TIMESTAMP,$gpu_id,{name},{temp_e},{temp_j},{power},{cap},{busy},{gfx},{mem},{fan},{vram_u},{vram_t},{pcie_s},{pcie_w}')
" <<< "$JSON_DATA" | sudo tee -a "$LOG_FILE" > /dev/null
        fi
    done
    
    sleep $INTERVAL
done
```

### 6.2 功耗数据分析仪表盘

```python
#!/usr/bin/env python3
# power_dashboard.py - GPU 功耗分析仪表盘

import json
import subprocess
import time
from datetime import datetime
import os

class GPUPowerDashboard:
    """GPU 功耗仪表盘"""
    
    def __init__(self, update_interval=1):
        self.interval = update_interval
        self.gpu_data = {}
        self.history = []
    
    def get_rocm_smi_data(self):
        """获取 rocm-smi 数据"""
        try:
            result = subprocess.run(
                ['rocm-smi', '--json', '--showallinfo', '--showtemp', '--showpower', '--showclock', '--showfan'],
                capture_output=True, text=True, timeout=5
            )
            if result.returncode == 0:
                return json.loads(result.stdout)
        except Exception as e:
            print(f"Error getting ROCm-SMI data: {e}")
        return {}
    
    def get_amdgpu_top_data(self):
        """获取 amdgpu_top 数据"""
        try:
            result = subprocess.run(
                ['amdgpu_top', '-J'],
                capture_output=True, text=True, timeout=5
            )
            if result.returncode == 0:
                return json.loads(result.stdout)
        except Exception as e:
            print(f"Error getting amdgpu_top data: {e}")
        return {}
    
    def update(self):
        """更新所有 GPU 数据"""
        rocm_data = self.get_rocm_smi_data()
        top_data = self.get_amdgpu_top_data()
        
        now = datetime.now()
        timestamp = now.timestamp()
        
        # 合并数据
        merged = {}
        
        # 从 rocm-smi 获取数据
        for gpu_id_str, gpu_info in rocm_data.items():
            gpu_id = int(gpu_id_str)
            merged[gpu_id] = {
                'timestamp': timestamp,
                'datetime': now.isoformat(),
                'name': gpu_info.get('name', 'Unknown'),
                'temperature': {
                    'edge': gpu_info.get('temperature', {}).get('edge_slow', 0),
                    'junction': gpu_info.get('temperature', {}).get('junction_slow', 0),
                },
                'power': {
                    'average': gpu_info.get('power', {}).get('average_slow', 0),
                    'cap': gpu_info.get('power', {}).get('cap_slow', 0),
                },
                'clocks': {
                    'gfx': gpu_info.get('clocks', {}).get('gfx_mhz', 0),
                    'mem': gpu_info.get('clocks', {}).get('mem_mhz', 0),
                },
                'fan': gpu_info.get('fan_speed', '0'),
                'usage': gpu_info.get('gpu_busy_percent', 0),
            }
        
        # 从 amdgpu_top 补充数据
        if 'gpus' in top_data:
            for gpu in top_data['gpus']:
                gpu_id = gpu['index']
                if gpu_id not in merged:
                    merged[gpu_id] = {'timestamp': timestamp, 'datetime': now.isoformat()}
                
                # 补充进程信息
                if 'processes' in gpu:
                    merged[gpu_id]['processes'] = gpu['processes']
        
        self.gpu_data = merged
        self.history.append(merged)
        
        # 限制历史数据量
        if len(self.history) > 3600:
            self.history = self.history[-3600:]
    
    def get_power_stats(self, gpu_id=0, window_seconds=60):
        """获取指定 GPU 的功耗统计"""
        relevant = [h.get(gpu_id, {}) for h in self.history 
                   if gpu_id in h and h[gpu_id].get('timestamp', 0) >= time.time() - window_seconds]
        
        if not relevant:
            return None
        
        powers = [r.get('power', {}).get('average', 0) for r in relevant if 'power' in r]
        
        if not powers:
            return None
        
        return {
            'current': powers[-1],
            'avg': sum(powers) / len(powers),
            'min': min(powers),
            'max': max(powers),
            'samples': len(powers),
        }
    
    def print_status(self):
        """打印状态"""
        os.system('clear' if os.name == 'posix' else 'cls')
        
        print("╔══════════════════════════════════════════════════════════════╗")
        print(f"║         GPU Power Dashboard - {datetime.now().strftime('%H:%M:%S')}           ║")
        print("╚══════════════════════════════════════════════════════════════╝")
        print()
        
        for gpu_id, data in sorted(self.gpu_data.items()):
            print(f"GPU {gpu_id}: {data.get('name', 'Unknown')}")
            
            temp = data.get('temperature', {})
            power = data.get('power', {})
            clocks = data.get('clocks', {})
            
            stats = self.get_power_stats(gpu_id, 120)
            
            print(f"  Temperature: {temp.get('edge', 0):.1f}°C (edge) / {temp.get('junction', 0):.1f}°C (junction)")
            print(f"  Power: {power.get('average', 0):.1f}W / {power.get('cap', 0):.1f}W", end='')
            if stats:
                print(f"  [2min: avg={stats['avg']:.1f}W min={stats['min']:.1f}W max={stats['max']:.1f}W]", end='')
            print()
            print(f"  Clocks: GFX={clocks.get('gfx', 0)}MHz MEM={clocks.get('mem', 0)}MHz")
            print(f"  Fan: {data.get('fan', 0)}%  Usage: {data.get('usage', 0)}%")
            
            processes = data.get('processes', [])
            if processes:
                print(f"  Processes ({len(processes)}):")
                for p in processes[:5]:
                    print(f"    PID {p.get('pid', 'N/A')}: {p.get('name', 'N/A')} - {p.get('gpu_usage', 0)}% GPU")
            print()
    
    def run(self, duration=0):
        """运行仪表盘"""
        start_time = time.time()
        
        while True:
            self.update()
            self.print_status()
            
            if duration > 0 and time.time() - start_time > duration:
                break
            
            time.sleep(self.interval)
    
    def export_history(self, filename='gpu_power_history.json'):
        """导出历史数据"""
        export = {
            'start_time': self.history[0][list(self.history[0].keys())[0]]['datetime'] if self.history else None,
            'end_time': self.history[-1][list(self.history[-1].keys())[0]]['datetime'] if self.history else None,
            'interval': self.interval,
            'data_points': len(self.history),
            'gpus': list(self.history[0].keys()) if self.history else [],
            'data': self.history
        }
        
        with open(filename, 'w') as f:
            json.dump(export, f, indent=2)
        
        print(f"History exported to {filename}")


if __name__ == "__main__":
    import argparse
    
    parser = argparse.ArgumentParser()
    parser.add_argument('--interval', type=float, default=1.0)
    parser.add_argument('--duration', type=int, default=0)
    parser.add_argument('--export', type=str, help='Export history to file')
    
    args = parser.parse_args()
    
    dashboard = GPUPowerDashboard(update_interval=args.interval)
    
    try:
        dashboard.run(duration=args.duration)
    except KeyboardInterrupt:
        print("\nStopping...")
    
    if args.export:
        dashboard.export_history(args.export)
```

### 6.3 功耗与性能的关联分析

```python
#!/usr/bin/env python3
# power_perf_analysis.py - 功耗性能关联分析

import subprocess
import json
import time
import numpy as np
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class PowerSample:
    timestamp: float
    power_w: float
    temp_c: float
    gfx_clk_mhz: int
    mem_clk_mhz: int
    gpu_busy: float

class PowerPerformanceAnalyzer:
    """功耗性能分析器"""
    
    def __init__(self):
        self.baseline = {}
        self.samples: List[PowerSample] = []
    
    def sample(self):
        """采集一个样本点"""
        try:
            result = subprocess.run(
                ['rocm-smi', '--json', '--showallinfo', '--showtemp', '--showpower', '--showclock'],
                capture_output=True, text=True, timeout=3
            )
            if result.returncode != 0:
                return
            
            data = json.loads(result.stdout)
            for gpu_id, info in data.items():
                sample = PowerSample(
                    timestamp=time.time(),
                    power_w=info.get('power', {}).get('average_slow', 0),
                    temp_c=info.get('temperature', {}).get('edge_slow', 0),
                    gfx_clk_mhz=info.get('clocks', {}).get('gfx_mhz', 0),
                    mem_clk_mhz=info.get('clocks', {}).get('mem_mhz', 0),
                    gpu_busy=info.get('gpu_busy_percent', 0),
                )
                self.samples.append(sample)
        except Exception as e:
            print(f"Sampling error: {e}")
    
    def collect_profile(self, duration=30, callback=None):
        """采集性能 profile"""
        print(f"Collecting profile for {duration}s...")
        
        self.samples = []
        start = time.time()
        
        while time.time() - start < duration:
            self.sample()
            if callback:
                callback(self)
            time.sleep(0.5)
        
        print(f"Collected {len(self.samples)} samples")
    
    def analyze(self):
        """分析采集的数据"""
        if len(self.samples) < 2:
            print("Not enough samples")
            return
        
        print("=== Power-Performance Analysis ===")
        
        # 基本统计
        powers = [s.power_w for s in self.samples]
        temps = [s.temp_c for s in self.samples]
        busy = [s.gpu_busy for s in self.samples]
        
        print(f"Power:")
        print(f"  Avg: {np.mean(powers):.1f}W")
        print(f"  Std: {np.std(powers):.1f}W")
        print(f"  Min: {np.min(powers):.1f}W")
        print(f"  Max: {np.max(powers):.1f}W")
        
        print(f"\nTemperature:")
        print(f"  Avg: {np.mean(temps):.1f}°C")
        print(f"  Min: {np.min(temps):.1f}°C")
        print(f"  Max: {np.max(temps):.1f}°C")
        
        print(f"\nGPU Busy:")
        print(f"  Avg: {np.mean(busy):.1f}%")
        print(f"  Min: {np.min(busy):.1f}%")
        print(f"  Max: {np.max(busy):.1f}%")
        
        # 能效分析
        if np.mean(powers) > 0:
            efficiency = np.mean(busy) / np.mean(powers)
            print(f"\nPower Efficiency: {efficiency:.2f} %/W")
        
        # 功耗-使用率相关性
        if len(powers) > 2 and len(busy) > 2:
            corr = np.corrcoef(powers, busy)[0, 1]
            print(f"\nPower-Usage Correlation: {corr:.3f}")
            
            if abs(corr) > 0.8:
                print("  Strong correlation: power closely follows GPU load")
            elif abs(corr) > 0.5:
                print("  Moderate correlation: power somewhat follows load")
            else:
                print("  Weak correlation: other factors affect power")
        
        # 温度分析
        max_temp = np.max(temps)
        if max_temp > 85:
            print(f"\n⚠ High temperature detected: {max_temp:.1f}°C")
            print("  GPU may be throttling")
        
        # 功耗分布
        print(f"\nPower Distribution:")
        for pct in [10, 25, 50, 75, 90]:
            val = np.percentile(powers, pct)
            print(f"  P{pct}: {val:.1f}W")
    
    def report_json(self):
        """输出 JSON 报告"""
        if not self.samples:
            return json.dumps({"error": "no data"})
        
        powers = [s.power_w for s in self.samples]
        temps = [s.temp_c for s in self.samples]
        busy = [s.gpu_busy for s in self.samples]
        
        return json.dumps({
            "samples": len(self.samples),
            "duration_s": self.samples[-1].timestamp - self.samples[0].timestamp,
            "power": {
                "avg_w": round(np.mean(powers), 1),
                "std_w": round(np.std(powers), 1),
                "min_w": round(np.min(powers), 1),
                "max_w": round(np.max(powers), 1),
                "p50_w": round(np.percentile(powers, 50), 1),
                "p95_w": round(np.percentile(powers, 95), 1),
            },
            "temperature": {
                "avg_c": round(np.mean(temps), 1),
                "max_c": round(np.max(temps), 1),
            },
            "gpu_busy": {
                "avg_pct": round(np.mean(busy), 1),
            },
            "efficiency_pct_per_w": round(np.mean(busy) / np.mean(powers), 2) if np.mean(powers) > 0 else 0,
        }, indent=2)


def run_benchmark_and_analyze(benchmark_cmd, duration=60):
    """运行基准测试并分析功耗"""
    import subprocess
    import threading
    
    analyzer = PowerPerformanceAnalyzer()
    
    def monitor():
        analyzer.collect_profile(duration=duration)
    
    def run_benchmark():
        try:
            subprocess.run(benchmark_cmd, shell=True, timeout=duration + 10)
        except subprocess.TimeoutExpired:
            pass
    
    # 启动监控线程
    monitor_thread = threading.Thread(target=monitor)
    monitor_thread.start()
    
    # 启动基准测试线程
    bench_thread = threading.Thread(target=run_benchmark)
    bench_thread.start()
    
    # 等待完成
    bench_thread.join()
    monitor_thread.join()
    
    # 分析结果
    analyzer.analyze()
    
    return analyzer


if __name__ == "__main__":
    import argparse
    
    parser = argparse.ArgumentParser()
    parser.add_argument('--benchmark', type=str, help='Benchmark command to run')
    parser.add_argument('--duration', type=int, default=30, help='Monitoring duration')
    parser.add_argument('--json', action='store_true', help='JSON output')
    
    args = parser.parse_args()
    
    if args.benchmark:
        analyzer = run_benchmark_and_analyze(args.benchmark, args.duration)
        if args.json:
            print(analyzer.report_json())
    else:
        # 手动采集模式
        analyzer = PowerPerformanceAnalyzer()
        print("Collecting data for analysis (Ctrl+C to stop)...")
        try:
            while True:
                analyzer.sample()
                time.sleep(0.5)
        except KeyboardInterrupt:
            pass
        
        analyzer.analyze()
        if args.json:
            print(analyzer.report_json())
```

## 第七节：amdgpu_top 源码架构分析

### 7.1 amdgpu_top 数据采集流程

amdgpu_top 使用 Rust 编写，主要通过以下方式获取 GPU 信息：

```
amdgpu_top 启动
  │
  ├── 初始化 drm/kfd 设备
  │     ├── /dev/dri/card0       (DRM 接口)
  │     └── /dev/kfd             (KFD 接口)
  │
  ├── 读取 sysfs 属性
  │     ├── /sys/class/drm/card0/device/gpu_busy_percent
  │     ├── /sys/class/drm/card0/device/mem_busy_percent
  │     ├── /sys/class/drm/card0/device/power1_average
  │     ├── /sys/class/drm/card0/device/pp_dpm_sclk
  │     ├── /sys/class/drm/card0/device/hwmon/hwmon*/temp*_input
  │     └── ...
  │
  ├── 读取进程 GPU 使用
  │     ├── /sys/class/drm/card0/device/process/
  │     └── /proc/<pid>/fd/ 检查 DRM 文件描述符
  │
  └── 渲染 TUI 界面
        └── 使用 ratatui 框架
```

### 7.2 关键数据结构

```rust
// amdgpu_top 核心数据结构（Rust）

/// GPU 设备信息
struct GpuInfo {
    index: u32,
    name: String,
    driver: String,
    temperature: TemperatureInfo,
    power: PowerInfo,
    clocks: ClockInfo,
    usage: UsageInfo,
    fan: FanInfo,
    memory: MemoryInfo,
    processes: Vec<ProcessInfo>,
}

/// 温度信息
struct TemperatureInfo {
    edge: Option<f32>,
    junction: Option<f32>,
    memory: Option<f32>,
}

/// 功耗信息
struct PowerInfo {
    power_ave: Option<f32>,    // 平均功耗 (W)
    power_cap: Option<f32>,    // 功耗上限 (W)
    voltage_gfx: Option<f32>,  // GFX电压 (V)
    voltage_mem: Option<f32>,  // 显存电压 (V)
}

/// 时钟频率信息
struct ClockInfo {
    gfx_clock: Option<f32>,    // GFX 时钟 (MHz)
    mem_clock: Option<f32>,    // 显存时钟 (MHz)
    soc_clock: Option<f32>,    // SoC 时钟 (MHz)
}

/// GPU 使用率信息
struct UsageInfo {
    gpu_busy: Option<f32>,     // GPU 使用率 (%)
    memory_used: Option<u64>,  // 显存使用量 (bytes)
    memory_total: Option<u64>, // 显存总量 (bytes)
}

/// 风扇信息
struct FanInfo {
    speed: Option<u32>,        // 风扇转速 (RPM)
    pwm: Option<u32>,          // PWM 占空比 (%)
}

/// 显存信息
struct MemoryInfo {
    vram_total: u64,           // VRAM 总量
    vram_used: u64,            // VRAM 使用量
    vram_type: String,         // 显存类型 (GDDR6/HBM2e)
    gtt_total: u64,            // GTT 总量
    gtt_used: u64,             // GTT 使用量
    visible_vram_total: u64,   // 可见 VRAM 总量
    visible_vram_used: u64,    // 可见 VRAM 使用量
}

/// 进程 GPU 使用信息
struct ProcessInfo {
    pid: u32,                  // 进程 PID
    name: String,              // 进程名称
    vram_usage: u64,           // VRAM 使用量
    gpu_usage: Option<f32>,    // GPU 使用率
    engine_usage: Vec<EngineUsage>, // 各引擎使用情况
}

/// 引擎使用信息
struct EngineUsage {
    engine_type: String,       // GFX/COMPUTE/SDMA/VCN
    usage: f32,                // 使用率
}
```

### 7.3 数据采集流程源码分析

amdgpu_top 的数据采集核心实现在 `src/backend/` 目录下，分为多个采集器：

```text
数据采集流程：

  定时器触发 (默认 1000ms)
        │
        ├── collect_gpu_info()
        │     ├── 读取 /sys/class/drm/card*/device/
        │     ├── 解析 gpu_busy_percent  →  GPU 使用率
        │     ├── 解析 power1_average    →  功耗
        │     ├── 解析 temp1_input       →  温度
        │     ├── 解析 fan1_input        →  风扇转速
        │     └── 解析 pp_dpm_sclk       →  时钟频率
        │
        ├── collect_memory_info()
        │     ├── 读取 /sys/class/drm/card*/device/mem_info_vram_total
        │     ├── 读取 mem_info_vram_used
        │     ├── 读取 mem_info_gtt_total
        │     └── 读取 mem_info_visible_vram_total
        │
        ├── collect_process_info()
        │     ├── 打开 /dev/kfd (KFD 接口)
        │     ├── ioctl(AMDKFD_IOC_GET_PROCESS_APERTURES)
        │     ├── 读取每个进程的队列信息
        │     └── 计算每个进程的 GPU 使用率
        │
        └── collect_perf_counters()
              ├── 读取 DRM fd 信息
              └── 解析 perf 计数器数据
```

### 7.4 关键源码片段

```rust
// amdgpu_top 数据采集核心函数简化示例

use std::fs;
use std::path::Path;

/// 读取 sysfs 文件并解析为字符串
fn read_sysfs(path: &str) -> Result<String, std::io::Error> {
    fs::read_to_string(path).map(|s| s.trim().to_string())
}

/// 采集 GPU 基本信息
fn collect_gpu_info(device_path: &Path) -> GpuInfo {
    let mut info = GpuInfo {
        index: 0,
        name: String::new(),
        driver: String::from("amdgpu"),
        temperature: TemperatureInfo { edge: None, junction: None, memory: None },
        power: PowerInfo { power_ave: None, power_cap: None, voltage_gfx: None, voltage_mem: None },
        clocks: ClockInfo { gfx_clock: None, mem_clock: None, soc_clock: None },
        usage: UsageInfo { gpu_busy: None, memory_used: None, memory_total: None },
        fan: FanInfo { speed: None, pwm: None },
        memory: MemoryInfo {
            vram_total: 0, vram_used: 0, vram_type: String::new(),
            gtt_total: 0, gtt_used: 0,
            visible_vram_total: 0, visible_vram_used: 0,
        },
        processes: Vec::new(),
    };

    // 读取名称
    if let Ok(name) = read_sysfs(&device_path.join("product_name").to_string_lossy()) {
        info.name = name;
    }

    // 读取 GPU 使用率
    let gpu_busy_path = device_path.join("gpu_busy_percent").to_string_lossy().to_string();
    if let Ok(val) = read_sysfs(&gpu_busy_path) {
        info.usage.gpu_busy = val.parse::<f32>().ok();
    }

    // 读取功耗
    let hwmon_path = find_hwmon_path(device_path);
    if let Some(hwmon) = hwmon_path {
        let power_path = hwmon.join("power1_average").to_string_lossy().to_string();
        if let Ok(val) = read_sysfs(&power_path) {
            info.power.power_ave = val.parse::<f32>().ok().map(|v| v / 1_000_000.0);
        }
    }

    info
}

/// 查找 hwmon 设备路径
fn find_hwmon_path(device_path: &Path) -> Option<std::path::PathBuf> {
    let hwmon_dir = device_path.join("hwmon");
    if hwmon_dir.exists() {
        if let Ok(entries) = fs::read_dir(&hwmon_dir) {
            for entry in entries.flatten() {
                if entry.file_name().to_string_lossy().starts_with("hwmon") {
                    return Some(entry.path());
                }
            }
        }
    }
    None
}
```

## 8. ROCm-SMI vs amdgpu_top 工具对比

| 特性 | ROCm-SMI (rocm-smi) | amdgpu_top |
|------|---------------------|------------|
| 开发语言 | Python/C++ | Rust |
| 界面类型 | CLI / 库 | TUI (终端界面) |
| 安装方式 | ROCm 套件 | cargo / 二进制 |
| 数据来源 | sysfs + KFD | sysfs + KFD + DRM |
| 实时监控 | ❌ (轮询采样) | ✅ (实时刷新) |
| 进程级监控 | ❌ (有限) | ✅ (每个进程 GPU 使用) |
| JSON 输出 | ✅ `--json` | ✅ `-J` |
| 功耗控制 | ✅ `--setpowercap` | ❌ (只读) |
| 超频/调压 | ✅ `--setsclk` `--setvc` | ❌ |
| 风扇控制 | ✅ `--setfan` | ❌ |
| GPU 选择 | ✅ `--gpu` | ✅ `-d` |
| 日志记录 | ✅ `--loglevel` | ❌ |
| CSV 导出 | ❌ | ✅ `-C` |
| 跨平台 | Linux only | Linux + BSD |
| 资源占用 | 中 (~50MB) | 低 (~15MB) |
| 社区活跃度 | AMD 官方维护 | 社区维护 |

### 8.1 选择建议

```text
场景                          推荐工具
──────────────────────────────────────────────────
日常 GPU 状态快速查看          amdgpu_top
深度功耗/频率调节              ROCm-SMI
脚本化监控/告警                ROCm-SMI (JSON)
进程级 GPU 使用分析            amdgpu_top
超频/降压调试                  ROCm-SMI
性能基准测试                   ROCm-SMI + amdgpu_top 结合
自动化测试集成                 ROCm-SMI (Python API)
实时 TUI 仪表盘                amdgpu_top
```

### 8.2 联合使用示例

```python
#!/usr/bin/env python3
"""
rocm-smi + amdgpu_top 联合监控脚本
使用 rocm-smi 进行功耗控制，amdgpu_top 进行实时监控
"""

import subprocess
import json
import time
import threading
import sys


class HybridMonitor:
    """混合监控器：结合 rocm-smi 控制能力与 amdgpu_top 监控能力"""

    def __init__(self, gpu_id: str = "0"):
        self.gpu_id = gpu_id
        self.running = False
        self.data = {}
        self.lock = threading.Lock()

    def get_rocm_smi_data(self) -> dict:
        """通过 rocm-smi 获取详细硬件数据"""
        try:
            result = subprocess.run(
                ["rocm-smi", "--json", "--showpower", "--showtemp",
                 "--showmeminfo", "vram", "--showuse"],
                capture_output=True, text=True, timeout=5
            )
            return json.loads(result.stdout)
        except Exception as e:
            return {"error": str(e)}

    def set_power_cap(self, watts: float) -> bool:
        """通过 rocm-smi 设置功耗上限"""
        try:
            result = subprocess.run(
                ["rocm-smi", "--setpowercap", str(watts)],
                capture_output=True, text=True, timeout=5
            )
            return result.returncode == 0
        except Exception:
            return False

    def set_performance_level(self, level: str) -> bool:
        """设置性能级别: auto/low/high/manual"""
        try:
            result = subprocess.run(
                ["rocm-smi", "--setperflevel", level],
                capture_output=True, text=True, timeout=5
            )
            return result.returncode == 0
        except Exception:
            return False

    def monitor_loop(self, interval: float = 2.0):
        """监控主循环"""
        self.running = True
        while self.running:
            data = self.get_rocm_smi_data()
            with self.lock:
                self.data = data
            time.sleep(interval)

    def start(self, interval: float = 2.0):
        """启动后台监控线程"""
        thread = threading.Thread(target=self.monitor_loop, args=(interval,))
        thread.daemon = True
        thread.start()

    def stop(self):
        self.running = False

    def print_summary(self):
        """打印监控摘要"""
        with self.lock:
            data = self.data

        gpu_key = f"card{self.gpu_id}"
        if gpu_key not in data:
            print("暂无数据")
            return

        gpu = data[gpu_key]
        print(f"{'='*60}")
        print(f"  GPU {self.gpu_id} 监控摘要")
        print(f"{'='*60}")
        print(f"  功耗:       {gpu.get('Power', 'N/A')}")
        print(f"  温度:       {gpu.get('Temperature', 'N/A')}")
        print(f"  显存:       {gpu.get('VRAM', 'N/A')}")
        print(f"  使用率:     {gpu.get('GPU Use', 'N/A')}")
        print(f"{'='*60}")


def main():
    monitor = HybridMonitor("0")
    monitor.start()

    try:
        # 演示: 逐步降低功耗上限
        for power_limit in [200, 180, 160, 140, 120]:
            print(f"\n设置功耗上限: {power_limit}W")
            monitor.set_power_cap(power_limit)
            time.sleep(3)
            monitor.print_summary()
    except KeyboardInterrupt:
        print("\n恢复自动模式...")
        monitor.set_performance_level("auto")
    finally:
        monitor.stop()


if __name__ == "__main__":
    main()
```

## 9. 综合实战：GPU 电源性能剖析报告

### 9.1 自动化剖析脚本

```bash
#!/bin/bash
# gpu_power_profiling.sh - GPU 电源性能自动剖析脚本
# 用途：综合使用 rocm-smi 和 amdgpu_top 生成电源性能报告

OUTPUT_DIR="gpu_profile_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$OUTPUT_DIR"

echo "=========================================="
echo "  GPU 电源性能剖析报告生成工具"
echo "=========================================="
echo "输出目录: $OUTPUT_DIR"
echo ""

# 1. 系统信息
echo "[1/6] 采集系统信息..."
{
    echo "=== 系统信息 ==="
    uname -a
    echo ""
    echo "=== ROCm 版本 ==="
    rocm-smi --version 2>/dev/null || echo "rocm-smi not found"
    echo ""
    echo "=== amdgpu_top 版本 ==="
    amdgpu_top --version 2>/dev/null || echo "amdgpu_top not found"
    echo ""
    echo "=== GPU 列表 ==="
    rocm-smi --showhw 2>/dev/null
} > "$OUTPUT_DIR/system_info.txt"

# 2. 基础状态快照
echo "[2/6] 采集基础状态..."
rocm-smi --json --showallinfo > "$OUTPUT_DIR/rocm_smi_all.json" 2>/dev/null

# 3. 功耗基准测试
echo "[3/6] 运行功耗基准测试..."
for workload in "idle" "light" "medium" "heavy"; do
    echo "  测试: $workload"
    case $workload in
        idle)
            # 空闲状态，等待 5 秒
            sleep 5
            ;;
        light)
            # 轻负载: 简单 glxgears
            timeout 10 glxgears 2>/dev/null &
            GLX_PID=$!
            sleep 3
            ;;
        medium)
            # 中负载: clinfo + 简单计算
            timeout 10 python3 -c "
import numpy as np
for _ in range(5):
    a = np.random.rand(4096, 4096)
    b = np.random.rand(4096, 4096)
    c = a @ b
" 2>/dev/null &
            PY_PID=$!
            sleep 3
            ;;
        heavy)
            # 重负载: 矩阵乘法 + 内存拷贝
            timeout 15 stress_gpu 2>/dev/null || timeout 15 rocminfo 2>/dev/null &
            STRESS_PID=$!
            sleep 5
            ;;
    esac

    # 采集数据
    rocm-smi --json --showpower --showtemp --showuse \
        > "$OUTPUT_DIR/power_${workload}.json" 2>/dev/null
    amdgpu_top -J -d 1 -n 1 \
        > "$OUTPUT_DIR/top_${workload}.json" 2>/dev/null

    # 清理
    kill $GLX_PID $PY_PID $STRESS_PID 2>/dev/null
    wait 2>/dev/null
done

# 4. 生成报告
echo "[4/6] 生成 HTML 报告..."
python3 << 'EOF'
import json
import os
from datetime import datetime

def load_json(path):
    try:
        with open(path) as f:
            return json.load(f)
    except:
        return {}

output_dir = "{{OUTPUT_DIR}}"
report = []
for workload in ["idle", "light", "medium", "heavy"]:
    data = load_json(os.path.join(output_dir, f"power_{workload}.json"))
    report.append(f"<tr>")
    report.append(f"  <td>{workload}</td>")
    # 解析功率
    power = "N/A"
    for key in data:
        if isinstance(data[key], dict) and "Power" in data[key]:
            power = data[key]["Power"]
            break
    report.append(f"  <td>{power}</td>")
    # 解析温度
    temp = "N/A"
    for key in data:
        if isinstance(data[key], dict) and "Temperature" in data[key]:
            temp = data[key]["Temperature"]
            break
    report.append(f"  <td>{temp}</td>")
    report.append(f"</tr>")

html = f"""<!DOCTYPE html>
<html>
<head>
    <title>GPU 电源性能剖析报告</title>
    <style>
        body {{ font-family: Arial; margin: 40px; }}
        h1 {{ color: #333; }}
        table {{ border-collapse: collapse; width: 100%; }}
        th, td {{ border: 1px solid #ddd; padding: 8px; text-align: left; }}
        th {{ background-color: #4CAF50; color: white; }}
    </style>
</head>
<body>
    <h1>GPU 电源性能剖析报告</h1>
    <p>生成时间: {datetime.now().isoformat()}</p>
    <h2>不同负载下功耗对比</h2>
    <table>
        <tr><th>负载类型</th><th>功耗</th><th>温度</th></tr>
        {''.join(report)}
    </table>
</body>
</html>"""

with open(os.path.join(output_dir, "report.html"), "w") as f:
    f.write(html)
print(f"报告已生成: {output_dir}/report.html")
EOF

# 5. 清理
echo "[5/6] 清理临时进程..."
kill %1 %2 %3 2>/dev/null
wait 2>/dev/null

# 6. 汇总
echo "[6/6] 完成!"
echo ""
echo "=========================================="
echo "  剖析完成！输出目录: $OUTPUT_DIR"
echo "  报告文件: $OUTPUT_DIR/report.html"
echo "=========================================="
```

### 9.2 剖析结果分析

```python
#!/usr/bin/env python3
"""
GPU 电源剖析结果分析器
解析 rocm-smi JSON 输出并生成分析报告
"""

import json
import sys
from pathlib import Path
from typing import Dict, List, Optional


class PowerProfileAnalyzer:
    """功耗剖析数据分析器"""

    def __init__(self, data_dir: str):
        self.data_dir = Path(data_dir)
        self.workloads = ["idle", "light", "medium", "heavy"]
        self.data: Dict[str, dict] = {}

    def load_data(self) -> bool:
        """加载所有工作负载数据"""
        for wl in self.workloads:
            path = self.data_dir / f"power_{wl}.json"
            if path.exists():
                with open(path) as f:
                    self.data[wl] = json.load(f)
            else:
                print(f"警告: {path} 不存在")
                self.data[wl] = {}
        return len(self.data) > 0

    def get_gpu_metric(self, workload: str, metric: str) -> Optional[float]:
        """获取指定工作负载下的 GPU 指标"""
        data = self.data.get(workload, {})
        for gpu_id, gpu_data in data.items():
            if isinstance(gpu_data, dict) and metric in gpu_data:
                val = gpu_data[metric]
                try:
                    return float(val.replace("W", "").replace("°C", "").replace("%", "").strip())
                except (ValueError, AttributeError):
                    pass
        return None

    def get_power(self, workload: str) -> Optional[float]:
        """获取功耗 (W)"""
        return self.get_gpu_metric(workload, "Power")

    def get_temp(self, workload: str) -> Optional[float]:
        """获取温度 (°C)"""
        return self.get_gpu_metric(workload, "Temperature")

    def get_usage(self, workload: str) -> Optional[float]:
        """获取使用率 (%)"""
        return self.get_gpu_metric(workload, "GPU Use")

    def generate_report(self) -> str:
        """生成分析报告"""
        lines = []
        lines.append("=" * 70)
        lines.append("  GPU 电源性能剖析分析报告")
        lines.append("=" * 70)
        lines.append(f"{'负载':<12} {'功耗(W)':<12} {'温度(°C)':<12} {'使用率(%)':<12} {'能效(GHz/W)':<12}")
        lines.append("-" * 70)

        for wl in self.workloads:
            power = self.get_power(wl)
            temp = self.get_temp(wl)
            usage = self.get_usage(wl)

            power_str = f"{power:.1f}" if power is not None else "N/A"
            temp_str = f"{temp:.1f}" if temp is not None else "N/A"
            usage_str = f"{usage:.1f}" if usage is not None else "N/A"

            # 估算能效: 使用率 / 功耗 (越高越好)
            efficiency_str = "N/A"
            if power is not None and usage is not None and power > 0:
                efficiency = usage / power
                efficiency_str = f"{efficiency:.2f}"

            lines.append(f"  {wl:<10} {power_str:<12} {temp_str:<12} {usage_str:<12} {efficiency_str:<12}")

        lines.append("=" * 70)
        return "\n".join(lines)

    def export_csv(self, path: str):
        """导出 CSV"""
        with open(path, "w") as f:
            f.write("workload,power_w,temp_c,usage_pct\n")
            for wl in self.workloads:
                power = self.get_power(wl) or 0
                temp = self.get_temp(wl) or 0
                usage = self.get_usage(wl) or 0
                f.write(f"{wl},{power},{temp},{usage}\n")
        print(f"CSV 已导出: {path}")


def main():
    if len(sys.argv) < 2:
        print("用法: python3 analyze_profile.py <data_directory>")
        sys.exit(1)

    analyzer = PowerProfileAnalyzer(sys.argv[1])
    if not analyzer.load_data():
        print("错误: 无法加载数据")
        sys.exit(1)

    print(analyzer.generate_report())
    analyzer.export_csv(Path(sys.argv[1]) / "power_profile.csv")


if __name__ == "__main__":
    main()
```

## 10. 常见问题与故障排查

### 10.1 ROCm-SMI 无法检测 GPU

```text
症状:
  $ rocm-smi
  No GPUs detected

可能原因:
  1. ROCm 内核模块未加载
  2. 权限不足 (需要 root 或 video 组)
  3. GPU 被其他驱动占用

排查步骤:
  # 1. 检查内核模块
  $ lsmod | grep amdgpu
  $ lsmod | grep radeon    # 与 amdgpu 冲突

  # 2. 检查 GPU 设备
  $ lspci -nn | grep -i "vga\|display\|3d"

  # 3. 检查权限
  $ groups $USER
  $ sudo rocm-smi          # 测试 root 权限

  # 4. 检查 ROCm 安装
  $ dpkg -l | grep rocm
  $ /opt/rocm/bin/rocminfo
```

### 10.2 amdgpu_top 无法显示进程信息

```text
症状:
  amdgpu_top 显示 GPU 信息正常，但进程列表为空

可能原因:
  1. KFD (Kernel Fusion Driver) 未加载
  2. 权限不足
  3. GPU 未被任何进程使用

排查步骤:
  # 1. 检查 KFD 设备
  $ ls -la /dev/kfd
  $ lsmod | grep kfd

  # 2. 检查权限
  $ sudo amdgpu_top        # 测试 root 权限

  # 3. 启动 GPU 负载后查看
  $ rocminfo &
  $ amdgpu_top -J
```

### 10.3 功耗读数异常

```text
症状:
  功耗读数为 0 或明显偏离预期

可能原因:
  1. hwmon 接口不支持该 GPU
  2. GPU 处于未初始化状态
  3. 驱动版本过旧

排查步骤:
  # 1. 检查 hwmon 设备
  $ find /sys/class/drm/card0/device/hwmon/ -name "power*"

  # 2. 直接读取原始值
  $ cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_average
  $ cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_cap

  # 3. 检查驱动版本
  $ cat /sys/class/drm/card0/device/uevent | grep DRIVER
  $ dmesg | grep -i amdgpu | head -5

  # 4. 验证单位转换
  # 注意: hwmon 返回的单位是微瓦 (μW)
  # 正确转换: 值 / 1_000_000 = 瓦 (W)
  $ echo "scale=3; $(cat power1_average) / 1000000" | bc
```

### 10.4 Python API 导入失败

```text
症状:
  >>> import pyrocm_smi
  ModuleNotFoundError: No module named 'pyrocm_smi'

解决方案:
  # 1. 检查 ROCm 安装
  $ ls /opt/rocm/lib/python*/site-packages/pyrocm_smi*

  # 2. 设置 Python 路径
  $ export PYTHONPATH=/opt/rocm/lib/python3.8/site-packages:$PYTHONPATH

  # 3. 手动安装
  $ pip3 install pyrocm-smi

  # 4. 使用 subprocess 替代 (备选方案)
  $ python3 -c "import subprocess; import json; \
      data = json.loads(subprocess.run(['rocm-smi','--json'], \
      capture_output=True).stdout); print(data)"
```

### 10.5 JSON 输出解析错误

```text
症状:
  json.decoder.JSONDecodeError: Invalid escape

原因:
  ROCm-SMI 某些版本 JSON 输出包含非标准字符

解决方案:
  # 1. 使用严格模式解析
  $ python3 -c "
import json, subprocess
result = subprocess.run(['rocm-smi', '--json', '--showtemp'],
    capture_output=True, text=True)
# 修复常见 JSON 问题
fixed = result.stdout.replace(\"\\n\", \" \")
data = json.loads(fixed)
print(json.dumps(data, indent=2))
"

  # 2. 升级 ROCm 版本
  $ sudo apt update && sudo apt upgrade rocm-smi

  # 3. 使用 --json 结合特定参数
  $ rocm-smi --json --showtemp --showpower | python3 -m json.tool
```

## 11. 扩展思考与进阶

### 11.1 ROCm-SMI 架构设计思考

```text
1. 为什么 ROCm-SMI 选择 Python 作为主要接口语言？
   - Python 在 AI/ML 社区广泛使用
   - 快速原型开发能力
   - 丰富的 JSON 处理库支持

2. ROCm-SMI 的性能瓶颈在哪里？
   - subprocess 调用开销 (每次命令 ~5ms)
   - JSON 序列化/反序列化开销 (~10ms 每次)
   - sysfs 读取 ~50μs 每次
   - 高频采集时 (10Hz+)，Python GIL 成为瓶颈

3. 如何设计一个高效的 GPU 监控系统？
   - 使用 C/C++ 扩展直接读取 sysfs
   - 共享内存 IPC 替代 subprocess
   - 环形缓冲区减少系统调用
   - epoll 事件驱动替代轮询
```

### 11.2 amdgpu_top 技术亮点

```text
1. Rust 语言优势
   - 零成本抽象, 无 GC 暂停
   - 内存安全, 无段错误
   - 跨平台编译支持

2. TUI 框架选择 (ratatui)
   - 基于 immediate mode GUI
   - 双向缓冲防止闪烁
   - 丰富的 widget 库

3. 数据采集优化
   - 批量读取 sysfs 减少上下文切换
   - 差异更新避免全量重绘
   - 异步 IO 不阻塞 UI 渲染
   - mmap 映射 KFD 设备减少拷贝

4. 进程追踪实现
   - 通过 KFD 枚举 GPU 队列
   - 解析 /proc/<pid>/cmdline 获取进程名
   - 按引擎类型统计 GPU 使用率
```

### 11.3 与 NVIDIA 工具对比

| 特性 | AMD (ROCm-SMI) | NVIDIA (nvidia-smi) | 差异分析 |
|------|----------------|--------------------|---------|
| 命令行工具 | rocm-smi | nvidia-smi | 功能相似 |
| Python 库 | pyrocm_smi | pynvml | API 设计不同 |
| TUI 监控 | amdgpu_top | nvtop | amdgpu_top 更轻量 |
| 性能分析 | rocprof | nvprof/nsys | 功能定位不同 |
| 进程追踪 | ✅ (KFD) | ✅ (NVML) | 实现机制不同 |
| 持久化模式 | ❌ | ✅ （nvidia-smi -pm） | NVIDIA 独占 |
| ECC 监控 | ✅ | ✅ | 功能对等 |
| GPU 拓扑 | ✅ (rocm-smi --showtopo) | ✅ (nvidia-smi topo) | 功能对等 |

### 11.4 未来发展方向

```text
1. ROCm-SMI 发展方向
   - 统一 sysfs 和 KFD 接口
   - 图形化界面 (GUI)
   - 远程监控能力
   - 历史数据持久化

2. amdgpu_top 发展方向
   - 图形模式 (GPU 加速 TUI)
   - 远程 SSH 监控
   - 告警通知集成
   - 插件系统扩展

3. 行业趋势
   - CXL 互联的多 GPU 电源管理
   - AI 驱动的动态功耗优化
   - 数据中心级功耗编排 (Kubernetes)
   - 开放固件电源管理接口
```

### 11.5 思考题

```text
1. 设计一个分布式 GPU 监控系统，要求支持 1000+ GPU 节点
   - 如何解决数据采集的性能瓶颈？
   - 如何实现告警聚合与降噪？
   - 如何保证监控系统本身的高可用？

2. 对比 rocm-smi --json 和直接读取 sysfs 的优缺点
   - 什么场景下应该选择哪种方式？
   - 如何结合两者优势？

3. 分析 amdgpu_top 的进程追踪准确性
   - 当多个进程共享 GPU 时如何分配使用率？
   - 如何区分计算和显存操作的使用率？

4. 预测未来 3 年 GPU 电源管理工具的发展趋势
   - CXL 3.0 对多 GPU 电源管理的影响
   - AI 工作负载的功耗特征如何影响工具设计？
```

## 参考资源

1. [ROCm-SMI 官方文档](https://rocm.docs.amd.com/projects/rocm_smi_lib/en/latest/)
2. [amdgpu_top GitHub 仓库](https://github.com/Umio-Yasuno/amdgpu_top)
3. [ROCm 核心库文档](https://rocm.docs.amd.com/projects/rocm-core/en/latest/)
4. [AMD ROCm 文档中心](https://rocm.docs.amd.com/)
5. [Linux kernel amdgpu 驱动文档](https://docs.kernel.org/gpu/amdgpu.html)
6. [pyrocm_smi Python 包](https://pypi.org/project/pyrocm-smi/)
7. [Linux hwmon 文档](https://docs.kernel.org/hwmon/)
