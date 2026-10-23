---
url: /posts/e9e6b9e4a8e4382f4bc3aba7aea65d4f/
title: 819 - 禁 JS 与 noScripts（1）：先把脚本全关掉这件事量在现在的站上
date: 2026-11-02T09:00:00+08:00
updated: 2026-11-02T09:00:00+08:00
author: cmdragon

summary:
  摘要：在浏览器里把 JavaScript 关掉，页面看着一切正常，正文、标题、链接都还在，于是很容易得出"我这站已经不需要脚本"的结论。这一节先把两件事分开：浏览器不执行，和服务端不发出去，删掉的东西完全不同。先拿现成的四条路由量出引入前的账，后面开那条路由规则时才知道读到的变化该归给谁。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - noScripts
  - 服务端组件
  - 渲染模式
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 屏幕上先出现的是"什么都没坏"

打开 DevTools 的设置，勾掉 JavaScript，再刷新首页。页面还是那个页面：标题在、正文在、导航链接能点，点了整屏换一次页面。服务端组件渲出来的那一段也在。

这个画面会让人想往下走一步：既然关掉脚本还都能看，那干脆不发脚本。问题是**"不发脚本"和"不执行脚本"是两个东西**，而你现在勾掉的那个只是后者。要区分它们，得先有一份能读的账——这个项目里四条路由在两种读法下各发出去多少东西，逐位数出来。

先约定几个词。**水合**指浏览器接手服务器已经渲好的那段 HTML，给它挂上事件与状态，挂完之后页面才开始响应点击。**资源提示**指 `<head>` 里那批 `<link rel="modulepreload">` 和 `<link rel="prefetch">`，它们告诉浏览器"这些文件待会儿要用，先取回来"。**负载**指页面末尾那个 `<script type="application/json" id="__NUXT_DATA__">`，服务器把这一趟取到的数据序列化进去，浏览器接手时读它，省掉一次重复请求。

## 官方给的是两个开关，不是一个

`features` 那一节写得很直白：`noScripts` 的作用是把 Nuxt 的脚本与 JavaScript 资源提示的**渲染**关掉，让页面以纯 HTML 加 CSS 发出去。三种取值：

| 写法 | 效果 |
| --- | --- |
| `false` | 默认值，脚本照常渲染 |
| `'production'`（或 `true`，会被归一成 `'production'`） | 只在生产构建里删，`nuxt dev` 里热更新这些开发特性照旧 |
| `'all'` | 开发和生产都删 |

同一段还给了另一条更细的路：**按路由**开。文档原话是这条路由规则"applies in all modes"，也就是说开发环境当场就生效。

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    '/blog/**': { noScripts: true },
  },
})
```

命中之后 HTML 里省掉的是四类东西，一行都不能少记：入口那几个 `<script>` 标签、入口 chunk 的 import map、内联的负载 `<script>`（负载外提时还有那条预取 link）、JavaScript 的资源提示。CSS 不受影响，样式链接与内联样式照旧，所以页面看起来一模一样——只是它永远不会水合。

还有两句要一起记住：命中这条规则的路由一律走缓冲（非流式）渲染；只渲静态 HTML 的服务端组件没问题，而 `nuxt-client` 标出来的那块和水合型岛屿 slot 靠一小段内联脚本把 teleport 的内容挪到位置上，那段脚本被省掉之后它们在 `noScripts` 的路上不会变可交互，**静态岛屿不受影响**。服务端组件那一节把同一件相互作用又写了一遍，两处口径一致。

## 引入前的账：把引擎关掉，量四条路由

这一步不改任何应用代码。新建 `test/e2e/no-scripts.spec.ts`——M4 那张验收表点名的就是这个文件名，因为它要证的正是「禁 JS 时 islands 区块仍出内容」那一句。

```ts [test/e2e/no-scripts.spec.ts]
const 四页 = ['/', '/docs', '/me', '/console']

/** HTML 里"发出去了多少 JS 相关东西"的形状。 */
function 数脚本面 (html: string) {
  return {
    入口模块脚本: [...html.matchAll(/<script type="module"[^>]*src="\/_nuxt\/[^"]+"/g)].length,
    全部脚本标签: (html.match(/<script\b/g) ?? []).length,
    modulepreload: (html.match(/<link rel="modulepreload"/g) ?? []).length,
    prefetch: (html.match(/<link rel="prefetch"/g) ?? []).length,
    负载段在: /<script[^>]*id="__NUXT_DATA__"/.test(html),
    importmap在: /<script[^>]*type="importmap"/.test(html),
    整页字节: Buffer.byteLength(html),
  }
}

/** 水合位：主客户端 app 接手过才算真。 */
const 水合位 = (页: Page) => 页.evaluate(() => Boolean(
  (document.querySelector('#__nuxt') as (Element & { __vue_app__?: unknown }) | null)?.__vue_app__,
))

const 上下文 = await browser.newContext({ javaScriptEnabled: false })
const 页 = await 上下文.newPage()
```

跑一条命令，五档都在里面：

```bash
npm run test:e2e -- test/e2e/no-scripts.spec.ts
# 5 passed (34.9s)
```

第一轮读到的账（关掉引擎这一路）：

| 路由 | 状态 | 全部脚本标签 | modulepreload | prefetch | 负载段 | 整页字节 | 正文可见字符 | 水合位 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `/` | 200 | 4 | 27 | 4 | 在 | 7067 | 354 | 假 |
| `/docs` | 200 | 4 | 25 | 1 | 在 | 6275 | 238 | 假 |
| `/me` | 200 | 4 | 27 | 1 | 在 | 4452 | 178 | 假 |
| `/console` | 200 | 4 | 27 | 1 | 在 | 4452 | 178 | 假 |

四个页面一位都没少。同一份 HTML 被两种浏览器读法各取一次时，两份数出来的东西逐项相等，连整页字节都一样（首页两份都是 7067）——唯一变的是那一位水合：开着引擎的是真，关掉的是假。这条相等关系就是判据本体：把它写成"字节大于零"什么都钉不住。

还有一位更容易被误读。关掉引擎之后，那一页仍然把脚本文件全取回来了：详情页那趟读到 **40** 条 `/_nuxt/*.js` 请求（开着引擎是 43 条），样本里有 `/_nuxt/xMZKfEGY.js` 这样一串。所以"关掉 JavaScript"绝不是省流量的开关，它只决定执不执行，不决定发不发、取不取。

最后钉住"这一站还没开过那条规则"。配置里那张 `routeRules` 表现在有十把键：`/`、`/me`、`/console/**`、`/docs/**`、`/sitemap.xml`、`/llms.txt`、`/robots.txt`、`/api/**`、`/api/docs`、`/**`，里面 `noScripts` 出现 **0** 次，`features` 那一位也是 0 处。判据读的是配置文件源文本里的计数，加上产物首页 `.output/public/index.html` 的指纹：入口模块脚本 1 条、负载段在。两边一起钉，是因为只有配置那一位会悄悄被人改成全站生效。

## 把开关推上去一次，才知道后面读什么

只量"引入前"，下一站的读数没法归因，所以这里做一次对照：给 `/docs/**` 加一位 `noScripts: true`，重跑同一套读法。列表页那一份立刻变成另一副样子——入口模块脚本 0、全部脚本标签 0、modulepreload 0、prefetch 0、负载段没有、import map 没有、整页 3419 字节（原来是 6275，少了 2856）。而同一页的标题位数还是 1、正文可见字符还是 **238**，一个字符没变；没被规则命中的 `/`、`/me`、`/console` 三份逐位不动。

同一趟里另外三个读数也一起变了，它们才是这一站要留下的东西：详情页那一档在"开着引擎等水合"的地方撞到 30 秒超时，因为这一页再也没人接手；关引擎那一路的 `.js` 请求从 40 条掉到 **0** 条，开引擎那一路同样 0 条——不发出去才是真的不下载；服务端组件渲的那一块内容还在，正文段 72 个字符、岛屿那块在、`props` 递进去的那一位是 `/docs/getting-started`。把这一档改回去之后，五档重新全绿。

## 容易读错的几处

| 症状 | 成因 | 怎么办 |
| --- | --- | --- |
| 关掉 JS 后页面照常显示，就判断"这页不需要脚本" | 只禁了执行，水合从没发生，页面里靠脚本才动的那几位永远停在初始值 | 读 `__vue_app__` 那一位，别拿"看得见"当"能交互" |
| 关着 JavaScript 的上下文里 `innerText`、`evaluate` 照样能用，怀疑开关没生效 | 自动化框架走的是协议层注入，跑的不是页面自己的脚本 | 用"两份脚本面逐项相等、水合位一假一真"这对关系判 |
| 开了 `noScripts` 之后等水合的那句一直不返回，被当成抖动跳过 | 那条路上永远不会水合 | 超时就是证据，把它写成断言而不是等待 |
| 想靠关浏览器脚本给用户省流量 | 资源提示与脚本文件全部照常下载 | 要减下载得让服务端不发，那是路由规则那一位的活 |
| dev 里加了配置看不到差别，以为规则没生效 | `features.noScripts: 'production'` 默认只在生产构建删 | 用 `routeRules` 里的按路由那条（它在所有模式都生效），或把全局位写成 `'all'` |

版本账要单独记一句。本机装的是 nuxt 4.5.2，`routeRules` 那一位确实有实现——渲染器里读的就是它（`@nuxt/nitro-server/dist/runtime/handlers/renderer.mjs:142` 把全局位与路由位合成一个判断，`:212` 用路由位决定入口脚本发不发）。而文档里另外两句在本机读不到对应实现：一是"命中规则且构建时已知的那些页面会从客户端包里整体剔除"，本机那段剔除逻辑挂在应用级开关上（`nuxt/dist/index.mjs:7704`，条件是 `features.noScripts && !dev`）；二是被覆盖路由的那个 speculation rules 标签，本机 `speculationrules` 这个字符串只出现在一个客户端插件里（`nuxt/dist/app/plugins/cross-origin-prefetch.client.js`），而客户端插件本身就要脚本才跑得起来。连配套的错误码 `E8007` 在本机安装包里也是零命中——那一节写的是"依赖客户端的路由上线后静默坏"，属于这一条线的下一站要防的事，但诊断信息比文档版本新，所以本机读不到。这几处都记成"版本差"，不要按文档断言。

## 现在能做的三件事

先把这份账留在这站：改一条路由的规则之前，站上任何一条 `noScripts` 都没有，四条路由都发脚本、都不省下载，正文可见量分别是 354、238、178、178 个字符。

自己站上要复现，顺序就三步：给判据加那个 `javaScriptEnabled: false` 的上下文，把上面那张表按自己项目的路由重填一遍，再用"两份逐项相等"钉住它。第三行最关键，因为它是唯一能区分"我关的是引擎"和"我删的是标签"的写法。

下一站把规则真的开一档，读同一页开关前后的脚本面、负载段与预取条数，并且回头看那条预渲染与它一起配会不会打架。

## 本文依据

- `guide/going-further/features` 的 noScripts 一段（取值、命中后省掉的四类、缓冲渲染、与服务端组件的相互作用）
- `guide/concepts/rendering` 的 hybrid rendering 表（`noScripts: boolean` 那一行）
- `guide/recipes/mostly-static-sites`（与 `prerender` 的组合写法、"does not hydrate" 那一句、两个方向的整页加载、静态岛屿仍工作）
- `guide/concepts/server-components`（island slot 与 `nuxt-client` 依赖那一段内联脚本，以及"Plain static islands are unaffected"）
- `getting-started/server`（`ssr`、`appMiddleware`、`noScripts` 都属于改变渲染行为的路由规则）
- `guide/best-practices/performance`（预渲染、`noScripts`、服务端组件、延迟水合组合成近零 JavaScript）
- `errors/e8007`（依赖客户端脚本的路由在 `noScripts` 下上线静默坏）
