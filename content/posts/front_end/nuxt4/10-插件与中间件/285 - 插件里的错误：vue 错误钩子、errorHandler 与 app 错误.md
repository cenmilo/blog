---
url: /posts/7f8dd712608660bd2c7715d968de1aa2/
title: 插件里的错误：vue 错误钩子、errorHandler 与 app 错误
date: 2027-01-15T09:00:00+08:00
updated: 2027-01-15T09:00:00+08:00
author: cmdragon

summary:
  摘要：插件跑在应用创建期，它出错会波及很大范围，所以得先分清"哪类错误走哪条路"。官方 error-handling 页给了三个接收点：vue:error 钩子（冒泡到顶层的 Vue 错误，基于 onErrorCaptured）、vueApp.config.errorHandler（全局兜底、连已被处理的也收）、app:error 钩子（启动期错误，明确包含"运行 Nuxt 插件"这一步）。这一篇把这三个接到插件语境里讲清，并落实那句 important——插件一旦抛错，在你清除错误之前它不会重跑，用依赖插件的东西（$route、useRouter）前要当心。

categories:
  - 前端开发

tags:
  - nuxt4
  - 错误处理
  - vue:error
  - app:error
  - 插件
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

插件在"应用创建的那一刻"就跑（270）。这个位置很特殊：它比页面组件早、又两端都跑，所以它出错的影响面比一个组件里出错大得多。要处理插件相关错误，先得知道官方给了**三个接收点**，各接哪类错。

## 接收点一：vue:error 钩子

官方 "Vue Errors" 一节：

> You can hook into Vue errors using `onErrorCaptured`.
> In addition, Nuxt provides a `vue:error` hook that will be called if any errors propagate up to the top level.

也就是说：组件级的错误先走 Vue 的 `onErrorCaptured`；没被接住、冒泡到顶层的错误，Nuxt 用 `vue:error` 这个钩子告诉你。它本质还是基于 `onErrorCaptured`（官方 note："the `vue:error` hook is based on `onErrorCaptured` lifecycle hook"）。

在插件里订阅它就是 275 讲过的 `nuxtApp.hook`：

```ts [plugins/error-handler.ts]
export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.hook('vue:error', (error, instance, info) => {
    // 处理错误，例如上报到某个服务
  })
})
```

参数是 `error, instance, info`（275 表里 `vue:error` 那行的 Arguments 就是这个三元组）。

## 接收点二：vueApp.config.errorHandler（全局兜底）

如果你用错误上报服务（Sentry 那类），官方给了更靠前的全局口子：

> If you are using an error reporting framework, you can provide a global handler through `vueApp.config.errorHandler`. It will receive all Vue errors, even if they are handled.

关键在最后半句 **"It will receive all Vue errors, even if they are handled."**——`errorHandler` 连"已经被 `onErrorCaptured` 处理掉的错误"都能收到，比 `vue:error` 覆盖面更大。官方在同一个插件示例里把两条写法并列给了：

```ts [plugins/error-handler.ts]
export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.vueApp.config.errorHandler = (error, instance, info) => {
    // handle error, e.g. report to a service
  }

  // 或者也可以
  nuxtApp.hook('vue:error', (error, instance, info) => {
    // handle error, e.g. report to a service
  })
})
```

注意 `errorHandler` 是通过 `nuxtApp.vueApp.config` 设的——这就把 273（vueApp）和错误处理连起来了：vueApp 除了挂指令/组件/插件，`vueApp.config` 也是配全局错误处理的地方。

`vue:error` 和 `errorHandler` 的区别记一句：**要"连已处理错误也上报" → errorHandler；要"只响应冒泡到顶层的错误" → vue:error。**

## 接收点三：app:error（启动期错误，含插件抛错）

前两个接的是 Vue 层错误。插件在"启动阶段"抛的错，走另一条：

> Nuxt will call the `app:error` hook if there are any errors in starting your Nuxt application.

官方把"启动期"包含哪些步骤列了出来，第一条就是插件：

> - running Nuxt plugins
> - processing `app:created` and `app:beforeMount` hooks
> - rendering your Vue app to HTML (during SSR)
> - mounting the app (on client-side), though you should handle this case with `onErrorCaptured` or with `vue:error`
> - processing the `app:mounted` hook

所以：**插件本身抛错，会触发 `app:error`**（它在"running Nuxt plugins"那一步）。要接住启动期错误，就在更早的地方订阅 `app:error`。这也和 `app:error` 在 275 表里的定位对上（Server & Client 都触发、参数 `err`）。

官方还专门建议：自定义错误优先用 `onErrorCaptured`（页面/组件 setup 里）或 `vue:error`（插件里配置）——原话：

> For custom errors we highly recommend using `onErrorCaptured` composable that can be called in a page/component setup function or `vue:error` runtime nuxt hook that can be configured in a nuxt plugin.

## 最该记住的一条：插件抛错后，不清错误就不会重跑

这条 important 是插件语境里最容易翻车的，官方写得很直接：

> Make sure to check before using anything dependent on Nuxt plugins, such as `$route` or `useRouter`, as if a plugin threw an error, then it won't be re-run until you clear the error.

拆开：一个插件如果抛了错，**在你清除这个错误之前，它不会被重新执行**。后果是——凡是依赖插件产出的东西（`$route`、`useRouter`、或某个 `provide` 出去的 `$helper`），在那个插件"抛错但没清、没重跑"的状态下都可能是坏的/缺失的。所以处理错误页/错误态时，别默认这些依赖还在。

顺带把错误页那条链接上（234 已讲机制，这里补错误侧视角）：致命错误（服务端未处理错误、或客户端 `fatal: true` 创建的）会渲染全屏错误页或返回 JSON；清除用 `clearError`，它可带一个要跳转的路径（"if you want to navigate to a 'safe' page"）——而 `clearError` 正是让"卡住的插件能重来"的那个动作。

## 一张分流表

| 错误来自 | 走哪个接收点 | 备注 |
| --- | --- | --- |
| 组件内、被接住的 Vue 错误 | `onErrorCaptured` | 组件/页面 setup 里 |
| 冒泡到顶层的 Vue 错误 | `nuxtApp.hook('vue:error', ...)` | 基于 onErrorCaptured |
| 所有 Vue 错误（含已处理） | `nuxtApp.vueApp.config.errorHandler` | 上报服务用 |
| 插件运行/启动期抛错 | `app:error` 钩子 | "running Nuxt plugins" 归这 |
| 服务端 Nitro 错误 | 不能自定义服务端处理 | 官方：只能渲染错误页 |

（最后一行来自官方 "Nitro Server Errors"："You cannot currently define a server-side handler for these errors, but can render an error page."）

## 自查

- 插件自己抛错 → 触发 `app:error`，且"不 clearError 不重跑"；依赖该插件的 `$route`/`useRouter`/`$helper` 此时不可信。
- 想连已被处理的 Vue 错误也上报 → `vueApp.config.errorHandler`，不是 `vue:error`。
- 只关心冒泡到顶层的错误/做兜底处理 → `vue:error`（`error, instance, info`）。
- 客户端组件级错误 → `onErrorCaptured`；官方明确推荐它或 `vue:error` 作为自定义错误的主路。
- 服务端 Nitro 错误没法在服务端定义 handler，只能靠错误页。

## 下一篇

下一篇 286 讲官方那页插件最佳实践：能组合就别用插件、插件别堆多（跑在水合期）、异步插件开 parallel。

## 本文依据

- Error Handling（`getting-started/error-handling`）：Vue Errors 一节（`onErrorCaptured`、`vue:error` hook "called if any errors propagate up to the top level"、`vueApp.config.errorHandler` "will receive all Vue errors, even if they are handled"、`plugins/error-handler.ts` 两种写法、note "vue:error is based on onErrorCaptured"）；Startup Errors（`app:error` 与"running Nuxt plugins"等五条清单）；Nitro Server Errors（不能自定义服务端 handler、只能渲染错误页）；"we highly recommend using onErrorCaptured ... or vue:error ... in a nuxt plugin"；important "if a plugin threw an error, then it won't be re-run until you clear the error"；clearError 可带 redirect。
- useNuxtApp（`api/composables/use-nuxt-app`）：`hook` 用法、`vueApp` 一节（vueApp 是应用实例）。
- Lifecycle Hooks（`api/advanced/hooks`）：`vue:error`（`err, target, info`）、`app:error`（`err`，Server & Client）两行。
