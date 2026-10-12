---
url: /posts/31238240f04cba5ae74e738a1d6b7eb1/
title: 接入第三方 Vue 插件与指令：vueApp.use 与 directive
date: 2027-01-15T11:00:00+08:00
updated: 2027-01-15T11:00:00+08:00
author: cmdragon

summary:
  摘要：这一篇把"往应用里接第三方能力"的几种官方路子摆在一起：接 Vue 插件走 Nuxt 插件里的 nuxtApp.vueApp.use()（Nuxt 没有 main.js，这就是替代）；接全局自定义指令走 directive()，并要求两端都注册；改整页 HTML 输出则是另一类插件——Nitro 服务端插件订阅 render:html 钩子，views 那页给了向 head 推 meta 的完整例子。最后用最佳实践那页的口径收一句：能写成 composable 的就别都塞插件。

categories:
  - 前端开发

tags:
  - nuxt4
  - Vue插件
  - 指令
  - Nitro插件
  - render-html
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

"往项目里接一个别人的库"是高频需求。Nuxt 里有两类"插件"、三个接入点，容易混，这一篇一次摆清：接 **Vue 插件**、接**自定义指令**、以及**改整页 HTML 的 Nitro 服务端插件**。

## 为什么 Vue 插件要套一层 Nuxt 插件

先补一个背景。写纯 Vue 时你会在 `main.js` 里 `app.use(第三方VuePlugin)`。views 那页点明了 Nuxt 没这个文件：

> If you are familiar with Vue, you might wonder where `main.js` is (the file that normally creates a Vue app). Nuxt does this behind the scene.

因为应用实例是 Nuxt 在背后造的，你拿不到那个 `main.js`。所以官方给的路子是：**写一个 Nuxt 插件，在里面通过 `nuxtApp.vueApp.use(...)` 装第三方 Vue 插件**——`vueApp` 就是那个被 Nuxt 创建好的应用实例（273 讲过）。

官方完整示例（plugins 页 "Vue Plugins" 一节）分两步。第一步装依赖：

```bash [npm]
npm install --save-dev vue-gtag-next
```

第二步建插件文件：

```ts [app/plugins/vue-gtag.client.ts]
import VueGtag, { trackRouter } from 'vue-gtag-next'

export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.vueApp.use(VueGtag, {
    property: {
      id: 'GA_MEASUREMENT_ID',
    },
  })
  trackRouter(useRouter())
})
```

两个要点对齐前面：`use()` 装 Vue 插件是 vueApp 的三个方法之一（273）；文件名 `.client.ts` 是因为 Analytics 只在浏览器有意义（270 讲后缀）。

## 接全局自定义指令：directive()，且两端都要注册

自定义指令（`v-focus` 那种）同样在 Nuxt 插件里注册（plugins 页 "Vue Directives" 一节）：

```ts [plugins/my-directive.ts]
export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.vueApp.directive('focus', {
    mounted (el) {
      el.focus()
    },
    getSSRProps (binding, vnode) {
      // 服务端专用 props 在这给
      return {}
    },
  })
})
```

指令接入最容易翻车的是那条 warning（273 详述过，这里只点结论）：**指令必须在两端都注册**，除非你确定它只在一侧渲染。所以要么这个插件不加后缀（默认两端跑），要么客户端放真身 `.client.ts`、服务端补一个 `.server.ts` 桩。`getSSRProps` 是服务端渲染阶段给指令产出属性的入口。

## 另一类插件：Nitro 服务端插件改整页 HTML

上面两类是"应用侧（Vue 层）"的插件。还有一类完全不同的——**服务端 Nitro 插件**，它跑在 Nitro 那一侧，能在水合之前就动整页 HTML。views 那页的 "Advanced: Extending the HTML Template" 给的就是它：

> You can have full control over the HTML template by adding a Nitro plugin that registers a hook. The callback function of the `render:html` hook allows you to mutate the HTML before it is sent to the client.

```ts [server/plugins/extend-html.ts]
export default defineNitroPlugin((nitroApp) => {
  nitroApp.hooks.hook('render:html', (html, { event }) => {
    // 这是 HTML 模板的对象表示
    console.log(html)
    html.head.push(`<meta name="description" content="My custom description" />`)
  })
  // 也能在这拦截响应
  nitroApp.hooks.hook('render:response', (response, { event }) => { console.log(response) })
})
```

关键区别，摆一张表免得混：

| | Nuxt 插件（vueApp） | Nitro 服务端插件（render:html） |
| --- | --- | --- |
| 文件位置 | `app/plugins/` | `server/plugins/` |
| 包装函数 | `defineNuxtPlugin` | `defineNitroPlugin` |
| 跑在哪 | 应用创建期（客户端/服务端各按后缀） | Nitro 服务端渲染期 |
| 典型用途 | 装 Vue 插件、注册指令 | 直接改要发出去的 HTML（往 head 推标签等） |

server 那页 "Server Plugins" 一节也对得上：

> Nuxt will automatically read any files in the `~~/server/plugins` directory and register them as Nitro plugins. This allows extending Nitro's runtime behavior and hooking into lifecycle events.

也就是说 `server/plugins/` 里的文件自动注册成 Nitro 插件，订的是 Nitro 的生命周期钩子（`render:html`、`render:response` 这些），跟 `app/plugins/` 里的 Nuxt 插件是两套系统。views 那句 note 还提醒：只改 `<head>` 里的 meta 的话，通常该走 SEO/meta 那套（206 一带），不一定要写这种底层 HTML 插件——`render:html` 是"要完全掌控模板"时才用。

## 收一句：别什么都塞插件

best-practices 那页的口径（286 已展开）在这里当总闸：**能写成 composable 的共享逻辑就别用插件**，因为插件跑在水合期、每次都跑。本篇这三类是"确实需要插件"的场景——装 Vue 插件、注册应用级指令、改服务端 HTML——除此之外优先考虑组合式。

## 自查

- 接第三方 **Vue** 插件 → `defineNuxtPlugin` 里 `nuxtApp.vueApp.use(...)`；只在浏览器用的加 `.client` 后缀。
- 接全局指令 → `nuxtApp.vueApp.directive(...)`，且两端都注册（不分后缀，或 `.client`+`.server` 桩配套），SSR 属性走 `getSSRProps`。
- 改整页 HTML → 是另一类插件：`server/plugins/` 下的 `defineNitroPlugin` 订 `render:html`，别和 Nuxt 应用插件混。
- 只是改 `<head>` 的 meta → 优先 SEO/meta 那套，不必上 `render:html`。

## 下一篇

下一篇 288 讲只在服务端跑的那类插件：`.server` 后缀、`env.islands:false`，以及"哪些代码只能在服务端跑"（`nuxt/server` 的 Server Code Only 边界）。

## 本文依据

- Views（`getting-started/views`）："If you are familiar with Vue, you might wonder where `main.js` is ... Nuxt does this behind the scene."；"Advanced: Extending the HTML Template" 一节（"full control over the HTML template by adding a Nitro plugin that registers a hook"、"callback function of the `render:html` hook allows you to mutate the HTML before it is sent to the client"、`server/plugins/extend-html.ts` 的 `defineNitroPlugin` + `render:html`/`render:response` 示例、只改 head 走 SEO/meta 的 note）。
- plugins 目录（`directory-structure/app/plugins`）：Vue Plugins 一节（`npm install vue-gtag-next` + `vue-gtag.client.ts` 的 `nuxtApp.vueApp.use(VueGtag, {...})`）；Vue Directives 一节（`directive('focus', { mounted, getSSRProps })` 与"两端注册/`.client`+`.server` 桩"warning）。
- server 目录（`directory-structure/server`）：Server Plugins 一节（"Nuxt will automatically read any files in the `~~/server/plugins` directory and register them as Nitro plugins ... hooking into lifecycle events."）。
- Nuxt Plugins（`guide/best-practices/plugins`）：Use Composition Whenever Possible（能组合就别插件）。
