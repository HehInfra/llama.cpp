# 为 `llama.cpp` 添加新模型架构

添加一个模型需要以下几个步骤：

1. 将模型转换为 GGUF
2. 在 `llama.cpp` 中定义模型架构
3. 构建 GGML 图实现
4. 可选：添加多模态编码器实现

完成这些步骤后，你就可以提交 PR。

此外，很重要的一点是检查示例程序和主要 GGML 后端（CUDA、METAL、CPU）是否能在新架构上正常工作，特别是：
- [cli](/tools/cli/)
- [completion](/tools/completion/)
- [imatrix](/tools/imatrix/)
- [quantize](/tools/quantize/)
- [server](/tools/server/)

### 1. 将模型转换为 GGUF

这一步通过 Python 使用 `convert` 脚本和 [gguf](https://pypi.org/project/gguf/) 库完成。
根据模型架构的不同，你可以使用 [convert_hf_to_gguf.py](/convert_hf_to_gguf.py) 或 [examples/convert_legacy_llama.py](/examples/convert_legacy_llama.py)（用于 `.pth` 格式的 `llama/llama2` 模型）。

convert 脚本会读取模型配置、分词器、张量名称+数据，并将它们转换为 GGUF 元数据和张量。

对于 HuggingFace 模型，需要实现的步骤如下：

1. 在 [conversion](/conversion) 文件夹中新建一个 `TextModel` 或 `MmprojModel` 子类，并添加 `ModelBase.register` 注解，例如：

```python
@ModelBase.register("MyModelForCausalLM")
class MyModel(TextModel):
    model_arch = gguf.MODEL_ARCH.MYMODEL
```

或

```python
@ModelBase.register("MyModelForConditionalGeneration")
class MyModel(MmprojModel):
    model_arch = gguf.MODEL_ARCH.MYMODEL
```

2. 在 [constants.py](/gguf-py/gguf/constants.py) 中定义 GGUF 张量的布局

在 `MODEL_ARCH` 中添加枚举项，在 `MODEL_ARCH_NAMES` 中添加模型的人类可读名称，在 `MODEL_TENSORS` 中添加 GGUF 张量名称。

以 `falcon` 模型为例：
```python
    MODEL_ARCH.FALCON: [
        MODEL_TENSOR.TOKEN_EMBD,
        MODEL_TENSOR.OUTPUT_NORM,
        MODEL_TENSOR.OUTPUT,
        MODEL_TENSOR.ATTN_NORM,
        MODEL_TENSOR.ATTN_NORM_2,
        MODEL_TENSOR.ATTN_QKV,
        MODEL_TENSOR.ATTN_OUT,
        MODEL_TENSOR.FFN_DOWN,
        MODEL_TENSOR.FFN_UP,
    ]
```

3. 将原始张量名称映射为 GGUF 中的标准化等价名称

一般来说，在向 GGUF 添加新张量名称之前，请确保不存在等价的命名。

找到等价的 GGUF 张量名称后，将其添加到 [tensor_mapping.py](/gguf-py/gguf/tensor_mapping.py) 文件中。

如果张量名称属于重复的层/块，关键字 `bid` 会替代它。

以注意力层中的归一化张量为例：

```python
block_mappings_cfg: dict[MODEL_TENSOR, tuple[str, ...]] = {
        # Attention norm
        MODEL_TENSOR.ATTN_NORM: (
            "gpt_neox.layers.{bid}.input_layernorm",                # gptneox
            "transformer.h.{bid}.ln_1",                             # gpt2 gpt-j refact qwen
            "transformer.blocks.{bid}.norm_1",                      # mpt
            ...
        )
}
```

`transformer.blocks.{bid}.norm_1` 将被映射为 GGUF 中的 `blk.{bid}.attn_norm`。

根据模型配置、分词器、代码和张量布局的不同，你可能需要重写：
- `TextModel#set_gguf_parameters`
- `MmprojModel#set_gguf_parameters`
- `ModelBase#set_vocab`
- `ModelBase#modify_tensors`

注意：张量名称必须以 `.weight` 或 `.bias` 后缀结尾，这是约定，且 `quantize` 等工具依赖此后缀来处理权重。

### 2. 在 `llama.cpp` 中定义模型架构

模型参数和张量布局必须在 `llama.cpp` 源文件中定义：
1. 在 `src/llama-arch.h` 中定义一个新的 `llm_arch` 枚举值。
2. 在 `src/llama-arch.cpp` 中：
    - 将架构名称添加到 `LLM_ARCH_NAMES` 映射中。
    - 你可能还需要更新 `LLM_KV_NAMES`、`LLM_TENSOR_NAMES` 和 `LLM_TENSOR_INFOS`
3. 在 `src/llama-model-loader.cpp` 的 `llama_model_loader` 构造函数中添加任何非标准元数据加载。
4. 如果模型包含 RoPE 操作，请在 `src/llama-model.cpp` 的 `llama_model_rope_type` 函数中为该架构添加 case。

注意：`ggml` 中的维度顺序通常与 `pytorch` 中的维度顺序相反。

### 3. 构建 GGML 图实现

这是最有趣的部分，你需要在 `src/llama-model.cpp` 中提供新模型架构的推理图实现：
1. 创建一个继承自 `llama_model_base` 的新结构体。
2. 在其 `build_arch_graph` 方法中实现图构建逻辑。
3. `build_arch_graph` 方法应返回一个构建好的图（继承自 `llm_graph_context`）。可以参考现有实现，如 `llama_model_llama`、`llama_model_dbrx` 或 `llama_model_bert`。
4. 然后，在 `llama_model_mapping` 函数中为你的架构添加一个 case，以实例化你的新图构建结构体。

某些 `ggml` 后端不支持所有操作。后端实现可以在单独的 PR 中添加。

注意：如需调试推理图，可以使用 [llama-eval-callback](/examples/eval-callback/)。

### 4. 可选：添加多模态编码器实现

如果新模型支持多模态输入，你需要在 `libmtmd` 中添加一个新的编码器定义。你可以在 [多模态文档](../multimodal.md) 和 `tools/mtmd` 源代码目录中找到关于 llama.cpp 多模态支持的更多信息。

1. 在 convert 脚本中，确保添加一个继承自 `MmprojModel` 或同一基类的子类。
2. 在 `clip.cpp` 中添加编码器定义。
3. 在 `mtmd.cpp` 中实现预处理器。在大多数情况下，你可以复用现有的预处理器。
4. 实现编码器的 GGML 图，如果模型与现有模型差异很大，可以放在单独的文件中；否则可以复用现有实现（例如 siglip、pixtral 或 qwen），并添加模型特定的投影器。

注意：
- 许多多模态编码器基于已支持的模型。在添加新模型之前，请务必阅读 `tools/mtmd/models` 中现有的编码器定义。在 `libmtmd` 中，通常最好扩展现有模型，而不是复制代码。
- 要调试多模态预处理器和编码器，可以使用 [llama-mtmd-debug](tools/mtmd/debug/mtmd-debug.cpp)。
- 在 `libmtmd` 中添加模型特定的 API 或 CLI 是一种反模式。`libmtmd` 的目标是为多模态流程提供一个易用、模型无关的库。
- 在大多数情况下，不应修改 `llama-mtmd-cli`。如果模型需要特定的提示词，可以让用户提供，或将其嵌入到 Jinja 聊天模板中。

## 技巧与提示

### 使用 ggml_rope_ext

PyTorch 实现通常倾向于显式计算 `freq_cis`/`sin`/`cos` 分量。然而，在 llama.cpp 中，大多数 RoPE 操作可以通过 `ggml_rope_ext` 处理，它不需要 sin/cos 矩阵。这样可以节省内存，同时允许 GGML RoPE 核函数与其他算子融合。

然而，由于 `ggml_rope_ext` 只提供模型所使用的一部分 RoPE 实现，将模型从 PyTorch 转换到 llama.cpp 时可能需要进行一些创造性的调整。

有关 `ggml_rope_ext` 的更多信息，请参阅 `ggml.h` 中的代码内文档。

示例：
- `libmtmd` 通过将输入张量分成两半，对每一半分别应用 `ggml_rope_ext`，然后使用 `ggml_concat` 重新拼接，从而用 `GGML_ROPE_TYPE_NORMAL` 排序实现 2D RoPE。
- [Kimi-K2.5](https://github.com/ggml-org/llama.cpp/pull/19170) 视觉编码器使用交错频率的视觉 RoPE。权重必须在转换过程中进行置换，以便复用 `build_rope_2d()` 函数。
- [Gemma 4](https://github.com/ggml-org/llama.cpp/pull/21309) 使用“proportional”RoPE。我们采用一种技巧，将最后维度的 `rope_freqs` 设置为一个非常大的值，以防止这些维度被旋转。详情请参见 `convert_hf_to_gguf.py` 中的 `Gemma4Model` 类。
- 某些模型需要对输入位置进行缩放。例如，`[0, 1, 2, ...]` 变为 `[0, 0.5, 1, ...]`。在这种情况下，你可以通过 `freq_scale = 0.5f` 提供缩放。
- 某些模型使用学习得到的 RoPE 频率，而不是依赖 `powf(freq_base, -2.0 * i / n_dims)`。在这种情况下，你可以通过 `rope_freqs` 张量提供学习得到的频率（对应 `ggml_rope_ext` 中的参数 `c`），然后将 `freq_base = 1.0f`。一个重要注意事项是，GGML 中的 `rope_freqs` 是**倒数**（`theta = pos[i] / rope_freqs`），因此你可能需要在转换过程中对 `rope_freqs` 取倒数。

## GGUF 规范

https://github.com/ggml-org/ggml/blob/master/docs/gguf.md

## 资源

- YaRN RoPE scaling https://github.com/ggml-org/llama.cpp/pull/2268
- support Baichuan serial models https://github.com/ggml-org/llama.cpp/pull/3009
- support attention bias https://github.com/ggml-org/llama.cpp/pull/4283
- Mixtral support https://github.com/ggml-org/llama.cpp/pull/4406
- BERT embeddings https://github.com/ggml-org/llama.cpp/pull/5423
- Grok-1 support https://github.com/ggml-org/llama.cpp/pull/6204
- Command R Plus support https://github.com/ggml-org/llama.cpp/pull/6491
- support arch DBRX https://github.com/ggml-org/llama.cpp/pull/6515
- How to convert HuggingFace model to GGUF format https://github.com/ggml-org/llama.cpp/discussions/2948
