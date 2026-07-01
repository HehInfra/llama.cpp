# 适用于 AMD ZenDNN 的 llama.cpp

> [!WARNING]
> **注意：** ZenDNN 与 zDNN 不同。
> - **ZenDNN**（本页）：AMD 面向 AMD EPYC CPU 的深度学习库
> - **zDNN**：IBM 面向 IBM Z 与 LinuxONE 大型机的深度神经网络加速库（[参见 zDNN 文档](zDNN.md)）

- [背景](#背景)
- [操作系统](#操作系统)
- [硬件](#硬件)
- [支持的操作](#支持的操作)
- [数据类型支持](#数据类型支持)
- [Linux](#linux)
- [环境变量](#环境变量)
- [性能优化](#性能优化)
- [已知问题](#已知问题)
- [待办事项](#待办事项)

## 背景

**ZenDNN**（Zen Deep Neural Network Library）是 AMD 面向 AMD EPYC™ CPU 优化的高性能深度学习推理库。它针对 AMD Zen 处理器架构上的神经网络工作负载，提供关键深度学习原语与操作的优化实现，可显著提升性能。

**Llama.cpp + ZenDNN**

llama.cpp 的 ZenDNN 后端利用 AMD 优化的矩阵乘法原语，加速 AMD CPU 上的推理。它采用 ZenDNN 的 **LowOHA（Low Overhead Hardware Accelerated）** MatMul 算子，实现高效的 GEMM 运算，具有极低的执行开销、内置权重缓存，并可直接访问后端库（AOCL DLP、LibXSMM、OneDNN）。

有关 ZenDNN 的更多信息，请访问：https://www.amd.com/en/developer/zendnn.html

## 操作系统

| 操作系统 | 状态    | 已验证                                |
|:-------:|:-------:|:------------------------------------:|
| Linux   | 支持    | Ubuntu 20.04, 22.04, 24.04           |

有关最新支持的操作系统列表，请参阅 [ZenDNN 支持的操作系统](https://github.com/amd/ZenDNN/blob/a18adf8c605fb5f5e52cefd7eda08a7b18febbaf/README.md#15-supported-os)。

## 硬件

### AMD CPU

**推荐处理器**

ZenDNN 针对基于 "Zen" 微架构及更新架构的 AMD EPYC™ 处理器和 AMD Ryzen™ 处理器进行了优化。

| CPU 系列                       | 状态    | 说明                                  |
|:-----------------------------:|:-------:|:------------------------------------:|
| AMD EPYC™ 9005 系列 (Turin)   | 支持    | 第 5 代 - Zen 5 架构                 |
| AMD EPYC™ 9004 系列 (Genoa)   | 支持    | 第 4 代 - Zen 4 架构                 |
| AMD EPYC™ 7003 系列 (Milan)   | 支持    | 第 3 代 - Zen 3 架构                 |
| AMD Ryzen™ AI MAX (Strix Halo)| 支持    | 高性能移动处理器                      |

*说明：*

- 在拥有大量核心的 AMD EPYC™ 处理器（例如 EPYC 9005 系列）上可获得最佳性能。
- ZenDNN 利用 AMD 的高级 CPU 特性，包括 AVX2 和 AVX-512 指令集。
- 为获得最佳性能，请确保系统具备充足的内存带宽。

## 支持的操作

ZenDNN 后端可加速**矩阵乘法（MUL_MAT）**和**基于专家的矩阵乘法（MUL_MAT_ID）**操作。其他操作由标准 CPU 后端处理。

| 操作       | 状态    | 说明                                        |
|:-----------|:-------:|:-------------------------------------------:|
| MUL_MAT    | 支持    | 通过 ZenDNN LowOHA MatMul 加速              |
| MUL_MAT_ID | 支持    | 通过 ZenDNN LowOHA MatMul 加速（MoE）        |

*注意：* 由于 MUL_MAT 和 MUL_MAT_ID 被加速，当矩阵乘法占据计算工作负载的主导地位时，模型从 ZenDNN 中获益最大（这对于基于 Transformer 的大语言模型和混合专家模型来说很典型）。

## 数据类型支持

| 数据类型                | 状态    | 说明                                         |
|:----------------------:|:-------:|:--------------------------------------------:|
| FP32                   | 支持    | 全精度浮点数                                  |
| BF16                   | 支持    | BFloat16（在 Zen 4/Zen 5 上性能最佳）         |
| Q8_0                   | 支持    | 通过[动态量化](https://github.com/amd/ZenDNN/blob/main/docs/operator/lowoha_matmul_operator.md)实现的 8 位量化权重 |

*说明：*

- **BF16** 在 Zen 4 和 Zen 5 EPYC™ 处理器（Genoa、Turin）上提供最佳性能。
- **Q8_0** 可用于量化模型权重，因为 ZenDNN 支持动态量化的 [LowOHA MatMul 算子](https://github.com/amd/ZenDNN/blob/main/docs/operator/lowoha_matmul_operator.md)。
- 其他量化格式将回退到标准 CPU 后端，除非 ZenDNN 后端明确支持。

## Linux

### I. 环境配置

配置 ZenDNN 有两种方式：

#### 选项 1：自动下载与构建（推荐）

CMake 将自动为你下载并构建 ZenDNN：

```sh
# 构建 llama.cpp - ZenDNN 将被自动下载并构建
cmake -B build -DGGML_ZENDNN=ON -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release -j $(nproc)
```

无需手动安装 ZenDNN。CMake 会自动处理所有内容。

#### 选项 2：使用自定义 ZenDNN 安装

如果你想自行构建 ZenDNN 或使用特定版本：

**步骤 1：从源码构建 ZenDNN**

```sh
# 克隆 ZenDNN 仓库
git clone https://github.com/amd/ZenDNN.git
cd ZenDNN

# 构建并安装（需要 CMake >= 3.25）
mkdir build && cd build
cmake ..
cmake --build . --target all
```

默认安装路径：`ZenDNN/build/install`

**有关详细的构建说明**，请参阅 [ZenDNN README](https://github.com/amd/ZenDNN/blob/a18adf8c605fb5f5e52cefd7eda08a7b18febbaf/README.md)。

**步骤 2：使用自定义 ZenDNN 路径构建 llama.cpp**

```sh
# 使用环境变量
export ZENDNN_ROOT=/path/to/ZenDNN/build/install
cmake -B build -DGGML_ZENDNN=ON -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release -j $(nproc)

# 或在 CMake 中直接指定路径
cmake -B build -DGGML_ZENDNN=ON -DZENDNN_ROOT=/path/to/ZenDNN/build/install -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release -j $(nproc)
```

### II. 运行服务器

#### 1. 下载模型

下载 LLaMA 3.1 8B Instruct BF16 模型：

```sh
# 从 Hugging Face 下载
huggingface-cli download meta-llama/Llama-3.1-8B-Instruct-GGUF --local-dir models/
```

你也可以使用 Q8_0 GGUF 模型：

```sh
# 从 Hugging Face 下载 Q8_0 GGUF 模型
huggingface-cli download meta-llama/Llama-3.1-8B-Instruct-GGUF \
    Llama-3.1-8B-Instruct-Q8_0.gguf \
    --local-dir models/
```

#### 2. 启动服务器

使用 ZenDNN 加速运行 llama.cpp 服务器：

```sh
# 设置最佳配置
export ZENDNNL_MATMUL_ALGO=1    # Blocked AOCL DLP 算法以获得最佳性能

# 启动服务器
./build/bin/llama-server \
    -m models/Llama-3.1-8B-Instruct.BF16.gguf \
    --host 0.0.0.0 \
    --port 8080 \
    -t 64
```

在 `http://localhost:8080` 访问服务器。

**性能提示**：
- 使用 `ZENDNNL_MATMUL_ALGO=1` 以获得最佳性能
- 对于 NUMA 系统：`numactl --cpunodebind=0 --membind=0 ./build/bin/llama-server ...`

## 环境变量

有关 ZenDNN 相关的环境变量，请参阅 [ZenDNN 环境变量文档](https://github.com/amd/ZenDNN/blob/a18adf8c605fb5f5e52cefd7eda08a7b18febbaf/docs/runtime_env.md)。

### 性能优化

ZenDNN 的 LowOHA MatMul 支持多种后端算法。为获得**最佳性能**，请使用 **Blocked AOCL DLP** 算法：

```sh
export ZENDNNL_MATMUL_ALGO=1    # Blocked AOCL DLP 算法（推荐）
```

有关可用算法的更多详细信息，请参阅 [ZenDNN MatMul 算法文档](https://github.com/amd/ZenDNN/blob/a18adf8c605fb5f5e52cefd7eda08a7b18febbaf/docs/runtime_env.md#algorithm-details)。

### Q8_0 性能说明

Q8_0 支持主要有益于提示词处理 / 预填充（prefill）工作负载，此时大型矩阵乘法占主导地位。根据模型、批次大小、线程数和 CPU 拓扑结构的不同，token 生成性能可能与标准 CPU 后端接近。

### 性能分析与调试

有关详细的性能分析和日志选项，请参阅 [ZenDNN 日志文档](https://github.com/amd/ZenDNN/blob/a18adf8c605fb5f5e52cefd7eda08a7b18febbaf/docs/logging.md)。

## 已知问题

- **有限的操作支持**：目前矩阵乘法（MUL_MAT）和基于专家的矩阵乘法（MUL_MAT_ID）通过 ZenDNN 加速。其他操作回退到标准 CPU 后端。未来更新可能会扩展支持的操作。
- **BF16 支持**：BF16 操作需要 AMD Zen 4 或 Zen 5 架构（EPYC 9004/9005 系列）。在较旧的 CPU 上，操作将使用 FP32。
- **Q8_0 支持范围**：Q8_0 加速适用于支持的矩阵乘法路径。其他量化格式仍回退到标准 CPU 后端。
- **NUMA 感知**：对于多路系统，可能需要手动进行 NUMA 绑定以获得最佳性能。

## 问答

**Q：如何验证是否正在使用 ZenDNN 后端？**

A：运行 llama.cpp 时检查日志输出。你应该能看到指示 ZenDNN 后端已初始化的消息。也可以在输出中查看后端名称。

**Q：我可以期待多大的性能提升？**

A：性能提升因模型大小、批次大小和 CPU 架构而异。在 AMD EPYC 处理器上，与标准 CPU 推理相比，矩阵乘法操作通常可获得 1.1 倍到 2 倍的加速。

**Q：我可以在非 AMD 处理器上使用 ZenDNN 吗？**

A：ZenDNN 专门针对 AMD 处理器进行了优化。虽然它可能在其他 x86-64 CPU 上运行，但只有在基于 AMD Zen 的架构上才能保证获得性能收益。

**Q：ZenDNN 支持量化模型吗？**

A：支持。ZenDNN 后端对支持的矩阵乘法操作支持 Q8_0 量化模型。FP32 和 BF16 也受支持。其他量化格式可能会回退到标准 CPU 后端，除非 ZenDNN 后端明确支持。

**Q：为什么使用 ZenDNN 后我的推理没有变快？**

A：请确认：
1. 你使用的是 AMD EPYC 或 Ryzen 处理器（Zen 2 或更新）
2. 为获得最佳性能已设置 `ZENDNNL_MATMUL_ALGO=1`（Blocked AOCL DLP）
3. 你使用的模型足够大（小模型可能收益不明显）
4. 启用性能分析以验证 ZenDNN MatMul 是否被调用

### **GitHub 贡献**：
请在 issue/PR 标题中添加 **[ZenDNN]** 前缀/标签，以便 ZenDNN 团队及时查看和处理。

## 待办事项

- 扩展 MUL_MAT 和 MUL_MAT_ID 之外的操作支持（attention 操作、激活函数等）
