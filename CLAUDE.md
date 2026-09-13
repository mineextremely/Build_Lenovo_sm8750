# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 这个仓库是什么

**编排仓库，不是内核源码仓库。** 仓库里没有一行内核代码，只有一个 GitHub Actions 工作流 + 4 个补丁文件。真正的内核源码（`common/`）在每次运行时从 `showdo/Build_Lenovo_sm8750` 的 releases 下载 `${{ os_patch_level }}.tar.gz` 解压得到，构建完即丢弃。

因此：**没有本地构建、没有测试、没有 lint。** 唯一的验证手段是推送后触发 Actions 并读日志。

```
.github/workflows/Build_Lenovo_sm8750.yml   # 主构建（992 行，全部逻辑都在这个文件里）
.github/workflows/Clear_All_Workflow.yml    # 清理 Actions 运行记录
patch/                                      # 4 个内核补丁，被主工作流按绝对路径引用
```

## 常用命令

```bash
# 触发构建（工作流必须已存在于默认分支）
gh workflow run Build_Lenovo_sm8750.yml \
  -f kernel_version=6.6.89 \
  -f enable_droidspaces=true

# 观察运行 / 读日志
gh run list --workflow=Build_Lenovo_sm8750.yml --limit 5
gh run watch
gh run view <run-id> --log

# 清理运行记录
gh workflow run Clear_All_Workflow.yml -f workflow_name=Build_Lenovo_sm8750 -f count=20

# 本地校验补丁能否干净应用（需先手动下载并解压对应 os_patch_level 的 common 源码）
cd path/to/common && patch -p1 --dry-run --forward < /path/to/patch/foo.patch

# 本地校验工作流改动（推送前必做，无需 CI 往返）
npx --yes js-yaml .github/workflows/Build_Lenovo_sm8750.yml    # YAML 语法
actionlint .github/workflows/*.yml                             # Actions 语义（需另装二进制）
```

改动工作流后至少要跑 `js-yaml`：**YAML 语法合法不等于语义正确**——`description: |` 这类块标量语法上完全合法，但会让输入说明变成一坨原始文本。想验证解析后的实际结构，用 node + js-yaml 读取对象树再断言字段，比肉眼可靠。

产物（Artifacts）：`kernel_image_<version>`、`Boot_<KSUVER>_<version>_<KSU_NAME>_AOSP_Signed`（`enable_signed` 时）、`Re-Kernel-v<version>`（`enable_ReKernel` 时）。

**本仓库不产出 AnyKernel3 刷入包**，只产出上述原始 `Image`、重打包签名后的 `boot.img` 和 Re-Kernel 模块。

## 构建流水线架构

主工作流是**单文件线性流水线**：输入开关 → 环境变量 → 条件步骤 → 追加 defconfig → 编译 → 产物。理解它只需抓住四条主线。

### 1. 输入开关 → `ENABLE_*` 环境变量 → 条件步骤

`workflow_dispatch` 的 `enable_*` 输入在 job 的 `env:` 块里映射为 `ENABLE_*`，然后被两处消费：步骤的 `if:` 条件，以及 `Set gki_defconfig` 步骤里的配置写入。

**`enable_ReSukiSU` 是总开关**：它关掉时，SUSFS 步骤整体跳过、KPM 跳过、`CONFIG_KSU_*`/`CONFIG_KSU_SUSFS_*` 一行都不写。工作流里对此有多处显式判断和提示文案，是刻意设计而非冗余。

`enable_ReKernel` 是唯一**同时**出现在两处的开关：它既在主 job 里决定是否把 Re-Kernel 源码编进内核，又通过 `github.event.inputs.enable_ReKernel` 控制第二个 job `upload_rekernel` 是否运行。

### 2. 内核版本矩阵分散在 4 个 `case` 块里

`kernel_version` 输入（四选一）驱动**四处**独立的 `case` 分支，全部在 `.github/workflows/Build_Lenovo_sm8750.yml` 中：

| kernel_version | os_patch_level | releases | 默认后缀 / 时间 |
|---|---|---|---|
| `6.6.56` | `2024-11` | `TB322FC_1.1.11.120` | `...ab12829524-4k` / 2024-12-19 |
| `6.6.89` | `2025-06` | `ColorOS_16_400_OPD_Boot` | `...ab13680582-4k` / 2025-06-23 |
| `6.6.102` | `2025-10` | `TB322FC_1.5.10.096` | `...ab14559830-4k` / 2025-12-09 |
| `6.6.118` | `2026-01` | `TB322_ZUXOS_1.5.10.183` | `...ab14943902-4k` / 2026-02-26 |

四处位置：`os_patch_level`+`releases`（~L117）、`DEFAULT_SUFFIX`（~L136）、`KERNEL_TIME`（~L157），以及 SUSFS 针对 6.6.56 的 `undeclared identifier 'vma'` 编译修复（~L503）。**新增版本必须同步这四处**，漏改任何一处都会静默退化成某个旧版本的配置。

`os_patch_level` 同时决定下载哪份 `common.tar.gz`，所以它是版本矩阵里最硬的一环。

### 3. 补丁应用：路径与工具的两条不变量

补丁从仓库根的 `patch/` 引用，但应用到 `kernel_workspace/common/` 里。两种写法并存，**不要统一它们**：

| 补丁 | 工具 | 失败处理 |
|---|---|---|
| `adios_ioscheduler.patch` | `patch -p1 --ignore-whitespace -F 3` | 容忍（`malformed patch` 时降级 `--force`） |
| `GKI-6.6-sysvipc_kabi_3_4_5.patch` | `patch -p1 --forward` | 容忍（warning） |
| `ntsync_base.patch` / `ntsync_compat_6.6.patch` | 先 `dos2unix`，再 `git apply --ignore-whitespace` | 容忍（warning） |

两条必须守住的不变量：

1. **路径写成 `${GITHUB_WORKSPACE}/patch/<name>`，且应用时 CWD 必须是 `kernel_workspace/common`。** `-p1` 剥掉补丁里的 `a/` 前缀，CWD 必须在被补丁树的根上。git 历史里连续多次修复（`ca3c477`、`0681218`、`44a404a`）全是踩这个坑。
2. **"补丁文件不存在"是硬失败（`exit 1`），"补丁应用失败"是有意容忍的（`|| true` / `--forward` + warning）。** 后者是为幂等性服务——重复运行时补丁可能已应用。不要因为看到 `|| true` 就"修复"成硬失败。

### 4. ccache 三级回退与时间劫持

缓存按 `KERNEL_VERSION` + 变体后缀（`ENABLE_DROIDSPACES` 时加 `-ds`）分桶：

1. `actions/cache/restore` 取本仓库缓存
2. 未命中时，从 `showdo/Public_Ccache` 的 `exported-ccaches` release 里按资产名匹配公共缓存（见 `Find public ccache in releases` 步骤的 JS）
3. 仍未命中则空缓存开跑

**回写规则**：仅在构建耗时 > 480 秒、或本次没命中任何缓存时才保存；耗时 > 480 秒时还会顺手删除同前缀的旧缓存。这个 480 秒阈值在 `if:` 条件里出现两次，改动时需同步。

**时间劫持**是这套缓存能生效的前提，也是最容易让人困惑的一环：构建阶段用 `libfakestat.so` + `libfaketimeMT.so` 通过 `LD_PRELOAD` 介入 `cc-wrapper`/`ld-wrapper`，把编译器看到的文件时间戳和当前时间**固定**在 `2025-05-25`，同时 `KBUILD_BUILD_TIMESTAMP` 取自版本矩阵里的 `KERNEL_TIME`。所以在构建日志里，`date` 和 `stat` 的输出是假的——这是设计，不是故障，调缓存命中率时尤其要记住。

## 改动时的注意事项

- **`name:` 字段是查找键。** `Clear_All_Workflow.yml` 用 `gh api ... select(.name == "...")` 按显示名定位工作流，并且清理自身时硬编码了 `"清理工作流运行记录"`。改任一工作流的 `name:` 会让清理工具失准。
- **判断开关是否真的接入，看 `ENABLE_*` 的出现次数。** 定义行之外还有引用才算生效。`enable_Adios` 曾经是个失效开关（`ENABLE_ADIOS` 只有定义、没有消费点），已于 2026-09-13 给该步骤补上 `if:` 门控。
- **defconfig 一律用 `>>` 追加**，不做去重。因为每次运行都重新解压 `common.tar.gz`，不存在累积问题——但别把这个模式搬到有持久状态的场景。
- **`repo 变量`可覆盖上游来源**：`BOOT_SIGNER_REPO`、`BOOT_SIGNER_REF`、`PUBLIC_CACHE_REPO`、`PUBLIC_CACHE_TAG` 优先读 `vars.*`，未设置时回退到工作流内默认值（`showdo/*`）。fork 后想换源应改 repository variables，而不是改工作流。
- **fork 标识被写进内核**：`CONFIG_KSU_FULL_NAME_FORMAT="%TAG_NAME%-%COMMIT_SHA%-GitHub@mineextremely"`。这是 fork 者的水印，非上游原值。
- **README 已于 2026-09-13 与实现对齐**：工作流表格、输入参数表、`patch/` 目录说明、状态徽章均已修正（此前它引用的 `build.yml`、`clean-caches.yml`、`clear_workflows.yml` 三个文件并不存在）。**改工作流文件名或增删输入时需同步两份 README 的表格。** README 中的 Telegram / 酷安 / GitHub 徽章仍指向上游 `qdykernel/`——那是刻意保留的出处标注，不要"顺手统一"成本仓库。
- **无 LICENSE 文件**，尽管 README 声明 GPL-3.0。

## Agent skills

### Issue tracker

Issues live as local markdown files under `.scratch/<feature>/` in this repo. See `docs/agents/issue-tracker.md`.

### Triage labels

The five canonical roles use their default label strings. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `CONTEXT.md` and `docs/adr/` at the repo root. See `docs/agents/domain.md`.
