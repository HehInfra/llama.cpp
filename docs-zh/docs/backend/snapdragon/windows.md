## 概述

本文档介绍安装最新 GPU 和 NPU 驱动程序，以及 OpenCL 和 Hexagon SDK 的步骤。


要在 Snapdragon Windows 设备上使用 Hexagon NPU，底层 HTP Ops 库（例如 libggml-htp-v73.so）必须包含在由受信任证书数字签名的 .cat 文件中。

本文档详细介绍如何生成个人证书文件（.pfx），以及如何配置系统以允许测试签名（即 test-signing）。

## 安装最新 Adreno OpenCL SDK

可以使用精简版本（针对 CI 优化），下载地址：

    https://github.com/snapdragon-toolchain/opencl-sdk/releases/download/v2.3.2/adreno-opencl-sdk-v2.3.2-arm64-wos.tar.xz

或下载完整官方版本：

    https://softwarecenter.qualcomm.com/catalog/item/Adreno_OpenCL_SDK?version=2.3.2

将压缩包解压到
```
c:\Qualcomm\OpenCL_SDK\2.3.2
```

## 安装最新 Hexagon SDK Community Edition

可以使用精简版本（针对 CI 优化），下载地址：

    https://github.com/snapdragon-toolchain/hexagon-sdk/releases/download/v6.6.0.0/hexagon-sdk-v6.6.0.0-arm64-wos.tar.xz

或下载完整官方版本：

    https://softwarecenter.qualcomm.com/catalog/item/Hexagon_SDK?version=6.6.0.0

将压缩包解压到
```
c:\Qualcomm\Hexagon_SDK\6.6.0.0
```

## 安装最新 Adreno GPU 驱动程序

从以下地址下载驱动程序：

    https://softwarecenter.qualcomm.com/catalog/item/Windows_Graphics_Driver

自动安装并重启后，请确保 GPU 设备在 `Device Manager`（`Display Adapters` 下）中显示。

## 安装最新 Qualcomm NPU 驱动程序

从以下地址下载驱动程序：

    https://softwarecenter.qualcomm.com/catalog/item/Qualcomm_HND

自动安装并重启后，请确保 Hexagon NPU 设备在 `Device Manager`（`Neural Processors` 下）中显示。

如果设备不可用，可以尝试手动安装所有组件（`qcnspmcdm8380`、`qcnspmcdm8380_ext`）。这些组件会被解压到
```
c:\QCDrivers\qcnspmcdm...
```

## 启用 NPU 驱动程序测试签名

请注意，以下步骤仅针对 Hexagon NPU 需要。Adreno GPU 后端不需要测试签名。

### 启用 testsigning

使用 `bcdedit` 启用 test-signing
```
> bcdedit /set TESTSIGNING ON
```
（可能需要禁用 Secure Boot 才能生效）

重启后请确认 test-signing 已启用
```
> bcdedit /enum
...
testsigning             Yes
...
```
更多详情请参考 Microsoft 指南：

   https://learn.microsoft.com/en-us/windows-hardware/drivers/install/the-testsigning-boot-configuration-option

### 创建个人证书

本过程所需的工具是 Windows SDK 和 Windows Driver Kit 的一部分，应在安装 MS Visual Studio 时一并安装。它们通常位于
```
c:\Program Files (x86)\Windows Kits\10\bin\10.0.26100.0
```
（将 10.0.26100.0 替换为正确的版本号）。

要创建个人自签名证书，请运行以下命令（在 cmd 或 power-shell 中均可）：
```
> cd c:\Users\MyUser
> mkdir Certs
> cd Certs
> makecert -r -pe -ss PrivateCertStore -n CN=GGML.HTP.v1 -eku 1.3.6.1.5.5.7.3.3 -sv ggml-htp-v1.pvk ggml-htp-v1.cer
> pvk2pfx.exe -pvk ggml-htp-v1.pvk -spc ggml-htp-v1.cer -pfx ggml-htp-v1.pfx
```
（将 `MyUser` 替换为你的用户名）。

将该证书添加到 `Trusted Root Certification Authorities` 和 `Trusted Publishers` 存储区。可以使用 `certlm` 证书管理器工具完成。右键单击证书存储区，选择 `All Tasks -> Import`，然后按照提示从上面创建的 PFX 文件导入证书。

更多详情请参考 Microsoft 指南：

    https://learn.microsoft.com/en-us/windows-hardware/drivers/install/introduction-to-test-signing

请务必保存 PFX 文件，构建过程会用到它。请注意，同一份证书可用于签署任意次数的构建。

## 构建带有已签名 HTP ops 库的 Hexagon 后端

Snapdragon 上 Windows 的 Hexagon 后端整体构建流程与其他平台相同。但生成和签名 HTP Ops 库需要额外设置。
```
> $env:OPENCL_SDK_ROOT="C:\Qualcomm\OpenCL_SDK\2.3.2"
> $env:HEXAGON_SDK_ROOT="C:\Qualcomm\Hexagon_SDK\6.6.0.0"
> $env:HEXAGON_TOOLS_ROOT="C:\Qualcomm\Hexagon_SDK\6.6.0.0\tools\HEXAGON_Tools\19.0.07"
> $env:HEXAGON_HTP_CERT="c:\Users\MyUsers\Certs\ggml-htp-v1.pfx"
> $env:WINDOWS_SDK_BIN="C:\Program Files (x86)\Windows Kits\10\bin\10.0.26100.0"

> cmake --preset arm64-windows-snapdragon-release -B build-wos
...
> cmake --install build-wos --prefix pkg-snapdragon
```

构建完成后，HTP ops 库将按如下方式安装：
```
> dir pkg-snapdragon/lib
...
-a----         1/22/2026   6:01 PM         187656 libggml-htp-v73.so
-a----         1/22/2026   6:01 PM         191752 libggml-htp-v75.so
-a----         1/22/2026   6:01 PM         187656 libggml-htp-v79.so
-a----         1/22/2026   6:01 PM         187656 libggml-htp-v81.so
-a----         1/22/2026   6:01 PM           4139 libggml-htp.cat
```

可以通过以下命令验证 .cat 文件、签名以及证书安装是否正确：

```
> signtool.exe verify /v /pa .\pkg-snapdragon\lib\libggml-htp.cat
Verifying: .\pkg-snapdragon\lib\libggml-htp.cat

Signature Index: 0 (Primary Signature)
Hash of file (sha256): 9820C664DA59D5EAE31DBB664127FCDAEF59CDC31502496BC567544EC2F401CF

Signing Certificate Chain:
        Issued to: GGML.HTP.v1
...
Successfully verified: .\pkg-snapdragon\lib\libggml-htp.cat
...
```
