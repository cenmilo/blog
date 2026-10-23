---
url: /posts/f69ef581eeb0f28ca7e2cdc64cf990ac/
title: 217 - 不服务端取数那一档：server 关掉之后 status 与 data 会怎样
date: 2026-12-23T10:00:00+08:00
updated: 2026-12-23T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`server` 这一档我一开始理解成"要不要在服务端渲染这块内容"，其实它管的只有一件事，官方两节的措辞都是"要不要在服务器侧调用/发起这次取数"：`useAsyncData` 写 "Whether to call the function on the server."，`useFetch` 写 "Whether to fetch on the server."，**默认都是 `true`**。把它设成 `false` 之后，文档给的症状描述非常具体，两节各有一模一样的 note：「If you have not fetched data on the server (for example, with `server: false`), then the data *will not* be fetched until hydration completes. This means **even if you await** `useAsyncData` on the client side, `data` will remain `undefined` within `<script setup>`.」——**await 也救不回来**，因为水合完成之前那次请求根本没开始。另一个可见后果写在 `status` 的定义里：`idle` 是 "Function has not been called yet (e.g. `{ immediate: false }` or `{ server: false }` on server render)"，也就是服务端那一趟读到的是 `idle`。这一篇把 `server`、`lazy`、`immediate` 三个布尔放在一起对齐：哪些组合文档写明了、哪些没写；顺便把它和 `serialize: false` 的区别分开——**`server: false` 是"服务端不取"，`serialize: false` 是"服务端取了但不塞进 payload"**，两者都会让客户端多发一次请求，但原因完全不同。

categories:
  - 前端开发

tags:
  - nuxt4
  - server
  - lazy
  - idle
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先把这一档能解决的问题说清：有些数据只有浏览器里才拿得到（本地设备信息、只在内网可达的接口），或者首屏不想要它、宁可空着也不要服务端等一次。这时候动的就是 `server`。

## 官方只说了"调用在哪一侧"

两页对 `server` 的定义各一句，都不长：

| 出处 | 原文 | 默认 |
| --- | --- | --- |
| `useAsyncData` 选项表 | "Whether to call the function on the server." | `true` |
| `useFetch` 选项表 | "Whether to fetch on the server." | `true` |

注意说的是**要不要调用 handler / 要不要发起请求**，不是"要不要在服务端渲染这段 UI"。这块内容渲不渲染出来取决于你有没有在模板里用它；`server` 只决定服务端那一趟跑不跑。

## 关掉之后的两个可见后果

后果一写在 note 里（`useAsyncData` 与 `useFetch` 两页各一份，措辞几乎相同）：

> If you have not fetched data on the server (for example, with `server: false`), then the data *will not* be fetched until hydration completes. This means even if you await `useAsyncData` on the client side, `data` will remain `undefined` within `<script setup>`.

我把它拆成三段因果：

1. 服务端没取 ⇒ 数据不在 payload 里带过来（210 讲的正是那条通道）。
2. 客户端这一次要等到**水合完成**才发起。
3. 所以 `await` 无效：等待的目标（那次请求）在你的代码执行时还不存在。

**症状就是"await 了但 `data` 还是 undefined"**。这是最容易被误报成"框架 bug"的一类写法问题。

后果二写在 `status` 的取值定义里：

- `idle`: Function has not been called yet (e.g. `{ immediate: false }` or `{ server: false }` on server render)

原话特意加了 "on server render"，说明 `server: false` 时**服务端那一趟的 status 是 `idle`**。这条很实用，因为 `idle` 和 `pending` 在界面上该给的东西不一样：`pending` 是"正在加载"，`idle` 是"还没开始"。

顺带一条能配上去的：`pending` 那个 ref 在开了 `experimental.pendingWhenIdle` 之后，**`status` 为 `idle` 且没有缓存数据时也会是 `true`**（214 抄过原句）。所以想让 `server: false` 的区块从首帧就显示骨架，这两条要一起看。

## server、lazy、immediate 三个布尔对齐

把三条定义并排放（全部来自选项表原话）：

| 选项 | 官方说明 | 默认 |
| --- | --- | --- |
| `server` | "Whether to call the function on the server." | `true` |
| `lazy` | "If true, resolves after route loads (does not block navigation)." | `false` |
| `immediate` | "If false, prevents function from being called immediately." | `true` |

它们管的是三个互不相干的维度：**在哪一侧跑** / **要不要挡住导航** / **要不要立刻开跑**。文档给这三者的关系只写明了两处：

1. `lazy` 的实现方式：「Under the hood, `lazy: false` uses `<Suspense>` to block the loading of the route before the data has been fetched. Consider using `lazy: true` and implementing a loading state instead for a snappier user experience.」
2. `idle` 的两种成因里同时点了 `immediate: false` 和 `server: false`（上面那句）。

**至于 `server: false` 加 `lazy: true`、或 `server: false` 加 `immediate: false` 各自会得到什么，文档没有写**。我以前的做法是把三种组合都试一遍再下结论，这一篇不替你猜：要用的时候按下面那张实测表跑一次，把自己的结果记在项目里。

## 与 serialize: false 的区别（最容易混）

两者都会让客户端多发一次请求，但方向不同：

| | `server: false` | `serialize: false` |
| --- | --- | --- |
| 服务端是否执行 handler | 否 | **是** |
| 结果是否进 payload（`__NUXT_DATA__`） | 无结果可进 | **不进**（"server-fetched data is kept out of the payload"） |
| 客户端为什么还要取 | 因为压根没取过 | 因为带不过来，"the client will refetch after hydration if a component renders it" |
| 官方附带建议 | 未写 | "Pair with lazy hydration to avoid hydration mismatches and unnecessary client fetches." |

`serialize` 那条说明整段我抄在 213，这里只用它的因果。这个区别我在两种场景下都排过错：首屏有内容但水合后又发一次请求 ⇒ 十有八九是 `serialize`；首屏就没内容 ⇒ 才轮到 `server`。

## 想确认自己项目里到底哪一侧跑了

最便宜的一次验证，两步：

1. **看服务端 HTML**：`curl` 目标路由，搜你要的那块文本。搜不到 ⇒ 服务端没取或取到没渲染（对应 `server: false` 或渲染条件）。
2. **看请求落点**：浏览器开发者工具的网络面板里，同一个接口只出现一次且发生在文档请求之后 ⇒ 服务端取过、payload 带过来了；出现两次或只在导航时出现 ⇒ 更可能是 `server: false` 或 `serialize: false`。

顺手一个更直接的判据：在 handler 第一行打 `console.log(import.meta.server)`（服务端日志进终端、客户端进浏览器控制台），一眼看出这次执行发生在哪一侧——这比读代码猜"到底有没有跑"靠谱。

## 自查

1. `server: false` 时服务端那一趟的 `status` 是什么？依据句？（答：`idle`；`idle` 的定义里点名 "or `{ server: false }` on server render"。）
2. 为什么 `await` 也拿不到数据？（答：那次客户端请求要到水合完成才发起，`await` 等的东西当时还不存在。）
3. `server: false` 与 `serialize: false` 的差别在哪一句上？（答：前者是服务端不执行 handler；后者服务端取了但不写进 payload，水合后由客户端重取。）

## 下一篇

218 - 六个行为选项细讲：default、pick、transform、dedupe、enabled、serialize
