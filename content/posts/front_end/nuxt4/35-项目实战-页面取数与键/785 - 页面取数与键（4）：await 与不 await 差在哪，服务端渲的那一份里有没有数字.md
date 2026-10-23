---
url: /posts/000b3d5038f2dd9777f568385b5bd0af/
title: 785 - 页面取数与键（4）：await 与不 await 差在哪，服务端渲的那一份里有没有数字
date: 2026-10-27T12:00:00+08:00
updated: 2026-10-27T12:00:00+08:00
author: cmdragon

summary:
  摘要：列表页那一行 await 到底要不要写，常见说法有两种：摘掉它页面更快，或者改用 lazy 别阻塞渲染。这一站把两半分开量：在生产产物里读不带 JS 的响应体，在运行时层读 setup 那一刻能不能取到值。结果是 HTML 里该有的一样都不少，变的只有当下读不读得到——而决定 HTML 的其实是另一件事。

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

## 那一行有两个人人都想改的地方

`app/pages/docs/index.vue` 里取列表就一句：

```ts
const { data: docs, status, error } = await useAsyncData('docs:list', () => $fetch<DocSummary[]>('/api/docs'))
```

`await` 摆在那里，就有两种声音冒出来：把它摘掉页面更快；或者改成 `lazy`，别让取数阻塞渲染。

讲 `useAsyncData` 与 `useFetch` 的那两节里都有同一段话，把这两句都挡回去了：

> You do not need to `await` `useFetch`. On the server, Nuxt waits for the promise to resolve before rendering in either case, so the returned HTML always contains the data.

> The `await` affects what happens after the call: with it, execution pauses until `data` is populated, and client-side navigation is blocked until the data is ready; without it, execution continues immediately, `data` starts as its default value until the request resolves

第二句还补了一句：不 await 的效果与 `lazy` 类似，只是 `lazy` 是**显式**选择不阻塞导航的那一位。

于是问题被拆成两处：服务端渲出来的 HTML 里有没有数据；以及 `setup` 那一刻读不读得到值。这两处要用不同的层去量。

## 服务端那一半：四种写法读同一份响应体

端到端层跑的是 `npm run build` 的产物，而且用 `request.get('/docs')` 直接读响应体——**一行 JS 都没跑**，读到的就是服务端渲出来的那份 HTML。同一判据下面这四种页面写法各跑一趟：

| 页面里那一行怎么写 | 响应体读数 | 那条判据 |
| --- | --- | --- |
| `await useAsyncData(...)` | `{"bytes":3043,"hrefs":["/docs/pricing-faq","/docs/getting-started"]}` | 绿 |
| 去掉 `await` | `{"bytes":3043,"hrefs":["/docs/pricing-faq","/docs/getting-started"]}` | 绿 |
| 加 `{ lazy: true }` | `{"bytes":3043,"hrefs":["/docs/pricing-faq","/docs/getting-started"]}` | 绿 |
| 加 `{ server: false }` | `{"bytes":2531,"hrefs":[]}` | **红** |

三行相同本身就是结论：**摘 `await` 和换 `lazy` 都不会让 HTML 变薄，也不会把数据抹掉**——服务端两种写法都等取数落定，`lazy` 也一样等。第四行才是分界：数据不进 HTML 的原因是"服务端根本没取"，不是"没 await"。这条判据第一次红在这里，我反而确认前面三行的绿是真的绿（改动确实进了产物）。

`lazy` 那一行值得多说一句。那一节的 `::tip` 建议是为"更快的体验"用 `lazy: true` 加一个加载态——那是**客户端导航**那一档的事；把它当成"服务端少等一点"的手段就误读了。

顺带记一条我自己的错读。第一轮跑基线时，"从首页点进文档区"那条客户端导航用例耗时 810ms，去掉 `await` 那趟是 261ms，看着正好印证"await 会挡住客户端导航"。回到原写法连跑两趟，同一条用例是 **281ms 与 254ms**——和去掉 await 那趟同一档。那个 810ms 是**那一趟里第一次导航**的代价（构建刚完成、路由块还没热过），不是 `await` 的代价。所以那句"执行会停、导航会等"是原话，可本机这几组单趟采秒量不出它，本文不把它写成结论。

## 客户端那一半：setup 那一刻读到什么

运行时层是裸挂载（没有服务端那一份、也没有水合），量的正是"之后"那一半：`setup` 里那一刻 `data` 是什么。六条判据的读数：

```
[await·等它] {"趟数":1,"setup那一刻":"1","渲染时":"1","status":"success"}
[await·不等] {"趟数":1,"setup那一刻":"(还没有)","渲染时":"1","status":"pending"}
[await·default 起步] {"趟数":1,"setup那一刻":"4242","渲染时":"1","status":"pending"}
[await·lazy 挡不住] {"趟数":1,"setup那一刻":"(还没有)","渲染时":"1","status":"idle"}
[await·server false 裸挂载] {"趟数":1,"setup那一刻":"1","渲染时":"1","status":"success"}
[await·then 可解构] {"then那位":"function","catch那位":"function","渲染时":"1"}
```

对得上原话的四条：等就有值（`"1"`）；不等就没有，等请求落定后渲染时才补上；不 await 时 `data` 的起步是 `default` 给的那一份（这里给 4242，读到的就是 4242）；`lazy: true` 即便写了 `await` 也照样读不到——它和"不 await"是同一档。

对不上的一条也得记：那一节还写着"如果你没在服务端取过数据（比如 `server: false`），要到水合完成才会取；哪怕你在客户端 await，`data` 在 `<script setup>` 里仍是 undefined"。这一层裸挂载读到的却是 `"1"`。原因是那句的前提是**整页先由服务端渲、再由客户端接手**：这里有服务端那一份可等，也有水合那一步；这一层两者都没有，客户端当场就取了。判据改成钉这一层的真实形状，那句文档留给有 SSR 的场景讲。

`status` 那一位在这一趟里露出了三种值（`success`／`pending`／`idle`），本站下一站专门数它。

还有一条不需要量的：那一节明写返回对象上的 `then`、`catch`、`finally` 可以安全解构，哪怕你没等它。判据解构出来一位是个函数——这条重要，因为"必须整个 await 才能用"的直觉会让人不敢把取数写进 `Promise.all`。

## 这一页该选哪种

把两半合起来看，选择很清楚：

- 想让爬虫与关掉 JS 的读者看见内容 ⇒ 服务端必须取（`server: false` 是唯一会让 HTML 变薄的那一位，别的都动不了它）
- 想在 `<script setup>` 里直接读 `data`（比如拼 SEO 标签、或者取数失败要先 `throw createError`）⇒ 得 `await`。`app/pages/docs/index.vue` 正是这种情况：服务端取数失败要赶在发 SEO 标签之前抛 503，那一刻必须读得到 `error`
- 只是想让客户端导航早点落地、自己画加载态 ⇒ 用 `lazy: true`，代价是那一刻读不到值

这一站**页面代码一行没改**，四种写法都是临时改上去、量完还原的（还原后基线复跑仍是 3043 字节、两篇都在）。判据只加在两个地方：`test/nuxt/auto-await.test.ts`（新增 6 条）与那条端到端判据旁边的注释（把"三种写法读数一致、只有 `server:false` 会掉"写下来，免得下一个人靠"摘 await"去做优化）。

## 撞上了去哪里看

| 撞上的现象 | 大概率是哪一位 | 先看哪里 |
| --- | --- | --- |
| 摘掉 `await` 想瘦身，HTML 字节数没变 | 服务端本来就等 | 端到端那条响应体判据（本文四行表） |
| 用 `lazy` 之后 HTML 还是带数据 | `lazy` 管的是导航，不是服务端等不等 | 同上 |
| 某页 HTML 里没有内容了 | 那一页 `server: false` 或没在服务端取 | 读响应体，别读 DOM |
| `setup` 里 `data` 是 undefined | 没 await，或 `lazy: true` | 给 `default`，或改回 await |
| 客户端导航"偶尔"慢一大截 | 第一次导航的冷启动 | 同一条用例连跑两趟再比 |

## 交付前自查

- 服务端那一半用**产物 + 不开 JS 的响应体**量，四档各跑一趟；`server:false` 那档特意留作"能红"的对照，证明前三档的绿不是没生效
- 一处采秒误读（810ms）被同一条用例连跑两趟否认了，正文按"量不出来"写，没写成结论
- 文档那句 `server:false` 的前提（有水合页）与本层的读数冲突，按本层真实形状改判据、把那句还给有 SSR 的场景
- 页面代码零改动：四种写法都是临时改上去量完还原的
- `status` 的三种值只报读数，留给下一站数它的取值集合

## 本文依据

- 官方：讲 `useAsyncData` 与 `useFetch` 的那两节（不需要 await、服务端两种写法都等 promise 落定、`await` 影响的是之后、`lazy` 是显式不阻塞导航、`default` 起步、`server:false` 与水合那段、`then`／`catch` 可安全解构、`::tip` 里"用 lazy 加加载态换更快体验"的建议）
- 仓库：`app/pages/docs/index.vue`（那一行，四种写法临时改过、已还原）、`test/nuxt/auto-await.test.ts`（新增，6 条）、`test/e2e/content-docs.spec.ts:33`（响应体判据，注释补了三档等价的读数）
- 端到端口径：这一层起的是 `npm run build` 的产物服务（`playwright.config.ts` 的 `webServer.command`），本机用 `E2E_PORT=3246` 避开临时端口区间
- 上一站（自动键的作用域与"读键要借 `getCachedData` 第一位"）：[784 - 页面取数与键（3）：自动生成的键只对调用点唯一，这是官方那句的原话](/posts/b108b99c151761c9d212d183da9724b6/)
- 下一篇：[786 - 页面取数与键（5）：状态那一栏有四种取值，页面上先显示什么](/posts/018cc61644f4fa84ddeddb301e2c65fb/)
