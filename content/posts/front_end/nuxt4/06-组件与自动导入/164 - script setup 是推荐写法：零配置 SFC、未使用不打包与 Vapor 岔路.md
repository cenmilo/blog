---
url: /posts/e687704ea2dad8c52e7f54d8c33be20d/
title: 164 - script setup 是推荐写法：零配置 SFC、未使用不打包与 Vapor 岔路
date: 2026-12-05T11:00:00+08:00
updated: 2026-12-05T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇把"组件该怎么写"这件事按官方原话定下来。官方推荐句出现了两处一模一样的位置——自动导入相关页与 `defineNuxtComponent` 页都写着："**Using `<script setup lang="ts">` is the recommended way of declaring Vue components in Nuxt.**"，数据获取页还有一条同义句："Using `<script setup>` or `<script setup lang="ts">` are the recommended way of declaring Vue components in Nuxt."。SFC 侧框架给的是零配置体验："**Nuxt provides a zero-config experience for SFCs with Hot Module Replacement that offers a seamless developer experience.**"；"没用到就不进包"这句在两个地方各说了一遍：组件侧"**If a component is not used anywhere, your production's code will not include it.**"，自动导入侧"**only includes what is used in your production code**"，再往底层是 Vue 3 的可摇树性："**most of Vue's functionality, including template directives and built-in components, is tree-shakable. Your production bundle will not include them if you don't use them.**"，官方给的量级是"**a minimal Vue 3 application can reduced to 12 kb gzipped**"。关于两套 API 的关系，官方特意澄清："> The Composition API introduced in Vue 3 is **not a replacement of the Options API**, but it enables better logic reuse throughout an application, and is a more natural way to group code by concern in complex components."，并给出框架目标："**The goal of Nuxt is to provide a great developer experience around the Composition API.**"。最后一条岔路是 Vapor：官方状态标注是"**Vapor Mode support is experimental and requires Vue 3.6 or newer. The API may change.**"，做法是 interop 模式——"**your application is still rendered with the Virtual DOM, and you opt individual components or pages into Vapor by writing them as Vapor single-file components**"，开启要 `vue: { vapor: true }`、装 `^3.6.0-rc.2`，"**If you enable `vue.vapor` on an older version of Vue, Nuxt warns and disables it.**"，官方建议"**trying Vapor Mode in a fresh project rather than adding it to an existing one**"；六条限制里两条直接推翻了前面几篇的写法：Options API 不支持（`asyncData`/`fetchKey` via `defineNuxtComponent` 不可用）、以及"**Nuxt composables called after `await` in a Vapor `<script setup>` may lose the Nuxt context**"。

categories:
  - 前端开发

tags:
  - nuxt4
  - script-setup
  - SFC
  - Composition-API
  - Vapor

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


前面几十篇里我几乎没讨论过"组件应该怎么写"——默认就在写 `<script setup>`。这一篇把默认值背后的官方立场、它承诺的收益、以及一条会让人改主意的岔路（Vapor）一次说清。

## 官方推荐句出现在三个地方

同一条推荐，措辞在三节里各写了一次：

> Using `<script setup lang="ts">` is the recommended way of declaring Vue components in Nuxt.

这句话在 `defineNuxtComponent` 那一节出现过一次（162），在数据获取页的 Options API 一节又见一次，只是那里写成 "Using `<script setup>` **or** `<script setup lang="ts">` are the recommended way..."。**三处重复说明这不是顺带一提**——框架把 `<script setup>` 当默认答案，其他写法是兼容路径。

配套的还有那句目标陈述：

> The goal of Nuxt is to provide a great developer experience around the **Composition API**.

具体给的两条做法是：

> - Use auto-imported Reactivity functions from Vue and Nuxt built-in composables.
> - Write your own auto-imported reusable functions in the `app/composables/` directory.

## SFC：零配置与 HMR

官方先给定义，再给框架提供什么：

> Vue's single-file components (SFC or `*.vue` files) encapsulate the markup (`<template>`), logic (`<script>`) and styling (`<style>`) of a Vue component. **Nuxt provides a zero-config experience for SFCs with Hot Module Replacement that offers a seamless developer experience.**

160 那节库作者例子的收尾句其实呼应了同一个承诺——改 `node_modules` 里的组件也能 HMR。**零配置 + HMR 是框架给 SFC 的两件东西**，不是 Vue 自带的。

## "没用到就不进包"这句被说了三遍，层次不同

| 原话 | 层次 |
| --- | --- |
| "Every Vue component created in the `app/components/` directory of a Nuxt project will be available in your project without having to import it. **If a component is not used anywhere, your production's code will not include it.**" | 组件自动注册 |
| "Contrary to a classic global declaration, Nuxt preserves typings, IDEs completions and hints, and **only includes what is used in your production code**." | 自动导入机制 |
| "With version 3, most of Vue's functionality, including template directives and built-in components, is tree-shakable. **Your production bundle will not include them if you don't use them.**" | Vue 3 本身的可摇树性 |

**三条叠起来才是完整答案**：我不写 import 却能上生产，是因为底层 Vue 可摇树、中间 unimport 按名注入、上层组件按使用裁剪。少任何一层，"自动导入不增加体积"就不成立。

官方还给了一个量级参照：

> This way, a minimal Vue 3 application can be reduced to **12 kb gzipped**.

渲染侧的收益也有对应说法：

> The Vue Virtual DOM (VDOM) has been rewritten from the ground up and allows for better rendering performance. On top of that, when working with compiled Single-File Components, the Vue compiler can further optimize them at build time by **separating static and dynamic markup**.

> This results in faster first rendering (component creation) and updates, and less memory usage. **In Nuxt 3, it enables faster server-side rendering as well.**

## Options API 不是被淘汰，而是"另一种"

这句澄清很重要，否则前面 162 那篇会读成"框架不让用 Options API"：

> The Composition API introduced in Vue 3 is **not a replacement of the Options API**, but it enables better logic reuse throughout an application, and is a more natural way to group code by concern in complex components.

官方给的前后对照（同一个计数器）：

```vue
<script>
export default {
  data () {
    return {
      count: 0,
    }
  },
  methods: {
    increment () {
      this.count++
    },
  },
}
</script>
```

```vue [components/Counter.vue]
<script setup lang="ts">
const count = ref(0)
const increment = () => count.value++
</script>
```

官方对后者的说明句是："Used with the `setup` keyword in the `<script>` definition, here is the above component rewritten with Composition API and **Nuxt 3's auto-imported Reactivity APIs**."——`ref` 那行没写 import，这正是 150 那条约定的落地。

## TypeScript 是可选的两步

> Both Vue 3 and Nuxt 3+ are written in TypeScript. A fully typed codebase prevents mistakes and documents APIs usage. **This doesn't mean that you have to write your application in TypeScript** to take advantage of it. With Nuxt 3, you can opt-in by **renaming your file from `.js` to `.ts`**, or **add `<script setup lang="ts">`** in a component.

**注意官方给的推荐句是带着 `lang="ts"` 的**：推荐写法是 `<script setup lang="ts">`，但语言本身是 opt-in 的两步之一。类型从哪来、四份 tsconfig 各管哪片代码，155 那篇。

## Vapor 这条岔路的现状

官方先标状态：

> **Vapor Mode support is experimental and requires Vue 3.6 or newer. The API may change.**

再给定义与做法：

> Vapor Mode is an alternative compilation strategy introduced in Vue 3.6 that **renders components without the Virtual DOM**, lowering memory use and improving runtime performance.

> Nuxt supports Vapor Mode in **interop mode**: your application is still rendered with the Virtual DOM, and you **opt individual components or pages into Vapor** by writing them as Vapor single-file components. This lets you adopt Vapor incrementally, and **the rest of Nuxt (routing, `useAsyncData`, layouts, and most built-in components) keeps working unchanged.**

开启与用法：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  vue: {
    vapor: true,
  },
})
```

```vue [app/pages/index.vue]
<script setup vapor lang="ts">
const count = ref(0)
</script>

<template>
  <button @click="count++">
    count is {{ count }}
  </button>
</template>
```

> This installs Vue's `vaporInteropPlugin`, which lets Vapor and Virtual DOM components render alongside each other. You can then mark any component or page as Vapor by **adding the `vapor` attribute to `<script setup>`**.

版本要求官方写得很具体，并给了降级行为：

> Vapor Mode needs a version of Vue that includes both Vapor and its interop fixes. While Vue 3.6 is in pre-release, install a release candidate (or newer) that contains those fixes: `{ "dependencies": { "vue": "^3.6.0-rc.2" } }`

> **If you enable `vue.vapor` on an older version of Vue, Nuxt warns and disables it.**

> We recommend **trying Vapor Mode in a fresh project rather than adding it to an existing one.**

### 六条限制里有两条会推翻前面的写法

官方的清单：

> - **Full Vapor apps are not yet supported.** The application root stays Virtual DOM and you opt in per component; we may support a full Vapor app later on.
> - **Template refs** on a Vapor component do not expose `$el`.
> - **Some built-in components inspect their slot content** and cannot read Vapor slot children. `<ClientOnly>` skips its attribute-forwarding fallthrough, and `<NuxtIsland>`, server components, and the head components that read text children (such as `<Title>`, `<Style>` and `<Noscript>`) warn if you pass a Vapor slot. Pass the value directly instead (for example, a string to `<Title>`).
> - **The Options API is not supported** in Vapor components, so `asyncData` and `fetchKey` via `defineNuxtComponent` are unavailable. Use `<script setup>` with `useAsyncData` instead.
> - **Keyed `onPrehydrate`** falls back to the unkeyed form (and warns in development), because there is no component instance to attach the key to.
> - **Nuxt composables called after `await` in a Vapor `<script setup>` may lose the Nuxt context.** Call them before the first `await` where possible.

两条与我前面写的东西直接冲突，值得单独记住：

| 冲突 | 影响 |
| --- | --- |
| 第 4 条 | 162 那套 Options API 写法（`asyncData`、`fetchKey`）在 Vapor 组件里**直接不可用** |
| 第 6 条 | 151 那条"必须同步调用 composable"在 Vapor 里**变成真实约束**——`<script setup>`  ordinarily 是官方列的四个可跨 `await` 例外之一，而 Vapor 组件里可能丢上下文 |

收尾那句是官方给的安心话，也解释了为什么大部分东西照旧：

> Most other Nuxt composables and built-in components work unchanged, **because they rely on injection rather than on a component instance**.

（第 3 条里 `<ClientOnly>` 与 `<NuxtIsland>`、head 组件的表现，正好对应 096、107 与标题类组件的插槽用法——Vapor 下要么别传 Vapor 插槽，要么把值直接传进去。）

## 我该按什么写

| 场景 | 写法 |
| --- | --- |
| 新组件、新项目 | `<script setup lang="ts">`，官方三处推荐的原文 |
| 存量 Options API 组件要用 `asyncData`/`head`/`fetchKey` | `defineNuxtComponent` 包装（162），并接受"这是兼容层" |
| `.ts` 里手写组件、要用带上下文的 composable | `defineNuxtComponent`（151 那条 tip） |
| 想试 Vapor | 新项目 + `vue: ^3.6.0-rc.2` + `vue.vapor: true`，逐组件加 `vapor` 属性；不要往已有项目里塞（官方原话） |

## 自查

1. 官方那句推荐一共出现在哪几处？措辞差别是什么？配套的框架目标句是哪句？
2. 框架给 SFC 的两件东西是什么？"没用到就不进包"分别在哪些层次被说了一遍？
3. Composition API 与 Options API 的关系，官方怎么澄清？这如何影响你怎么读 162 那篇？
4. TypeScript 的两步 opt-in 是什么？官方对"必须用 TS 吗"的说法是什么？
5. Vapor 在 Nuxt 里是哪种模式？根组件是什么状态？开启需要改哪两处？
6. Vue 版本不够时会发生什么？官方对项目选择给的建议是什么？
7. 六条限制里哪两条会推翻前面篇目的写法？请分别说明影响。
8. 为什么大多数 composable 与内置组件在 Vapor 下照旧？官方给的原因是什么？

## 下一篇

165 讲"只在客户端渲染"的三条官方路径：`<ClientOnly>` 的两种占位写法、`.client.vue` 后缀及其两条限制、以及整页关 SSR，各自付出的代价分别是什么。

## 本文依据

- `https://nuxt.com/docs/4.x/guide/concepts/vuejs-development`
  - 原文（开篇与选择理由）："Nuxt uses Vue.js and adds features such as component auto-imports, file-based routing and composables for an SSR-friendly usage."；"Nuxt integrates Vue 3, the new major release of Vue that enables new patterns for Nuxt users."；note "While an in-depth knowledge of Vue is not required to use Nuxt, we recommend that you read the documentation and go through some of the examples on vuejs.org."；三条理由句（reactivity model / component-based templating / performance at scale）
  - 原文（SFC 与自动导入）："[Vue's single-file components] (SFC or `*.vue` files) encapsulate the markup (`<template>`), logic (`<script>`) and styling (`<style>`) of a Vue component. Nuxt provides a zero-config experience for SFCs with Hot Module Replacement that offers a seamless developer experience."；"Every Vue component created in the `app/components/` directory of a Nuxt project will be available in your project without having to import it. If a component is not used anywhere, your production's code will not include it."
  - 原文（Vapor Mode）：warning "Vapor Mode support is experimental and requires Vue 3.6 or newer. The API may change."；"Vapor Mode is an alternative compilation strategy introduced in Vue 3.6 that renders components without the Virtual DOM, lowering memory use and improving runtime performance."；"Nuxt supports Vapor Mode in interop mode: your application is still rendered with the Virtual DOM, and you opt individual components or pages into Vapor by writing them as Vapor single-file components. This lets you adopt Vapor incrementally, and the rest of Nuxt (routing, `useAsyncData`, layouts, and most built-in components) keeps working unchanged."；`vue: { vapor: true }`；"This installs Vue's `vaporInteropPlugin`, which lets Vapor and Virtual DOM components render alongside each other. You can then mark any component or page as Vapor by adding the `vapor` attribute to `<script setup>`."；示例 `<script setup vapor lang="ts">` + `<button @click="count++">count is {{ count }}</button>`；"`{ dependencies: { vue: '^3.6.0-rc.2' } }`"；"If you enable `vue.vapor` on an older version of Vue, Nuxt warns and disables it."；"We recommend trying Vapor Mode in a fresh project rather than adding it to an existing one."；六条 Known limitations 全文（Full Vapor apps not yet supported / Template refs 不暴露 `$el` / built-in components 读不到 Vapor slot children 且 `<ClientOnly>` skips attribute-forwarding fallthrough、`<NuxtIsland>`、server components 与 `<Title>`/`<Style>`/`<Noscript>` 会 warn / Options API is not supported so `asyncData` and `fetchKey` via `defineNuxtComponent` are unavailable / Keyed `onPrehydrate` falls back / Nuxt composables called after `await` may lose the Nuxt context）；"Most other Nuxt composables and built-in components work unchanged, because they rely on injection rather than on a component instance."
  - 原文（Differences with Nuxt 2 / Vue 2）："Nuxt 3+ is based on Vue 3. The new major Vue version introduces several changes that Nuxt takes advantage of: Better performance / Composition API / TypeScript support"；Faster Rendering 两段（VDOM rewritten、separating static and dynamic markup、"In Nuxt 3, it enables faster server-side rendering as well."）；Smaller Bundle 段（"most of Vue's functionality, including template directives and built-in components, is tree-shakable. Your production bundle will not include them if you don't use them."、"a minimal Vue 3 application can be reduced to 12 kb gzipped"）；Composition API 段（"The Composition API introduced in Vue 3 is not a replacement of the Options API, but it enables better logic reuse throughout an application, and is a more natural way to group code by concern in complex components."、Options API 的 `data`/`methods` 示例与 `components/Counter.vue` 的 `<script setup>` 示例、"Used with the `setup` keyword in the `<script>` definition, here is the above component rewritten with Composition API and Nuxt 3's auto-imported Reactivity APIs."、"The goal of Nuxt is to provide a great developer experience around the Composition API." 与两条做法）；TypeScript Support 段（"Both Vue 3 and Nuxt 3+ are written in TypeScript. A fully typed codebase prevents mistakes and documents APIs usage. This doesn't mean that you have to write your application in TypeScript to take advantage of it. With Nuxt 3, you can opt-in by renaming your file from `.js` to `.ts`, or add `<script setup lang=\"ts\">` in a component."）
- `https://nuxt.com/docs/4.x/api/utils/define-nuxt-component` 与 `https://nuxt.com/docs/4.x/getting-started/data-fetching`
  - 原文（两处推荐 note）："Using `<script setup lang=\"ts\">` is the recommended way of declaring Vue components in Nuxt."；"Using `<script setup>` or `<script setup lang=\"ts\">` are the recommended way of declaring Vue components in Nuxt."
- `https://nuxt.com/docs/4.x/guide/concepts/auto-imports`
  - 原文（第三处承诺）："Contrary to a classic global declaration, Nuxt preserves typings, IDEs completions and hints, and only includes what is used in your production code."；"Vue exposes Reactivity APIs like `ref` or `computed`, as well as lifecycle hooks and helpers that are auto-imported by Nuxt."
