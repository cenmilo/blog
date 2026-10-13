---
url: /posts/6f7bbe86a3a512e0636e30d801385544/
title: 094 - NuxtLayout 与布局的启用时机：name、fallback 与 layoutRef
date: 2026-11-12T10:00:00+08:00
updated: 2026-11-12T10:00:00+08:00
author: cmdragon

summary:
  摘要：`<NuxtLayout>` 的官方定位是"在页面和错误页上显示布局"，用它可以在 `app.vue` 或 `error.vue` 上**激活** `default` 布局——注意这是"激活"而不是"引用某个文件"。它文档化的 prop 只有一个：`name`（类型 `string | false`、默认 `default`，可以传字符串、响应式引用或计算属性，且**必须**匹配 `app/layouts/` 里对应布局文件的名字，或者传 `false` 来禁用布局）。名字有一条规范化规则：布局名会被规范化为 kebab-case，所以文件名叫 `errorLayout.vue` 时，传给 `name` 的应当是 `error-layout`。另一个 prop 是 `fallback`（类型 `string`、默认 `null`），它存在的前提正是官方写明的行为——**如果传给 `name` 的布局无效，则什么都不渲染**，此时 `fallback` 指定的布局才会顶上，而它同样必须匹配 `app/layouts/` 里的文件名。此外这个组件接受任意额外 prop 并作为 attribute 暴露（官方示例 `title="I am a custom layout"` 在 `custom.vue` 里用 `$attrs.title` 或 `useAttrs().title` 取），但如果布局是通过 `definePageMeta` 的对象式 `layout` 指定的，props 会自动传给布局组件、布局侧要用 `defineProps` 接收。过渡那一节给出两条要点：`<NuxtLayout />` 通过 `<slot />` 渲染传入内容、随后被包到 Vue 的 `<Transition />` 外面来激活布局过渡，而**为了让它按预期工作，官方建议 `<NuxtLayout />` 不要作为页面组件的根元素**；拿布局实例要走 `ref.value.layoutRef`。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxtlayout
  - 布局
  - fallback
  - 过渡

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


布局文件放在哪儿、名字怎么生成，前面讲过了。这一篇只看那个组件本身：**`<NuxtLayout>` 什么时候真的把布局套上去、名字传错了会怎样、参数怎么进到布局里。**

## 它是"激活"布局，不是"引用"文件

官方定位："Nuxt 提供 `<NuxtLayout>` 组件，在页面和错误页上显示布局。"紧接着的用法说明更准确：**你可以使用 `<NuxtLayout />` 组件在 `app.vue` 或 `error.vue` 上激活 `default` 布局。**

```vue [app/app.vue]
<template>
  <NuxtLayout>
    some page content
  </NuxtLayout>
</template>
```

"激活"这个词解释了它的两个行为：

- 不写这个组件，`app/layouts/` 里的文件不会因为存在就生效。
- 什么都不传时，它激活的是 `default`——也就是 `name` 的默认值，而不是"自动匹配当前页面"。

## `name`：必须对得上，或者传 `false`

官方对 `name` 的完整描述：指定要渲染的布局名，可以是字符串、响应式引用或计算属性，它**必须**匹配 `app/layouts/` 目录下对应布局文件的名字，**或者传 `false` 来禁用布局**。

- 类型：`string | false`
- 默认：`default`

传变量是官方示例的做法：

```vue [app/pages/index.vue]
<script setup lang="ts">
// layouts/custom.vue
const layout = 'custom'
</script>

<template>
  <NuxtLayout :name="layout">
    <NuxtPage />
  </NuxtLayout>
</template>
```

注意注释那行——`const layout = 'custom'` 之所以能用，是因为磁盘上真有 `layouts/custom.vue`。

### 一条规范化规则

官方的说明："请注意，**布局名会被规范化为 kebab-case**，所以如果你的布局文件名为 `errorLayout.vue`，当它作为 `name` 属性传给 `<NuxtLayout />` 时会变成 `error-layout`。"

```vue [error.vue]
<template>
  <NuxtLayout name="error-layout">
    <NuxtPage />
  </NuxtLayout>
</template>
```

这条规则的方向容易记反：**是文件名被转换成名字，不是你以为什么就有什么。** `errorLayout.vue` 这个文件，你要传 `error-layout`；传 `errorLayout` 是无效的。

## 名字传错了：默认什么都不渲染，`fallback` 才顶上

`fallback` 这个 prop 存在的前提，官方在 `fallback` 的说明里写得非常直白：**"如果传了一个无效的布局给 `name` 属性，则不会渲染任何布局。请指定一个 `fallback` 布局在这个情形下渲染。"**

- 类型：`string`
- 默认：`null`
- 它同样**必须**匹配 `app/layouts/` 里对应布局文件的名字

这一句是排查"布局莫名消失"的关键。默认值 `null` 意味着不配 `fallback` 时，无效名字的后果不是退回 `default`，也不是报错，而是**没有布局**——内容还在，外壳没了。

| 你传的 `name` | 磁盘上有没有对应文件 | 结果 |
| --- | --- | --- |
| 不传 | 有 `default.vue` | 渲染 `default` 布局（默认值就是 `default`） |
| `'custom'` | 有 `custom.vue` | 渲染 `custom` |
| `'errorLayout'` | 只有 `errorLayout.vue` | 无效（规范化后应是 `error-layout`）→ 不渲染任何布局 |
| 同上 + `fallback="default"` | 有 `default.vue` | 渲染 `default` |
| `false` | —— | 官方语义：禁用布局 |

## 参数怎么进到布局里：两条路，走法不同

### 路径一：额外 prop → 变成 attribute

官方说明：`NuxtLayout` 还接受你可能需要的任何额外 prop，**这些自定义 prop 随后作为 attribute 提供**。

```vue [app/pages/some-page.vue]
<template>
  <div>
    <NuxtLayout
      name="custom"
      title="I am a custom layout"
    >
      <!-- ... -->
    </NuxtLayout>
  </div>
</template>
```

官方的取法说明：在上面的例子里，`title` 的值在模板中可通过 `$attrs.title` 使用，在 `custom.vue` 的 `<script setup>` 中可通过 `useAttrs().title` 使用。

```vue [app/layouts/custom.vue]
<script setup lang="ts">
const layoutCustomProps = useAttrs()

console.log(layoutCustomProps.title) // I am a custom layout
</script>
```

这条路的特征是**布局侧不声明也要能拿到**——因为它走的是 attribute，不是 prop。

### 路径二：页面元数据里的对象式 `layout` → 走 `defineProps`

官方说明：当使用 `definePageMeta` 的**对象语法**指定 `layout` 时，props 会被自动传给布局组件，**布局可以用 `defineProps` 接收它们**：

```vue [app/pages/dashboard.vue]
<script setup lang="ts">
definePageMeta({
  layout: {
    name: 'admin',
    props: {
      sidebar: true,
    },
  },
})
</script>
```

```vue [app/layouts/admin.vue]
<script setup lang="ts">
const props = defineProps<{
  sidebar?: boolean
}>()
</script>
```

两条路的差别不是写法风格，而是**接收声明的位置**：路径一用 `useAttrs()`，路径二用 `defineProps`。用错了取法就是 `undefined`，而且页面照样渲染，很难在第一时间发现。

## 做过渡时：官方建议它别当根元素

`<NuxtLayout />` 与 `<Transition>` 的关系，官方写得很具体：**`<NuxtLayout />` 通过 `<slot />` 渲染传入的内容，随后（该内容）被包裹到 Vue 的 `<Transition />` 组件外面以激活布局过渡。为了让这按预期工作，官方建议 `<NuxtLayout />` 不要是页面组件的根元素。**

```vue [app/pages/index.vue]
<template>
  <div>
    <NuxtLayout name="custom">
      <template #header>
        Some header template content.
      </template>
    </NuxtLayout>
  </div>
</template>
```

```vue [app/layouts/custom.vue]
<template>
  <div>
    <!-- named slot -->
    <slot name="header" />
    <slot />
  </div>
</template>
```

这段还顺带说明了另一件事：**布局可以有名插槽**。页面用 `<template #header>` 往里灌内容，布局侧用 `<slot name="header" />` 接，普通 `<slot />` 接默认内容。

配置侧的对应项也要分清，两个是分开的开关：

```ts
export default defineNuxtConfig({
  app: {
    layoutTransition: { name: 'layout', mode: 'out-in' },
  },
})
```

而官方的页面过渡那节还有一条会直接影响效果的说明：**如果你在换页面的同时也换布局，那里设置的页面过渡不会运行；应当改设布局过渡。** 更根本的一条限制是——因为 Nuxt 使用 Vue 的 `<Transition>` 组件，你想动画化的页面或布局必须有**单个根元素**；多根元素（片段）无法被动画化，过渡不会运行，在路由之间导航甚至可能报错，Nuxt 会在开发模式下就此警告。

## 拿布局实例：走 `layoutRef`

官方原文："要获取一个布局组件的 ref，通过 `ref.value.layoutRef` 访问它。"

```vue [app/app.vue]
<script setup lang="ts">
const layout = ref()

function logFoo () {
  layout.value.layoutRef.foo()
}
</script>

<template>
  <NuxtLayout ref="layout">
    default layout
  </NuxtLayout>
</template>
```

```vue [app/layouts/default.vue]
<script setup lang="ts">
const foo = () => console.log('foo')
defineExpose({
  foo,
})
</script>

<template>
  <div>
    default layout
    <slot />
  </div>
</template>
```

和 `<NuxtPage>` 的 `pageRef` 是同一个套路：`ref` 指向的是包装组件，真正的布局在它里面。

## 自查

1. 官方用哪个词描述 `<NuxtLayout>` 与 `default` 布局的关系？不写这个组件会怎样？
2. 文件叫 `errorLayout.vue` 时，`name` 该传什么？依据是哪一句？
3. 传了一个无效的布局名、且没配 `fallback`，官方说会发生什么？
4. 布局侧要拿到 `<NuxtLayout>` 上写的额外 prop，用哪个 API？对象式 `layout` 的 props 又用哪个 API？
5. 官方为什么建议 `<NuxtLayout>` 不要当页面根元素？同时换布局和页面时应该配哪一个过渡？
6. 要调用布局里 `defineExpose` 出来的方法，`ref` 应该怎么取？

## 下一篇

095 讲 `<NuxtErrorBoundary>`：它只兜默认插槽里的客户端错误、底层是 `onErrorCaptured`、`#error` 插槽拿到 `error` 与 `clearError`，以及"把错误设成 null 会重新渲染默认插槽"这条容易踩的前提。

## 本文依据

- `https://nuxt.com/docs/4.x/api/components/nuxt-layout`
  - 原文："Nuxt provides the \<NuxtLayout> component to show layouts on pages and error pages."；"You can use `<NuxtLayout />` component to activate the `default` layout on `app.vue` or `error.vue`."
  - 原文示例：`<NuxtLayout>some page content</NuxtLayout>`（文件标注 `app/app.vue`）
  - 原文（Props - name）："`name`: Specify a layout name to be rendered, can be a string, reactive reference or a computed property. It **must** match the name of the corresponding layout file in the `app/layouts/` directory, or `false` to disable the layout."；type `string | false`；default `default`
  - 原文示例：`const layout = 'custom'` 配注释 `// layouts/custom.vue`，模板 `<NuxtLayout :name="layout"><NuxtPage /></NuxtLayout>`
  - 原文（note）："Please note the layout name is normalized to kebab-case, so if your layout file is named `errorLayout.vue`, it will become `error-layout` when passed as a `name` property to `<NuxtLayout />`."；示例 `<NuxtLayout name="error-layout"><NuxtPage /></NuxtLayout>`（文件标注 `error.vue`）
  - 原文（fallback）："If an invalid layout is passed to the `name` prop, no layout will be rendered. Specify a `fallback` layout to be rendered in this scenario. It **must** match the name of the corresponding layout file in the `app/layouts/` directory."；type `string`；default `null`
  - 原文（Additional Props）："`NuxtLayout` also accepts any additional props that you may need to pass to the layout. These custom props are then made accessible as attributes."；示例 `<NuxtLayout name="custom" title="I am a custom layout">`；"In the above example, the value of `title` will be available using `$attrs.title` in the template or `useAttrs().title` in `<script setup>` at custom.vue."；布局侧 `const layoutCustomProps = useAttrs()`、`console.log(layoutCustomProps.title) // I am a custom layout`
  - 原文（Layout Props from Page Meta）："When using `definePageMeta` with the object syntax for `layout`, props are automatically passed to the layout component. The layout can receive them with `defineProps`"；示例 `layout: { name: 'admin', props: { sidebar: true } }` 与 `const props = defineProps<{ sidebar?: boolean }>()`
  - 原文（Transitions）："`<NuxtLayout />` renders incoming content via `<slot />`, which is then wrapped around Vue's `<Transition />` component to activate layout transition. For this to work as expected, it is recommended that `<NuxtLayout />` is **not** the root element of the page component."
  - 原文示例：页面侧 `<div><NuxtLayout name="custom"><template #header>Some header template content.</template></NuxtLayout></div>`；布局侧 `<!-- named slot --> <slot name="header" /> <slot />`
  - 原文（Layout's Ref）："To get the ref of a layout component, access it through `ref.value.layoutRef`."；示例 `layout.value.layoutRef.foo()`、`<NuxtLayout ref="layout">default layout</NuxtLayout>`、布局侧 `defineExpose({ foo })` 与 `const foo = () => console.log('foo')`
  - 原文 Source：`packages/nuxt/src/app/components/nuxt-layout.ts`
- `https://nuxt.com/docs/4.x/getting-started/transitions`
  - 原文（important）："Because Nuxt uses Vue's `<Transition>` component, a page or layout you want to animate must have a **single root element**. A page or layout with multiple root elements (a fragment) cannot be animated, so the transition will not run and navigating between routes may error. Nuxt warns about this in development. Wrap the template in a single root element (for example a `<div>`)."
  - 原文（note）："If you are changing layouts as well as pages, the page transition you set here will not run. Instead, you should set a layout transition."
  - 原文（Layout Transitions）："You can enable layout transitions to apply an automatic transition for all your layouts."；示例 `app: { layoutTransition: { name: 'layout', mode: 'out-in' } }`
  - 原文（note）："Nuxt leverages Vue's `<Transition>` component to apply transitions between pages and layouts."
