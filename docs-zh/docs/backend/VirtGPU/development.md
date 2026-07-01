# 开发与测试

## 开发

### 代码生成

该后端使用 YAML 配置生成代码：

```bash
# 重新生成协议代码
cd ggml-virtgpu/
python regenerate_remoting.py
```

### 添加新算子

1. 将函数定义添加到 `ggmlremoting_functions.yaml`
2. 使用 `regenerate_remoting.py` 重新生成代码
3. 在 `virtgpu-forward-*.cpp` 中实现客机侧转发
4. 在 `backend-dispatched-*.cpp` 中实现主机侧处理

## 测试

本文档介绍如何在 macOS 上使用容器构建和测试 GGML-VirtGPU 后端。

### 前置条件

测试环境需要：

- macOS 主机系统
- 支持 `libkrun` 提供者的容器运行时（podman machine）
- 可访问 VirglRenderer 开发补丁集

### 所需补丁集

该后端需要当前正在审查中的补丁：

- **Virglrenderer APIR 上游 PR**：https://gitlab.freedesktop.org/virgl/virglrenderer/-/merge_requests/1590（仅供参考）
- **macOS 版 Virglrenderer（用于 krunkit）**：https://gitlab.freedesktop.org/kpouget/virglrenderer/-/tree/main-macos
- **Linux 版 Virglrenderer（用于 krun）**：https://gitlab.freedesktop.org/kpouget/virglrenderer/-/tree/main-linux

### 构建说明

#### 1. 构建 ggml-virtgpu-backend（主机侧，macOS）

```bash
# 构建原生运行在 macOS 上的后端
mkdir llama.cpp
cd llama.cpp
git clone https://github.com/ggml-org/llama.cpp.git src
cd src

LLAMA_MAC_BUILD=$PWD/build/ggml-virtgpu-backend

cmake -S . -B $LLAMA_MAC_BUILD \
      -DGGML_NATIVE=OFF \
      -DLLAMA_CURL=ON \
      -DGGML_VIRTGPU=ON \
      -DGGML_VIRTGPU_BACKEND=ONLY \
      -DGGML_METAL=ON

TARGETS="ggml-metal"
cmake --build $LLAMA_MAC_BUILD --parallel 8 --target $TARGETS

# 构建用于原生基准测试的附加工具
EXTRA_TARGETS="llama-run llama-bench"
cmake --build $LLAMA_MAC_BUILD --parallel 8 --target $EXTRA_TARGETS
```

#### 2. 构建 virglrenderer（主机侧，macOS）

```bash
# 构建支持 APIR 的 virglrenderer
mkdir virglrenderer
cd virglrenderer
git clone https://gitlab.freedesktop.org/kpouget/virglrenderer -b main-macos src
cd src

VIRGL_BUILD_DIR=$PWD/build

# -Dvenus=true 和 VIRGL_ROUTE_VENUS_TO_APIR=1 通过 Venus 后端路由 APIR 请求，
# 便于在未打补丁的虚拟机监控程序上进行测试

meson setup $VIRGL_BUILD_DIR \
      -Dvenus=true \
      -Dapir=true

ninja -C $VIRGL_BUILD_DIR
```

#### 3. 构建 ggml-virtgpu（客机侧，Linux）

选项 A：通过脚本构建：

```bash
# 在 Linux 容器内
mkdir llama.cpp
git clone https://github.com/ggml-org/llama.cpp.git src
cd src

LLAMA_LINUX_BUILD=$PWD/build-virtgpu

cmake -S . -B $LLAMA_LINUX_BUILD \
      -DGGML_VIRTGPU=ON

ninja -C $LLAMA_LINUX_BUILD
```

选项 B：构建带前端的容器镜像：

```bash
cat << EOF > remoting.containerfile
FROM quay.io/fedora/fedora:43
USER 0

WORKDIR /app/remoting

ARG LLAMA_CPP_REPO="https://github.com/ggml-org/llama.cpp.git"
ARG LLAMA_CPP_VERSION="master"
ARG LLAMA_CPP_CMAKE_FLAGS="-DGGML_VIRTGPU=ON"
ARG LLAMA_CPP_CMAKE_BUILD_FLAGS="--parallel 4"

RUN dnf install -y git cmake gcc gcc-c++ libcurl-devel libdrm-devel

RUN git clone "\${LLAMA_CPP_REPO}" src \\
 && git -C src fetch origin \${LLAMA_CPP_VERSION} \\
 && git -C src reset --hard FETCH_HEAD

RUN mkdir -p build \\
 && cd src \\
 && set -o pipefail \\
 && cmake -S . -B ../build \${LLAMA_CPP_CMAKE_FLAGS} \\
 && cmake --build ../build/ \${LLAMA_CPP_CMAKE_BUILD_FLAGS}

ENTRYPOINT ["/app/remoting/src/build/bin/llama-server"]
EOF

mkdir -p empty_dir
podman build -f remoting.containerfile ./empty_dir -t localhost/llama-cpp.virtgpu
```

### 环境配置

#### 设置 krunkit 环境变量

```bash
# 定义基础目录（根据你的系统调整这些路径）
VIRGL_BUILD_DIR=$HOME/remoting/virglrenderer/build
LLAMA_MAC_BUILD=$HOME/remoting/llama.cpp/build-backend

# 供 krunkit 加载自定义 virglrenderer 库
export DYLD_LIBRARY_PATH=$VIRGL_BUILD_DIR/src

# 供 Virglrenderer 加载 ggml-remotingbackend 库
export VIRGL_APIR_BACKEND_LIBRARY="$LLAMA_MAC_BUILD/bin/libggml-virtgpu-backend.dylib"

# 供 llama.cpp remotingbackend 加载 ggml-metal 后端
export APIR_LLAMA_CPP_GGML_LIBRARY_PATH="$LLAMA_MAC_BUILD/bin/libggml-metal.dylib"
export APIR_LLAMA_CPP_GGML_LIBRARY_REG=ggml_backend_metal_reg
```

#### 启动容器环境

```bash
# 将容器提供者设置为 libkrun
export CONTAINERS_MACHINE_PROVIDER=libkrun
podman machine start
```

#### 验证环境

确认 krunkit 正在使用正确的 virglrenderer 库：

```bash
lsof -c krunkit | grep virglrenderer
# 预期输出：
# krunkit 50574 user  txt  REG  1,14  2273912  10849442 ($VIRGL_BUILD_DIR/src)/libvirglrenderer.1.dylib
```

### 运行测试

#### 启动测试容器

```bash
# 可选：模型缓存
mkdir -p models
PODMAN_CACHE_ARGS="-v models:/models --user root:root --cgroupns host --security-opt label=disable -w /models"

podman run $PODMAN_CACHE_ARGS -it --rm --device /dev/dri localhost/llama-cpp.virtgpu
```

#### 在容器中测试 llama.cpp

```bash

# 运行性能基准测试
/app/remoting/build/bin/llama-bench -m ./llama3.2
```

预期输出（性能可能有所不同）：
```
| model                          |       size |     params | backend    | ngl |          test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | --: | ------------: | -------------------: |
| llama 3B Q4_K - Medium         |   1.87 GiB |     3.21 B | ggml-virtgpu |  99 |         pp512 |        991.30 ± 0.66 |
| llama 3B Q4_K - Medium         |   1.87 GiB |     3.21 B | ggml-virtgpu |  99 |         tg128 |         85.71 ± 0.11 |
```

### 故障排除

#### SSH 环境变量问题

⚠️ **警告**：在 macOS 上通过 SSH 设置 `DYLD_LIBRARY_PATH` 无效。以下是解决方法：

**解决方案 1：替换系统库**
```bash
VIRGL_BUILD_DIR=$HOME/remoting/virglrenderer/build  # ⚠️ 根据你的系统调整
BREW_VIRGL_DIR=/opt/homebrew/Cellar/virglrenderer/0.10.4d/lib
VIRGL_LIB=libvirglrenderer.1.dylib

cd $BREW_VIRGL_DIR
mv $VIRGL_LIB ${VIRGL_LIB}.orig
ln -s $VIRGL_BUILD_DIR/src/$VIRGL_LIB
```
