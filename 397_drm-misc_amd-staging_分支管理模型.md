# Day 397：drm-misc / amd-staging 分支管理模型

## 学习目标

- 理解 drm-misc 和 amd-staging 分支的层次结构
- 掌握分支流转规则和合并策略
- 学会跟踪不同分支的补丁状态
- 理解内核发布周期与分支的关系

## 知识详解

### 概念原理

#### DRM 子系统分支层次

```
DRM 子系统分支架构:

┌─────────────────────────────────────────────────────────────────────────┐
│                       Linux 主线内核 (Linus Torvalds)                    │
│                                                                         │
│  tags: v6.6, v6.7-rc1, v6.7-rc2, ...                                   │
│  接收: 合并窗口期间从各个子系统拉取                                     │
└────────────────────────┬────────────────────────────────────────────────┘
                         │
                         │ 合并窗口 (2 周)
                         │
    ┌────────────────────┼────────────────────────────┐
    │                    │                            │
    ▼                    ▼                            ▼
┌──────────┐    ┌──────────────┐    ┌──────────────────┐
│ drm-misc  │    │ drm-intel     │    │ drm-msm          │
│ (DRM 核心 │    │ (Intel 专用)  │    │ (Qualcomm)       │
│  + 驱动)  │    └──────────────┘    └──────────────────┘
└──────────┘
    │
    │ drm-misc 子分支:
    │
    ├── drm-misc-next       ← 新功能 (feature)
    │   └── 合并到 linux-next
    │       └── 下一个合并窗口进入主线
    │
    ├── drm-misc-next-fixes ← 下一周期的修复
    │   └── 合并到 drm-misc-next
    │
    ├── drm-misc-fixes      ← 当前内核版本的修复
    │   └── 直接合并到主线 (rc 周期中)
    │
    └── drm-misc-fixes-stable ← 稳定版向后移植
        └── 发送到 stable 内核

AMDGPU 特定的分支路径:

    amd-staging (AMD 内部开发分支)
    │
    ├── 新功能: → drm-misc-next → linux-next → 主线
    ├── 修复: → drm-misc-fixes → 主线 (rc 周期)
    └── 紧急修复: → 直接到主线 (drm-misc 维护者转发)
```

#### 分支流转规则

```
补丁流转规则:

┌──────────────────────────────────────────────────────────────────┐
│  新功能补丁流转路径:                                              │
│                                                                  │
│  ┌───────────┐    ┌───────────────┐    ┌──────────┐            │
│  │ 开发者提交 │ → │ AMD 内部审查   │ → │ amd-staging│            │
│  │ (邮件列表) │    │ (+ 社区 Review)│    │ (合并测试) │            │
│  └───────────┘    └───────────────┘    └─────┬────┘            │
│                                              │                  │
│                                              ▼                  │
│  ┌───────────┐    ┌───────────────┐    ┌──────────┐            │
│  │ Linus 主线 │ ← │ linux-next    │ ← │ drm-misc- │            │
│  │ (rc-1)    │    │ (集成测试)     │    │ next     │            │
│  └───────────┘    └───────────────┘    └──────────┘            │
│                                                                  │
│  修复补丁流转路径:                                                │
│                                                                  │
│  ┌───────────┐    ┌──────────┐    ┌──────────────┐             │
│  │ 开发者提交 │ → │ amd-staging→ │ drm-misc-fixes │             │
│  │ (邮件列表) │    │ (验证修复) │    │ (合并主线 rc) │             │
│  └───────────┘    └──────────┘    └──────┬───────┘             │
│                                         │                       │
│                                         ▼                       │
│                                    ┌──────────┐                │
│                                    │ Linus 主线│                │
│                                    │ (rc-N)   │                │
│                                    └──────────┘                │
│                                                                  │
│  紧急修复流转路径:                                                │
│                                                                  │
│  ┌───────────┐    ┌───────────┐    ┌──────────────┐           │
│  │ 紧急 bug   │ → │ amd-staging│ → │ drm-misc      │           │
│  │ 发现/报告  │    │ (hotfix)   │    │ (紧急 PR)     │           │
│  └───────────┘    └───────────┘    └──────┬───────┘           │
│                                         │                       │
│                                         ▼                       │
│                                    ┌──────────┐                │
│                                    │ Linus 主线│                │
│                                    │ (immediate)│               │
│                                    └──────────┘                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

各分支的合并窗口:

  分支               │ 合并到           │ 时机
  ──────────────────┼─────────────────┼────────────────
  drm-misc-next     │ linux-next      │ 每个 linux-next 发布周期
  linux-next        │ Linus 主线       │ 合并窗口 (MW, 2 周)
  drm-misc-fixes    │ Linus 主线       │ rc1-rc7 (每个 rc 周期)
  drm-misc-fixes-st│ stable 树        │ 需要时 (重要修复)
  amd-staging       │ drm-misc-next   │ 每 1-2 周批量推送
                    │ drm-misc-fixes   │ 修复确认后
```

#### Branch Maintainer 结构

```
DRM 子系统的维护者角色:

┌─────────────────────────────────────────────────────────────────┐
│  子系统              │  维护者                                  │
├─────────────────────────────────────────────────────────────────┤
│  DRM 核心             │  David Airlie, Daniel Vetter            │
│  drm-misc 分支         │  Maarten Lankhorst, Maxime Ripard,     │
│                       │  Thomas Zimmermann, David Airlie        │
│  AMDGPU (amd-staging) │  Alex Deucher, Christian König          │
│  Intel (drm-intel)    │  Jani Nikula, Rodrigo Vivi, Joonas      │
│                       │  Lahtinen                               │
│  Nouveau              │  Karol Herbst, Lyude Paul, Danilo Krumm │
│  linux-next           │  Stephen Rothwell                       │
│  Linus 主线            │  Linus Torvalds                        │
└─────────────────────────────────────────────────────────────────┘

drm-misc 维护者工作流程:

  1. 从 amd-staging 拉取补丁
  2. 检查补丁质量 (Reviewed-by, CI 结果)
  3. 合并到相应的 drm-misc 分支
  4. 向 linux-next 发送 pull request
  5. 在合并窗口向 Linus 发送 pull request

AMD 内部工作流程:

  1. 内部 CI 测试 (数百块 GPU)
  2. 内部 code review
  3. 合并到 AMD 内部的 amd-staging
  4. 同步到公共 amd-staging (gitlab)
  5. 发送到 drm-misc (由 Alex/Christian 操作)
```

#### 内核发布周期

```
Linux 内核发布周期:

                6 月                9 月                12 月
                │                   │                    │
                ▼                   ▼                    ▼
         ┌────────────┐    ┌────────────┐    ┌────────────┐
         │ v6.9        │    │ v6.10       │    │ v6.11       │
         │ 合并窗口     │    │ 合并窗口     │    │ 合并窗口     │
         └──────┬─────┘    └──────┬─────┘    └──────┬─────┘
                │                 │                 │
                ▼                 ▼                 ▼
         ┌────────────┐    ┌────────────┐    ┌────────────┐
         │ rc1         │    │ rc1         │    │ rc1         │
         │ rc2 (修复)  │    │ rc2         │    │ rc2         │
         │ rc3         │    │ rc3         │    │ rc3         │
         │ ...         │    │ ...         │    │ ...         │
         │ rc7         │    │ rc7         │    │ rc7         │
         └──────┬─────┘    └──────┬─────┘    └──────┬─────┘
                │                 │                 │
                ▼                 ▼                 ▼
         ┌────────────┐    ┌────────────┐    ┌────────────┐
         │ v6.9 正式版│    │ v6.10 正式版│    │ v6.11 正式版│
         └────────────┘    └────────────┘    └────────────┘

AMDGPU 驱动在内核周期中的活动:

  ══════════════════════════════════════════════════════════
  阶段      │ 活动                                        │
  ══════════════════════════════════════════════════════════
  合并窗口前 │ 累积新功能到 amd-staging                    │
            │ 准备向 drm-misc-next 的批量推送              │
  ──────────┼──────────────────────────────────────────────
  合并窗口   │ 向 Linus 发送 drm-misc-next pull request    │
            │ 新功能进入主线                               │
  ──────────┼──────────────────────────────────────────────
  rc 周期    │ 修复补丁通过 drm-misc-fixes 推送到主线       │
            │ 紧急修复直接发送                             │
  ──────────┼──────────────────────────────────────────────
  发布后     │ 开始下一个周期的开发                        │
            │ 向后移植修复到 stable                        │
  ══════════════════════════════════════════════════════════
```

#### Tag 和 Pull Request 机制

```
drm-misc Pull Request 格式:

From: Maarten Lankhorst <maarten.lankhorst@linux.intel.com>
To: Dave Airlie <airlied@redhat.com>,
    Daniel Vetter <daniel.vetter@ffwll.ch>
Subject: [GIT PULL] drm-misc-next for v6.11

Hi Dave, Daniel,

The following changes since commit abc123...

drm-misc-next-2024-06-01:
  https://gitlab.freedesktop.org/drm/misc/kernel.git
  tag: drm-misc-next-2024-06-01

Changes:
- New panel driver for XYZ
- AMDGPU: RAS event notification support
- i915: Fix display flicker issue
- Core: Add DRM format modifier helper

are available in the Git repository at:

  https://gitlab.freedesktop.org/drm/misc/kernel.git
  drm-misc-next-2024-06-01

for you to fetch changes up to def456...

----------------------------------------------------------------
Alex Deucher (3):
      drm/amdgpu: add RAS event structure definitions
      drm/amdgpu: implement RAS event ring buffer
      drm/amdgpu: add sysfs interface for RAS events

Christian König (1):
      drm/amdgpu: fix fence handling in CS path

Zhang San (1):
      drm/amdgpu: add RAS event test module
----------------------------------------------------------------

...full diffstat...

Tag 命名规范:

  drm-misc-next-YYYY-MM-DD     ← 新功能 pull request
  drm-misc-fixes-YYYY-MM-DD    ← 修复 pull request
  amd-staging-YYYY-MM-DD       ← AMD 内部同步标签

内核树中的标签位置:

  主线内核:
  v6.10
  v6.10-rc1
  v6.10-rc2
  ...
  v6.10-rc7
  v6.11-rc1

  子系统分支 (drm-misc):
  drm-misc-next-2024-05-01
  drm-misc-next-2024-05-15
  drm-misc-next-2024-06-01
  drm-misc-fixes-2024-05-01
```

### 实践操作

#### 分支管理工具

```bash
#!/bin/bash
# drm-misc / amd-staging 分支管理工具

set -e

echo "=============================================="
echo "  DRM 分支管理工具"
echo "=============================================="
echo ""

WORK_DIR="${HOME}/drm-work"
KERNEL_DIR="${WORK_DIR}/linux"
AMD_DIR="${WORK_DIR}/amd"

# 配置远程仓库
setup_remotes() {
    echo "[1/8] 配置远程仓库..."
    
    cd "${KERNEL_DIR}"
    
    # 添加 drm-misc 远程
    git remote add drm-misc \
        https://gitlab.freedesktop.org/drm/misc/kernel.git 2>/dev/null || true
    
    # 添加 AMD 远程
    git remote add amd \
        https://gitlab.freedesktop.org/drm/amd.git 2>/dev/null || true
    
    # 添加 linux-next 远程
    git remote add linux-next \
        https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git 2>/dev/null || true
    
    # 更新
    git remote update --prune
    
    echo "  ✅ 远程仓库已配置"
    echo ""
    git remote -v | head -10
}

# 查看分支状态
branch_status() {
    echo "[2/8] 查看分支状态..."
    
    cd "${KERNEL_DIR}"
    
    echo "  drm-misc 分支:"
    for branch in "drm-misc/drm-misc-next" "drm-misc/drm-misc-fixes"; do
        local commits=$(git rev-list --count HEAD.."${branch}" 2>/dev/null || echo 0)
        echo "    ${branch}: ${commits} 个提交未合并"
    done
    
    echo ""
    echo "  amd-staging:"
    local amd_commits=$(git rev-list --count HEAD..amd/amd-staging 2>/dev/null || echo 0)
    echo "    amd/amd-staging: ${amd_commits} 个提交未合并"
    
    echo ""
    echo "  linux-next:"
    local next_commits=$(git rev-list --count HEAD..linux-next/master 2>/dev/null || echo 0)
    echo "    linux-next/master: ${next_commits} 个提交未合并"
    
    echo ""
    echo "  ⚠️  注意: 大量未合并提交可能表示有冲突"
}

# 获取最新分支
fetch_all() {
    echo "[3/8] 获取所有分支更新..."
    
    cd "${KERNEL_DIR}"
    
    echo "  获取 drm-misc..."
    git fetch drm-misc --prune 2>&1 | tail -3
    
    echo ""
    echo "  获取 amd..."
    git fetch amd --prune 2>&1 | tail -3
    
    echo ""
    echo "  获取 linux-next..."
    git fetch linux-next --prune 2>&1 | tail -3
    
    echo ""
    echo "  ✅ 所有分支已更新"
    
    # 显示最新标签
    echo ""
    echo "  最新标签:"
    git tag -l "drm-misc-*" --sort=-creatordate | head -5
}

# 查看某个补丁在各个分支的状态
patch_status() {
    echo "[4/8] 补丁分支跟踪..."
    
    local commit_id="$1"
    
    if [ -z "${commit_id}" ]; then
        echo "  用法: $0 patch-status <commit-id>"
        return 1
    fi
    
    cd "${KERNEL_DIR}"
    
    echo "  跟踪补丁: ${commit_id}"
    echo ""
    
    # 检查在各个分支中是否存在
    for branch in "drm-misc/drm-misc-next" "drm-misc/drm-misc-fixes" \
                  "amd/amd-staging" "linux-next/master" "master"; do
        if git merge-base --is-ancestor "${commit_id}" "${branch}" 2>/dev/null; then
            echo "  ✅ ${branch}: 已包含"
        else
            echo "  ❌ ${branch}: 未包含"
        fi
    done
    
    echo ""
    echo "  补丁详情:"
    git log --oneline -1 "${commit_id}" 2>/dev/null || echo "    commit 未找到"
}

# 创建跟踪分支
create_tracking_branches() {
    echo "[5/8] 创建本地跟踪分支..."
    
    cd "${KERNEL_DIR}"
    
    # drm-misc 分支
    git checkout -b drm-misc-next drm-misc/drm-misc-next 2>/dev/null || \
        git checkout drm-misc-next && git merge --ff-only drm-misc/drm-misc-next
    
    git checkout -b drm-misc-fixes drm-misc/drm-misc-fixes 2>/dev/null || \
        git checkout drm-misc-fixes && git merge --ff-only drm-misc/drm-misc-fixes
    
    git checkout -b amd-staging amd/amd-staging 2>/dev/null || \
        git checkout amd-staging && git merge --ff-only amd/amd-staging
    
    git checkout master
    
    echo "  ✅ 跟踪分支已创建"
    echo ""
    git branch | grep -E "drm-misc|amd-staging"
}

# 查看补丁上游进度
check_upstream_progress() {
    echo "[6/8] 查看上游进度..."
    
    cd "${KERNEL_DIR}"
    
    # 获取当前内核版本
    local kernel_version=$(make kernelversion 2>/dev/null || \
                          head -5 Makefile | grep VERSION | head -1 | awk '{print $3}')
    
    echo "  当前内核版本: ${kernel_version}"
    echo ""
    
    # 查看 drm-misc-next 相对于主线的进度
    echo "  drm-misc-next 新提交 (尚未进入主线):"
    git log --oneline master..drm-misc/drm-misc-next --max-count=10 2>/dev/null || \
        echo "    (无法获取)"
    
    echo ""
    echo "  amd-staging 新提交 (尚未进入 drm-misc-next):"
    git log --oneline drm-misc/drm-misc-next..amd/amd-staging --max-count=10 2>/dev/null || \
        echo "    (无法获取)"
    
    echo ""
    echo "  linux-next 新提交 (尚未进入主线):"
    git log --oneline master..linux-next/master --max-count=5 2>/dev/null || \
        echo "    (无法获取)"
}

# 生成分支报告
generate_report() {
    echo "[7/8] 生成分支状态报告..."
    
    cd "${KERNEL_DIR}"
    
    local report_file="${WORK_DIR}/branch-report-$(date +%Y%m%d).md"
    
    cat > "${report_file}" << 'HEADER'
# DRM 分支状态报告

## 概述
HEADER
    
    echo "" >> "${report_file}"
    echo "报告生成时间: $(date)" >> "${report_file}"
    echo "" >> "${report_file}"
    
    # 内核版本
    echo "## 内核版本" >> "${report_file}"
    echo "- 主线: $(git describe --tags master 2>/dev/null || echo 'N/A')" >> "${report_file}"
    echo "- linux-next: $(git describe --tags linux-next/master 2>/dev/null || echo 'N/A')" >> "${report_file}"
    echo "" >> "${report_file}"
    
    # 分支差距
    echo "## 分支差距" >> "${report_file}"
    echo "| 分支 | 领先 | 落后 |" >> "${report_file}"
    echo "|------|------|------|" >> "${report_file}"
    
    for branch in "drm-misc/drm-misc-next" "amd/amd-staging" "linux-next/master"; do
        local ahead=$(git rev-list --count master.."${branch}" 2>/dev/null || echo 0)
        local behind=$(git rev-list --count "${branch}"..master 2>/dev/null || echo 0)
        echo "| ${branch} | ${ahead} | ${behind} |" >> "${report_file}"
    done
    
    echo "" >> "${report_file}"
    
    # 近期重要提交
    echo "## amd-staging 近期提交" >> "${report_file}"
    echo '```' >> "${report_file}"
    git log --oneline master..amd/amd-staging --max-count=20 \
        -- "drivers/gpu/drm/amd/" 2>/dev/null >> "${report_file}"
    echo '```' >> "${report_file}"
    
    echo "  ✅ 报告已生成: ${report_file}"
}

# 同步分支
sync_branches() {
    echo "[8/8] 同步本地分支..."
    
    cd "${KERNEL_DIR}"
    
    # 同步 drm-misc-next
    echo "  同步 drm-misc-next..."
    git checkout drm-misc-next 2>/dev/null
    git merge --ff-only drm-misc/drm-misc-next 2>/dev/null && \
        echo "    ✅ 已同步" || echo "    ⚠️ 需要手动处理"
    
    # 同步 amd-staging
    echo "  同步 amd-staging..."
    git checkout amd-staging 2>/dev/null
    git merge --ff-only amd/amd-staging 2>/dev/null && \
        echo "    ✅ 已同步" || echo "    ⚠️ 需要手动处理"
    
    # 回到 master
    git checkout master 2>/dev/null
    
    echo ""
    echo "  ✅ 同步完成"
}

# 主流程
case "$1" in
    setup)
        setup_remotes
        create_tracking_branches
        ;;
    status)
        branch_status
        ;;
    fetch)
        fetch_all
        ;;
    patch-status)
        patch_status "$2"
        ;;
    report)
        generate_report
        ;;
    sync)
        sync_branches
        ;;
    all)
        fetch_all
        sync_branches
        generate_report
        ;;
    *)
        echo "用法: $0 {setup|status|fetch|patch-status|report|sync|all}"
        echo ""
        echo "  setup        - 配置远程仓库和跟踪分支"
        echo "  status       - 查看分支状态"
        echo "  fetch        - 获取所有分支更新"
        echo "  patch-status - 跟踪补丁状态"
        echo "  report       - 生成分支状态报告"
        echo "  sync         - 同步本地分支"
        echo "  all          - 完整流程"
        ;;
esac
```

#### Cherry-Pick 和向后移植

```bash
#!/bin/bash
# 补丁向后移植工具

set -e

echo "=============================================="
echo "  补丁向后移植和 Cherry-Pick 工具"
echo "=============================================="
echo ""

# 配置
KERNEL_DIR="${HOME}/linux"
STABLE_DIR="${HOME}/linux-stable"

# 1. 从主线 cherry-pick 到稳定分支
cherry_pick_to_stable() {
    echo "[1/5] Cherry-pick 到稳定分支..."
    
    local commit_id="$1"
    local stable_version="$2"  # e.g., 6.6.y
    
    if [ -z "${commit_id}" ] || [ -z "${stable_version}" ]; then
        echo "用法: $0 cp-stable <commit-id> <stable-version>"
        echo "例如: $0 cp-stable abc123 6.6.y"
        return 1
    fi
    
    cd "${KERNEL_DIR}"
    
    # 检查 commit 是否存在
    if ! git cat-file -e "${commit_id}" 2>/dev/null; then
        echo "  ❌ Commit 不存在: ${commit_id}"
        return 1
    fi
    
    echo "  Commit: ${commit_id}"
    echo "  主题: $(git log --oneline -1 ${commit_id})"
    echo "  目标: ${stable_version}"
    echo ""
    
    # 切换到稳定分支
    local stable_branch="linux-${stable_version}.y"
    if [ ! -d "${STABLE_DIR}" ]; then
        echo "  克隆稳定树..."
        git clone \
            https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git \
            "${STABLE_DIR}" \
            --branch "${stable_branch}" \
            --depth 100
    fi
    
    cd "${STABLE_DIR}"
    
    # 确保分支最新
    git checkout "${stable_branch}"
    git pull origin "${stable_branch}"
    
    # 应用补丁
    echo "  正在 cherry-pick..."
    if git cherry-pick "${commit_id}"; then
        echo "  ✅ Cherry-pick 成功"
        
        # 添加向后移植标签
        local commit_msg=$(git log --format="%B" -1 HEAD)
        local new_msg="$(echo "${commit_msg}" | head -1)

$(echo "${commit_msg}" | tail -n +3)

(cherry picked from commit ${commit_id})
Cc: stable@vger.kernel.org
"
        
        git commit --amend -m "${new_msg}"
        echo "  ✅ 已添加向后移植标记"
    else
        echo "  ❌ Cherry-pick 失败，存在冲突"
        echo "  请手动解决冲突后运行:"
        echo "    git cherry-pick --continue"
        echo "  或取消:"
        echo "    git cherry-pick --abort"
    fi
}

# 2. 创建向后移植补丁
create_backport_patch() {
    echo "[2/5] 创建向后移植补丁..."
    
    local commit_id="$1"
    local target_kernel="$2"
    local patch_file="$3"
    
    if [ -z "${commit_id}" ] || [ -z "${target_kernel}" ]; then
        echo "用法: $0 backport <commit-id> <target-kernel> [patch-file]"
        echo "例如: $0 backport abc123 5.15 /tmp/backport.patch"
        return 1
    fi
    
    cd "${KERNEL_DIR}"
    
    if [ -z "${patch_file}" ]; then
        patch_file="${HOME}/backport-$(basename ${commit_id})-${target_kernel}.patch"
    fi
    
    # 生成补丁
    git format-patch \
        --stdout \
        -1 "${commit_id}" \
        > "${patch_file}"
    
    # 修改 Subject 添加向后移植信息
    sed -i "1s/^Subject:/Subject: [BACKPORT ${target_kernel}]/" "${patch_file}"
    
    # 添加向后移植说明
    cat >> "${patch_file}" << EOF

---
Backport notes for ${target_kernel}:
- Original commit: ${commit_id}
- Target kernel: ${target_kernel}
- Required modifications:
  - None (clean apply)

EOF
    
    echo "  ✅ 向后移植补丁已创建: ${patch_file}"
}

# 3. 测试向后移植
test_backport() {
    echo "[3/5] 测试向后移植..."
    
    local patch_file="$1"
    local test_kernel="$2"
    
    if [ -z "${patch_file}" ] || [ -z "${test_kernel}" ]; then
        echo "用法: $0 test <patch-file> <test-kernel>"
        return 1
    fi
    
    cd "${KERNEL_DIR}"
    
    # 切换到测试内核版本
    echo "  切换到内核 ${test_kernel}..."
    git checkout "v${test_kernel}" 2>/dev/null || {
        echo "  ❌ 内核版本 v${test_kernel} 不存在"
        return 1
    }
    
    # 尝试应用补丁
    echo "  测试补丁应用..."
    if git am "${patch_file}" 2>&1; then
        echo "  ✅ 补丁干净应用"
        git am --abort 2>/dev/null
    else
        echo "  ❌ 补丁存在冲突"
        git am --abort 2>/dev/null
        echo ""
        echo "  尝试 3-way merge..."
        if git am -3 "${patch_file}" 2>&1; then
            echo "  ✅ 3-way merge 成功"
            git am --abort 2>/dev/null
        else
            echo "  ❌ 3-way merge 也失败"
            git am --abort 2>/dev/null
        fi
    fi
    
    git checkout master
}

# 4. 生成分支流转图
generate_flow_diagram() {
    echo "[4/5] 生成分支流转图..."
    
    cd "${KERNEL_DIR}"
    
    local output_file="${HOME}/branch-flow-$(date +%Y%m%d).dot"
    
    cat > "${output_file}" << 'EOF'
digraph drm_branch_flow {
    rankdir=LR;
    node [shape=box, style=filled, fillcolor=lightblue];
    
    subgraph cluster_upstream {
        label="上游";
        style=filled;
        fillcolor=lightyellow;
        
        "开发者提交" -> "邮件列表 Review";
        "邮件列表 Review" -> "amd-staging";
    }
    
    subgraph cluster_drm {
        label="DRM 子系统";
        style=filled;
        fillcolor=lightgreen;
        
        "amd-staging" -> "drm-misc-next" [label="功能"];
        "amd-staging" -> "drm-misc-fixes" [label="修复"];
        "drm-misc-next" -> "linux-next" [label="定期 PR"];
    }
    
    subgraph cluster_linus {
        label="Linus 主线";
        style=filled;
        fillcolor=lightcoral;
        
        "linux-next" -> "主线合并窗口" [label="MW PR"];
        "drm-misc-fixes" -> "主线 rc" [label="rc PR"];
        "主线合并窗口" -> "正式发布";
        "主线 rc" -> "正式发布";
    }
    
    subgraph cluster_stable {
        label="稳定树";
        style=filled;
        fillcolor=lightgray;
        
        "正式发布" -> "stable 树" [label="向后移植"];
    }
}
EOF
    
    echo "  ✅ DOT 文件已生成: ${output_file}"
    echo "  使用 graphviz 渲染: dot -Tpng ${output_file} -o branch-flow.png"
}

# 5. 查看特定提交的流转历史
commit_journey() {
    echo "[5/5] 查看提交流转历史..."
    
    local commit_id="$1"
    
    if [ -z "${commit_id}" ]; then
        echo "用法: $0 journey <commit-id>"
        return 1
    fi
    
    cd "${KERNEL_DIR}"
    
    echo "  提交: ${commit_id}"
    echo ""
    
    # 查看提交元数据
    echo "  提交信息:"
    git log --format="%H%nAuthor: %an <%ae>%nDate: %ad%nSubject: %s" -1 "${commit_id}"
    echo ""
    
    # 查看 Fixes 标签
    local fixes_tag=$(git log --format="%b" -1 "${commit_id}" | grep "^Fixes:" || true)
    if [ -n "${fixes_tag}" ]; then
        echo "  修复: ${fixes_tag}"
    fi
    
    # 查看向后移植历史
    local cherry_pick=$(git log --all --oneline --grep="cherry picked from commit ${commit_id:0:12}" \
                       2>/dev/null || true)
    if [ -n "${cherry_pick}" ]; then
        echo ""
        echo "  向后移植到:"
        echo "${cherry_pick}"
    fi
    
    echo ""
    echo "  分支流转:"
    for branch in $(git branch -r --contains "${commit_id}" 2>/dev/null); do
        echo "    ✅ ${branch}"
    done
}

# 主流程
case "$1" in
    cp-stable)
        cherry_pick_to_stable "$2" "$3"
        ;;
    backport)
        create_backport_patch "$2" "$3" "$4"
        ;;
    test)
        test_backport "$2" "$3"
        ;;
    flow)
        generate_flow_diagram
        ;;
    journey)
        commit_journey "$2"
        ;;
    *)
        echo "用法: $0 {cp-stable|backport|test|flow|journey}"
        echo ""
        echo "  cp-stable  - Cherry-pick 到稳定分支"
        echo "  backport   - 创建向后移植补丁"
        echo "  test       - 测试补丁向后移植"
        echo "  flow       - 生成分支流转图"
        echo "  journey    - 查看提交流转历史"
        ;;
esac
```

### 案例分析

#### 案例一：amd-staging 到 drm-misc-next 的批量推送

**背景：**
AMD GPU 驱动团队完成了 3 周的新功能开发，积累了一批经过内部 CI 测试的补丁，需要批量推送到 drm-misc-next。

**操作过程：**

```
AMD 内部:

1. 内部 CI 运行:
   - 300+ GPU 覆盖测试
   - 性能回归测试
   - 压力测试 (72 小时)

2. Code Review:
   - Alex Deucher 最终 review
   - 确认所有补丁都有 Reviewed-by
   - 检查补丁对内核配置的影响

3. 推送到公共 amd-staging:
   git push gitlab amd-staging

4. 发送 Pull Request 到 drm-misc:
   Alex 发送邮件到 Maarten Lankhorst
   Subject: [GIT PULL] amd-staging for drm-misc-next
```

**drm-misc 维护者处理：**

```
Maarten Lankhorst:

1. 拉取 amd-staging 分支
2. 检查补丁质量:
   - CI 结果 (drm-ci)
   - Reviewed-by 标签
   - 与 drm-misc-next 的冲突
3. 合并到 drm-misc-next
4. 发送到 linux-next

冲突处理:

如果 amd-staging 和 drm-misc-next 有冲突:
  1. Maarten 通知 Alex 需要 rebase
  2. Alex rebase amd-staging 到最新 drm-misc-next
  3. 重新运行 CI
  4. 重新发送 PR
```

**时间线：**

```
6月1日  : AMD 内部开始 3 周开发周期
6月15日 : 内部代码冻结，开始 CI 测试
6月18日 : CI 通过，推送到公共 amd-staging
6月19日 : Alex 发送 PR 到 drm-misc
6月20日 : Maarten 合并到 drm-misc-next
6月21日 : drm-misc-next 进入 linux-next
7月15日 : 合并窗口打开，进入 Linus 主线
```

#### 案例二：修复补丁的紧急流转

**背景：**
AMD 发现一个严重的 GPU 挂起问题，影响所有 RDNA3 用户，需要在 rc 周期中紧急修复。

**问题描述：**
```
驱动在特定的游戏场景下触发 GPU hang:
  - 问题: GFX 引擎在特定 shader 指令序列后挂起
  - 影响: Navi31, Navi32
  - 严重性: 高 (用户数据可能丢失)

修复补丁:
  drm/amdgpu: fix GFX hang on RDNA3 with specific shader sequence
```

**紧急流转路径：**

```
时间线:

Day 1 (rc2 发布后 2 天):
  09:00 - AMD 内部验证修复
  10:00 - Alex Deucher 确认修复
  11:00 - 合并到 amd-staging
  12:00 - 内部 CI 运行
  14:00 - CI 通过

Day 2:
  10:00 - Alex 发送补丁到 amd-gfx@ 邮件列表
  10:30 - Christian König 回复 Reviewed-by
  11:00 - 合并到 drm-misc-fixes
  12:00 - Maarten 发送 PR 到 Linus
  14:00 - Linus 合并到主线

Day 3:
  补丁包含在 rc3 中发布
```

```
补丁格式:

From: Alex Deucher <alexander.deucher@amd.com>
Subject: [PATCH] drm/amdgpu: fix GFX hang on RDNA3 with
 specific shader sequence

[PATCH] drm/amdgpu: fix GFX hang on RDNA3 with specific shader sequence

Fix a GFX hang that occurs on RDNA3 GPUs when a specific sequence
of shader instructions is executed. The root cause is a race
condition in the shader program counter update logic.

v2: Add proper synchronization barrier (Christian)

Fixes: abc123 ("drm/amdgpu: add RDNA3 GFX support")
Cc: stable@vger.kernel.org
Signed-off-by: Alex Deucher <alexander.deucher@amd.com>
Reviewed-by: Christian König <christian.koenig@amd.com>
```

#### 案例三：linux-next 集成测试发现冲突

**背景：**
在 linux-next 集成测试中，发现 AMDGPU 的补丁和 Intel 的补丁在 DRM 核心代码上产生了冲突。

**冲突详情：**
```
冲突文件: drivers/gpu/drm/drm_gem.c

AMDGPU 修改: 添加 drm_gem_object_put_locked() 函数
Intel 修改: 重构 drm_gem_object_put() 的锁语义

两个补丁修改了同一个函数的相邻行
```

**冲突解决流程：**

```
1. linux-next 维护者 (Stephen Rothwell) 发现冲突
   发送邮件:
   Subject: linux-next: manual merge of drm trees

2. 相关维护者协调:
   - Maarten Lankhorst (drm-misc) 协调
   - Alex Deucher (AMD) 提供 AMD 侧修改说明
   - Jani Nikula (Intel) 提供 Intel 侧修改说明

3. 解决方案:
   - AMD 修改补丁，使用 Intel 新的锁语义
   - 重新提交 v2 补丁到 amd-staging
   - Intel 补丁保持不变

4. 验证:
   - linux-next 重新构建
   - 确认两个补丁兼容

5. 结果:
   - AMD 提交 v2 补丁
   - linux-next 集成成功
   - 两个补丁都在下一个合并窗口进入主线
```

### 相关链接

- [DRM 维护者分支](https://dri.freedesktop.org/wiki/DRM-Maintainer-Branches/)
- [drm-misc Git 仓库](https://gitlab.freedesktop.org/drm/misc/kernel)
- [AMD Staging Git 仓库](https://gitlab.freedesktop.org/drm/amd)
- [linux-next 集成测试](https://linux-next.sourceforge.net/)
- [Linux 内核发布周期](https://www.kernel.org/doc/html/latest/process/2.Process.html)
- [稳定内核向后移植](https://www.kernel.org/doc/html/latest/process/stable-kernel-rules.html)

### 今日小结

drm-misc 和 amd-staging 分支管理模型是 Linux GPU 驱动开发的核心基础设施：

1. **三层架构**: amd-staging (AMD 内部) → drm-misc (DRM 子系统) → Linus 主线
2. **分支类型**: next (新功能), fixes (修复), stable (向后移植)
3. **流转规则**: 功能 → drm-misc-next → linux-next → 合并窗口 → 主线
4. **维护者角色**: AMD 维护者 (Alex/Christian), drm-misc 维护者, 内核维护者
5. **发布周期**: 2 个月内核版本, rc 周期修复, 稳定版向后移植
6. **Pull Request**: 标签驱动, 定期 PR, linux-next 集成测试
7. **冲突处理**: 跨子系统冲突需要维护者协调解决

### 扩展思考

1. 如何设计自动化的分支状态监控系统，实时跟踪补丁在各分支的流转状态？
2. AMD 内部 tree 和公共 amd-staging 的同步机制是否可以进一步优化？
3. 在多个 GPU 厂商 (AMD, Intel, NVIDIA) 同时修改 DRM 核心时，如何减少冲突？
4. 如果希望为新硬件贡献支持，需要在哪个时间点开始准备才能赶上特定的内核版本？
