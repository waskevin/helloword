# TOS 7 新手开发体验记录

## 目标

模拟第一次开发 TOS 7 应用的小白开发者，制作一个最小 Deb 单包应用，并记录从文档理解到打包测试过程中容易产生的疑问。

## 记录 1：到底选 Deb 还是 Docker？

**新手疑问**：只是一个 HTML 页面，为什么不能直接上传 HTML？

**实际原因**：TOS 7 新提交应用只接受 Deb 或 Docker。HTML 必须放进应用包，并由 Web 服务提供访问。

**本次决定**：选择 Deb 单包，使用 Python 标准库 HTTP 服务，减少额外依赖。

**文档改进建议**：在 Overview 章节增加“纯静态 HTML 应用最小示例”。

## 记录 2：`x86_64` 和 `amd64` 是不是同一个值？

**新手疑问**：配置中写 `x86_64`，Deb 的 Architecture 也写 `x86_64` 是否正确？

**实际原因**：TOS 配置的 `platform` 使用 `x86_64`，Debian `DEBIAN/control` 的 `Architecture` 使用 `amd64`；两者表达的是同一类 CPU，但字段值不同。

## 记录 3：`.ini` 文件为什么里面是 JSON？

**新手疑问**：文件后缀是 `.ini`，是不是应该使用 `key=value` 格式？

**实际原因**：TOS 为兼容历史配置系统保留了 `.ini` 后缀，但 TOS 7 的 `config.ini` 内容实际使用 JSON。

## 记录 4：`app.lang` 是 JSON 还是 INI？

**第一次尝试**：误写成 JSON，并使用了 `en_US`、`zh_CN` 节点。

**实际原因**：官方 `app.lang` 使用 `[en-us]` 形式的 INI 分节；语言标签是小写连字符格式，而且必须提供 14 个节点。

## 记录 5：为什么 Deb 应用需要 `system_id`、`package` 和 `open_path`？

**实际原因**：`id` 是应用标识，`system_id` 是 systemd 服务标识，`package` 是 Debian 包名，`open_path` 是 Web UI 入口。最小示例中它们可以相同，但概念不同。

## 记录 6：版本号为什么要写这么多次？

**实际要求**：`config.ini`、`app.lang`、Deb `control` 和 Release Tag 必须保持一致。本项目暂定 `1.0.0`。

**风险**：只改其中一个文件会触发自动校验失败。

## 记录 7：为什么要创建专用用户？

**新手误区**：Web 服务用 root 启动最简单。

**实际原因**：TOS 7 要求应用以专用非 root 用户运行，避免应用获得系统级权限。本项目使用 `helloworld` 用户。

## 记录 8：Windows 换行符可能导致什么问题？

**新手疑惑**：脚本在 Windows 上能打开，为什么放到 Linux/TOS 后执行失败？

**实际原因**：TOS 使用 Linux，脚本必须使用 LF 换行，不能使用 Windows 默认 CRLF。后续打包前需要检查并转换脚本换行符。

## 记录 9：监听地址为什么不能只写 `127.0.0.1`？

**实际原因**：`127.0.0.1` 只允许本机访问；TOS 代理或其他设备访问时需要服务监听 `0.0.0.0`。本项目监听 `0.0.0.0:8080`。

## 记录 10：SSH 能登录，为什么 SCP/SFTP 不能上传？

**实际操作**：使用 SSH 登录 NAS 成功，可以执行 `uname`、`tos --version` 和 `dpkg-deb` 检查；随后使用 SCP 和 SFTP 上传源码时，NAS 主动关闭连接。

**新手误解**：能 SSH 登录，就一定能用 SCP 上传文件。

**实际原因**：SSH 登录 shell、SCP 子系统和 SFTP 子系统是不同能力。设备可能只开放命令行登录，未开放文件传输子系统，或者账号的 shell/策略限制了传输。

**影响**：无法直接把开发机生成的源码包传到 NAS，因此不能继续远程 `dpkg-deb` 构建。

**文档改进建议**：开发者环境章节应明确说明 SSH、SCP、SFTP 是否都需要开启，并提供“文件上传失败”的替代方案。

**当前状态**：未安装任何应用，NAS 上的临时上传文件已清理。

## 记录 11：开发者平台登录页不是 NAS 管理页

**实际操作**：当前浏览器打开的是 `developer.terra-master.com`，显示开发者平台登录页；它与 `10.18.8.122` 的 TOS NAS 管理界面是两个不同系统。

**新手误区**：以为在开发者平台登录后就能直接把本地文件传到 NAS 测试。

**实际原因**：开发者平台负责应用登记、版本提交和审核；NAS Web 管理界面负责设备管理和应用安装，两者账号、地址和操作入口不同。NAS 的 SSH 账号密码不能用于开发者平台登录。

**影响**：当前只能确认开发者平台登录页存在，不能通过这个页面完成 NAS 文件上传或安装测试。

**文档改进建议**：发布流程应明确区分“开发者平台账号”“TOS 管理员账号”和“测试设备入口”，并给出从 Release/本地包到测试设备的完整路径。

## 记录 12：`dpkg-deb` 拒绝权限为 640 的维护脚本

**真实错误**：在 NAS 上执行 `dpkg-deb --build` 时提示：`maintainer script 'preinst' has bad permissions 640 (must be >=0555 and <=0775)`。

**实际原因**：`DEBIAN/preinst`、`postinst`、`prerm`、`postrm` 是可执行维护脚本，必须具有执行权限。通过网页上传或 Windows 压缩/解压后，Linux 文件执行位可能丢失，变成普通的 `640` 文件。

**正确做法**：在 Linux/TOS 上构建前执行 `chmod 755 DEBIAN/*`，并检查脚本使用 LF 换行。

**文档改进建议**：Deb 打包章节应把“脚本权限”和“Windows 上传后权限丢失”列为构建前检查项，并给出完整命令。

## 记录 13：服务启动成功，但 8080 返回的是 Apache

**真实结果**：Deb 安装成功，`helloworld.service` 显示 active (running)，进程 UID 为 997 的 `helloworld` 用户；但访问 `http://127.0.0.1:8080/` 返回 Apache 的 `302`，并跳转到 `/lam/`，没有返回应用的 `index.html`。

**新手误区**：看到 systemd 状态是 running，就认为 Web 页面一定可访问。

**实际原因**：服务状态只说明进程启动，不代表端口已被正确代理到该进程。TOS 系统或已有服务可能占用了同一端口，或者 TOS 对该端口配置了反向代理。

**正确做法**：安装前检查端口占用（例如 `ss -tlnp`），选择未占用端口，并核对 `config.ini` 的入口字段、systemd 监听端口和 TOS 代理规则是否一致。

**当前结论**：应用生命周期的“安装”和“启动”通过；“Web 入口访问”未通过，仍不能提交审核。

**进一步定位**：`ss -ltnp` 显示 8080 由 `docker-proxy` 占用；`journalctl` 显示 Python 报 `OSError: [Errno 98] Address already in use`，systemd 随后不断重启服务。`active (running)` 的瞬时状态不能替代实际 HTTP 健康检查。

**修正**：将应用端口改为 18080，并同步修改 `config.ini` 的 `open_path`。

## 记录 14：服务刚启动时立即访问可能得到连接失败

**真实结果**：重新安装 v2 后 systemd 很快显示 active，但紧接着执行 curl 曾得到 `Connection refused`；稍后再次检查，18080 已正常监听并返回 Hello World HTML。

**实际原因**：安装脚本启动服务是异步过程，systemd 进入启动状态和应用真正开始监听端口之间存在很短的时间窗口。

**正确做法**：测试脚本应等待端口就绪后再请求，不能把安装命令刚结束时的一次 HTTP 失败直接判定为应用不可用。

## 当前真实测试结论

- Deb 构建：通过
- SHA-256 生成：通过
- 安装：通过
- 专用非 root 用户运行：通过（UID 997）
- 页面访问：通过（`http://127.0.0.1:18080/` 返回 Hello World）
- 停止/启动：通过
- 原 8080 配置：失败，因端口已被 Docker 占用
- 正式上架：暂未达到条件；还需重新打包为递增版本并创建公开 Release，再通过平台自动校验和人工审核

## 记录 15：修复后的包不能继续使用旧版本号

**真实问题**：端口从 8080 修正为 18080 后，测试包仍然是 `1.0.0`。

**实际原因**：TOS 平台要求新版本号严格递增，历史版本号不能重复使用；Release Tag、`config.ini` 和 Deb `control` 必须同步更新。

**修正**：将修复版本统一提升为 `1.0.1`，并重新构建升级包。

## 记录 16：自动化修改 JSON 时引号可能导致字段未更新

**真实结果**：第一次通过远程命令修改 `config.ini` 时，Deb `control` 已变成 `1.0.1`，但 JSON 中的 `version` 仍是 `1.0.0`；原因是嵌套 shell 命令中的引号被截断，`sed` 报 `unterminated 's' command`。

**正确做法**：修改后必须重新读取并逐项核对 `config.ini`、Deb 控制信息和 `app.lang`，不能只看命令是否执行过。

**最终修正**：已补齐 NAS 上的 `config.ini`，重新生成 `helloworld_x86_64_1.0.1-fixed.deb` 和校验文件。

## 检修后的真实状态

- NAS 安装版本：`1.0.1`
- Deb 元数据版本：`1.0.1`
- `config.ini` 版本：`1.0.1`
- 服务状态：active
- 端口：18080
- Hello World 页面检查：通过
- 仍未完成：开发者平台 Release 上传、自动校验和人工审核

## 第二轮文档复读：容易误解的规范点

### 1. 单包和双包的目录结构混在不同章节

发布流程只强调“Deb 单包是一个 `.deb`”，但 Deb 目录规范又将 `config.ini`、语言文件、图标、WebUI 和服务文件分别放在 `/usr/local/<appid>/` 下；双包章节还把配置包和源包拆开。新手很容易做出一个“能用 dpkg 安装、但不符合 TOS 应用识别结构”的普通 Debian 包。

### 2. WebUI External Open 的字段容易写错

外部打开示例要求 `open_path: true`，同时用 `path: "http://${ip}:端口"`，而不是把 URL 写入 `open_path`。早期最小配置示例容易让人误以为 `open_path` 本身就是 URL。本项目最初就踩到了这个坑，已改为记录问题，但当前包仍需按官方完整 External Open 结构重构。

### 3. External WebUI 不只是一个 HTTP 端口

官方要求同时提供：

- `webui.bz2`
- `nginx/<app_id>.conf`
- `path`
- `open_path: true`
- 后端监听端口

只在 systemd 中启动 Python HTTP 服务并填写端口，虽然可以用 curl 访问，但不等于 TOS 桌面入口可用。

### 4. 服务文件位置和安装位置描述不统一

目录规范要求服务文件放在 `/usr/local/<appid>/init.d/<system_id>.service`，而普通 Debian 生命周期示例又容易让人直接放到 `/etc/systemd/system/`。新手照后者制作的包可能能被 systemd 启动，但平台扫描不到标准应用结构。

### 5. 图标“存在”不等于图标合规

规范要求图标位于 `images/icons/`，并且 `config.ini.icon` 必须写完整匹配路径，例如 `/images/icons/helloworld.svg`。本项目早期把图标放在包根目录、配置也写成 `icon.svg`；这能通过普通文件存在检查，但可能无法通过平台图标校验。

### 6. `app.lang` 文件名存在明显认知成本

发布流程和部分示例称它为 `app.lang`，目录规范及语言规范又要求 `<appid>.lang`，例如 `helloworld.lang`。这不是普通新手能自行推断的细节，应以开发者平台实际模板为最终依据，并在文档中给出明确优先级。

### 7. 14 种语言列表在不同页面出现不同版本

语言规范列出 `hu-hu`、`tr-tr`、`pt-pt` 等标签；审核标准摘要又出现 `ar`、`th`、`vi` 的列表。新手如果直接复制不同章节，可能得到不同的 14 个语言集合。官方应提供唯一的机器可校验语言清单。

### 8. 端口冲突检查写在 FAQ，不在最小模板

8080 在真实 NAS 上被 Docker 占用，造成 Python 服务重启循环。文档虽说明推荐端口范围和保留端口，但最小模板没有自动检查端口或给出端口选择策略；新手通常会直接照抄 8080。

### 9. `active` 状态不能证明应用可用

systemd 的 `Restart=on-failure` 会让失败进程不断重启，短时间内仍可能显示 active。必须同时检查 `systemctl is-active`、`journalctl`、`ss -ltnp` 和实际 HTTP 请求；官方测试章节应把这些作为一个完整检查命令。

### 10. 包名规则在不同章节看起来互相矛盾

发布页示例使用 `<app_id>_<platform>.deb`，打包章节又展示 `<appid>_<version>_<arch>.deb`。前者适用于 Release 资产命名，后者更像本地构建产物命名，但文档没有明确区分“本地文件名”和“最终上传文件名”。

### 11. `low_version` 与实际兼容版本的关系不够直观

文档要求声明最低 TOS 版本，同时又说明 TOS 7.x 保持 ABI/API 兼容。新手可能误以为填当前测试版本即可，实际上应填应用真正需要的最低版本，并在最新 TOS 7.x 上回归测试。

### 12. 真实审核前缺少一份最终包结构示例

当前文档分别解释配置、语言、图标、Nginx、systemd 和 Deb，但没有在发布流程中放一个“最终可上传单包的完整目录树 + 完整包内容”。这是本项目目前最重要的后续验证点。

## 第三轮：问题定位、原因与修改意见

> 行号采用官方帮助中心页面当前网页抓取行号，页面更新后可能变化；同时记录页面标题和章节，方便人工复核。

| 问题 | 官方位置 | 为什么会误解 | 修改意见 |
|---|---|---|---|
| WebUI 外部打开的字段容易写反 | [Complete Examples](https://help.terra-master.com/developer/development-docs/deb-development-specification/complete-examples)，Example 2，网页行 138–152；[Subtypes](https://help.terra-master.com/pl/developer/development-docs/deb-development-specification/subtypes)，行 164–187 | `path` 是访问路径/URL，`open_path` 是布尔开关；但字段名看起来像 `open_path` 应该存 URL。本项目第一次把 URL 写进了 `open_path`。 | 在配置规范前增加字段表：`path = URL 或路由`、`open_path = true`；给出正确和错误各一个完整例子。 |
| 外部 WebUI 需要 Nginx，不是只开端口 | [Complete Examples](https://help.terra-master.com/developer/development-docs/complete-examples)，行 122–164；[Subtypes](https://help.terra-master.com/pl/developer/development-docs/deb-development-specification/subtypes)，行 143–212 | 新手看到后端端口和 `open_path`，容易认为 TOS 会自动把端口接入桌面；实际上还必须有 `webui.bz2` 和 `nginx/<app_id>.conf`。 | 在发布流程中增加“WebUI 四件套”检查：`webui.bz2`、Nginx 配置、`path`、`open_path: true`。 |
| systemd 文件位置容易选错 | [Complete Examples](https://help.terra-master.com/developer/development-docs/complete-examples)，行 124–136；[Subtypes](https://help.terra-master.com/pl/developer/development-docs/deb-development-specification/subtypes)，行 152–155 | 普通 Linux 教程习惯把 unit 文件放 `/etc/systemd/system`，TOS 应用目录规范却要求放在 `/usr/local/<appid>/init.d/`，新手不知道哪个是“打包目录”哪个是“系统安装后的目录”。 | 明确区分“包内路径”和“安装后路径”，并给出单包最终目录树，不要只给片段。 |
| 图标路径不能只写文件名 | [Complete Examples](https://help.terra-master.com/developer/development-docs/complete-examples)，行 124–136、140–152 | `icon.svg` 在包根目录也确实存在，普通检查会通过；但 TOS 要求 `/images/icons/<file>.svg`，属于平台约定路径。 | 在自动校验脚本中同时检查文件实际路径和 `config.ini.icon` 完整匹配。 |
| `app.lang` 的语言标签和文件名成本高 | [app.lang](https://help.terra-master.com/developer/development-docs/deb-development-specification/app-lang)，行 60–80；规则和格式位于同页后续章节 | 页面标题叫 `app.lang`，目录示例又使用 `<appid>.lang`；语言标签是 `zh-cn`，而很多开发者会自然写成 `zh_CN`。 | 提供一个可直接复制的 14 语言模板，并明确文件名优先以官方模板为准；自动检查大小写、连字符和 UTF-8/LF。 |
| 14 种语言列表需要唯一来源 | [app.lang](https://help.terra-master.com/developer/development-docs/deb-development-specification/app-lang)，行 63–80；[Review Standards](https://help.terra-master.com/developer/development-docs/review-standards)，行 62–66 | 审核页摘要和语言页列表可能出现不同标签，开发者复制不同页面会产生不同集合。 | 发布一个官方 `required_languages.json`，文档各处只引用它，避免手工维护多份列表。 |
| 单包和双包的文件名规则看起来冲突 | [Packaging and Verification](https://help.terra-master.com/developer/development-docs/deb-development-specification/packaging)，行 89–112；发布流程页的 Release 规则另有命名示例 | 打包页示例包含版本号，发布页示例又要求 Release 资产名不含版本号；新手无法判断哪个用于本地、哪个用于上传。 | 明确写成两栏：“本地构建文件名”和“Release 上传文件名”，并给出最终唯一推荐值。 |
| Release Tag 与包内版本是多处同步 | [Publishing Process](https://help.terra-master.com/developer/development-docs/publishing-process)，行 73–94、117–128 | 版本号同时出现在配置、control、语言文件、平台表单和 Release Tag，手工修改极易漏改。本项目曾出现 `control=1.0.1` 但 `config.ini=1.0.0`。 | 提供统一版本变量和校验脚本；提交前一次性打印所有版本并比较。 |
| `active` 不等于 HTTP 健康 | [Local Testing](https://help.terra-master.com/developer/development-docs/local-testing)，行 66–76；[Review Standards](https://help.terra-master.com/developer/development-docs/review-standards)，行 141–143 | systemd 的自动重启可能让失败服务短暂显示 active；本项目 8080 冲突时正是如此。 | 官方测试命令应包含 `systemctl is-active`、`journalctl`、`ss -ltnp` 和实际 HTTP 请求四步。 |
| 端口冲突提示出现得太晚 | [FAQ](https://help.terra-master.com/developer/development-docs/faq)，端口问题章节；[Subtypes](https://help.terra-master.com/pl/developer/development-docs/deb-development-specification/subtypes)，行 188–192 | 最小示例通常直接使用 8080，但 NAS 上可能已有 Docker 占用；文档没有在第一个示例中强制提醒。 | 在最小模板中选高位默认端口，并在 `preinst` 或构建校验阶段检查冲突；明确保留端口清单。 |
| 包构建成功不代表平台合规 | [Packaging and Verification](https://help.terra-master.com/developer/development-docs/deb-development-specification/packaging)，行 91–112；[Review Standards](https://help.terra-master.com/developer/development-docs/review-standards)，行 190–220 | `dpkg-deb` 只检查 Debian 包基本结构，不会检查 TOS 的 WebUI、图标、语言、Nginx 或应用字段。 | 提供官方 `tos-app-lint`，把 Debian 校验和 TOS 结构校验合并为一次命令。 |
| 真机、开发者平台、审核平台入口不同 | [Review Standards](https://help.terra-master.com/developer/development-docs/review-standards)，行 169–170；[Publishing Process](https://help.terra-master.com/developer/development-docs/publishing-process)，行 48–60 | 新手容易以为开发者平台上传后就能直接安装到自己的 NAS，实际上提交、测试、审核是不同入口和流程。 | 发布流程增加一张流程图：本地构建 → NAS 真机安装 → GitHub/Gitee Release → Developer Platform 提交 → 审核平台验证。 |
| 审核失败有时限和累计机制 | [Review Standards](https://help.terra-master.com/developer/development-docs/review-standards)，行 223–230 | 新手只看到“修复后重提”，容易忽略 30 天期限、连续失败计数和版本必须递增。 | 在提交按钮旁显示失败处理规则、截止时间和下一版本要求。 |

## 记录 17：能运行的普通 Deb 包不等于 TOS 标准应用包

**真实发现**：最初的包可以通过 `dpkg-deb` 构建并启动 systemd，但目录放置方式、图标路径、语言文件命名、WebUI 入口和 Nginx 配置均不完整。

**原因**：Debian 的“可安装”标准与 TOS 的“可被 App Center 识别并打开”标准是两层规范。前者只关心 `DEBIAN/control` 和文件安装，后者还要扫描 TOS 约定目录和 `config.ini` 关联字段。

**修正**：增加官方 External WebUI 所需的 `bin/`、`init.d/`、`web/`、`nginx/`、`images/icons/`，并将 `config.ini` 改为 `path` + `open_path: true`。

## 记录 18：Windows 本地生成 `webui.bz2` 不可靠

**真实结果**：尝试在 Windows 使用系统 `tar` 根据 Web 目录生成 `webui.bz2` 时，命令因路径访问失败退出，留下的文件只有 42 字节，不能视为有效前端归档。

**原因**：TOS 文档给出的打包命令基于 Linux shell 和 GNU tar；Windows 的 tar 实现、路径编码和压缩参数行为可能不同。文件存在不代表归档有效。

**修改意见**：官方应提供跨平台打包脚本，或直接提供 `webui.bz2` 生成工具；文档还应给出 `tar -tjf webui.bz2` 的完整验证命令。

**当前处理**：本轮只完成了标准目录和配置重构；`webui.bz2` 必须在 NAS/Ubuntu 环境重新生成并验证后，才能进入最终打包。

## 记录 19：官方 External WebUI 示例在真机上的入口仍有隐藏前置条件

**真实结果**：NAS 上安装 1.0.2 后，以下检查通过：

- `dpkg -i` 升级通过
- systemd 服务 active
- `http://127.0.0.1:18080/` 返回 Hello World
- `nginx -t` 通过

但访问 `http://127.0.0.1/helloworld/` 时，TOS Nginx 将请求重定向到 `8181/helloworld/`，随后返回 404。

**原因分析**：Nginx 配置文件语法正确，不等于 TOS 应用代理已经注册。官方示例展示了 `nginx/<app_id>.conf` 的内容，但没有在同一处说明配置何时被 TOS 加载、应用入口何时注册，以及 `path` 应使用路由还是完整 URL。

**修改意见**：文档应给出“安装后验证代理注册”的完整步骤，包括：配置文件加载位置、Nginx reload/注册触发条件、最终访问 URL、预期状态码和日志位置；同时统一 `path` 的格式定义。

**当前结论**：后端服务可用，TOS WebUI 代理入口仍未通过，不能把 `nginx -t` 通过当作平台入口通过。

## 记录 20：手工放入 `conf.d` 仍不能代替 App Center 注册

**真实结果**：确认 `/etc/nginx/nginx.conf` 包含 `conf.d/*.conf`，修正配置后 `nginx -t` 通过；但访问 TOS 的 8181 应用入口仍返回 404，且 `tos app info helloworld` 返回 `app not found`。

**根因**：TOS App Center 不只依赖 Nginx 文件。它还需要在自己的应用数据库/注册目录中记录应用，并把应用安装到平台管理的路径（现有应用使用 `/Volume1/@apps/<appid>/`）。手工 `dpkg -i` 和手工复制 Nginx 配置只能验证后端与 Nginx 语法，无法完成平台注册。

**修改意见**：文档应明确禁止把“手工 dpkg 安装”当成完整 App Center 测试；应提供官方本地安装/注册命令或说明如何通过 App Center 安装本地包，并明确注册后生成哪些文件、链接和数据库记录。

**当前结论**：TOS 原生服务测试通过；App Center 注册和桌面入口测试仍需要使用平台正式安装流程。

## 记录 21：官方文档是否说明了“手工 dpkg 安装”和“App Center 注册”的区别

### 核查结论

官方文档**分别提到了相关步骤，但没有把两者的边界讲清楚**：

| 已提及内容 | 官方位置 | 能说明什么 |
|---|---|---|
| Deb 本地测试使用 `dpkg -i`、`systemctl`、`curl`、`dpkg --purge` | [Local Testing & Debugging](https://help.terra-master.com/developer/development-docs/local-testing)，网页行 44–100 | 官方确实把手工安装定义为 Deb 的本地功能测试方式，主要验证安装脚本、服务、端口、WebUI 和卸载清理。 |
| Deb 安装阶段包括解包、执行 `postinst`、启动服务 | [Package Specification](https://help.terra-master.com/developer/development-docs/package-specification)，网页行 60–70 | 说明 `dpkg` 能验证 Debian 生命周期，但不等于已经完成 TOS App Center 的平台注册。 |
| TOS App Center 的“手动安装”入口支持上传 `.deb` 或 `.tpk` | [TOS 7 App Center](https://help.terra-master.com/docs/TOS7/app-center/)，网页行 37–42 | 这才是面向 TOS 用户的应用安装入口；文档没有明确说明它会额外完成哪些注册动作。 |
| App Center 可查看安装位置、端口、启用状态和安装日志 | [TOS 7 App Center](https://help.terra-master.com/docs/TOS7/app-center/)，网页行 51–59 | 这些平台管理能力不是普通 `dpkg -i` 命令本身提供的，因此应单独验证。 |

### 本项目实际遇到的对应问题

我们在 NAS 上直接执行 `dpkg -i` 后，服务和 `18080` 端口可以正常工作，但 `tos app info helloworld` 返回 `app not found`，TOS 的 `/helloworld/` 入口也返回 404。这与官方本地测试文档并不矛盾：文档要求验证服务和 `curl localhost:<port>`，并没有承诺命令行安装会建立 App Center 的应用记录、桌面入口或反向代理路由。

### 文档中最容易让新手误解的地方

1. “手工安装”在开发文档中指 `sudo dpkg -i`，在 TOS 用户手册中又指 App Center 的“Manual Install”按钮；同一个中文概念对应两个不同入口。
2. Package Specification 行 64 写实际安装路径为 `/Volume*/@apps/<appid>/`，但 Deb 目录示例使用 `/usr/local/<appid>/`。文档没有明确区分“包内路径”和“App Center 安装后的运行路径”。
3. 文档没有给出 App Center 注册成功后的可观察结果，例如 `tos app info` 应返回什么、应用记录存在哪里、Nginx 配置何时被加载。

### 修改建议

官方应在 Local Testing 页面增加醒目的边界说明：

> `dpkg -i` 仅用于验证 Deb 生命周期和应用进程；若要验证 App Center 识别、应用注册、桌面入口、安装位置和反向代理，必须通过 TOS App Center 的“手动安装”流程，或使用开发者平台提供的 TOS 7 开发者虚拟机/注册测试流程。

同时建议增加一份“命令行测试”和“App Center 集成测试”的对照表，并明确包内 `/usr/local/<appid>/` 与安装后 `/Volume*/@apps/<appid>/` 的关系。

**当前判断**：这不是我们单纯修正 systemd 或 Nginx 语法即可解决的问题，而是文档没有明确说明“Deb 包功能测试”和“TOS 平台集成测试”是两层测试。当前包可以继续做本地 Deb 测试，但在 App Center 正式安装验证通过前，不应提交审核。

## 记录 22：补充验证——代理入口可以恢复，但应用仍未注册

在 NAS 上继续检查后得到以下结果：

- `dpkg -s helloworld`：安装状态为 `install ok installed`，版本 `1.0.2`。
- `systemctl is-active helloworld`：`active`。
- 服务监听：`0.0.0.0:18080`。
- `http://127.0.0.1:8181/helloworld/`：返回 `200 OK`，Hello World 页面可打开。
- `tos app list` 中没有 `helloworld`，`tos app info helloworld` 仍返回 `app not found`。
- `tos app install --help` 显示的参数是 `<app-id>`，说明该 CLI 安装的是应用中心中已有的应用，不是本地 `.deb` 文件。

### 新结论

此前的 404 主要是临时 Nginx 配置未加载导致；补充配置后，TOS 入口可以访问。但这次验证也进一步证明：

> “Nginx 入口可访问”与“应用已经被 TOS App Center 注册”是两个独立条件。

当前包已经通过 Deb、systemd、HTTP 和 Nginx 入口测试，但尚未通过 App Center 注册/管理测试。下一步应使用 TOS App Center 页面中的“手动安装”上传本地 `.deb`，再检查应用列表、详情页、启停、卸载、安装位置和日志，而不是继续用 `dpkg -i` 模拟该流程。

## 待验证问题

- 官方 `config.ini` 的完整字段和 WebUI 字段命名需要与模板逐项核对。
- 14 个语言节点已按官方标签补齐，但非中文内容暂用英文占位，正式上架前应人工翻译。
- 需要在 Ubuntu 22.04/TOS 7 真机上验证安装目录、systemd 服务和端口代理行为。
- 需要用 `dpkg-deb` 构建并生成最终 Release 文件名和 SHA-256 文件。
