---
url: /posts/f94392560f34e0a6d2bc704a595947f4/
title: 自动导入的完整范围：哪些东西不用写 import
date: 2026-12-01T09:00:00+08:00
updated: 2026-12-01T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇先把"不用写 import 的东西到底从哪来"划清。官方定位句："**Nuxt auto-imports components, composables, helper functions and Vue APIs.**"，正文第一句补了范围："Nuxt auto-imports components, composables and **Vue.js APIs** to use across your application without explicitly importing them."。它和普通全局声明的区别官方写成三个承诺："**Contrary to a classic global declaration, Nuxt preserves typings, IDEs completions and hints, and only includes what is used in your production code.**"。目录侧的来源官方列了三条："**Thanks to its opinionated directory structure, Nuxt can auto-import your `app/components/`, `app/composables/` and `app/utils/`.**"，以及"Directory-based Auto-imports"那节的三条清单（`app/components/` for Vue components、`app/composables/` for Vue composables、`app/utils/` for helper functions and other utilities）。服务端另有一条独立的 note："**In the `server` directory, Nuxt auto-imports exported functions and variables from `server/utils/`**"。第三方包与自定义目录靠 `imports` 一节："You can also auto-import functions exported from custom folders or third-party packages by configuring the `imports` section of your `nuxt.config` file."，官方示例是 `presets: [{ from: 'vue-i18n', imports: ['useI18n'] }]`。组件这一类官方特意说明它**是另一套配置**："Nuxt also automatically imports components from your `~/components` directory, **although this is configured separately from auto-importing composables and utility functions.**"。还有一条与直觉相反的 warning："**Auto-imported `ref` and `computed` won't be unwrapped in a component `<template>`.**"，理由是"This is due to how Vue works with refs that aren't top-level to the template."。想反过来显式写出来，官方给的口子是 `#imports`："Nuxt exposes every auto-import with the `#imports` alias that can be used to make the import explicit if needed"。

categories:
  - 前端开发

tags:
  - nuxt4
  - 自动导入
  - composables
  - 目录结构
  - imports

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


前面四十多篇里我大量直接写 `ref`、`useFetch`、`navigateTo`，一行 import 都没写。这一篇开始讲这件事本身：**自动导入覆盖哪些来源、边界在哪、怎么关掉、以及怎么读框架给的文档**。先把"范围"这张图画清楚。

## 三个承诺：它不是"挂到 window 上"

官方定性：

> Nuxt auto-imports components, composables, helper functions and Vue APIs.

> Nuxt auto-imports components, composables and Vue.js APIs to use across your application **without explicitly importing them**.

紧接着这句是它与传统全局声明的分界，官方给了三样东西：

> Contrary to a classic global declaration, Nuxt **preserves typings**, **IDEs completions and hints**, and **only includes what is used in your production code**.

| 承诺 | 含义 |
| --- | --- |
| 保留类型 | 自动导入不等于 `any`；编辑器知道 `ref` 从哪来 |
| 保留补全与提示 | 跳到定义、悬停文档这些常规操作照常可用 |
| **只打包用到的** | 这一条是和"全局声明"最实的区别：没用到的不会进产物 |

最后一条能解释一件我早期想不通的事：自动导入看起来像"把所有 API 都塞进全局"，但按官方说法它是**编译期按名字补 import**——所以它既省掉书写，又不带来全局污染的代价。

## 一条读文档的约定

`::note` 里那句其实是整个 API 章节的读法说明：

> In the docs, **every function that is not explicitly imported is auto-imported by Nuxt** and can be used as-is in your code. You can find a reference for auto-imported components, composables and utilities in the API section.

所以前面那些官方示例里没写 import 的 `useLoadingIndicator`、`defineNuxtLink`、`preloadRouteComponents`——它们**都在自动导入名单里**。这一句也给了查证入口：API 那一大节就是清单所在地。

## 五个来源

| 来源 | 官方原话 | 典型内容 |
| --- | --- | --- |
| ① 框架内置 composables | "Nuxt auto-imports functions and composables to perform data fetching, get access to the app context and runtime config, manage state or define components and plugins." | `useFetch` / `useAsyncData`、`useNuxtApp`、`useRuntimeConfig`、`useState`、`defineNuxtPlugin`、`defineNuxtRouteMiddleware` |
| ② Vue 侧 API | "Vue exposes Reactivity APIs like `ref` or `computed`, **as well as lifecycle hooks and helpers** that are auto-imported by Nuxt." | `ref`、`computed`、`watch`，以及 `onMounted` 这类生命周期钩子与工具 |
| ③ 你的三个目录 | "Thanks to its opinionated directory structure, Nuxt can auto-import your `app/components/`, `app/composables/` and `app/utils/`." | 自己写的组件、组合式函数、工具函数 |
| ④ 服务端 utils | "In the `server` directory, Nuxt auto-imports **exported functions and variables** from `server/utils/`." | 服务端专用的导出函数/变量 |
| ⑤ 自定义目录与第三方包 | "You can also auto-import functions exported from custom folders or third-party packages by configuring the `imports` section of your `nuxt.config` file." | `imports.dirs`、`imports.presets` |

官方给的最小示例就是 ①②：

```vue [app/app.vue]
<script setup lang="ts">
const count = ref(1) // ref is auto-imported
</script>
```

```vue
<script setup lang="ts">
/* useFetch() is auto-imported */
const { data, refresh, status } = await useFetch('/api/hello')
</script>
```

```vue
<script setup lang="ts">
/* ref() and computed() are auto-imported */
const count = ref(1)
const double = computed(() => count.value * 2)
</script>
```

第三个例子有个细节：`double` 那行写的是 `count.value`，**在 `<script setup>` 里 `.value` 是必需的**，这与下面那条模板 warning 是两件不同的事，别混。

## ③ 目录这一类的原文清单

> Nuxt directly auto-imports files created in defined directories:
>
> - `app/components/` for Vue components.
> - `app/composables/` for Vue composables.
> - `app/utils/` for helper functions and other utilities.

三条各自指向目录文档页。这一篇只确认**范围**；两个更细的问题留给后面：

- 组件那条线的规则（默认只扫哪里、同名怎么办、前缀怎么加）比这三行字多得多，我从 156 起专门讲。
- `composables/` 与 `utils/` 的命名约定（`useXxx` 前缀、文件名怎么变成导入名）在 063 那条线已经过。

**一个命名混用值得注意**：这一页同时出现 `app/components/`（4.x 的目录写法）与 `~/components`（旧别名写法）——组件那一节原话是 "Nuxt also automatically imports components from your `~/components` directory"。两处指同一个东西，`~` 是 srcDir 的别名（051 讲过那张别名表）。

## ④ 服务端那条线的范围更窄

原文只有一句，但限定得很清楚：

> In the `server` directory, Nuxt auto-imports exported functions and variables from `server/utils/`.

拆出来是三层限定：

| 限定 | 含义 |
| --- | --- |
| "**In the `server` directory**" | 这条自动导入只在服务端文件里成立 |
| "**from `server/utils/`**" | 服务端只扫这一个目录 |
| "**exported functions and variables**" | 得是有导出的成员 |

**服务端没有 `ref` 那一套 Vue API**——这条边界与两侧名字冲突的坑，151 展开。

## ⑤ 第三方包：一行 presets

官方示例给得很短，但把形状说全了：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  imports: {
    presets: [
      {
        from: 'vue-i18n',
        imports: ['useI18n'],
      },
    ],
  },
})
```

前面挂了一条 tip，能省掉不少重复配置：

> If you are using the Nuxt module for that package, it is likely that the module has already configured auto-imports for that package.

**所以我的顺序是：先装模块，再看要不要手写 `presets`**。同一个包既装了模块又自己配 `presets`，名字冲突的风险是我自己制造的。

`presets` 里还能写 `as` 重命名、`types` 之类更多字段，`imports` 那一节的完整配置我放到 153 讲"往里加东西"时一起摊开。

## 组件这一类是另一套开关

> Nuxt also automatically imports components from your `~/components` directory, **although this is configured separately from auto-importing composables and utility functions.**

这一句对排障很关键：**`imports.autoImport: false` 关不掉组件自动导入**，组件侧的开关是 `components.dirs`。官方给的关闭写法：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  components: {
    dirs: [],
  },
})
```

并注明："though **note that this will not affect components added by modules**"。三条关闭路径的差别在 152 一次讲完。

## 一条与直觉相反的 warning

> **Auto-imported `ref` and `computed` won't be unwrapped in a component `<template>`.**
>
> This is due to how Vue works with refs that aren't top-level to the template. You can read more about it in the Vue documentation.

官方的结论与原因就这两句，指向的是 Vue 文档里"模板中解包的一个 caveat"。**它没给替代写法**——我遇到"模板里显示成对象/函数而不是值"这类症状时，先按这条 warning 怀疑：这个 ref 是不是自动导入来的、而不是 `<script setup>` 顶层绑定的。真要确认，看渲染出来的文本和 `typeof` 一次就够。

## 显式写出来：`#imports`

> Nuxt exposes every auto-import with the `#imports` alias that can be used to make the import explicit if needed:

```vue
<script setup lang="ts">
import { computed, ref } from '#imports'

const count = ref(1)
const double = computed(() => count.value * 2)
</script>
```

"every auto-import" 这个范围词值得留意：**能被自动导入的，都能从 `#imports` 显式导**。哪些场合我倾向于显式写（例如纯 `.ts` 文件被别的工具链读、或想让依赖关系可见），154 专门讲。

## 一张范围图（本篇版）

| 你在写 | 能直接用 |
| --- | --- |
| `app/` 下的 `.vue`（页面、组件、布局） | ①②③ + 组件 |
| `app/composables/`、`app/utils/` 里的 `.ts` | ①②（自定义文件本身被别人自动导入） |
| `server/` 下的 `.ts` | ④，**不含** Vue 那一套 |
| 别处的文件（比如 `packages/` 里） | 默认不在范围内，需要 ⑤ 或模块提供 |

最后一行的"别处"是猜边界最容易撞墙的地方，151 用官方目录规则把它钉死。

## 自查

1. 官方说自动导入与普通全局声明的三个差别分别是什么？哪一个直接关系到产物体积？
2. "文档里没写 import 的函数"该怎么理解？清单在文档的哪一大节？
3. 五个来源分别是什么？服务端那条的三层限定怎么说？
4. `app/components/`、`app/composables/`、`app/utils/` 三行清单各自对应什么？
5. 组件的自动导入与 composables 的自动导入在配置上是同一个开关吗？官方哪句话说明这点？
6. `components.dirs: []` 有什么它管不到的东西？
7. 自动导入的 `ref` 在 `<script setup>` 里要不要 `.value`？在 `<template>` 里官方给了哪条 warning 和哪条原因？
8. 想让某个第三方 composable 也能不写 import，官方给的字段是什么？配置前应该先看什么？
9. `#imports` 覆盖的范围有多大？

## 下一篇

151 划边界：目录到底扫到哪一层、composable 必须在什么上下文里同步调用、`Nuxt instance is unavailable` 是怎么来的、以及服务端与客户端两套自动导入名字撞车时怎么办。

## 本文依据

- `https://nuxt.com/docs/4.x/guide/concepts/auto-imports`（本篇主要依据）
  - 原文（标题句与开篇）："Nuxt auto-imports components, composables, helper functions and Vue APIs."；"Nuxt auto-imports components, composables and Vue.js APIs to use across your application without explicitly importing them."；示例 `const count = ref(1) // ref is auto-imported`（文件标注 `app/app.vue`）
  - 原文（三个承诺）："Contrary to a classic global declaration, Nuxt preserves typings, IDEs completions and hints, and only includes what is used in your production code."
  - 原文（目录来源）："Thanks to its opinionated directory structure, Nuxt can auto-import your `app/components/`, `app/composables/` and `app/utils/`."
  - 原文（读文档约定 note）："In the docs, every function that is not explicitly imported is auto-imported by Nuxt and can be used as-is in your code. You can find a reference for auto-imported components, composables and utilities in the API section."
  - 原文（服务端 note）："In the `server` directory, Nuxt auto-imports exported functions and variables from `server/utils/`."
  - 原文（自定义/第三方 note）："You can also auto-import functions exported from custom folders or third-party packages by configuring the `imports` section of your `nuxt.config` file."
  - 原文（Built-in Auto-imports）："Nuxt auto-imports functions and composables to perform data fetching, get access to the app context and runtime config, manage state or define components and plugins."；示例 `const { data, refresh, status } = await useFetch('/api/hello')` 与注释 `/* useFetch() is auto-imported */`；"Vue exposes Reactivity APIs like `ref` or `computed`, as well as lifecycle hooks and helpers that are auto-imported by Nuxt."；示例 `const count = ref(1)` + `const double = computed(() => count.value * 2)` 与注释 `/* ref() and computed() are auto-imported */`
  - 原文（Directory-based Auto-imports）："Nuxt directly auto-imports files created in defined directories: `app/components/` for Vue components. `app/composables/` for Vue composables. `app/utils/` for helper functions and other utilities."
  - 原文（warning）："Auto-imported `ref` and `computed` won't be unwrapped in a component `<template>`."；"This is due to how Vue works with refs that aren't top-level to the template. You can read more about it in the Vue documentation."
  - 原文（Explicit Imports）："Nuxt exposes every auto-import with the `#imports` alias that can be used to make the import explicit if needed:" + `import { computed, ref } from '#imports'`
  - 原文（Auto-imported Components）："Nuxt also automatically imports components from your `~/components` directory, although this is configured separately from auto-importing composables and utility functions."；"To disable auto-importing components from your own `~/components` directory, you can set `components.dirs` to an empty array (though note that this will not affect components added by modules)." + `components: { dirs: [] }`
  - 原文（Auto-Import from Third-Party Packages）："Nuxt also allows auto-importing from third-party packages."；tip "If you are using the Nuxt module for that package, it is likely that the module has already configured auto-imports for that package."；示例 `imports: { presets: [{ from: 'vue-i18n', imports: ['useI18n'] }] }`
  - 原文（页尾）：`::link-example{to="https://nuxt.com/docs/examples/features/auto-imports"}`
- `https://nuxt.com/docs/4.x/directory-structure/app/components`、`https://nuxt.com/docs/4.x/directory-structure/app/composables`、`https://nuxt.com/docs/4.x/directory-structure/app/utils`、`https://nuxt.com/docs/4.x/directory-structure/server`
  - 对照用（见 063、151、156）：四个目录页，本页三条目录来源与 server note 都以它们为链接目标
- `https://nuxt.com/docs/4.x/api/nuxt-config`
  - 对照用（见 152、153）：`imports` 一节（本页 note 与 `presets` 示例都链接到 `api/nuxt-config#imports`）
