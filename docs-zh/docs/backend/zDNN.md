# 在 IBM zDNN 加速器上运行 llama.cpp

> [!WARNING]
> **注意：** zDNN **并非** ZenDNN。
> - **zDNN**（本页）：IBM 面向 IBM Z 与 LinuxONE 大型机的深度神经网络加速库
> - **ZenDNN**：AMD 面向 AMD EPYC CPU 的深度学习库（[参见 ZenDNN 文档](ZenDNN.md)）

## 背景

IBM zDNN（Z Deep Neural Network）是一款硬件加速库，专门用于利用 IBM Telum I 与 II 处理器中集成的 IBM NNPA（Neural Network Processor Assist）加速器，能够显著提升神经网络推理操作的性能。

### Llama.cpp + IBM zDNN

llama.cpp 的 zDNN 后端旨在通过 IBM zDNN 硬件加速库，使 llama.cpp 能够在 IBM z17 及更高版本的系统上运行。

## 软件与硬件支持

| 硬件层级             | 状态          | 已验证                      |
| -------------------- | ------------- | --------------------------- |
| IBM z17 / LinuxONE 5 | 支持          | RHEL 9.6、IBM z17、40 IFLs  |
| IBM z16 / LinuxONE 4 | 不支持        |                             |

## 支持的数据类型

| 数据类型 | 状态   |
| ---------| ------|
| F32      | 支持   |
| F16      | 支持   |
| BF16     | 支持   |

## CMake 选项

IBM zDNN 后端提供以下 CMake 选项，用于控制后端行为。

| CMake 选项  | 默认值   | 说明                            |
| ------------| -------- | ------------------------------- |
| `GGML_ZDNN` | `OFF`    | 编译带 zDNN 支持的 llama.cpp     |
| `ZDNN_ROOT` | `""`     | 覆盖 zDNN 库查找路径             |

## 1. 安装 zDNN 库

注意：通过 `apt` 或 `yum` 提供的 zDNN 库可能无法正常工作，如 [#15772](https://github.com/ggml-org/llama.cpp/issues/15772) 所述。建议从源码编译。

```sh
git clone --recurse-submodules https://github.com/IBM/zDNN
cd zDNN

autoreconf .
./configure --prefix=/opt/zdnn-libs

make build
sudo make install
```

## 2. 构建 llama.cpp

```sh
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp

cmake -S . -G Ninja -B build \
    -DCMAKE_BUILD_TYPE=Release \
    -DGGML_ZDNN=ON \
    -DZDNN_ROOT=/opt/zdnn-libs
cmake --build build --config Release -j$(nproc)
```
