# DSH 0.2.0-rc.2 兼容审计

**状态：审计完成，代码未改。** 本文是证据与结论，改动按 §待办 执行。
审计日期：2026-10-10。宿主：`0.2.0-rc.2`（nightly 通道）。插件：`dsh-comfyui` 0.5.4。

## 结论一句话

**唯一的障碍是 `peerDependencies` 范围没写 0.2.x。** 插件用到的每一个接缝在 0.2.0-rc.2 上都存在且行为一致 —— 这不是 API 破坏，是 manifest 声明过期。

## 根因

`package.json:55-58`：

```json
"peerDependencies": {
  "@deepseek-ai/cordis": "^4.0.1",
  "@deepseek-ai/dsh-settings": "^0.1.2-alpha.4 || ^0.1.5-alpha.1 || ^0.1.6-alpha.1 || ^0.1.7-alpha.1"
}
```

装机 `@deepseek-ai/dsh-settings` = `0.2.0-rc.2`，四条范围全是 0.1.x → semver 失败 → peer 闸门拒绝，refusal code `incompatible-version`。`cordis ^4.0.1` 对装机 4.0.4 通过。

症状来源是 **dshmarket 的兼容性提示**（发现缓存 `~/.dsh/profiles/desktop/.dsh-market/discovery-compatibility-v1.json` 里 `entries/dsh-comfyui` 带着这组 peers），**不是运行时报错** —— 插件目前根本没装进 profile，所以没有运行时证据。

旁证：`~/.dsh/data/dsh-comfyui/` 里的记录停在 **2026-09-08**，即 0.1.x 时代运行过，数据完好。

## 逐接缝核查

对照装机产物（`app.asar` 解包在 `/tmp/dsh-asar`）与宿主源码 worktree `/Users/smallmj/deepseek-harness-0.2.0`（tag `dsh-v0.2.0-rc.2`）。

| 接缝 | 插件位置 | 0.2.0-rc.2 实况 |
| --- | --- | --- |
| `ctx.tools.register(definition)` | `src/tools.ts:1053`（`ctx as unknown as {tools}` 强转）；4 个工具 `comfyui_run`(:325)、`comfyui_object_info`(:459)、`comfyui_workflow`(:521)、`comfyui_skill`(:889) | `dsh-tools` lib 有 `register`；插件的 `ToolDefinition` 含 `output:{schema, render, presentationMeta?}`（`src/tools.ts:175-186`），与宿主强制的 `output` 一致 ✅ |
| `ctx.inject(['settings'])` → `installSection` | `src/index.ts:363-372` | 该方法在 0.2.0 产物与 0.1.7 源码里**都不存在** —— 这条分支早已死路，`typeof === 'function'` 守卫安全跳过 ✅ |
| `ctx.inject(['settings'])` → `configure({auto:false}, fiber)` | `src/index.ts:378-380` | `configure` 在 0.2.0 `dsh-settings` lib 存在 ✅ |
| `ctx.get('settings')` | `src/index.ts:191,195` | `dsh-settings` 编译产物 0.1.7-rc.2 与 0.2.0-rc.2 **逐字节相同**（`diff` 无输出）✅ |
| `ctx.get('credentials').resolve(envName)` | `src/index.ts:84` | credentials 接缝存在 ✅ |
| `ctx.get('skills').register(...)` | `src/index.ts:395-397` | `@deepseek-ai/dsh-skill` lib 有 `register` ✅ |
| `ctx.inject(['webServer'])` + `webServer.tapIndex(html)` | `src/index.ts:408-425` | `tapIndex` 在 `@deepseek-ai/dsh-host-webserver` lib 存在 ✅ |
| `ctx.on('loader/volatile-update')` | `src/index.ts:162` | loader 源码逐字节相同 ✅ |
| `ctx.slots.inject` 四个槽名 | `src/client/index.ts:33,38,43,48,53` | `tool.call.toolview`、`settings.section`、`shell.overlay`、`conversation.session.header.actions` 全部存在 ✅ |
| `import z from '@deepseek-ai/schemastery'` | `src/config.ts:13` | 装机 3.18.4，要求 `^3.18.4` ✅ |
| `export const inject = ['tools']` | `src/index.ts:39` | cordis 契约未变 ✅ |
| cordis / loader / include 核心 | — | `diff -rq` 对 asar 全部干净，版本一致（cordis 4.0.4、loader 1.0.5、include 1.0.9、group 1.0.4）✅ |

## 排除的三个怀疑（假警报，别再去改）

1. **`dsh.client.inject` 里的 `@deepseek-ai/dsh-client-runtime` 在 0.2.0 已不存在**（0.1.7 里它是 `packages/test-support/client-runtime`，测试用包，不随宿主发布）。**无害**：宿主类型 `DshClientManifest.inject` 的注释写明「Informational package-name dependencies, **not** Cordis service injection」，纯展示字段，不参与组合校验。
2. **`tsdown.config.mjs` 的 `PLATFORM_EXTERNALS` 里 `@deepseek-ai/dsh-client-web-react` 与 `@deepseek-ai/dsh-client-schema-form` 在 0.2.0 不存在。** **无害**：客户端源码的全部 import 只有 `react` 与相对路径（实测枚举 `src/client/*.ts|tsx` 的 import 说明符），这两个模块从未被 import，external 声明不产生请求。基线表 `PLATFORM_MODULES` 在 0.1.7 与 0.2.0 **逐行相同**（`packages/client/web/src/platform.ts`）。
3. **`installSection` 被摘了。** **无害**：0.1.7 起就没有，插件早已走 volatile + `configure` 路径（见 `CLAUDE.md` §index.ts，#11）。

## 决策（本轮 grilling 定案）

| # | 决策 |
| --- | --- |
| 适配定义 | 三层（安装/加载/功能兼容），**验收定在功能兼容** —— 见 `CONTEXT.md` |
| peer 范围 | **追加** `\|\| ^0.2.0-rc.2`，不替换 —— 保住 0.1.x 宿主，也是给上游 PR 的正确形态 |
| 类型基线 | 宿主源码 worktree `/Users/smallmj/deepseek-harness-0.2.0`（tag `dsh-v0.2.0-rc.2`）已建，类型对齐真实运行版本 |
| 发布路径 | **fork 自发布 + 同时给上游 `fandc520/dsh-comfyui` 提 PR**（npm 发布权在上游作者手里，自发布需改名） |
| 安装方式 | 走 **dshmarket** 安装 |
| 验证 | 复用现有 4 个离线自测（`scripts/test-convert.mjs`、`test-skillpack.mjs`、`test-store-params.mjs`、`test-transfer.mjs`，均需先 `npm run build`）**+ 新增一个加载冒烟测试** |
| 范围 | **只修兼容，不采纳 0.2.0 新能力** —— 审计没有发现接缝破坏，也没有新特性需要采纳 |
| 豁免 | **不采用** `dsh plugin allow-version` —— 每次宿主升级都要重豁免，是绕过不是适配 |

## 待办（代码改动，尚未执行）

1. `package.json:57` peer 追加 `\|\| ^0.2.0-rc.2`；`devDependencies["@deepseek-ai/dsh-settings"]`（`package.json:61`，现为 `^0.1.2-alpha.4`）同步到 0.2.x，让 typecheck 跑在目标宿主类型上。
2. `version` 0.5.4 → 0.5.5；`repository.url`（`package.json:7`）指向 fork。
3. `npm install && npm run typecheck && npm run build`，跑 4 个离线自测。
4. 新增加载冒烟测试：装进 profile → 启动 → 断言 `apply` 注册成功 → 断言一次真实 `comfyui_run` 返回。现有 4 个脚本只测纯逻辑，测不到「闸门放行但接缝静默失效」。
5. 发自己的包 + dshmarket 安装验证；给上游提 PR（就是第 1 条那一行）。
6. 发版同步义务（`CLAUDE.md` §相关文档）：改功能 → 两份 README + `src/skill.ts`；发版 → `DEVELOPMENT.md` 版本表与时间线（本地文件，不入版本库）。

## 复现命令

```sh
# 装机宿主版本
plutil -p "/Applications/DeepSeek Harness.app/Contents/Info.plist" | grep CFBundleShortVersionString

# 装机 peer 版本（app.asar 需先解包）
npx @electron/asar extract "/Applications/DeepSeek Harness.app/Contents/Resources/app.asar" /tmp/dsh-asar
python3 -c "import json;print(json.load(open('/tmp/dsh-asar/dsh/node_modules/@deepseek-ai/dsh-settings/package.json'))['version'])"

# 闸门逻辑
sed -n '61,88p' /Users/smallmj/deepseek-harness-0.2.0/packages/boot/app-boot/src/plugin-compatibility.ts

# dshmarket 兼容性提示的数据源
python3 -c "import json;print(json.dumps(json.load(open('/Users/smallmj/.dsh/profiles/desktop/.dsh-market/discovery-compatibility-v1.json'))['entries']['dsh-comfyui'],ensure_ascii=False,indent=2))"
```
