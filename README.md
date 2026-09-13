# Lenovo SM8750 内核构建项目

**`简体中文`** | [English](README-en.md)<br>

[![GitHub](https://img.shields.io/badge/-GitHub|@qdykernel-181717?logo=github&logoColor=white&style=flat-square)](https://github.com/qdykernel/Build_Lenovo_sm8750)
[![Telegram](https://img.shields.io/badge/Telegram-频道-blue.svg?logo=telegram)](https://t.me/qdykernel)
[![酷安|主页](https://img.shields.io/badge/酷安|主页-3DDC84?style=flat-square&logo=android&logoColor=white)](http://www.coolapk.com/u/1624571)
[![Workflow Status](https://img.shields.io/github/actions/workflow/status/mineextremely/Build_Lenovo_sm8750/Build_Lenovo_sm8750.yml?label=Build&logo=github-actions&style=flat-square)](https://github.com/mineextremely/Build_Lenovo_sm8750/actions)
<br>

---

## 📖 项目简介

本项目提供基于 **GitHub Actions** 的自动化内核编译工作流，支持 **Lenovo** 搭载 **SM8750** 平台的设备。通过高度集成的脚本，实现一键编译包含 **(Re)SukiSU**、**SUSFS**、**ADIOS调度** 等功能的 GKI 内核。

### ✨ 主要特性

- 🚀 **全自动化编译** - 基于 GitHub Actions，无需本地环境
- 🔧 **ReSukiSU 集成** - 可开关的 ReSukiSU (KernelSU) 注入，联动 SUSFS / KPM
- ⚡ **性能优化** - 集成ADIOS I/O调度补丁
- 💾 **ccache 缓存** - 智能缓存管理，首次编译使用公共缓存提速 50%，二次编译提速 80%
- 📦 **开箱即用** - 自动重打包 boot.img 并使用 AOSP 测试密钥签名

---

## 🎯 快速开始

### 方式一：GitHub Actions 云编译（推荐）

#### 步骤 1. Fork 本仓库

点击仓库右上角的 **Fork** 按钮，将本仓库复制到你自己的 GitHub 账户。

#### 步骤 2. 运行工作流

1. 进入你 Fork 的仓库
2. 点击 **Actions** 标签页
3. 选择对应的工作流（见下方说明）
4. 点击 **Run workflow** 按钮
5. 填写编译参数
6. 等待编译完成（首次构建约12分钟）
7. 在 **Artifacts** 中下载编译产物

### 📋 可用工作流

| 工作流 | 说明 | 适用场景 |
|--------|------|----------|
| [Build_Lenovo_sm8750.yml](.github/workflows/Build_Lenovo_sm8750.yml) | 完整内核编译（含 KSU / SUSFS / ADIOS / Droidspaces 等） | 编译并集成所需功能 |
| [Clear_All_Workflow.yml](.github/workflows/Clear_All_Workflow.yml) | 清理工作流运行记录 | 保证 Actions 界面整洁 |

---

## 🔧 高级功能

### 自定义编译选项

在工作流运行时，您可以配置以下参数：

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `kernel_version` | 选项 | `6.6.89` | 内核版本：`6.6.56` / `6.6.89` / `6.6.102` / `6.6.118` |
| `custom_kernel_suffix` | 文本 | 空 | 自定义内核名称，留空则使用官方内核名 |
| `custom_kernel_time` | 文本 | 空 | 自定义构建时间，留空则使用官方内核时间 |
| `enable_ReSukiSU` | 布尔 | `true` | 启用 ReSukiSU (KernelSU) 注入；**关闭后 SUSFS / KPM 同步失效** |
| `enable_susfs` | 布尔 | `true` | 启用 SUSFS（依赖 ReSukiSU） |
| `enable_kpm` | 布尔 | `true` | 启用 KPM（依赖 ReSukiSU） |
| `enable_ReKernel` | 布尔 | `true` | 集成 Re-Kernel，并额外产出 Re-Kernel 模块压缩包 |
| `enable_Adios` | 布尔 | `true` | 启用 ADIOS I/O 调度器 |
| `enable_droidspaces` | 布尔 | `false` | Droidspaces 支持（含 SYSVIPC kABI 修复与 ntsync 驱动） |
| `enable_bbg` | 布尔 | `false` | 开启 BBG 基带守护 |
| `enable_signed` | 布尔 | `true` | 使用 AOSP 测试密钥签名 boot |

#### 使用公共缓存

项目会自动从公共缓存仓库下载预编译缓存，首次编译也能享受加速效果。

#### 清理旧缓存

当构建耗时超过 8 分钟时，工作流会在保存新缓存后自动删除同机型同变体的旧缓存，避免占用过多 GitHub 存储空间。缓存按内核版本分桶，启用 Droidspaces 时使用独立的 `-ds` 变体，不同功能组合的缓存不会互相污染。

### 内核补丁

`patch/` 目录存放本仓库自行维护的内核补丁，构建时在源码下载完成后自动应用：

| 补丁 | 作用 | 应用条件 |
|------|------|----------|
| `adios_ioscheduler.patch` | ADIOS 自适应 I/O 调度器 | `enable_Adios` |
| `GKI-6.6-sysvipc_kabi_3_4_5.patch` | 修复 SYSVIPC 结构体破坏 GKI kABI 的问题 | `enable_droidspaces` |
| `ntsync_base.patch` | NT 同步原语驱动 `ntsync` | `enable_droidspaces` |
| `ntsync_compat_6.6.patch` | 将 `ntsync` 接入 6.6 的 Kconfig / Makefile | `enable_droidspaces` |

---

## 🐛 故障排除

### 常见问题

#### 1. 编译失败

**可能原因**:
- 网络问题导致依赖下载失败
- 源代码存在冲突
- 内存不足

**解决方案**:
- 检查 Actions 日志中的错误信息
- 尝试重新运行工作流
- 清理缓存后重新编译


### 日志分析

#### 查看编译日志

1. 进入 **Actions** 页面
2. 点击对应的 workflow 运行记录
3. 展开各个步骤查看详细输出
4. 重点关注报错步骤

#### 关键日志标记

```
[INFO]    - 一般信息
[SUCCESS] - 成功完成
[ERROR]   - 错误（需立即处理）
```

---

## 🔗 相关资源

### 相关项目

- [OnePlus/Realme 内核编译](https://github.com/qdykernel/Build_Oneplus_Realme_Action)
- [SukiSU-Ultra 项目](https://github.com/SukiSU-Ultra/SukiSU-Ultra)
- [ReSukiSU 项目](https://github.com/ReSukiSU/ReSukiSU)

### 社区支持

- **Telegram 频道**: [@qdykernel](https://t.me/qdykernel)
- **酷安主页**: [@qdykernel](http://www.coolapk.com/u/1624571)
- **GitHub Issues**: [提交问题](https://github.com/qdykernel/Build_Lenovo_sm8750/issues)

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

如有问题或建议，请通过以下方式联系：

- **Telegram**: [@qdykernel](https://t.me/qdykernel)
- **GitHub Issues**: [新建 Issue](https://github.com/qdykernel/Build_Lenovo_sm8750/issues)
- **酷安**: 私信 [@Q1udaoyu](http://www.coolapk.com/u/1624571)

---

**维护状态**: 🟢 活跃维护中
