---
url: /posts/59da860f76e8679fa8247a7ec069f2ce/
title: 273 - nuxtApp.vueApp：往 Vue 应用实例上挂指令、组件与插件
date: 2027-01-11T09:00:00+08:00
updated: 2027-01-11T09:00:00+08:00
author: cmdragon

summary:
  摘要：上一篇说插件的唯一入参是 nuxtApp。这一篇讲 nuxtApp 身上那个 vueApp——它就是 Vue 自己的应用实例（createApp 返回的那个 app）。注册全局组件、全局自定义指令、装第三方 Vue 插件，这三件"必须作用在整个应用上、只配一次"的事，都在 vueApp 上做。useNuxtApp 页给了三个方法的官方一句话说明，plugins 页给了 vue-gtag 和自定义指令两个完整例子；指令那条还附了"必须两端注册"的 warning，一并讲清。

categories:
  - 前端开发

tags:
  - nuxt4
  - vueApp
  - 指令
  - Vue插件
  - 插件
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇（270）留了个尾巴：插件的入参 `nuxtApp` 上挂着 `vueApp.use(...)`、`vueApp.directive(...)`。这一篇就讲这个 `vueApp` 是什么、能干什么。

## vueApp 就是 Vue 的应用实例

use-nuxt-app 页的定位句：

> `vueApp` is the global Vue.js [application instance](https://vuejs.org/api/application#application-api) that you can access through `nuxtApp`.

背景补一句：一个 Vue 应用是用 `createApp(...)` 造出来的，返回那个对象官方叫 "application instance"，它上面有 `.use()`、`.component()`、`.directive()` 这些"配置整个应用"的方法。Nuxt 已经帮你把应用实例造好了，你不用自己 `createApp`——通过 `nuxtApp.vueApp` 就能拿到它，然后调这几个方法。

官方在 use-nuxt-app 页给了三个方法的说明，逐字：

- `component()` - Registers a global component if passing both a name string and a component definition, or retrieves an already registered one if only the name is passed.
- `directive()` - Registers a global custom directive if passing both a name string and a directive definition, or retrieves an already registered one if only the name is passed.
- `use()` - Installs a Vue.js Plugin.

三个共同点：它们都是**应用级**的——注册一次，全应用的组件都能用。这就是为什么这类事放插件里、而不是放某个组件的 setup 里（组件 setup 会随组件反复跑，应用级配置不该反复做）。

## 装一个第三方 Vue 插件

`use()` 用来把"Vue 插件"（生态里那些 `xxx` 带 `.install` 的包）装进应用。plugins 页给的是接 Google Analytics 的 `vue-gtag-next`，完整例子：

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

注意文件名是 `vue-gtag.client.ts`——带了 `.client` 后缀（270 讲过后缀限定插件只在客户端加载）。Analytics 只在浏览器有意义，服务端渲染 HTML 时用不着它，所以放 `.client`。这也是"用 vueApp 装第三方 Vue 插件"最常见的落点。

## 注册一个全局自定义指令

`directive()` 用来注册自定义指令（比如 `v-focus`）。plugins 页的例子：

```ts [plugins/my-directive.ts]
export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.vueApp.directive('focus', {
    mounted (el) {
      el.focus()
    },
    getSSRProps (binding, vnode) {
      // 这里可以提供服务端专用的 props
      return {}
    },
  })
})
```

这里有个 Vue 客户端指令没有、Nuxt/SSR 场景必须面对的东西：`getSSRProps`。指令默认只在浏览器有水合后才跑（`mounted` 是客户端钩子），但 SSR 阶段服务端渲染 HTML 时也要能"认识"这个指令，否则模板里写了 `v-focus` 服务端解析不了。`getSSRProps` 就是给服务端返回要渲染到元素上的属性。

## 指令的坑：必须两端都注册

这是这一篇最该记住的一条。plugins 页的 warning：

> If you register a Vue directive, you *must* register it on both client and server side unless you are only using it when rendering one side. If the directive only makes sense from a client side, you can always move it to `~/plugins/my-directive.client.ts` and provide a 'stub' directive for the server in `~/plugins/my-directive.server.ts`.

拆成人话：

- 一个不带后缀的插件（如上面的 `my-directive.ts`）**两端都会跑**，所以指令自动在两端都注册了——这是最省心的写法。
- 如果你把指令插件写成 `.client.ts`（只在客户端），服务端就没有这个指令，模板里用到它会在 SSR 报错。官方的补救是：客户端放真身 `my-directive.client.ts`，同时补一个"桩" `my-directive.server.ts`（内容可以是个空实现的同名指令），让服务端也认得它。
- 只有当你确定"这个指令在某一段渲染里根本不会出现"时，才可以只注册在一端。

这条 warning 的本质，和 271 篇讲 SSR 时反复出现的那句是同一个：**插件默认两端都跑，你要么让它两端都跑，要么自己把另一端补齐。**

## 什么时候不用 vueApp

vueApp 只解决"应用级、一次性"的注册。另外两件事别错用它：

- 想往 Nuxt 实例上挂一个 helper（`$hello`） → 那是 `nuxtApp.provide` / 返回 `{ provide: {...} }`，下一篇（274）讲。
- 想共享一段响应式状态、复用一段逻辑 → 优先自动导入组合式（`app/composables/`），不是 vueApp，也不是插件（270 那条官方 important 已经说过"能组合式就别用 provide"）。

## 自查

- 你要配的是"整个应用一次性"的东西（全局组件/指令/Vue 插件）吗？是 → vueApp；是共享状态或小 helper → 组合式。
- 用 `directive()` 注册了指令？确认它在两端都注册：要么插件不带后缀（两端跑），要么 `.client.ts` + `.server.ts` 桩配套。
- 需要 SSR 阶段的指令属性？实现 `getSSRProps`。
- 装第三方 Vue 插件（`use()`）时，若它只该在浏览器跑，记得给插件文件加 `.client` 后缀。

## 下一篇

下一篇 274 讲 `nuxtApp.provide` 和 `useNuxtApp().$xxx` 的取用规则——往 NuxtApp 上挂全局 helper、以及取出来时那个 `$` 前缀和"模板不解包"的坑。

## 本文依据

- useNuxtApp（`api/composables/use-nuxt-app`）`vueApp` 一节："`vueApp` is the global Vue.js application instance that you can access through `nuxtApp`."；`component()`/`directive()`/`use()` 三个方法的官方说明。
- plugins 目录（`directory-structure/app/plugins`）：
  - Vue Plugins 一节 `vue-gtag.client.ts` 完整示例（`nuxtApp.vueApp.use(VueGtag, {...})` + `trackRouter(useRouter())`）。
  - Vue Directives 一节 `nuxtApp.vueApp.directive('focus', { mounted(el){...}, getSSRProps(binding, vnode){...} })` 示例。
  - warning "If you register a Vue directive, you *must* register it on both client and server side..." 与 `.client.ts`/`.server.ts` stub 的补救说法。
