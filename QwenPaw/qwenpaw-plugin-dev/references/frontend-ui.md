# frontend-ui.md — 控制台前端 UI 扩展实战参考

来源：v2.2.1 tag 与 main 分支 `console/src/` 源码，并经 6 个实战插件验证（侧边栏应用入口、品牌词标替换、隐藏顶栏按钮、侧边栏可拖分隔栏、设置菜单精简、聊天头像替换）。**写任何 frontend 插件前通读本文；改 manifest 前另读 manifest.md。**

## 0. 四条铁律（每条都踩过坑）

1. **宿主把 antd 的 prefixCls 配置成 `qwenpaw`**（`App.tsx` 里 `<ConfigProvider prefixCls="qwenpaw">`）。真实 DOM 中只有 `qwenpaw-btn`、`qwenpaw-layout-header`，**不存在 `ant-*` 类**——写 `ant-btn` 选择器一条都匹配不上。选择器一律前缀无关：`[class*="layout-header"]`、原生标签、`[role=...]`。`.anticon-*` 类来自 `@ant-design/icons`，**不受 prefixCls 影响**（宿主自己的 less 也直接引用 `.anticon`），可作元素判别条件。
2. **vite 构建的 CSS module 类名保留局部名**：`vite.config.ts` 的 `generateScopedName: "[name]__[local]__[hash:base64:5]"`。宿主源码里 `styles.sessionArea` 在真实 DOM 中是 `…__sessionArea__哈希`，因此 `div[class*="sessionArea"]` 是稳定选择器。同名局部名可能出现在多个模块——务必加作用域与结构指纹防误伤（见 §4）。
3. **`window.QwenPaw.modules` 运行时是空的**：`dynamicModuleRegistry.registerHostModulesDynamic` 定义了但无人调用。不要指望从里面拿宿主页面组件；要复用宿主组件用 `route.wrap` 截获（§3.2）。
4. **插件脚本只在控制台启动空闲时执行一次**（PluginProvider.initialize → `installHostSdk()` 先挂好 chat 命名空间 → `loadAllPlugins()`：`GET /api/frontend_plugin` → fetch 入口 JS（带鉴权）→ Blob `import()`）。之后 SPA 会不断重渲染/重挂载宿主节点——一切 DOM 注入都要配 MutationObserver 幂等续命（§5），不能只做一次。

## 1. 宿主 API 精确语义（console/src/plugins/registry/）

### menu（侧边栏）

```js
QwenPaw.menu.add(pluginId, item | items)  // → Disposable；重复 id → no-op + audit
// item: { id, location, parentId, before, after, order, label,
//         icon, route, href, visible?, isGroup?, divider? }
```

- **高亮契约**：侧边栏选中态 = 当前路径匹配的路由 id（`routeSelection.ts` 的 `pickSelectedKey`），再与菜单项 id 比较。**菜单项 id 必须与它指向的路由 id 完全相同**，否则点击后高亮落在别的项上。
- `location`：侧边栏第一组是 `"primary.agentScoped"`，设置组是 `"primary.settings"`。内置项 order：收件箱 10、扩展（`core.marketplace`）15、导入 17、定时任务 20…（见 `layouts/registry/builtinMenu.ts`）。`before/after` 约束优先，`order` 作后备。
- `label` 传函数可在每次渲染时求值——按 `localStorage["language"]`（宿主 i18n 的语言存储键）返回「应用 / Apps」这类双语文案。
- `icon`：传 ReactNode 时用 span 包裹控制字号（`h("span",{style:{fontSize:16,display:"inline-flex"}}, h(host.antdIcons.AppstoreOutlined))`）；侧边栏以 16/18px 渲染。
- 旧 shim `registerRoutes(pluginId, routes)` 会自动合成 "Plugins" 分组、路由 id 变成 `legacy:<pluginId>:<path>`——需要精确控制位置/高亮时直接用 menu.add + route.add。

### route（页面路由）

```js
QwenPaw.route.add(pluginId, { id, path, component })   // id 或 path 重复 → no-op + audit
QwenPaw.route.wrap(pluginId, targetRouteId, wrapper)   // 见下
```

- 路径避开内置：`/market`、`/apps/:appId`、`/chat/*`…；MainLayout 会过滤掉 `/apps/<id>` 形态（应用内嵌专用），自己的路径别撞 `/apps/` 前缀。
- **`wrap` 在注册时立即应用**：registry 每次 invalidate 都会对目标路由重放 wrapper（`resolveAll`），wrapper 以 `(Inner) => Component` 调用。因此 wrapper 会被**多次**调用——必须幂等，且**原样返回 Inner**（宿主路由行为不变），把 Inner 存下来即为对宿主组件的引用。这是复用宿主页面（如 `core.marketplace` 的「扩展」页）的唯一干净通道。宿主页面组件多为 React.lazy，渲染时包一层 `<Suspense>`。
- 必须留兜底：若捕获不到（宿主未来改名/改机制），页面自动跳转到原路由路径。

### slot（布局填充点）

```js
QwenPaw.slot.replace(pluginId, name, render)  // render(defaultContent) => ReactNode
QwenPaw.slot.fill(pluginId, name, render, opts)
```

- replace 只取**最后一个**；render 返回 `null/undefined` 时回退宿主默认内容；外层有 SlotErrorBoundary，崩溃不会拖垮布局。
- 已知 slot：`header.left` / `header.right`（Header 填充）、**`header.logo`（replace，包着 AppBrand 的 QwenPaw 词标图——侧边栏顶部与设置中心顶栏共用）**、`sider.top`（上区内）、`sider.bottom`（历史对话区之后）、`content.statusBar`、`overlay.global`。
- 品牌换字套路：`slot.replace(id, "header.logo", () => h("span",{style:{fontSize:19,fontWeight:700,color:"inherit"}},"PingZhi"))`——文字用 `currentColor` 深浅主题自适应；版本徽标、折叠按钮都不受影响。

### chat（聊天界面标量）

```js
QwenPaw.chat.welcome.set(pluginId, { greeting?, description?, avatar?, nick?, prompts? })
```

- `avatar` **字符串按图片 URL 渲染**（默认 `"/qwenpaw.png"` 小吉祥物，见 `pages/Chat/index.tsx` 的 `avatar: extAvatar ?? "/qwenpaw.png"`）。传 SVG data URL（`data:image/svg+xml;charset=utf-8,` + `encodeURIComponent(svg)`）即得自适应头像：欢迎页大圆、回复卡片小头像走同一标量（`install.ts` 里 response 默认身份复用 `welcome.avatar`），一次设置全局生效。
- `greeting/description/...` 支持 `Localized<T>`（值或 `(locale)=>T`）。
- 二级界面定制同法：`chat.leftHeader.set/render`、`sender.set`、`response.set` 等（完整面见 `types/qwenpaw.d.ts`）。

## 2. 实战套路速查（需求 → 扩展点）

| 需求 | 套路 | 要点 |
|---|---|---|
| 侧边栏加菜单项 + 独立页面 | route.add 新路由 + menu.add（**id = 路由 id**） | `after:"core.marketplace"` + `order:16` 双保险；manifest `type:"frontend"` 只有 `entry.frontend` |
| 页面内容直接复用宿主现有页面 | route.wrap 截获宿主组件 + 自己的路由渲染（Suspense 包住）+ 失败兜底跳转 | wrapper 幂等、原样返回 Inner |
| 改品牌词标（左上角 logo） | slot.replace("header.logo") | 返回 null 会回退默认；配色用 currentColor |
| 隐藏宿主硬编码按钮/元素 | 注入 `<style>`，前缀无关选择器 | 用 anticon 类或结构指纹判别，范围收敛到最小容器 |
| 两块宿主区域之间加东西（拖动条等） | MutationObserver 找容器 + `insertAdjacentElement` + data 标记幂等 | 上/下区定位用 `[class*="局部名"]` |
| 让注入在折叠/展开、Popover 开合后存活 | 观察者幂等重建；宿主 Popover 多带 `destroyOnHidden`（每次打开全新节点） | 处理过的节点打 data 标记防重复 |
| 精简「一列按钮」式菜单 | 结构指纹定位容器（前 N 个子元素都是 button）→ 隐藏第 N+1 个起 | 不要按文案匹配（i18n 会变） |
| 改聊天欢迎页头像/问候语/昵称 | chat.welcome.set({ avatar: dataURL }) | 回复卡片同步生效 |

## 3. 选择器与定位规则（血泪教训）

- 反面教材：`header.ant-layout-header button.ant-btn:has(.anticon-down)` 在真实应用一条都匹配不上（prefixCls=qwenpaw），导致「隐藏文档资料/GitHub 按钮」第一版失效。修复写法：

  ```css
  header[class*="layout-header"] button:has(.anticon-down),
  header[class*="layout-header"] button:has(.anticon-github) { display: none !important; }
  ```

- 「局部名 + 结构指纹」双层防护示例（齿轮设置菜单）：锚点 = overlay 的 `[class*="quickSettingsPopover"]`（overlayClassName 局部名，全局唯一）；面板根 = 其中「前三个子元素都是 `<button>` 且前两个带 `aria-haspopup="menu"`」的 div；命中后隐藏第 4 个起的子元素。指纹不匹配（宿主改版）就**不动手**——宁可失效不要猜。
- 语言/文案永远不要当定位依据（i18n 多语言）；aria 属性、标签序列、局部名才是稳定特征。

## 4. DOM 注入的生存策略

- 宿主重渲染场景：侧边栏折叠/展开（整组节点重挂载）、Popover `destroyOnHidden` 开合、路由切换、移动端抽屉。
- 标准模式：

  ```js
  // 启动: injectStyle(); ensure(); new MutationObserver(debouncedEnsure).observe(document.body, {childList:true, subtree:true});
  function ensure() {
    var parts = findParts();                    // 局部名 + 结构指纹
    if (!parts) return;                          // 目标不存在（折叠态等）→ 静默退出
    var h = document.getElementById(MY_ID);
    if (h && h.isConnected && h.parentElement === parts.parent) return;  // 已就位
    if (h) h.remove();
    parts.anchor.insertAdjacentElement("afterend", buildHandle());
  }
  ```

- 拖动实现：pointerdown/move/up + `setPointerCapture`（try/catch 包住，合成事件会抛）；拖动结束才写 localStorage；`dblclick` 重置；↑/↓ 键盘微调；拖动期间给 body 加 class（`cursor:row-resize` + `user-select:none` + 其余元素 `pointer-events:none`）。
- **数值漂移陷阱**：宿主元素多为 content-box（padding+border 不计入 flex-basis）。用 flex-basis 固定高度时同时设 `style.boxSizing="border-box"`，存取都用 `Math.round(getBoundingClientRect().height)` 的整数外框值——否则每刷新一次尺寸涨一截。
- React 内联样式共存：对宿主元素设置 inline style 后，恢复用 `style.removeProperty(...)`（clearSplit 模式），不要留半套样式。

## 5. 侧边栏内部结构速查（v2.2.1）

```
aside.qwenpaw-layout-sider（.sider：height 100vh, overflow auto, padding 0 16px；.siderExpanded 时 display:flex column）
├─ AppBrand：header.logo slot（词标图）+ 版本徽标 + 折叠按钮
├─ div [class*="agentScopedSection"]        ← 上区，flex-shrink:0
│   ├─ AgentSelector、Slot sider.top、新建任务按钮
│   ├─ div [class*="navigationScroll"]      ← 导航滚动区，宿主默认 max-height: min(198px, 30vh)
│   └─ 更多设置按钮
├─ div [class*="sessionArea"]               ← 下区（历史对话），flex:1, min-height:0
└─ Slot sider.bottom（+ authActions，authEnabled 时）
```

- 折叠态不渲染上/下区（走 collapsedNav 分支）——注入物必须处理「消失/重现」。
- 「可拖分隔栏」原理：手柄插在两区之间；拖动时上区 `flex: 0 0 Hpx`（+ box-sizing:border-box + 内部导航区 `flex:1 1 auto; max-height:none` 吸收变化），下区保持宿主 `flex:1` 自动收缩；双击 `removeProperty` 全部内联样式恢复默认。

## 6. 齿轮设置菜单（SidebarSettingsPanel）速查

- 组件：`layouts/SidebarSettingsPanel.tsx`；不是 antd Menu，是一列普通 `<button>` + `div.divider`，渲染在齿轮按钮的 Popover 内，**`destroyOnHidden` 每次打开重新挂载**。
- 顺序：外观(FlyoutItem)、消息展示(FlyoutItem)、设置、分隔线、教程、更新日志、常见问题、分隔线、关于 QwenPaw(+版本)，authEnabled 时末尾还有分隔线+账户+退出登录。
- 精简套路：锚 overlay → 指纹找面板根 → 隐藏 `children[3..]`。外观/消息展示的二级弹层（语言、主题、宽度等）是独立 portal，不受影响。
- 弹层若需修改：嵌套 Popover 的 overlay 类名局部名 `nestedPopover`，内容根局部名 `flyoutPanel`。

## 6b. 聊天页真实 DOM 链路（改对话界面必读）

用户实测（v2.2.1 暗色模式 DevTools）聊天页容器链，vendor（@agentscope-ai/chat）与宿主混合：

```
html.dark-mode → body#mnt → .qwenpaw-app
└─ .qwenpaw-layout.qwenpaw-layout-has-sider [class*=mainLayout]
   ├─ aside.qwenpaw-layout-sider [class*=sider]            ← 侧边栏（AppBrand 在顶部）
   └─ .qwenpaw-layout [class*=mainContentLayout]
      ├─ header.qwenpaw-layout-header [class*=header]
      └─ main.qwenpaw-layout-content [class*=content]      ← html.dark-mode 下 background: var(--app-bg) !important（layout.css）
         └─ .page-container
            └─ [class*="chatPageRoot"] → [class*="chatWindowDrop"] → [class*="chatAnyWhere"]
               └─ .qwenpaw-chat-anywhere-layout
                  ├─ .qwenpaw-chat-anywhere-header（模型选择条）
                  ├─ .qwenpaw-chat-anywhere-content
                  │  └─ .qwenpaw-chat-anywhere-message-list-wrapper
                  │     └─ .qwenpaw-bubble-list-wrapper     ← vendor 暗色规则 background: var(--app-bg) !important（特异性 (0,2,1)）
                  └─ .qwenpaw-chat-anywhere-input
```

- **改对话区背景的套路**（theme-deepseek v1.3.0 最终方案）：**不要用"容器透明化 + 后置画布"**——消息列表内部还有未见过的 vendor 不透明层，透明链必然漏（v1.2.0 实测失效）。正确做法两层：
  1. 渐变背景直接写成对话区容器自己的 `background: linear-gradient(...) !important`（在 `html.dark-mode.theme-ds-chat` 前缀 + 双类名 (0,4,1) 下），覆盖 vendor 的 `var(--app-bg) !important`——背景就在容器上，不存在遮挡问题；
  2. 鲸鱼粒子等"动"的部分用**前景叠加画布**：fixed 全屏、`pointer-events:none`、z-index 500（高于内容、低于 antd 弹层 1000+），只画粒子不画背景，不依赖任何容器透明度，必然可见。
  3. 激活条件用 MutationObserver（html class）+ 轻量 setInterval（SPA 路由）双通道启停；亮色模式不透明化（深底浅字不可读）。
- 全局 body 层画布（v1.1.0 教训）：会被 `.qwenpaw-layout-content`/vendor 容器的不透明背景完全盖住——背景效果永远不要指望"垫在底层透出来"。

## 7. 本机没有 QwenPaw 时的验证方法

1. **源码对照**：`git clone --depth 1` + `git fetch --depth 1 origin tag <用户版本>` + worktree；行为以用户版本 tag 为准（main 只用来确认 API 演进方向）。所有「宿主会怎么渲染」的结论必须落到具体源码行，禁止凭印象。
2. **mock 宿主 harness**：按已核实的语义仿写注册表（menu/route/slot/chat 标量存储、wrap 立即应用、重复 id no-op）+ 真实 React UMD（unpkg）+ 按真实类名风格（`qwenpaw-layout-header`、`index__局部名__哈希`）搭静态结构；用浏览器（如 browser-use）做 DOM 断言 + 截图；Popover 类需求要模拟「重挂载后再处理一遍」。
3. `node --check` 校验插件 JS 语法；打包 zip（压缩包内是单个插件目录或根上有 plugin.json）。
4. 有目标机时：`qwenpaw plugin validate` → `install` → **刷新控制台**（前端插件只在启动时加载）→ 按需求逐条目检；卸载重装验证幂等。

## 8. 已知不可用 / 不要碰

- `window.QwenPaw.modules`：运行时为空（见铁律 3）。
- 不要用 `ant-*` 类写选择器（铁律 1）。
- 宿主未暴露 react-router——插件组件里拿不到 `useNavigate`，跳转用 `window.location` / `history`。
- 不要按菜单文案定位（i18n 会变）。
- legacy `registerRoutes` 的路由 id 与自动分组不可控（见 §1 menu）。
- 审计自查：改完把 `QwenPaw.audit.overrides()` 打到控制台，确认没有 `menu.conflict` / `route.conflict`。
