# plugin.json 字段全表

Schema 来源：`src/qwenpaw/plugins/architecture.py` 的 `PluginManifest`（Pydantic 模型，`extra="ignore"` —— 未知顶层字段被静默忽略，因此 `pack_exclude`、`icon`、`capabilities` 等扩展字段可以安全携带，由其他组件消费，不参与加载逻辑）。

## 字段一览

| 字段 | 必填 | 类型 | 默认值 | 说明 |
|---|---|---|---|---|
| `id` | **是** | str（非空） | — | 全局唯一 ID。**同时作为安装目录名**（装到 `~/.qwenpaw/plugins/<id>/`，有路径穿越防护）。入口模块名为 `plugin_<id 中 - 换 _>`。推荐小写字母+连字符。 |
| `version` | **是** | str（非空） | — | 插件自身语义化版本 |
| `name` | 否 | str | 缺失/为空时回退为 `id` | 显示名 |
| `description` | 否 | str | `""` | 描述。兼容 i18n 映射写法 `{"zh-CN": ..., "en-US": ...}`（取 en-US > en > zh-CN > zh 第一个非空值） |
| `description_i18n` | 否 | dict[str,str] | `{}` | 多语言描述，仅供 UI 展示 |
| `author` | 否 | str | `""` | 作者（同样兼容 i18n 映射） |
| `entry.backend` | 与 frontend 至少其一 | str | null | 后端 Python 入口文件（相对插件根目录），如 `"plugin.py"`、`"backend/main.py"`。旧写法顶层 `entry_point` 会自动合并进来 |
| `entry.frontend` | 同上 | str | null | 前端 JS bundle 入口，如 `"ui/index.js"`、`"frontend/dist/index.js"` |
| `dependencies` | 否 | list[str] | `[]` | Python 依赖（主要用于展示；**实际安装读的是目录里的 `requirements.txt`**） |
| `qwenpaw_version` | 否 | `{"min": str, "max": str?}` | null | 推荐的版本约束，语义 `>=min, <max`，见下 |
| `min_version` / `max_version` | 否 | str | `"0.1.0"` / null | 旧版兼容字段，仅当没有 `qwenpaw_version` 时参与判断 |
| `meta` | 否 | dict | `{}` | 自由元数据，见下节 |
| `type` | 否 | str | 从 meta 推断 | 插件类型，见下节 |

`qwenpaw plugin validate` 的必填检查是 `id`、`name`、`version`（比 Pydantic 严），外加后端入口可导入、前端文件存在（缺失仅告警）。

## 版本约束 `qwenpaw_version`

```json
"qwenpaw_version": { "min": "2.0.1", "max": "2.2.0" }
```

- 语义为左闭右开区间 `>=min, <max`。
- **`max` 省略时自动推导为 `{major}.{minor+1}.0`**：min=2.0.1 → 允许 2.0.x 全部补丁版（上界 2.1.0）。
- 当前运行版本是预发布（如 `2.0.0b2`）时按基线版本参与比较。
- 当前实现临时只强制 `>= min`（上界检查被注释掉）；不兼容的插件仍在列表中显示，但 `enabled=False`、不执行 `register()`，并给出诊断信息。

## `type` 枚举与推断

合法值（`PluginType` 枚举）：

```python
tool      # 注册 agent 工具
provider  # 注册自定义 LLM Provider
hook      # 应用启动/关闭时执行代码
command   # 注册 /slash 命令
channel   # 消息渠道
memory    # 长期记忆后端（必须在 workspace 启动前注册）
frontend  # 纯前端 JS 插件
app       # PawApp 完整应用
general   # 兜底
```

`type` 缺失或非法时按 `meta` 推断（顺序如下，命中即停）：

| meta 信号 | 推断为 |
|---|---|
| `meta.tools` 或 `meta.tool_name` | tool |
| `meta.chat_model` 或 `meta.provider_id` | provider |
| `meta.hook_type` | hook |
| `meta.command_name` 或 `meta.commands` | command |
| `meta.channel` | channel |
| 有 `entry.frontend` | frontend |
| 其余 | general |

**`memory` 无法被推断**——记忆型插件必须显式写 `"type": "memory"`。仓库里还有 `"type": "bundle"` 的写法（枚举外，按 general 加载），仅作目录分类用。PawApp 的权威判定依据是 **`meta.pawapp` 是否存在**，`"type": "app"` 用于在 App Center 正确分类展示。

## `meta` 的用途

### 工具配置声明（tool 型）：`meta.tools[]`

声明 UI 配置表单（App Center / 设置页渲染），每个工具一个条目：

```json
"meta": {
  "tools": [
    {
      "name": "generate_image_qwen",
      "description": "Generate images from text prompts",
      "icon": "🖼️",
      "requires_config": true,
      "config_fields": [
        { "name": "api_key", "label": "API Key", "type": "password", "required": true },
        { "name": "model", "label": "Model", "type": "select",
          "options": ["qwen-image", "wan-x"], "default": "qwen-image" },
        { "name": "timeout", "label": "Timeout(s)", "type": "number", "min": 1, "max": 300 }
      ]
    }
  ],
  "api_key_url": "https://.../apikey",
  "api_key_hint": "获取 API Key 的说明"
}
```

- `config_fields[].type` 取值：**工具只允许 `text | password | number | select`**（渠道才有 `switch`）。
- 其余可用键：`label`、`required`、`placeholder`、`help`、`default`、`options`、`min`、`max`。
- 旧的单工具格式 `meta.tool_name` 仍兼容（卸载清理、离线安装同步到 agent 配置时会读取）。
- 运行时读取配置：工具函数内 `from qwenpaw.plugins import get_tool_config`。

### 记忆后端（memory 型）：`meta.memory_backends`

```json
"meta": { "memory_backends": [{ "id": "adbpg", "label": "ADBPG" }] }
```

### PawApp 声明：`meta.pawapp`

```json
"meta": {
  "pawapp": {
    "icon": "📋",                       // emoji，或用 icon_url 指向打包内静态资源
    "icon_url": "/api/frontend_plugin/<id>/files/ui/dist/app/logo.png",
    "category": "productivity",         // App Center 分类过滤
    "entry_page": "/apps/agent-kanban", // 前端注册的路由路径
    "launch_scope": "page"              // 默认 "page"
  },
  "permissions": { "chat": true, "storage": true, "network": ["loopback"] },
  "settings": []                        // App Center 设置表单；运行时 ctx.settings.get(key) 读取
}
```

### 其他

- `meta.runtime_dependencies`、`meta.features`：app 型插件的运行依赖/特性声明（qwenpaw-data 用法）。
- `meta.hook_type` / `meta.command_name` / `meta.channel` / `meta.provider_id`：类型推断信号。
- `pack_exclude`：发布 zip 中剔除开发文件（如 `["tests", "frontend/src", "frontend/package.json"]`）。
- `pack_requires`：app 型打包时必须包含的 dist 产物清单。

## 发现与加载行为

- 唯一加载目录 `~/.qwenpaw/plugins/`（`QWENPAW_WORKING_DIR` 可改根），扫描**一级子目录**中直接含 `plugin.json` 的。
- 跳过：`.` 开头目录、`.disabled` 结尾目录（禁用方式）、含 `state=="prepared"` 的 `.qwenpaw-pawport.json` marker 的目录。
- 两阶段启动：**Phase 1** 只加载 `channel` 和 `memory` 类型（必须在 agent/workspace 启动前）；**Phase 2** 加载其余全部。
- 依赖安装：检查 `requirements.txt`，缺失则跨进程文件锁后 `pip install`（失败回退 `uv pip install`；frozen 桌面版装进 `WORKING_DIR/plugin_runtime/.../site`）。
- 加载失败会完整回滚（registry、sys.modules、sys.path、命名空间隔离），只记 error 日志，不影响其他插件。
- 插件间内部模块通过私有命名空间重定向（裸导入指向插件自己目录），不同插件的同名顶层模块不冲突。
- HTTP 路由挂在 Console SPA catch-all 之前；`register_http_router` 的 prefix 全局独占，重复注册抛 `ValueError`。

## 发布与分发

- 打包：`zip -r my-plugin-1.0.0.zip my-plugin/`（要求压缩包内是单个插件目录或根上有 plugin.json）。
- 安装：`qwenpaw plugin install <zip|目录|URL>`（运行中热安装；URL 下载上限 500MB，防 Zip Slip）。
- 上架：在 AgentScope Platform（platform.agentscope.io）发布；官方 CDN 目录 `download.qwenpaw.agentscope.io` 按用户 QwenPaw 版本过滤条目——发布时 manifest 的版本约束要与市场索引条目一致。
