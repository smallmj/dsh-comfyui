# 真机验收清单（宿主 0.2.0-rc.2）

三层验收标准见 `CONTEXT.md`；离线部分已全绿（见 `docs/DSH-0.2.0-COMPAT.md` 待办 1–4）。这份清单只覆盖**必须在你机器上跑**的部分。

前置：ComfyUI 在跑（默认 `http://127.0.0.1:8188`），DSH 桌面应用已启动。

## 路线 A：本地装机（已执行，不需要发布）

插件包：`/Users/smallmj/My Projects/dsh-comfyui/smallmj-dsh-comfyui-0.5.5.tgz`（3.2MB，50 文件）。宿主插件管理器接受四种安装 spec：**registry 包名 / git 地址 / tarball / 本机绝对路径**（`dsh-plugin-manager`），所以不发布也能装机验收。

**已按 profile 的装配约定手工装好**（桌面 profile 由 Electron 应用独占管理，`dsh plugin --profile desktop add` 命令行被拒，UI 与手工写入等价）：

1. `~/.dsh/profiles/desktop/package.json` → `dependencies["@smallmj/dsh-comfyui"] = "file:<tarball 绝对路径>"`，并在 `dsh.profile.bundles` 追加同名条目（插件自带 `cordis.patch.yml` 负责把 `id: comfyui` 插进层栈，**profile 的 `cordis.patch.yml` 不需要动**——现有插件条目都不写在那里）
2. `cd ~/.dsh/profiles/desktop && pnpm install`（`nodeLinker: hoisted`，装完落在 `node_modules/@smallmj/dsh-comfyui`；`file:` spec 不受 `minimumReleaseAge` 门限约束，无需 `minimumReleaseAgeExclude`）
3. **重启应用**（host 侧代码要重启才生效；client 侧只需刷新页面）

回滚：装机前留了三份备份 `~/.dsh/profiles/desktop/{package.json,pnpm-lock.yaml,pnpm-workspace.yaml}.bak.20261010_123319.pre-smallmj-comfyui`，还原后 `pnpm install` 即可。

> 走 UI 的等价做法：侧边栏 → **Plugins** 页 → 安装 → spec 填上面那个 tarball 路径。两条路结果一致，UI 会自己写这三处。

**第一次重启后未挂载（已修复）**：装机 + 重启后 dshmarket 持续提示「已进入 profile bundle 层但本次未能热挂载；重启后生效」，`/comfyui/*` 全 404。根因是插件自带 `cordis.patch.yml` 里的 `name` 是 cordis 加载器要 import 的**模块说明符**，改名后残留的 `dsh-comfyui` 在 profile 里解析不到（peer 闸门本身是过的：宿主 `loadProfileDirectory` 复现 = 22 层 0 skipped）。修法与排查全过程见 `docs/DSH-0.2.0-COMPAT.md` §装机后未挂载 与 CLAUDE.md 契约 17。

重装新 tarball 时的 pnpm 陷阱：`file:` 依赖的 integrity 被 profile 的 `pnpm-lock.yaml` 钉住，`pnpm install --force` 与 `pnpm install --lockfile-only` + `pnpm install` **都装出旧内容**。有效做法：从 profile 的 `package.json` 删掉该依赖 → `pnpm install` → 原样加回 → `pnpm install`，然后核对 `node_modules/@smallmj/dsh-comfyui/cordis.patch.yml` 的内容。

已核对：`node_modules/@smallmj/dsh-comfyui/{lib/index.js,client/client.js,cordis.patch.yml}` 齐全，包内 `package.json` 是 `@smallmj/dsh-comfyui@0.5.5` 且 peer 含 `^0.2.0-rc.2`；宿主 asar 里的 `@deepseek-ai/dsh-settings` 是 `0.2.0-rc.2`（闸门比对的即此版本）；运行时依赖 `@deepseek-ai/schemastery@3.18.4`、`fflate@0.8.3` 已落位；历史数据目录 `~/.dsh/data/dsh-comfyui/` 原样保留。

### 第 1 层 · 安装兼容

- [ ] 安装过程不报 peer 依赖冲突，日志里没有 refusal code `incompatible-version`
- [ ] 应用启动后插件被加载（不是被闸门拒绝）

判据：`~/.dsh/profiles/desktop/node_modules/@smallmj/dsh-comfyui` 存在，且启动日志无 `incompatible-version`。

### 第 2 层 · 加载兼容

- [ ] 设置页出现 **ComfyUI** 分区，能打开、能保存（保存后无需重启即生效 = 0.1.7+ 设置机制）
- [ ] 侧边栏/会话头部出现面板入口，点开是**工作流 / 资产 / 队列**三页签
- [ ] 面板底部加载区在
- [ ] Agent 侧 4 个工具都在：`comfyui_run`、`comfyui_object_info`、`comfyui_workflow`、`comfyui_skill`
- [ ] 配套 skill `dsh-comfyui-workflows` 可被 `skill` 工具加载

判据（同源路由探活，DSH 端口按你的 profile，桌面默认 `http://127.0.0.1:3080`）：

```sh
curl -s http://127.0.0.1:3080/comfyui/ping
curl -s http://127.0.0.1:3080/comfyui/config | head -c 400
```

期望：`ping` 返回带 `ok` 的 JSON；`config` 返回脱敏配置（含 `skillsDir` / `skillsRoot`）。

### 第 3 层 · 功能兼容（验收线）

- [ ] `comfyui_object_info` 用 `filter: "LoadImage"` 能列出节点定义
- [ ] `comfyui_workflow action: list` 返回 `env`（`baseUrl` + `comfyuiDirs`）与 `loadArea`
- [ ] 跑通一次生成：让 Agent 用 `comfyui_run` 跑 `template: "txt2img"`，给一句 prompt
- [ ] 对话内出现媒体卡片（生成中 → 结果 → 点击放大）
- [ ] 面板资产页能看到这次产出，卡片里的图能显示（走 `/comfyui/media` 同源代理）
- [ ] 设置页「测试连接」通过
- [ ] 若你的 ComfyUI 有画布工作流：面板「提取」一次（验证多分量拆分），运行提取出的执行流
- [ ] 视频模板（可选，需 ComfyUI-WanVideoWrapper）：`template: "video"` 用 `mode: "async"` —— **这一项同时是 jobs owner 修复（契约 16）的验收点**：失败签名是 `session "[object Object]" has no live agent (background job owner must be live)`，出现即说明 `jobsOwner` 的特征探测没生效

真机冒烟脚本（打同源路由，需 DSH + ComfyUI 都在跑）：

```sh
node scripts/run-16x9.mjs        # 默认打 http://127.0.0.1:3080
node scripts/run-cg-portrait.mjs
```

## 路线 B：dshmarket 安装（发布后）—— **已放弃，仅留作参考**

维护者决定不自发布（npm 发布权在上游作者，fork 只能用 `@smallmj-` 前缀名，且需要 npm 登录与上架审核）。日常使用走路线 A 的本地装机。下面三条若将来要发布，缺一不可：

1. **npm 登录**：本机 `~/.npmrc` 的 registry 指向 npmmirror 镜像，镜像不接受发布；`package.json` 已用 `publishConfig.registry` 锁 `https://registry.npmjs.org`。需要先跑 `npm adduser --registry=https://registry.npmjs.org`（当前状态：`npm whoami` 报 `ENEEDAUTH`）。
2. **发布**：`npm publish`（`prepack` 会自动 build）。验证：`npm view @smallmj/dsh-comfyui version --registry=https://registry.npmjs.org`。
3. **上架 awesome-dsh-plugin**：dshmarket 以 awesome-dsh-plugin 列表为数据源自动收录（约一天同步）。条目模板在 `AWESOME.md`，**需按新包名 `@smallmj/dsh-comfyui` 改写**（现模板写的还是上游 `dsh-comfyui`）。

然后：dshmarket 搜 `@smallmj/dsh-comfyui` → 安装 → 重启 → 跑上面第 1/2/3 层清单。

**dshmarket 横幅判据**：兼容性提示的数据源是
`~/.dsh/profiles/desktop/.dsh-market/discovery-compatibility-v1.json` 的 `entries['@smallmj/dsh-comfyui']`。
装机宿主版本核对：

```sh
plutil -p "/Applications/DeepSeek Harness.app/Contents/Info.plist" | grep CFBundleShortVersionString
```

## 失败时给我什么

- 安装/启动日志里的 refusal 行（含 refusal code 全文）
- `curl -s http://127.0.0.1:3080/comfyui/ping` 的原始输出
- 设置页/面板哪一项没出现
- 生成失败时 Agent 的报错原文（不要贴完整工作流 JSON）

## 同步义务（CLAUDE.md 规定）

- `DEVELOPMENT.md` 的版本表与时间线 —— **该文件在本机不存在**（gitignored，clone 拿不到），需要时新建
- `AWESOME.md` 条目已按新包名 `@smallmj/dsh-comfyui` 改写；因放弃自发布，该文件目前是**未提交的上架材料**，不产生动作
- 两份 README 的「0.5.5 起」是仓库内的版本陈述；npm 首页不发布就不生效，安装说明以路线 A 的本地装机为准
