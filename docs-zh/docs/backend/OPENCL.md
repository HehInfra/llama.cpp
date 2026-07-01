# 适用于 OpenCL 的 llama.cpp

- [背景](#background)
- [操作系统](#os)
- [硬件](#hardware)
- [数据类型支持](#datatype-supports)
- [模型准备](#model-preparation)
- [CMake 选项](#cmake-options)
- [Android](#android)
- [Windows 11 Arm64](#windows-11-arm64)
- [Linux](#Linux)
- [已知问题](#known-issues)
- [待办事项](#todo)

## 背景

OpenCL（Open Computing Language，开放计算语言）是一种开放的、免版税的标准，用于在超级计算机、云服务器、个人计算机、移动设备和嵌入式平台中的各类加速器上进行跨平台并行编程。OpenCL 定义了一种编程语言（基于 C99）用于为这些设备编写程序，并定义了应用程序编程接口（API）来控制平台以及在计算设备上执行程序。与 CUDA 类似，OpenCL 已被广泛用于 GPU 编程，并受到大多数 GPU 厂商的支持。

### Llama.cpp + OpenCL

llama.cpp 的 OpenCL 后端旨在首先通过 OpenCL 在 **Qualcomm Adreno GPU** 上启用 llama.cpp。得益于 OpenCL 的可移植性，OpenCL 后端也可以运行在某些 Intel GPU 上，例如那些没有 [SYCL](/docs/backend/SYCL.md) 支持的 GPU，不过性能并非最优。

## 操作系统

| OS      | 状态  | 已验证                                         |
|---------|-------|------------------------------------------------|
| Android | 支持  | Snapdragon 8 Gen 3, Snapdragon 8 Elite         |
| Windows | 支持  | Windows 11 Arm64 with Snapdragon X Elite       |
| Linux   | 支持  | Ubuntu 22.04 WSL2 with Intel 12700H            |

## 硬件

### Adreno GPU

**已验证设备**

| Adreno GPU                           | 状态  |
|:------------------------------------:|:-----:|
| Adreno 750 (Snapdragon 8 Gen 3)      | 支持  |
| Adreno 830 (Snapdragon 8 Elite)      | 支持  |
| Adreno X85 (Snapdragon X Elite)      | 支持  |

> 带有较新驱动和编译器的 A6x GPU 受支持；它们通常出现在物联网平台中。
然而，手机中的 A6x GPU 很可能不受支持，因为驱动和编译器过于陈旧。

## 数据类型支持

| 数据类型               | 状态                       |
|:----------------------:|:--------------------------:|
| Q4_0                   | 支持                       |
| Q6_K                   | 支持，但未优化             |
| Q8_0                   | 支持                       |
| MXFP4                  | 支持                       |

## 模型准备

关于将 Hugging Face safetensor 格式的模型量化为 GGUF 的步骤，可以参考通用的 [llama-quantize 工具](/tools/quantize/README.md)。

目前我们支持 `Q4_0` 量化并已对其进行优化。为了在 Adreno GPU 上获得最佳性能，请在 `llama-quantize` 中添加 `--pure`（即让所有权重都使用 `Q4_0`）。例如：

```sh
./llama-quantize --pure ggml-model-qwen2.5-3b-f16.gguf ggml-model-qwen-3b-Q4_0.gguf Q4_0
```

由于也支持 `Q6_K`，不使用 `--pure` 的 `Q4_0` 量化同样可以运行。但相比纯 `Q4_0` 量化，性能会更差。

### `MXFP4` MoE 模型

OpenAI gpt-oss 模型是采用 `MXFP4` 的 MoE 模型。量化后的模型将为 `MXFP4_MOE`，即 `MXFP4` 和 `Q8_0` 的混合。
对于这种量化，无需指定 `--pure`。
对于 gpt-oss-20b 模型，你可以直接从 Hugging Face [下载](https://huggingface.co/ggml-org/gpt-oss-20b-GGUF) 已量化为 `MXFP4_MOE` 的 GGUF 文件。

虽然可以将 gpt-oss-20b 模型量化为纯 `Q4_0`（所有权重均为 `Q4_0`），但并不推荐，因为 `MXFP4` 已针对 MoE 进行优化，而 `Q4_0` 没有。此外，采用这种纯 `Q4_0` 量化可能会导致精度下降。
因此，建议对此模型使用默认的 `MXFP4_MOE` 量化（见上方链接）。

> 注意，[此处](https://huggingface.co/unsloth/gpt-oss-20b-GGUF/blob/main/gpt-oss-20b-Q4_0.gguf) 的 `Q4_0` 模型是 `Q4_0`、`Q8_0` 和 `MXFP4` 的混合，其性能优于 `MXFP4_MOE` 量化。

## CMake 选项

OpenCL 后端提供以下 CMake 选项，用于控制后端的行为。

| CMake 选项                        | 默认值         | 描述                                      |
|:---------------------------------:|:--------------:|:------------------------------------------|
| `GGML_OPENCL_EMBED_KERNELS`       | `ON`           | 将 OpenCL kernel 嵌入可执行文件。         |
| `GGML_OPENCL_USE_ADRENO_KERNELS`  | `ON`           | 使用针对 Adreno 优化的 kernel。           |

## Android

针对 Android 构建时使用 Ubuntu 22.04。请确保以下工具可从命令行访问：

* Git
* CMake 3.29
* Ninja
* Python3

### I. 配置环境

1. **安装 NDK**

```sh
cd ~
wget https://dl.google.com/android/repository/commandlinetools-linux-8512546_latest.zip && \
unzip commandlinetools-linux-8512546_latest.zip && \
mkdir -p ~/android-sdk/cmdline-tools && \
mv cmdline-tools latest && \
mv latest ~/android-sdk/cmdline-tools/ && \
rm -rf commandlinetools-linux-8512546_latest.zip

yes | ~/android-sdk/cmdline-tools/latest/bin/sdkmanager "ndk;26.3.11579264"
```

2. **安装 OpenCL 头文件和库**

```sh
mkdir -p ~/dev/llm
cd ~/dev/llm

git clone https://github.com/KhronosGroup/OpenCL-Headers && \
cd OpenCL-Headers && \
cp -r CL ~/android-sdk/ndk/26.3.11579264/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/include

cd ~/dev/llm

git clone https://github.com/KhronosGroup/OpenCL-ICD-Loader && \
cd OpenCL-ICD-Loader && \
mkdir build_ndk26 && cd build_ndk26 && \
cmake .. -G Ninja -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_TOOLCHAIN_FILE=$HOME/android-sdk/ndk/26.3.11579264/build/cmake/android.toolchain.cmake \
  -DOPENCL_ICD_LOADER_HEADERS_DIR=$HOME/android-sdk/ndk/26.3.11579264/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/include \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=24 \
  -DANDROID_STL=c++_shared && \
ninja && \
cp libOpenCL.so ~/android-sdk/ndk/26.3.11579264/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/aarch64-linux-android
```

### II. 构建 llama.cpp

```sh
cd ~/dev/llm

git clone https://github.com/ggml-org/llama.cpp && \
cd llama.cpp && \
mkdir build-android && cd build-android

cmake .. -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE=$HOME/android-sdk/ndk/26.3.11579264/build/cmake/android.toolchain.cmake \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-28 \
  -DBUILD_SHARED_LIBS=OFF \
  -DGGML_OPENCL=ON

ninja
```

## Windows 11 Arm64

使用搭载 Windows 11 Arm64 的 Snapdragon X Elite 设备。请确保以下工具可从命令行访问：

* Git
* CMake 3.29
* Clang 19
* Ninja
* Visual Studio 2022
* Powershell 7
* Python

Visual Studio 提供必要的头文件和库，虽然构建时不会直接使用它。或者，也可以安装 Visual Studio Build Tools，而无需安装完整版 Visual Studio。

> 注意，不支持使用 Visual Studio 的 cl 编译器进行构建。必须使用 Clang。Clang 需要依赖 Visual Studio 提供的库才能工作。因此，必须安装 Visual Studio。或者，也可以安装 Visual Studio Build Tools，而无需安装完整版 Visual Studio。

以下命令使用 Powershell 7 执行。如果使用较旧版本的 Powershell，这些命令可能无法按原样运行。

### I. 配置环境

1. **安装 OpenCL 头文件和库**

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

### II. 构建 llama.cpp

```powershell

mkdir -p ~/dev/llm
cd ~/dev/llm

git clone https://github.com/ggml-org/llama.cpp && cd llama.cpp
mkdir build && cd build

cmake .. -G Ninja `
  -DCMAKE_TOOLCHAIN_FILE="$HOME/dev/llm/llama.cpp/cmake/arm64-windows-llvm.cmake" `
  -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_PREFIX_PATH="$HOME/dev/llm/opencl" `
  -DBUILD_SHARED_LIBS=OFF `
  -DGGML_OPENCL=ON
ninja
```

## Linux

上述两个步骤同样适用于 Linux。为 Linux 构建时，命令与 Windows 上 PowerShell 的命令大致相同，但在第二步中没有 `-DCMAKE_TOOLCHAIN_FILE` 参数，并且两步中的反引号都替换为反斜杠。

如果尚未安装，请安装 Git、CMake、Clang、Ninja 和 Python，然后在终端中运行以下内容：

### I. 配置环境

1. **安装 OpenCL 头文件和库**

```bash
mkdir -p ~/dev/llm

cd ~/dev/llm
git clone https://github.com/KhronosGroup/OpenCL-Headers && cd OpenCL-Headers
mkdir build && cd build
cmake .. -G Ninja \
  -DBUILD_TESTING=OFF \
  -DOPENCL_HEADERS_BUILD_TESTING=OFF \
  -DOPENCL_HEADERS_BUILD_CXX_TESTS=OFF \
  -DCMAKE_INSTALL_PREFIX="$HOME/dev/llm/opencl"
cmake --build . --target install

cd ~/dev/llm
git clone https://github.com/KhronosGroup/OpenCL-ICD-Loader && cd OpenCL-ICD-Loader
mkdir build && cd build
cmake .. -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_PREFIX_PATH="$HOME/dev/llm/opencl" \
  -DCMAKE_INSTALL_PREFIX="$HOME/dev/llm/opencl"
cmake --build . --target install
```

### II. 构建 llama.cpp

```bash
mkdir -p ~/dev/llm
cd ~/dev/llm

git clone https://github.com/ggml-org/llama.cpp && cd llama.cpp
mkdir build && cd build

cmake .. -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_PREFIX_PATH="$HOME/dev/llm/opencl" \
  -DBUILD_SHARED_LIBS=OFF \
  -DGGML_OPENCL=ON
ninja
```

## 已知问题

- Flash attention 并不总能提升性能。
- 目前 OpenCL 后端可在带有较新驱动和编译器的 A6xx GPU 上运行（通常见于物联网平台）。
  然而，它无法在带有旧驱动和编译器的手机 A6xx GPU 上运行。

## 待办事项

- Q6_K 优化
- Q4_K 支持与优化
- 改进 flash attention
