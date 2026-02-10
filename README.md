# MikroTik RouterOS UTM 虚拟机构建工具

## 项目概述

本项目通过 Pkl 配置语言自动化构建 UTM 虚拟机包，支持从 MikroTik 官方源或 mikrotik.ltd 镜像源下载 RouterOS CHR 镜像，生成可直接导入 UTM 的 `.utm` 虚拟机包。

主要特性：

- **自动化构建**：通过 Pkl 语言定义虚拟机配置，自动生成 UTM 所需的所有文件
- **双镜像源支持**：官方源 + 已修补源（mikrotik.ltd），满足不同需求
- **多架构支持**：x86_64 (Intel/AMD) 和 aarch64 (Apple Silicon)
- **多后端支持**：QEMU 和 Apple Virtualization Framework
- **定时任务**：自动检测上游版本更新并触发构建
- **智能清理**：自动清理失败的 workflow 运行记录

## 核心原理

### 1. Pkl 语言简介

Pkl 是一种配置即代码语言，支持：

- 类型安全的配置定义
- 模块化和继承机制
- 多文件输出支持
- 丰富的内置类型（DataSize、UUID 等）

### 2. UTM 虚拟机包结构

UTM 虚拟机是一个以 `.utm` 为后缀的包文件夹，包含：

```
<虚拟机名称>.utm/
├── config.plist      # UTM 配置文件（XML 格式）
├── config.pkl        # Pkl 原始配置（用于调试）
├── Data/
│   ├── <图标文件>     # SVG 图标
│   ├── <镜像名>.url  # 镜像下载 URL
│   ├── efi_vars.fd.localcp  # Apple 虚拟化所需
│   └── qdisk*.size   # 额外磁盘大小（可选）
```

### 3. 固件下载流程

**官方源**：
1. `CHR.pkl` 中的 `RouterOSVersion` 类定义版本信息
2. 从 `upgrade.mikrotik.com/routeros/NEWESTa7.<channel>` 获取指定通道的最新版本号
3. 从 `download.mikrotik.com` 下载官方 RouterOS CHR 镜像

**PATCHED 源**：
1. `Patched.pkl` 中的 `PatchedVersion` 类定义版本信息
2. 从 `mikrotik.ltd` 获取版本信息
3. 从 `github.com/elseif/MikroTikPatch/releases` 下载已修补镜像

**镜像类型**：
- `chr-<版本>.img.zip` - UEFI 模式镜像
- `chr-<版本>-<arch>.img.zip` - 带架构后缀的镜像

## 项目结构

```
mikropkl/
├── Pkl/                    # 核心 Pkl 模块
│   ├── CHR.pkl            # RouterOS 官方版本和图标定义
│   ├── Patched.pkl        # RouterOS 已修补版本定义
│   ├── URL.pkl            # 下载资源类定义
│   ├── SVG.pkl            # SVG 图标基类
│   ├── UTM.pkl            # UTM 配置类型定义
│   ├── Randomish.pkl      # 随机数据生成工具
│   ├── utmzip.pkl         # UTM 包生成核心逻辑
│   ├── chr-version.pkl     # 官方版本查询脚本
│   └── patched-version.pkl     # 已修补版本查询脚本
├── Templates/              # 虚拟机模板
│   ├── chr.utmzip.pkl     # CHR 虚拟机模板（官方）
│   ├── chr_patched_utmzip.pkl # CHR 虚拟机模板（已修补）
│   └── rose.chr.utmzip.pkl # ROSE 虚拟机模板（带额外磁盘）
├── Manifests/             # 虚拟机清单（具体配置）
│   ├── chr.x86_64.qemu.pkl
│   ├── chr.aarch64.qemu.pkl
│   ├── chr.x86_64.apple.pkl
│   ├── rose.chr.x86_64.qemu.pkl
│   ├── rose.chr.aarch64.qemu.pkl
│   ├── chr_patched_x86_64_qemu.pkl   # 已修补版本清单
│   └── chr_patched_aarch64_qemu.pkl  # 已修补版本清单
├── Files/                 # 静态文件
│   └── efi_vars.fd       # Apple 虚拟化 EFI 变量
├── Makefile              # 构建脚本
└── README.md             # 本项目说明
```

## 构建流程

### 1. 环境准备

```bash
# macOS/Linux
brew install pkl      # 安装 Pkl 编译器
brew install qemu-img  # 安装 QEMU 工具（可选，用于额外磁盘）
brew install wget     # 下载工具
```

### 2. 本地构建

```bash
git clone https://github.com/wanjiban/mikropkl
cd mikropkl
git checkout pkl
make              # 构建所有虚拟机
```

### 3. 环境变量配置

```bash
# 指定 RouterOS 版本或通道
export CHR_VERSION=stable      # stable, long-term, testing, development, upgrade
# 或指定具体版本号
# export CHR_VERSION=7.15.3
# 指定架构
export UTM_ARCHITECTURE=aarch64
```

### 4. GitHub Actions 自动构建

- 触发条件：手动触发 workflow_dispatch 或定时任务检测到新版本
- 构建产物：`.zip` 文件包含 `.utm` 包
- 发布位置：GitHub Releases

## 虚拟机类型说明

### 1. CHR（Cloud Hosted Router）- 官方版

- 标准 RouterOS CHR 虚拟机
- 无额外磁盘
- 适用于一般路由功能测试

### 2. CHR-PATCHED - 已修补版

- 基于 mikrotik.ltd 提供的已修补 RouterOS 镜像
- 已修补公钥，支持完整功能
- 支持在线更新、在线授权、云备份、DDNS
- 无额外磁盘

### 3. ROSE（RouterOS Storage Edition）

- 基于 CHR，增加了 4 个 10GB 虚拟磁盘
- 用于测试 RouterOS 存储功能（RAID、BTRFS 等）
- 默认磁盘处于未格式化状态

### 4. 架构支持

| 架构 | QEMU | Apple | 说明 |
|------|------|-------|------|
| x86_64 | ✅ | ✅ | 64位 Intel/AMD |
| aarch64 | ✅ | ❌ | 64位 ARM（Apple Silicon） |

### 5. 虚拟化后端

- **QEMU**：支持更广泛的设备和功能，包括额外磁盘、网络配置等
- **Apple**：使用 Apple Virtualization Framework，启动更快，但功能受限

## 定时任务

### schedule workflow

- **每日检查**：每天凌晨自动检查上游版本
- **自动构建**：版本更新时自动触发构建
- **清理**：自动删除失败的 workflow 运行记录

## 注意事项

### 1. 许可证限制

- 免费版 CHR 限速 1Mb/s
- 可通过注册 MikroTik 账号获取试用许可证（10Gb/s）
- 付费许可证可解除所有限制

### 2. 网络配置

- **Shared 模式**：NAT 转发，虚拟机可访问互联网
- **Bridged 模式**：桥接到物理网卡，获取独立 IP
- Apple 虚拟化不支持端口转发，需要 Bridged 模式暴露服务

### 3. 版本兼容性

- RouterOS 6.x 仅支持 x86 架构
- RouterOS 7.x 支持 x86 和 ARM64
- 较新的功能可能需要特定版本

### 4. 构建注意事项

- 每次 `make` 会重新下载所有镜像
- 本地构建产物在 `./Machines` 目录
- GitHub Actions 构建产物在 Releases 页面

### 5. 镜像下载

**官方源**：
- 使用 MikroTik 官方镜像源 `download.mikrotik.com`
- 版本检查使用 `upgrade.mikrotik.com/routeros/NEWESTa7.<channel>`

**PATCHED 源**：
- 使用 mikrotik.ltd 提供的已修补镜像（来自 GitHub elseif/MikroTikPatch releases）

### 6. Apple 虚拟化特殊要求

- 需要 macOS 13+ (Ventura) 或更高版本
- 需要 Apple Silicon Mac（M1/M2/M3）
- 需要 `efi_vars.fd` 文件

## 常见问题

### Q: 构建失败怎么办？

A: 检查网络连接，确认可以访问 GitHub 和 mikrotik.com

### Q: 虚拟机无法启动？

A:
1. 检查 UTM 版本是否支持
2. 确认镜像下载完整
3. 查看系统日志

### Q: 如何更新到新版本？

A: 修改 `CHR_VERSION` 环境变量，重新构建

### Q: ROSE 磁盘如何使用？

A: 参考 README 中的 ROSE 使用说明，进行格式化和挂载

## 参考链接

- [Pkl 官方文档](https://pkl-lang.org)
- [UTM 文档](https://docs.getutm.app)
- [RouterOS 文档](https://help.mikrotik.com/docs)
- [MikroTik 官方下载](https://mikrotik.com/download)
- [mikrotik.ltd - 已修补 RouterOS](https://mikrotik.ltd/)
- [MikroTikPatch 项目](https://github.com/elseif/MikroTikPatch)
