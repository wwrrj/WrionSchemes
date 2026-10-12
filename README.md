# WrionSchemes

WrionTools 使用的移植方案仓库。只收录在真机上验证过、能在 `system`、`system_ext`、`product`
上完成的改动；`vendor`、`odm` 等保护分区一律不改。

| 方案 | 源系统 | 目标机型 |
|---|---|---|
| `schemes/flyme12` | Flyme 12.6.0.0A（meizu21Pro，SDK 36） | Xiaomi 14（houji / houjiin） |

## flyme12 插件

| 插件 | 内容 |
|---|---|
| `flyme12.cleanup` | 删除 `product/custom`、Flyme 音乐、广告、社区、商店、阅读，移除 `media.settings.xml` 属性。魅族相机保留。 |
| `flyme12.wrion-patcher-houji` | WrionPatcher 静态 DEX 补丁：services（指纹、人脸、色彩、无线反充、手势）、魅族与 QTI telephony（读卡）、Settings（指纹录入、人脸预览、抬手亮屏、色彩风格页、亮度）、SystemUI（亮度、AOD、光学指纹与 mBack）、魅族相机（小米相机 HAL）。改过的 APK 用 Harusora 密钥重签，并补上对应的 `seinfo=default` seapp 规则；魅族相机移到 priv-app。 |

`wrion-patcher-<机型>` 插件由 WrionTools 的 **WrionPatcher** 页面生成和维护：在页面里为机型勾选插桩
方案并保存，会改写插件文件和该机型 `device.toml` 的 `enabled` 列表。不要手工编辑这些插件。

## 仍需 Magisk 模块的部分

这些改动依赖保护分区或 vendor SELinux 类型，方案里不做：

- `wrion_static_composer_als`：vendor composer 的环境光传感器机型门槛。
- `flyme_fuxi_vibrator_hal`：振动 HAL 与 AAC 效果文件。
- `wrion_static_camera` 的 `sepolicy.rule`：魅族相机人像分割读取 CDSP FastRPC 库。
- `wrion_systemui_doze_sepolicy`：SystemUI 写 AOD 面板亮度节点。
- `wrion_miui_camera`：小米相机与小米相册。

## 新增机型

1. 复制 `devices/houji/device.toml` 到 `devices/<codename>/`，改机型信息和 `[flash]` 映射。
2. 在 WrionTools 的 WrionPatcher 页面为新机型勾选方案（可以从 houji 复制），保存。
3. 在 WrionTools 里规划一次，确认没有诊断再提交。

WrionPatcher 只接受已审阅的基线包，源 ROM 版本不同时补丁会直接失败而不是猜测。
