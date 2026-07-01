# GGML-VirtGPU 后端配置

本文档介绍 ggml-virtgpu 后端系统使用的环境变量，涵盖前端（Guest 侧）和后端（Host 侧）组件。

## 环境变量概览

ggml-virtgpu 后端使用环境变量对三个主要组件进行配置：
- **前端（Guest）**：在虚拟机中运行的 GGML 应用程序
- **Hypervisor**：Virglrenderer/APIR 系统
- **后端（Host）**：Host 侧 GGML 后端集成

## 前端（Guest 侧）配置

### GGML_REMOTING_USE_APIR_CAPSET
- **位置**：`ggml/src/ggml-virtgpu/virtgpu.cpp`
- **类型**：布尔标志（基于存在性）
- **用途**：控制通信时使用的 virtio-gpu 能力集
- **取值**：
  - 已设置（任意值）：使用 APIR capset（长期设置）
  - 未设置：使用 Venus capset（在未修改的 hypervisor 上测试更方便）
- **默认**：未设置（Venus capset）
- **用法**：
  ```bash
  export GGML_REMOTING_USE_APIR_CAPSET=1  # Use APIR capset
  # or leave unset for Venus capset
  ```

## Hypervisor（Virglrenderer/APIR）配置

这些环境变量用于在未修改的 hypervisor（不支持 VirglRenderer APIR 组件）上运行的过渡阶段。它们将在未来被移除，届时 hypervisor 将改用 APIR _Configuration Key_ 来配置 VirglRenderer。

### VIRGL_APIR_BACKEND_LIBRARY
- **位置**：`virglrenderer/src/apir/apir-context.c`
- **Configuration Key**：`apir.load_library.path`
- **类型**：文件路径字符串
- **用途**：指定 virglrenderer 应动态加载的 APIR 后端库路径
- **必需**：是
- **示例**：
  ```bash
  export VIRGL_APIR_BACKEND_LIBRARY="/path/to/libggml-remotingbackend.so"
  ```

### VIRGL_ROUTE_VENUS_TO_APIR
- **位置**：`virglrenderer/src/apir/apir-renderer.h`
- **类型**：布尔标志（基于存在性）
- **用途**：在 hypervisor 过渡期间，将 Venus capset 调用路由到 APIR 的临时变通方案
- **状态**：一旦 hypervisor 原生支持 APIR 后将被移除
- **警告**：会破坏正常的 Vulkan/Venus 功能
- **用法**：
  ```bash
  export VIRGL_ROUTE_VENUS_TO_APIR=1  # For testing with an unmodified hypervisor
  ```

### VIRGL_APIR_LOG_TO_FILE
- **位置**：`virglrenderer/src/apir/apir-renderer.c`
- **环境变量**：`VIRGL_APIR_LOG_TO_FILE`
- **类型**：文件路径字符串
- **用途**：启用 VirglRenderer APIR 组件到指定文件的调试日志
- **必需**：否（可选调试）
- **默认**：输出日志到 `stderr`
- **用法**：
  ```bash
  export VIRGL_APIR_LOG_TO_FILE="/tmp/apir-debug.log"
  ```

## 后端（Host 侧）配置

这些环境变量用于在未修改的 hypervisor（不支持 VirglRenderer APIR 组件）上运行的过渡阶段。它们将在未来被移除，届时 hypervisor 将改用 APIR _Configuration Key_ 来配置 VirglRenderer。

### APIR_LLAMA_CPP_GGML_LIBRARY_PATH
- **位置**：`ggml/src/ggml-virtgpu/backend/backend.cpp`
- **环境变量**：`APIR_LLAMA_CPP_GGML_LIBRARY_PATH`
- **Configuration Key**：`ggml.library.path`
- **类型**：文件路径字符串
- **用途**：实际 GGML 后端库的路径（Metal、CUDA、Vulkan 等）
- **必需**：**是** —— 缺少此项后端初始化将失败
- **示例**：
  ```bash
  # macOS with Metal backend
  export APIR_LLAMA_CPP_GGML_LIBRARY_PATH="/opt/llama.cpp/lib/libggml-metal.dylib"

  # Linux with CUDA backend
  export APIR_LLAMA_CPP_GGML_LIBRARY_PATH="/opt/llama.cpp/lib/libggml-cuda.so"

  # macOS or Linux with Vulkan backend
  export APIR_LLAMA_CPP_GGML_LIBRARY_PATH="/opt/llama.cpp/lib/libggml-vulkan.so"
  ```

### APIR_LLAMA_CPP_GGML_LIBRARY_REG
- **位置**：`ggml/src/ggml-virtgpu/backend/backend.cpp`
- **环境变量**：`APIR_LLAMA_CPP_GGML_LIBRARY_REG`
- **Configuration Key**：`ggml.library.reg`
- **类型**：函数符号名字符串
- **用途**：加载库后要调用的后端注册函数名称
- **必需**：否（默认值为 `ggml_backend_init`）
- **默认**：`ggml_backend_init`
- **示例**：
  ```bash
  # Metal backend
  export APIR_LLAMA_CPP_GGML_LIBRARY_REG="ggml_backend_metal_reg"

  # CUDA backend
  export APIR_LLAMA_CPP_GGML_LIBRARY_REG="ggml_backend_cuda_reg"

  # Vulkan backend
  export APIR_LLAMA_CPP_GGML_LIBRARY_REG="ggml_backend_vulkan_reg"

  # Generic fallback (default)
  # export APIR_LLAMA_CPP_GGML_LIBRARY_REG="ggml_backend_init"
  ```

### APIR_LLAMA_CPP_LOG_TO_FILE
- **位置**：`ggml/src/ggml-virtgpu/backend/backend.cpp:62`
- **环境变量**：`APIR_LLAMA_CPP_LOG_TO_FILE`
- **类型**：文件路径字符串
- **用途**：启用 GGML 后端到指定文件的调试日志
- **必需**：否（可选调试）
- **用法**：
  ```bash
  export APIR_LLAMA_CPP_LOG_TO_FILE="/tmp/ggml-backend-debug.log"
  ```

## 配置流程

配置系统的工作流程如下：

1. **Hypervisor 设置**：Virglrenderer 加载由 `VIRGL_APIR_BACKEND_LIBRARY` 指定的 APIR 后端库

2. **上下文创建**：创建 APIR 上下文时，会使用以下环境变量填充配置表：
   - `apir.load_library.path` ← `VIRGL_APIR_BACKEND_LIBRARY`
   - `ggml.library.path` ← `APIR_LLAMA_CPP_GGML_LIBRARY_PATH`
   - `ggml.library.reg` ← `APIR_LLAMA_CPP_GGML_LIBRARY_REG`
   - 此步骤最终将由 hypervisor 自身完成，使用命令行参数而非环境变量。

3. **后端初始化**：后端通过回调查询配置：
   - `virgl_cbs->get_config(ctx_id, "ggml.library.path")` 返回库路径
   - `virgl_cbs->get_config(ctx_id, "ggml.library.reg")` 返回注册函数

4. **库加载**：后端动态加载并初始化指定的 GGML 库

## 错误信息

常见错误场景及其信息：

- **缺少库路径**：`"cannot open the GGML library: env var 'APIR_LLAMA_CPP_GGML_LIBRARY_PATH' not defined"`
- **缺少注册函数**：`"cannot register the GGML library: env var 'APIR_LLAMA_CPP_GGML_LIBRARY_REG' not defined"`

## 完整配置示例

以下是在使用 Metal 后端的 macOS Host 上的完整配置示例：

```bash
# Hypervisor environment
export VIRGL_APIR_BACKEND_LIBRARY="/opt/llama.cpp/lib/libggml-virtgpu-backend.dylib"

# Backend configuration
export APIR_LLAMA_CPP_GGML_LIBRARY_PATH="/opt/llama.cpp/lib/libggml-metal.dylib"
export APIR_LLAMA_CPP_GGML_LIBRARY_REG="ggml_backend_metal_reg"

# Optional logging
export VIRGL_APIR_LOG_TO_FILE="/tmp/apir.log"
export APIR_LLAMA_CPP_LOG_TO_FILE="/tmp/ggml.log"

# Guest configuration
export GGML_REMOTING_USE_APIR_CAPSET=1
```
