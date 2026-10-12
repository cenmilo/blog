---
url: /posts/87c9bba9e52b6831addfe49afda6daad/
title: 模块里的自动导入陷阱：#imports 显式写法
date: 2027-05-27T11:00:00+08:00
updated: 2027-05-27T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：模块往用户项目里塞一个组合式函数，靠的是自动导入，而自动导入是可以被关掉的。`imports.autoImport: false` 会把组合式函数和工具的自动导入全部关掉，`imports.scan: false` 只关你自己代码的那一半，`components.dirs: []` 关组件目录但不影响模块注册的组件。三行配置把同一个模块变成三种结果。Nuxt 给了一条不受开关影响的路：每个自动导入的东西都从 `#imports` 这个别名暴露出来，写一行 `import { ref } from '#imports'` 就定死了。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块开发
  - 自动导入
  - 图层
  - 模块作者
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 先把"自动导入"说清楚

如果你只写过 JavaScript，可以这样理解：正常的前端项目里，用一个函数之前必须在文件顶部 `import` 它。Nuxt 不用——它按目录约定去猜东西放在哪，然后把名字直接给你用。官方文档对这件事的说法是：多亏了这套有主张的目录结构，Nuxt 能自动导入 `app/components/`、`app/composables/` 和 `app/utils/` 里的东西，Vue 自己的响应式 API（`ref`、`computed`）和生命周期钩子也一样。

它跟传统的全局声明不是一回事，原文给了三点差别：类型保留、IDE 的补全和提示都在、**生产代码里只打进真正用到的那部分**。

问题就藏在"只打进用到的那部分"之外的那半句：这东西是 Nuxt 替你插进去的，而它可以被你关掉。

## `#imports` 是什么，为什么模块该用它

官方那一页有个小节叫 Explicit Imports，就一句话加一段代码：Nuxt 把每一个自动导入的东西都暴露在 `#imports` 这个别名下面，需要的时候可以用它把导入写成显式的。

```vue
<script setup lang="ts">
import { computed, ref } from '#imports'

const count = ref(1)
const double = computed(() => count.value * 2)
</script>
```

这段是文档原样，只是把上面那种"不写 import"的写法换成了写。差别在于：**它不依赖任何开关**。这一点在关闭自动导入那一节被明确说了——把 `imports.autoImport` 设成 `false` 之后，"自动导入被完全关闭，但仍然可以使用从 `#imports` 的显式导入"。

模块的作者处境很特别：你写的代码最终跑在别人的项目里，而别人项目的配置你看不见。`addImports` 那类注册函数做的事是把名字挂进自动导入清单，用户一关开关，清单就不生效了；显式导入那一行则一直在。

## 三个关闭配置分别关到哪一步

| 配置 | 关掉了什么 | 还留着什么 | 文档里的说法 |
| --- | --- | --- | --- |
| `imports.autoImport: false` | 组合式函数与工具的自动导入 | `#imports` 显式导入仍然可用 | "This will disable auto-imports completely" |
| `imports.scan: false` | 只扫描你自己代码那一半 | `ref`、`computed`、`watch` 这些框架函数照旧自动 | 自定义代码（比如 composables）需要在文件里手动导入 |
| `components.dirs: []` | 从你自己的 `~/components` 目录自动导入组件 | 模块注册的组件不受影响 | "this will not affect components added by modules" |
| `imports.presets` | 不是关闭，是反向配置 | 从第三方包里自动导入指定函数 | 例子是把 `vue-i18n` 的 `useI18n` 加进来 |

第三行那句括号很值得记住：**用户把组件目录清空了，你的模块注册的组件依然会出来**。反过来说，一个模块想靠"用户关掉了自动导入"来避免自己注册的组件被加载，是行不通的。

第四行的例子是文档原样，配置长这样：

```ts
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

## `imports.scan: false` 的两条限制，第一条落在图层上

文档在这一节后面单独挂了一条 warning，写明了这种半关闭配置的代价：

> 如果你的项目用了图层（layers）来组织代码，那么每一层的 composables 都必须显式导入，不能再依赖自动导入。

> 这会破坏图层系统的覆盖（override）特性。用了 `imports.scan: false`，就要理解这个副作用，并相应地调整你的架构。

图层是什么：把半个 Nuxt 应用（组件、组合式函数、配置）放在另一个目录或另一个包里，让主项目继承。覆盖特性指的是同名的东西按优先级取一个。`scan: false` 之后这条链路断了——每个层的东西都得在文件里点名。

对一个模块来说，这意味着用户如果同时用了图层加 `scan: false`，你的模块的自动导入清单和图层之间的优先级会互相看不见。**文档没有说明模块能否在这种情况下替用户绕过开关**，要确认只能自己开个 playground，把这两行配置一起加上，看注册的那个函数还拿不拿得到。

## 前缀是另一道防线，跟开关无关

模块开发的最佳实践那一页列了十条左右该遵守的规矩（异步 setup 超过 1 秒会警告、用生命周期钩子做一次性工作、TypeScript 友好、ESM、写文档、给 demo、别绑定版本号、跟 starter 的代码风格保持一致），其中专门有一条讲导出名字要加前缀，还给了对照表：

| 类型 | ❌ 避免 | ✅ 建议 |
| --- | --- | --- |
| 组件 | `<Button>`、`<Modal>` | `<FooButton>`、`<FooModal>` |
| 组合式函数 | `useData()`、`useModal()` | `useFooData()`、`useFooModal()` |
| 服务端路由 | `/api/track`、`/api/data` | `/api/_foo/track`、`/api/_foo/data` |

表里的 `nuxt-foo` 是举例的模块名，前缀取 `Foo` / `_foo`。这一页还特别指出服务端路由最需要加前缀，因为 `/api/auth`、`/api/login`、`/api/user` 这类路径**很可能已经被应用自己占用了**；给出的两种写法是 `/api/_foo/...`（带下划线前缀）和 `/_foo/...`（非 API 路由）。

顺带一句：这一页通篇没有出现 `#imports` 这个别名，它属于自动导入那一页。两页讲的是同一件事的两面——一面是名字怎么被挂上去，一面是名字挂上去之后可能撞车。

## 两个不属于开关问题的报错

第一种：**模板里 ref 没有解包**。文档挂了一条 warning，原话是自动导入的 `ref` 和 `computed` 在组件的 `<template>` 里不会被解包，原因是 Vue 对不是顶层的 ref 的处理方式。这不是配置问题，改法是把值在 `<script setup>` 里取好。

第二种：`Nuxt instance is unavailable`。文档对这句报错的解释很直接——你大概是在 Vue 或 Nuxt 生命周期的错误位置调了 Nuxt 的组合式函数。它的规则是这样：这类函数依赖一个临时记录的上下文实例（Vue 记当前组件，Nuxt 记 `nuxtApp`），并且在同一个 tick 里就清掉，服务端渲染时靠这个避免两个请求之间串数据。所以除了极少数例外，**只能在 Nuxt 插件、路由中间件或 Vue 的 setup 函数里调用，而且必须同步调用**——`await` 之后就不行了，除非在 `<script setup>`、`defineNuxtComponent`、`defineNuxtPlugin`、`defineNuxtRouteMiddleware` 这几处，那里 Nuxt 做了转换，跨过 `await` 仍保留同步上下文。

文档还补了一条 tip：在非 SFC 组件里要用需要 Nuxt 上下文的组合式函数，得用 `defineNuxtComponent` 包一层，而不是 `defineComponent`。

## 交付模块之前，这四个问题问一遍

- 用户写了 `imports: { autoImport: false }`，你的模块还能跑吗？跑不了的话，报错落在他的哪一行代码上。
- 你注册进自动清单的组件、组合式函数，名字带前缀了吗？服务端路由是不是走了 `/api/_你的模块名/`。
- 文档里有没有直接给出"从 `#imports` 显式导入"的用法示例，让用户关掉开关也能照抄。
- 用户同时用图层和 `imports.scan: false` 时，你的模块的行为有没有一句说明。这一条官方没有给答案，只能自己在 playground 里测。

下一篇写模块怎么往终端里打印信息：`useLogger` 的两个参数，和 `useTerminal` 那五个成员什么时候用得上。

## 本文依据

- https://nuxt.com/docs/4.x/guide/concepts/auto-imports
- https://nuxt.com/docs/4.x/guide/modules/best-practices
