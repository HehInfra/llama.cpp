# Docker

## 前置条件
* 系统中必须已安装并运行 Docker。
* 创建一个文件夹，用于存放较大的模型和中间文件（例如 /llama/models）。

## 镜像
本项目提供三种 Docker 镜像：

1. `ghcr.io/ggml-org/llama.cpp:full`：该镜像同时包含 `llama-cli` 和 `llama-completion` 可执行文件，以及将 LLaMA 模型转换为 ggml 并进行 4-bit 量化的工具。（支持平台：`linux/amd64`、`linux/arm64`、`linux/s390x`）
2. `ghcr.io/ggml-org/llama.cpp:light`：该镜像仅包含 `llama-cli` 和 `llama-completion` 可执行文件。（支持平台：`linux/amd64`、`linux/arm64`、`linux/s390x`）
3. `ghcr.io/ggml-org/llama.cpp:server`：该镜像仅包含 `llama-server` 可执行文件。（支持平台：`linux/amd64`、`linux/arm64`、`linux/s390x`）

此外，还有以下与上述类似的镜像：

- `ghcr.io/ggml-org/llama.cpp:full-cuda`：与 `full` 相同，但使用 CUDA 12 编译支持。（支持平台：`linux/amd64`、`linux/arm64`）
- `ghcr.io/ggml-org/llama.cpp:full-cuda13`：与 `full` 相同，但使用 CUDA 13 编译支持。（支持平台：`linux/amd64`、`linux/arm64`）
- `ghcr.io/ggml-org/llama.cpp:light-cuda`：与 `light` 相同，但使用 CUDA 12 编译支持。（支持平台：`linux/amd64`、`linux/arm64`）
- `ghcr.io/ggml-org/llama.cpp:light-cuda13`：与 `light` 相同，但使用 CUDA 13 编译支持。（支持平台：`linux/amd64`、`linux/arm64`）
- `ghcr.io/ggml-org/llama.cpp:server-cuda`：与 `server` 相同，但使用 CUDA 12 编译支持。（支持平台：`linux/amd64`、`linux/arm64`）
- `ghcr.io/ggml-org/llama.cpp:server-cuda13`：与 `server` 相同，但使用 CUDA 13 编译支持。（支持平台：`linux/amd64`、`linux/arm64`）
- `ghcr.io/ggml-org/llama.cpp:full-rocm`：与 `full` 相同，但使用 ROCm 编译支持。（支持平台：`linux/amd64`）
- `ghcr.io/ggml-org/llama.cpp:light-rocm`：与 `light` 相同，但使用 ROCm 编译支持。（支持平台：`linux/amd64`）
- `ghcr.io/ggml-org/llama.cpp:server-rocm`：与 `server` 相同，但使用 ROCm 编译支持。（支持平台：`linux/amd64`）
- `ghcr.io/ggml-org/llama.cpp:full-musa`：与 `full` 相同，但使用 MUSA 编译支持。（支持平台：`linux/amd64`）
- `ghcr.io/ggml-org/llama.cpp:light-musa`：与 `light` 相同，但使用 MUSA 编译支持。（支持平台：`linux/amd64`）
- `ghcr.io/ggml-org/llama.cpp:server-musa`：与 `server` 相同，但使用 MUSA 编译支持。（支持平台：`linux/amd64`）
- `ghcr.io/ggml-org/llama.cpp:full-intel`：与 `full` 相同，但使用 SYCL 编译支持。（支持平台：`linux/amd64`）
- `ghcr.io/ggml-org/llama.cpp:light-intel`：与 `light` 相同，但使用 SYCL 编译支持。（支持平台：`linux/amd64`）
- `ghcr.io/ggml-org/llama.cpp:server-intel`：与 `server` 相同，但使用 SYCL 编译支持。（支持平台：`linux/amd64`）
- `ghcr.io/ggml-org/llama.cpp:full-vulkan`：与 `full` 相同，但使用 Vulkan 编译支持。（支持平台：`linux/amd64`、`linux/arm64`）
- `ghcr.io/ggml-org/llama.cpp:light-vulkan`：与 `light` 相同，但使用 Vulkan 编译支持。（支持平台：`linux/amd64`、`linux/arm64`）
- `ghcr.io/ggml-org/llama.cpp:server-vulkan`：与 `server` 相同，但使用 Vulkan 编译支持。（支持平台：`linux/amd64`、`linux/arm64`）
- `ghcr.io/ggml-org/llama.cpp:full-openvino`：与 `full` 相同，但使用 OpenVino 编译支持。（支持平台：`linux/amd64`）
- `ghcr.io/ggml-org/llama.cpp:light-openvino`：与 `light` 相同，但使用 OpenVino 编译支持。（支持平台：`linux/amd64`）
- `ghcr.io/ggml-org/llama.cpp:server-openvino`：与 `server` 相同，但使用 OpenVino 编译支持。（支持平台：`linux/amd64`）
- `ghcr.io/ggml-org/llama.cpp:full-s390x`：与 `full` 完全相同，是 `s390x` 平台的别名。（支持平台：`linux/s390x`）
- `ghcr.io/ggml-org/llama.cpp:light-s390x`：与 `light` 完全相同，是 `s390x` 平台的别名。（支持平台：`linux/s390x`）
- `ghcr.io/ggml-org/llama.cpp:server-s390x`：与 `server` 完全相同，是 `s390x` 平台的别名。（支持平台：`linux/s390x`）

目前启用 GPU 的镜像除了构建外未经过 CI 进一步测试。它们的构建与 [.devops/](../.devops/) 中定义的 Dockerfile 以及 [.github/workflows/docker.yml](../.github/workflows/docker.yml) 中定义的 GitHub Action 完全一致，没有额外变体。如果你需要不同的设置（例如不同的 CUDA、ROCm 或 MUSA 库），目前需要在本地自行构建镜像。

## 使用方法

下载模型、转换为 ggml 并优化的最简单方式是使用 `--all-in-one` 命令，它会使用完整的 Docker 镜像。

将下方的 `/path/to/models` 替换为你实际下载模型所在的路径。

```bash
docker run -v /path/to/models:/models ghcr.io/ggml-org/llama.cpp:full --all-in-one "/models/" 7B
```

完成后，你就可以开始使用了！

```bash
docker run -v /path/to/models:/models ghcr.io/ggml-org/llama.cpp:full --run -m /models/7B/ggml-model-q4_0.gguf
docker run -v /path/to/models:/models ghcr.io/ggml-org/llama.cpp:full --run-legacy -m /models/32B/ggml-model-q8_0.gguf -no-cnv -p "Building a mobile app can be done in 15 steps:" -n 512
```

或者使用 light 镜像：

```bash
docker run -v /path/to/models:/models --entrypoint /app/llama-cli ghcr.io/ggml-org/llama.cpp:light -m /models/7B/ggml-model-q4_0.gguf
docker run -v /path/to/models:/models --entrypoint /app/llama-completion ghcr.io/ggml-org/llama.cpp:light -m /models/32B/ggml-model-q8_0.gguf -no-cnv -p "Building a mobile app can be done in 15 steps:" -n 512
```

或者使用 server 镜像：

```bash
docker run -v /path/to/models:/models -p 8080:8080 ghcr.io/ggml-org/llama.cpp:server -m /models/7B/ggml-model-q4_0.gguf --port 8080 --host 0.0.0.0 -n 512
```

在上述示例中，`--entrypoint /app/llama-cli` 是为了清晰起见而显式指定的，但你也可以安全地省略它，因为它是容器中的默认入口点。

## 使用 CUDA 的 Docker

假设你在 Linux 上已正确安装 [nvidia-container-toolkit](https://github.com/NVIDIA/nvidia-container-toolkit)，或者正在使用支持 GPU 的云环境，那么 `cuBLAS` 在容器内应该是可用的。

## 本地构建 Docker 镜像

```bash
docker build -t local/llama.cpp:full-cuda --target full -f .devops/cuda.Dockerfile .
docker build -t local/llama.cpp:light-cuda --target light -f .devops/cuda.Dockerfile .
docker build -t local/llama.cpp:server-cuda --target server -f .devops/cuda.Dockerfile .
```

根据容器主机支持的 CUDA 环境以及 GPU 架构，你可能需要传入不同的 `ARGS`。

默认值如下：

- `CUDA_VERSION` 设置为 `12.8.1`
- `CUDA_DOCKER_ARCH` 设置为 cmake 构建默认值，包含所有支持的架构

生成的镜像本质上与未启用 CUDA 的镜像相同：

1. `local/llama.cpp:full-cuda`：该镜像同时包含 `llama-cli` 和 `llama-completion` 可执行文件，以及将 LLaMA 模型转换为 ggml 并进行 4-bit 量化的工具。
2. `local/llama.cpp:light-cuda`：该镜像仅包含 `llama-cli` 和 `llama-completion` 可执行文件。
3. `local/llama.cpp:server-cuda`：该镜像仅包含 `llama-server` 可执行文件。

## 使用方法

本地构建完成后，使用方法与非 CUDA 示例类似，但你需要添加 `--gpus` 参数。同时你可能需要使用 `--n-gpu-layers` 参数。

```bash
docker run --gpus all -v /path/to/models:/models local/llama.cpp:full-cuda --run -m /models/7B/ggml-model-q4_0.gguf -p "Building a website can be done in 10 simple steps:" -n 512 --n-gpu-layers 1
docker run --gpus all -v /path/to/models:/models local/llama.cpp:light-cuda -m /models/7B/ggml-model-q4_0.gguf -p "Building a website can be done in 10 simple steps:" -n 512 --n-gpu-layers 1
docker run --gpus all -v /path/to/models:/models local/llama.cpp:server-cuda -m /models/7B/ggml-model-q4_0.gguf --port 8080 --host 0.0.0.0 -n 512 --n-gpu-layers 1
```

## 使用 MUSA 的 Docker

假设你在 Linux 上已正确安装 [mt-container-toolkit](https://developer.mthreads.com/musa/native)，那么 `muBLAS` 在容器内应该是可用的。

## 本地构建 Docker 镜像

```bash
docker build -t local/llama.cpp:full-musa --target full -f .devops/musa.Dockerfile .
docker build -t local/llama.cpp:light-musa --target light -f .devops/musa.Dockerfile .
docker build -t local/llama.cpp:server-musa --target server -f .devops/musa.Dockerfile .
```

根据容器主机支持的 MUSA 环境以及 GPU 架构，你可能需要传入不同的 `ARGS`。

默认值如下：

- `MUSA_VERSION` 设置为 `rc4.3.0`

生成的镜像本质上与未启用 MUSA 的镜像相同：

1. `local/llama.cpp:full-musa`：该镜像同时包含 `llama-cli` 和 `llama-completion` 可执行文件，以及将 LLaMA 模型转换为 ggml 并进行 4-bit 量化的工具。
2. `local/llama.cpp:light-musa`：该镜像仅包含 `llama-cli` 和 `llama-completion` 可执行文件。
3. `local/llama.cpp:server-musa`：该镜像仅包含 `llama-server` 可执行文件。

## 使用方法

本地构建完成后，使用方法与非 MUSA 示例类似，但你需要将 `mthreads` 设置为默认 Docker 运行时。可以通过在主机上执行 `(cd /usr/bin/musa && sudo ./docker setup $PWD)` 完成，并通过执行 `docker info | grep mthreads` 验证是否生效。同时你可能需要使用 `--n-gpu-layers` 参数。

```bash
docker run -v /path/to/models:/models local/llama.cpp:full-musa --run -m /models/7B/ggml-model-q4_0.gguf -p "Building a website can be done in 10 simple steps:" -n 512 --n-gpu-layers 1
docker run -v /path/to/models:/models local/llama.cpp:light-musa -m /models/7B/ggml-model-q4_0.gguf -p "Building a website can be done in 10 simple steps:" -n 512 --n-gpu-layers 1
docker run -v /path/to/models:/models local/llama.cpp:server-musa -m /models/7B/ggml-model-q4_0.gguf --port 8080 --host 0.0.0.0 -n 512 --n-gpu-layers 1
```
