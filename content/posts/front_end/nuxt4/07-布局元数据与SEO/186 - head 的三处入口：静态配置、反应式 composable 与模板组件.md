---
url: /posts/51529ab0d0e03903f1f803f040ea0828/
title: 186 - head 的三处入口：静态配置、反应式 composable 与模板组件
date: 2026-12-13T09:00:00+08:00
updated: 2026-12-13T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：head 这件事在同构应用里变复杂，是因为**同一份标签要在服务端渲染与水合两侧都被管理**，而 Nuxt 给了三处入口、各自能做的事不一样。官方开篇是："Nuxt head tag management is powered by Unhead. **It provides sensible defaults, several powerful composables and numerous configuration options to manage your app's head and SEO meta tags.**"**入口一是 `nuxt.config` 的 `app.head**`"：Providing an `app.head` property in your `nuxt.config.ts` allows you to **statically** customize the head for your entire app."，紧跟一条限制："**This method does not allow you to provide reactive data. We recommend using `useHead()` in `app.vue`.**"，并给了该放什么："**It's good practice to set tags here that won't change such as your site title default, language and favicon.**"（示例即 `title: 'Nuxt'` 注释 "default fallback title"、`htmlAttrs.lang: 'en'`、`link: [{ rel: 'icon', ... }]`）。**入口二是 `useHead`**："The `useHead` composable function supports reactive input, allowing you to manage your head tags programmatically."。**入口三是模板组件**："While **using `useHead` is recommended in all cases**, you may have a personal preference for defining your head tags in your template using components. Nuxt provides the following components for this purpose: **`<Title>`, `<Base>`, `<NoScript>`, `<Style>`, `<Meta>`, `<Link>`, `<Body>`, `<Html>` and `<Head>`. Note the capitalization of these components ensuring we don't use invalid native HTML tags.**"，并连三条："`<Head>` and `<Body>` can accept nested meta tags (for aesthetic reasons) **but this does not affect where the nested meta tags are rendered in the final HTML**"、"It's suggested to wrap your components in either a `<Head>` or `<Html>` component **as tags will be deduped more intuitively**"、warning "**If you need to duplicate tags across client-server boundaries, apply a `key` attribute on the `<Head>` component.**"**默认标签只有两个**：`viewport`: `width=device-width, initial-scale=1` 与 `charset`: `utf-8`，可用键式快捷项 `charset`/`viewport` 覆写。还有一条关于 CDN 的坑：静态 `app.head` 里的 `href: '/favicon.ico'`"is a literal path and is **not** resolved against `cdnURL`"。

categories:
  - 前端开发

tags:
  - nuxt4
  - SEO
  - head
  - app-head
  - unhead

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


`<title>` 该写在哪，我换过三次位置：先写在 `nuxt.config`，因为它是全站默认；再挪到 `app.vue`，因为要拼站点名；最后有动态字段时只能进页面。这三次挪动正好对应官方给的三个入口——它们不是风格选择，能力差别写在文档里。

## 底座：Unhead，三处入口

官方第一段就定了范围：

> Nuxt head tag management is powered by Unhead. It provides sensible defaults, several powerful composables and numerous configuration options to manage your app's head and SEO meta tags.

"同构里更难"具体难在哪：文档里那句关于**跨客户端-服务端边界重复标签**的警告（下面第三节）就是证据——同一个标签在服务端渲染一次、客户端水合一次，去重与位置都由管理器决定，不是由你写在模板哪一层决定。

| 入口 | 官方定性 | 能反应式吗 |
| --- | --- | --- |
| `nuxt.config` 的 `app.head` | "allows you to **statically** customize the head for your entire app" | ❌ 明确不能 |
| `useHead()` | "supports **reactive input**, allowing you to manage your head tags programmatically" | ✅ |
| 模板组件（`<Title>` 等） | 个人偏好路线，官方先声明"using `useHead` is recommended in all cases" | ✅（`:content="title"` 这种绑定） |

## 入口一：`app.head`，放不变的东西

官方给的规则句与例子是配着的：

> It's good practice to set tags here that won't change such as your site title default, language and favicon.

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  app: {
    head: {
      title: 'Nuxt', // default fallback title
      htmlAttrs: {
        lang: 'en',
      },
      link: [
        { rel: 'icon', type: 'image/x-icon', href: '/favicon.ico' },
      ],
    },
  },
})
```

限制句也在这里，别跳过：

> This method does not allow you to provide reactive data. We recommend using `useHead()` in `app.vue`.

注释里那句 "default fallback title" 与 187 的 `titleTemplate` 是一对：**这里的 `title` 是兜底值**，不是最终标题。

### 一条会静默失效的 CDN 细节

官方专门用一条 note 讲了 favicon 与 `cdnURL` 的关系，我认为这是这一节最容易踩的：

> When you set `app.cdnURL`, assets in your `public/` directory (including `favicon.ico`) are served from that CDN. Nuxt resolves public assets against `cdnURL`, falling back to `app.baseURL`. However, **a static `app.head` link such as `href: '/favicon.ico'` above is a literal path and is not resolved against `cdnURL`**. To point the favicon at the resolved location, build the `href` from runtime config with `useHead()` in `app.vue`:

```vue [app/app.vue]
<script setup lang="ts">
const { cdnURL, baseURL } = useRuntimeConfig().app
useHead({
  link: [
    { rel: 'icon', type: 'image/x-icon', href: `${cdnURL || baseURL}favicon.ico` },
  ],
})
</script>
```

**三个词连读**：配置里的字符串是**字面量**（不解析）；要解析就得走运行时配置；而运行时配置只能在 `useHead` 里读。**这正是"入口一的能力上限"最具体的样例**。

### Nuxt 默认给的两个标签

> Some tags are provided by Nuxt by default to ensure your website works well out of the box.
>
> - `viewport`: `width=device-width, initial-scale=1`
> - `charset`: `utf-8`

> While most sites won't need to override these defaults, you can update them using the keyed shortcuts.

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  app: {
    head: {
      // update Nuxt defaults
      charset: 'utf-16',
      viewport: 'width=device-width, initial-scale=1, maximum-scale=1',
    },
  },
})
```

**注意有专门的键 `charset` / `viewport`**，不用手搓 `meta` 数组元素；这也说明默认值是"键式"管理的。

## 入口二：`useHead`，官方推荐位

官方在 SEO 页给的形态（与 composable 页同一例）：

```vue [app/app.vue]
<script setup lang="ts">
useHead({
  title: 'My App',
  meta: [
    { name: 'description', content: 'My amazing site.' },
  ],
  bodyAttrs: {
    class: 'test',
  },
  script: [{ innerHTML: 'console.log(\'Hello world\')' }],
})
</script>
```

官方在这一节直接指了两条后续："We recommend taking a look at the `useHead` and **`useHeadSafe`** composables."——把"用户输入"这件事单独拎出来，说明它不是可选优化（189 展开）。字段全表在 188。

## 入口三：九个组件，以及"位置是假的"

组件清单官方特意带了大小写说明：

> While using `useHead` is recommended in all cases, you may have a personal preference for defining your head tags in your template using components.
>
> Nuxt provides the following components for this purpose: `<Title>`, `<Base>`, `<NoScript>`, `<Style>`, `<Meta>`, `<Link>`, `<Body>`, `<Html>` and `<Head>`. **Note the capitalization of these components ensuring we don't use invalid native HTML tags.**

**"capitalization"这句是有实用价值的**：写成小写 `<title>`/`<meta>` 不会成为 head 组件，而是被当成非法/原生标签处理。

然后是那条与 184 对照着记最好懂的说明：

> `<Head>` and `<Body>` can accept nested meta tags (for aesthetic reasons) **but this does not affect where the nested meta tags are rendered in the final HTML**.

也就是说：嵌套只是**书写层级**，不决定最终落在 `<head>` 还是 `<body>`——决定位置的是字段本身（`tagPosition`，见 188；以及 186 这里没有、组件版也没有对应的位置参数）。

官方还建议了包裹方式，理由是去重：

> It's suggested to wrap your components in either a `<Head>` or `<Html>` component as **tags will be deduped more intuitively**.

最后是同构专属的那条警告，我认为这是"SEO 在同构里更难"的最直接证据：

> If you need to duplicate tags **across client-server boundaries**, apply a `key` attribute on the `<Head>` component.

**为什么需要"重复"？** 因为一份模板会在服务端渲染一次、水合一次；默认去重会把两侧视为同一个标签。官方没有在这一句展开更多，所以我的处理是：**除非确实需要两份并存的标签，否则不要主动加 `key`**——加了就是明确告诉管理器"这是两个标签"。

官方给的组件写法示例（反应式与 `<Head>` 包裹同时出现）：

```vue [app/app.vue]
<script setup lang="ts">
const title = ref('Hello World')
</script>

<template>
  <div>
    <Head>
      <Title>{{ title }}</Title>
      <Meta
        name="description"
        :content="title"
      />
      <Style>
        body { background-color: green; }
      </Style>
    </Head>

    <h1>{{ title }}</h1>
  </div>
</template>
```

## 我按什么决定写在哪

| 内容 | 写在哪 | 依据 |
| --- | --- | --- |
| 站点默认标题、`lang`、favicon | `app.head` | "good practice to set tags here that won't change" |
| `titleTemplate`（函数形态） | `app.vue` 的 `useHead` | "If you want to use a function (for full control), then this cannot be set in your `nuxt.config`"（187） |
| 需要读运行时配置拼 URL | `useHead` | favicon 那条 note |
| 每页的 `description`/OG | 页面里的 `useSeoMeta`（190） | 类型安全 + XSS 安全 |
| 来自用户或 CMS 的值 | `useHeadSafe`（189） | "the recommended way to provide head data with user input" |
| 团队习惯模板写法 | 九个组件，**包在 `<Head>` 或 `<Html>` 里** | 去重更直观 |

## 自查

1. "同构里 head 更难"这句话在官方文档里有哪两处直接证据？
2. 三处入口各自的定性词是什么？哪一处被明确写了"不能提供反应式数据"？
3. favicon 与 `cdnURL` 那条 note 的三个关键词是什么？为什么最后要靠 `useHead` 解决？
4. Nuxt 默认给哪两个标签？怎么覆写它们？
5. 九个 head 组件为什么要大写？`<Head>` 嵌套为什么不决定位置？
6. 为什么官方建议包在 `<Head>` 或 `<Html>` 里？`key` 那条警告在解决什么问题？
7. 把你项目里的 head 内容按那张表归一次位，说说每一条放哪、为什么。

## 下一篇

187 讲 `useHead` 本体：它的入参可以静态也可以反应式（`ref`/`computed`/`reactive`/函数），十个字段各自的映射规则，以及那一页里"返回值"与类型签名互相矛盾的地方该怎么处理。
