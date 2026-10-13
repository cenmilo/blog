---
url: /posts/4d1442c9bc66bcffa114518c7f2990cc/
title: 442 - NuxtLink 的 prefetch 策略与代价
date: 2027-03-08T10:00:00+08:00
updated: 2027-03-08T10:00:00+08:00
author: cmdragon

summary:
  摘要：`<NuxtLink>` 的预取默认开着，触发条件是链接进入视口或被滚动到；官方同时写明了它的让步规则和代价——浏览器忙时不取，断网或只有 2g 时直接跳过，而 `prefetch` 取的不只是页面组件，还包括 middleware、layouts 和 payload。两种触发器（`visibility` 与 `interaction`）同时开会重复取同一份资源，官方那句原话是"这大概不是你想要的"。这一篇把四个预取属性、两种触发机制、全局默认值表以及 `custom` 下预取失效的三个坑一次列清。

categories:
  - 前端开发

tags:
  - nuxt4
  - NuxtLink
  - prefetch
  - 预取
  - 性能
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

列表页渲染完，你在网络面板里看到首屏之后又陆续下载了二十多个 JS 文件。没人写这些请求——是 `<NuxtLink>` 干的。

想关掉它、或者换成只在鼠标悬停时取，得先知道这一层的全部规则。147 那篇讲过这个组件的三组属性和内外链接判定，这一篇只讲预取。

## 默认在做什么

两节都写了这句：

> Nuxt automatically includes smart prefetching. That means it detects when a link is visible (by default), either in the viewport or when scrolling and prefetches the JavaScript for those pages so that they are ready when the user clicks the link.

紧接着组件那一节补了让步条件，这句经常被漏掉：

> Nuxt only loads the resources when the browser isn't busy and skips prefetching if your connection is offline or if you only have 2g connection.

三条：浏览器不忙才取、离线不取、只有 2g 不取。所以你在弱网下测不出预取效果，不是配置没生效。

## 四个属性与一条生效顺序

| 属性 | 官方原文要点 |
| --- | --- |
| `prefetch` | 启用时会预取视口内链接的 **middleware、layouts 和 payload**（payload 部分要 `payloadExtraction` 在用）；同时被实验项 `crossOriginPrefetch` 使用 |
| `prefetchOn` | 自定义什么时候预取，可选 `interaction` 与 `visibility`（默认 `visibility`），也可以传对象做完全控制；**只在 `prefetch` 开着且没设 `noPrefetch` 时使用** |
| `noPrefetch` | 关闭预取 |
| `prefetchedClass` | 给已经预取过的链接加一个类 |

第一行是这一篇最有用的一句：**预取取的东西不止页面代码**。`prefetch` 的定义里明确包含中间件和布局，这就是为什么一个链接预取之后，页面里那套 layout 也一起到位了。

关掉的写法官方给了两种等价的：

```vue
<NuxtLink to="/about" no-prefetch>About page not pre-fetched</NuxtLink>

<NuxtLink to="/about" :prefetch="false">About page not pre-fetched</NuxtLink>
```

## 两种触发器的机制不同

`v3.13.0` 之后支持 `prefetch-on` 传字符串或对象：

```vue
<NuxtLink prefetch-on="visibility">进入视口时预取（默认）</NuxtLink>
<NuxtLink prefetch-on="interaction">悬停或获得焦点时预取</NuxtLink>
<NuxtLink :prefetch-on="{ interaction: true }">悬停或获得焦点时预取</NuxtLink>
```

官方对两个值的说明是机制层面的：

- **`visibility`**：链接进入视口时预取，用 Intersection Observer 监测元素与视口的相交，**滚动把它带进视野时触发**。
- **`interaction`**：悬停或聚焦时预取，监听的是 `pointerenter` 和 `focus` 两个事件，在用户表达出交互意图时提前取资源。

然后是一句带感叹号的原文：**"That you probably don't want both enabled!"**，并解释了原因：

> This configuration will observe when the element enters the viewport and also listen for `pointerenter` and `focus` events. This may result in unnecessary resource usage or redundant prefetching, as both triggers can prefetch the same resource under different conditions.

两个触发器会在不同条件下取同一份资源，结果是**资源浪费或重复预取**。性能那一节给的"改成按交互预取"的配置，正是把另一个关掉：

```ts
export default defineNuxtConfig({
  experimental: {
    defaults: {
      nuxtLink: {
        prefetchOn: {
          interaction: true,
          visibility: false,
        },
      },
    },
  },
})
```

## 全局默认值八个键

逐个链接写太累，官方给的位置是 `experimental.defaults.nuxtLink`，默认值原样列出来：

```ts
nuxtLink: {
  componentName: 'NuxtLink',
  externalRelAttribute: 'noopener noreferrer',
  activeClass: 'router-link-active',
  exactActiveClass: 'router-link-exact-active',
  prefetchedClass: undefined, // can be any valid string class name
  trailingSlash: undefined, // can be 'append' or 'remove'
  prefetch: true,
  prefetchOn: { visibility: true },
}
```

预取相关的是最后两行加 `prefetchedClass`。这一节有一条 `::important` 必须一起记住：

> These options will likely be moved elsewhere in the future, such as into `app.config` or into the `app/` directory.

也就是说 `experimental.defaults` 这个位置是过渡形态，将来大概率搬走。

另一种全局做法是自己造一个组件：`defineNuxtLink` 的参数类型里预取相关有三个字段——`prefetch`（是否默认预取）、`prefetchOn`（默认应用哪些策略）、`prefetchedClass`（已预取链接的默认类）。写 `app/components/MyNuxtLink.ts` 导出一次，模板里按文件名用 `<MyNuxtLink />`；`componentName` 只设 Vue DevTools 里显示的内部名，**不改模板里的用法**。

## custom 会让预取整套失效

这是最容易踩的一条。官方在预取那一节里先写了一句：

> When using the `custom` prop, `<NuxtLink>` does not automatically attach prefetch handlers. You can call `prefetch` from the custom slot yourself.

然后又用一条 `::note` 把范围扩大：用了 `custom`，`prefetch`、`prefetchOn`、`prefetchedClass` **都不再自动挂处理器或加类**，要靠插槽给出的 `prefetch`、`prefetched`、`shouldPrefetch` 自己实现。`v4.5` 起插槽里的写法是这样：

```vue
<NuxtLink v-slot="{ href, navigate, prefetch, shouldPrefetch }" to="/about" custom>
  <a
    :href="href"
    @click="navigate"
    @pointerenter="shouldPrefetch('interaction') && prefetch()"
    @focus="shouldPrefetch('interaction') && prefetch()"
  >About page</a>
</NuxtLink>
```

注意 `shouldPrefetch('interaction')` 是**判断**、`prefetch()` 是**执行**，两者都要你写。少了前者，就等于绕过了官方那套让步规则（浏览器忙、离线、2g）。

## 跨域预取与提示标签

`experimental.crossOriginPrefetch: true` 会借助 Speculation Rules API 启用跨域预取，官方的 `prefetch` 属性定义里也点了它一句。另一件相关的 `experimental.prefetchPreloadTags` 在 424 那篇提过（默认关闭）。这两项的行为留给 443 那篇细讲，这里只留一句：**它们都不改上面这四个属性的规则**。

## 自查

- 默认按可见性预取；浏览器忙不取，离线或 2g 直接跳过。
- `prefetch` 取的是 middleware、layouts 和 payload（payload 需 `payloadExtraction`），不止页面 JS。
- `prefetchOn` 有字符串和对象两种写法；`visibility` 走 Intersection Observer，`interaction` 走 `pointerenter` 与 `focus`；两个同开会造成重复预取。
- 全局改在 `experimental.defaults.nuxtLink`（八个默认键，官方说将来可能搬到 `app.config` 或 `app/` 目录），或用 `defineNuxtLink` 自建组件。
- 用 `custom` 就没有任何自动预取：要自己调 `prefetch()`，并最好先用 `shouldPrefetch(...)` 判断。

## 下一篇

下一篇 443 把两个跨域/提示用的实验开关放到一起做对照：`crossOriginPrefetch` 走的是 Speculation Rules API，`prefetchPreloadTags` 决定 `<NuxtImg preload>` 这类提示标签会不会被转发，两者的默认状态一开一关。

## 本文依据

- 智能预取的两句原文（默认检测可见性、视口与滚动、为点击准备好 JS；浏览器不忙才取、离线与 2g 跳过）、`custom` 不自动挂处理器与插槽自行调用 `prefetch`、`no-prefetch` 与 `:prefetch="false"` 两种写法（`api/components/nuxt-link`、`guide/best-practices/performance`）
- `prefetch`/`prefetchOn`/`noPrefetch`/`prefetchedClass` 四个属性定义原文、`prefetch` 含 middleware、layouts、payload 与 `payloadExtraction`、被 `crossOriginPrefetch` 使用、`prefetchOn` 的生效条件句（`api/components/nuxt-link`）
- `prefetch-on` 字符串与对象两种示例、`visibility` 与 `interaction` 各自的机制说明、"That you probably don't want both enabled!" 与重复预取的后果原文、`v4.5` 插槽 `prefetch`/`shouldPrefetch` 示例、`::note` 三项在 `custom` 下不挂处理器不加类（`api/components/nuxt-link`）
- `experimental.defaults.nuxtLink` 八个默认键、`::important` 关于搬家的原文、按交互预取的配置示例（`api/components/nuxt-link`、`guide/best-practices/performance`）
- `defineNuxtLink` 的 `NuxtLinkOptions` 签名与 `prefetch`/`prefetchOn`/`prefetchedClass` 三项说明、按文件名自动导入与 `componentName` 只影响内部名（`api/components/nuxt-link`）
- `crossOriginPrefetch` 与 Speculation Rules API 的配置示例（`api/components/nuxt-link`、`guide/going-further/experimental-features`）
