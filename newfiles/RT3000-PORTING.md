# H3C RT3000 iStoreOS 移植记录

## 当前状态

- 基线为官方 iStoreOS `istoreos-25.12`，工作分支为 `rt3000/istoreos-25.12`。
- 当前基线提交为 `96c7500f01945e2652ff71478aa74afea5503ccd`；官方 `ath11k-bdencoder` 源码已克隆至 `C:\Users\16547\Desktop\nrf\.codex-tmp\qca-swiss-army-knife`。
- 本轮只读核对 `origin/istoreos-25.12`，远端 HEAD 与本地基线仍相同；没有把其他分支或更新版本混入此副本。
- 移植目标使用单一 UBI 根分区；不保留原厂 `rootfs_1`/`rootfs` 活动槽切换。
- RT3000 DTS、ipq50xx 设备/40 MiB UBI 镜像定义、DSA 网络映射、ART 校准读取和 `rootfs` 升级分支已加入工作区。
- 三份原厂 BDF 已从设备只读导出并通过字节数和 MD5 校验；专用 board-2 容器已生成，记录字段和 variant 已用官方编码器核对。首版启用集成 IPQ5018 和 QCN6122 第一实例；第二个 QCN6122 实例沿用上游默认禁用状态。
- RT3000 镜像包排除 Docker 及 Dockerman 组件；iStore、feeds、普通磁盘挂载和标准升级流程保留。相关 iStoreOS 启动/默认配置脚本按 RT3000 条件跳过 Docker 专属操作。
- 尚未完成固件构建和设备刷写。当前 Windows 环境没有 GNU `make`、`dtc`、Docker，也没有已安装的 WSL Linux 发行版；`wsl --status` 返回退出码 50，报告 WSL 未安装。当前会话不是管理员，查询 Windows 可选功能也返回“需要提升权限”，因此本轮无法启用 WSL；不要把源码静态检查当成构建通过。
- `feeds.conf.default` 中的 packages、luci、routing、telephony、video、store、third 仓库均已克隆到 `feeds/`；`third` 当前浅克隆提交为 `335fa42`。store feed 内已确认有官方 `luci-app-store`，Luci feed 内有 `luci-app-dockerman`。尚未成功运行 `scripts/feeds update`/`install`，`package/feeds/` 尚未生成，因此完整最终包清单尚未解析。
- RT3000 UBI 目标使用 40 MiB `IMAGE_SIZE`、128 MiB NAND 参数、128 KiB 擦除块和 2 KiB 页；上游 `append-ubi` 规则据此对产物执行 37,888 KiB 上限检查（含 FIT 内核）。最终镜像大小尚未实际测量。
- 六个已修改的 shell 脚本分别通过 Git Bash `sh -n`；`git diff --check` 通过。DTS 编译、固件构建和设备验证仍未进行。
- 本轮实机只读检查 `/etc/openwrt_release` 得到 `OpenWrt Chaos Calmer 15.05.1`、目标 `ipq/ipq50xx`。设备有 `/bin/opkg`；已安装包查询没有命中 `istore`、`store` 或 `luci-app` 包名。由此确认实机当前系统不是官方 iStoreOS 25.12；市场页面是否由其他名称的组件提供，单凭这些包名查询不能判断。

## 设备证据

- SoC 为 IPQ5018，内存 256 MiB，NAND 128 MiB；集成射频和两个 QCN6122 射频均在原厂系统工作。
- 原厂 NAND 中两个系统区连续：`rootfs_1` 从 `0x900000` 起、大小 `0x2800000`；`rootfs` 从 `0x3100000` 起、大小 `0x2800000`。运行设备的 `/proc/cmdline` 指向 `ubi.mtd=rootfs`，`/proc/mtd` 显示该 MTD 上挂载 `kernel`、`wifi_fw`、`bt_fw`、`ubi_rootfs`、`rootfs_data` 等 UBI 卷；`mount` 显示系统根只从 `ubi_rootfs` 启动。只读 `fw_printenv boot_part bootcmd` 仅返回 `bootcmd=bootipq`，没有 `boot_part` 变量。目标仅对齐当前 `rootfs` 分区（`0x3100000/0x2800000`，40 MiB），不实现活动槽切换，不把两个分区在 Linux 侧硬合并；`rootfs_1` 不进入镜像或升级目标，避免 Linux 与 U-Boot 对分区起点不一致。
- 后续原厂数据分区：`pdt_data` `0x5900000/0x600000`、`pdt_data_1` `0x5f00000/0x600000`、`exp` `0x6500000/0x100000`、`plugin` `0x6600000/0x1700000`。
- 当前运行态实际挂载还确认 `pdt_data` 和 `plugin` 是 UBI 卷/分区，Wi-Fi 固件来自独立 `wifi_fw` SquashFS 卷。新的 rootfs 镜像只替换标准系统所需的 `rootfs` UBI 内容，不在升级逻辑里删改这些原厂数据分区或引入原厂双槽控制。
- `0:ART` 位于 `0x780000`，大小 `0x100000`。ath11k 校准区偏移和长度：IPQ5018 `0x1000/0x20000`；QCN6122 PCI0 `0x26800/0x20000`；QCN6122 PCI1 `0x4c000/0x20000`。原厂驱动参数中的板号为集成射频 `0x10`、PCI0 `0x50`、PCI1 `0x60`，并有 `skip_radio_bmap=4`。
- 原厂设备树给出的 ESS MAC0 到 PHY 地址 7，MAC1 经 SGMII 到 QCA8337；交换机复位 GPIO 为 26，MDIO GPIO 为 36/37。设备运行配置将 `eth0`、`eth1` 加入 `br-lan`，`switch1` VLAN 使用 LAN 端口 `2、3、4` 和 CPU 端口 `6t`，VLAN 过滤关闭。原厂交换机 LAN 位图是 `0x1e`，但实机 UCI 明确只把端口 `2、3、4` 放进该 VLAN；不要据位图额外臆造 WAN 口。
- iStoreOS 使用 DSA 描述同一拓扑：`eth0` 加入 LAN 桥，QCA8337 的 2/3/4 号口分别命名为 `lan1/lan2/lan3`。`eth1` 是 DSA CPU 主接口，不作为独立 LAN 端口加入桥；端口 `1`、`5` 未出现在原厂运行 VLAN 配置中。ipq50xx 内核配置已将 `CONFIG_NET_DSA_QCA8K=y` 编入内核。
- 原厂复位键 GPIO 34、WPS GPIO 38；绿色 LED GPIO 19。MAC 基地址为 `7c:de:78:a2:c1:c6`。

## 无线启用与 MAC 结论

- 实机 `iw dev` 当前只枚举两张 PHY：集成 2.4 GHz 和一张 5 GHz。结合原厂 `skip_radio_bmap=4`，首版只配置这两张；`wifi2`/第三张射频及 `b60` 校准文件暂不启用，待实机证据确认后再调整。保留 `b60` 原始文件，不删除、不拿它替代其他射频。
- 原厂 MAC 基址为 `7c:de:78:a2:c1:c6`；实机两张 Wi-Fi 的 MAC 分别为基址 `+2` 与 `+3`。DTS/无线脚本应按此偏移生成，不能把基址直接分配给两张无线网卡。

## 无线板级数据结论

- 设备端 `bdwlan.b10`、`qcn6122/bdwlan.b50`、`qcn6122/bdwlan.b60` 各 131072 字节，MD5 分别为 `7860cf626b078ef10a8880dbf8b89ee2`、`b977efcd1a56cae91be00956ff837daf`、`6fe96b4b7d7db5084a50c7f516c89936`。
- 三份原厂文件已导出到 `C:\Users\16547\Desktop\nrf\.codex-tmp\rt3000-bdf-export\`。导出文件大小均为 131072 字节，MD5 与设备端逐块查询值完全相同；文件名分别为 `bdwlan.b10`、`bdwlan.b50`、`bdwlan.b60`。已用这些原始文件生成 RT3000 board-2 容器；保留原始文件，不能用其他机型内容替代。
- 已将这三个原厂文件与官方 `ipq-wifi-glinet_gl-b3000`、`ipq-wifi-cmcc_mr3000d-ci` 的 board-2 容器做 16 KiB 分块比较。数据只在少量区块相同，没有任何一份完整匹配；不能据型号或 SoC 相同就复用这些 BDF。
- 之前的大段 Telnet base64 失败有两个已定位问题：长命令会被 80 列终端拆行，且旧脚本在回显命令中先匹配到结束标记，未等到数据输出。改用短提示符、每次 `dd` 读取 8192 字节、等待 shell 提示符后解析 base64，三份文件均成功并通过完整 MD5 校验。设备 BusyBox `nc` 不支持 `-w`；反向 TFTP 上传也超时，接收端没有收到 UDP 请求。Windows 防火墙启用，尝试添加仅限设备/主机地址的临时规则时返回 `Access is denied`，未创建规则；不能把防火墙拦截记作已证实的唯一原因。串口检查只看到 COM3/4/5/6（蓝牙串口），旧脚本期待的 COM7 不存在。
- 官方 `ath11k-bdencoder` 的 board-2 QMI 匹配字段已确认使用 `qmi-chip-id=0`、`qmi-board-id=255`；b10、b50、b60 已分别写入独立 variant，并与 DTS 中引用一致。容器已通过官方查询工具核对；后续若修改 BDF，才需要重新生成和核对，不要重复调查板号或把裸 BDF 当作 board-2 容器。

### BDF 分块比对明细

- 比对对象及容器 SHA256：GL-B3000 IPQ5018 `685184ace8ab091228c9546ea86e5b8613169c147c2c795bc7a4d7506921ae1c`；GL-B3000 QCN6122 `8d8192d06a06c29c2fc4e7de228040490e7a18edba23c7e27c1090ff1cb91e14`；CMCC MR3000D-CI IPQ5018 `61d81cce410dcc113aee2088bf8c46f481dc1533ec89f93df5c528c4b2360b1a`；CMCC MR3000D-CI QCN6122 `8f810c5ee16afe844e55845c1c5c78337dadc71aaada6623f54b4f063620b934`。
- GL-B3000：原厂 b10 的块 2 在 GL IPQ5018 容器偏移 108 命中；b50 的块 2、b60 的块 2 和块 6 在 GL QCN6122 容器偏移 108 命中。
- CMCC MR3000D-CI：原厂 b10 的块 2 和块 7 在 IPQ5018 容器偏移 116 命中；b50 的块 2 和块 7、b60 的块 2、块 6 和块 7 在 QCN6122 容器偏移 116 命中。
- 这里的命中只是 16 KiB 分块比较中的局部相同数据，不代表板级文件兼容或校准完整。官方 ath11k 源码已有 QCN6122 支持、设备树 `qcom,bdf-addr` 支持和标准校准数据提取脚本；RT3000 专用 board-2 记录现已按原厂 BDF 制作并接入校准路径。

### 取证位置与避免重复

- 原厂设备树导出文件：`C:\Users\16547\Desktop\nrf\.codex-tmp\rt3000-live.dtb`；解析脚本：`C:\Users\16547\Desktop\nrf\.codex-tmp\rt3000_dump_fdt.py`。WorkBuddy 原始分析目录：`C:\Users\16547\WorkBuddy AI\2026-10-20-08-51-15`。
- 连接脚本 `rt_telnet.py` 已有只读 Telnet 参数；不要把凭据复制到记录或输出。设备 SSH 不可用。
- 开发机此前未发现可用 WSL、7-Zip 或 `unsquashfs`。`vols.tar` 内的 `wifi_fw.bin` 为 3588096 字节 SquashFS 镜像，SHA256 `e967434be08cf0c640fbbe131e157c0316da9ff1c4a3c942783c6e0df702177c`；直接扫描常见 BDF 标记未命中，但由于尚未解包，不能据此断言镜像中没有板级文件。串口 COM7 也不存在。
- `pull_rt3000_bdf.py` 已改为按 8192 字节分块经 Telnet 读取并验证哈希，成功产物在 `.codex-tmp/rt3000-bdf-export/`。`rt3000_bdf_tftp_rx.py` 曾启动但没有收到请求，已停止；不必重复 TFTP 尝试。`receive_rt3000_bdf.py` 是此前的 TCP 接收器，设备侧 `nc` 参数限制导致没有成功，不要原样重跑。
- 后续无线故障定位直接检查 ath11k 固件请求、variant 匹配和校准加载日志。不要重复解析容器格式、调查 metadata、跑大段 Telnet 输出、长行命令或已失败的 `nc`/TFTP 传输。

## 工作区改动与验证状态

- `target/linux/qualcommax/files/arch/arm64/boot/dts/qcom/ipq5018-rt3000.dts`：新增板级 DTS，覆盖原厂无线内存区域、MAC/校准地址、按钮/LED、MDIO 与 QCA8337 DSA 连接。
- `target/linux/qualcommax/image/ipq50xx.mk`、`target/linux/qualcommax/ipq50xx/base-files/`：新增设备镜像与单一 `rootfs` UBI 升级目标，按原厂实测布局写入 40 MiB 分区，不实现原厂活动槽切换。
- `package/firmware/ipq-wifi/Makefile`、`package/firmware/ipq-wifi/files/`：注册 RT3000 board-2 包并把本地板文件复制到包构建目录；实际包构建尚未验证。
- `package/istoreos-files/`：只在 RT3000 上跳过 Docker 专属初始化、NAT、关机清理和 `/opt` 路径改写；保留 iStoreOS 应用、feeds、磁盘挂载及其余系统脚本。
- `.cache/firmware_qca-wireless` 是未跟踪的上游缓存，必须保留。
- 已检查原厂 FDT、设备只读运行配置和上游 ipq50xx/QCA8K 实现。源码差异检查 `git diff --check` 曾通过；固件和 DTB 均未编译，设备未经刷写。
- 当前设备只读 Telnet 参数来自 WorkBuddy 会话目录下已有脚本；不要把口令复制进记录、源码或答复。SSH 不可用。

## iStoreOS 默认包与远端构建记录（2026-10-21）

- 官方基线源码 `include/target.mk` 使用 `DEFAULT_PACKAGES += luci-app-store` 加入 iStore。`make defconfig` 后 `.config` 不一定保留 `CONFIG_PACKAGE_luci-app-store=y` 显式行；应验证官方默认包规则和最终镜像包清单，不能把显式行缺失误判为 iStore 未启用。
- 官方 `store` feed 中 `luci-app-store` 依赖 `luci-lib-taskd`、`luci-lib-xterm` 和 `taskd`。这些是 iStore 功能依赖，不应为精简 Docker 而移除。
- RT3000 设备定义排除 `docker`、`dockerd`、`docker-compose`、`containerd`、`runc`、`luci-app-dockerman` 和 Dockerman 中文包；`.config` 中这些项目均未设为 `y`。Docker 专属初始化、NAT、关机清理和 `/opt` 路径改写仅对 RT3000 跳过，iStoreOS 其余实现保留。
- GitHub Actions 曾因错误要求 `.config` 显式包含 iStore 包而在配置阶段退出。已修正为核验 RT3000 profile、官方 iStore 默认规则、设备包排除声明和 Docker 包未启用。提交 `39148ea` 的运行 `39451706224` 已通过配置阶段并进入固件构建；最终结果和镜像产物待该 run 结束后确认。
- Linux 构建主机 `192.168.2.157` 的容器 `rt3000-build-20261020` 正在 `/work` 执行 `FORCE_UNSAFE_CONFIGURE=1 make -j2`，日志 `/tmp/rt3000-build2.log`。当前已推进至 `toolchain/kernel-headers` 解包；退出码文件 `/tmp/rt3000-build2.exit` 尚不存在，固件目录尚无产物。最近磁盘余量约 7.4 GiB，继续监看空间和进程，不要据运行中状态声称成功。

不要把原厂活动槽作为移植设计约束，也不要在未经用户明确要求时刷写设备。

## 下一步

1. 等待 Linux 构建退出码或 GitHub Actions run `39451706224` 完成；失败时依据对应日志定位后继续，不重置 Linux 的已编译工具链。
2. 构建完成后核验目标镜像体积不超过 40 MiB 分区布局、FIT 配置与 UBI 卷名，并检查最终镜像包清单：保留官方 iStore、存储和其他默认组件，只排除 Docker/Dockerman。
3. 设备尚未刷写。后续若验证升级，目标仅写 `rootfs` (`0x3100000/0x2800000`)，不更改 `rootfs_1` 或 U-Boot 环境；未经用户明确要求不要刷机。
4. 无需重复原厂 BDF 导出、失败的 TFTP/`nc` 传输、board-2 metadata 调查或活动槽调查。已有原始数据和编码器位置见上文。
