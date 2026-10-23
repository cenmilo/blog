---
url: /posts/27879edd7172f903702b5f7118818ff4/
title: 790 - 两写法分工（2）：列表页为什么把取数交给 useAsyncData，而不是就地 useFetch
date: 2026-10-28T10:00:00+08:00
updated: 2026-10-28T10:00:00+08:00
author: cmdragon

summary:
  摘要：列表页那行注释写着，走 HTTP 是唯一一条两条路都通的口子。这一站把这句话拆开判：换成就地 useFetch 再跑一遍，两条路的结果一模一样；加一个 server:false 立刻塌；真去直接调服务端函数则构建就断。判完，注释按事实改了。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 取数与缓存
  - useAsyncData
  - SSR
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 注释里那句话，得能被打断

`app/pages/docs/index.vue` 里那行取数上面，原本写着：

```ts
// 为什么用 useAsyncData ＋ $fetch 而不是在服务端直接调 listDocs()：
// 这一页的正文以后要走"服务端渲一次、客户端导航再取一次"两条路，
// 而 $fetch('/api/docs') 是唯一一条两条路都通的口子（详情页同理）。
const { data: docs, status, error } = await useAsyncData('docs:list', () => $fetch<DocSummary[]>('/api/docs'))
```

上一站（789）只把现状量清、没有替这句话背书。这一站就是来判它的。判的办法不是讲道理，是**换写法再跑同一套读数**——三条对照，一条必须红，不然三条一样的绿什么都说明不了。

先说要判的东西是什么。"两条路"指：

- **路 A**：地址栏直接进 `/docs`，服务端渲一份 HTML 交给浏览器，浏览器接手水合
- **路 B**：从首页点导航进来，服务端那份 HTML 里没有这一页的内容，取数发生在客户端

新加的 `test/e2e/two-doors-two-ways.spec.ts` 把这两条路各自钉住，基线读数：

```
[两路·A 直接进] {"html有标题":true,"html里链接数":2,"页面发的取数":[]}
[两路·B 点进来] {"页面发的取数":["/api/docs"],"列表条数":2}
```

A 那一条钉两件事：列表在响应体里，而且浏览器接手后**不该**再打一次接口（数据在负载里）。B 钉的是：`/docs` 没有可用的载荷文件，那一次取数必须真发。

## 对照一：换成就地 `useFetch`，结果一模一样

把那一行整段换成 `await useFetch<DocSummary[]>('/api/docs')`（不写键、不包函数），跑两份端到端：

```
[两路·A 直接进] {"html有标题":true,"html里链接数":2,"页面发的取数":[]}
[两路·B 点进来] {"页面发的取数":["/api/docs"],"列表条数":2}
[docs ssr html] {"bytes":3129,"hrefs":["/docs/pricing-faq","/docs/getting-started"]}
```

`6 passed`。两条路的结论一位没变，唯一动过的是响应体字节数（3043 变 3129）。**那 86 个字节不是这一站的结论**——它是负载里那串自动生成的键名比手写的 `'docs:list'` 长带来的，形状随实现变，不该被任何人拿去当依据。

真正的事实是：**"两条路都走得通"这件事，`useFetch` 与 `useAsyncData` 谁都行**。注释里那句"唯一一条口子"，说的不是这一对选择。

## 对照二：一定得红的那一档

如果两档绿就收工，那可能只是判据没在看着东西。所以再加一档注定要变的：给同一行加 `{ server: false }`。

```
[docs ssr html] {"bytes":2612,"hrefs":[]}
[两路·A 直接进] {"html有标题":false,"html里链接数":0,"页面发的取数":[]}
  2 failed
  4 passed (20.0s)
```

响应体掉到 2612 字节、一条链接都没有，两条判据当场红。⇒ **改得出"两条路"的是服务端取不取，不是用哪条口子**。对照一那三行绿到这里才变成证据。

顺带这一档暴露了判据自己的一个假绿。修之前，路 A 那行 `页面发的取数` 读到的是空数组，看起来正好支持"客户端不该再打接口"；可 `{ server: false }` 那档里浏览器明明会在水合后补一次取数，读数却还是空——因为**监听器收得早、读得也快**，那一次请求发生在读之后。于是把路 A 改成"等应用接手完、再等 300ms 才读"，同一位现在才是可信的：基线读到空，`server:false` 那一档读到非空。判据被变异教了一次，这个细节写在代码注释里。

## 对照三：真去直接调服务端那个函数

注释里那句"而不是在服务端直接调 `listDocs()`"到底成不成立？照它写的做一遍：从页面里 `import { listDocs } from '../../../server/services/content'`，把取数函数换成它，跑构建。

```
 WARN  [plugin rolldown:vite-resolve] Module "node:path" has been externalized for browser compatibility, imported by ".../server/services/content/index.ts".
 WARN  [plugin rolldown:vite-resolve] Module "node:fs/promises" has been externalized for browser compatibility, ...
[nitro]  ERROR  RollupError: Could not resolve "../../../../../shared/schemas/content.ts" from ".../.nuxt/dist/server/_nuxt/docs-kmee4HJO.js"
```

构建退出码 1。前端那份包试图把读文件的模块吞进去，服务端那一份则断在解析上。**这一位是真的过不去**：客户端要用不上的能力（读磁盘）没法在浏览器侧执行，所以初始数据要么服务端渲进 HTML、要么走一次 HTTP 口子中转到浏览器，两条路才都通。这句留着，措辞改掉：它对立的是"直接调服务端函数"，不是"`useFetch` 还是 `useAsyncData`"。

改完之后，那一行代码没动，注释换成按三档对照写出来的版本，并把判据文件名留在旁边。

## 那为什么这里还是 `useAsyncData`

不是为了两条路，是为了**把键写死**。讲 data fetching 那一节的原话是：自动生成的键只认 `useAsyncData` 被调用的位置，所以**建议永远自己写键**，尤其"当你在做一个包住 `useAsyncData` 的自定义 composable 时"。内容区这两处的做法是把键放在第一位（`'docs:list'`、`` `docs:${slug}` ``），会员区那两处则是把键锁进唯一那个出口。同一条规矩，两种写法——区别只在于**这份数有几个调用点**：一个就直接写键，多个就烤进出口。

上一站那条 `[两口子·不写键] {"跑了几次":2,…}` 就是这一步的理由；今天内容区只有一个调用点，写死键是够用的，写成出口反而是多余的一层。

## 撞上了去哪里看

| 撞上的现象 | 大概率是哪一位 | 先看哪里 |
| --- | --- | --- |
| 把某页改成 `server: false`，爬虫看不到内容 | 服务端不再取那一份 | `[docs ssr html]` 的字节数与 `hrefs` |
| 页面 HTML 里没数据但列表还是出来了 | 数据是客户端补的（水合后） | 路 A 那条：响应体与浏览器补取分开钉 |
| 换写法后"结果没变"就以为写法不重要 | 缺一档一定红的对照 | 本文对照二那一档 |
| 页面里 import 了 `server/` 下的模块 | 读磁盘的能力进不了浏览器包 | `npm run build` 那两条错误 |
| 客户端导航那一次读数看着不对 | 监听器读得太早 | 等应用接手完再读（路 A 的修法） |

## 交付前自查

- 那句注释被拆成三档各跑一遍：换口子（不变）、`server:false`（必红）、直接调服务端函数（构建断），只有第二条能红就不足以说明第一条的绿
- 一处判据自身的假绿由对照二暴露，修完复跑基线仍绿（`2 passed (20.3s)`）
- 响应体那 86 字节的差标成"形状随实现变、不当依据"，没写成结论
- 代码那行没改，改的是注释；`npm run check` 八项在还原后复跑全 ok
- 805ms 与 261ms 那一对时间只报读数不归因（上一轮同形教训：单趟采秒不比耗时）

## 本文依据

- 官方：讲 data fetching 那一节（"永远自己写键"那句建议、`useFetch(url)` 与 `useAsyncData(url, () => event.$fetch(url))` 的等价式、裸 `$fetch` 没有去重与负载转发）；讲 `useAsyncData` 与 `useFetch` 的那两节（服务端等 promise 落定再渲；`server:false` 与水合那段）；讲通用渲染的那一节（服务端渲一次、客户端接手）
- 仓库：`app/pages/docs/index.vue`（那一行未改，注释按三档对照重写）、`test/e2e/two-doors-two-ways.spec.ts`（新增，2 条；路 A 含"等水合再读"的修法）、`test/e2e/content-docs.spec.ts`、`server/api/docs.get.ts` 与 `server/services/content`（被对照三试图直接调过去的那一层）
- 上一站（两条口子的键位差异、四条运行层读数）：[789 - 两写法分工（1）：同一份数有两条口子，会员区选的是哪一条](/posts/15c99d7476b7ba439553ce315ce275f1/)
- 下一篇：[791 - 两写法分工（3）：类型从哪来，本站接口认地址、第三方接口要自己写](/posts/49efdf2b2e881d15daef241b8cc90152/)
