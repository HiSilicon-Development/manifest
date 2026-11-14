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

1004 Hi3130/R858 配置的 `hisilicon,suspend-disabled` 只禁止系统休眠，不删除 CVPM 控制器或 C51：CPU 生命周期和原有红外唤醒关机仍需同一套监控器。不得为了禁用深睡而省略该变体的 C51/监控器，或把短按电源键改成关机。

产物与发布边界
---------------

内核产物位于 `out/kernel/<板型>/`，包含内核、板型启动容器、配套 DTB/启动与 PM 固件及模块，并附带配置、摘要和构建元数据：

- 1001、1006：`trustedcore.img`。
- 1002：`kernel.p4.img`。
- 1004：共享根目录的 `Image`、`modules.tar.gz` 等；各 RF 变体的 `recovery.img`、DTB 和交接产物分别位于 `variants/mxl214/`、`variants/r858/`。只构建一个变体时，不保留另一变体的旧产物。
- EC6110T：`kernel.img`。

容器、DTB、启动/PM 固件与模块必须来自同次构建。构建成功不代表实机启动、RF、待机或唤醒功能已验收。

每份 DTB 的 `system_suspend_enabled` 元数据记录其实际休眠策略；1004 分别位于 `variants.mxl214` 与 `variants.r858`，不能用板型共用的监控器能力替代变体策略。当前 R858 为 `false`，MxL214 与 1001/1002/1006 保留入口；这不是硬件验收结论，构建产物的 `deep_standby_hardware_verified` 仍为 `false`。已有实机记录包括 1006 和 1004 MxL214 的同次启动 IR 恢复，1001/1002 尚待测试；新增按键策略的验证范围与低层 STR 记录分开。

公共内核配置启用 `CONFIG_HI3798CV200_POWERKEY`：正常 CV200 DVB-IP 板型默认内核处理短按 deep/五秒长按有序关机，R858 默认短按 ignore。更新包不再需要额外安装 logind 文件来取得默认行为，也不覆盖管理员配置。root 可使用 `standby-controller/power_key_policy` 改动作或显式切换 `userspace` 交还标准 `KEY_POWER` 事件；需要 systemd sleep hooks/inhibitors 的系统应选择后者。该配置不改变 CVPM/C51 的睡眠中红外唤醒机制，接口与状态含义见内核 `sysfs-driver-hi3798cv200-pm` ABI 文档。

每个板型内核编译、容器校验和模块安装完成后，会自动在 `out/updates/` 生成可安装的更新归档及 `.sha256`：

- `update-1001-<version>-<YYYYMMDD>.tar.gz`、`update-1002-<version>-<YYYYMMDD>.tar.gz`、`update-1006-<version>-<YYYYMMDD>.tar.gz`。
- 1004 按所选 RF 分别生成 `update-1004-mxl214-<version>-<YYYYMMDD>.tar.gz`、`update-1004-r858-<version>-<YYYYMMDD>.tar.gz`，不能交叉使用。
- EC6110T 生成 `update-ec6110t-<version>-<YYYYMMDD>.tar.gz`。

版本取自实际构建元数据，日期使用制包主机的本地日期，不使用可复现编译的 `SOURCE_DATE_EPOCH=0`。同名的已生成归档会在新归档验证成功后替换；其它板型、RF 或日期的包保留。单独编译 FFmpeg 不生成内核 update。

更新包包含同次构建的完整内核容器、模块、配置/符号版本/构建元数据、许可证和 `update.sh`，不包含 rootfs、Fastboot、bootargs 或运行时 firmware，不修改账户和网络。设备上已有的 firmware 保留；需要新增 firmware 时另行安装有授权的匹配文件，不能把“模块已打包”视为对应 firmware 已提供。内嵌 C51 的许可边界不变。

也可仅对已有编译产物制包，不重新编译：

```sh
python3 scripts/updates.py --kernel-output out/kernel --kernel-source kernel --output out/updates --boards 1002
python3 scripts/updates.py --kernel-output out/kernel --kernel-source kernel --output out/updates --boards 1004 --1004-variant r858
```

在匹配板型的现有 ARM64 Linux 紧凑分区系统中，先验证归档旁的 `.sha256`，解压到全新目录，进入 `update-<board>`（1004 为 `update-1004-<RF>`），先执行 `sudo sh update.sh --check`，再执行 `sudo sh update.sh`。安装器检查板型、RF、完整 eMMC 布局/rootfs 边界、载荷摘要和模块 ABI，写入后完整读回验证 SHA-256；不支持按容量猜分区，也不用于直接迁移原厂布局。

默认不重启，不删除旧版本模块或包外文件；成功后手动重启。同版本覆盖也须重启，不能现场重载模块。`--force` 只跳过板型标签，不能跳过 RF、布局、ABI 或摘要检查；`--kernel-partition` 只接受目标分区的同一块设备别名。编译或制包过程不会自动接触任何目标设备。

制包归档的安全边界测试可在工作区根目录运行 `PYTHONDONTWRITEBYTECODE=1 python3 -m unittest discover -s scripts -p test_updates.py`；该测试不编译内核，也不访问目标设备。

FFmpeg 每次先从同一工作区的 `kernel/` 执行 `headers_install`，再使用新导出的 V4L2、VPSS 和 DRM UAPI 编译。导出的 `out/uapi/include/` 与 `out/uapi/include/drm/` 优先于系统头文件；libdrm 的用户态 API 和链接库仍由 ARM64 开发包提供。单独构建 FFmpeg 不要求先编译 Image，但必须有配套内核源码。

FFmpeg 产物位于 `out/ffmpeg/`：`payload/bin/hivxe-ffmpeg`、`hivxe-ffprobe`，以及 `payload/lib/hivxe-ffmpeg/` 下的配套共享库；`metadata/` 记录实际源码、内核版本、UAPI 摘要与 ABI 信息。许可证和版权材料位于 `licenses/`。为兼容 Debian 12/13，使用 Debian 12 编译主机；ABI 检查拒绝要求高于 GLIBC 2.36 的产物。

本仓库负责内核、配套 FFmpeg 编译及内核/模块 update 归档，不提供 rootfs 制作或 HiTool 操作。随包安装器仅在用户显式执行后才写设备；编译和制包不自动写入设备。发布源码时不上传 `out/`、`.repo/` 或 Python 缓存；C51 二进制保留原厂来源与许可边界。发布 FFmpeg 二进制还须保留许可证、第三方版权声明，并提供匹配的完整对应源码、接口定义及构建脚本；摘要清单不能替代源码交付。

相关资源
---------------

板级硬件参数与启动架构请查看 [Hi3798CV200 文档](https://github.com/HiSilicon-Development/Hi3798Cv200)。
