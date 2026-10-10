# 真机验收清单（宿主 0.2.0-rc.2）

三层验收标准见 `CONTEXT.md`；离线部分已全绿（见 `docs/DSH-0.2.0-COMPAT.md` 待办 1–4）。这份清单只覆盖**必须在你机器上跑**的部分。

前置：ComfyUI 在跑（默认 `http://127.0.0.1:8188`），DSH 桌面应用已启动。

## 路线 A：本地 tarball 装机（不需要发布，推荐先跑这条）

插件包已打好：`/Users/smallmj/My Projects/dsh-comfyui/smallmj-dsh-comfyui-0.5.5.tgz`（3.2MB，50 文件）。
宿主侧插件管理器接受四种安装 spec：**registry 包名 / git 地址 / tarball / 本机绝对路径**（`dsh-plugin-manager`），所以不发布也能装机验收。

1. 侧边栏 → **Plugins** 页 → 安装（Install）→ 在 spec 输入框填 tarball 的绝对路径：
   `/Users/smallmj/My Projects/dsh-comfyui/smallmj-dsh-comfyui-0.5.5.tgz`
2. 安装完成后**重启应用**（host 侧代码要重启才生效；client 侧只需刷新页面）。

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

## 路线 B：dshmarket 安装（发布后）

前置三件事，缺一不可：

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

## 发布后的同步义务（CLAUDE.md 规定）

- `DEVELOPMENT.md` 的版本表与时间线 —— **该文件在本机不存在**（gitignored，clone 拿不到），发版时需新建
- `AWESOME.md` 条目改写成新包名
- 两份 README 的「0.5.5 起」措辞在 0.5.5 真正发布后才成立
