> [!IMPORTANT]
> 本构建文档仅适用于 IBM Z & LinuxONE 大型机（s390x）。其他架构的构建文档请参见 [build.md](build.md)。

# 本地构建 llama.cpp（适用于 s390x）

本项目的主要产物是 `llama` 库。其 C 风格接口位于 [include/llama.h](../include/llama.h)。

该项目还包含许多使用 `llama` 库的示例程序与工具。示例涵盖从简单的极简代码片段到复杂的子项目（例如兼容 OpenAI 的 HTTP 服务器）。

**获取代码：**

```bash
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
```

## 使用 BLAS 构建 CPU 版本

强烈建议启用 BLAS 支持来构建 llama.cpp，因为这已被证明可以提升性能。请确保环境中已安装 OpenBLAS。

```bash
cmake -S . -B build             \
    -DCMAKE_BUILD_TYPE=Release  \
    -DGGML_BLAS=ON              \
    -DGGML_BLAS_VENDOR=OpenBLAS

cmake --build build --config Release -j $(nproc)
```

**说明**：

-   如需加快重复编译速度，请安装 [ccache](https://ccache.dev/)
-   默认情况下 VXE/VXE2 已启用。如需禁用它（不推荐）：

    ```bash
    cmake -S . -B build             \
        -DCMAKE_BUILD_TYPE=Release  \
        -DGGML_BLAS=ON              \
        -DGGML_BLAS_VENDOR=OpenBLAS \
        -DGGML_VXE=OFF

    cmake --build build --config Release -j $(nproc)
    ```

-   调试构建：

    ```bash
    cmake -S . -B build             \
        -DCMAKE_BUILD_TYPE=Debug    \
        -DGGML_BLAS=ON              \
        -DGGML_BLAS_VENDOR=OpenBLAS
    cmake --build build --config Debug -j $(nproc)
    ```

-   静态构建时，请添加 `-DBUILD_SHARED_LIBS=OFF`：

    ```bash
    cmake -S . -B build             \
        -DCMAKE_BUILD_TYPE=Release  \
        -DGGML_BLAS=ON              \
        -DGGML_BLAS_VENDOR=OpenBLAS \
        -DBUILD_SHARED_LIBS=OFF

    cmake --build build --config Release -j $(nproc)
    ```

## IBM zDNN 加速器

该加速器利用 Telum I 和 Telum II 处理器中的 IBM zAIU 协处理器提供加速。请确保已安装 [IBM zDNN 库](https://github.com/IBM/zDNN)。

#### 从 IBM 源码编译

官方构建说明见：[Building and Installing zDNN](https://github.com/IBM/zDNN?tab=readme-ov-file#building-and-installing-zdnn)

### 编译

```bash
cmake -S . -B build             \
    -DCMAKE_BUILD_TYPE=Release  \
    -DGGML_ZDNN=ON
cmake --build build --config Release -j$(nproc)
```

## 获取 GGUF 模型

所有模型都需要转换为 Big-Endian。可通过以下三种方式实现：

1. **使用已针对 IBM Z & LinuxONE 验证过的预转换模型（最简单）**

    ![File Type - gguf](https://img.shields.io/badge/File_Type-gguf-fff)

    你可以在 [s390x Verified Models](https://huggingface.co/collections/taronaeo/s390x-verified-models-672765393af438d0ccb72a08) 或 [s390x Runnable Models](https://huggingface.co/collections/taronaeo/s390x-runnable-models-686e951824198df12416017e) 找到经过预转换与验证的热门模型。

    这些模型已从 `safetensors` 转换为 `GGUF` Big-Endian，并且其对应的分词器已在 IBM z15 及更高版本系统上验证可正常运行。

2. **直接将 safetensors 模型转换为 GGUF Big-Endian（推荐）**

    ![File Type - safetensors](https://img.shields.io/badge/File_Type-safetensors-da1e28)

    待转换的模型必须是 `safetensors` 文件格式（例如 [IBM Granite 3.3 2B](https://huggingface.co/ibm-granite/granite-3.3-2b-instruct)）。请确保已下载该模型仓库。

    请确保提前安装所需的依赖包

    ```bash
    pip3 install -r requirements.txt
    ```

    将 `safetensors` 模型转换为 `GGUF`

    ```bash
    python3 convert_hf_to_gguf.py \
        --outfile model-name-be.f16.gguf \
        --outtype f16 \
        --bigendian \
        model-directory/
    ```

    例如，

    ```bash
    python3 convert_hf_to_gguf.py \
        --outfile granite-3.3-2b-instruct-be.f16.gguf \
        --outtype f16 \
        --bigendian \
        granite-3.3-2b-instruct/
    ```

3. **将现有的 GGUF Little-Endian 模型转换为 Big-Endian**

    ![File Type - gguf](https://img.shields.io/badge/File_Type-gguf-fff)

    待转换的模型必须是 `gguf` 文件格式（例如 [IBM Granite 3.3 2B GGUF](https://huggingface.co/ibm-granite/granite-3.3-2b-instruct-GGUF)）。请确保已下载该模型文件。

    ```bash
    python3 gguf-py/gguf/scripts/gguf_convert_endian.py model-name.f16.gguf BIG
    ```

    例如，

    ```bash
    python3 gguf-py/gguf/scripts/gguf_convert_endian.py granite-3.3-2b-instruct-le.f16.gguf BIG
    mv granite-3.3-2b-instruct-le.f16.gguf granite-3.3-2b-instruct-be.f16.gguf
    ```

    **说明：**

    - 当前 GGUF 字节序转换脚本可能尚未支持所有数据类型，某些模型或量化方式可能会转换失败。如遇此情况，请尝试通过步骤 2 手动将 safetensors 模型转换为 GGUF Big-Endian。

## IBM 加速器

### 1. SIMD 加速

仅在 IBM z15/LinuxONE 3 或更高版本系统上可用，需使用 `-DGGML_VXE=ON` 编译标志（默认开启）。在较旧的系统（如 IBM z14/arch12）上，llama.cpp 无法使用硬件加速。在这些系统上，API 仍可运行，但会使用标量实现。

### 2. zDNN 加速器（进行中）

仅在 IBM z17/LinuxONE 5 或更高版本系统上可用，需使用 `-DGGML_ZDNN=ON` 编译标志。在较旧的系统（如 IBM z15/arch13）上，llama.cpp 无法使用硬件加速。在这些系统上，API 将回退到 CPU 例程。

### 3. Spyre 加速器

_仅适用于 IBM z17 / LinuxONE 5 或更高版本系统。当前暂不支持。_

## 性能调优

### 1. 虚拟化配置

为获得最佳性能，强烈建议仅使用 LPAR（Type-1）虚拟化。

注意：当前不支持 Type-2 虚拟化，虽然可以运行，但性能不会达到最佳。

### 2. IFL（核心）数量

建议为 LPAR 分配至少 8 个共享 IFL。将 IFL 数量增加到超过 8 个共享 IFL 只会提升提示处理（Prompt Processing）性能，而不会提升 Token 生成（Token Generation）性能。

注意：IFL 数量不等于 vCPU 数量。

### 3. SMT 与 NOSMT（同步多线程）

由于 SMT 会对性能产生负面影响，强烈建议通过内核启动参数禁用 SMT。请参考你所使用的 Linux 发行版关于通过内核启动参数禁用 SMT 的指南。

### 4. BLAS 与 NOBLAS

IBM VXE/VXE2 SIMD 加速依赖于 BLAS 实现。强烈建议使用 BLAS。

## 常见问题解答（FAQ）

1. 加载模型时遇到以下错误信息：`gguf_init_from_file_impl: failed to load model: this GGUF file version 50331648 is extremely large, is there a mismatch between the host and model endianness?`

    回答：请确保你下载或转换的模型是 GGUFv3 Big-Endian。这些模型通常以 `-be` 后缀标识，例如 `granite-3.3-2b-instruct-be.F16.gguf`。

    你可以参考 [获取 GGUF 模型](#getting-gguf-models) 部分，手动将 `safetensors` 模型转换为 `GGUF` Big Endian。

2. 对模型进行推理时性能极差

    回答：请参考 [附录 B：SIMD 支持矩阵](#appendix-b-simd-support-matrix)，检查你的模型量化方式是否受 SIMD 加速支持。

3. 在 IBM z17 上构建时遇到以下错误信息：`invalid switch -march=z17`

    回答：请确保 GCC 编译器版本至少为 GCC 15.1.0，并将 `binutils` 更新到最新版本。如果问题仍未解决，请提交 issue。

4. 使用 GCC 15+ 安装 `sentencepiece` 包失败

    回答：`sentencepiece` 团队已经知道此问题，详见 [此 issue](https://github.com/google/sentencepiece/issues/1108)。

    作为临时解决方案，请在安装命令中使用以下环境变量。

    ```bash
    export CXXFLAGS="-include cstdint"
    ```

    例如，

    ```bash
    CXXFLAGS="-include cstdint" pip3 install -r requirements.txt
    ```

## 获取 IBM Z & LinuxONE 相关帮助

1. **Bug 与功能请求**

    请在 llama.cpp 仓库提交 issue，并确保标题包含 "s390x"。

2. **其他问题**

    请直接联系 [aionz@us.ibm.com](mailto:aionz@us.ibm.com)。

## 附录 A：硬件支持矩阵

|          | 支持情况 | 最低编译器版本 |
| -------- | ------- | -------------- |
| IBM z15  | ✅      |                |
| IBM z16  | ✅      |                |
| IBM z17  | ✅      | GCC 15.1.0     |
| IBM zDNN | ✅      |                |

-   ✅ - 已支持并验证可按预期运行
-   🚫 - 不支持，我们可能无法提供支持

## 附录 B：SIMD 支持矩阵

|            | VX/VXE/VXE2 | zDNN | Spyre |
|------------|-------------|------|-------|
| FP32       | ✅           | ✅    | ❓     |
| FP16       | ✅           | ✅    | ❓     |
| BF16       | ✅           | ✅    | ❓     |
| Q4_0       | ✅           | ❓    | ❓     |
| Q4_1       | ✅           | ❓    | ❓     |
| MXFP4      | ✅           | ❓    | ❓     |
| Q5_0       | ✅           | ❓    | ❓     |
| Q5_1       | ✅           | ❓    | ❓     |
| Q8_0       | ✅           | ❓    | ❓     |
| Q2_K       | 🚫           | ❓    | ❓     |
| Q3_K       | ✅           | ❓    | ❓     |
| Q4_K       | ✅           | ❓    | ❓     |
| Q5_K       | ✅           | ❓    | ❓     |
| Q6_K       | ✅           | ❓    | ❓     |
| TQ1_0      | 🚫           | ❓    | ❓     |
| TQ2_0      | 🚫           | ❓    | ❓     |
| IQ2_XXS    | 🚫           | ❓    | ❓     |
| IQ2_XS     | 🚫           | ❓    | ❓     |
| IQ2_S      | 🚫           | ❓    | ❓     |
| IQ3_XXS    | 🚫           | ❓    | ❓     |
| IQ3_S      | 🚫           | ❓    | ❓     |
| IQ1_S      | 🚫           | ❓    | ❓     |
| IQ1_M      | 🚫           | ❓    | ❓     |
| IQ4_NL     | ✅           | ❓    | ❓     |
| IQ4_XS     | ✅           | ❓    | ❓     |
| FP32->FP16 | 🚫           | ❓    | ❓     |
| FP16->FP32 | 🚫           | ❓    | ❓     |

-   ✅ - 支持加速
-   🚫 - 不支持加速，仍将使用标量实现运行
-   ❓ - 加速情况未知，如果你能自行测试，欢迎贡献

最后更新者：**Aaron Teo (aaron.teo1@ibm.com)**，2026 年 2 月 15 日。
