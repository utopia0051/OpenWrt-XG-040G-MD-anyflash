# XG-040G-MD 任意闪存容量 OpenWrt 补丁集

本补丁集让 Nokia XG-040G-MD（Airoha AN7581）的 OpenWrt 官方 UBI 版本不再把
SPI-NAND 容量写死成 256 MiB，换成 128M / 256M / 512M / 1G 的颗粒都能直接用同一份
固件，同时补上 U-Boot 里缺失的复旦微 FM25G01B / FM25G02B 识别。

基线：OpenWrt main（验证提交 `ee6ef8d27ee0276d1d5405dcdafdb630266470ba`），
uboot-airoha 2026.07，arm-trusted-firmware-airoha 2.10。

## 补丁清单

### `patches/openwrt/001-airoha-xg-040g-ubi-parts-autosize.patch`

改 `target/linux/airoha/dts/an758x-nokia_xg-040g-ubi-parts.dtsi`：
把 `all_flash` 的 `0x10000000` 和 `ubi` 的 `0xffe0000` 都改成 `0x0`。
`fixed-partitions` 里 size 为 0 表示 `MTDPART_SIZ_FULL`，Linux 的
`drivers/mtd/mtdpart.c` 会自动算成「芯片总容量 - 起始偏移」。同一 target 里的
`an7581-evb.dts` 早就在用这个写法，不是新玩法。

### `patches/uboot-airoha/403-mtd-spinand-fmsh-add-support-for-FM25G01B-FM25G02B.patch`

U-Boot 的 `drivers/mtd/nand/spi/fmsh.c` 只认 FM25S01A，装了 FM25G02B 的机器
BL2 能起来、内核能认，但 U-Boot 探测不到闪存。这里把内核侧那份补丁
（OpenWrt `target/linux/generic/pending-6.18/401-mtd-spinand-fmsh-add-support-for-FM25G0102B.patch`）
移植过去，注意 U-Boot 的 ooblayout 成员叫 `.rfree` 而不是 `.free`。

### `patches/uboot-airoha/404-nokia-xg-040g-md-autosize-ubi-partition.patch`

同样把 U-Boot 内置 DTS 里 `ubi` 分区的 `0x20000 0xffe0000` 改成 `0x20000 0x0`，
`add_mtd_partitions_of()` 会按实际芯片容量展开。默认环境里
`ubi create rootfs_data - dynamic`、`ubi create fit $filesize dynamic` 都是相对
写法，不需要额外改动。

## 各层的容量来源

- BL2（ATF `spi_nand_flash_table.c`）：容量来自内建颗粒表，已包含
  FM25G01B / FM25G02B / FM25G02 / S35ML02G3 / W25N01GV / W25N01KV / W25N02KV /
  W25N04KV / MX35LF1G/2G / GD5F1G/2G/4G 等。换的颗粒必须在这张表里，否则
  BL2 起不来，这一层不是靠 DTS 能救的。
- U-Boot：靠 spinand 驱动探测 + 上面两个补丁。
- 内核：spinand 驱动探测 + 分区 size 0 自动展开。FM25G01B/02B 官方 main 已经带了。

## 已知限制

- 只对 UBI 版本（刷 `preloader.bin` + `bl31-uboot.fip`）有效。
  刷 `tcboot.bin` 的原厂布局版本用的是 `an758x-nokia_xg-040g-stock-parts.dtsi`，
  里面 `nsb_1` / `nsb_2` / `config` / `data` / `log` 的偏移是原厂 bootloader 写死的，
  换容量必须同步改原厂 bootloader 的分区表，靠改 DTS 解决不了。想换颗粒就走 UBI 路线。
- 换 SkyHigh S35ML02G300 偶发读超时的话，可以再叠一个 spinand 读等待的
  workaround（社区里有现成实现，判断 `spinand->id.data[0] == 0x01` 后走轮询等待）。
  这个补丁没有收进来，因为它依赖具体内核版本的行号，需要按当时的 core.c 重新对齐。
- 4 KiB 页 / 256 KiB block 的颗粒（比如某些 4Gbit 型号）除了容量还会改 `BLOCKSIZE`
  和 `PAGESIZE`，要同步改 `target/linux/airoha/image/an7581.mk` 里
  `Device/nokia_xg-040g-md-common` 的这两个值。2 KiB 页 / 128 KiB block 的颗粒
  不用动。

## 怎么打补丁

```sh
git clone https://github.com/openwrt/openwrt.git
cd openwrt
patch -p1 < /path/to/patches/openwrt/001-airoha-xg-040g-ubi-parts-autosize.patch
cp /path/to/patches/uboot-airoha/40[34]-*.patch package/boot/uboot-airoha/patches/
./scripts/feeds update -a && ./scripts/feeds install -a
```

然后在 `make menuconfig` 里选 Target `Airoha ARM` / Subtarget `AN7581` /
Profile `Nokia XG-040G-MD (UBI)`，`make -j$(nproc)`。
产物在 `bin/targets/airoha/an7581/`：`*-preloader.bin`、`*-bl31-uboot.fip`、
`*-squashfs-sysupgrade.itb`、`*-initramfs-recovery.itb`。
