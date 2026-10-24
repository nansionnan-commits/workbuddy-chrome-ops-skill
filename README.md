# workbuddy-chrome-ops

> 直接接管**你已登录的浏览器标签页**来操作，不用另开浏览器、不用重新扫码登录。

## 解决的问题

自动化浏览器时最烦的不是"点哪个按钮"，是**登录**。

- Playwright 开一个实例 → 干净环境、没有登录态 → ERP/后台要重新扫码
- 钉钉/企微这种扫码登录，每次都变二维码、每次都要手机确认
- 折腾半小时，正事还没开始

这个 skill 换一条路：**用 WorkBuddy 的 Chrome 扩展通道，接管用户自己开着的那个标签页**。登录态是现成的，30 秒进入正题。

## 核心流程

```
browsers_list          找到 type=extension 的那个（用户的 Chrome）
user_open_tabs         看用户开着什么，找目标页
tabs_claim             接管
dom_cua_get_visible_dom 拿 node_id 快照
dom_cua_click / type   按 node_id 操作
tabs_finalize          收尾，把标签还给用户
```

## 关键认知

| 认知 | 说明 |
|---|---|
| **node_id 是快照级的** | 每步动作后页面一变，旧 node_id 全废，必须重新取快照 |
| **坐标点击在复杂前端不准** | Vue/React 后台一律用 `dom_cua_*`，别用 x/y |
| **`user_open_tabs` ≠ `tabs_list`** | 前者是"能接管的"，后者是"已接管的" |
| **别关用户的标签** | 用 `tabs_finalize(keep=...)` 还回去 |

## 何时不该用

- 需要并发抓大量页面 → Playwright
- 需要无人值守定时跑 → Playwright + storage_state
- 页面要登录但用户没登录 → 先让用户登录一次，再接管

## 实测案例

在兰谷 ERP 后台按 SKU 查产品：接管已登录标签 → 输入 SKU → 查询 → 点预览 → 读全部字段，全程 2 分钟，零登录。

## 安装

```bash
# 用户级（推荐）
git clone <this-repo> ~/.workbuddy/skills/workbuddy-chrome-ops
```

或在 WorkBuddy 里直接把这个目录放到 `~/.workbuddy/skills/` 下。

## 许可

内部使用。
