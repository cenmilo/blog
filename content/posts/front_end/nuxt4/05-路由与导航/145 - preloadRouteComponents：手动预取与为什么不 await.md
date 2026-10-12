---
url: /posts/718999952499fc9d3a58bbcd20cc4096/
title: preloadRouteComponents：手动预取与为什么不 await
date: 2026-11-29T10:00:00+08:00
updated: 2026-11-29T10:00:00+08:00
author: cmdragon

summary:
  摘要：官方给这一页的定位句是："**preloadRouteComponents allows you to manually preload individual pages in your Nuxt app.**"，紧接着是机制与收益："Preloading routes loads **the components of a given route that the user might navigate to in future**. This ensures that the components are available earlier and **less likely to block the navigation**, improving performance."。但官方同时挂了一条 tip 把它自己大部分使用场景取消掉了：**"Nuxt already automatically preloads the necessary routes if you're using the `NuxtLink` component."**——所以这个 API 服务对象是"**不用 `<NuxtLink>` 的那部分导航**"，官方示例正是这个：在 `navigateTo` 之前先预热。示例里最有信息量的是那两行注释："**// we don't await this async function, to avoid blocking rendering // this component's setup function**"——这与 124 里 `navigateTo` "**必须** await 或 return"形成一组对照：**跳转的调用不能不等，预取的调用不能等**。还有一条硬边界放在 note 里："**On server, `preloadRouteComponents` will have no effect.**"。同源的兄弟 API `preloadComponents` 形状不同：它预加载的是"**individual components that have been registered globally**"，且"**You must use the Pascal-cased version of the component name**"，官方示例用 `await preloadComponents('MyGlobalComponent')` 与数组形式 `await preloadComponents(['MyGlobalComponent1', 'MyGlobalComponent2'])`，同样"On server, `preloadComponents` will have no effect."。至于 `<NuxtLink>` 的自动预取到底做了什么，官方在 props 里写得比这一页更宽：`prefetch` "**When enabled will prefetch middleware, layouts and payloads (when using payloadExtraction) of links in the viewport.**"——不止组件。

categories:
  - 前端开发

tags:
  - nuxt4
  - 预加载
  - 性能
  - navigateTo
  - NuxtLink

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


登录成功后跳 `/dashboard`：那段等待里浏览器还要先下载 dashboard 的 JS chunk。能不能在等认证接口返回的时候就把 chunk 提前拿下来？官方给的工具是 `preloadRouteComponents`，而它有一行注释比函数名更值得琢磨。这一篇讲清它到底预取什么、以及为什么官方偏偏在这里**不写 `await`**。

## 定位句：手动预取"单个页面"

官方原文两层：

> preloadRouteComponents allows you to manually preload individual pages in your Nuxt app.

> Preloading routes loads the components of a given route that the user might navigate to in future. This ensures that the components are available earlier and less likely to block the navigation, improving performance.

拆开看：

| 说法 | 含义 |
| --- | --- |
| **manually** | 不是靠观察可见性触发的，是你主动点的 |
| **individual pages** | 一次给一条路由，不是批量策略 |
| **loads the components of a given route** | 加载对象是**该路由的组件**（分块代码） |
| **that the user might navigate to in future** | 赌的是"接下来可能去"，赌错了就是浪费一次下载 |
| **less likely to block the navigation** | 收益表述得很克制：不是"变快"，而是"别挡住导航" |

最后那条值得记：文件式路由天然按页面切代码（110），所以导航本质上要等一次 chunk 下载。预取做的事就是把这次下载提前到你还有空闲的时候。

## 那条 tip 先把自己取消了一遍

紧接着官方放了一条 rocket 图标的 tip：

> Nuxt already automatically preloads the necessary routes if you're using the `NuxtLink` component.

所以第一个判断不是"怎么写这个函数"，而是"**我需要写它吗**"：

| 你要去的那条路由 | 需不需要手动预取 |
| --- | --- |
| 页面上有 `<NuxtLink to="...">`，链接进了视口 | **不需要**，官方自动做 |
| 该链接显式关了预取（`no-prefetch` / `:prefetch="false"`） | 需要时才手动 |
| 靠 `navigateTo` 程序化跳转（登录、提交表单后） | **这就是官方示例的场景** |
| 你自己用原生 `<a>` 或 `custom` 插槽渲染的链接 | `<a>` 完全不进这套机制；`custom` 见下一节 |

`<NuxtLink>` 的自动预取具体做什么，官方在 `prefetch` 那条 prop 里写得比这一页宽：

> `prefetch`: When enabled will prefetch **middleware, layouts and payloads** (when using payloadExtraction) of links in the viewport.

也就是说自动预取的范围比"路由组件"更大（中间件、布局、以及开启 payloadExtraction 时的 payload）。手动 `preloadRouteComponents` 只覆盖组件那一层——**别把它当成"`<NuxtLink>` 预取的等价手动版"**。

## 官方示例：为什么这里不写 await

```ts
// we don't await this async function, to avoid blocking rendering
// this component's setup function
preloadRouteComponents('/dashboard')

const submit = async () => {
  const results = await $fetch('/api/authentication')

  if (results.token) {
    await navigateTo('/dashboard')
  }
}
```

官方注释是这一页最重要的两行：

> we don't await this async function, to avoid blocking rendering this component's setup function

把它和 124 那条规则并排看，这个系列的"await 观"就统一了：

| 调用 | 官方要求 | 理由 |
| --- | --- | --- |
| `navigateTo('/dashboard')` | **必须 await 或 return**（124） | 它是导航指令，不 await 就是"发了不确保送出去"，且官方明确说中间件里不返回会出问题 |
| `preloadRouteComponents('/dashboard')` | **故意不 await** | await 会把"下载 dashboard 的 chunk"这件事插进当前组件的渲染路径上，当前页要等它才画出来 |

一句话记法：**跳转要等结果，预取不能等结果。**

示例的时序也是刻意的：预取放在 `submit` **之前**、`$fetch` 那次请求之前。这样在用户还在等认证接口返回时，dashboard 的代码已经在后台下完了。官方把这一段写成顶层形式（`const submit = async () => {...}`），形态上是组件的 setup 里；`preloadRouteComponents` 属于 `api/utils` 那一类（自动导入范围见 150 起那篇），示例没有写 import，也没写参数类型的其他形态。

## 服务器上不生效

`::note` 只有一句：

> On server, `preloadRouteComponents` will have no effect.

这条对代码组织有实际影响：**服务端渲染路径上调它不会报错，但也什么都不会发生**。所以如果你在通用组件（ SSR + 客户端都跑）里预取，别指望首屏 SSR 时就把下一页带下来；真正的预取发生在浏览器里。想验证很简单：在服务端 `setup` 里调一次，Network 面板不会多任何 chunk。

## 别和 preloadComponents 混

同一份源码文件（`app/composables/preload.ts`）里还有个兄弟 API，形状完全不同：

```ts
await preloadComponents('MyGlobalComponent')

await preloadComponents(['MyGlobalComponent1', 'MyGlobalComponent2'])
```

官方对它的描述：

> Preloading components loads components that your page will need very soon, which you want to start loading early in rendering lifecycle.
>
> Use `preloadComponents` to manually preload individual components **that have been registered globally** in your Nuxt app. By default Nuxt registers these as async components. **You must use the Pascal-cased version of the component name.**

| | `preloadRouteComponents` | `preloadComponents` |
| --- | --- | --- |
| 预取对象 | 一条**路由**的组件 | 全局注册的**某个/某几个组件** |
| 参数 | 路由路径（官方示例给 `'/dashboard'`） | 组件名，**PascalCase**；可以是数组 |
| 官方示例是否 await | **不 await**（附注释） | **await** |
| 服务器 | 无效果 | 无效果 |
| 用途差别 | 为"下一次导航"热身 | 为"这一页马上要用"热身 |

两者的服务器行为一致：`"On server, `preloadComponents` will have no effect."`

`preloadComponents` 那句 "your page will need very soon, which you want to start loading early in **rendering lifecycle**" 与路由预取的 "**in future**" 是两组时间尺度：一个是本页渲染早期，一个是未来某次导航。

## 想手动接自动预取：custom 插槽那条路

`<NuxtLink>` 用了 `custom` 之后，官方在两处写了同一件事：

> When using the `custom` prop, `<NuxtLink>` does not automatically attach prefetch handlers. You can call `prefetch` from the custom slot yourself.

并给出 v4.5 形态的例子：

```vue
<template>
  <NuxtLink
    v-slot="{ href, navigate, prefetch, shouldPrefetch }"
    to="/about"
    custom
  >
    <a
      :href="href"
      @click="navigate"
      @pointerenter="shouldPrefetch('interaction') && prefetch()"
      @focus="shouldPrefetch('interaction') && prefetch()"
    >
      About page
    </a>
  </NuxtLink>
</template>
```

props 那节的 note 也补了一句：

> With the `custom` prop, `prefetch`, `prefetchOn` and `prefetchedClass` do not attach handlers or classes automatically. Use the custom slot's `prefetch`, `prefetched` and `shouldPrefetch` values to implement this behavior.

于是三种"自己控制预取"的手感分清楚了：

| 想要的 | 用 |
| --- | --- |
| 保留 `<NuxtLink>` 的 `<a>` 渲染，只在别的时机预取 | `prefetchOn`（`visibility` / `interaction`） |
| 已经用 `custom` 完全自渲染 | 插槽里的 `prefetch()` + `shouldPrefetch()` |
| 根本没用 `<NuxtLink>`（程序化跳转） | `preloadRouteComponents(path)`，不 await |

自动预取的钩子也存在：官方 App Hooks 表里有 `link:prefetch`（参数 `to`，Client）："Called when a `<NuxtLink>` is observed to be prefetched."——排查"到底预取没预取"时可以用它，比猜 Network 面板确定。

## 什么时候真的该写这一行

我的判断标准（前三条来自文档，第四条是使用取向）：

1. **这条路由是程序化跳过去的**，不是点 `<NuxtLink>` 跳的 → 写。
2. **这个跳转发生在一次网络往返之后**（登录、提交、支付回跳）→ 收益最大：请求等待期正好下载 chunk。
3. **目标页组件体积不小**，且这条路径是主流程 → 值得写。
4. 只是"提前热个身"到处塞 → 别。官方对范围的描述就是 `individual pages`，一次一条；塞多了就是在跟浏览器抢带宽，而 `<NuxtLink>` 的自动预取本身还带着"浏览器不忙才加载、离线或 2g 直接跳过"的节制（147 会展开）。

## 自查

1. 官方对 `preloadRouteComponents` 的定位句是什么？它加载的对象、以及收益的原话分别怎么说？
2. 哪条 tip 让大部分场景不需要手动调用它？那"需要手动"的典型场景是什么？
3. 官方示例为什么不 `await`？这与 `navigateTo` 的 await 规则怎么并存？
4. 服务端调用它会怎样？这条边界对 SSR 首屏有什么含义？
5. `preloadComponents` 与 `preloadRouteComponents` 在参数形态、是否 await、预取对象上分别有什么区别？
6. `custom` 用了之后自动预取会怎样？官方给的补救办法用到插槽里的哪几个值？
7. 有一个钩子能确认"`<NuxtLink>` 被预取了"，是哪个？它的参数与环境？

## 下一篇

146 讲 `useLoadingIndicator`：`isLoading`、`progress`、`isLoadingForced` 三个状态，`start`/`finish`/`clear`/`force` 四个方法，以及 `duration` 与 `throttle` 两个节流值怎么配合顶部的进度条。

## 本文依据

- `https://nuxt.com/docs/4.x/api/utils/preload-route-components`
  - 原文（标题句与描述）："preloadRouteComponents allows you to manually preload individual pages in your Nuxt app."；"Preloading routes loads the components of a given route that the user might navigate to in future. This ensures that the components are available earlier and less likely to block the navigation, improving performance."
  - 原文（tip）："Nuxt already automatically preloads the necessary routes if you're using the `NuxtLink` component."
  - 原文（Example）："Preload a route when using `navigateTo`." + 代码 "// we don't await this async function, to avoid blocking rendering \n // this component's setup function \n preloadRouteComponents('/dashboard') \n\n const submit = async () => { const results = await $fetch('/api/authentication'); if (results.token) { await navigateTo('/dashboard') } }"
  - 原文（note）："On server, `preloadRouteComponents` will have no effect."
  - 原文（Source 链接）：源码位于 `packages/nuxt/src/app/composables/preload.ts`；页尾 read-more 指向 `https://nuxt.com/docs/4.x/api/components/nuxt-link` 与 `https://nuxt.com/docs/4.x/api/utils/navigate-to`
- `https://nuxt.com/docs/4.x/api/utils/preload-components`
  - 原文："Nuxt provides utilities to give you control over preloading components."；"Preloading components loads components that your page will need very soon, which you want to start loading early in rendering lifecycle. This ensures they are available earlier and are less likely to block the page's render, improving performance."；"Use `preloadComponents` to manually preload individual components that have been registered globally in your Nuxt app. By default Nuxt registers these as async components. You must use the Pascal-cased version of the component name."
  - 原文示例：`await preloadComponents('MyGlobalComponent')`；`await preloadComponents(['MyGlobalComponent1', 'MyGlobalComponent2'])`；note "On server, `preloadComponents` will have no effect."
- `https://nuxt.com/docs/4.x/api/components/nuxt-link`
  - 原文（Prefetch Links 开篇）："Nuxt automatically includes smart prefetching. That means it detects when a link is visible (by default), either in the viewport or when scrolling and prefetches the JavaScript for those pages so that they are ready when the user clicks the link. Nuxt only loads the resources when the browser isn't busy and skips prefetching if your connection is offline or if you only have 2g connection."
  - 原文（custom 与 prefetch）："When using the `custom` prop, `<NuxtLink>` does not automatically attach prefetch handlers. You can call `prefetch` from the custom slot yourself."；（note）"With the `custom` prop, `prefetch`, `prefetchOn` and `prefetchedClass` do not attach handlers or classes automatically. Use the custom slot's `prefetch`, `prefetched` and `shouldPrefetch` values to implement this behavior."
  - 原文（v4.5 示例）：`v-slot="{ href, navigate, prefetch, shouldPrefetch }"` 配 `@pointerenter="shouldPrefetch('interaction') && prefetch()"` 与 `@focus="shouldPrefetch('interaction') && prefetch()"`
  - 原文（`prefetch` prop 定义）："When enabled will prefetch middleware, layouts and payloads (when using payloadExtraction) of links in the viewport. Used by the experimental crossOriginPrefetch config."
  - 原文（关闭预取）：`<NuxtLink to="/about" no-prefetch>` 与 `<NuxtLink to="/about" :prefetch="false">`
- `https://nuxt.com/docs/4.x/api/advanced/hooks`
  - 原文（App Hooks 表）："`link:prefetch` | `to` | Client | Called when a `<NuxtLink>` is observed to be prefetched."
- `https://nuxt.com/docs/4.x/api/utils/navigate-to`
  - 对照用（见 124）：`navigateTo` 必须 await 或 return 的规则；本页通过 read-more 指向它
- `https://nuxt.com/docs/4.x/guide/going-further/experimental-features`
  - 原文（`payloadExtraction` 与 `crossOriginPrefetch` 两处条目，被本页 props 定义引用）：`prefetch` 的定义里括号写明 "(when using payloadExtraction)"，并链接到 "the experimental crossOriginPrefetch config"
