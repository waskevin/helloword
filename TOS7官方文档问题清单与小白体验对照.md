# TOS 7 官方开发者文档问题清单与小白体验对照

> 本文把《TOS7 新手开发体验记录》中的实际问题，与 TerraMaster 官方开发者文档逐项对照。网页行号按 2026-09-10 抓取结果记录；官方页面更新后，行号可能变化，应同时以章节标题和链接复核。

## 一、总体结论

官方文档覆盖面较广，但目前更像“规范片段的集合”，缺少一条新手可以照着走通的闭环路径。最容易产生的误解是：

> 一个能够被 `dpkg-deb` 构建、被 `dpkg -i` 安装并启动 systemd 的 Debian 包，不一定已经成为可被 TOS App Center 识别、打开、启停和卸载的完整 TOS 应用。

本项目在真实 NAS 上验证到：Deb 包、systemd 服务、HTTP 服务和 Nginx 入口均可分别通过；但直接执行 `dpkg -i` 后，`tos app info helloworld` 仍返回 `app not found`。因此，官方必须把“Deb 功能测试”和“App Center 集成测试”明确分成两层。

## 二、问题等级说明

- **P0**：可能导致开发者无法安装、无法打开或误判可以提交审核。
- **P1**：容易导致首次开发失败或审核失败。
- **P2**：增加理解成本、重复劳动或跨平台失败概率。

## 三、逐项问题与官方修正建议

### P0-01：`dpkg -i` 与 App Center“手动安装”是两个不同入口，但文档没有明确区分

**官方位置与表达**：

- [Local Testing & Debugging](https://help.terra-master.com/developer/development-docs/local-testing)，`Deb Application Testing`，网页行 64–100：使用 `sudo dpkg -i <appid>_<version>_amd64.deb`。
- [TOS 7 App Center](https://help.terra-master.com/docs/TOS7/app-center/)，`Manual Installation`，网页行 37–42：通过 App Center 上传 `.deb` 或 `.tpk`。

**小白困惑**：两个页面都叫“安装”，会自然认为二者结果相同。

**实际原因**：`dpkg -i` 验证 Debian 生命周期和服务；App Center 还涉及应用记录、安装位置、桌面入口、启停状态、日志和代理路由。官方没有说明二者的输出差异，也没有说明本地 `.deb` 如何进入 App Center 流程。

**官方应修改为**：在本地测试页增加醒目标注：

> `dpkg -i` 仅用于 Deb 包功能测试；要验证 TOS 应用注册、App Center 管理、桌面入口和反向代理，必须通过 App Center“手动安装”或开发者平台提供的集成测试环境。

并增加对照表：入口、安装路径、是否注册、验证命令、预期结果。

**理由**：这是本项目最直接的阻塞点，避免开发者把“服务运行”误判为“平台安装完成”。

### P0-02：安装路径描述互相矛盾，未区分“包内路径”和“安装后路径”

**官方位置与表达**：

- [Package Specification](https://help.terra-master.com/developer/development-docs/package-specification)，网页行 60–70：称 TOS 7 实际安装路径为 `/Volume*/@apps/<appid>/`。
- [Deb Complete Examples](https://help.terra-master.com/developer/development-docs/deb-development-specification/complete-examples)，Example 1/2，网页行 60–164：示例目录使用 `/usr/local/<appid>/`。
- [Local Testing & Debugging](https://help.terra-master.com/developer/development-docs/local-testing)，网页行 97–100：又把 `/Volume*/@apps/<appid>/` 定义为运行数据目录。

**小白困惑**：不知道 Deb 控制文件、服务文件、Web 文件究竟应打包到 `/usr/local` 还是 `/Volume*/@apps`，也不知道安装后谁负责搬运文件。

**实际原因**：文档没有明确说明这两个路径分别属于“包内容路径”“平台部署路径”“运行数据路径”还是“持久化用户数据路径”。直接 `dpkg -i` 时，开发者看到的结果可能与 App Center 安装结果不同。

**官方应修改为**：给出一张路径映射表和完整示例：

| 阶段 | 路径 | 用途 | 由谁创建 |
|---|---|---|---|
| 包内 | `/usr/local/<appid>/...` | 应用代码和平台扫描文件 | 开发者打包 |
| 安装后 | `/Volume*/@apps/<appid>/...` | 平台运行目录 | App Center/安装流程 |
| 用户数据 | `/Volume*/<appid>/` | 不应因升级删除的数据 | 应用按规范创建 |

**理由**：路径错误会造成服务找不到文件、Nginx 找不到配置、升级丢数据或平台无法识别。

### P0-03：External WebUI 的入口链路不完整，`path` 的含义不统一

**官方位置与表达**：

- [Complete Examples](https://help.terra-master.com/developer/development-docs/deb-development-specification/complete-examples)，网页行 138–164：`path` 示例为 `/weather/`，并配合 Nginx。
- [Deb Subtypes](https://help.terra-master.com/developer/development-docs/deb-development-specification/subtypes)，网页行 164–212：`path` 最小示例为 `http://${ip}:8686`，同时要求 Nginx、后端端口和 `open_path: true`。

**小白困惑**：`path` 到底是路由、完整 URL、内部端口，还是桌面按钮打开地址？第一次很容易把 URL 写进 `open_path`，或只启动端口而没有 Nginx 配置。

**实际原因**：示例表达了不同场景，却没有在字段表中声明场景差异和优先级。

**官方应修改为**：统一定义：

- `path`：明确写成“路由”或“完整 URL”，两者只能选一种规范。
- `open_path`：明确写成布尔值，只允许 `true/false`。
- 给出从桌面点击到后端服务的完整链路图：App Center 注册 → TOS 入口 → Nginx 路由 → `0.0.0.0:端口` → Web 页面。
- 给出成功响应、最终访问 URL和失败日志位置。

**理由**：本项目曾出现 Nginx 语法通过但 TOS 入口 404；“配置文件存在”不等于“平台入口已注册”。

### P1-04：目录结构、安装脚本和服务文件位置没有形成一个可打包的最终目录树

**官方位置**：[Complete Examples](https://help.terra-master.com/developer/development-docs/deb-development-specification/complete-examples)，网页行 60–164；[Subtypes](https://help.terra-master.com/developer/development-docs/deb-development-specification/subtypes)，网页行 138–212。

**小白困惑**：示例分别展示 `bin`、`init.d`、`images/icons`、`nginx`、`webui.bz2`，但没有一份“最终可上传单包”的完整目录树。

**实际原因**：开发者容易只复制后端服务，漏掉图标、语言、Nginx 或 WebUI 归档；普通 Debian 校验不会发现这些 TOS 约定文件缺失。

**官方应修改为**：提供一个最小可运行模板，并在每个文件旁标注“包内位置 / 安装后位置 / 是否必需 / 被哪个字段引用”。

**理由**：降低复制错误，减少“包能安装但平台不能识别”的情况。

### P1-05：`app.lang` 的文件名、格式和语言标签认知成本高

**官方位置**：[app.lang](https://help.terra-master.com/developer/development-docs/deb-development-specification/app-lang)，网页行 60–80；[Review Standards](https://help.terra-master.com/developer/development-docs/review-standards)，网页行 52–66。

**小白困惑**：页面标题使用 `app.lang`，目录示例使用 `<appid>.lang`；语言标签使用 `zh-cn`、`en-us` 等连字符格式，开发者容易写成 `zh_CN`。

**实际原因**：文件名规则、节点规则、编码规则分散在不同章节，缺少机器可复制模板。

**官方应修改为**：提供官方模板文件，并明确：文件名唯一规则、14 个必需节点、大小写、UTF-8 无 BOM、空值是否允许。将语言列表维护成单一来源。

**理由**：审核会检查节点数量、非空和编码，属于高频低级拒审原因。

### P1-06：`config.ini` 实际是 JSON，扩展名会误导新手

**官方位置**：[FAQ](https://help.terra-master.com/developer/development-docs/faq)，配置文件章节；[Review Standards](https://help.terra-master.com/developer/development-docs/review-standards)，网页行 52–66。

**小白困惑**：看到 `.ini` 会按传统 INI 写成 `[section] key=value`，而平台实际要求 JSON；同时字段约束和类型要求没有集中展示。

**实际原因**：扩展名、格式和校验错误提示不一致，普通编辑器不会主动提示。

**官方应修改为**：改用 `.json`，或保留 `.ini` 但明确写成“JSON 格式配置文件”；提供 JSON Schema 和一条本地校验命令，明确布尔值、数组、字符串不可混写。

**理由**：本项目曾因远程命令引号嵌套导致 JSON 字段未更新，版本不一致，属于新手高概率问题。

### P1-07：版本号在多个位置重复，且本地包名与 Release 包名规则容易冲突

**官方位置**：

- [Package Specification](https://help.terra-master.com/developer/development-docs/package-specification)，网页行 90–128：要求版本同步，Release 资产名不含版本号。
- [Packaging and Verification](https://help.terra-master.com/developer/development-docs/deb-development-specification/packaging)，网页行 89–112：本地构建示例含版本号。

**小白困惑**：不知道本地测试、Release 上传和平台提交分别采用哪个文件名；修复代码后又容易重复使用旧版本号。

**官方应修改为**：分成“本地构建命名”和“Release 上传命名”两栏，并提供自动生成 Release 资产的脚本；提交前自动比较 `control`、`config.ini`、语言文件、Release Tag。

**理由**：版本不一致会被自动拒绝，且平台禁止版本回退。

### P1-08：端口规则和端口冲突检查放置太靠后

**官方位置**：[Deb Subtypes](https://help.terra-master.com/developer/development-docs/deb-development-specification/subtypes)，网页行 188–192；[FAQ](https://help.terra-master.com/developer/development-docs/faq)，端口章节。

**小白困惑**：最小示例常使用 8080，但 NAS 上可能已被 Apache、Docker 或其他应用占用；文档没有在第一个模板中强调监听地址和冲突检查。

**实际结果**：本项目第一次使用 8080，服务显示 active，但实际返回的是已有 Apache 页面。

**官方应修改为**：最小模板使用高位端口；构建或安装前检查端口；强制说明监听 `0.0.0.0`，并给出 `ss -ltnp` 检查方法和保留端口清单。

**理由**：端口冲突会造成“服务正常但访问错应用”的隐蔽故障。

### P1-09：`systemctl active` 被过度暗示为“应用正常”

**官方位置**：[Local Testing & Debugging](https://help.terra-master.com/developer/development-docs/local-testing)，网页行 66–96；[Review Standards](https://help.terra-master.com/developer/development-docs/review-standards)，网页行 141–143。

**小白困惑**：看到 `active` 就认为应用完成；但 systemd 自动重启、端口占用或启动后崩溃都可能造成误判。

**官方应修改为**：标准验收必须同时包含 `is-active`、日志、监听端口、实际 HTTP 请求、返回内容校验和短时间稳定性测试。

**理由**：本项目曾出现服务 active 但 8080 实际访问到 Apache，也曾出现服务刚 active 时端口尚未准备好。

### P1-10：审核标准分散，缺少提交前一键检查

**官方位置**：[Review Standards](https://help.terra-master.com/developer/development-docs/review-standards)，网页行 52–66、141–143、169–220。

**小白困惑**：需要分别检查 JSON、语言、端口、UI、权限、架构、重复 ID、恶意行为和数据包内容，不知道哪些是提交前必查项。

**官方应修改为**：提供官方 `tos-app-lint`，至少检查：包格式、脚本权限、目录树、JSON Schema、14 种语言、版本一致性、架构、图标、Nginx、端口、WebUI 首屏和禁止项。

**理由**：`dpkg-deb` 构建成功只能说明 Debian 包基本合法，不能证明 TOS 平台合规。

### P1-11：发布、真机验证和审核平台是不同阶段，但流程图不够明显

**官方位置**：[Publishing Process](https://help.terra-master.com/developer/development-docs/publishing-process)；[Review Standards](https://help.terra-master.com/developer/development-docs/review-standards)，网页行 169–170。

**小白困惑**：容易以为 GitHub Release、开发者平台提交和 NAS 安装是同一个动作；也不清楚审核人员下载哪个 Release 资产。

**官方应修改为**：首页提供固定流程：开发 → 本地 Deb 测试 → TOS App Center 集成测试 → GitHub/Gitee Release → 创建应用/版本 → 自动校验 → 人工审核，并明确每一步的入口、输入和输出。

**理由**：减少“已上传但不能安装”“已能运行但不能提交”的流程误判。

### P2-12：Windows 开发者缺少官方跨平台打包和验证方式

**实际问题**：Windows 自带 `tar` 生成 `webui.bz2` 时出现路径/压缩结果不可靠，文件存在但归档不可用。

**官方位置**：Deb 打包与校验章节、WebUI 相关章节。

**官方应修改为**：提供 Windows、macOS、Ubuntu 三套脚本，或提供容器化构建命令；每套脚本都应包含归档列表、校验和、Deb 内容、脚本权限检查。

**理由**：当前命令默认 Linux shell，Windows 小白很难判断是自己操作错误还是工具行为差异。

### P2-13：维护脚本权限要求没有在打包前置条件中突出

**实际问题**：Deb 构建时因 `preinst/postinst/prerm/postrm` 权限为 `640` 失败，要求范围是 `0555`–`0775`。

**官方应修改为**：在最小目录模板和构建命令旁直接注明维护脚本必须可执行，并增加构建前自动修复/检查命令。

**理由**：这是 Debian 基础要求，却通常是新手第一次构建才发现的问题。

### P2-14：升级、卸载、权限和数据目录虽有说明，但缺少可观察验收结果

**官方位置**：[Package Specification](https://help.terra-master.com/developer/development-docs/package-specification)，网页行 129–155；[Local Testing & Debugging](https://help.terra-master.com/developer/development-docs/local-testing)，网页行 80–100、189–200。

**小白困惑**：知道要测试升级和卸载，但不知道哪些文件必须保留、哪些文件必须清理、服务用户何时创建、失败时看哪份日志。

**官方应修改为**：为安装、升级、停止、禁用、卸载各提供“操作—预期状态—预期文件—预期日志—失败处理”的验收表。

**理由**：避免应用表面升级成功但用户数据丢失，或卸载后残留服务和端口。

## 四、建议官方新增的最小文档结构

1. **五分钟 Hello World**：只保留一个官方模板，明确 Deb 类型、架构、端口、目录和最终访问地址。
2. **包内路径与安装后路径对照**：解决 `/usr/local/<appid>` 与 `/Volume*/@apps/<appid>` 的歧义。
3. **Deb 测试与 App Center 集成测试对照**：明确 `dpkg -i` 不代表平台注册。
4. **External WebUI 完整链路**：`config.ini`、`webui.bz2`、Nginx、服务、访问 URL、日志和成功响应。
5. **一键校验工具**：替代开发者手工检查十几个页面。
6. **Release 上传模板**：明确本地文件名、Release 文件名、Tag、版本字段的关系。
7. **跨平台构建说明**：至少覆盖 Windows 开发者。
8. **审核前清单**：将 Review Standards 中的硬性拒绝项前置到提交页面。

## 五、给官方的最终修改建议

建议官方把当前文档从“按主题查阅”补充成“按任务完成”，每个任务页面都必须包含：

- 前置条件；
- 可复制的完整文件；
- 执行命令；
- 预期输出；
- 常见错误及原因；
- TOS App Center 验证步骤；
- 提交审核前的机器检查结果。

本项目认为最优先修正的是 P0-01、P0-02、P0-03。它们不是文字风格问题，而是会直接造成开发者得到错误结论：以为应用已经安装、已经注册或已经具备可审核状态。

## 六、当前项目对应状态

截至本次验证：

- Deb 安装和维护脚本：通过；
- systemd 服务：通过；
- `18080` HTTP 服务：通过；
- TOS `8181/helloworld/` 入口：已能返回 `200 OK`；
- App Center 应用注册：未通过，`tos app info helloworld` 返回 `app not found`；
- 审核提交条件：暂不满足，仍需通过 App Center 正式“手动安装”流程验证。

## 参考官方页面

- [Local Testing & Debugging](https://help.terra-master.com/developer/development-docs/local-testing)
- [Package Specification](https://help.terra-master.com/developer/development-docs/package-specification)
- [Deb Development Specification](https://help.terra-master.com/developer/development-docs/deb-development-specification/complete-examples)
- [Deb Subtypes](https://help.terra-master.com/developer/development-docs/deb-development-specification/subtypes)
- [App Language](https://help.terra-master.com/developer/development-docs/deb-development-specification/app-lang)
- [TOS 7 App Center](https://help.terra-master.com/docs/TOS7/app-center/)
- [Review Standards](https://help.terra-master.com/developer/development-docs/review-standards)
- [Publishing Process](https://help.terra-master.com/developer/development-docs/publishing-process)
