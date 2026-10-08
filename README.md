# Pixel 3 (blueline) KernelSU-Next 内核云端编译

用 GitHub Actions 云端编译 Pixel 3 的 KernelSU-Next 内核，**本机不需要装 Linux**。

## 为什么需要这样做

Pixel 3 的内核是 **4.9**，而 KernelSU-Next 官方明确说明不提供非 GKI 内核的预编译 boot.img：

> 非 GKI 内核高度碎片化，没有一种通用的构建方式，因此 we cannot provide a non-GKI boot.img

但同时官方也说明 KernelSU-Next 已回移到 4.14 及更早版本，所以 Pixel 3 属于"支持但需自行编译"。

## 关键前提条件

| 项目 | 值 | 说明 |
|---|---|---|
| 内核源码 | `LineageOS/android_kernel_google_msm-4.9` | 分支 `lineage-22.2` |
| 精确 commit | `df5cccf77f2a78959014d9b9206241a128691764` | 与你手机当前内核完全一致 |
| defconfig | `sdm845_defconfig` | 骁龙 845 = sdm845 |
| 工具链 | `clang-r416183b` | LineageOS 官方 prebuilt |
| KSU 分支 | **`legacy`** | ⚠️ 4.9 < 4.14 必须用 legacy |

**为什么用 legacy 分支**：KernelSU-Next 的 `setup.sh` 支持多个分支参数。Pixel 3 的 4.9 内核早于 4.14，只能走 `legacy`。传默认的 latest 会失败。

## 使用步骤

### 1. 创建 GitHub 仓库

在 GitHub 新建一个空仓库（名字随意，例如 `kernelsu-blueline`）。

### 2. 上传工作流

把本目录下的文件放到仓库里，结构必须是：

```
你的仓库/
└── .github/
    └── workflows/
        └── build-kernelsu.yml
```

> `.github` 是隐藏目录，注意不要漏掉 `workflows` 这一层。

### 3. 触发编译

进入仓库 → **Actions** 标签页 → 左侧选择 **Build KernelSU-Next kernel for Pixel 3** → 右侧 **Run workflow**。

参数保持默认即可：
- `ksu_ref` = `legacy`
- `defconfig` = `sdm845_defconfig`

### 4. 等待编译

编译约需 **30-60 分钟**（GitHub 免费额度每月 2000 分钟，够用）。

失败时我会自动上传 `build-log` artifact，可以下载看具体错误。

### 5. 下载产物

编译成功后，在 Actions 页面底部 **Artifacts** 下载 `kernelsu-next-blueline`，里面有两个文件：

| 文件 | 用途 |
|---|---|
| `boot-kernelsu-next-blueline.img` | fastboot 直刷镜像（64MB） |
| `KernelSU-Next-blueline-AnyKernel3.zip` | recovery 里刷的 zip（更安全） |

## 刷入方法

### 方式 A：Recovery 刷入（推荐，更安全）

手机进 recovery → Apply update → Apply from ADB：

```bash
adb sideload KernelSU-Next-blueline-AnyKernel3.zip
```

失败会自动回滚，不容易变砖。

### 方式 B：fastboot 直刷

```bash
adb reboot bootloader
fastboot flash boot boot-kernelsu-next-blueline.img
adb reboot
```

⚠️ Pixel 3 有 A/B 分区，只刷当前槽位。若变砖，可回到fastboot 恢复原kernel。

## 刷完后的操作

1. 下载 KernelSU-Next Manager APK（从官方 release 获取）
2. 安装后如果显示"未安装"，说明需要重启或内核 hook 未生效
3. 打开 Manager → 用 su 给Superuser 权限

## 风险提示

- 编译失败：不会影响手机，只是拿不到文件
- 内核编译成功但不能启动：可通过 fastboot 刷回 LineageOS 官方 boot.img 救砖
- **务必先备份当前可用的 boot.img**，便于随时回退

你的系统里当前 boot 就是官方 LineageOS recovery 之外的 boot 分区内容，
建议先把能用的 boot.img 单独备份一份。

## 常见问题

**Q: `legacy` 分支会不会不稳定？**
A: 官方文档说明 3.18+ 是实验性的。4.9 属于此类，可能需要调试。如果 bootloop，
官方给的方法是注释 `KernelSU/kernel/ksu.c` 里的 `ksu_sucompat_init()` 和 `ksu_ksud_init()`。

**Q: 能改成用 GKI/LKM 模式吗？**
A: 不能。LKM 模式要求 GKI 2.0（5.10+）内核，Pixel 3 的 4.9 不满足。

**Q: 为什么要 pin 精确 commit？**
A: 你手机上运行的是 `4.9.337-gdf5cccf77f2a`，就是这个 commit 编译出来的。
用同一 commit 才能保证内核与系统其他部分完美匹配。