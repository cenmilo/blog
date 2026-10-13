---
url: /posts/74dc2ae2a12b44f7015be2fa262350c8/
title: 405 - 组件 props 与 emits：Nuxt 文档实际覆盖到哪一层
date: 2027-02-24T09:00:00+08:00
updated: 2027-02-24T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：`defineProps<{...}>` 的用法本身是 Vue 的内容，Nuxt 4.x 文档里**没有**一页专门讲组件 props/emits 怎么定型——这本身就是要知道的事。Nuxt 文档真正明确写"完全定型"的是四处：布局的 props（`definePageMeta` 用对象语法传，官方说布局用 `defineProps` 定义后就 fully typed、有补全和类型检查）、`<NuxtPage>` 往页面透传的自定义 props（页面里 `defineProps<{ foobar: number }>()` 接，没定义则从 `useAttrs()` 取）、`error.vue` 的 `error: NuxtError` prop。另外 Nuxt 只把 `defineProps`/`defineEmits` 当作"编译器宏"的类比对象提了一句（`definePageMeta` 同它们一样会被编译掉、不能在自己组件里引用），以及 `vue.propsDestructure` 这个会影响 `defineProps` 解构行为的开关（默认 `true`）。

categories:
  - 前端开发

tags:
  - nuxt4
  - defineProps
  - defineEmits
  - 布局props
  - propsDestructure
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

搜"Nuxt 怎么给组件 props 定类型"会一无所获——因为这不是 Nuxt 管的层面。但 Nuxt 文档里确实有四处**明确写了"props 完全定型"**，以及两个会影响这块行为的细节。把边界划清，比抄一份 Vue 教程有用。

## 先划边界：props/emits 本身是 Vue 的事

Nuxt 文档里唯一把 `defineProps`/`defineEmits` 一起提到的地方，是拿它们做**类比**（pages 那一节讲 `definePageMeta` 时）：

> Much like `defineEmits` or `defineProps` (see Vue docs), `definePageMeta` is a **compiler macro**. It will be compiled away so you cannot reference it within your component. Instead, the metadata passed to it will be hoisted out of the component.

这句话对读者的价值在两点：一是官方自己把 props/emits 的用法指向 Vue 文档，二是"编译器宏会被编译掉、不能在组件内引用自己"这条规则同样适用于 `definePageMeta`。除此之外，组件内怎么写 props/emits，Nuxt 4.x 文档不重复规定。

## 明确写了"完全定型"的四处

**① 布局的 props。** 官方说法很确定（`define-page-meta`）：

> You can also pass an object with `name` and `props` to pass typed props to your layout component. When your layout defines props with `defineProps`, they will be fully typed in `definePageMeta`.

配合 `tip`："Layout props set via `definePageMeta` are fully typed based on the layout's `defineProps`. You'll get autocomplete and type-checking in your editor."

也就是布局用 `defineProps<{ sidebar?: boolean; title?: string }>()` 声明后，**页面侧 `definePageMeta` 里写的那些 props 会被按布局的声明检查、有补全**——这是 Nuxt 里跨文件类型打通的一处实例。

```vue [app/pages/dashboard.vue]
<script setup lang="ts">
definePageMeta({
  layout: {
    name: 'panel',
    props: {
      sidebar: true,
      title: 'Dashboard',
    },
  },
})
</script>
```

**② 页面接收 `<NuxtPage>` 透传的自定义 prop。** nuxt-page 那一节给的就是类型化写法：

```vue [app/pages/page.vue]
<script setup lang="ts">
const props = defineProps<{ foobar: number }>()

console.log(props.foobar) // Outputs: 123
```

以及没声明时的退路——官方："If you have not defined the prop with `defineProps`, any props passed down to `NuxtPage` can still be accessed directly from the page `attrs`"（用 `useAttrs()`）。

**③ `error.vue` 的 prop。** 官方示例就是类型化形式：`const props = defineProps<{ error: NuxtError }>()`（`directory-structure/app/error`，另见 056/328）。

**④ 服务端组件的 props。** server-components 那一节给的也是 `const props = defineProps<{ variant: string }>()`（见 008 相关）。

## 一个会改变行为的开关：`vue.propsDestructure`

nuxt-config 的 `## vue` 下：

### `propsDestructure`

> Enable reactive destructure for `defineProps`

- 类型：`boolean`
- 默认：`true`

默认开着，意味着从 `defineProps` 的返回值里解构出来用是**保持响应性**的。这条对类型没影响，但对"props 变了界面不更新"这类现象有直接影响，归到 props 这块讲是因为它只作用在 `defineProps` 上。

## 顺一条限制：app.config 里不能用

组件类型这块还有一条边界要记住：`app.config.ts` 里**不能直接 import Vue 组件**，因为 Nitro 处理 app config 时没有完整 Vue 组件支持（055/348 那条 Known Limitations）。要放组件相关配置，别走 `app.config`。

## 自查

- 组件内 props/emits 的写法归 Vue 文档；Nuxt 4.x 文档只在讲 `definePageMeta` 时把它们当"编译器宏"类比（编译器宏被编译掉、不能自引用）。
- Nuxt 明确写 fully typed 的是跨文件的 props 传递：布局（`definePageMeta` 对象语法 ↔ 布局的 `defineProps`）、`<NuxtPage>` 透传（没声明就 `useAttrs()`）、`error.vue` 的 `error: NuxtError`、服务端组件。
- `vue.propsDestructure` 默认 `true`，管 `defineProps` 解构后的响应性（不影响类型）。
- `app.config.ts` 里不能 import Vue 组件。

## 下一篇

下一篇 406 专讲 runtimeConfig 的类型：官方说 Nuxt 会用 untyped 自动生成接口，但保留手动增强 `RuntimeConfig` 与 `PublicRuntimeConfig` 的做法，以及为什么模块作者要增强 `@nuxt/schema`。

## 本文依据

- `definePageMeta` 是编译器宏、与 `defineEmits`/`defineProps` 同类的说法（`directory-structure/app/pages`）；布局 props fully typed 与 `tip`、对象语法示例（`api/utils/define-page-meta`、`api/components/nuxt-layout`、`directory-structure/app/layouts`）；`<NuxtPage>` 自定义 prop 的 `defineProps<{ foobar: number }>()` 与未声明时走 `useAttrs()`（`api/components/nuxt-page`）；`defineProps<{ error: NuxtError }>()`（`directory-structure/app/error`）；`defineProps<{ variant: string }>()`（`guide/concepts/server-components`）；`vue.propsDestructure` 默认 `true`（`api/nuxt-config`）；app.config 不能 import Vue 组件（`directory-structure/app/app-config`，见 055）。
