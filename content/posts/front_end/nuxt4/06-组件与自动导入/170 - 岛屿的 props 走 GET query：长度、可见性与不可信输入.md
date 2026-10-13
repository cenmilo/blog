---
url: /posts/9d6118f911540048c31d574dcd06fde2/
title: 170 - 岛屿的 props 走 GET query：长度、可见性与不可信输入
date: 2026-12-07T11:00:00+08:00
updated: 2026-12-07T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：169 说"可缓存"与"看不见路由"是同一件事的两面，这一篇讲这件实现细节本身——"**Props are serialized and sent as GET query parameters.**"，官方接着写："**This makes island responses cacheable, but it also means: props must be JSON-serializable；props are limited by URL length, so avoid passing large amounts of data；props may be visible in server access logs, CDN caches and `Referer` headers**"。`<NuxtIsland>` 那一节把可见性三处再列了一遍："**Component props and context are sent as GET query parameters to enable caching. Query parameters may be visible in server access logs, CDN caches, and HTTP `Referer` headers.**"。由这条机制推出的一段安全指引是官方写得最具体的一处："**Because props come from the request (URL query or body), treat them as untrusted input. Nuxt rejects the props most likely to leak through unintentionally: a top-level `as` that the island does not declare (an undeclared prop falls through as an attribute onto the island's root), and, with `vue.runtimeCompiler` enabled, a `template` anywhere in the props. Beyond that, avoid feeding props you have not validated into dynamic component resolution (`<component :is>`, `h()`, `resolveDynamicComponent()`, or a polymorphic `as` / `asChild` prop), since a string can resolve to any registered component or HTML element.**" 还有一条属性穿透的具体形状值得单独记："**Props a component does not declare fall through as attributes onto its single root element, so an island whose root is a polymorphic component (e.g. from `reka-ui` / `@nuxt/ui`) can receive attributes you did not bind. Set `defineOptions({ inheritAttrs: false })` on such islands, or declare the props you accept.**"官方给的替代写法是把判别值过一遍**白名单**再映射到已导入组件（`const allowed: Record<string, Component> = { a: CardA, b: CardB }` + `const component = allowed[props.variant] ?? CardA`）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 岛屿
  - 安全
  - props
  - 缓存

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


知道岛屿的 props 会变成 query 参数之后，我原先的写法连着被否掉三处：往 props 里塞一大段结构化数据、把用户输入直接透传给 `<component :is>`、以及给根是 `@nuxt/ui` 组件的岛屿传没声明的属性。这三件事官方在同一节里都写了。

## 机制：props 被序列化进 GET query

> Props are serialized and sent as **GET query parameters**. This makes island responses cacheable, but it also means:
>
> - props must be **JSON-serializable**
> - props are **limited by URL length**, so avoid passing large amounts of data
> - props **may be visible** in server access logs, CDN caches and `Referer` headers

这一句是把 169 那句"缓存键只有 name/props/context"落到实现层：**为了让 CDN 能缓存，props 必须进 URL**。三处后果分别对应三类常见写法错误。

| 后果 | 我原先会怎么写 | 官方给的标准 |
| --- | --- | --- |
| 必须 JSON 可序列化 | 传 `Map`、`Set`、函数、Date 实例、类实例 | "props must be JSON-serializable" |
| 受 URL 长度限制 | 把整个列表或富文本原样塞进 props | "avoid passing large amounts of data" |
| 会出现在日志与缓存里 | 传 token、用户邮箱、内部 ID 之类 | 三处暴露点：server access logs、CDN caches、`Referer` headers |

`<NuxtIsland>` 那一节把第三点又列了一遍，范围词多了一个 `context`：

> Component props **and context** are sent as GET query parameters to enable caching. Query parameters may be visible in server access logs, CDN caches, and HTTP `Referer` headers.

**`Referer` 这一项是很多人漏掉的路径**：岛屿页面里有外链时，query 会随请求发给对方站点。我把这条当作"岛屿 props 里不放敏感值"的最硬理由。

## 由此推出的一条安全原则：把 props 当不可信输入

官方这一段我认为值得逐句读，因为它给了"框架挡了什么"和"你要自己挡什么"的分界：

> Because props come from the request (URL query or body), **treat them as untrusted input**. Nuxt **rejects the props most likely to leak through unintentionally**: a top-level `as` that the island does not declare (an undeclared prop falls through as an attribute onto the island's root), and, with `vue.runtimeCompiler` enabled, a `template` anywhere in the props. Beyond that, **avoid feeding props you have not validated into dynamic component resolution** (`<component :is>`{lang="vue"}, `h()`, `resolveDynamicComponent()`, or a polymorphic `as` / `asChild` prop), **since a string can resolve to any registered component or HTML element**.

拆成三层：

| 层 | 内容 |
| --- | --- |
| 前提 | props 来自请求（URL query 或 body）→ 按不可信输入对待 |
| **框架已经拦的** | ① 岛屿未声明的顶层 `as`；② 开了 `vue.runtimeCompiler` 时，props 里任何位置的 `template` |
| **框架拦不了、你要做的** | 不要把未校验的 props 喂给四种动态解析：`<component :is>`、`h()`、`resolveDynamicComponent()`、多态的 `as`/`asChild` |

最后一句的理由值得单独抄下来：**"a string can resolve to any registered component or HTML element"**——一个字符串能把渲染目标换成任意已注册组件或原生标签。

## 属性穿透：根是多态组件时最容易被吃到

> Props a component does not declare **fall through as attributes onto its single root element**, so an island whose root is a polymorphic component (e.g. from `reka-ui` / `@nuxt/ui`) can receive **attributes you did not bind**. **Set `defineOptions({ inheritAttrs: false })` on such islands, or declare the props you accept.**

官方点名的正是 `reka-ui` / `@nuxt/ui` 这类根为多态组件的情况——这类库的组件本身就吃 `as`、`class`、`style` 等属性，穿透过去的值会被真的应用。

两个解法官方写成"或"，我的选择顺序是：**优先声明你要的 props**（`defineProps` 明确契约），只有当岛屿根本不该接受任何外部属性时才上 `inheritAttrs: false`。前者对调用方更可预期，后者是"关掉门"。

## 官方的替代写法：白名单映射

> To switch components based on caller input, **map a discriminator through an allowlist of imported components rather than passing the raw prop**:

```vue
<script setup lang="ts">
import type { Component } from 'vue'
import CardA from './CardA.vue'
import CardB from './CardB.vue'

const props = defineProps<{ variant: string }>()
const allowed: Record<string, Component> = { a: CardA, b: CardB }
const component = allowed[props.variant] ?? CardA
</script>

<template>
  <component :is="component" />
</template>
```

**三点值得照抄**：

1. 组件是**显式 import** 进来的（不是名字字符串），所以候选集封闭。
2. 传进来的是**判别值**（`variant: string`），不是组件名。
3. **有兜底**（`?? CardA`）——未在白名单里的值不会得到 `undefined`，也不会被解析成别的东西。

这一例同时满足了 161 那条对 `<component :is>` 的要求：不依赖 `resolveComponent` 的字面量分析，因为交给 `:is` 的是**组件对象**。

## 一处两节不一致：`context` 这个 prop

169 引过这句——岛屿要拿路由信息时"**pass it in explicitly, either as props or via the `context` prop on `<NuxtIsland>`**"；缓存键也写作 "name, props and context"。

但 `<NuxtIsland>` 那一节的 Props 清单只有五项：`name`（required）、`lazy`、`props`、`source`、`dangerouslyLoadClientComponents`——**没有 `context`**。

**我不替文档补结论**。可用的处理办法：`context` 按官方那句话写（它有明确的语义描述与读取路径 `nuxtApp.ssrContext.islandContext`），并**在自己的项目里验证一次**：岛屿内读 `nuxtApp.ssrContext?.islandContext`，看传入的 context 是否出现、以及换掉 context 值时响应是否重新请求（缓存键的证据）。验证成本很低：一次请求面板就能看出来。

## 我的岛屿 props 检查清单

| 检查 | 依据 |
| --- | --- |
| 值能用 `JSON.stringify` 干净表达吗 | "props must be JSON-serializable" |
| 数据量小到能进 URL 吗？大的话改成在服务端按 id 取 | "limited by URL length, so avoid passing large amounts of data" |
| 里面有没有敏感值（会进日志、CDN、`Referer`） | 三处可见性原文 |
| 岛屿内有没有 `<component :is>`、`h()`、`resolveDynamicComponent()`、`as`/`asChild` 直接吃 props | "Beyond that, avoid feeding props you have not validated..." |
| 组件是否声明了它接受的 props；根若是多态组件，是否 `inheritAttrs: false` | 属性穿透那条 |
| 有没有把 `props`/`context` 当"父页面传来的可信值"用 | "treat them as untrusted input" |

最后一条我特别标出来：169 那句"岛屿里的 `useRoute()` 反映的是岛屿自己的请求"与这条合起来看，**岛屿收到的 props 与它自己请求的 query 是同一来源**——都是外部输入，不是父组件的内存对象。

## 自查

1. props 为什么走 GET query？官方用哪个词点出这份收益？
2. 三条后果分别禁止了什么写法？`Referer` 这一项的实际暴露路径是什么？
3. props 与 context 都被序列化进 query，这句分别出现在哪两节？
4. 框架已经拦掉的是哪两种 props？各自的触发条件是什么（第二个要开什么）？
5. 四种"动态解析"是哪四种？官方的理由句是什么？
6. 未声明的 props 会落到哪里？官方点名的两类库是什么？两个解法你按什么顺序选？
7. 白名单示例的三个可照抄要点分别是什么？它为什么避开了 `resolveComponent` 的字面量限制？
8. `context` prop 这件事两节哪里对不上？你会怎么验证而不是猜？

## 下一篇

171 讲这套能力的开关与实验状态：`componentIslands` 的默认值是 `'auto'`、对象形态里 `selectiveClient`（`true` 与 `'deep'` 的差别）与 `remoteIsland`、以及 `nuxt-client` 只能用在本地 SFC 上那条警告。

## 本文依据

- `https://nuxt.com/docs/4.x/guide/concepts/server-components`
  - 原文（props 序列化与三条后果）："Props are serialized and sent as **GET query parameters**. This makes island responses cacheable, but it also means: props must be JSON-serializable；props are limited by URL length, so avoid passing large amounts of data；props may be visible in server access logs, CDN caches and `Referer` headers"
  - 原文（不可信输入整段）："Because props come from the request (URL query or body), treat them as untrusted input. Nuxt rejects the props most likely to leak through unintentionally: a top-level `as` that the island does not declare (an undeclared prop falls through as an attribute onto the island's root), and, with `vue.runtimeCompiler` enabled, a `template` anywhere in the props. Beyond that, avoid feeding props you have not validated into dynamic component resolution (`<component :is>`{lang="vue"}, `h()`, `resolveDynamicComponent()`, or a polymorphic `as` / `asChild` prop), since a string can resolve to any registered component or HTML element."
  - 原文（属性穿透与两种解法）："Props a component does not declare fall through as attributes onto its single root element, so an island whose root is a polymorphic component (e.g. from `reka-ui` / `@nuxt/ui`) can receive attributes you did not bind. Set `defineOptions({ inheritAttrs: false })` on such islands, or declare the props you accept."
  - 原文（白名单示例）："To switch components based on caller input, map a discriminator through an allowlist of imported components rather than passing the raw prop:" + `import type { Component } from 'vue'`、`import CardA from './CardA.vue'`、`import CardB from './CardB.vue'`、`const props = defineProps<{ variant: string }>()`、`const allowed: Record<string, Component> = { a: CardA, b: CardB }`、`const component = allowed[props.variant] ?? CardA`、`<component :is="component" />`
  - 原文（路由信息显式传入与缓存键）："If an island needs route information, pass it in explicitly, either as props or via the `context` prop on `<NuxtIsland>` (read inside the island from `nuxtApp.ssrContext.islandContext`)"；"The prefetched response is cached against the island's name, props and context, which is the same key the island uses when it mounts."；"Note that island responses being keyed only on name, props and context is exactly what keeps them cacheable independently of the page they appear on; this is also why they cannot see the current route."
- `https://nuxt.com/docs/4.x/api/components/nuxt-island`
  - 原文（note 与 Props 清单）："Component props and context are sent as GET query parameters to enable caching. Query parameters may be visible in server access logs, CDN caches, and HTTP `Referer` headers."；Props 五项：`name`（"Name of the component to render."，required）、`lazy`（"Make the component non-blocking."，default `false`）、`props`（"Props to send to the component to render."，type `Record<string, any>`）、`source`（"Remote source to call the island to render."）、`dangerouslyLoadClientComponents`（default `false`）——**清单中未列 `context`**
- `https://nuxt.com/docs/4.x/directory-structure/app/components`
  - 对照用（见 161、165）：`<component :is>` 的两种入口与 `resolveComponent` 字面量限制；单根元素要求
- `https://nuxt.com/docs/4.x/guide/going-further/experimental-features`
  - 对照用（见 171）：`vue.runtimeCompiler` 与 `componentIslands` 两个实验开关的位置
