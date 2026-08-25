# UNISOC/Spreadtrum SC9832A（贝尔丰 BF T61）研究记录

本仓库记录一台贝尔丰 `BF T61 / sp9832a_3h10cmcc` 的固件识别、展讯下载链、分区备份、启动校验、FactoryTest/Calibration 工程模式、Root 路线和 Wi‑Fi 故障定位过程。内容以实机读取和离线反编译结果为准。

> **风险提示**：SC9832A 不同板级的 FDL、内存地址、分区表和签名链不能混用。写错 `splloader`、`uboot`、NV 或校准分区可能造成无基带、丢失设备标识或永久黑砖。刷写前应至少备份 SPL/U-Boot、Boot/Recovery、NV、Persist、Miscdata 和分区表，并校验哈希。

## 目标设备

| 项目 | 实测值 |
| --- | --- |
| 品牌/型号 | 贝尔丰 BF T61 |
| 产品/设备 | `sp9832a_3h10_5mvoltecmcc` / `sp9832a_3h10cmcc` |
| SoC | Spreadtrum SC9832A，ARMv7 32 位 |
| Android | 5.1 |
| 内核 | Linux 3.10.65 |
| 启动方案 | Legacy Android boot image，非 A/B |
| 下载模式 | SPRD3/BROM、SPRD4/AutoD |
| SELinux | Enforcing（正常系统） |

## 研究历程

### 1. 识别设备和板级

早期仅凭 SC9832A、Android 5.1 和 Linux 3.10.65 搜索通用刷机包，无法确认板级。ADB、USB 枚举和实机分区最终确认设备为贝尔丰 BF T61，产品标识为：

```text
sp9832a_3h10_5mvoltecmcc
sp9832a_3h10cmcc
```

同为 SC9832A 的 BF T66、`SP9832A_3H10_VOLTE` 或其他 `3h10` 固件也不能因此直接混刷，LCD、触摸、射频、存储布局和签名配置仍可能不同。

### 2. Android 运行期 Root 尝试

先后研究或测试了 Framaroot、z4root、SuperOneClick/zergRush、Dirty COW、KingRoot 及若干旧式一键 Root：

- Framaroot 判断设备不适配其内置漏洞；
- z4root 在执行临时/永久 Root 时崩溃；
- zergRush 能识别 Android 2.2/2.3 路线，但在本机失败；
- DirtyCow Checker 与多个 PoC 均未形成可验证的文件替换；
- 普通 `adb shell` 中 `su` 返回 `permission denied`。

这些结果只证明相应实现没有在本固件上形成可靠利用链，不能仅凭内核版本号断言某个漏洞必然存在或已修复。

### 3. 展讯下载链和分区备份

实机可以进入 SPRD3/BROM 与 SPRD4/AutoD。通用或其他机型的 FDL 即使能上传 FDL1，也可能在切换 FDL2 时超时；必须同时匹配芯片代际、DDR 初始化、存储控制器和加载地址。

通过已工作的下载链完成了关键分区读取，并保存 SHA-256。当前备份覆盖：

```text
splloader  uboot  boot  recovery  system  oem
prodnv  miscdata  persist  misc
l_fixnv1  l_fixnv2  l_runtimenv1  l_runtimenv2
l_modem  l_warm  l_gdsp  l_ldsp  pm_sys
wcnfdl  wcnmodem  logo  fbootlogo
```

按约定没有备份 `userdata` 和 `cache`。普通逻辑分区备份也不等于已经备份 eMMC `boot0`、`boot1` 和 RPMB。

### 4. 启动链与签名校验

对 SPL、U-Boot、Boot 和 Recovery 做字符串、引用和 Ghidra 反编译后，得到以下结论：

1. SPL 会校验后续 U-Boot 镜像，镜像带有展讯 VLR/RSA 结构；
2. U-Boot 在正常启动路径校验 Boot/Recovery；
3. 本机不是 AVB 设备，没有发现 Android Verified Boot 2.0 元数据；
4. `secure_verify_system` 位于 AutoD/BSL 下载写入路径，不是正常 Android 挂载 `/system` 时的 dm-verity；
5. 标准 AutoD 写入修改后的裸 `system.img` 仍可能因下载协议的 secure image 包装校验失败。

因此，“System 正常启动时没有 dm-verity”和“下载工具允许任意写 System”是两件不同的事。修改 Boot/Recovery 也不能只重算普通 SHA-1：若启动链强制 RSA 校验，必须保留有效签名或拥有匹配的 OEM 私钥。

### 5. FactoryTest 模式

U-Boot 的实机按键映射为：

- 音量上 + 电源：FactoryTest；
- 音量下 + 电源：Recovery。

FactoryTest 仍启动 Recovery 内核/ramdisk，但设置 `ro.bootmode=factorytest`。该路径先挂载 `/data` 再启动 adbd，所以 recovery adbd 能读取 `/data/misc/adb/adb_keys` 并完成认证。

`/system/bin/factorytest` 本身以 UID 0 和完整 capabilities 运行，但它是展讯 MMI 硬件测试界面。其 `system()` 调用主要是固定的 Wi-Fi、蓝牙、音频和日志命令，没有发现公开的任意命令 socket。ADB shell 仍是 UID 2000，`adb root` 会被 production build 拒绝。

### 6. ENGPC、cmd_services 与 DIAG

`cmd_services` 是一个很小的本地命令服务：监听 Android abstract socket `cmd_skt`，读取命令，附加 `2>&1`，通过 `popen()` 执行并返回输出。反编译中没有发现认证或命令白名单；如果它由 init 以 root 启动且 shell 可连接，它就是直接的 root 命令桥。

但 FactoryTest 环境中的实际状态是：

```text
persist.sys.cmdservice.enable=disable
persist.sys.engpc.disable=1
```

FactoryTest 没有启动 `cmd_services`，也不存在 `@cmd_skt`。`engpc` 属于 recovery init 的 `class cali`，而 FactoryTest 只启动 `class factorytest`，因此两种模式不能混为一谈。

正常 Android 中打开 Wi-Fi EUT 后曾观察到 `persist.sys.cmdservice.enable=enable`。后续验证重点应是此时是否同时出现 root 身份的 `/system/bin/cmd_services` 和 `@cmd_skt`。

Calibration 模式由 U-Boot 在开机早期与 PC 校准工具通过 USB/UART 握手触发，不是已确认的独立按键组合。ENGPC 是二进制 DIAG 路由器，不是文本 shell；其中包含 NV、寄存器和 efuse 写操作，不应盲目探测。

### 7. Wi-Fi 故障观察

Wi-Fi 曾出现开关失败、重启后仍不可用，但断电并静置较长时间后偶尔恢复。FactoryTest 的 Wi-Fi 测试会主动执行：

```text
rmmod sprdwl
insmod /system/lib/modules/sprdwl.ko
wpa_supplicant ...
```

因此排障需要同时记录 `dmesg`、模块状态、`wlan0`、固件加载、WCN 状态与 `/data/misc/wifi` 内容。间歇性断电恢复更像 WCN/供电/复位状态卡死或硬件接触问题，不能仅凭 Android 设置界面判断是 ROM 缺文件。

## 已确认结论

- 已识别准确机型和板级标识；
- 已进入并验证 SPRD3、SPRD4、Recovery 和 FactoryTest；
- 已读取除 `userdata`、`cache` 之外的主要逻辑分区；
- 已确认 SPL → U-Boot 及 U-Boot → Boot/Recovery 的签名校验路径；
- 已确认非 AVB、System 正常挂载路径未见 dm-verity；
- 已反编译 FactoryTest 与 `cmd_services`，确认二者不是同一服务；
- 尚未获得可复现、可持久化且不会破坏签名链的 Root 方案。

## 仓库内容

- `README.md`：完整研究历程与当前结论；
- `docs/`：启动校验、FactoryTest、ENGPC/cmdservice 和诊断记录；
- `checksums/`：公开镜像及本地备份的 SHA-256；
- `release-manifest/`：Release 归档清单、分区范围和恢复注意事项；
- `sc9832root/`：早期研究材料；其中第三方 APK/压缩包不代表本仓库对其安全性或适配性的认可。

## 分区镜像发布说明

镜像体积超过 GitHub 普通 Git 对象 100 MB 限制，因此使用 GitHub Release 发布已经校验的分区归档。`userdata` 与 `cache` 未读取，不在归档内。

NV、Persist、Miscdata 等分区可能包含 IMEI、序列号、Wi-Fi/蓝牙 MAC、射频校准和其他设备唯一数据，不能以明文公开。它们位于单独的 AES-256 加密包中，密码不写入公开仓库；刷写前必须核对机型、分区名、长度和 SHA-256。

## 许可

本仓库原创文档和脚本采用 [MIT License](LICENSE)。设备固件、驱动、第三方工具和第三方项目仍归各自权利人所有，不因收录校验值或研究记录而改变其许可。

