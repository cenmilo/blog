---
url: /posts/e3fb70bf061f39c3a88b881f9cd32e06/
title: 165 - 只在客户端渲染组件的三条路：ClientOnly、client 后缀与整页关 SSR
date: 2026-12-06T09:00:00+08:00
updated: 2026-12-06T09:00:00+08:00
author: cmdragon

summary: >-
  摘要："不让某个组件在服务端渲染"这件事，官方给了三个层次的做法，粒度与代价完全不同。**第一层是包一个组件**：`<ClientOnly>`——"**The `<ClientOnly>` component is used for purposely rendering a component only on client side.**"，代价与性质写在第一条 note 里："**The content of the default slot will be tree-shaken out of the server build. (This does mean that any CSS used by components within it may not be inlined when rendering the initial HTML.)**"，占位有两对 prop（"`placeholderTag` | `fallbackTag`：specify a tag to be rendered server-side"、"`placeholder` | `fallback`：specify a content to be rendered server-side"）与一个 `#fallback` 插槽（"specify a content to be rendered on the server and **displayed until `<ClientOnly>` is mounted in the browser**"）。**第二层是改文件名**：`.client` 后缀——"If a component is meant to be rendered only client-side, you can add the `.client` suffix to your component."，两条限制分别是"**This feature only works with Nuxt auto-imports and `#components` imports. Explicitly importing these components from their real paths does not convert them into client-only components.**" 与 "`\.client` components are rendered only after being mounted. To access the rendered template using `onMounted()`, add `await nextTick()` in the callback of the `onMounted()` hook."。**第三层是关整页**：`ssr: false`，两种写法与一句连带建议——整页 `export default defineNuxtConfig({ ssr: false })` 与按路由 `'/admin/**': { ssr: false }`（官方定义："**Disables server-side rendering of the HTML for sections of your app and make them render only in the browser with `ssr: false`**"），以及"If you do use `ssr: false`, you should also place an HTML file in `~/spa-loading-template.html`..."。三者之外还有一条容易混的反方向做法：`noScripts` 是"**永不水合**"，与"只在客户端渲染"正好相反。

categories:
  - 前端开发

tags:
  - nuxt4
  - 客户端渲染
  - ClientOnly
  - SSR
  - 水合

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


用到一个浏览器才有的 API 时，我有过三种反应：套 `<ClientOnly>`、把文件改名加 `.client`、或者干脆整页关 SSR。这三条不是"任选其一"——粒度、限制、代价都不一样。这一篇把三条路各自的官方条件摆出来。

## 第一条路：包一层 `<ClientOnly>`

官方定位句：

> The `<ClientOnly>` component is used for purposely rendering a component only on client side.

它的性质与代价在同一条 note 里说完，括号那半句容易被跳过：

> The content of the default slot will be **tree-shaken out of the server build**. (This does mean that **any CSS used by components within it may not be inlined** when rendering the initial HTML.)

**读法**：默认插槽的内容不进服务端构建 → 服务端不认识它 → 首屏 HTML 里可能没有它的样式。这是"只在客户端"的定价，不是配置能改掉的。

占位有三套写法，官方分两节列（096 逐项拆过，这里只放对照）：

| 入口 | 官方定义 |
| --- | --- |
| `placeholderTag` \| `fallbackTag` | "specify a tag to be rendered server-side" |
| `placeholder` \| `fallback` | "specify a content to be rendered server-side" |
| `#fallback` 插槽 | "specify a content to be rendered on the server and **displayed until `<ClientOnly>` is mounted in the browser**" |

官方给的用法：

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

插槽那版更适合占位里有结构（标签、类名）；prop 那两版适合一行文字。

### 一个真实需求：拿渲染后的 DOM

官方给的 Accessing HTML Elements 例子，把时机这件事讲得很实用：

> Components inside `<ClientOnly>` are rendered **only after being mounted**. To access the rendered elements in the DOM, you can watch a template ref:

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

**注意官方选的组合是 `watch` + `{ once: true }`**，而不是在 `onMounted` 里直接读 ref——因为内容是挂载之后才出现的。第二条路也有同款限制，值得对照着记。

## 第二条路：文件名加 `.client`

官方定义与例子：

> If a component is meant to be rendered only client-side, you can add the `.client` suffix to your component.

```bash [Directory Structure]
| components/
--| Comments.client.vue
```

```vue [app/pages/example.vue]
<template>
  <div>
    <!-- this component will only be rendered on client side -->
    <Comments />
  </div>
</template>
```

模板里写的还是 `<Comments />`——**后缀不改变标签名**。

两条限制，官方各用一个块写着：

> This feature **only works with Nuxt auto-imports and `#components` imports**. Explicitly importing these components from their real paths **does not convert them into client-only components**.

> `.client` components are rendered **only after being mounted**. To access the rendered template using `onMounted()`, **add `await nextTick()`** in the callback of the `onMounted()` hook.

第一条限制我认为更重要：**"改文件名的魔法只在自动导入这一层生效"**。如果你写成 `import Comments from '~/components/Comments.client.vue'`，它就变成一个普通组件、会在服务端渲染——症状是"我明明加了 `.client`，服务端还是报错"。这条与 154 那条（模块 runtime 必须走 `#components`/`#imports`）是同一条边界的两个方向。

第二条与 `<ClientOnly>` 的挂载时机是同一件事，只是解法不同：`<ClientOnly>` 里用 `watch(ref, ..., { once: true })`，`.client` 组件里官方建议 `onMounted` + `await nextTick()`。

官方也在 `<ClientOnly>` 那一节留了互指句："You can also achieve a similar result with the `<ClientOnly>` component."（反向从 096 那一节也指回来过）。

### 成对写法：服务端一份、客户端一份

> In this case, the `.server` + `.client` components are two 'halves' of a component and can be used in advanced use cases for separate implementations of a component on server and client side.

```bash [Directory Structure]
-| components/
---| Comments.client.vue
---| Comments.server.vue
```

```vue [app/pages/example.vue]
<template>
  <div>
    <!-- this component will render Comments.server on the server then Comments.client once mounted in the browser -->
    <Comments />
  </div>
</template>
```

**这条路的语义与"只在客户端"不同**：服务端仍然渲染一点东西（那份 `.server`），只是切换实现。还有一个连带后果在 169 会用到——`.server` + `.client` 成对时"the component is **not an island**: the client half hydrates normally"。

## 第三条路：整页或整段关 SSR

前两条是"某个组件不上服务端"，这一条是"这一页/这个应用不上服务端"。

按路由关（141 引过定义）：

> `ssr: boolean` - Disables server-side rendering of the HTML for sections of your app and make them render only in the browser with `ssr: false`

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    // Admin dashboard renders only on client-side
    '/admin/**': { ssr: false },
  },
})
```

整应用关：

> You can enable client-only rendering with Nuxt in your `nuxt.config.ts`:

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  ssr: false,
})
```

配套的连带建议官方写成一条 note：

> If you do use `ssr: false`, you should also place an HTML file in `~/spa-loading-template.html` with some HTML you would like to use to render a loading screen that will be rendered until your app is hydrated.

（这条模板文件我在 104 单独讲过：它只在 `ssr: false` 时生效、真实路径是 `app/spa-loading-template.html`。）

**代价与收益官方在这一节写得最坦白**，因为它讨论的是整种渲染模式。三条收益：

> - **Development speed**: When working entirely on the client-side, we don't have to worry about the server compatibility of the code, for example, by using browser-only APIs like the `window` object.
> - **Cheaper:** ... We can host client-only applications on any static server with HTML, CSS, and JavaScript files.
> - **Offline:** Because code entirely runs in the browser, it can nicely keep working while the internet is unavailable.

两条代价：

> - **Performance**: The user has to wait for the browser to download, parse and run JavaScript files... this can take some time and impact the user's experience.
> - **Search Engine Optimization**: Indexing and updating the content delivered via client-side rendering takes more time than with a server-rendered HTML document... as search engine crawlers won't wait for the interface to be fully rendered on their first try to index the page.

官方的适用判断句值得整段记住，它把"什么时候关 SSR 是对的"说成了一道题：

> Client-side rendering is a good choice for heavily interactive **web applications** that don't need indexing or whose users visit frequently. It can leverage browser caching to skip the download phase on subsequent visits, such as **SaaS, back-office applications, or online games**.

关 SSR 还有一项连带好处在 141 讲过：`ssr: false` 覆盖的路由，其页面组件被排除出服务端 bundle（三种例外见那篇）。

## 反方向的第四个词：`noScripts`

这个很容易和"客户端渲染"混，但它是**相反**的意思，官方定义是"Disables rendering of Nuxt scripts and JS resource hints"，后果是"the page simply never hydrates, which means nothing on it is interactive"（141 详述）。

| 手段 | 服务端 | 客户端 |
| --- | --- | --- |
| `<ClientOnly>` / `.client` | **不渲染**这个组件 | 渲染并可用 |
| `ssr: false`（整页） | 不渲染 HTML | 渲染并可用 |
| `noScripts` | **渲染 HTML** | **不水合**，交互全无 |

还有一条属于第 4 种场景的实践句，官方在静态托管那节给的是"包裹请求相关内容"：

> Because a single file is served for every missing path, the error page cannot depend on the request: `useRoute()`, `useRequestURL()` and data fetched with the current path will hold the build-time values in the prerendered HTML and be corrected on hydration. **Wrap request-specific markup in `<ClientOnly>`**, and skip request-specific data fetching with `import.meta.prerender`.

这是 `<ClientOnly>` 少数"不是为了浏览器 API，而是为了内容正确"的用法。

## 三条路怎么选

| 症状/需求 | 用 |
| --- | --- |
| 组件里用了 `window`、`document` 等浏览器 API | `<ClientOnly>`（单个位置）或 `.client.vue`（整个组件都这样） |
| 同一个组件服务端要出一份简化版、客户端要交互版 | `.server.vue` + `.client.vue` 成对 |
| 首屏里有一块内容跟请求相关（预渲染的 200/404） | `<ClientOnly>` 包住那块 |
| 整段后台都不需要 SEO、用户天天来 | `routeRules` 给那段 `ssr: false` |
| 整个应用是内部工具、离线也要能用 | 顶层 `ssr: false` + spa-loading-template |
| 只是想让某页不下载 JS（内容型页面） | 那是 `noScripts`，方向相反，别用前三条 |

## 自查

1. `<ClientOnly>` 的默认插槽内容会从哪里被摇掉？由此带来的样式后果是哪半句？
2. 占位有三套写法，`#fallback` 插槽的显示条件官方怎么说？
3. `.client` 后缀的两条限制分别是什么？为什么"按真实路径 import"会让它失效？
4. `<ClientOnly>` 里取 DOM 与 `.client` 组件里取 DOM，官方给的两种解法分别是什么？
5. `.server` + `.client` 成对时，那个组件还是岛屿吗？依据是哪句？
6. 整应用关 SSR 与按路由关 SSR 分别怎么写？关 SSR 的三条收益、两条代价官方分别怎么说？
7. 官方认为纯客户端渲染适合哪三类应用？判断依据是那两句话里的哪些条件？
8. `noScripts` 与前三条的方向差在哪？表格里的三列分别是什么？
9. 预渲染的静态错误页为什么需要 `<ClientOnly>`？官方给的另一个配套手段是什么？

## 下一篇

166 讲 `defineLazyHydrationComponent`：七种策略怎么填、每种对应模板里哪个 prop，以及它和 `Lazy` 前缀的分工差别。

## 本文依据

- `https://nuxt.com/docs/4.x/api/components/client-only`
  - 原文（定位与 note）："Render components only in client-side with the `<ClientOnly>` component."；"The `<ClientOnly>` component is used for purposely rendering a component only on client side."；note "The content of the default slot will be tree-shaken out of the server build. (This does mean that any CSS used by components within it may not be inlined when rendering the initial HTML.)"
  - 原文（Props 与 Slots）："`placeholderTag` | `fallbackTag`: specify a tag to be rendered server-side."；"`placeholder` | `fallback`: specify a content to be rendered server-side."；"`#fallback`: specify a content to be rendered on the server and displayed until `<ClientOnly>` is mounted in the browser."；两个示例（`fallback-tag="span"` + `fallback="Loading comments..."`；`<ClientOnly fallback-tag="span">` 配 `<template #fallback>`）
  - 原文（Example / Accessing HTML Elements）："Components inside `<ClientOnly>` are rendered only after being mounted. To access the rendered elements in the DOM, you can watch a template ref:" + `const nuxtWelcomeRef = useTemplateRef('nuxtWelcomeRef')` + `watch(nuxtWelcomeRef, () => {...}, { once: true })` + `<ClientOnly><NuxtWelcome ref="nuxtWelcomeRef" /></ClientOnly>`
  - 原文（read-more）："You can also achieve a similar result with the `<ClientOnly>` component."（反向：components 目录页指过来）；Source `packages/nuxt/src/app/components/client-only.ts`
- `https://nuxt.com/docs/4.x/directory-structure/app/components`
  - 原文（Client Components）："If a component is meant to be rendered only client-side, you can add the `.client` suffix to your component."；`| components/ --| Comments.client.vue` 与 `<Comments />` 用法；note "This feature only works with Nuxt auto-imports and `#components` imports. Explicitly importing these components from their real paths does not convert them into client-only components."；important "`.client` components are rendered only after being mounted. To access the rendered template using `onMounted()`, add `await nextTick()` in the callback of the `onMounted()` hook."
  - 原文（Paired with a Client component）："In this case, the `.server` + `.client` components are two 'halves' of a component and can be used in advanced use cases for separate implementations of a component on server and client side." + `Comments.client.vue`/`Comments.server.vue` 与注释 "this component will render Comments.server on the server then Comments.client once mounted in the browser"
- `https://nuxt.com/docs/4.x/guide/concepts/rendering`
  - 原文（客户端渲染的取舍）："Out of the box, a traditional Vue.js application is rendered in the browser (or **client**)."；三条 Benefits（Development speed / Cheaper / Offline）与两条 Downsides（Performance / Search Engine Optimization）原文；"Client-side rendering is a good choice for heavily interactive **web applications** that don't need indexing or whose users visit frequently. It can leverage browser caching to skip the download phase on subsequent visits, such as **SaaS, back-office applications, or online games**."；"You can enable client-only rendering with Nuxt in your `nuxt.config.ts`:" + `ssr: false`；note "If you do use `ssr: false`, you should also place an HTML file in `~/spa-loading-template.html` with some HTML you would like to use to render a loading screen that will be rendered until your app is hydrated."
  - 原文（routeRules 里的 ssr）："`ssr: boolean` - Disables server-side rendering of the HTML for sections of your app and make them render only in the browser with `ssr: false`"；示例 `'/admin/**': { ssr: false }`；"Server Bundle Size with `ssr: false`" 段与三种例外
  - 原文（静态托管的错误页）："Because a single file is served for every missing path, the error page cannot depend on the request: `useRoute()`, `useRequestURL()` and data fetched with the current path will hold the build-time values in the prerendered HTML and be corrected on hydration. Wrap request-specific markup in `<ClientOnly>`, and skip request-specific data fetching with `import.meta.prerender`."
- `https://nuxt.com/docs/4.x/guide/going-further/features` 与 `https://nuxt.com/docs/4.x/guide/concepts/rendering`
  - 对照用（见 141）：`noScripts` 的定义句与"never hydrates / nothing is interactive"的后果
