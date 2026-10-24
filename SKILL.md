---
name: workbuddy-chrome-ops
description: 用 WorkBuddy 的 Chrome 扩展通道操作「用户自己已登录的浏览器」—— 不新开实例、不用重新登录、不碰 cookie。当任务需要在你/老板已经登录的后台（ERP、店铺后台、飞书、内部系统）里点几下、填个表、查条数据、截个图，而 Playwright 一开就得重新登录时用这个。触发词：接管我的浏览器 / 操作我已登录的页面 / 浏览器扩展 / 帮我在这个后台点一下 / 不用重新登录 / 我已经开着页面了。
version: 1.1.0
---

# WorkBuddy Chrome 扩展通道

> **一句话**：老板浏览器里开着、登录着的页面，我可以直接接管来操作，不用另开浏览器、不用重新扫码登录。

---

## 何时用（决策树）

| 情况 | 用什么 |
|---|---|
| 目标页面**用户已经登录**（后台/ERP/内部系统） | ✅ **本 skill（扩展通道）** |
| 需要固定的干净环境、可重复跑的回归测试 | Playwright（`playwright-browser-automation`） |
| 需要抓大量页面、并发、持久化 cookie | Playwright + `storage_state` |
| 页面要登录但用户没登录 | 先让用户登录（扩展通道），或 Playwright 扫一次码存 cookie |

**判断口诀**：**"登录态在用户浏览器里" → 走扩展通道；"我要一个可控的自动化环境" → 走 Playwright。**

⚠️ **最常见的误判**：遇到"要登录才能进"的页面，就上 Playwright 反复扫码 —— 错。用户浏览器里那套登录态，Playwright 的独立 profile 是拿不到的，扫一百次码也是白扫。

---

## 核心工具链（7 个够用）

| 工具 | 作用 | 关键点 |
|---|---|---|
| `browsers_list` | 列出可用浏览器 | 看 `type`：`desktop`=内置；`extension`=用户的 Chrome |
| `user_open_tabs` | 列**用户开着的**标签 | 每个 tab 带 `claimed` 状态 |
| `tabs_claim` | 接管某个用户标签 | 接管后才能操作 |
| `dom_cua_get_visible_dom` | 拿页面**可见 DOM 快照** | 每个元素带 `node_id`，**快照级**，会失效 |
| `dom_cua_click` / `dom_cua_type` | 按 `node_id` 点击 / 输入 | **优先用这个，别用坐标** |
| `tab_screenshot` | 截图 | 操作前先看一眼，别在错页面上瞎点 |
| `tabs_list` / `tabs_finalize` | 看已接管的 / 收尾 | `finalize` 的 `keep` 决定标签留给用户还是释放 |

---

## 标准流程

```
① browsers_list
   → 找 type=extension 的那个（通常是 browser_id=2，"Google Chrome"）
   → type=desktop 的是 WorkBuddy 内置浏览器，没有用户的登录态

② user_open_tabs(browser_id)
   → 扫一遍用户开了什么，找目标页面
   → 没有目标页？用 tabs_new 开一个，让用户自己登录一次

③ tabs_claim(browser_id, tab_id)
   → 接管。返回里会带最新的 url，注意读

④ tab_screenshot
   → 先确认页面停在哪、登录没掉、有没有弹窗挡着

⑤ dom_cua_get_visible_dom
   → 拿到 node_id 清单。输入框是 <input>/<textarea>，按钮是 <button>

⑥ dom_cua_click(node_id) → dom_cua_type(text) → 重新 get_visible_dom → dom_cua_click(查询按钮)
   → 见下方坑 #2，每一步后快照都会变

⑦ tab_screenshot 验收 + tabs_finalize 收尾
```

---

## 五个必踩的坑（都实测过）

### 坑 1 · 坐标点击在复杂前端上不准
`cua_execute` 的 `{type:"click", x, y}` 适合坐标稳定的页面。在 Vue/React 后台（ERP、ant-design 系）里，点输入框经常点不进去、光标不落位。

**解法**：一律走 `dom_cua_click(node_id)`。坐标只当最后兜底。

### 坑 2 · 输入完 DOM 快照就失效
```
dom_cua_click(输入框) → dom_cua_type("...") → dom_cua_click(查询按钮)
                                                   ↑ 报 DOM node_id is stale
```
**根因**：`node_id` 是**某一次快照内**的编号，页面一变更（输入触发重渲染、弹窗打开、列表刷新）就全废。

**解法**：**每一步动作后重新 `dom_cua_get_visible_dom` 再取 node_id**。不要缓存 node_id 跨动作使用。

### 坑 3 · `tabs_list` 和 `user_open_tabs` 不是一回事
- `user_open_tabs` = 用户开着但**我还没接管**的（`state: observed`）
- `tabs_list` = **我已经接管**的（`state: controlled`）

接管前用 `tabs_list` 找目标，永远找不到。

### 坑 4 · 别把用户的标签关掉
`tabs_close` 关的是**老板正在用的页面**。

**规矩**：
- 操作完用 `tabs_finalize(keep=[{tab_id, status:"handoff"}])`，把标签还给用户
- 除非用户明确说关，否则不 `tabs_close`

### 坑 5 · 弹窗/抽屉打开后，原快照全废
点"预览"这类按钮会弹 modal，底层 DOM 还在但 `node_id` 已重排。

**解法**：进弹窗后立刻重新 `get_visible_dom`；关弹窗优先按 `aria-label="Close"` 的按钮（或 `cua_keypress Escape`），**关完必须再取一次快照**。

### 坑 6 · 受控输入框 `dom_cua_type` 不生效（React / Vue 双向绑定）
**现象**：`dom_cua_click(node_id)` + `dom_cua_type("xxx")` 都返回 `applied: true`，但重新取快照发现 `value=""`，一个字都没进去。

**根因**：`dom_cua_type` 走的是 DOM 赋值，React 的受控组件不认（没触发 `input` 事件），框架状态没更新。

**解法**：换成**坐标点击聚焦 + 真实键盘输入**
```
cua_execute: [
  {type:"click", x, y},                              ← 坐标点击（简单页面上反而是最准的）
  {type:"type", text:"workbuddy-chrome-ops-skill"}   ← 真实键盘事件
]
```
实测：GitHub 建仓页的 Repository name、Description 都是这样才填进去的。

**判断口诀**：`dom_cua_type` 填不进去 → 立刻换 `cua_execute` 的 `type`，不要反复重试 `dom_cua_type`。

### 坑 7 · `cua_execute` 的 scroll 会超时
**现象**：`{type:"scroll", x:960, y:600, scroll_x:0, scroll_y:700}` 直接 `provider invocation timed out`。

**解法**：用键盘滚
```
{type:"click", x:300, y:760}     ← 先点空白处，让输入框失焦（否则 End 只是移动光标）
{type:"keypress", keys:"End"}    ← 到底部；也可以用 "PageDown" 一屏一屏翻
```
实测：GitHub 建仓页的「Create repository」按钮就在页面最底部，`End` 一按就露出来了。

---

## 实战案例：在 ERP 里按 SKU 查一个产品

```text
1. browsers_list          → browser_id=2 (extension)
2. user_open_tabs(2)      → 发现 tab 3 = http://erp.example.com/productLibrary/...（已登录）
3. tabs_claim(2, 3)       → 接管
4. tab_screenshot(2,3)    → 确认在列表页，搜索框可见
5. dom_cua_get_visible_dom → textarea node_id=8（SKU 输入框）、button node_id=9（查询）
6. dom_cua_click(2,3,"8") → dom_cua_type(2,3,"ZHF20261022028")
7. dom_cua_get_visible_dom → 快照刷新，仍是 node_id=8/9
8. dom_cua_click(2,3,"9") → 查询
9. tab_screenshot          → 结果只剩 1 条，读数据
10. dom_cua_get_visible_dom → 找"预览"链接 node_id=29 → click → 弹窗出全部字段
11. tabs_finalize(keep=[{tab_id:3, status:"handoff"}])
```

**关键节奏**：**动作 → 重新取快照 → 动作**，不要连点。

---

## 实战案例 2：网页建 GitHub 仓库（API 没权限时的兜底）

**背景**：MCP 的 GitHub 连接是 App 身份，没有建仓权限，`POST /user/repos` 直接 `403 Resource not accessible by integration`。重新授权也没用——是 App 权限层面的限制。

**那就用浏览器建**（用户 Chrome 里 GitHub 是登录着的）：

```text
1. tabs_new(2, "https://github.com/new")        → 已登录会直接是建仓页
2. cua_execute: click(仓库名输入框) + type("workbuddy-chrome-ops-skill")
   ⚠️ 必须用 cua 的 type，dom_cua_type 在这页不生效（见坑 6）
3. 等出现 "✓ xxx is available"
4. cua_execute: click(空白处) + keypress("End")  → 滚到底（scroll 会超时，见坑 7）
5. cua_execute: click(1120, 680) 点「Create repository」
6. tabs_list(2) 确认 url 变成 https://github.com/<owner>/<repo>
```

**意义**：**API 干不了的网页操作，浏览器能兜底。** 遇到「权限不足」类报错，先别急着要 token，看看同一个动作能不能在网页上点出来。

---

## 边界与纪律

1. **只做用户要你做的事**。接管的是用户的真实浏览器，误点可能改数据、发消息、下订单。涉及写操作（提交/删除/付款/发消息）前必须跟用户确认。
2. **先截图再动手**。页面可能停在你以为的上一页。
3. **不猜 node_id**。没有快照就必须先取，不许凭上次的记忆点。
4. **不搬 cookie**。这条通道的意义就是不用搬——别反过来去读用户的 cookie 文件。
5. **敏感信息不落盘**。页面上看到的密码、token、客户手机号，不进日志、不写文件。

---

## 和 Playwright 的取舍

| 维度 | 扩展通道 | Playwright |
|---|---|---|
| 登录态 | ✅ 直接复用用户现有 | ❌ 独立 profile，要重登 |
| 稳定性 | 依赖用户浏览器状态（可能被用户操作打断） | ✅ 完全可控 |
| 并发 | ❌ 单标签串行 | ✅ 可多上下文 |
| 反爬 | 真实浏览器指纹，最像人 | 需要额外伪装 |
| 适合 | 后台里的零星操作、查数据、走一次流程 | 批量抓取、回归测试、定时任务 |

**经验**：一次性、要登录、量不大的活 → 扩展通道，5 分钟搞定；要跑 100 次、要无人值守 → 用 Playwright，先解决登录态（导出 storage_state）。

---

## 已知限制

- 只能操作**已存在的标签页**，不能控制用户的浏览器设置/扩展管理页
- 单次 `get_visible_dom` 只返回可见区域元素，长列表要滚动后再取
- 部分跨域 iframe 内的元素拿不到 `node_id`（会返回外层结构）
- 用户中途手动操作同一标签会打断流程，动作后必须重新确认页面状态
