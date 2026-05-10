# Day 394：DKMS 与 out-of-tree 驱动构建

## 学习目标

- 理解 DKMS (Dynamic Kernel Module Support) 的工作原理
- 掌握为 amdgpu 配置 DKMS 构建环境的方法
- 学会 out-of-tree 模块的编译和部署
- 掌握 DKMS 驱动的签名和分发流程

## 知识详解

### 概念原理

#### DKMS 架构

DKMS 是一种框架，允许内核模块源码独立于内核源码树，在内核更新时自动重建：

```
DKMS 架构:

┌────────────────────────────────────────────────────────┐
│                    DKMS 框架                            │
├────────────────────────────────────────────────────────┤
│                                                        │
│  模块源码目录: /usr/src/<module>-<version>/             │
│  ┌──────────────────────────────────────────────────┐  │
│  │ amdgpu-dkms-6.1.0/                                │  │
│  │  ├── dkms.conf          ← DKMS 配置文件          │  │
│  │  ├── Makefile           ← 构建文件                │  │
│  │  ├── amdgpu/            ← 驱动源码                │  │
│  │  ├── include/           ← 头文件                  │  │
│  │  └── firmware/          ← 固件文件                │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  构建过程:                                              │
│  ┌──────────────────────────────────────────────────┐  │
│  │ dkms build -m amdgpu -v 6.1.0 -k 6.6.0          │  │
│  │     │                                              │  │
│  │     ▼                                              │  │
│  │ 1. 复制源码到 /var/lib/dkms/                      │  │
│  │ 2. 创建内核版本符号链接                            │  │
│  │ 3. 调用 make -C /lib/modules/.../build            │  │
│  │ 4. 复制 .ko 到 /lib/modules/.../extra/            │  │
│  │ 5. 更新模块依赖 (depmod)                          │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  生命周期管理:                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │ dkms install   → 构建并安装模块                  │  │
│  │ dkms remove    → 移除模块                        │  │
│  │ dkms status    → 查看模块状态                    │  │
│  │ dkms autoinstall → 为新内核自动安装              │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
└────────────────────────────────────────────────────────┘

DKMS 工作流程:

  内核升级事件:
  
  新内核安装
      │
      ▼
  DKMS 检测到新内核
      │
      ▼
  ┌────────────────────────────────────────┐
  │ dkms autoinstall                       │
  │                                        │
  │ 对于每个已注册的模块:                   │
  │   ┌──────────────────────────────┐     │
  │   │ 1. dkms build -k <新内核>    │     │
  │   │ 2. dkms install -k <新内核>  │     │
  │   │ 3. depmod <新内核>           │     │
  │   └──────────────────────────────┘     │
  └────────────────────────────────────────┘
      │
      ▼
  新内核启动时，模块可用
```

#### dkms.conf 配置

```bash
# DKMS 配置文件: /usr/src/amdgpu-dkms-6.1.0/dkms.conf

# 包名 (用于 DKMS 注册)
PACKAGE_NAME="amdgpu"

# 包版本 (与源码版本一致)
PACKAGE_VERSION="6.1.0"

# 编译后要安装的内核模块列表
BUILT_MODULE_NAME[0]="amdgpu"

# 模块在源码树中的相对路径
BUILT_MODULE_LOCATION[0]="amdgpu"

# 是否自动安装到内核模块目录
DEST_MODULE_LOCATION[0]="/kernel/drivers/gpu/drm/amd/amdgpu"

# 构建后自动执行 depmod
AUTOINSTALL="yes"

# 保留构建目录用于调试
REMAKE_INITRD="yes"

# 是否需要 Makefile 在源码根目录
STRIP="yes"

# 模块签名配置
if [ -n "$(ls /usr/src/kernels/${kernelver}/certs/signing_key.pem 2>/dev/null)" ]; then
    SIGN_TOOL="scripts/sign-file"
    SIGN_KEY="/usr/src/kernels/${kernelver}/certs/signing_key.pem"
    SIGN_CERT="/usr/src/kernels/${kernelver}/certs/signing_key.x509"
fi
```

#### Out-of-Tree 构建原理

```
Out-of-Tree vs In-Tree 构建:

┌────────────────────────────────────────────────────────┐
│  In-Tree 构建 (内核源码树内)                            │
│                                                        │
│  /usr/src/linux-6.6.0/                                 │
│  └── drivers/gpu/drm/amd/amdgpu/                       │
│      ├── Makefile                                       │
│      ├── amdgpu_device.c                               │
│      ├── amdgpu_gem.c                                  │
│      └── ...                                            │
│                                                        │
│  make -C /usr/src/linux-6.6.0 M=drivers/gpu/drm/amd/   │
│                                                        │
└────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────┐
│  Out-of-Tree 构建 (独立源码树)                          │
│                                                        │
│  /home/user/amdgpu-dkms/                              │
│  ├── Makefile   ← 调用内核构建系统的接口                │
│  ├── amdgpu/                                          │
│  │   ├── Makefile                                      │
│  │   ├── amdgpu_device.c                              │
│  │   └── ...                                           │
│  └── dkms.conf                                         │
│                                                        │
│  make -C /lib/modules/$(uname -r)/build M=$PWD         │
│                                                        │
└────────────────────────────────────────────────────────┘

Out-of-Tree Makefile 模板:

obj-m := amdgpu.o

amdgpu-y := \
    amdgpu_device.o \
    amdgpu_gem.o \
    amdgpu_kms.o \
    amdgpu_drv.o \
    amdgpu_fence.o \
    amdgpu_ring.o \
    amdgpu_irq.o \
    amdgpu_bo_list.o \
    amdgpu_cs.o \
    amdgpu_bios.o \
    amdgpu_benchmark.o \
    amdgpu_test.o \
    amdgpu_pm.o \
    amdgpu_pll.o \
    amdgpu_ucode.o \
    amdgpu_acp.o \
    amdgpu_gart.o \
    amdgpu_vm.o \
    amdgpu_vm_pt.o \
    amdgpu_ttm.o \
    amdgpu_object.o \
    amdgpu_trace_points.o \
    amdgpu_atombios.o \
    amdgpu_display.o \
    amdgpu_i2c.o \
    amdgpu_fb.o \
    amdgpu_gem_prime.o \
    amdgpu_psp.o \
    amdgpu_smu.o \
    amdgpu_xgmi.o \
    amdgpu_virt.o \
    amdgpu_gmc.o \
    amdgpu_mes.o \
    amdgpu_rlc.o \
    amdgpu_ras.o \
    gfxhub_v1_0.o \
    gfxhub_v1_1.o \
    gfxhub_v2_0.o \
    mmhub_v1_0.o \
    mmhub_v2_0.o \
    mmhub_v9_4.o \
    mmhub_v1_7.o \
    nbio_v6_1.o \
    nbio_v7_0.o \
    nbio_v7_2.o \
    nbio_v7_4.o \
    soc15.o \
    nv.o \
    arcturus.o \
    aldebaran.o \
    aqua_vanjeram.o \
    vega10_reg_init.o \
    vega20_reg_init.o \
    vega12_reg_init.o \
    gmc_v8_0.o \
    gmc_v9_0.o \
    gfx_v8_0.o \
    gfx_v9_0.o \
    gfx_v9_4.o \
    gfx_v10_0.o \
    sdma_v3_0.o \
    sdma_v4_0.o \
    sdma_v5_0.o \
    sdma_v5_2.o \
    vcn_v1_0.o \
    vcn_v2_0.o \
    vcn_v2_5.o \
    jpeg_v1_0.o \
    jpeg_v2_0.o

# 内核构建系统接口
KBUILD_EXTRA_SYMBOLS := $(obj)/Module.symvers

all:
    $(MAKE) -C $(KERNELDIR) M=$(PWD) modules

clean:
    $(MAKE) -C $(KERNELDIR) M=$(PWD) clean

install:
    $(MAKE) -C $(KERNELDIR) M=$(PWD) modules_install
    depmod -a
```

#### 模块签名机制

```
模块签名流程:

┌─────────────────────────────────────────────────────┐
│                 模块签名机制                          │
├─────────────────────────────────────────────────────┤
│                                                      │
│  为什么需要签名:                                      │
│  - Secure Boot 要求内核模块必须签名                  │
│  - 未签名的模块在 Secure Boot 启用时会被拒绝加载      │
│                                                      │
│  签名流程:                                            │
│                                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐       │
│  │ 模块源码  │ → │  编译     │ → │ amdgpu.ko│       │
│  └──────────┘    └──────────┘    └────┬─────┘       │
│                                       │               │
│                                       ▼               │
│  ┌────────────────────────────────────────────┐       │
│  │ scripts/sign-file sha256                    │       │
│  │   certs/signing_key.pem                     │       │
│  │   certs/signing_key.x509                    │       │
│  │   amdgpu.ko                                │       │
│  └────────────────────────────────────────────┘       │
│                                       │               │
│                                       ▼               │
│  ┌──────────┐                                      │
│  │amdgpu.ko │ ← PKCS#7 签名附加在模块末尾           │
│  │+ signature│  ~模块尾部添加 ~512 字节签名         │
│  └──────────┘                                       │
│                                                      │
│  验证过程:                                            │
│  1. 内核加载模块                                       │
│  2. 提取 PKCS#7 签名                                  │
│  3. 使用系统信任的密钥验证                             │
│  4. 签名有效 → 加载模块                               │
│  5. 签名无效 → 拒绝加载                               │
│                                                      │
└─────────────────────────────────────────────────────┘

自己生成签名密钥:

# 生成私钥和证书
openssl req -new -x509 -newkey rsa:4096 \
    -keyout signing_key.pem \
    -outform DER -out signing_key.x509 \
    -nodes -days 36500 \
    -subj "/CN=GPU Driver Module/"

# 签名模块
/usr/src/kernels/$(uname -r)/scripts/sign-file \
    sha256 signing_key.pem signing_key.x509 amdgpu.ko

# 验证签名
modinfo amdgpu.ko | grep signature
# signature: PKCS#7, RSA 4096-bit
```

### 实践操作

#### DKMS 完整配置示例

```bash
#!/bin/bash
# 为 amdgpu 配置 DKMS

set -e

echo "=============================================="
echo "  AMDGPU DKMS 配置"
echo "=============================================="
echo ""

# 配置变量
DKMS_PACKAGE_NAME="amdgpu"
DKMS_PACKAGE_VERSION="6.1.0"
DKMS_SOURCE_DIR="/usr/src/${DKMS_PACKAGE_NAME}-${DKMS_PACKAGE_VERSION}"
KERNEL_SRC_DIR="/lib/modules/$(uname -r)/build"

# 阶段 1: 准备源码
prepare_source() {
    echo "[1/5] 准备源码..."
    
    # 创建源码目录
    sudo mkdir -p "${DKMS_SOURCE_DIR}"
    
    # 复制 amdgpu 驱动源码
    # (假设从内核源码中提取或从 AMD 获取)
    echo "  从内核源码复制 amdgpu 驱动..."
    
    KERNEL_AMDGPU_SRC="${KERNEL_SRC_DIR}/../drivers/gpu/drm/amd/amdgpu"
    if [ -d "${KERNEL_AMDGPU_SRC}" ]; then
        sudo cp -r "${KERNEL_AMDGPU_SRC}" "${DKMS_SOURCE_DIR}/amdgpu"
        echo "  ✅ 已从内核源码复制 amdgpu 驱动"
    else
        echo "  ❌ 内核源码中未找到 amdgpu 驱动"
        echo "  请从 https://gitlab.freedesktop.org/drm/amd 下载"
        exit 1
    fi
    
    # 复制头文件依赖
    for include_dir in "drm" "uapi/drm" "linux" "asm-generic"; do
        if [ -d "${KERNEL_SRC_DIR}/include/${include_dir}" ]; then
            sudo mkdir -p "${DKMS_SOURCE_DIR}/include/${include_dir}"
            sudo cp -r "${KERNEL_SRC_DIR}/include/${include_dir}/"*.h \
                "${DKMS_SOURCE_DIR}/include/${include_dir}/" 2>/dev/null || true
        fi
    done
    
    # 创建 dkms.conf
    create_dkms_conf
    
    # 创建 Makefile
    create_makefile
    
    echo "  ✅ 源码准备完成"
}

# 阶段 2: 创建 dkms.conf
create_dkms_conf() {
    echo "[2/5] 创建 dkms.conf..."
    
    sudo tee "${DKMS_SOURCE_DIR}/dkms.conf" > /dev/null << EOF
PACKAGE_NAME="${DKMS_PACKAGE_NAME}"
PACKAGE_VERSION="${DKMS_PACKAGE_VERSION}"

MAKE[0]="make -C \${kernel_source_dir} M=\${dkms_tree}/\${PACKAGE_NAME}-\${PACKAGE_VERSION}/amdgpu modules"

CLEAN="make -C \${kernel_source_dir} M=\${dkms_tree}/\${PACKAGE_NAME}-\${PACKAGE_VERSION}/amdgpu clean"

BUILT_MODULE_NAME[0]="amdgpu"
BUILT_MODULE_LOCATION[0]="amdgpu"
DEST_MODULE_LOCATION[0]="/kernel/drivers/gpu/drm/amd/amdgpu"

AUTOINSTALL="yes"
REMAKE_INITRD="yes"
STRIP="yes"
EOF
    
    echo "  ✅ dkms.conf 已创建"
}

# 阶段 3: 创建构建用 Makefile
create_makefile() {
    echo "[3/5] 创建 Makefile..."
    
    sudo tee "${DKMS_SOURCE_DIR}/amdgpu/Makefile" > /dev/null << 'MAKEFILE'
# AMDGPU DKMS Makefile
# 调用内核构建系统

MODULE_NAME := amdgpu

# 模块对象文件
$(MODULE_NAME)-y := \
    amdgpu_device.o \
    amdgpu_gem.o \
    amdgpu_kms.o \
    amdgpu_drv.o \
    amdgpu_fence.o \
    amdgpu_ring.o \
    amdgpu_irq.o \
    amdgpu_virt.o \
    amdgpu_psp.o \
    amdgpu_smu.o \
    amdgpu_gmc.o \
    amdgpu_xgmi.o \
    amdgpu_ras.o

obj-m += $(MODULE_NAME).o

# 添加额外的包含路径
ccflags-y += -I$(src)/../include

all:
    $(MAKE) -C $(KERNELRELEASE) M=$(PWD) modules

clean:
    $(MAKE) -C $(KERNELRELEASE) M=$(PWD) clean

install:
    $(MAKE) -C $(KERNELRELEASE) M=$(PWD) modules_install
    depmod -a
MAKEFILE
    
    echo "  ✅ Makefile 已创建"
}

# 阶段 4: DKMS 注册和构建
register_and_build() {
    echo "[4/5] DKMS 注册和构建..."
    
    # 添加模块到 DKMS
    sudo dkms add "${DKMS_SOURCE_DIR}"
    echo "  ✅ DKMS 注册成功"
    
    # 构建模块
    echo "  正在构建..."
    sudo dkms build -m "${DKMS_PACKAGE_NAME}" \
                     -v "${DKMS_PACKAGE_VERSION}" \
                     -k "$(uname -r)"
    
    if [ $? -eq 0 ]; then
        echo "  ✅ 构建成功"
    else
        echo "  ❌ 构建失败，查看日志:"
        sudo dkms status -m "${DKMS_PACKAGE_NAME}" \
                          -v "${DKMS_PACKAGE_VERSION}"
        
        echo ""
        echo "  构建日志位置:"
        echo "  /var/lib/dkms/${DKMS_PACKAGE_NAME}/${DKMS_PACKAGE_VERSION}/build/make.log"
        exit 1
    fi
}

# 阶段 5: 安装和测试
install_and_test() {
    echo "[5/5] 安装和测试..."
    
    # 安装模块
    sudo dkms install -m "${DKMS_PACKAGE_NAME}" \
                       -v "${DKMS_PACKAGE_VERSION}" \
                       -k "$(uname -r)"
    echo "  ✅ 安装成功"
    
    # 检查模块位置
    echo ""
    echo "  模块位置:"
    find /lib/modules/$(uname -r) -name "amdgpu.ko" -type f | while read line; do
        echo "    ${line}"
    done
    
    # 检查模块信息
    echo ""
    echo "  模块信息:"
    modinfo amdgpu | head -10 | while read line; do
        echo "    ${line}"
    done
    
    # (可选) 卸载旧模块加载新模块
    # sudo rmmod amdgpu
    # sudo modprobe amdgpu
    
    echo ""
    echo "  ✅ AMDGPU DKMS 配置完成"
    echo "  下次内核更新时将自动重建该模块"
}

# 主流程
prepare_source
register_and_build
install_and_test

echo ""
echo "=============================================="
echo "  DKMS 配置完成"
echo "=============================================="
```

#### Out-of-Tree 构建独立环境

```bash
#!/bin/bash
# Out-of-tree 构建环境配置

set -e

echo "=============================================="
echo "  AMDGPU Out-of-Tree 构建环境"
echo "=============================================="
echo ""

# 配置
BUILD_ROOT="${HOME}/amdgpu-build"
AMDGPU_SRC="${BUILD_ROOT}/src"
BUILD_LOG="${BUILD_ROOT}/build.log"
KERNEL_VERSION=$(uname -r)
KERNEL_BUILD="/lib/modules/${KERNEL_VERSION}/build"

# 环境设置
setup_environment() {
    echo "[1/6] 创建构建环境..."
    
    mkdir -p "${BUILD_ROOT}"
    mkdir -p "${AMDGPU_SRC}"
    
    # 检查内核构建系统
    if [ ! -d "${KERNEL_BUILD}" ]; then
        echo "  ❌ 内核构建系统不存在: ${KERNEL_BUILD}"
        echo "  请安装: linux-headers-${KERNEL_VERSION}"
        exit 1
    fi
    
    echo "  ✅ 内核构建目录: ${KERNEL_BUILD}"
    echo "  ✅ 构建根目录: ${BUILD_ROOT}"
    
    # 创建环境变量文件
    cat > "${BUILD_ROOT}/env.sh" << EOF
#!/bin/bash
# AMDGPU 构建环境变量

export AMDGPU_SRC="${AMDGPU_SRC}"
export KERNEL_VERSION="${KERNEL_VERSION}"
export KERNEL_BUILD="${KERNEL_BUILD}"
export CROSS_COMPILE=""
export ARCH="$(uname -m)"

# 额外编译标志
export EXTRA_CFLAGS="-Wall -Werror"
EOF
    chmod +x "${BUILD_ROOT}/env.sh"
    
    echo "  ✅ 环境变量已保存: ${BUILD_ROOT}/env.sh"
}

# 获取源码
get_source() {
    echo "[2/6] 获取 amdgpu 源码..."
    
    if [ -f "${AMDGPU_SRC}/Makefile" ]; then
        echo "  ⚠️  源码已存在，跳过下载"
        return
    fi
    
    # 从上游获取源码
    echo "  从上游获取 (可能需要 git)..."
    
    if command -v git &> /dev/null; then
        # 克隆 amd-staging 分支
        git clone \
            https://gitlab.freedesktop.org/drm/amd.git \
            "${AMDGPU_SRC}" \
            --depth 1 \
            --branch amd-staging 2>&1 | tail -5
        
        echo "  ✅ 源码已克隆"
    else
        echo "  ❌ git 不可用"
        # 回退: 从内核源码复制
        echo "  从内核源码复制..."
        cp -r "${KERNEL_BUILD}/../drivers/gpu/drm/amd/amdgpu"/* \
            "${AMDGPU_SRC}/" 2>/dev/null || true
        echo "  ⚠️  已复制内核中的 amdgpu 源码"
    fi
}

# 配置构建
configure_build() {
    echo "[3/6] 配置构建参数..."
    
    # 检查内核配置文件
    local kernel_config="${KERNEL_BUILD}/.config"
    if [ ! -f "${kernel_config}" ]; then
        echo "  ⚠️  内核 .config 不存在"
        # 生成默认配置
        make -C "${KERNEL_BUILD}" defconfig 2>/dev/null || true
    fi
    
    # 确保 amdgpu 相关配置
    local needed_configs=(
        "CONFIG_DRM_AMDGPU=m"
        "CONFIG_DRM_AMDGPU_SI=n"
        "CONFIG_DRM_AMDGPU_CIK=n"
        "CONFIG_DRM_AMD_DC=y"
    )
    
    echo "  检查内核配置..."
    for config in "${needed_configs[@]}"; do
        local config_name=$(echo "${config}" | cut -d= -f1)
        if grep -q "^${config}" "${kernel_config}" 2>/dev/null; then
            echo "  ✅ ${config}"
        else
            echo "  ⚠️  ${config_name} 未设置 (使用默认)"
        fi
    done
    
    # 创建本地构建配置
    cat > "${BUILD_ROOT}/config.mk" << EOF
# 本地构建配置
KERNELDIR := ${KERNEL_BUILD}
ARCH := $(uname -m)
CROSS_COMPILE :=

# 编译选项
EXTRA_CFLAGS += -DAMDGPU_DKMS
EXTRA_CFLAGS += -include ${KERNEL_BUILD}/include/generated/autoconf.h
EOF
    
    echo "  ✅ 构建配置已生成"
}

# 构建模块
build_module() {
    echo "[4/6] 构建模块..."
    echo "  开始时间: $(date)"
    echo ""
    
    cd "${AMDGPU_SRC}"
    
    # 执行构建
    make -C "${KERNEL_BUILD}" \
         M="${AMDGPU_SRC}" \
         modules 2>&1 | \
         tee "${BUILD_LOG}" | \
         while read line; do
        if echo "${line}" | grep -qi "error"; then
            echo "  ❌ ${line}"
        elif echo "${line}" | grep -qi "warning"; then
            echo "  ⚠️  ${line}"
        fi
    done
    
    # 检查结果
    if [ -f "${AMDGPU_SRC}/amdgpu.ko" ]; then
        local size=$(stat -c%s "${AMDGPU_SRC}/amdgpu.ko" 2>/dev/null || \
                     stat -f%z "${AMDGPU_SRC}/amdgpu.ko" 2>/dev/null)
        echo ""
        echo "  ✅ 构建成功"
        echo "  模块大小: $((size / 1024)) KB"
        
        # 显示模块信息
        echo ""
        echo "  模块信息:"
        modinfo "${AMDGPU_SRC}/amdgpu.ko" | head -5
    else
        echo ""
        echo "  ❌ 构建失败"
        echo "  查看日志: ${BUILD_LOG}"
        return 1
    fi
}

# 测试模块
test_module() {
    echo "[5/6] 测试模块..."
    
    local ko_file="${AMDGPU_SRC}/amdgpu.ko"
    
    if [ ! -f "${ko_file}" ]; then
        echo "  ❌ 模块文件不存在"
        return 1
    fi
    
    # 1. 语法检查
    echo "  1. 模块格式检查..."
    if modinfo "${ko_file}" &>/dev/null; then
        echo "     ✅ 模块格式有效"
    else
        echo "     ❌ 模块格式无效"
        return 1
    fi
    
    # 2. 符号检查
    echo "  2. 符号检查..."
    local unresolved=$(modprobe --dump-modversions "${ko_file}" 2>/dev/null | \
                      grep -c "unknown" || true)
    if [ "${unresolved}" -eq 0 ]; then
        echo "     ✅ 所有符号已解析"
    else
        echo "     ⚠️  有 ${unresolved} 个未解析符号"
    fi
    
    # 3. 签名检查
    echo "  3. 签名检查..."
    if modinfo "${ko_file}" | grep -q "signature"; then
        echo "     ✅ 模块已签名"
    else
        echo "     ⚠️  模块未签名 (Secure Boot 可能拒绝)"
    fi
    
    # 4. 加载测试 (如果当前没有 amdgpu)
    echo "  4. 加载测试..."
    if lsmod | grep -q "^amdgpu"; then
        echo "     ⚠️  amdgpu 已加载，跳过加载测试"
    else
        echo "     ⚠️  需要 root 权限加载测试"
    fi
    
    echo ""
    echo "  模块测试完成"
}

# 部署模块
deploy_module() {
    echo "[6/6] 部署模块..."
    
    local ko_file="${AMDGPU_SRC}/amdgpu.ko"
    local dest_dir="/lib/modules/${KERNEL_VERSION}/extra/amdgpu"
    local dest_file="${dest_dir}/amdgpu.ko"
    
    if [ ! -f "${ko_file}" ]; then
        echo "  ❌ 模块不存在"
        return 1
    fi
    
    # 创建目标目录
    sudo mkdir -p "${dest_dir}"
    
    # 备份现有模块
    if [ -f "${dest_file}" ]; then
        echo "  备份现有模块..."
        sudo cp "${dest_file}" "${dest_file}.bak"
    fi
    
    # 复制新模块
    sudo cp "${ko_file}" "${dest_file}"
    echo "  ✅ 模块已部署到 ${dest_file}"
    
    # 更新模块依赖
    sudo depmod -a
    echo "  ✅ 模块依赖已更新"
    
    # 创建部署记录
    local deploy_log="${BUILD_ROOT}/deploy.log"
    echo "$(date): Deployed ${ko_file} to ${dest_file}" >> "${deploy_log}"
    echo "  部署记录已保存: ${deploy_log}"
}

# 主流程
case "$1" in
    setup)
        setup_environment
        get_source
        configure_build
        echo ""
        echo "环境已准备好。运行: $0 build"
        ;;
    build)
        source "${BUILD_ROOT}/env.sh" 2>/dev/null || true
        build_module
        ;;
    test)
        test_module
        ;;
    deploy)
        deploy_module
        ;;
    all)
        setup_environment
        get_source
        configure_build
        build_module
        test_module
        deploy_module
        ;;
    *)
        echo "用法: $0 {setup|build|test|deploy|all}"
        echo ""
        echo "  setup  - 初始化构建环境"
        echo "  build  - 编译模块"
        echo "  test   - 测试模块"
        echo "  deploy - 部署模块到系统"
        echo "  all    - 完整流程"
        echo ""
        echo "示例:"
        echo "  $0 setup   # 首次使用"
        echo "  $0 build   # 编译"
        echo "  $0 deploy  # 部署"
        ;;
esac
```

#### 模块签名配置

```bash
#!/bin/bash
# 模块签名配置

echo "=============================================="
echo "  内核模块签名工具"
echo "=============================================="
echo ""

# 检查 Secure Boot 状态
check_secure_boot() {
    echo "[1/5] 检查 Secure Boot 状态..."
    
    if [ -d /sys/firmware/efi ]; then
        local sb_state=$(mokutil --sb-state 2>/dev/null || echo "unknown")
        echo "  Secure Boot: ${sb_state}"
        
        if [ "${sb_state}" = "enabled" ]; then
            echo "  ⚠️  Secure Boot 已启用，模块必须签名"
            return 0
        else
            echo "  ✅ Secure Boot 未启用，模块不需要签名"
            return 1
        fi
    else
        echo "  ⚠️  非 UEFI 系统"
        return 1
    fi
}

# 生成签名密钥
generate_keys() {
    echo "[2/5] 生成签名密钥..."
    
    local key_dir="${HOME}/.module-signing"
    mkdir -p "${key_dir}"
    
    local key_file="${key_dir}/signing_key.pem"
    local cert_file="${key_dir}/signing_key.x509"
    
    if [ -f "${key_file}" ] && [ -f "${cert_file}" ]; then
        echo "  ⚠️  密钥已存在:"
        echo "  Key:  ${key_file}"
        echo "  Cert: ${cert_file}"
        return
    fi
    
    # 生成 RSA 4096 密钥对
    openssl req -new -x509 -newkey rsa:4096 \
        -keyout "${key_file}" \
        -outform DER -out "${cert_file}" \
        -nodes -days 36500 \
        -subj "/O=GPU Driver Development/CN=AMDGPU Module Signing/"
    
    echo "  ✅ 密钥已生成:"
    echo "  私钥: ${key_file}"
    echo "  证书: ${cert_file}"
}

# 注册密钥到 MOK
register_mok_key() {
    echo "[3/5] 注册 MOK 密钥..."
    
    local cert_file="${HOME}/.module-signing/signing_key.x509"
    
    if [ ! -f "${cert_file}" ]; then
        echo "  ❌ 证书文件不存在"
        return 1
    fi
    
    echo "  注册证书到 MOK (Machine Owner Key)..."
    echo "  系统将在下次重启时提示输入密码"
    echo ""
    
    sudo mokutil --import "${cert_file}"
    
    if [ $? -eq 0 ]; then
        echo "  ✅ MOK 注册请求已提交"
        echo "  请在下次重启时完成注册"
    else
        echo "  ❌ MOK 注册失败"
    fi
}

# 签名模块
sign_module() {
    echo "[4/5] 签名模块..."
    
    local ko_file="$1"
    local key_dir="${HOME}/.module-signing"
    local key_file="${key_dir}/signing_key.pem"
    local cert_file="${key_dir}/signing_key.x509"
    
    if [ ! -f "${ko_file}" ]; then
        echo "  ❌ 模块文件不存在: ${ko_file}"
        return 1
    fi
    
    if [ ! -f "${key_file}" ] || [ ! -f "${cert_file}" ]; then
        echo "  ❌ 签名密钥不存在"
        echo "  请先运行: $0 gen-keys"
        return 1
    fi
    
    # 查找 sign-file 脚本
    local sign_tool="/usr/src/kernels/$(uname -r)/scripts/sign-file"
    if [ ! -f "${sign_tool}" ]; then
        sign_tool="/lib/modules/$(uname -r)/build/scripts/sign-file"
    fi
    
    if [ ! -f "${sign_tool}" ]; then
        echo "  ❌ sign-file 工具未找到"
        echo "  请安装 linux-headers 包"
        return 1
    fi
    
    # 备份原始模块
    cp "${ko_file}" "${ko_file}.unsigned"
    
    # 签名模块
    "${sign_tool}" sha256 \
        "${key_file}" \
        "${cert_file}" \
        "${ko_file}"
    
    if [ $? -eq 0 ]; then
        echo "  ✅ 模块签名成功: ${ko_file}"
        
        # 验证签名
        echo ""
        echo "  签名验证:"
        modinfo "${ko_file}" | grep -i "signature\|sig_hash"
    else
        echo "  ❌ 签名失败"
        # 恢复原始模块
        mv "${ko_file}.unsigned" "${ko_file}"
    fi
}

# 批量签名 DKMS 模块
sign_dkms_modules() {
    echo "[5/5] 签名所有 DKMS 模块..."
    
    local dkms_status=$(sudo dkms status 2>/dev/null)
    echo "  DKMS 模块列表:"
    
    echo "${dkms_status}" | while read line; do
        echo "    ${line}"
        local module_info=$(echo "${line}" | awk '{print $1}')
        local module_name=$(echo "${module_info}" | cut -d/ -f1)
        local module_version=$(echo "${module_info}" | cut -d/ -f2)
        
        # 查找模块 .ko 文件
        local ko_file=$(find /lib/modules/$(uname -r)/extra \
                       -name "${module_name}.ko" 2>/dev/null | head -1)
        
        if [ -n "${ko_file}" ]; then
            echo "    签名为: ${ko_file}"
            sign_module "${ko_file}"
        fi
    done
}

# 主流程
case "$1" in
    check)
        check_secure_boot
        ;;
    gen-keys)
        generate_keys
        ;;
    register)
        register_mok_key
        ;;
    sign)
        if [ -z "$2" ]; then
            echo "用法: $0 sign <module.ko>"
            exit 1
        fi
        sign_module "$2"
        ;;
    sign-dkms)
        sign_dkms_modules
        ;;
    all)
        check_secure_boot
        if [ $? -eq 0 ]; then
            generate_keys
            register_mok_key
            sign_dkms_modules
            echo ""
            echo "  请重启以完成 MOK 注册"
        else
            echo "  不需要签名 (Secure Boot 未启用)"
        fi
        ;;
    *)
        echo "用法: $0 {check|gen-keys|register|sign|sign-dkms|all}"
        echo ""
        echo "  check       - 检查 Secure Boot 状态"
        echo "  gen-keys    - 生成签名密钥"
        echo "  register    - 注册 MOK 密钥"
        echo "  sign <ko>   - 签名单个模块"
        echo "  sign-dkms   - 签名所有 DKMS 模块"
        echo "  all         - 完整签名流程"
        ;;
esac
```

### 案例分析

#### 案例一：DKMS 构建在多内核版本环境下失败

**问题现象：**
```
系统安装了 5.15 和 6.6 两个内核
DKMS 为 5.15 构建成功，为 6.6 构建失败
错误:
/var/lib/dkms/amdgpu/6.1.0/6.6.0/build/make.log:
  error: 'struct drm_gem_object' has no member named 'vma_node'
```

**诊断过程：**
```bash
# 1. 检查 DKMS 状态
dkms status
# amdgpu/6.1.0, 5.15.0: installed
# amdgpu/6.1.0, 6.6.0: failed

# 2. 查看构建日志
cat /var/lib/dkms/amdgpu/6.1.0/6.6.0/build/make.log | grep error

# 3. 确定原因
# 内核 6.6 移除了 drm_gem_object.vma_node
# 源码需要条件编译适配
```

**解决方案：**
```bash
# 方案 1: 为 6.6 内核创建补丁
cat > /usr/src/amdgpu-6.1.0/compat-6.6.patch << 'EOF'
--- a/amdgpu_gem.c
+++ b/amdgpu_gem.c
@@ -123,7 +123,11 @@ static int amdgpu_gem_object_mmap(struct drm_gem_object *obj,
     struct amdgpu_device *adev = amdgpu_get_adev(obj->dev);
     struct amdgpu_bo *bo = gem_to_amdgpu_bo(obj);
     
+#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 6, 0)
+    struct drm_vma_offset_node *node = drm_gem_object_get_vma_node(obj);
+#else
     struct drm_vma_offset_node *node = &obj->vma_node;
+#endif
     
     /* ... */
 }
EOF

# 应用补丁 (仅 6.6 内核)
cd /usr/src/amdgpu-6.1.0
patch -p1 < compat-6.6.patch

# 重建
sudo dkms build -m amdgpu -v 6.1.0 -k 6.6.0
sudo dkms install -m amdgpu -v 6.1.0 -k 6.6.0

# 方案 2: 使用 dkms.conf 的内核版本条件
# 修改 dkms.conf 添加版本检测
cat >> /usr/src/amdgpu-6.1.0/dkms.conf << 'EOF'
# 内核版本特定配置
MAKE[1]="make -C ${kernel_source_dir} M=${dkms_tree}/${PACKAGE_NAME}-${PACKAGE_VERSION}/amdgpu modules EXTRA_CFLAGS=-DLINUX_VERSION_CODE=${kernelver}"

# 只有指定内核版本才执行的命令
POST_BUILD[0]="patch -p0 < /usr/src/amdgpu-6.1.0/post-build-patch.patch"
EOF

# 方案 3: 更新源码到支持新内核的版本
# 从上游获取最新代码
```

#### 案例二：Secure Boot 导致 DKMS 模块加载失败

**问题现象：**
```
启用 Secure Boot 后，amdgpu 模块加载失败:
[    5.678] amdgpu: module verification failed: signature and/or required key missing - tainting kernel
[    5.679] amdgpu: module loading failed (err -126)
```

**诊断过程：**
```bash
# 1. 检查 Secure Boot
mokutil --sb-state
# SecureBoot: enabled

# 2. 检查模块签名
modinfo amdgpu | grep signature
# (无输出 - 模块未签名)

# 3. 检查内核要求
cat /sys/module/module/parameters/sig_enforce
# 1 (强制签名验证)
```

**解决方案：**
```bash
# 方案 1: 签名 DKMS 模块
# 生成密钥
openssl req -new -x509 -newkey rsa:4096 \
    -keyout /root/signing_key.pem \
    -outform DER -out /root/signing_key.x509 \
    -nodes -days 36500 \
    -subj "/CN=AMDGPU DKMS/"

# 注册 MOK
sudo mokutil --import /root/signing_key.x509
# 重启完成注册

# 配置 DKMS 自动签名
cat > /etc/dkms/signing.conf << 'EOF'
# DKMS 自动签名配置
sign_tool="/lib/modules/$(uname -r)/build/scripts/sign-file"
sign_key="/root/signing_key.pem"
sign_cert="/root/signing_key.x509"
EOF

# 重建并签名 DKMS 模块
sudo dkms remove amdgpu/6.1.0 --all
sudo dkms install amdgpu/6.1.0

# 方案 2: 临时禁用 Secure Boot
# BIOS 设置中禁用 Secure Boot
# 或通过 mokutil
sudo mokutil --disable-validation
# 重启生效

# 方案 3: 使用内核已签名的模块
# 安装发行版提供的已签名 amdgpu 包
sudo apt install linux-modules-extra-$(uname -r)
```

### 相关链接

- [DKMS 官方文档](https://linux.dell.com/dkms/)
- [DKMS Git 仓库](https://github.com/dell/dkms)
- [Linux 内核模块签名文档](https://www.kernel.org/doc/html/latest/admin-guide/module-signing.html)
- [AMDGPU DKMS 源码](https://gitlab.freedesktop.org/drm/amd)
- [Ubuntu DKMS 指南](https://wiki.ubuntu.com/Kernel/Dev/DKMS)

### 今日小结

DKMS 和 out-of-tree 构建是 GPU 驱动开发部署的核心技术：

1. **DKMS 框架**: 内核模块自动重建系统，内核更新时自动适配
2. **配置要点**: dkms.conf 定义构建规则，Makefile 调用内核构建系统
3. **构建流程**: add → build → install，三步完成模块部署
4. **Out-of-Tree 构建**: 独立于内核源码树的模块开发方式
5. **模块签名**: Secure Boot 环境下必须签名模块
6. **版本兼容**: 不同内核版本需要条件编译适配
7. **部署策略**: DKMS 自动安装 vs 手动 out-of-tree 部署

### 扩展思考

1. 如何设计一个 CI/CD 流水线，自动为多个内核版本构建和测试 DKMS 模块？
2. 在企业环境中，如何批量管理多台机器的 DKMS 模块签名？
3. out-of-tree 构建的性能优化：是否可以使用分布式编译（distcc）加速？
4. DKMS 的模块加载顺序问题：多个 DKMS 模块的依赖关系如何处理？
