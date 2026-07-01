# Android Kernel for LeMax2 (x2) — Droidspaces + ReSukiSU Edition

> 🤖 **本项目由 AI 全力驱动开发** — 从内核配置、补丁集成、编译调试到 Git 工作流，全程由 [Claude](https://claude.ai) 完成。

基于 [LineageOS android_kernel_leeco_msm8996](https://github.com/LineageOS/android_kernel_leeco_msm8996) 的乐视 Max2 (x2) 内核源码，集成了 [Droidspaces](https://github.com/ravindu644/Droidspaces-OSS) 容器运行时和 [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU) 内核级 Root 方案。

---

## 目录

- [设备信息](#设备信息)
- [功能特性](#功能特性)
- [相对原版 LineageOS 内核的修改](#相对原版-lineageos-内核的修改)
  - [Droidspaces 内核配置](#droidspaces-内核配置)
  - [Droidspaces 内核补丁](#droidspaces-内核补丁)
  - [ReSukiSU 内核集成](#resukisu-内核集成)
  - [GCC 兼容性修复](#gcc-兼容性修复)
- [编译指南](#编译指南)
  - [环境要求](#环境要求)
  - [下载工具链](#下载工具链)
  - [编译步骤](#编译步骤)
  - [打包刷机包](#打包刷机包)
- [刷机指南](#刷机指南)
  - [方法一：AnyKernel3 刷机包](#方法一anykernel3-刷机包)
  - [方法二：magiskboot 手动替换](#方法二magiskboot-手动替换)
- [Root 方案对比](#root-方案对比)
- [Droidspaces 容器运行时](#droidspaces-容器运行时)
  - [安装步骤](#安装步骤)
  - [验证内核配置](#验证内核配置)
  - [已知限制](#已知限制)
- [ReSukiSU Root 方案](#resukisu-root-方案)
  - [安装 ReSukiSU Manager](#安装-resukisu-manager)
  - [Root 使用](#root-使用)
- [CI/CD 自动编译](#cicd-自动编译)
- [故障排除](#故障排除)
- [项目结构](#项目结构)
- [致谢](#致谢)
- [许可证](#许可证)

---

## 设备信息

| 项目 | 参数 |
|------|------|
| 设备 | 乐视 Max2 (LeMax2 / x2) |
| SoC | 高通骁龙 820 (MSM8996) |
| 架构 | ARM64 (AArch64) |
| 内核版本 | 3.18.140 |
| ROM | LineageOS 18.1 (Android 11) |
| 内核源码分支 | lineage-18.1 |

---

## 功能特性

本内核在原版 LineageOS 内核基础上集成了两大功能：

### 1. Droidspaces 容器运行时

在 Android 上运行完整 Linux 发行版（Ubuntu、Debian、Alpine 等），支持 systemd、网络隔离、GPU 加速等。

### 2. ReSukiSU 内核级 Root

基于 KernelSU 的内核级 Root 方案，支持到 3.4+ 内核版本，提供 su 权限管理和模块系统。

---

## 相对原版 LineageOS 内核的修改

### Droidspaces 内核配置

启用了 Droidspaces 容器运行时所需的全部内核配置（共 39 项）：

| 类别 | 配置项 | 说明 | 必需性 |
|------|--------|------|--------|
| **IPC 机制** | `CONFIG_SYSVIPC=y` | System V IPC 支持 | 必需 |
| | `CONFIG_POSIX_MQUEUE=y` | POSIX 消息队列 | 必需 |
| **核心命名空间** | `CONFIG_NAMESPACES=y` | 命名空间总开关 | 必需 |
| | `CONFIG_PID_NS=y` | PID 命名空间 | 必需（致命） |
| | `CONFIG_UTS_NS=y` | UTS 命名空间 | 必需（致命） |
| | `CONFIG_IPC_NS=y` | IPC 命名空间 | 必需（致命） |
| | `CONFIG_USER_NS=y` | 用户命名空间 | 推荐 |
| | `CONFIG_NET_NS=y` | 网络命名空间 | 推荐（NAT/None 模式） |
| **Seccomp** | `CONFIG_SECCOMP=y` | 系统调用过滤 | 必需 |
| | `CONFIG_SECCOMP_FILTER=y` | Seccomp 过滤器 | 必需 |
| **Cgroup** | `CONFIG_CGROUPS=y` | 控制组总开关 | 必需 |
| | `CONFIG_CGROUP_DEVICE=y` | 设备 Cgroup | 必需（致命） |
| | `CONFIG_MEMCG=y` | 内存 Cgroup | 推荐 |
| | `CONFIG_CGROUP_SCHED=y` | 调度器 Cgroup | 推荐 |
| | `CONFIG_FAIR_GROUP_SCHED=y` | 公平调度 | 推荐 |
| | `CONFIG_CGROUP_FREEZER=y` | 冻结器 Cgroup | 推荐 |
| | `CONFIG_CGROUP_NET_PRIO=y` | 网络优先级 Cgroup | 推荐 |
| **文件系统** | `CONFIG_DEVTMPFS=y` | 设备文件系统 | 必需（致命） |
| | `CONFIG_OVERLAY_FS=y` | OverlayFS | 推荐（易失模式） |
| | `CONFIG_TMPFS_POSIX_ACL=y` | tmpfs POSIX ACL | 推荐（NixOS） |
| | `CONFIG_TMPFS_XATTR=y` | tmpfs 扩展属性 | 推荐 |
| **固件** | `CONFIG_FW_LOADER=y` | 固件加载 | 推荐 |
| | `CONFIG_FW_LOADER_USER_HELPER=y` | 固件用户辅助 | 推荐 |
| **网络** | `CONFIG_VETH=y` | 虚拟以太网对 | 推荐（NAT 模式） |
| | `CONFIG_BRIDGE=y` | 网桥设备 | 推荐（NAT 模式） |
| | `CONFIG_BRIDGE_NETFILTER=y` | 网桥 Netfilter | 推荐 |
| | `CONFIG_NETFILTER_ADVANCED=y` | 高级 Netfilter | 推荐 |
| | `CONFIG_NF_CONNTRACK=y` | 连接跟踪 | 推荐 |
| | `CONFIG_NF_NAT=y` | NAT 支持 | 推荐 |
| | `CONFIG_NF_TABLES=y` | nftables | 推荐 |
| | `CONFIG_IP_NF_IPTABLES=y` | iptables | 推荐 |
| | `CONFIG_IP_NF_FILTER=y` | IP 过滤 | 推荐 |
| | `CONFIG_IP_NF_TARGET_MASQUERADE=y` | MASQUERADE 目标 | 推荐 |
| | `CONFIG_NETFILTER_XT_TARGET_TCPMSS=y` | TCP MSS 目标 | 推荐 |
| | `CONFIG_NETFILTER_XT_MATCH_ADDRTYPE=y` | 地址类型匹配 | 推荐 |
| | `CONFIG_IP_ADVANCED_ROUTER=y` | 高级路由 | 推荐 |
| | `CONFIG_IP_MULTIPLE_TABLES=y` | 多路由表 | 推荐 |
| **内核版本** | `# CONFIG_LOCALVERSION_AUTO is not set` | 禁用自动版本后缀 | 推荐（避免 fingerprint 不一致） |

### Droidspaces 内核补丁

应用了两个 Droidspaces 官方 non-GKI 补丁：

1. **`01.fix_kernel_panic_in_xt_qtaguid.patch`**
   - 修复 `net/netfilter/xt_qtaguid.c` 中的内核 panic
   - 移除在非活跃网络接口上不安全的 `dev_get_stats` 调用
   - 防止容器网络导致内核崩溃

2. **`02.fix_restore cgroup file prefix handling.patch`**
   - 修复 `kernel/cgroup.c` 中的 cgroup 文件前缀处理
   - 恢复 `CGRP_ROOT_NOPREFIX` 模式下的符号链接创建
   - 兼容 Droidspaces/LXC 的 cgroup 挂载

### ReSukiSU 内核集成

集成了 [ReSukiSU v4.1.0](https://github.com/ReSukiSU/ReSukiSU)（基于 KernelSU 的 fork，支持 3.4+ 内核），使用手动 Hook 方式集成。

#### 集成方式

- **Hook 方式**：Manual Hook（手动 Hook，适用于 non-GKI 内核）
- **Kconfig**：`CONFIG_KSU=y`、`CONFIG_KSU_MANUAL_HOOK=y`

#### 内核源码修改

| 文件 | 修改内容 | 说明 |
|------|---------|------|
| `fs/exec.c` | 添加 `ksu_handle_execveat` hook | 拦截 execve 系统调用 |
| `fs/open.c` | 添加 `ksu_handle_faccessat` hook | 拦截 faccessat 系统调用 |
| `fs/stat.c` | 添加 `ksu_handle_stat` hook | 拦截 stat 系统调用 |
| `fs/stat.c` | 添加 `ksu_handle_newfstat_ret` hook | 拦截 newfstat 返回值 |
| `fs/stat.c` | 添加 `ksu_handle_fstat64_ret` hook | 拦截 fstat64 返回值（32-bit su） |
| `drivers/input/input.c` | 添加 `ksu_handle_input_handle_event` hook | 拦截输入事件 |
| `kernel/reboot.c` | 添加 `ksu_handle_sys_reboot` hook | 拦截重启系统调用 |

### GCC 兼容性修复

| 文件 | 修改 | 说明 |
|------|------|------|
| `include/linux/compiler.h` | 添加 `__GCC4_has_attribute___fallthrough__` | GCC 4.9 缺少 fallthrough 属性定义 |

---

## 编译指南

### 环境要求

- Ubuntu 20.04+ / Kali 2022.3+
- GCC aarch64 交叉编译工具链
- 8GB+ 内存（推荐 16GB）
- 50GB+ 磁盘空间

### 下载工具链

```bash
git clone --depth=1 -b lineage-18.1 \
  https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-android-4.9.git
```

### 编译步骤

```bash
# 设置环境变量
export PATH=$PATH:$(pwd)/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-android-4.9/bin
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-android-

# 清理
make O=out mrproper

# 生成配置
make O=out lineage_x2_defconfig

# 编译（-j 后面的数字根据 CPU 核心数调整）
make O=out -j$(nproc)
```

> **注意**：编译过程中可能因为环境缺少特定工具而报错，根据报错信息用 `apt install` 安装缺少的依赖即可。

### 打包刷机包

使用 [AnyKernel3](https://github.com/osm0sis/AnyKernel3) 打包：

```bash
# 下载 AnyKernel3
git clone --depth=1 https://github.com/osm0sis/AnyKernel3.git

# 配置 AnyKernel3
cd AnyKernel3
sed -i 's/do.devicecheck=1/do.devicecheck=0/' anykernel.sh
sed -i 's/device.name1=maguro/device.name1=x2/' anykernel.sh
sed -i 's|BLOCK=.*|BLOCK=/dev/block/bootdevice/by-name/boot|' anykernel.sh

# 复制内核镜像
cp ../out/arch/arm64/boot/Image.gz-dtb .

# 打包
zip -r9 ../Droidspaces-APatch-x2.zip * -x '*.git*' 'README.md' 'LICENSE'
```

---

## 刷机指南

### 方法一：AnyKernel3 刷机包

```bash
# 推送到手机
adb push Droidspaces-APatch-x2.zip /sdcard/

# 进入 Recovery
adb reboot recovery

# 在 Recovery 中选择：
# Apply update → Choose from internal storage → 选择 zip 文件

# 刷入完成后重启
adb reboot
```

### 方法二：magiskboot 手动替换

适用于需要精确控制 boot.img 的场景：

```bash
# 解包原厂 boot.img
./magiskboot unpack boot.img

# 用编译好的内核替换 kernel
mv -f Image.gz-dtb kernel

# 重新打包
./magiskboot repack boot.img

# 刷入
fastboot flash boot new-boot.img
```

> **注意**：APatch 修补 boot.img 时需要使用原厂 boot.img，不要使用已被其他工具修补过的 boot.img。

---

## Root 方案对比

| 方案 | 3.18 支持 | 需要内核源码 | 需要编译内核 | 安装方式 |
|------|----------|-------------|-------------|---------|
| **ReSukiSU** | ✅（3.4+） | 是 | 是 | 源码集成 |
| **APatch** | ✅（3.18-6.12） | 否 | 否 | boot.img 修补 |
| **KernelSU** | ❌（仅 GKI） | 是 | 是 | — |
| **SukiSU Ultra** | ❌（最低 4.14） | 是 | 是 | — |
| **Magisk** | ✅ | 否 | 否 | boot.img 修补 |

本内核集成了 **ReSukiSU**，同时也兼容 **APatch**（可通过 kptools 修补 boot.img）。

---

## Droidspaces 容器运行时

### 安装步骤

1. **获取 Root**：本内核自带 ReSukiSU Root
2. **安装 Droidspaces APK**：从 [GitHub Releases](https://github.com/ravindu644/Droidspaces-OSS/releases/latest) 下载
3. **验证内核配置**：
   ```bash
   su -c droidspaces check
   ```
4. **选择 Linux 发行版 rootfs** 开始使用

### 验证内核配置

运行以下命令检查内核是否满足 Droidspaces 要求：

```bash
su -c droidspaces check
```

检查项包括：

| 检查项 | 本内核状态 |
|--------|----------|
| Root 权限 | ✅ ReSukiSU |
| PID 命名空间 | ✅ |
| 挂载命名空间 | ✅ |
| UTS 命名空间 | ✅ |
| IPC 命名空间 | ✅ |
| Seccomp 支持 | ✅ |
| devtmpfs | ✅ |
| OverlayFS | ✅ |
| 网络命名空间 | ✅ |
| VETH / Bridge | ✅ |

### 已知限制

| 限制 | 说明 |
|------|------|
| Cgroup v2 | 不可用（3.18 仅支持 cgroup v1） |
| Cgroup 命名空间 | 不可用（4.6+ 才引入） |
| 嵌套容器 | 不推荐（Docker-in-Droidspaces 在 3.18 上不稳定） |
| 现代发行版 | 可能不稳定，推荐使用 **Alpine** |
| IPC 命名空间 | 3.18 中不存在 `CONFIG_IPC_NS` 选项，已通过启用 `SYSVIPC` 间接启用 |

---

## ReSukiSU Root 方案

### 安装 ReSukiSU Manager

由于 ReSukiSU Manager 仍在开发中，需要从 CI 获取：

1. 前往 [ReSukiSU GitHub Actions](https://github.com/ReSukiSU/ReSukiSU/actions)
2. 下载最新的 nightly 构建
3. 安装 APK

### Root 使用

ReSukiSU 已在内核中编译集成，开机后 Manager 应自动检测到 Root 状态。

- **su 命令**：通过 Manager 授权 App 使用
- **超级密钥**：`123456789`（APatch 修补时使用）
- **模块系统**：支持 KernelSU 兼容模块

---

## CI/CD 自动编译

本项目配置了 GitHub Actions，推送代码后自动编译。

### 触发条件

- 推送到 `lineage-18.1` 分支
- 手动触发（workflow_dispatch）

### 构建产物

| 文件 | 说明 |
|------|------|
| `Droidspaces-x2-kernel.zip` | 纯 Droidspaces 内核（AnyKernel3 格式） |
| `Droidspaces-APatch-x2-kernel.zip` | Droidspaces + APatch 内核（SuperKey: `123456789`） |
| `boot-stock-lineage.img` | 原厂 LineageOS boot.img（备用） |

### 查看构建状态

https://github.com/huanghao680/android_kernel_leeco_msm8996/actions

---

## 故障排除

### 编译错误

| 错误 | 解决方案 |
|------|---------|
| `aarch64-linux-android-gcc: No such file or directory` | 工具链 Python wrapper shebang 问题，将 `#!/usr/bin/python` 改为 `#!/usr/bin/python3` |
| `ISO C90 forbids mixed declarations and code` | 将 `#ifdef CONFIG_KSU` 代码块移到变量声明之后 |
| `implicit declaration of function` | 检查 extern 声明位置，确保在函数外部 |
| `__GCC4_has_attribute___fallthrough__ is not defined` | 在 `include/linux/compiler.h` 中添加定义 |
| `undefined reference to groups_sort` | 在 `kernel/groups.c` 中将 `static` 改为 `EXPORT_SYMBOL` |
| `undefined reference to d_is_reg` | 将 `d_is_reg()` 替换为 `S_ISREG(d_inode->i_mode)` |

### 刷机问题

| 问题 | 解决方案 |
|------|---------|
| "您的设备内部出现了问题" 弹窗 | Build fingerprint 不一致，禁用 `LOCALVERSION_AUTO` 并使用 `.scmversion` |
| 开机后卡在 Logo | 使用 `fastboot boot boot.img` 先测试，不直接 flash |
| KernelSU Manager 显示"不支持" | 版本不匹配，使用对应版本的 Manager APK |

### Droidspaces 问题

| 问题 | 解决方案 |
|------|---------|
| `su: inaccessible or not found` | 使用 `adb root` 而非 `adb shell su`（userdebug 版本） |
| 容器无法启动 | 运行 `su -c droidspaces check` 检查缺失的内核配置 |
| 网络不通 | 确保 `CONFIG_NET_NS`、`CONFIG_VETH`、`CONFIG_BRIDGE` 已启用 |

---

## 项目结构

```
android_kernel_leeco_msm8996/
├── .github/workflows/     # CI/CD 配置
│   └── build.yml          # GitHub Actions 构建流程
├── arch/arm64/configs/
│   └── lineage_x2_defconfig  # 内核配置（含 Droidspaces + ReSukiSU）
├── drivers/kernelsu -> ReSukiSU/kernel  # ReSukiSU 软链接
├── KernelSU/              # ReSukiSU 源码（原名 KernelSU）
├── prebuilts/
│   └── boot-stock-lineage.img  # 原厂 boot.img 备份
├── fs/
│   ├── exec.c             # execve hook
│   ├── open.c             # faccessat hook
│   └── stat.c             # stat hook
├── kernel/
│   ├── reboot.c           # reboot hook
│   └── groups.c           # groups_sort 导出
├── drivers/input/
│   └── input.c            # input hook
└── include/linux/
    └── compiler.h         # GCC 4.9 兼容性修复
```

---

## 致谢

| 项目 | 说明 |
|------|------|
| [LineageOS](https://lineageos.org/) | 原始内核源码 |
| [Droidspaces](https://github.com/ravindu644/Droidspaces-OSS) | 容器运行时及内核配置/补丁 |
| [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU) | 内核级 Root 方案（支持 3.4+ 内核） |
| [KernelSU](https://github.com/tiann/KernelSU) | ReSukiSU 的上游项目 |
| [APatch](https://github.com/bmax121/APatch) | 可选的内核级 Root 方案 |
| [AnyKernel3](https://github.com/osm0sis/AnyKernel3) | 内核刷机包打包工具 |
| [KernelPatch](https://github.com/bmax121/KernelPatch) | APatch 底层工具 |

---

## 许可证

- 内核源码遵循 [GPL-2.0](COPYING) 许可证
- ReSukiSU 遵循 MIT 许可证
- Droidspaces 遵循 [GPL-3.0](https://github.com/ravindu644/Droidspaces-OSS/blob/main/LICENSE) 许可证
