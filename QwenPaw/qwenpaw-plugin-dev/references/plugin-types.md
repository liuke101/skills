# Plugin 各类型开发参考

来源：QwenPaw 源码 `src/qwenpaw/plugins/api.py`（`PluginApi`）+ 官方示例 `plugins/tool/qwen-image`、`plugins/channel/azure_bot`、`plugins/memory/adbpg`、`plugins/middleware-demo/tracing-middleware`。签名均为源码原文。

通用约定（所有类型都适用）：

- 入口模块导出 `plugin = XxxPlugin()`；类实现 `register(self, api: PluginApi)`（sync/async 均可）。
- `api.config` = 用户配置里 `config.plugins[<plugin_id>]` 的内容。
- `api.plugin_id`、`api.manifest` 可用；`api.runtime` 提供运行时辅助（`get_provider` / `list_providers` / `log_info` / `log_error` / `log_debug`）。
- 工具/斜杠命令/运行时注册会延迟到 startup hook 执行；插件加载失败整体回滚，不影响其他插件。
- 内部模块导入：入口以顶层模块名加载（非包），**裸导入**（`from weather import x`）会被宿主重定向到插件自己的目录（`module_isolation.py`），flat 布局推荐裸导入；azure_bot 用相对导入 + `__init__.py` 也可行。
- 数据持久化：插件没有内置存储，需要落盘时自己选位置并处理并发（官方示例有把数据放插件目录 `data/` 子目录的先例，也有写 workspace `.qwenpaw/` 的）；多进程并发写要加文件锁 + 原子写（tempfile + `os.replace`）。

---

## 1. tool —— 给 Agent 加工具

```python
api.register_tool(
    tool_name: str,              # 全局唯一；跨插件重名 fail-closed，同插件重复注册幂等
    tool_func: Callable,         # sync 或 async
    description: str = "",
    icon: str = "🔧",
    enabled: bool = False,       # 默认关闭，用户在设置里启用后生效
    tool_type: str = "network",  # 治理类型: file | network | shell | internal
    target_param: str = "",      # 治理目标参数名（可选）
) -> None
```

注册流程（宿主自动处理）：认领工具名所有权 → 写治理白名单 → 挂到 `qwenpaw.agents.tools` 模块并 bridge 到各 workspace 的运行时 ToolRegistry → 写入 agent 配置（`enabled=False`）。

配套：

```python
from qwenpaw.plugins import get_tool_config       # 模块级：读当前工具配置（无 agent 维度）
cfg = get_tool_config("generate_image_qwen")       # dict 或 None
# 实例方法（带 agent 维度）：
api.get_tool_config(tool_name: str, agent_id: str) -> dict
api.set_tool_config(tool_name: str, agent_id: str, config: dict) -> None
```

工具函数体内拿不到 agent_id，用模块级 `get_tool_config(name)`；需要按 agent 区分配置（且能拿到 agent_id）时才用实例方法。

### 工具函数模板（qwen-image 模式）

```python
import asyncio
from agentscope.message import DataBlock, TextBlock, URLSource, ToolResultState
from agentscope.tool import ToolChunk
from qwenpaw.plugins import get_tool_config

async def generate_image_qwen(
    prompt: str,
    size: str = "2048*2048",
    n: int = 1,
) -> ToolChunk:
    """Generate images from text prompts using Qwen-Image.

    Args:
        prompt: What the image should depict.
        size: Output resolution, e.g. "1024*1024".
        n: Number of images to generate.
    """
    tool_config = get_tool_config("generate_image_qwen") or {}
    api_key = tool_config.get("api_key")
    if not api_key:
        return ToolChunk(state=ToolResultState.ERROR,
                         content=[TextBlock(type="text",
                                            text="Error: api_key not configured")])
    # 重阻塞调用放线程池
    path = await asyncio.to_thread(_blocking_sdk_call, prompt, size, n, api_key)
    return ToolChunk(state=ToolResultState.SUCCESS, content=[
        DataBlock(source=URLSource(url="file://" + path, media_type="image/png")),
        TextBlock(type="text", text=f"Saved to {path}"),
    ])
```

要点：**docstring 就是给 LLM 的工具说明**（写清 Args）；配置缺失返回 `ToolChunk(state=ERROR, ...)` 而不是抛异常；重 IO 用 `asyncio.to_thread`；返回值用 agentscope 的 `ToolChunk`/`TextBlock`/`DataBlock`/`URLSource`。

manifest 里的 `meta.tools[].config_fields` 声明配置表单（type 只允许 text/password/number/select），`requires_config: true` 标记该工具需配置后才能用（UI 展示用），见 [manifest.md](manifest.md)。

## 2. channel —— 接入消息平台

```python
api.register_channel(
    channel_class: Type,         # BaseChannel 具体子类，必须有 'channel' 类属性作 key
    label: str = "",
    description: str = "",
    config_fields: list[dict] | None = None,   # type ∈ text|password|number|switch|select
    icon: str = "",              # http(s) URL
    doc_url: str | {"zh": ..., "en": ...} = "",
) -> None
```

宿主校验：渠道 key 小写规范化、不得覆盖内置渠道、不得与其他插件重复、必须是 `BaseChannel` 子类。注意渠道插件的 `config_fields` 在 `register_channel` 参数里声明（不像工具写在 meta 里）。

### BaseChannel 子类模式（azure_bot）

```python
# 导入（源码路径）：
from qwenpaw.app.channels.base import (
    BaseChannel, ProcessHandler, TextContent, ImageContent,
    FileContent, AudioContent, ContentType, OnReplySent,
)

class AzureBotChannel(BaseChannel):
    channel = "azure_bot"            # 渠道唯一 key（必填类属性）
    uses_manager_queue = True

    def __init__(self, process: ProcessHandler, enabled: bool,
                 app_id: str, app_password: str, ...,
                 http_host: str = "0.0.0.0", http_port: int = 3978,
                 workspace_dir=None, on_reply_sent=None, display_config=None, ...):
        super().__init__(...)        # 把 process/enabled/workspace_dir 等传给基类

    @classmethod
    def from_config(cls, process, config, on_reply_sent=None, display_config=None, ...):
        # host 用它构造实例；config 是 SimpleNamespace（不是 dict！），
        # 用 getattr(config, "field", default) 逐字段取
        ...

    async def start(self): ...       # 起 HTTP webhook / 轮询等接收循环
    async def stop(self): ...
    async def send(self, ...): ...           # 发文本消息到平台
    async def send_media(self, ...): ...
    async def health_check(self) -> dict: ...
    def build_agent_request_from_native(self, ...): ...  # 平台消息 → AgentRequest
    def resolve_session_id(self, ...): ...
```

`plugin.py` 注册原文模式（config_fields 带多语言 label、switch、default）：

```python
class AzureBotChannelPlugin:
    def register(self, api: PluginApi):
        from .channel import AzureBotChannel
        api.register_channel(
            channel_class=AzureBotChannel,
            label="Azure Bot",
            description="Azure Bot Service (Bot Framework) integration",
            icon="https://img.alicdn.com/....png",
            doc_url={"zh": "https://...", "en": "https://..."},
            config_fields=[
                {"name": "app_id", "label": "App ID", "type": "text", "required": True},
                {"name": "app_password", "label": "App Password", "type": "password", "required": True},
                {"name": "http_port", "label": "HTTP Port", "type": "number", "default": 3978},
                {"name": "share_session_in_group", "label": {...}, "type": "switch",
                 "required": False, "default": False},
            ],
        )

plugin = AzureBotChannelPlugin()
```

channel 类型插件在 **Phase 1**（agent 启动前）加载。完整实现参考 `plugins/channel/azure_bot/channel.py`。

## 3. memory —— 长期记忆后端

```python
api.register_memory_backend(
    *, backend_id: str,                      # 如 "adbpg"
    factory: Type,                           # BaseMemoryManager 子类
    label: str = "",
    config_schema: Type | None = None,       # Pydantic 配置模型
    metadata: dict | None = None,            # 可带 tools 治理映射、secret_fields 等
) -> None
```

- **必须显式写 manifest `"type": "memory"`**（推断不出来），且在 Phase 1（workspace 启动前）注册。
- manifest 里同时声明 `meta.memory_backends: [{"id": ..., "label": ...}]` 供 UI 展示。

### BaseMemoryManager 模式（adbpg）

```python
from qwenpaw.memory import BaseMemoryManager   # 经 qwenpaw.memory 包 re-export

class ADBPGMemoryManager(BaseMemoryManager):
    def __init__(self, context: MemoryBackendContext):
        # context (frozen dataclass): agent_id, working_dir, host_working_dir,
        #                             backend_config, language, token_estimate_divisor
        self._cfg = MyMemoryConfig.model_validate(context.backend_config)  # dict → Pydantic

    # 必须实现（abstractmethod）：
    async def start(self): ...
    async def memory_search(self, ...): ...
    async def auto_memory(self, ...): ...
    async def _close_backend(self) -> bool: ...

    # 可选覆写：
    def get_memory_prompt(self) -> str: ...
    def list_memory_tools(self) -> list[Callable[..., ToolChunk]]: ...
    def get_auto_memory_interval(self) -> int: ...
    def get_auto_memory_search_options(self) -> AutoMemorySearchOptions | None: ...
```

`metadata` 里把内部工具注册进治理白名单：

```python
metadata={
    "description": "AnalyticDB for PostgreSQL memory",
    "network_access": True,
    "secret_fields": ["rest_api_key"],
    "tools": {"memory_search": {"policy_name": "ADBPGMemorySearch",
                                "tool_type": "network", "target_param": "query"}},
}
```

配置模型用 Pydantic（`rest_base_url: str = ""`、`search_timeout: float = Field(default=10.0, ge=1.0)` 等带校验字段）。完整实现参考 `plugins/memory/adbpg/backend/`。

## 4. middleware —— 拦截 Agent 执行

```python
api.register_middleware(
    middleware_factory: Callable,   # factory(ctx, agent_config) -> MiddlewareBase | None
    *, priority: int = 100,         # 小 = 洋葱模型更外层；返回 None = 本请求跳过
) -> None
```

完整示例（tracing-middleware，源码原文模式）：

```python
import os, time
from pathlib import Path
from agentscope.middleware import MiddlewareBase
from qwenpaw.plugins.api import PluginApi

class TracingMiddleware(MiddlewareBase):
    """Logs tool call name, input, and execution duration."""
    def __init__(self, trace_file: Path) -> None:
        self._trace_file = trace_file
        self._trace_file.parent.mkdir(parents=True, exist_ok=True)

    async def on_acting(self, agent, input_kwargs: dict, next_handler):
        tool_call = input_kwargs["tool_call"]
        start = time.perf_counter()
        try:
            async for item in next_handler():
                yield item
        finally:
            ...  # 把耗时写进 workspace/.qwenpaw/trace.log

def _tracing_factory(ctx, agent_config) -> TracingMiddleware | None:
    del agent_config
    if not os.environ.get("QWENPAW_TRACE"):
        return None
    workspace_dir = getattr(ctx, "workspace_dir", None)
    if workspace_dir is None:
        return None
    return TracingMiddleware(trace_file=Path(workspace_dir) / ".qwenpaw" / "trace.log")

class TracingPlugin:
    def register(self, api: PluginApi) -> None:
        api.register_middleware(_tracing_factory, priority=50)

plugin = TracingPlugin()
```

`on_acting`（以及 `on_reasoning` 等）是洋葱模型：`async for item in next_handler(): yield item` 前后插入自己的逻辑。

## 5. provider —— 自定义模型端点

```python
api.register_provider(
    provider_id: str,            # 唯一标识
    provider_class: Type,        # Provider 类（继承宿主 BaseProvider 契约）
    label: str = "",
    base_url: str = "",
    **metadata,                  # chat_model="OpenAIChatModel", require_api_key=True 等
) -> None
```

`provider_class` 的具体基类导入路径随版本变化，写之前在已安装环境里 `python -c "import qwenpaw; ..."` 或 grep 源码确认（`grep -rn "register_provider" src/qwenpaw/`），并参考已有 provider 实现。

## 6. 生命周期 Hook

```python
api.register_startup_hook(hook_name: str, callback, priority: int = 100)
api.register_shutdown_hook(hook_name: str, callback, priority: int = 100)
api.register_uninstall_hook(hook_name: str, callback, priority: int = 100)
    # uninstall 回调签名: (plugin_id: str, delete_files: bool)（按 kwargs 调用）
api.register_workspace_created_hook(hook_name: str, callback, priority: int = 100,
                                    reload_safe: bool = False)
    # 回调签名: (workspace_info: dict) -> None，至少含 agent_id / workspace_dir
```

- priority 越小越早执行（0 最先、100 默认、200 最后）；callback sync/async 均可。
- 卸载顺序：shutdown hooks → uninstall hooks → 清理模块/命令/工具 → unregister。

## 7. 运行时注册（workspace 级，延迟到 startup hook）

```python
api.register_slash_command(
    name: str, handler, *,
    aliases: tuple = (), category: str = "plugin",
    help_text: str = "", metadata: dict | None = None,
) -> None
# handler: async (ctx, args) -> Msg | None

api.register_runtime_hook(hook)          # HookBase 实例（qwenpaw.runtime.hooks），
                                         # 需有 phase / name / run()；phase 取
                                         # qwenpaw.runtime.phases.Phase 枚举 8 阶段:
                                         # pre_dispatch, post_dispatch, pre_agent_build,
                                         # post_agent_build, pre_execute, post_response,
                                         # on_error, finally

api.register_agent_stop_handler(handler, *, priority: int = 100, name: str = "")
    # handler: async (ctx) -> StopHandlerResult（qwenpaw.loop.gates），可返回 BLOCK 让 agent 继续

api.register_mode(mode_cls)              # AgentMode 子类，需唯一 name 类属性；
                                         # 每个 workspace 新建实例（有状态 mode 不可共享单例）

api.register_prompt_section(
    name: str, after: str, provider, *,
    priority: int = 100, condition=None, agent_id=None,
) -> None
# after: 宿主锚点（workspace / multimodal / env_context）
# provider: (agent) -> str；condition: (ctx) -> bool

api.register_skill_provider(skills_dir: Path, *,
    enabled_by_default: bool = True, channels: list[str] | None = None) -> None
# skills_dir 下每个含 SKILL.md 的子目录会被复制进 workspace；卸载时自动清理

api.register_control_command(handler, priority_level: int = 10)
    # handler: BaseControlCommandHandler，需有 command_name 属性
```

## 8. HTTP 路由（普通插件挂 API）

```python
api.register_http_router(router, *, prefix: str, tags: list[str] | None = None) -> None
```

- `router` 是 FastAPI `APIRouter`；最终挂到 **`/api` + prefix**（prefix=`"/pets"` → `/api/pets/...`）。
- prefix 必须 `/` 开头且不能只是 `/`；**全局唯一**（跨插件重复抛 `ValueError`）。
- OpenAPI tag 默认 `plugin:<plugin_id>`。
- PawApp 不需要手动调这个——SDK 自动以 `/{app_id}` 为 prefix 挂载。

## 9. frontend —— 纯前端 UI 扩展

manifest 只有 `entry.frontend`（无 backend），按 "frontend-only plugin" 加载。Console 启动时 `GET /api/frontend_plugin` 拿列表 → fetch 入口 JS → Blob URL 动态 `import()` 执行。静态资源经 `GET /api/frontend_plugin/{id}/files/{path}` 提供（免鉴权、有路径穿越防护）。

扩展点（`window.QwenPaw.*`，所有注册方法第一参数是 pluginId，返回 `{dispose()}` 可撤销）：

```ts
window.QwenPaw.host            // React, ReactDOM, antd, antdIcons, apiBaseUrl,
                               // getApiUrl(path), getApiToken(), fetch(path, init),
                               // useTheme(), useLocale(), useSelectedAgent(),
                               // useCurrentSession(), getSelectedAgentId(),
                               // getCurrentSessionId(), prepareBrowserSession(appId),
                               // usesBrowserSession(appId)
window.QwenPaw.chat            // welcome / theme / leftHeader / rightHeader / sender /
                               // actions / request / response / toolRender / approval /
                               // card —— 各有 set / render / add 三个动词
window.QwenPaw.menu / route / slot
window.QwenPaw.memoryBackends.register(pluginId, extension)
window.QwenPaw.audit.overrides()
```

- Hooks（`useTheme` 等）只能在插件组件内部调用。
- 构建：Vite library 模式，`jsxRuntime: "classic"`，`rollupOptions.external: ["react", "react-dom"]`（宿主共享），产物单文件（如 `dist/index.js`）。
- 类型提示：把 `console/src/plugins/types/qwenpaw.d.ts` 拷进项目。
- UI 扩展示例（memory 配置卡片，adbpg）：

```ts
window.QwenPaw.memoryBackends.register("memory-adbpg", {
  id: "adbpg", label: "ADBPG",
  configPath: ["memory_backend_configs", "adbpg"],   // 挂到配置树的路径
  tabKey: "adbpgMemory",
  ConfigComponent: ADBPGConfigCard,                  // 用宿主 antd Form.Item 渲染
});
```

## 10. 官方示例目录速查

```
plugins/tool/qwen-image/        tool 型：plugin.json + 入口 + 工具模块 + requirements.txt
plugins/channel/azure_bot/      channel 型：plugin.py + channel.py(BaseChannel) + __init__.py
plugins/memory/adbpg/           memory 型：plugin.py + backend/(config/client/manager) + frontend/ 配置卡片
plugins/middleware-demo/tracing-middleware/   仅 plugin.json + 单文件，最小中间件
```

共通点：`plugin.json` + 入口导出 `plugin` 实例 + `register(api)` 里只做「注册」，重活放在被注册的类/函数里。
