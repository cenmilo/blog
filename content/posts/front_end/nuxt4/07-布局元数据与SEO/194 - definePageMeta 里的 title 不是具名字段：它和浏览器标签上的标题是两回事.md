---
url: /posts/dc504d174644a550f8a77c4cd0062670/
title: 194 - definePageMeta 里的 title 不是具名字段：它和浏览器标签上的标题是两回事
date: 2026-12-15T11:00:00+08:00
updated: 2026-12-15T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇纠正我自己一个想当然：以为 `definePageMeta({ title: '...' })` 会改浏览器标签页上的标题。把 `PageMeta` 接口逐行数一遍，具名字段是 15 个——`validate`、`redirect`、`name`、`path`、`props`、`alias`、`groups`、`pageTransition`、`layoutTransition`、`viewTransition`、`key`、`keepalive`、`layout`、`middleware`、`scrollToTop`，最后一行是 `[key: string]: unknown`，**`title` 不在那 15 个里，它是从这个索引签名进来的自定义键**。文档里 `title` 的用法只有一处：SEO 页的 "With `definePageMeta`" 示例把 `title: 'Some Page'` 写进页面、再在布局里用 `route.meta.title` 去拼 `og:title`，也就是说它是一份**给别的组件读的路由数据**，而 `<title>` 的 documented 来源是另外三处——`app.head.title`（注释写成 "default fallback title"）、`useHead({ title })`（"Sets the page title."）、以及 `titleTemplate`。另外两处容易混的：`scanPageMeta` 在构建期只暴露固定六个键（`alias`、`name`、`path`、`redirect`、`props`、`middleware`），`title` 要模块在构建期读到得靠 `extraPageMetaExtractionKeys`；而 `extractSerializablePageMeta` 的官方示例里恰好就写着 `title: 'Dashboard'`。

categories:
  - 前端开发

tags:
  - nuxt4
  - definePageMeta
  - 页面标题
  - route meta
  - SEO

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


我刚学的时候把 `definePageMeta` 当成"这个页面的所有设置都写这儿"，所以理所当然地认为 `title` 就是页面标题。直到有次发现标签页上的字纹丝不动，才回去把接口一行行看。

## 先数一遍 PageMeta 到底有哪些具名字段

`definePageMeta` 页给的签名和接口是这样的（逐行照抄）：

```ts [Signature]
export function definePageMeta (meta: PageMeta): void

interface PageMeta {
  validate?: ((route: RouteLocationNormalized) => boolean | Promise<boolean> | Partial<NuxtError> | Promise<Partial<NuxtError>>)
  redirect?: RouteRecordRedirectOption
  name?: string
  path?: string
  props?: RouteRecordRaw['props']
  alias?: string | string[]
  groups?: string[]
  pageTransition?: boolean | TransitionProps
  layoutTransition?: boolean | TransitionProps
  viewTransition?: ViewTransitionPageOptions['enabled'] | ViewTransitionPageOptions
  key?: false | string | ((route: RouteLocationNormalizedLoaded) => string)
  keepalive?: boolean | KeepAliveProps
  layout?: false | LayoutKey | Ref<LayoutKey> | ComputedRef<LayoutKey> | { name?: LayoutKey | false, props?: Record<string, unknown> /* or the selected layout's props */ }
  middleware?: MiddlewareKey | NavigationGuard | Array<MiddlewareKey | NavigationGuard>
  scrollToTop?: boolean | ((to: RouteLocationNormalizedLoaded, from: RouteLocationNormalizedLoaded) => boolean)
  [key: string]: unknown
}
```

具名字段 15 个，倒数第二行是 `scrollToTop`，最后一行是索引签名 `[key: string]: unknown`。**`title` 一个都没有出现在这 15 个里。** 它能写、也不报错，靠的就是那个索引签名——这也是文档为什么专门有一句"自定义元数据"的说法（`pages` 页的 "typing custom metadata" 那节，194 只在需要时引它，不重讲）。

## 那文档里的 title 用在哪

只有一处，就是 SEO 页的 `With definePageMeta` 小节。原文连着说：

> Within your [`app/pages/` directory], you can use `definePageMeta` along with [`useHead`] to set metadata based on the current route.
>
> For example, you can first set the current page title (this is extracted at build time via a macro, so it can't be set dynamically):

```vue [pages/some-page.vue]
<script setup lang="ts">
definePageMeta({
  title: 'Some Page',
})
</script>
```

> And then in your layout file, you might use the route's metadata you have previously set:

```vue [layouts/default.vue]
<script setup lang="ts">
const route = useRoute()

useHead({
  meta: [{ property: 'og:title', content: `App Name - ${route.meta.title}` }],
})
</script>
```

这个例子的读法很关键：`title` 写进路由 meta，然后**由布局把它读出来，塞给 `og:title`**。整段里没有任何一句说它会变成 `<title>`；`og:title` 是 `useHead` 现场生成的。文档在这里给的是一个"数据从页面流到布局"的通道，不是一个标题开关。

顺带把括号里那句也记住：**(this is extracted at build time via a macro, so it can't be set dynamically)**。`definePageMeta` 是编译器宏，宏的参数会被提升到组件外面，所以这里只能是静态值——这一点和 117 那篇讲的宏限制是同一条。

## 标签页上的标题，文档写的来源是这三处

| 来源 | 文档原话 | 特点 |
| --- | --- | --- |
| `app.head.title` | 配置示例里的注释 `// default fallback title` | 静态，"does not allow you to provide reactive data" |
| `useHead({ title })` | 参数表 "Sets the page title." | 支持 `ref`/`computed`/函数 |
| `titleTemplate` | "Configures a dynamic template to customize the page title. Can be a string with `%s` placeholder or a function." | 函数形态不能放 `nuxt.config`，要放在 `app.vue` |

`app.head.title` 那条注释我觉得起名字起得很准：它是**兜底**值。真正让标签页文字变起来的是 `useHead` 的 `title` 加上 `titleTemplate` 的组合，187/188 已经细讲过。

## 构建期能读到谁：另一个容易和 title 打架的清单

`scanPageMeta` 那节给的是固定六个键：

> Nuxt exposing some route metadata defined in `definePageMeta` at build-time to modules (specifically `alias`, `name`, `path`, `redirect`, `props` and `middleware`).
>
> This only works with static or strings/arrays rather than variables or conditional assignment.

`title` 不在这份清单里。如果你的模块想在构建期读到它，文档给的口子是 `extraPageMetaExtractionKeys`：

> This option allows passing additional keys to extract from the page metadata when using `scanPageMeta`.

它的官方示例提取的键叫 `foo`（示例里 `definePageMeta({ foo: 'bar' })` 配 `extraPageMetaExtractionKeys: ['foo']`，然后在 `hooks: { 'pages:resolved' (ctx) { /* ✅ foo is available */ } }` 里读），并且补了一句："If you are using this within a module, it's recommended also to augment the `NuxtPage` types with your keys."

有意思的是紧挨着的 `extractSerializablePageMeta` 一节，示例里偏偏就用了 `title`：

```vue
<script lang="ts" setup>
definePageMeta({
  layout: { name: 'admin', props: { collapsed: true } },
  title: 'Dashboard',
})
</script>
```

配合它上面那句"By default, Nuxt only reads a fixed list of `definePageMeta()` keys at build time (plus anything in `extraPageMetaExtractionKeys`); everything else is resolved at runtime from a separate module generated per page."——这两节放一起能读出的差别是：前者讲的是**暴露给模块的固定清单**，后者讲的是**能不能把可 JSON 序列化的属性直接写进路由记录**。所以 `title` 在 `scanPageMeta` 的清单外、却出现在 `extractSerializablePageMeta` 的示例里，这两件事不矛盾，但它们各自"读得到"的场合不一样：**要在 `pages:resolved` 里拿到 `title`，`extraPageMetaExtractionKeys: ['title']` 这一行不能省**。这是我按两节原文推的，文档没有一处把这两句话写在一起，所以我标成需要实测：加上那个配置，在 `pages:resolved` 钩子里打印 `ctx.pages.meta.title`（或你的递归遍历结果），看有没有值。

## 自查

1. `PageMeta` 的具名字段有几个，`title` 是其中之一吗？（答：15 个，`title` 不是，它走 `[key: string]: unknown`。）
2. 文档给 `title` 设计的用途是什么？（答：作为路由 meta 被别的组件读——示例是布局里读 `route.meta.title` 拼 `og:title`。）
3. 想让模块在构建期读到自定义键，要配什么？（答：`experimental.extraPageMetaExtractionKeys`，并在 `pages:resolved` 里读；同时建议补 `NuxtPage` 的类型增强。）

## 下一篇

195 - head 标签的去重与清理：key、patch 与 dispose 三处原文

## 本文依据

- `api/utils/define-page-meta`：`PageMeta` 接口全文（15 个具名字段 + `[key: string]: unknown`）；页首定位句 "a compiler macro that you can use to set metadata for your **page** components located in the [`app/pages/`] directory (unless set otherwise)"。
- `getting-started/seo-meta`：`With definePageMeta` 小节三句原文与两段示例（`title: 'Some Page'` 与 `layouts/default.vue` 里的 `content: \`App Name - ${route.meta.title}\``）；`app.head` 示例里的 `title: 'Nuxt', // default fallback title` 注释；important 框 "This method does not allow you to provide reactive data. We recommend using `useHead()` in `app.vue`."。
- `api/composables/use-head`：参数表 `title` 行 "Sets the page title."；`titleTemplate` 行 "Configures a dynamic template to customize the page title. Can be a string with `%s` placeholder or a function."。
- `guide/going-further/experimental-features`：`scanPageMeta` 节 "specifically `alias`, `name`, `path`, `redirect`, `props` and `middleware`" 与 "This only works with static or strings/arrays rather than variables or conditional assignment."；`extraPageMetaExtractionKeys` 节 "This option allows passing additional keys to extract from the page metadata when using `scanPageMeta`." 与 `foo: 'bar'` 示例；`extractSerializablePageMeta` 节 "By default, Nuxt only reads a fixed list of `definePageMeta()` keys at build time (plus anything in `extraPageMetaExtractionKeys`); everything else is resolved at runtime from a separate module generated per page." 与含 `title: 'Dashboard'` 的示例。
