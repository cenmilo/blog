---
url: /posts/376ab34c41deb7820d37110380616eaa/
title: 574 - features：与 future 指向同一锚点的空壳节
date: 2027-04-21T10:00:00+08:00
updated: 2027-04-21T10:00:00+08:00
author: cmdragon

summary:
  摘要：`## features` 与 `## future` 一样，**在配置参考里没有正文**，只有一条 read-more，而且两条指向**同一个锚点**。真实内容在特性页的 `## features` 小节：**三个开关 `devLogs`、`inlineStyles`、`noScripts`，全都没有 `**Type**` 行、没有 `**Default**` 行**，取值范围只能从正文句子和示例里读。三处容易被漏掉的记载是：`devLogs` 除了布尔还有一个 `'silent'`；`inlineStyles` 的默认值写在正文里（`(id) => id.includes('.vue')`）**而不是 Default 行**，且官方限定它"目前只在用 Vite 时可用"；`noScripts` 的 `true` 会被**归一化成 `'production'`**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 特性开关
  - noScripts
  - 内联样式
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

想让页面只发 HTML 和 CSS、想让服务端日志出现在浏览器控制台、或者反过来不想把 CSS 内联进 HTML——这三个开关都在 `features` 下面。但你在配置参考里搜 `features`，看到的是空的一节。

## 配置页这一节里有什么

`## features` 整节只有一条 read-more：

> Learn more about Nuxt's opt-in features.

**没有定性句、没有 `**Type**`、没有 `**Default**`、没有子项清单。** 而且它指向的地址是 `guide/going-further/features#features`——和上一节 `## future` 指向的是**同一个锚点**。落地页上 `## features` 在上、`## future` 在下，所以两条链接都只会把你带到 `features` 那一节，`future` 要自己往下翻。

内容都在特性页。那一节的开场句给整个页面定性："Some features of Nuxt are available on an opt-in basis, or can be disabled based on your needs."

## 三个开关，零个类型行

特性页 `## features` 下面是三个 `###` 小节，**三个都没有类型行与默认值行**，能确证的取值只能从正文与示例拼出来。

### devLogs

> Stream server logs to the client as you are developing. These logs can be handled in the `dev:ssr-logs` hook.

三句状态说明：

- "By default, this is enabled in development (**when test mode is not active**)." —— 默认开着，但**带一个测试模式的例外条件**。
- "If set to `silent`, the logs will not be printed to the browser console."
- 示例给的是 `devLogs: true`。

也就是说这一项在文档里出现的取值至少有 `true` 和 `'silent'`，**没有一张表列出全部合法取值**，`'silent'` 与 `false` 的差别（日志还发不发到客户端）也没写。句子里那句 "These logs can be handled in the `dev:ssr-logs` hook" 是唯一的下游线索：**要拿这些日志得去订那个钩子**，不是自己 `console.log`。

### inlineStyles

> Inlines styles when rendering HTML. This is currently available only when using Vite.

第二句是适用范围限制：**内联样式这条路绑的是 Vite 构建器**。配合 `server.builder` 那一节（默认 `@nuxt/nitro-server`，可写 `"nitro"` 或 `"vite"`），如果你的构建链不是 Vite，这一项的行为文档没有承诺。

函数形式与默认值都写在正文里：

- 可以传一个**函数**，接收 Vue 组件的路径、返回布尔值表示该组件是否内联样式。
- 默认值是 `(id) => id.includes('.vue')`。

示例只给了一种写法：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  features: {
    inlineStyles: false, // or a function to determine inlining
  },
})
```

注意注释那句 "or a function to determine inlining" 是文档在这里给出的**唯一提示**：`false` 关掉头、函数按组件挑，两种形状并存。至于能不能传函数同时保留默认判断，文档没写。

### noScripts

三行取值是这一节写得最清楚的一份：

| 取值 | 官方原文要点 |
| --- | --- |
| `false` | 默认（default）：正常渲染脚本 |
| `'production'` | 只在生产构建里省略脚本，所以 `nuxt dev` 里 HMR 等开发特性照常工作；**写 `true` 会被归一化成 `'production'`** |
| `'all'` | 开发和生产都省略脚本 |

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  features: {
    noScripts: 'production', // or 'all' | false
  },
})
```

那句 "`true` is normalized to `'production'`" 是这一节最实用的一条：**你以为写 `true` 是"全部关掉"，文档说它等于 `'production'`**，也就是开发期照常发 JavaScript。上一节讲 E8007 时提到过的"dev 好的、上线静默坏"就是这个归一化造成的落差；要按路径关，用 `noScripts` **路由规则**，官方写明它"applies in all modes"（开发期也生效），落差当场就能看见。

`features.noScripts`（全站）与 `routeRules` 里的 `noScripts`（按路径）之间，文档没有写覆盖顺序——**两处同时存在时谁说了算，未说明**。

## 全站关掉脚本时，官方额外交代的那些

这一条开关在特性页里得到的篇幅远超另两条，因为它牵动的东西多。文档明写的有：

**导航方式会变。** 没有脚本的页面只能用整页加载跳转。Nuxt 会在这类页面上发一个 **speculation rules** 标签（声明式 JSON，本身不执行 JavaScript），支持的浏览器会在用户点之前预取、预渲染目标。规则的范围被限定在你的页面路由（可安全 `GET` 的那些），**所以指向服务端路由的链接（例如 `/logout`）不会被 speculative 地抓取**。但有一句例外：**catch-all 页面（如 `pages/[...slug].vue`）匹配所有路径，这会把范围又放宽回所有同源链接。**

**反向也一样。** 客户端导航到一个被 `noScripts` 路由规则覆盖的路由时，会触发整文档加载，而不是让客户端路由去渲染；建时能确定被覆盖的页面组件会**整个从客户端包里排除**。有脚本的页面也会发自己的 speculation rules 标签，范围限定在那些 `noScripts` 路由模式上；开了视图过渡时，这些页面还会用声明式的跨文档视图过渡来替代闪屏。

**两条渲染层面的连带后果**，官方写在 "A few interactions to be aware of" 里：

- 带 `noScripts` 路由规则的路由**总是用 buffered（非流式）渲染器**渲染。
- 只输出静态 HTML 的服务端组件没问题，但用 `nuxt-client` 水合的组件、以及 interactive island 的插槽，**依赖一段内联脚本来搬移 teleported 内容，因而在 `noScripts` 路由上不会变成可交互**；当 `features.noScripts` 是全站设置且启用了 component islands，Nuxt 还会因为这个原因退回 buffered 渲染。

最后一条 read-more 指向 mostly-static-sites 配方，把 `noScripts` 与预渲染、islands、懒水合的组合放在那里讲。

## 自查

- 配置页 `## features`：**只有一条 read-more**，与 `## future` 指向同一锚点；无定性句、无类型、无默认值、无子项清单。
- 特性页里三个开关**都没有 Type / Default 行**，取值靠正文与示例：`devLogs`（`true` / `'silent'`，默认在开发开启且"test mode 不激活"，日志走 `dev:ssr-logs` 钩子）、`inlineStyles`（`false` / 函数，默认函数 `(id) => id.includes('.vue')`，**只在用 Vite 时可用**）、`noScripts`（`false` / `'production'` / `'all'`，**`true` 归一化为 `'production'`**）。
- 全站 `noScripts` 的连带记载：speculation rules 标签与其范围、catch-all 会放宽范围、进出被覆盖路由是整页加载、页面组件从客户端包排除、**这些路由恒用 buffered 渲染**、`nuxt-client` 与 island 插槽不会变可交互、全站设置＋islands 时额外退回 buffered。
- `features.noScripts` 与 `routeRules` 的 `noScripts` 同时存在时的优先级：**文档未说明**。

## 下一篇

下一篇 575 看第三个同样只有链接的节 `## experimental`：它指向的那一节有 **57 个开关小节**，但**只有 17 条用了固定句式写明默认状态**（14 条 "enabled by default"、3 条 "disabled by default"），其余要读正文；而且那句"可以看 API 文档"的链接**指回它自己这一节**。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## features` 与 `## future` 两条 read-more 同锚点；`### server.builder` 的 `"nitro"` / `"vite"` 两种简写）
- https://nuxt.com/docs/4.x/guide/going-further/features（页面开场定性句、`devLogs` 三句、`inlineStyles` 的 Vite 限制句与默认函数、`noScripts` 三取值表与 `true` 归一化句、示例两处、speculation rules 与 catch-all 那段、"applies in all modes"、两条 interactions、指向 mostly-static-sites 的 read-more）
- https://nuxt.com/docs/4.x/errors/e8007（`features.noScripts` 默认 `'production'` 时 dev/生产落差、路由规则在开发期同样生效）
- https://nuxt.com/docs/4.x/guide/recipes/mostly-static-sites（`noScripts` 与预渲染、islands、懒水合的组合）
