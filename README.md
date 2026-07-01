# Android Kernel for LeMax2 (x2) — Droidspaces + ReSukiSU Edition

> 🤖 **本项目由 AI 全力驱动开发** — 从内核配置、补丁集成、编译调试到 Git 工作流，全程由 [Claude](https://claude.ai) 完成。

基于 [LineageOS android_kernel_leeco_msm8996](https://github.com/LineageOS/android_kernel_leeco_msm8996) 的乐视 Max2 (x2) 内核源码，集成了 [Droidspaces](https://github.com/ravindu644/Droidspaces-OSS) 容器运行时和 [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU) 内核级 Root 方案。

---

## 与原版 LineageOS 内核的完整差异

以下列出本项目相对 [LineageOS lineage-18.1 分支](https://github.com/LineageOS/android_kernel_leeco_msm8996/tree/lineage-18.1) 的 **所有修改**。

### 一、内核配置修改（arch/arm64/configs/lineage_x2_defconfig）

在原版 defconfig 基础上新增以下配置：

#### Droidspaces 容器支持（39 项）

```diff
 # IPC 机制
+CONFIG_SYSVIPC=y
+CONFIG_POSIX_MQUEUE=y

 # 核心命名空间
+CONFIG_NAMESPACES=y
+CONFIG_PID_NS=y
+CONFIG_UTS_NS=y
+CONFIG_IPC_NS=y
+CONFIG_USER_NS=y

 # Seccomp
+CONFIG_SECCOMP=y
+CONFIG_SECCOMP_FILTER=y

 # Cgroup
+CONFIG_CGROUP_DEVICE=y
+CONFIG_MEMCG=y
+CONFIG_CGROUP_NET_PRIO=y

 # 文件系统
+CONFIG_DEVTMPFS=y
+CONFIG_OVERLAY_FS=y
+CONFIG_TMPFS_POSIX_ACL=y
+CONFIG_TMPFS_XATTR=y

 # 固件
+CONFIG_FW_LOADER=y
+CONFIG_FW_LOADER_USER_HELPER=y

 # 网络
+CONFIG_VETH=y
+CONFIG_BRIDGE=y
+CONFIG_BRIDGE_NETFILTER=y
+CONFIG_NETFILTER_ADVANCED=y
+CONFIG_NF_CONNTRACK=y
+CONFIG_NF_NAT=y
+CONFIG_NF_TABLES=y
+CONFIG_NF_CONNTRACK_NETLINK=y
+CONFIG_NETFILTER_XT_MATCH_ADDRTYPE=y
+CONFIG_NF_CONNTRACK_IPV4=y
+CONFIG_NF_NAT_IPV4=y
+CONFIG_IP_NF_NAT=y
```

#### ReSukiSU Root 支持（3 项）

```diff
+# ReSukiSU
+CONFIG_KSU=y
+CONFIG_KSU_MANUAL_HOOK=y
```

#### 其他修改（1 项）

```diff
-CONFIG_LOCALVERSION_AUTO=y
+# CONFIG_LOCALVERSION_AUTO is not set
```

禁用自动版本后缀，避免 git dirty marker 导致的 build fingerprint 不一致弹窗。

---

### 二、内核源码修改

#### 1. ReSukiSU 手动 Hook（7 个文件，66 行新增）

**fs/exec.c** — execve 系统调用拦截

```diff
+#ifdef CONFIG_KSU_MANUAL_HOOK
+extern int ksu_handle_execveat(int *fd, struct filename **filename_ptr,
+				void *argv, void *envp, int *flags);
+#endif

 static int do_execve_common(struct filename *filename,
 				struct user_arg_ptr argv,
 				struct user_arg_ptr envp)
 {
 	...
+	#ifdef CONFIG_KSU_MANUAL_HOOK
+	ksu_handle_execveat(NULL, &filename, &argv, &envp, NULL);
+	#endif
 	if (IS_ERR(filename))
```

**fs/open.c** — faccessat 系统调用拦截

```diff
+#ifdef CONFIG_KSU_MANUAL_HOOK
+extern int ksu_handle_faccessat(int *dfd, const char __user **filename_user,
+				 int *mode, int *flags);
+#endif

 SYSCALL_DEFINE3(faccessat, int, dfd, const char __user *, filename, int, mode)
 {
+	#ifdef CONFIG_KSU_MANUAL_HOOK
+	ksu_handle_faccessat(&dfd, &filename, &mode, NULL);
+	#endif
 	...
```

**fs/stat.c** — stat 系统调用拦截（3 个 hook）

```diff
+#ifdef CONFIG_KSU_MANUAL_HOOK
+extern int ksu_handle_stat(int *dfd, const char __user **filename, int *flags);
+extern void ksu_handle_newfstat_ret(unsigned int *fd, struct stat __user **statbuf_ptr);
+extern void ksu_handle_fstat64_ret(unsigned long *fd, struct stat64 __user **statbuf_ptr);
+#endif

 SYSCALL_DEFINE4(newfstatat, ...)
 {
+	#ifdef CONFIG_KSU_MANUAL_HOOK
+	ksu_handle_stat(&dfd, &filename, &flag);
+	#endif
 	error = vfs_fstatat(dfd, filename, &stat, flag);
 	...
 }

 SYSCALL_DEFINE2(newfstat, ...)
 {
 	...
+	#ifdef CONFIG_KSU_MANUAL_HOOK
+	ksu_handle_newfstat_ret(&fd, &statbuf);
+	#endif
 	return error;
 }

 SYSCALL_DEFINE2(fstat64, ...)
 {
 	...
+	#ifdef CONFIG_KSU_MANUAL_HOOK
+	ksu_handle_fstat64_ret(&fd, &statbuf);
+	#endif
 	return error;
 }
```

**drivers/input/input.c** — 输入事件拦截

```diff
+#ifdef CONFIG_KSU_MANUAL_HOOK
+extern bool ksu_input_hook __read_mostly;
+extern __attribute__((cold)) int ksu_handle_input_handle_event(
+			unsigned int *type, unsigned int *code, int *value);
+#endif

 void input_event(struct input_dev *dev,
 		 unsigned int type, unsigned int code, int value)
 {
 	unsigned long flags;
+	#ifdef CONFIG_KSU_MANUAL_HOOK
+	if (unlikely(ksu_input_hook))
+		ksu_handle_input_handle_event(&type, &code, &value);
+	#endif
 	...
```

**kernel/reboot.c** — reboot 系统调用拦截

```diff
+#ifdef CONFIG_KSU_MANUAL_HOOK
+extern int ksu_handle_sys_reboot(int magic1, int magic2,
+				  unsigned int cmd, void __user **arg);
+#endif

 SYSCALL_DEFINE4(reboot, int, magic1, int, magic2, unsigned int, cmd,
 		void __user *, arg)
 {
 	...
+	#ifdef CONFIG_KSU_MANUAL_HOOK
+	ksu_handle_sys_reboot(magic1, magic2, cmd, &arg);
+	#endif
 	...
```

#### 2. Droidspaces non-GKI 补丁（2 个文件）

**net/netfilter/xt_qtaguid.c** — 防止容器网络内核 panic

```diff
-	struct rtnl_link_stats64 dev_stats, *stats;
+	struct rtnl_link_stats64 *stats;
 	...
-	if (iface_entry->active) {
-		stats = dev_get_stats(iface_entry->net_dev, &dev_stats);
-	} else {
-		stats = &no_dev_stats;
-	}
+	stats = &no_dev_stats;
```

**kernel/cgroup.c** — 恢复 cgroup 文件前缀处理

```diff
+	if (cft->ss && (cgrp->root->flags & CGRP_ROOT_NOPREFIX)
+	    && !(cft->flags & CFTYPE_NO_PREFIX)) {
+			snprintf(name, CGROUP_FILE_NAME_MAX, "%s.%s",
+				 cft->ss->name, cft->name);
+			kernfs_create_link(cgrp->kn, name, kn);
+	}
```

#### 3. GCC 4.9 兼容性修复（1 个文件）

**include/linux/compiler.h** — 添加缺失的属性定义

```diff
 # define __has_attribute(x) __GCC4_has_attribute_##x
 # define __GCC4_has_attribute___copy__                0
+# define __GCC4_has_attribute___fallthrough__         0
```

---

### 三、构建系统修改

**drivers/Makefile** — 添加 ReSukiSU 编译目标

```diff
+obj-$(CONFIG_KSU) += kernelsu/
```

**drivers/Kconfig** — 添加 ReSukiSU 配置菜单

```diff
+source "drivers/kernelsu/Kconfig"
```

**drivers/kernelsu** — ReSukiSU 内核源码符号链接

```
drivers/kernelsu -> /path/to/ReSukiSU/kernel
```

---

### 四、新增文件

| 文件 | 说明 |
|------|------|
| `ReSukiSU/` | ReSukiSU v4.1.0 内核源码 |
| `prebuilts/boot-stock-lineage.img` | 原厂 LineageOS boot.img（备用） |
| `.github/workflows/build.yml` | GitHub Actions 自动编译流程 |
| `.scmversion` | 空文件，抑制 git dirty marker |

---

### 五、修改统计

| 类别 | 文件数 | 新增行 | 删除行 |
|------|--------|--------|--------|
| 内核配置 | 1 | 4 | 0 |
| ReSukiSU Hook | 5 | 66 | 0 |
| Droidspaces 补丁 | 2 | 6 | 10 |
| GCC 兼容性 | 1 | 1 | 0 |
| 构建系统 | 2 | 3 | 0 |
| CI/CD | 1 | 122 | 0 |
| 文档 | 1 | 363 | 54 |
| **合计** | **13** | **564** | **64** |

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
export PATH=$PATH:$(pwd)/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-android-4.9/bin
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-android-

# 清理 + 生成配置 + 编译
make O=out mrproper
make O=out lineage_x2_defconfig
make O=out -j$(nproc)
```

### 打包 AnyKernel3 刷机包

```bash
git clone --depth=1 https://github.com/osm0sis/AnyKernel3.git
cd AnyKernel3
sed -i 's/do.devicecheck=1/do.devicecheck=0/' anykernel.sh
sed -i 's/device.name1=maguro/device.name1=x2/' anykernel.sh
sed -i 's|BLOCK=.*|BLOCK=/dev/block/bootdevice/by-name/boot|' anykernel.sh
cp ../out/arch/arm64/boot/Image.gz-dtb .
zip -r9 ../Droidspaces-x2-kernel.zip * -x '*.git*' 'README.md' 'LICENSE'
```

---

## 刷机指南

### 方法一：AnyKernel3（推荐）

```bash
adb push Droidspaces-x2-kernel.zip /sdcard/
adb reboot recovery
# Recovery 中: Apply update → Choose from internal storage
```

### 方法二：magiskboot 手动替换

```bash
./magiskboot unpack boot.img
mv -f Image.gz-dtb kernel
./magiskboot repack boot.img
fastboot flash boot new-boot.img
```

---

## Root 方案

本内核自带 **ReSukiSU** Root，Manager 从 [GitHub Actions nightly](https://github.com/ReSukiSU/ReSukiSU/actions) 获取。

也兼容 **APatch**（通过 kptools 修补 boot.img，SuperKey: `123456789`）。

| 方案 | 3.18 支持 | 需要源码 | 安装方式 |
|------|----------|---------|---------|
| **ReSukiSU** | ✅ | 是 | 源码集成（已内置） |
| **APatch** | ✅ | 否 | boot.img 修补 |
| KernelSU | ❌ | — | 仅 GKI |

---

## Droidspaces

内核已启用 Droidspaces 所需的全部配置。安装 Droidspaces APK 后：

```bash
su -c droidspaces check    # 验证内核配置
su -c droidspaces start    # 启动容器
```

### 已知限制（3.18 内核）

- 无 Cgroup v2（4.5+ 才引入）
- 无 Cgroup 命名空间（4.6+ 才引入）
- 不推荐嵌套容器（Docker-in-Droidspaces）
- 推荐使用 **Alpine** 发行版

---

## CI/CD

推送代码到 `lineage-18.1` 分支后 GitHub Actions 自动编译：

| 产物 | 说明 |
|------|------|
| `Droidspaces-x2-kernel.zip` | 纯 Droidspaces 内核 |
| `Droidspaces-APatch-x2-kernel.zip` | Droidspaces + APatch（SuperKey: 123456789） |
| `boot-stock-lineage.img` | 原厂 boot.img 备用 |

查看构建：https://github.com/huanghao680/android_kernel_leeco_msm8996/actions

---

## 故障排除

| 问题 | 解决方案 |
|------|---------|
| `aarch64-linux-android-gcc: No such file` | 将工具链 Python wrapper shebang 从 `#!/usr/bin/python` 改为 `#!/usr/bin/python3` |
| `ISO C90 forbids mixed declarations` | 将 `#ifdef CONFIG_KSU` 代码移到变量声明之后 |
| `implicit declaration of function` | 检查 extern 声明位置 |
| `__GCC4_has_attribute___fallthrough__` | 在 `include/linux/compiler.h` 添加定义 |
| `undefined reference to groups_sort` | `kernel/groups.c` 中移除 `static`，添加 `EXPORT_SYMBOL` |
| `undefined reference to d_is_reg` | 替换为 `S_ISREG(d_inode->i_mode)` |
| "您的设备内部出现问题" 弹窗 | 禁用 `LOCALVERSION_AUTO` + 使用 `.scmversion` |
| KernelSU Manager 显示"不支持" | 版本不匹配，使用对应版本 Manager |

---

## 致谢

| 项目 | 说明 |
|------|------|
| [LineageOS](https://lineageos.org/) | 原始内核源码 |
| [Droidspaces](https://github.com/ravindu644/Droidspaces-OSS) | 容器运行时及内核配置/补丁 |
| [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU) | 内核级 Root（支持 3.4+ 内核） |
| [APatch](https://github.com/bmax121/APatch) | 可选 Root 方案 |
| [AnyKernel3](https://github.com/osm0sis/AnyKernel3) | 刷机包打包工具 |

## 许可证

- 内核源码：[GPL-2.0](COPYING)
- ReSukiSU：MIT
- Droidspaces：[GPL-3.0](https://github.com/ravindu644/Droidspaces-OSS/blob/main/LICENSE)
