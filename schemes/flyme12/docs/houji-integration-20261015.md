# houji / Flyme 12.6 处理方案

更新日期：2026-10-15。适用设备：Xiaomi 14 / houji，系统：Flyme 12.6.0.0A / Android 16。

本方案整理已完成的真机适配、文件归属和接入顺序。**钱包短信登录已验证成功**；
交通卡等入口的后续账号授权仍未完成。不能把旧 Claude 会话结尾的登录失败当作当前状态，
也不能把登录成功写成钱包所有功能均已恢复。

现有两个执行插件保留，新增选择清单位于 `drafts/houji-integration/selection.toml`，
尚未作为可执行插件启用。随后已完成授权迁移：现有 WrionPatcher 六组选择由生成器
改为授权配置，勾选和启用状态不变。最新 Builder 已加入 framework / miui-camera 作用域，
但新增 framework 组合仍须配套 ART 构建与整包验收，不能据此直接启用在研增量。
迁移边界和签发流程见 [授权插件接入记录](authorization-migration-20261015.md)。

## 证据与版本

1. 历史来源：Claude 会话 `72b21b5d-5f8b-4aa6-8e75-a34ec90d99e6`。
2. 最新来源：Codex 对话「继续处理指定会话记录」，ID
   `01a13966-2ae1-77f2-9d69-dc8d2327e4d4`。
3. Codex 完成的登录修复：WrionPatcher 提交 `947eb6a`、`9fa37cb`。对话内用户明确确认
   “登录成功”，对应记录为 `verification/xiaomi-finddevice-window-houji-20261014.md`。
4. 同一对话随后继续处理 TSM 入口；最后停在“无法获取账号信息”，并因 USB 断连未能安装
   最后准备的回调诊断。相关修改仍有未提交文件，不能用 `9fa37cb` 代表全部后续修改。
5. 本次只读核对的设备模块文件映射见
   `../devices/houji/integration-inventory-20261015.json`。记录的是当前部署状态，
   不意味着其中每项均已通过功能验收。

以下路径以本机 `D:/Prjs/WrionPatcher/` 为根引用上游源码和验证记录。

## 最新功能状态

| 项目 | 已有证据 | 接入判定 |
| --- | --- | --- |
| 原厂钱包、TSM、账号 APK 与原签名 | 已安装并启动；没有修改三者的 APK | 按原文件导入，记录哈希及公开签名摘要 |
| 小米账号短信登录 | 用户在 FindDevice / 云 SDK 修复后确认成功 | 纳入已验证基础能力 |
| FindDevice 服务与云 SDK 依赖 | 主进程和状态查询服务运行，实际 SDK 依赖图含 MIUI JAR | 纳入登录依赖；保留原厂状态查询 |
| framework 接口、进程限定属性和电话库 Android 回退 | 结构检查、启动、登录流程实测 | 与配套 ART 产物整体接入 |
| 安全窗口参数传递 | 本地 52 项测试与真机 Parcel / copyFrom 检查通过 | 属于系统兼容实现；真实锁定窗口、导航及 FOD 联动仍单列验收 |
| TSM 账号可见性 | 40 项测试通过；真机 TSM 可见，NFC 包仍不可见；交通卡进入 TransitEntryActivity | 后续已验证增量，尚未提交；独立冻结源码后接入 |
| NFC eSE 路由查询 | 26 项测试；真实 NXP 查询返回路由 1；“默认钱包”提示消失 | 仅导入实际查询接口，不宣称默认钱包角色或支付已配置完成 |
| TSM 共享 UID 自调用包名顺序 | 真机不再出现此前 NFC 包误识别 | 已验证增量；保留真实包成员与签名检查 |
| Flyme VAID 接口适配 | 真机已读到本机提供器返回的真实值 | 只证明接口读取成功；未解决后续账号授权 |
| 交通卡、门卡、Mi Pay、车钥匙 | 后续曾进入账号信息错误 | 未完成，不作功能可用承诺 |

## 分区和文件处理原则

- 插件只写 `system`、`system_ext`、`product`。vendor / odm 的文件由用户维护的固定基线提供，
  不改 `scheme.toml` 的保护分区约束。
- Magisk 的 `system/framework/x.jar` 对应 `/system/framework/x.jar`，stage 使用
  `stage:system/system/framework/x.jar`；`system/system_ext/...`、`system/product/...`
  分别对应独立分区 `stage:system_ext/...`、`stage:product/...`。
- 不导入 `.replace`、`module.prop`、`customize.sh`、`post-fs-data.sh`、`service.sh` 或
  `sepolicy.rule` 作为 ROM 文件。脚本产生的运行时行为必须逐项实现和验收。
- Settings、SystemUI、魅族相机模块都带 `plat_seapp_contexts`；不能按模块顺序整文件覆盖。
  沿用生成插件中包名限定的 seapp 定点合并，保留源 ROM 其余规则。
- 新增文件分别配置真实分区下的 uid/gid/mode、file_contexts 和必要 XML 权限清单。
  ZIP 权限和模块目录的 SELinux 标签不能代替最终镜像属性。
- 包名、原签名、真实 UID / 用户和设备限制沿用已验证策略。钱包、TSM、账号、FindDevice
  保留原包和原签名；设置、SystemUI、魅族相机、小米相机等已有静态补丁产物分别处理签名。

## 逐模块归属

| 模块 | 分区内容与处理方式 |
| --- | --- |
| flyme_fuxi_vibrator_hal | vendor / odm 内容已收集到固定基线；另行处理触感设置同步，见运行时依赖 |
| wrion_static_composer_als | vendor 二进制已进入固定基线；另有 `ro.vendor.display.alsid=houji` 运行时属性依赖 |
| wrion_static_flyme_ril_meizu | 现有生成插件对 system 下 meizu-telephony-common.jar 静态修补 |
| wrion_static_flyme_ril_qti | 现有生成插件对 system_ext 下 qti-telephony-common.jar 静态修补 |
| wrion_static_settings | 沿用现有补丁选择、重签名及包名限定 seapp 规则 |
| wrion_static_systemui | 沿用现有亮度、AOD、指纹 / mBack 补丁；另有 sysfs 运行时依赖 |
| wrion_static_camera | 沿用现有补丁、重签名、移到 priv-app 和权限 XML；sepolicy 独立处理 |
| wrion_leica_stream | system/bin/cameraserver；按已审阅 ARM64 指令方案修补，不以任意 ELF 整文件覆盖代替 |
| wrion_miui_camera | system 下相机、相册及其库；system_ext 下 MIUI / 云 SDK / cameraopt 依赖；product 下 houji.xml |
| wrion_static_services | 在现有基础方案上组合已验证钱包策略；由一次组合构建产出配套 services.jar |
| wrion_static_framework | framework.jar、6 个主 / 扩展启动镜像文件与 75 个 system_server 产物，共 82 个文件；整体处理 |
| wrion_xiaomi_account_probe | 原厂 product/app/MIUIXiaomiAccount；正式接入时去掉 probe 命名 |
| wrion_xiaomi_wallet_probe | 原厂 product/app/MIpay；正式接入时去掉 probe 命名 |
| wrion_xiaomi_tsm | 原厂 product/app/MITSMClient 与包名限定权限 XML；依赖对应 services 策略 |
| wrion_xiaomi_finddevice | system_ext 下 FindDevice、凭据 SDK JAR / XML、3 个原生库及权限 XML |
| wrion_systemui_doze_sepolicy | 没有分区文件；只有独立 SELinux 规则和 sysfs 权限调整，不作为普通文件复制插件 |

## 建议的插件拆分

现有 `flyme12.cleanup` 保留。`flyme12.wrion-patcher-houji` 继续由 WrionTools 的
WrionPatcher 页面生成；不直接手写新增步骤，以免下次保存丢失配置。

后续新增内容按以下职责接入，ID 为设计值，**尚未注册或启用**：

| 建议 ID | 内容 | 执行前提 |
| --- | --- | --- |
| flyme12.houji-xiaomi-libraries | MIUI JAR、云 SDK、miuisystem、cameraopt、凭据 SDK 与对应 XML / SO | 原文件哈希、库依赖、路径及镜像属性校验 |
| flyme12.houji-leica-camera | 已验证 cameraserver 指令修补、相机 / 相册与 houji.xml | 小米相机作用域及 ELF recipe 已接入 Builder |
| flyme12.houji-xiaomi-apps | 原厂账号、钱包、TSM、FindDevice 及精确权限清单 | 同批系统补丁、共享库与 ART 构建成功 |
| flyme12.houji-runtime | 触感设置同步、Composer 属性、相机附加 ID、AOD sysfs 权限 | 运行时实现经过独立设计与真机验收 |

framework 和 services 的静态修改由同一套补丁选择生成，不再在另一个插件中覆盖不同版本。
ART 构建作为所有影响 boot class path / system_server 的修改完成后的最终阶段。
依赖通过实际 `needs`、执行顺序和冲突检查表达，不能只靠这张表中的排列顺序。

## 系统补丁选择

选择表见 `../drafts/houji-integration/selection.toml`。它是接入清单，不是当前 Builder 插件 Schema。

登录已验证版本以 WrionPatcher `9fa37cbd9c33d2d5492b21512dce29cc5c1464c3` 为基准：

- framework 使用 `patches/flyme/framework/finddevice_window/points.json`，包含四项：
  `framework.xiaomi_app_compat`、`framework.xiaomi_account_telephony_fallback`、
  `framework.xiaomi_account_device_name`、`framework.finddevice_window_transport`。
- services 使用 `patches/flyme/services/wallet/finddevice_window/points.json`，组合原有指纹、
  人脸、色彩、无线反充、手势，以及 `services.xiaomi_tsm_shared_uid`、
  `services.xiaomi_account_library`、`services.finddevice_window`、
  `services.xiaomi_cloud_builtin_dependency`。
- 后续可见性、NFC 查询、自调用包名顺序、VAID 四项仍在未提交工作树中。
  单独记录源码哈希和证据；整理成独立提交、重建 helper 后才能作为可重复发布的输入。
- 不从当前合并 helper.dex 随意挑选部分计划后直接发布；helper 中可能仍包含后续或诊断代码。
  必须由审核后的闭合源码集合重建，核对导出的 helper API 和结构指纹。
- 不选择临时异常 / 页面关闭 / 账号回调诊断。独立计数限流试验已证明不是根因并被撤回，
  不将它恢复进方案。

### 可复核的登录成功版本

本机备份 `D:/Prjs/WrionPatcher-device-backups/resume-20261014/wrion-before-tsm-entry-diag.tar`
保留登录成功后的配套模块。本次已直接从归档中读取 JAR 并核对到以下哈希：

| 文件 | SHA-256 |
| --- | --- |
| framework.jar | `570e4954e8039d749caacf8588de80d20321f16c56f43dd9f8a66ea00c34ed9e` |
| services.jar | `57d0d75436fcdae5a10e2756b43df7ea780d0d3108f9e60bcf9ce9b954f1085e` |

备份用于复核或整体回退，不直接作为适配其他 ROM 的输入包。本次设备实际 JAR 已是之后的
TSM 入口候选版本，哈希记录在 inventory JSON 中，不能称为上述登录成功版本的字节副本。

## ART 与预编译处理

只改 framework.jar 会导致启动镜像和 system_server 预编译失配，这是此前全局性能下降的
已验证原因。当前方案的 `drop-stale-preopt` 不足以承担 framework 更新。

接入必须满足：

1. 先完成 framework、services、电话 JAR 等最终内容和 JAR 内 DEX 对齐。
2. 使用与目标系统一致的 ART 工具、APEX、boot class path、ISA 及编译参数；依据最终 JAR
   校验和重新编码 boot-image / services profile。
3. 从同一输入集合生成 6 个启动镜像文件及本基线对应的 75 个 system_server 编译产物。
   75 是当前基线观察值，换基线后由实际编译清单核验，不能沿用猜测。
4. 校验产物齐全、DEX 与 bootclasspath 校验和一致，再整体写入 stage。失败时停止发布，
   保留原配套集合，不能只输出新的 JAR。
5. 若使用设备作为编译节点，必须使用已验证的私有挂载命名空间、独立临时目录和退出清理；
   不能直接执行现有 Magisk 安装器作为 Builder 插件，也不能让绑定挂载传播到全局。
6. 真机验证 zygote 的可执行 boot.oat 映射、system_server 的可执行 services.odex 映射、
   无匿名 DEX 副本、正常开机及性能；允许一次性的应用后台优化，不接受每次启动都失配。

历史设备重编译发生在 Magisk 安装阶段。ROM 内直接打包后的启动时序与模块挂载不同，
必须对最终镜像重新验收，不能照搬模块测试结论。

## 原厂应用与共享库

按模块 inventory 中的相对路径导入应用和依赖，保持账号、钱包、TSM、FindDevice APK 字节与
签名不变。不要把这些原厂 APK 送入默认会重签的 `wrion_patch` APK 流程。

- TSM 保留 `/product/app/MITSMClient`、`android.uid.nfc`，以及原模块的六项明确拒绝权限。
  不改 NFC 原共享 UID 的签名记录；安装准入依赖包名、路径、原签名和设备限定策略。
- FindDevice 保留 `/system_ext/priv-app/FindDevice`；权限 XML 的五项允许、九项明确拒绝
  全部保留。云 SDK 实际类加载图要包含 `micloud-sdk → com.miui.rom / com.miui.system`。
- 账号、FindDevice 的库补充与云 SDK builtin JAR 依赖图修复是两项独立策略，均要验证。
- 小米相机依赖的 MIUI JAR / 云 SDK 也被账号栈使用，应由公共依赖插件提供；即便不选相机，
  账号插件仍必须能获得它所需的共享库。
- 原生库的 public.libraries、权限 XML、privapp 规则、context 与 uid/gid/mode 一起核对。
  不把 `com.miui.rom` 加入全局 boot class path。

## 运行时依赖及独立 SELinux 项

vendor / odm 文件已收集不等于以下行为已写入基线：

| 来源 | 行为 | 本次处理 |
| --- | --- | --- |
| Composer system.prop | `ro.vendor.display.alsid=houji` | 记录依赖；确认属性加载时序和既有属性覆盖机制后实现 |
| 相机 post-fs-data.sh | 给 `vendor.camera.aux.packagelist` 保留原值并追加 `com.android.camera` | 不用固定值覆盖其他包；需有受控实现 |
| 振动 meizu-haptic-sync.sh | 同步系统触感强度到 `/mnt/vendor/flyme_haptic` 与 `vendor.flyme.haptic` | `/mnt/vendor` 为运行时数据目录；不能误作 `/vendor` 文件打包 |
| doze service.sh | 根据真实 SystemUI UID 设置 doze_brightness 节点所有权和 0664 | 不固化本机 UID；按最终镜像注册的应用身份实现 |

`flyme_fuxi_vibrator_hal`、`wrion_static_camera`、`wrion_systemui_doze_sepolicy` 三份
sepolicy.rule 维持独立清单；用户自行决定 vendor sepolicy 集成。本方案不通过放宽保护分区
来加入这些规则。用户的 vendor/odm 收集报告在
`/mnt/e/TIK/xm14/magisk_vendor_odm_collection_20261015/`，其中含三份原文。

## 尚未解决或尚未验收

1. **TSM 无法获取账号信息**：最新对话中连续请求来自 TSM 自身；独立计数仍触发原阈值，
   所以限流不是根因。后续应观察原厂账号回调是否要求重试，保持其原结果与授权流程。
   最后准备的临时回调诊断尚未安装，不能写成已获得回调结论。
2. **交通卡、门卡、Mi Pay、车钥匙**：只记录入口与错误变化，未完成端到端功能验收。
3. **安全窗口**：Parcel、复制和本地策略已测；真实窗口显示、实体 / 手势导航与 FOD 恢复
   尚需单独验证，登录成功不能代替此项。
4. **相机**：已验证照片 / 录像链路按 `verification/leica-camera-modes-houji-20261013.md`
   的模式与样片描述。真人虚化质量、低光稳定性、更多录像规格等不扩大结论；变焦触感的
   源码仍有未提交改动，按实际验收版本单独固定。
5. **ROM 集成**：当前 Builder 对新 scope / ART 阶段的支持、生成器更新和依赖插件是接入
   工作，不因 Magisk 真机验证通过而自动完成。

## 实施顺序与完成标准

1. 固定登录成功的 `9fa37cb` 及上游测试证据；后续四项增量独立固定源码和测试。
2. 更新 Builder 内嵌 WrionPatcher 和通用作用域、生成器支持，保留原机型选择；不要只改方案
   manifest 填写当前引擎不支持的 `framework` / 小米相机 scope。
3. 建立精确的原厂应用 / 共享库资产清单和分片哈希；从本地审核输入准备资产，不提交原始 ROM、
   账号数据、原始日志或私有签名材料。
4. 建立 ART 配套构建 / 导入的完整输入校验和原子发布，再在 WrionPatcher 页面生成新选择。
5. 接入运行时依赖，核对最终镜像权限与 SELinux；保护分区继续由固定基线提供。
6. 执行结构 lint、完整资产 lint 和 houji plan，检查 active/skipped、写分区、步骤依赖与冲突。
7. 构建最终镜像后按功能表真机回归，记录镜像 / JAR 哈希。只有该阶段通过后才能声称
   “WrionSchemes 已集成可用”，此前只称“处理方案与验证清单已整理”。

本次整理没有运行 ROM 打包、安装模块、重启设备、修改其他仓库、提交或推送。

## 本次整理校验

使用本机 WrionTools 已构建的 `harusora-rom-builder.exe` 完成：

- `eng port scheme lint <flyme12> --structure-only --json`：无诊断。
- `eng port scheme lint <flyme12> --json`：完整资源检查无诊断。
- `eng port scheme plan <flyme12> --set device_codename=houji --set source_version=12.6 --workers 4 --structure-only --json`：
  仍只启用原有两个插件，共 41 个动作；没有保护分区写入。
- 草案 TOML / JSON 解析、16 个模块路径映射、`.replace` 排除、哈希格式和 README 链接检查通过。
- 所列 14 个登录基础计划均在指定提交中存在；后续 4 项增量记录了当前源码 SHA-256。

这些检查验证方案仓库没有被本次文档和草案破坏，不代表新增 framework / ART 接入已实现，
也不构成新的钱包功能真机验收。
