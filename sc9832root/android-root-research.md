# Android 5.1 提权研究记录

> 研究时间：2026-08-18  
> 目标设备：Android 5.1 (ARM 32-bit)  
> 研究目标：分析并尝试利用已知漏洞获取 root 权限

---

## 一、iovyroot (CVE-2015-1805) 偏移量分析

### 1.1 仓库信息

- 仓库地址：[dosomder/iovyroot](https://github.com/dosomder/iovyroot)
- 漏洞编号：CVE-2015-1805
- 支持架构：32 位和 64 位，需要绝对内核地址

### 1.2 偏移量结构体

```c
struct offsets {
    char* devname;          // ro.product.model 设备型号
    char* kernelver;        // /proc/version 内核版本字符串
    union {
        void* fsync;        // 32位: ptmx_fops -> fsync
        void* check_flags;  // 64位: ptmx_fops -> check_flags
    };
    void* joploc;           // JOP gadget 地址 (仅64位)
    void* jopret;           // check_flags() 返回地址 (仅64位)
    void* sidtab;           // SELinux sidtab
    void* policydb;         // SELinux policydb
    void* selinux_enabled;
    void* selinux_enforcing;
};
```

### 1.3 偏移量提取方法

每个偏移量都是**绝对内核地址**（如 `0xffffffc0xxxxxxxx`），需要针对具体设备型号和固件版本逐一提取。

| 偏移量 | 提取方法 | 工具 |
|--------|---------|------|
| `check_flags` / `fsync` | 从 kallsyms 获取 `ptmx_fops` 基址，加上 `20 * sizeof(void*)` 或 `14 * sizeof(void*)` 偏移 | kallsymsprint |
| `joploc` (64位) | 在 IDA 中搜索 ARM64 特征码（LDR + ADD + BLR 模式） | IDA Pro |
| `jopret` (64位) | 从 kallsyms 获取 `sys_fcntl` 地址，在 IDA 中定位 `BLR X1` 后面的 `SBFM` 指令地址 | IDA Pro + kallsymsprint |
| `sidtab` / `policydb` 等 | 直接从 kallsyms 符号表获取 | kallsymsprint |

### 1.4 整体流程

```
固件 → 提取内核镜像 → kallsymsprint/IDA Pro 分析 → 写入 offsets.c → 编译 → 运行
```

---

## 二、刷入浏览器

### 2.1 最后一个支持 Android 5.1 的 Firefox

| 项目 | 信息 |
|------|------|
| 版本 | **Firefox 143.0.4** |
| 内部代号 | Fenix |
| ARM 32 位文件名 | `fenix-143.0.4-android-armeabi-v7a.apk` |
| 下载地址 | `https://archive.mozilla.org/pub/fenix/releases/143.0.4/android/` |

### 2.2 关键背景

- Mozilla 官方宣布 Firefox 144 起最低 Android 版本要求提升至 **Android 8.0 (Oreo)**
- 32 位 **x86** 设备同时停止支持，32 位 ARM 继续支持
- Firefox 143.0.4 将不再收到安全更新

---

## 三、CVE-2016-5195 (Dirty COW) 分析与编译

### 3.1 仓库信息

- 仓库地址：[timwr/CVE-2016-5195](https://github.com/timwr/CVE-2016-5195)
- 漏洞编号：CVE-2016-5195 (Dirty COW)
- 平台：Android
- 影响范围：Linux 内核 2.6.22 ~ 4.8.3（2016年10月前）

### 3.2 编译产物

| 产物 | 源代码 | 功能 |
|------|--------|------|
| `dirtycow` | `dirtycow.c` + `dcow.c` | 核心利用程序，利用 Dirty COW 竞争条件覆盖只读文件 |
| `run-as` | `run-as.c` | 恶意 payload，被覆盖后执行 `setresuid(0,0,0)` 提权到 root |

### 3.3 工作原理

1. `dirtycow` 通过 `madvise(MADV_DONTNEED)` + `/proc/self/mem` 写入的竞争条件，覆盖 `/system/bin/run-as`
2. 执行被篡改的 `run-as`，获取 root shell
3. 注意：该工具**不会禁用 SELinux**，不会安装 SuperSU

### 3.4 编译环境

- 编译工具：Android NDK r21e
- 编译命令：`ndk-build NDK_PROJECT_PATH=. APP_BUILD_SCRIPT=./Android.mk APP_ABI=<arch> APP_PLATFORM=android-22`

### 3.5 编译结果

| 架构 | 适用设备 | dirtycow | run-as |
|------|---------|----------|--------|
| armeabi-v7a | 32位 ARM | 9.6 KB | 5.6 KB |
| arm64-v8a | 64位 ARM | 14 KB | 10 KB |
| x86 | 32位 Intel | 14 KB | 5.6 KB |

---

## 四、实际利用尝试

### 4.1 操作步骤

```bash
# 1. 推送二进制到设备
adb push libs/armeabi-v7a/dirtycow /data/local/tmp/dirtycow
adb push libs/armeabi-v7a/run-as   /data/local/tmp/run-as

# 2. 执行利用
adb shell 'chmod 777 /data/local/tmp/dirtycow'
adb shell '/data/local/tmp/dirtycow /data/local/tmp/run-as /system/bin/run-as --no-pad'

# 3. 尝试获取 root
adb shell /system/bin/run-as
```

### 4.2 运行结果

```
[*] using /proc/self/mem method
[*] madvise thread starts, address 0xb6ed3000, size 5708
[*] check thread starts, address 0xb6ed3000, size 5708
[*] check thread stops, timeout, iterations 1000    ← 竞争条件未命中
[*] madvise thread stops, return code sum 0, iterations 39367422
[*] /proc/self/mem -282853872 1455340
[*] finished pid=0 sees 0xb6ed3000=464c457f
```

执行后 `run-as` 仍显示原始行为 `Usage: run-as <package-name>`，说明覆盖失败。

### 4.3 失败原因分析

| 可能原因 | 说明 |
|---------|------|
| 内核已打补丁 | 设备内核可能已包含 CVE-2016-5195 的修复 |
| 竞争条件未命中 | `/proc/self/mem` 路径竞争是概率性的，需要多次尝试 |
| dm-verity 保护 | `/system` 分区可能启用了完整性校验 |

### 4.4 后续建议

1. 确认内核版本：`adb shell "uname -a; cat /proc/version"`
2. 多次重试（race condition 本质是概率性的）
3. 尝试 ptrace 替代路径
4. 检查 dm-verity：`adb shell mount | grep system`

---

## 五、研究总结

| 漏洞 | 编号 | 原理 | 状态 |
|------|------|------|------|
| iovyroot | CVE-2015-1805 | 内核 pipe 缓冲区溢出 | 已分析偏移量提取方法，尚未实际操作 |
| Dirty COW | CVE-2016-5195 | 写时复制竞争条件 | 已编译，实际利用失败（竞争条件未命中） |

### 关键发现

1. **iovyroot** 需要针对每个设备型号和固件版本手动提取内核绝对地址，适合有 IDA 逆向经验的场景
2. **Dirty COW** 利用门槛较低，但竞争条件具有概率性，需要在合适的内核版本上多次尝试
3. Android 5.1 设备的安全生态已基本停止更新，Firefox 143.0.4 是最后的兼容浏览器版本