# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 这个仓库是什么

**编排仓库，不是内核源码仓库。** 仓库里没有一行内核代码，只有两个 GitHub Actions 工作流 + 一个补丁文件。真正的内核源码（`common/`）在每次运行时从 `showdo/Build_Lenovo_sm8750` 的 releases 下载 `${{ os_patch_level }}.tar.gz` 解压得到，构建完即丢弃。

因此：**没有本地构建、没有测试、没有 lint。** 唯一的验证手段是推送后触发 Actions 并读日志。

```
.github/workflows/Build_Lenovo_sm8750.yml   # 主构建（千行量级，全部逻辑都在这个文件里）
.github/workflows/Clear_All_Workflow.yml    # 清理 Actions 运行记录
patch/                                      # 仅 adios_ioscheduler.patch，按绝对路径引用
```

`patch/` 只剩一个补丁。Droidspaces 与 ntsync 的补丁**已不再入库**，改为构建时从上游现拉（见「补丁与构建期外部拉取」）。

## 分支：main 与 dev

**两者的唯一差异是 `CONFIG_USER_NS`**，dev 启用、main 不启用，其余完全一致。这条不变量是刻意的，改动时不要引入别的分叉；`git diff main dev` 应当只显示工作流里那一个 USER_NS 配置块。

- `CONFIG_USER_NS` 对应上游 Droidspaces 清单里标注 *"Fix for docker unsafe procfs error"* 的可选项，缺它时容器内再跑 Docker 会撞 procfs 报错。
- 但 user namespace 历来是本地提权漏洞的高发区，Android GKI 有意默认关闭。**所以只在 dev 上启用** —— 这是为换取嵌套容器能力做的有意取舍，不是遗漏。

推 dev 前先推 main，否则 `origin/main` 会落后于 `origin/dev` 里已包含的历史。

## 常用命令

```bash
# 触发构建（工作流必须已存在于默认分支）
gh workflow run Build_Lenovo_sm8750.yml \
  -f kernel_version=6.6.89 \
  -f enable_droidspaces=678          # choice：off / 678 / 123 / 345

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

如果这两个工具都装不上，**至少做两件事**：把每个 `run:` 块抽出来逐个 `bash -n`（能抓到解合并冲突留下的悬空 `fi`——这类错误 YAML 层面完全合法），以及 grep 一遍残留的冲突标记。再顺手校验一遍：`github.event.inputs.*` 都有声明、`env.*` 引用都有定义、引用的 `patch/*.patch` 在磁盘上存在。

产物（Artifacts）：`kernel_image_<version>`、`Boot_<KSUVER>_<version>_<KSU_NAME>_AOSP_Signed`（`enable_signed` 时）、`Re-Kernel-v<version>`（`enable_ReKernel` 时）。`KSU_NAME` 是 `BakaSU`，或关闭 KSU 注入时的 `NoKSU`；`KSUVER` 取自 `git rev-list --count origin/main + 30700`（取不到时回退 `114514`），关闭注入时为 `none`——**即产物名依赖 BakaSU 仓库的提交数**，因此这个数字会随上游提交漂移，不能当成固定版本号写进文档或做断言（历史上曾因算式的分支名不存在而整体回退成 `114514`，见 `53316bd`）。

**本仓库不产出 AnyKernel3 刷入包**，只产出上述原始 `Image`、重打包签名后的 `boot.img` 和 Re-Kernel 模块。

## 构建流水线架构

主工作流是**单文件线性流水线**：输入开关 → 环境变量 → 条件步骤 → 追加 defconfig → 编译 → 产物。理解它只需抓住四条主线。

### 1. 输入开关 → `ENABLE_*` 环境变量 → 条件步骤

`workflow_dispatch` 的 `enable_*` 输入在 job 的 `env:` 块里映射为 `ENABLE_*`，然后被两处消费：步骤的 `if:` 条件，以及 `Set gki_defconfig` 步骤里的配置写入。

**`enable_BakaSU` 是总开关**：它关掉时，SUSFS 步骤整体跳过、`CONFIG_KSU_*`/`CONFIG_KSU_SUSFS_*` 一行都不写，`KSUVER=none`、`KSU_NAME=NoKSU`。工作流里对此有多处显式判断和提示文案，是刻意设计而非冗余。

**`enable_droidspaces` 是 `choice` 不是布尔**：取值 `off` / `678` / `123` / `345`，默认 `678`。所以判断一律写成 `!= 'off'`，**不要写 `== 'true'`** —— 后者在新取值下永远为假。历史上缓存变体判断就踩过这个坑（见 §4）。`off` 之外的三个值是要应用的 kABI 补丁槽位。

`enable_droidspaces_ntsync` 是 `enable_droidspaces` 的从属开关，只有 Droidspaces 不为 `off` 时才可能生效。

`enable_ReKernel` 是唯一**同时**出现在两处的开关：它既在主 job 里决定是否把 Re-Kernel 源码编进内核，又通过 `github.event.inputs.enable_ReKernel` 控制第二个 job `upload_rekernel` 是否运行。

**`enable_bbg` 是唯一会改 `security/Kconfig` 的开关**：它先从 `vc-teahouse/Baseband-guard` 拉 `setup.sh` 直接管道进 `bash` 执行，再直写 `CONFIG_BBG=y`，最后用一段**范围式 `sed`** 把 LSM 的 `default` 行补成 `lockdown,baseband_guard`。那段 sed 同时依赖三件事——`^config LSM$` 到 `^help$` 的区间、`default` 行里已存在 `lockdown`、以及 `baseband_guard` 尚未出现——上游 Kconfig 措辞一变就会**静默不生效**，构建不报任何错。

**KPM 已整体移除**（`enable_kpm` 输入、`CONFIG_KPM`、打补丁步骤全删），两个分支都不保留。工作流里任何 `KPM` 字样都是遗漏。

### 2. 内核版本矩阵分散在 4 个 `case` 块里

`kernel_version` 输入（四选一）驱动**四处**独立的 `case` 分支，全部在 `.github/workflows/Build_Lenovo_sm8750.yml` 中：

| kernel_version | os_patch_level | releases | 默认后缀 / 时间 |
|---|---|---|---|
| `6.6.56` | `2024-11` | `TB322FC_1.1.11.120` | `...ab12829524-4k` / 2024-12-19 |
| `6.6.89` | `2025-06` | `ColorOS_16_400_OPD_Boot` | `...ab13680582-4k` / 2025-06-23 |
| `6.6.102` | `2025-10` | `TB322FC_1.5.10.096` | `...ab14559830-4k` / 2025-12-09 |
| `6.6.118` | `2026-01` | `TB322_ZUXOS_1.5.10.183` | `...ab14943902-4k` / 2026-02-26 |

四处位置：`os_patch_level`+`releases`（~L116）、`DEFAULT_SUFFIX`（~L138）、`KERNEL_TIME`（~L159），以及 SUSFS 针对 6.6.56 的 `undeclared identifier 'vma'` 编译修复（~L506）。**新增版本必须同步这四处**，漏改任何一处都会静默退化成某个旧版本的配置。

`os_patch_level` 同时决定下载哪份 `common.tar.gz`，所以它是版本矩阵里最硬的一环。

**另有一处版本专属覆盖**：`6.6.118` 在同一个 `case` 块里额外写入 `BOOT_IMG_REPO` / `BOOT_IMG_TAG`，让签名改用自有 release 上传的 boot.img（~L130）。这只影响从哪个 boot.img 取容器，不影响内核名——内核名里的版本串来自 `DEFAULT_SUFFIX`，与 `releases` 无关。

### 3. 补丁与构建期外部拉取

**仓库内补丁只剩一个**，从 `patch/` 按绝对路径引用，应用到 `kernel_workspace/common/`：

| 补丁 | 工具 | 失败处理 |
|---|---|---|
| `adios_ioscheduler.patch` | `patch -p1 --ignore-whitespace -F 3` | 容忍（`malformed patch` 时降级 `--force`） |

**其余源码改动都在构建时从外部拉取**，仓库里不留副本：

| 内容 | 来源 | 门控 |
|---|---|---|
| BakaSU 内核侧注入 | `raw.githubusercontent.com/Baka-SU/BakaSU/.../kernel/setup.sh` | `enable_BakaSU`（该步没有 `if:`，开关为假时内部 `exit 0`） |
| SUSFS 内核补丁 | `git clone cctv18/susfs4oki -b oki-android15-6.6` | `ENABLE_BAKASU` **且** `ENABLE_SUSFS` |
| hide_stuff 补丁 | `raw.githubusercontent.com/cctv18/oppo_oplus_realme_sm8750/...` | 同上（在 SUSFS 步骤内） |
| Droidspaces kABI 补丁 | `git clone ravindu644/Droidspaces-OSS` → `Documentation/resources/kernel-patches/GKI/` | `ENABLE_DROIDSPACES != 'off'` |
| ntsync 补丁 ×2 | `raw.githubusercontent.com/Goldzxcbug/Droidspaces_Kernel_patch/.../NTsync/` | 另需 `ENABLE_DROIDSPACES_NTSYNC` |
| zram lz4 / zstd 补丁 | `git clone cctv18/oppo_oplus_realme_sm8750` → `zram_patch/`（`001-lz4.patch`、`lz4armv8.S`、`002-zstd.patch`） | **无条件**（该步没有 `if:`，与任何开关无关） |
| BBG 基带守护 | `vc-teahouse/Baseband-guard` 的 `setup.sh`，管道进 `bash` | `ENABLE_BBG` |

**工具链是整套缓存哈希的一部分**：`clang-r510928` 与 `build-tools` 都取自 `cctv18/oneplus_sm8650_toolchain` 的同一个 release，构建时按 `LLVM=1 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- O=out` 编 `Image`。换工具链等于换一套 ccache，不要指望旧缓存还能命中。

两条必须守住的不变量：

1. **仓库内补丁的路径写成 `${GITHUB_WORKSPACE}/patch/<name>`，且应用时 CWD 必须是 `kernel_workspace/common`。** `-p1` 剥掉补丁里的 `a/` 前缀，CWD 必须在被补丁树的根上。git 历史里连续多次修复（`ca3c477`、`0681218`、`44a404a`）全是踩这个坑。
2. **"补丁文件不存在"是硬失败（`exit 1`），"补丁应用失败"是有意容忍的（`|| true` / `--forward` + warning）。** 后者是为幂等性服务——重复运行时补丁可能已应用。不要因为看到 `|| true` 就"修复"成硬失败。

### 4. ccache 三级回退与时间劫持

缓存按 `KERNEL_VERSION` + 变体后缀（`ENABLE_DROIDSPACES != 'off'` 时加 `-ds`）分桶：

1. `actions/cache/restore` 取本仓库缓存
2. 未命中时，从 `showdo/Public_Ccache` 的 `exported-ccaches` release 里按资产名匹配公共缓存（见 `Find public ccache in releases` 步骤的 JS）
3. 仍未命中则空缓存开跑

**回写规则**：仅在构建耗时 > 480 秒、或本次没命中任何缓存时才保存；耗时 > 480 秒时还会顺手删除同前缀的旧缓存。这个 480 秒阈值在 `if:` 条件里出现两次，改动时需同步。

**`enable_droidspaces` 改成 choice 后，`-ds` 后缀一度变成死代码**——判断仍写着 `== "true"`，而取值已是 `678` 这类字符串。已修为 `!= "off"`。注意上游自己的那份仍带着这个 bug，所以两边的缓存分桶行为不同。

**时间劫持**是这套缓存能生效的前提，也是最容易让人困惑的一环：构建阶段用 `libfakestat.so` + `libfaketimeMT.so` 通过 `LD_PRELOAD` 介入 `cc-wrapper`/`ld-wrapper`，把编译器看到的文件时间戳和当前时间**固定**在 `2025-05-25`，同时 `KBUILD_BUILD_TIMESTAMP` 取自版本矩阵里的 `KERNEL_TIME`。所以在构建日志里，`date` 和 `stat` 的输出是假的——这是设计，不是故障，调缓存命中率时尤其要记住。

## Droidspaces 支持

`enable_droidspaces` 只做**内核侧准备**——产出的是"支持 Droidspaces 的内核"，**不是**注入了 Droidspaces 的内核。Droidspaces 本体是运行在设备上的用户态程序（[ravindu644/Droidspaces-OSS](https://github.com/ravindu644/Droidspaces-OSS)，Kotlin 编写），需在设备上另行安装。

**补丁与配置清单的来源**：kABI 补丁构建时从上游仓库现拉，因此**与上游的补丁是绑定关系，不再是手工复刻**；只有内核配置清单仍是照上游 `Documentation/Kernel-Configuration.md` 的 **"Configuring GKI Kernels"** 章节手工维护的。

该开关实际做三件事：

1. `git clone` 上游仓库，按槽位应用 `below-kernel-6.12/001.GKI-below-6.12-fix_sysvipc_kabi_${SLOT}.patch`。槽位就是 `enable_droidspaces` 的取值（`678` 为上游推荐，`123` / `345` 是备选）
2. 应用 ntsync 补丁 —— **上游 Droidspaces 文档中根本没有 ntsync**，这是本仓库额外添加的（NT 同步原语，服务于 Wine/Proton 类场景），来源是第三方的 `Goldzxcbug/Droidspaces_Kernel_patch`
3. 在 `Set gki_defconfig` 步骤写入内核配置（见下）

### 配置项与上游 GKI 清单的差异

**上游必需项 —— 全部已覆盖**：`CONFIG_SYSVIPC`、`CONFIG_POSIX_MQUEUE`、`CONFIG_IPC_NS`、`CONFIG_PID_NS`、`CONFIG_DEVTMPFS`、`CONFIG_NETFILTER_XT_MATCH_ADDRTYPE`

> **注意隐式依赖**：`init/Kconfig` 里 `config IPC_NS` 是 `depends on (SYSVIPC || POSIX_MQUEUE)`（而 `config USER_NS` 无任何 `depends`）。所以 `CONFIG_SYSVIPC` 与 `CONFIG_POSIX_MQUEUE` **至少要留一个**——两个都关掉时 `IPC_NS` 会被 Kconfig 静默解析回 `n`，构建不报任何错。这三项须同进同退。

**上游「可选但推荐」—— 已覆盖**：`CONFIG_NETFILTER_XT_TARGET_REJECT`、`CONFIG_NETFILTER_XT_TARGET_LOG`、`CONFIG_NETFILTER_XT_MATCH_RECENT`、`CONFIG_IP_SET` / `_HASH_IP` / `_HASH_NET`、`CONFIG_NETFILTER_XT_SET`。`CONFIG_TMPFS_POSIX_ACL` / `CONFIG_TMPFS_XATTR` 也有，但写在 Droidspaces 块**之外**、无条件生效。

**上游「可选但推荐」—— 分支相关**：

- `CONFIG_USER_NS` —— 上游标注为 *"Fix for docker unsafe procfs error"*。**dev 分支启用，main 不启用**，这是两个分支唯一的差异，理由见开头的「分支」一节。

**本仓库额外启用的（上游 GKI 清单之外）**：

- `CONFIG_IP_NF_TARGET_REJECT` —— 来自上游的「可选·防火墙」清单（UFW/Fail2ban）
- 禁用 `CONFIG_ANDROID_PARANOID_NETWORK` —— 来自上游的**非 GKI** 清单，上游注释是 "Disable this on **older kernels** to make internet work"，用在 6.6 上属于越界借用
- `CONFIG_NTSYNC=y` —— 上游 Droidspaces 文档中完全没有，受 `enable_droidspaces_ntsync` 单独门控

### 两条必须记住的

1. **kABI 补丁跟上游走，配置清单不跟。** 补丁是构建时现拉的，上游改了会自动生效；但配置清单是手工维护的，上游新增依赖、改名或调整推荐项时，本仓库**不会自动跟上，也不会报错**——只会安静地构建出一个缺项的内核，由你在设备上撞到问题。定期对照上游文档是唯一的防线。
2. **真正的验收手段只有上游 App 自带的检查器**：设备上 **Settings → Requirements → Check Requirements**，或终端 `su -c droidspaces check`。逐项比对配置清单只能证明"照着文档抄齐了"，不能证明"内核真的够用"。

另外，上游对 GKI 的警告原文是 **"DO NOT enable anything other than the recommended GKI configuration"**；但 kABI 真正敏感的只有 `SYSVIPC` / `IPC_NS` / `POSIX_MQUEUE`（它们会移动 `task_struct` 偏移量），而这三项正是 kABI 补丁所覆盖的。

## AOSP 签名（`enable_signed`）

签名步骤把新编出来的 `Image` 塞进一个**现成的 boot.img** 再重新签名，所以有两组独立来源：

- `BOOT_SIGNER_REPO` / `BOOT_SIGNER_REF` —— 签名**工具**（AOSP-Testkey-BootSigner）来源
- `BOOT_IMG_REPO` / `BOOT_IMG_TAG` —— 待签名**源 boot.img** 来源；`TAG` 留空则回退到本版本的 `releases`

两组都可被 repository variables 覆盖，且 `BOOT_IMG_*` 可按版本在版本矩阵里覆盖（目前只有 6.6.118 这么做）。**这两组必须各自独立**：启动时 `curl -f` 预检与随后实际下载都走 `BOOT_IMG_*`，早先版本里预检用一个、下载硬编码另一个，覆盖 `BOOT_SIGNER_REPO` 时会出现"校验一个仓库、下载另一个仓库"。

签名链路：`magiskboot repack`（替换内核）→ `python3 rebuild_avb.py --chained-mode`（重新签名）→ 校验产物存在。任一环节失败即 `exit 1`，不留半个产物。

## 改动时的注意事项

- **`name:` 字段是查找键。** `Clear_All_Workflow.yml` 用 `gh api ... select(.name == "...")` 按显示名定位工作流，并且清理自身时硬编码了 `"清理工作流运行记录"`。改任一工作流的 `name:` 会让清理工具失准。
- **判断开关是否真的接入，看 `ENABLE_*` 的出现次数。** 定义行之外还有引用才算生效。`enable_Adios` 曾经是个失效开关（`ENABLE_ADIOS` 只有定义、没有消费点），已于 2026-09-13 给该步骤补上 `if:` 门控。
- **读写 defconfig 必须同时识别 Kconfig 的两种写法**：已启用是 `CONFIG_X=y`，已禁用是 `# CONFIG_X is not set`（**不是** `CONFIG_X=n`）。`enable_config` 与 `disable_config` 都需两种都认，漏掉注释形式就会给已存在的项**重复追加**一行。基线 `gki_defconfig` 中绝大多数选项根本不出现，只有 `CONFIG_PID_NS` 这类是显式注释形式，所以这条很容易被漏测。重复行不跨运行累积（每次重新解压 `common.tar.gz`），但同一次运行内就会产生。
- **改 defconfig 一律通过那几个 helper**（`enable_config` / `disable_config` / `enable_config_if_defined` / `disable_config_if_defined`），不要直接 `echo >>`。只有 `CONFIG_TMPFS_XATTR` / `CONFIG_NTSYNC` 这类是刻意直写。`_if_defined` 变体会先用 `config_defined()` 扫 Kconfig，跳过当前内核没有的项。
- **`repo 变量`可覆盖上游来源**：`BOOT_SIGNER_REPO`、`BOOT_SIGNER_REF`、`BOOT_IMG_REPO`、`BOOT_IMG_TAG`、`PUBLIC_CACHE_REPO`、`PUBLIC_CACHE_TAG` 优先读 `vars.*`，未设置时回退到工作流内默认值。fork 后想换源应改 repository variables，而不是改工作流。
- **上游来源是 `qdykernel/Build_Lenovo_sm8750`**，本仓库是 `mineextremely/` 的 fork。合并上游时注意：上游把 KSU 供应商叫 `BakaSU`，而 `Baka-SU/BakaSU` 就是 `ReSukiSU` 改名后的**同一个仓库**（旧地址 301 重定向过去），别当成两个项目。
- **fork 标识被写进内核**：`CONFIG_KSU_FULL_NAME_FORMAT="%TAG_NAME%-%COMMIT_SHA%-GitHub@mineextremely"`。这是 fork 者的水印，非上游原值。
- **README 需与实现同步**：改工作流文件名、增删输入、改动产物命名或分支差异时，两份 README（`README.md` / `README-en.md`）的对应章节都要改——参数表、版本矩阵、产物表、补丁来源表、分支说明，逐处对齐。两份文件必须一一对应：章节结构一致，行数可以不同但内容要能逐条对上。
- **README 开头的 fork 警告块是刻意的**：它是读者看到的第一段内容，声明「个人 fork、与上游功能集已分叉、可能含不稳定 feature、刷机风险自负」。删掉它等于把 fork 伪装成上游产物，不要以"文档要精炼"为由砍掉；「与上游的差异」对照表和「分支说明」同理。
- README 中的 Telegram / 酷安 / GitHub 徽章仍指向上游 `qdykernel/`——那是刻意保留的出处标注，不要"顺手统一"成本仓库。
- **无 LICENSE 文件**，尽管 README 声明 GPL-3.0。

## Agent skills

### Issue tracker

Issues live as local markdown files under `.scratch/<feature>/` in this repo. See `docs/agents/issue-tracker.md`.

### Triage labels

The five canonical roles use their default label strings. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `CONTEXT.md` and `docs/adr/` at the repo root. See `docs/agents/domain.md`.
