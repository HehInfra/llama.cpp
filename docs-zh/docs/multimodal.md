# 多模态

llama.cpp 通过 `libmtmd` 支持多模态输入。目前有两个工具支持该功能：
- [llama-mtmd-cli](../tools/mtmd/README.md)
- [llama-server](../tools/server/README.md) 通过兼容 OpenAI 的 `/chat/completions` API

目前，我们支持**图像**和**音频**输入。音频功能仍处于高度实验阶段，质量可能有所下降。

要启用多模态功能，可使用以下两种方法之一：

- 对支持的模型使用 `-hf` 选项（参见下方的预量化模型列表）
    - 使用 `-hf` 加载模型并禁用多模态时，请使用 `--no-mmproj`
    - 使用 `-hf` 加载模型并使用自定义 mmproj 文件时，请使用 `--mmproj local_file.gguf`
- 使用 `-m model.gguf` 选项并配合 `--mmproj file.gguf`，分别指定文本模型和多模态 projector

默认情况下，多模态 projector 会被卸载到 GPU。如需禁用，请添加 `--no-mmproj-offload`

例如：

```sh
# CLI 简单用法
llama-mtmd-cli -hf ggml-org/gemma-3-4b-it-GGUF

# server 简单用法
llama-server -hf ggml-org/gemma-3-4b-it-GGUF

# 使用本地文件
llama-server -m gemma-3-4b-it-Q4_K_M.gguf --mmproj mmproj-gemma-3-4b-it-Q4_K_M.gguf

# 禁用 GPU 卸载
llama-server -hf ggml-org/gemma-3-4b-it-GGUF --no-mmproj-offload
```

> [!IMPORTANT]
>
> OCR 模型使用特定的 prompt 和输入结构进行训练，如需了解更多信息，请参考以下讨论：
> - PaddleOCR-VL: https://github.com/ggml-org/llama.cpp/pull/18825
> - GLM-OCR: https://github.com/ggml-org/llama.cpp/pull/19677
> - Deepseek-OCR: https://github.com/ggml-org/llama.cpp/pull/17400
> - Dots.OCR: https://github.com/ggml-org/llama.cpp/pull/17575
> - HunyuanOCR: https://github.com/ggml-org/llama.cpp/pull/21395

## 预量化模型

这些是开箱即用的模型，其中大多数默认使用 `Q4_K_M` 量化。它们可以在 ggml-org 的 Hugging Face 页面找到：https://huggingface.co/collections/ggml-org/multimodal-ggufs-68244e01ff1f39e5bebeeedc

将 `(tool_name)` 替换为你想要使用的二进制文件名。例如，`llama-mtmd-cli` 或 `llama-server`

注意：某些模型可能需要较大的上下文窗口，例如：`-c 8192`

**视觉模型**：

```sh
# Gemma 3
(tool_name) -hf ggml-org/gemma-3-4b-it-GGUF
(tool_name) -hf ggml-org/gemma-3-12b-it-GGUF
(tool_name) -hf ggml-org/gemma-3-27b-it-GGUF

# SmolVLM
(tool_name) -hf ggml-org/SmolVLM-Instruct-GGUF
(tool_name) -hf ggml-org/SmolVLM-256M-Instruct-GGUF
(tool_name) -hf ggml-org/SmolVLM-500M-Instruct-GGUF
(tool_name) -hf ggml-org/SmolVLM2-2.2B-Instruct-GGUF
(tool_name) -hf ggml-org/SmolVLM2-256M-Video-Instruct-GGUF
(tool_name) -hf ggml-org/SmolVLM2-500M-Video-Instruct-GGUF

# Pixtral 12B
(tool_name) -hf ggml-org/pixtral-12b-GGUF

# Qwen 2 VL
(tool_name) -hf ggml-org/Qwen2-VL-2B-Instruct-GGUF
(tool_name) -hf ggml-org/Qwen2-VL-7B-Instruct-GGUF

# Qwen 2.5 VL
(tool_name) -hf ggml-org/Qwen2.5-VL-3B-Instruct-GGUF
(tool_name) -hf ggml-org/Qwen2.5-VL-7B-Instruct-GGUF
(tool_name) -hf ggml-org/Qwen2.5-VL-32B-Instruct-GGUF
(tool_name) -hf ggml-org/Qwen2.5-VL-72B-Instruct-GGUF

# Mistral Small 3.1 24B (IQ2_M quantization)
(tool_name) -hf ggml-org/Mistral-Small-3.1-24B-Instruct-2503-GGUF

# InternVL 2.5 and 3
(tool_name) -hf ggml-org/InternVL2_5-1B-GGUF
(tool_name) -hf ggml-org/InternVL2_5-4B-GGUF
(tool_name) -hf ggml-org/InternVL3-1B-Instruct-GGUF
(tool_name) -hf ggml-org/InternVL3-2B-Instruct-GGUF
(tool_name) -hf ggml-org/InternVL3-8B-Instruct-GGUF
(tool_name) -hf ggml-org/InternVL3-14B-Instruct-GGUF

# Llama 4 Scout
(tool_name) -hf ggml-org/Llama-4-Scout-17B-16E-Instruct-GGUF

# Moondream2 20250414 version
(tool_name) -hf ggml-org/moondream2-20250414-GGUF

# Gemma 4
(tool_name) -hf ggml-org/gemma-4-E2B-it-GGUF
(tool_name) -hf ggml-org/gemma-4-E4B-it-GGUF
(tool_name) -hf ggml-org/gemma-4-26B-A4B-it-GGUF
(tool_name) -hf ggml-org/gemma-4-31B-it-GGUF
```

**音频模型**：

```sh
# Ultravox 0.5
(tool_name) -hf ggml-org/ultravox-v0_5-llama-3_2-1b-GGUF
(tool_name) -hf ggml-org/ultravox-v0_5-llama-3_1-8b-GGUF

# Qwen2-Audio and SeaLLM-Audio
# 注意：这些模型没有预量化的 GGUF，因为效果很差
# 参考：https://github.com/ggml-org/llama.cpp/pull/13760

# Mistral's Voxtral
(tool_name) -hf ggml-org/Voxtral-Mini-3B-2507-GGUF

# Qwen3-ASR
(tool_name) -hf ggml-org/Qwen3-ASR-0.6B-GGUF
(tool_name) -hf ggml-org/Qwen3-ASR-1.7B-GGUF
```

**混合模态**：

```sh
# Qwen2.5 Omni
# 能力：音频输入、图像输入
(tool_name) -hf ggml-org/Qwen2.5-Omni-3B-GGUF
(tool_name) -hf ggml-org/Qwen2.5-Omni-7B-GGUF

# Qwen3 Omni
# 能力：音频输入、图像输入
(tool_name) -hf ggml-org/Qwen3-Omni-30B-A3B-Instruct-GGUF
(tool_name) -hf ggml-org/Qwen3-Omni-30B-A3B-Thinking-GGUF

# Gemma 4
# 能力：音频输入、图像输入
(tool_name) -hf ggml-org/gemma-4-E2B-it-GGUF
(tool_name) -hf ggml-org/gemma-4-E4B-it-GGUF
```

## 查找更多模型：

可以在以下地址找到 Hugging Face 上具备视觉能力的 GGUF 模型：https://huggingface.co/models?pipeline_tag=image-text-to-text&sort=trending&search=gguf
