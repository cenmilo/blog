---
url: /posts/b234bffc61ecdf9b56bf75938a8dc80b/
title: 622 - Bridge 下的 meta 与运行时配置：head 搬到 app.head 与 runtimeConfig 要配好 nitro
date: 2027-05-07T10:00:00+08:00
updated: 2027-05-07T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：Bridge 剩下的两节各有一处前置条件与一处改名。meta 那一节要**先开 `bridge.meta: true`**，再把配置里的 **`head` 改名成 `app.head`**（**对象不再有用于去重的 `hid` 键**）；`useHead` 在 Bridge 下**底层走 `@unhead/vue` 而不是 `vue-meta`**，官方同时警告**别把 Nuxt 2 原生 `head()` 与 `useHead` 混用，两者可能冲突**；Options API 的 `head()` 必须包在 `defineNuxtComponent` 里，**它收到 nuxt app 但访问不到组件实例**（读 `this`／`this.$data` 的代码要改走 `useHead`）；**函数形式的 `titleTemplate` 不能设在 `nuxt.config`，官方建议放进 `/layouts`**。runtime config 那一节只有一条 `::warning`：**用 `runtimeConfig` 之前 `nitro` 必须已经配好**；改动是把 `publicRuntimeConfig` 与 `privateRuntimeConfig` **合成一个 `runtimeConfig`，公开那份放进 `public` 键**，模板里随之从 `$config.websiteURL` 改成 **`$config.public.websiteURL`**。

categories:
  - 前端开发

tags:
  - nuxt4
  - bridge
  - useHead
  - runtimeConfig
  - 升级迁移
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

Bridge 项目的最后两块是元信息和运行时配置。两块的共同点是：**都得先确认前置开关**，否则改了没反应——meta 要开 `bridge.meta`，runtime config 要先把 `nitro` 配好。

## meta：先开桥，再改名

页面开头两句是判据：**要用 `head` 读组件状态，就迁到 `useHead`**；**要用 Options API，就有一个 `head()` 方法可用，前提是配合 `defineNuxtComponent`**。

`### Set bridge.meta` 与前面几节同构：

```js [nuxt.config.ts]
import { defineNuxtConfig } from '@nuxt/bridge'

export default defineNuxtConfig({
  bridge: {
    meta: true,
    nitro: false, // If migration to Nitro is complete, set to true
  },
})
```

`### Update head properties` 是真正的改名动作：

> In your `nuxt.config`, rename `head` to `app.head`. (Note that objects no longer have a `hid` key for deduplication.)

前后对照（三条 meta 里，第三条去掉了 `hid`）：

```ts [Nuxt 2]
export default {
  head: {
    titleTemplate: '%s - Nuxt',
    meta: [
      { charset: 'utf-8' },
      { name: 'viewport', content: 'width=device-width, initial-scale=1' },
      { hid: 'description', name: 'description', content: 'Meta description' },
    ],
  },
}
```

```ts [Nuxt 3]
export default defineNuxtConfig({
  app: {
    head: {
      titleTemplate: '%s - Nuxt',
      meta: [
        { charset: 'utf-8' },
        { name: 'viewport', content: 'width=device-width, initial-scale=1' },
        { name: 'description', content: 'Meta description' },
      ],
    },
  },
})
```

**这里有一处跨页措辞差异要提醒**：迁移文档那侧（`migration/meta`）第一步写的是 "rename `head` to `meta`"，**Bridge 这一节写的是 rename `head` to `app.head`**。两节指向的不是同一个键名，而配置参考里承载这些默认值的小节是 **`## app` 之下的 `### head`**（"Set default configuration for `<head>` on every page."）——**按 `app.head` 写才是与 4.x 配置参考一致的做法**。

## useHead：底层换了，别与旧写法混用

> Nuxt Bridge provides a new Nuxt 3 meta API that can be accessed with a new `useHead` composable.

```vue [app/pages/index.vue]
<script setup lang="ts">
useHead({
  title: 'My Nuxt App',
})
</script>
```

一条 `::tip` 说明实现：**"This `useHead` composable uses `@unhead/vue` under the hood (rather than `vue-meta`) to manipulate your `<head>`."**

紧跟的 `::warning` 是这节最容易踩的：**"We recommend not using the native Nuxt 2 `head()` properties in addition to `useHead`, as they may conflict."** ——**两套同时写会互相冲突**，不是"后者覆盖前者"这么简单，所以迁移期要么走 `useHead`，要么保留旧的，别叠着来。

## Options API：那个 `this` 拿不到了

```vue [app/pages/index.vue]
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

配套 `::warning` 把后果说成了"破坏性变更"：**`head` 收到的是 nuxt app、访问不到组件实例；如果你的 `head` 里通过 `this` 或 `this.$data` 读 data，就得改走 `useHead`。**

这一条和 4.x 迁移文档那一节的 Options API 段是同一件事（同一份代码、同一句注释），**所以 Bridge 阶段就会撞上，不用等到真上 3**。

## titleTemplate 的函数形式在布局里

> If you want to use a function (for full control), then this cannot be set in your nuxt.config, and it is recommended instead to set it within your `/layouts` directory.

```vue [app/layouts/default.vue]
<script setup lang="ts">
useHead({
  titleTemplate: (titleChunk) => {
    return titleChunk ? `${titleChunk} - Site Title` : 'Site Title'
  },
})
</script>
```

理由是配置对象要能序列化，函数进不去——**这一节的措辞是"不能在 `nuxt.config` 里设、建议放布局"**，替代方案就是上面这种 `titleChunk` 三元式（有值拼接、无值给默认标题）。

## runtime config：前置条件是 nitro 已配好

这一节第一条就是 `::warning`：

> When using `runtimeConfig` option, **nitro** must have been configured.

**顺序要求很硬：Bridge 的 `nitro` 这一层没配好，`runtimeConfig` 就用不了。** 与 meta 那节同款的 `nitro: false // If migration to Nitro is complete, set to true` 注释，在这里不是可选装饰，而是前置状态检查。

改名动作是**两个旧键合成一个**：

> First, you'll need to combine your `publicRuntimeConfig` and `privateRuntimeConfig` properties into a new one called `runtimeConfig`, **with the public config within a key called `public`**.

```diff
// nuxt.config.js
- privateRuntimeConfig: {
-   apiKey: process.env.NUXT_API_KEY || 'super-secret-key'
- },
- publicRuntimeConfig: {
-   websiteURL: 'https://public-data.com'
- }
+ runtimeConfig: {
+   apiKey: process.env.NUXT_API_KEY || 'super-secret-key',
+   public: {
+     websiteURL: 'https://public-data.com'
+   }
+ }
```

模板侧跟着改一层：

```diff
// MyWidget.vue
- <div>Website: {{ $config.websiteURL }}</div>
+ <div>Website: {{ $config.public.websiteURL }}</div>
```

三点注意：私密那半直接落在顶层（示例里 `apiKey`），公开那半进 `public`；**读公开值要多写一层 `.public`**，这是漏改最常出现的地方；而示例里 `process.env.NUXT_API_KEY || 'super-secret-key'` 这种**把默认值写死在配置里**的写法，到了 4.x 一側官方给的是"默认值给空串、由运行时环境变量覆盖"的路子（`## runtimeConfig` 那一节）。

## 自查

- meta：先 `bridge.meta: true`；**`head` → `app.head`**；`hid` 去重键没了。
- **两节措辞不同**（`migration/meta` 说改名为 `meta`，Bridge 说 `app.head`），按配置参考的 `app.head` 为准。
- `useHead` 底层是 `@unhead/vue`（不是 `vue-meta`）；**不要与原生 `head()` 混用**。
- Options API 的 `head()` 必须包在 `defineNuxtComponent` 里，**拿不到组件实例**，读 `this`／`this.$data` 的要改。
- 函数版 `titleTemplate` 不能进 `nuxt.config`，放布局。
- runtime config：**必须先配好 `nitro`**；两个旧键合成 `runtimeConfig`，公开项进 `public`，**模板里改 `$config.public.xxx`**。

## 下一篇

600-629 与 Bridge 的这一段还没走完。下一篇 623 读 Bridge 的 Nitro 与 Vite 两节：**它们各自对应前面反复出现的那两个开关——`bridge.nitro` 与 `bridge.vite`**，而每节示例里都跟着那句"迁移完成就把它改成 true"的注释。

## 本文依据

- https://nuxt.com/docs/4.x/bridge/meta（开头两句判据；`### Set bridge.meta` 代码块与 `nitro: false` 注释；`### Update head properties` 的改名句与 `hid` 括注、Nuxt 2／Nuxt 3 两段对照代码；`## useHead Composables` 定性句、示例、`::tip`（`@unhead/vue` 而非 `vue-meta`）、`::warning`（不要与原生 `head()` 并用，可能冲突）与指向 seo-meta 的链接；`## Options API` 代码块（含两行原注释）与其 `::warning`（`this`／`this.$data` 要改走 `useHead`）；`## Title Template` 的"不能设在 nuxt.config、建议放 /layouts"句与 `app/layouts/default.vue` 示例）
- https://nuxt.com/docs/4.x/bridge/runtime-config（`::warning`：使用 `runtimeConfig` 时 nitro 必须已配置、`## Update Runtime Config` 的合并说明句与两段 diff（配置侧 `privateRuntimeConfig`＋`publicRuntimeConfig`→`runtimeConfig` ＋ `public`；模板侧 `$config.websiteURL`→`$config.public.websiteURL`））
- https://nuxt.com/docs/4.x/migration/meta（第一步 "rename `head` to `meta`" 的原句，用于对照两节措辞差异）
- https://nuxt.com/docs/4.x/api/nuxt-config（`### app.head` 的 "Set default configuration for `<head>` on every page." 句，用于判定改名后的正确落点；`## runtimeConfig` 的示例与"默认值给空串、运行时由环境变量覆盖"的写法）
