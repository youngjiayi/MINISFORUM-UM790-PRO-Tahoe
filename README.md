# Minisforum UM790 Pro (Ryzen 9 7940HS / Radeon 780M) — macOS Tahoe Hackintosh

一台 **UM790 Pro 迷你主机**（Zen4 Phoenix / 7940HS / Radeon 780M 1002:15BF）跑通 macOS Tahoe 的完整配置：OpenCore 引导、780M 的 Metal 3 硬件加速、M2 Liquid Glass 桌面合成、Wi‑Fi / 蓝牙 / AirDrop、CPU 降频静音，以及一整套 NVRAM 取巧。

> **本仓库是脱敏后的公开版**：SMBIOS 序列号 / MLB / UUID / ROM(MAC) 已全部替换为占位值，请用 GenSMBIOS 生成你自己的。随仓库提供的是脱敏 config、私有 ACPI `.aml`、以及本机自编译的两个 kext（`PhxBringup` / `PhxCPPC`）。其余第三方 kext / driver / OpenCore **不在仓库内**，只给出处与 SHA‑256 校验值——原因是这些组件版权归各自作者、部分许可限制再分发（尤其 nullarc 派生的加速驱动为 PolyForm Noncommercial，仅非商业用途），请自行从原项目获取并遵守其许可。

---

## 1. 硬件与状态

| 项目 | 值 |
|---|---|
| 机型 | MINISFORUM UM790 Pro（迷你主机，AMD Phoenix 平台，无电池、无独显） |
| CPU | AMD Ryzen 9 7940HS（Zen4 Phoenix，8 核 16 线程，TDP 45W） |
| iGPU | AMD Radeon 780M，RDNA3 `gfx1103`，Device ID `0x15bf` / `0x15bf1002`，12 CU |
| 显存 | 从 BIOS 手动设置 8 GB；系统显示 VRAM 8 GB |
| 内存 | 64 GB |
| 无线 | Intel AX1675x（AX210 系列，非 CNVi）— itlwm / ForbiddenItlwm + IntelBluetoothFirmware |
| 有线 | Realtek RTL8125（2.5G）— RTL812xLucy |
| 系统 | macOS Tahoe 26.7.3（25G342） |
| SMBIOS | `MacPro7,1`（软件解码 + 无电池台式机语义，避免笔记本唤醒/电池报错） |

已验证可用：Metal 3、M2/Liquid Glass 玻璃效果、隔空投送 AirDrop、Wi‑Fi、蓝牙、HiDPI、CPU 名称正确显示、CPPC 温控降噪。效果图与验收截图见 [`evidence/`](evidence/)（后续补充）。

---

## 2. 三个真正的坑（本仓库重点）

### 2.1 xHCI 丢中断 → 安装器卡死、键鼠不亮
原生 EFI 缺 AMD Phoenix 的 xHCI 中断修复，安装阶段 `pci ... flags 0xc3080...` 卡死、键鼠不亮。
**修复**：`PhxXhciKick.kext`（ring1 轮询）+ `WujieXhciIntx.kext`（MSI→INTx），均挂 IOResources；boot-args 加 `ipc_control_port_options=0 -nohe phxkick=0 wujiexhci=1`；启用 `SSDT-PLUG-ALT.aml`。

### 2.2 780M 没有 Metal → 玻璃效果全无
nullarc 的 `PhxAccel` 原生只认 `0x1900`，需给 Info.plist 补 `0x15bf1002`；`NMXAccel` 同理。装完后是 headless 模式（合成走 CPU），要解锁 M2 玻璃必须走下面的 NVRAM 握手。

### 2.3 `phxfb-boot` 重启就丢 → M2 永远起不来
驱动读 `/options` 的 `phxfb-boot` 决定是否接管 framebuffer。手动 `nvram phxfb-boot=1` **重启后丢失**（缺 commit），FORCESYNCNOW 在本机也报 general error。
**最终修复**：不用手动 nvram，直接在 **config.plist 的 `NVRAM → Add`** 里注入（每次 OpenCore 引导自动写入）：

```xml
<key>7C436110-AB2A-4BBB-A880-FE41995C9F82</key>
<dict>
  <key>phxfb-boot</key>
  <data>MDE=</data>   <!-- ASCII "01"；驱动只看变量存在与否 -->
</dict>
```

`Delete` 段**不要**列该项，否则每次引导又被删掉。

`phxboot.sh` 里的三个 `sleep 1` 千万别删——分别等 framebuffer 注册、显示 pipe/mode change、AGDC 上报，删了 M2 起不来。

---

## 3. 复现步骤

1. 用 GenSMBIOS 为 `MacPro7,1` 生成**你自己的** Serial / MLB / UUID，ROM 填你有线网卡 MAC。
2. 按第 5 节的出处表下载其余 kext / driver / OpenCore，用 SHA‑256 核对；本机自编译的 `PhxBringup` / `PhxCPPC` 已随仓库提供，直接放入 `EFI/OC/Kexts/` 即可。
3. 把本仓库 `efi/OC/config.plist` 的占位符替换为你的值；确认 `NVRAM → Add` 里 `phxfb-boot = MDE=` 在、`Delete` 里没有它。
4. 按 nullarc 文档安装 GPU 驱动（`install.sh --check` / `--windowserver`），并给 `PhxAccel` / `NMXAccel` 的 Info.plist 补 `0x15bf1002`。
5. 打开自动登录（登录窗在 M2 下可能黑屏），重启两次：第一次自检并 arm，第二次接管。

---

## 4. 验收命令

```bash
sw_vers                                          # 26.7.3 / 25G342
system_profiler SPDisplaysDataType               # 780M / Metal 3 / 8GB
sysctl -n debug.nmxaccel_headless                # 0 = 已接管
log show --last boot --style compact 2>/dev/null | grep -iE 'phxfb-boot|panel published'
tail -5 /var/log/phxboot.log                     # done / armed and released
```

---

## 5. 组件出处与校验值

下表列出整套 EFI 用到的第三方组件出处与 SHA‑256。除本机自编译的 `PhxBringup` / `PhxCPPC` 已随仓库提供外，其余均需你从原项目获取。版权归各自作者，请遵守其许可；表中标注了本机自编译项与官方包哈希不同。

### OpenCore / drivers
| 文件 | 出处 | SHA‑256 |
|---|---|---|
| OpenCore.efi | Acidanthera/OpenCorePkg | `5af6acdaabdf02f8727fd3ee2aac3ae8388f4059ce939970e167a503d12aa82a` |
| BOOTx64.efi | Acidanthera/OpenCorePkg | `c4c8e264ad0f59ea9acd17f48457b1f30dd0b08439e5ac960dcc2043288236e2` |
| HfsPlus.efi | Acidanthera/OpenCorePkg | `a55b5fff36578864ba6792c4c6369c71f6f35b61dd5a853ddf8583cd36c31d8f` |
| OpenCanopy.efi | Acidanthera/OpenCorePkg | `2f1e051b8339803e4941daeea46a8e4033638e178ee971e56828519f6fba8503` |
| OpenRuntime.efi | Acidanthera/OpenCorePkg | `12b9c3048bf4cd21133765bdc1355db513d7c60430347defcade1445e2a0b693` |
| ResetNvramEntry.efi | Acidanthera/OpenCorePkg | `5217eff381595ecd1a642d735c4179870c69a72c1302cc38279e034d4bc0a95f` |
| apfs_aligned.efi | Acidanthera/OpenCorePkg | `1663fa54103cf808b45fda3f0385626d1f1899a9ed8258d21a827db4426ed408` |

### Kexts（23 个）
| Kext | 出处 | SHA‑256 |
|---|---|---|
| AMFIPass.kext | dhinakg/AMFIPass | `7261dc4513223c69003354120f2a674b07c68e7d952021f056959623853b3f21` |
| AppleALC.kext | Acidanthera/AppleALC | `53208ad1e3f8ea7ae9636cf315cca352605cb2d74cb624f137cf8f8d4ea61eca` |
| AppleMCEReporterDisabler.kext | xlnc / 社区 | `5402e046bf1525644a97a7641106401011b95217d9a97b8916b41b7c957a7d57` |
| BTLEAdvFixup.kext | forbidden-itlwm | `9a867697e36b94758ed4a1901c0f069a1af5a84c8d7d7797543e29681b589ffc` |
| BlueToolFixup.kext | Acidanthera/BrcmPatchRAM | `68d82d4c838f6388dd035a87ffc3d4ffddd85e70ed8c3648ccbd7de8981c9234` |
| BrightnessKeys.kext | Acidanthera | `b888269cb596e72daed773e1b52301f112c31825d9f7fb5ee2329760743d7ce3` |
| ForbiddenItlwm.kext | forbidden-itlwm | `2ecddc8c50aaabb1927a24f6fe3efef61e97217164e619940f4d5a5107ebc005` |
| IntelBTPatcher.kext | zxystd/IntelBluetoothFirmware | `8ff0f509845a565c1702e5c38138d9ced6427709ef80fd7ee560fb8e1eff350a` |
| IntelBluetoothFirmware.kext | zxystd/IntelBluetoothFirmware | `665bf22ad5f2345fb9001c2bcc9ecbc1d6057cbcaaccefa00e38590b5575973c` |
| Lilu.kext | Acidanthera/Lilu | `232ce93a96d9b97ec18161172940f141317ec0179fa27c9841f84bd66fa99dab` |
| NVMeFix.kext | Acidanthera/NVMeFix | `cef734775c1b11f15bd8be9ff6c98db2f03b42f0b4bde37ca544b4a8d642793e` |
| **PhxBringup.kext** ✅随仓库 | nullarc 源码本机 15BF 编译版（非官方 0x1900 版，勿覆盖） | 二进制 `bb1389d7d93b17543454ab9fa7c90aa128e1086a344ba58ffa12a927974de820` |
| **PhxCPPC.kext** ✅随仓库 | nullarc issue #21 源码本机编译（Zen4 CPU 降频） | 二进制 `18bf10f1144adc0225c83829fe74ba0132aa8ffc978af82f64f1dd42079b5760` |
| PhxXhciKick.kext | phx / mechrevo 参考机 | `7a45accce13d23eae3f7b6d2bc9c5ad4bcee8b1e1e35bd25d2f2efee0c5d58d9` |
| RTL812xLucy.kext | Mieze/RTL8125Ethernet (Lucy fork) | `7490ef2099f6edb4f771c08899a3932ad33d976dd03830ba49e1a632299b1c22` |
| RestrictEvents.kext | Acidanthera/RestrictEvents | `b2cf92f78ff82f765d6238638352933edd4b6f9ad49059b3ceb6177bd273e5dc` |
| SMCBatteryManager.kext | Acidanthera/VirtualSMC | `d5c732f45db17e9838d9aa8e4ee841133c4dc1c3351158d1f6af7626226d66de` |
| SMCLightSensor.kext | Acidanthera/VirtualSMC | `4e7789ac60a406fde3f3a655c2cf685501c76f41add2aefade6e84a1ddec8e4a` |
| USBToolBox.kext | dhinakg/USBToolBox | `f0ecfb8f827420adcc205c7339d47fe0a8e356cdb0af3220d5edf6cf80d5991a` |
| UTBDefault.kext | dhinakg/USBToolBox | `6a9ef45b1ad3552dc4e3615c02312c76590017b1af25ef09291836f227228157` |
| VirtualSMC.kext | Acidanthera/VirtualSMC | `32381d46a5589d655c39b580655b2676837056b2e7e43419373704e79ff2cfe7` |
| WujieXhciIntx.kext | local.wujie（社区） | `132295b18b72ee373aa72e82b7be1b7ecb532a8b5ab3277a3e2da0034a7f41b2` |
| itlwm.kext | OpenIntelWireless/itlwm | `e11d9c4edbe9c873b73b86942c486b1ca7c8da1f055a379e3b9838e8a0058e7a` |

> 注：`PhxBringup.kext` 是**本机针对 0x15bf 自编译的版本**，切勿用参考机的 0x1900 版覆盖，否则 780M 起不来。

### 关键 boot-args
```
keepsyms=1 -amfipassbeta -ibtcompatbeta revpatch=sbvmm,cpuname
ipc_control_port_options=0 -nohe phxkick=0 wujiexhci=1
amfi=0x80 amfi_get_out_of_my_way=0x1 phxdebug=1
-btleadv ... (itlwm 蓝牙共存调优若干)
```

> 启动参数刻意**不加 `-v`、`debug=0x100`**，配合 `Misc → Debug → Target=1`、`AppleDebug=false`、`LogModules=""`：不写 OpenCore 日志到 ESP、panic 自动重启，实测 SSH 就绪时间 45–55s → ~30s。保留 `keepsyms=1` 仍可在 panic 时看栈；排查开机问题时可临时把 `-v` 加回去。

---

## 6. 恢复手段

| 手段 | 作用 |
|---|---|
| boot-arg `-phxfboff` | PhxAMDFB 不接管屏幕，回固件帧缓冲（救砖用，临时加） |
| `sudo touch /Users/Shared/gpu/phxboot-off` | 阻止下次启动 arm 接管 |
| `sudo nvram -d phxfb-boot` | 清除握手变量（注意 config 每次引导会重注入） |
| `sudo ./uninstall.sh` | 卸载 nullarc（先看官方 docs/INSTALL.md） |
| U 盘 EFI 启动 | 挂死时的救援路径 |

M2 内置看门狗：WindowServer 崩 2 次自动重启、3 次退回 headless。真黑屏长按电源键 10 秒，下次自动回稳定态。

---

## 7. 脱敏与许可

- SMBIOS（Serial / MLB / UUID）与 ROM(MAC) 在 config 中为占位符 / 全零；请用 GenSMBIOS 生成你自己的，否则可能与他人冲突导致 iMessage / iCloud 异常。
- 本仓库只随附**本机自编译**的 `PhxBringup` / `PhxCPPC`（源码来自 nullarc，对应部分为 MIT / GPL‑2.0‑only）；其余第三方组件不分发，版权归各自作者，见上方出处表。本仓库自有文字与配置以 **MIT** 发布。
- **关于 nullarc GPU 驱动**：`PhxAccel` / `NMXAccel` 等派生自 NullMoth 驱动，采用 **PolyForm Noncommercial 1.0.0（仅非商业用途）**，故不在本仓库分发——请从 [luuumi-dev/nullarc](https://github.com/luuumi-dev/nullarc) 按原许可自行获取；原始 Required Notice：Copyright (c) 2026 NullMoth Systems。
- 仅供学习研究；黑苹果与实验性 GPU 驱动有损坏系统 / 数据风险，操作前做好备份。

## 8. 致谢

- **Acidanthera**：OpenCore、Lilu、VirtualSMC、AMFIPass、RestrictEvents、AppleALC 等
- **luuumi-dev/nullarc**：780M 加速与 GPU bring-up
- **sfyyy/mechrevo-f7bsc-hackintosh**：同代 Phoenix 迷你主机参考、xHCI 经验
- **OpenIntelWireless / forbidden-itlwm / zxystd**：Intel 无线与蓝牙驱动
- **Mieze**：RTL812xLucy；**dhinakg**：AMFIPass / USBToolBox
- **izerok/k8-tahoe-hackintosh**：本仓库 README 结构的参考范本
