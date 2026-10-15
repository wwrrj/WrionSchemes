# houji 小米钱包与账号（MiApps 版本）方案记录（2026-10-15）

`flyme12.houji-xiaomi-apps` 把 ColorOS 方案中的小米 Apps 载荷（0.8.19）接入 Flyme 12 houji，并已默认启用。
本文记录来源、与 ColorOS 插件的差异、与设备当前版本的差异，以及未包含项与风险。

## 一、来源

- 载荷：`HarusoraRepairSchemes/schemes/coloros16/assets/common/xiaomi-apps/xiaomi-apps-0.8.19.zip.part000..003`，
  已拼接校验，archive sha256 `a8d5caea232b1913778197e5b98e71f3be24ab1fe9acdb2c3b8ee421eb925174`，
  51 个条目，解压后 277145165 字节。
- 复制到本方案：`assets/apps/xiaomi-apps/`（4 个分片与 manifest，均已入库）；
  三份权限 XML 复制到 `assets/common/xiaomi-apps/`。
- 依据插件：`HarusoraRepairSchemes/schemes/coloros16/plugins/xiaomi-apps/plugin.toml`（0.8.20），README 记录了账号 compat jar、
  NFC、钱包入口等 ColorOS 侧的已知问题。

## 二、与 ColorOS 插件的差异

保留：

- 安装 product/app、product/priv-app、product/etc/permissions、system_ext/priv-app、system_ext/app、system_ext/framework、
  system_ext/etc/permissions，冲突即失败（`conflict = "fail"`）。
- default-permissions、privapp 拒绝列表、CloudService privapp allowlist 三份 XML。
- `ro.miui.ui.version.code=816`、`ro.miui.ui.version.name=V816`（`product/etc/build.prop`）。账号 LoginActivity 会直接解析该属性。
- 全部 `set_fs_config`（目录 0755、APK 0644、uid/gid 0）。

去掉（ColorOS 专属，Flyme 基线不同）：

- `framework-location.jar` 替换（ColorOS 基线的 dex 布局修复）。
- `product_mac_permissions.xml` 替换（Flyme 上 vendor_mac_permissions 已把小米证书映射为 platform，见 houji 记忆）。
- `my_region/etc/extension/default_wallet_configuration.xml`（双击电源键入口）。
  Flyme 上该路径未核对，因此未放入。双击电源键入口需要补核对。
- `FidoAsm` 删除（ColorOS 的 Oplus FIDO 实现，与 Flyme 无关）。
- `ro.harusora.xiaomi_apps.mode=miui-apps` 标记（仅 ColorOS 的 edition 逻辑读取）。

新增：

- system_ext 的 RtMiCloudSDK 与 miuisystem 由本插件安装（载荷版本），不在徕卡的系统库插件中，避免重复。

## 三、与设备当前版本的差异（重要）

当前 houji 设备（由 Magisk 模块与 OS3 基线提供）上观测到的哈希与载荷对比：

| 包 | 设备观测 | 载荷（MiApps） | 结论 |
| --- | --- | --- | --- |
| MIpay | `3fcee587…` | `3fcee587…` | 相同 |
| MIUIXiaomiAccount | `a5fd990e…`（product/app） | `20eb4e72…`（product/priv-app） | 版本与位置都不同 |
| MITSMClient（TSM） | `d18dddd6…`（1865） | `c79df91c…`（1905） | 版本不同 |
| FindDevice | `c36e3167…` | `d3a72762…` | 版本不同 |

因此启用本插件后，账号与 TSM 从 OS3 版本换成了 MiApps 版本。`docs` 旧记录中“登录已验证”针对的是 OS3 Account。
MiApps 版账号尚未在 houji 上验证。

## 四、已知依赖与风险（启用前必须处理）

1. **TSM 需要 services 的共享 UID 补丁。** `com.miui.tsmclient` 的 sharedUserId 为 `android.uid.nfc`（aapt2 确认），
   Flyme NFC 为 Meizu 证书签名。没有 services 的 `tsm_shared_uid` 补丁时，预计 TSM 会因签名不匹配而安装失败（未在真机验证）。
   该补丁在 WrionPatcher 授权配置中，ART 匹配未验证，`publishable=false`，本方案没有启用它。
   启用它之前不要期待 TSM 可用。
2. **账号需要 framework 作用域补丁。** 旧记录显示 OS3 Account 依赖 framework 的 `Intent.getMiuiFlags`、
   `getContentResolverForUser` 和包级 `ro.miui.ui.version.*` 读取。framework 作用域在 Builder 中仍为 pending，
   本方案未包含。账号登录可能在缺少这些方法时崩溃。
3. **FindDevice 曾崩溃循环。** 旧记录中，非 priv-app 的 FindDevice 在 `ActivateSIMsJob` 中因 micloud-sdk 无法看见 `com.miui.rom` 而崩溃。
   本插件把它安装为 system_ext/priv-app 并附带权限 XML，行为未验证。
4. **TSM 的 privapp 问题。** 旧记录显示，TSM 在 Flyme 上以特权身份运行而缺少 privapp 权限时会 bootloop。
   本插件的拒绝列表与 allowlist 来自 ColorOS 且未在 houji 上验证。按第 1 项的推断，TSM 当前不会安装，
   但一旦第 1 项补丁被启用，此风险会重新出现。
5. **版本漂移。** 账号、TSM、FindDevice 与 houji 上曾验证的 OS3 版本不同，登录流程需要整体重测。

## 五、与其它插件的关系

- 与 `flyme12.houji-xiaomi-libraries` 无文件冲突：该插件只含相机所需的 MIUI JAR、cameraopt、相机库与 XML。
- 与 `flyme12.houji-whole-app-leica-camera` 无文件冲突：徕卡插件的相册（MIUIGallery）与本插件的载荷不含相同包。
  但载荷的 `product/etc/permissions` 含 `xiaomi-gallery-privapp-permissions.xml`，`system_ext/etc/permissions` 含
  `xiaomi-gallery-shared-libraries.xml`。两者都指向相册包，需在真机上确认无副作用。
- 与 `flyme12.wrion-patcher-houji` 无冲突，但 TSM 与账号的 services / framework 补丁仍未在方案中启用。

## 六、验证状态

已做：

- 载荷 sha256 与分片校验通过；manifest 与分片一致。
- `wrion-tools eng port scheme lint`：0 诊断，7 个模块，20 步，80 个动作。
- `plan --structure-only` 的启用模块包含 `flyme12.houji-xiaomi-apps`。
- aapt2 确认 TSM 的 sharedUserId 与 Account 的权限声明。

未做：

- 未刷入，未在真机上验证任何账号、钱包或 TSM 行为。
- 未验证 MiApps 版本的 Account 与 houji 上 OS3 版本的登录差异。
- 未验证 `set_prop` 在 Flyme `product/etc/build.prop` 上不与已有键冲突（源 ROM 中已确认无这两个键）。
