# 文件说明

按目录列出每个文件的作用。行数为当前版本实测。

---

## 根目录

| 文件 | 作用 |
|---|---|
| `package.json` | 包元信息。`dsh.bundle.patch` 指向 `cordis.patch.yml`（宿主加载入口）、`dsh.client.inject` 声明前端要注入的 6 个 `@deepseek-ai/dsh-client-*` 平面、`dsh.install` 提供三种安装方式（tarball / zip / git）、`postinstall` 跑 `scripts/ensure-profile-bundle.mjs` |
| `cordis.patch.yml` | Cordis 插件清单。声明本体插件 `dsh-purge` 与演练台子插件 `dsh-purge-redteam` 的 insert 段、默认配置（`enabled`、`autoApplyOnStart`、`autoUpdateOnStart`、`autoRevertOnMissing`、`postPromptOrder`、`postPrompt`） |
| `client.js` | **生成物**（9354 行）。Web 前端 bundle，`window.__ModuleLoader__.load({ id: "dsh-purge" })` 开头。含侧栏两页签（清洗 / 演练台）的 React 组件、中英 i18n 文案（`welcome.*`、`patch.1`~`patch.90`、`dock.*`）、以及 `scripts/build-client.mjs` 从 `lib/redteam/client.js` 抽进来的演练台 UI。由 `scripts/patch-client-dock.mjs` + `scripts/build-client.mjs` 生成，不要手改结构 |
| `screenshots.json` | 界面预览图清单，插件市场展示用 |
| `bin/dsh-purge.js` | CLI 入口（171 行）。解析 `--apply` / `--revert` / `--uninstall` / `--status` / `--edit` / `--help`，转调 `lib/core.js`、`lib/rules.js`、`lib/reapply.js`。对应聊天里的 `/purge` 命令 |
| `README.md` / `README.zh-CN.md` / `README.ja.md` | 英 / 中 / 日三份说明：安装分步、界面、工作原理、还原、更新、适配范围 |
| `NOTICE` | Apache-2.0 要求的来源与版权声明 |
| `LICENSE` | Apache License 2.0 全文 + 版权行 |
| `LICENSE-COMMERCIAL.md` | 商业授权与二开署名说明 |
| `release-notes.md` | 历史版本变更记录 |
| `.npmignore` / `files` 字段（在 `package.json`） | 控制发包内容：只带 `lib`、`bin`、`docs`、`presets`、`scripts`、前端 bundle，明文提示词源不发包 |
| `.github/workflows/release.yml` | GitHub Actions 发版流水线 |
| `docs/FILES.md` | 本文件 |

---

## lib/ —— 主逻辑

### 注入与身份

| 文件 | 行数 | 作用 |
|---|---|---|
| `identity.js` | 1078 | 身份段装配。定义 `GLOBAL_PROMPT_ORDER = -100000`（注入正文排 system prompt 最前）、`OBJECTIVE_SECTION`、`REDTEAM_ROLE_SECTION`、`IDENTITY_CARD_PREFIX`；`isPluginPromptSection()` 识别 `dsh-purge*` / `dshpg-*` 段；`escapeDshInterpolation()` 把非法 `{{...}}` 拆成 `{ {...}`（DSH 只注册 `cwd/model/provider` 三个变量，其它会让整轮对话失败）；`normalizeChatmlLayout()` 规整 ChatML 布局 |
| `anchor.js` | 132 | 尾部锚点。注入正文落在最前会被工具描述稀释，这里在 system prompt **末尾**补一段身份与格式锁：主锚点做成 section（`dsh-purge:anchor`，`order 10150`，每轮重渲染），子会话锚点做成 context（`dsh-purge:child-anchor`，`order 235`，压在宿主 `subagent:delegation` 之后） |
| `capability-protocol.js` | 32 | 能力选用协议段：先读 skill / MCP 目录再按任务选用。不改官方 skill 注册、不改 MCP 挂载、不 deny 任何工具 |
| `phase1-keep.js` | 21 | 把宿主 phase-1 白名单放宽到 `dsh-purge*` + `dshpg-*`，防止注入段被宿主过滤掉 |
| `asset-table.js` | 170 | **生成物**。出厂默认提示词的 AES 密封槽（`SLOT` / `MASK` / `TAG` / `IV`），由 `scripts/embed-default-prompt.mjs` 从 `lib/default-prompt-inject.md` 生成 |
| `table-key.js` | 27 | 密封算法：`sealSlot()` 加密、`mixKey()` 派生密钥 |
| `table-read.js` | 18 | 解封：`openSlot()` 还原出厂默认正文 |
| `default-prompt-inject.md` | — | 出厂默认提示词**明文源**（`.gitignore` 排除，不进仓库、不发包）。改提示词改这里，然后跑 embed 脚本重新封槽 |
| `default-prompt-inject.zh.md` / `.en.md` | — | 中 / 英两套可读提示词模板 |

### 补丁引擎

| 文件 | 行数 | 作用 |
|---|---|---|
| `core.js` | 8809 | 引擎主体。`ALL_PATCHES` 共 86 条（`PATCHES` + `CODE_PATCHES` + `ENGINE_PATCHES` + `NEW_TOOL_PATCHES`），每条含 `id` / `name` / `layer` / `file` / `replacements` / `markers`，对 `@deepseek-ai` 包做精确字符串替换；`PATCH_DISPLAY_GROUPS` 把同一功能的升级链在 UI 上合成一行；`applyPatches()` / `revertPatches()` / `patchStatus()` 为执行与状态；`defaultOverrideText()` / `isKeepableUserOverride()` / `overrideTextChanged` 管理 `prompt-inject.md` 的读取、写入与「是不是用户自己改的」判定；另有宿主定位（`locate*`）、原子写、路径记忆 |
| `host-clean.js` | 26 | 「宿主提示词已清洗」判定。`HOST_CLEAN_MARKERS` 一份标记，`desktop.js` 生成 WSH 重启脚本时复用 |
| `hooks-deny.js` | 224 | hooks `deny` / `ask` 旁路（补丁 #30/#31/#64/#65）。四条开关永不同拆；负责写回官方形态、状态标注、设置持久化。不 import `core.js`，由 core 回调，避免循环依赖 |
| `subagent-control.js` | 67 | 子代理审批接管。宿主给子会话写死 `approval/policy = "never"`，这里接管该值 |
| `rules.js` | 560 | 规则集管理。多套 `AGENTS.md` / `CLAUDE.md`（`RULE_TARGETS`、`WORKSPACE_POLICIES`），启用即写入 `$DSH_HOME`，删除即从列表移除；含 id 校验与元数据 |
| `preset-default.js` | 205 | 停用 / 卸载时清掉指向本插件的 `selectedDefault`。宿主 `resolve()` 对未知预设是硬抛，不回退 `config.default`（#108） |
| `reapply.js` / `reapply-cli.js` | 388 / 5 | 启动自愈。比对「已应用戳」与磁盘实际状态：`stampMatches()`、`hostStillHasAppliedPatches()`、严格比对 `patchesMatchInstallStrict()`；npm 升级盖掉 `node_modules` 后自动重打。`reapply-cli.js` 是它的命令行包装 |
| `apply-progress.js` | 66 | 应用 / 还原 / 卸载的内存进度，只给 `/dsh-purge/status` 轮询，不落盘 |

### 宿主适配与重启

| 文件 | 行数 | 作用 |
|---|---|---|
| `surface.js` | 139 | 识别当前宿主面：`web` / `desktop` / `gui` / `tui`，未适配的回退 web |
| `host.js` | 28 | 宿主适配器工厂：`currentSurface()`、`hostAdapter()`、`scheduleRestart()`、`restartNote()` |
| `desktop.js` | 2007 | 桌面端专项：定位官方 EXE / `app.asar`、判断密封路径、生成 WSH 重启脚本、清理历史残留脚本 |
| `extract-asar.js` | 479 | 真实磁盘 fs。Electron 的 fs 会把 `resources/app` 映射进仍在的 `app.asar`，往里建文件会 ENOENT，所以解目录与打补丁走 `diskFs` / `hostFs`，含 `renameAsarAside()` 与探测缓存 |
| `hide-console.js` | 323 | 启动宿主时隐藏控制台窗口（Windows）。宿主 lib 下放 sidecar + `bin.js` 顶部加一行 import |
| `bin-preflight.js` | 85 | 启动前回收残留 web server，挂在宿主 `bin.js` 顶部（Cordis 激活之前）。0.2.x 里 webserver 绑端口失败是 fatal |
| `web-preflight-entry.js` | 167 | 上面那个 sidecar 的实体，由 `bin-preflight.js` 复制进宿主 lib |
| `web-reclaim.js` | 151 | detached 重启留下的孤儿 `dsh web` 进程回收：写记录、扫监听端口、杀旧进程 |
| `restart-web.js` | 144 | Web 端重启执行器：`DSH_PURGE_RESTART` 环境变量传参，保留端口与宿主参数重新拉起 |
| `restart-desktop.js` | 485 | 桌面端重启执行器：等旧进程退出、按原 argv 拉起、失败重试 |
| `uninstall-restart.js` | 80 | 卸载后的重启执行器，与上面两个同构 |
| `settle-policy.js` | 61 | 重启 / 重打时机判定。桌面启动时 `Fiber._reload` 还在挂 `desktop-office`，此时不得解包、不得杀进程 |
| `official-update.js` | 57 | 官方客户端升级后的补丁回写标记与 WSH 字面量转义 |
| `net-scope.js` | 839 | 本地靶场范围判定：私网 / 回环识别、域名先解析再判 IP、允许清单读写、`ownServerNote()` 文案 |
| `zip-extract.js` | 186 | 解压。DSH 压缩包条目遍历与解出 |

### 其他

| 文件 | 行数 | 作用 |
|---|---|---|
| `index.js` | 2913 | Cordis 插件主入口（`name` / `inject` / `apply`）。注册 prompt section、Web 路由（`/dsh-purge/*`）、聊天命令、设置项，串起 apply / revert / status / update / uninstall |
| `update.js` | 1137 | 版本更新：读本地版本、渠道（stable / beta）、GitHub Releases 比对、下载与替换、失败回滚 |
| `uninstall.js` | 307 | 卸载：先还原已打补丁 → 从 profile 的 `dependencies` / `bundles` / `cordis.patch.yml` 摘除 → 删插件树 → 排重启 |
| `profile-insert.js` | 382 | 往用户层 `profile/cordis.patch.yml` 补本体 insert、往 `package.json` 的 `dsh.profile.bundles` 补包名（#154 / #159 / #191）。pnpm / `dsh plugin add` 都不会自己写 insert |
| `skills.js` | 747 | 技能库：kebab-case 校验、扫描、备份、同步副本、扁平文件、对外展示字段 |
| `rewind.js` | 644 | 撤回：`planRewind()` 支持「撤最后一步 / 撤最后一整轮」，处理 inbox、子会话、draft 与待发消息 |
| `continue-retry.js` | 364 | 续跑与重试：`autoRetry`（request-error 重试）、`autoContinue`（idle 后发「继续」）、永久错误码表（AUTH / QUOTA / CONTEXT_OVERFLOW 等不重试）；设置落盘与合并 |
| `drill-grant.js` | 50 | 演练台授权记录落在 `$DSH_HOME/dsh-purge`，插件更新不丢；卸载时删除。Web 与桌面各记各的 |

---

## lib/redteam/ —— 演练台（资产 / 证据 / 计分）

| 文件 | 行数 | 作用 |
|---|---|---|
| `index.js` | 640 | 演练台 host 半侧自举：包路径解析、数据目录、安装脚本、工具注册 |
| `ui.js` | 1158 | 演练台 host 半侧 UI 接口：资产页、证据页、计分页的读写与校验（含 `githubRepoOf()` 兜底仓库名） |
| `client.js` | 4393 | 演练台浏览器半侧手写 bundle，最终被 `scripts/build-client.mjs` 抽进 `client.js` |
| `store.js` | 80 | 资产库的 Cordis 插件壳（`name` / `inject` / `apply`） |
| `store-core.js` | 5418 | 资产库核心，无 Cordis 依赖：靶标、阶段（`DEFAULT_STAGES`）、得分点、证据关联 |
| `tools.js` | 2265 | 智能体工具集（preset 平面）：资产增删查、证据登记、计分、并发闸门（同靶标最多 3 个执行智能体） |
| `schema.js` | 630 | 事实库建表与增量迁移，含 FTS 全文索引 DDL |
| `validate.js` | 88 | 写库前校验与规范化：webshell 类型 / 状态枚举、路径必须在靶标根内 |
| `atomic-write.js` | 28 | 先写临时文件、fsync、再 rename，避免半截 JSON |
| `ip-utils.js` | 140 | 纯函数：`ipToInt`、`isIpv`、`expandIpv`、`cidrOf`、`scopeOfIp` |
| `net` 相关判定在 `lib/net-scope.js` | — | 公网 / 私网 / 回环放行规则统一在那里 |
| `platform-config.js` | 612 | 平台适配配置与工具目录（`TOOL_CATALOG`） |
| `skill-availability.js` | 354 | 技能可用性自检，tools 与 UI 共用的唯一判定逻辑 |
| `skill-fixes.js` | 529 | 技能 / 工具缺失时的修复文案（`TOOL_FIXES`、`SKILL_FIXES`） |
| `score-rules.js` | 940 | 计分口径：服务识别、得分点、上限（`applyScoreCaps`）、隧道合法性 |
| `evidence-ledger.mjs` | 507 | 证据账本：置信度、权威性、需复核标记、DDL |
| `report-replay.js` | 213 | 报告复现工具（纯函数），把证据与步骤整理成可复现步骤 |
| `role-section.js` | 178 | 红队子会话角色稿：解析、捕获、下发 |
| `preset-composition.js` | 403 | 解析 `presets/redteam/{agent.cordis.yml,preset.yml}` 供 `registry.register()` |
| `settings.js` | 78 | 插件级设置，目前只有并发执行智能体上限 |

---

## scripts/

| 文件 | 行数 | 作用 |
|---|---|---|
| `embed-default-prompt.mjs` | 33 | 把 `lib/default-prompt-inject.md` 加密成 `lib/asset-table.js`。**改完提示词必跑**：`node scripts/embed-default-prompt.mjs` |
| `build-client.mjs` | 74 | 把 `lib/redteam/client.js` 的 ModuleLoader 工厂体抽出来，塞进 `client.js` 的 `__DSH_PURGE_DRILL_BEGIN__` / `END` 标记之间 |
| `patch-client-dock.mjs` | 454 | 把 `client.js` 改成右侧双页签（清洗 + 演练台）并嵌入 vendor 演练台 UI |
| `ensure-profile-bundle.mjs` | 89 | `postinstall`。往 profile 的 `package.json` 补 `dsh.profile.bundles`、往用户层 `cordis.patch.yml` 补本体 insert；在插件仓库里跑 `pnpm install` 时整段跳过 |
| `export-engagement-report.mjs` | 50 | 把某个靶标的得分报告导出成可搬运的交付包：`<outDir>/<靶标>-红队成果复现报告.md` + `shots/`，markdown 用相对路径，避免发出去碎图 |
| `redteam-setup.sh` | 487 | 演练台环境初始化脚本 |

---

## presets/redteam/

| 文件 | 作用 |
|---|---|
| `preset.yml` | 演练台预设声明 |
| `agent.cordis.yml` | 演练台智能体的 Cordis 装配（工具平面、角色） |
| `anchor-gate.mjs` | 红队锚点门：给子会话下发角色与边界 |

---

## docs/

| 文件 | 作用 |
|---|---|
| `FILES.md` | 本文件 |
| `banner.svg` | README 顶部横幅 |
| `v1.1.34-prompt-inject.md` | 历史版本的提示词存档 |
| `drill-auth-legal.html` | 演练台授权声明页（弹窗正文） |
| `redteam-upstream-LICENSE` / `redteam-upstream-DISCLOSURE` | 演练台部分所引用上游组件的许可与披露 |
| `preview/*.png` | 界面预览：清洗页、设置页、规则集、演练台、授权弹窗、本地靶场 |
| `notice/*.jpg|png` | 公告用配图 |

---

## 生成的、不用手改的

| 文件 | 由谁生成 |
|---|---|
| `client.js`（演练台段） | `scripts/build-client.mjs` |
| `lib/asset-table.js` | `scripts/embed-default-prompt.mjs` |

---

## 改了东西要跑什么

| 你改了什么 | 要跑什么 |
|---|---|
| 提示词（`lib/default-prompt-inject.md` 或 `lib/default-prompt-inject.zh.md` / `.en.md`） | `node scripts/embed-default-prompt.mjs`，然后重启宿主并点「应用」 |
| 演练台前端（`lib/redteam/client.js`） | `node scripts/build-client.mjs` |
| 补丁表（`lib/core.js` 的 `ALL_PATCHES`） | 无需构建，重启宿主后「清洗」页会按 `markers` 重新判定状态：`applied` / `pending` / `skipped` / `drifted` / `unmatched` |
| 只改 `lib/*.js` 里的逻辑 | 无需构建，重启宿主即可 |
