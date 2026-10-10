# CONTEXT — 词汇表

本仓库的领域词汇表。只放**定义**，不放实现细节与决策记录（决策在 `docs/adr/`，实现契约在 `CLAUDE.md`）。
命名领域概念时（issue 标题、重构提案、测试名）使用此处定义的说法，不要漂移到同义词。

## 兼容性（Compatibility）

「适配」不是一个词，是**三层**，验收只认第三层。讨论时不要再用笼统的「适配」。

| 术语 | 定义 |
| --- | --- |
| **安装兼容**（install-compatible） | 宿主的 peer 闸门放行，插件装得进 profile。**闸门放行不等于可用** —— 单独达成此层是假适配。 |
| **加载兼容**（load-compatible） | 插件在 cordis 组合树里跑通：入口 `apply(ctx)` 执行完毕，注入的服务到位，不抛异常。 |
| **功能兼容**（function-compatible） | 插件对外提供的能力在宿主上真的可用：4 个 Agent 工具、`/comfyui/*` 路由、浏览器面板与卡片、设置页。**这是本项目的验收标准。** |

## 宿主与闸门

| 术语 | 定义 |
| --- | --- |
| **宿主**（host） | 运行插件的 DeepSeek Harness 运行时，以其版本号标识（如 `0.2.0-rc.2`）。插件不拥有宿主，只声明能跑在哪些宿主版本上。 |
| **peer 闸门**（peer gate） | 唯一被强制执行的版本检查：宿主对插件 `peerDependencies` 里 `@deepseek-ai/dsh*` 与 `@deepseek-ai/cordis` 做 semver 匹配（`includePrerelease: true`），不匹配则拒绝安装，refusal code `incompatible-version`。实现在宿主侧 `packages/boot/app-boot/src/plugin-compatibility.ts`。 |
| **声明式 engines** | `package.json` 的 `engines.dsh` 是**纯声明，宿主不执行**。写它不产生任何兼容效果，不要拿它当适配手段。 |
| **豁免**（exemption） | 闸门逃生口：`dsh plugin allow-version <pkg@ver> --dsh-version <host> --accept-risk`，按 `包名@版本` 精确放行，存进 profile 的 `compatibility.json`。**不是适配**：每次宿主升级都要重新豁免，本项目不采用。 |
| **预发布逐条列** | semver 的预发布版只匹配同一 `x.y.z`，`^0.1.7-alpha.1` 不覆盖 `0.1.7-rc.2`。所以每条宿主线都要在 peer 范围里**单列一段**，用 `\|\|` 串起来。 |

## 接缝（Seam）

| 术语 | 定义 |
| --- | --- |
| **接缝**（seam） | 插件与宿主之间的接触点：注入的服务（`ctx.tools` / `ctx.get('settings')` / `ctx.get('skills')` / `webServer` / `credentials`）、槽位名（`ctx.slots.inject` 的字符串）、入口契约（`apply` / `Config` / `inject`）、manifest 声明（`dsh.bundle.patch` / `dsh.client`）。适配工作 = 逐个核查接缝，不是「读 changelog」。 |
| **结构化切片**（structural slice） | 插件对宿主服务只做鸭子类型声明（本地 `interface` + `ctx.get(...) as X \| undefined`），不引包依赖。好处：宿主换版本时运行时零风险，peer 声明只是 manifest 上的事。`src/index.ts` 的 `SettingsService` / `CredentialsService` 就是切片。 |
| **基线模块表**（baseline） | 浏览器侧插件的 external 只对着一张**宿主冻结的模块表**解析（宿主侧 `packages/client/web/src/platform.ts` 的 `PLATFORM_MODULES`：react 全家、cordis、store、ui-slots、ui-primitives、ui-dockkit）。表里没有的模块被 import 就是「缺供应商」，组合期直接拒绝。`tsdown.config.mjs` 的 `PLATFORM_EXTERNALS` 是**声明**，只有真被 import 才产生请求。 |
| **两代设置宿主** | 设置页有两套契约：`installSection`（0.1.2–0.1.5，**当前宿主已无此方法**）与 volatile Config 字段 + `settings.configure({ auto: false }, fiber)` + `loader/volatile-update`（0.1.7+，现行路径）。两代都用 `typeof === 'function'` 守卫，缺哪代就静默走另一代。 |

## 插件自身的核心概念

见 `CLAUDE.md` §架构 —— **图工作流**（衍生主题，ComfyUI 画布，不能直接运行）与 **API 工作流**（运行主题，node id → `{class_type, inputs}`）这对主题是贯穿全项目的语言，不在本表重复定义。
