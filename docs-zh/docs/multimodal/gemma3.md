# Gemma 3 视觉能力

> [!IMPORTANT]
>
> 此功能非常实验性，仅供演示使用。

## 快速开始

你可以使用 [ggml-org](https://huggingface.co/ggml-org) Hugging Face 账号中的预量化模型。

```bash
# build
cmake -B build
cmake --build build --target llama-mtmd-cli

# alternatively, install from brew (MacOS)
brew install llama.cpp

# run it
llama-mtmd-cli -hf ggml-org/gemma-3-4b-it-GGUF
llama-mtmd-cli -hf ggml-org/gemma-3-12b-it-GGUF
llama-mtmd-cli -hf ggml-org/gemma-3-27b-it-GGUF

# note: 1B model does not support vision
```

## 如何获取 mmproj.gguf？

只需在使用 `convert_hf_to_gguf.py` 转换模型时添加 `--mmproj`：

```bash
cd gemma-3-4b-it
python ../llama.cpp/convert_hf_to_gguf.py --outfile model.gguf --outtype f16 --mmproj .
# output file: mmproj-model.gguf
```

## 如何运行？

你需要准备：
- 文本模型 GGUF，可使用 `convert_hf_to_gguf.py` 进行转换
- 上一步生成的 mmproj 文件
- 一张图片文件

```bash
# build
cmake -B build
cmake --build build --target llama-mtmd-cli

# run it
./build/bin/llama-mtmd-cli -m {text_model}.gguf --mmproj mmproj.gguf --image your_image.jpg
```
