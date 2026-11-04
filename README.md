# xiaomi-chiron — Xiaomi Mi MIX 2 Ubuntu Touch 移植

把 Ubuntu Touch 24.04 移植到 Xiaomi Mi MIX 2（chiron，骁龙 835 / msm8998）。
本仓库是**移植骨架**，结构完全继承自官方支持的 Mi 6（sagit）移植组：
https://gitlab.com/ubports/porting/community-ports/android9/xiaomi-mi-6/xiaomi-sagit

- 底座：LineageOS 16.0（Android 9）+ Halium 9.0
- 目标：Ubuntu Touch 24.04（arm64/android9plus 通道）
- 路线：community-ports GSI 式 CI（rootfs 由 system-image-upgrader 安装进 /data）
- **不在范围内（用户已确认）：相机、指纹**
- 已知平台限制：**Halium 9 不支持 VoLTE**，通话走 2G/3G 回落

## 目录结构

```
deviceinfo              设备描述（内核源/defconfig/cmdline/分区参数）
.gitlab-ci.yml          GitLab CI（引用 UBports 共享管线）
build.sh                CI 里调用的构建入口
kernel/chiron_defconfig 生成的 Halium 9 内核配置（放入内核 fork 用）
overlay/                设备适配文件（halium.yaml、ofono、udev、powerd…）
ramdisk-overlay/        boot ramdisk 脚本（halium 引导逻辑）
recovery/overlay/       UT recovery 相关（fstab、upgrader、ramdisk）
manifests/xiaomi_chiron.xml  备用路线：Halium 9.0 legacy 全量 system.img 构建
```

## 前置条件

- 手机：bootloader 已解锁 ✔、Magisk root ✔、MIUI 12 / Android 9 ✔（这台机器已确认）
- 电脑：adb/fastboot（platform-tools），Windows PowerShell 即可
- GitLab 账号（CI 用免费 shared runner 构建，**不需要本机构建**）

## 步骤 0 —— 备份（必做，防变砖）

```powershell
powershell -ExecutionPolicy Bypass -File tools\backup-partitions.ps1
```

会备份 44 个小分区到 `C:\00\chiron-backup\`（含 persist/fsg/modemst1/2 等
存 IMEI 与校准数据的分区）。**核对输出里 modemst1/modemst2/persist/fsg 是否成功。**

## 步骤 1 —— 提取 vendor blobs（必做）

```powershell
powershell -ExecutionPolicy Bypass -File tools\extract-vendor.ps1
```

产物：`out\vendor_full\system_vendor\`（约 700MB，MIUI 12 的完整
/system/vendor 树）及 modem/蓝牙固件分区副本。这棵树随后由
`tools\twrp-install-vendor.ps1` 装进 cust 分区（见步骤 3.1）。

> 备用：`tools\build-vendor-repo.ps1`（按 LOS16 的 proprietary-files.txt
> 把 blobs 整理成 TheMuppets 风格仓库），仅备用 manifest 路线
> （manifests/xiaomi_chiron.xml）需要，主路线 CI 用不到。

## 步骤 2 —— 准备内核 fork + 推送本仓库并跑 CI

**必做：fork 内核**。共享内核 `android_kernel_xiaomi_msm8998` 的
halium-9.0 分支只有 `sagit_defconfig`（已实测确认没有
`chiron_defconfig`），所以：

1. 在 GitLab 上 fork `ubports/community-ports/android9/xiaomi-mi-6/android_kernel_xiaomi_msm8998`；
2. 把本仓库的 `kernel/chiron_defconfig`（已按 Halium 9 要求生成，基准取自
   你手机运行中的内核配置 + Mi 6 Halium 增量）上传到 fork 的
   `arch/arm64/configs/chiron_defconfig`；
3. 把 `deviceinfo` 里的 `deviceinfo_kernel_source` 改成你的 fork 地址。

然后：

1. 在 GitLab 新建项目，把本目录（去掉 `sagit-ref`、`out`、`tools` 可留可去）push 上去。
2. 管线会自动跑 `build` job（构建内核 + halium-boot.img）。
3. 构建成功后手动触发 `devel-flashable` job → 下载 artifacts：
   `boot.img`、`recovery.img`、`dtbo.img`、`ubuntu.img.zst`。

## 步骤 3 —— 刷机（会清空全部数据）

顺序很重要：**先装 cust（vendor），再刷 boot/system**。

### 3.1 把 vendor blobs 装进 cust 分区（一次性）

```powershell
powershell -ExecutionPolicy Bypass -File tools\twrp-install-vendor.ps1
```

原理：UT 把 cust 分区挂载为 Android 容器的 `/vendor`（见
`recovery/overlay/etc/recovery.fstab`），与 Mi 6 官方安装器的
`vendor.img → flash cust` 完全同一套路。脚本用 TWRP 临时启动
（`fastboot boot`，不覆盖 recovery），mkfs 后直接把上一步提取的
vendor 文件推入 cust。带 `-ExportVendorImg` 可顺手把 cust 整分区
dd 成 `vendor_chiron.img`，将来用于分发/官方收录。

### 3.2 刷 CI 构建产物

```powershell
$fb  = 'C:\00\tools\platform-tools\fastboot.exe'
$out = 'C:\00\chiron-port\artifacts'   # CI devel-flashable job 下载的 artifacts

# 1) ubuntu.img.zst 用 zstd -d（或 7-Zip 22+）解压为 ubuntu.img
# 2) 手机进 fastboot：关机后按住 音量下+电源
& $fb flash boot   "$out\boot.img"
& $fb flash system "$out\ubuntu.img"
# 可选：& $fb flash recovery "$out\recovery.img"（不刷则保留 TWRP）
& $fb reboot
```

首次开机：halium 引导 → 进入 system-image-upgrader → 自动把 Ubuntu Touch
rootfs（24.04-2.x/arm64/android9plus 通道，随 ubuntu.img 自带）安装进 /data
→ 重启进入 UT。**期间不要断电**，耐心等待（10–30 分钟）。

> 通道说明：`devel-flashable` 产物自带离线 OTA（rootfs 打包在 ubuntu.img
> 里），首次安装不需要手机联网。装完后系统的升级通道指向
> system-image.ubports.com 的 `24.04-2.x/arm64/android9plus/daily`，
> 在官方收录 chiron 之前不会收到 OTA，属正常现象。

> 应急恢复：若变砖/想回 MIUI，进 fastboot 后用备份的 `boot.img`/`system` 或
> 小米官方线刷包（EDL/miflash）刷回。备份的 persist/fsg/modemst 用
> `fastboot flash <分区名> <对应 .img>` 即可恢复。

## 调试备忘（bringup 期）

- 触摸：synaptics_dsx（i2c 5-0020）；无双击唤醒节点（已注释 halium.yaml）
- 音频：tasha codec（WCD9335）+ tas2559，与 Mi 6 同方案
- 闪光灯/手电：`/sys/class/leds/led:torch_0/brightness`、`led:switch_0`
- 单卡：ofono 只配 slot1（qti.conf / chiron.conf）
- RIL 挂起：`sudo service ofono restart`
- 相机、指纹：按用户要求跳过调试
- 验收对照：https://gitlab.com/ubports/community-ports/general/-/blob/master/DeviceChecklist.md
- 官方收录：向 https://github.com/ubports/installer-configs 提 PR（v2/devices/chiron.yml）

## 已知问题 / 限制

- VoLTE 不可用（Halium 9 平台限制）
- 相机、指纹未适配
- 无同机型先例，显示/触屏/音频/RIL 需实测调整
