---
url: /posts/85af453c8943950f0a21bc0d18eb919a/
title: 188 - useHead 十个字段：title 一族、九类标签与两个 attrs
date: 2026-12-13T11:00:00+08:00
updated: 2026-12-13T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇是 `useHead` 的字段对照表，全部按官方 Parameters 表原文抄。**十个字段**：`title`（`string`，"Sets the page title."）、`titleTemplate`（`string \| ((title?: string) => string)`，"**Configures a dynamic template to customize the page title. Can be a string with `%s` placeholder or a function.**"）、`base`（`Base`，"Sets the `<base>` tag for the document."）、`link`/`meta`/`style`/`script`/`noscript`（都是数组，官方对每一行的句式都一样："Array of xxx objects. **Each element is mapped to a `<xxx>` tag, where object properties correspond to HTML attributes.**"）、`htmlAttrs`（"Sets attributes of the `<html>` tag. Each object property is mapped to the corresponding attribute."）与 `bodyAttrs`（同句式，作用于 `<body>`）。**标题一族有三段规则**：`titleTemplate` 可以是含 `%s` 的字符串或函数；"**If you want to use a function (for full control), then this cannot be set in your `nuxt.config`. It is recommended instead to set it within your `app.vue` file where it will apply to all pages on your site**"；效果句"Now, if you set the title to `My Page` with `useHead` on another page of your site, the title would appear as 'My Page - Site Title' in the browser tab. **You could also pass `null` to default to 'Site Title'.**"；再加一层 `templateParams`："You can use `templateParams` to provide additional placeholders in your `titleTemplate` besides the default `%s`"，官方示例是 ``titleChunk ? \`${titleChunk} %separator %siteName\` : '%siteName'`` 配 `{ siteName: 'Site Title', separator: '-' }`）。**位置选项只有一个**："> You can use the `tagPosition: 'bodyClose'` option on applicable tags to append them to the end of the `<body>` tag."，注释写明合法值三种：`'head' | 'bodyClose' | 'bodyOpen'`。最后一处**两节不一致**要留意：`api/composables/use-head` 的 `MetaObject` 是十项，`getting-started/seo-meta` 的同名接口多一项 `templateParams`。

categories:
  - 前端开发

tags:
  - nuxt4
  - useHead
  - titleTemplate
  - tagPosition
  - head

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


`useHead` 的字段名一眼能懂，但里面有三个坑：`titleTemplate` 的函数形态**不能写在 `nuxt.config`**、想让标签落进 `<body>` 只有一个专门的选项名、以及"对象属性=HTML 属性"这句话对九类标签各成立一次。这一篇当字典用。

## 十个字段：官方表原文

`meta`（Parameters 里那个对象）支持的十个键：

| 字段 | 类型 | 官方定义 |
| --- | --- | --- |
| `title` | `string` | Sets the page title. |
| `titleTemplate` | `string \| ((title?: string) => string)` | Configures a dynamic template to customize the page title. **Can be a string with `%s` placeholder or a function.** |
| `base` | `Base` | Sets the `<base>` tag for the document. |
| `link` | `Link[]` | Array of link objects. Each element is mapped to a `<link>` tag, **where object properties correspond to HTML attributes**. |
| `meta` | `Meta[]` | Array of meta objects. Each element is mapped to a `<meta>` tag, where object properties correspond to HTML attributes. |
| `style` | `Style[]` | Array of style objects. Each element is mapped to a `<style>` tag, where object properties correspond to HTML attributes. |
| `script` | `Script[]` | Array of script objects. Each element is mapped to a `<script>` tag, where object properties correspond to HTML attributes. |
| `noscript` | `Noscript[]` | Array of noscript objects. Each element is mapped to a `<noscript>` tag, where object properties correspond to HTML attributes. |
| `htmlAttrs` | `HtmlAttributes` | Sets attributes of the `<html>` tag. **Each object property is mapped to the corresponding attribute.** |
| `bodyAttrs` | `BodyAttributes` | Sets attributes of the `<body>` tag. Each object property is mapped to the corresponding attribute. |

那句重复了五次的 "object properties correspond to HTML attributes" 是有用的：**`link` 元素里写 `crossorigin: ''` 就会成为 `<link crossorigin>`**，不需要框架为每个属性开一个字段。所以字段表里没有的东西（`rel`、`hreflang`、`integrity`…）不是"不支持"，而是走属性透传。

## 标题一族：三层规则

**第一层是 `titleTemplate` 的两种形态**，字符串版和函数版官方各给一例：

```vue [app/app.vue]
<script setup lang="ts">
useHead({
  // as a string,
  // where `%s` is replaced with the title
  titleTemplate: '%s - Site Title',
})
</script>
```

```vue [app/app.vue]
<script setup lang="ts">
useHead({
  // or as a function
  titleTemplate: (productCategory) => {
    return productCategory
      ? `${productCategory} - Site Title`
      : 'Site Title'
  },
})
</script>
```

**第二层是函数形态的位置限制**，官方写得很具体：

> If you want to use a function (for full control), **then this cannot be set in your `nuxt.config`. It is recommended instead to set it within your `app.vue` file where it will apply to all pages on your site.**

同一句在 SEO 页的另一处又强调了一遍：

> `nuxt.config` is also used as an alternative way of setting the page title. However, **`nuxt.config` does not allow the page title to be dynamic**. Therefore, it is recommended to use `titleTemplate` in the `app.vue` file to add a dynamic title, which is then applied to all routes of your Nuxt app.

**这两句和 186 那条 "This method does not allow you to provide reactive data" 是同一个约束的三个说法**——配置层不能放函数、也不能放响应式。

**第三层是效果与 `null`**：

> Now, if you set the title to `My Page` with `useHead` on another page of your site, the title would appear as 'My Page - Site Title' in the browser tab. You could also pass `null` to default to 'Site Title'.

**函数入参那个位置就是"当前页的标题片段"**，所以 `titleChunk` 为假值时的兜底要在函数里自己写（官方两例都写了 `: 'Site Title'`）。

### templateParams：`%s` 之外的占位

> You can use `templateParams` to provide additional placeholders in your `titleTemplate` besides the default `%s`. This allows for more dynamic title generation.

```vue
<script setup lang="ts">
useHead({
  titleTemplate: (titleChunk) => {
    return titleChunk ? `${titleChunk} %separator %siteName` : '%siteName'
  },
  templateParams: {
    siteName: 'Site Title',
    separator: '-',
  },
})
</script>
```

**注意例子里的混搭**：函数形态的 `titleTemplate` 里仍写着 `%separator`、`%siteName`——也就是说 `templateParams` 的占位符**在函数里也不会被自动插值成 JS 变量**，它仍是模板字符串那套。这一点官方只给了示例，没写成规则；我按示例照抄，不改成"自己拼接"那种写法（自己拼接当然也能跑，只是偏离官方形状）。

SEO 页的类型清单里，`templateParams` 的类型是 `Record<string, string | Record<string, string>>`——值可以是字符串，也可以是**一层嵌套记录**。

## 位置：只有 tagPosition 这一个开关

想让标签落在 `<body>` 末尾（典型是第三方脚本），官方给的选项名是 `tagPosition`：

> You can use the `tagPosition: 'bodyClose'` option on applicable tags to append them to the end of the `<body>` tag.

```vue
<script setup lang="ts">
useHead({
  script: [
    {
      src: 'https://third-party-script.com',
      // valid options are: 'head' | 'bodyClose' | 'bodyOpen'
      tagPosition: 'bodyClose',
    },
  ],
})
</script>
```

那句行注释把三个合法值一次列全：`'head' | 'bodyClose' | 'bodyOpen'`。**"on applicable tags"这个限定别忽略**——attrs 一族（`htmlAttrs`/`bodyAttrs`）不是标签，没有位置可言。

这里要和 186 那条对照着记：用组件写法时"`<Head>` 嵌套**不改变**渲染位置"；想改位置只能靠 `tagPosition`。**"写在哪个标签下面"与"落在文档哪里"是两件事。**

## 属性透传的实际用法：外部字体

官方"External CSS"一例同时演示了 `link` 数组、属性透传与 `crossorigin: ''`：

```vue
<script setup lang="ts">
useHead({
  link: [
    {
      rel: 'preconnect',
      href: 'https://fonts.googleapis.com',
    },
    {
      rel: 'stylesheet',
      href: 'https://fonts.googleapis.com/css2?family=Roboto&display=swap',
      crossorigin: '',
    },
  ],
})
</script>
```

`script` 一族同理，官方给的最小外部脚本形态是：

```vue
useHead({
  link: [{ rel: 'stylesheet', href: 'https://cdn.example.com/styles.css' }],
  script: [{ src: 'https://cdn.example.com/script.js', async: true }],
})
```

——`async: true` 直接成为 `<script async>`，这就是那句"properties correspond to HTML attributes"的落地。

## attrs 一族：htmlAttrs 与 bodyAttrs

官方"Body and HTML Attributes"一例把两个 attrs 字段与反应式（187）叠在一起：

```vue [app/pages/themed.vue]
<script setup lang="ts">
const isDark = ref(true)

useHead({
  htmlAttrs: {
    lang: 'en',
    class: computed(() => isDark.value ? 'dark' : 'light'),
  },
  bodyAttrs: {
    class: 'themed-page',
  },
})
</script>
```

**这一例是"用 head 管理主题类"的标准形状**：类挂在 `<html>` 上、随 `ref` 变。它与 186 的 `lang` 配置形成分工——静态语言进 `app.head.htmlAttrs.lang`，会变的 class 进 `useHead`。

## 一处两页不一致

| 位置 | `MetaObject` 成员 |
| --- | --- |
| `api/composables/use-head` 的 Type 段 | 十项：`title`、`titleTemplate`、`base`、`link`、`meta`、`style`、`script`、`noscript`、`htmlAttrs`、`bodyAttrs` |
| `getting-started/seo-meta` 的 Types 段 | 十一项：**多一个 `templateParams`** |

**所以 `templateParams` 在 composable 参考页的接口里没有、但它确实可用**（Features 一节有整段示例）。我按"可用 + 以 SEO 页的类型为准"处理；真要写进 `nuxt.config` 的话，回到上一节那条限制：函数形态不行，字符串 `%s` 形态可以。

## 一张速查

| 我要 | 用 |
| --- | --- |
| 站点名后缀、前缀 | `titleTemplate`（函数写进 `app.vue`） |
| 标题里带额外变量 | `templateParams` + `%name` 占位 |
| 单页描述、OG 标签 | `meta` 数组（或 190 的 `useSeoMeta`） |
| 文档级 `<base>` | `base` |
| 外部样式/字体 | `link` 数组，属性直接透传 |
| 第三方脚本、异步 | `script` 数组 + `async`/`src` |
| 脚本落到 body 末尾 | 该元素加 `tagPosition: 'bodyClose'` |
| 无 JS 时的内容 | `noscript` 数组 |
| `lang`、`dir`、根 class | `htmlAttrs` |
| body 上的 class | `bodyAttrs` |

## 自查

1. 十个字段里哪五个是数组？官方对它们用的是同一个句式吗？那句话解决了什么问题？
2. `titleTemplate` 的两种形态分别是什么？函数形态为什么不能写在 `nuxt.config`？官方在哪三处说了同一件事？
3. 函数版 `titleTemplate` 的入参是什么？假值时官方怎么处理？
4. `templateParams` 与 `%s` 的关系是什么？它的值可以是什么类型？示例里函数形态还写 `%separator` 说明什么？
5. 想让标签进 `<body>` 用哪个选项？三个合法值是什么？"on applicable tags"排除了什么？
6. `<link>` 元素里怎么加 `crossorigin`？这依据官方哪句规则？
7. `htmlAttrs` 与 `bodyAttrs` 各自作用在哪个标签？官方示例把什么状态挂在了 `htmlAttrs.class` 上？
8. 两页的 `MetaObject` 哪里不一致？你按哪一页处理？

## 下一篇

189 讲 `useHeadSafe`：它到底放行哪些属性（官方给的是六组白名单），`innerHTML` 与 `http-equiv` 会被怎么处理，以及为什么"内容"字段在 `script` 与 `style` 里变成了 `textContent`。

## 本文依据

- `https://nuxt.com/docs/4.x/api/composables/use-head`
  - 原文（Parameters 表十字段与逐字定义）：见正文表格（`title` — "Sets the page title."；`titleTemplate` — "Configures a dynamic template to customize the page title. Can be a string with `%s` placeholder or a function."；`base` — "Sets the `<base>` tag for the document."；`link`/`meta`/`style`/`script`/`noscript` — "Array of xxx objects. Each element is mapped to a `<xxx>` tag, where object properties correspond to HTML attributes."；`htmlAttrs` — "Sets attributes of the `<html>` tag. Each object property is mapped to the corresponding attribute."；`bodyAttrs` — "Sets attributes of the `<body>` tag. Each object property is mapped to the corresponding attribute."）
  - 原文（Type 段的 MetaObject 十项）：`interface MetaObject { title?: string; titleTemplate?: string \| ((title?: string) => string); base?: Base; link?: Link[]; meta?: Meta[]; style?: Style[]; script?: Script[]; noscript?: Noscript[]; htmlAttrs?: HtmlAttributes; bodyAttrs?: BodyAttributes }`
  - 原文（五个示例）：Basic Meta Tags（`app/pages/about.vue`，`title: 'About Us'` + `og:title`/`og:description`）；Adding External Scripts and Styles（`app/pages/external.vue`，`link: [{ rel: 'stylesheet', href: 'https://cdn.example.com/styles.css' }]` + `script: [{ src: 'https://cdn.example.com/script.js', async: true }]`）；Body and HTML Attributes（`app/pages/themed.vue`，`htmlAttrs: { lang: 'en', class: computed(() => isDark.value ? 'dark' : 'light') }` + `bodyAttrs: { class: 'themed-page' }`）
- `https://nuxt.com/docs/4.x/getting-started/seo-meta`
  - 原文（标题模板三段）："You can use the `titleTemplate` option to provide a dynamic template for customizing the title of your site. For example, you could add the name of your site to the title of every page."；"The `titleTemplate` can either be a string, where `%s` is replaced with the title, or a function."；"If you want to use a function (for full control), then this cannot be set in your `nuxt.config`. It is recommended instead to set it within your `app.vue` file where it will apply to all pages on your site:" + `titleTemplate: (titleChunk) => { return titleChunk ? \`${titleChunk} - Site Title\` : 'Site Title' }`；"Now, if you set the title to `My Page` with `useHead` on another page of your site, the title would appear as 'My Page - Site Title' in the browser tab. You could also pass `null` to default to 'Site Title'."；Dynamic Title 一节两例（`'%s - Site Title'` 与 `(productCategory) => ...`）与"`nuxt.config` is also used as an alternative way of setting the page title. However, `nuxt.config` does not allow the page title to be dynamic..."
  - 原文（Template Parameters）："You can use `templateParams` to provide additional placeholders in your `titleTemplate` besides the default `%s`. This allows for more dynamic title generation." + 示例 ``titleTemplate: (titleChunk) => titleChunk ? `${titleChunk} %separator %siteName` : '%siteName'`` 与 `templateParams: { siteName: 'Site Title', separator: '-' }`
  - 原文（Body Tags）："You can use the `tagPosition: 'bodyClose'` option on applicable tags to append them to the end of the `<body>` tag." + `script: [{ src: 'https://third-party-script.com', // valid options are: 'head' | 'bodyClose' | 'bodyOpen' \n tagPosition: 'bodyClose' }]`
  - 原文（External CSS 与 Types）：Google Fonts 两例（`useHead` 的 `link` 数组与 `<Link>` 组件版，含 `rel: 'preconnect'`、`crossorigin: ''`）；`interface MetaObject { ... templateParams?: Record<string, string | Record<string, string>> ... }`
