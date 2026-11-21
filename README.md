HiSilicon-Development
===========

开始使用
---------------

推荐 Linux `x86_64` / Debian 12 主机，准备 Git、repo 和 Python 3.11+；工作区路径不要含空格。

```sh
mkdir DVB-IP && cd DVB-IP
repo init -u https://github.com/HiSilicon-Development/manifest.git -b main
repo sync -c -j4
bash scripts/install-deps.sh
```

manifest 从 HiSilicon-Development 的 GitHub 仓库同步 `kernel/`（`linux-6.12.y`）、`ffmpeg/`（`FFmpeg` 的 `9.0.1` 分支）与 `scripts/`（`build`）。
依赖脚本在编译主机安装内核工具链、ARM64 交叉编译器及 libdrm/libudev multiarch 开发库，不操作目标设备。查看说明可运行 `bash scripts/install-deps.sh --help`。

编译说明
---------------

工作区根目录的 `./build.sh` 和 `./scripts/build.sh` 共用构建入口；`--target` 可选 `kernel`（默认）、`ffmpeg`、`all`。工作区由脚本自身位置确定。

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

完整参数形式为：

```text
./build.sh [--target kernel|ffmpeg|all] [--boards 1001,1002,1004,1006,ec6110t|all] [--jobs N] [--1004-variant all|mxl214|r858] [--c51-firmware PATH]
```

`kernel` / `all` 必须明确指定 `--boards`，不默认全编；`ffmpeg` 不接受板型参数。`--jobs` 可省略。1004 的 `--1004-variant` 可选 `all`（默认）、`mxl214`（通常为 2+8）或 `r858`（通常为 2+4），按实际 RF 芯片选择，不按容量猜测；指定该参数时，`--boards` 必须包含 1004。

C51 与启动固件
---------------

CV200 PM 构建默认使用仓库中的 `scripts/boot/hi3798cv200-pm-c51.bin`。该二进制纳入版本管理，构建前仍校验大小和摘要。也可通过 `--c51-firmware PATH` 指定匹配的原始二进制或 BSP 的 `source/component/pm/c51/output_ca_release.txt` 字数组，例如：

```sh
./build.sh --boards 1004 --c51-firmware /path/to/authorized/output_ca_release.txt
```

C51 是独立 MCU 的固件，不是 `scripts/boot/*.S` 中的 ARM 启动/交接汇编；ARM `.S` 不能替代它。字数组输入仅允许 32 位十六进制字和空白/逗号分隔，按小端还原后与原始二进制执行相同校验：

- 大小必须为 **21468 字节**。
- SHA-256 必须为 `1ae4b66809f09bce59469c71bcba649856c66f9bcd24366cf0f5b741697959e0`。
- 源码记录的来源为 [HiSTBLinuxV100R005C00SPC060 / output_ca_release.txt](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060/blob/a82e5fee18a794ecdf28401b21c25cddec0d21fb/source/component/pm/c51/output_ca_release.txt)。来源记录不授予使用或再分发许可，须自行确认授权。

构建脚本不下载固件；缺失或校验失败即停止，不自动禁用 PM。EC6110T（Hi3798MV310）不需要 C51。将固件纳入仓库不会改变原厂许可条件，也不表示它受监控器源码的 GPL 许可覆盖。

产物与发布边界
---------------

内核产物位于 `out/kernel/<板型>/`，包含内核、板型启动容器、配套 DTB/启动与 PM 固件及模块，并附带配置、摘要和构建元数据：

- 1001、1006：`trustedcore.img`。
- 1002：`kernel.p4.img`。
- 1004：共享根目录的 `Image`、`modules.tar.gz` 等；各 RF 变体的 `recovery.img`、DTB 和交接产物分别位于 `variants/mxl214/`、`variants/r858/`。只构建一个变体时，不保留另一变体的旧产物。
- EC6110T：`kernel.img`。

容器、DTB、启动/PM 固件与模块必须来自同次构建。构建成功不代表实机启动、RF、待机或唤醒功能已验收。

FFmpeg 每次先从同一工作区的 `kernel/` 执行 `headers_install`，再使用新导出的 V4L2、VPSS 和 DRM UAPI 编译。导出的 `out/uapi/include/` 与 `out/uapi/include/drm/` 优先于系统头文件；libdrm 的用户态 API 和链接库仍由 ARM64 开发包提供。单独构建 FFmpeg 不要求先编译 Image，但必须有配套内核源码。

FFmpeg 产物位于 `out/ffmpeg/`：`payload/bin/hivxe-ffmpeg`、`hivxe-ffprobe`，以及 `payload/lib/hivxe-ffmpeg/` 下的配套共享库；`metadata/` 记录实际源码、内核版本、UAPI 摘要与 ABI 信息。许可证和版权材料位于 `licenses/`。为兼容 Debian 12/13，使用 Debian 12 编译主机；ABI 检查拒绝要求高于 GLIBC 2.36 的产物。

本仓库仅负责内核及配套 FFmpeg 编译，不提供 rootfs 制作、HiTool 操作、刷写/更新包或设备安装脚本，不自动写入设备。发布源码时不上传 `out/`、`.repo/` 或 Python 缓存；C51 二进制保留原厂来源与许可边界。发布 FFmpeg 二进制还须保留许可证、第三方版权声明，并提供匹配的完整对应源码、接口定义及构建脚本；摘要清单不能替代源码交付。

相关资源
---------------

板级硬件参数与启动架构请查看 [Hi3798CV200 文档](https://github.com/HiSilicon-Development/Hi3798Cv200)。
