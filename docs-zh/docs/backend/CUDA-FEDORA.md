# 在 Fedora 上设置 CUDA

本指南介绍如何在 toolbox 容器中安装 [Nvidia CUDA](https://docs.nvidia.com/cuda/)。本指南适用于：

- [Fedora Workstation](https://fedoraproject.org/workstation/)
- [Atomic Desktops for Fedora](https://fedoraproject.org/atomic-desktops/)
- [Fedora Spins](https://fedoraproject.org/spins)
- [其他发行版](https://containertoolbx.org/distros/)，包括 `Red Hat Enterprise Linux >= 8.5`、`Arch Linux` 和 `Ubuntu`。

## 目录

- [前置条件](#前置条件)
- [使用 Fedora 41 CUDA 仓库](#使用-fedora-41-cuda-仓库)
- [创建 Fedora Toolbox 环境](#创建-fedora-toolbox-环境)
- [安装基础开发工具](#安装基础开发工具)
- [添加 CUDA 仓库](#添加-cuda-仓库)
- [安装 Nvidia 驱动库](#安装-nvidia-驱动库)
- [安装 CUDA 元包](#安装-cuda-元包)
- [配置环境](#配置环境)
- [验证安装](#验证安装)
- [总结](#总结)
- [故障排查](#故障排查)
- [补充说明](#补充说明)
- [参考资料](#参考资料)

## 前置条件

- **宿主系统已安装 Toolbox**：`Fedora Silverblue` 和 `Fedora Workstation` 默认已安装 toolbox，其他发行版可能需要安装 [toolbox 包](https://containertoolbx.org/install/)。
- **宿主系统已安装 NVIDIA 驱动和显卡（推荐）**：要运行 CUDA 程序（如 `llama.cpp`），宿主系统应已配置为可访问 NVIDIA 硬件。Fedora 宿主可以使用 [RPM Fusion 仓库](https://rpmfusion.org/Howto/NVIDIA)。
- **网络连接**：用于下载软件包。

### 使用 Fedora 41 CUDA 仓库

最新版本为 41。

- [Fedora 41 CUDA 仓库](https://developer.download.nvidia.com/compute/cuda/repos/fedora41/x86_64/)

**注意：** 建议使用 toolbox 环境，以避免与系统发生冲突。

## 创建 Fedora Toolbox 环境

本指南主要针对 Fedora 宿主，但稍作调整后也可用于其他宿主。使用 Fedora Toolbox 可以安装所需软件包，而不会影响宿主系统。

**注意：** Toolbox 也适用于其他系统；即使没有 Toolbox，也可以使用 Podman 或 Docker。

1. **创建 Fedora 41 Toolbox：**

   ```bash
   toolbox create --image registry.fedoraproject.org/fedora-toolbox:41 --container fedora-toolbox-41-cuda
   ```

2. **进入 Toolbox：**

   ```bash
   toolbox enter --container fedora-toolbox-41-cuda
   ```

   进入 toolbox 后，你拥有 root 权限，可以安装软件包而不会影响宿主系统。

## 安装基础开发工具

1. **同步 DNF 包管理器：**

   ```bash
   sudo dnf distro-sync
   ```

2. **安装默认文本编辑器 **Vim**（可选）：**

   ```bash
   sudo dnf install vim-default-editor --allowerasing
   ```

   `--allowerasing` 标志允许删除与之冲突的 `nano-default-editor` 包。

3. **安装开发工具和库：**

   ```bash
   sudo dnf install @c-development @development-tools cmake
   ```

   这会安装编译软件所需的基础包，包括 `gcc`、`make` 和其他开发头文件。

## 添加 CUDA 仓库

将 NVIDIA CUDA 仓库添加到你的 DNF 配置中：

```bash
sudo dnf config-manager addrepo --from-repofile=https://developer.download.nvidia.com/compute/cuda/repos/fedora41/x86_64/cuda-fedora41.repo
```

添加仓库后，再次同步包管理器：

```bash
sudo dnf distro-sync
```

## 安装 Nvidia 驱动库

首先，需要检测宿主系统是否已将 [NVIDIA 驱动库传递到 toolbox](https://github.com/containers/toolbox/blob/main/src/pkg/nvidia/nvidia.go) 中：

```bash
ls -la /usr/lib64/libcuda.so.1
```

### 如果 *`libcuda.so.1`* 缺失：

```
ls: cannot access '/usr/lib64/libcuda.so.1': No such file or directory
```

**说明：**
宿主系统未提供 CUDA 驱动，**请立即安装：**

#### 在 Guest 中安装 Nvidia 驱动库：

```bash
sudo dnf install nvidia-driver-cuda nvidia-driver-libs nvidia-driver-cuda-libs nvidia-persistenced
```

### 如果 *`libcuda.so.1`* 存在：

```
lrwxrwxrwx. 1 root root 21 Mar 24 11:26 /usr/lib64/libcuda.so.1 -> libcuda.so.570.133.07
```

**说明：**
宿主系统已提供 CUDA 驱动，**我们需要相应地更新 guest 的 RPM 数据库：**

#### 更新 Toolbox 的 RPM 数据库以包含宿主提供的库：

注意：我们并不真正安装这些库，只是更新数据库，让 guest 系统知道它们已由宿主提供。

##### 1. 下载宿主 RPM 提供的 `nvidia-` 相关部分（含依赖）

```bash
sudo dnf download --destdir=/tmp/nvidia-driver-libs --resolve --arch x86_64 nvidia-driver-cuda nvidia-driver-libs nvidia-driver-cuda-libs nvidia-persistenced
```

##### 2. 更新 RPM 数据库，使其假定这些包已安装。

```bash
sudo rpm --install --verbose --hash --justdb /tmp/nvidia-driver-libs/*
```

**注意：**

- `--justdb` 选项仅更新 RPM 数据库，不会修改文件系统的其他部分。

##### 检查 RPM 数据库是否已正确更新：

**注意：** 该命令与 *`libcuda.so.1`* 缺失时“在 Guest 中安装 Nvidia 驱动库”中的命令相同。

```bash
sudo dnf install nvidia-driver-cuda nvidia-driver-libs nvidia-driver-cuda-libs nvidia-persistenced
```

*（这次不会安装任何内容，因为数据库认为这些包已经安装）*

```
Updating and loading repositories:
Repositories loaded.
Package "nvidia-driver-cuda-3:570.124.06-1.fc41.x86_64" is already installed.
Package "nvidia-driver-libs-3:570.124.06-1.fc41.x86_64" is already installed.
Package "nvidia-driver-cuda-libs-3:570.124.06-1.fc41.x86_64" is already installed.
Package "nvidia-persistenced-3:570.124.06-1.fc41.x86_64" is already installed.

Nothing to do.
```

## 安装 CUDA 元包

驱动库安装完成后，继续安装 CUDA：

```bash
sudo dnf install cuda
```

这会安装 CUDA 工具包及其相关软件包。

## 配置环境

要使用 CUDA，需要将其二进制目录添加到系统的 `PATH` 中。

1. **创建 profile 脚本：**

   ```bash
   sudo sh -c 'echo "export PATH=\$PATH:/usr/local/cuda/bin" >> /etc/profile.d/cuda.sh'
   ```

   **说明：**

   - 我们选择添加到 `/etc/profile.d/`，因为 `/etc/` 目录是此容器独有的，不会与其他容器或宿主系统共享。
   - `$PATH` 前的反斜杠 `\` 确保该变量被正确写入脚本。

2. **赋予脚本可执行权限：**

   ```bash
   sudo chmod +x /etc/profile.d/cuda.sh
   ```

3. **加载脚本以更新当前环境：**

   ```bash
   source /etc/profile.d/cuda.sh
   ```

   **注意：** 该命令会用新的 `PATH` 更新当前 shell 会话。`/etc/profile.d/cuda.sh` 脚本会确保 CUDA 二进制文件在你所有后续会话的 `PATH` 中可用。

## 验证安装

要确认 CUDA 已正确安装和配置，请检查 NVIDIA CUDA 编译器（`nvcc`）的版本：

```bash
nvcc --version
```

你应该看到类似如下输出：

```
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2025 NVIDIA Corporation
Built on Fri_Feb_21_20:23:50_PST_2025
Cuda compilation tools, release 12.8, V12.8.93
Build cuda_12.8.r12.8/compiler.35583870_0
```

该输出表明 CUDA 编译器可访问，并显示了已安装的版本。

## 总结

你已成功在 Fedora 的 toolbox 环境中，通过 Fedora 41 CUDA 仓库完成了 CUDA 设置。通过手动更新 RPM 数据库并配置环境，你可以在不影响宿主系统的情况下开发 CUDA 应用。

## 故障排查

- **安装失败：**

  - 如果在安装过程中遇到错误，请仔细阅读错误信息。它们通常会指出文件冲突或缺少依赖。
  - 可以在手动安装 RPM 时使用 `--excludepath` 选项来排除冲突文件。

- **重启容器：**

  - 有时 NVIDIA 驱动的宿主透传可能存在 bug（例如缺少某个共享库）。重启容器可能会解决此问题：

  ```bash
  # on the host system
  podman container restart --all
  ```

- **环境变量未设置：**
  - 如果安装后找不到 `nvcc`，请确保 `/usr/local/cuda/bin` 已在你的 `PATH` 中。
  - 运行 `echo $PATH` 检查该路径是否已包含。
  - 重新加载 profile 脚本或打开新的终端会话。

## 补充说明

- **未来更新 CUDA：**

  - 关注 NVIDIA 官方仓库中适用于你的 Fedora 版本的更新。
  - 当有可用的新仓库时，相应地调整你的 `dnf` 配置。

- **构建 `llama.cpp`：**

  - 安装 CUDA 后，你可以按照 [`llama.cpp` 的构建说明](https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md) 编译带有 CUDA 支持的版本。
  - 确保构建配置中正确设置了所有 CUDA 相关的构建标志和路径。

- **使用 Toolbox 环境：**
  - Toolbox 环境与宿主系统隔离，有助于避免冲突。
  - 请记住，toolbox 内的系统文件和配置与宿主是分离的。默认情况下，用户的主目录在宿主和 toolbox 之间共享。

---

**免责声明：** 手动安装和修改系统软件包可能导致容器不稳定。以上步骤仅供参考，可能需要根据你的具体系统配置进行调整。在进行重大系统更改之前，请务必备份重要数据，尤其是主目录可写并与 toolbox 共享。

**致谢：** 特别感谢 Fedora 社区和 NVIDIA 文档为本指南的编写提供资源。

## 参考资料

- [Fedora Toolbox 文档](https://docs.fedoraproject.org/en-US/fedora-silverblue/toolbox/)
- [NVIDIA CUDA 安装指南](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html)
- [Podman 文档](https://podman.io/get-started)

---
