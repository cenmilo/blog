---
url: /posts/e41cab52bbb038507e82722ba1a19366/
title: useSeoMeta：100+ 个类型化字段、getter 写法与"包一层 server"那一招
date: 2026-12-14T10:00:00+08:00
updated: 2026-12-14T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：官方定位句："**The `useSeoMeta` composable lets you define your site's SEO meta tags as a flat object with full TypeScript support.**"，它省掉的错误被逐条点名："This helps you avoid common mistakes, such as **using `name` instead of `property`**, as well as typos - **with over 100+ meta tags fully typed**."，并给了一句推荐级评价（important）："**This is the recommended way to add meta tags to your site as it is XSS safe and has full TypeScript support.**"。字段清单官方不在文档里展开："There are over 100 parameters. See the full list of parameters in the source code."（链接指向 `zhead/src/metaFlat.ts`）。**反应式写法这里换了一种**："> When inserting tags that are reactive, **you should use the computed getter syntax (`() => value`)**" + `useSeoMeta({ title, description: () => \`This is a description for the ${title.value} page\` })`。官方示例里六个字段的形状是：`title`、`ogTitle`、`description`、`ogDescription`、`ogImage`、`twitterCard: 'summary_large_image'`。**性能那一节是这一页最有用的判断**："In most instances, **SEO meta tags don't need to be reactive as search engine robots primarily scan the initial page load.**"，做法是把静态部分包进服务端条件：`if (import.meta.server) { useSeoMeta({ robots: 'index, follow', description: 'Static description that does not need reactivity', ogImage: 'https://example.com/image.png' }) }`，并配一句"Only use reactive meta tags outside the condition when necessary"；最后一句是版本信息："**This previously used the `useServerSeoMeta` composable, but it has been deprecated in favor of this approach.**"（191 处理那个已废弃项）。

categories:
  - 前端开发

tags:
  - nuxt4
  - useSeoMeta
  - SEO
  - 类型安全
  - 性能

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


我第一版 OG 标签是用 `useHead` 手搓 `meta` 数组的，`og:title` 写成 `name="og:title"` 也照样渲染出来——但抓取时就是不生效。`useSeoMeta` 解决的正是这类错误：**它把"该用 name 还是 property"这件事变成类型**。这一篇写它的形状、反应式写法和一条省水合的判断。

## 定位：扁平对象 + 全类型化

官方三句一起读：

> The `useSeoMeta` composable lets you define your site's SEO meta tags as a **flat object** with full TypeScript support.

> This helps you avoid common mistakes, such as **using `name` instead of `property`**, as well as typos - with over 100+ meta tags fully typed.

> **This is the recommended way to add meta tags to your site as it is XSS safe and has full TypeScript support.**

三条对应三种收益：

| 收益 | 机制 |
| --- | --- |
| 不会把 `og:title` 写成 `name` | 字段是 `ogTitle`，`name`/`property` 的选择由框架决定，你写不到错的位置 |
| 不会拼错 | 100+ 个字段全部有类型 |
| 值本身是安全的 | "it is XSS safe"——对照 189：它没有给你自由属性通道 |

**注意"flat"这个词**：不是 `meta: [{...}]` 数组，而是一层键值对象。

## 官方示例：六个字段一条 OG

```vue [app/app.vue]
<script setup lang="ts">
useSeoMeta({
  title: 'My Amazing Site',
  ogTitle: 'My Amazing Site',
  description: 'This is my amazing site, let me tell you all about it.',
  ogDescription: 'This is my amazing site, let me tell you all about it.',
  ogImage: 'https://example.com/image.png',
  twitterCard: 'summary_large_image',
})
</script>
```

从这个形状能读出三件事：

1. **它不只写 meta**：`title` 也在管（与 188 的 `title` 同一个效果）。
2. **OG 与 Twitter 是并列的字段族**：`ogTitle` / `ogDescription` / `ogImage` / `twitterCard`（映射表见 192）。
3. **`twitterCard` 的取值是字符串字面量**，示例用的是 `summary_large_image`。

字段清单官方明确不在文档里给，只留了出处：

> There are over 100 parameters. See the full list of parameters in the source code.

链接指向 `zhead/src/metaFlat.ts`。**所以"某字段支持吗"的权威答案是编辑器补全 + 那个源文件**，不是文档正文。我这一篇也不去抄一份可能过期的清单。

## 反应式：这里要写 getter

`useHead` 允许字段值直接放 `computed`（187），`useSeoMeta` 给的是另一种：

> When inserting tags that are reactive, you should use the **computed getter syntax (`() => value`)**:

```vue [app/app.vue]
<script setup lang="ts">
const title = ref('My title')

useSeoMeta({
  title,
  description: () => `This is a description for the ${title.value} page`,
})
</script>
```

两个细节：`title` 直接放 ref 是示例本身写的，`description` 用 getter 是官方推荐形态；**同一页 Performance 一节里也是这个写法**（`useSeoMeta({ title: () => dynamicTitle.value, ogTitle: () => dynamicTitle.value })`）。

SEO 那页的反应式 code-group 用的又是 `useSeoMeta({ description })`（ref 直传）。**三处写法能同时成立说明两种都吃**，但官方带说明的那句指向 getter，所以我默认写 `() => value`。这与 `useHead` 的差别值得记：**同一个"反应式"在两个 composable 里推荐形态不一样。**

## 一条性能判断：大部分 SEO 标签不需要反应式

这一节我认为是整页最有用的部分，先给理由再给做法：

> In most instances, SEO meta tags **don't need to be reactive** as search engine robots primarily scan the initial page load.

> For better performance, you can wrap your `useSeoMeta` calls in a server-only condition when the meta tags don't need to be reactive:

```vue [app/app.vue]
<script setup lang="ts">
if (import.meta.server) {
  // These meta tags will only be added during server-side rendering
  useSeoMeta({
    robots: 'index, follow',
    description: 'Static description that does not need reactivity',
    ogImage: 'https://example.com/image.png',
    // other static meta tags...
  })
}

const dynamicTitle = ref('My title')
// Only use reactive meta tags outside the condition when necessary
useSeoMeta({
  title: () => dynamicTitle.value,
  ogTitle: () => dynamicTitle.value,
})
</script>
```

拆开看这个模式：

| 位置 | 内容 | 效果 |
| --- | --- | --- |
| `if (import.meta.server)` 内 | 静态、爬虫要看的（`robots`、`description`、`ogImage`） | 只在服务端加入，客户端不再重复维护 |
| 外面 | 真会变的（示例里是 `title`/`ogTitle`） | 保持反应式 |

两句行注释都是官方写的，等于给了使用规则："These meta tags will only be added during server-side rendering" 与 "Only use reactive meta tags outside the condition when necessary"。

**这套做法的来源是替代品**，官方一句话交代：

> This previously used the `useServerSeoMeta` composable, but it **has been deprecated in favor of this approach**.

也就是说：**新代码不要再找 `useServerSeoMeta`**（191 专门讲这个已废弃项与它的自动导入去除条件）。

## 什么时候不用它

| 需求 | 该用 |
| --- | --- |
| SEO 元信息（title/description/OG/Twitter/robots…） | `useSeoMeta`，官方推荐位 |
| 值来自用户或 CMS 的**任意**字段 | 仍然安全（它没有自由属性通道）；若要写 `script`/`link` 则 `useHeadSafe`（189） |
| 需要 `<link rel="canonical">`、`<meta charset>` 这类**非 meta 标签** | `useHead`（188 的字段表里有 `link`、`base`、`script`…，`useSeoMeta` 不覆盖这些元素类型） |
| 需要 `titleTemplate` 的函数形态 / `templateParams` | `useHead`（188） |
| 全站静态默认值 | `app.head`（186） |

**最后一行的边界我要标一下**：文档没有一句"canonical 只能用 `useHead`"，我是从 `useSeoMeta` 的定位（"SEO meta tags"、扁平字段）与 `useHead` 的字段表对照得出的。真要写 `<link rel="canonical">`，用 `useHead` 的 `link` 数组是有官方示例支撑的那条路（196/204 那条线会再用到）。

## 自查

1. 官方给 `useSeoMeta` 的三句（定位、避免的错误、推荐评价）分别怎么说？"flat object"相对什么而言？
2. 它避免的两类具体错误是什么？为什么它自称 XSS safe？
3. 字段清单为什么不在文档里？官方给的权威出处是哪一处？
4. 这里推荐的反应式写法是什么？与 `useHead` 的 `computed` 写法差别在哪？官方三种写法各出现在哪？
5. "SEO 标签大多不需要反应式"的理由句是什么？官方给的两段代码分别放哪类字段？
6. `useServerSeoMeta` 现在应该怎么写？官方哪句话说明了这个替换？
7. 哪些 head 内容 `useSeoMeta` 管不到？你怎么确认这条边界？

## 下一篇

191 讲那个已经被废弃的 `useServerSeoMeta`：它的原语义（只服务端运行、从客户端包摇掉）、官方给的替代写法，以及"在 `future.compatibilityVersion: 5` 下自动导入被移除"这句的确切含义。

## 本文依据

- `https://nuxt.com/docs/4.x/api/composables/use-seo-meta`（本篇主要依据）
  - 原文（定位与推荐）："The useSeoMeta composable lets you define your site's SEO meta tags as a flat object with full TypeScript support."；"This helps you avoid common mistakes, such as using `name` instead of `property`, as well as typos - with over 100+ meta tags fully typed."；important "This is the recommended way to add meta tags to your site as it is XSS safe and has full TypeScript support."
  - 原文（Usage 与反应式）：`useSeoMeta({ title: 'My Amazing Site', ogTitle: 'My Amazing Site', description: '...', ogDescription: '...', ogImage: 'https://example.com/image.png', twitterCard: 'summary_large_image' })`；"When inserting tags that are reactive, you should use the computed getter syntax (`() => value`):" + `useSeoMeta({ title, description: () => \`This is a description for the ${title.value} page\` })`
  - 原文（Parameters 与 Performance）："There are over 100 parameters. See the full list of parameters in the source code."（链接 `github.com/harlan-zw/zhead/blob/main/packages/zhead/src/metaFlat.ts#L1035`）；"In most instances, SEO meta tags don't need to be reactive as search engine robots primarily scan the initial page load."；"For better performance, you can wrap your `useSeoMeta` calls in a server-only condition when the meta tags don't need to be reactive:" + `if (import.meta.server) { useSeoMeta({ robots: 'index, follow', description: 'Static description that does not need reactivity', ogImage: 'https://example.com/image.png', // other static meta tags... }) }` + `const dynamicTitle = ref('My title')` + `useSeoMeta({ title: () => dynamicTitle.value, ogTitle: () => dynamicTitle.value })` 与两处注释；"This previously used the `useServerSeoMeta` composable, but it has been deprecated in favor of this approach."；Source `packages/vue/src/composables.ts`（unhead 仓库）
- `https://nuxt.com/docs/4.x/getting-started/seo-meta`
  - 原文（同一示例与反应式 code-group）："The `useSeoMeta` composable lets you define your site's SEO meta tags **as an object** with full type safety."；"This helps you avoid typos and common mistakes, such as using `name` instead of `property`."；六字段示例；Reactivity 一栏 `const description = ref('My amazing site.')` + `useSeoMeta({ description })`
- `https://nuxt.com/docs/4.x/api/composables/use-server-seo-meta`
  - 对照用（见 191）：warning "`useServerSeoMeta` is deprecated. Wrap `useSeoMeta` in an `if (import.meta.server)` block instead. The auto-import is removed under `future.compatibilityVersion: 5`."
- `https://nuxt.com/docs/4.x/api/composables/use-head`
  - 对照用（见 187、188）：`link`/`base`/`script`/`style`/`noscript` 元素族与 `titleTemplate`、`templateParams` 归属 `useHead`
