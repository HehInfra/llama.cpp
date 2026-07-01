BLIS 安装手册
------------------------

BLIS 是一个可移植的软件框架，用于构建高性能的类 BLAS 密集线性代数库。它曾获得多项荣誉和认可，包括 2023 年 James H. Wilkinson 数值软件奖和 2020 年 SIAM 超级计算活动组最佳论文奖。BLIS 提供了一种新的类 BLAS API，以及一个用于传统 BLAS 例程调用的兼容性层。它具有对象式 API、类型化 API、BLAS 和 CBLAS 兼容层等特性。

项目 URL：https://github.com/flame/blis

### 准备工作：

编译 BLIS：

```bash
git clone https://github.com/flame/blis
cd blis
./configure --enable-cblas -t openmp,pthreads auto
# 默认将安装到 /usr/local/
make -j
```

安装 BLIS：

```bash
sudo make install
```

我们推荐使用 openmp，因为它更便于调整使用的核心数。

### llama.cpp 编译

CMake：

```bash
mkdir build
cd build
cmake -DGGML_BLAS=ON -DGGML_BLAS_VENDOR=FLAME ..
make -j
```

### llama.cpp 运行

根据 BLIS 文档，我们可以设置以下环境变量来调整 openmp 的行为：

```bash
export GOMP_CPU_AFFINITY="0-19"
export BLIS_NUM_THREADS=14
```

然后正常运行二进制文件即可。


### Intel 特定问题

有些人可能会遇到 `libimf.so` 找不到的错误提示。请参考这个 [stackoverflow 页面](https://stackoverflow.com/questions/70687930/intel-oneapi-2022-libimf-so-no-such-file-or-directory-during-openmpi-compila)。

### 参考：

1. https://github.com/flame/blis#getting-started
2. https://github.com/flame/blis/blob/master/docs/Multithreading.md
