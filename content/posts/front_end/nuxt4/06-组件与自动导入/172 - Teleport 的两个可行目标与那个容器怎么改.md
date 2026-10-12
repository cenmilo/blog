---
url: /posts/238ba12b55cf7ea3cd12f103c7ad91cc/
title: Teleport 的两个可行目标与那个容器怎么改
date: 2026-12-08T10:00:00+08:00
updated: 2026-12-08T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：098 讲过"哪个挂载点有服务端支持"，这一篇补上另一半：那个容器本身是怎么来的、能改成什么。官方那条边界只有一句："The `to` target of `<Teleport>` expects a CSS selector string or an actual DOM node. **Nuxt currently has SSR support for teleports to `#teleports` only, with client-side support for other targets using a `<ClientOnly>` wrapper.**"`#teleports` 不是页面里手写出来的元素，而是 Nuxt 生成、并由 `app` 节三个键控制的容器："### `teleportAttrs` — Customize Nuxt Teleport element attributes.（Type `object`，Default `{ "id": "teleports" }`）"、"**### `teleportId` — Customize Nuxt Teleport element id.（Type `string`，Default: `"teleports"`）**"、"**### `teleportTag` — Customize Nuxt Teleport element tag.（Type `string`，Default: `"div"`）**"。同一节里还有两组形状完全相同的键：根容器 `rootAttrs`（Default `{ "id": "__nuxt" }`）/ `rootId`（Default `"__nuxt"`）/ `rootTag`（Default `"div"`），以及 SPA 加载模板 `spaLoaderAttrs`（`{ "id": "__nuxt-loader" }`）/ `spaLoaderTag`（`"div"`）——**149 里 `app.mount('#__nuxt')` 那个选择器就是 `rootId` 的默认值**。官方给的两种写法对应那两条通路：Body Teleport 用 `to="#teleports"`，Client-side Teleport 用 `<ClientOnly>` 包住 `to="#some-selector"`。岛屿那一侧的连带事实有两条：内置组件上用 `nuxt-client` 会出现 "**`Failed to locate Teleport target`, or the link disappears with no error**"，而流式渲染下"island teleport 会被输出成文档末尾一个 **inert `<template>`**，再由一段**水合前运行的内联脚本**归位；`features.noScripts` 时该脚本无法运行，因此回退到缓冲渲染器。另有一处官方描述重复我照着字段名理解：`rootAttrs` 的说明句写成了"Customize Nuxt root element **id**"，与 `rootId` 同一句。

categories:
  - 前端开发

tags:
  - nuxt4
  - Teleport
  - SSR
  - app-config
  - 岛屿

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


098 那篇我记住了一句"只有 `#teleports` 有服务端支持"。后来我想把弹窗 teleport 到 `#modal-root`，才发现问题变成两个：**其它目标要靠什么代价支持**，以及**那个 `#teleports` 容器本身到底是谁生成的**。

## 那条边界只有一句话

官方 `<Teleport>` 那页的第一条 warning 就是全部结论：

> The `to` target of [`<Teleport>`](https://vuejs.org/guide/built-ins/teleport) expects **a CSS selector string or an actual DOM node**. Nuxt currently has **SSR support for teleports to `#teleports` only**, with **client-side support for other targets using a `<ClientOnly>` wrapper**.

拆成两栏，选择其实只有两个：

| 目标 | 服务端有没有 | 需要什么 |
| --- | --- | --- |
| `#teleports` | ✅ 有 SSR 支持 | 直接用 |
| 其它选择器/DOM 节点 | ❌ 没有 | 包一层 `<ClientOnly>`，只在客户端生效 |

`to` 接受的形态官方也说明了：CSS 选择器字符串**或**一个真实 DOM 节点。

## 官方给的两种写法

**能走服务端的那条**（页面上叫 "Body Teleport"）：

```vue
<template>
  <button @click="open = true">
    Open Modal
  </button>
  <Teleport to="#teleports">
    <div
      v-if="open"
      class="modal"
    >
      <p>Hello from the modal!</p>
      <button @click="open = false">
        Close
      </button>
    </div>
  </Teleport>
</template>
```

**必须走客户端的那条**：

```vue
<template>
  <ClientOnly>
    <Teleport to="#some-selector">
      <!-- content -->
    </Teleport>
  </ClientOnly>
</template>
```

这两例放在一起看，代价很清楚：**第二个例子里的 `<ClientOnly>` 不是可选装饰**。它会带上 165 那套连带后果——默认插槽内容被摇出服务端构建、其中的 CSS 可能不进首行内联、内容是挂载之后才出现。

所以我按目标位置分两种做法：**能接受"落到 `#teleports`"的（弹窗、通知、遮罩层这类浮层）一律不包 `<ClientOnly>`**；必须落到页面里某个特定容器时，才接受那份代价。

## `#teleports` 是 Nuxt 生成的容器，三处可改

这是我这次真正想查的东西。`api/nuxt-config` 的 `app` 节里有三个专门键：

> ### `teleportAttrs`
> Customize Nuxt Teleport element attributes.
> - **Type**: `object`
> - **Default** `{ "id": "teleports" }`
>
> ### `teleportId`
> Customize Nuxt Teleport element id.
> - **Type**: `string`
> - **Default:** `"teleports"`
>
> ### `teleportTag`
> Customize Nuxt Teleport element tag.
> - **Type**: `string`
> - **Default:** `"div"`

也就是说：`#teleports` 是**一个默认 `<div id="teleports">`**，标签、id、属性三样都能改。**它不需要你在模板或 HTML 里自己写出来**——这一点解释了为什么我从来没在自己的代码里"准备"过这个容器。

`app` 节里还有两组形状完全相同的键，放在一起更好记：

| 容器 | attrs | id | tag | 默认 |
| --- | --- | --- | --- | --- |
| Teleport 容器 | `app.teleportAttrs` | `app.teleportId` | `app.teleportTag` | `<div id="teleports">` |
| 应用根容器 | `app.rootAttrs` | `app.rootId` | `app.rootTag` | `<div id="__nuxt">` |
| SPA 加载模板 | `app.spaLoaderAttrs`（其下还有一个 `id` 子键，默认 `"__nuxt-loader"`） | — | `app.spaLoaderTag`（默认 `"div"`） | `<div id="__nuxt-loader">` |

**根容器那一行的价值在于闭环**：149 里那句 `app.mount('#__nuxt')` 用的选择器，就是 `rootId` 的默认值。**如果你改了 `app.rootId`，那个挂载点的名字也变了**——这是两个不同篇章里各说一半的事。

顺带一处官方描述重复，我按字段名理解：`rootAttrs` 的说明句写的是 "Customize Nuxt root element **id**."（与 `rootId` 同一句），而 `teleportAttrs` 那句写的是 "…**attributes**"。前者更像是笔误，两处语义都以字段名为准。

## 改了 id 会不会让 `to="#teleports"` 失效

官方没有直接回答这个问题，所以我不写成结论，只写我会怎么确认：

1. 设 `app.teleportId: 'my-teleports'`，跑一次带 SSR 的页面。
2. 看渲染出的 HTML 里那个容器实际是什么 id。
3. 模板里保持 `to="#teleports"` 的那处 teleport 是否还在服务端出现（还是只能在客户端才出现）。

**如果它确实失效（我预期会，因为选择器字符串是写死的），正确写法就是把 `to` 一起改掉**——但这是我的推断，验证一次只用两分钟，不值得我替你保证。

（一个替代思路：与其改全局 id，不如给需要精确落位的容器走 `<ClientOnly>` 那条官方通路。后者是文档明确支持的形状。）

## 岛屿那一侧的两个 Teleport 事实

171 讲 `nuxt-client` 时引过那条警告，它的两种症状里就有这句：

> After client navigation you may see **`Failed to locate Teleport target`**, or the link disappears with no error.

**"找不到 Teleport 目标"是岛屿那套归位机制失败的表象**，不是模板拼写错误——原因在流式渲染那条说明里：

> Instead, the renderer **emits each island teleport as an inert `<template>` at the end of the document and relocates it into place with an inline script that runs before hydration**. This is transparent to app code. The exception is apps built with `features.noScripts` and island components, which **fall back to the buffered renderer since the relocation script cannot run**.

以及特性页对 `noScripts` 路由的那句：

> components hydrated with `nuxt-client` and interactive island slots **rely on an inline script to relocate teleported content**, so they will not become interactive on a `noScripts` route.

把三处叠起来，`<Teleport>` 在 Nuxt 里其实出现在三个不同层面：

| 层面 | 谁在用 Teleport | 失败时的表现 |
| --- | --- | --- |
| 你写的 `<Teleport>` | 你选的目标 | 目标不存在就归位不到 |
| 岛屿插槽与 `nuxt-client` | 框架生成 inert `<template>` + 内联脚本 | `Failed to locate Teleport target` 或静默消失 |
| `noScripts` 路由 | 脚本被整体去掉 | 岛屿里的可交互组件不水合（静态岛屿不受影响） |

## 该怎么做：一张表

| 需求 | 写法 | 代价 |
| --- | --- | --- |
| 弹窗/通知/遮罩这类浮层 | `<Teleport to="#teleports">` | 无（有 SSR 支持） |
| 必须落到页面里某个特定容器 | `<ClientOnly>` 包 `<Teleport to="#xxx">` | 服务端不渲染 + CSS 内联缺失风险 + 挂载后才出现 |
| 想让容器标签不是 `div` | `app.teleportTag` | 只改容器形态，不改变上面两条规则 |
| 想让容器 id 不叫 `teleports` | `app.teleportId` + 同步检查所有 `to` | 需要自己验证（上一节） |
| 岛屿里要让某组件可交互 | 别把 `nuxt-client` 放在内置组件上（171） | 症状可能静默 |

## 自查

1. 官方那句话给出的两条通路分别是什么？`to` 能接受哪两种形态？
2. 为什么第二个例子必须包 `<ClientOnly>`？它带来哪几条 165 说过的连带后果？
3. `#teleports` 这个容器是谁生成的？它的默认标签与默认 id 是什么？
4. 改 Teleport 容器的三个键分别叫什么？和 `app.root*`、`app.spaLoader*` 的形状有什么共同点？
5. `app.mount('#__nuxt')` 那个选择器来自哪个配置默认值？改了它要连带注意什么？
6. 官方对 `rootAttrs` 的说明句有什么问题？你怎么取舍？
7. `Failed to locate Teleport target` 在岛屿场景下意味着什么？流式渲染时岛屿 teleport 是怎么归位的？`noScripts` 会怎样？
8. 你会在什么情况下接受 `<ClientOnly>` 那条代价？

## 下一篇

173 收 P06 这一段：把"一个名字/标签是怎么被解析出来的"画成一张顺序图，从自动导入名单、目录扫描、编译器宏到全局注册，标出每一步失败时你会看到什么。

## 本文依据

- `https://nuxt.com/docs/4.x/api/components/teleports`
  - 原文（定位与边界）："The `<Teleport>` component teleports a component to a different location in the DOM."；warning "The `to` target of [`<Teleport>`](https://vuejs.org/guide/built-ins/teleport) expects a CSS selector string or an actual DOM node. Nuxt currently has SSR support for teleports to `#teleports` only, with client-side support for other targets using a `<ClientOnly>` wrapper."
  - 原文（两个示例）：Body Teleport——`<button @click="open = true">Open Modal</button>` + `<Teleport to="#teleports"><div v-if="open" class="modal">…<button @click="open = false">Close</button></div></Teleport>`；Client-side Teleport——`<ClientOnly><Teleport to="#some-selector"><!-- content --></Teleport></ClientOnly>`；页尾 `::link-example{to="https://nuxt.com/docs/examples/advanced/teleport"}`
- `https://nuxt.com/docs/4.x/api/nuxt-config`
  - 原文（`app` 节三组键）："### `teleportAttrs` — Customize Nuxt Teleport element attributes. — Type: `object` — Default `{ "id": "teleports" }`"；"### `teleportId` — Customize Nuxt Teleport element id. — Type: `string` — Default: `"teleports"`"；"### `teleportTag` — Customize Nuxt Teleport element tag. — Type: `string` — Default: `"div"`"；"### `rootAttrs` — Customize Nuxt root element id. — Type: `object` — Default `{ "id": "__nuxt" }`"（说明句与 `rootId` 重复）；"### `rootId` — Customize Nuxt root element id. — Default: `"__nuxt"`"；"### `rootTag` — Customize Nuxt root element tag. — Default: `"div"`"；"### `spaLoaderAttrs` — Customize Nuxt SPA loading template element attributes. — Default `{ "id": "__nuxt-loader" }`"、其下 `#### id`（Default `"__nuxt-loader"`）与 "### `spaLoaderTag` — Customize Nuxt SpaLoader element tag. — Default: `"div"`"
- `https://nuxt.com/docs/4.x/guide/concepts/server-components` 与 `https://nuxt.com/docs/4.x/guide/going-further/experimental-features`
  - 原文（岛屿与 Teleport 的关系）：warning "After client navigation you may see `Failed to locate Teleport target`, or the link disappears with no error."；note "Component islands are compatible with streaming. … Instead, the renderer emits each island teleport as an inert `<template>` at the end of the document and relocates it into place with an inline script that runs before hydration. This is transparent to app code. The exception is apps built with `features.noScripts` and island components, which fall back to the buffered renderer since the relocation script cannot run."；"island slots and `nuxt-client` components rely on a small inline script to relocate teleported content into place before hydration. On routes rendered with `noScripts`, that script is omitted, so fully interactive `nuxt-client` components will not hydrate there. Plain static islands are unaffected."
- `https://nuxt.com/docs/4.x/guide/concepts/rendering`
  - 原文（"Calling `app.mount('#__nuxt')` mounts the Vue application to the DOM."）
- `https://nuxt.com/docs/4.x/api/components/client-only`
  - 对照用（见 165、096）：默认插槽内容被 tree-shake 出服务端构建、CSS 内联的那半句括号、"rendered only after being mounted"
- `https://nuxt.com/docs/4.x/guide/going-further/features`
  - 对照用（见 141、171）：`noScripts` 与"rely on an inline script to relocate teleported content"
