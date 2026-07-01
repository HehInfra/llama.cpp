# 适用于 SYCL 的 llama.cpp

- [背景](#background)
- [推荐版本](#recommended-release)
- [更新动态](#news)
- [操作系统](#os)
- [硬件](#hardware)
- [性能参考](#performance-reference)
- [Docker](#docker)
- [Linux](#linux)
- [Windows](#windows-1)
- [环境变量](#environment-variable)
- [设计规范](#design-rule)
- [已知问题](#known-issues)
- [常见问题](#qa)
- [待办事项](#todo)

## 背景

**SYCL** 是一种高级并行编程模型，旨在提高开发者在 CPU、GPU 和 FPGA 等多种硬件加速器上编写代码的生产力。它是一种面向异构计算的单源语言，基于标准 C++17。

**oneAPI** 是一个开放的生态系统和基于标准的规范，支持包括 Intel CPU、GPU 和 FPGA 在内的多种架构。oneAPI 生态系统的关键组件包括：

- **DPCPP** *(Data Parallel C++)*：主要的 oneAPI SYCL 实现，包含 icpx/icx 编译器。
- **oneAPI Libraries**：一套面向多个领域的高度优化库 *(例如 Intel oneMKL、oneMath 和 oneDNN)*。
- **oneAPI LevelZero**：一个高性能底层接口，用于对 Intel 核显和独显进行细粒度控制。

### Llama.cpp + SYCL

llama.cpp 的 SYCL 后端主要为 **Intel GPU** 设计。
SYCL 的跨平台能力也使其能够支持其他厂商的 GPU。

## 推荐版本

### Windows

以下版本已经过验证并推荐使用：

|Commit ID|Tag|Release|验证平台|更新日期|
|-|-|-|-|-|
|24e86cae7219b0f3ede1d5abdf5bf3ad515cccb8|b5377 |[llama-b5377-bin-win-sycl-x64.zip](https://github.com/ggml-org/llama.cpp/releases/download/b5377/llama-b5377-bin-win-sycl-x64.zip) |Arc B580/Linux/oneAPI 2025.1<br>LNL Arc GPU/Windows 11/oneAPI 2025.1.1|2025-05-15|
|3bcd40b3c593d14261fb2abfabad3c0fb5b9e318|b4040 |[llama-b4040-bin-win-sycl-x64.zip](https://github.com/ggml-org/llama.cpp/releases/download/b4040/llama-b4040-bin-win-sycl-x64.zip) |Arc A770/Linux/oneAPI 2024.1<br>MTL Arc GPU/Windows 11/oneAPI 2024.1| 2024-11-19|
|fb76ec31a9914b7761c1727303ab30380fd4f05c|b3038 |[llama-b3038-bin-win-sycl-x64.zip](https://github.com/ggml-org/llama.cpp/releases/download/b3038/llama-b3038-bin-win-sycl-x64.zip) |Arc A770/Linux/oneAPI 2024.1<br>MTL Arc GPU/Windows 11/oneAPI 2024.1||

### Ubuntu 24.04

Ubuntu 24.04 x64（FP32/FP16）的发布包仅包含 llama.cpp SYCL 后端的二进制文件。目标机器需要预先安装与构建包版本相同的 Intel GPU 驱动和 oneAPI 包。要获取版本和安装信息，请参考 release.yml：ubuntu-24-sycl -> Download & Install oneAPI。

建议与 Intel Docker 一起使用。

FP32 和 FP16 包在 LLM 上具有不同的精度和性能。请根据测试结果进行选择。

## 更新动态

- 2026.04-05
  - 通过重排序特性优化数据类型 Q4_K、Q5_K、Q6_K、Q8_0 的 mul_mat。
  - 融合 MoE。
  - 升级 CI 并为 oneAPI 2025.3.3 构建包，支持 Ubuntu 24.04 构建包。

- 2026.03
  - 支持 Flash-Attention：减少内存占用，性能影响取决于 LLM。

- 2026.02
  - 移除对 Nvidia 和 AMD GPU 的支持，因为 Nvidia 和 AMD GPU 的 oneAPI 插件已不可用：下载/安装渠道已失效。用户无法为 Nvidia 和 AMD GPU 构建该软件。

- 2025.11
  - 支持在设备上分配超过 4GB 的内存。

- 2025.2
  - 针对自 MTL 起的所有 Intel 独立显卡和内置显卡优化 MUL_MAT Q4_0。在 Intel GPU（MTL、ARL-H、Arc、Flex、PVC）上，LLM（llama-2-7b.Q4_0.gguf）的性能提升 21%-87%。
    |GPU|基础 tokens/s|提升后 tokens/s|百分比|
    |-|-|-|-|
    |PVC 1550|39|73|+87%|
    |Flex 170|39|50|+28%|
    |Arc A770|42|55|+30%|
    |MTL|13|16|+23%|
    |ARL-H|14|17|+21%|

- 2024.11
  - 使用 syclcompat 提升部分平台性能。需要使用 oneAPI 2025.0 或更新版本。

- 2024.8
  - 使用 oneDNN 作为默认 GEMM 库，提升对新 Intel GPU 的兼容性。

- 2024.5
  - 性能提升：Arc A770 上 llama-2-7b.Q4_0 达到 34 -> 37 tokens/s。
  - Arch Linux 验证成功。

- 2024.4
  - 支持数据类型：GGML_TYPE_IQ4_NL, GGML_TYPE_IQ4_XS, GGML_TYPE_IQ3_XXS, GGML_TYPE_IQ3_S, GGML_TYPE_IQ2_XXS, GGML_TYPE_IQ2_XS, GGML_TYPE_IQ2_S, GGML_TYPE_IQ1_S, GGML_TYPE_IQ1_M。

- 2024.3
  - 发布 Windows 二进制文件。
  - 博客发布：**Run LLM on all Intel GPUs Using llama.cpp**：[intel.com](https://www.intel.com/content/www/us/en/developer/articles/technical/run-llm-on-all-gpus-using-llama-cpp-artical.html) 或 [medium.com](https://medium.com/@jianyu_neo/run-llm-on-all-intel-gpus-using-llama-cpp-fd2e2dcbd9bd)。
  - 新基线已就绪：[tag b2437](https://github.com/ggml-org/llama.cpp/tree/b2437)。
  - 支持多卡：**--split-mode**：[none|layer]；不支持 [row]，正在开发中。
  - 支持通过 **--main-gpu** 指定主 GPU，替代 $GGML_SYCL_DEVICE。
  - 支持通过 level-zero 检测所有具有相同最大计算单元的 GPU。
  - 支持算子
    - hardsigmoid
    - hardswish
    - pool2d

- 2024.1
  - 为 Intel GPU 创建 SYCL 后端。
  - 支持 Windows 构建

## 操作系统

| 操作系统 | 状态  | 已验证                                       |
|---------|---------|------------------------------------------------|
| Linux   | 支持 | Ubuntu 22.04, Fedora Silverblue 39, Arch Linux |
| Windows | 支持 | Windows 11                                     |


## 硬件

### Intel GPU

SYCL 后端支持的 Intel GPU 系列：

- Intel Data Center Max Series
- Intel Flex Series, Arc Series
- Intel Built-in Arc GPU
- Intel Core CPU 中的核显（第 11 代酷睿及更新版本，参考 [oneAPI 支持的 GPU](https://www.intel.com/content/www/us/en/developer/articles/system-requirements/intel-oneapi-base-toolkit-system-requirements.html#inpage-nav-1-1)）。

在较老的 Intel GPU 上，你可以尝试 [OpenCL](/docs/backend/OPENCL.md)，尽管性能不是最优，而且某些 GPU 可能不支持 OpenCL 或没有任何 GPGPU 能力。

#### 已验证设备

| Intel GPU                     | 状态  | 已验证型号                        |
|-------------------------------|---------|---------------------------------------|
| Intel Data Center Max Series  | 支持 | Max 1550, 1100                        |
| Intel Data Center Flex Series | 支持 | Flex 170                              |
| Intel Arc A-Series            | 支持 | Arc A770, Arc A730M, Arc A750         |
| Intel Arc B-Series            | 支持 | Arc B580                              |
| Intel built-in Arc GPU        | 支持 | built-in Arc GPU in Meteor Lake, Arrow Lake, Lunar Lake |
| Intel iGPU                    | 支持 | iGPU in 13700k, 13400, i5-1250P, i7-1260P, i7-1165G7  |

*说明：*

- **内存**
  - 运行大模型时，设备内存是限制因素。加载的模型大小，*`llm_load_tensors: buffer_size`*，会在运行 `./bin/llama-completion` 时显示在日志中。
  - 请确保来自主机的 GPU 共享内存足够大，以容纳模型大小。例如，*llama-2-7b.Q4_0* 在集成 GPU 上至少需要 8.0GB，在独立显卡上至少需要 4.0GB。

- **执行单元 (EU)**
  - 如果核显的 EU 数量少于 80，推理速度可能过慢，不便于实际使用。

### 其他厂商 GPU

不适用

## 性能参考


要获取支持的 LLM、GPU 和性能参考，请查看 [Performance of llama.cpp on Intel GPU with SYCL backend](https://github.com/ggml-org/llama.cpp/discussions/23313)。

你可以直接在讨论中更新测试结果。

## Docker

Docker 构建选项目前仅限于 *Intel GPU* 目标。

### 构建镜像

```sh
# 使用 FP32
docker build -t llama-cpp-sycl --build-arg="GGML_SYCL_F16=OFF" --target light -f .devops/intel.Dockerfile .

# 使用 FP16
docker build -t llama-cpp-sycl --build-arg="GGML_SYCL_F16=ON" --target light -f .devops/intel.Dockerfile .
```

*说明*：

你也可以使用 `.devops/llama-server-intel.Dockerfile`，它用于构建 *"server"* 替代版本。
查看 [Docker 文档](../docker.md) 以了解可用的镜像。

### 运行容器

```sh
# 首先，查找所有 DRI 显卡
ls -la /dev/dri
# 然后，选择你要使用的显卡（例如 /dev/dri/card1）。
docker run -it --rm -v "/path/to/models:/models" --device /dev/dri/renderD128:/dev/dri/renderD128 --device /dev/dri/card0:/dev/dri/card0 llama-cpp-sycl -m /models/7B/ggml-model-q4_0.gguf -p "Building a website can be done in 10 simple steps:" -n 400 -e -ngl 33 -c 4096 -s 0
```

*说明：*
- Docker 已在原生 Linux 上成功测试。WSL 支持尚未验证。
- 你可能需要在**主机**上安装 Intel GPU 驱动（详情请参考 [Linux 配置](#linux)）。

## Linux

### I. 配置环境

1. **安装 GPU 驱动**

  - **Intel GPU**

Intel 数据中心 GPU 驱动安装指南和下载页面可在此处获取：[Get intel dGPU Drivers](https://dgpu-docs.intel.com/driver/installation.html#ubuntu-install-steps)。

*注意*：对于客户端 GPU *(核显和 Arc A 系列)*，请参考 [client iGPU driver installation](https://dgpu-docs.intel.com/driver/client/overview.html)。

安装完成后，将用户添加到 `video` 和 `render` 组。

```sh
sudo usermod -aG render $USER
sudo usermod -aG video $USER
```

*注意*：注销并重新登录以使更改生效。

通过 `clinfo` 验证安装：

```sh
sudo apt install clinfo
sudo clinfo -l
```

示例输出：

```sh
Platform #0: Intel(R) OpenCL Graphics
 `-- Device #0: Intel(R) Arc(TM) A770 Graphics

Platform #0: Intel(R) OpenCL HD Graphics
 `-- Device #0: Intel(R) Iris(R) Xe Graphics [0x9a49]
```

2. **安装 Intel® oneAPI Base toolkit**

SYCL 后端依赖：
  - Intel® oneAPI DPC++/C++ 编译器/运行时。
  - Intel® oneAPI DPC++/C++ 库 (oneDPL)。
  - Intel® oneAPI 深度神经网络库 (oneDNN)。
  - Intel® oneAPI 数学核心库 (oneMKL)。

- **面向 Intel GPU**

以上所有内容均包含在 **Intel® oneAPI Base toolkit** 和 **Intel® Deep Learning Essentials** 包中。

建议安装 **Intel® Deep Learning Essentials**，它仅提供必要的库，体积更小。

可从官方 [Intel® oneAPI Base Toolkit](https://www.intel.com/content/www/us/en/developer/tools/oneapi/base-toolkit.html) 页面获取 **Intel® oneAPI Base toolkit** 和 **Intel® Deep Learning Essentials**。

请按照 Linux 工具包的下载和安装说明进行操作，最好保持默认安装值不变，尤其是安装路径 *（默认为 `/opt/intel/oneapi`）*。

以下指南/代码片段假设使用默认安装值。否则，请确保在适用位置反映必要的更改。

成功安装后，SYCL 将在可用的 Intel 设备上启用，同时包含面向 Intel GPU 的 oneAPI oneDNN 等相关库。

|已验证版本|
|-|
|2025.3.3 |
|2025.2.1|
|2025.1|
|2024.1|

3. **验证安装和环境**

要检查机器上可用的 SYCL 设备，请使用 `sycl-ls` 命令。
```sh
source /opt/intel/oneapi/setvars.sh
sycl-ls
```

- **Intel GPU**

当目标为 Intel GPU 时，用户应在可用的 SYCL 设备中看到一个或多个设备。请确保通过 `sycl-ls` 至少能检测到一个 GPU，例如以下示例输出中的 `[level_zero:gpu]`：

```
[level_zero:gpu][level_zero:0] Intel(R) oneAPI Unified Runtime over Level-Zero, Intel(R) Arc(TM) A770 Graphics 12.55.8 [1.3.29735+27]
[level_zero:gpu][level_zero:1] Intel(R) oneAPI Unified Runtime over Level-Zero, Intel(R) UHD Graphics 730 12.2.0 [1.3.29735+27]
[opencl:cpu][opencl:0] Intel(R) OpenCL, 13th Gen Intel(R) Core(TM) i5-13400 OpenCL 3.0 (Build 0) [2025.20.8.0.06_160000]
[opencl:gpu][opencl:1] Intel(R) OpenCL Graphics, Intel(R) Arc(TM) A770 Graphics OpenCL 3.0 NEO  [24.39.31294]
[opencl:gpu][opencl:2] Intel(R) OpenCL Graphics, Intel(R) UHD Graphics 730 OpenCL 3.0 NEO  [24.39.31294]
```

### II. 构建 llama.cpp

#### Intel GPU

```sh
./examples/sycl/build.sh
```

或

```sh
# 导出相关环境变量
source /opt/intel/oneapi/setvars.sh

# 选项 1：使用 FP32（在大多数情况下推荐以获得更好性能）
cmake -B build -DGGML_SYCL=ON -DCMAKE_C_COMPILER=icx -DCMAKE_CXX_COMPILER=icpx

# 选项 2：使用 FP16
cmake -B build -DGGML_SYCL=ON -DCMAKE_C_COMPILER=icx -DCMAKE_CXX_COMPILER=icpx -DGGML_SYCL_F16=ON

# 构建所有二进制文件
cmake --build build --config Release -j -v
```

运行测试时可能会遇到一些精度问题，这是由于使用了更快的指令引起的，可以通过将环境变量 `SYCL_PROGRAM_COMPILE_OPTIONS` 设置为 `-cl-fp32-correctly-rounded-divide-sqrt` 来规避。

### III. 运行推理

#### 获取并准备模型

你可以参考通用的 [*获取与量化模型*](../../README.md#obtaining-and-quantizing-models) 指南来准备模型，或直接下载已量化的模型，例如 [llama-2-7b.Q4_0.gguf](https://huggingface.co/TheBloke/Llama-2-7B-GGUF/resolve/main/llama-2-7b.Q4_0.gguf?download=true) 或 [Meta-Llama-3-8B-Instruct-Q4_0.gguf](https://huggingface.co/aptha/Meta-Llama-3-8B-Instruct-Q4_0-GGUF/resolve/main/Meta-Llama-3-8B-Instruct-Q4_0.gguf)。

##### 检查设备

1. 启用 oneAPI 运行环境

```sh
source /opt/intel/oneapi/setvars.sh
```

2. 列出设备信息

与原生 `sycl-ls` 类似，可按如下方式查询可用的 SYCL 设备：

```sh
./build/bin/llama-ls-sycl-device
```

该命令仅显示 SYCL 支持的已选后端。默认后端为 level_zero。例如，在具有 2 个 *Intel GPU* 的系统中，输出如下：
```
found 2 SYCL devices:

|  |                  |                                             |Compute   |Max compute|Max work|Max sub|               |
|ID|       Device Type|                                         Name|capability|units      |group   |group  |Global mem size|
|--|------------------|---------------------------------------------|----------|-----------|--------|-------|---------------|
| 0|[level_zero:gpu:0]|               Intel(R) Arc(TM) A770 Graphics|       1.3|        512|    1024|     32|    16225243136|
| 1|[level_zero:gpu:1]|                    Intel(R) UHD Graphics 770|       1.3|         32|     512|     32|    53651849216|
```

#### 选择 level-zero 设备

|选择的设备 ID|设置|
|-|-|
|0|`export ONEAPI_DEVICE_SELECTOR="level_zero:0"` 或不执行任何操作|
|1|`export ONEAPI_DEVICE_SELECTOR="level_zero:1"`|
|0 和 1|`export ONEAPI_DEVICE_SELECTOR="level_zero:0;level_zero:1"`|

#### 执行

选择以下方法之一来运行。

1. 脚本

- 使用设备 0：

```sh
./examples/sycl/test.sh -mg 0
```
- 使用多设备：

```sh
./examples/sycl/test.sh
```

- 运行 llama-server：

```sh
./examples/sycl/start-svr.sh -m PATH/MODEL_FILE
```

2. 命令行
启动推理

有两种设备选择模式：

- 单设备：使用用户指定的一个设备。默认设备 ID 为 0。
- 多设备：自动选择具有相同后端的设备。

在两种设备选择模式下，默认 SYCL 后端均为 level_zero，你可以通过设置环境变量 ONEAPI_DEVICE_SELECTOR 来选择 SYCL 支持的其他后端。

| 设备选择 | 参数                              |
|------------------|----------------------------------------|
| 单设备    | --split-mode none --main-gpu DEVICE_ID |
| 多设备 | --split-mode layer（默认）           |

示例：

- 使用设备 0：

```sh
ZES_ENABLE_SYSMAN=1 ./build/bin/llama-completion -no-cnv -m models/llama-2-7b.Q4_0.gguf -p "Building a website can be done in 10 simple steps:" -n 400 -e -ngl 99 -sm none -mg 0 --mmap
```

- 使用多设备：

```sh
ZES_ENABLE_SYSMAN=1 ./build/bin/llama-completion -no-cnv -m models/llama-2-7b.Q4_0.gguf -p "Building a website can be done in 10 simple steps:" -n 400 -e -ngl 99 -sm layer --mmap
```

*说明：*

- 执行时，请在输出日志中确认所选设备的 ID，例如可能显示为：

```sh
detect 1 SYCL GPUs: [0] with top Max compute units:512
```
或
```sh
use 1 SYCL GPUs: [0] with Max compute units:512
```

## Windows

### 安装 GPU 驱动

Intel GPU 驱动说明指南和下载页面可在此处获取：[Get Intel GPU Drivers](https://www.intel.com/content/www/us/en/products/docs/discrete-gpus/arc/software/drivers.html)。

### 选项 1：直接下载二进制包

从 https://github.com/ggml-org/llama.cpp/releases 下载 Windows 二进制包。

将包解压到本地文件夹，直接运行 llama 工具。参考[运行推理](#iii-run-the-inference-1)。

注意，该包包含 SYCL 运行时和所有依赖的 dll 文件，无需安装 oneAPI 包并激活它们。

### 选项 2：从源代码本地构建。

#### I. 配置环境

1. 安装 Visual Studio

如果你已经安装了较新版本的 Microsoft Visual Studio，可以跳过此步骤。否则，请参考 [Microsoft Visual Studio](https://visualstudio.microsoft.com/) 官方下载页面。

2. 安装 Intel® oneAPI Base toolkit

SYCL 后端依赖：
  - Intel® oneAPI DPC++/C++ 编译器/运行时。
  - Intel® oneAPI DPC++/C++ 库 (oneDPL)。
  - Intel® oneAPI 深度神经网络库 (oneDNN)。
  - Intel® oneAPI 数学核心库 (oneMKL)。

以上所有内容均包含在 **Intel® oneAPI Base toolkit** 和 **Intel® Deep Learning Essentials** 包中。

建议安装 **Intel® Deep Learning Essentials**，它仅提供必要的库，体积更小。

可从官方 [Intel® oneAPI Base Toolkit](https://www.intel.com/content/www/us/en/developer/tools/oneapi/base-toolkit.html) 页面获取 **Intel® oneAPI Base toolkit** 和 **Intel® Deep Learning Essentials**。

请按照 Windows 工具包的下载和安装说明进行操作，最好保持默认安装值不变，尤其是安装路径 *（默认为 `C:\Program Files (x86)\Intel\oneAPI`）*。

以下指南/代码片段假设使用默认安装值。否则，请确保在适用位置反映必要的更改。

b. 启用 oneAPI 运行环境：

- 在搜索栏中输入 "oneAPI"，然后打开 `Intel oneAPI command prompt for Intel 64 for Visual Studio 2022` 应用。

- 在命令提示符中，使用以下命令启用运行时环境：
```
"C:\Program Files (x86)\Intel\oneAPI\setvars.bat" intel64
```

- 如果你使用的是 Powershell，使用以下命令启用运行时环境：

```
cmd.exe "/K" '"C:\Program Files (x86)\Intel\oneAPI\setvars.bat" && powershell'
```

c. 验证安装

在 oneAPI 命令行中，运行以下命令以打印可用的 SYCL 设备：

```
sycl-ls.exe
```

应显示一个或多个 *level-zero* GPU 设备，标识为 **[ext_oneapi_level_zero:gpu]**。以下是检测到的 *Intel Iris Xe* GPU 作为 Level-zero SYCL 设备的示例输出：

输出（示例）：
```
[opencl:acc:0] Intel(R) FPGA Emulation Platform for OpenCL(TM), Intel(R) FPGA Emulation Device OpenCL 1.2  [2023.16.10.0.17_160000]
[opencl:cpu:1] Intel(R) OpenCL, 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz OpenCL 3.0 (Build 0) [2023.16.10.0.17_160000]
[opencl:gpu:2] Intel(R) OpenCL Graphics, Intel(R) Iris(R) Xe Graphics OpenCL 3.0 NEO  [31.0.101.5186]
[ext_oneapi_level_zero:gpu:0] Intel(R) Level-Zero, Intel(R) Iris(R) Xe Graphics 1.3 [1.3.28044]
```

3. 安装构建工具

a. 下载并安装 Windows 版 cmake：https://cmake.org/download/（也可通过 Visual Studio Installer 安装 CMake）
b. 新版 Visual Studio 会默认安装 Ninja。（如果没有，请手动安装：https://ninja-build.org/）


#### II. 构建 llama.cpp

你可以直接下载 Windows 发布包，其中包含二进制文件和依赖的 oneAPI dll 文件。

选择以下方法之一从源代码构建。

##### 选项 1：脚本

```sh
.\examples\sycl\win-build-sycl.bat
```

##### 选项 2：CMake

在 oneAPI 命令行窗口中，进入 llama.cpp 主目录并运行以下命令：

```
@call "C:\Program Files (x86)\Intel\oneAPI\setvars.bat" intel64 --force

# 选项 1：使用 FP32（在大多数情况下推荐以获得更好性能）
cmake -B build -G "Ninja" -DGGML_SYCL=ON -DCMAKE_C_COMPILER=cl -DCMAKE_CXX_COMPILER=icx  -DCMAKE_BUILD_TYPE=Release

# 选项 2：或 FP16
cmake -B build -G "Ninja" -DGGML_SYCL=ON -DCMAKE_C_COMPILER=cl -DCMAKE_CXX_COMPILER=icx  -DCMAKE_BUILD_TYPE=Release -DGGML_SYCL_F16=ON

cmake --build build --config Release -j
```

或者，使用 CMake presets 构建：

```sh
cmake --preset x64-windows-sycl-release
cmake --build build-x64-windows-sycl-release -j --target llama-completion

cmake -DGGML_SYCL_F16=ON --preset x64-windows-sycl-release
cmake --build build-x64-windows-sycl-release -j --target llama-completion

cmake --preset x64-windows-sycl-debug
cmake --build build-x64-windows-sycl-debug -j --target llama-completion
```

##### 选项 3：Visual Studio

使用 Visual Studio 构建 llama.cpp 有两种选择：
- 作为使用 CMake presets 的 CMake 项目。
- 创建 Visual Studio 解决方案来管理项目。

**注意**：

以下所有命令均在 PowerShell 中执行。

###### - 作为 CMake 项目打开

你可以使用 Visual Studio 直接将 `llama.cpp` 文件夹作为 CMake 项目打开。在编译之前，选择以下 SYCL CMake presets 之一：

- `x64-windows-sycl-release`

- `x64-windows-sycl-debug`

*说明：*
- 对于最小化的实验环境，你可以仅构建推理可执行文件：

    ```Powershell
    cmake --build build --config Release -j --target llama-completion
    ```

###### - 生成 Visual Studio 解决方案

你可以使用 Visual Studio 解决方案在 Windows 上构建和开发 llama.cpp。需要将 CMake 项目转换为 `.sln` 文件。

如果你想在整个 `llama.cpp` 项目中使用 Intel C++ Compiler，请运行以下命令：

```Powershell
cmake -B build -G "Visual Studio 17 2022" -T "Intel C++ Compiler 2025" -A x64 -DGGML_SYCL=ON -DCMAKE_BUILD_TYPE=Release
```

如果你只想在 `ggml-sycl` 中使用 Intel C++ Compiler，请确保 `ggml` 及其后端库以共享库形式构建（即 `-DBUILD_SHARED_LIBRARIES=ON`，这是默认行为）：

```Powershell
cmake -B build -G "Visual Studio 17 2022" -A x64 -DGGML_SYCL=ON -DCMAKE_BUILD_TYPE=Release \
      -DSYCL_INCLUDE_DIR="C:\Program Files (x86)\Intel\oneAPI\compiler\latest\include" \
      -DSYCL_LIBRARY_DIR="C:\Program Files (x86)\Intel\oneAPI\compiler\latest\lib"
```

如果成功，构建文件将写入：*path/to/llama.cpp/build*
使用 Visual Studio 打开项目文件 **build/llama.cpp.sln**。

创建 Visual Studio 解决方案后，请按照以下步骤操作：

1. 在 Visual Studio 中打开解决方案。

2. 右键单击 `ggml-sycl` 并选择 **属性**。

3. 在左栏中，展开 **C/C++** 并选择 **DPC++**。

4. 在右侧面板中，找到 **Enable SYCL Offload** 并将其设置为 `Yes`。

5. 应用更改并保存。


*导航路径：*

```
Properties -> C/C++ -> DPC++ -> Enable SYCL Offload (Yes)
```

现在，你可以将带有 SYCL 后端的 `llama.cpp` 作为 Visual Studio 项目进行构建。
通过菜单执行：`Build -> Build Solution`。
构建完成后，最终结果将位于 **build/Release/bin**

*补充说明*

- 你可以通过设置环境变量来避免在 CMake 命令中指定 `SYCL_INCLUDE_DIR` 和 `SYCL_LIBRARY_DIR`：

    - `SYCL_INCLUDE_DIR_HINT`

    - `SYCL_LIBRARY_DIR_HINT`

- 以上说明已在 Visual Studio 17 Community 版和 oneAPI 2025.0 上测试通过。如果相应调整说明，我们预计也能用于未来版本。

### III. 运行推理

#### 获取并准备模型

你可以参考通用的 [*获取与量化模型*](../../README.md#obtaining-and-quantizing-models) 指南来准备模型，或直接下载已量化的模型，例如 [llama-2-7b.Q4_0.gguf](https://huggingface.co/TheBloke/Llama-2-7B-GGUF/blob/main/llama-2-7b.Q4_0.gguf) 或 [Meta-Llama-3-8B-Instruct-Q4_0.gguf](https://huggingface.co/aptha/Meta-Llama-3-8B-Instruct-Q4_0-GGUF/resolve/main/Meta-Llama-3-8B-Instruct-Q4_0.gguf)。

##### 检查设备

1. 启用 oneAPI 运行环境

在 oneAPI 命令行窗口中，运行以下命令并进入 llama.cpp 目录：
```
"C:\Program Files (x86)\Intel\oneAPI\setvars.bat" intel64
```

2. 列出设备信息

与原生 `sycl-ls` 类似，可按如下方式查询可用的 SYCL 设备：

```
build\bin\llama-ls-sycl-device.exe
```

该命令仅显示 SYCL 支持的已选后端。默认后端为 level_zero。例如，在具有 2 个 *Intel GPU* 的系统中，输出如下：
```
found 2 SYCL devices:
|  |                  |                                             |Compute   |Max compute|Max work|Max sub|               |
|ID|       Device Type|                                         Name|capability|units      |group   |group  |Global mem size|
|--|------------------|---------------------------------------------|----------|-----------|--------|-------|---------------|
| 0|[level_zero:gpu:0]|               Intel(R) Arc(TM) A770 Graphics|       1.3|        512|    1024|     32|    16225243136|
| 1|[level_zero:gpu:1]|                    Intel(R) UHD Graphics 770|       1.3|         32|     512|     32|    53651849216|

```

##### 选择 level-zero 设备

|选择的设备 ID|设置|
|-|-|
|0|默认选项。你也可以设置 `set ONEAPI_DEVICE_SELECTOR="level_zero:0"`|
|1|`set ONEAPI_DEVICE_SELECTOR="level_zero:1"`|
|0 和 1|`set ONEAPI_DEVICE_SELECTOR="level_zero:0;level_zero:1"` 或 `set ONEAPI_DEVICE_SELECTOR="level_zero:*"`|

##### 执行

选择以下方法之一来运行。

1. 脚本

- 运行测试：

```
examples\sycl\win-test.bat
```

- 运行 llama-server：

```
examples\sycl\win-start-svr.bat -m PATH\MODEL_FILE
```

2. 命令行

启动推理

有两种设备选择模式：

- 单设备：使用用户指定的一个设备。默认设备 ID 为 0。
- 多设备：自动选择具有相同后端的设备。

在两种设备选择模式下，默认 SYCL 后端均为 level_zero，你可以通过设置环境变量 ONEAPI_DEVICE_SELECTOR 来选择 SYCL 支持的其他后端。

| 设备选择 | 参数                              |
|------------------|----------------------------------------|
| 单设备    | --split-mode none --main-gpu DEVICE_ID |
| 多设备 | --split-mode layer（默认）           |

示例：

- 使用设备 0：

```
build\bin\llama-completion.exe -no-cnv -m models\llama-2-7b.Q4_0.gguf -p "Building a website can be done in 10 simple steps:\nStep 1:" -n 400 -e -ngl 99 -sm none -mg 0 --mmap
```

- 使用多设备：

```
build\bin\llama-completion.exe -no-cnv -m models\llama-2-7b.Q4_0.gguf -p "Building a website can be done in 10 simple steps:\nStep 1:" -n 400 -e -ngl 99 -sm layer --mmap
```


注意：

- 执行时，请在输出日志中确认所选设备的 ID，例如可能显示为：

```sh
detect 1 SYCL GPUs: [0] with top Max compute units:512
```

或

```sh
use 1 SYCL GPUs: [0] with Max compute units:512
```


## 环境变量

### 构建

| 名称               | 值                                 | 功能                                    |
|--------------------|---------------------------------------|---------------------------------------------|
| GGML_SYCL          | ON（必需）                        | 启用 SYCL 代码路径构建。           |
| GGML_SYCL_TARGET   | INTEL *（默认）*                     | 设置 SYCL 目标设备类型。            |
| GGML_SYCL_DEVICE_ARCH | 可选                           | 设置 SYCL 设备架构。设置设备架构可以提升性能。有关有效架构列表，请参阅 [--offload-arch](https://github.com/intel/llvm/blob/sycl/sycl/doc/design/OffloadDesign.md#--offload-arch) 表格。 |
| GGML_SYCL_F16      | OFF *（默认）* \| ON *（可选）*     | 启用 SYCL 代码路径的 FP16 构建。（1.） |
| GGML_SYCL_GRAPH    | OFF *（默认）* \| ON *（可选）*     | 启用 [SYCL Graph extension](https://github.com/intel/llvm/blob/sycl/sycl/doc/extensions/experimental/sycl_ext_oneapi_graph.asciidoc) 构建。 |
| GGML_SYCL_DNN      | ON *（默认）* \| OFF *（可选）*     | 启用 oneDNN 构建。                   |
| GGML_SYCL_HOST_MEM_FALLBACK | ON *（默认）* \| OFF *（可选）* | 在量化权重重排序期间设备内存不足时允许回退到主机内存。这样可以使推理以较低速度继续运行（通过 PCIe 读取），而不是失败。需要 Linux 内核 6.8+。 |
| GGML_SYCL_SUPPORT_LEVEL_ZERO | ON *（默认）* \| OFF *（可选）* | 启用 Level Zero API 进行设备内存分配。构建时需要 Level Zero 头文件/库，运行时需要 Intel GPU 驱动（Level Zero 运行时）。可减少多 GPU 推理期间的系统内存占用。 |
| CMAKE_C_COMPILER   | `icx` *（Linux）*、`icx/cl` *（Windows）* | 为 SYCL 代码路径设置 `icx` 编译器。      |
| CMAKE_CXX_COMPILER | `icpx` *（Linux）*、`icx` *（Windows）*   | 为 SYCL 代码路径设置 `icpx/icx` 编译器。 |

1. FP32 或 FP16 对 LLM 的性能影响不同。建议在你的模型上测试它们以获得更好的提示处理性能。更改 `GGML_SYCL_F16=OFF/ON` 后需要重新构建代码。

### 运行时

| 名称              | 值            | 功能                                                                                                                  |
|-------------------|------------------|---------------------------------------------------------------------------------------------------------------------------|
| GGML_SYCL_DEBUG   | 0（默认）或 1 | 通过宏启用日志功能：GGML_SYCL_DEBUG。                                                                             |
| GGML_SYCL_ENABLE_FLASH_ATTN | 1（默认）或 0| 启用 Flash-Attention。可以减少内存占用。性能影响取决于 LLM。|
| GGML_SYCL_DISABLE_OPT | 0（默认）或 1 | 禁用 Intel GPU 的优化功能。（对于低于 Gen 10 的 Intel 设备，建议设为 1） |
| GGML_SYCL_DISABLE_GRAPH | 0 或 1（默认） | 禁用过 SYCL Graph 特性运行计算。默认禁用，因为 SYCL Graph 仍在开发中，暂无更好性能。 |
| GGML_SYCL_ENABLE_LEVEL_ZERO | 1（默认）或 0 | 使用 Level Zero API 替代 SYCL 进行设备内存分配。通过在 Intel 独显上避免 DMA-buf/TTM 主机内存暂存，减少系统内存占用。构建时需要 GGML_SYCL_SUPPORT_LEVEL_ZERO=ON。 |
| GGML_SYCL_DISABLE_DNN | 0（默认）或 1 | 禁用过 oneDNN 运行计算，始终使用 oneMKL。 |
| GGML_SYCL_ENABLE_VMM | 0 或 1（默认） | 启用虚拟内存设备池。 |
| ZES_ENABLE_SYSMAN | 0（默认）或 1 | 支持通过 sycl::aspect::ext_intel_free_memory 获取 GPU 空闲内存。<br>建议在 --split-mode = layer 时使用。 |
| UR_L0_ENABLE_RELAXED_ALLOCATION_LIMITS | 0（默认）或 1 | 允许 SYCL/Unified Runtime Level Zero 设备分配大于 4 GiB。当 GGML_SYCL_ENABLE_LEVEL_ZERO=1 时，llama.cpp 的直接 Level Zero 分配路径会自行请求放宽的最大尺寸限制。 |

## 编译时标志

通过 `CXXFLAGS` 传递这些标志，或添加一次性 `#define` 以即时启用某个标志。

| 名称            | 功能                                                                         |
|-----------------|----------------------------------------------------------------------------------|
| DEBUG_SYCL_POOL | 启用关闭时的设备内存池日志。有助于分析分配情况。 |
| DEBUG_SYCL_MALLOC | 启用设备内存池分配/释放操作的详细逐调用日志。 |

## 设计规范

- 欢迎所有贡献者参与。

- 所有代码变更都应是对用户有用的：
    - 修复 bug。
    - 添加新功能。
    - 提升性能/易用性。
    - 使代码易于维护。
    - ...

- 不接受以下情况的代码：
    - 破坏旧功能。
    - 默认情况下降低旧用例的性能。
    - 未完成的工作/功能无法演示。

- 鼓励使用环境变量来控制功能的开启/关闭。
    - 用户无需重新构建代码即可评估功能。
    - 通过将最佳功能默认开启来推荐给用户。

- 基于已发布的 oneAPI 包官方版本设计代码：编译器、库、驱动、操作系统内核。

- 开发者需要维护他们提交的代码。

## 已知问题

- 不支持 `Split-mode:[row]`。

- 构建中缺少 AOT（Ahead-of-Time）。
  - 优点：构建速度快，二进制文件体积更小。
  - 缺点：首次启动较慢（JIT），但后续性能不受影响。

## 常见问题

- 错误：`error while loading shared libraries: libsycl.so: cannot open shared object file: No such file or directory`。

  - 可能原因：未安装 oneAPI 或未设置环境变量。
  - 解决方案：安装 *oneAPI base toolkit* 并通过 `source /opt/intel/oneapi/setvars.sh` 启用其环境变量。

- 一般编译错误：

  - 删除 **build** 文件夹或尝试干净构建。

- 在 Linux 上安装 GPU 驱动后，我看不到 `[ext_oneapi_level_zero:gpu]`。

  请使用 `sudo sycl-ls` 再次检查。

  如果它出现在列表中，请将 video/render 组添加到你的用户，然后**注销/登录**或重启系统：

  ```
  sudo usermod -aG render $USER
  sudo usermod -aG video $USER
  ```
  否则，请再次检查 GPU 驱动安装步骤。

- 我可以将向 Intel GPU 上的 Ollama 问题报告给 llama.cpp SYCL 后端吗？

  不可以。我们无法直接支持 Ollama 相关问题，因为我们不熟悉 Ollama。

  建议在 llama.cpp 上复现问题，并向 llama.cpp 报告类似问题。我们会提供支持。

  其他包含 llama.cpp SYCL 后端的项目也是如此。

- `Native API failed. Native API returns: 39 (UR_RESULT_ERROR_OUT_OF_DEVICE_MEMORY)`、`ggml_backend_sycl_buffer_type_alloc_buffer: can't allocate 3503030272 Bytes of memory on device`，或 `failed to allocate SYCL0 buffer`

  你的设备内存不足。

  |原因|解决方案|
  |-|-|
  | 默认上下文过大，导致内存使用过多。|设置 `-c 8192` 或更小的值。|
  | 模型过大，所需内存超过可用内存。|选择更小的模型或更小的量化格式，例如 Q5 -> Q4；<br>或者使用多个设备加载模型。|

- `ggml_backend_sycl_buffer_type_alloc_buffer: can't allocate 5000000000 Bytes of memory on device`

  在默认 `GGML_SYCL_ENABLE_LEVEL_ZERO=1` 时，llama.cpp 会直接请求 Level Zero 的放宽最大尺寸分配限制。如果 Level Zero 支持在构建时或运行时被禁用，分配转而通过 SYCL/Unified Runtime 进行，则可通过以下方式启用大于 4 GiB 的分配支持：
  ```
    export UR_L0_ENABLE_RELAXED_ALLOCATION_LIMITS=1
    set UR_L0_ENABLE_RELAXED_ALLOCATION_LIMITS=1
  ```

### **GitHub 贡献**：
请在 issue/PR 标题中添加 `[SYCL]` 前缀/标签，以便 SYCL 贡献者及时查看和处理。

## 待办事项

- 审查 ZES_ENABLE_SYSMAN：https://github.com/intel/compute-runtime/blob/master/programmers-guide/SYSMAN.md#support-and-limitations
