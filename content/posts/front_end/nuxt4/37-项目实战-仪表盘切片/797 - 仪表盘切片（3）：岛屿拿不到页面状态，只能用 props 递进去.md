---
url: /posts/3ec9b7ce1d854bc6cdc108e30c2f05b1/
title: 797 - 仪表盘切片（3）：岛屿拿不到页面状态，只能用 props 递进去
date: 2026-10-29T11:00:00+08:00
updated: 2026-10-29T11:00:00+08:00
author: cmdragon

summary:
  摘要：把一块内容切成服务端组件之后，页面与它之间那层边界到底是什么形状。本站加了一个会自报读数的探针岛屿：页面写过的状态读不到、页面提供的注入读不到、`useRoute()` 读到的是根路径而不是用户那一页，而声明过的 props 一位不落全读得到。顺带量到两道拦截：外部改查询串撞的是文件名里那串哈希，自己模板里写错 `as` 撞的是保留键，后者会让整块内容静默消失。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 服务端组件
  - 岛屿
  - props
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 先让那一块自己开口

上一站把文档正文切成了服务端组件。这一站回答另一个问题：切下去之后，页面与它之间那层边界到底是什么形状。

浏览器打开 `http://localhost:3100/docs/getting-started`，拉到"回到列表"下面，那里多了一小块文字。DevTools 里它是这样一位：

```text
路由 /　·　页面地址 /docs/getting-started　·　状态 (没值)　·　注入 (读不到)
```

地址栏明明停在 `/docs/getting-started`，而那一块自己报出来的路由是 `/`；页面在同一个组件里写过的状态与注入，它一位都没接到。这正好对上官方讲服务端组件那一节的三条硬话：

- 「you cannot share state (provide/inject, Pinia, `useState`) between the page and the island; pass data via props instead」
- 「`useRoute()` inside an island reflects the island's own request, not the page the user is on」
- 「Props are serialized and sent as GET query parameters」

## 探针怎么摆

新增 `app/components/IslandFacts.server.vue`（本站第二个岛屿），页面侧在 `app/pages/docs/[slug].vue` 里给它三位：

```ts
// 页面这一侧先把两位写好
useState('docs:island-marker', () => '页面写的状态位')
provide('docs-island-marker', '页面 provide 的位')
```

```vue
<IslandFacts :slug="slug" :page-path="route.path" label="页面给的标签" />
```

`pagePath` 是岛屿声明过的一位（`defineProps<{ slug: string, pagePath: string }>()`），`label` 是没声明的那一位。两位并排放着，就是为了把"声明的读得到、没声明的落到属性上"一次量开。

岛屿内部每读一位都包一层 `试()`：

```ts
const 事实 = {
  route: 试(() => useRoute().path),
  state: 试(() => useState('docs:island-marker').value),
  inject: 试(() => inject('docs-island-marker', '(读不到)')),
  attrsKeys: 试(() => Object.keys(useAttrs()).sort()),
}
```

为什么要包这一层：岛屿里抛出的错不会冒到页面上。包岛屿的那一层有个 `onErrorCaptured`，它只写一条内部诊断，而生产构建里连诊断都不打——于是整块内容直接不存在，页面上一点痕迹都没有。探针不自己把原因写进 DOM，"这一位读不到值"和"这一块根本没渲出来"两种形状就分不开。

## 首屏那一趟读到什么

```text
[隔离·首屏] {"路由":"/","页面地址":"/docs/getting-started","状态":"(没值)","注入":"(读不到)","props键":"[\"pagePath\",\"slug\"]","属性键":"[\"data-island-uid=\"a0a9a794-c8de-4779-9b6a-7406846da891\"\",\"label\"]","上下文键":"[\"components\",\"id\",\"name\",\"props\",\"slots\",\"url\"]","上下文地址":"/","根上有label":true,"根上有as":false,"岛屿请求数":0,"用户在这一页":"/docs/getting-started"}
```

逐位读：

- **状态 `(没值)`、注入 `(读不到)`**：官方那句"不能共享状态"在这一层是可数的。岛屿是另一个独立的 Vue app（那一节的原话是 creates a new, isolated Vue app on the server to render just that component），`provide`/`inject`、Pinia、`useState` 都过不去。
- **路由 `/`**：不是用户所在那一页。服务端日志里跟着留下一条 `No match found for location with path "/__nuxt_island/IslandFacts_….json?props=…"`——岛屿那个 app 的路由表里没有这一条，`useRoute()` 就停在初始值上。
- **上下文地址 `/`**：`nuxtApp.ssrContext.islandContext` 确实带 `url` 这一位（键名读回来是 `components, id, name, props, slots, url`），但它的值来自调用方：没显式给 `context` 时，实现里默认就是 `/`。所以这一位同样给不出用户那一页。
- **props 键 `["pagePath","slug"]`**，而页面地址读回来是 `/docs/getting-started`：官方给的补救是那句"If an island needs route information, pass it in explicitly, either as props or via the `context` prop"。**声明过的那一位读得到，值就是页面给的那个字符串。**
- 属性键里有那位 `label`，根上有label 读回 `true`：没声明的 `label` 没进 props，它当属性落到了岛屿那一个根元素上。同一位里还能看到 `data-island-uid="…"` 这种键名里带等号的形状，那是包装自己塞进去的，别照着它推官方语义。官方那段 note 的落点就在这儿：「Props a component does not declare fall through as attributes onto its single root element」，补救是给那句「Set `defineOptions({ inheritAttrs: false })` on such islands, or declare the props you accept」。
- **岛屿请求数 0**：与上一站同口径，首屏是内联渲的，没有额外请求。

## 从列表页点进来，props 全躺在地址里

```text
[隔离·点进来] {"岛屿请求数":2,"岛屿名":["DocBody_QcL6U4ZyZxRekA0g1GULBbu6Zk1grdtT3jyhQMaGI","IslandFacts_jdygYGDOUPHKxsCpvFmYLDjqbJGfqiVEKWtsffZAno"],"查询串":"props={\"slug\":\"getting-started\",\"page-path\":\"/docs/getting-started\",\"label\":\"页面给的标签\"}","地址长度":262,"状态还在":"(没值)","注入还在":"(读不到)","路由还在":"/","上下文地址还在":"/"}
```

三件事一起说清。

页面上有两个岛屿，这一趟就是两往返。796 只量了"多一次"，这一位给出"有几个就多几趟"。

props 是序列化进查询串的：`props={"slug":"getting-started","page-path":"/docs/getting-started","label":"页面给的标签"}`，整条地址 262 字符。用户在哪一页这件事，现在跟着查询串一起进服务端访问日志、CDN 缓存键和 `Referer`——那条"avoid passing large amounts of data"的警告管的是大小，而**声明过的 props 同样进地址**这条管的是可见性，两件事都得知道。

名字两边不同形：查询串里是模板上那个 kebab 名 `page-path`，岛屿里 `Object.keys(props)` 给的是 camel 名 `pagePath`。按一边猜另一边会错。

## 两道闸门：外部改包与自己写错，撞的不是同一条

那段 note 里还有一句「Nuxt rejects the props most likely to leak through unintentionally: a top-level `as` that the island does not declare」。这一句要分两档量，因为能不能走到这条判断，取决于查询串里的哈希对不对。

外部把 `props` 改了再打（判据 `test/e2e/island-isolation.spec.ts` 档三）：

```text
[隔离·改包] {"原样":{"status":200,"短语":"OK","字节":732,"渲出探针块":true},"带as":{"status":400,"短语":"Invalid island request hash","字节":2433,"渲出探针块":false},"带陌生":{"status":400,"短语":"Invalid island request hash","字节":2451,"渲出探针块":false},"哈希在文件名里":true}
```

塞 `as` 是 400，塞一个无关的 `nickname` 也是 400，状态短语都是 **`Invalid island request hash`**。原因写在地址里：岛屿文件名那串（`IslandFacts_jdyg…`）就是"名字＋props＋context"的指纹。改包连那条保留键判断都进不到，先被哈希拦掉。

于是保留键那条只有自己把模板写错才撞得到。在页面那一行后面加一个 `as="p"`，再跑一次——这一档要故意写错才量得到，量完就得改回来，所以它不是常驻判据：

```text
[变异·as·首屏] {"探针块数":0,"正文还在":true,"页面上有报错":false}
[变异·as·点进来] {"探针块数":0,"岛屿响应":[{"名":"DocBody_QcL6U4ZyZxRekA0g1GULBbu6Zk1grdtT3jyhQMaGI","status":200,"短语":"OK","字节":264},{"名":"IslandFacts_CcCCG2xtCqNBAAE9XfrpbL1X00WkbkIFj3eDJu05Y","status":400,"短语":"Invalid island request props","字节":426}]}
```

这一档的形状比上一条更该记住：构建照样成功、页面照样 200、正文照样在，**首屏那一块整段静默消失**，控制台一声不响。只有从列表页点进来时那一趟才给出 400，短语换成 **`Invalid island request props`**（这一回哈希是对的，因为它是浏览器按含 `as` 的 props 自己算出来的）；同一页另一个岛屿仍然 200。

同一个 400，短语不同就是不同的门：`hash` 那道管"外面改包"，`props` 那道管"你把没声明的保留位写进了模板"。

## 第二个岛屿的代价比第一个小两个数量级

796 那一步（第一个岛屿）多花了 13,416 字节。同一份基线判据再跑一次：

```text
[切片基线·正文] {"status":200,"正文字节":255,"响应字节":5736,"负载字节":1611,"DOM出现次数":1,"负载里也带着这一份":false,"开标签":"<div class=\"doc-body\" data-testid=\"doc-body\" data-island-uid=\"cffd6d7e-1aaa-4fa8-8163-70b86be514cf\">","正文开头":"<p>这一篇讲第一次进来看见什么。</p>\n<h2>你需要知道的三件事</h2>"}
[切片基线·下载] {"脚本条数":15,"原始字节合计":322143,"正文字节":255,"倍数":1263,"最大的三条":[{"地址":"/_nuxt/DMpvWO_S.js","字节":80966},{"地址":"/_nuxt/BvhoSpNT.js","字节":78654},{"地址":"/_nuxt/DZ36QNkQ.js","字节":70597}]}
[切片基线·渲染器] {"服务端文件数":527,"服务端命中文件":["_/index2.mjs"],"客户端chunk数":20,"客户端命中数":0}
```

| 读数 | 796 之后 | 再加一个岛屿 |
| --- | --- | --- |
| 那段正文的字节 | 255 | 255 |
| 客户端 chunk 数 | 20 | 20 |
| 下载原始字节合计 | 321,828 | 322,143 |
| 整页响应字节 | 4,610 | 5,736 |
| 内联负载字节 | 1,249 | 1,611 |
| 服务端产物文件数 | 525 | 527 |

下载只多了 315 字节，chunk 数一位没涨——那套包装只在第一位岛屿时装一次，之后每位增加的是它自己那段 HTML 与一次往返。反过来说，796 那条"第一个岛屿净亏 13,416 字节"也不会因为多切几块而继续变差，但它同样不会自己变好：**收益仍然取决于那块自己拖不拖依赖**。

## 关掉脚本，这一位照样在 HTML 里

```text
[隔离·关脚本] {"探针块在":true,"状态那一行":"(没值)","注入那一行":"(读不到)","路由那一行":"/"}
```

岛屿的内容是服务端嵌进那份 HTML 的，爬虫与不发 JS 的浏览器读到的就是上面那六位。这也是这一线敢在公开页上动刀的原因：切坏内容的风险极低，风险全在边界与地址这两件事上。

## 排错表

| 撞上的现象 | 先查哪里 | 为什么 |
| --- | --- | --- |
| 岛屿里读到 `(没值)`，页面明明写过 | `useState`／`provide` 那两位 | 岛屿是独立 app，状态不过边界，只能走 props |
| 岛屿里 `useRoute()` 是 `/` | 模板里有没有声明并传进那一位 props | 它反映的是岛屿那一趟，不是用户那一页 |
| 页面上整块不见了、控制台干净 | 有没有把 `as` 这类保留位写进模板 | 首屏那一趟的抛出被吞掉，只有导航那趟给出 400 |
| 改 `props` 查询串就 400 | 状态短语是 `…hash` 还是 `…props` | 前者是文件名指纹，后者才是保留键那条 |
| 页面给的值在日志里出现了 | 那一位是不是 props | props 走 GET 查询串，声明与否都会进地址 |

## 自己动手跑一遍

```bash
npm run build
E2E_PORT=3293 npm run test:e2e -- test/e2e/island-isolation.spec.ts
```

预期 `4 passed`，四组读数与上面一致（岛屿文件名那串哈希与 `data-island-uid` 会随构建变，属正常）。想亲眼看到"整块静默消失"那一档：在模板那一行后面加 `as="p"`，重跑一次——首屏那一块会不见，而从列表页点进来时那趟岛屿请求回 400、短语是 `Invalid island request props`；试完记得删掉。

## 本文依据

- 官方：讲服务端组件的那一节（三条隔离的原话、`islandContext` 与 `context` prop 的用法、props 走 GET 查询串与"避免传大量数据"、那段 note 里"未声明的 prop 落到根元素／顶层 `as` 会被拒／`inheritAttrs: false` 或声明你接受的 props"、`<NuxtIsland>` 才暴露 load 与 error 事件）；讲 `<NuxtIsland>` 的那一节（`props` 与 `context` 两个入位、「Component props and context are sent as GET query parameters」）
- 仓库：`app/components/IslandFacts.server.vue`（新增探针）、`app/pages/docs/[slug].vue`（页面侧那三位与 `useState`／`provide`）、`test/e2e/island-isolation.spec.ts`（新增 4 条）、`test/e2e/island-baseline.spec.ts` 与 `test/e2e/console-baseline.spec.ts`（组件名单那一条由一位更新为两位）
- 上一批：[796 - 仪表盘切片（2）：加一个后缀，这一块就成了服务端组件](/posts/04485c034c69ad98cdedded4b5c6107a/)
- 下一篇：[798 - 仪表盘切片（4）：客户端导航会多一次岛屿请求，lazy 与 fallback 才不挡](/posts/c24ab414eeafab5f696b479c4a8b3d4e/)
