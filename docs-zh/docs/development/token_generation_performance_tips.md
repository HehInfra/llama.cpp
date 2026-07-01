# Token 生成性能故障排查

## 确认模型已使用 CUDA 在 GPU 上运行
请确保按照[本指南](/docs/build.md#cuda)使用正确的环境变量编译 llama，这样 llama 才能接受 `-ngl N`（或 `--n-gpu-layers N`）参数。运行 llama 时，你可以将 `N` 设置为一个非常大的值，llama 会自动将尽可能多的层卸载到 GPU，即使实际数量小于你设置的值。例如：
```shell
./llama-cli -m "path/to/model.gguf" -ngl 200000 -p "Please sir, may I have some "
```

运行 llama 时，在开始推理工作之前，它会输出诊断信息，显示 cuBLAS 是否将工作卸载到 GPU。请留意以下行：
```shell
llama_model_load_internal: [cublas] offloading 60 layers to GPU
llama_model_load_internal: [cublas] offloading output layer to GPU
llama_model_load_internal: [cublas] total VRAM used: 17223 MB
... rest of inference
```

如果你看到这些行，说明 GPU 正在被使用。

## 确认 CPU 没有过载
llama 接受 `-t N`（或 `--threads N`）参数。该参数非常重要，不能设置得过大。如果你的 token 生成速度极慢，请尝试将其设置为 1。如果这显著提升了生成速度，说明你的 CPU 已经过载，需要显式将该参数设置为你机器上物理 CPU 核心数（即使你使用了 GPU）。如果不确定，可以从 1 开始，逐步翻倍，直到遇到性能瓶颈，再适当降低。

# 运行时参数对推理速度影响的示例
以下测试运行在以下机器上：
GPU: A6000 (48GB VRAM)
CPU: 7 个物理核心
RAM: 32GB

模型: `TheBloke_Wizard-Vicuna-30B-Uncensored-GGML/Wizard-Vicuna-30B-Uncensored.q4_0.gguf`（300 亿参数，4bit 量化，GGML）

运行命令: `./llama-cli -m "path/to/model.gguf" -p "An extremely detailed description of the 10 best ethnic dishes will follow, with recipes: " -n 1000 [additional benchmark flags]`

结果：

| command | tokens/second (越高越好) |
| - | - |
| -ngl 2000000 | N/A (小于 0.1) |
| -t 7 | 1.7 |
| -t 1 -ngl 2000000 | 5.5 |
| -t 7 -ngl 2000000 | 8.7 |
| -t 4 -ngl 2000000 | 9.1 |
