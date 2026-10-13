---
url: /posts/772e2673cd54c0d2e214c19fa5bf7db4/
title: 026 - 页面与布局的分界：NuxtPage、NuxtLayout 与只有一层布局时的建议
date: 2026-10-20T11:00:00+08:00
updated: 2026-10-20T11:00:00+08:00
author: cmdragon

summary:
  摘要：布局是页面的包装器，用来放跨页面共用的界面（页头、页脚）。启用方式是在 app.vue 里用 <NuxtLayout> 包住 <NuxtPage />；布局文件用 <slot /> 来显示页面内容，没指定名字时使用 app/layouts/default.vue。给某个页面换布局有三条官方路径：在 definePageMeta 里设 layout 属性、设 <NuxtLayout> 的 name prop、在路由规则里设 appLayout。两条容易忽略的硬规则：布局名会被规范化成 kebab-case（someLayout 变成 some-layout），以及布局必须有单一根元素且这个根不能就是 <slot />，否则页面切换过渡会出问题。最后一条反而是官方用 important 块提醒的：如果你整个应用只有一层布局，建议直接用 app.vue，不要建布局。

categories:
  - 前端开发

tags:
  - nuxt4
  - 布局
  - nuxtlayout
  - 页面
  - 入门

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


上一篇我们把 `app.vue` + `<NuxtPage />` 接通，`/about` 终于显示了自己的内容。现在新问题来了：**首页和关于页都要有同一个页头和同一个页脚，难道在每个页面文件里各写一遍？** 布局就是为这件事存在的。

## 官方给的定义

> 布局是页面的包装器（wrapper around pages），它们为多个页面包含共用的界面，例如页头与页脚显示。布局是使用 `<slot />` 组件来显示**页面**内容的 Vue 文件。

三行信息量很足：

1. 它是**包装器**——不是页面本身，也不管路由。
2. 它的价值在于**跨页面共用**的界面。
3. 页面内容从哪儿冒出来，靠 `<slot />`。

## 怎么启用

官方在 layouts 目录页给的第一步就是这一小段：

```vue [app/app.vue]
<template>
  <NuxtLayout>
    <NuxtPage />
  </NuxtLayout>
</template>
```

注意这一段的**层级**：`<NuxtLayout>` 在外，`<NuxtPage />` 在内。写反了就没有"包装"关系了。

（Views 页给的例子里，外面还套了一层 `<div>`；layouts 目录页给的是裸的两个组件。两种都在文档里，不是错误。）

然后布局文件本体：

```vue [app/layouts/default.vue]
<template>
  <div>
    <AppHeader />
    <slot />
    <AppFooter />
  </div>
</template>
```

`AppHeader`、`AppFooter` 不用 import——025 讲的自动导入在这里生效（只要它们在 `app/components/` 下）。`<slot />` 那一行的位置，就是页面内容出现的位置。

官方对默认那一条写得很短：

> 如果没有指定布局，会使用 `app/layouts/default.vue`。

文件名 `default` 就是"默认"的意思，这是目录规则，不是配置项。

## 给某个页面换布局：官方给的三条路

layouts 页用列表原样给了三种：

> - 在页面里用 `definePageMeta` 设置 `layout` 属性。
> - 设置 `<NuxtLayout>` 的 `name` prop。
> - 在路由规则里设置 `appLayout` 属性。

**第一条**是最常见的：

```vue [app/pages/about.vue]
<script setup lang="ts">
definePageMeta({
  layout: 'custom',
})
</script>
```

对应目录：

```bash [Directory Structure]
-| layouts/
---| default.vue
---| custom.vue
```

**第二条**是一次性覆盖所有页面的默认布局：

```vue [app/app.vue]
<script setup lang="ts">
// You might choose this based on an API call or logged-in status
const layout = 'custom'
</script>

<template>
  <NuxtLayout :name="layout">
    <NuxtPage />
  </NuxtLayout>
</template>
```

文档里那句注释值得留着：**"你可能根据一次 API 调用结果或登录状态来选它"**——它暗示这一条路的价值是"布局名可以是运行时算出来的"。

**第三条**（`appLayout`）写在 `nuxt.config` 的路由规则里，官方标了 v4.3 起可用，例如 `'/admin': { appLayout: 'admin' }`。它解决的是"我不想逐个页面文件里写布局"，包括那些没有对应页面组件的路径（官方举的例子是 catch-all 页面）。这一条在路由规则那一篇展开。

## 两条硬规则，会直接报错或行为怪异

**一、布局名会被规范化成 kebab-case。**

> 布局名会被规范化为 kebab-case，所以 `someLayout` 变成 `some-layout`。

也就是说 `layouts/someLayout.vue` 这个文件，你要写的名字是 `some-layout`。目录里嵌套时更明显，官方给了三行对照表：

| 文件 | 布局名 |
| --- | --- |
| `~/layouts/desktop/default.vue` | `desktop-default` |
| `~/layouts/desktop-base/base.vue` | `desktop-base` |
| `~/layouts/desktop/index.vue` | `desktop` |

（重复的路径段会被去掉，所以 `desktop/default` 是 `desktop-default` 而不是 `desktop-desktop-default`。）官方还顺势给了一句建议：**让文件名和它生成的名字一致**，例如 `layouts/desktop/DesktopDefault.vue` → `desktop-default`。

**二、单一根元素，而且根不能就是 `<slot />`。**

> 与其它组件不同，你的布局**必须有单一根元素**，以便 Nuxt 在布局之间应用过渡——并且这个根元素**不能是 `<slot />`**。

这条跟 025 讲的"页面内容显示在 `<slot />` 处"合起来读才有意义：`<slot />` 必须在某个根元素**里面**，不能整个布局就一个 `<slot />`。上面那个 `default.vue` 例子用 `<div>` 包住三行，正是在满足这条。

## 官方那条反直觉的建议

这一条我用 important 级别标注它，因为新手几乎都会多建一层：

> 如果你的应用里只有一层布局，我们建议使用 `app.vue` 来代替。

Views 页同一件事又说了一遍（"If you only have a single layout..."）。意思是：**页头页脚全站唯一，那就直接写在 `app.vue` 里，不要建 `layouts/default.vue`。** 布局这一层是为"有多种包装方式"准备的，不是为"有个地方放页头"准备的。

## 这一篇故意没讲的

layouts 页还有三块内容，等你写到需要时再回来：

- **给布局传 props**（`definePageMeta({ layout: { name: 'panel', props: { ... } } })`，官方标注 v4.4 起，且类型会根据布局的 `defineProps` 自动推出来）；
- **运行时切换布局**（`setPageLayout('custom')`，配合 `definePageMeta({ layout: false })`）；
- **在页面内部自己用 `<NuxtLayout>`**（此时它不能是页面的根元素）。

这三块在布局目录专篇逐条展开。

## 自查

- [ ] `<NuxtLayout>` 和 `<NuxtPage />` 谁包谁？写反了会怎样？
- [ ] 页面内容在布局文件的哪个位置显示出来？
- [ ] `layouts/someLayout.vue` 生成的布局名是什么？
- [ ] 为什么布局必须有单一根元素、而且根不能是 `<slot />`？
- [ ] 全站只有一层页头页脚时，官方的建议是什么？

## 下一篇

027 讲一个更少用但会救命的口子：当 `<head>` 里的东西还不够、你要直接改 HTML 模板本身时，官方给的是 Nitro 插件加 `render:html` 钩子。

## 本文依据

- `getting-started/views`（Layouts 一节：布局是页面包装器、包含跨页面共用界面如页头页脚、布局是使用 <slot /> 显示页面内容的 Vue 文件、app/layouts/default.vue 默认使用、自定义布局通过页面元数据设置、note"只有一个布局时建议改用 app.vue 配 <NuxtPage />"、示例 app/app.vue 中 <NuxtLayout> 包 <NuxtPage />、default.vue 里 AppHeader + <slot /> + AppFooter）
- `directory-structure/app/layouts`（Enable Layouts：在 app.vue 添加 <NuxtLayout> 包住 <NuxtPage />；使用布局的三种方式 definePageMeta 的 layout、<NuxtLayout> 的 name prop、route rules 的 appLayout；布局名规范化为 kebab-case（someLayout→some-layout）；未指定布局时用 app/layouts/default.vue；只有一个布局时建议使用 app.vue；layouts 必须单一根元素且根不能是 <slot />；Named Layout 的 layouts/custom.vue 与 definePageMeta({layout:'custom'}) 示例、app.vue 中 :name="layout" 与"根据 API 调用或登录状态选择"的注释；嵌套目录命名对照表 desktop-default/desktop-base/desktop 与建议文件名匹配；setPageLayout 与 layout:false；appLayout 标注 v4.3 与 /admin、/dashboard/**、/landing:false 例子；props 标注 +4.4、对象语法、类型基于 defineProps 自动推导；页面内使用 <NuxtLayout> 时不能作根元素；目录内组件异步导入的性能提示）
