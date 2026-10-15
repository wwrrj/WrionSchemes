# houji 整包相机方案记录（2026-10-15）

魅族相机与徕卡（小米）相机改为整包方案：方案只放置已改好的 APK，不做运行时插桩，也不依赖 ID 运行时。
相册与徕卡绑定为同一方案；系统库、cameraserver 各自独立成方案。sepolicy 不进方案，而是并入 vendor 策略源。
本文记录每个产物的来源、哈希、改动方法、签名与分片方式，便于后续重建。

## 一、方案结构

| 插件 | 状态 | 内容 |
| --- | --- | --- |
| `flyme12.houji-whole-app-meizu-camera` | 已默认启用 | 删除原 `system/app/Camera`，解出 `priv-app/Camera/Camera.apk`，并复制权限 XML |
| `flyme12.houji-whole-app-leica-camera` | 已默认启用 | 徕卡相机与相册（绑定）：`MiuiCamera.apk`、`MIUIGallery.apk` 与其 arm64 库、权限 XML、`product/etc/device_features/houji.xml` |
| `flyme12.houji-xiaomi-libraries` | 已默认启用 | 相机用小米系统库（`system_ext`，11 个文件）：MIUI JAR、cameraopt、相机 JNI / HAL 库、XML 与 public.libraries |
| `flyme12.houji-leica-cameraserver` | 已默认启用，**待验证** | `cameraserver` 流桥接，ARM64 RX 段方式（见第四节） |
| `flyme12.houji-xiaomi-apps` | 已默认启用，**有阻塞项** | 小米钱包与账号（MiApps 0.8.19 载荷），见 `docs/houji-xiaomi-apps-20261015.md` |
| `flyme12.wrion-patcher-houji` | 已改 | 删除原方法级相机步骤（`patch-authorized-camera`、`drop-camera-apk-preopt-0`、`install-camera-priv-app`）。保留包名限定 seapp 规则 |

默认启用的设备列表见 `devices/houji/device.toml`。

- 徕卡相机依赖系统库插件、cameraserver 插件与相册，三者随徕卡默认启用。
- sepolicy 不进方案，已并入 xm14 的 vendor 策略源，见第五节。
- `wrion-patcher-houji/plugin.toml` 由 WrionTools 的 WrionPatcher 页面生成。在页面里保存且仍勾选魅族相机方法级方案时，
  会重新生成旧的相机步骤。若要保持整包，需在页面中取消相机方案。

## 二、魅族相机（Camera.apk）

来源与改动：

- 原厂：`WrionPatcher/outputs/Camera-original.apk`，sha256 `1ccc1f9176760433f5865f0cb3aeedca37a4ef93aa185b80c0caa191d86cd6de`，
  签名证书 SHA-256 `94:15:32:99:…`（Meizu 原证书）。
- 补丁（当前源码，默认 flyme 相机方案，`patches/flyme/camera/`）：
  - `plan.json`（2026-10-12 23:16）、`points.json`（2026-10-12 23:29）。
  - `helper/helper.dex`（2026-10-12 16:49，commit `e2589f4`），sha256 `1114e9565f448e9842416dd022521fd695cdc1f4404bd00bccff0f8689576399`，即补丁输出的 `classes3.dex`。
  - 命令：`wrion-patcher compat --scope camera --package <原厂 APK> --root <WrionPatcher 仓库>`（39/39 匹配），
    然后 `wrion-patcher patch --scope camera --package <原厂 APK> --output <输出> --root <WrionPatcher 仓库>`。
  - 输出（未签名）：sha256 `57a93a39e252d4b5428333a9ded9cdcdbd3f1884136aae704c0dbb6032908763`，49207238 字节；
    只改 `classes2.dex`、`classes3.dex`，移除 META-INF 签名。
- 签名：用 WrionTools 内置发布密钥（`crates/harusora-engine/resources/harusora-release.keystore.b64`，
  与 `D:\Prjs\HarusoraRomBuilder\workspace\.cache\signing\harusora-release.keystore` 完全一致，keystore sha256 `994569a9…dfa256`），
  别名 `harusora`，`apksigner sign --v2 --v3`（v1/v4 关闭）。密码取 `source_build/environment.rs` 中的
  `HARUSORA_KEYSTORE_PASSWORD` 常量，通过环境变量 `HARUSORA_APKSIGNER_PASS` 传入，不写入命令行或记录。
- 输出（已签名，入方案）：sha256 `aeb3a0e0a9c7c190a1f29e4559ee7b4b2f8dba5b280b35f89528868c87f19ab5`，
  49222862 字节，证书 SHA-256 `9f4619ff…d68a54`，v3 验证通过。

不用的候选（已核对）：

- `WrionPatcher/outputs/Camera-v3-patched.apk`（sha256 `4dcda367…`，2026-10-12 13:45）。
  早于 helper 提交 `e2589f4`（16:49）和 points 更新（23:29）。classes3 为 6472 字节，旧于当前 helper 的 7708 字节。不用。
- `verified-artifacts/camera.apk`（sha256 `82edfe85…`，ID 授权版）。
  classes3 含 `com/wrion/patch/*` 运行时类（31540 字节），依赖 ID 运行时，不符合整包要求。不用。

分片与入库：

- `schemes/flyme12/assets/apps/meizu-camera/meizu-camera.toml`（archive sha256 `04d80d26…`，2 个分片）。
- 源 ZIP：`WrionPatcher/.local/whole-app-camera-20261015/meizu-camera-whole-app.zip`（stored，内含 `system/priv-app/Camera/Camera.apk`）。
- 分片由 `WrionTools/scripts/split-scheme-asset.ps1` 生成。

## 三、徕卡相机与相册（MiuiCamera.apk、MIUIGallery.apk）

相册与徕卡绑定，共用一个 chunk 集合与一个插件。

来源：

- 徕卡原厂：`WrionPatcher/.local/leica-reprocess-20261013/MiuiCamera.original.apk`，sha256 `df80a1755b220ecf5bb21f23188dd5be405af761a4182d003c17bd52d6599e05`，
  证书 SHA-256 `9f4619ff…d68a54`（与 Builder 密钥一致）。
- 徕卡发布版：`MiuiCamera.bokeh-complete.signed.apk`，sha256 `ad8955d69d35e72725914f5b03048a3907727b0dd3088249dd5fb4eff3ea5bb4`，
  211364495 字节，证书同上，v3 验证通过。该哈希与 `verification/leica-camera-modes-houji-20261013.md` 中“设备挂载 APK”的记录一致。
- 相册：来自 Magisk 模块 `wrion_leica_bokeh_complete.zip`（sha256 `ce50c52e…`）内的 `system/priv-app/MIUIGallery/`，
  `MIUIGallery.apk`（sha256 `f76233a6…`，73252065 字节）与 23 个 `lib/arm64/*.so`。该 APK 哈希与设备观测记录一致。
- 同一模块内的 `MiuiCamera.apk` 与徕卡发布版哈希一致（`ad8955d6…`）。
- 已被取代的版本：`MiuiCamera.front.signed.apk`（sha256 `d424b7d2…`，早期）。不用。
- 附带文件（取自模块 zip）：`system/etc/permissions/privapp-permissions-wrion-miuicamera.xml`（2131 字节）与
  `system/product/etc/device_features/houji.xml`（38259 字节）。

分片：`schemes/flyme12/assets/apps/leica-miui-camera/leica-miui-camera.toml`（archive sha256 `4b9f5723…`，7 个分片，
源 ZIP 28 个条目，334438702 字节）。源 ZIP：`WrionPatcher/.local/whole-app-camera-20261015/leica-miui-camera-whole-app.zip`。

**重建缺口（须如实记录）：** `bokeh-complete` 的生成步骤尚未收敛为单一命令。它由
`WrionPatcher/.local/leica-reprocess-20261013/` 下多轮实验（mivi、portrait、session、video、zoom、bokeh 等）累积形成，
`patches/flyme/miui_camera/points.json`（2026-10-15 15:34）是其中的补丁点。当前 `wrion-patcher` 二进制在本机不认识
`miui-camera` 作用域，因此本记录未能用源码重新生成该 APK 并比对哈希。后续重建须先补全该流程。

## 四、系统库与 cameraserver

### 相机系统库（`flyme12.houji-xiaomi-libraries`）

- 来源：`wrion_leica_bokeh_complete.zip` 中 `system/system_ext/` 下的 11 个文件（9990992 字节，ZIP 9992968 字节）。
  包括 `miui-framework.jar`、`miui-cameraopt.jar`、`libcameraopt*.so`、`libcamera_algoup_jni.xiaomi.so`、
  `libcamera_mianode_jni.xiaomi.so`、`libmicampostproc_client.so`、`vendor.xiaomi.hardware.campostproc@1.0.so`，
  以及 `public.libraries-xiaomi.txt` 与两个权限 XML（`wrion-miui-rom-library.xml`、`miui-cameraopt.xml`）。
- 分片：`assets/apps/xiaomi-libraries/xiaomi-libraries.toml`（archive sha256 `19b06219…`，1 个分片）。
  该分片由 Python 按 `split-scheme-asset.ps1` 的 manifest 格式写出（脚本在本次运行中超时），已核对分片与 archive 哈希。
- `RtMiCloudSDK.apk` 与 `miuisystem.apk` 已从本插件移出，由 `flyme12.houji-xiaomi-apps` 安装。
  模块版本（`df82730a…`、`d83bc35e…`）与载荷版本（`61224e5d…`、`d717b5ce…`）不同，因此不能同时存在。

### cameraserver（`flyme12.houji-leica-cameraserver`）

- 基线（原厂）：`WrionPatcher/.local/leica-reprocess-20261013/cameraserver.original`，sha256 `be5b8da94a33f5953704378305c7d32e581b5906be0caf66bb599d66df5b9b9b`。
- 参考产物（Magisk 模块 `wrion_leica_stream_v4`）：`wrion_leica_stream_v4.cameraserver`，sha256 `0ba7049217ec3d58f6824db976333c29119cf748545f023dc27ad7220273e4c0`，
  与设备当前观测一致；记录 `wrion_leica_stream_v4.json` / `PATCH.json`。
- 与基线的差异（反汇编确认）：
  - `0x2b9a78`：`sub sp,sp,#0x120` → `b 0x3b0000`（入口 0x0）。
  - `0x143b54`：`cbz x2,0x143c38` → `b 0x3b0254`。
  - `0x143ba4`：`cmp w21,#1` → `b 0x3b0264`。
  - RX 段：VA `0x3b0000`，文件偏移 `0x3cc000`，1211 字节（stub 末尾为数据串 `YUV input 4080x3060`）。
  - stub 内共 15 处指向基线外部地址的分支（BL 到 PLT / 原函数，以及 B 回原代码），已写入 `branches` 修正表。
- 插件 `plugin.toml` 用 `patch_elf_arm64` 的 `append_rx_stub` 表达上述 3 个 hook、1 段 stub、15 个分支修正。
  stub_hex 为 1212 字节（末尾补 1 个 00，满足引擎 4 字节对齐）。
- **待验证（已默认启用，风险最高）：**
  - 三处入口字不是被替换的原指令（如 0x0 处为 `sub sp,sp,#0xe0`），stub 自行完成帧调整，语义未核对。
  - 引擎 `append_rx_stub` 会写入元数据 section，输出字节与 v4 不同，不能用 sha256 证明等价。
  - cameraserver 是所有相机共用的系统服务。如果修补错误，魅族相机也会受影响。构建后应先与 v4 做语义对比，再刷入。
  - 本机只完成了 lint（结构校验），未运行引擎对基线生成输出。

### 相册

- 与徕卡绑定，同一插件与分片，见第三节。
- 载荷的 `xiaomi-gallery-privapp-permissions.xml` 与 `xiaomi-gallery-shared-libraries.xml` 指向相册包，
  来自 `flyme12.houji-xiaomi-apps`，需在真机上确认无副作用。

## 五、sepolicy（已直接合并进 vendor_a 的 vendor_sepolicy.cil，不进方案）

- 来源：`xm14/magisk_vendor_odm_collection_20261015/sepolicy-report.md` 中三个模块的 `sepolicy.rule`，共 18 条 `allow` / `typeattribute`，
  与原文逐条一致。原归档保留在 `xm14/magisk_vendor_odm_collection_20261015/evidence/sepolicy/`。
- 合并位置：`xm14/vendor_a/etc/selinux/vendor_sepolicy.cil`，使用 CIL 语法，平台名按该文件的约定写成 `*_34_0`。
  - 合并前已有 10 条 vibrator 规则（2026-09-12 的早期工作），本次不重复。
  - 本次追加 11 条：vibrator 缺失的 `vendor_file`（2 条）与 `system_data_file`（2 条）；
    摄像头的 `adsprpcd_file`（2 条）与 `vendor_file`（1 条）；doze 的 `vendor_sysfs_displayfeature`（2 条）、
    `mlstrustedobject` 属性（1 条）；以及 `genfscon sysfs "/class/qcom-haptics"`（1 条）。
  - 合并前备份：`magisk_vendor_odm_collection_20261015/previous_files/vendor_a/etc/selinux/vendor_sepolicy.cil.before-wrion-sepolicy-20261015`。
- 校验：与 OS3 平台 CIL 组合编译时，合并前后报错位置相同（`vendor_sepolicy.cil:541`），说明语法无误。
  完整编译受阻：本机没有与 vendor 版本匹配的完整平台策略，无法做到解析与名称解析全部通过。
  新增符号均已核对存在于该文件或 plat_pub 中。neverallow 检查和真机加载尚未做。
- 编译后才生效：该文件是 vendor 策略源，需要进入构建才会被编译。
- 另有 `vendor_sysfs_haptics` 等类型已由该文件声明；`platform_app_34_0` 与 `vendor_file_34_0` 的规则须通过 neverallow 检查，此项未做。
- 徕卡与 cameraserver 没有 sepolicy 规则。`wrion_leica_stream_v4.zip` 不含 sepolicy 文件，其 `customize.sh` 仅用 `set_perm`
  设置 `u:object_r:cameraserver_exec:s0`。

## 六、入库策略

- 按用户要求，四组应用与库的 14 个分片（`assets/apps/*/*.part*`）与 manifest 一并入库，合计 543492970 字节。
- `.gitattributes` 将分片标为 binary，避免 Windows 换行转换破坏原始字节。
- 克隆方案仓库即可使用分片；Builder 按 manifest 校验分片及完整档案的大小与 SHA-256。
- 本机 `.local` 中的源 ZIP 与本文记录用于复核和重新生成分片，不再是换机器使用方案的前置要求。
- WrionPatcher 中的 plan / points / helper 源仍在对应源码仓库维护。

## 七、验证状态

已做：

- `wrion-tools eng port scheme lint` 全量校验：0 诊断（7 个模块，20 步，80 个动作）。
- `plan --set device_codename=houji --structure-only`：启用模块为 `cleanup`、`wrion-patcher-houji`、
  `houji-whole-app-meizu-camera`、`houji-xiaomi-libraries`、`houji-whole-app-leica-camera`、`houji-leica-cameraserver`、`houji-xiaomi-apps`。
- 分片重组：魅族、徕卡、系统库、小米 Apps 四组 manifest 的 archive sha256 与分片 sha256 均核对通过；内部 APK 哈希与上表一致。
- 签名：魅族与徕卡输出的 v3 验证通过，证书 SHA-256 与 Builder 密钥一致。
- 魅族补丁：当前源码 compat 39/39，输出 classes2 与旧 v3 一致，classes3 与当前 helper 一致。
- cameraserver：基线哈希与 3 个检测点的原字节已核对；v4 的分支目标与 stub 内分支已反汇编确认。

未做：

- 未刷入、未在真机上验证任何整包或插件（本次设备未连接）。
- 未运行引擎的 `append_rx_stub` 对基线生成输出，因而未证明 cameraserver 插件与 v4 语义等价。
- 未验证删除 `app/Camera`、`MiuiCamera/oat`、`MIUIGallery/oat` 后 PackageManager 与 seapp 的实际表现。
- Magisk 模块中的 `post-fs-data.sh`（`vendor.camera.aux.packagelist` 追加 `com.android.camera`）未迁入方案，需要另行接入。

## 八、未决事项

1. **启用前必须处理（小米钱包）：**
   - TSM 的 services 共享 UID 补丁未在方案中启用，ART 匹配未验证。
   - 账号的 framework 作用域补丁未在方案中启用。
   - MiApps 版账号与 TSM 与 houji 上验证过的 OS3 版本不同，登录需整体重测。
2. **启用前必须处理（cameraserver）：** 构建后与 v4 做语义对比，确认后再刷入。
3. sepolicy 已合并进 `vendor_sepolicy.cil`。仍需完整平台策略做编译与 neverallow 检查，并真机验证。

## 九、重建步骤

魅族：

1. 取原厂 `Camera-original.apk`（sha256 `1ccc1f91…`）。
2. 用当前 WrionPatcher 源码 `patch --scope camera`（默认 flyme 方案，`--root` 指向 `WrionPatcher`）得到未签名输出。
   核对 classes3 与 `helper/helper.dex` 一致。
3. 用 Builder 密钥（`harusora`，`apksigner --v2 --v3`）签名。核对签名证书 `9f4619ff…`。
4. 以 `system/priv-app/Camera/Camera.apk` 为路径写入 stored ZIP，运行 `split-scheme-asset.ps1` 生成分片。
5. 核对 manifest 的 archive sha256 与 ZIP 一致，然后 lint 方案。

徕卡与相册、系统库：

1. 从 `wrion_leica_bokeh_complete.zip` 取 `system/priv-app/MIUIGallery/**`、`system/system_ext/**`（去掉 RtMiCloudSDK 与 miuisystem）、
   两个权限 XML 与 `houji.xml`。
2. 徕卡 `MiuiCamera.apk` 取 `MiuiCamera.bokeh-complete.signed.apk`（sha256 `ad8955d6…`）。
3. 各自写入 stored ZIP，运行 `split-scheme-asset.ps1` 生成分片，并核对 archive sha256。
