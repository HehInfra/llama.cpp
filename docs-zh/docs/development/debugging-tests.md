# 调试测试技巧

## 如何仅运行、执行或调试某个特定测试，以缩短反馈循环？

`scripts` 文件夹中有一个名为 `debug-test.sh` 的脚本，其参数接受一个正则表达式（REGEX）和一个可选的测试编号。

例如，运行以下命令将输出一个交互式列表，您可以从中选择测试。其格式如下：

`debug-test.sh [OPTION]... <test_regex> <test_number>`

随后它会为您构建并在调试器中运行。

若只想执行某个测试并返回 PASS 或 FAIL 消息，请运行：

```bash
./scripts/debug-test.sh test-tokenizer
```

要在 GDB 中测试，请使用 `-g` 标志启用 gdb 测试模式。

```bash
./scripts/debug-test.sh -g test-tokenizer

# Once in the debugger, i.e. at the chevrons prompt, setting a breakpoint could be as follows:
>>> b main
```

为加快测试循环，如果您知道测试编号，可以像下面这样直接运行：

```bash
./scripts/debug-test.sh test 23
```

更多详情请使用 `debug-test.sh -h` 打印帮助信息。

&nbsp;

### 脚本是如何工作的？

如果您希望能够单独使用脚本中包含的概念，下面简要列出了其中的重要部分。

#### 步骤 1：重置并设置文件夹上下文

从本仓库根目录开始，我们创建 `build-ci-debug` 作为构建上下文。

```bash
rm -rf build-ci-debug && mkdir build-ci-debug && cd build-ci-debug
```

#### 步骤 2：设置构建环境并编译测试二进制文件

在调试模式下配置并触发构建。您可以根据需要调整参数，但此处是一些合理的默认值。

```bash
cmake -DCMAKE_BUILD_TYPE=Debug -DLLAMA_CUDA=1 -DLLAMA_FATAL_WARNINGS=ON ..
make -j
```

#### 步骤 3：查找所有匹配 REGEX 的可用测试

该命令的输出会提供运行 GDB 所需的命令和参数。

* `-R test-tokenizer`：查找所有名为 `test-tokenizer*` 的测试文件（R=Regex）
* `-N`："show-only" 禁用测试执行，并显示可传递给 GDB 的测试命令。
* `-V`：详细模式

```bash
ctest -R "test-tokenizer" -V -N
```

其返回的输出可能类似于以下内容（重点关注需要注意的关键行）：

```bash
...
1: Test command: ~/llama.cpp/build-ci-debug/bin/test-tokenizer-0 "~/llama.cpp/tests/../models/ggml-vocab-llama-spm.gguf"
1: Working Directory: .
Labels: main
  Test  #1: test-tokenizer-0-llama-spm
...
4: Test command: ~/llama.cpp/build-ci-debug/bin/test-tokenizer-0 "~/llama.cpp/tests/../models/ggml-vocab-falcon.gguf"
4: Working Directory: .
Labels: main
  Test  #4: test-tokenizer-0-falcon
...
```

#### 步骤 4：确定用于调试的测试命令

因此对于上面的测试 #1，我们可以得到以下两条相关信息：

* 测试二进制文件：`~/llama.cpp/build-ci-debug/bin/test-tokenizer-0`
* 测试 GGUF 模型：`~/llama.cpp/tests/../models/ggml-vocab-llama-spm.gguf`

#### 步骤 5：在测试命令上运行 GDB

根据上面 ctest 的 "test command" 报告，我们可以使用以下命令运行 gdb 会话：

```bash
gdb --args ${Test Binary} ${Test GGUF Model}
```

示例：

```bash
gdb --args ~/llama.cpp/build-ci-debug/bin/test-tokenizer-0 "~/llama.cpp/tests/../models/ggml-vocab-llama-spm.gguf"
```
