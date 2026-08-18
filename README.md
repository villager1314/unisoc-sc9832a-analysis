# 展讯 SC9832A 手机 Root 研究心路历程

在一台展讯平台 Android 5.1 手机上，我把常用一键 Root、漏洞型正规 Root、Dirty Cow、需要偏移量的内核提权、Bootloader 解锁、深刷/下载模式这几条路从浅到深全部走了一遍。下面按时间线还原每一层的尝试、失败现象与根因，并附上涉及到的每一款工具与漏洞的出处。

## 手机型号、处理器等信息

| 项目 | 信息 |
| --- | --- |
| 平台 | 展讯 / 紫光展锐（Spreadtrum / Unisoc） |
| 型号标识 | `sp9832a_3h10cmcc` |
| 处理器 | SC9832A，4 × ARM Cortex-A7 @ 1.3–1.5 GHz，Mali-400MP2，28nm 制程 |
| CPU 架构 | `armv7l`（32 位 ARM） |
| 内核版本 | Linux `3.10.65` |
| 系统版本 | Android 5.1 |
| SELinux | enforcing |
| 默认 shell | 未提权的 `shell` 用户 |

SC9832A 是展讯面向入门级 4G 手机的四核芯片 [SC9832A 规格](https://devicebeast.com/processor/spreadtrum-sc9832a)。这台机器纸面上"内核很老、漏洞很多"，实际却因厂商定制而处处封闭，是后续所有尝试接连受挫的大背景。

## 第一层：市面上常见的一键 Root 工具

先从最省事的一键 Root 工具入手，它们的共同思路是把 `su` 与授权管理程序写进 `/system` 分区，靠自动化脚本完成提权。结果全部失败：要么直接报"不支持"，要么提示成功但重启后 root 丢失、授权管理消失。

- **360 一键 Root / 360 超级 Root**：国产一键 Root 代表，借助修改 `install-recovery.sh`、利用 init 阶段不受 SELinux 约束的窗口完成提权并写入授权模块 [360 一键 Root 官方说明](http://shouji.360.cn/root/relief.html)。这种"改 `/system`"的老方案在厂商深度定制的 ROM 上成功率低，版本越新越容易失败 [360 Root 原理](https://blog.csdn.net/hu3167343/article/details/43311779)。
- **百度一键 Root**：同样是把 `su` 与授权管理写进 `/system`，依赖的漏洞主要是 `zergRush`（Android 2.2–2.3.6）、`Gingerbreak`、`psneuter` 这类上古漏洞 [百度一键 Root 原理](https://m.duote.com/android/962643.html)，覆盖面仅到 Android 4.4 上下，对 3.10 内核 + Android 5.1 基本失效。
- **KingRoot**：知名一键 Root，授权管理端叫 KingUser。其主要利用集中在 Android 5/6 及更早的旧漏洞，之后成功率断崖式下降 [KingRoot FAQ](https://kingroot.ru/faq.html)。
- **KingoRoot**：分 PC 版与 APK 版，同样靠打包系统漏洞取 root。官方 FAQ 明确两个排除条件：Android 5.1 以上"暂不支持"，以及"Bootloader 被厂商锁定就无能为力" [KingoRoot FAQ](https://www.kingoapp.com/faq.htm)。

这一层的共同死因：旧漏洞已在这台内核补丁等级上闭合、`/system` 受只读或 dm-verity 保护导致 `su` 写不进去、Bootloader 被锁定，即使重启也保不住 root。

## 第二层：漏洞型"正规" Root 工具

一键工具失效后，转向直接封装内核漏洞利用链的正规工具，结果卡在同一个门槛——**补丁等级太高**。

- **Framaroot**：纯 APK 一键 Root，内置多个以《指环王》角色命名的漏洞利用（Gandalf、Boromir、Pippin、Legolas、Aragorn、Sam、Frodo、Gimli），分别对应不同内核与机型 [Framaroot 漏洞列表](https://framaroot.en.malavida.com/android/)。它面向的大多是 2014 年前后的 Android 4.x 内核，放到 3.10.65 上已无适用 exploit。
- **PowerRoot**：与 Framaroot 同类的漏洞注入式一键 Root 工具，公开资料极少，其依赖的漏洞链早已被高版本补丁堵死，本机同样不适用。
- **PingPongRoot**：对应 [CVE-2015-3636](https://pwnies.com/pingpongroot-cve-2015-3636/)，是 Keen Team 提交、Black Hat USA 2015 公开的内核 Use-After-Free 提权漏洞，曾获 Pwnie Awards 2015 "最佳提权漏洞"，可绕过 PXN [PingPongRoot exploit 仓库](https://github.com/android-rooting-tools/libpingpong_exploit)。它的支持范围止于部分 Android ≥4.3 设备，且官方修复随 2015 年内核补丁下发，这台"补丁等级更高"的机器已修掉它。

## 第三层：Dirty Cow 与"设备已被修复"的确认

随后转向当时最有希望的内核漏洞 **Dirty Cow（CVE-2016-5195）**：Linux 2.6.22–4.8.3 的写时复制竞态漏洞，通过 `madvise(MADV_DONTNEED)` 与 `/proc/self/mem` 写入的竞态，覆盖本应只读的内存映射，进而篡改只读系统文件。

先编译运行 [timwr/CVE-2016-5195](https://github.com/timwr/CVE-2016-5195)，用 `dirtycow` 覆盖 `/system/bin/run-as`，期望篡改后的 `run-as` 执行 `setresuid(0,0,0)` 提权。又试了思路更完整的 [GetRoot-Android-DirtyCow](https://github.com/timwr/CVE-2016-5195)，替换 `run-as` 后导出并改写 SELinux 策略与 `init`、重载策略挂载 `su` 镜像。

两个方案的共同结果是**无法修改目标文件**：替换程序跑完后手动执行 `run-as -s2`，返回 `Package '-s2' is unknown`，说明 `run-as` 仍是原始版本、从未被真正写入。

为了确认是"竞态没赢"还是"内核已被封堵"，下载了 **DirtyCow Checker** 这款 app（免 root、直接探测当前内核对 Dirty Cow 的脆弱性，输出 "vulnerable / not vulnerable"）[DirtyCow Checker XDA](https://xdaforums.com/t/dirtycow-checker-app-2-3.3585546/)。检测结论指向：内核已包含 CVE-2016-5195 的修复，**设备已被（厂商以保持版本号不变的反向移植方式）修复**，漏洞实际上已经闭合。

结合 `dmesg` 还能看到第二个独立障碍：CPU1 几乎全程热插拔关闭，"唤醒 cpu1"后不到十分之一秒就 `CPU1 shutdown`，只有 CPU0 持续工作。Dirty Cow 依赖两个线程在不同核心上真正并行，核心被关掉后攻击退化成单核串行轮转，命中窗口不复存在。同时，SELinux 拦截与 CPU 架构不匹配这两个常见嫌疑也被日志和 `armv7l` 信息排除。一条清晰的因果链由此成立：**目标文件改不动 → Checker 判定已修复 → 外加 CPU1 关闭，竞态无从成形**。

## 第四层：需要设备偏移量的内核提权

Dirty Cow 封死后，沿着 Android 历史提权漏洞清单继续排查 iovyroot 与 Bad Binder，结果卡在同一个门槛——**需要具体机型的绝对内核地址/偏移量**。

- **iovyroot（CVE-2015-1805）**：`fs/pipe.c` 中 pipe 读写未同步造成的 I/O vector 数组越界（`iov array overrun`），影响 Linux 内核 3.16 之前，可造成任意内核内存写 [CVE-2015-1805 分析](https://www.anquanke.com/post/id/83682)。它的 exploit 需要为每台设备预填一组绝对内核地址：`ptmx_fops` 的 `fsync`/`check_flags`、`sidtab`、`policydb`、`selinux_enabled`、`selinux_enforcing` 等 [iovyroot 仓库](https://github.com/dosomder/iovyroot)。这些偏移量只能从对应固件的内核镜像里用 `kallsyms`/IDA Pro 逐项提取。
- **Bad Binder（CVE-2019-2215）**：Android Binder 驱动里的 Use-After-Free，可本地提权到内核 [NVD CVE-2019-2215](https://nvd.nist.gov/vuln/detail/CVE-2019-2215)。它主要针对 Android 8.x 及以上、3.18/4.4 等较新内核，且公开 exploit 同样需要按具体机型、内核编译调整偏移与内存布局。

对这台 32 位、Android 5.1、又拿不到公开固件包的展讯机器来说，这是双重不匹配：要么系统版本不符，要么缺偏移量，而提取偏移量本身又卡在"拿不到固件"上——这条线索直接引出后面的底层路线。

## 第五层：展讯深层漏洞解锁 Bootloader

运行期提权全部走不通后，改走"从底层解除校验"的路线：解锁 Bootloader，去掉厂商签名固件校验，再刷入自制 `boot`/`system` 镜像。

展讯官方解锁有硬门槛：需要拿设备序列号生成 token，签名后通过 `fastboot flashing unlock_bootloader signature.bin` 发回才能触发 unlock，能否解锁取决于签名私钥是否匹配手机内置 RSA 公钥，冷门机型通常拿不到这条通路 [展讯解锁原理](https://m.bilibili.com/opus/1080829284600250424)。

于是转向展讯的深层 BootROM 漏洞：针对 Unisoc BootROM 的 [CVE-2022-38694](https://github.com/TomKing062/CVE-2022-38694_unlock_bootloader) 是一个任意地址写（AAW）漏洞，攻击者可在签名校验完成前覆写函数指针、以 BootROM 权限执行代码，从而绕过校验解锁 Bootloader [NCC Group 分析](https://mutur4.github.io/2026/03/19/cve-2022-38694.html)。这类漏洞本质上是"针对具体芯片批次与 BootROM 版本"的精细适配产物，对这台 SC9832A 没有现成适配，最终**因没有适配的机型而失败**。

## 第六层：深刷模式 BROM 与下载模式 FDL2

最后一层是直接进入底层刷机/下载模式，把固件（尤其是 FDL 加载器）提取出来，为前面的偏移量分析和自制镜像铺路。

- **SPD 下载模式 / SPD3**：展讯的下载（烧录）模式由"USB 连接 + 特定按键组合"触发并维持，PC 端 ResearchDownload / SPD 工具需要依次加载 `fdl1.bin`、`fdl2.bin` 两个加载器，再由它们接管存储读写完成刷写或导出，FDL 文件本身藏在厂商 `.pac` 固件包里 [FDL 提取方法](https://github.com/emtee40/spreadtrum_flash) [ResearchDownload 用法](https://flashguidehub.com/research-download/)。
- **BROM 深刷模式**：对应 BootROM 阶段的底层下载入口。典型流程是先用 `spd_dump` 进入 brom，再 `fdl fdl1-dl.bin`、`fdl fdl2 ...`、`exec` 逐级拉起 [个人开发记录](https://www.getce.cn/show/252.html)。

这台机器的现实是：**相关进入流程被厂商阉割**，既无法稳定进入 BROM/深刷模式，也拿不到匹配的 FDL1/FDL2 加载器，导致"提取固件 → 计算偏移量 → 自制可刷镜像"这条最后的通路从入口处就被掐断，**无法提取所需固件**。

## 总结

一层层推进下来，这台机器的每一扇门都被关上：一键 Root 因旧漏洞闭合与 `/system` 保护失效，正规漏洞工具因补丁太高失效，Dirty Cow 因内核已修复 + CPU 热插拔失效，偏移型提权因缺固件无偏移量失效，Bootloader 解锁因无适配机型失效，深刷/下载模式因进入流程被阉割失效。

最核心的一课是"纸面条件"与"实际条件"的差距：内核 3.10.65 低于 Dirty Cow 官方修复线 3.10.73，`armv7l` 架构也与利用程序完全匹配，从版本号看处处"有戏"，却处处落空。国产厂商"保持版本号不变的反向移植补丁"与"阉割进入流程"的定制，让所有依赖公开版本号判断的攻击都失效。而在基于竞态的提权里，"脚本显示成功"与"真正成功"是两回事——任何不做结果校验的自动化流程，都会把失败包装成成功。
