# PawApp SDK 参考

来源：QwenPaw 源码 `src/qwenpaw/pawapp/`（app.py / context.py / deps.py / task.py / service.py / dependency.py / agent.py）+ 三个官方示例 `plugins/apps/agent-kanban`、`qwenpaw-creator`、`qwenpaw-data`。签名均为源码原文。

**PawApp 是什么**：一个同时拥有前后端入口、manifest 带有 `meta.pawapp` 的插件（`type: "app"`）。走和普通插件完全相同的 PluginLoader 流水线，区别是：

- 后端入口导出 **`app`** 变量（`PawApp` 实例，实现同样的 `register(api)`）；习惯上文件末尾再写 `plugin = app` 兜底（loader 先找 `plugin` 再找 `app`）。
- `register()` 时把所有缓冲的路由以 **`/{app_id}`** 为前缀挂到主 FastAPI（最终 URL `/api/{app_id}/...`），并逐项应用 tool/command/middleware/hook 等缓冲注册。
- 只在 **App Center** 展示（不进侧边栏），前端路由约定 `/apps/{app_id}`。
- 与主服务同进程同端口，无独立端口。

`app_id` 规则：开启标准能力路由时强制 `^[a-z0-9][a-z0-9-]*$`；干脆始终用小写连字符，并与 manifest `id` 保持一致。

---

## 1. PawApp 类：装饰器式注册

```python
from qwenpaw.pawapp import PawApp

app = PawApp(name="My App", app_id="my-app")
```

全部注册方法（`app.py` 原文签名）：

```python
app.route(path: str, *, methods: list[str] | None = None)
    # 默认 methods=["POST"]；函数首个参数名为 ctx 时自动注入 Depends(get_ctx)
app.tool(name: str, *, description: str = "", icon: str = "🔧",
         enabled: bool = True, tool_type: str = "network", target_param: str = "")
app.command(name: str, *, description: str = "")      # 斜杠命令
app.middleware(factory: Callable, *, priority: int = 100)
    # factory(ctx, agent_config) -> MiddlewareBase | None
app.hook(phase: str, *, priority: int = 100)          # phase: "startup" | "shutdown"
app.on_install(func)      # → startup hook, priority=90
app.on_launch(func)       # → startup hook, priority=100（app 启动后的初始化点）
app.on_terminate(func)    # → shutdown hook, priority=100（清理 asyncio 任务）
app.on_uninstall(func)    # → uninstall hook
app.include_router(router: APIRouter, **kwargs)       # 挂自有 FastAPI 路由（最常用）
app.skill_provider(skills_dir, *, enabled_by_default=True, channels=None)
app.prompt_section(name, content, *, after="workspace", priority=100,
                   condition=None, agent_id=None)
app.on_workspace_created(func=None, *, priority=100)
app.runtime_hook(hook)                                # HookBase 实例
app.agent_profile(agent_id: str, *, name: str, description: str = "",
                  persona_dir=None, language=None, plan_enabled=True, pinned=True)
app.managed_service(name: str, *, command: Sequence[str], health_path="/health",
                    host="127.0.0.1", startup_timeout=30.0, shutdown_timeout=10.0,
                    cwd=None, env=None, external_url_env=None, mode_env=None,
                    on_before_start=None, display_name=None, capabilities=(),
                    required=True, expose_dependency=True, runtime_remediation=None)
app.dependency(dependency_id: str, *, display_name=None, ownership="external",
               capabilities=(), required=True, probe: DependencyProbe,
               lifecycle: DependencyLifecycle | None = None, replace=False)
app.remove_dependency(dependency_id) -> bool
app.enable_standard_capabilities() -> PawApp          # 开启 /chat /storage 等标准路由
app.enable_dependency_agent_tools() -> PawApp         # 注册 {app_id}_dependency_status/_action 工具
```

注意：

- **没有 `@app.get` / `@app.post`**——HTTP 装饰器只有 `@app.route`；实际项目更常用 `app.include_router(router)` + 路由函数参数 `ctx=Depends(get_ctx)`。
- `@app.tool` 默认 `enabled=True`（普通插件 `register_tool` 默认 False）。
- 生命周期优先级：install(90) → 启动 hook(35 agent_profile / 70 managed_service / 100 on_launch)；shutdown 反向。
- `agent_profile` 会在卸载时 `detach()`（只摘 profile、保留 workspace 数据）。

## 2. ctx（PawAppContext）——调用 Agent / 存储 / 推送

```python
from qwenpaw.pawapp import get_ctx          # 常规注入（FastAPI Depends）
from qwenpaw.pawapp import get_scoped_ctx   # 身份绑定认证主体：伪造 user_id/channel → 403

@router.get("/items")
async def list_items(ctx=Depends(get_ctx)): ...
```

`ctx` 字段：`app_id`、`agent_id`（默认 "default"，可用 `?agent_id=` 覆盖）、`channel`（默认 console，取 X-Channel header）、`user_id`。

| 成员 | 签名 | 说明 |
|---|---|---|
| `chat` | `async chat(message: str, *, skill=None, session_id=None, channel=None, user_id=None) -> ChatReply` | **反向驱动 Agent**；session 缺省 `pawapp:{app_id}`；`ChatReply.text` 是最终文本 |
| `chat_stream` | 同上 → `AsyncIterator[Any]` | 流式版本。**迭代产出的是宿主流式事件对象**（有 `.type` / `.delta` 等属性，可 `model_dump()`；结构随版本可能变化）。消费时用防御式写法（kanban 原文模式）：`getattr(ev, "type", "?")`、`getattr(ev, "delta", False)` 跳过增量、`hasattr(ev, "model_dump")` 再序列化 |
| `get_session_history` | `async get_session_history(session_id=None) -> list[dict]` | 读会话历史（过滤 reasoning） |
| `ensure_chat_session` | `async ensure_chat_session(session_id=None, *, name=None) -> dict \| None` | 注册/收养 app 拥有的对话 |
| `list_chat_sessions` / `create_chat_session(*, name="New analysis")` / `rename_chat_session(chat_id, *, name)` / `archive_chat_session(chat_id)` / `pin_chat_session(chat_id, *, pinned)` / `delete_chat_session(chat_id) -> bool` | 多对话管理（只能操作本 app 命名空间的会话） | |
| `storage` | `AppStorage`：`async get(key, *, default=None) / set(key, value) / delete(key) / keys() / clear_namespace()` | 内置 JSON 存储，命名空间 `pawapp:{app_id}` |
| `tools` | `ToolProxy.invoke(name: str, params: dict \| None = None)` | 调用已注册的工具 |
| `ui` | `async push(event_type: str, data=None)`；`async confirm(message: str, *, data=None, timeout: int = 300) -> {"action": "approve"\|"timeout", "data"}` | SSE 推 UI 事件 / 推送并阻塞等用户确认。**依赖 TaskManager 注入的 `_sse_channel`**（`get_task_manager().create_task` 时注入，见 §4）；手写 SSE 的 app 里不可用 |
| `notify` | `async notify(*, channels=None, title="", body="", user_id="", session_id="")` | 经渠道管理器发通知（best-effort） |
| `toast` | `async toast(message: str, *, kind: str = "info")` | 前端 toast |
| `settings` | `AppSettings.get(key, *, default=None)` | 读 App Center 设置（manifest `meta.settings`） |
| `is_app_session_id` | `(session_id) -> bool` | 会话是否属于 `pawapp:{app_id}` 命名空间 |
| `user` / `config` | 占位实现（TODO） | 别依赖 |

后台协程里需要 ctx 时：把 HTTP 请求带来的轻量 ctx 缓存到模块级变量（workspace registry 是长命单例），或 `dataclasses.replace(ctx, agent_id=...)` 换身份（kanban 模式）。

## 3. 标准能力路由（`enable_standard_capabilities()` 后可用）

开启后自动挂在 `/api/{app_id}` 下（全部用 `get_scoped_ctx`，身份绑定）：

| 路由 | 说明 |
|---|---|
| `POST /chat`、`POST /chat/stream`（SSE）、`GET /chat/history` | 对话三件套 |
| `GET/POST /chat/sessions`、`PATCH /chat/sessions/{id}`、`POST .../archive`、`POST .../pin`、`DELETE /chat/sessions/{id}` | 会话管理 |
| `GET /storage`、`GET/PUT/DELETE /storage/{key}` | 键值存储 |
| `POST /toast`、`POST /notify` | 轻提示/通知 |

会话 ID 安全规则：显式 session_id 必须属于 `pawapp:{app_id}` 命名空间，否则 404。小应用直接开标准能力 + 自写少量业务路由即可。

## 4. 长任务与 SSE

SDK（`qwenpaw.pawapp.task`）：

```python
from qwenpaw.pawapp.task import SSEChannel, get_task_manager

channel = SSEChannel()              # asyncio.Queue(1000)；30s 无事件发 keepalive
await channel.send_event(data)      # 产出 data: {json}\n\n
task_id = get_task_manager().create_task(app_id, handler, ctx, params)
# handler(ctx, **params) 在后台执行；结束发 {"type":"done","data":...} / {"type":"error","message":...}
# create_task 会把 channel 注入 ctx._sse_channel → ctx.ui.push()/confirm() 生效
```

kanban 的手写等价模式（更透明，推荐抄）：

```python
@router.get("/issues/{issue_id:path}/stream")
async def stream_issue(issue_id: str) -> StreamingResponse:
    async def _gen():
        ch = SSEChannel()
        _CHANNELS[issue_id] = ch
        try:
            async for event in ch:
                yield event
        finally:
            _CHANNELS.pop(issue_id, None)
    return StreamingResponse(_gen(), media_type="text/event-stream",
                             headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"})
```

**订阅竞态与交接（最常踩的坑）**：触发任务的端点（如 `POST .../recommend`）和流端点（`GET .../stream`）是两个请求，channel 的交接方式决定事件会不会丢：

- 触发端点：创建后台任务，把状态（`status`/`text`）放在模块级结构里，事件广播给该 id 注册的所有 channel。
- 流端点：**先注册 channel、再在同一个事件循环切片内（中间不要 await）做状态快照**先发给前端——先注册保证不丢后续事件，快照保证刷新/重连能续上进度。
- 前端断线重连由 `EventSource` 自动完成，后端靠「注册 + 快照」模式保证续传。

**鉴权**：宿主鉴权中间件支持从 `?token=` query 读取凭证（`src/qwenpaw/app/auth.py`），所以 SSE 的 query-token 是官方支持的鉴权路径；kanban 的 stream 路由本身不再挂 `Depends(get_ctx)`，鉴权由宿主全局处理。

Agent 执行用 `asyncio.create_task` 后台跑，逐步 `await channel.send_event({"type": "tool_start"|"tool_done"|"done"|"error", ...})`。**没有 WebSocket API，一律 SSE。**

## 5. ManagedService（sidecar 子进程）与 Dependency 控制面

```python
service = app.managed_service(
    "context",
    command=(python_exe, "-m", "uvicorn", "my.api:app", "--host", "{host}", "--port", "{port}"),
    health_path="/api/health", cwd=..., env={...},
    external_url_env="MY_SERVICE_URL", mode_env="MY_SERVICE_MODE",  # 支持附加外部服务
    startup_timeout=45, display_name="Context API", required=True,
)
```

- 只允许 loopback host；端口 bind 0 自动分配，`{host}`/`{port}` 占位符自动替换。
- 双模式：managed（spawn 子进程 + 轮询健康检查）或 external（设了 `external_url_env` 就直接附加外部 URL）。
- 自动登记为 `host_managed` dependency，挂到 host startup/shutdown hook。
- 状态/诊断：`service.status()`（浏览器安全）/ `service.diagnostics()`（含 url/pid/recent_logs）。

Dependency 健康控制面：

```python
from qwenpaw.pawapp import DependencyHealth, DependencyProbe, DependencyLifecycle

app.dependency("graph-store", display_name="Graph Store", ownership="external",
               required=False, capabilities=("context-graph",),
               probe=DependencyProbe(callback=_probe_fn, timeout_seconds=5, cache_seconds=8))
```

- `ownership` ∈ `host_managed`（managed_service 自动登记）/ `app_managed` / `external`（external 不能带 lifecycle）。
- 自动生成路由：`GET /dependencies`、`GET /dependencies/{id}`、`POST /dependencies/{id}/actions/{action}`（check/start/stop/restart/provision）。
- `enable_dependency_agent_tools()` 后 agent 可用 `{app_id}_dependency_status` / `{app_id}_dependency_action` 工具查询和操作依赖。

## 6. 前端

### 静态资源与加载链路

- 前端 entry 及所有静态文件经 `GET /api/frontend_plugin/{app_id}/files/{相对路径}` 提供（免鉴权、路径穿越防护）。
- Console 加载：`GET /api/frontend_plugin` 列表 → fetch 入口 JS → Blob URL 动态 `import()` → 脚本执行时调用 `registerRoutes` 注册组件 → App Center 点击卡片时按 `meta.pawapp.entry_page` 从路由注册表找组件 inline 渲染（URL 深链 `/apps/<id>` 可刷新恢复）。
- `loadPawApp` 校验 `plugin_type === "app"` 且执行后注册表里出现 `/apps/` 前缀路由。

### 宿主 API（`window.QwenPaw.host`）

| 成员 | 说明 |
|---|---|
| `React`, `ReactDOM`, `antd`, `antdIcons`, `apiBaseUrl` | 共享依赖，**不要自己打包** |
| `getApiUrl(path)` | 补全为 `/api/...` 完整 URL |
| `getApiToken()` | Bearer token |
| `fetch(path, init)` | hostFetch，自动注入 Authorization / X-Agent-Id |
| `useTheme()` / `useLocale()` / `useSelectedAgent()` / `useCurrentSession()` | Hooks（只能在插件组件内用） |
| `getSelectedAgentId()` / `getCurrentSessionId()` | 命令式取值 |
| `prepareBrowserSession(appId)` / `usesBrowserSession(appId)` | 浏览器会话 |

其他命名空间：`QwenPaw.chat.*`（welcome/sender/actions/toolRender/card 等 UI 扩展）、`QwenPaw.menu/route/slot`、遗留 shim **`QwenPaw.registerRoutes(pluginId, routes)`**（`/apps/` 前缀 = 只注册路由不进侧边栏）、新版 **`QwenPaw.paw.forApp(appId)`**。

### 两种写法

**A. 零构建 IIFE（agent-kanban，1614 行单文件 `ui/index.js`）**——首选起步方式：

```js
(function () {
  var QwenPaw = window.QwenPaw;
  if (!QwenPaw || !QwenPaw.host || !QwenPaw.registerRoutes) return;
  var host = QwenPaw.host;
  var h = host.React.createElement;

  function apiFetch(path, opts) {          // kanban 原文模式
    var url = host.getApiUrl(path);
    var token = host.getApiToken ? host.getApiToken() : "";
    opts = opts || {};
    opts.headers = Object.assign({"Content-Type": "application/json"},
      token ? {Authorization: "Bearer " + token} : {}, opts.headers || {});
    return fetch(url, opts);
  }
  function apiStreamUrl(path) {            // EventSource 带不了 header → token 放 query
    var url = host.getApiUrl(path);
    var token = host.getApiToken ? host.getApiToken() : "";
    if (token) url += (url.indexOf("?") >= 0 ? "&" : "?") + "token=" + encodeURIComponent(token);
    return url;
  }
  var es = new EventSource(apiStreamUrl("/my-app/items/42/stream"));

  QwenPaw.registerRoutes("my-app", [
    { path: "/apps/my-app", component: MyApp, label: "My App", icon: "🧩" },
  ]);
})();
```

**B. Vite library 模式（qwenpaw-data）**——复杂 UI 时：`build.lib` 入口 `src/index.tsx`、formats `["es"]`、产物固定 `index.js`、`external: ["react", "react-dom"]`；用新版 SDK 注册页面：

```tsx
const paw = (window as any).QwenPaw?.paw?.forApp("my-app");
paw.ui.registerPage({
  path: "/apps/my-app",                    // 强制 /apps/{appId} 前缀校验
  label: "My App", icon: "🧩", priority: 20,
  mount(container) {                       // mount 模式允许非 React 技术栈
    const root = createRoot(container);
    root.render(<App />);
    return () => root.unmount();           // 返回清理函数
  },
});
```

`paw.forApp(appId)` 提供四组能力：`api.request/get/post/.../stream/events/task`（自动拼 `/api/{app_id}` + auth）、`host.{chat, chatStream, chatSessions, storage, toast, notify, getSelectedAgentId, getCurrentSessionId}`、`ui.registerPage(...)`、`dependencies.{list,get,check,action,subscribe}`。

**C. iframe 方案（qwenpaw-creator）**——大型独立 SPA：宿主侧 entry 注册一个 iframe 包装组件，`src` 指向 `/api/frontend_plugin/<id>/files/ui/dist/app/index.html`，iframe 内是完整 Vite React 应用，用 postMessage 同步路由。仅在 UI 规模很大、需要独立路由/依赖时考虑。

### 数据持久化（kanban 模式）

`ctx.storage` 和自管数据文件是**两种并存的选择**：storage 适合小体量键值（每个 key 整值读写，覆盖式写入；在 asyncio 单线程内读-改-写整表是安全的，但跨进程没有保护）；数据体量大、结构复杂或要跨进程共享时用自管文件——kanban 模式：

- 数据文件放插件目录下 `data/*.json`（如 `<plugin_dir>/data/issues.json`）。
- 写入：`asyncio.Lock` + 跨进程 `fcntl` 文件锁 + 原子写（tempfile + `os.replace`）。
- 周期落盘：`@app.on_launch` 里 `asyncio.create_task(_persist_loop())`（每 10s），`@app.on_terminate` 里 cancel + 最终落盘。

## 7. App Center 与生命周期

- 列表：`GET /api/pawapps`（过滤 manifest 含 `meta.pawapp` 的插件），卡片字段来自 `meta.pawapp`（icon/icon_url/category/entry_page/launch_scope）。
- 卸载：`DELETE /api/pawapps/{app_id}`（= 卸载插件，执行 shutdown/uninstall hooks、清理路由和工具）。
- 加载时机：Phase 2（agent 启动后）加载 app 型插件，路由立即挂上；热安装同理。
- app 专属 agent（`agent_profile`）的 workspace 固定在 `WORKING_DIR/workspaces/{agent_id}`，模板 ID `pawapp:{app_id}`。

## 8. 三个官方示例——抄哪个

| 示例 | 架构 | 什么时候抄它 |
|---|---|---|
| **agent-kanban** | 零构建 IIFE 前端（1614 行单文件）+ 单文件后端（1341 行）；自建 SSE、后台派发/落盘循环、文件持久化、审批集成 | 绝大多数 app 的起点：单页面 + 中等交互量 + 要驱动 agent |
| **qwenpaw-creator** | Vite + React + TS 大型 SPA，iframe 嵌入；多 agent 工具（`meta.tools[]` 十余个带 config_fields）；requirements 复杂 | UI 规模大、需要独立路由/国际化/复杂依赖 |
| **qwenpaw-data** | 全家桶：`enable_standard_capabilities` + `managed_service`（两个 uvicorn sidecar）+ `dependency` 控制面 + `agent_profile` + `skill_provider` + `prompt_section` + middleware + command + `@app.tool` | 需要独立后台服务/外部依赖健康管理/专属 agent/技能包注入 |

三者共同结尾：`plugin = app`。三者共同原则：**运行时只做「装饰器 → 注册」和「ctx → 委托」，重活复用现有子系统；app 自有逻辑（数据层、后台服务、外部 API）不受 SDK 约束。**
