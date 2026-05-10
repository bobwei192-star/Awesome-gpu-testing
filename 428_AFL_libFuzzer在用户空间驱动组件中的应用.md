# AFL / libFuzzer 在用户空间驱动组件中的应用

## 学习目标

- 理解 AFL 和 libFuzzer 的原理及在 GPU 驱动测试中的适用场景
- 掌握 AFL 的安装、配置和运行方法
- 学会使用 libFuzzer 对 Mesa 等用户空间组件进行 Fuzz 测试
- 了解用户空间 Fuzz 与内核 Fuzz 的互补关系
- 能够将用户空间 Fuzz 集成到 GPU 驱动测试流程中

## 知识详解

### 一、概念原理

#### 1.1 用户空间 vs 内核空间 Fuzz 测试

```
┌─────────────────────────────────────────────────────────────────────┐
│              用户空间 vs 内核空间 Fuzz 测试                          │
│                                                                     │
│   内核空间 Fuzz (syzkaller)          用户空间 Fuzz (AFL/libFuzzer) │
│   ─────────────────────────          ────────────────────────────── │
│                                                                     │
│   ● 目标: 内核驱动接口                ● 目标: 用户空间库/工具        │
│   ● 入口: ioctl / sysfs               ● 入口: 文件 / 网络 / API     │
│   ● 工具: syzkaller                   ● 工具: AFL / libFuzzer       │
│   ● 环境: QEMU VM                     ● 环境: 原生进程              │
│   ● 速度: 较慢（VM 开销）             ● 速度: 快（原生执行）         │
│                                                                     │
│   GPU 驱动中的应用：                                                 │
│   ┌─────────────────────────────────────────────────────────┐      │
│   │  内核空间                    用户空间                     │      │
│   │  ─────────────               ─────────────              │      │
│   │  ● amdgpu 内核模块           ● Mesa 3D 驱动             │      │
│   │  ● DRM 子系统                ● libdrm 库                │      │
│   │  ● KFD 计算驱动              ● ROCm 运行时              │      │
│   │  ● 显示控制器                ● Vulkan/OpenGL 应用       │      │
│   │  ● 内存管理                  ● 着色器编译器              │      │
│   └─────────────────────────────────────────────────────────┘      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 1.2 AFL 原理

```
┌─────────────────────────────────────────────────────────────────────┐
│              AFL (American Fuzzy Lop) 原理                           │
│                                                                     │
│   编译阶段                    运行阶段                               │
│   ──────────                  ──────────                            │
│                                                                     │
│   ┌─────────────┐             ┌─────────────┐                      │
│   │ 源代码      │             │ 输入语料库  │                      │
│   │             │             │             │                      │
│   │ ● 插桩      │             │ ● 种子文件  │                      │
│   │   (afl-gcc) │             │ ● 最小集合  │                      │
│   │             │             │             │                      │
│   │ ● 生成      │             │ ● 覆盖率    │                      │
│   │   控制流图  │             │   反馈      │                      │
│   └──────┬──────┘             └──────┬──────┘                      │
│          │                           │                              │
│          ▼                           ▼                              │
│   ┌─────────────┐             ┌─────────────┐                      │
│   │ 插桩二进制  │◀────────────│ 变异引擎    │                      │
│   │             │   覆盖率    │             │                      │
│   │ ● 边覆盖    │   反馈      │ ● 位翻转    │                      │
│   │   计数器    │             │ ● 算术变异  │                      │
│   │             │             │ ● 字典插入  │                      │
│   │ ● 崩溃检测  │────────────▶│ ● 拼接      │                      │
│   │             │   新输入    │             │                      │
│   └─────────────┘             └─────────────┘                      │
│                                                                     │
│   关键特点：                                                         │
│   ┌─────────────────────────────────────────────────────────┐      │
│   │  1. 编译时插桩，运行时收集边覆盖                          │      │
│   │  2. 遗传算法选择高价值输入进行深度变异                    │      │
│   │  3. 自动检测崩溃和挂起                                    │      │
│   │  4. 支持并行 fuzz（多核/多机）                            │      │
│   └─────────────────────────────────────────────────────────┘      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 二、实践操作

#### 2.1 AFL 安装与配置

```bash
#!/bin/bash
# install_afl.sh - AFL 安装脚本

set -e

AFL_DIR="${1:-$HOME/afl}"

echo "=== AFL 安装 ==="
echo ""

# 安装依赖
sudo apt-get update
sudo apt-get install -y \
    build-essential \
    clang \
    llvm \
    libtool \
    libtool-bin \
    bison \
    flex

# 下载 AFL
git clone https://github.com/google/AFL.git "$AFL_DIR"
cd "$AFL_DIR"

# 编译 AFL
make
sudo make install

# 编译 AFL LLVM 模式（推荐）
cd llvm_mode
make
sudo make install

echo "AFL 安装完成"
echo ""
echo "关键工具:"
echo "  afl-gcc      - GCC 插桩包装器"
echo "  afl-clang    - Clang 插桩包装器"
echo "  afl-fuzz     - Fuzz 主程序"
echo "  afl-showmap  - 覆盖率查看"
echo "  afl-tmin     - 输入最小化"
echo "  afl-cmin     - 语料库最小化"
```

#### 2.2 Mesa 着色器编译器 Fuzz 测试

```bash
#!/bin/bash
# fuzz_mesa_compiler.sh - Mesa 着色器编译器 Fuzz 测试

set -e

MESA_DIR="${1:-$HOME/mesa}"
AFL_DIR="${2:-$HOME/afl}"
OUTPUT_DIR="${3:-$HOME/mesa-fuzzing}"

mkdir -p "$OUTPUT_DIR"

echo "=== Mesa 着色器编译器 Fuzz 测试 ==="
echo ""

# 1. 编译带 AFL 插桩的 Mesa
echo "【步骤 1】编译插桩 Mesa"
cd "$MESA_DIR"

export CC="$AFL_DIR/afl-gcc"
export CXX="$AFL_DIR/afl-g++"

meson setup build-fuzz \
    -Dbuildtype=debug \
    -Dllvm=enabled \
    -Dgallium-drivers=swrast \
    -Dvulkan-drivers=[] \
    -Dplatforms=x11 \
    -Dprefix="$OUTPUT_DIR/mesa-install"

ninja -C build-fuzz
ninja -C build-fuzz install

echo "Mesa 编译完成"
echo ""

# 2. 准备语料库
echo "【步骤 2】准备语料库"
CORPUS_DIR="$OUTPUT_DIR/shader_corpus"
mkdir -p "$CORPUS_DIR"

# 创建简单的着色器种子
cat > "$CORPUS_DIR/simple.vert" << 'EOF'
#version 330
in vec3 position;
void main() {
    gl_Position = vec4(position, 1.0);
}
EOF

cat > "$CORPUS_DIR/simple.frag" << 'EOF'
#version 330
out vec4 color;
void main() {
    color = vec4(1.0, 0.0, 0.0, 1.0);
}
EOF

echo "语料库准备完成"
echo ""

# 3. 创建 fuzz 目标程序
echo "【步骤 3】创建 fuzz 目标"
cat > "$OUTPUT_DIR/fuzz_shader.c" << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <GL/gl.h>

// 简化示例：实际应使用 Mesa 的编译器 API
int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    if (size == 0) return 0;
    
    // 创建临时着色器
    GLuint shader = glCreateShader(GL_VERTEX_SHADER);
    if (!shader) return 0;
    
    // 编译着色器
    const char *source = (const char *)data;
    glShaderSource(shader, 1, &source, NULL);
    glCompileShader(shader);
    
    // 清理
    glDeleteShader(shader);
    
    return 0;
}
EOF

# 编译 fuzz 目标
"$AFL_DIR/afl-gcc" -o "$OUTPUT_DIR/fuzz_shader" \
    "$OUTPUT_DIR/fuzz_shader.c" \
    -I"$OUTPUT_DIR/mesa-install/include" \
    -L"$OUTPUT_DIR/mesa-install/lib" \
    -lGL

echo "Fuzz 目标编译完成"
echo ""

# 4. 运行 AFL
echo "【步骤 4】运行 AFL"
echo "执行命令:"
echo "  afl-fuzz -i $CORPUS_DIR -o $OUTPUT_DIR/fuzz_out $OUTPUT_DIR/fuzz_shader @@"
echo ""
```

#### 2.3 libFuzzer 与 Mesa 集成

```cpp
// fuzz_glsl_parser.cpp - Mesa GLSL 解析器 Fuzz 目标
// 参考：Mesa 项目 fuzz 测试实践

#include <stdint.h>
#include <stddef.h>
#include <string.h>

// Mesa 头文件
extern "C" {
#include "main/mtypes.h"
#include "compiler/glsl/glsl_parser_extras.h"
#include "compiler/glsl/ir_optimization.h"
}

// Fuzz 目标函数
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    // 限制输入大小
    if (size > 65536) return 0;
    
    // 创建内存上下文
    struct gl_context *ctx = calloc(1, sizeof(*ctx));
    if (!ctx) return 0;
    
    // 初始化解析器
    struct _mesa_glsl_parse_state *state =
        new(ctx) _mesa_glsl_parse_state(ctx, shader_vertex, NULL);
    
    // 复制输入数据（确保以 null 结尾）
    char *source = (char *)malloc(size + 1);
    if (!source) {
        free(ctx);
        return 0;
    }
    memcpy(source, data, size);
    source[size] = '\0';
    
    // 解析着色器
    state->error = false;
    _mesa_glsl_compile_shader(ctx, state, source, false);
    
    // 清理
    free(source);
    delete state;
    free(ctx);
    
    return 0;
}
```

#### 2.4 ROCm 编译器 Fuzz 测试

```bash
#!/bin/bash
# fuzz_rocm_compiler.sh - ROCm 编译器 Fuzz 测试

set -e

ROCM_DIR="${1:-/opt/rocm}"
OUTPUT_DIR="${2:-$HOME/rocm-fuzzing}"
mkdir -p "$OUTPUT_DIR"

echo "=== ROCm 编译器 Fuzz 测试 ==="
echo ""

# 创建 HIP 代码 fuzz 目标
cat > "$OUTPUT_DIR/fuzz_hip.cpp" << 'EOF'
#include <stdint.h>
#include <stddef.h>
#include <string.h>
#include <hip/hip_runtime.h>

extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    if (size == 0 || size > 65536) return 0;
    
    // 写入临时文件
    char tmpfile[] = "/tmp/hip_fuzz_XXXXXX";
    int fd = mkstemp(tmpfile);
    if (fd < 0) return 0;
    
    write(fd, data, size);
    close(fd);
    
    // 尝试编译 HIP 代码
    // 使用 hipcc 编译
    char cmd[1024];
    snprintf(cmd, sizeof(cmd), 
             "hipcc -c %s -o /dev/null 2>/dev/null", 
             tmpfile);
    int ret = system(cmd);
    
    // 清理
    unlink(tmpfile);
    
    return 0;
}
EOF

# 编译（使用 libFuzzer）
clang++ -fsanitize=fuzzer,address \
    -o "$OUTPUT_DIR/fuzz_hip" \
    "$OUTPUT_DIR/fuzz_hip.cpp" \
    -I"$ROCM_DIR/include" \
    -L"$ROCM_DIR/lib" \
    -lhiprtc

echo "Fuzz 目标编译完成"
echo ""
echo "运行命令:"
echo "  $OUTPUT_DIR/fuzz_hip -max_len=65536 corpus/"
```

### 三、案例分析

#### 3.1 Mesa 着色器编译器漏洞案例

**背景**：通过 AFL fuzz Mesa GLSL 编译器发现空指针解引用。

**输入样本**：
```glsl
#version 330
uniform struct {
    vec4 field;
} s;
void main() {
    gl_Position = s.field;
}
```

**崩溃信息**：
```
ASAN:DEADLYSIGNAL
=================================================================
==1234==ERROR: AddressSanitizer: SEGV on unknown address 0x000000000000
==1234==The signal is caused by a READ memory access to address 0x000000000000
    #0 in glsl_type::glsl_type(...)
    #1 in ast_struct_specifier::hir(...)
    #2 in _mesa_ast_to_hir(...)
```

**根因**：结构体类型解析时未正确处理 uniform 块中的匿名结构体。

#### 3.2 ROCm hipcc 编译器崩溃案例

**背景**：通过 libFuzzer 发现 hipcc 在处理特定模板代码时崩溃。

**修复**：在编译器前端添加更严格的语法检查。

### 四、相关链接

- AFL 项目：https://github.com/google/AFL
- libFuzzer 文档：https://llvm.org/docs/LibFuzzer.html
- Mesa Fuzz 测试：https://docs.mesa3d.org/faq.html
- ROCm 文档：https://rocm.docs.amd.com/
- AddressSanitizer：https://github.com/google/sanitizers/wiki/AddressSanitizer

## 今日小结

- AFL 和 libFuzzer 适用于用户空间组件的 fuzz 测试，与 syzkaller 形成互补
- Mesa 着色器编译器、libdrm、ROCm 运行时都是用户空间 fuzz 的重要目标
- 编译时插桩和覆盖率反馈是 AFL/libFuzzer 的核心机制
- 用户空间 fuzz 执行速度快，可以快速发现解析器和编译器漏洞

## 扩展思考

1. 用户空间 fuzz 发现的漏洞如何与内核驱动测试关联？例如，Mesa 的无效输入是否会传递到内核驱动？
2. 如何设计跨用户空间和内核空间的端到端 fuzz 测试？
