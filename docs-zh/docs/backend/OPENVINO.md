# 用于 llama.cpp 的 OpenVINO 后端

> [!NOTE]
> 性能和内存优化、精度验证、更广泛的量化覆盖、更广泛的算子和模型支持仍在进行中。

[OpenVINO](https://docs.openvino.ai/) 是一个开源工具包，用于优化和部署高性能 AI 推理，专为 Intel 硬件设计，包括 CPU、GPU 和 NPU，支持云端、本地和边缘部署。[llama.cpp 的 OpenVINO 后端](../../ggml/src/ggml-openvino) 可在 **Intel® CPU、GPU 和 NPU** 上实现硬件加速推理，同时保持与现有 **GGUF 模型生态** 的兼容性。该后端将 GGML 计算图转换为 OpenVINO 图，并利用图编译、算子融合和设备专属优化来提升受支持 Intel 硬件上的推理性能。

OpenVINO 后端实现在 `ggml/src/ggml-openvino` 中，为核心 GGML 运算提供转换层。OpenVINO 后端用 Intel 的 OpenVINO 推理引擎替换了标准的 GGML 图执行路径。这种方式允许同一个 GGUF 模型文件在 Intel CPU、Intel GPU（集成和独立）以及 Intel NPU 上运行，而无需修改模型或 llama.cpp 栈的其余部分。当某个 `ggml_cgraph` 被分派到 OpenVINO 后端时，它会：

- 遍历 GGML 图，识别输入、输出、权重和 KV 缓存张量。
- 使用 OpenVINO 的前端 API 将 GGML 运算转换为 `ov::Model`。
- 针对目标设备编译并缓存模型。
- 将 GGML 张量内存绑定到 OpenVINO 推理张量并执行推理。

## 支持的设备

OpenVINO 后端支持以下硬件：

- Intel CPU
- Intel GPU（集成和独立）
- Intel NPU

尽管 OpenVINO 支持广泛的 [Intel 硬件](https://docs.openvino.ai/2026/about-openvino/release-notes-openvino/system-requirements.html)，但 llama.cpp 的 OpenVINO 后端已专门在 AI PC 上完成验证，例如 Intel® Core™ Ultra 第一代和第二代。

## 支持的模型精度

- `FP16`
- `BF16`（在 Intel Xeon 上）
- `Q8_0`
- `Q4_0`
- `Q4_1`
- `Q4_K`
- `Q4_K_M`
- `Q5_K`（运行转换为 Q8_0_C）
- `Q6_K`（运行转换为 Q8_0_C）

> [!NOTE]
> 量化模型的精度验证和性能优化仍在进行中。

## 量化支持详情

### CPU 和 GPU

- **`Q4_0`、`Q4_1`、`Q4_K_M`、`Q6_K` 模型已支持**
- `Q5_K` 和 `Q6_K` 张量会转换为 `Q8_0_C`

### NPU

- **主要支持的量化方案为 `Q4_0`**
- `Q6_K` 张量通常会被重新量化为 `Q4_0_128`。对于嵌入权重，`Q6_K` 张量会重新量化为 `Q8_0_C`，但词嵌入矩阵会被反量化为 fp16

### 其他说明

- `Q4_0` 和 `Q4_1` 模型都会使用 `Q6_K` 来存储词嵌入张量和最终的 matmul 权重张量（通常是同一个张量）
- 如果在使用 `llama-quantize` 进行量化时提供了 imatrix，`Q4_0` 模型可能会产生一些 `Q4_1` 张量
- `Q4_K_M` 模型可能同时包含 `Q6_K` 和 `Q5_K` 张量（在 Phi-3 中观察到）

## 已验证的模型

以下模型已在 Intel® Core™ Ultra 第二代上完成验证。虽然我们的测试有限，但 OpenVINO 后端预计可在广泛的 [Intel 硬件](https://docs.openvino.ai/2026/about-openvino/release-notes-openvino/system-requirements.html) 上工作。
- 使用 GPU 设备时，请设置 `GGML_OPENVINO_STATEFUL_EXECUTION=1`。
- 使用 OpenVINO 后端运行 llama-bench 时，需要 `-fa 1`。
- 更多模型支持、量化格式和验证工作仍在进行中。

| 模型  | 已验证   | 已知问题  |
| :------| :---------- | :-------------|
| [Llama-3.2-1B-Instruct](https://huggingface.co/unsloth/Llama-3.2-1B-Instruct-GGUF/) | `FP16`、`Q8_0`、`Q4_0`、`Q4_1`、`Q4_K_M` 在 CPU/GPU/NPU 上 | — |
| [Meta-Llama-3.1-8B-Instruct](https://huggingface.co/bartowski/Meta-Llama-3.1-8B-Instruct-GGUF) | `Q8_0`、`Q4_K_M` 在 CPU/GPU/NPU 上 | `Q4_0_8_8`、`Q4_0_4_8`、`Q4_0_4_4` 失败 |
| [Phi-3-mini-4k-instruct](https://huggingface.co/microsoft/Phi-3-mini-4k-instruct-gguf) | `FP16`、`Q4` 在 CPU/NPU 上 | GPU 不支持 `FP16` 和 `Q4`（`llama-cli`、`llama-bench`） |
| [Qwen2.5-1.5B-Instruct](https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF) | `FP16`、`Q8_0`、`Q4_0`、`Q4_1`、`Q4_K_M` 在 CPU/GPU/NPU 上 | — |
| [Qwen3-8B-Instruct](https://huggingface.co/Qwen/Qwen3-8B-GGUF) | `FP16`、`Q8_0`、`Q4_0`、`Q4_1`、`Q4_K_M` 在 CPU/NPU 上；GPU 通过 `llama-bench` 工作 | GPU 上所有量化均不支持 `llama-cli` |
| [MiniCPM-V-2_6-GGUF](https://huggingface.co/openbmb/MiniCPM-V-2_6-gguf) | `Q4_0` 在 CPU/GPU/NPU 上 | — |
| [DeepSeek-R1-Distill-Llama-8B](https://huggingface.co/bartowski/DeepSeek-R1-Distill-Llama-8B-GGUF) | `Q8_0`、`Q4_0`、`Q4_1`、`Q4_K_M` 在 CPU/GPU/NPU 上 | — |
| [Hunyuan-7B-Instruct](https://huggingface.co/bartowski/tencent_Hunyuan-7B-Instruct-GGUF) | CPU：`Q8_0`、`Q4_0`、`Q4_1`、`Q4_K_M`；GPU：`Q8_0`、`Q4_0`、`Q4_1`；NPU（仅 `llama-bench`）：`Q4_0`、`Q4_1`、`Q4_K_M` | GPU 不支持 `Q4_K_M`；NPU 不支持 `llama-cli` |
| [Mistral-7B-Instruct-v0.3](https://huggingface.co/bartowski/Mistral-7B-Instruct-v0.3-GGUF/) | CPU/GPU：`Q8_0`、`Q4_K_M`；NPU：`Q8_0`、`Q4_K_M`（通过 `llama-bench`） | NPU 不支持 `Q8_0`、`Q4_K_M` 的 `llama-cli` |

## 构建说明

### 前置条件

- 配备 Intel 硬件（CPU、GPU 或 NPU）的 Linux 或 Windows 系统
- **对于 Intel GPU 或 NPU 使用**：请安装适用于 Intel GPU 或 NPU 的硬件驱动程序。详细说明请参见：[硬件加速的额外配置](https://docs.openvino.ai/2025/get-started/install-openvino/configurations.html)。

- **Linux：**
    - 需要 Git、CMake 和 Ninja 等软件工具。
    ```bash
      sudo apt-get update
      sudo apt-get install -y build-essential libcurl4-openssl-dev libtbb12 cmake ninja-build python3-pip curl wget tar
    ```
    - OpenCL
    ```bash
      sudo apt install ocl-icd-opencl-dev opencl-headers opencl-clhpp-headers intel-opencl-icd
    ```

- **Windows：**
  - 下载并安装 [Microsoft Visual Studio 2022 Build Tools](https://aka.ms/vs/17/release/vs_BuildTools.exe)。安装过程中请选择 **"Desktop development with C++"** 工作负载。

  - 安装所需工具：
    ```powershell
    # Windows PowerShell
    winget install Git.Git
    winget install GNU.Wget
    winget install Ninja-build.Ninja
    ```

  - 使用 **vcpkg** 安装 **OpenCL**：
    ```powershell
    # Windows PowerShell
    cd C:\
    git clone https://github.com/microsoft/vcpkg
    cd vcpkg
    .\bootstrap-vcpkg.bat
    .\vcpkg install opencl
    # 可选但推荐：将 vcpkg 集成到 Visual Studio / CMake：
    .\vcpkg integrate install
    ```

### 1. 安装 OpenVINO Runtime

- 请按照指南从压缩包安装 OpenVINO Runtime：[Linux](https://docs.openvino.ai/2026/get-started/install-openvino/install-openvino-archive-linux.html) | [Windows](https://docs.openvino.ai/2026/get-started/install-openvino/install-openvino-archive-windows.html)

- **Linux：**

    <details>
    <summary>📦 点击展开在 Ubuntu 上从压缩包安装 OpenVINO</summary>
    <br>

    ```bash
    wget https://raw.githubusercontent.com/ravi9/misc-scripts/main/openvino/ov-archive-install/install-openvino-from-archive.sh
    chmod +x install-openvino-from-archive.sh
    ./install-openvino-from-archive.sh
    ```

    验证 OpenVINO 是否已正确初始化：
    ```bash
    echo $OpenVINO_DIR
    ```
    </details>


### 2. 使用 OpenVINO 后端构建 llama.cpp

克隆启用 OpenVINO 的 llama.cpp 仓库并构建：

```bash
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
```

- **Linux：**
    ```bash
    source /opt/intel/openvino/setupvars.sh
    cmake -B build/ReleaseOV -G Ninja -DCMAKE_BUILD_TYPE=Release -DGGML_OPENVINO=ON
    cmake --build build/ReleaseOV --parallel
    ```

- **Windows：**
    ```cmd
    # x64 Native Tools Command Prompt for VS 2022
    "C:\Program Files (x86)\Intel\openvino_2026.0\setupvars.bat"
    cmake -B build\ReleaseOV -G Ninja -DCMAKE_BUILD_TYPE=Release -DGGML_OPENVINO=ON -DLLAMA_CURL=OFF -DCMAKE_TOOLCHAIN_FILE=C:\vcpkg\scripts\buildsystems\vcpkg.cmake
    cmake --build build\ReleaseOV --parallel
    ```
> [!NOTE]
> Windows 构建请使用 `x64 Native Tools Command Prompt`。构建完成后，可以使用 `cmd` 或 `PowerShell` 运行 OpenVINO 后端。

### 3. 下载示例模型

下载用于测试的模型：

```bash
# Linux
mkdir -p ~/models/
wget https://huggingface.co/unsloth/Llama-3.2-1B-Instruct-GGUF/resolve/main/Llama-3.2-1B-Instruct-Q4_0.gguf \
     -O ~/models/Llama-3.2-1B-Instruct-Q4_0.gguf

# Windows PowerShell
mkdir C:\models
Invoke-WebRequest -Uri https://huggingface.co/unsloth/Llama-3.2-1B-Instruct-GGUF/resolve/main/Llama-3.2-1B-Instruct-Q4_0.gguf -OutFile C:\models\Llama-3.2-1B-Instruct-Q4_0.gguf

# Windows Command Line
mkdir C:\models
curl -L https://huggingface.co/unsloth/Llama-3.2-1B-Instruct-GGUF/resolve/main/Llama-3.2-1B-Instruct-Q4_0.gguf -o C:\models\Llama-3.2-1B-Instruct-Q4_0.gguf
```

### 4. 使用 OpenVINO 后端运行推理

使用 OpenVINO 后端时，第一个推理 token 的延迟可能略高，因为需要动态转换为 OpenVINO 图。后续的 token 和运行会更快。

> [!NOTE]
> 默认上下文大小设置为模型训练时的上下文，可能非常大。例如 Llama 3.2 1B 为 131072，这可能会导致性能下降，尤其是在边缘/笔记本设备上。可以在支持的 llama.cpp 工具中使用 `-c` 限制上下文大小以获得更好性能，例如 `-c 512`。

```bash
# If device is unset or unavailable, defaults to CPU.
# If the system has multiple GPUs, use GPU.0 or GPU.1 to explicitly target a specific GPU.

# Linux
export GGML_OPENVINO_DEVICE=GPU
# Enable stateful execution with GPU device to avoid known stateless execution failures.
export GGML_OPENVINO_STATEFUL_EXECUTION=1
# To run llama-simple:
./build/ReleaseOV/bin/llama-simple -m ~/models/Llama-3.2-1B-Instruct-Q4_0.gguf -n 50 "The story of AI is "
# To run in chat mode:
./build/ReleaseOV/bin/llama-cli -m ~/models/Llama-3.2-1B-Instruct-Q4_0.gguf -c 1024
# To run llama-bench, -fa 1 is needed
GGML_OPENVINO_STATEFUL_EXECUTION=1 GGML_OPENVINO_DEVICE=GPU ./build/ReleaseOV/bin/llama-bench -m ~/models/Llama-3.2-1B-Instruct-Q4_0.gguf -fa 1

# NPU: keep context small to avoid failures from very large model context windows.
export GGML_OPENVINO_DEVICE=NPU
./build/ReleaseOV/bin/llama-cli -m ~/models/Llama-3.2-1B-Instruct-Q4_0.gguf -c 512

# Windows Command Line
set GGML_OPENVINO_DEVICE=GPU
# Enable stateful execution with GPU device to avoid known stateless execution failures.
set GGML_OPENVINO_STATEFUL_EXECUTION=1
# Windows PowerShell
$env:GGML_OPENVINO_DEVICE = "GPU"
$env:GGML_OPENVINO_STATEFUL_EXECUTION = "1"

# To run llama-simple
build\ReleaseOV\bin\llama-simple.exe -m "C:\models\Llama-3.2-1B-Instruct-Q4_0.gguf" -n 50 "The story of AI is "
# To run in chat mode:
build\ReleaseOV\bin\llama-cli.exe -m "C:\models\Llama-3.2-1B-Instruct-Q4_0.gguf" -c 1024
# To run llama-bench, -fa 1 is needed
build\ReleaseOV\bin\llama-bench.exe -m "C:\models\Llama-3.2-1B-Instruct-Q4_0.gguf" -fa 1

# NPU: keep context small to avoid failures from very large model context windows.
# Windows Command Line
set GGML_OPENVINO_DEVICE=NPU
# Windows PowerShell
$env:GGML_OPENVINO_DEVICE = "NPU"
build\ReleaseOV\bin\llama-cli.exe -m "C:\models\Llama-3.2-1B-Instruct-Q4_0.gguf" -c 512
```
> [!NOTE]
> 在多 GPU 系统上，请使用 `GPU.0` 或 `GPU.1` 显式指定具体 GPU。更多详情请参见 [OpenVINO GPU 设备](https://docs.openvino.ai/2026/openvino-workflow/running-inference/inference-devices-and-modes/gpu-device.html)。

### 已知问题与当前解决方法

- GPU 无状态执行目前存在一个已知问题。
  - 解决方法：使用 GPU 设备时设置 `GGML_OPENVINO_STATEFUL_EXECUTION=1`。
- 当上下文大小过大时，NPU 可能会失败。llama.cpp 最近的版本可能会将上下文大小解析为模型训练时的上下文（例如 Llama 3.2 1B 的 131072），这对当前 NPU 使用来说过大，也可能在较大模型上给笔记本 CPU/GPU 带来压力。要查看选定的上下文大小，请使用 `-lv 3` 运行 `llama-cli` 或 `llama-server`。
  - 解决方法：显式设置上下文大小，例如 NPU 运行使用 `-c 1024`。较低的上下文大小会带来更好的性能。
- 其他 NPU 限制：
  - 模型缓存尚不支持。
  - 不支持 `llama-server -np > 1`（多个并行序列）。
  - `llama-perplexity` 仅在 `-b 512` 或更小时受支持。
- 目前在 CPU、GPU 和 NPU 设备上，OpenVINO 后端不支持 `llama-cli` 的 `--context-shift`。
- 编码器模型（嵌入、重排序）在当前 OpenVINO 后端实现中不受支持。
- 使用 OpenVINO 后端运行 llama-bench 时需要 `-fa 1`。
  - `GGML_OPENVINO_STATEFUL_EXECUTION=1 GGML_OPENVINO_DEVICE=GPU ./llama-bench -fa 1`
- 启用 `GGML_OPENVINO_STATEFUL_EXECUTION=1` 时，OpenVINO 后端的 `llama-server` 仅支持一个聊天会话/线程。

> [!NOTE]
> OpenVINO 后端正在积极开发中。修复工作正在进行，本文档将随着问题的解决持续更新。


### Docker 构建

您可以使用 Docker 构建并运行带有 OpenVINO 后端的 llama.cpp。

```bash
# Build the base runtime image with compiled shared libraries and minimal dependencies.
docker build -t llama-openvino:base -f .devops/openvino.Dockerfile .

# Build the complete image with all binaries, Python tools, gguf-py library, and model conversion utilities.
docker build --target=full -t llama-openvino:full -f .devops/openvino.Dockerfile .

# Build a minimal CLI-only image containing just the llama-cli executable.
docker build --target=light -t llama-openvino:light -f .devops/openvino.Dockerfile .

# Builds a server-only image with llama-server executable, health check endpoint, and REST API support.
docker build --target=server -t llama-openvino:server -f .devops/openvino.Dockerfile .

# If you are behind a proxy:
docker build --build-arg http_proxy=$http_proxy --build-arg https_proxy=$https_proxy --target=light -t llama-openvino:light -f .devops/openvino.Dockerfile .
```

运行带有 OpenVINO 后端的 llama.cpp Docker 容器。
将示例模型保存到 `~/models`，如[上文所示](#3-download-sample-model)。在下面的示例中，它将被挂载到容器中。


```bash
#  Run Docker container
docker run --rm -it -v ~/models:/models llama-openvino:light --no-warmup -c 1024 -m /models/Llama-3.2-1B-Instruct-Q4_0.gguf

# With Intel GPU access (iGPU or dGPU)
docker run --rm -it -v ~/models:/models \
--device=/dev/dri --group-add=$(stat -c "%g" /dev/dri/render* | head -n 1) -u $(id -u):$(id -g) \
--env=GGML_OPENVINO_DEVICE=GPU --env=GGML_OPENVINO_STATEFUL_EXECUTION=1 \
llama-openvino:light --no-warmup -c 1024 -m /models/Llama-3.2-1B-Instruct-Q4_0.gguf

# With Intel NPU access
docker run --rm -it -v ~/models:/models \
--device=/dev/accel --group-add=$(stat -c "%g" /dev/dri/render* | head -n 1) -u $(id -u):$(id -g) \
--env=GGML_OPENVINO_DEVICE=NPU \
llama-openvino:light --no-warmup -c 1024 -m /models/Llama-3.2-1B-Instruct-Q4_0.gguf
```

运行带有 OpenVINO 后端的 Llama.cpp Server。
> [!NOTE]
> 启用 `GGML_OPENVINO_STATEFUL_EXECUTION=1` 时，OpenVINO 后端的 `llama-server` 仅支持一个聊天会话/线程。

```bash
# Run the Server Docker container
docker run --rm -it -p 8080:8080 -v ~/models:/models llama-openvino:server --no-warmup -m /models/Llama-3.2-1B-Instruct-Q4_0.gguf -c 1024
# Or Using llama-server executable
./build/ReleaseOV/bin/llama-server -m ~/models/Llama-3.2-1B-Instruct-Q4_0.gguf --port 8080 -c 1024

# If you are behind a proxy, make sure to set NO_PROXY to avoid proxy for localhost
export NO_PROXY=localhost,127.0.0.1

# Option 1: Open your browser to http://localhost:8080 to access the web UI for the llama.cpp server.
# Option 2: In a NEW terminal, test the server with curl

# Test health endpoint
curl -f http://localhost:8080/health

# Test with a simple prompt
curl -X POST "http://localhost:8080/v1/chat/completions" -H "Content-Type: application/json" \
 -d '{"messages":[{"role":"user","content":"Write a poem about OpenVINO"}],"max_tokens":100}' | jq .
```

## 运行时配置

OpenVINO 后端可在运行时使用以下环境变量进行配置，以控制设备选择、缓存、调试和分析行为。

### 配置选项

| Variable                          | Default    | Description                                                                                                 |
|-----------------------------------|------------|-------------------------------------------------------------------------------------------------------------|
| `GGML_OPENVINO_DEVICE`            | `CPU`      | 指定目标设备（CPU、GPU、NPU）。在多 GPU 系统上，使用 `GPU.0` 或 `GPU.1` 显式指定具体 GPU。详情请参见 [OpenVINO GPU 设备](https://docs.openvino.ai/2026/openvino-workflow/running-inference/inference-devices-and-modes/gpu-device.html)。设置为 **NPU** 时，将启用静态编译模式以获得最佳性能。 |
| `GGML_OPENVINO_CACHE_DIR`         | `not set`  | OpenVINO 模型缓存目录（推荐：`/tmp/ov_cache`）。设置后启用模型缓存。**NPU 设备不支持。** |
| `GGML_OPENVINO_PREFILL_CHUNK_SIZE`| `256`      | **NPU** prefill 的 token 分块大小。                                                                       |
| `GGML_OPENVINO_STATEFUL_EXECUTION`| `0`        | 启用有状态 KV 缓存以获得更好性能。推荐在 CPU、GPU 上使用。                                |
| `GGML_OPENVINO_PROFILING`         | `0`        | 启用执行时性能分析。                                                                            |
| `GGML_OPENVINO_DUMP_CGRAPH`       | `0`        | 将 GGML 计算图转储到 `cgraph_ov.txt`。                                                             |
| `GGML_OPENVINO_DUMP_IR`           | `0`        | 序列化带时间戳的 OpenVINO IR 文件。                                                                |
| `GGML_OPENVINO_DEBUG_INPUT`       | `0`        | 启用输入调试并打印输入张量信息。                                                         |
| `GGML_OPENVINO_DEBUG_OUTPUT`      | `0`        | 启用输出调试并打印输出张量信息。                                                       |
| `GGML_OPENVINO_PRINT_CGRAPH_TENSOR_ADDRESS` | `0` | 打印一次张量地址映射。                                                                           |

> [!NOTE]
> `GGML_OPENVINO_STATEFUL_EXECUTION` 是一项**实验性**功能，允许有状态执行以在 OpenVINO 模型内部管理 KV 缓存，从而提升 CPU 和 GPU 上的性能。有状态执行在 NPU 上效果不佳，且并非所有模型目前都支持该功能。该功能为实验性质，仅在 llama-simple、llama-cli、llama-bench 和 llama-run 应用中完成验证，建议启用以获得最佳性能。llama-server 和 llama-perplexity 等其他应用目前尚不支持。

### 使用示例

#### 使用性能分析的 GPU 推理

```bash
# If the system has multiple GPUs, use GPU.0 or GPU.1 to explicitly target a specific GPU.

# Linux
export GGML_OPENVINO_CACHE_DIR=/tmp/ov_cache
export GGML_OPENVINO_PROFILING=1
export GGML_OPENVINO_DEVICE=GPU
export GGML_OPENVINO_STATEFUL_EXECUTION=1

./build/ReleaseOV/bin/llama-simple -m ~/models/Llama-3.2-1B-Instruct-Q4_0.gguf -n 50 "The story of AI is "

# Windows Command Line
set GGML_OPENVINO_CACHE_DIR=C:\tmp\ov_cache
set GGML_OPENVINO_PROFILING=1
set GGML_OPENVINO_DEVICE=GPU
set GGML_OPENVINO_STATEFUL_EXECUTION=1

# Windows PowerShell
$env:GGML_OPENVINO_CACHE_DIR = "C:\tmp\ov_cache"
$env:GGML_OPENVINO_PROFILING = "1"
$env:GGML_OPENVINO_DEVICE = "GPU"
$env:GGML_OPENVINO_STATEFUL_EXECUTION = "1"

build\ReleaseOV\bin\llama-simple.exe -m "C:\models\Llama-3.2-1B-Instruct-Q4_0.gguf" -n 50 "The story of AI is "

```

## Llama.cpp 工具

以下工具可在 CPU、GPU、NPU 上与 OpenVINO 后端配合使用：
- llama-bench
- llama-cli
- llama-completion
- llama-perplexity
- llama-server
- llama-simple

## 进行中工作

- 性能和内存优化
- 精度验证
- 更广泛的量化覆盖
- 对更多模型架构的支持
