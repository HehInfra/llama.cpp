# 基于 Snapdragon 的 Linux 设备

## Docker 环境设置

为基于 Snapdragon 的 Linux 设备构建 llama.cpp 最简单的方式是使用工具链 Docker 镜像（参见 [github.com/snapdragon-toolchain](https://github.com/snapdragon-toolchain)）。
该镜像包含 OpenCL SDK、Hexagon SDK、CMake 以及 ARM64 Linux 交叉编译工具链。

交叉编译支持在 **Linux X86** 主机上进行。生成的二进制文件将部署到目标 **Qualcomm Snapdragon ARM64 Linux** 设备上运行。

```
~/src/llama.cpp$ docker run -it -u $(id -u):$(id -g) --volume $(pwd):/workspace --platform linux/amd64 ghcr.io/snapdragon-toolchain/arm64-linux:v0.1
[d]/> cd /workspace
```

注意：后续 **Linux** 构建流程均假设你正在工具链容器内运行。


## 如何构建

下面通过 CMake presets 构建包含 CPU、OpenCL 和 Hexagon 后端的 llama.cpp：

```
[d]/workspace> cp docs/backend/snapdragon/CMakeUserPresets.json .

[d]/workspace> cmake --preset arm64-linux-snapdragon-release -B build-snapdragon

[d]/workspace> cmake --build build-snapdragon -j $(nproc)
```

要生成可安装的“包”，只需使用 cmake --install，然后打包为 zip：

```
[d]/workspace> cmake --install build-snapdragon --prefix pkg-snapdragon
[d]/workspace> zip -r pkg-snapdragon.zip pkg-snapdragon
```

## 如何安装

在此步骤中，你需要将构建好的二进制文件和库部署到目标 Linux 设备。将 `pkg-snapdragon.zip` 传输到目标设备，然后解压并设置环境变量：

```
$ unzip pkg-snapdragon.zip
$ cd pkg-snapdragon
$ export LD_LIBRARY_PATH=./lib
$ export ADSP_LIBRARY_PATH=./lib
```

此时，你还应该在设备上下载一些模型：

```
$ wget https://huggingface.co/bartowski/Llama-3.2-3B-Instruct-GGUF/resolve/main/Llama-3.2-3B-Instruct-Q4_0.gguf
```

## 如何运行

接下来，由于我们已经设置好了环境变量，可以使用 Hexagon 后端运行 llama-cli：
```
$ ./bin/llama-cli -m Llama-3.2-3B-Instruct-Q4_0.gguf --device HTP0 -ngl 99 -p "what is the most popular cookie in the world?"
```
