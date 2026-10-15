# WrionSchemes

WrionTools 使用的移植方案仓库。收录在 `system`、`system_ext`、`product` 上完成的改动，
已验证内容、待验证方案和未解决项分别注明；`vendor`、`odm` 等保护分区一律不改。

| 方案 | 源系统 | 目标机型 |
|---|---|---|
| `schemes/flyme12` | Flyme 12.6.0.0A（meizu21Pro，SDK 36） | Xiaomi 14（houji / houjiin） |

## flyme12 插件

| 插件 | 内容 |
|---|---|
| `flyme12.cleanup` | 删除 `product/custom`、Flyme 音乐、广告、社区、商店、阅读，移除 `media.settings.xml` 属性。魅族相机保留。 |
| `flyme12.wrion-patcher-houji` | WrionPatcher 授权修补：services（指纹、人脸、色彩、无线反充、手势）、魅族与 QTI telephony（读卡）、Settings（指纹录入、人脸预览、抬手亮屏、色彩风格页、亮度）、SystemUI（亮度、AOD、光学指纹与 mBack）。实现通过 ID 加密下发，固定调用桩留在 ROM；改过的 APK 用 Builder 内置发布密钥重签，并补上对应的 `seinfo=default` seapp 规则。魅族相机已改为整包方案，见下。 |
| `flyme12.houji-whole-app-meizu-camera` | 魅族相机整包：静态补丁（小米相机 HAL）后的 `Camera.apk`，发布密钥重签，放入 priv-app。不做运行时插桩，不依赖 ID。已启用。 |
| `flyme12.houji-whole-app-leica-camera` | 徕卡相机与相册（绑定）整包：`MiuiCamera.apk`、`MIUIGallery.apk` 及库、权限 XML、`device_features/houji.xml`。已默认启用。 |
| `flyme12.houji-xiaomi-libraries` | 相机用小米系统库（`system_ext`，11 个文件）独立方案。已默认启用。 |
| `flyme12.houji-leica-cameraserver` | `cameraserver` 流桥接（ARM64 RX 段）。已默认启用，**待构建后与参考版本核对**，cameraserver 为所有相机共用。 |
| `flyme12.houji-xiaomi-apps` | 小米钱包与账号（MiApps 0.8.19 载荷）。已默认启用，**TSM 与账号的 services / framework 补丁未启用**，见 [记录](schemes/flyme12/docs/houji-xiaomi-apps-20261015.md)。 |

魅族与徕卡整包的来源、哈希、改动方法与分片重建见 [整包相机记录](schemes/flyme12/docs/whole-app-camera-20261015.md)。
sepolicy 不进方案，已直接合并进 `xm14/vendor_a/etc/selinux/vendor_sepolicy.cil`（CIL，待构建编译）。
分片文件不入库（`assets/apps/*/*.part*`）。

`wrion-patcher-<机型>` 插件由 WrionTools 的 **WrionPatcher** 页面生成和维护：在页面里为机型勾选插桩
方案并保存，会改写插件文件和该机型 `device.toml` 的 `enabled` 列表。不要手工编辑这些插件。

## houji 增量处理方案

[2026-10-15 处理方案](schemes/flyme12/docs/houji-integration-20261015.md) 已核对历史 Claude
记录、后续 Codex 对话及设备文件，列出全部 16 个模块的分区归属、已验证改动和未解决项。
钱包短信登录已由用户验证成功；TSM 后续仍有“无法获取账号信息”，不代表卡片、支付或车钥匙已可用。

[补丁选择草案](schemes/flyme12/drafts/houji-integration/selection.toml) 与
[设备文件清单](schemes/flyme12/devices/houji/integration-inventory-20261015.json) 用于后续接入。
新选择尚未启用：Builder 已支持授权 framework / miui-camera 配置，配套 ART 产物与整包验收仍须完成；
不能只替换 framework.jar 或沿用不匹配的启动镜像。当前启用的整包相机、依赖库和小米 Apps
以本文插件表及机型配置为准，不代表上述 framework 增量已经启用。
原六组选择已迁移到授权配置，之后魅族相机拆分为整包方案；签发流程与验证范围见
[授权插件接入记录](schemes/flyme12/docs/authorization-migration-20261015.md)。

## 固定基线与运行时依赖

vendor composer、振动 HAL 和 AAC 文件由用户维护的 vendor/odm 固定基线提供，插件仍不写保护分区。
原先依赖模块脚本的 Composer 属性、相机附加 ID、触感强度同步和 AOD sysfs 权限需分别接入，
不能由文件复制自动恢复。

振动、魅族相机、SystemUI doze 三份 `sepolicy.rule` 独立记录，未并入方案。
小米相机、相册和小米账号栈已接入 system/system_ext/product 的整包插件，具体依赖及验证边界
见上述处理方案。

## 新增机型

1. 复制 `devices/houji/device.toml` 到 `devices/<codename>/`，改机型信息和 `[flash]` 映射。
2. 准备该机型配套的固定授权 ABI 与发布数据；当前 Builder 授权基线适用于 houji，不能仅复制配置代替适配。
3. 在 WrionTools 的 WrionPatcher 页面为新机型勾选方案并保存。
4. 在 WrionTools 里规划一次，确认没有诊断再提交。

WrionPatcher 只接受已审阅的基线包，源 ROM 版本不同时补丁会直接失败而不是猜测。
