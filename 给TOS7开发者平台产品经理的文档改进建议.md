# 给 TOS 7 开发者平台产品经理的文档改进建议

## 邮件主题

关于 TOS 7 Deb 应用开发文档和上架流程的体验反馈与改进建议

## 邮件正文

尊敬的 TOS 7 开发者平台产品经理：

您好！

我们以“完全不了解 TOS 应用开发的新手开发者”视角，按照官方文档从零创建了一个最简单的 Hello World Deb 应用，并在实际 TOS 7 NAS 上进行了安装、启动、访问、升级、卸载和 WebUI 入口验证。同时，我们将开发过程中遇到的疑问与官方文档逐项对照。

感谢官方提供了较完整的开发规范、Deb 生命周期说明、审核标准和发布流程。经过实际体验，我们认为当前文档的主要问题不是内容完全缺失，而是多个关键规范分散在不同页面，部分表达存在歧义，导致新手容易得到错误结论：能构建、能安装、能启动的 Deb 包，可能仍未完成 TOS App Center 注册和平台集成。

下面列出我们认为属于官方文档表达或流程设计的问题，并给出具体修改建议。

## 一、建议优先修正的问题

### 1. 没有明确区分 `dpkg -i` 与 App Center“手动安装”

**官方位置**：

- [Local Testing & Debugging](https://help.terra-master.com/developer/development-docs/local-testing)，`Deb Application Testing`，网页行 64–100。
- [TOS 7 App Center](https://help.terra-master.com/docs/TOS7/app-center/)，`Manual Installation`，网页行 37–42。

**当前表达的问题**：开发文档使用 `sudo dpkg -i` 作为 Deb 安装测试；TOS 用户手册又把上传 `.deb` 称为“手动安装”。两个入口名称相同，但实际承担的职责可能不同。

**实际影响**：我们在 NAS 上使用 `dpkg -i` 后，服务、端口和 HTTP 页面均正常，但 `tos app info helloworld` 返回 `app not found`。这说明命令行 Deb 测试不代表 App Center 已注册应用。

**建议修改**：在本地测试页增加醒目标注：

> `dpkg -i` 仅用于验证 Deb 包生命周期和应用服务；如需验证 App Center 应用注册、桌面入口、启停、卸载、安装位置、日志和反向代理，必须通过 App Center“手动安装”或官方集成测试环境完成。

同时增加“命令行测试 / App Center 集成测试”对照表。

**理由**：这是最容易导致开发者误判“已经可以上架”的问题，应列为 P0 级说明。

### 2. `/usr/local/<appid>` 与 `/Volume*/@apps/<appid>` 的关系不清楚

**官方位置**：

- [Package Specification](https://help.terra-master.com/developer/development-docs/package-specification)，网页行 60–70，称实际安装路径为 `/Volume*/@apps/<appid>/`。
- [Deb Complete Examples](https://help.terra-master.com/developer/development-docs/deb-development-specification/complete-examples)，网页行 60–164，示例使用 `/usr/local/<appid>/`。
- [Local Testing & Debugging](https://help.terra-master.com/developer/development-docs/local-testing)，网页行 97–100，将 `/Volume*/@apps/<appid>/`称为运行数据目录。

**当前表达的问题**：文档没有清晰区分包内路径、平台安装后的运行路径和用户持久化数据路径，也没有说明由谁负责路径转换或目录创建。

**建议修改**：新增路径映射表，明确说明：

| 阶段 | 路径 | 用途 | 创建者 |
|---|---|---|---|
| Deb 包内 | `/usr/local/<appid>/...` | 应用代码、服务、图标、Nginx 等 | 开发者 |
| App Center 安装后 | `/Volume*/@apps/<appid>/...` | 平台运行目录 | TOS 安装流程 |
| 用户数据 | `/Volume*/<appid>/` | 应长期保留的数据 | 应用按规范创建 |

**理由**：路径概念不清会直接导致服务找不到文件、升级数据迁移错误或平台扫描失败。

### 3. External WebUI 的完整入口链路没有讲完整

**官方位置**：

- [Deb Complete Examples](https://help.terra-master.com/developer/development-docs/deb-development-specification/complete-examples)，网页行 122–164。
- [Deb Subtypes](https://help.terra-master.com/developer/development-docs/deb-development-specification/subtypes)，网页行 164–212。

**当前表达的问题**：不同示例中的 `path` 分别出现 `/weather/` 和 `http://${ip}:8686`；`open_path`、Nginx、后端监听端口、`webui.bz2` 之间的关系没有用一条完整链路解释。

**建议修改**：统一字段定义，并提供完整流程图：

`App Center 注册 → TOS 应用入口 → Nginx 路由 → 0.0.0.0:应用端口 → Web 页面`

同时明确：

- `path` 是路由还是完整 URL；
- `open_path` 只能填写布尔值；
- Nginx 配置何时加载；
- App Center 注册成功后如何验证；
- 入口成功和失败时分别查看什么日志。

**理由**：Nginx 配置语法通过，并不代表 TOS 应用入口已经注册。这个区别目前需要开发者自行推断。

### 4. 缺少“最终可上传单包”的完整示例

**官方位置**：[Deb Complete Examples](https://help.terra-master.com/developer/development-docs/deb-development-specification/complete-examples)，网页行 60–164；[Packaging and Verification](https://help.terra-master.com/developer/development-docs/deb-development-specification/packaging)，网页行 89–112。

**当前表达的问题**：目录、配置、语言、图标、Nginx、systemd、WebUI 归档分别在不同章节出现，没有一份从源码目录到最终 `.deb` 文件的完整样例。

**建议修改**：提供一个可下载的最小 Hello World 模板，包含：

- 完整目录树；
- 完整 `DEBIAN/control` 和维护脚本；
- 完整 `config.ini`；
- 完整语言文件；
- 图标；
- systemd 文件；
- External WebUI 的 Nginx 配置和 `webui.bz2`；
- 构建命令、安装命令、验证命令和卸载命令。

**理由**：新手最需要的是“复制后能成功”的基线模板，而不是分别阅读多个规范片段后自行拼装。

### 5. `config.ini` 的格式和字段类型需要集中说明

**官方位置**：[FAQ](https://help.terra-master.com/developer/development-docs/faq)，配置文件章节；[Review Standards](https://help.terra-master.com/developer/development-docs/review-standards)，网页行 52–66。

**当前表达的问题**：文件扩展名是 `.ini`，实际内容却是 JSON；字段类型、必填字段、不同应用类型的字段差异分散在多页。

**建议修改**：

- 明确标注“`config.ini` 是 JSON 格式”；
- 提供 JSON Schema；
- 标注每个字段的类型、是否必填、适用应用类型和默认值；
- 提供官方校验命令，并展示成功和失败输出；
- 评估将文件名改为 `config.json`，或至少在文件名旁长期保留“JSON”提示。

**理由**：扩展名与内容格式不一致，是新手最容易犯错且最容易通过普通编辑器误写的问题。

### 6. 语言文件规则缺少单一、可复制的官方模板

**官方位置**：[App Language](https://help.terra-master.com/developer/development-docs/deb-development-specification/app-lang)，网页行 60–80；[Review Standards](https://help.terra-master.com/developer/development-docs/review-standards)，网页行 52–66。

**当前表达的问题**：页面标题使用 `app.lang`，目录示例使用 `<appid>.lang`；语言标签、编码和必填规则分散，开发者容易混淆文件名、大小写和标签格式。

**建议修改**：提供官方 14 语言模板，明确唯一文件名规则、节点列表、大小写、UTF-8 无 BOM 和非空要求，并让所有页面引用同一个语言清单文件。

**理由**：语言字段属于自动审核硬性检查项，不应依赖开发者从多个页面拼接规则。

### 7. 版本号与 Release 文件名规则需要统一说明

**官方位置**：

- [Package Specification](https://help.terra-master.com/developer/development-docs/package-specification)，网页行 90–128。
- [Packaging and Verification](https://help.terra-master.com/developer/development-docs/deb-development-specification/packaging)，网页行 89–112。

**当前表达的问题**：本地构建示例的文件名包含版本号，而 Release 资产命名要求不包含版本号；版本还要同时写入 `control`、`config.ini`、语言文件和 Release Tag。

**建议修改**：用表格明确区分“本地构建文件名”和“Release 上传文件名”，提供自动生成资产的脚本，并在平台提交前自动比较所有版本来源。

**理由**：这是官方规则之间的呈现冲突，而不是开发者单纯粗心；版本不一致会直接导致自动拒绝。

### 8. 审核标准没有转化为提交前的一键检查

**官方位置**：[Review Standards](https://help.terra-master.com/developer/development-docs/review-standards)，网页行 52–66、141–143、169–220。

**当前表达的问题**：审核条件分散在配置、语言、UI、端口、权限、架构、重复 ID 和禁止项等多个章节，缺少统一的提交前清单或工具。

**建议修改**：提供官方 `tos-app-lint` 或平台预检功能，至少检查：

- Deb 结构和维护脚本；
- `config.ini` Schema；
- 语言文件；
- 版本一致性；
- 架构和最低 TOS 版本；
- 图标和目录；
- Nginx、WebUI 和端口；
- 禁止项和高风险权限；
- 首次加载时间和浏览器控制台错误。

**理由**：`dpkg-deb` 构建成功不代表满足 TOS 审核标准，官方目前把大量检查责任交给了新手开发者。

## 二、官方文档缺少的关键指南

以下不是单个字段的表达问题，而是目前缺少、但对新手和上架成功非常重要的完整指南。

### 1. “从零到审核”的单一路径教程

应从注册开发者账号开始，连续覆盖：创建应用 → 选择 Deb → 创建目录 → 编写配置 → 构建 → NAS 安装 → App Center 验证 → GitHub/Gitee Release → 平台提交 → 处理审核意见。

### 2. “本地 Deb 测试”和“App Center 集成测试”指南

应明确两种测试的目标、入口和预期结果，尤其要解释：

- `dpkg -i` 是否会注册 App Center；
- 如何安装本地 `.deb` 以模拟用户安装；
- 如何确认应用已注册；
- 如何确认桌面入口和 Nginx 路由已生成；
- 如何检查 TOS 应用列表、详情、日志和安装位置。

### 3. 最小 Hello World Deb 模板

官方应提供可以直接下载的最小模板，而不是让新手自行组合多个章节的示例。

### 4. External WebUI 专题指南

需要说明内部 WebUI 与外部 WebUI 的区别，以及 `config.ini`、`webui.bz2`、Nginx 配置、systemd 服务、端口和最终访问地址之间的关系。

### 5. 应用生命周期专题指南

应分别说明首次安装、启用、禁用、启动、停止、升级、回滚、卸载和清理的触发方式、脚本参数、数据保留规则和失败恢复方式。

### 6. 路径、权限和用户模型指南

应解释应用用户、root 限制、`/usr/local`、`/Volume*/@apps`、持久化数据目录、日志目录和升级时可写目录的职责边界。

### 7. 完整错误排查指南

按现象组织，而不是只按组件组织，例如：

- App Center 找不到应用；
- 应用显示已安装但桌面没有入口；
- 服务 active 但页面打不开；
- 页面打开的是其他应用；
- Nginx `-t` 通过但入口 404；
- 升级后数据消失；
- 审核提示配置或语言不合规。

每个问题应给出：现象、原因、检查命令、日志位置、修复方法和成功标准。

### 8. 发布资产和版本管理指南

应明确本地包、Release 资产、Release Tag、平台版本、`control`、`config.ini` 和语言文件之间的唯一版本来源，并提供自动化示例。

### 9. 审核拒绝案例库

应提供真实的“错误包—审核提示—原因—修复前后差异”案例，帮助开发者在提交前理解审核标准。

## 三、希望产品团队优先落地的改进顺序

1. 先补充“`dpkg -i` 与 App Center 手动安装的区别”说明；
2. 统一安装路径、`path`、`open_path` 和 External WebUI 规范；
3. 发布可下载的最小 Hello World Deb 模板；
4. 增加平台预检工具或提交前自动检查；
5. 补齐从注册到审核的端到端教程和故障排查指南。

## 四、测试项目当前结论

本项目目前已验证：

- Deb 包可以安装；
- systemd 服务可以运行；
- 应用端口和 HTTP 页面可以访问；
- TOS 8181 入口可以返回 Hello World 页面；
- 直接 `dpkg -i` 不会自动证明 App Center 已注册应用。

因此，在 App Center 正式“手动安装”并验证应用列表、详情、启停、卸载、安装位置和日志之前，我们不会把该包标记为“已满足审核条件”。

希望以上反馈能够帮助产品团队从新手开发者视角改进文档和平台流程。感谢阅读，也欢迎官方提供适合开发者验证本地 `.deb` 的正式测试方法。

此致

敬礼

测试开发者

## 相关记录

- [TOS7 新手开发体验记录](./TOS7新手开发体验记录.md)
- [TOS7 官方文档问题清单与小白体验对照](./TOS7官方文档问题清单与小白体验对照.md)
