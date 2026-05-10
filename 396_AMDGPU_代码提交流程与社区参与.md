# Day 396：AMDGPU 代码提交流程与社区参与

## 学习目标

- 理解 AMDGPU 驱动的上游开发流程和提交规范
- 掌握补丁的格式要求和 review 流程
- 学会使用 git send-email 提交补丁
- 了解 AMDGPU 社区沟通渠道和贡献指南

## 知识详解

### 概念原理

#### AMDGPU 社区开发模型

```
AMDGPU 社区开发架构:

┌────────────────────────────────────────────────────────────────────┐
│                      上游开发流程                                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────────┐      │
│  │ 内核主线     │ ←─ │ drm-misc     │ ←─ │ amd-staging       │      │
│  │ (Linus Torvalds)│    │ (drm-misc-next)│    │ (内部开发分支)     │      │
│  └─────────────┘    └──────────────┘    └──────────────────┘      │
│        ↑                     ↑                    ↑                 │
│        │                     │                    │                 │
│        │      ┌──────────────┴─────────────┐      │                 │
│        │      │      补丁 Review 流程       │      │                 │
│        │      │  ┌─────────────────────┐   │      │                 │
│        │      │  │ Reviewed-by:         │   │      │                 │
│        │      │  │ Acked-by:            │   │      │                 │
│        │      │  │ Tested-by:           │   │      │                 │
│        │      │  │ Signed-off-by:       │   │      │                 │
│        │      │  └─────────────────────┘   │      │                 │
│        │      └────────────────────────────┘      │                 │
│        │                                           │                 │
│  ┌─────┴──────┐    ┌──────────────┐    ┌───────────┴────────┐     │
│  │ 稳定版内核   │    │ 修复补丁      │    │ AMD 内部 CI 测试    │     │
│  │ (stable)    │ ←─ │ (fixes)      │ ←─ │ (硬件验证)          │     │
│  └─────────────┘    └──────────────┘    └────────────────────┘     │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘

角色与职责:

┌─────────────────────────────────────────────────────┐
│  角色               │  职责                            │
├─────────────────────────────────────────────────────┤
│  AMD 内部开发者      │ 开发新功能、修复 bug、内部 CI    │
│  AMD 维护者          │ Review 补丁、管理分支、决策      │
│  (Alex Deucher,      │ 特性合并                        │
│   Christian König)   │                                 │
│  drm-misc 维护者     │ 管理 drm-misc 分支、合并到主线   │
│  社区贡献者          │ 提交补丁、参与 review            │
│  测试人员            │ 报告 bug、测试新特性              │
│  Linus Torvalds      │ 最终合并到主线内核               │
└─────────────────────────────────────────────────────┘

补丁生命周期:

  提交者创建补丁
      │
      ▼
  ┌─────────────────────────────┐
  │ 自测: checkpatch.pl, 编译测试 │
  └─────────────────────────────┘
      │
      ▼
  ┌─────────────────────────────┐
  │ git send-email 发送到邮件列表 │
  │ amd-gfx@lists.freedesktop.org│
  └─────────────────────────────┘
      │
      ▼
  ┌─────────────────────────────┐
  │ Review 阶段 (1-4 周)        │
  │ 维护者/社区成员 Review       │
  │ 可能要求修改 (V2, V3, ...)   │
  └─────────────────────────────┘
      │
      ├── 被拒绝 → 修改后重新提交
      │
      ▼
  ┌─────────────────────────────┐
  │ Alex Deucher 或 Christian   │
  │ König 应用补丁              │
  │ → amd-staging               │
  └─────────────────────────────┘
      │
      ▼
  ┌─────────────────────────────┐
  │ amd-staging → drm-misc-next  │
  │ (drm-misc 维护者)            │
  └─────────────────────────────┘
      │
      ▼
  ┌─────────────────────────────┐
  │ drm-misc-next → linux-next   │
  │ (集成为下一个合并窗口)         │
  └─────────────────────────────┘
      │
      ▼
  ┌─────────────────────────────┐
  │ 合并窗口 → Linus Torvalds   │
  │ → 主线内核                   │
  └─────────────────────────────┘
      │
      ▼
  ┌─────────────────────────────┐
  │ 稳定版内核 (stable)          │
  │ (重要修复向后移植)            │
  └─────────────────────────────┘
```

#### 补丁格式规范

```
AMDGPU 补丁格式:

From: Developer Name <developer@example.com>
Date: Mon, 1 Jan 2024 10:00:00 +0800
Subject: [PATCH v3 1/5] drm/amdgpu: add support for new feature

补丁标题格式:
  [PATCH]                  - 第一版补丁
  [PATCH v2]               - 第二版
  [PATCH v3 1/5]           - 第三版，系列中的第 1/5 个补丁
  [PATCH RESEND]           - 重新发送
  [PATCH RFC]              - 请求评论 (Request for Comments)

补丁正文格式:
  空行
  详细的提交信息
  解释为什么需要这个补丁
  说明是怎么实现的

  空行
  Signed-off-by: Developer Name <developer@example.com>

  空行
  ---

  空行 (diffstat 区域)
  drivers/gpu/drm/amd/amdgpu/amdgpu_device.c | 10 ++++++++++
  drivers/gpu/drm/amd/amdgpu/amdgpu_drv.c    |  3 ++-
  2 files changed, 12 insertions(+), 1 deletion(-)

  diff --git a/drivers/gpu/drm/amd/amdgpu/amdgpu_device.c
  ...

Commit message 规范:

    drm/amdgpu: add RAS error counter sysfs interface

    Add a new sysfs interface under /sys/class/drm/card<N>/device/ras/
    that exposes per-IP block RAS error counters. This allows userspace
    to monitor correctable and uncorrectable error counts at runtime.

    The implementation reads CE/UE counts from the respective IP block's
    RAS register range and formats them as:
      - <ip>_ce_count:  correctable error count
      - <ip>_ue_count:  uncorrectable error count

    v2: Address review feedback from Alex:
        - Use sysfs_emit instead of sprintf
        - Add proper locking around register access
    
    v3: Fix build warning on 32-bit platforms (Christian)

    Signed-off-by: Zhang San <zhangsan@example.com>
    Reviewed-by: Alex Deucher <alexander.deucher@amd.com>
    Reviewed-by: Christian König <christian.koenig@amd.com>

Review Tags:

  Signed-off-by:  提交者认证 (Developer Certificate of Origin)
  Reviewed-by:    Reviewer 确认代码无误
  Acked-by:       维护者同意，但不一定详细检查过代码
  Tested-by:      测试者验证过补丁功能
  Reported-by:    报告问题的用户
  Suggested-by:   提出建议的人
  Fixes:          格式: Fixes: <12-char-sha> ("<commit title>")
  Cc:             抄送相关人员
  Closes:         格式: Closes: <URL-to-bug-report>
```

#### checkpatch.pl 检查

```bash
#!/bin/bash
# 补丁检查流程

echo "=============================================="
echo "  AMDGPU 补丁检查工具"
echo "=============================================="
echo ""

# 配置
KERNEL_TREE="/home/user/linux"
PATCH_DIR="/home/user/patches"

# 1. 运行 checkpatch.pl
check_patch() {
    local patch_file="$1"
    
    if [ ! -f "${patch_file}" ]; then
        echo "❌ 补丁文件不存在: ${patch_file}"
        return 1
    fi
    
    echo "[1/5] 运行 checkpatch.pl..."
    echo ""
    
    ${KERNEL_TREE}/scripts/checkpatch.pl \
        --strict \
        --show-types \
        --subjective \
        "${patch_file}" 2>&1 | tee "${patch_file}.checkpatch"
    
    local errors=$(grep -c "^ERROR:" "${patch_file}.checkpatch" 2>/dev/null || echo 0)
    local warnings=$(grep -c "^WARNING:" "${patch_file}.checkpatch" 2>/dev/null || echo 0)
    
    echo ""
    echo "  结果:"
    echo "    错误: ${errors}"
    echo "    警告: ${warnings}"
    
    if [ "${errors}" -gt 0 ]; then
        echo "  ❌ 需要修复错误"
        return 1
    fi
    
    echo "  ✅ checkpatch 通过"
}

# 2. 签名检查
check_signoff() {
    local patch_file="$1"
    
    echo "[2/5] 检查 Signed-off-by..."
    
    local author=$(grep "^From:" "${patch_file}" | sed 's/From: //')
    local signoff=$(grep "^Signed-off-by:" "${patch_file}" | sed 's/Signed-off-by: //')
    
    echo "  Author: ${author}"
    echo "  Signed-off-by: ${signoff}"
    
    if [ -z "${signoff}" ]; then
        echo "  ❌ 缺少 Signed-off-by"
        return 1
    fi
    
    echo "  ✅ Signed-off-by 存在"
}

# 3. 编译测试
build_test() {
    local patch_file="$1"
    
    echo "[3/5] 编译测试..."
    echo ""
    
    cd "${KERNEL_TREE}"
    
    # 保存原始 .config
    if [ -f ".config" ]; then
        cp ".config" ".config.backup"
    fi
    
    # 确保 amdgpu 启用
    make amd64_defconfig 2>/dev/null
    ./scripts/config --module CONFIG_DRM_AMDGPU
    ./scripts/config --enable CONFIG_DRM_AMD_DC
    make olddefconfig 2>/dev/null
    
    # 应用补丁
    git am "${patch_file}" 2>/dev/null
    if [ $? -ne 0 ]; then
        echo "  ❌ 补丁应用失败"
        git am --abort 2>/dev/null
        return 1
    fi
    
    # 编译 amdgpu 驱动
    make -j$(nproc) M=drivers/gpu/drm/amd/amdgpu 2>&1 | tail -20
    
    local built=$?
    
    # 恢复
    git am --abort 2>/dev/null
    
    if [ "${built}" -eq 0 ]; then
        echo ""
        echo "  ✅ 编译成功"
    else
        echo ""
        echo "  ❌ 编译失败"
        return 1
    fi
}

# 4. 内核文档检查
check_kernel_doc() {
    local patch_file="$1"
    
    echo "[4/5] 内核文档检查..."
    
    cd "${KERNEL_TREE}"
    git am "${patch_file}" 2>/dev/null
    
    # 运行 kernel-doc 检查
    find drivers/gpu/drm/amd/amdgpu/ -name "*.c" -o -name "*.h" | \
    while read file; do
        local issues=$(${KERNEL_TREE}/scripts/kernel-doc -v "${file}" 2>&1 | \
                      grep -i "warning\|error" || true)
        if [ -n "${issues}" ]; then
            echo "  ${file}:"
            echo "${issues}" | while read line; do
                echo "    ⚠️  ${line}"
            done
        fi
    done
    
    git am --abort 2>/dev/null
    
    echo "  ✅ kernel-doc 检查完成"
}

# 5. 完整补丁检查
full_check() {
    local patch_file="$1"
    
    echo "[5/5] 完整补丁检查..."
    echo ""
    
    # 补丁基本信息
    echo "  补丁信息:"
    echo "    文件: ${patch_file}"
    echo "    大小: $(wc -c < "${patch_file}") bytes"
    echo "    行数: $(wc -l < "${patch_file}") lines"
    echo ""
    
    # 检查补丁格式
    local subject=$(grep "^Subject:" "${patch_file}" | head -1)
    echo "  主题: ${subject}"
    echo ""
    
    # 统计文件变更
    local files=$(grep "^--- a/" "${patch_file}" | sed 's|--- a/||' | sort)
    echo "  变更文件:"
    echo "${files}" | while read file; do
        local lines=$(grep -c "^+.*" "${patch_file}" 2>/dev/null || echo 0)
        echo "    + ${file}"
    done
    
    echo ""
    check_patch "${patch_file}" && \
    check_signoff "${patch_file}" && \
    echo ""
    echo "  ✅ 完整检查通过"
}

# 主流程
case "$1" in
    check)
        if [ -z "$2" ]; then
            echo "用法: $0 check <patch_file>"
            exit 1
        fi
        check_patch "$2"
        ;;
    signoff)
        if [ -z "$2" ]; then
            echo "用法: $0 signoff <patch_file>"
            exit 1
        fi
        check_signoff "$2"
        ;;
    build)
        if [ -z "$2" ]; then
            echo "用法: $0 build <patch_file>"
            exit 1
        fi
        build_test "$2"
        ;;
    doc)
        if [ -z "$2" ]; then
            echo "用法: $0 doc <patch_file>"
            exit 1
        fi
        check_kernel_doc "$2"
        ;;
    full)
        if [ -z "$2" ]; then
            echo "用法: $0 full <patch_file>"
            exit 1
        fi
        full_check "$2"
        ;;
    *)
        echo "用法: $0 {check|signoff|build|doc|full} <patch_file>"
        echo ""
        echo "  check   - 运行 checkpatch.pl"
        echo "  signoff - 检查 Signed-off-by"
        echo "  build   - 编译测试"
        echo "  doc     - 文档检查"
        echo "  full    - 完整检查"
        ;;
esac
```

### 实践操作

#### 完整补丁提交流程

```bash
#!/bin/bash
# AMDGPU 补丁提交流程

set -e

echo "=============================================="
echo "  AMDGPU 补丁提交流程"
echo "=============================================="
echo ""

# 配置
KERNEL_DIR="${HOME}/linux"
BRANCH="amd-staging"
PATCH_DIR="${HOME}/patches"
PATCH_PREFIX="amdgpu"

# 阶段 1: 准备开发环境
setup_env() {
    echo "[1/8] 准备开发环境..."
    
    # 克隆内核源码
    if [ ! -d "${KERNEL_DIR}" ]; then
        echo "  克隆内核源码树..."
        git clone \
            https://gitlab.freedesktop.org/drm/amd.git \
            "${KERNEL_DIR}" \
            --branch "${BRANCH}"
    fi
    
    cd "${KERNEL_DIR}"
    
    # 确保分支最新
    echo "  更新分支..."
    git checkout "${BRANCH}"
    git pull origin "${BRANCH}"
    
    # 创建补丁目录
    mkdir -p "${PATCH_DIR}"
    
    # 配置 git send-email
    if ! git config --get sendemail.smtpserver &>/dev/null; then
        echo "  配置 git send-email..."
        git config sendemail.smtpserver smtp.gmail.com
        git config sendemail.smtpserverport 587
        git config sendemail.smtpencryption tls
        git config sendemail.smtpuser "your-email@gmail.com"
        # 使用 App Password 或 GPG 签名
    fi
    
    echo "  ✅ 开发环境已准备好"
}

# 阶段 2: 创建开发分支
create_branch() {
    echo "[2/8] 创建开发分支..."
    
    cd "${KERNEL_DIR}"
    
    local branch_name="dev/$(date +%Y%m%d)-${PATCH_PREFIX}-$(echo $RANDOM | md5sum | head -c 8)"
    
    git checkout -b "${branch_name}" "${BRANCH}"
    
    echo "  分支: ${branch_name}"
    echo "  基础: ${BRANCH}"
    
    echo "${branch_name}" > "${PATCH_DIR}/.current_branch"
    
    echo "  ✅ 开发分支已创建"
}

# 阶段 3: 开发并提交
make_changes() {
    echo "[3/8] 进行代码修改..."
    
    cd "${KERNEL_DIR}"
    
    echo "  请完成代码修改后执行以下 git 命令:"
    echo ""
    echo "    git add <files>"
    echo "    git commit -s"
    echo ""
    echo "  然后运行: $0 format"
    echo ""
}

# 阶段 4: 生成补丁
format_patches() {
    echo "[4/8] 生成补丁..."
    
    cd "${KERNEL_DIR}"
    
    local branch_name=$(cat "${PATCH_DIR}/.current_branch" 2>/dev/null || echo "unknown")
    local base_commit=$(git merge-base HEAD "${BRANCH}")
    
    # 检查是否有未提交的修改
    if [ -n "$(git status --porcelain)" ]; then
        echo "  ⚠️  有未提交的修改，请先提交"
        git status --short
        return 1
    fi
    
    # 获取从 base 到 HEAD 的提交数
    local commit_count=$(git rev-list "${base_commit}"..HEAD --count 2>/dev/null || echo 1)
    
    echo "  基准提交: $(git rev-parse --short ${base_commit})"
    echo "  提交数量: ${commit_count}"
    echo ""
    
    # 生成补丁文件
    if [ "${commit_count}" -eq 1 ]; then
        # 单补丁
        git format-patch \
            --stdout \
            --signoff \
            "${base_commit}" \
            > "${PATCH_DIR}/0001-${PATCH_PREFIX}-patch.patch"
        echo "  生成的补丁: ${PATCH_DIR}/0001-${PATCH_PREFIX}-patch.patch"
    else
        # 补丁系列
        git format-patch \
            --cover-letter \
            --signoff \
            -o "${PATCH_DIR}" \
            "${base_commit}"
        echo "  生成的补丁系列:"
        ls -la "${PATCH_DIR}"/0*.patch
    fi
    
    echo "  ✅ 补丁已生成"
}

# 阶段 5: 补丁检查
check_patches() {
    echo "[5/8] 补丁检查..."
    
    cd "${KERNEL_DIR}"
    
    for patch_file in "${PATCH_DIR}"/*.patch; do
        if [ -f "${patch_file}" ]; then
            echo "  检查: $(basename ${patch_file})"
            echo ""
            
            # 运行 checkpatch
            ${KERNEL_DIR}/scripts/checkpatch.pl \
                --strict "${patch_file}" 2>&1 | \
                grep -E "^ERROR|^WARNING|total" || true
            
            echo ""
        fi
    done
    
    echo "  ✅ 补丁检查完成"
}

# 阶段 6: 发送到邮件列表
send_patches() {
    echo "[6/8] 发送补丁..."
    
    cd "${KERNEL_DIR}"
    
    local patch_files=()
    for f in "${PATCH_DIR}"/*.patch; do
        patch_files+=("$f")
    done
    
    if [ ${#patch_files[@]} -eq 0 ]; then
        echo "  ❌ 没有找到补丁文件"
        echo "  请先运行: $0 format"
        return 1
    fi
    
    echo "  将发送 ${#patch_files[@]} 个补丁"
    echo "  目标: amd-gfx@lists.freedesktop.org"
    echo ""
    
    # 确认发送
    echo "  补丁列表:"
    for f in "${patch_files[@]}"; do
        echo "    - $(basename $f): $(head -1 $f)"
    done
    echo ""
    
    read -p "  确认发送? (y/N) " confirm
    
    if [ "${confirm}" = "y" ] || [ "${confirm}" = "Y" ]; then
        # 如果有多补丁，需要 cover letter
        if [ ${#patch_files[@]} -gt 1 ]; then
            local cover_letter=$(ls "${PATCH_DIR}"/0000-cover-letter.patch 2>/dev/null || echo "")
            if [ -n "${cover_letter}" ]; then
                # 有 cover letter
                git send-email \
                    --to amd-gfx@lists.freedesktop.org \
                    --cc dri-devel@lists.freedesktop.org \
                    --cc "Alex Deucher <alexander.deucher@amd.com>" \
                    --cc "Christian König <christian.koenig@amd.com>" \
                    --cover-letter \
                    "${patch_files[@]}"
            else
                # 无 cover letter
                git send-email \
                    --to amd-gfx@lists.freedesktop.org \
                    --cc dri-devel@lists.freedesktop.org \
                    "${patch_files[@]}"
            fi
        else
            # 单补丁
            git send-email \
                --to amd-gfx@lists.freedesktop.org \
                --cc dri-devel@lists.freedesktop.org \
                "${patch_files[0]}"
        fi
        
        if [ $? -eq 0 ]; then
            echo "  ✅ 补丁已发送"
        else
            echo "  ❌ 发送失败"
        fi
    else
        echo "  ⚠️  取消发送"
    fi
}

# 阶段 7: 版本迭代
revision() {
    echo "[7/8] 补丁版本迭代..."
    
    cd "${KERNEL_DIR}"
    
    # 获取当前补丁版本
    local branch_name=$(cat "${PATCH_DIR}/.current_branch" 2>/dev/null || echo "unknown")
    
    # 获取截止到当前为止的提交
    local base_commit=$(git merge-base HEAD "${BRANCH}")
    
    # 检查是否有 review 反馈需要 rebase
    echo "  当前分支: ${branch_name}"
    echo "  基准: $(git rev-parse --short ${base_commit})"
    echo ""
    echo "  操作选项:"
    echo "  1. 修复提交 (修复当前提交中的问题)"
    echo "  2. Rebase 到最新 ${BRANCH}"
    echo "  3. 更新版本号"
    echo ""
    
    # 示例: 修复并更新版本
    # git commit --amend -s
    # git rebase ${BRANCH}
    
    echo "  Review 反馈处理步骤:"
    echo "  1. 修复 review 指出的问题"
    echo "  2. git add <files>"
    echo "  3. git commit --amend -s"
    echo "  4. 更新版本号在 subject 中: [PATCH v2]"
    echo "  5. 在新版本补丁说明中记录变更"
    echo ""
}

# 阶段 8: 后续处理
cleanup() {
    echo "[8/8] 清理..."
    
    cd "${KERNEL_DIR}"
    
    local branch_name=$(cat "${PATCH_DIR}/.current_branch" 2>/dev/null || echo "unknown")
    
    echo "  可选清理操作:"
    echo "  1. 保留分支: git checkout ${BRANCH}"
    echo "  2. 删除分支: git branch -D ${branch_name}"
    echo "  3. 保留补丁: rm ${PATCH_DIR}/*.patch"
    echo ""
    
    # 清理补丁目录
    # rm -f "${PATCH_DIR}"/*.patch
    # rm -f "${PATCH_DIR}/.current_branch"
}

# 主流程
case "$1" in
    setup)
        setup_env
        ;;
    start)
        create_branch
        make_changes
        ;;
    format)
        format_patches
        ;;
    check)
        check_patches
        ;;
    send)
        send_patches
        ;;
    revise)
        revision
        ;;
    cleanup)
        cleanup
        ;;
    all)
        setup_env
        create_branch
        # 提示用户自行修改
        echo "请完成代码修改后运行:"
        echo "  $0 format"
        echo "  $0 check"
        echo "  $0 send"
        ;;
    *)
        echo "用法: $0 {setup|start|format|check|send|revise|cleanup|all}"
        echo ""
        echo "  setup   - 初始化开发环境"
        echo "  start   - 创建开发分支"
        echo "  format  - 生成补丁文件"
        echo "  check   - 运行补丁检查"
        echo "  send    - 发送补丁到邮件列表"
        echo "  revise  - 版本迭代"
        echo "  cleanup - 清理临时文件"
        ;;
esac
```

#### DCO (Developer Certificate of Origin)

```bash
# Developer Certificate of Origin (DCO)
# 版本 1.1

# 通过添加 Signed-off-by 标签，开发者认证:
# (a) 该贡献完全或部分由我创建，我有权提交到开源项目；或
# (b) 该贡献基于之前已知的开源作品，且原作品有适当的许可证；
# (c) 该贡献直接或间接由第三方提供
# (d) 我理解并同意该项目和该贡献是公开的

# DCO 配置
git config user.name "Zhang San"
git config user.email "zhangsan@example.com"

# 自动添加 Signed-off-by
# 使用 -s 或 --signoff 选项
git commit -s -m "drm/amdgpu: fix fence timeout handling"

# 也可以全局设置
git config format.signoff true

# 验证 DCO
git log --format="%H %ae %s" --signoff
```

#### Review 反馈处理

```bash
#!/bin/bash
# Review 反馈处理工具

echo "=============================================="
echo "  Review 反馈处理"
echo "=============================================="
echo ""

KERNEL_DIR="${HOME}/linux"

# 1. 收集 Review 反馈
collect_review() {
    echo "[1/4] 收集 Review 反馈..."
    
    # 从邮件列表获取反馈
    local thread_id="$1"
    
    echo "  从邮件列表获取反馈..."
    echo "  人工处理: 阅读 amd-gfx 邮件列表中的回复"
    echo ""
    echo "  常见 Review 标签:"
    echo "  Reviewed-by: 代码审查通过"
    echo "  Acked-by:    同意合并"
    echo "  Tested-by:   测试验证通过"
    echo "  Nacked-by:   反对 (需要解决)"
    echo ""
    
    # 创建 review 反馈记录文件
    cat > review_feedback.md << 'EOF'
# Review 反馈记录

## 补丁信息
- 提交版本: v3
- 日期:

## Review 反馈列表
| Reviewer | 标签 | 意见 | 状态 |
|----------|------|------|------|
| Alex Deucher | Reviewed-by | 代码质量 OK | ✅ 已处理 |
| Christian König | - | 建议使用 atomic 操作 | ✅ 已修复 |

## 修改记录
1. 问题: 锁使用不当
   修复: 改用 spin_lock_irqsave
   
EOF
    
    echo "  ✅ review_feedback.md 已创建"
}

# 2. 修复问题并更新
apply_fixes() {
    echo "[2/4] 应用 Review 修复..."
    
    cd "${KERNEL_DIR}"
    
    # 检查当前是否在开发分支
    local current_branch=$(git rev-parse --abbrev-ref HEAD)
    echo "  当前分支: ${current_branch}"
    
    # 使用 fixup! 或 amend
    echo "  修复并更新提交:"
    echo ""
    echo "  # 方法 1: 修复后 amend"
    echo "  git add <fixed-files>"
    echo "  git commit --amend -s"
    echo ""
    echo "  # 方法 2: 使用 fixup (需 rebase)"
    echo "  git add <fixed-files>"
    echo "  git commit --fixup HEAD"
    echo "  git rebase -i --autosquash HEAD~2"
    echo ""
}

# 3. 更新补丁版本
bump_version() {
    echo "[3/4] 更新补丁版本..."
    
    cd "${KERNEL_DIR}"
    
    local base_commit=$(git merge-base HEAD "amd-staging")
    local commit_count=$(git rev-list "${base_commit}"..HEAD --count 2>/dev/null || echo 1)
    
    # 获取当前版本号
    local current_version=$(git log HEAD~0..HEAD --format="%s" | \
                           grep -oP "v\d+" || echo "v1")
    local next_version="v$(( ${current_version#v} + 1 ))"
    
    echo "  当前版本: ${current_version}"
    echo "  下一版本: ${next_version}"
    echo ""
    echo "  使用 git rebase 更新提交信息中的版本号"
    echo "  Subject: [PATCH ${next_version}]"
    echo ""
}

# 4. 重新发送
resend() {
    echo "[4/4] 重新发送..."
    
    cd "${KERNEL_DIR}"
    
    # 重新生成补丁
    local base_commit=$(git merge-base HEAD "amd-staging")
    git format-patch \
        --stdout \
        --signoff \
        -o "${HOME}/patches_v2" \
        "${base_commit}"
    
    echo "  补丁已重新生成"
    echo "  请检查后使用 git send-email 发送"
}

case "$1" in
    collect)
        collect_review "$2"
        ;;
    fix)
        apply_fixes
        ;;
    bump)
        bump_version
        ;;
    resend)
        resend
        ;;
    *)
        echo "用法: $0 {collect|fix|bump|resend}"
        ;;
esac
```

### 案例分析

#### 案例一：新提交者初次提交补丁

**背景：**
开发者 Xiao Wang 发现 amdgpu 驱动中一个 `drm_gem_object_put` 调用缺失导致的引用计数泄漏，准备提交修复补丁。这是他的首个内核补丁。

**操作过程：**

```bash
# 1. 环境准备
git clone https://gitlab.freedesktop.org/drm/amd.git linux-amd
cd linux-amd
git checkout amd-staging

# 2. 创建分支
git checkout -b fix-gem-ref-leak

# 3. 修改代码
# 编辑 amdgpu_cs.c 添加缺少的 drm_gem_object_put()

# 4. 提交
git add drivers/gpu/drm/amd/amdgpu/amdgpu_cs.c
git commit -s -m "drm/amdgpu: fix GEM object reference leak in CS ioctl

    ...详细说明... "

# 5. 运行 checkpatch
./scripts/checkpatch.pl --strict \
    < git format-patch --stdout HEAD~1

# 6. 编译测试
make -j$(nproc) M=drivers/gpu/drm/amd/amdgpu

# 7. 发送补丁
git send-email \
    --to amd-gfx@lists.freedesktop.org \
    --cc dri-devel@lists.freedesktop.org \
    --cc "Alex Deucher <alexander.deucher@amd.com>" \
    --cc "Christian König <christian.koenig@amd.com>" \
    --suppress-cc=self \
    HEAD~1
```

**处理过程：**
- Day 1: 发送 v1 补丁
- Day 3: Christian König 回复，建议使用不同的修复方式
- Day 5: 提交 v2 补丁，根据反馈修改
- Day 8: Alex Deucher 回复 Reviewed-by
- Day 10: 补丁被合并到 amd-staging 分支

**经验总结：**
```
首次提交注意事项:

1. 从小补丁开始:
   - 修复小 bug 比新功能更容易被接受
   - 单补丁（非系列）更简单

2. 认真对待 review:
   - 及时回复 review 意见
   - 感谢 reviewer 的时间
   - 不要气馁，多次迭代很正常

3. 遵循社区规范:
   - 英文提交信息
   - 正确的 Subject 格式
   - 详细的提交说明

4. 保持耐心:
   - 首次补丁可能需要 2-4 周
   - 学习社区的文化和流程
```

#### 案例二：复杂特性提交的多轮 Review

**背景：**
需要为 amdgpu 添加一个新的 RAS 事件通知机制，涉及 5 个补丁系列，跨越 3 个模块。

**补丁系列结构：**

```
[PATCH v1 0/5] drm/amdgpu: add RAS event notification mechanism

[PATCH v1 1/5] drm/amdgpu: add RAS event structure definitions
[PATCH v1 2/5] drm/amdgpu: implement RAS event ring buffer
[PATCH v1 3/5] drm/amdgpu: add sysfs interface for RAS events
[PATCH v1 4/5] drm/amdgpu: add RAS event polling support
[PATCH v1 5/5] drm/amdgpu: add RAS event test module
```

**Review 过程：**

```
Round 1 (v1):
  - Alex: "事件结构定义需要包含时间戳"
  - Christian: "ring buffer 实现应该使用 lockless 方法"
  - Lijo: "sysfs 接口应该遵循 existing pattern"

Round 2 (v2):
  - Christian: "lockless 实现正确，但需要 memory barrier 文档"
  - Alex: "sysfs 接口文档需要补充"

Round 3 (v3):
  - Alex: Reviewed-by 1/5, 2/5, 5/5
  - Christian: Reviewed-by 3/5, 4/5
  - 所有 Reviewed-by 收集完毕，合并到 amd-staging
```

**经验总结：**
```
复杂特性提交策略:

1. 提前发送 RFC:
   - 在实现之前征求架构意见
   - 避免实现完成后才发现方向错误

2. 保持补丁系列合理拆分:
   - 每个补丁只做一件事
   - 系列内补丁依赖关系清晰

3. 维护变更日志:
   - 每个版本在 --- 后记录变更
   - 方便 reviewer 跟踪修改

4. 主动询问:
   - 如果 review 沉默超过 2 周，礼貌性追问
   - 在 #dri-devel IRC 或邮件列表询问
```

### 相关链接

- [AMDGPU 开发指南](https://docs.kernel.org/gpu/amdgpu/index.html)
- [AMDGPU 邮件列表 amd-gfx](https://lists.freedesktop.org/mailman/listinfo/amd-gfx)
- [内核补丁提交指南](https://www.kernel.org/doc/html/latest/process/submitting-patches.html)
- [内核开发者 DCO](https://developercertificate.org/)
- [DRM 维护者分支](https://dri.freedesktop.org/wiki/DRM-Maintainer-Branches/)
- [Linux 内核编码规范](https://www.kernel.org/doc/html/latest/process/coding-style.html)

### 今日小结

AMDGPU 社区贡献遵循标准的 Linux 内核上游开发流程：

1. **开发模型**: amd-staging → drm-misc-next → linux-next → Linus 主线
2. **补丁规范**: [PATCH vN] drm/amdgpu: 前缀，详细提交说明
3. **Review 标签**: Signed-off-by, Reviewed-by, Acked-by, Tested-by
4. **检查工具**: checkpatch.pl, 编译测试, kernel-doc
5. **发送方式**: git send-email 发送到 amd-gfx@lists.freedesktop.org
6. **迭代过程**: v1 → v2 → v3 ...，处理 review 反馈
7. **DCO 认证**: 每次提交必须包含 Signed-off-by

### 扩展思考

1. 如何自动化补丁检查流程？可以设计 CI pipeline 自动运行 checkpatch 和编译测试吗？
2. AMD 内部维护的 amd-staging 分支和公共 amd-staging 分支有何不同？
3. 社区驱动的开发和 AMD 内部开发如何协调？如何避免重复工作？
4. 作为新贡献者，如何选择第一个要贡献的补丁方向？
