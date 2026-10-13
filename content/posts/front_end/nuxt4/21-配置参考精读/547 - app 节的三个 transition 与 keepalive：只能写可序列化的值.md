---
url: /posts/76ac9c237ebdcc099e839939fc7d8ccf/
title: 547 - app 节的三个 transition 与 keepalive：只能写可序列化的值
date: 2027-04-12T10:00:00+08:00
updated: 2027-04-12T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`app` 节下这四个开关共享同一句规则：**"This can be overridden with `definePageMeta` on an individual page."**，其中三个还多一句 **"Only JSON-serializable values are allowed."**。类型各有差别：`pageTransition` 与 `layoutTransition` 都是 `boolean | TransitionProps`、`keepalive` 与 `viewTransition` 是 `boolean`，四项默认全是 `false`。`viewTransition` 有一条前置：**只有在你配置里开了实验性的 View Transitions 支持才有效果**（那是 `experimental.viewTransition: true`，不是 `app.viewTransition`）。另外要分清三处设置位置的差别：`app.pageTransition` 说的"可被 `definePageMeta` 覆盖"，而用 `<NuxtPage :transition="...">` 设的那一条，官方明确写了**不能**被 `definePageMeta` 覆盖。

categories:
  - 前端开发

tags:
  - nuxt4
  - 过渡动画
  - keepalive
  - definePageMeta
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

页面切换动画有四个地方能配：`app.pageTransition`、`app.layoutTransition`、`<NuxtPage :transition>`、`definePageMeta({ transition })`。

前两个是"整站默认值"，后两个是"这一页要什么"。它们之间的覆盖关系，官方写得比想象中细。

## 四节并排看

| 子项 | 官方说明 | 类型 | 默认 |
| --- | --- | --- | --- |
| `app.keepalive` | Default values for KeepAlive configuration between pages. | `boolean` | `false` |
| `app.layoutTransition` | Default values for layout transitions. | `boolean \| TransitionProps` | `false` |
| `app.pageTransition` | Default values for page transitions. | `boolean \| TransitionProps` | `false` |
| `app.viewTransition` | Default values for view transitions. | `boolean` | `false` |

四节的后半句共享同一套措辞：**"This can be overridden with `definePageMeta` on an individual page."**；`keepalive`、`layoutTransition`、`pageTransition` 三项还各带一句 **"Only JSON-serializable values are allowed."**

这句限制就是新手最容易撞的墙：**配置里的动画对象必须是 JSON 能表达的**。Vue 的 `<Transition>` 里那些钩子函数（`@enter`、`@leave` 之类回调）没法写在这里——真要函数形态的钩子，只能在组件里用 `<Transition>` 或走运行时那条路。`viewTransition` 那一节没有这句（它的类型是 `boolean`，本来也没什么可序列化）。

一个措辞与类型不一致的地方要留意：`keepalive` 写的是 **"Default values for KeepAlive configuration"**（复数 values），类型却是 `boolean`。它只留了一个 **See** 指向 Vue 的 KeepAlive 文档。**"能配哪些 KeepAlive 字段"在这一节没有展开**，只能按 `boolean` 用。

## viewTransition 有一个开关在别处

`app.viewTransition` 的说明多一段前置：

> This only has an effect when **experimental** support for View Transitions is enabled in your nuxt.config file.

那句话指向的链接是 transitions 页的 View Transitions API 小节，那里给的启用写法是**另一个键**：

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    viewTransition: true,
  },
})
```

**两个同名键分属两处**：`experimental.viewTransition` 是"要不要用这套实验实现"，`app.viewTransition` 是"整站默认开还是关这个效果"。所以只设 `app.viewTransition: true` 而没开 `experimental.viewTransition`，按官方那句 "only has an effect when ... enabled" 是不起作用的。

transitions 那一节还给了一条官方注释，正是"默认关、单页开"的用法：

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  app: {
    // Disable view transitions globally, and opt-in on a per page basis
    viewTransition: false,
  },
})
```

## 三处设置位置的覆盖关系不一样

配置参考这一侧说 `app.pageTransition` **"can be overridden with `definePageMeta`"**。但换到 `<NuxtPage />` 那一侧，官方在同一页写了相反的话：

> When `<NuxtPage />` is used in `app.vue`, transitions can be configured with the `transition` prop to activate transitions globally.

然后紧跟一条 `::note`：**"Remember, this page transition cannot be overridden with `definePageMeta` on individual pages."**

所以三处的规则不是一条，而是两条：

| 你设的位置 | 官方说法 | 单页能否用 `definePageMeta` 覆盖 |
| --- | --- | --- |
| `app.pageTransition` | Default values for page transitions | **能**（原文："can be overridden with `definePageMeta`"） |
| `<NuxtPage :transition>` | 在 `app.vue` 里全局激活过渡 | **不能**（原文 `::note`） |
| `definePageMeta({ transition })` | 单页设置 | — |

官方给的 `<NuxtPage>` 写法长这样：

```vue [app/app.vue]
<template>
  <div>
    <NuxtLayout>
      <NuxtPage
        :transition="{
          name: 'bounce',
          mode: 'out-in',
        }"
      />
    </NuxtLayout>
  </div>
</template>
```

**这条差别很容易踩**：想全局设一次、再让个别页面自己换动画，就得走 `app.pageTransition`；一旦把全局动画写在 `<NuxtPage :transition>` 上，单页的 `definePageMeta` 就不再生效。

## 自查

- 四项默认全是 `false`；`pageTransition`/`layoutTransition` 是 `boolean | TransitionProps`，`keepalive`/`viewTransition` 是 `boolean`。
- 三项写着 **"Only JSON-serializable values are allowed"**——过渡钩子函数不能放这里。
- `keepalive` 的措辞是"KeepAlive 配置的默认值"但类型只有 `boolean`，可配字段本节没展开，只留了 Vue 文档链接。
- `app.viewTransition` 有前置：要 `experimental.viewTransition` 开启才有效果；两处同名键管的事不同。
- 全局动画写在 `app.pageTransition` 可被 `definePageMeta` 覆盖；写在 `<NuxtPage :transition>` 则官方明说**不能**被覆盖。

## 下一篇

下一篇 548 读 `app` 节剩下的七个项：`rootId`/`rootTag`/`rootAttrs` 与 `teleportId`/`teleportTag`/`teleportAttrs`、`spaLoaderAttrs`/`spaLoaderTag`，官方对其中两行的说明文字是完全一样的——这一节有复制痕迹。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`### keepalive`/`### layoutTransition`/`### pageTransition`/`### viewTransition` 四节全部记载：各自定义句、definePageMeta 覆盖句、Only JSON-serializable values 那句（viewTransition 节无此句）、四项类型与默认 `false`、See 指向 Vue KeepAlive 与 Vue Transition 文档、viewTransition 的"only has an effect when experimental support ... enabled"及其指向 transitions 页的链接）
- https://nuxt.com/docs/4.x/getting-started/transitions（View Transitions API (Experimental) 小节与 `experimental.viewTransition: true` 示例、`app.viewTransition: false` 全局关单页开的示例与注释；Transition with NuxtPage 小节的 `:transition` 用法示例与"this page transition cannot be overridden with `definePageMeta`"的 `::note`）
