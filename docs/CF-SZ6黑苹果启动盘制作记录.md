# Panasonic CF-SZ6 黑苹果（OpenCore）启动盘制作记录

> 本文为公开版：内网 IP、主机账号与制作机绝对路径已隐去。

- **日期：2026.10.20**
- 目标主机：一台内网 NixOS 笔记本（IP 与账号已隐去）
- 制作机：本地 Debian 13
- 目标系统：macOS Ventura 13.7.8（22H730）

---

## 一、目标主机硬件

| 部件 | 型号 | macOS 兼容性 |
|---|---|---|
| 机型 | Panasonic Let'snote **CF-SZ6**（CF-SZ6RDQVS），主板 CFSZ6-2，AMI BIOS V2.00L18 | 白名单外 |
| CPU | Intel **i5-7300U**（Kaby Lake，2C4T） | 原生支持到 macOS Ventura 13 |
| 核显 | Intel **HD Graphics 620**（8086:5916） | 原生驱动 |
| 内存 | 8GB LPDDR3 | 可用 |
| 硬盘 | GeIL 1TB SATA SSD + HL-DT-ST DVD-RW | 可用 |
| 声卡 | Realtek **ALC256**（10f7:0200） | AppleALC（layout-id） |
| 有线网卡 | Intel **I219-LM**（8086:15d7） | IntelMausi |
| 无线/蓝牙 | Intel **Wireless-AC 8265**（8086:24fd） | 需 itlwm/HeliPort（无 AirDrop） |
| 读卡器 | Ricoh 1217:8520（PCIe SDHCI） | 基本不可用 |
| 固件 | UEFI，SecureBoot 已关闭 | 可 |

原有系统现状：单盘 NixOS（ESP 500M + swap 8G + root 923G），无 macOS 分区。

---

## 二、GitHub 上的参考 EFI

未找到 CF-SZ6 专属仓库，采用以下仓库组合：

1. **tpc-pascal/OpenCore-PANA-CFXZ6-Monterey**（主用）
   - 同为松下 Let'snote，CPU **i5-7300U + HD620 + Intel AC8265** 完全一致，OpenCore 1.0.x
   - 网卡为 RTL8111，本机为 I219-LM（EFI 内已带 IntelMausiEthernet，适配本机）
   - 作者标注部分功能不完整：键盘/触摸板、摄像头、睡眠唤醒

2. **waldoxhm/Panasonic_CF-SZ5_SZ5PDYVS_hackintosh_EFI**（参考）
   - CF-SZ5 是 CF-SZ6 前代，同 ALC256 + I219-LM，可参考其 ACPI/EC/睡眠补丁
   - CPU 为 Skylake i5-6300U/HD520，需改 platform-id

3. **stdrice/dell-latitude-7290-kabylake-opencore-efi**（参考，Codeberg）
   - i5-7300U / HD620 / Intel 8265 / I219 / ALC256，可作 kext 与音频 layout 参考

> 已有实机案例：有人在 CF-SZ6（i5-7300U）上成功安装 macOS Ventura 13.7.2（显卡/音频/摄像头/触控板/Intel WiFi 可用）。

---

## 三、U 盘启动盘结构

设备：`/dev/sda`，SanDisk 3.2Gen1，28.7G（已清空重建 GPT）

```
sda1  1G   FAT32  OPENCORE
  ├─ EFI/BOOT/BOOTx64.efi
  ├─ EFI/OC/{ACPI, Drivers, Kexts, Resources, OpenCore.efi, config.plist}
  └─ com.apple.recovery.boot/{BaseSystem.dmg, BaseSystem.chunklist}   (674MB，校验通过)
sda2  27.7G ExFAT  UnPlugged
  ├─ InstallAssistant.pkg (12.2GB，Ventura 13.7.8 / 22H730)
  ├─ InstallInfo.plist / BuildManifest.plist / MajorOSInfo.pkg / UpdateBrain.zip / Info.plist
  ├─ BaseSystem.dmg + BaseSystem.chunklist（备份）
  └─ UnPlugged.command
```

### EFI 关键配置
- 来源：CF-XZ6 EFI，SMBIOS `MacBookPro14,1`
- boot-args：`-v keepsyms=1 -igfxblr alcid=56`
- 核显：`AAPL,ig-platform-id = 00001b59`（HD620）
- 声卡：ALC256，`alcid=56` / DeviceProperties layout-id 56
- 有线：IntelMausiEthernet（I219-LM）
- 无线：AirportItlwm（Ventura 专用，21.0.0–21.99.99）
- 蓝牙：IntelBluetoothFirmware + BlueToolFixup + IntelBTPatcher
- 键盘：VoodooPS2Controller.kext 2.3.7（PS2，仅启用 `VoodooPS2Keyboard` 插件）
- 触摸板：VoodooI2C + VoodooI2CHID（I2C，含 VoodooInput）
- USB：USBToolBox.kext + UTBDefault.kext + UTBMap.kext（沿用 CF-XZ6 映射）

---

## 四、制作过程（Linux 端）

1. GitHub 直连不通，改用镜像 `ghfast.top` 下载 EFI 仓库与工具。
2. 工具：`macrecovery.py`（OpenCorePkg）、`gibMacOS`、`UnPlugged`。
3. 用 `macrecovery` 下载 Ventura 恢复镜像（BaseSystem.dmg，重试 3 次后校验通过）。
4. 用 `gibMacOS.py -v 13 --no-interactive` 下载完整安装器（InstallAssistant.pkg，12.2GB）。
5. `sgdisk` 重建 GPT；`mkfs.vfat`（OPENCORE）+ `mkfs.exfat`（UnPlugged，cluster 128K）。
6. 拷贝 EFI、恢复镜像、安装器文件、`UnPlugged.command`，`sync` 后卸载。
7. 校验：USB 上 config.plist 的 boot-args 正确，EFI/BOOT、EFI/OC、com.apple.recovery.boot 齐全。

---

## 五、安装使用方法

1. 目标机用 U 盘启动 → OpenCore 选择 `macOS Base System`。
2. 进入恢复环境后打开「磁盘工具」，将**整块内置盘**格式化为 **APFS / GUID 分区图**
   - ⚠️ **会清空现有系统及全部数据，操作前务必备份！**
3. 关闭磁盘工具，打开终端执行：
   ```bash
   cd /Volumes/UnPlugged
   ./UnPlugged.command
   ```
   该脚本用 InstallAssistant.pkg **离线**组装完整安装器并启动，随后正常安装到 APFS 卷。
4. 安装完成后，可把 U 盘的 `EFI` 文件夹复制到内置盘 ESP，实现脱离 U 盘引导。

---

## 六、注意事项

- **USB 端口映射**沿用 CF-XZ6 的 `UTBMap.kext`（HS01/03/06/07/08/09、SS01/13）。
  CF-SZ6 硬件可能不同；若恢复环境中看不到 U 盘，或安装中途 U 盘掉线，
  可在 `config.plist` 中将 `UTBMap.kext` 的 `Enabled` 设为 `false`。
  （目标机上线后可读取其 ACPI，将 USB 映射调成 CF-SZ6 精确匹配。）
- **SMBIOS 序列号**为仓库作者的，装机可用；但登录 iCloud/iMessage 建议重新生成。
- Intel 无线网卡无法使用 AirDrop/隔空投送，需 HeliPort 连接 WiFi。
- CF-SZ5 参考仓库建议解锁 CFG-lock；本 EFI 中 `AppleCpuPmCfgLock` / `AppleXcpmCfgLock` 已设为 `true` 以兼容未解锁状态。
- 制作当日（2026.10.20）目标机曾短暂离线（ping/SSH 不通）。

---

## 七、关键文件位置（制作机）

- EFI 仓库：`~/hackintosh/efi/`
- 恢复镜像：`~/hackintosh/installer/com.apple.recovery.boot/`
- 完整安装器：`~/hackintosh/installer/macosdownloads/`
- 工具：`~/hackintosh/tools/`
- 最终 EFI 暂存：`~/hackintosh/usb/OPENCORE/`

---

## 八、键盘无法识别修复记录（2026.10.22）

### 现象

装好的 macOS 内置键盘完全无反应（外接 USB 键盘可用）。

### 根因

EFI 参考的 CF-XZ6 仓库只带了 I2C 触摸板驱动（VoodooI2C / VoodooI2CHID），**没有 PS2 键盘驱动**（该仓库作者标注「键盘/触摸板功能不完整」）。而 CF-SZ6 的键盘是 PS2 接口（与前代 CF-SZ5 相同），config.plist 里未引用 `VoodooPS2Controller`，键盘自然无驱动。

### 修复

1. 加入 `VoodooPS2Controller.kext` **2.3.7**（Ventura 兼容，含 4 插件），来源 `hackintosh-build/EFI/OC/Kexts/`。
2. 修改 `config.plist` 的 `Kernel/Add`，新增 5 条 kext 引用：

   | kext | Enabled |
   |---|---|
   | VoodooPS2Controller.kext | ✅ |
   | VoodooPS2Controller/PlugIns/VoodooPS2Keyboard.kext（键盘） | ✅ |
   | VoodooPS2Controller/PlugIns/VoodooPS2Mouse.kext | ❌ |
   | VoodooPS2Controller/PlugIns/VoodooPS2Trackpad.kext | ❌ |
   | VoodooPS2Controller/PlugIns/VoodooInput.kext | ❌ |

   - 禁用鼠标/触摸板/VoodooInput 是为了避免与已有的 I2C 触摸板（VoodooI2C 自带的 VoodooInput）冲突；触摸板继续走 I2C。
   - VoodooPS2Keyboard 只依赖 VoodooPS2Controller，不依赖 VoodooInput，禁用后者不影响键盘。

3. 原 config 备份为 `config.plist.bak-before-ps2fix`；校验 24 个启用 kext 文件齐全。

### 应用到已装系统

1. 用修复后的 U 盘启动（此时内置键盘已可用）。
2. 进系统后挂载内置盘 ESP，把 U 盘 `EFI` 文件夹覆盖到 ESP。
3. 之后不插 U 盘、内置启动，键盘同样可用。

---

## 九、遗留问题

- **USB 端口映射**仍为 CF-XZ6 版本，未针对 CF-SZ6 重新定制。
- **触摸板**走 I2C，尚未验证多点触控与手势是否完整。
- **睡眠唤醒**、**摄像头**、**读卡器** 未验证。
- **SMBIOS** 未重新生成，未测试 iCloud / iMessage / FaceTime。
