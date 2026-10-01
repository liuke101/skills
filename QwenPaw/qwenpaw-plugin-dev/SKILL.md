---
name: qwenpaw-plugin-dev
description: 创建 QwenPaw 插件或 PawApp 应用。覆盖：给 Agent 加工具（tool）、接入消息渠道（channel）、记忆后端（memory）、中间件、Provider、生命周期 Hook、斜杠命令、前端 UI 扩展，以及带独立前端页面与后端路由的 PawApp（type: app，App Center 应用）。也覆盖控制台前端界面定制：侧边栏菜单/页面、品牌 logo、隐藏按钮或菜单、聊天头像与欢迎页、设置菜单等。当用户提到 QwenPaw 插件/扩展/PawApp/应用中心，想给 QwenPaw agent 加功能，或想改 QwenPaw 的界面（侧边栏、顶栏、菜单、头像、主题元素）时使用；开工前先让用户在 plugin 与 app 之间选择，明确两者边界。
---

# QwenPaw 插件 / PawApp 开发

为 QwenPaw（github.com/agentscope-ai/QwenPaw）编写扩展。**所有扩展都是「插件」**：一个含 `plugin.json` 的目录，走同一条 PluginLoader 流水线（发现 manifest → 解析 → 版本兼容 → 安装 requirements.txt → 加载入口模块 → `register(api)`）。PawApp 只是这条流水线上的一种类型（`type: "app"`），多了一层 SDK 和 App Center 展示。

> 本 Skill 依据 2026-09 的 QwenPaw main 分支（v2.2.x）整理，前端 UI 扩展部分另经 v2.2.1 上 6 个实战插件验证（详见 references/frontend-ui.md）。开工前自己先跑 `qwenpaw --version`（或 `pip show qwenpaw`；跑不了再问用户；QwenPaw 未安装则先按官方文档安装），旧版本可能缺少较新 API（如 PawApp SDK、`register_slash_command`）。manifest 的 `qwenpaw_version.min` 按用户实际版本填写。细节拿不准时查官方文档 https://qwenpaw.agentscope.io/docs/plugins ，或克隆源码看 `src/qwenpaw/plugins/api.py` 与 `src/qwenpaw/pawapp/`。

## 第一步（固定动作）：决策门 —— plugin 还是 app

**跳过本问的唯一条件**：用户原话已经明确点名了层级（说了「插件/plugin」或「应用/PawApp/App」这类词）。仅仅「看起来明显偏向某一种」不构成跳过理由——仍要问，只是把有证据的推荐项放第一个。在错误的抽象层上开工返工成本很高。

用 AskUserQuestion 让用户选择（没有该工具就直接在回复里提问并给出对照表，等答复再动手）。其他澄清问题（如数据放哪、给谁用）可以合并到同一次提问里。

问题形式（有证据时把推荐项放第一个并标 Recommended）：

- **Plugin（扩展 Agent 能力）** — 给 Agent 补充能力：工具、消息渠道、记忆后端、中间件、生命周期钩子、模型 Provider、UI 扩展点。用户在设置页配置，没有独立应用页面。
- **PawApp（App Center 应用）** — 一个完整应用：独立前端页面（App Center 里 `/apps/<app_id>`）+ 独立后端路由（`/api/<app_id>/...`）+ ctx 对象（主动调用 Agent、存储数据、推送 UI）。

给用户解释边界时的速查表：

| 维度 | Plugin | PawApp |
|---|---|---|
| 定位 | 向内供给：让 Agent 变强，用户间接受益 | 向外交付：给用户一个应用 |
| 独立 UI 页面 | ✗（frontend 插件可挂 UI 扩展点/侧边栏小组件，但没有独立应用入口和自己的后端路由空间） | ✓ App Center 独立页面 |
| 驱动方向 | 只能被 Agent 调用（Tool），**不能反向驱动 Agent** | ✓ `ctx.chat()` / `ctx.chat_stream()` 主动驱动 |
| 实时推送（后端→前端） | ✗ | ✓ SSE / `ctx.ui.push()` |
| 内置存储 | ✗ | ✓ `ctx.storage()` |
| 后台服务/任务 | 仅生命周期 Hook | ✓ asyncio 任务、ManagedService sidecar、专属 Agent 档案 |
| 典型例子 | 查图工具、Telegram 渠道、ADBPG 记忆 | 看板应用 agent-kanban、QwenPaw Creator |

> 表中「实时推送」的 `ctx.ui.push()` 需要 SDK TaskManager 注入的 SSE 通道（`get_task_manager().create_task`）才生效；手写 SSE（kanban 模式）时直接用 `SSEChannel`，此时 `ctx.ui.push()` 不可用。

判断口诀：

- 「给 Agent 多一个函数 / 接一个聊天平台 / 换记忆后端 / 拦截 Agent 行为 / 加斜杠命令 / 自定义模型端点」→ **Plugin**
- 「要一个有界面、能主动驱动 Agent、要实时刷新、要存数据的应用」→ **PawApp**
- 「只是给工具加个 API Key 配置界面」→ tool 插件的 `meta.tools[].config_fields` 就够，不需要 app
- 还想不清楚：需要「页面」或「反向调用 Agent」→ PawApp；否则 Plugin

PawApp 是 Plugin 的超集（`@app.tool` 也能注册工具），但小工具不要上 app——plugin 更轻、装卸更简单。

## 共同基础

### 目录与安装

- 开发目录随意；安装位置固定 `~/.qwenpaw/plugins/<id>/`（环境变量 `QWENPAW_WORKING_DIR` 可改根目录）。
- `id` 即安装目录名，推荐小写字母加连字符（如 `my-plugin`），不得含路径分隔符；入口模块会被以 `plugin_<id 中 - 换 _>` 的名字加载。
- 有第三方 Python 依赖时加 `requirements.txt`——**实际安装读的是它，不是 manifest 的 `dependencies`**（后者仅展示）；loader 发现缺失会自动 pip/uv 安装。
- 插件内部模块导入：入口被当作顶层模块加载，**裸导入**（如 `from weather import get_weather`）会被宿主重定向到插件自己的目录，flat 布局直接用裸导入；azure_bot 示例用相对导入 + `__init__.py` 也可行。
- 没有脚手架命令（`qwenpaw plugin create` 之类不存在）——手工建目录，或从本 Skill 的骨架复制。
- 禁用 = 目录改名加 `.disabled` 后缀；`qwenpaw plugin uninstall <id>` 卸载。

### CLI（验证与安装闭环）

```bash
qwenpaw plugin validate <dir>     # 交付前必跑：manifest 必填项 + 后端入口可导入 + 前端文件存在
qwenpaw plugin install <dir|zip|URL> [--force]   # 服务运行中也行（热安装，无需重启）
qwenpaw plugin list               # 装完确认出现
qwenpaw plugin info <id>          # 看清单详情
qwenpaw plugin uninstall <id>
qwenpaw app                       # 启动服务，默认 http://127.0.0.1:8088
```

### 最小 plugin.json

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "0.1.0",
  "description": "一句话说清它做什么",
  "type": "tool",
  "entry": { "backend": "plugin.py" },
  "qwenpaw_version": { "min": "2.0.1" }
}
```

- `entry.backend` / `entry.frontend` 至少声明一个，且文件必须真实存在。
- 有第三方 Python 依赖时，在 manifest 旁加 `requirements.txt`（实际安装读它）。
- 字段全表、type 推断规则、meta 各用途见 [references/manifest.md](references/manifest.md)。

## 路线 A：Plugin（扩展 Agent 能力）

入口模块必须导出一个名为 **`plugin`** 的对象（惯例是类实例），类实现 `register(self, api)`（同步或 async 均可）。骨架：

```python
# plugin.py
from qwenpaw.plugins.api import PluginApi

class MyPlugin:
    def register(self, api: PluginApi) -> None:
        api.register_tool(
            tool_name="my_tool",          # 全局唯一，跨插件重名直接失败
            tool_func=my_tool_func,
            description="给 LLM 看的工具说明",
            tool_type="network",          # file | network | shell | internal
        )

plugin = MyPlugin()   # ← PluginLoader 找的就是这个变量（名字必须是 plugin）
```

按用户需求选类型，**先读 [references/plugin-types.md](references/plugin-types.md) 对应小节再写代码**（每节有完整签名和官方示例的代码模式）：

| 需求 | type | 关键调用 |
|---|---|---|
| Agent 可调用的新函数 | `tool` | `api.register_tool(...)` + 模块级 `get_tool_config()` |
| 接入新聊天平台 | `channel` | `BaseChannel` 子类 + `api.register_channel(...)` |
| 新长期记忆后端 | `memory` | `BaseMemoryManager` 子类 + `api.register_memory_backend(...)`；必须显式写 `"type": "memory"`（推断不出来），且在 workspace 启动前加载 |
| 拦截/增强 Agent 执行 | `general` | `agentscope.middleware.MiddlewareBase` + `api.register_middleware(factory, priority=)` |
| 自定义模型端点 | `provider` | `api.register_provider(provider_id, provider_class, ...)` |
| 启动/关闭时执行代码 | 任意 | `api.register_startup_hook / register_shutdown_hook / register_uninstall_hook` |
| `/斜杠命令` | 任意 | `api.register_slash_command(name, handler)`，handler: `async (ctx, args) -> Msg \| None` |
| 补充 HTTP API | 任意 | `api.register_http_router(router, prefix="/xxx")` → 挂到 `/api/xxx`（prefix 全局唯一） |
| 纯前端 UI 扩展 | `frontend` | 只有 `entry.frontend`；用 `window.QwenPaw.chat/menu/route/slot` 扩展点，**先读 [references/frontend-ui.md](references/frontend-ui.md)**（见下方路线 C） |

工具函数写法要点（docstring 就是给 LLM 的说明）：类型注解 + 完整 docstring；配置用 `from qwenpaw.plugins import get_tool_config` 读取；重阻塞调用包 `asyncio.to_thread`；失败返回错误文本而不是抛异常。完整模板见 references。

## 路线 B：PawApp（App Center 应用）

**写代码前先读 [references/pawapp-sdk.md](references/pawapp-sdk.md)**（PawApp 类全部注册方法、ctx 全部方法、前端宿主 API、SSE/后台任务/ManagedService，均摘自源码与三个官方示例）。

推荐目录结构（agent-kanban 模式，无构建步骤）：

```
my-app/
├── plugin.json
├── backend/main.py      # PawApp 后端入口
└── ui/index.js          # 前端（IIFE，无 JSX、无打包器）
```

manifest 要点：`"type": "app"` + `meta.pawapp` + `meta.permissions`：

```json
{
  "id": "my-app", "name": "My App", "version": "0.1.0",
  "type": "app",
  "entry": { "backend": "backend/main.py", "frontend": "ui/index.js" },
  "qwenpaw_version": { "min": "2.0.1" },
  "meta": {
    "pawapp": { "icon": "🧩", "category": "productivity",
                "entry_page": "/apps/my-app", "launch_scope": "page" },
    "permissions": { "chat": true, "storage": true }
  }
}
```

后端骨架（`backend/main.py`）：

```python
from fastapi import APIRouter, Depends
from qwenpaw.pawapp import PawApp, get_ctx

router = APIRouter()

@router.get("/items")
async def list_items(ctx=Depends(get_ctx)):
    ...  # ctx.chat() / ctx.storage / ctx.ui / ctx.tools 都可用

app = PawApp(name="My App", app_id="my-app")
app.include_router(router)          # 路由最终挂在 /api/my-app/...

@app.on_launch
async def _start(): ...             # 起 asyncio 后台任务、初始化数据

@app.on_terminate
async def _stop(): ...              # 取消任务、收尾

plugin = app   # 兜底：PluginLoader 先找 plugin 再找 app，显式双写保证任何分支都加载成功
```

前端骨架（`ui/index.js`）——不需要 React 依赖、不需要构建工具，宿主全部提供：

```js
(function () {
  var QwenPaw = window.QwenPaw;
  if (!QwenPaw || !QwenPaw.host || !QwenPaw.registerRoutes) return;
  var host = QwenPaw.host;
  var h = host.React.createElement;

  function apiFetch(path, opts) {
    var url = host.getApiUrl(path);
    var token = host.getApiToken ? host.getApiToken() : "";
    opts = opts || {};
    opts.headers = Object.assign({ "Content-Type": "application/json" },
      token ? { Authorization: "Bearer " + token } : {}, opts.headers || {});
    return fetch(url, opts);
  }

  function MyApp() {
    var React = host.React;
    var state = React.useState(null);
    var items = state[0], setItems = state[1];
    React.useEffect(function () {
      apiFetch("/my-app/items").then(function (r) { return r.json(); })
        .then(function (d) { setItems(d.items || []); });
    }, []);
    return h("div", null, (items || []).map(function (it) {
      return h("div", { key: it.id }, it.title);
    }));
  }

  // path 以 /apps/ 开头 = 只在 App Center 展示，不进侧边栏
  QwenPaw.registerRoutes("my-app", [
    { path: "/apps/my-app", component: MyApp, label: "My App", icon: "🧩" },
  ]);
})();
```

前端规则：

- React / ReactDOM / antd 从宿主拿（`host.React` / `host.antd`），**不要把 react/react-dom 打进 bundle**。
- 简单应用：IIFE + `React.createElement` 零构建（kanban 模式）；复杂应用：Vite library 模式构建单文件 `dist/index.js`（`external: ["react", "react-dom"]`），或 iframe 方案（creator 模式）。
- SSE 推送：`EventSource` 不能带 Authorization header，token 放 query 参数（kanban 的 `apiStreamUrl` 模式）。
- 需要更强的 app 作用域 API（`api.request`、`ui.registerPage(mount)`、`dependencies`）时用 `window.QwenPaw.paw.forApp(appId)` 新版 SDK，见 references。

装好后验证：控制台左侧 Apps → 卡片出现 → 点击进入 `/apps/my-app`。

## 路线 C：前端界面定制（frontend 插件实战套路）

改 QwenPaw 界面（侧边栏、顶栏、菜单、头像、欢迎页……）一律用 `type: "frontend"` 插件（无后端，只有 `entry.frontend`，IIFE 零构建）。**动手前必读 [references/frontend-ui.md](references/frontend-ui.md)**——那里有 menu/route/slot/chat 各扩展点的精确语义、选择器规则和 6 个实战套路。先记四条铁律：

1. 宿主 antd `prefixCls="qwenpaw"`：真实 DOM 没有 `ant-*` 类，CSS 选择器必须前缀无关；`.anticon-*` 来自 @ant-design/icons 不受影响，可作判别。
2. CSS module 类名保留局部名（`[name]__[local]__[hash]`）：用 `[class*="局部名"]` 定位宿主元素，配结构指纹防误伤。
3. `window.QwenPaw.modules` 运行时为空——复用宿主页面组件用 `route.wrap` 截获（wrapper 幂等、原样返回 Inner，并留兜底跳转）。
4. 插件脚本只在控制台启动时执行一次，SPA 会不断重挂载宿主节点——DOM 注入必须配 MutationObserver 幂等续命。

需求 → 套路速查（详见 frontend-ui.md §2）：

| 需求 | 关键调用 |
|---|---|
| 侧边栏加菜单项 + 页面 | `route.add` + `menu.add`（**菜单项 id = 路由 id**，否则高亮错位） |
| 页面 = 复用宿主现有页面 | `route.wrap` 截获宿主组件 + 自己路由渲染 + 兜底跳转 |
| 改左上角品牌词标 | `slot.replace("header.logo", render)`（返回 null 回退默认） |
| 隐藏硬编码按钮/菜单项 | 注入 `<style>`（前缀无关选择器）/ 结构指纹 + 隐藏 |
| 聊天头像、欢迎页、问候语 | `chat.welcome.set(pluginId, { avatar: SVG dataURL, ... })`（回复卡片同步生效） |
| 齿轮设置菜单精简 | overlay 局部名锚点 + 指纹找面板 + 隐藏多余子元素 |

## 交付前检查清单

- [ ] `qwenpaw plugin validate <dir>` 通过
- [ ] `qwenpaw plugin install <dir>` 成功，`qwenpaw plugin list` 能看到
- [ ] plugin：服务日志无加载报错；工具型**先在设置里启用工具**（`register_tool` 默认 `enabled=False`），再让 agent 实际调用一次；渠道型从平台发一条消息；UI 扩展刷新控制台确认渲染
- [ ] app：App Center 出现卡片；`/apps/<id>` 页面能打开；`curl -H "Authorization: Bearer <token>" http://127.0.0.1:8088/api/<app_id>/...` 后端路由通
- [ ] `qwenpaw plugin uninstall <id>` 再重装一遍（验证 register 幂等、无残留）

## 常见坑（每条都有源码依据）

1. 忘记导出 `plugin` 变量（app 忘写 `plugin = app`）→ `AttributeError: Plugin module must export a 'plugin' object`。
2. `entry.backend` / `entry.frontend` 都不声明或文件不存在 → `FileNotFoundError`。
3. 工具名跨插件重名 → fail-closed 冲突报错；同插件内重复注册是幂等的。
4. `register_http_router` 的 prefix 必须 `/` 开头且不能只是 `/`，全局唯一（PawApp 自动用 `/{app_id}`，不用管）。
5. memory 插件必须显式写 `"type": "memory"`（推断不出来），否则错过 Phase 1（workspace 启动前）导致注册无效。
6. `register_tool` 默认 `enabled=False`——装完要在设置里启用才对 agent 生效；`@app.tool` 装饰器默认 `enabled=True`。
7. `config_fields` 的 `type` 取值：工具只允许 text/password/number/select；渠道才多一个 switch。
8. 版本约束语义是 `>=min, <max` 左闭右开；`max` 省略时自动推导为下一 minor（min=2.0.1 → 允许 2.0.x）。当前实现只强制下界。
9. manifest 的未知顶层字段会被忽略（`pack_exclude`、`icon` 等可以安全写），但别指望它们参与加载逻辑。
10. 单个插件加载失败会整体回滚且不影响其他插件——报错去服务日志里找。
11. PawApp 没有 WebSocket API，实时推送用 SSE（SDK 的 `SSEChannel` 或自建 `StreamingResponse`）。
12. 前端把 react/react-dom 打进 bundle 会与宿主冲突——必须 external。
13. `qwenpaw_version.min` 别照抄示例，按用户装的实际版本填。
14. 宿主 antd `prefixCls="qwenpaw"`：真实 DOM 没有 `ant-layout-header`/`ant-btn` 这些类，CSS 选择器写 `ant-*` 全部落空（前端插件实测踩坑）；用 `[class*="..."]` + 原生标签，`.anticon-*` 不受 prefixCls 影响。
15. `window.QwenPaw.modules` 运行时为空（注册函数无人调用）——别指望从里面复用宿主页面组件，用 `route.wrap` 截获。
16. 侧边栏菜单项高亮要求**菜单项 id 与路由 id 相同**（选中态按路由 id 匹配菜单项 id）；不一致时点击后高亮落在别的项。
17. 一次性 DOM 注入会被 SPA 重挂载吞掉（侧边栏折叠、Popover `destroyOnHidden` 每次打开都换新节点）——必须 MutationObserver + 幂等重建；对宿主元素设固定高度记得 `box-sizing:border-box`（content-box 下按测量值回写会持续漂移）。

## 深入参考（按需读）

- [references/frontend-ui.md](references/frontend-ui.md) — 控制台前端 UI 扩展实战参考：menu/route/slot/chat 精确语义与高亮契约、选择器规则（prefixCls 坑）、侧边栏/设置菜单内部结构、DOM 注入生存策略、无宿主环境下的 mock 验证方法（**写任何 frontend 插件前通读**）
- [references/manifest.md](references/manifest.md) — plugin.json 全部字段、type 推断、版本约束、meta 各用途（写/改 manifest 时读）
- [references/plugin-types.md](references/plugin-types.md) — 各类型注册 API 完整签名 + 四个官方插件示例的代码模式（写 Plugin 时读对应小节）
- [references/pawapp-sdk.md](references/pawapp-sdk.md) — PawApp SDK 全量参考 + 三个官方 app 示例架构 + 前端宿主 API（写 PawApp 前通读）
