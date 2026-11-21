HiSilicon-Development 编译教程
===========

开始使用
---------------

使用 `x86_64` Debian 12 主机，准备 Git、repo 和 Python 3.11+。工作区路径使用无空格的目录名。

```sh
mkdir DVB-IP && cd DVB-IP
repo init -u https://github.com/HiSilicon-Development/manifest.git -b main
repo sync -c -j4
bash scripts/install-deps.sh
```

同步后的目录为 `kernel/`、`ffmpeg/` 和 `scripts/`。

编译说明
---------------

在工作区根目录运行 `./build.sh`：

```sh
# 单板编译
./build.sh --boards 1001 --jobs 32
./build.sh --boards 1002 --jobs 32
./build.sh --boards 1006 --jobs 32
./build.sh --boards ec6110t --jobs 32
# 1004：共用 Image/modules，分别生成两种 recovery
./build.sh --boards 1004 --1004-variant all --jobs 32
# 只生成指定的 1004 RF 变体
./build.sh --boards 1004 --1004-variant mxl214 --jobs 32
./build.sh --boards 1004 --1004-variant r858 --jobs 32
# 指定组合或全部板型（all 包含 EC6110T）
./build.sh --boards 1001,1002,1006 --jobs 32
./build.sh --boards all --jobs 32
# 单独编译 FFmpeg，或编译内核及 FFmpeg
./build.sh --target ffmpeg --jobs 32
./build.sh --target all --boards all --jobs 32
```

内核构建用 `--boards` 选板，适用值为 `1001`、`1002`、`1004`、`1006`、`ec6110t` 和 `all`。1004 按板载射频芯片选 `--1004-variant mxl214` 或 `r858`，默认 `all`。运行 `./build.sh --help` 查看参数。

C51 与启动固件
---------------

CV200 默认使用 `scripts/boot/hi3798cv200-pm-c51.bin`。用 `--c51-firmware PATH` 指定匹配的二进制或 BSP 字数组：

```sh
./build.sh --boards 1004 --c51-firmware /path/to/authorized/output_ca_release.txt
```

脚本将 BSP 字数组按小端转换成 21,468 字节的二进制，并检查 SHA-256：`1ae4b66809f09bce59469c71bcba649856c66f9bcd24366cf0f5b741697959e0`。EC6110T 和单独编译 FFmpeg 时跳过 C51。

产物位置
---------------

内核、DTB、模块和构建元数据位于 `out/kernel/<板型>/`：

- 1001、1006：`trustedcore.img`。
- 1002：`kernel.p4.img`。
- 1004：共享 `Image` 和 `modules.tar.gz`；在 `variants/mxl214/`、`variants/r858/` 中查找对应的 `recovery.img` 和 DTB。
- EC6110T：`kernel.img`。

更新包及其 SHA-256 文件位于 `out/updates/`。

FFmpeg 使用同一工作区内核导出的 UAPI 头文件。用 Debian 12 主机构建 Debian 12／13 的 ARM64 产物。

FFmpeg 产物位于 `out/ffmpeg/`：`payload/bin/` 下有 `hivxe-ffmpeg` 和 `hivxe-ffprobe`，配套库在 `payload/lib/hivxe-ffmpeg/`，构建记录在 `metadata/`。

相关资源
---------------

板级硬件参数与启动架构请查看 [Hi3798CV200 文档](https://github.com/HiSilicon-Development/Hi3798Cv200)。
