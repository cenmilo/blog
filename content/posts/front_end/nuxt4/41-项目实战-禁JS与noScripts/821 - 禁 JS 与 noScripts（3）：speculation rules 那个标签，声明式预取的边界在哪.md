---
url: /posts/b8a18ba354bb20617467f17928bcadae/
title: 821 - 禁 JS 与 noScripts（3）：speculation rules 那个标签，声明式预取的边界在哪
date: 2026-11-02T11:00:00+08:00
updated: 2026-11-02T11:00:00+08:00
author: cmdragon

summary:
  摘要：一条不发脚本的页面，从地址栏进去和从站内点进去，行为并不一样。这一站把这两种到达方式分开量，顺手去找官方说的那个 speculation rules 标签——它是声明式 JSON，作用是让浏览器提前把下一跳取好。结果在本机的安装包里有意思：那个标签一处也找不到，而导航那一半的读数和文档写的话相反。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - noScripts
  - speculation-rules
  - 客户端导航
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 同一条地址，两种到达方式

上一站留了两条路由：`/no-js-lab` 被 `noScripts` 盖住（服务端不发那四类东西），`/with-js-lab` 什么都不开。这一站给它们各加了一颗 `<NuxtLink>`，然后问一句看起来不用问的话：**从站内点进那条不发脚本的页，会发生什么？**

先说官方打算解决什么。命中 `noScripts` 的路由没有客户端路由可用，所以每一次跳进跳出都应该是整页文档加载——浏览器要重新拿一份 HTML、重新解析、重新渲染，慢就慢在这儿。官方的补法是发一个 speculation rules 标签，让支持的浏览器在你点之前就把目标取好：

> Pages served without scripts navigate with full-page loads. To keep navigation fast, Nuxt emits a speculation rules tag on these pages…

再往下两句把范围说得很死：

> The tag is declarative JSON and executes no JavaScript. The rules are scoped to your page routes (which are safe to `GET`) rather than every same-origin link, so a link to a server route such as `/logout` is never speculatively fetched.

还有一句要留意：catch-all 页面（比如 `pages/[...slug].vue`）匹配所有路径，官方原话是这会 "widen this back to all same-origin links"，范围又被放宽回所有同源链接。

**声明式 JSON**是指它是浏览器自己读的一段配置，不是让你写的脚本；**整页加载**指换新文档，和客户端路由在同一个文档里换内容是两回事。这两句决定了这一站要读两位：标签在不在，以及跳转到底是谁接的管。

## 标签在本机一处都读不到，但先证明扫描真的看得见

判据文件加到 `test/e2e/no-scripts.spec.ts` 里（这一条线的判据都放同一个文件），四个地址各读一份 HTML，数那个标签，同时把这一页实际发出的 `<script>` 类型列出来：

| 地址 | `speculationrules` 次数 | 实际出现的脚本类型 | 全部脚本标签 |
| --- | --- | --- | --- |
| `/` | 0 | `application/json`、`importmap`、`module` | 4 |
| `/docs` | 0 | `application/json`、`importmap`、`module` | 4 |
| `/with-js-lab` | 0 | `application/json`、`importmap`、`module` | 4 |
| `/no-js-lab` | 0 | （一个都没有） | 0 |

右边那两列是这一档的全部意义：如果只报"0 次"，那可能是没实现，也可能是数的方式不对。同一份 HTML 里确实数得出 4 个脚本标签、三种 `type`，说明扫描读得见标签，只是没有这一种。而 `/no-js-lab` 那一行连类型清单都是空的，正是上一站那条规则的效果。

再从包内定一次位置。递归扫 `node_modules/nuxt/dist/app/plugins` 找这个串，命中一个文件：`cross-origin-prefetch.client.js`；扫 `@nuxt/nitro-server/dist` 命中 **0** 个文件。这一个小结可以写成一句：**本机 4.5.2 里唯一会发 `speculationrules` 的地方是一个客户端插件，而客户端插件本身要脚本才跑得起来**——它给的是跨域预取那个实验特性，与 `noScripts` 不是一件事。服务端渲染那一路一处都没有，所以"服务端为被盖住的路由发这个标签"这件事在本机不可能发生。

## 让那条"计数为 0"的判据红一次

读不到这件事要红一次才算钉住。临时往 `/with-js-lab` 塞一个真的标签：

```ts
useHead({ script: [{ type: 'speculationrules', innerHTML: '{"source_order":[{"href":"/no-js-lab"}]}' }] })
```

同一趟判据立刻变成：

```
[821·标签在不在] … "/with-js-lab":{"speculationrules次数":1,
  "脚本类型清单":["application/json","importmap","module","speculationrules"],"全部脚本标签":5}
Error: /with-js-lab 出现了 speculation rules 标签 ⇒ 本机版本已实现，这一档要改成读内容
Expected: 0  Received: 1
```

清单里多出那一位、标签数从 4 变 5、判据红，三件事同时发生，说明这条判据读的就是它。删掉这两行，14 条重新全绿。**"读不到"和"没测"的区别就在这里**：前者你手里有一次红。

## 导航那一半读得到，结论和文档相反

这一档要判"是谁接的管"。办法是在带脚本的那一页水合之后，往 `window` 上写一个标记，然后点链接：

```ts
await page.evaluate(() => { (window as unknown as Record<string, string>).__spa_marker = 'ALIVE' })
const 起点 = 文档请求.length
await page.locator('[data-testid="lab-link-nojs"]').click()
await page.waitForURL('**/no-js-lab')
const 标记 = await page.evaluate(() => (window as unknown as Record<string, string> | undefined)?.__spa_marker ?? '(没了)')
```

标记还在，说明文档没换（客户端路由接管）；标记没了并且多出一条 `document` 请求，说明真的整页加载。本轮读数：

```
[821·两向到达] {"正向标记":"ALIVE","正向新增文档":0,"正向落地水合":true,
 "直接进那页水合":false,"反向新增进页":1,"反向出去新增文档":1,
 "文档请求全清单":["/with-js-lab","/no-js-lab","/with-js-lab"]}
```

正向那三个数连在一起读：标记是 `ALIVE`、`document` 请求一条都没多、落地那页的水合位是 `true`。**在本机的 4.5.2 上，从带脚本的页面点向一条 `noScripts` 路由，是客户端路由接管了这一次跳转**，官方那句"triggers a full document load"读不到对应实现。反向那一向则完全按文档走：先用地址栏直接进那一页（`直接进那页水合: false`，页面确实没有脚本），再点出去——`document` 请求加了一条。整趟里 `document` 清单就是那三条地址。

这两向合起来是一条能带走的规矩：**`noScripts` 管的是服务端发出去的那一份 HTML，管不到已经被客户端路由接手的那次跳转。**同一地址的两种到达方式，一份带负载与脚本、一份都不带；判据要分别钉，别拿其中一次的读数去说另一种。

顺带记一个读数陷阱：落地之后 `page.content()` 读到的是**当前那份文档**的 DOM，也就是上一跳的 HTML——本轮它读到入口脚本 1 条、`modulepreload` 38 条、负载段在。把它当成"这一页发了多少脚本"会得出完全相反的结论，所以这一位只打印不断言。

## 预取那一半：对照组先空了

`<NuxtLink>` 会不会替看不见的下一跳提前取东西，本站读数是这样：

```
[821·预取对照] {"水合之后请求条数":18,"指向无脚本页":[],"指向详情页":[]}
```

水合之后再等 2.5 秒，一共 18 条请求，但指向 `/no-js-lab` 的 0 条，指向 `/docs/getting-started` 的也 0 条。**对照组是空的**——这意味着"没预取"这件事不能算给 `noScripts`，本站在这个构建里根本没读到 `<NuxtLink>` 的预取行为。判据里那段因此写成两种分支：对照组有数时才断言"被盖住那页 0 条"，对照组空时只记"归因未成立"。这是这条线攒下的另一条通用做法：**要说某个开关造成了某个缺席，先证明那个行为本来会存在。**

## 容易读错的几处

| 你看到的 | 实际是什么 | 该怎么办 |
| --- | --- | --- |
| HTML 里数不到 speculation rules 标签 | 本机版本没有这个实现，不是页面坏了 | 同时报"这一页实际发了哪几类脚本"，别只报 0 |
| 点链接进了不发脚本的那页，页面还能点 | 那次跳转由客户端路由接管了 | 用 `window` 标记加 `document` 请求数两位一起判谁接的管 |
| 落地后 `page.content()` 里全是脚本标签 | 那是上一跳那份文档的 head | 要读这一页发多少脚本，去读它的响应体 |
| 没有预取请求，就说是 `noScripts` 挡的 | 对照组本身也是 0 条 | 先证明这站读得到预取，再谈归因 |
| 断言 `读数[路径].次数` 直接通过本地跑 | `noUncheckedIndexedAccess` 下报 `TS2532 Object is possibly 'undefined'` | 索引取值加 `!`，或先落成具名变量 |

## 这一站的三条结论

标签：本机 4.5.2 读不到，且包内位置只在一个客户端插件里——这一句现在是带证据的（四份 HTML 的类型清单 ＋ 一次把判据读红的变异）。
导航：`noScripts` 只约束服务端发出的那一份 HTML；客户端路由接手的那一跳不受它管，两种到达方式必须分开钉。
预取：本站没读到，所以不能把"没预取"记在 `noScripts` 头上。

下一站处理这条线最容易被误会的一件事：那一块岛屿内容在 `noScripts` 的路上还剩什么——静态的那块内容仍在，`nuxt-client` 标出来的那块点了不动，而这是文档写明的取舍，不是缺陷。

## 本文依据

- `guide/going-further/features` 的 noScripts 一段（整页加载、speculation rules 标签那句原文、作用域只圈页面路由、catch-all 会放宽、带脚本页也发自己的标签）
- `guide/recipes/mostly-static-sites`（"does not hydrate"、"no client-side navigation away from the page"、链接退化成普通 `<a>`）
- `guide/concepts/rendering` 的 hybrid rendering 表（`noScripts: boolean` 那一行）
- 包内：`node_modules/nuxt/dist/app/plugins/cross-origin-prefetch.client.js`（本机唯一出现 `speculationrules` 的文件），`@nuxt/nitro-server/dist` 内该串 0 处命中
