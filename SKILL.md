---
name: coding-field-notes
description: AI 编码/调试的实战坑位清单，覆盖前端 PWA、IndexedDB、Service Worker、Playwright、Capacitor/Android WebView、CSS 移动端与多轮回归。当你要写或改 Web/PWA/移动封装应用，或遇到白屏、旧代码不生效、样式错位、测试误报、APK 缺文件、源码被改动、异步竞态时加载。
whenToUse: 用户在讨论或开发前端/单页应用/PWA/Android 封装；或出现“页面空白、改了没生效、点击被挡、320 窄屏、离线打不开、APK 打包、测试通过但实际不对”等信号时。纯后端/算法/数据任务不需要。
---

# AI 编码实战坑位清单（Coding Field Notes）

> 目的：把「看起来能跑」变成「有证据地能用」。
> 用法：动手前扫一眼第 0 节；出错时按症状跳到对应节；修完补一条回归证据。

## 0. 交付判据（先立规矩，再写代码）

一条修复只有同时满足下面三条才算完成：

1. **复现**：有最小步骤/脚本能稳定看到问题。
2. **根因**：能指出是哪个文件、哪一行、哪个 API 契约错了。
3. **证据**：修复后有命令输出、DOM 断言、截图、哈希或测试计数，而不是「我觉得好了」。

改动前先做三件事：

- 记录当前测试基线（哪些通过、哪些失败）。
- 如果是「源文件必须保留」类任务，先做**哈希快照**，并明确哪些目录算源文件、哪些是测试产物。
- 找出现有约定（目录、命名、mount/unmount、错误处理），不要另起一套。

## 1. 前端 DOM 与状态

### 1.1 `clear(root)` 会把常驻节点一起摘掉

**症状**：首屏只有顶栏；搜索框、文件选择器、滚动容器消失；事件绑了但点不到。

**根因**：`clear(root)` 清空所有子节点，之后只把内容 append 到已经被摘掉的子容器上，从未重新挂回 `root`。

**修法**：

```js
clear(root);
root.appendChild(header);
root.appendChild(scroll); // 关键：clear 后重新挂回
clear(scroll);
scroll.appendChild(content);
```

**预防**：`mount()` 必须幂等；把「壳节点」和「内容节点」分开，重绘只清内容节点，不反复重建壳。

### 1.2 覆盖层挡住点击

**症状**：Playwright 报 `element is not visible / intercepted by .overlay__backdrop`；真机上点按钮没反应。

**排查**：

- 查 `.overlay.is-open`、`.drawer.is-open` 是否仍存在。
- 注意：`page.goto(BASE + '#/route')` 若只改变 hash，**不会重新加载页面**，首启 18+ 弹层还在。
- 修法：测试中先 `page.reload()` 或显式关闭弹层；应用内路由切换要确保 `closeAllOverlays()`。

### 1.3 重复事件

**症状**：双击一次收藏被切换两次；点一次发送发两条。

**根因**：同时绑定 `click` 双击检测和 `dblclick`；或 `mount()` 重入时重复 `addEventListener`。

**修法**：只保留一种手势实现；事件用 `AbortController`，卸载时 `abort()`；或在节点替换后重新绑定。

### 1.4 声明顺序 / TDZ

**症状**：`Cannot access 'viewApi' before initialization`，页面白屏。

**根因**：`new MessageList({ view: viewApi })` 写在 `const viewApi = {...}` 之前。

**修法**：先定义依赖，再构造；大组件用函数声明或延迟到 `mount()` 内。

## 2. CSS 与移动端

### 2.1 类挂在控件上，CSS 却写后代选择器

**症状**：输入框是浏览器默认灰底；placeholder 颜色不对；`height` 不生效。

**根因**：JS 写 `h('input', { class: 'field' })`，CSS 只写 `.field input { ... }`。

**修法**：同时支持直接类：

```css
input.field, textarea.field, select.field { /* ... */ }
```

加一个回归：查 `getComputedStyle` 的 `backgroundColor/height`，不要只看截图。

### 2.2 开关状态与结构不一致

**症状**：开关能点，但视觉永远关闭；或设置里开关不可见。

**根因**：

- JS 生成 `button.switch.is-on`，CSS 只写 `:has(.switch-input:checked)`。
- JS 传裸 `input.switch-input`，CSS 只认 `.switch` 外壳。

**修法**：统一组件契约：`.switch` 外壳 + `.switch__knob` + 输入/按钮二选一；两种状态都写：

```css
.switch.is-on::before,
.switch:has(.switch-input:checked)::before { background: var(--amber); }
```

### 2.3 窄屏与固定底部

- 320px 必须单独跑：统计条、标签条、按钮组、双列网格最容易溢出。
- 禁止页面横向滚动；内部条可以 `overflow-x:auto`，但必须有可见滚动提示，不能裁掉关键操作。
- 固定 footer/抽屉底部会盖住内容：渲染后把滚动容器 `scrollTo(0)`；内容区留 `padding-bottom`。
- 等宽双列按钮放长中文时，加 `white-space:nowrap` 并缩小水平 padding；不要靠换行解决。
- 触达目标 ≥44×44：用 Playwright 量 `getBoundingClientRect()`，别靠目测。
- 软键盘用 `visualViewport`，不要只用 `100vh`；`env(safe-area-inset-*)` 要留。

### 2.4 状态色/主题变量

- 只用一套 token（`--bg/--surface/--text/--muted/--amber/--danger`），不要在不同组件里发明 `--bubble-char` 之类的新变量。
- 换主题时同时更新 `color-scheme` 与 `theme-color` meta；否则原生控件/状态栏不跟随。

## 3. 数据与 API 契约

### 3.1 返回形状不一致

**真实坑**：

- `truncateToTokens()` 返回 `{ text, tokens, truncated }`，调用方却当字符串塞进 `content`，后续 `content.match is not a function`。
- `exportCardPng()` 直接返回 `Blob`，调用方却读 `png.blob`。

**规则**：跨模块边界先归一化：

```js
const text = typeof value === 'string' ? value : String(value?.text ?? '');
const blob = raw instanceof Blob ? raw : raw?.blob;
```

在 API 入口用 JSDoc/单测固定契约；改返回值时全仓搜索调用点。

### 3.2 Set / Map / 数组混用

**症状**：世界书命中条目全是 `undefined`；预览显示「无标题」；注入为空。

**根因**：扫描函数返回 `Set` 只存 key，渲染层却按对象数组读 `uid/comment/content`。

**修法**：去重和结果分开：

```js
const seen = new Set();
const matched = new Map(); // id -> 完整对象
```

返回前 `[...matched.values()]`；trace 里保留原始字段，不要只留 key。

### 3.3 IndexedDB 异步竞态

- `saveSettings()` 后立刻整页 reload，可能还没落库，首启弹层又出现。测试要等待 save resolve 或等 UI 状态。
- 数据库升级改 `DB_VERSION`，在 `onupgradeneeded` 建新 store；旧连接失败时把 `dbPromise = null` 允许重试。
- 临时流程（试穿/草稿）只写独立 store；「转正」是唯一写正式表的入口。写库前判重（内容哈希），保证幂等。
- 写入后要能证明：查 store 计数、查绑定关系，不要只看 toast。

## 4. Service Worker / PWA

### 4.1 改了代码但页面还是旧的

**根因**：SW 是 cache-first，模块被旧缓存命中；或 HTML 被缓存但 JS 没更新。

**修法**：

- 开发时 unregister SW + 删除 cache 再测。
- 每次发布 bump `CACHE/VERSION`，`install` 里 `skipWaiting()`，`activate` 里清旧 cache + `clients.claim()`。
- 只缓存应用壳；**绝不缓存 API、密钥、聊天响应**。
- 插件 entry 带 `?v=` 时，`cache.match(key, { ignoreSearch: true })`；壳路径用 `Set` 精确判断，不要 `startsWith` 误伤 `/v1`。

### 4.2 离线测试的假阳性

- `navigator.onLine` 在 headless 下可能仍为 true，不能作为离线判据。
- 用 Playwright `context.setOffline(true)` + 同源网络探针返回 `blocked`；忽略探针自身的 `ERR_INTERNET_DISCONNECTED`。
- 离线要断言：应用壳能开、演示数据能看、插件面板能加载、没有白屏。

## 5. Playwright / 测试

### 5.1 启动与模块解析

- `chromium.launch()` 默认找 `chromium_headless_shell-*` 可能不存在；显式 `executablePath` 指向已安装的 `chrome.exe`。
- `playwright-core` 不在项目 node_modules 时，设置 `NODE_PATH` 后调用 `require('module').Module._initPaths()`。
- 每个 context 独立，IndexedDB 不串；但同一 page 的 hash 导航不会 reload。

### 5.2 断言要断到根上

- 多角色群聊「最后一条是谁」很容易误判：点名的角色先回，其他角色随后也能接话。验收应断言「用户消息之后的第一条回复是谁」，不是最后一条。
- 不要只断言 `toast`；同时查 DB（`cards/chats/worlds` 计数、绑定 id、hash）。
- 收集 `pageerror` 与 `console.error`，并断言为空；网络 4xx/5xx 单独收集。
- 截图用于人看，不能代替断言；断言通过也要跑 320/390 两档。

### 5.3 测试数据与真实数据

- 测试写库用独立 context/临时 profile；不要污染交付库。
- mock 服务的行为不是产品行为：mock 复述「续写请求」、带「设定文本：」前缀，都应在 mock 层修，不要改主程序迁就。
- 大列表压测：注入 120 条，断言挂载节点数保持窗口大小、p95 帧耗时；不要只截图。

## 6. Capacitor / Android WebView

### 6.1 `cap sync` 会删掉 `assets/public/plugins`

**根因**：Capacitor 的 Cordova 清理逻辑会 `remove(pluginsDir)`。

**修法**：每次 `cap sync` 后，手动把 `www/plugins` 复制回 `android/app/src/main/assets/public/plugins`，再 Gradle 构建。顺序不能反。

**更稳**：不跑 `cap sync`；直接 `www` → `assets/public` 复制（保留 `cordova.js` 等运行时文件），再构建。

### 6.2 WebView 不支持 blob 下载

`URL.createObjectURL` + `a[download]` 在 WebView 里不会落盘。需要原生桥：

- `addJavascriptInterface(bridge, "PTNative")`；
- JS 把 Blob 分块 base64 传给 `begin/chunk/end`；
- Android 10+ 写 `MediaStore.Downloads`，低版本写 app-specific 目录；
- 桥不存在时 JS 必须 no-op，不能报错。

### 6.3 打包验证

不要只看 `BUILD SUCCESSFUL`。至少查：

- `aapt2 dump badging`：包名、versionCode/Name、label、launchable activity。
- `apksigner verify --print-certs`：v1/v2 通过，证书指纹正确。
- `zipalign -c -v 4`。
- 解包 APK：`assets/public` 里确实有新增模块、插件、`native-download.js`；SW 版本正确。
- 把 APK 的 `assets/public` 单独用 Chromium 跑一遍冒烟，或直接跑关键 Playwright 用例。
- 没有真机/模拟器就明说「未做启动验证」，不要把静态校验说成安装测试。

### 6.4 源文件保护

- 构建工程放在源目录之外；`www` 只复制运行时文件，不带 README/dev。
- 构建前后做源哈希清单；测试会生成 `dev/` 截图，比较时排除这些产物，否则会误报「源码被改」。
- 覆盖安装：同签名 + 更高 `versionCode` 才能保留数据；新旧包都保留，给回滚路径。

## 7. Windows / 工具链

- 含空格/中文的路径：`Start-Process` 用 `-WorkingDirectory` + `-ArgumentList` 数组，不要拼一整条带引号的命令。
- 文本编辑一律用 write/edit；不要用 PowerShell 做批量替换/重写，UTF-8 中文会被弄坏。
- 二进制、复制、哈希、解压可以用 PowerShell/Python。
- 端口占用/进程秒退：先看退出码和监听端口，不要连续重试同一条命令。

## 8. 修复流程（固定动作）

1. **复现**：最小页面/脚本，记录控制台和 DOM。
2. **定位**：从症状逆推到数据/生命周期，不先改样式。
3. **最小修**：只改一个契约点，不顺手重构。
4. **回归**：原用例 + 相邻用例 + 320/390 + 离线/打包（如适用）。
5. **证据**：命令输出、计数、截图、哈希；写进交付说明。
6. **收尾**：bump SW/版本、更新 README/novice、说明未验证项。

## 9. 一分钟自检

- [ ] 首屏/刷新/路由切换都不白屏？
- [ ] 常驻节点在 `clear` 后重新挂回？
- [ ] 输入框/开关/标签在 320px 不溢出、不折行？
- [ ] 所有触达 ≥44×44？
- [ ] API 返回形状在边界归一化？
- [ ] 试穿/草稿只写临时 store，转正才写正式表？
- [ ] SW 版本已 bump，离线能开且插件在？
- [ ] Playwright 断到 DB/第一条回复，不只断 toast？
- [ ] APK 内新增文件齐全，签名/zipalign 通过？
- [ ] 源哈希前后一致（排除 dev 产物）？
- [ ] 没验证的部分明确写出来？
