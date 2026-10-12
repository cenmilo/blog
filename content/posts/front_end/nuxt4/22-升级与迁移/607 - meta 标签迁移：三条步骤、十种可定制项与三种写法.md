---
url: /posts/eed7df0f26a1719d48dad56cc803bb49/
title: meta 标签迁移：三条步骤、十种可定制项与三种写法
date: 2027-05-02T10:00:00+08:00
updated: 2027-05-02T10:00:00+08:00
author: cmdragon

summary:
  摘要：`migration/meta` 给了三种管理元信息的位置——**通过 `nuxt.config`、通过 `useHead` 组合式函数、通过全局元组件**——并列出**十种可定制项**：`title`、`titleTemplate`、`base`、`script`、`noscript`、`style`、`meta`、`link`、`htmlAttrs`、`bodyAttrs`。Migration 是**三步**：把配置里的 `head` 改名成 `meta`（并考虑把这份共享配置移进 `app.vue`，**注意对象不再有用于去重的 `hid` 键**）、需要读组件状态时改用 `useHead`、用 Options API 时 `head()` 方法**必须在 `defineNuxtComponent` 里**（它收到的是 nuxt app，**访问不到组件实例**）。`::important` 有一条硬性写法要求：**这些元组件要用大写字母**，`<Title>` 而不是 `<title>`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 元信息
  - useHead
  - head迁移
  - 升级迁移
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

旧项目里那个 `head: { meta: [{ hid: 'description', … }] }` 搬过来之后不生效了。这一页讲的就是这件事：配置项改名、`hid` 没了、以及现在有三种位置可以写元信息。

## 三种写法与十种可定制项

页首定性句："Manage your meta tags, from Nuxt 2 to Nuxt 3."，接着列出三种途径：

1. Through your `nuxt.config`.
2. Through the `useHead` composable.
3. Through global meta components.

可定制的东西一共**十项**，原文一行给全："You can customize `title`, `titleTemplate`, `base`, `script`, `noscript`, `style`, `meta`, `link`, `htmlAttrs` and `bodyAttrs`."

一条 `::tip` 交代实现归属与风险：**"Nuxt currently uses `Unhead` to manage your meta tags, but implementation details may change."** ——注意这个措辞是 "currently"，与配置参考里 `unhead` 那一节只给两个子项的薄记载是配套的：**细节属于 Unhead，不在 Nuxt 文档里承诺。**

## Migration 三步

**第一步，改名并考虑搬家。** 原文三个动作挤在一句里："In your `nuxt.config`, **rename `head` to `meta`**. Consider moving this shared meta configuration into your `app.vue` instead. (**Note that objects no longer have a `hid` key for deduplication.**)"

这句里的括号是最容易出问题的地方：**旧写法靠 `hid` 做去重，新版对象没有这个键。** 顺带一条 4.x 侧的对应记载：`unhead` 那一节的 `legacy` 开关打开后会加回 `DeprecationsPlugin`，支持的正是 `hid`、`vmid`、`children`、`body` 这四个 props——**也就是说本页说的"没有了"，在 4.x 里有一个明确的恢复开关**，但它默认是 `false`，且 `future.compatibilityVersion: 5` 会让这个开关被忽略。

**第二步，读组件状态就换 `useHead`。** 原文："If you need to access the component state with `head`, you should migrate to using `useHead`. You might also consider using the built-in meta-components."

对照代码最直观——旧写法把 `head` 写成方法、返回 `this.title`：

```vue [Nuxt 2]
<script>
export default {
  data: () => ({
    title: 'My App',
    description: 'My App Description',
  }),
  head () {
    return {
      title: this.title,
      meta: [{
        hid: 'description',
        name: 'description',
        content: this.description,
      }],
    }
  },
}
</script>
```

新写法是 `useHead` 直接吃响应式引用，官方注释写着 **"This will be reactive when you change title/description above"**：

```vue [Nuxt 3]
<script setup lang="ts">
const title = ref('My App')
const description = ref('My App Description')

// This will be reactive when you change title/description above
useHead({
  title,
  meta: [{
    name: 'description',
    content: description,
  }],
})
</script>
```

两处差别要一起看：**`hid` 这一项消失了**、**`content` 从字符串变成可以直接传 ref**（这就是"响应式"的来源）。

**第三步，Options API 的 `head()` 必须在 `defineNuxtComponent` 里。** 原文："If you need to use the Options API, there is a `head()` method you can use **when you use `defineNuxtComponent`**."

```vue [Nuxt 3 (Options API)]
<script>
// if using options API `head` method you must use `defineNuxtComponent`
export default defineNuxtComponent({
  head (nuxtApp) {
    // `head` receives the nuxt app but cannot access the component instance
    return {
      meta: [{
        name: 'description',
        content: 'This is my page description.',
      }],
    }
  },
})
</script>
```

注释那句是这一节的关键限制：**`head` 收到的是 nuxt app，拿不到组件实例**——所以旧代码里 `head()` 内部读 `this.xxx` 的写法在这里必然要改（改成从 `nuxtApp` 拿，或者换 `useHead`）。

## 元组件与那条大写要求

第三种写法是模板里的元组件，示例把 `title` 与 `meta` 都换成了组件形式：

```vue [Nuxt 3]
<template>
  <div>
    <Head>
      <Title>My App</Title>
      <Meta
        name="description"
        content="My app description"
      />
    </Head>
    <!-- -->
  </div>
</template>
```

官方对它的说明是："Nuxt 3 also provides meta components that you can use to accomplish the same task. **While these components look similar to HTML tags, they are provided by Nuxt and have similar functionality.**"

`::important` 有两条：**必须用大写字母写这些组件名，以区别于原生 HTML 元素（`<Title>` 而不是 `<title>`）**；以及**这些组件可以放在页面模板的任何位置**。

第一条是迁移时最容易忽略的：小写的 `<title>` 会被当成原生 HTML 标签，不走 Nuxt 的元信息处理。

## 自查

- 三种位置：`nuxt.config`、`useHead`、元组件；可定制项**十种**。
- `head`→`meta` 改名；**`hid` 去重键已不存在**（要恢复得开 `unhead.legacy`，而它会被 `compatibilityVersion: 5` 忽略）。
- `useHead` 可以直接传 ref，这是"响应式"的来源。
- Options API 的 `head()` **必须包在 `defineNuxtComponent` 里**，且**访问不到组件实例**。
- 元组件必须大写；可放模板任意位置。
- 实现归属是 Unhead，官方措辞 "currently…implementation details may change"。

## 下一篇

下一篇 608 读插件与中间件那一页：四步迁移（插件两步、路由中间件两步），核心是两个形状变化——插件只接一个参数 `nuxtApp`、`inject(...)` 换成 `nuxtApp.provide(...)` 或 `return { provide: {...} }`；中间件从 `({ store, redirect })` 换成 `defineNuxtRouteMiddleware((to, from) => ...)` ＋ `navigateTo()`。另外两种文件名约定也在这页：`.client`／`.server` 后缀与 `.global` 后缀。

## 本文依据

- https://nuxt.com/docs/4.x/migration/meta（页首定性句、三种途径编号、十项可定制内容原文、`::tip` 的 Unhead "implementation details may change" 句、read-more 指向 seo-meta、Migration 三步原文、`### useHead` 与 `### Meta-components` 两组前后对照代码（含 "This will be reactive…" 注释与 `hid` 消失）、`::important` 两条（大写字母／可放任意位置）、`### Options API` 代码块及其两行注释）
- https://nuxt.com/docs/4.x/api/nuxt-config（`### unhead.legacy`：默认 `false`、加 `DeprecationsPlugin` 支持 `hid`/`vmid`/`children`/`body`、加 `PromisesPlugin`）
- https://nuxt.com/docs/4.x/getting-started/upgrade（十九项清单里 "Server-only head composables… `unhead.legacy` is ignored… Capo sorting is always on" 那一条）
