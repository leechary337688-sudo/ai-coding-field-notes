# AI Coding Field Notes

> 一份给 AI 编码 agent 看的实战坑位清单：前端 PWA、IndexedDB、Service Worker、Playwright、Capacitor / Android WebView、移动端 CSS 与多轮回归。

不是“最佳实践合集”，而是从真实项目里踩出来的**症状 → 根因 → 修法 → 预防**。目标只有一个：把“看起来能跑”变成“有证据地能用”。

## 这个 skill 解决什么

AI 写代码常见的失败不是不会写，而是：

- 改了代码，页面还是旧的（Service Worker 缓存）
- 首屏只有顶栏（`clear(root)` 把常驻节点摘掉了）
- 测试全绿，但用户点不到按钮（透明 overlay 挡着）
- 构建成功，APK 里却少了插件目录（`cap sync` 清理）
- API 返回值形状变了，调用方还在按旧结构读
- 320px 下按钮换行、横向滚动、触达目标小于 44px
- 说“已修复”，但拿不出命令输出、DOM 断言或哈希证据

这些坑的共同点是：**局部看起来对，整体契约错了**。这份清单把它们按层拆开，方便在出错时快速定位。

## 安装

把整个目录放到你的全局 skills 目录：

```text
~/.dsh/skills/ai-coding-field-notes/SKILL.md
```

Windows 对应：

```text
C:\Users\<你>\.dsh\skills\ai-coding-field-notes\SKILL.md
```

重启会话后，技能目录里会出现 `coding-field-notes`。它是**按需加载**的：不会每轮把全文塞进上下文，只在任务匹配描述时展开。

> 也适用于其他读取 `SKILL.md`（YAML frontmatter + Markdown）的 agent 运行时；不同运行时的触发机制可能不同。

## 内容地图

| 节 | 覆盖 |
|---|---|
| 0. 交付判据 | 复现、根因、证据；源文件哈希快照 |
| 1. DOM 与状态 | `clear(root)`、overlay 挡点击、重复事件、TDZ |
| 2. CSS 与移动端 | 类选择器契约、开关结构、320px、44px、安全区 |
| 3. 数据与 API | 返回形状、Set/Map 混用、IndexedDB 竞态、临时 store |
| 4. Service Worker / PWA | 旧缓存、bump 策略、离线测试假阳性 |
| 5. Playwright / 测试 | executablePath、hash 导航、断言到 DB/第一条回复 |
| 6. Capacitor / Android | `cap sync` 删插件、WebView 下载桥、APK 验证、源文件保护 |
| 7. Windows / 工具链 | 中文路径、`Start-Process` 参数、UTF-8 |
| 8. 修复流程 | 复现 → 定位 → 最小修 → 回归 → 证据 → 收尾 |
| 9. 一分钟自检 | 发布前逐条勾选 |

## 几条最常复发的坑

1. **`clear(root)` 之后没把常驻容器挂回去**
   先 `clear(root)`，再 append header/scroll，最后才清内容层。

2. **跨模块返回值形状不一致**
   `truncateToTokens()` 返回对象却被当字符串；`exportCardPng()` 返回 Blob 却被读 `.blob`。边界先归一化。

3. **Service Worker 缓存优先**
   开发时清 SW + 删 cache；发布 bump 版本；只缓存壳，不缓存 API/密钥。

4. **测试里的 hash 导航不刷新页面**
   `page.goto(BASE + '#/cards')` 不会重新加载；首启弹层还在。测试要 `reload` 或显式关 overlay。

5. **CSS 契约和 JS 结构没对齐**
   类挂在控件上、CSS 却写后代选择器；开关有 `.is-on`、CSS 只写 `:has(input:checked)`。

6. **`cap sync` 会删 `assets/public/plugins`**
   sync 后手动复制插件再 Gradle；或者直接复制 `www` 到 `assets/public`，不跑 sync。

7. **Playwright 断言错对象**
   群聊最后一条不一定是被点名的人；应断言“用户消息之后的第一条回复是谁”。

8. **构建成功 ≠ APK 对**
   解包查新增模块、插件、native bridge；跑 `apksigner` / `zipalign`；没真机就明说没做启动验证。

## 适用范围与边界

- 主要面向 Web / PWA / 移动端封装项目，不覆盖纯后端、算法、数据任务。
- 清单里的命令以 Windows + PowerShell 为主；Linux/macOS 只影响第 7 节的工具链细节。
- 不提供“万能架构”；它只帮你少踩重复的坑，具体取舍仍要看项目约束。

## 来源

整理自一个移动优先 PWA「袖珍酒馆 PocketTavern」的开发与 Android 封装过程：角色卡、世界书、创意工坊、试穿沙盒、群聊、IndexedDB 迁移、Playwright 回归、Capacitor APK 构建。

## 许可

暂未附 License。如果你要把它作为公开模板分发，建议补一个 MIT 或 CC BY 4.0；需要的话可以在这个仓库里加。
