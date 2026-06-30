# Android Kernel for LeMax2 (x2) — Droidspaces Edition

基于 [LineageOS android_kernel_leeco_msm8996](https://github.com/LineageOS/android_kernel_leeco_msm8996) 的乐视 Max2 (x2) 内核源码，集成了 [Droidspaces](https://github.com/ravindu644/Droidspaces-OSS) 容器运行时所需的全部内核配置和补丁。

## 设备信息

| 项目 | 参数 |
|------|------|
| 设备 | 乐视 Max2 (LeMax2 / x2) |
| SoC | 高通骁龙 820 (MSM8996) |
| 内核版本 | 3.18.140 |
| ROM | LineageOS 18.1 |
| 架构 | ARM64 |

## 相对原版 LineageOS 内核的修改

### 内核配置（Droidspaces 必需）

| 类别 | 配置项 | 说明 |
|------|--------|------|
| **IPC** | `SYSVIPC`, `POSIX_MQUEUE` | IPC 命名空间依赖 |
| **命名空间** | `PID_NS`, `UTS_NS`, `IPC_NS`, `USER_NS`, `NET_NS` | 容器隔离核心 |
| **Cgroup** | `CGROUP_DEVICE`, `MEMCG`, `CGROUP_NET_PRIO` | 资源控制 |
| **文件系统** | `DEVTMPFS`, `OVERLAY_FS`, `TMPFS_POSIX_ACL`, `TMPFS_XATTR` | 设备文件、易失模式 |
| **网络** | `VETH`, `NF_TABLES`, `NF_NAT`, `BRIDGE`, `BRIDGE_NETFILTER` | NAT 网络隔离 |
| **安全** | `SECCOMP`, `SECCOMP_FILTER` | 系统调用过滤 |

### 内核补丁

1. **xt_qtaguid 内核 panic 修复** — 移除在非活跃网络接口上不安全的 `dev_get_stats` 调用，防止容器网络导致内核崩溃
2. **cgroup 文件前缀处理修复** — 恢复 `CGRP_ROOT_NOPREFIX` 模式下的符号链接创建，兼容 Droidspaces/LXC

### 其他修改

- 禁用 `LOCALVERSION_AUTO`，确保内核版本字符串一致性

## 编译

### 环境要求

- Ubuntu/Kali Linux (推荐 20.04+)
- GCC aarch64 交叉编译工具链

### 工具链

```bash
git clone --depth=1 -b lineage-18.1 https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-android-4.9.git
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

# 编译
make O=out -j$(nproc)
```

### 打包刷机包

使用 [AnyKernel3](https://github.com/osm0sis/AnyKernel3)：

```bash
git clone https://github.com/osm0sis/AnyKernel3.git
cp out/arch/arm64/boot/Image.gz-dtb AnyKernel3/
cd AnyKernel3
# 编辑 anykernel.sh：
#   do.devicecheck=0
#   BLOCK=/dev/block/bootdevice/by-name/boot
zip -r9 Droidspaces-x2.zip * -x '*.git*' 'README.md' 'LICENSE'
```

### 刷入

```bash
adb push Droidspaces-x2.zip /sdcard/
adb reboot recovery
# 在 Recovery 中选择 Apply update → 选择 zip 文件
```

## Root 方案

本内核兼容以下 Root 方案：

| 方案 | 支持 | 说明 |
|------|------|------|
| [APatch](https://github.com/bmax121/APatch) | ✅ | 推荐，无需内核源码，直接修补 boot.img |
| [KernelSU](https://github.com/tiann/KernelSU) | ❌ | 仅支持 GKI 内核 (5.10+) |
| [SukiSU Ultra](https://github.com/SukiSU-Ultra/SukiSU-Ultra) | ❌ | 最低支持 4.14 |
| Magisk | ✅ | 传统方案 |

## Droidspaces

编译并刷入本内核后，配合 APatch 获取 root，即可使用 [Droidspaces](https://github.com/ravindu644/Droidspaces-OSS) 运行完整 Linux 容器环境。

### 验证内核配置

```bash
su -c droidspaces check
```

### 已知问题

- 3.18 内核为最低支持版本，仅提供基础命名空间支持
- 不支持 `CGROUP_PIDS`（4.6+ 才引入）
- 不支持嵌套容器（Docker-in-Droidspaces）
- 现代 Linux 发行版可能不稳定，推荐使用 Alpine

## 致谢

- [LineageOS](https://lineageos.org/) — 原始内核源码
- [Droidspaces](https://github.com/ravindu644/Droidspaces-OSS) — 容器运行时及内核补丁
- [APatch](https://github.com/bmax121/APatch) — 内核级 Root 方案
- [AnyKernel3](https://github.com/osm0sis/AnyKernel3) — 内核刷机包打包工具

## 许可证

内核源码遵循 [GPL-2.0](COPYING) 许可证。
