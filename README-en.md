# Lenovo SM8750 Kernel Build Project

[简体中文](README.md) | **`English`**<br>

> [!WARNING]
> **This is a personal fork, not the upstream project**
>
> This repository is a personal fork of
> [qdykernel/Build_Lenovo_sm8750](https://github.com/qdykernel/Build_Lenovo_sm8750)
> ([mineextremely/Build_Lenovo_sm8750](https://github.com/mineextremely/Build_Lenovo_sm8750)), and it **has already
> diverged from upstream** in its feature set: the BakaSU rename, removal of KPM, Droidspaces
> switched to a slot-based patch scheme, `CONFIG_USER_NS` enabled on the `dev` branch, and more.
>
> - These changes have **not been reviewed by upstream**, some are personal experiments, and they
>   **may contain unstable features**
> - Artifact behaviour, parameter names, and log output **may all differ from upstream** — do not
>   compare them against upstream docs directly when troubleshooting
> - Builds are for personal use and testing only. **Flashing is at your own risk**
>
> Upstream project: <https://github.com/qdykernel/Build_Lenovo_sm8750>

[![GitHub](https://img.shields.io/badge/-GitHub|@qdykernel-181717?logo=github&logoColor=white&style=flat-square)](https://github.com/qdykernel/Build_Lenovo_sm8750)
[![Telegram](https://img.shields.io/badge/Telegram-Channel-blue.svg?logo=telegram)](https://t.me/qdykernel)
[![CoolApk|Profile](https://img.shields.io/badge/CoolApk%7CProfile-3DDC84?style=flat-square&logo=android&logoColor=white)](http://www.coolapk.com/u/1624571)
[![Workflow Status](https://img.shields.io/github/actions/workflow/status/mineextremely/Build_Lenovo_sm8750/Build_Lenovo_sm8750.yml?label=Build&logo=github-actions&style=flat-square)](https://github.com/mineextremely/Build_Lenovo_sm8750/actions)
<br>

---

## 📖 Introduction

This project provides an automated kernel compilation workflow based on **GitHub Actions**, supporting
**Lenovo** devices equipped with the **SM8750** platform. Through highly integrated scripts, it enables
one-click compilation of GKI kernels featuring **BakaSU (KernelSU)**, **SUSFS**, **ADIOS IOScheduler**,
**Droidspaces container support**, **Re-Kernel**, and more.

**This repository is an orchestration repo — it contains no kernel source.** Each build downloads the
`common.tar.gz` matching the selected `os_patch_level` from `showdo/Build_Lenovo_sm8750` releases,
compiles it, and discards it. The repo itself holds only the workflows and a single patch file.

### ✨ Key Features

- 🚀 **Fully Automated Compilation** - Based on GitHub Actions, no local environment required
- 🔧 **BakaSU Integration** - Toggleable BakaSU (KernelSU) injection, linked with SUSFS; when disabled the kernel is tagged `NoKSU`
- 🫥 **SUSFS Support** - Kernel-level hiding (sus_path / sus_mount / sus_map / open_redirect, etc.)
- ⚡ **Performance Optimization** - ADIOS I/O scheduler patch, plus lz4 1.10.0 / zstd 1.5.7 patches for zram
- 🐳 **Droidspaces Support** - SYSVIPC kABI fix (selectable slot) with optional ntsync driver
- 🧩 **Re-Kernel Integration** - Toggleable kernel module injection, plus a Re-Kernel module zip artifact
- 💾 **Three-tier ccache Fallback** - Repo cache → public cache → empty cache, with time hijacking for reproducibility and high hit rates
- 📦 **Ready to Use** - Automatically repacks boot.img and signs it with the AOSP test key

---

## 🆚 Differences from Upstream

Since forking, this repository has made the following **structural** changes — know them before use:

| Item | Upstream | This fork |
|------|----------|-----------|
| KSU vendor naming | `ReSukiSU` / `enable_ReSukiSU` | **`BakaSU` / `enable_BakaSU`** (same project, renamed) |
| KPM | Optional `enable_kpm` switch | **Removed entirely** — input, `CONFIG_KPM`, and patching step all gone |
| Droidspaces | Bundled patch files | Pulled from upstream at build time, **switched to slots** `off` / `678` / `123` / `345` |
| Kernel patches | Multiple patches under `patch/` | **Only `adios_ioscheduler.patch` remains**; the rest are fetched at build time |
| `dev` branch | No stable diff convention vs. main | **Its only difference is `CONFIG_USER_NS`** (see below) |
| Source boot.img | Fixed repo and tag | Overridable per kernel version (`6.6.118` uses its own release) |
| Kernel watermark | Upstream signature | `CONFIG_KSU_FULL_NAME_FORMAT` carries `GitHub@mineextremely` |

> ⚠️ Upstream is still actively maintained; the changes in this fork **are not contributed back**.
> If you want to follow upstream, use the
> [upstream repository](https://github.com/qdykernel/Build_Lenovo_sm8750) directly.

---

## 🚦 Branches: `main` and `dev`

The **only difference between the two branches is `CONFIG_USER_NS`**: enabled on `dev`, disabled on
`main`. Everything else is identical.

- `CONFIG_USER_NS` is the optional item upstream's Droidspaces checklist labels *"Fix for docker unsafe
  procfs error"* — without it, running Docker inside the container hits a procfs error.
- But user namespaces have historically been a hotspot for local privilege-escalation bugs, and Android
  GKI deliberately disables them by default. **So it is enabled on `dev` only** — a deliberate
  trade-off for nested-container capability, not an oversight.

**Use `main` if you want stability**; only consider `dev` if you genuinely need nested Docker inside
containers.

---

## 🎯 Quick Start

### Step 1. Fork This Repository

Click the **Fork** button in the upper right corner of the repository to copy it to your own GitHub account.

### Step 2. Run the Workflow

1. Enter your forked repository
2. Click the **Actions** tab
3. Select the `Build_Lenovo_sm8750` workflow
4. Click the **Run workflow** button on the right
5. Fill in the compilation parameters (see the table below)
6. Wait for the build to finish (duration depends on cache hits — it is usually much faster once a public cache is matched)
7. Download the artifacts from **Artifacts** on the run page

### 📋 Available Workflows

| Workflow | Description | Use Case |
|----------|-------------|----------|
| [Build_Lenovo_sm8750.yml](.github/workflows/Build_Lenovo_sm8750.yml) | Full kernel compilation (BakaSU / SUSFS / ADIOS / Droidspaces / Re-Kernel, etc.) | Compile and integrate the features you need |
| [Clear_All_Workflow.yml](.github/workflows/Clear_All_Workflow.yml) | Clear workflow run records | Keep the Actions page tidy |

> ⚠️ The workflow must already exist on the **default branch** before it can be triggered manually.

---

## 🔧 Compilation Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `kernel_version` | choice | `6.6.89` | Kernel version: `6.6.56` / `6.6.89` / `6.6.102` / `6.6.118` |
| `custom_kernel_suffix` | string | empty | Custom kernel name; empty uses the official one |
| `custom_kernel_time` | string | empty | Custom build time; empty uses the official one |
| `enable_BakaSU` | boolean | `true` | **Master switch**: enable BakaSU (KernelSU) injection; disabling it also disables SUSFS and tags the kernel `NoKSU` |
| `enable_susfs` | boolean | `true` | Enable SUSFS (**requires `enable_BakaSU`**; ignored when the master switch is off) |
| `enable_Adios` | boolean | `true` | Enable the ADIOS I/O scheduler |
| `enable_droidspaces` | choice | `678` | Droidspaces container support: `off` / `678` (recommended) / `123` / `345`; anything but `off` applies that slot's kABI patch |
| `enable_droidspaces_ntsync` | boolean | `true` | Enable the ntsync driver (**requires `enable_droidspaces` not `off`**) |
| `enable_ReKernel` | boolean | `false` | Integrate Re-Kernel and additionally publish a Re-Kernel module zip |
| `enable_bbg` | boolean | `false` | Enable the BBG baseband guard (Baseband-guard) |
| `enable_signed` | boolean | `true` | Sign boot with the AOSP test key |

> 💡 `enable_droidspaces` is a **choice, not a boolean**, so the checks are always "not `off`".
> The three values other than `off` are the kABI patch slots to apply; `678` is upstream's recommended one.

### 🧩 Kernel Version Matrix

| Kernel version | `os_patch_level` | Source / boot.img release |
|----------------|------------------|---------------------------|
| `6.6.56` | `2024-11` | `TB322FC_1.1.11.120` |
| `6.6.89` | `2025-06` | `ColorOS_16_400_OPD_Boot` |
| `6.6.102` | `2025-10` | `TB322FC_1.5.10.096` |
| `6.6.118` | `2026-01` | `TB322_ZUXOS_1.5.10.183` |

`os_patch_level` also decides which `common.tar.gz` gets downloaded. Both are driven by the
`kernel_version` input, so **picking the wrong version means picking a different source tree and config**.

---

## 📦 Build Artifacts

| Artifact | Contents | Produced when |
|----------|----------|---------------|
| `kernel_image_<version>` | Raw kernel image `Image` (renamed to `kernel`) | Always |
| `Boot_<KSUVER>_<version>_<KSU_NAME>_AOSP_Signed` | `boot.img` with the new kernel repacked and re-signed | `enable_signed` |
| `Re-Kernel-v<version>` | Re-Kernel module zip (taken from the upstream release) | `enable_ReKernel` |

`KSU_NAME` is `BakaSU`, or `NoKSU` when KSU injection is disabled; `KSUVER` is `none` in that case.

> ❗ **This repository does not produce an AnyKernel3 flashing package.** It only produces the raw
> `Image`, the repacked and signed `boot.img`, and the Re-Kernel module. Flashing is up to you
> (e.g. `fastboot flash boot`), and you must confirm the image matches your device partitions.

---

## 💾 ccache and Time Hijacking

### Three-tier fallback

Caches are bucketed by `kernel_version` plus a variant suffix (`-ds` when Droidspaces is enabled):

1. `actions/cache/restore` pulls this repository's cache
2. On a miss, a public cache is matched by asset name from the public cache repo (by default the
   `exported-ccaches` release of `showdo/Public_Ccache`)
3. On another miss, the build starts from an empty cache

**Save rules**: the cache is saved only when the build took longer than 480 seconds, or when no cache
was matched at all. When the build exceeds 480 seconds, old caches sharing the same prefix are deleted
as well, to avoid using up too much GitHub storage.

### Time hijacking (important)

During the build, `libfakestat.so` + `libfaketimeMT.so` are loaded via `LD_PRELOAD`, pinning the
**file timestamps and current time seen by the compiler to `2025-05-25`**, while
`KBUILD_BUILD_TIMESTAMP` comes from the kernel time in the version matrix. This is what makes stable
ccache hits possible.

> ⚠️ As a result, **the `date` and `stat` output in the build log is fake** — that is by design, not a failure.

### Overridable repository variables

The following sources can be overridden through the repository's **Variables**. After forking, change
repository variables rather than editing the workflow:

| Variable | Default | Purpose |
|----------|---------|---------|
| `BOOT_SIGNER_REPO` | `showdo/AOSP-Testkey-BootSigner` | Signing **tool** source repo |
| `BOOT_SIGNER_REF` | `main` | Signing tool ref |
| `BOOT_IMG_REPO` | same as signer repo | Source **boot.img** repo to sign |
| `BOOT_IMG_TAG` | empty (falls back to this version's `releases`) | Release tag of the source boot.img |
| `PUBLIC_CACHE_REPO` | `showdo/Public_Ccache` | Public cache repo |
| `PUBLIC_CACHE_TAG` | `exported-ccaches` | Public cache release tag |

---

## 🧩 Kernel Patch Sources

**Only one patch lives in the repository**; it is applied automatically once the source tree is downloaded:

| Patch | Purpose | Applied when |
|-------|---------|--------------|
| [`patch/adios_ioscheduler.patch`](patch/adios_ioscheduler.patch) | ADIOS adaptive I/O scheduler | `enable_Adios` |

**Everything else is fetched from external sources at build time — no copies are kept in the repo**:

| Content | Source | Gated by |
|---------|--------|----------|
| BakaSU kernel-side injection | `Baka-SU/BakaSU` → `kernel/setup.sh` | `enable_BakaSU` |
| SUSFS kernel patch | `cctv18/susfs4oki` (`oki-android15-6.6` branch) | `enable_BakaSU` **and** `enable_susfs` |
| hide_stuff patch | `cctv18/oppo_oplus_realme_sm8750` | same as above |
| Droidspaces kABI patch | `ravindu644/Droidspaces-OSS` → `Documentation/resources/kernel-patches/GKI` | `enable_droidspaces != off` |
| ntsync patches ×2 | `Goldzxcbug/Droidspaces_Kernel_patch` → `NTsync/` | also needs `enable_droidspaces_ntsync` |
| zram lz4 / zstd patches | `cctv18/oppo_oplus_realme_sm8750` → `zram_patch` | Always applied |
| BBG baseband guard | `vc-teahouse/Baseband-guard` → `setup.sh` | `enable_bbg` |

> 💡 **Patches follow upstream; the config list does not.** Patches are pulled fresh at build time, so
> upstream changes take effect automatically. The kernel config list, however, is maintained by hand —
> when upstream adds a dependency or changes a recommendation, this repo **will not follow and will not
> error out**.

### 🔨 Toolchain

`LLVM=1` with **LLVM Clang 18** (`clang-r510928`) + LLD, `ARCH=arm64`,
`CROSS_COMPILE=aarch64-linux-gnu-`, building `Image` with `O=out`.

---

## 🐳 Droidspaces Support

`enable_droidspaces` only does the **kernel-side preparation** — it produces a
"kernel that supports Droidspaces", **not** a kernel with Droidspaces baked in. Droidspaces itself is a
user-space program running on the device
([ravindu644/Droidspaces-OSS](https://github.com/ravindu644/Droidspaces-OSS)) and must be installed separately.

The switch does three things:

1. Applies upstream's SYSVIPC kABI fix patch — the slot is exactly the value of `enable_droidspaces`
2. Optionally applies the **ntsync** patches (NT synchronization primitives, for Wine / Proton-like
   workloads) — note that upstream's Droidspaces docs contain **no mention of ntsync**; it is an extra
   added by this repo
3. Writes kernel config, including `CONFIG_SYSVIPC`, `CONFIG_POSIX_MQUEUE`, `CONFIG_IPC_NS`,
   `CONFIG_PID_NS`, `CONFIG_DEVTMPFS` and a set of netfilter / ipset options

> ⚠️ **Implicit dependency**: in `init/Kconfig`, `IPC_NS` depends on `(SYSVIPC || POSIX_MQUEUE)`, so at
> least one of those two must stay enabled — turning off both makes Kconfig silently resolve `IPC_NS`
> back to `n` with no build error, and you only find out on the device.

### ✅ Verification

The only real acceptance test is upstream's own checker: on the device, go to
**Settings → Requirements → Check Requirements**, or run:

```bash
su -c droidspaces check
```

Checking the config list item by item only proves you "copied the docs"; it does not prove the kernel
is actually sufficient.

---

## 🔏 AOSP Signing (`enable_signed`)

The signing step drops the freshly built `Image` into an **existing boot.img** and re-signs it, so
there are two independent sources: the signing **tool** (`BOOT_SIGNER_*`) and the source **boot.img**
(`BOOT_IMG_*`).

The chain:

```
magiskboot repack (replace kernel) → python3 rebuild_avb.py --chained-mode (re-sign) → verify artifact exists
```

Any step failing means `exit 1` — no half-finished artifact is left behind. For `6.6.118` the version
matrix overrides `BOOT_IMG_REPO` / `BOOT_IMG_TAG` to use a `boot.img` uploaded to this fork's own release.

---

## 🐛 Troubleshooting

### 1. Compilation Failure

**Possible causes**: network issues downloading dependencies / sources / patches; upstream source or
patch context changes breaking a patch; insufficient disk or memory.

**Solutions**:
- Check the Actions log for `::error::` markers and the failing step
- Try re-running the workflow (patch application is intentionally tolerant of failure, so a re-run
  usually covers already-applied patches)
- Clean the cache and rebuild

### 2. Missing or Unexpectedly Named Artifacts

`KSUVER` / `KSU_NAME` in the artifact name are driven by `enable_BakaSU`: with injection disabled they
become `none` and `NoKSU`. Also, **artifacts must be downloaded from the run page** — they disappear
once the workflow run expires.

### 3. Something Is Wrong After Flashing

This repo **does not produce a flashing package**; make sure the image matches your device partitions,
and fall back to the last known-good boot image if things go wrong. This fork's changes have not been
reviewed by upstream — when something misbehaves, first rule out "this fork's changes caused it".

### Log Analysis

1. Go to the **Actions** page
2. Click on the corresponding workflow run
3. Expand each step to view detailed output
4. Pay special attention to the failing steps

Common log markers:

```
[INFO]      - General information
[SUCCESS]   - Completed successfully
[ERROR]     - Error (requires immediate attention)
::warning:: - Warning; usually safe to continue
```

---

## 🔗 Related Resources

### Related Projects

- [Upstream: qdykernel/Build_Lenovo_sm8750](https://github.com/qdykernel/Build_Lenovo_sm8750)
- [OnePlus/Realme Kernel Compilation](https://github.com/qdykernel/Build_Oneplus_Realme_Action)
- [BakaSU Project](https://github.com/Baka-SU/BakaSU) (the renamed ReSukiSU)
- [Droidspaces Project](https://github.com/ravindu644/Droidspaces-OSS)
- [SUSFS Patch Source](https://github.com/cctv18/susfs4oki)
- [Re-Kernel Project](https://github.com/Sakion-Team/Re-Kernel)

### Community Support

- **Telegram Channel**: [@qdykernel](https://t.me/qdykernel) (upstream)
- **CoolApk Profile**: [@qdykernel](http://www.coolapk.com/u/1624571) (upstream)
- **Upstream GitHub Issues**: [qdykernel/Build_Lenovo_sm8750](https://github.com/qdykernel/Build_Lenovo_sm8750/issues)
- **This fork**: [mineextremely/Build_Lenovo_sm8750](https://github.com/mineextremely/Build_Lenovo_sm8750)

> The badges and channel links above intentionally point at upstream as an attribution of origin.
> Note, however, that **upstream has no obligation to answer questions about this fork** — do not take
> fork-specific behaviour there.

---

## 📜 License

This project is licensed under the **GPL-3.0** License.

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

## 📈 Project Statistics

[![Star History Chart](https://api.star-history.com/svg?repos=qdykernel/Build_Lenovo_sm8750&type=Date)](https://star-history.com/#qdykernel/Build_Lenovo_sm8750&Date)

---

## 📞 Contact

Upstream contact channels (kept here as attribution of origin):

- **Telegram**: [@qdykernel](https://t.me/qdykernel)
- **GitHub Issues**: [qdykernel/Build_Lenovo_sm8750](https://github.com/qdykernel/Build_Lenovo_sm8750/issues)
- **CoolApk**: DM [@Q1udaoyu](http://www.coolapk.com/u/1624571)

---

**Maintenance Status**: 🟡 Personal fork, updated as needed, no commitment to track upstream
