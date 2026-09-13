# Lenovo SM8750 Kernel Build Project

[简体中文](README.md) | **`English`**<br>

[![GitHub](https://img.shields.io/badge/-GitHub|@qdykernel-181717?logo=github&logoColor=white&style=flat-square)](https://github.com/qdykernel/Build_Lenovo_sm8750)
[![Telegram](https://img.shields.io/badge/Telegram-Channel-blue.svg?logo=telegram)](https://t.me/qdykernel)
[![CoolApk|Profile](https://img.shields.io/badge/CoolApk%7CProfile-3DDC84?style=flat-square&logo=android&logoColor=white)](http://www.coolapk.com/u/1624571)
[![Workflow Status](https://img.shields.io/github/actions/workflow/status/mineextremely/Build_Lenovo_sm8750/Build_Lenovo_sm8750.yml?label=Build&logo=github-actions&style=flat-square)](https://github.com/mineextremely/Build_Lenovo_sm8750/actions)
<br>

---

## 📖 Introduction

This project provides an automated kernel compilation workflow based on **GitHub Actions**, supporting **Lenovo** devices equipped with the **SM8750** platform. Through highly integrated scripts, it enables one-click compilation of GKI kernels featuring **(Re)SukiSU**, **SUSFS**, **ADIOS IOScheduler**, and other functionalities.

### ✨ Key Features

- 🚀 **Fully Automated Compilation** - Based on GitHub Actions, no local environment required
- 🔧 **ReSukiSU Integration** - Toggleable ReSukiSU (KernelSU) injection, linked with SUSFS / KPM
- ⚡ **Performance Optimization** - Integrated ADIOS I/Oscheduler patch
- 💾 **ccache Caching** - Intelligent cache management, 50% speed boost for first compilation with public cache, 80% for subsequent compilations
- 📦 **Ready to Use** - Automatically repacks boot.img and signs it with the AOSP test key

---

## 🎯 Quick Start

### Method 1: GitHub Actions Cloud Compilation (Recommended)

#### Step 1. Fork This Repository

Click the **Fork** button in the upper right corner of the repository to copy this repository to your own GitHub account.

#### Step 2. Run Workflow

1. Enter your forked repository
2. Click the **Actions** tab
3. Select the corresponding workflow (see instructions below)
4. Click the **Run workflow** button
5. Fill in the compilation parameters
6. Wait for compilation to complete (first build takes approximately 12 minutes)
7. Download the compiled artifacts from **Artifacts**

### 📋 Available Workflows

| Workflow | Description | Use Case |
|----------|-------------|----------|
| [Build_Lenovo_sm8750.yml](.github/workflows/Build_Lenovo_sm8750.yml) | Full kernel compilation (KSU / SUSFS / ADIOS / Droidspaces, etc.) | Compile and integrate the features you need |
| [Clear_All_Workflow.yml](.github/workflows/Clear_All_Workflow.yml) | Clear workflow run records | Keep the Actions page tidy |

---

## 🔧 Advanced Features

### Custom Compilation Options

During workflow execution, you can configure the following parameters:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `kernel_version` | choice | `6.6.89` | Kernel version: `6.6.56` / `6.6.89` / `6.6.102` / `6.6.118` |
| `custom_kernel_suffix` | string | empty | Custom kernel name; empty uses the official one |
| `custom_kernel_time` | string | empty | Custom build time; empty uses the official one |
| `enable_ReSukiSU` | boolean | `true` | Enable ReSukiSU (KernelSU) injection; **disabling it also disables SUSFS / KPM** |
| `enable_susfs` | boolean | `true` | Enable SUSFS (requires ReSukiSU) |
| `enable_kpm` | boolean | `true` | Enable KPM (requires ReSukiSU) |
| `enable_ReKernel` | boolean | `true` | Integrate Re-Kernel and additionally publish a Re-Kernel module zip |
| `enable_Adios` | boolean | `true` | Enable the ADIOS I/O scheduler |
| `enable_droidspaces` | boolean | `false` | Droidspaces support (includes the SYSVIPC kABI fix and the ntsync driver) |
| `enable_bbg` | boolean | `false` | Enable the BBG baseband guard |
| `enable_signed` | boolean | `true` | Sign boot with the AOSP test key |

#### Using Public Cache

The project will automatically download pre-compiled cache from the public cache repository, allowing acceleration even for the first compilation.

#### Cleaning Old Cache

When a build takes longer than 8 minutes, the workflow saves the new cache and then deletes old caches for the same device and variant, avoiding excessive GitHub storage usage. Caches are bucketed by kernel version, and enabling Droidspaces uses a separate `-ds` variant so different feature combinations never pollute each other.

### Kernel Patches

The `patch/` directory holds the kernel patches maintained by this repository. They are applied automatically once the source tree has been downloaded:

| Patch | Purpose | Applied when |
|-------|---------|--------------|
| `adios_ioscheduler.patch` | ADIOS adaptive I/O scheduler | `enable_Adios` |
| `GKI-6.6-sysvipc_kabi_3_4_5.patch` | Fix SYSVIPC structs breaking the GKI kABI | `enable_droidspaces` |
| `ntsync_base.patch` | NT synchronization primitive driver `ntsync` | `enable_droidspaces` |
| `ntsync_compat_6.6.patch` | Wire `ntsync` into the 6.6 Kconfig / Makefile | `enable_droidspaces` |

---

## 🐛 Troubleshooting

### Common Issues

#### 1. Compilation Failure

**Possible Causes**:
- Network issues causing dependency download failures
- Source code conflicts
- Insufficient memory

**Solutions**:
- Check error messages in Actions logs
- Try re-running the workflow
- Clean cache and recompile


### Log Analysis

#### Viewing Compilation Logs

1. Go to the **Actions** page
2. Click on the corresponding workflow run record
3. Expand each step to view detailed output
4. Pay special attention to error steps

#### Key Log Markers

```
[INFO]    - General information
[SUCCESS] - Successfully completed
[ERROR]   - Error (requires immediate attention)
```

---

## 🔗 Related Resources

### Related Projects

- [OnePlus/Realme Kernel Compilation](https://github.com/qdykernel/Build_Oneplus_Realme_Action)
- [SukiSU-Ultra Project](https://github.com/SukiSU-Ultra/SukiSU-Ultra)
- [ReSukiSU Project](https://github.com/ReSukiSU/ReSukiSU)

### Community Support

- **Telegram Channel**: [@qdykernel](https://t.me/qdykernel)
- **CoolApk Profile**: [@qdykernel](http://www.coolapk.com/u/1624571)
- **GitHub Issues**: [Submit Issue](https://github.com/qdykernel/Build_Lenovo_sm8750/issues)

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

If you have any questions or suggestions, please contact us through the following methods:

- **Telegram**: [@qdykernel](https://t.me/qdykernel)
- **GitHub Issues**: [Create New Issue](https://github.com/qdykernel/Build_Lenovo_sm8750/issues)
- **CoolApk**: DM [@Q1udaoyu](http://www.coolapk.com/u/1624571)

---

**Maintenance Status**: 🟢 Actively Maintained
