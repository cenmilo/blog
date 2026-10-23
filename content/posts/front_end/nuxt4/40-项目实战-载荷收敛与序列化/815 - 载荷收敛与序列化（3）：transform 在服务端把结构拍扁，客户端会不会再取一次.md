---
url: /posts/e7b14bf1338c432695e76bb534098f36/
title: 815 - 载荷收敛与序列化（3）：transform 在服务端把结构拍扁，客户端会不会再取一次
date: 2026-11-01T11:00:00+08:00
updated: 2026-11-01T11:00:00+08:00
author: cmdragon

summary:
  摘要：列表页的负载里，每条都带着这一页根本没渲染的那一位，而上一站那个只筛顶层键的选项在数组上什么也选不出来。这一站换 transform：把每条压成页面真正读的三位，读到的差值是五十六个字节。第二件事更要紧——把三种进法分开数：整页加载那一趟不发，客户端点进来会发一次，再点进来还会发一次，而这个数字与改不改结构毫无关系。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 负载序列化
  - useAsyncData
  - 性能优化
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 这一位没人渲染，但每条都带着

列表页 `/docs` 上你能看见的只有两样：每行一个标题（连着地址）和一个日期。模板在 `app/pages/docs/index.vue:105-112`，读的是 `d.slug`（当 `:key`、拼链接）、`d.title`、`d.date`——**`d.tags` 一次都没出现**。

可负载里每条是四位。上一站那个"只留指定键"的选项在这里用不了：这一次取数的结果是个**数组**，而它的实现是三行 `newObj[key] = obj[key]`（`node_modules/nuxt/dist/app/composables/asyncData.js:281-285`），给它 `['tags']` 得到的是 `{}`（793 量过这个形状）。**要收窄"每一条里的某一位"，只有 `transform` 这条路。**

官方讲 `useAsyncData` 那一节，参数表里这一项也只有一句：「Function to transform the result after resolving.」——结果拿到手之后，用一个函数改它。

## 写法

```ts
const { data: docs, status, error } = await useAsyncData('docs:list', () => $fetch<DocSummary[]>('/api/docs'), {
  transform: 列表 => 列表.map(({ slug, title, date }) => ({ slug, title, date })),
})
```

顺序在源码里挨着两行，值得看一眼：

```js
// asyncData.js:355-356
if (options.transform) result = await options.transform(_result);
if (options.pick) result = pick(result, options.pick);
```

**先 `transform`，再 `pick`**，而且 `transform` 是可以返回 Promise 的（那句 `await` 就是为它留的）。两个一起给时，`pick` 筛的是 transform 之后的形状——如果你的 transform 把顶层换了形，`pick` 里写的键就全落空了。

## 前后各读一趟

```text
改前 [transform·列表负载] {"整页字节":5841,"负载字节":453,"占比":7.8,"条数":2,
  "每条键清单":["date","slug","tags","title"],
  "第一条字符串位":{"slug":11,"title":10,"date":10},"负载里tags键次数":2}
改后 [transform·列表负载] {"整页字节":5785,"负载字节":397,"占比":6.9,"条数":2,
  "每条键清单":["date","slug","title"],
  "第一条字符串位":{"slug":11,"title":10,"date":10},"负载里tags键次数":0}
```

| 读数 | 改前 | 改后 | 差 |
| --- | --- | --- | --- |
| 整页字节 | 5841 | 5785 | -56 |
| 负载字节 | 453 | 397 | -56 |
| 负载占整页 | 7.8% | 6.9% | -0.9 个点 |
| 每条的键清单 | 四位 | 三位（`tags` 没了） | 少一位 |
| 页面第一行日期 | `2026-11-24` | `2026-11-24` | 一个字没变 |

56 个字节是四条标签加两个键名的总账，两个差值仍是同一个数——少掉的完全在负载上。**别凭这个数判断"值不值"**：这一页只有两条、标签都短，标签多的站点上同一处能差出成百个字节。要判断只有一句话：读你这一页的实际响应。

## 客户端会不会再取一次：三种进法分开数

这一问的答复不在字节里，在请求次数里。判据在同一个浏览器里走三档：

```text
[transform·客户端] {"整页加载那一趟":0,
  "点回列表那一趟":["http://localhost:3435/api/docs"],
  "再回来那一趟":["http://localhost:3435/api/docs"],
  "那段全部":["/api/docs/pricing-faq","/__nuxt_island/DocBody_….json?props=…",
             "/__nuxt_island/IslandFacts_….json?props=…","/api/docs"],
  "接口首条键清单":["date","slug","tags","title"],
  "浏览器里那份":{"是不是数组":true,"条数":2,"第一条键清单":["date","slug","title"]},
  "页面日期首位":"2026-11-24"}
```

- **整页加载 `/docs`：0 趟。** 数据跟着 HTML 一起到了，水合那一份从负载里读（这是 787 立的那条，本步重跑仍绿）。
- **点进详情、再点回列表：1 趟。** 这一趟是浏览器发的，正常——`docs:list` 从没在这台浏览器里出现过。
- **再点进详情、再点回来：还是 1 趟，不是 0。** 这条容易读成"缓存坏了"。要说清楚的是：判据里没有断言它该是 0，读到的就是这个数；而**它和 transform 无关**——这一档在加 transform 之前跑过，那时也是 1 趟，加完还是 1 趟。
  为什么这一趟会发，本步没把它钉到底，只把两条相关的事实放在这里：默认那次读的是 `nuxtApp.static.data`（`asyncData.js:413-415`：水合那趟读 `nuxtApp.payload.data[key]`，否则读 `nuxtApp.static.data[key]`——两个分支读的是两个地方），而填 `static.data` 的那一段在 `payload.client.js:26-34`，它拷的是 `loadPayload(toURL)` 取回来的载荷文件里的 `payload.data`。本站只有 `/` 是预渲染的（`nuxt.config.ts:65-71` 给 `'/docs/**'` 写的是 `ssr: true, prerender: false`，产物里只有一份 `/_payload.json` 这件事在 `test/e2e/payload.spec.ts` 钉过），但"这一路的载荷取回里没有这一键"还是"根本没去取"，本步没读到，留在这里等下一趟补。

同一份数据、两个分支，读的是两个不同的地方：水合那趟读 `payload.data`，导航那趟读 `static.data`。把这两件事记成"数据在浏览器里存过就不会再取"，是这一站最容易犯的错——真要不要这一趟，办法在这个分支之外（要么让这一路有载荷文件，要么自己给 `getCachedData`）。

而 `transform` 有没有让客户端多取一次？**没有**：改之前那一趟是 1 次、再进来 1 次（本步改前也跑过同一档判据），改之后一模一样。它改的只是"手上那份长什么样"。

## 它在浏览器那一趟也跑了

`接口首条键清单` 是四位，`浏览器里那份` 的键清单是三位。第二个数来自浏览器内存里的 `payload.data['docs:list']`（走 `$nuxt` 那个入口读，750 那一站立过写法）。**浏览器自己发的那一趟也过了 transform**——不是只有服务端那一趟生效。

这决定了一个容易漏的口径：接口换了字段名、或者多回了一位，服务端渲的那份和客户端点进来的那份会同时受影响，不会一个有一个没有。

## 挑错一位会怎样：这次不炸整页，是静默少一行

把 transform 里少留的那一位换成页面要渲的那一位（`date` 也不留）：

```text
app/pages/docs/index.vue(118,44): error TS2339: Property 'date' does not exist on type '{ slug: string; title: string; }'.
[transform·列表负载] {"负载字节":351,"每条键清单":["slug","title"],"负载里tags键次数":0}
[transform·客户端] {"页面标题数":2,"页面日期首位":""}
```

返回类型跟着变成 `{ slug: string; title: string; }`，模板里 `{{ d.date }}` 那一处 TS2339（上一站是七处，因为那一页读 `frontmatter` 读了七次）。运行时那一趟也没炸：**日期那一栏变成空的**，标题两条照旧在。

两位坏法要分开记。上一站是 `doc.frontmatter.title`——**读一个不存在的对象上的位**，服务端渲染期直接抛，代价是整页错误页；这一站是 `{{ d.date }}`——**对象在、某一位是空的**，Vue 把空值渲成空串，页面不报错，只是少了一栏。后一种更难发现：判据要是只盯着"页面还在不在"，它就过去了。所以判据里那位日期是单独一条：读 `docs-date` 第一行的文本，断它是 `2026-11-24` 这样的非空串。

顺带一条源码里的兜底：`asyncData.js:357-360` 在开发模式的服务端检查这一次取数的结果是不是 `undefined`，是就报 `NUXT_E3006`。写了 transform 却忘了 `return`（比如写了个 `{ }` 块没返回值），会被它报出来——但只限开发模式，别指望线上有提示。

## 与上一站那张分工表

| 要削的东西 | 用哪个 | 为什么 |
| --- | --- | --- |
| 顶层对象的某几位 | `pick: ['a','b']` | 三行 `newObj[key] = obj[key]`，只按顶层键取 |
| 数组里每一条的某几位 | `transform` | 数组上 `pick` 只会得到 `{}` |
| 往下一层（`frontmatter.title`） | `transform` | `pick` 不认带点的键，会写成 `{'frontmatter.title': undefined}` |
| 换形状、拼字符串、按条件留位 | `transform` | `pick` 只能整位取舍 |

两个都给时记住顺序：`transform` 在前、`pick` 在后（`:355-356` 那两行）。

## 判据

`test/e2e/payload-shape.spec.ts` 加到 7 条，这一站新增两条：

```bash
E2E_PORT=3433 npm run test:e2e -- test/e2e/payload-shape.spec.ts
#   7 passed (20.7s)
```

基线⑥断的是相对形状（`每条键清单` 等于三位、`负载里tags键次数` 等于 0），不是字节数——字节会随内容长度与构建时间戳漂。基线⑦断的是三个数字（0／1／1）加那对"接口四位 vs 手上三位"的对照；它读的是请求计数，所以必须用浏览器档（`page.on('request')`），走 HTTP 层的判据看不到这一位。

## 对不上表

| 看到的现象 | 大概率是 | 怎么确认 |
| --- | --- | --- |
| 页面某一栏变空了，但没报错 | transform 少留了页面要渲的一位 | 跑类型检查看模板；再读那一栏的文本是否为空串 |
| 整页变成错误页 | 模板在不存在的对象上取位（`a.b.c` 里 `a.b` 就没了） | 看响应状态码与整页字节，服务端那一拍边界接不住（811） |
| 客户端每次点进来都发一趟 | 这一路没有载荷文件，`static.data` 排不上 | 数 `page.on('request')`；再看产物里有没有对应的 `_payload.json` |
| `pick` 加了等于没加，结果变成 `{}` | 取数结果是数组 | 换 transform |
| 两个选项一起给，`pick` 全落空 | 顺序是先 transform 再 pick | 读 transform 之后的形状再决定 pick 写什么 |

下一站 816 换一个问题：**什么样的值根本进不了负载**。函数、Symbol、类实例、`Map`、`Date` 一位一位试，读它们各自断在哪一步——`useState` 那一节有一句 important 明说"会被 JSON 序列化，不能放不可序列化的东西"，而它旁边的错误段里那句 `Cannot stringify arbitrary non-POJOs` 到底什么时候真的打出来，这一站要现读。

## 本文依据

- 讲 `useAsyncData` 那一节（参数表 `transform`／`pick` 两行、`await` 与不 `await` 那段、"同键多处调用选项要一致"那段）：https://nuxt.com/docs/4.x/api/composables/use-async-data
- 讲 `useFetch` 那一节（同名选项）：https://nuxt.com/docs/4.x/api/composables/use-fetch
- 讲 `useState` 那一节（会被 JSON 序列化那句，下一站的主线）：https://nuxt.com/docs/4.x/api/composables/use-state
- 顺序、数组上的行为、`static.data` 与 `payload.data` 的分工、`NUXT_E3006`：官方文档未记载，本节引的是本机 `node_modules/nuxt/dist/app/composables/asyncData.js` 的 `:281-285`、`:355-356`、`:357-360`、`:413-415` 与 `node_modules/nuxt/dist/app/plugins/payload.client.js` 的 `:26-34`
