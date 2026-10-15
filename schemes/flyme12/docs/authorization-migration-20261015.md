# houji 授权插件接入记录

2026-10-15，接续 Claude 的 ID 内存插件首例，WrionTools / WrionPatcher 已完成授权链路扩展。

当前可用的八个宿主配置统一为 `patches/authorized/<scope>/points.json`：services、settings、
systemui、camera、miui-camera、framework、telephony-meizu、telephony-qti。它们由七个 ID
模块提供实现，电话两库共用一个模块。当前共 36 个修补计划、94 个类型化契约入口。

现有 `flyme12.wrion-patcher-houji` 通过 WrionTools 生成器重新保存，只迁移原来的六组选择，
未启用新增 framework / 徕卡 / 钱包增量。后续也应在 WrionPatcher 页面保存，
不要直接手改生成的 plugin.toml。`flyme12.cleanup` 是构建时文件清理，不是设备 Java 插件，保留原状。

具体 Java helper 实现由本机 ID `release` 加密签名，装入 mi_ext 的 `etc/wrion/patch.wrp`。
设备端验签、核对主板授权后内存加载。Builder / ROM 只包含固定运行时、公钥、接口、调用桩、
原厂回退；短的内联修补通过授权门启用。不存在授权的设备保留原厂方法或入口约定的安全回退。

维护者机器上的 `wrion-tools.authorization.json` 提供签发工具、私钥、名单、实现 DEX 根目录和
只读 mi_ext 基线。私钥、名单、明文实现 DEX 均不作为方案资产或 ROM 文件。当前沿用已有小米14
系列名单，没有联网刷新或对外发布；实际打包时将重新签发并核验内置实现摘要与固定公钥。

vendor/odm、HAL、SELinux、权限清单、原厂应用资源和 ART 产物保持各自固定基线职责。
framework 的字段与 Parcel 格式不能按授权开关切换；这部分固定保留，相关适配决策走加密实现。
每次变更 framework / system_server 调用桩都必须重新构建对应 ART 产物并进行启动验收。

已完成的证据：八宿主真实输入重写及重复执行字节一致、分批计划运行时复用、运行时篡改拒绝，
以及真机独立进程七模块内存加载、越权拒绝、加密包篡改拒绝。详细记录在 WrionPatcher 的
`verification/authorized-migration-houji-20261015.md`；操作说明在 WrionTools 的
`docs/09-AUTHORIZED-PLUGINS.md`。

本次未修改设备模块、未刷入完整 ROM。**钱包短信登录已验证成功**；TSM 后续“无法获取账号信息”
仍单列未解决，不因授权迁移而改写其状态。新增组合仍按原处理方案分阶段接入和真机验收。
