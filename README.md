# Lenovo SM8750 内核构建项目

**`简体中文`** | [English](README-en.md)<br>

> [!WARNING]
> **这是个人 fork，不是上游项目**
>
> 本仓库是 [qdykernel/Build_Lenovo_sm8750](https://github.com/qdykernel/Build_Lenovo_sm8750) 的个人优化 fork
> （[mineextremely/Build_Lenovo_sm8750](https://github.com/mineextremely/Build_Lenovo_sm8750)），**已经在功能集上与上游分叉**：
> BakaSU 改名收口、移除 KPM、Droidspaces 改为槽位方案补丁、`dev` 分支启用 `CONFIG_USER_NS` 等。
>
> - 这些改动**未经上游审阅**，部分属于个人实验性调整，**可能存在不稳定的 feature**
> - 产物行为、参数名称、日志内容**均可能与上游不一致**，遇到问题请勿直接对照上游文档
> - 编译产物仅供自用与测试，**刷机风险自负**
>
> 上游项目：<https://github.com/qdykernel/Build_Lenovo_sm8750>

[![GitHub](https://img.shields.io/badge/-GitHub|@qdykernel-181717?logo=github&logoColor=white&style=flat-square)](https://github.com/qdykernel/Build_Lenovo_sm8750)
[![Telegram](https://img.shields.io/badge/Telegram-频道-blue.svg?logo=telegram)](https://t.me/qdykernel)
[![酷安|主页](https://img.shields.io/badge/酷安|主页-3DDC84?style=flat-square&logo=android&logoColor=white)](http://www.coolapk.com/u/1624571)
[![Workflow Status](https://img.shields.io/github/actions/workflow/status/mineextremely/Build_Lenovo_sm8750/Build_Lenovo_sm8750.yml?label=Build&logo=github-actions&style=flat-square)](https://github.com/mineextremely/Build_Lenovo_sm8750/actions)
<br>

---

## 📖 项目简介

本项目提供基于 **GitHub Actions** 的自动化内核编译工作流，支持 **Lenovo** 搭载 **SM8750** 平台的设备。通过高度集成的脚本，实现一键编译包含 **BakaSU (KernelSU)**、**SUSFS**、**ADIOS 调度**、**Droidspaces 容器支持**、**Re-Kernel** 等功能的 GKI 内核。

**本仓库是编排仓库，不含内核源码。** 每次构建时才从 `showdo/Build_Lenovo_sm8750` 的 releases 下载对应
`os_patch_level` 的 `common.tar.gz` 解压编译，构建完即丢弃；仓库里只有工作流与一个补丁文件。

### ✨ 主要特性

- 🚀 **全自动化编译** - 基于 GitHub Actions，无需本地环境
- 🔧 **BakaSU 集成** - 可开关的 BakaSU (KernelSU) 注入，联动 SUSFS；关闭后内核名标记为 `NoKSU`
- 🫥 **SUSFS 支持** - 内核级隐藏（sus_path / sus_mount / sus_map / open_redirect 等）
- ⚡ **性能优化** - 集成 ADIOS I/O 调度补丁、zram 的 lz4 1.10.0 / zstd 1.5.7 补丁
- 🐳 **Droidspaces 容器支持** - SYSVIPC kABI 修复（槽位可选）+ 可选的 ntsync 驱动
- 🧩 **Re-Kernel 集成** - 可开关的内核模块注入，并额外产出 Re-Kernel 模块压缩包
- 💾 **ccache 三级回退** - 本仓库缓存 → 公共缓存 → 空缓存，配合时间劫持保证可复现与高命中率
- 📦 **开箱即用** - 自动重打包 boot.img 并使用 AOSP 测试密钥签名

---

## 🆚 与上游的差异

从上游 fork 后，本仓库做过以下**结构性**调整，使用前请务必知悉：

| 项目 | 上游 | 本仓库 |
|------|------|--------|
| KSU 供应商命名 | `ReSukiSU` / `enable_ReSukiSU` | **`BakaSU` / `enable_BakaSU`**（同一项目，已改名） |
| KPM | 可选的 `enable_kpm` 开关 | **已整体移除**，输入项、`CONFIG_KPM`、打补丁步骤全删 |
| Droidspaces | 内嵌补丁文件 | 构建时从上游现拉，**改为槽位方案** `off` / `678` / `123` / `345` |
| 内核补丁 | `patch/` 下的多个补丁 | **仅保留 `adios_ioscheduler.patch`**，其余改为构建期从外部拉取 |
| `dev` 分支 | 与 main 无稳定差异约定 | **唯一差异为 `CONFIG_USER_NS`**（见下节） |
| 源 boot.img | 固定仓库与 tag | 可按内核版本覆盖（`6.6.118` 使用自有 release） |
| 内核水印 | 上游署名 | `CONFIG_KSU_FULL_NAME_FORMAT` 带 `GitHub@mineextremely` |

> ⚠️ 上游仍在对本项目进行上游维护；本 fork 的改动**不会回流**。若你想要跟随上游，
> 请直接使用 [上游仓库](https://github.com/qdykernel/Build_Lenovo_sm8750)。

---

## 🚦 分支说明：`main` 与 `dev`

两个分支的**唯一差异是 `CONFIG_USER_NS`**：`dev` 启用、`main` 不启用，其余完全一致。

- `CONFIG_USER_NS` 对应上游 Droidspaces 清单里标注 *"Fix for docker unsafe procfs error"* 的可选项，
  缺它时容器内再跑 Docker 会撞 procfs 报错。
- 但 user namespace 历来是本地提权漏洞的高发区，Android GKI 有意默认关闭。
  **所以只在 `dev` 上启用** —— 这是为换取嵌套容器能力做的有意取舍，不是遗漏。

**追求稳定请用 `main`**；只有确实需要在容器内嵌套 Docker 时，才考虑 `dev`。

---

## 🎯 快速开始

### 步骤 1. Fork 本仓库

点击仓库右上角的 **Fork** 按钮，将本仓库复制到你自己的 GitHub 账户。

### 步骤 2. 运行工作流

1. 进入你 Fork 的仓库
2. 点击 **Actions** 标签页
3. 选择 `Build_Lenovo_sm8750` 工作流
4. 点击右侧的 **Run workflow** 按钮
5. 填写编译参数（见下方参数表）
6. 等待编译完成（耗时取决于缓存命中情况，命中公共缓存后通常会快很多）
7. 在运行页面的 **Artifacts** 中下载编译产物

### 📋 可用工作流

| 工作流 | 说明 | 适用场景 |
|--------|------|----------|
| [Build_Lenovo_sm8750.yml](.github/workflows/Build_Lenovo_sm8750.yml) | 完整内核编译（BakaSU / SUSFS / ADIOS / Droidspaces / Re-Kernel 等） | 编译并集成所需功能 |
| [Clear_All_Workflow.yml](.github/workflows/Clear_All_Workflow.yml) | 清理工作流运行记录 | 保证 Actions 界面整洁 |

> ⚠️ 工作流必须已存在于**默认分支**上，才能手动触发。

---

## 🔧 编译参数

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `kernel_version` | 选项 | `6.6.89` | 内核版本：`6.6.56` / `6.6.89` / `6.6.102` / `6.6.118` |
| `custom_kernel_suffix` | 文本 | 空 | 自定义内核名称，留空则使用官方内核名 |
| `custom_kernel_time` | 文本 | 空 | 自定义构建时间，留空则使用官方内核时间 |
| `enable_BakaSU` | 布尔 | `true` | **总开关**：启用 BakaSU (KernelSU) 注入；关闭后 SUSFS 同步失效，且内核名标记为 `NoKSU` |
| `enable_susfs` | 布尔 | `true` | 启用 SUSFS（**依赖 `enable_BakaSU`**，总开关关闭时本项无效） |
| `enable_Adios` | 布尔 | `true` | 启用 ADIOS I/O 调度器 |
| `enable_droidspaces` | 选项 | `678` | Droidspaces 容器支持：`off` / `678`（推荐）/ `123` / `345`，非 `off` 时应用对应槽位的 kABI 补丁 |
| `enable_droidspaces_ntsync` | 布尔 | `true` | 启用 ntsync 驱动（**需 `enable_droidspaces` 不为 `off`**） |
| `enable_ReKernel` | 布尔 | `false` | 集成 Re-Kernel，并额外产出 Re-Kernel 模块压缩包 |
| `enable_bbg` | 布尔 | `false` | 开启 BBG 基带守护（Baseband-guard） |
| `enable_signed` | 布尔 | `true` | 使用 AOSP 测试密钥签名 boot |

> 💡 `enable_droidspaces` 是**选项类型而非布尔开关**，所以判断一律是「不为 `off`」。
> `off` 之外的三个值是要应用的 kABI 补丁槽位，`678` 为上游推荐值。

### 🧩 内核版本矩阵

| 内核版本 | `os_patch_level` | 源码 / boot.img 来源 release |
|----------|------------------|------------------------------|
| `6.6.56` | `2024-11` | `TB322FC_1.1.11.120` |
| `6.6.89` | `2025-06` | `ColorOS_16_400_OPD_Boot` |
| `6.6.102` | `2025-10` | `TB322FC_1.5.10.096` |
| `6.6.118` | `2026-01` | `TB322_ZUXOS_1.5.10.183` |

`os_patch_level` 同时决定下载哪份 `common.tar.gz`。两者都由 `kernel_version` 输入驱动，
所以**选错版本等于选了另一套源码与配置**。

---

## 📦 编译产物

| 产物 | 内容 | 产出条件 |
|------|------|----------|
| `kernel_image_<version>` | 原始内核镜像 `Image`（重命名为 `kernel`） | 始终产出 |
| `Boot_<KSUVER>_<version>_<KSU_NAME>_AOSP_Signed` | 替换内核并重新签名的 `boot.img` | `enable_signed` |
| `Re-Kernel-v<版本号>` | Re-Kernel 模块压缩包（取自上游 release） | `enable_ReKernel` |

其中 `KSU_NAME` 是 `BakaSU`，或关闭 KSU 注入时的 `NoKSU`；`KSUVER` 在关闭注入时为 `none`。

> ❗ **本仓库不产出 AnyKernel3 刷入包**，只产出上述原始 `Image`、重打包签名后的 `boot.img`
> 和 Re-Kernel 模块。刷入需自备工具（如 `fastboot flash boot`），并自行确认与设备分区匹配。

---

## 💾 ccache 缓存与时间劫持

### 三级回退

缓存按 `kernel_version` + 变体后缀（启用 Droidspaces 时加 `-ds`）分桶：

1. `actions/cache/restore` 取本仓库缓存
2. 未命中时，从公共缓存仓库（默认 `showdo/Public_Ccache` 的 `exported-ccaches` release）
   按资产名匹配公共缓存
3. 仍未命中则空缓存开跑

**回写规则**：仅在构建耗时 > 480 秒、或本次没命中任何缓存时才保存；
耗时 > 480 秒时还会顺手删除同前缀的旧缓存，避免占用过多 GitHub 存储空间。

### 时间劫持（重要）

构建阶段通过 `LD_PRELOAD` 加载 `libfakestat.so` + `libfaketimeMT.so`，把编译器看到的
**文件时间戳和当前时间固定在 `2025-05-25`**，同时 `KBUILD_BUILD_TIMESTAMP` 取自版本矩阵里的
内核时间。这是 ccache 能稳定命中的前提。

> ⚠️ 因此**构建日志里的 `date` 和 `stat` 输出是假的**，这是设计而非故障。

### 可覆盖的仓库变量

下列来源可通过仓库的 **Variables** 覆盖，fork 后想换源应改 repository variables 而非改工作流：

| 变量 | 默认值 | 作用 |
|------|--------|------|
| `BOOT_SIGNER_REPO` | `showdo/AOSP-Testkey-BootSigner` | 签名**工具**来源仓库 |
| `BOOT_SIGNER_REF` | `main` | 签名工具引用 |
| `BOOT_IMG_REPO` | 同签名器仓库 | 待签名**源 boot.img** 来源仓库 |
| `BOOT_IMG_TAG` | 空（回退到本版本 `releases`） | 源 boot.img 的 release tag |
| `PUBLIC_CACHE_REPO` | `showdo/Public_Ccache` | 公共缓存仓库 |
| `PUBLIC_CACHE_TAG` | `exported-ccaches` | 公共缓存 release 标签 |

---

## 🧩 内核补丁来源

**仓库内补丁只剩一个**，构建时在源码下载完成后自动应用：

| 补丁 | 作用 | 应用条件 |
|------|------|----------|
| [`patch/adios_ioscheduler.patch`](patch/adios_ioscheduler.patch) | ADIOS 自适应 I/O 调度器 | `enable_Adios` |

**其余改动都在构建时从外部现拉，仓库里不留副本**：

| 内容 | 来源 | 门控 |
|------|------|------|
| BakaSU 内核侧注入 | `Baka-SU/BakaSU` 的 `kernel/setup.sh` | `enable_BakaSU` |
| SUSFS 内核补丁 | `cctv18/susfs4oki`（`oki-android15-6.6` 分支） | `enable_BakaSU` **且** `enable_susfs` |
| hide_stuff 补丁 | `cctv18/oppo_oplus_realme_sm8750` | 同上 |
| Droidspaces kABI 补丁 | `ravindu644/Droidspaces-OSS` → `Documentation/resources/kernel-patches/GKI` | `enable_droidspaces != off` |
| ntsync 补丁 ×2 | `Goldzxcbug/Droidspaces_Kernel_patch` 的 `NTsync/` | 另需 `enable_droidspaces_ntsync` |
| zram lz4 / zstd 补丁 | `cctv18/oppo_oplus_realme_sm8750` 的 `zram_patch` | 始终应用 |
| BBG 基带守护 | `vc-teahouse/Baseband-guard` 的 `setup.sh` | `enable_bbg` |

> 💡 **补丁跟上游走，配置清单不跟。** 补丁是构建时现拉的，上游改了会自动生效；
> 但内核配置清单是手工维护的，上游新增依赖或调整推荐项时本仓库**不会自动跟上，也不会报错**。

### 🔨 工具链

`LLVM=1` + **LLVM Clang 18**（`clang-r510928`）+ LLD，`ARCH=arm64`，
`CROSS_COMPILE=aarch64-linux-gnu-`，`O=out` 编译 `Image`。

---

## 🐳 Droidspaces 支持

`enable_droidspaces` 只做**内核侧准备** —— 产出的是「支持 Droidspaces 的内核」，
**不是**注入了 Droidspaces 的内核。Droidspaces 本体是运行在设备上的用户态程序
（[ravindu644/Droidspaces-OSS](https://github.com/ravindu644/Droidspaces-OSS)），需在设备上另行安装。

该开关实际做三件事：

1. 应用上游的 SYSVIPC kABI 修复补丁 —— 槽位就是 `enable_droidspaces` 的取值
2. 可选应用 **ntsync** 补丁（NT 同步原语，服务于 Wine / Proton 类场景）
   —— 注意上游 Droidspaces 文档中**根本没有 ntsync**，这是本仓库额外添加的
3. 写入内核配置，包括 `CONFIG_SYSVIPC`、`CONFIG_POSIX_MQUEUE`、`CONFIG_IPC_NS`、
   `CONFIG_PID_NS`、`CONFIG_DEVTMPFS` 及一组 netfilter / ipset 项

> ⚠️ **隐式依赖**：`init/Kconfig` 里 `IPC_NS` 依赖 `(SYSVIPC || POSIX_MQUEUE)`，
> 所以这两项**至少要留一个** —— 两个都关掉时 `IPC_NS` 会被 Kconfig 静默解析回 `n`，
> 构建不报任何错，只会在设备上撞问题。

### ✅ 验收方式

真正的验收手段只有上游 App 自带的检查器：设备上 **Settings → Requirements → Check Requirements**，
或终端执行：

```bash
su -c droidspaces check
```

逐项比对配置清单只能证明「照着文档抄齐了」，不能证明「内核真的够用」。

---

## 🔏 AOSP 签名（`enable_signed`）

签名步骤把新编出来的 `Image` 塞进一个**现成的 boot.img** 再重新签名，
所以有两组独立来源：签名**工具**（`BOOT_SIGNER_*`）与待签名**源 boot.img**（`BOOT_IMG_*`）。

签名链路：

```
magiskboot repack（替换内核） → python3 rebuild_avb.py --chained-mode（重新签名） → 校验产物存在
```

任一环节失败即 `exit 1`，不留半个产物。`6.6.118` 在版本矩阵里覆盖了 `BOOT_IMG_REPO` / `BOOT_IMG_TAG`，
改用自有 release 上传的 `boot.img`。

---

## 🐛 故障排除

### 1. 编译失败

**可能原因**：网络问题导致依赖 / 源码 / 补丁下载失败；上游源码或补丁上下文变化导致补丁应用失败；磁盘或内存不足。

**解决方案**：
- 检查 Actions 日志中的 `::error::` 标记与报错步骤
- 尝试重新运行工作流（补丁应用是有意容忍失败的，重跑通常能覆盖已应用的补丁）
- 清理缓存后重新编译

### 2. 找不到产物 / 产物名与预期不符

产物名里的 `KSUVER` / `KSU_NAME` 由 `enable_BakaSU` 决定：关闭注入时分别为 `none` 与 `NoKSU`。
另外，**Artifacts 必须在运行页面下载**，工作流运行记录过期后产物会一并消失。

### 3. 刷入后异常

本仓库**不产出刷入包**，请确认你用的是与设备分区匹配的镜像；出问题优先回到上一个可用 boot 镜像。
本 fork 的改动未经上游审阅，遇到异常也请先排除「是本 fork 的改动导致」这一可能。

### 日志分析

1. 进入 **Actions** 页面
2. 点击对应的 workflow 运行记录
3. 展开各个步骤查看详细输出
4. 重点关注报错步骤

常见日志标记：

```
[INFO]    - 一般信息
[SUCCESS] - 成功完成
[ERROR]   - 错误（需立即处理）
::warning:: - 警告，通常可继续
```

---

## 🔗 相关资源

### 相关项目

- [上游项目：qdykernel/Build_Lenovo_sm8750](https://github.com/qdykernel/Build_Lenovo_sm8750)
- [OnePlus/Realme 内核编译](https://github.com/qdykernel/Build_Oneplus_Realme_Action)
- [BakaSU 项目](https://github.com/Baka-SU/BakaSU)（即改名后的 ReSukiSU）
- [Droidspaces 项目](https://github.com/ravindu644/Droidspaces-OSS)
- [SUSFS 补丁源](https://github.com/cctv18/susfs4oki)
- [Re-Kernel 项目](https://github.com/Sakion-Team/Re-Kernel)

### 社区支持

- **Telegram 频道**: [@qdykernel](https://t.me/qdykernel)（上游）
- **酷安主页**: [@qdykernel](http://www.coolapk.com/u/1624571)（上游）
- **上游 GitHub Issues**: [qdykernel/Build_Lenovo_sm8750](https://github.com/qdykernel/Build_Lenovo_sm8750/issues)
- **本 fork 仓库**: [mineextremely/Build_Lenovo_sm8750](https://github.com/mineextremely/Build_Lenovo_sm8750)

> 上文徽章与频道链接均指向上游，作为**出处标注刻意保留**。
> 但请注意：**上游没有义务解答本 fork 的问题**，与上游行为不一致的现象请勿拿去上游提问。

---

## 📜 许可证

本项目采用 **GPL-3.0** 许可证。

```
Copyright (C) 2024 qdykernel

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <https://www.gnu.org/licenses/>.
```

---

## 📈 项目统计

[![Star History Chart](https://api.star-history.com/svg?repos=qdykernel/Build_Lenovo_sm8750&type=Date)](https://star-history.com/#qdykernel/Build_Lenovo_sm8750&Date)

---

## 📞 联系方式

上游联系方式（本 fork 保留其作为出处标注）：

- **Telegram**: [@qdykernel](https://t.me/qdykernel)
- **GitHub Issues**: [qdykernel/Build_Lenovo_sm8750](https://github.com/qdykernel/Build_Lenovo_sm8750/issues)
- **酷安**: 私信 [@Q1udaoyu](http://www.coolapk.com/u/1624571)

---

**维护状态**: 🟡 个人 fork，按需更新，不承诺与上游同步进度
