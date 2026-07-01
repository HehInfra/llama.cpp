# Hexagon 后端开发者详情

## 后端库

Hexagon 后端由两部分组成：

  - `libggml-hexagon`
    这是常规的 CPU 端 GGML 后端库，可以是动态库或静态链接。

  - `libggml-htp-vNN`
    这是 NPU 端（HTP 即 Hexagon Tensor Processor）的动态库，包含算子调度器与内核。
    运行时会根据硬件版本自动选择正确的库。

以下是构建产物的示例：

```
~/src/llama.cpp$ ls -l pkg-adb/llama.cpp/lib/libggml*
pkg-adb/llama.cpp/lib/libggml-base.so
pkg-adb/llama.cpp/lib/libggml-cpu.so
pkg-adb/llama.cpp/lib/libggml-hexagon.so      <<< CPU library
pkg-adb/llama.cpp/lib/libggml-htp-v73.so      <<< HTP op/kernels for Hexagon v73
pkg-adb/llama.cpp/lib/libggml-htp-v75.so
pkg-adb/llama.cpp/lib/libggml-htp-v79.so
pkg-adb/llama.cpp/lib/libggml-htp-v81.so
```

## 内存缓冲区

Hexagon NPU 后端利用 Snapdragon 的统一内存模型，所有缓冲区均可被 CPU 与 GPU 完全访问。
NPU 拥有一片专用的紧耦合内存 VTCM，但它仅用于中间数据（例如动态量化张量）或临时数据（通过 DMA 获取的权重张量分块）。

请注意，当前 Hexagon 后端没有实现 SET/GET_ROWS 算子，因为目前将这些算子卸载到 NPU 上并没有优势。

后端会为需要重新打包的数据类型（Q4_0、Q8_0、MXFP4）的张量分配 non-host 缓冲区。
从 MMU 角度看，这些缓冲区仍然是普通缓冲区（CPU 可正常访问），之所以标记为 non-host，只是为了强制进行重新打包。

## 大模型处理

Hexagon NPU 会话（在 Hexagon 文档中也称为 Process Domain，简称 PD）的内存映射限制约为 3.5GB。
在 llama.cpp/GGML 中，一个 Hexagon 会话对应单个 GGML 后端设备（HTP0、HTP1 等）。

为了运行超过 3.5GB 的模型，我们需要分配多个设备并对模型进行切分。
为此，我们利用了 llama.cpp/GGML 的多 GPU 层切分支持。
每个 Hexagon 设备在卸载与模型切分方面都表现得像一块 GPU。

以下是在配备 16GB DDR 的较新 Snapdragon 设备上运行 GPT-OSS-20B 模型的示例：

```
M=gpt-oss-20b-Q4_0.gguf NDEV=4 D=HTP0,HTP1,HTP2,HTP3 P=surfing.txt scripts/snapdragon/adb/run-completion.sh -f surfing.txt -n 32
...
LD_LIBRARY_PATH=/data/local/tmp/llama.cpp/lib
ADSP_LIBRARY_PATH=/data/local/tmp/llama.cpp/lib
GGML_HEXAGON_NDEV=4 ./bin/llama-cli --no-mmap -m /data/local/tmp/llama.cpp/../gguf/gpt-oss-20b-Q4_0.gguf
      -t 4 --ctx-size 8192 --batch-size 128 -ctk q8_0 -ctv q8_0 -fa on -ngl 99 --device HTP0,HTP1,HTP2,HTP3 -no-cnv -f surfing.txt
...
llama_model_loader: - type  f32:  289 tensors
llama_model_loader: - type q4_0:   96 tensors
llama_model_loader: - type q8_0:    2 tensors
llama_model_loader: - type mxfp4:  72 tensors
...
load_tensors: offloaded 25/25 layers to GPU
load_tensors:          CPU model buffer size =  1182.09 MiB
load_tensors:         HTP1 model buffer size =     6.64 MiB
load_tensors:  HTP1-REPACK model buffer size =  2505.94 MiB
load_tensors:         HTP3 model buffer size =     5.55 MiB
load_tensors:  HTP3-REPACK model buffer size =  2088.28 MiB
load_tensors:         HTP0 model buffer size =     7.75 MiB
load_tensors:  HTP0-REPACK model buffer size =  2923.59 MiB
load_tensors:         HTP2 model buffer size =     6.64 MiB
load_tensors:  HTP2-REPACK model buffer size =  2505.94 MiB
...
llama_context: n_ctx_per_seq (8192) < n_ctx_train (131072) -- the full capacity of the model will not be utilized
llama_context:        CPU  output buffer size =     0.77 MiB
llama_kv_cache_iswa: creating non-SWA KV cache, size = 8192 cells
llama_kv_cache:       HTP1 KV buffer size =    25.50 MiB
llama_kv_cache:       HTP3 KV buffer size =    25.50 MiB
llama_kv_cache:       HTP0 KV buffer size =    25.50 MiB
llama_kv_cache:       HTP2 KV buffer size =    25.50 MiB
llama_kv_cache: size =  102.00 MiB (  8192 cells,  12 layers,  1/1 seqs), K (q8_0):   51.00 MiB, V (q8_0):   51.00 MiB
llama_kv_cache_iswa: creating     SWA KV cache, size = 256 cells
llama_kv_cache:       HTP1 KV buffer size =     0.80 MiB
llama_kv_cache:       HTP3 KV buffer size =     0.53 MiB
llama_kv_cache:       HTP0 KV buffer size =     1.06 MiB
llama_kv_cache:       HTP2 KV buffer size =     0.80 MiB
llama_kv_cache: size =    3.19 MiB (   256 cells,  12 layers,  1/1 seqs), K (q8_0):    1.59 MiB, V (q8_0):    1.59 MiB
llama_context:       HTP0 compute buffer size =    16.06 MiB
llama_context:       HTP1 compute buffer size =    16.06 MiB
llama_context:       HTP2 compute buffer size =    16.06 MiB
llama_context:       HTP3 compute buffer size =    16.06 MiB
llama_context:        CPU compute buffer size =    98.19 MiB
...
llama_perf_context_print: prompt eval time =    3843.67 ms /   197 tokens ( 19.51 ms per token, 51.25 tokens per second)
llama_perf_context_print:        eval time =    1686.13 ms /    31 runs   ( 54.39 ms per token, 18.39 tokens per second)
llama_perf_context_print:       total time =    6266.30 ms /   228 tokens
llama_perf_context_print:    graphs reused =         30
llama_memory_breakdown_print: | memory breakdown [MiB] | total   free    self   model   context   compute    unaccounted |
llama_memory_breakdown_print: |   - HTP0 (Hexagon)     |  2048 = 2048 + (   0 =     0 +       0 +       0) +           0 |
llama_memory_breakdown_print: |   - HTP1 (Hexagon)     |  2048 = 2048 + (   0 =     0 +       0 +       0) +           0 |
llama_memory_breakdown_print: |   - HTP2 (Hexagon)     |  2048 = 2048 + (   0 =     0 +       0 +       0) +           0 |
llama_memory_breakdown_print: |   - HTP3 (Hexagon)     |  2048 = 2048 + (   0 =     0 +       0 +       0) +           0 |
llama_memory_breakdown_print: |   - Host               |                 1476 =  1208 +     105 +     162                |
llama_memory_breakdown_print: |   - HTP1-REPACK        |                 2505 =  2505 +       0 +       0                |
llama_memory_breakdown_print: |   - HTP3-REPACK        |                 2088 =  2088 +       0 +       0                |
llama_memory_breakdown_print: |   - HTP0-REPACK        |                 2923 =  2923 +       0 +       0                |
llama_memory_breakdown_print: |   - HTP2-REPACK        |                 2505 =  2505 +       0 +       0                |
```
