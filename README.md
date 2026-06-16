# ER1-uboot-gpt

京东云太乙（RE-CS-07）的 U-Boot 与 GPT 分区相关资源。



---

## 仓库文件说明

| 文件 | 说明 |
|------|------|
| `JDC-TAIYI-1GB-GPT.bin` | 原始 GPT 导出的 34 扇区镜像（18 个分区，HLOS = 6MB，1GB rootfs） |
| `JDC-TAIYI-1GB-GPT-MODIFIED.bin` | 修改版 GPT（HLOS 扩至 12MB + 新增 HLOS_1 备份分区，适应 daed/BPF 需求） |
| `uboot-chenxin-er.bin` | chenxin527 编译的 U-Boot（gl.inet 风格，仅认 6M/12M 内核） |
| `uboot.bin` | 其他版本 U-Boot hugo大佬的uboot不带dhcp|
| `chenxin.txt` | [chenxin527/uboot-ipq60xx-emmc-build](https://github.com/chenxin527/uboot-ipq60xx-emmc-build) 源码分析 |
| `xiaozhu.txt` | [1980490718/u-boot-2016](https://github.com/1980490718/u-boot-2016) 源码分析 |
| `教程.txt` | 完整刷写教程（三种方案） |
| `README.md` | 本文件 |

---

## 背景：为什么需要改 GPT？

原厂 JDCloud AX1800 的 eMMC 分区布局中，内核分区 `0:HLOS` 只有 **6MB**。随着固件功能增加（尤其是 dae/daed 这类需要 eBPF/BPF 支持的大型内核），6MB 空间已不够用。

**目标**：将 HLOS 从 6MB 扩大到 **12MB**，同时新增一个 6MB 的 HLOS_1 备份分区，实现 A/B 双分区启动。

**前置条件**：必须先刷 U-Boot 再改 GPT，以免旧 U-Boot 不认识新分区布局。
- 推荐使用 [1980490718/u-boot-2016](https://github.com/1980490718/u-boot-2016)（自动检测 4MB~16MB 内核大小）
- 也可用 [chenxin527/uboot-ipq60xx-emmc-build](https://github.com/chenxin527/uboot-ipq60xx-emmc-build)（仅认 6MB/12MB）

---

## 分区布局对比

### 原始布局（`JDC-TAIYI-1GB-GPT.bin`）

| # | 名称 | 起始 LBA | 结束 LBA | 大小 | 说明 |
|---|------|---------|---------|------|------|
| 0 | 0:SBL1 | 34 | 1569 | 768K | 一级引导 |
| 1 | 0:BOOTCONFIG | 1570 | 2081 | 256K | 启动配置 |
| 2 | 0:BOOTCONFIG1 | 2082 | 2593 | 256K | 启动配置备份 |
| 3 | 0:QSEE | 2594 | 6177 | 2M | 安全执行环境 |
| 4 | 0:QSEE_1 | 6178 | 9761 | 2M | QSEE 备份 |
| 5 | 0:DEVCFG | 9762 | 10273 | 256K | 设备配置 |
| 6 | 0:DEVCFG_1 | 10274 | 10785 | 256K | DEVCFG 备份 |
| 7 | 0:RPM | 10786 | 11297 | 256K | 资源电源管理 |
| 8 | 0:RPM_1 | 11298 | 11809 | 256K | RPM 备份 |
| 9 | 0:CDT | 11810 | 12321 | 256K | 配置数据表 |
| 10 | 0:CDT_1 | 12322 | 12833 | 256K | CDT 备份 |
| 11 | 0:APPSBLENV | 12834 | 13345 | 256K | U-Boot 环境变量 |
| 12 | 0:APPSBL | 13346 | 14625 | 640K | U-Boot 主分区 |
| 13 | 0:APPSBL_1 | 14626 | 15905 | 640K | U-Boot 备份 |
| 14 | 0:ART | 15906 | 16929 | 512K | 射频校准数据 |
| **15** | **0:HLOS** | **16930** | **29217** | **6M** | **内核（过小！）** |
| **16** | **rootfs** | **29218** | **2126369** | **1.0G** | **根文件系统** |
| **17** | **primary** | **2127872** | **15269854** | **~6.3G** | **用户数据 / 剩余空间** |

> 磁盘大小：7.3GiB，GPT 分区表共 20 个条目，最后一个可用 LBA = 15269854

### 修改后布局（`JDC-TAIYI-1GB-GPT-MODIFIED.bin`）

| # | 名称 | 起始 LBA | 结束 LBA | 大小 | 变化 |
|---|------|---------|---------|------|------|
| 0-14 | 同原始布局 | — | — | — | 不变 |
| **15** | **0:HLOS** | **16930** | **41505** | **12M 🔺** | **从 6MB 翻倍** |
| **16** | **0:HLOS_1** | **41506** | **53793** | **6M 🆕** | **新增备份内核** |
| **17** | **rootfs** | **53794** | **2150945** | **1.0G** | 起始偏移调整 |
| **18** | **storage** | **2150946** | **15269854** | **~6.3G** | 原 primary 更名 |

> **CRC32 已验证通过**：GPT 头 CRC = `0x81F7FD3E`，分区表 CRC = `0xEE76F413`

---

## U-Boot 源码分析

本仓库包含两份详细的 U-Boot 源码分析，帮助理解固件刷写流程和兼容性。

### [chenxin.txt](./chenxin.txt) — chenxin527/uboot-ipq60xx-emmc-build

基于 GL.iNet 风格的 U-Boot，核心文件 `lib/gl_api.c`：

- **`check_fw_type()`**：通过检测固件的魔数（Magic Number）判断类型
  - FIT 镜像中，在偏移 `0x600000`（6MB）或 `0xC00000`（12MB）处查找 SquashFS 魔数 `hsqs`
  - **只认 6MB 和 12MB 两种内核大小**，其他大小无法识别
- **`image_greater_than_partition()`**：读取 GPT 分区大小，校验固件是否放得下
- **`check_fw_compat()`**：刷写前综合校验（固件类型 + 分区大小）
- 有专用 `FW_TYPE_JDCLOUD` 类型，支持京东云官方固件格式

**局限性**：如果 GPT 中 HLOS 是 12MB，但上传的固件也是 12MB，可以正常工作。但如果 HLOS 是 6MB 且固件是 12MB，校验会卡住。反过来 HLOS 12MB 但传 6MB 固件没问题。

### [xiaozhu.txt](./xiaozhu.txt) — 1980490718/u-boot-2016

增强型 U-Boot，核心文件 `lib/ipq_api.c`：

- **`check_fw_type_ex()`**：相比 chenxin527 的版本，增加了 **自动检测内核大小** 功能
  - 依次检测偏移 `0x400000`(4MB)、`0x600000`(6MB)、`0x800000`(8MB)、`0xC00000`(12MB)、`0xE00000`(14MB)、`0x1000000`(16MB)
  - 返回实际 HLOS 大小（不硬编码）
- **分区信息动态读取**：优先查询 SMEM 表，GPT 作为回退
- **完整 A/B 分区刷写**：自动刷 `0:HLOS` → `rootfs` → `0:HLOS_1` → `rootfs_1`，然后更新 BOOTCONFIG
- **额外功能**：
  - GPIO 命令行：运行时查询/控制 GPIO 引脚
  - 按键检测：长按 3 秒进 HTTPD 恢复模式
  - LED 初始化与闪烁控制
  - PHY 链路监测：断线自动重连
  - initramfs 启动支持

**推荐使用此版本**，因为它能自动适应不同的内核大小，对 GPT 修改的兼容性更好。

### 对比总结

| 特性 | chenxin527 | 1980490718 |
|------|-----------|------------|
| HLOS 检测 | 仅 6MB/12MB 硬编码 | 4MB~16MB 自动检测 |
| 分区大小来源 | 硬编码比较 | 动态读取 GPT/SMEM |
| 固件类型 | 含 JDCLOUD 专用 | 标准类型 |
| 升级兼容性 | 只认 6M/12M | 自动适应任意大小 |
| GPIO 控制 | 仅 LED/Button | 完整 gpio 命令 |
| initramfs | 不支持 | 支持 |
| PHY 监测 | 无 | 有，自动重连 |

---

## 刷写教程

详细教程请见 [`教程.txt`](./教程.txt)，包含三种方案的完整操作步骤：

| 方案 | 安全性 | 复杂度 | 适用场景 |
|------|--------|--------|----------|
| **initramfs 过渡** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | 新手 / 首次操作，先让系统运行在内存再改 GPT |
| **SSH 一步到位** | ⭐⭐⭐ | ⭐⭐ | 老手快速操作，直接在运行系统中用 sgdisk 改分区 |
| **HTTPD 分步刷** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 最稳妥的远程操作，U-Boot 网页上传 GPT + 固件 |

**所有操作前请务必备份当前 GPT：**

```bash
sgdisk -b /tmp/gpt_backup.bin /dev/mmcblk0
dd if=/dev/mmcblk0 of=/tmp/emmc_backup.bin bs=512 count=34
```

---

## 推荐刷写流程（最稳妥）

1. **编译**支持 daed 的 OpenWRT 固件（内核大小选 12MB）
2. **刷 U-Boot**：推荐 1980490718 版本，支持自动检测内核大小
   ```bash
   mtd write /tmp/u-boot.bin APPSBL
   mtd write /tmp/u-boot.bin APPSBL_1
   ```
3. **重启进 U-Boot HTTPD**（长按 Reset 键约 5 秒）
4. **上传 GPT**：在网页选择 GPT 类型，上传 `JDC-TAIYI-1GB-GPT-MODIFIED.bin`
5. **重启再次进 HTTPD**
6. **上传固件**：在网页选择 Firmware 类型，上传 12MB 内核固件
7. **自动重启完成**

---

## 参考链接

- [chenxin527/uboot-ipq60xx-emmc-build](https://github.com/chenxin527/uboot-ipq60xx-emmc-build) — 6M/12M 硬编码 U-Boot（emmc 版）
- [chenxin527/uboot-ipq60xx-nand-build](https://github.com/chenxin527/uboot-ipq60xx-nand-build) — NAND 版
- [chenxin527/uboot-ipq60xx-nor-build](https://github.com/chenxin527/uboot-ipq60xx-nor-build) — NOR 版
- [1980490718/u-boot-2016](https://github.com/1980490718/u-boot-2016) — 增强型 U-Boot（推荐）
- [VIKINGYFY/OpenWRT-CI](https://github.com/VIKINGYFY/OpenWRT-CI) — 上游 OpenWRT 自动编译 CI
- [kenzok8/openwrt-daede](https://github.com/kenzok8/openwrt-daede) — dae/daed 包（需求来源）

---

## 设备信息

| 项目 | 值 |
|------|-----|
| 型号 | JDCloud AX1800 (RE-CS-02) |
| SoC | IPQ6000 (Qualcomm IPQ60xx) |
| RAM | 512MB |
| eMMC | 7.3GB / 115GB（因批次而异） |
| U-Boot 位置 | mtd 分区 0:APPSBL / 0:APPSBL_1 |
| 原始 GPT | 18 分区，7.3GB 磁盘几何 |
