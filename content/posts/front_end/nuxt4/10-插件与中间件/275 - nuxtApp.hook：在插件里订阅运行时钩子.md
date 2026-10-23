---
url: /posts/43d634f3ab12024e5549d334ae23e17a/
title: 275 - nuxtApp.hook：在插件里订阅运行时钩子
date: 2027-01-11T11:00:00+08:00
updated: 2027-01-11T11:00:00+08:00
author: cmdragon

summary:
  摘要：nuxtApp 上除了 provide、vueApp，还有个 hook(name, cb)——在渲染生命周期的某个点插一段自己的逻辑。这一篇讲它在哪几个位置能用（组合式与插件）、官方给的两个示例（page:start 与 vue:error），再把 App Hooks (Runtime) 那张表按"哪一侧触发"拆成服务端、客户端、两端三堆，挑最常用的几个说明参数与时机。要点提醒：钩子名是固定的、vue:setup 必须同步，以及对象写法能用 hooks 键静态预加载、不必操心注册顺序。

categories:
  - 前端开发

tags:
  - nuxt4
  - hook
  - 运行时钩子
  - nuxtApp
  - 生命周期
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


页面从"服务端渲染"到"浏览器可交互"之间，有一串固定发生的时刻（应用创建了、渲染完了、挂载完了、页面开始加载了、出错了……）。Nuxt 在这些时刻上挂了名字，让你能在旁边插一段代码，这就是**运行时钩子（runtime hooks）**。`nuxtApp.hook(name, cb)` 就是订阅它的函数。

## hook 用在哪、长什么样

use-nuxt-app 页对 `hook` 的说明：

> Hooks available in `nuxtApp` allows you to customize the runtime aspects of your Nuxt application. You can use runtime hooks in Vue.js composables and Nuxt plugins to hook into the rendering lifecycle.
> `hook` function is useful for adding custom logic by hooking into the rendering lifecycle at a specific point. `hook` function is mostly used when creating Nuxt plugins.

两个信息：可用位置是**组合式与插件**；最常见的落点是**写插件时**。官方示例（`app/plugins/test.ts`）：

```ts [app/plugins/test.ts]
export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.hook('page:start', () => {
    /* 你的代码 */
  })
  nuxtApp.hook('vue:error', (..._args) => {
    console.log('vue:error')
    // if (import.meta.client) {
    //   console.log(..._args)
    // }
  })
})
```

这里订了两个钩子：`page:start`（`NuxtPage` 里 Suspense 处于 pending 时触发）和 `vue:error`（错误冒泡到根组件时触发，参数是 `err, target, info`）。注意官方那段注释：客户端才想 `console.log(..._args)` 时要用 `import.meta.client` 挡一下——因为这些钩子有些两端都会跑，参数结构两端未必一致。

## 有哪些钩子：按"哪一侧触发"分

官方 App Hooks (Runtime) 表里每条都标了 Environment（Server & Client / 只 Server / 只 Client）。挑常用、且和学生场景贴得近的几个，按侧分组：

**两端都有（Server & Client）**

| 钩子 | 参数 | 时机 |
| --- | --- | --- |
| `app:created` | `vueApp` | 初始 `vueApp` 实例创建时 |
| `app:error` | `err` | 发生致命错误时 |
| `app:error:cleared` | `{ redirect? }` | （表里描述同 app:error 那句）错误相关 |
| `vue:setup` | - | Nuxt 根 setup 初始化时；**这个回调必须同步** |
| `vue:error` | `err, target, info` | vue 错误冒泡到根组件时 |

**只服务端（Server）**

| 钩子 | 参数 | 时机 |
| --- | --- | --- |
| `app:rendered` | `renderContext` | SSR 渲染完成时 |
| `app:redirected` | - | SSR 重定向之前 |

**只客户端（Client）**

| 钩子 | 参数 | 时机 |
| --- | --- | --- |
| `app:beforeMount` | `vueApp` | 挂载应用之前，仅客户端 |
| `app:mounted` | `vueApp` | Vue 应用在浏览器初始化并挂载完成 |
| `app:suspense:resolve` | `appComponent` | Suspense resolved 事件 |
| `app:manifest:update` | `{ id, timestamp }` | 检测到应用有新版本时 |
| `app:data:refresh` | `keys?` | `refreshNuxtData` 被调用时 |
| `link:prefetch` | `to` | `<NuxtLink>` 被观察到将预取时 |
| `page:start` / `page:finish` | `pageComponent?` | `NuxtPage` 内 Suspense pending / resolved |
| `page:loading:start` / `page:loading:end` | - | 路由导航开始/结束（详见下方那条坑） |
| `page:transition:finish` | `pageComponent?` | 页面过渡 onAfterLeave 之后 |
| `dev:ssr-logs` | `logs` | 服务端日志透传到客户端时（需 `features.devLogs`） |
| `page:view-transition:start` | `transition` | `document.startViewTransition` 调用后（需实验性 viewTransition） |

这张表是"应用运行时钩子"，别和**构建期钩子**（`Nuxt Hooks (Build Time)` 那张，`pages:extend`、`build:done` 那些）混——那是模块作者扩展构建用的，不在 `nuxtApp.hook` 上订。

几个要单独记的时机细节，都来自表格描述本身：

- `vue:setup` 那句明确写了 "This callback must be synchronous."——它必须同步，别在里面 await。
- `page:loading:start` 描述里有一句很实用："May fire without the page component's `setup()` re-running if the page is reused (e.g. with a static `key` in `definePageMeta`)."——页面被复用（比如固定 key）时它会触发、但页面 setup 不重跑。
- `app:manifest:update` 就是"检测到新版本"，配合 281 篇的 `reloadNuxtApp` 是常见的"发现新包就提示重载"场景。

完整清单官方让你查源文件（表头那句 "Check the app source code for all available hooks"），这张文档表是节选口径；要精确到每一个，看 `packages/nuxt/src/app/nuxt.ts`。

## 对象写法的 hooks 键：不用管顺序

271 篇埋的那个伏笔在这里收成。用函数写法 `nuxtApp.hook('app:created', ...)` 有个顺序顾虑：这个钩子可能在你注册之前就已经被 Nuxt 广播过了。对象写法把钩子写在 `hooks:` 键里，plugins 页那句 note 解释它为什么能绕开顺序：

> Nuxt does statically pre-load any hook listeners when using object-syntax, allowing you to define hooks without needing to worry about order of plugin registration.

```ts
export default defineNuxtPlugin({
  name: 'my-plugin',
  hooks: {
    'app:created' () {
      // ...
    },
  },
})
```

也就是：想订 `app:created` 这类"很早就会触发"的钩子，用对象写法的 `hooks` 键更稳，因为 Nuxt 在构建阶段就把"这个插件订了这个钩子"读到了、提前挂好，不依赖插件运行时注册的先后。

## 自查

- `nuxtApp.hook` 主要用在插件（官方明说）；组合式里也能用，但要拿到 nuxtApp。
- 参数结构两端不同的钩子（如 `vue:error`），客户端逻辑用 `import.meta.client` 包起来再打印/处理。
- `vue:setup` 必须同步；别在它里面 await。
- 订的是应用运行时钩子（`app:*`/`page:*`/`vue:*`/`link:prefetch`/`dev:ssr-logs`），别和构建期 `Nuxt Hooks`（`pages:extend` 那些）混。
- 要订很早触发的钩子（`app:created` 等），优先对象写法 `hooks` 键，省掉注册顺序的坑。

## 下一篇

下一篇 276 讲 `nuxtApp.callHook`——反过来，由你去"触发"一个钩子、并等它的监听跑完（返回 Promise）；顺带澄清一个常见误解：文档里没有 `nuxtApp.callAsync` 这个东西。
