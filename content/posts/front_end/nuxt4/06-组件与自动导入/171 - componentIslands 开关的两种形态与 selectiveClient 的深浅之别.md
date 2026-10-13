---
url: /posts/e2d3ef524ace23359c17ded40916b2ce/
title: 171 - componentIslands 开关的两种形态与 selectiveClient 的深浅之别
date: 2026-12-08T09:00:00+08:00
updated: 2026-12-08T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：先给结论：**多数项目一行都不用配**。官方原话是"Component islands are controlled by `experimental.componentIslands`. **The default value is `'auto'`, which enables the feature automatically as soon as your app contains a server component or island, so in most cases you do not need any configuration.**"**需要显式配的只有两种情况**："Set the option explicitly **if you want remote islands or selective client hydration**"，即对象形态 `componentIslands: { selectiveClient: true, /* or 'deep', to enable `nuxt-client` */ remoteIsland: false /* allow rendering islands from a remote source */ }`。`selectiveClient` 的两档差别官方写成一句："Setting `selectiveClient: 'deep'` **additionally allows passing slots to `nuxt-client` components. Those slots are rendered on the server and are not interactive on the client.**"`nuxt-client` 的机制句是："The component marked with `nuxt-client` is **server-rendered as part of the island, then hydrated by the main client app. Only its chunk is shipped to the client; the rest of the island remains static.**"，而启用条件是"**This requires `experimental.componentIslands.selectiveClient` to be enabled**"。一条会静默失败的警告必须记住："**Use `nuxt-client` only on local `.vue` SFCs. Built-ins like `<NuxtLink>` skip the islands transform. After client navigation you may see `Failed to locate Teleport target`, or the link disappears with no error. Wrap the built-in in your own `.vue` file and put `nuxt-client` on that wrapper.**"远程岛屿另有前提："**Remote islands need `experimental.componentIslands` to be `'local+remote'` in your `nuxt.config`.**"，以及一段官方称为"inherently dangerous"的警告，收尾句是"**Even with `dangerouslyLoadClientComponents` disabled (the default), you are still trusting the remote server's HTML output.**"还有一个流式渲染的连带：岛屿 teleport 靠一段水合前的内联脚本归位，"**The exception is apps built with `features.noScripts` and island components, which fall back to the buffered renderer since the relocation script cannot run.**"

categories:
  - 前端开发

tags:
  - nuxt4
  - 岛屿
  - 实验特性
  - nuxt-client
  - 远程岛屿

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


我一开始以为岛屿要手动开开关，结果 `.server.vue` 一写就生效。真正的开关其实只管两件进阶能力——**让岛屿里某个组件可交互**、**从别的站点渲染岛屿**。这两件各有硬条件，其中一条是静默失效。

## 默认值让大多数人不需要配置

官方那句定调很明确：

> Component islands are controlled by `experimental.componentIslands`. **The default value is `'auto'`, which enables the feature automatically as soon as your app contains a server component or island, so in most cases you do not need any configuration.**

**"as soon as your app contains a server component or island"** 这个触发条件解释了那个现象：写 `.server.vue` 就等于"包含"，于是特性自己开了。

需要显式配的只有两种：

> Set the option explicitly **if you want remote islands or selective client hydration**:

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    componentIslands: {
      selectiveClient: true, // or 'deep', to enable `nuxt-client`
      remoteIsland: false, // allow rendering islands from a remote source
    },
  },
})
```

### 两节的措辞对不上，我按哪一处读

| 页面 | 它给的形态 |
| --- | --- |
| `guide/concepts/server-components`（上一段） | 默认 `'auto'`；对象形态两个键 `selectiveClient`、`remoteIsland` |
| `guide/going-further/experimental-features` | "Enables experimental component islands support **with `<NuxtIsland>` and `.island.vue` files**." + `componentIslands: true, // false or 'local+remote'` |
| `api/nuxt-config` | **没有 `componentIslands` 条目**（那一节的 `experimental` 相关清单里没有它） |

三处的差异不是文字风格：**布尔写法那一版没有 `'auto'`，也没有对象形态；而 `'local+remote'` 只出现在那一版里**。我的处理是：
- 以"特性页"（server-components）为主读默认值与对象形态，因为它给的说明最完整；
- `'local+remote'` 这个字符串值照采信，因为远程岛屿那一节也用同一个值提了要求（下面第五节）；
- **`.island.vue` 文件后缀这个说法只出现在实验特性页**，而目录页与特性页讲岛屿注册时用的是 `~/components/islands/` 目录——所以我不把 `.island.vue` 当成有两条来源支持的写法。

这种"同一开关两处文档不同"的情况，我只按"能验证的那部分"写代码，验证方法见最后。

## `selectiveClient`：true 与 deep 的分别

先给 `nuxt-client` 的存在理由与前提：

> An island is static by default, but you can hydrate individual components inside it **by adding the `nuxt-client` attribute**. **This requires `experimental.componentIslands.selectiveClient` to be enabled.**

官方的写法示例：

```vue [app/components/ServerWithClient.server.vue]
<template>
  <div>
    <HighlightedMarkdown markdown="# Headline" />
    <!-- Counter will be loaded and hydrated client-side -->
    <Counter
      nuxt-client
      :count="5"
    />
  </div>
</template>
```

机制句我认为这是整节最值的一句，因为它说明"省了什么、没省什么"：

> The component marked with `nuxt-client` is server-rendered as part of the island, then hydrated by the main client app. **Only its chunk is shipped to the client; the rest of the island remains static.**

两档的差别官方只给了 `deep` 的增量：

> Setting `selectiveClient: 'deep'` **additionally allows passing slots to `nuxt-client` components. Those slots are rendered on the server and are not interactive on the client.**

排成表：

| 值 | 解锁 | 插槽 |
| --- | --- | --- |
| 不启用 | 岛屿整体静态 | — |
| `true` | `nuxt-client` 组件可水合 | 不能给 `nuxt-client` 组件传插槽 |
| `'deep'` | 同上 + **允许**传插槽给 `nuxt-client` 组件 | 那些插槽**服务端渲染、客户端不可交互** |

**`'deep'` 那半句里的"not interactive"很容易看漏**：它解锁的是"能传插槽"，不是"插槽里的东西可交互"。而岛屿普通插槽的语义相反，官方写得很清楚：

> Slots can be passed to an island component if declared in the island. Slot content is provided by the parent, so it belongs to the main client app and **is** interactive (it is wrapped in a `<div>` with `display: contents;`).

两组对照：**岛屿自己的插槽（父级提供）可交互；给 `nuxt-client` 组件的插槽不可交互**。

## 一条会静默失败的警告：`nuxt-client` 只能用本地 SFC

原文我整段抄，因为两种症状都不报错：

> **Use `nuxt-client` only on local `.vue` SFCs. Built-ins like `<NuxtLink>` skip the islands transform. After client navigation you may see `Failed to locate Teleport target`, or the link disappears with no error.** Wrap the built-in in your own `.vue` file and put `nuxt-client` on that wrapper.

拆成三行：

| 要点 | 内容 |
| --- | --- |
| 适用范围 | 只有本地 `.vue` SFC |
| 为什么内置组件不行 | "**Built-ins like `<NuxtLink>` skip the islands transform**"——它们被跳过的是那套转换 |
| 两种症状 | `Failed to locate Teleport target`，或**链接直接消失且没有报错** |
| 官方解法 | 自己包一层 `.vue`，把 `nuxt-client` 加在那个包装上 |

**"没有报错"这一种最危险**，因为它看起来像渲染问题而不是配置问题。实验特性页也单独挂了一条同内容的 note："Skip `nuxt-client` on non-SFC components such as `<NuxtLink>`."

顺带解释那个错误串：`<NuxtIsland>` 用 Teleport 机制归位内容，而 `#teleports` 是 Nuxt 唯一有服务端支持的挂载点（172 那条线）——**找不到目标就是这句话的意思**。

## remoteIsland 与 `'local+remote'`：官方自称"inherently dangerous"的一段

远程岛屿的前提只有一句：

> Remote islands need `experimental.componentIslands` to be `'local+remote'` in your `nuxt.config`.

配套的键是对象形态里的 `remoteIsland`（示例注释写的是"allow rendering islands from a remote source"，默认给的是 `false`）。

然后是官方给 `<NuxtIsland>` 的 `source` 那段警告，用词很重：

> Using the `source` prop to render content from a remote server is **inherently dangerous**. When you specify a remote `source`, you are **fully trusting that server to provide safe HTML content that will be rendered directly in your application**.
>
> This is similar to using `v-html` with external content - the remote server can inject any HTML, including potentially malicious content. **Only use `source` with servers you fully trust and control.**
>
> The `dangerouslyLoadClientComponents` prop controls an additional layer of risk: whether to also **download and execute client components** from the remote source. **Even with `dangerouslyLoadClientComponents` disabled (the default), you are still trusting the remote server's HTML output.**

三层的含义我按顺序理解：

| 层 | 关掉能否消除 |
| --- | --- |
| 远端返回的 HTML 直接进你的页面 | **不能**——即使默认配置也仍然信任它 |
| 额外下载并执行远端客户端组件 | 能——靠 `dangerouslyLoadClientComponents`，默认 `false` |
| 是否允许远程岛屿这件事本身 | 能——不设为 `'local+remote'` |

**第一行那句"即使关掉第二层你依然在信任远端 HTML"，是这一节我最想让读到这的人记住的**：`dangerouslyLoadClientComponents: false` 不是"安全模式"，只是少了一层。那个 prop 的名字里带 `dangerously` 是官方有意为之。

## 与流式渲染、`noScripts` 的三条连带

岛屿与水合前那段脚本的关系，官方给了两条：

> **Component islands are compatible with streaming.** Island slot content and selective-client (`nuxt-client`) components are normally stitched into the HTML in a post-render pass, which is impossible once the body has streamed past the island anchors. Instead, the renderer **emits each island teleport as an inert `<template>` at the end of the document and relocates it into place with an inline script that runs before hydration**. This is transparent to app code. **The exception is apps built with `features.noScripts` and island components, which fall back to the buffered renderer since the relocation script cannot run.**

> One interaction to be aware of: island slots and `nuxt-client` components rely on a small inline script to relocate teleported content into place before hydration. **On routes rendered with `noScripts`, that script is omitted, so fully interactive `nuxt-client` components will not hydrate there. Plain static islands are unaffected.**

静态那份与 169 的取舍合起来看，结论是一句可执行的话：**在 `noScripts` 路由上可以有岛屿，但别指望里面有可交互的组件。**

CSP 那一条对自写脚本策略的人有用：

> **CSP nonce.** The streaming renderer emits several inline scripts and styles that bypass unhead: the bootstrap queue, the IIFE, suspense head pushes, **island-teleport relocation**, and route `<style>` blocks. If a `nonce` is present on the rendered head scripts, the renderer **reuses it on all of them automatically**...

## 我会怎么验

| 要确认的事 | 最小验证动作 |
| --- | --- |
| 不配任何东西时岛屿是否可用 | 只写一个 `.server.vue` 引到页面上，看服务端 HTML 里有没有它的内容、客户端包里有没有它的库 |
| `selectiveClient` 是否真的生效 | 加 `nuxt-client`，看客户端网络面板只多了那个组件的 chunk，其余岛屿内容仍是静态 HTML |
| `deep` 的增量 | 给 `nuxt-client` 组件传插槽：`true` 时不通、`'deep'` 时通且插槽内元素点不动 |
| 远程岛屿是否可用 | `source` 指向另一个本机端口，先在非 `'local+remote'` 下确认不渲染，再开一次对比 |
| 内置组件上的 `nuxt-client` | 直接给 `<NuxtLink nuxt-client>`，客户端导航后观察是"报错"还是"链接消失" |

**这套验证顺序同时也是我配置这套特性的顺序**：先确认默认够用，再只解锁需要的那一项。

## 自查

1. 默认值 `'auto'` 的触发条件是什么？官方由此给出的结论句是哪半句？
2. 只有哪两种能力需要显式配置？对应的两个键叫什么？
3. `selectiveClient: true` 与 `'deep'` 的差别是什么？`'deep'` 打开的插槽为什么不交互？
4. 岛屿自己的插槽与 `nuxt-client` 组件的插槽，官方分别用哪句话描述交互性？
5. `nuxt-client` 只能用在哪类组件上？为什么内置组件不行？两种症状分别是什么、官方给的解法是什么？
6. 远程岛屿需要什么值？`dangerouslyLoadClientComponents` 关掉的是哪一层风险、哪一层关不掉？
7. 岛屿与流式渲染的兼容靠什么机制实现？`noScripts` 下会发生什么、哪类岛屿不受影响？
8. 三处文档对 `componentIslands` 的写法有哪些不一致？你按什么原则取舍？

## 下一篇

172 讲 `<Teleport>` 那侧的另一件事：`#teleports` 这个容器可以由 `app.teleportId`、`app.teleportTag`、`app.teleportAttrs` 三处配置改写，以及岛屿归位那段内联脚本与它的关系。

## 本文依据

- `https://nuxt.com/docs/4.x/guide/concepts/server-components`
  - 原文（Enabling Server Components）："Component islands are controlled by `experimental.componentIslands`. The default value is `'auto'`, which enables the feature automatically as soon as your app contains a server component or island, so in most cases you do not need any configuration."；"Set the option explicitly if you want remote islands or selective client hydration:" + `experimental: { componentIslands: { selectiveClient: true, // or 'deep', to enable `nuxt-client` \n remoteIsland: false // allow rendering islands from a remote source } }`；read-more "Server components are still marked experimental. You can follow the roadmap on GitHub."
  - 原文（Selective Hydration with `nuxt-client`）："An island is static by default, but you can hydrate individual components inside it by adding the `nuxt-client` attribute. This requires `experimental.componentIslands.selectiveClient` to be enabled."；示例 `app/components/ServerWithClient.server.vue` 里 `<Counter nuxt-client :count="5" />` 与注释 "Counter will be loaded and hydrated client-side"；"The component marked with `nuxt-client` is server-rendered as part of the island, then hydrated by the main client app. Only its chunk is shipped to the client; the rest of the island remains static."；"Setting `selectiveClient: 'deep'` additionally allows passing slots to `nuxt-client` components. Those slots are rendered on the server and are **not interactive** on the client."；warning "Use `nuxt-client` only on local `.vue` SFCs. Built-ins like `<NuxtLink>` skip the islands transform. After client navigation you may see `Failed to locate Teleport target`, or the link disappears with no error. Wrap the built-in in your own `.vue` file and put `nuxt-client` on that wrapper. See #29251 and #26002."
  - 原文（Slots）："Slots can be passed to an island component if declared in the island. Slot content is provided by the parent, so it belongs to the main client app and **is** interactive (it is wrapped in a `<div>` with `display: contents;`)."；"`<NuxtIsland>` reserves the `#fallback` slot to specify content rendered before the island loads (when `lazy` is set) or when fetching the island fails."
  - 原文（noScripts 连带）："One interaction to be aware of: island slots and `nuxt-client` components rely on a small inline script to relocate teleported content into place before hydration. On routes rendered with `noScripts`, that script is omitted, so fully interactive `nuxt-client` components will not hydrate there. Plain static islands are unaffected."；"If you are building a mostly-static site, islands combine well with `prerender` and `noScripts` route rules."
  - 原文（`context` prop 与 `islandContext`）："If an island needs route information, pass it in explicitly, either as props or via the `context` prop on `<NuxtIsland>` (read inside the island from `nuxtApp.ssrContext.islandContext`)"
- `https://nuxt.com/docs/4.x/guide/going-further/experimental-features`
  - 原文（componentIslands 条目）："Enables experimental component islands support with `<NuxtIsland>` and `.island.vue` files."；示例 `componentIslands: true, // false or 'local+remote'`；note "Skip `nuxt-client` on non-SFC components such as `<NuxtLink>`. See selective hydration with `nuxt-client`."；read-more "You can follow the server components roadmap on GitHub."
  - 原文（streaming 连带与 CSP）：note "Component islands are compatible with streaming. Island slot content and selective-client (`nuxt-client`) components are normally stitched into the HTML in a post-render pass, which is impossible once the body has streamed past the island anchors. Instead, the renderer emits each island teleport as an inert `<template>` at the end of the document and relocates it into place with an inline script that runs before hydration. This is transparent to app code. The exception is apps built with `features.noScripts` and island components, which fall back to the buffered renderer since the relocation script cannot run."；note "CSP nonce. The streaming renderer emits several inline scripts and styles that bypass unhead: the bootstrap queue, the IIFE, suspense head pushes, island-teleport relocation, and route `<style>` blocks. If a `nonce` is present on the rendered head scripts, the renderer reuses it on all of them automatically..."
- `https://nuxt.com/docs/4.x/api/components/nuxt-island`
  - 原文（远程与风险）：note "Remote islands need `experimental.componentIslands` to be `'local+remote'` in your `nuxt.config`."；warning "Using the `source` prop to render content from a remote server is inherently dangerous. When you specify a remote `source`, you are fully trusting that server to provide safe HTML content that will be rendered directly in your application. This is similar to using `v-html` with external content - the remote server can inject any HTML, including potentially malicious content. **Only use `source` with servers you fully trust and control.** The `dangerouslyLoadClientComponents` prop controls an additional layer of risk: whether to also download and execute client components from the remote source. Even with `dangerouslyLoadClientComponents` disabled (the default), you are still trusting the remote server's HTML output."；props `source` 与 `dangerouslyLoadClientComponents`（default `false`）
- `https://nuxt.com/docs/4.x/directory-structure/app/components`
  - 对照用（见 161、170）：岛屿默认扫描目录 `~/components/islands/`、`.server.vue` 与单根要求
- `https://nuxt.com/docs/4.x/api/nuxt-config`
  - 事实核对：那一节未提供 `experimental.componentIslands` 条目
- `https://nuxt.com/docs/4.x/guide/going-further/features`
  - 对照用（见 141、172）：`noScripts` 定义与"components hydrated with `nuxt-client` and interactive island slots rely on an inline script to relocate teleported content"
