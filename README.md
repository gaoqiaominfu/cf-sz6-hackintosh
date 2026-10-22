# Panasonic CF-SZ6 黑苹果（OpenCore）折腾记录

把一台 Panasonic Let'snote **CF-SZ6** 装上 macOS Ventura 的全过程记录：硬件摸底、参考 EFI 选型、U 盘启动盘制作、安装步骤，以及装机后踩的坑（内置 PS2 键盘无驱动等）。

> 目标系统：**macOS Ventura 13.7.8 (22H730)** ｜ 引导：**OpenCore** ｜ SMBIOS：`MacBookPro14,1`

## 机器配置

| 部件 | 型号 | 状态 |
|---|---|---|
| 机型 | Panasonic Let'snote CF-SZ6（CF-SZ6RDQVS，AMI BIOS V2.00L18） | 白名单外 |
| CPU | Intel i5-7300U（Kaby Lake，2C4T） | ✅ 原生 |
| 核显 | Intel HD Graphics 620（8086:5916） | ✅ 原生驱动 |
| 内存 | 8GB LPDDR3 | ✅ |
| 硬盘 | GeIL 1TB SATA SSD | ✅ |
| 声卡 | Realtek ALC256（10f7:0200） | ✅ AppleALC，alcid=56 |
| 有线网卡 | Intel I219-LM（8086:15d7） | ✅ IntelMausi |
| 无线/蓝牙 | Intel Wireless-AC 8265（8086:24fd） | ⚠️ itlwm + HeliPort，无 AirDrop |
| 读卡器 | Ricoh 1217:8520 | ❌ 基本不可用 |
| 内置键盘 | PS/2 | ✅ 加 VoodooPS2Controller 后修复（见文档第八节） |

## 目录

- [`docs/CF-SZ6黑苹果启动盘制作记录.md`](docs/CF-SZ6黑苹果启动盘制作记录.md) — 完整制作记录
  - 硬件与参考 EFI 选型
  - U 盘双分区结构（OpenCore + UnPlugged 离线安装器）
  - Linux 端制作流程（macrecovery / gibMacOS / sgdisk）
  - 安装步骤与注意事项
  - **键盘修复记录**：CF-XZ6 的 EFI 只带了 I2C 触摸板驱动，CF-SZ6 键盘是 PS/2，需补 VoodooPS2Controller 并只启用 Keyboard 插件
  - 遗留问题清单

## 关键结论（省流）

1. **没有 CF-SZ6 专属 EFI**，最接近的是 `tpc-pascal/OpenCore-PANA-CFXZ6-Monterey`（同为 i5-7300U + HD620 + AC8265）。
2. 该 EFI **缺 PS/2 键盘驱动**，装完内置键盘完全没反应 —— 这是最容易卡住的一步。
3. 修复要点：加 `VoodooPS2Controller.kext`（2.3.7），`Kernel/Add` 只启用 Controller 与 Keyboard，Mouse/Trackpad/VoodooInput 保持禁用，避免和 I2C 触摸板的 VoodooInput 冲突。
4. USB 端口映射沿用 CF-XZ6 的 `UTBMap.kext`，**未针对 CF-SZ6 定制**；恢复环境认不到 U 盘时先禁用该 kext。
5. Intel 无线网卡只能走 itlwm + HeliPort，**没有 AirDrop / 隔空投送**。

## 参考仓库（已锁版本）

复现时请 checkout 到下列 commit，避免上游后续改动导致配置对不上：

| 仓库 | 用途 | 锁定 commit | 上游最后更新 |
|---|---|---|---|
| [tpc-pascal/OpenCore-PANA-CFXZ6-Monterey](https://github.com/tpc-pascal/OpenCore-PANA-CFXZ6-Monterey) | **主用 EFI**（i5-7300U + HD620 + AC8265） | [`096e81a`](https://github.com/tpc-pascal/OpenCore-PANA-CFXZ6-Monterey/tree/096e81a01ab128b4792bca34759a7358671c5fee) | 2026-04-28 |
| [waldoxhm/Panasonic_CF-SZ5_SZ5PDYVS_hackintosh_EFI](https://github.com/waldoxhm/Panasonic_CF-SZ5_SZ5PDYVS_hackintosh_EFI) | 前代机型，ACPI/EC/睡眠参考 | [`7820c4a`](https://github.com/waldoxhm/Panasonic_CF-SZ5_SZ5PDYVS_hackintosh_EFI/tree/7820c4a01bdd86d8b3ac0a5cdae0838c14204a8a) | 2021-11-24 |
| `stdrice/dell-latitude-7290-kabylake-opencore-efi`（Codeberg） | 同平台 kext 与音频 layout 参考 | —（未锁） | — |

```bash
# 复现时拉到锁定版本
git clone https://github.com/tpc-pascal/OpenCore-PANA-CFXZ6-Monterey.git
git -C OpenCore-PANA-CFXZ6-Monterey checkout 096e81a
```

> **本项目不是 fork。** 这里只记录 CF-SZ6 的适配过程与差异，不复制上游 EFI 二进制；
> 上游是 MIT 协议，按协议保留署名即可，无需 fork 派生（fork 还会让仓库永久公开、无法转为 private）。

## 上游反馈

- 已向上游提交 PR，补充「EFI 缺 PS/2 键盘驱动」的说明：
  [tpc-pascal/OpenCore-PANA-CFXZ6-Monterey#1](https://github.com/tpc-pascal/OpenCore-PANA-CFXZ6-Monterey/pull/1)
  - 新增 `KEYBOARD-FIX.md`（现象 / 根因 / 5 条 kext 配置表 / 为何只启用两个插件）
  - 在上游 README 的 Issues 区加了一行指向该说明

## 免责声明

黑苹果属于非官方支持的用法，涉及重装系统与全盘格式化，**有丢数据风险，请自行备份**。本仓库只记录个人折腾过程，不附带任何 Apple 版权文件（安装镜像请自行通过官方渠道获取）。参考本记录造成的任何损失由使用者自负。

## License

文档内容采用 [MIT License](LICENSE)。
