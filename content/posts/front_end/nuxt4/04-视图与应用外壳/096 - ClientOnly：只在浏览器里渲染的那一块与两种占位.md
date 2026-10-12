---
url: /posts/d4369d2381ddeddc6f7576d9cdb8bb62/
title: ClientOnly：只在浏览器里渲染的那一块与两种占位
date: 2026-11-13T09:00:00+08:00
updated: 2026-11-13T09:00:00+08:00
author: cmdragon

summary:
  摘要：`<ClientOnly>` 的官方用途是"专门让一个组件只在客户端渲染"，而渲染概念页从另一侧确认了这个能力：**组件可以被强制只在客户端运行**（同一句话里还写明 middlewares 与 pages 在服务端跑、水合期在客户端再跑一次，插件可以在服务端或客户端或两侧，组合式函数与工具按使用上下文渲染）。这个组件有一条必须一起记住的副作用，官方写成 note：**默认插槽的内容会被从服务端构建里 tree-shake 掉（这也意味着其中组件所用到的 CSS 可能不会在渲染初始 HTML 时被内联）**。占位有两种写法，官方分成两组：prop 形式是 `placeholderTag` 或 `fallbackTag`（"指定一个在服务端渲染的标签"）加 `placeholder` 或 `fallback`（"指定一个在服务端渲染的内容"），官方示例是 `fallback-tag="span"` 配 `fallback="Loading comments..."`；插槽形式是 `#fallback`，官方说明是"指定一个在服务端渲染、并显示到 `<ClientOnly>` 在浏览器中挂载为止的内容"，可以放真正的节点。想拿到里面渲染出来的 DOM，官方的做法是 watch 一个模板 ref，因为"`<ClientOnly>` 内的组件只在被挂载后才渲染"。两个必须用它的具体场景：一是**依赖浏览器 API 且有副作用的库**——官方警告"打包器不会 tree-shake 含副作用模块的 import"，所以要保证导入它的组件只在客户端；二是**预渲染出来的兜底错误页**——单个文件服务于所有缺失路径，`useRoute()`、`useRequestURL()` 与按当前路径取的数据在预渲染 HTML 里持有的是构建时的值，会在水合时被纠正，官方明确写着"把请求相关的标记包进 `<ClientOnly>`"。

categories:
  - 前端开发

tags:
  - nuxt4
  - clientonly
  - 水合
  - 占位
  - 渲染模式

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


服务端没有 `window`。于是有一类代码必须在浏览器里跑——问题不是"能不能"，而是**圈多大一块**。`<ClientOnly>` 就是这个圈，这一篇讲它的边界、两种占位写法和两个真正需要它的场景。

## 定位与那条副作用

| 官方原话 | 要点 |
| --- | --- |
| "使用 `<ClientOnly>` 组件仅在客户端渲染组件。" | 页面标题级的定位 |
| "`<ClientOnly>` 组件用于**专门**在客户端渲染一个组件。" | 是"有意为之"，不是自动降级 |
| 渲染概念页："**组件也可以被强制只在客户端运行。**" | 双侧分工清单里的正式一项 |
| （note）"默认插槽的内容会被 **tree-shake 出服务端构建**。（这确实意味着其中组件所使用的 CSS 可能不会在渲染初始 HTML 时被内联。）" | **代价：首屏 HTML 里没有这块，样式也可能缺席** |

最后一条 note 容易被跳过，但它决定了两件可见的事：搜索引擎和"查看网页源代码"看到的首屏里不含这块内容；如果这块内容参与首屏布局，它的样式可能是后到的。

## 两种占位写法：官方分成两组

### 写法一：prop

官方给的 prop 表：

| prop | 官方说明 |
| --- | --- |
| `placeholderTag` 或 `fallbackTag` | 指定一个要在服务端渲染的标签 |
| `placeholder` 或 `fallback` | 指定一个要在服务端渲染的内容 |

```vue [app/pages/example.vue]
<template>
  <div>
    <Sidebar />
    <!-- The <Comment> component will only be rendered on client-side -->
    <ClientOnly
      fallback-tag="span"
      fallback="Loading comments..."
    >
      <Comment />
    </ClientOnly>
  </div>
</template>
```

这一组是**别名成对**的：`placeholderTag`/`fallbackTag`、`placeholder`/`fallback` 各自是同一件事的两种叫法。官方示例里给的内容是一句文本 `"Loading comments..."`，外面套一个 `span`。

### 写法二：`#fallback` 插槽

官方插槽表只有一行："`#fallback`：指定一个要在服务端渲染、并显示到 `<ClientOnly>` 在浏览器中挂载为止的内容。"

```vue [app/pages/example.vue]
<template>
  <div>
    <Sidebar />
    <!-- This renders the "span" element on the server side -->
    <ClientOnly fallback-tag="span">
      <!-- this component will only be rendered on client side -->
      <Comments />
      <template #fallback>
        <!-- this will be rendered on server side -->
        <p>Loading comments...</p>
      </template>
    </ClientOnly>
  </div>
</template>
```

这段官方示例的三条注释正好把三种落点标清了：`fallback-tag="span"` 决定服务端渲染出的那个**元素**；默认插槽里的 `<Comments />` **只在客户端渲染**；`<template #fallback>` 里的 `<p>` **在服务端渲染**。

| 你要的占位 | 该用哪个 |
| --- | --- |
| 一句文本 + 指定标签 | prop 形式：`fallback-tag` + `fallback` |
| 一段真实结构（`<p>`、骨架、另一个组件） | `#fallback` 插槽 |
| 只指定标签、内容放插槽里 | 官方示例正是这种混搭：`<ClientOnly fallback-tag="span">` + `#fallback` |

"显示到挂载为止"这半句也值得注意：`#fallback` 的内容不是永久占位，它撑到 `<ClientOnly>` 在浏览器里挂载为止——这就是它适合放骨架的原因。

## 场景一：库依赖浏览器 API 且有副作用

这是官方用一条警告专门圈出来的情形：**"当导入一个依赖浏览器 API 并且有副作用的库时，请确保导入它的组件只在客户端被调用。打包器不会 tree-shake 含有副作用的模块的 import。"**

为什么这条一定要配 `<ClientOnly>`：只要那个模块被 import 了，它的顶层代码就跟着进包，**不会因为"没被用到"而被甩掉**。把导入它的那个组件包进 `<ClientOnly>`，才能让它不进服务端构建（前面那条 note 说的正是这件事）。

对照渲染概念页里那条分工清单，可以更准确地理解"哪里会跑到服务端"：

| 代码 | 在哪一侧执行 |
| --- | --- |
| `<script setup>` 里的 `const counter = ref(0)` | 服务端和客户端都执行 |
| 事件回调 `handleClick` 的函数体 | **只在客户端环境执行**（官方注释） |
| middlewares 与 pages | 服务端运行，水合时在客户端再运行 |
| 插件 | 可以在服务端、客户端或两侧 |
| 组件 | **也可以被强制只在客户端运行** |
| 组合式函数与工具 | 按其使用上下文渲染 |

也就是说，如果你的第三方库调用写在事件回调里，本来就不会在服务端跑；**真正危险的是写在 `<script setup>` 顶层或模块顶层**——那才是 `<ClientOnly>` 要救的位置。

## 场景二：预渲染的兜底错误页里的"请求相关"内容

官方在讲预渲染 404 兜底页时写得很直接：因为单个文件要服务于每一个缺失的路径，错误页不能依赖请求——**`useRoute()`、`useRequestURL()` 以及用当前路径取回的数据，在预渲染的 HTML 里持有的是构建时的值，并会在水合时被纠正。把请求相关的标记包进 `<ClientOnly>`，并用 `import.meta.prerender` 跳过请求相关的数据取用。**

官方的 `error.vue` 示例正是这样写的：

```vue [error.vue]
<template>
  <div>
    <h1>{{ error.statusCode }}</h1>
    <ClientOnly>
      <p>Nothing found at {{ route.path }}</p>
    </ClientOnly>
    <!-- ... -->
  </div>
</template>
```

`error.statusCode` 来自 props（构建时就定下来了），所以直接渲染；`route.path` 是请求相关的，所以只有那一行被包进 `<ClientOnly>`。**这段的取舍就是判据：一个值是不是"取决于这次请求"。**

## 想操作里面的 DOM：watch 模板 ref

官方"Accessing HTML Elements"一节的说明：`<ClientOnly>` 内的组件**只在被挂载之后才渲染**；要在 DOM 中访问渲染出来的元素，你可以 watch 一个模板 ref。

```vue [app/pages/example.vue]
<script setup lang="ts">
const nuxtWelcomeRef = useTemplateRef('nuxtWelcomeRef')

// The watch will be triggered when the component is available
watch(nuxtWelcomeRef, () => {
  console.log('<NuxtWelcome /> mounted')
}, { once: true })
</script>

<template>
  <ClientOnly>
    <NuxtWelcome ref="nuxtWelcomeRef" />
  </ClientOnly>
</template>
```

官方这里的两个细节都别丢：`useTemplateRef` + `watch` 的组合，以及 `{ once: true }`（只需要在元素出现的那一次响应）。**直接在 `onMounted` 里读那个 ref 是不可行的**——那时里面的组件还没渲染出来。

## 别把它和这两个搞混

| 你以为 | 实际 |
| --- | --- |
| `ssr: false` 和 `<ClientOnly>` 是一回事 | `ssr: false` 是**整站**的渲染模式开关（官方："你可以在 `nuxt.config.ts` 里启用仅客户端渲染"）；`<ClientOnly>` 只圈它包住的这一块 |
| 水合不匹配应该用 `<ClientOnly>` 修 | 官方在这里给的定位是"专门让组件只在客户端渲染"；水合匹配的通用写法在 Vue 文档与 `import.meta.client` 那一侧，`<ClientOnly>` 是"整块不进服务端"的手段 |
| 里面有内容不影响首屏 | 默认插槽内容被 tree-shake 出服务端构建，其 CSS 也可能不进初始 HTML |

## 自查

1. `<ClientOnly>` 的默认插槽会发生什么？官方由此点明的副作用是哪一句？
2. `placeholder` 与 `#fallback` 插槽分别适合放什么？官方示例里那两个 prop 是别名关系吗？
3. 为什么"有副作用的库"必须靠 `<ClientOnly>` 挡，而不是靠打包器自动甩掉？
4. 预渲染的兜底错误页里，哪一类内容该包进 `<ClientOnly>`？官方还配了哪个开关来跳过取数？
5. 想访问 `<ClientOnly>` 里的 DOM，官方的写法是什么？为什么不能直接在 `onMounted` 里读？

## 下一篇

097 讲 `<DevOnly>`：它的内容为什么不会进生产构建、`#fallback` 那一支在生产里顶替什么，以及官方那句"一定要用 `nuxt preview` 来测这一支"。

## 本文依据

- `https://nuxt.com/docs/4.x/api/components/client-only`
  - 原文（标题与描述）："Render components only in client-side with the \<ClientOnly> component."；"The `<ClientOnly>` component is used for purposely rendering a component only on client side."
  - 原文（note）："The content of the default slot will be tree-shaken out of the server build. (This does mean that any CSS used by components within it may not be inlined when rendering the initial HTML.)"
  - 原文（Props）："`placeholderTag` | `fallbackTag`: specify a tag to be rendered server-side."；"`placeholder` | `fallback`: specify a content to be rendered server-side."
  - 原文示例：`<ClientOnly fallback-tag="span" fallback="Loading comments..."><Comment /></ClientOnly>`，注释 `<!-- The <Comment> component will only be rendered on client-side -->`
  - 原文（Slots）："`#fallback`: specify a content to be rendered on the server and displayed until `<ClientOnly>` is mounted in the browser."
  - 原文示例：`<ClientOnly fallback-tag="span">` 三条注释 `<!-- This renders the "span" element on the server side -->`、`<!-- this component will only be rendered on client side -->`、`<!-- this will be rendered on server side -->`，插槽内 `<p>Loading comments...</p>`
  - 原文（Accessing HTML Elements）："Components inside `<ClientOnly>` are rendered only after being mounted. To access the rendered elements in the DOM, you can watch a template ref:"；示例 `useTemplateRef('nuxtWelcomeRef')`、`// The watch will be triggered when the component is available`、`console.log('<NuxtWelcome /> mounted')`、`}, { once: true })`、`<ClientOnly><NuxtWelcome ref="nuxtWelcomeRef" /></ClientOnly>`
  - 原文 Source：`packages/nuxt/src/app/components/client-only.ts`
- `https://nuxt.com/docs/4.x/guide/concepts/rendering`
  - 原文（示例注释）：`const counter = ref(0) // executes in server and client environments`；`counter.value++ // executes only in a client environment`
  - 原文："On the initial request, the `counter` ref is initialized in the server since it is rendered inside the `<p>` tag. The contents of `handleClick` is never executed here. During hydration in the browser, the `counter` ref is re-initialized. The `handleClick` finally binds itself to the button; Therefore it is reasonable to deduce that the body of `handleClick` will always run in a browser environment."
  - 原文："Middlewares and pages run in the server and on the client during hydration. Plugins can be rendered on the server or client or both. **Components can be forced to run on the client only as well.** Composables and utilities are rendered based on the context of their usage."
  - 原文（important）："When importing a library that relies on browser APIs and has side effects, make sure the component importing it is only called client-side. Bundlers do not treeshake imports of modules containing side effects."
  - 原文（预渲染兜底页）："Because a single file is served for every missing path, the error page cannot depend on the request: `useRoute()`, `useRequestURL()` and data fetched with the current path will hold the build-time values in the prerendered HTML and be corrected on hydration. Wrap request-specific markup in `<ClientOnly>`, and skip request-specific data fetching with `import.meta.prerender`:"
  - 原文示例（`error.vue`）：`<h1>{{ error.statusCode }}</h1>` 与 `<ClientOnly><p>Nothing found at {{ route.path }}</p></ClientOnly>`
  - 原文："You can enable client-side only rendering with Nuxt in your `nuxt.config.ts`:" + `export default defineNuxtConfig({ ssr: false, })`
  - 原文（tip）："For more examples about writing Vue code without hydration mismatch, see the Vue docs."
