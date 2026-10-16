# WrionSchemes

[WrionTools](https://github.com/wwrrj/WrionTools) 的 Android ROM 移植方案仓库。

本仓库维护系统方案、机型定义、插件和资源清单。WrionTools 负责解析、检查与执行，方案负责声明改动内容、分区范围和依赖关系。

## 支持范围

| 方案 | 源 ROM 基线 | 目标设备 |
| --- | --- | --- |
| [flyme12](schemes/flyme12/) | Flyme 12.6.0.0A · meizu21Pro · SDK 36 | Xiaomi 14 · `houji` / `houjiin` |

当前方案允许修改 `system`、`system_ext` 和 `product`。移植阶段将 `vendor`、`odm`、`vendor_dlkm`、`odm_dlkm` 作为保护分区，要求提供目标机型配套基线。

保护分区约束针对移植执行阶段；完整刷机包中的固件映射和刷入操作由机型的 `[flash]` 配置单独声明。

## 使用

```sh
git clone https://github.com/wwrrj/WrionSchemes.git
cd WrionSchemes
```

在 WrionTools 中加载方案仓库，选择源 ROM 和目标机型，核对插件与执行计划后构建。当前方案面向特定基线；仅修改设备代号不能得到可用的新机型适配。

安装 WrionTools 后，也可在本仓库根目录检查方案结构：

```sh
wrion-tools eng port scheme lint schemes/flyme12 --structure-only --json
wrion-tools eng port scheme plan schemes/flyme12 --set device_codename=houji --structure-only --json
```

`--structure-only` 用于检查配置结构，不要求所有资源就绪，也不证明产物可以启动。正式构建前还需检查完整资源、补丁基线和设备依赖。

## 仓库结构

```text
schemes/flyme12/
├── scheme.toml       方案标识、引擎约束、分区边界与参数
├── devices/          机型定义、插件选择和刷机映射
├── plugins/          声明式处理步骤
├── assets/           应用、系统库及资源清单
├── docs/             集成说明与验证记录
└── drafts/           尚未纳入正式流程的方案
```

配置入口为 [scheme.toml](schemes/flyme12/scheme.toml) 和 [houji/device.toml](schemes/flyme12/devices/houji/device.toml)。启用状态以机型配置及插件文件为准。

## 当前插件

| 插件 | 职责 |
| --- | --- |
| `flyme12.cleanup` | 精简源系统应用与相关配置 |
| `flyme12.wrion-patcher-houji` | 集成 services、telephony、Settings 和 SystemUI 的授权补丁，处理相关预编译文件与应用域规则；具体 scope 以插件配置为准 |
| `flyme12.houji-whole-app-meizu-camera` | 集成适配后的魅族相机整包 |
| `flyme12.houji-whole-app-leica-camera` | 集成徕卡相机、相册及配套资源 |
| `flyme12.houji-xiaomi-libraries` | 提供相机依赖的小米系统库 |
| `flyme12.houji-leica-cameraserver` | 应用 cameraserver 流桥接 |
| `flyme12.houji-xiaomi-apps` | 集成小米钱包与账号组件 |

以上插件已列入当前 houji 配置。启用表示参与构建，具体功能的设备验证状态见对应记录。

应用与库采用分片资源和 manifest 管理，分片及完整档案由 SHA-256 校验。修改资源时应同时更新来源、版本、校验值与重建说明。

`wrion-patcher-<机型>` 插件由 WrionTools 的 WrionPatcher 页面生成。请在该页面选择计划并保存；手工修改生成文件会在下次保存时被覆盖。

## 新增或调整适配

1. 在 `devices/<codename>/` 添加机型定义，核对别名、分区映射和固件输入。
2. 声明所需插件、资源与依赖，仅在方案允许的分区内执行修改。
3. 为静态补丁准备对应目标包的审阅基线；授权方案还需配套运行时、接口和发布数据。
4. 检查方案结构、资源完整性和执行计划，再构建与验证。
5. 提交复现步骤及验证记录，分别说明构建结果、启动结果和已测试功能。

修改 framework 时必须核对配套 ART 产物。保护分区中的 HAL、SELinux 策略和设备配置由固定基线提供，不能依靠系统侧文件复制补齐。

## 集成记录

- [houji 集成清单](schemes/flyme12/docs/houji-integration-20261015.md)
- [授权插件接入](schemes/flyme12/docs/authorization-migration-20261015.md)
- [相机整包与资源重建](schemes/flyme12/docs/whole-app-camera-20261015.md)
- [小米应用集成](schemes/flyme12/docs/houji-xiaomi-apps-20261015.md)

记录对应各自的版本与验证时间；当前选择以配置文件为准。账号登录、卡片读写、支付与车钥匙是独立的验收项，不能相互替代。

问题报告请提供 WrionTools 版本、方案提交号、源 ROM、目标机型、插件选择和失败日志。新增适配应保持改动集中，避免将机型差异写入通用执行引擎。
