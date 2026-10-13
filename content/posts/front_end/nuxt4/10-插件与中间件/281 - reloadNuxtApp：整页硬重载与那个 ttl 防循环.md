---
url: /posts/783a614039cad213ab7e07550bbe45f0/
title: 281 - reloadNuxtApp：整页硬重载与那个 ttl 防循环
date: 2027-01-13T11:00:00+08:00
updated: 2027-01-13T11:00:00+08:00
author: cmdragon

summary:
  摘要：reloadNuxtApp 做的是整页硬重载——重新向服务端要这一页和它的依赖，不是路由级的软跳转。官方最看重的一点是它自带"防重载循环"：默认 ttl 是 10000 毫秒，10 秒内重复调用不会真的再重载；force 可以绕过这层保护。另外它默认会把当前 useState 状态存进 sessionStorage（persistState），但要真正把状态找回来，得配合 263 那个 experimental.restoreState 开关。这一篇把四个选项（path/ttl/force/persistState）的默认值和行为逐条落地。

categories:
  - 前端开发

tags:
  - nuxt4
  - reloadNuxtApp
  - 重载
  - sessionStorage
  - restoreState
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

什么时候需要"把整个应用重新来一遍"？典型是检测到线上新版本（275 里那个 `app:manifest:update` 钩子），或者用户操作要求彻底重置运行时。软路由跳转做不到"重新拉依赖、重建应用"，那就要 `reloadNuxtApp`。

## 它是硬重载

官方定位句和 note：

> reloadNuxtApp will perform a hard reload of the page.
> `reloadNuxtApp` will perform a hard reload of your app, re-requesting a page and its dependencies from the server.

"hard reload" + "re-requesting a page and its dependencies from the server" 就是重点：它不是 Vue Router 那种不换页面的前进后退，而是**重新向服务器要这个 HTML 和它依赖的 JS**。整棵应用树重建、运行时状态清零——所以它才有"重载循环"的风险（下面 ttl 讲）。

它还有一句关于状态的话：

> By default, it will also save the current `state` of your app (that is, any state you could access with `useState`).

注意用词是 "save"——默认它会把当前 `useState` 的状态存起来。但存了不等于自动恢复：能不能找回来，取决于你有没有开 `experimental.restoreState`。官方紧跟一句 read-more：

> You can enable experimental restoration of this state by enabling the `experimental.restoreState` option in your `nuxt.config` file.

这条正好接上 263：`restoreState` 是把状态存进 sessionStorage、页面重载后恢复的那个开关。`reloadNuxtApp` 是"触发一次这样的重载"的动作；两者配起来，才是"重载了但状态不丢"。

## 四个选项逐条

函数签名 `reloadNuxtApp(options?: ReloadNuxtAppOptions)`，`options` 官方给了四个键，连默认值一起：

- **`path`**（`string`，默认 `window.location.pathname`）

> The path to reload (defaulting to the current path). If this is different from the current window location it will trigger a navigation and add an entry in the browser history.

默认重载当前路径。给一个不同的 `path`，等于"重载到那个地址"，而且因为位置变了会触发一次导航、往浏览器历史里加一条记录。

- **`ttl`**（`number`，默认 `10000`）

> The number of milliseconds in which to ignore future reload requests. If called again within this time period, `reloadNuxtApp` will not reload your app to avoid reload loops.

这是防重载循环的核心。默认 10000 毫秒（10 秒）：在这段窗口里再次调 `reloadNuxtApp`，第二次**不会**真的重载。意义很直白——如果某个逻辑"一进来就触发重载"，重载完又进来又触发，没这层保护就会无限刷页面。10 秒窗口把它截断成"至多刷一次"。

- **`force`**（`boolean`，默认 `false`）

> This option allows bypassing reload loop protection entirely, forcing a reload even if one has occurred within the previously specified TTL.

`force: true` 直接**绕过**上面那层保护，哪怕刚在 ttl 窗口内重载过，也照样强制再重载。它是个明确给你"我就是要无视防循环"的逃生口——用之前你得确定真不会滚成循环。

- **`persistState`**（`boolean`，默认 `false`）

> Whether to dump the current Nuxt state to sessionStorage (as `nuxt:reload:state`). By default this will have no effect on reload unless `experimental.restoreState` is also set, or unless you handle restoring the state yourself.

存到哪：sessionStorage，键名 `nuxt:reload:state`。最关键那句是 **"By default this will have no effect on reload unless `experimental.restoreState` is also set"**——单开 `persistState: true` 只是把状态 dump 进 sessionStorage，重载本身并不会读它回来；要么同时开 `experimental.restoreState` 让框架帮你恢复，要么你自己写代码从那个 sessionStorage 键里读回来。

## 拼起来看：什么时候用哪几个

| 目标 | 怎么配 |
| --- | --- |
| 重载当前页、且不担心状态 | 直接 `reloadNuxtApp()`（默认防循环在，安全） |
| 重载到别的地址 | `reloadNuxtApp({ path: '/somewhere' })` |
| 确定要连续重载、不需要防循环保护 | 加 `force: true` |
| 重载后想把 useState 找回 | `persistState: true` **且** `experimental.restoreState: true`（或自己从 `nuxt:reload:state` 读） |

`persistState` + `restoreState` 这一对，本质是把 263 讲的机制接到了"主动重载"场景：263 说 `restoreState` 在 chunk error 或"manual `reloadNuxtApp()` call"之后从 sessionStorage 恢复——`reloadNuxtApp` 就是这个"手动调用"的入口，`persistState` 负责在重载前把状态塞进 sessionStorage。

## 自查

- `reloadNuxtApp` 是硬重载（重新向服务端要页面和依赖），不是软路由跳转——运行时状态会重建清零。
- 默认 ttl 10000 毫秒防重载循环；10 秒内重复调用不生效。要用 `force` 才绕过，且先确认不会滚成循环。
- 想让 useState 状态活过重载：`persistState: true` 只是存进 sessionStorage，**还得**开 `experimental.restoreState`（或自己读 `nuxt:reload:state`）才会恢复。
- 传不同的 `path` 会触发一次导航并加历史记录，不只是"原地重载"。

## 下一篇

下一篇 282 讲 `onNuxtReady`：只在客户端、应用初始化完成后才跑的回调，适合塞那些"不该挡住首屏渲染"的活（比如动态 import 一个大的分析库）。

## 本文依据

- reloadNuxtApp（`api/utils/reload-nuxt-app`）：定位句 "reloadNuxtApp will perform a hard reload of the page."；note "hard reload ... re-requesting a page and its dependencies from the server."；"By default, it will also save the current `state` of your app (that is, any state you could access with `useState`)." 与 read-more 关于 `experimental.restoreState`；`ReloadNuxtAppOptions` 签名与四选项（`path` 默认 `window.location.pathname` 及其导航/加历史说明、`ttl` 默认 `10000` 防重载循环、`force` 默认 `false` 绕过保护、`persistState` 默认 `false`、"dump the current Nuxt state to sessionStorage (as `nuxt:reload:state`)"、"no effect on reload unless `experimental.restoreState` is also set, or unless you handle restoring the state yourself"）。
- restoreState（`guide/going-further/experimental-features`，263 已引）："restored from sessionStorage when reloading the page after a chunk error or manual `reloadNuxtApp()` call."
