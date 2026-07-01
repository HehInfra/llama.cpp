# 基于 Snapdragon 的设备

## 设置

### Android

为基于 Snapdragon 的 Android 设备构建 llama.cpp 最简单的方法是使用工具链 Docker 镜像（参见 github.com/snapdragon-toolchain）。
该镜像包含 Android NDK、OpenCL SDK、Hexagon SDK、CMake 等。

此方法适用于 Linux、macOS 和 Windows。macOS 和 Windows 用户应安装 Docker Desktop。

```
~/src/llama.cpp$ docker run -it -u $(id -u):$(id -g) --volume $(pwd):/workspace --platform linux/amd64 ghcr.io/snapdragon-toolchain/arm64-android:v0.7
[d]/> cd /workspace
```

注意：**Android** 构建过程的后续步骤假定你在工具链容器内运行。

### Windows On Snapdragon

原生 Windows 11 arm64 构建需要以下工具依赖：
- MS Visual Studio 2026（社区版或专业版）
  - MSVC arm64 标准库和运行时库
  - UCRT 和驱动程序工具包
- LLVM 核心库和 Clang 编译器（winget）
- CMake、Git、Python（winget）
- Hexagon SDK Community Edition 6.6 或更高版本（参见 windows.md）
- OpenCL SDK 2.3 或更高版本（参见 windows.md）

注意：**Windows** 构建过程的后续步骤假定你在 Powershell 中原生运行。
请根据情况调整下方的构建命令。

## 如何构建

下面通过 CMake 预设构建包含 CPU、OpenCL 和 Hexagon 后端的 llama.cpp：

```
[d]/workspace> cp docs/backend/snapdragon/CMakeUserPresets.json .

[d]/workspace> cmake --preset arm64-android-snapdragon-release -B build-snapdragon
Preset CMake variables:
  ANDROID_ABI="arm64-v8a"
  ...
  CMAKE_TOOLCHAIN_FILE="/opt/android-ndk-r28b/build/cmake/android.toolchain.cmake"
  GGML_HEXAGON="ON"
  GGML_OPENCL="ON"
  GGML_OPENMP="OFF"
  HEXAGON_SDK_ROOT="/opt/hexagon/6.6.0.0"
...
-- Including OpenCL backend
-- Including Hexagon backend
...
-- Build files have been written to: /workspace/build-snapdragon

[d]/workspace> cmake --build build-snapdragon
...
[144/356] Performing build step for 'htp-v73'
[1/16] Generating htp_iface_skel.c, htp_iface_stub.c, htp_iface.h
[2/16] Building C object CMakeFiles/ggml-htp-v73.dir/hvx-sigmoid.c.obj
[3/16] Building C object CMakeFiles/ggml-htp-v73.dir/htp-dma.c.obj
[4/16] Building C object CMakeFiles/ggml-htp-v73.dir/worker-pool.c.obj
...
-- Installing: /workspace/build-snapdragon/ggml/src/ggml-hexagon/libggml-htp-v73.so
-- Installing: /workspace/build-snapdragon/ggml/src/ggml-hexagon/libggml-htp-v75.so
...
```

要生成可安装的“包”，只需使用 cmake --install：

```
[d]/workspace> cmake --install build-snapdragon --prefix pkg-snapdragon/llama.cpp
-- Install configuration: "Release"
-- Installing: /workspace/pkg-snapdragon/llama.cpp/lib/libggml-cpu.so
-- Installing: /workspace/pkg-snapdragon/llama.cpp/lib/libggml-opencl.so
-- Installing: /workspace/pkg-snapdragon/llama.cpp/lib/libggml-hexagon.so
-- Installing: /workspace/pkg-snapdragon/llama.cpp/lib/libggml-htp-v73.so
-- Installing: /workspace/pkg-snapdragon/llama.cpp/lib/libggml-htp-v75.so
-- Installing: /workspace/pkg-snapdragon/llama.cpp/lib/libggml-htp-v79.so
-- Installing: /workspace/pkg-snapdragon/llama.cpp/lib/libggml-htp-v81.so
-- Installing: /workspace/pkg-snapdragon/llama.cpp/lib/libggml.so
...
-- Installing: /workspace/pkg-snapdragon/llama.cpp/bin/llama-bench
-- Installing: /workspace/pkg-snapdragon/llama.cpp/bin/llama-cli
...
```

## 如何安装

### Android

此步骤需要先将设备配置为可用于设备端开发。
详情请参见 https://developer.android.com/studio/debug/dev-options。

启用 ADB 后，使用 `adb push` 将 `pkg-snapdragon` 安装到设备上。
**注意，工具链 Docker 镜像中没有 ADB，也不会设置 ADB 桥接。请使用宿主机上的原生 ADB。**

```
~/src/llama.cpp$ adb push pkg-snapdragon/llama.cpp /data/local/tmp/
pkg-snapdragon/llama.cpp/bin/: 67 files pushed, 0 skipped. 190.2 MB/s (919095042 bytes in 4.607s)
pkg-snapdragon/llama.cpp/include/: 19 files pushed, 0 skipped. 20.5 MB/s (255173 bytes in 0.012s)
pkg-snapdragon/llama.cpp/lib/: 16 files pushed, 0 skipped. 144.4 MB/s (43801382 bytes in 0.289s)
102 files pushed, 0 skipped. 186.9 MB/s (963151597 bytes in 4.914s)
```

此时，你还需要安装一些模型：

```
~/src/llama.cpp$ wget https://huggingface.co/bartowski/Llama-3.2-1B-Instruct-GGUF/resolve/main/Llama-3.2-1B-Instruct-Q4_0.gguf
...
2025-10-11 12:04:52 (10.7 MB/s) - ‘Llama-3.2-1B-Instruct-Q4_0.gguf’ saved [773025920/773025920]

~/src/llama.cpp$ adb push Llama-3.2-1B-Instruct-Q4_0.gguf /data/local/tmp/gguf
Llama-3.2-1B-Instruct-Q4_0.gguf: 1 file pushed, 0 skipped. 38.3 MB/s (773025920 bytes in 19.250s)
```

### Windows

所有构建产物已经安装在 `pkg-snapdragon` 文件夹中。
要运行，请根据下方说明，使用 `scripts/snapdragon/windows` 中的 Powershell 脚本。

## 如何运行

运行 llama.cpp cli 工具最简单的方法是使用提供的包装脚本，它们会正确设置所有必需的环境变量。

llama.cpp 在基于 Snapdragon 的设备上支持三种后端：CPU、Adreno GPU（GPUOpenCL）和 Hexagon NPU（HTP0-4）。
你可以使用 `D=` 变量来选择模型运行在哪个后端上，该变量对应 `--device` 选项。

对于 `-ngl` 和其他卸载相关选项，Hexagon NPU 的行为类似于“GPU”设备。

下面是一些通过 ADB 运行各种 llama.cpp 工具的示例。

Llama-3.2-1B 的简单问答

```
~/src/llama.cpp$ M=Llama-3.2-1B-Instruct-Q4_0.gguf D=HTP0 ./scripts/snapdragon/adb/run-completion.sh -p "what is the most popular cookie in the world?"
...
ggml-hex: Hexagon backend (experimental) : allocating new registry : ndev 1
ggml-hex: Hexagon Arch version v79
ggml-hex: allocating new session: HTP0
ggml-hex: new session: HTP0 : session-id 0 domain-id 3 uri file:///libggml-htp-v79.so?htp_iface_skel_handle_invoke&_modver=1.0&_dom=cdsp&_session=0 handle 0xb4000072c7955e50
...
load_tensors: offloading output layer to GPU
load_tensors: offloaded 17/17 layers to GPU
load_tensors:          CPU model buffer size =   225.49 MiB
load_tensors:         HTP0 model buffer size =     0.26 MiB
load_tensors:  HTP0-REPACK model buffer size =   504.00 MiB
...
I hope this helps you understand the world's most popular cookies! [end of text]
...
llama_perf_sampler_print:    sampling time =      30.08 ms /   487 runs   (    0.06 ms per token, 16191.77 tokens per second)
llama_perf_context_print:        load time =     617.94 ms
llama_perf_context_print: prompt eval time =      80.76 ms /    11 tokens (    7.34 ms per token,   136.21 tokens per second)
llama_perf_context_print:        eval time =    9210.59 ms /   475 runs   (   19.39 ms per token,    51.57 tokens per second)
llama_perf_context_print:       total time =    9454.92 ms /   486 tokens
llama_perf_context_print:    graphs reused =        473
llama_memory_breakdown_print: | memory breakdown [MiB] | total   free    self   model   context   compute    unaccounted |
llama_memory_breakdown_print: |   - HTP0 (Hexagon)     |  2048 = 2048 + (   0 =     0 +       0 +       0) +           0 |
llama_memory_breakdown_print: |   - Host               |                  439 =   225 +     136 +      77                |
llama_memory_breakdown_print: |   - HTP0-REPACK        |                  504 =   504 +       0 +       0                |
```

OLMoE-1B-7B 的摘要请求。这是一个大模型，需要两个 HTP 会话/设备

```
~/src/llama.cpp$ M=OLMoE-1B-7B-0125-Instruct-Q4_0.gguf NDEV=2 D=HTP0,HTP1 ./scripts/snapdragon/adb/run-completion.sh -f surfing.txt
...
ggml-hex: Hexagon backend (experimental) : allocating new registry : ndev 1
ggml-hex: Hexagon Arch version v81
ggml-hex: allocating new session: HTP0
ggml-hex: allocating new session: HTP1
...
load_tensors: offloading output layer to GPU
load_tensors: offloaded 17/17 layers to GPU
load_tensors:          CPU model buffer size =   143.86 MiB
load_tensors:         HTP1 model buffer size =     0.23 MiB
load_tensors:  HTP1-REPACK model buffer size =  1575.00 MiB
load_tensors:         HTP0 model buffer size =     0.28 MiB
load_tensors:  HTP0-REPACK model buffer size =  2025.00 MiB
...
llama_context:        CPU  output buffer size =     0.19 MiB
llama_kv_cache:       HTP1 KV buffer size =   238.00 MiB
llama_kv_cache:       HTP0 KV buffer size =   306.00 MiB
llama_kv_cache: size =  544.00 MiB (  8192 cells,  16 layers,  1/1 seqs), K (q8_0):  272.00 MiB, V (q8_0):  272.00 MiB
llama_context:       HTP0 compute buffer size =    15.00 MiB
llama_context:       HTP1 compute buffer size =    15.00 MiB
llama_context:        CPU compute buffer size =    24.56 MiB
...
llama_perf_context_print: prompt eval time =    1730.57 ms /   212 tokens (    8.16 ms per token,   122.50 tokens per second)
llama_perf_context_print:        eval time =    5624.75 ms /   257 runs   (   21.89 ms per token,    45.69 tokens per second)
llama_perf_context_print:       total time =    7377.33 ms /   469 tokens
llama_perf_context_print:    graphs reused =        255
llama_memory_breakdown_print: | memory breakdown [MiB] | total   free    self   model   context   compute    unaccounted |
llama_memory_breakdown_print: |   - HTP0 (Hexagon)     |  2048 = 2048 + (   0 =     0 +       0 +       0) +           0 |
llama_memory_breakdown_print: |   - HTP1 (Hexagon)     |  2048 = 2048 + (   0 =     0 +       0 +       0) +           0 |
llama_memory_breakdown_print: |   - Host               |                  742 =   144 +     544 +      54                |
llama_memory_breakdown_print: |   - HTP1-REPACK        |                 1575 =  1575 +       0 +       0                |
llama_memory_breakdown_print: |   - HTP0-REPACK        |                 2025 =  2025 +       0 +       0                |
```

MUL_MAT 的 Op 测试

```
~/src/llama.cpp$ HB=0 ./scripts/snapdragon/adb/run-tool.sh test-backend-ops -b HTP0 -o MUL_MAT
...
Backend 2/3: HTP0
Device description: Hexagon
Device memory: 2048 MB (2048 MB free)
MUL_MAT(type_a=q4_0,type_b=f32,m=16,n=1,k=256,bs=[1,1],nr=[1,1],per=[0,1,2,3],v=0,o=1): OK
MUL_MAT(type_a=q4_0,type_b=f32,m=16,n=2,k=256,bs=[1,1],nr=[1,1],per=[0,1,2,3],v=0,o=1): OK
MUL_MAT(type_a=q4_0,type_b=f32,m=16,n=3,k=256,bs=[1,1],nr=[1,1],per=[0,1,2,3],v=0,o=1): OK

~/src/llama.cpp-hexagon$ M=Llama-3.2-1B-Instruct-Q4_0.gguf ./scripts/snapdragon/adb/run-bench.sh -p 128 -n 64
...
ggml-hex: Hexagon backend (experimental) : allocating new registry : ndev 1
ggml-hex: Hexagon Arch version v79
ggml-hex: allocating new session: HTP0
ggml-hex: new session: HTP0 : session-id 0 domain-id 3 uri file:///libggml-htp-v79.so?htp_iface_skel_handle_invoke&_modver=1.0&_dom=cdsp&_session=0 handle 0xb400007d4b231090
| model          |       size | params | backend    | ngl | threads | n_batch | mmap |  test |           t/s |
| ---------------| ---------: | -----: | ---------- | --: | ------: | ------: | ---: | ----: | ------------: |
| llama 1B Q4_0  | 729.75 MiB | 1.24 B | HTP        |  99 |       4 |     128 |    0 | pp128 | 169.42 ± 1.75 |
| llama 1B Q4_0  | 729.75 MiB | 1.24 B | HTP        |  99 |       4 |     128 |    0 |  tg64 |  51.54 ± 1.13 |

build: 6a8cf8914 (6733)
```

## 环境变量

- `GGML_HEXAGON_NDEV=1`
  控制要分配的设备/会话数量。默认值为 1。
  大多数 4B 以下的量化模型可以放入单个会话；8B 模型需要两个，20B 模型需要四个。

- `GGML_HEXAGON_NHVX=0`
  控制要使用的 HVX 硬件线程数。默认值为全部（实际数量取决于硬件版本）。

- `GGML_HEXAGON_HOSTBUF=1`
  控制 Hexagon 后端是否分配主机缓冲区。默认情况下，除 REPACK 外的所有缓冲区都是主机缓冲区。
  测试需要 REPACK 缓冲区的 Op（MUL_MAT 和 MUL_MAT_ID）时需要使用此选项。

- `GGML_HEXAGON_VERBOSE=1`
  启用后端 Op 的详细日志记录。示例输出：

  ```
  ggml-hex: HTP0 graph-compute n_nodes 2
  ggml-hex: HTP0 matmul : blk.27.ffn_up.weight x ffn_norm-27 -> ffn_up-27 : 3072:8192 x 3072:1 -> 8192:1 : q4_0 x f32 -> f32 : HTP0 x HTP0 -> HTP0 : flags 0x1
  ggml-hex: HTP0 matmul : blk.27.ffn_gate.weight x ffn_norm-27 -> ffn_gate-27 : 3072:8192 x 3072:1 -> 8192:1 : q4_0 x f32 -> f32 : HTP0 x HTP0 -> HTP0 : flags 0x3
  ggml-hex: HTP0 graph-compute n_nodes 1
  ggml-hex: HTP0 matmul : blk.27.ffn_down.weight x ffn_gate_par-27 -> ffn_out-27 : 8192:3072 x 8192:1 -> 3072:1 : q4_0 x f32 -> f32 : HTP0 x HTP0 -> HTP0 : flags 0x0
  ggml-hex: HTP0 get-tensor result_output : data 0x7592487000 offset 0 size 513024
  ```

- `GGML_HEXAGON_PROFILE=1`
  启用 Op 性能分析：

  - `1` 基本分析，包含每个 Op 的 `usecs` 和 `cycles` 计数器
  - `2` 扩展分析，包含每个 Op 的 `usecs`、`cycles` 和默认 PMU 计数器数据
  - `0x1,...,0x8` 扩展分析，包含每个 Op 的 `usecs`、`cycles` 和自定义 PMU 计数器数据

  日志输出可以保存到文件中以便后处理，也可以直接通过管道输入到后处理工具以生成报告。
  示例：

      `GGML_HEXAGON_PROFILE=1 llama-completion ... |& ./scripts/snapdragon/ggml-hexagon-profile.py -`

- `GGML_HEXAGON_OPSTAGE=0x0`
  允许启用 Op 处理流水线的特定阶段：

  - `0x1` 启用 Op 队列（即，将 Op 排队到 NPU）
  - `0x2` 启用 Op 计算（MUL_MAT 等）

  示例：

      `GGML_HEXAGON_OPSTAGE=0x1 llama-completion ...` - Op 被排队到 NPU，但 dma 和计算被禁用
      `GGML_HEXAGON_OPSTAGE=0x3 llama-completion ...` - 完整排队并处理 Op（默认）

- `GGML_HEXAGON_OPFILTER=regex`
  允许过滤（禁用）匹配正则表达式模式的 Op：

  示例：

      `GGML_HEXAGON_OPFILTER="FLASH_ATTN_EXT" llama-completion ...` - 在 Hexagon 上禁用 Flash Attention（回退到 CPU 或 GPU）
      `GGML_HEXAGON_OPFILTER="ADD\|SUB" llama-completion ...` - 在 Hexagon 上禁用 ADD 和 SUB（回退到 CPU 或 GPU）
