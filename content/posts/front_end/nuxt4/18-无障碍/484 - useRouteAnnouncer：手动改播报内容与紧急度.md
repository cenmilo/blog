---
url: /posts/6c09574c5f40521003d35cf093ebd1a8/
title: 484 - useRouteAnnouncer：手动改播报内容与紧急度
date: 2027-03-22T10:00:00+08:00
updated: 2027-03-22T10:00:00+08:00
author: cmdragon

summary:
  摘要：`useRouteAnnouncer()` 是 `<NuxtRouteAnnouncer>` 背后的那个实例——官方原话是"A composable which observes the page title changes and updates the announcer message accordingly. Used by `<NuxtRouteAnnouncer>` and controllable"，实现方式写得很具体：**挂进 Unhead 的 `dom:rendered` 钩子读页面标题，并把它设为播报消息**。它返回两个属性（`message`、`politeness`，都是 `Ref`）与三个方法（`set(message, politeness = "polite")`、`polite(message)`、`assertive(message)`），唯一入参是 `politeness`（`off`/`polite`/`assertive`，默认 `polite`）。版本要求 v3.12+。要留意两节之间的口径差：这个组件页说"页面内的动态变化请用 `useAnnouncer`"，而无障碍那一节的示例恰恰是用 `useRouteAnnouncer` 的 `set()` 播报搜索结果数量。

categories:
  - 前端开发

tags:
  - nuxt4
  - 可访问性
  - useRouteAnnouncer
  - 屏幕阅读器
  - Unhead
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

搜索页的结果数量每次变，标题不变。装了 `<NuxtRouteAnnouncer>` 也听不到"找到 12 条结果"——因为它播的是标题。

要把这句话播出去，用的就是这个组合式函数：`useRouteAnnouncer`。

## 它和组件是同一个实例

官方描述段两句连着读：

> A composable which observes the page title changes and updates the announcer message accordingly. Used by `<NuxtRouteAnnouncer>` and controllable. It hooks into Unhead's `dom:rendered` hook to read the page's title and set it as the announcer message.

三个信息：

- **它观察标题变化并更新"播报消息"**——这个 message 就是组件渲染的内容。
- **"Used by `<NuxtRouteAnnouncer>`"**：组件用的就是它，所以两者不是两套机制，是同一实例的两个面。
- **通道写明了是 Unhead 的 `dom:rendered` 钩子**：标题在 `<head>` 里渲染完成后，它读那个标题、写进播报消息。

`dom:rendered` 这个位置解释了 483 末尾那条限制：**它读的是 Unhead 渲出的 `<title>`**，标题没变化就没有播报。

## 返回的东西：两个属性三个方法

版本要求一条 `::important`：available in Nuxt **v3.12+**。

参数只有一个：

- `politeness`：`off`（关闭播报）、`polite`（等待安静）、`assertive`（立即打断），默认 `polite`。

属性两项：

| 属性 | 类型 | 说明 |
| --- | --- | --- |
| `message` | `Ref<string>` | The message to announce |
| `politeness` | `Ref<string>` | 紧急度级别 `off`/`polite`/`assertive` |

方法三项，官方原文逐条：

| 方法 | 官方说明 |
| --- | --- |
| `set(message, politeness = "polite")` | Sets the message to announce with its urgency level. |
| `polite(message)` | Sets the message with `politeness = "polite"` |
| `assertive(message)` | Sets the message with `politeness = "assertive"` |

两点读法：**`set` 的第二个参数有默认值 `polite`**，所以不传紧急度时它不会继承实例当前的 `politeness`；`polite()` 与 `assertive()` 就是这两个取值的快捷方式。

官方给的解构示例，同时演示了"建实例时改默认紧急度"：

```vue [app/pages/index.vue]
<script setup lang="ts">
const { message, politeness, set, polite, assertive } = useRouteAnnouncer({
  politeness: 'assertive',
})
</script>
```

## 官方自己用它播非标题内容

无障碍那一节的搜索示例，用的正是 `set()`：

```vue [app/pages/search.vue]
<script setup lang="ts">
const { set } = useRouteAnnouncer()
const { data: results } = await useFetch('/api/search')

watch(results, (results) => {
  set(`${results?.length ?? 0} results found`)
})
</script>
```

这段是官方原文（放在 Route Announcements 一节里），说明**播报内容不必真是标题**——`set()` 写什么就播什么。

但这里有一个两节之间的口径差要说明白：

| 出处 | 说法 |
| --- | --- |
| `useRouteAnnouncer` 页末尾的 `callout` | "For announcing **dynamic in-page content changes** (form validation, toasts, loading states), use `useAnnouncer` instead." |
| 无障碍页的 Route Announcements 一节 | 用 `useRouteAnnouncer()` 的 `set()` 播报搜索结果数量，并把这写成"When you need to announce something else, or to change how urgently it is announced" |

**搜索结果变化算不算 in-page content change？两节没给出裁决**（文档未说明）。可执行的区分方式是那句官方措辞：**要换紧急度或播"别的东西"用这个**，**做表单校验、toast、加载状态这类页面内反馈用 `useAnnouncer`**（485）——后者还有官方给的三个成型示例正好对着这三类场景。

## 一个没写的点

如果**只调 `useRouteAnnouncer()` 而没有在模板里放 `<NuxtRouteAnnouncer>`**，`set()` 写进去的消息有没有渲染处、会不会被播报——**两节都没说**。文档未记载这一点，但组件页那句"Used by `<NuxtRouteAnnouncer>`"暗示渲染责任在组件那边。

要确认，做法很直接：只调 `set()` 不装组件，用读屏软件（或系统自带的朗读功能）听一次；再对照 DOM 里有没有那个隐藏元素。**别只看 `message.value` 变了就认为播出去了**——变量变了和有没有 live region 承载是两件事。

## 自查

- 它是 `<NuxtRouteAnnouncer>` 用的那一个实例，靠 Unhead 的 `dom:rendered` 读标题写入 `message`；v3.12+。
- 返回 `message`/`politeness` 两个 ref 与 `set`/`polite`/`assertive` 三个方法；`set` 的第二参默认 `polite`。
- `set()` 能播任意文本，官方自己的搜索示例就这么用；但它与 `useAnnouncer` 的分工在两节间表述不一致，页面内反馈按官方 callout 走 `useAnnouncer`。
- 不装组件时 `set()` 有没有效果，文档未记载，要用读屏与 DOM 双重确认。
- 标题不可区分时（483 那条）这个函数就是唯一的补救入口——但那是治标，标题还是要改。

## 下一篇

下一篇 485 讲 v4.4.2+ 新那一对：`useAnnouncer` 与 `<NuxtAnnouncer>`。官方给的差别表只有四行，其中"atomic 默认值"这一行是两个组件唯一数值不同的地方——路由播报那个默认 `false`，这个默认 `true`。

## 本文依据

- `useRouteAnnouncer` 页：定性句（observes the page title changes and updates the announcer message accordingly、"Used by `<NuxtRouteAnnouncer>` and controllable"、hooks into Unhead's `dom:rendered` 读标题并设为消息）、`::important` v3.12+、参数 `politeness` 三档与默认、Properties 两项类型与说明、Methods 三条原文（`set` 的第二参默认值）、解构示例（`politeness: 'assertive'`）、callout（页面内动态变化改用 `useAnnouncer`）、指向无障碍页 Route Announcements 的 read-more、源文件 `composables/route-announcer.ts`（`api/composables/use-route-announcer`）
- 无障碍页：`const { set } = useRouteAnnouncer()` 配 `watch(results)` 播报"N results found"的完整示例与"When you need to announce something else, or to change how urgently it is announced, use `useRouteAnnouncer`"句、"announcer reads the title that Unhead rendered"（`guide/best-practices/accessibility`）
