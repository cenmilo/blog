---
url: /posts/9ba4ad19738415676069389f3c8328ee/
title: 820 - 禁 JS 与 noScripts（2）：noScripts 路由规则开一档，脚本标签与预取一起变
date: 2026-11-02T10:00:00+08:00
updated: 2026-11-02T10:00:00+08:00
author: cmdragon

summary:
  摘要：上一站把浏览器的 JavaScript 引擎关掉，量出来是"一位都没少、只是不执行"。这一站真的开那条路由规则，让服务端别把脚本写进 HTML。同一个构建里留两条同形的页面做对照，逐项相减就知道省掉的是哪四类东西，也顺带读到一件意外：这一页的代码仍然躺在客户端包里。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - noScripts
  - 客户端包
  - 渲染模式
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 两页并排，屏幕上看不出差别

上一站的账摆在那儿：关掉引擎之后，`/`、`/docs`、`/me`、`/console` 四页的脚本标签、资源提示、负载段一位都没少，连整页字节都和开着引擎那一份逐字相同，唯一变的是水合。这一站要读的是另一种禁法——让服务端不发。

问题是"不发"这件事没法在同一页上既开又关。所以第一步是造两条**同形**的路由：

```vue [app/pages/no-js-lab.vue]
<script setup lang="ts">
useSeoMeta({ title: '无脚本实验位', robots: 'index, follow' })
const 计数 = ref(0)
</script>

<template>
  <section>
    <h1 data-testid="lab-title">无脚本实验位</h1>
    <p data-testid="lab-marker">这一段由服务端渲出来，脚本发不发都看得见</p>
    <button type="button" data-testid="lab-button" @click="计数 += 1">
      点一下计数 <span data-testid="lab-count">{{ 计数 }}</span>
    </button>
  </section>
</template>
```

另一页 `app/pages/with-js-lab.vue` 逐字同形，只有标题不同。差别全在路由规则那两位上：

```ts [nuxt.config.ts]
routeRules: {
  '/no-js-lab': { ssr: true, prerender: false, noScripts: true },
  '/with-js-lab': { ssr: true, prerender: false },
}
```

为什么不直接把规则开在现成的路由上。`/docs/**` 那一路被好几档端到端判据读着，它们要水合、要点按钮、要等客户端导航，一旦不发脚本，那些等水合的读数会一直等到超时；首页 `/` 是预渲染的，它的产物 `index.html` 被首屏 JS 与载荷那几档读着，改它等于同时改一堆别人的基线。新开一对页面看着多两个文件，换来的是**同一趟构建里的对照组**：两页相减，差掉的东西只能归给那一条规则。

## 逐项相减，差掉的恰好是四类

`npm run test:e2e -- test/e2e/no-scripts.spec.ts` 这一趟 11 passed (39.7s)，两条路由的读数并排：

| 读的是哪一位 | `/no-js-lab`（开了规则） | `/with-js-lab`（对照） |
| --- | --- | --- |
| 入口模块脚本 | 0 | 1 |
| 全部脚本标签 | 0 | 4 |
| `modulepreload` 提示 | 0 | 23 |
| `prefetch` 提示 | 0 | 1 |
| import map | 没有 | 有 |
| 负载段 `__NUXT_DATA__` | 没有 | 有 |
| 内联样式块 | 1 | 1 |
| 整页字节 | 1028 | 3513 |

少了 2485 个字节，少掉的正是官方那段清单列的四类：入口 `<script>` 标签、入口 chunk 的 import map、内联的那段负载、JavaScript 的资源提示。样式那一位不受影响——这一句要读准，本站的样式是**内联**发的（首页产物里读到 3 个 `<style>` 块、0 个 `stylesheet` 链接），所以对照的是内联块的数量，不是 `<link>` 的数量；两页各 1 块，一项不差。

内容呢？两页 `[data-testid="lab-marker"]` 里那句"这一段由服务端渲出来，脚本发不发都看得见"**逐字相同**，`<button>` 这个标签也在，计数初值两边都是 `0`，标题各自是"无脚本实验位"和"带脚本对照位"。删的是脚本，不是页面。

再往下读两位，这两行才是这一站最想留给你的东西：

- **下载也停了。** 详情页开着引擎跑一趟会请求 29 个 `.js`；换成开了规则的那一页，**0 个**。上一站"关掉引擎"时那条数是 40——不执行照样下载，不发出去才是真的不取回。
- **水合等的不是超时。** 对照页 `waitForFunction` 等得到 `__vue_app__`；实验页刚加载读一次是假，再等 1.2 秒仍是假。所以如果你在判据里等这一页水合，30 秒后拿到的是 `Test timeout`，那不是抖动。

## 一件意外：这一页的代码还在客户端包里

官方那节还有一句："build 期已知会被这类规则盖住的页面，会整个从客户端包里排除。"本机读不到这个效果。扫一遍 `.output/public/_nuxt` 里的 79 个 `.js`，"无脚本实验位"那句在 `BV5Q9oxp.js` 里，"带脚本对照位"那句在 `BLm4fmnX.js` 里——**两页的代码都在包里**，规则盖住的那一页没有被排除掉。

成因在包内能读到，两位开关管的段不是同一段。渲染器里有一条把全局位与路由位合成一个判断（`@nuxt/nitro-server/dist/runtime/handlers/renderer.mjs:142`），入口脚本那一段却只看路由位（`:212`）；而真正会删文件的另一段挂在应用级开关上（`nuxt/dist/index.mjs:7704`，条件是 `features.noScripts && !dev`）。

为了把这句读成实证而不是猜测，单独跑一趟应用级的构建：把 `features: { noScripts: 'all' }` 加上，只 `npm run build`，**跑完立刻改回**。那一趟读到的是：预渲染首页里 `<script>` 标签 **0** 个、import map 0、`modulepreload` 0、没有负载段，`_nuxt` 目录里的 `.js` 从 79 掉到 **9**，两个实验页的独有句子都不再出现在包里；而 `<style>` 仍是 3 块、CSS 文件仍是 3 个。

这一档不能常驻配置——应用级那一位一开，全站都不再水合，那些要水合、要点击的判据会一起红。它属于"要换一趟构建才读得到"的那一类开关，读一次记下数就够了。两种开关的分工可以记成一句：**路由级删的是这一页 HTML 里的四类东西，应用级还额外把包里的脚本文件删掉**。

## 加两条页面，红了三条单测

第一次跑 `npm run check` 时 `test:unit` 直接失败，三条红的都出自 `test/unit/indexing-surface.test.ts`：

```
AssertionError: 这些页面没表态，会跟着默认值走：
["app/pages/no-js-lab.vue","app/pages/with-js-lab.vue"]: expected […] to deeply equal []
```

这条判据是收录面那一站立的：**每条页面路由都必须显式发 `robots` 那一位**，取值只许两种，门后页与账户页是 `noindex`，其余公开页是 `index, follow`。新页面没表态，就跟着默认值走——而那正是这条判据要拦的情况。补上 `useSeoMeta({ ..., robots: 'index, follow' })` 之后 **278 条全绿**（43 个文件）。

值得把这件事记住的不是"要加一行"，而是它的形状：往一个已经建立过公开面账表的站里加页面，加的不是一个文件，是一个**必须表态的新地址**。这两页没有会话、没有私人数据，正文就是服务端渲的那一句，按站里的分类属于公开页，所以表态是 `index, follow`。

## 容易读错的几处

| 你看到的 | 实际是什么 | 该怎么办 |
| --- | --- | --- |
| 配置里写了规则，`nuxt dev` 里页面还有脚本 | 应用级那位默认 `'production'`，只在生产构建删 | 用 `routeRules` 里的 `noScripts`，它在所有模式都生效 |
| 判据等水合等了 30 秒超时 | 这一页永远不会水合 | 把"等不到"写成断言，别加超时重试 |
| 按钮点了不计数 | 不发脚本，处理器没人挂 | 这是文档明示的取舍，不是缺陷；交互那一半单独量 |
| 以为资源提示没了就省流量 | 只有"服务端不发"才停下载；关浏览器引擎时 40 个文件照样取回 | 想减下载，减的是 HTML 里那四类，不是勾设置 |
| 以为规则会把页面从客户端包里剔掉 | 本机 4.5.2 的路由级不剔，79 个 `.js` 里两页都在 | 剔包只挂应用级那位，读它要另开一趟构建 |
| 新增页面后单测红三条 | 站里有一条"每个地址都要表态"的账 | 给新页显式发 `robots`，别让它跟默认值 |

## 收住这一站的三条结论

内容不受影响：两页正文逐字相同、按钮标签在、样式内联块数量一致，删掉的只有那四类。
"不发"与"不执行"能分开量：前者的 `.js` 请求数是 0，后者是 40。
包大小不受路由级规则管：要减客户端包，得开应用级那一位，而那一位会把全站的水合一起关掉。

下一站读那条被官方称作"声明式 JSON、不执行 JavaScript"的标签——它会为被盖住的路由发出预取规则，而本机安装包里的实现位置只在一句客户端插件里，所以"读不到"本身也要有证据。

## 本文依据

- `guide/going-further/features` 的 noScripts 一段（命中时省掉的四类、CSS 不受影响、缓冲渲染、与服务端组件的相互作用、build 期已知页面从客户端包排除）
- `guide/recipes/mostly-static-sites`（与 `prerender` 的组合写法、"does not hydrate / nothing on it is interactive"）
- `guide/concepts/rendering` 的 hybrid rendering 表（`noScripts: boolean` 那一行）
- `errors/e8007`（依赖客户端脚本的路由在 `noScripts` 下上线静默坏）
- 包内：`@nuxt/nitro-server/dist/runtime/handlers/renderer.mjs:142`、`:212`，`nuxt/dist/index.mjs:7704`
