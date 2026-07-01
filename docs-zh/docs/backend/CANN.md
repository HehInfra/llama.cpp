# 适用于 CANN 的 llama.cpp

 - [背景](#背景)
 - [新闻](#新闻)
 - [操作系统](#操作系统)
 - [硬件](#硬件)
 - [模型支持](#模型支持)
 - [数据类型支持](#数据类型支持)
 - [Docker](#docker)
 - [Linux](#linux)
 - [环境变量设置](#环境变量设置)
 - [TODO](#todo)


## 背景

**Ascend NPU** 是一系列采用神经网络处理单元（Neural Processing Unit）的 AI 处理器，可高效处理矩阵乘法、点积和标量运算。

**CANN**（Compute Architecture for Neural Networks，神经网络计算架构）是面向 AI 场景的异构计算架构，上层支持多种 AI 框架，下层服务 AI 处理器及编程开发。它在上下层之间起到关键的桥梁作用，是提升昇腾 AI 处理器计算效率的核心平台。同时，CANN 为不同应用场景提供了高效且易用的编程接口，帮助用户基于昇腾平台快速构建 AI 应用与服务。

**Llama.cpp + CANN**

llama.cpp 的 CANN 后端专为支持昇腾 NPU 而设计。它利用 AscendC 与 ACLNN 的能力，这两者已集成到 CANN Toolkit 和内核中，可直接使用昇腾 NPU。

## 新闻

- 2024.11
  - 支持 Ascend 310P NPU 上的 F16 和 F32 数据类型模型。
- 2024.8
  - 支持昇腾 NPU 上的 `Q4_0` 和 `Q8_0` 数据类型。
- 2024.7
  - 为昇腾 NPU 创建 CANN 后端。

## 操作系统

| 操作系统 | 状态  | 已验证                                         |
|:-------:|:-------:|:----------------------------------------------:|
| Linux   | 支持 | Ubuntu 22.04, OpenEuler22.03                   |


## 硬件

### 昇腾 NPU

您可以使用以下命令获取昇腾设备 ID：

```sh
lspci -n | grep -Eo '19e5:d[0-9a-f]{3}' | cut -d: -f2
```

**设备**

| 设备 ID | 产品系列 | 产品型号 | 芯片型号 | 验证状态 |
|:---------:|----------------|----------------|:----------:|:---------------:|
|    d803   | Atlas A3 Train |                |    910C    |                 |
|    d803   | Atlas A3 Infer |                |    910C    |                 |
|    d802   | Atlas A2 Train |                |    910B    |                 |
|    d802   | Atlas A2 Infer | Atlas 300I A2  |    910B    |     支持     |
|    d801   | Atlas Train    |                |     910    |                 |
|    d500   | Atlas Infer    | Atlas 300I Duo |    310P    |     支持     |

*注意：*

- 如果您的昇腾 NPU 设备遇到问题，请创建带有 **[CANN]** 前缀或标签的 issue。
- 如果您已成功在昇腾 NPU 设备上运行，请帮助更新上表。


## 模型支持

<details>
<summary>纯文本</summary>

| 模型名称                  | FP16  | Q4_0 | Q8_0 |
|:----------------------------|:-----:|:----:|:----:|
| Llama-2                     |   √   |   √  |   √  |
| Llama-3                     |   √   |   √  |   √  |
| Mistral-7B                  |   √   |   √  |   √  |
| Mistral MOE                 |   √   |   √  |   √  |
| DBRX                        |   -   |   -  |   -  |
| Falcon                      |   √   |   √  |   √  |
| Chinese LLaMA/Alpaca        |   √   |   √  |   √  |
| Vigogne(French)             |   √   |   √  |   √  |
| BERT                        |   x   |   x  |   x  |
| Koala                       |   √   |   √  |   √  |
| Baichuan                    |   √   |   √  |   √  |
| Aquila 1 & 2                |   √   |   √  |   √  |
| Starcoder models            |   √   |   √  |   √  |
| Refact                      |   √   |   √  |   √  |
| MPT                         |   √   |   √  |   √  |
| Bloom                       |   √   |   √  |   √  |
| Yi models                   |   √   |   √  |   √  |
| stablelm models             |   √   |   √  |   √  |
| DeepSeek models             |   x   |   x  |   x  |
| Qwen models                 |   √   |   √  |   √  |
| PLaMo-13B                   |   √   |   √  |   √  |
| Phi models                  |   √   |   √  |   √  |
| PhiMoE                      |   √   |   √  |   √  |
| GPT-2                       |   √   |   √  |   √  |
| Orion                       |   √   |   √  |   √  |
| InternlLM2                  |   √   |   √  |   √  |
| CodeShell                   |   √   |   √  |   √  |
| Gemma                       |   √   |   √  |   √  |
| Mamba                       |   √   |   √  |   √  |
| Xverse                      |   √   |   √  |   √  |
| command-r models            |   √   |   √  |   √  |
| Grok-1                      |   -   |   -  |   -  |
| SEA-LION                    |   √   |   √  |   √  |
| GritLM-7B                   |   √   |   √  |   √  |
| OLMo                        |   √   |   √  |   √  |
| OLMo 2                      |   √   |   √  |   √  |
| OLMoE                       |   √   |   √  |   √  |
| Granite models              |   √   |   √  |   √  |
| GPT-NeoX                    |   √   |   √  |   √  |
| Pythia                      |   √   |   √  |   √  |
| Snowflake-Arctic MoE        |   -   |   -  |   -  |
| Smaug                       |   √   |   √  |   √  |
| Poro 34B                    |   √   |   √  |   √  |
| Bitnet b1.58 models         |   √   |   x  |   x  |
| Flan-T5                     |   √   |   √  |   √  |
| Open Elm models             |   x   |   √  |   √  |
| chatGLM3-6B + ChatGLM4-9b +  GLMEdge-1.5b + GLMEdge-4b    |   √   |   √  |   √  |
| GLM-4-0414                  |   √   |   √  |   √  |
| SmolLM                      |   √   |   √  |   √  |
| EXAONE-3.0-7.8B-Instruct    |   √   |   √  |   √  |
| FalconMamba Models          |   √   |   √  |   √  |
| Jais Models                 |   -   |   x  |   x  |
| Bielik-11B-v2.3             |   √   |   √  |   √  |
| RWKV-6                      |   -   |   √  |   √  |
| QRWKV-6                     |   √   |   √  |   √  |
| GigaChat-20B-A3B            |   x   |   x  |   x  |
| Trillion-7B-preview         |   √   |   √  |   √  |
| Ling models                 |   √   |   √  |   √  |

</details>

<details>
<summary>多模态</summary>

| 模型名称                  | FP16  | Q4_0 | Q8_0 |
|:----------------------------|:-----:|:----:|:----:|
| LLaVA 1.5 models, LLaVA 1.6 models      |   x   |   x  |   x  |
|  BakLLaVA                   |   √   |   √  |   √  |
|  Obsidian                   |   √   |   -  |   -  |
|  ShareGPT4V                 |   x   |   -  |   -  |
|  MobileVLM 1.7B/3B models   |   -   |   -  |   -  |
|  Yi-VL                      |   -   |   -  |   -  |
|  Mini CPM                   |   √   |   √  |   √  |
|  Moondream                  |   √   |   √  |   √  |
|  Bunny                      |   √   |   -  |   -  |
|  GLM-EDGE                   |   √   |   √  |   √  |
|  Qwen2-VL                   |   √   |   √  |   √  |

</details>



## 数据类型支持

| 数据类型               | 910B    | 310P    |
|:----------------------:|:-------:|:-------:|
| FP16                   | 支持 | 支持 |
| Q8_0                   | 支持 | 部分 |
| Q4_0                   | 支持 | 部分 |
| BF16                   | 支持 |         |

> **310P 说明**
> - `Q8_0`：已实现数据转换 / buffer 路径，并支持 `GET_ROWS`，但不支持量化的 `MUL_MAT` / `MUL_MAT_ID`。
> - `Q4_0`：已实现数据转换 / buffer 路径，但不支持量化的 `MUL_MAT` / `MUL_MAT_ID`。

## Docker

### 构建镜像
一条命令即可获取内置 llama.cpp 的镜像。
```sh
docker build -t llama-cpp-cann -f .devops/llama-cli-cann.Dockerfile .
```

### 运行容器

```sh
# 查看所有卡。
npu-smi info

# 选择您要使用的卡，确保这些卡没有被其他人占用。
# 以下示例使用 device0。
docker run --name llamacpp \
  --device /dev/davinci0 \
  --device /dev/davinci_manager \
  --device /dev/devmm_svm \
  --device /dev/hisi_hdc \
  -v /usr/local/dcmi:/usr/local/dcmi \
  -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi \
  -v /usr/local/Ascend/driver/lib64/:/usr/local/Ascend/driver/lib64/ \
  -v /usr/local/Ascend/driver/version.info:/usr/local/Ascend/driver/version.info \
  -v /PATH_TO_YOUR_MODELS/:/app/models \
  -it llama-cpp-cann \
  -m /app/models/MODEL_PATH \
  -ngl 32 \
  -p "Building a website can be done in 10 simple steps:"
```

*注意：*

- 您可能需要在**宿主机**上安装昇腾驱动和固件 *(详情请参考 [Linux 配置](#linux))*。

## Linux

### I. 环境配置

1. **配置昇腾用户和用户组**

    ```sh
    sudo groupadd HwHiAiUser
    sudo useradd -g HwHiAiUser -d /home/HwHiAiUser -m HwHiAiUser -s /bin/bash
    sudo usermod -aG HwHiAiUser $USER
    ```

2. **安装依赖**

    **Ubuntu/Debian：**
    ```sh
    sudo apt-get update
    sudo apt-get install -y gcc python3 python3-pip linux-headers-$(uname -r)
    ```

    **RHEL/CentOS：**
    ```sh
    sudo yum makecache
    sudo yum install -y gcc python3 python3-pip kernel-headers-$(uname -r) kernel-devel-$(uname -r)
    ```

3. **安装 CANN（驱动 + toolkit）**

    > `Ascend-cann` 包包含驱动和 toolkit。
    > `$ARCH` 可以是 `x86_64` 或 `aarch64`，`$CHIP` 可以是 `910b` 或 `310p`。

    ```sh
    wget https://ascend-repo.obs.cn-east-2.myhuaweicloud.com/CANN/CANN%208.5.T63/Ascend-cann_8.5.0_linux-$ARCH.run
    sudo bash ./Ascend-cann_8.5.0_linux-$ARCH.run --install

    wget https://ascend-repo.obs.cn-east-2.myhuaweicloud.com/CANN/CANN%208.5.T63/Ascend-cann-$CHIP-ops_8.5.0_linux-$ARCH.run
    sudo bash ./Ascend-cann-$CHIP-ops_8.5.0_linux-$ARCH.run --install
    ```

4. **验证安装**

    ```sh
    npu-smi info
    ```

    如果能正确显示设备信息，说明驱动工作正常。

    ```sh
    # 设置环境变量（路径可按需调整）
    source /usr/local/Ascend/cann/set_env.sh

    python3 -c "import acl; print(acl.get_soc_name())"
    ```

    如果命令输出芯片型号，则表示安装成功。

### II. 构建 llama.cpp

```sh
cmake -B build -DGGML_CANN=on -DCMAKE_BUILD_TYPE=release
cmake --build build --config release
```

### III. 运行推理

1. **获取并准备模型**

    您可以参考通用的 [*获取和量化模型*](../../README.md#obtaining-and-quantizing-models) 指南来准备模型。

    **注意**：

      - CANN 后端目前仅支持 FP16/Q4_0/Q8_0 模型。

2. **启动推理**

    设备选择有两种模式：

    - 单设备：使用用户指定的一个设备。
    - 多设备：自动选择使用相同后端的设备。

    | 设备选择 | 参数                              |
    |:----------------:|:--------------------------------------:|
    | 单设备    | --split-mode none --main-gpu DEVICE_ID |
    | 多设备 | --split-mode layer (默认)           |

    示例：

    - 使用 device 0：

    ```sh
    ./build/bin/llama-cli -m path_to_model -p "Building a website can be done in 10 simple steps:" -n 400 -e -ngl 33 -sm none -mg 0
    ```

    - 使用多设备：

    ```sh
    ./build/bin/llama-cli -m path_to_model -p "Building a website can be done in 10 simple steps:" -n 400 -e -ngl 33 -sm layer
    ```

### **GitHub 贡献**：
请在 issue/PR 标题中添加 **[CANN]** 前缀或标签，以便 CANN 团队及时查看和处理。

## Updates
### Basic Flash Attention Support
The basic FA kernel with aclnnops has been added in aclnn_ops.cpp.
Currently, the FA only supports the cases with FP16 KV tensors and NO logit softcap.
Since the aclnn interface for flash attention cannot support the logit softcap, we will only update the quantized version in the future.

Authors from Peking University: Bizhao Shi (bshi@pku.edu.cn), Yuxin Yang (yxyang@pku.edu.cn), Ruiyang Ma (ruiyang@stu.pku.edu.cn), and Guojie Luo (gluo@pku.edu.cn).

We would like to thank Tuo Dai, Shanni Li, and all of the project maintainers from Huawei Technologies Co., Ltd for their help during the code development and pull request.

## 环境变量设置

### GGML_CANN_MEM_POOL

指定内存池管理策略，默认为 vmm。

- vmm：使用虚拟内存管理器内存池。如果硬件不支持 VMM，则回退到传统（leg）内存池。

- prio：使用基于优先级队列的内存池管理。

- leg：使用固定大小的 buffer 池。

### GGML_CANN_DISABLE_BUF_POOL_CLEAN

控制内存池的自动清理。此选项仅在使用 prio 或 leg 内存池策略时生效。

### GGML_CANN_WEIGHT_NZ

将 matmul 权重格式从 ND 转换为 NZ 以提升性能。默认启用。

### GGML_CANN_ACL_GRAPH

使用 ACL 图执行方式运行算子，而不是逐算子（eager）模式。默认启用。此选项仅在编译时启用了 `USE_ACL_GRAPH` 的情况下生效。如需启用，请重新编译：

```sh
cmake -B build -DGGML_CANN=on -DCMAKE_BUILD_TYPE=release -DUSE_ACL_GRAPH=ON
cmake --build build --config release
```

### GGML_CANN_GRAPH_CACHE_CAPACITY

LRU 缓存中保留的已编译 CANN 图的最大数量，默认为 12。当缓存的图数量超过该容量时，最近最少使用的图将被淘汰。

### GGML_CANN_PREFILL_USE_GRAPH

在 prefill 阶段启用 ACL 图执行，默认为 false。此选项仅在启用 FA 时生效。

### GGML_CANN_OPERATOR_FUSION

在计算过程中启用算子融合，默认为 false。此选项将兼容的算子（例如 ADD + RMS_NORM）进行融合，以减少开销并提升性能。
