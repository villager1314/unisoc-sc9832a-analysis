# BF T61 SC9832A 分区归档清单

## 公开固件归档

文件：`BF-T61-SC9832A-public-partitions.7z.001`

SHA-256：`4B3C041E4A5492B9AB9946D4E9DE50DE315AA1B9E1CF2D0F815D9A3E69AA672A`

包含：

```text
splloader.img  uboot.img  boot.img  recovery.img
system.img  oem.img
l_modem.img  l_warm.img  l_gdsp.img  l_ldsp.img  pm_sys.img
wcnfdl.img  wcnmodem.img
logo.img  fbootlogo.img
```

这是 7-Zip 分卷格式；虽然当前压缩后只有一个 `.001` 卷，也应从 `.001` 文件打开或解压。解压后原始数据约 2.30 GiB。

## 设备私有 NV 归档

文件：`BF-T61-SC9832A-device-private-NV.7z`

SHA-256：`04ED6F622D30B48AF14A522861ECA000D9F344CC5C801859E6436B5C68999672`

包含：

```text
prodnv.img  miscdata.img  persist.img  misc.img
l_fixnv1.img  l_fixnv2.img
l_runtimenv1.img  l_runtimenv2.img
```

该归档使用 7-Zip AES-256 并加密文件名。密码不写入公开仓库或 Release 说明。上述分区可能包含设备唯一标识、MAC、射频校准和运行时 NV；不能刷入其他设备。

## 未包含

- `userdata`：按研究约定没有读取；
- `cache`：按研究约定没有读取；
- eMMC `boot0`、`boot1`、RPMB：不属于本次普通逻辑分区备份。

## 恢复注意事项

1. 先核对目标必须是贝尔丰 BF T61、`sp9832a_3h10cmcc`；
2. 核对镜像长度和 `checksums/SHA256SUMS.md`；
3. 不要把其他设备的 NV/Persist/Miscdata 写入本机；
4. 在没有匹配 FDL、加载地址和可用恢复路径前，不写 SPL/U-Boot；
5. Boot/Recovery/SPL/U-Boot 带签名或 VLR，修改后不能假设只重算普通哈希即可启动。

