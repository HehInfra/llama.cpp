# 本地构建 llama.cpp

本项目的主要产物是 `llama` 库。其 C 风格接口可在 [include/llama.h](../include/llama.h) 中找到。

该项目还包含许多使用 `llama` 库的示例程序和工具。示例范围从简单的、最小化的代码片段到复杂的子项目，例如兼容 OpenAI 的 HTTP 服务器。

**获取代码：**

```bash
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
```

以下章节介绍如何使用不同的后端和选项进行构建。

* [CPU 构建](#cpu-构建)
* [BLAS 构建](#blas-构建)
* [Metal 构建](#metal-构建)
* [SYCL](#sycl)
* [CUDA](#cuda)
* [MUSA](#musa)
* [HIP](#hip)
* [Vulkan](#vulkan)
* [CANN](#cann)
* [ZenDNN](#zendnn)
* [Arm® KleidiAI™](#arm-kleidiai)
* [OpenCL](#opencl)
* [Android](#android-1)
* [OpenVINO](#openvino)
* [GPU 加速后端说明](#gpu-加速后端说明)

## CPU 构建

使用 `CMake` 构建 llama.cpp：

```bash
cmake -B build
cmake --build build --config Release
```

**注意**：

- 为加快编译速度，请添加 `-j` 参数以并行运行多个作业，或使用会自动执行此操作的生成器，例如 Ninja。例如，`cmake --build build --config Release -j 8` 将并行运行 8 个作业。
- 为加快重复编译速度，请安装 [ccache](https://ccache.dev/)
- 调试构建分两种情况：

    1. 单配置生成器（例如默认 = `Unix Makefiles`；注意它们会忽略 `--config` 标志）：

       ```bash
       cmake -B build -DCMAKE_BUILD_TYPE=Debug
       cmake --build build
       ```

    2. 多配置生成器（`-G` 参数设置为 Visual Studio、XCode...）：

       ```bash
       cmake -B build -G "Xcode"
       cmake --build build --config Debug
       ```

    有关更多详细信息和支持的生成器列表，请参阅 [CMake 文档](https://cmake.org/cmake/help/latest/manual/cmake-generators.7.html)。
- 静态构建请添加 `-DBUILD_SHARED_LIBS=OFF`：
  ```
  cmake -B build -DBUILD_SHARED_LIBS=OFF
  cmake --build build --config Release
  ```

- 使用 MSVC 或 clang 编译器为 Windows（x86、x64 和 arm64）构建：
    - 安装 Visual Studio 2022，例如通过 [Community Edition](https://visualstudio.microsoft.com/vs/community/)。在安装程序中，至少选择以下选项（这也会自动安装所需的附加工具，如 CMake 等）：
    - 工作负载选项卡：使用 C++ 的桌面开发
    - 组件选项卡（可通过搜索快速选择）：适用于 Windows 的 C++ _CMake_ 工具、适用于 Windows 的 _Git_、适用于 Windows 的 C++ _Clang_ 编译器、用于 LLVM 工具集 (clang) 的 MS-Build 支持
    - 请务必始终使用 VS2022 开发人员命令提示符 / PowerShell 进行 git、构建和测试
    - 为 Windows on ARM（arm64、WoA）构建，请使用：
    ```bash
    cmake --preset arm64-windows-llvm-release -D GGML_OPENMP=OFF
    cmake --build build-arm64-windows-llvm-release
    ```
    若要将 ninja 生成器和 clang 编译器作为默认，请进行如下设置：
      -set path:set LIB=C:\Program Files (x86)\Windows Kits\10\Lib\10.0.22621.0\um\x64;C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.41.34120\lib\x64\uwp;C:\Program Files (x86)\Windows Kits\10\Lib\10.0.22621.0\ucrt\x64
      ```bash
      cmake --preset x64-windows-llvm-release
      cmake --build build-x64-windows-llvm-release
      ```
- 如果你需要 HTTPS/TLS 功能，可以安装 OpenSSL 开发库。如果未安装，项目将在没有 SSL 支持的情况下构建和运行。
  - **Debian / Ubuntu：** `sudo apt-get install libssl-dev`
  - **Fedora / RHEL / Rocky / Alma：** `sudo dnf install openssl-devel`
  - **Arch / Manjaro：** `sudo pacman -S openssl`

## BLAS 构建

使用 BLAS 支持构建程序可能会在使用大于 32（默认为 512）的批处理大小进行提示词处理时带来一些性能提升。使用 BLAS 不会影响生成性能。目前有多种不同的 BLAS 实现可用于构建和使用：

### Accelerate Framework

这仅在 Mac 电脑上可用，并且默认启用。你可以直接按照常规说明进行构建。

### OpenBLAS

这仅使用 CPU 提供 BLAS 加速。请确保你的机器上已安装 OpenBLAS。

- 在 Linux 上使用 `CMake`：

    ```bash
    cmake -B build -DGGML_BLAS=ON -DGGML_BLAS_VENDOR=OpenBLAS
    cmake --build build --config Release
    ```

### BLIS

查看 [BLIS.md](./backend/BLIS.md) 了解更多信息。

### Intel oneMKL

通过 oneAPI 编译器进行构建可以使不支持 avx512 和 avx512_vnni 的 Intel 处理器使用 avx_vnni 指令集。请注意，此构建配置**不支持 Intel GPU**。有关 Intel GPU 支持，请参阅 [llama.cpp for SYCL](./backend/SYCL.md)。

- 手动安装 oneAPI：
  默认情况下，`GGML_BLAS_VENDOR` 设置为 `Generic`，因此如果你已经 source 了 Intel 环境脚本并在 cmake 中指定了 `-DGGML_BLAS=ON`，mkl 版本的 Blas 将被自动选择。否则请安装 oneAPI 并按照以下步骤操作：
    ```bash
    source /opt/intel/oneapi/setvars.sh # 如果你处于 oneapi-basekit docker 镜像中，可以跳过此步骤，仅手动安装时需要
    cmake -B build -DGGML_BLAS=ON -DGGML_BLAS_VENDOR=Intel10_64lp -DCMAKE_C_COMPILER=icx -DCMAKE_CXX_COMPILER=icpx -DGGML_NATIVE=ON
    cmake --build build --config Release
    ```

- 使用 oneAPI Docker 镜像：
  如果你不想 source 环境变量并手动安装 oneAPI，也可以使用 Intel Docker 容器进行构建：[oneAPI-basekit](https://hub.docker.com/r/intel/oneapi-basekit)。然后，你可以使用上面给出的命令。

查看 [Optimizing and Running LLaMA2 on Intel® CPU](https://builders.intel.com/solutionslibrary/optimizing-and-running-llama2-on-intel-cpu) 了解更多信息。

### 其他 BLAS 库

任何其他 BLAS 库都可以通过设置 `GGML_BLAS_VENDOR` 选项来使用。有关支持的供应商列表，请参阅 [CMake 文档](https://cmake.org/cmake/help/latest/module/FindBLAS.html#blas-lapack-vendors)。

## Metal 构建

在 macOS 上，Metal 默认启用。使用 Metal 可以使计算在 GPU 上运行。
要在编译时禁用 Metal 构建，请使用 `-DGGML_METAL=OFF` cmake 选项。

使用 Metal 支持构建后，你可以通过 `--n-gpu-layers 0` 命令行参数显式禁用 GPU 推理。

## SYCL

SYCL 是一种更高级别的编程模型，旨在提高各种硬件加速器上的编程效率。

基于 SYCL 的 llama.cpp 用于**支持 Intel GPU**（Data Center Max 系列、Flex 系列、Arc 系列、内置 GPU 和 iGPU）。

有关详细信息，请参阅 [llama.cpp for SYCL](./backend/SYCL.md)。

## CUDA

这使用 NVIDIA GPU 提供 GPU 加速。请确保已安装 [CUDA toolkit](https://developer.nvidia.com/cuda-toolkit)。

#### 直接从 NVIDIA 下载

你可以在以下位置找到官方下载：[NVIDIA developer site](https://developer.nvidia.com/cuda-downloads)。


#### 在 Fedora Toolbox 容器中编译并运行

我们还提供了一份指南，介绍如何在 Fedora [toolbox container](https://containertoolbx.org/) 中设置 CUDA toolkit：[guide](./backend/CUDA-FEDORA.md)。

**推荐用于：**
- 对于 [Atomic Desktops for Fedora](https://fedoraproject.org/atomic-desktops/) 用户是***必需的***；例如：[Silverblue](https://fedoraproject.org/atomic-desktops/silverblue/) 和 [Kinoite](https://fedoraproject.org/atomic-desktops/kinoite/)。
  - （这些系统没有受支持的 CUDA 包）
- 对于主机不是 [Supported Nvidia CUDA Release Platform](https://developer.nvidia.com/cuda-downloads) 的用户是***必需的***。
  - （例如，你的主机操作系统可能是 [Fedora 42 Beta](https://fedoramagazine.org/announcing-fedora-linux-42-beta/)）
- 对于运行 [Fedora Workstation](https://fedoraproject.org/workstation/) 或 [Fedora KDE Plasma Desktop](https://fedoraproject.org/spins/kde) 并希望保持主机系统干净的用户来说***很方便***。
- *可选地* toolbox 包可用：[Arch Linux](https://archlinux.org/)、[Red Hat Enterprise Linux >= 8.5](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux) 或 [Ubuntu](https://ubuntu.com/download)


### 编译

请务必阅读 CPU 构建部分的说明，了解通用指令，例如如何加快编译速度。

```bash
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release
```

### 非原生构建

默认情况下，llama.cpp 会针对当时连接到系统的硬件进行构建。
要构建覆盖所有 CUDA GPU 的版本，请禁用 `GGML_NATIVE`：

```bash
cmake -B build -DGGML_CUDA=ON -DGGML_NATIVE=OFF
```

生成的二进制文件应能在所有 CUDA GPU 上以最佳性能运行，但可能需要进行一些即时编译。

### 覆盖计算能力规格

如果 `nvcc` 无法检测到你的 GPU，你可能会收到如下编译警告：
 ```text
nvcc warning : Cannot find valid GPU for '-arch=native', default arch is used
```

一个选择是按照上述方法进行非原生构建。
但是，这将生成一个需要很长时间编译的大型二进制文件。
Alternatively it is also possible to explicitly specify CUDA architectures.
对于非原生构建，这同样可能有意义，可以查看 `ggml/src/ggml-cuda/CMakeLists.txt` 中的逻辑作为起点。

要覆盖默认 CUDA 架构：

#### 1. 记录你的 NVIDIA 设备的`计算能力`：["CUDA：Your GPU Compute > Capability"](https://developer.nvidia.com/cuda-gpus)。

```text
GeForce RTX 4090      8.9
GeForce RTX 3080 Ti   8.6
GeForce RTX 3070      8.6
```

#### 2. 在 `CMAKE_CUDA_ARCHITECTURES` 列表中手动列出每个不同的`计算能力`。

```bash
cmake -B build -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES="86;89"
```

### 覆盖 CUDA 版本

如果你的系统上有多个 CUDA 安装，并希望为特定版本编译 llama.cpp，例如安装在 `/opt/cuda-11.7` 的 CUDA 11.7：

```bash
cmake -B build -DGGML_CUDA=ON -DCMAKE_CUDA_COMPILER=/opt/cuda-11.7/bin/nvcc -DCMAKE_INSTALL_RPATH="/opt/cuda-11.7/lib64;\$ORIGIN" -DCMAKE_BUILD_WITH_INSTALL_RPATH=ON
```

#### 修复旧版 CUDA 与新版 glibc 的兼容性问题

如果你尝试将旧版 CUDA（例如 v11.7）与新版 glibc 一起使用，可能会遇到如下错误：

```
/usr/include/bits/mathcalls.h(83): error: exception specification is
  incompatible with that of previous function "cospi"


  /opt/cuda-11.7/bin/../targets/x86_64-linux/include/crt/math_functions.h(5545):
  here
```

看起来最不糟糕的解决方案是修补 CUDA 安装以声明正确的签名。
在 `/path/to/your/cuda/installation/targets/x86_64-linux/include/crt/math_functions.h` 中替换以下行：

```C++
// original lines
extern __DEVICE_FUNCTIONS_DECL__ __device_builtin__ double                 cospi(double x);
extern __DEVICE_FUNCTIONS_DECL__ __device_builtin__ float                  cospif(float x);
extern __DEVICE_FUNCTIONS_DECL__ __device_builtin__ double                 sinpi(double x);
extern __DEVICE_FUNCTIONS_DECL__ __device_builtin__ float                  sinpif(float x);
extern __DEVICE_FUNCTIONS_DECL__ __device_builtin__ double                 rsqrt(double x);
extern __DEVICE_FUNCTIONS_DECL__ __device_builtin__ float                  rsqrtf(float x);

// edited lines
extern __DEVICE_FUNCTIONS_DECL__ __device_builtin__ double                 cospi(double x) noexcept (true);
extern __DEVICE_FUNCTIONS_DECL__ __device_builtin__ float                  cospif(float x) noexcept (true);
extern __DEVICE_FUNCTIONS_DECL__ __device_builtin__ double                 sinpi(double x) noexcept (true);
extern __DEVICE_FUNCTIONS_DECL__ __device_builtin__ float                  sinpif(float x) noexcept (true);
extern __DEVICE_FUNCTIONS_DECL__ __device_builtin__ double                 rsqrt(double x) noexcept (true);
extern __DEVICE_FUNCTIONS_DECL__ __device_builtin__ float                  rsqrtf(float x) noexcept (true);
```

### CUDA 运行时环境变量

你可以在运行时设置 [cuda environmental variables](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#env-vars)。

```bash
# 使用 `CUDA_VISIBLE_DEVICES` 隐藏第一个计算设备。
CUDA_VISIBLE_DEVICES="-0" ./build/bin/llama-server --model /srv/models/llama.gguf
```

#### CUDA_SCALE_LAUNCH_QUEUES

环境变量 [`CUDA_SCALE_LAUNCH_QUEUES`](https://docs.nvidia.com/cuda/cuda-programming-guide/05-appendices/environment-variables.html#cuda-scale-launch-queues) 控制 CUDA 命令缓冲区的大小，它决定了在 CPU 必须等待 GPU 赶上之前可以排队多少个 GPU 操作。较大的缓冲区可减少 CPU 端的停顿，并允许在 GPU 上排队更多工作。

考虑设置 `CUDA_SCALE_LAUNCH_QUEUES=4x`，这会将 CUDA 命令缓冲区增加到默认大小的 4 倍。此优化对于**使用流水线并行的多 GPU 设置**特别有益，因为它允许跨 GPU 排队更多操作，从而显著提高提示词处理吞吐量。

#### GGML_CUDA_FORCE_CUBLAS_COMPUTE_32F

使用环境变量 `GGML_CUDA_FORCE_CUBLAS_COMPUTE_32F` 可在所有 GPU 的 FP16 cuBLAS 中使用 FP32 计算类型，以防止可能的数值溢出，代价是提示词处理速度变慢（对 RTX PRO/数据中心产品影响较小，对 GeForce 产品影响显著）。

#### GGML_CUDA_FORCE_CUBLAS_COMPUTE_16F

使用环境变量 `GGML_CUDA_FORCE_CUBLAS_COMPUTE_16F` 可强制在 V100、CDNA 和 RDNA4 的 FP16 cuBLAS 中使用 FP16 计算类型（而不是默认的 FP32）。

### 统一内存

环境变量 `GGML_CUDA_ENABLE_UNIFIED_MEMORY=1` 可用于在 Linux 中启用统一内存。当 GPU 显存耗尽时，这允许交换到系统内存而不是崩溃。在 Windows 中，此设置可在 NVIDIA 控制面板中以 `System Memory Fallback` 的形式使用。

### 点对点访问

环境变量 `GGML_CUDA_P2P` 可以设置为启用多个 GPU 之间的点对点访问，使它们能够直接传输数据，而不必经过系统内存。
需要驱动支持（通常限于工作站/数据中心 GPU）。
某些主板和 BIOS 设置（例如 IOMMU）可能会导致崩溃或输出损坏。

### 性能调优

还提供了以下编译选项用于调整性能：

| 选项                          | 合法值           | 默认值  | 说明                                                                                                                                                                                                                                                |
|-------------------------------|------------------|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| GGML_CUDA_FORCE_MMQ           | Boolean          | false   | 强制使用自定义矩阵乘法核处理量化模型，即使没有可用的 int8 tensor core 实现也使用，而不是 FP16 cuBLAS（影响 V100、CDNA 和 RDNA3+）。对于支持 int8 tensor core 的 GPU，默认启用 MMQ 核。强制启用 MMQ 后，大批量大小下的速度会变差，但显存消耗会更低。 |
| GGML_CUDA_FORCE_CUBLAS        | Boolean          | false   | 强制使用 FP16 cuBLAS 而不是自定义矩阵乘法核处理量化模型。可能会出现数值溢出问题（V100、CDNA 和 RDNA4 默认使用 FP32 计算类型除外），并且内存使用会更高。在最新的数据中心 GPU 上，提示词处理可能会更快（自定义核主要针对 RTX 3000/4000 进行了调优）。 |
| GGML_CUDA_PEER_MAX_BATCH_SIZE | Positive integer | 128     | 启用多 GPU 之间点对点访问的最大批处理大小。点对点访问需要 Linux 或 NVLink。使用 NVLink 时，为较大的批处理大小启用点对点访问可能是有益的。                                                                                                          |
| GGML_CUDA_FA_ALL_QUANTS       | Boolean          | false   | 为 FlashAttention CUDA 核编译支持所有 KV cache 量化类型（组合）。对 KV cache 大小有更细粒度的控制，但编译时间会更长。                                                                                                                              |

## MUSA

这使用摩尔线程 GPU 提供 GPU 加速。请确保已安装 [MUSA SDK](https://developer.mthreads.com/musa/musa-sdk)。

#### 直接从摩尔线程下载

你可以在以下位置找到官方下载：[Moore Threads developer site](https://developer.mthreads.com/sdk/download/musa)。

### 编译

```bash
cmake -B build -DGGML_MUSA=ON
cmake --build build --config Release
```

#### 覆盖计算能力规格

默认情况下，启用所有支持的计算能力。要自定义此行为，可以在 CMake 命令中指定 `MUSA_ARCHITECTURES` 选项：

```bash
cmake -B build -DGGML_MUSA=ON -DMUSA_ARCHITECTURES="21"
cmake --build build --config Release
```

此配置在编译期间仅启用计算能力 `2.1`（MTT S80），有助于减少编译时间。

#### 编译选项

CUDA 可用的大多数编译选项也应该可用于 MUSA，但尚未经过充分测试。

- 静态构建请添加 `-DBUILD_SHARED_LIBS=OFF` 和 `-DCMAKE_POSITION_INDEPENDENT_CODE=ON`：
  ```
  cmake -B build -DGGML_MUSA=ON \
    -DBUILD_SHARED_LIBS=OFF -DCMAKE_POSITION_INDEPENDENT_CODE=ON
  cmake --build build --config Release
  ```

### MUSA 运行时环境变量

你可以在运行时设置 [musa environmental variables](https://docs.mthreads.com/musa-sdk/musa-sdk-doc-online/programming_guide/Z%E9%99%84%E5%BD%95/)。

```bash
# 使用 `MUSA_VISIBLE_DEVICES` 隐藏第一个计算设备。
MUSA_VISIBLE_DEVICES="-0" ./build/bin/llama-server --model /srv/models/llama.gguf
```

### 统一内存

环境变量 `GGML_CUDA_ENABLE_UNIFIED_MEMORY=1` 可用于在 Linux 中启用统一内存。当 GPU 显存耗尽时，这允许交换到系统内存而不是崩溃。

## HIP

这在使用 HIP 的 AMD GPU 上提供 GPU 加速。
请确保已安装 ROCm。
你可以从 Linux 发行版的包管理器下载，或从这里下载：[ROCm Quick Start (Linux)](https://rocm.docs.amd.com/projects/install-on-linux/en/latest/tutorial/quick-start.html#rocm-install-quick)。

- 在 Linux 上使用 `CMake`（假设为兼容 gfx1030 的 AMD GPU）：
  ```bash
  HIPCXX="$(hipconfig -l)/clang" HIP_PATH="$(hipconfig -R)" \
      cmake -S . -B build -DGGML_HIP=ON -DGPU_TARGETS=gfx1030 -DCMAKE_BUILD_TYPE=Release \
      && cmake --build build --config Release -- -j 16
  ```

  注意：`GPU_TARGETS` 是可选的，省略它将为当前系统中的所有 GPU 构建代码。

  为了增强 RDNA3+ 或 CDNA 架构上的 flash attention 性能，你可以通过启用 `-DGGML_HIP_ROCWMMA_FATTN=ON` 选项来使用 rocWMMA 库。这要求构建系统上已安装 rocWMMA 头文件。

  通过 AMD 提供的 `rocm` 元包安装 ROCm SDK 时，默认会包含 rocWMMA 库。或者，如果你不使用元包，可以使用系统的包管理器安装 `rocwmma-dev` 或 `rocwmma-devel` 包。

  作为替代方案，你可以从官方 [GitHub 仓库](https://github.com/ROCm/rocWMMA) 手动克隆该库，签出相应的版本标签（例如 `rocm-6.2.4`），并在 CMake 中设置 `-DCMAKE_CXX_FLAGS="-I<path/to/rocwmma>/library/include/"`。尽管 AMD 未正式支持 Windows，但此方法在 Windows 下也有效。

  注意，如果你遇到以下错误：
  ```
  clang: error: cannot find ROCm device library; provide its path via '--rocm-path' or '--rocm-device-lib-path', or pass '-nogpulib' to build without ROCm device library
  ```
  尝试在 `HIP_PATH` 下搜索包含文件
  `oclc_abi_version_400.bc` 的目录。然后，在命令开头添加
  `HIP_DEVICE_LIB_PATH=<你找到的目录>`，如下所示：
  ```bash
  HIPCXX="$(hipconfig -l)/clang" HIP_PATH="$(hipconfig -p)" \
  HIP_DEVICE_LIB_PATH=<directory-you-just-found> \
      cmake -S . -B build -DGGML_HIP=ON -DGPU_TARGETS=gfx1030 -DCMAKE_BUILD_TYPE=Release \
      && cmake --build build -- -j 16
  ```

- 在 Windows 上使用 `CMake`（使用 VS 的 x64 本机工具命令提示符，并假设为兼容 gfx1100 的 AMD GPU）：
  ```bash
  set PATH=%HIP_PATH%\bin;%PATH%
  cmake -S . -B build -G Ninja -DGPU_TARGETS=gfx1100 -DGGML_HIP=ON -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_BUILD_TYPE=Release
  cmake --build build
  ```
  如果需要，请根据你要编译的 GPU 架构调整 `GPU_TARGETS`。上面的示例使用 `gfx1100`，对应 Radeon RX 7900XTX/XT/GRE。你可以在[这里](https://llvm.org/docs/AMDGPUUsage.html#processors)找到目标列表。
  通过将 `rocminfo | grep gfx | head -1 | awk '{print $2}'` 输出的最重要版本信息与处理器列表匹配来找到你的 GPU 版本字符串，例如 `gfx1035` 映射到 `gfx1030`。


环境变量 [`HIP_VISIBLE_DEVICES`](https://rocm.docs.amd.com/en/latest/understand/gpu_isolation.html#hip-visible-devices) 可用于指定将使用哪些 GPU。
如果你的 GPU 未获得官方支持，可以使用环境变量 [`HSA_OVERRIDE_GFX_VERSION`] 设置为类似的 GPU，例如在 RDNA2（例如 gfx1030、gfx1031 或 gfx1035）上设置为 10.3.0，或在 RDNA3 上设置为 11.0.0。请注意，[`HSA_OVERRIDE_GFX_VERSION`] [在 Windows 上不受支持](https://github.com/ROCm/ROCm/issues/2654)

### 统一内存

在 Linux 上，可以通过设置环境变量 `GGML_CUDA_ENABLE_UNIFIED_MEMORY=1` 来使用统一内存架构（UMA），在 CPU 和集成 GPU 之间共享主内存。但是，这会降低非集成 GPU 的性能（但可以使其与集成 GPU 一起工作）。

## Vulkan

### 对于 Windows 用户：
**w64devkit**

下载并解压 [`w64devkit`](https://github.com/skeeto/w64devkit/releases)。

下载并安装默认设置的 [`Vulkan SDK`](https://vulkan.lunarg.com/sdk/home#windows)。

启动 `w64devkit.exe` 并运行以下命令复制 Vulkan 依赖：
```sh
SDK_VERSION=1.3.283.0
cp /VulkanSDK/$SDK_VERSION/Bin/glslc.exe $W64DEVKIT_HOME/bin/
cp /VulkanSDK/$SDK_VERSION/Lib/vulkan-1.lib $W64DEVKIT_HOME/x86_64-w64-mingw32/lib/
cp -r /VulkanSDK/$SDK_VERSION/Include/* $W64DEVKIT_HOME/x86_64-w64-mingw32/include/
cat > $W64DEVKIT_HOME/x86_64-w64-mingw32/lib/pkgconfig/vulkan.pc <<EOF
Name: Vulkan-Loader
Description: Vulkan Loader
Version: $SDK_VERSION
Libs: -lvulkan-1
EOF

```

切换到 `llama.cpp` 目录并使用 CMake 构建。
```sh
cmake -B build -DGGML_VULKAN=ON
cmake --build build --config Release
```

**Git Bash MINGW64**

下载并安装默认设置的 [`Git-SCM`](https://git-scm.com/downloads/win)。

下载并安装 [`Visual Studio Community Edition`](https://visualstudio.microsoft.com/)，并确保选择 `C++`。

下载并安装默认设置的 [`CMake`](https://cmake.org/download/)。

下载并安装默认设置的 [`Vulkan SDK`](https://vulkan.lunarg.com/sdk/home#windows)。

进入你的 `llama.cpp` 目录并右键单击，选择 `Open Git Bash Here`，然后运行以下命令：

```
cmake -B build -DGGML_VULKAN=ON
cmake --build build --config Release
```

现在你可以使用 `Vulkan` 以对话模式加载模型。

```sh
build/bin/Release/llama-cli -m "[PATH TO MODEL]" -ngl 100 -c 16384 -t 10 -n -2 -cnv
```

**MSYS2**

安装 [MSYS2](https://www.msys2.org/)，然后在 UCRT 终端中运行以下命令安装依赖。
```sh
pacman -S git \
    mingw-w64-ucrt-x86_64-gcc \
    mingw-w64-ucrt-x86_64-cmake \
    mingw-w64-ucrt-x86_64-vulkan-devel \
    mingw-w64-ucrt-x86_64-shaderc \
    mingw-w64-ucrt-x86_64-spirv-headers
```

切换到 `llama.cpp` 目录并使用 CMake 构建。
```sh
cmake -B build -DGGML_VULKAN=ON
cmake --build build --config Release
```

### 对于 Docker 用户：

你无需安装 Vulkan SDK。它将在容器内安装。

```sh
# Build the image
docker build -t llama-cpp-vulkan --target light -f .devops/vulkan.Dockerfile .

# Then, use it:
docker run -it --rm -v "$(pwd):/app:Z" --device /dev/dri/renderD128:/dev/dri/renderD128 --device /dev/dri/card1:/dev/dri/card1 llama-cpp-vulkan -m "/app/models/YOUR_MODEL_FILE" -p "Building a website can be done in 10 simple steps:" -n 400 -e -ngl 33
```

### 对于 Linux 用户：

#### 使用 LunarG Vulkan SDK

首先，按照官方 LunarG 的说明在 [Getting Started with the Linux Tarball Vulkan SDK](https://vulkan.lunarg.com/doc/sdk/latest/linux/getting_started.html) 指南中安装和设置 Vulkan SDK。

> [!IMPORTANT]
> 完成第一步后，请确保在当前终端会话中已对 Vulkan SDK 内的 `setup_env.sh` 文件使用 `source` 命令。否则，构建将无法进行。此外，如果你关闭终端，再次构建时必须执行此步骤。但是，有多种方法可以使此设置持久化。有关更多信息，请参阅第一步中链接的 Vulkan SDK 指南。

#### 使用系统包

在 Debian / Ubuntu 上，你可以使用以下命令安装所需的依赖：
```sh
sudo apt-get install libvulkan-dev glslc spirv-headers
```

Vulkan 后端需要 SPIRV-Headers（`spirv/unified1/spirv.hpp`），仅安装 Vulkan loader 开发包并不总是会拉取它们。其他发行版使用的名称如 `spirv-headers`（Ubuntu / Debian / Arch）或 `spirv-headers-devel`（Fedora / openSUSE）。在 Windows 上，LunarG Vulkan SDK 的 `Include` 目录已经包含这些头文件。

#### 通用步骤

其次，在确认你已按照所有 SDK 安装/设置步骤操作后，请使用以下命令确保无误：
```bash
vulkaninfo
```

然后，假设你已 `cd` 到 llama.cpp 文件夹并且运行 `vulkaninfo` 没有错误，你可以继续使用下面的 CMake 命令构建 llama.cpp：
```bash
cmake -B build -DGGML_VULKAN=1
cmake --build build --config Release
```

最后，完成构建后，你应该可以执行类似以下操作：
```bash
# 测试输出二进制文件
# 对于大多数（如果不是全部）模型，"-ngl 99" 应将所有层都卸载到 GPU。
./build/bin/llama-cli -m "PATH_TO_MODEL" -p "Hi you how are you" -ngl 99

# 你应该会在输出中看到 ggml_vulkan 检测到了你的 GPU。例如：
# ggml_vulkan: Using Intel(R) Graphics (ADL GT2) | uma: 1 | fp16: 1 | warp size: 32
```

### 对于 Mac 用户：

通常，请按照 LunarG 的 [Getting Started with the MacOS Vulkan SDK](https://vulkan.lunarg.com/doc/sdk/latest/mac/getting_started.html) 指南安装和设置 Vulkan SDK。macOS 上有两种 Vulkan 驱动程序可供选择，它们都实现了将 Vulkan 映射到 Metal 的转换层。可以通过将 `VK_ICD_FILENAMES` 环境变量设置为指向相应的 ICD JSON 文件来在它们之间热切换。

在 LunarG Vulkan SDK 安装过程中勾选 "KosmicKrisp"。

安装后设置 LunarG Vulkan SDK 的环境变量（并可选地添加到你的 shell 配置文件中以持久化）：
```bash
source /path/to/vulkan-sdk/setup-env.sh
```

#### 使用 MoltenVK

MoltenVK 是 macOS 上 LunarG Vulkan SDK 安装时默认的 Vulkan 驱动程序，因此你可以按上述方式设置环境变量。

#### 使用 KosmicKrisp

为 KosmicKrisp 覆盖环境变量：
```bash
export VK_ICD_FILENAMES=$VULKAN_SDK/share/vulkan/icd.d/libkosmickrisp_icd.json
export VK_DRIVER_FILES=$VULKAN_SDK/share/vulkan/icd.d/libkosmickrisp_icd.json
```

#### 构建

这是唯一与[上面](#通用步骤)说明不同的步骤。
```bash
cmake -B build -DGGML_VULKAN=1 -DGGML_METAL=OFF
cmake --build build --config Release
```

## CANN

这使用昇腾 NPU 的 AI 核心提供 NPU 加速。[CANN](https://www.hiascend.com/en/software/cann) 是一套分层的 API，可帮助你基于昇腾 NPU 快速构建 AI 应用和服务。

有关昇腾 NPU 的更多信息，请访问 [Ascend Community](https://www.hiascend.com/en/)。

请确保已安装 CANN toolkit。你可以从这里下载：[CANN Toolkit](https://www.hiascend.com/developer/download/community/result?module=cann)

进入 `llama.cpp` 目录并使用 CMake 构建。
```bash
cmake -B build -DGGML_CANN=on -DCMAKE_BUILD_TYPE=release
cmake --build build --config release
```

你可以通过以下命令进行测试：

```bash
./build/bin/llama-cli -m PATH_TO_MODEL -p "Building a website can be done in 10 steps:" -ngl 32
```

如果屏幕上输出以下信息，则表示你正在使用 CANN 后端的 `llama.cpp`：
```bash
llm_load_tensors:       CANN model buffer size = 13313.00 MiB
llama_new_context_with_model:       CANN compute buffer size =  1260.81 MiB
```

有关详细信息，例如模型/设备支持、CANN 安装等，请参阅 [llama.cpp for CANN](./backend/CANN.md)。

## ZenDNN

ZenDNN 为 AMD EPYC™ CPU 提供优化的深度学习原语。它可加速推理工作负载中的矩阵乘法运算。

### 编译

- 在 Linux 上使用 `CMake`（自动构建）：

    ```bash
    cmake -B build -DGGML_ZENDNN=ON
    cmake --build build --config Release
    ```

    首次构建将自动下载并构建 ZenDNN，可能需要 5-10 分钟。后续构建会快得多。

- 在 `CMake` 中使用自定义 ZenDNN 安装：

    ```bash
    cmake -B build -DGGML_ZENDNN=ON -DZENDNN_ROOT=/path/to/zendnn/install
    cmake --build build --config Release
    ```

### 测试

你可以通过以下命令进行测试：

```bash
./build/bin/llama-cli -m PATH_TO_MODEL -p "Building a website can be done in 10 steps:" -n 50
```

有关硬件支持、设置说明和性能优化的详细信息，请参阅 [llama.cpp for ZenDNN](./backend/ZenDNN.md)。

## Arm® KleidiAI™

KleidiAI 是一个针对 AI 工作负载优化的微内核库，专为 Arm CPU 设计。这些微内核可提升性能，并可以由 CPU 后端启用。

要启用 KleidiAI，请进入 llama.cpp 目录并使用 CMake 构建。
```bash
cmake -B build -DGGML_CPU_KLEIDIAI=ON
cmake --build build --config Release
```

你可以通过运行以下命令验证 KleidiAI 是否正在使用：
```bash
./build/bin/llama-cli -m PATH_TO_MODEL -p "What is a car?"
```

如果启用了 KleidiAI，输出中将包含类似以下的一行：
```
load_tensors: CPU_KLEIDIAI model buffer size =  3474.00 MiB
```

KleidiAI 的微内核使用 Arm CPU 特性（如 dotprod、int8mm、SVE 和 SME）实现优化的张量操作。Llama.cpp 会根据检测到的 CPU 能力在运行时选择最高效的核。
在支持 SME 的 CPU 上，SME 微内核会通过运行时检测自动启用。
环境变量 GGML_KLEIDIAI_SME 可用于控制 SME 行为：
- 未设置：如果支持并检测到，则自动启用 SME。
- 0：禁用 SME。
- <n> > 0：启用 SME 并假设有 <n> 个可用的 SME 单元（覆盖自动检测）。
如果 CPU 不支持 SME，SME 微内核将始终被禁用。

根据你的构建目标，其他更高优先级的后端可能会默认启用。要确保使用 CPU 后端，你必须在编译时禁用更高优先级的后端，例如 `-DGGML_METAL=OFF`，或者在运行时使用命令行选项 `--device none`。

## OpenCL

这通过 OpenCL 在较新的 Adreno GPU 上提供 GPU 加速。有关 OpenCL 后端的更多信息，请参阅 [OPENCL.md](./backend/OPENCL.md)。

### Android

假设 NDK 位于 `$ANDROID_NDK`。首先，如果尚未安装 OpenCL 头文件和 ICD loader 库，请先安装：

```sh
mkdir -p ~/dev/llm
cd ~/dev/llm

git clone https://github.com/KhronosGroup/OpenCL-Headers && \
cd OpenCL-Headers && \
cp -r CL $ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/include

cd ~/dev/llm

git clone https://github.com/KhronosGroup/OpenCL-ICD-Loader && \
cd OpenCL-ICD-Loader && \
mkdir build_ndk && cd build_ndk && \
cmake .. -G Ninja -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
  -DOPENCL_ICD_LOADER_HEADERS_DIR=$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/include \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=24 \
  -DANDROID_STL=c++_shared && \
ninja && \
cp libOpenCL.so $ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/aarch64-linux-android
```

然后使用启用 OpenCL 的选项构建 llama.cpp，

```sh
cd ~/dev/llm

git clone https://github.com/ggml-org/llama.cpp && \
cd llama.cpp && \
mkdir build-android && cd build-android

cmake .. -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-28 \
  -DBUILD_SHARED_LIBS=OFF \
  -DGGML_OPENCL=ON

ninja
```

### Windows Arm64

首先，如果尚未安装 OpenCL 头文件和 ICD loader 库，请先安装：

```powershell
mkdir -p ~/dev/llm

cd ~/dev/llm
git clone https://github.com/KhronosGroup/OpenCL-Headers && cd OpenCL-Headers
mkdir build && cd build
cmake .. -G Ninja `
  -DBUILD_TESTING=OFF `
  -DOPENCL_HEADERS_BUILD_TESTING=OFF `
  -DOPENCL_HEADERS_BUILD_CXX_TESTS=OFF `
  -DCMAKE_INSTALL_PREFIX="$HOME/dev/llm/opencl"
cmake --build . --target install

cd ~/dev/llm
git clone https://github.com/KhronosGroup/OpenCL-ICD-Loader && cd OpenCL-ICD-Loader
mkdir build && cd build
cmake .. -G Ninja `
  -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_PREFIX_PATH="$HOME/dev/llm/opencl" `
  -DCMAKE_INSTALL_PREFIX="$HOME/dev/llm/opencl"
cmake --build . --target install
```

然后使用启用 OpenCL 的选项构建 llama.cpp，

```powershell
cmake .. -G Ninja `
  -DCMAKE_TOOLCHAIN_FILE="$HOME/dev/llm/llama.cpp/cmake/arm64-windows-llvm.cmake" `
  -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_PREFIX_PATH="$HOME/dev/llm/opencl" `
  -DBUILD_SHARED_LIBS=OFF `
  -DGGML_OPENCL=ON
ninja
```

## Android

要阅读有关如何在 Android 上构建的文档，[请点击这里](./android.md)

## WebGPU

WebGPU 后端依赖 [Dawn](https://dawn.googlesource.com/dawn)。请按照[这里](https://dawn.googlesource.com/dawn/+/refs/heads/main/docs/quickstart-cmake.md)的说明在本地安装 Dawn，以便 llama.cpp 可以使用 CMake 找到它。当前实现与 Dawn commit `18eb229` 保持同步。

在 llama.cpp 目录中，使用 CMake 构建：

```
cmake -B build -DGGML_WEBGPU=ON
cmake --build build --config Release
```

### 浏览器支持

WebGPU 允许从支持的浏览器跨平台访问 GPU。我们使用 [Emscripten](https://emscripten.org/) 将 ggml 的 WebGPU 后端编译为 WebAssembly。Emscripten 尚未正式支持 WebGPU 绑定，但 Dawn 目前维护自己的 WebGPU 绑定，称为 emdawnwebgpu。

请按照[这里](https://dawn.googlesource.com/dawn/+/refs/heads/main/src/emdawnwebgpu/)的说明下载或构建 emdawnwebgpu 包（注意，在本地构建 emdawnwebgpu 包可能更安全，以便它与你上面安装的 Dawn 版本保持同步）。使用 CMake 构建时，需要使用 `EMDAWNWEBGPU_DIR` 标志设置 emdawnwebgpu port 文件的路径。

## IBM Z & LinuxONE

要阅读有关如何在 IBM Z & LinuxONE 上构建的文档，[请点击这里](./build-s390x.md)

## OpenVINO

[OpenVINO](https://docs.openvino.ai/) 是一个用于优化和部署高性能 AI 推理的开源工具包，专为 Intel 硬件（CPU、GPU 和 NPU）设计。

有关构建说明和用法示例，请参阅 [OPENVINO.md](backend/OPENVINO.md)。


---
## GPU 加速后端说明

即使使用 `-ngl 0` 选项，GPU 仍可能用于加速部分计算。你可以通过 `--device none` 完全禁用 GPU 加速。

在大多数情况下，可以同时构建和使用多个后端。例如，你可以通过 CMake 使用 `-DGGML_CUDA=ON -DGGML_VULKAN=ON` 选项同时构建支持 CUDA 和 Vulkan 的 llama.cpp。在运行时，你可以使用 `--device` 选项指定要使用的后端设备。要查看可用设备列表，请使用 `--list-devices` 选项。

后端可以作为动态库构建，以便在运行时动态加载。这允许你在具有不同 GPU 的不同机器上使用相同的 llama.cpp 二进制文件。要启用此功能，请在构建时使用 `GGML_BACKEND_DL` 选项。
