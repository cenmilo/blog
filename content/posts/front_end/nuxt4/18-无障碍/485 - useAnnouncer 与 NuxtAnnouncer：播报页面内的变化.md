---
url: /posts/1c1077035663d9afd41b0b19ad2620ca/
title: useAnnouncer 与 NuxtAnnouncer：播报页面内的变化
date: 2027-03-22T11:00:00+08:00
updated: 2027-03-22T11:00:00+08:00
author: cmdragon

summary:
  摘要：表单校验失败、提交成功、数据加载中——这些都不换标题，所以轮不到 `<NuxtRouteAnnouncer>`，官方给的是 **v4.4.2+** 那一对：`useAnnouncer` 负责写消息、`<NuxtAnnouncer>` 负责把它渲进 DOM，两者缺一不可（官方的 `callout` 原话是"需要把 `<NuxtAnnouncer>` 加进应用，播报才会渲染到 DOM"）。官方在组件页给了一张四行对照表，两列差别是：用途（路由变化／任意动态内容）、触发（导航时自动／手动调 `polite()`/`assertive()`）、消息来源（页面 `<title>`／开发者提供）、**`atomic` 默认值 `false` 对 `true`**——这是两个组件唯一取值不同的属性。`useAnnouncer` 的三组官方示例正好对着三类场景：表单校验用 `assertive`，加载状态与搜索结果用 `polite`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 可访问性
  - useAnnouncer
  - NuxtAnnouncer
  - 屏幕阅读器
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

点提交，接口 400，页面上红字写着"邮箱必填"。屏幕阅读器什么都不说——因为标题没变、路由也没变。

这类"页面内变化"官方给了一对新的能力：`useAnnouncer` 与 `<NuxtAnnouncer>`，都是 **v4.4.2+**。

## 一对，缺一个就不响

组件页的定性：

> The <NuxtAnnouncer> component adds a hidden element to announce dynamic content changes to assistive technologies.

**加一个隐藏元素**，用来播报动态内容变化。放的位置官方写的是 `app.vue` 或 `app/layouts/`，而且示例里两个组件并排：

```vue [app/app.vue]
<template>
  <NuxtAnnouncer />
  <NuxtRouteAnnouncer />
  <NuxtLayout>
    <NuxtPage />
  </NuxtLayout>
</template>
```

**两个可以共存，各管一类事件**。而 `useAnnouncer` 那一页末尾第一条 `callout` 是硬性条件：

> You need to add the `<NuxtAnnouncer>` component to your app for the announcements to be rendered in the DOM.

也就是说：**只调 composable 不装组件，消息没有落点**。这正好回答 484 那篇在路由那对上没写清的问题——至少这一对，官方把要求说明白了。

## 分工：官方那张四行表

组件页有一节标题就叫 "Differences from `<NuxtRouteAnnouncer>`"，四行原文照录：

| 方面 | `<NuxtRouteAnnouncer>` | `<NuxtAnnouncer>` |
| --- | --- | --- |
| Purpose | Announces route/page changes | Announces any dynamic content |
| Trigger | Automatic on navigation | Manual via `polite()`/`assertive()` |
| Message source | Page `<title>` | Developer-provided |
| `atomic` default | `false` | `true` |

前三行是定位差别，第四行是**唯一一处默认值不同**：`atomic` 在路由播报里默认 `false`（只播变化），在通用播报里默认 **`true`**（更新时把整块内容读出来）。

两个组件的属性表措辞几乎相同（"Controls if screen readers announce only changes or the entire content..."），只有默认值一句不同——**改默认值时别照抄另一页**。

## useAnnouncer 的接口

定性句先给分工："Unlike `useRouteAnnouncer` which automatically announces route changes, `useAnnouncer` gives you **manual control over what and when to announce**"，适用对象官方列了四类：表单校验、异步操作、toast 通知、实时内容变化。

参数一个（`politeness`，默认 `polite`），属性两项：

| 属性 | 类型 | 说明 |
| --- | --- | --- |
| `message` | `Ref<string>` | 当前要播报的消息 |
| `politeness` | `Ref<'polite' \| 'assertive' \| 'off'>` | 紧急度 |

方法三个，官方比 484 那一组多写了一句"什么时候用"：

| 方法 | 官方说明 |
| --- | --- |
| `set(message, politeness = "polite")` | Sets the message to announce with its urgency level. |
| `polite(message)` | 设 `politeness = "polite"`。**用于不紧急、可以等屏幕阅读器把手上的事做完的更新** |
| `assertive(message)` | 设 `politeness = "assertive"`。**用于应当立即打断屏幕阅读器的紧急更新** |

有一处类型细节值得对读：**`useAnnouncer` 的 `politeness` 写的是 `Ref<'polite' | 'assertive' | 'off'>`，而 `useRouteAnnouncer` 那一页同一属性写的是 `Ref<string>`**。两页类型标注不一致（文档未解释），按各自页面写就可能出现一边收窄一边放宽的情况。

## 三组官方示例

**表单校验用 `assertive`：**

```vue [app/components/LoginForm.vue]
<script setup lang="ts">
const { assertive } = useAnnouncer()

function validateForm () {
  const errors = []
  if (!email.value) { errors.push('Email is required') }
  if (!password.value) { errors.push('Password is required') }

  if (errors.length) {
    assertive(`Form has ${errors.length} errors: ${errors.join(', ')}`)
    return false
  }
  return true
}
</script>
```

播报的是**错误条数加错误清单拼出来的一句话**，不是逐条红字——红字本身在 DOM 里，但不会被自动读。

**加载状态跟着 `status` 播：**

```vue [app/pages/dashboard.vue]
<script setup lang="ts">
const { polite } = useAnnouncer()

const { data, status } = await useFetch('/api/data')

watch(status, (newStatus) => {
  if (newStatus === 'pending') {
    polite('Loading data...')
  } else if (newStatus === 'success') {
    polite('Data loaded successfully')
  }
})
</script>
```

这里用的正是 446 讲过的那个 `status`（`pending`/`success` 等四值之一），**不是 `pending` 那个布尔**——`idle` 段官方在这里没有对应播报。

**搜索结果数用 `polite`：**

```vue [app/components/Search.vue]
<script setup lang="ts">
const { polite } = useAnnouncer()

const results = ref([])

watch(results, (newResults) => {
  polite(`Found ${newResults.length} results`)
})
</script>
```

这第三个用例和 484 记的那段"官方用 `useRouteAnnouncer` 的 `set()` 播搜索结果"是**同一件事的两种写法**。两页都这么写，官方没有给裁决；本系列按各自页面的分工句处理：**这条场景两个 API 都演示过**，选哪个取决于你更想统一入口（组件页那句"页面/路由变化请用路由那一套"）还是保留紧急度语义。

**提交成功/失败这对写法**在组件页与函数页各出现一次，是同一个例子：

```vue [app/pages/contact.vue]
<script setup lang="ts">
const { polite, assertive } = useAnnouncer()

async function submitForm () {
  try {
    await $fetch('/api/contact', { method: 'POST', body: formData })
    polite('Message sent successfully')
  } catch (error) {
    assertive('Error: Failed to send message')
  }
}
</script>
```

成功走 `polite`、失败走 `assertive`——这两句就是"紧急度按后果分"的官方示范。

## 自查

- 页面内变化（表单校验、异步操作、toast、实时内容）归这一对；**v4.4.2+**，且**必须真的把 `<NuxtAnnouncer>` 放进 app.vue 或布局**，否则消息没落点。
- 与路由那一对的四行差别里，只有 `atomic` 的默认值不同（`false` 对 **`true`**）。
- `polite` 等屏幕阅读器空闲、`assertive` 立即打断；成功提示用前者、报错用后者是官方示范。
- `set()` 的第二参默认 `polite`，不会沿用实例的紧急度。
- `politeness` 的类型标注两页不一致（`Ref<'polite'|'assertive'|'off'>` 对 `Ref<string>`）。

## 下一篇

下一篇 486 讲两件"看起来很小但官方口径不一样"的事：**语言属性有明确写法**（`app.head.htmlAttrs.lang`，官方还把 title 默认值、语言、favicon 一起列为"该放在这里的不会变的东西"），而**图片替代文本在 4.x 文档里没有任何规则句**，只有 `getting-started/assets` 的示例里出现过 `alt` 与 `:alt`。

## 本文依据

- `<NuxtAnnouncer>` 组件页：标题句（adds a hidden element to announce dynamic content changes）、`::important` v4.4.2+、Usage 段（`app.vue` 或 `app/layouts/`、四类场景）、`app/app.vue` 里两个 announcer 并排的示例、"Then use the `useAnnouncer` composable anywhere in your app"与 contact.vue 示例、Slots 段（`#default="{ message }"`）、Props 两条（`atomic` 默认 **`true`**、`politeness` 默认 `polite`）、"Differences from `<NuxtRouteAnnouncer>`" 四行对照表原文、两条 callout（组件可选可照源码自写、可用 `useAnnouncer` 挂钩）（`api/components/nuxt-announcer`）
- `useAnnouncer` 页：定性句（与 `useRouteAnnouncer` 相反提供手动控制 what and when、四类适用场景）、`::important` v4.4.2+、参数与两个属性（含 `Ref<'polite' | 'assertive' | 'off'>`）、三个方法原文（`polite`/`assertive` 各带"不紧急可等／紧急应打断"的用途句）、Use Cases 三段（表单校验、加载状态跟随 `status`、搜索结果）、两条 callout（**必须加 `<NuxtAnnouncer>` 才会渲染到 DOM**；页面/路由变化改用路由那一套）、源文件 `composables/announcer.ts`（`api/composables/use-announcer`）
- `status` 四值（`api/composables/use-async-data`）与 `pendingWhenIdle` 语义（`guide/going-further/experimental-features`，446 依据）
