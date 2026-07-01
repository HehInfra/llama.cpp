> [!IMPORTANT]
> 本构建文档仅适用于 RISC-V SpacemiT 处理器。

## 本地构建 llama.cpp（适用于 riscv64）

1. 准备 RISC-V 工具链
~~~
wget https://github.com/spacemit-com/toolchain/releases/download/v1.2.4/spacemit-toolchain-linux-glibc-x86_64-v1.2.4.tar.xz
~~~

2. 构建
以下是构建脚本：需要使用 RISC-V 向量指令进行加速。确保启用 `GGML_CPU_RISCV64_SPACEMIT` 编译选项。当前支持的优化版本为 `RISCV64_SPACEMIT_IME1` 和 `RISCV64_SPACEMIT_IME2`，对应 `RISCV64_SPACEMIT_IME_SPEC` 编译选项。编译器配置定义在 `riscv64-spacemit-linux-gnu-gcc.cmake` 文件中。请确保已安装 RISC-V 编译器，并通过 `export RISCV_ROOT_PATH={your_compiler_path}` 设置环境变量。
```bash

cmake -B build \
    -DCMAKE_BUILD_TYPE=Release \
    -DGGML_CPU_RISCV64_SPACEMIT=ON \
    -DGGML_CPU_REPACK=OFF \
    -DLLAMA_OPENSSL=OFF \
    -DGGML_RVV=ON \
    -DGGML_RV_ZVFH=ON \
    -DGGML_RV_ZFH=ON \
    -DGGML_RV_ZICBOP=ON \
    -DGGML_RV_ZIHINTPAUSE=ON \
    -DGGML_RV_ZBA=ON \
    -DCMAKE_TOOLCHAIN_FILE=${PWD}/cmake/riscv64-spacemit-linux-gnu-gcc.cmake \
    -DCMAKE_INSTALL_PREFIX=build/installed

cmake --build build --parallel $(nproc) --config Release

pushd build
make install
popd
```

## 仿真
您可以使用 QEMU 在非 RISC-V 架构上执行仿真。

1. 下载 QEMU
~~~
wget https://archive.spacemit.com/spacemit-ai/qemu/jdsk-qemu-v0.0.14.tar.gz
~~~

2. 运行仿真
构建 llama.cpp 后，您可以通过 QEMU 运行可执行文件进行仿真，例如：
~~~
export QEMU_ROOT_PATH={your QEMU file path}
export RISCV_ROOT_PATH_IME1={your RISC-V compiler path}

${QEMU_ROOT_PATH}/bin/qemu-riscv64 -L ${RISCV_ROOT_PATH_IME1}/sysroot -cpu max,vlen=256,elen=64,vext_spec=v1.0 ${PWD}/build/bin/llama-cli -m ${PWD}/models/Qwen2.5-0.5B-Instruct-Q4_0.gguf -t 1
~~~

## 矩阵量化支持

| 量化类型 | X60 | A100 |
| ---: | ---: | ---: |
| Q2_K |  | :heavy_check_mark: |
| Q3_K |  | :heavy_check_mark: |
| Q4_0 | :heavy_check_mark: | :heavy_check_mark: |
| Q4_1 | :heavy_check_mark: | :heavy_check_mark: |
| Q4_K | :heavy_check_mark: | :heavy_check_mark: |
| Q5_0 |  | :heavy_check_mark: |
| Q5_1 |  | :heavy_check_mark: |
| Q5_K |  | :heavy_check_mark: |
| Q6_K |  | :heavy_check_mark: |
| Q8_0 |  | :heavy_check_mark: |


## 性能
* Spacemit(R) X60
~~~
model name      : Spacemit(R) X60
isa             : rv64imafdcv_zicbom_zicboz_zicntr_zicond_zicsr_zifencei_zihintpause_zihpm_zfh_zfhmin_zca_zcd_zba_zbb_zbc_zbs_zkt_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_zvkt_sscofpmf_sstc_svinval_svnapot_svpbmt
mmu             : sv39
uarch           : spacemit,x60
mvendorid       : 0x710
marchid         : 0x8000000058000001
~~~

| model                          |       size |     params | backend    | threads | n_ubatch | fa | mmap |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | ------: | -------: | -: | ---: | --------------: | -------------------: |
| qwen35 2B Q4_1                 |   1.19 GiB |     1.88 B | CPU        |       4 |      128 |  1 |    0 |           pp128 |         10.32 ± 0.02 |
| qwen35 2B Q4_1                 |   1.19 GiB |     1.88 B | CPU        |       4 |      128 |  1 |    0 |           tg128 |          3.07 ± 0.01 |
| qwen3 0.6B Q4_0                | 358.78 MiB |   596.05 M | CPU        |       4 |      128 |  1 |    0 |           pp128 |         49.15 ± 0.25 |
| qwen3 0.6B Q4_0                | 358.78 MiB |   596.05 M | CPU        |       4 |      128 |  1 |    0 |           tg128 |         11.73 ± 0.02 |


* Spacemit(R) A100
~~~
model name      : Spacemit(R) A100
isa             : rv64imafdcvh_zicbom_zicbop_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zimop_zaamo_zalrsc_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zcmop_zba_zbb_zbc_zbs_zkt_zvbb_zvbc_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_zvkb_zvkg_zvkned_zvknha_zvknhb_zvksed_zvksh_zvkt_smaia_smstateen_ssaia_sscofpmf_sstc_svinval_svnapot_svpbmt_sdtrig
mmu             : sv39
mvendorid       : 0x710
marchid         : 0x8000000041000002
mimpid          : 0x10000000d5686200
hart isa        : rv64imafdcv_zicbom_zicbop_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zimop_zaamo_zalrsc_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zcmop_zba_zbb_zbc_zbs_zkt_zvbb_zvbc_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_zvkb_zvkg_zvkned_zvknha_zvknhb_zvksed_zvksh_zvkt_smaia_smstateen_ssaia_sscofpmf_sstc_svinval_svnapot_svpbmt_sdtrig
~~~

| model                          |       size |     params | backend    | threads | n_ubatch | fa | mmap |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | ------: | -------: | -: | ---: | --------------: | -------------------: |
| qwen3 0.6B Q4_0                | 358.78 MiB |   596.05 M | CPU        |       8 |      128 |  1 |    0 |           pp128 |        565.83 ± 0.31 |
| qwen3 0.6B Q4_0                | 358.78 MiB |   596.05 M | CPU        |       8 |      128 |  1 |    0 |           tg128 |         55.77 ± 0.02 |
| qwen3 4B Q4_0                  |   2.21 GiB |     4.02 B | CPU        |       8 |      128 |  1 |    0 |           pp128 |         79.74 ± 0.04 |
| qwen3 4B Q4_0                  |   2.21 GiB |     4.02 B | CPU        |       8 |      128 |  1 |    0 |           tg128 |         11.29 ± 0.00 |
| qwen3moe 30B.A3B Q4_0          |  16.18 GiB |    30.53 B | CPU        |       8 |      128 |  1 |    0 |           pp128 |         57.88 ± 0.31 |
| qwen3moe 30B.A3B Q4_0          |  16.18 GiB |    30.53 B | CPU        |       8 |      128 |  1 |    0 |           tg128 |         12.79 ± 0.00 |
| qwen35 2B Q4_1                 |   1.19 GiB |     1.88 B | CPU        |       8 |      128 |  1 |    0 |           pp128 |        115.23 ± 0.04 |
| qwen35 2B Q4_1                 |   1.19 GiB |     1.88 B | CPU        |       8 |      128 |  1 |    0 |           tg128 |         16.49 ± 0.01 |
| gemma4 E4B Q4_K - Medium       |   4.76 GiB |     7.52 B | CPU        |       8 |      128 |  1 |    0 |           pp128 |         21.13 ± 0.01 |
| gemma4 E4B Q4_K - Medium       |   4.76 GiB |     7.52 B | CPU        |       8 |      128 |  1 |    0 |           tg128 |          5.66 ± 0.00 |
