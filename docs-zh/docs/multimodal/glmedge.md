# GLMV-EDGE

当前实现支持 [glm-edge-v-2b](https://huggingface.co/THUDM/glm-edge-v-2b) 和 [glm-edge-v-5b](https://huggingface.co/THUDM/glm-edge-v-5b)。

## 用法

构建 `llama-mtmd-cli` 二进制文件。

构建完成后，运行：`./llama-mtmd-cli` 查看用法。例如：

```sh
./llama-mtmd-cli -m model_path/ggml-model-f16.gguf --mmproj model_path/mmproj-model-f16.gguf
```

**注意**：为了获得更好的效果，建议使用较低的温度值，例如 0.1。可在命令中添加 `--temp 0.1`。
**注意**：要进行 GPU 卸载，请像往常一样使用 `-ngl` 标志

## GGUF 转换

1. 克隆 GLMV-EDGE 模型（[2B](https://huggingface.co/THUDM/glm-edge-v-2b) 或 [5B](https://huggingface.co/THUDM/glm-edge-v-5b)）。例如：

```sh
git clone https://huggingface.co/THUDM/glm-edge-v-5b or https://huggingface.co/THUDM/glm-edge-v-2b
```

2. 使用 `glmedge-surgery.py` 将 GLMV-EDGE 模型拆分为 LLM 和多模态投影器组件：

```sh
python ./tools/mtmd/glmedge-surgery.py -m ../model_path
```

4. 使用 `glmedge-convert-image-encoder-to-gguf.py` 将 GLMV-EDGE 图像编码器转换为 GGUF：

```sh
python ./tools/mtmd/glmedge-convert-image-encoder-to-gguf.py -m ../model_path --llava-projector ../model_path/glm.projector --output-dir ../model_path
```

5. 使用 `examples/convert_hf_to_gguf.py` 将 GLMV-EDGE 的 LLM 部分转换为 GGUF：

```sh
python convert_hf_to_gguf.py ../model_path
```

现在 LLM 部分和图像编码器都在 `model_path` 目录中。
