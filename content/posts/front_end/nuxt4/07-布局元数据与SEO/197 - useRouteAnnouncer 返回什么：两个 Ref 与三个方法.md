---
url: /posts/f67639122f42898f6c8cf6f6a2d179d9/
title: 197 - useRouteAnnouncer 返回什么：两个 Ref 与三个方法
date: 2026-12-16T11:00:00+08:00
updated: 2026-12-16T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：前面 101 讲过 `<NuxtRouteAnnouncer>` 这个组件本身，144 讲过它和 `<NuxtAnnouncer>` 的区别，这一篇只看 `useRouteAnnouncer` 这个 composable 交出来的东西。文档把它的返回面拆成两小节：**Properties** 两个——`message`（`Ref<string>`，"The message to announce"）和 `politeness`（`Ref<string>`，"Screen reader announcement urgency level `off`, `polite`, or `assertive`"）；**Methods** 三个——`set(message, politeness = "polite")`、`polite(message)`、`assertive(message)`，后两个的作用各自写成 "Sets the message with `politeness = \"polite\"`" 和 "...= \"assertive\""，也就是**两个便捷方法只是 `set` 的固定第二参**。官方示例是一次完整解构：`const { message, politeness, set, polite, assertive } = useRouteAnnouncer({ politeness: 'assertive' })`——**六个成员全取，说明参数就是一个选项对象**。最让我混乱的是 `politeness` 这个词同时出现在三个地方：组件的 prop（默认 `polite`）、组合式函数的选项（默认 `polite`）、以及返回的那个 Ref；文档没写三者的优先级，所以这一篇把能确认的列成表，把没写的留给实测。

categories:
  - 前端开发

tags:
  - nuxt4
  - 无障碍
  - useRouteAnnouncer
  - 屏幕阅读器
  - live region

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


加完 `<NuxtRouteAnnouncer />` 想改播报内容，很自然会去调 `useRouteAnnouncer()`，然后卡在第一个问题上：它到底返回什么？官方那一节把返回面写得很规整，下面照它的路子列一遍。

## 它自己是怎么描述的

`useRouteAnnouncer` 页的 Description 一节整段：

> A composable which observes the page title changes and updates the announcer message accordingly. Used by `<NuxtRouteAnnouncer>` and controllable.
> It hooks into Unhead's `dom:rendered` hook to read the page's title and set it as the announcer message.

那一节开头的定位句是 "This composable observes the page title changes and updates the announcer message accordingly."，两处都在强调**观察的是标题**。第二句 "and controllable" 是这一节存在的理由：组件是被动消费方，组合式函数才是控制口。

版本要求是一条 important：**"This composable is available in Nuxt v3.12+."**

## Parameters 一个，Properties 两个，Methods 三个

| 分类 | 成员 | 文档给的信息 |
| --- | --- | --- |
| 参数 | `politeness` | "Sets the urgency for screen reader announcements: `off` (disable the announcement), `polite` (waits for silence), or `assertive` (interrupts immediately). (default `polite`)." |
| 属性 | `message` | type `Ref<string>`，description "The message to announce" |
| 属性 | `politeness` | type `Ref<string>`，description "Screen reader announcement urgency level `off`, `polite`, or `assertive`" |
| 方法 | `set(message, politeness = "polite")` | "Sets the message to announce with its urgency level." |
| 方法 | `polite(message)` | "Sets the message with `politeness = \"polite\"`" |
| 方法 | `assertive(message)` | "Sets the message with `politeness = \"assertive\"`" |

两个细节我是从这个表里才看明白的：

1. **`politeness` 是可选的，而且默认值写在方法签名里**（`politeness = "polite"`），不在说明句里。这意味着 `set(msg)` 和 `polite(msg)` 在文档语义上是一回事——但反过来不成立：`polite()`/`assertive()` 没有第二参位置。
2. **返回的两个成员是 `Ref<string>`，不是普通 string**。`message` 是个 ref，意味着我可以只读不写：组件消费的就是这个 ref。文档没有说写 `message.value` 会发生什么（会不会被下一次标题变化覆盖），我不猜，交给实测。

官方示例只有一块，但信息都在：

```vue [app/pages/index.vue]
<script setup lang="ts">
const { message, politeness, set, polite, assertive } = useRouteAnnouncer({
  politeness: 'assertive',
})
</script>
```

一次解构六个成员、参数是一个带 `politeness` 键的对象——这两点都是从这五行里读出来的，文档没有单独用文字写。注意这个示例**什么都没做**：没有调用 `set`，也没有渲染 `message`。它只是"拿到入口"的样子，所以我不会把它当成推荐用法。

## politeness 在三个地方出现，优先级文档没写

| 出现处 | 出处 | 默认值 |
| --- | --- | --- |
| 组件 prop | `<NuxtRouteAnnouncer>` 的 Props 一节 | `polite` |
| 组合式函数选项 | `useRouteAnnouncer` 的 Parameters | `polite` |
| 组合式函数返回值 | 返回的 `politeness` Ref | 未写初值来源 |

我原先以为这三处是同一个东西的三个别名，但文档从没把它们的关系写成规则。所以合理的做法是：把 `useRouteAnnouncer` 的选项参数**当作调用方自己那一次控制的默认**，把组件 prop 当作渲染出来的 live region 的属性。哪个赢，只能试——最直接的判据是 `off`：如果 `useRouteAnnouncer({ politeness: 'off' })` 之后屏幕上仍有内容被读出来，说明 prop 侧生效；完全安静则说明选项侧赢了。测这个需要真的开屏幕阅读器（或者用可访问性检查工具读 `aria-live` 元素的属性），不是看代码能看出来的。

## 组件侧留的三条出口

`<NuxtRouteAnnouncer>` 页末尾有三个 callout，正好把"什么时候不需要用这个 composable"说清了：

> This component is optional.
> To achieve full customization, you can implement your own one based on its source code.
> You can hook into the underlying announcer instance using the `useRouteAnnouncer` composable, which allows you to set a custom announcement message.

第三条是这一节与那一节之间的正式桥梁：**"hook into the underlying announcer instance"** ——它证实组件和 composable 用的是同一个实例，而不是各建一套。第一条 "This component is optional." 我也抄了下来，因为很多人以为不加组件就不能播报；组件只是那个隐藏 live region 的载体。

还有一条要记住的分工（组件页最后一个 callout）：

> For announcing in-page content changes (form validation, toast notifications, loading states, etc.), use the `<NuxtAnnouncer>` component with the `useAnnouncer` composable instead.

这是 144 讲过的两族播报器，路由级用这一族，页内变化用另一族，别混。

## 自查

1. `useRouteAnnouncer()` 返回几个属性和几个方法？（答：两个属性 `message`、`politeness`，都是 `Ref<string>`；三个方法 `set`、`polite`、`assertive`。）
2. `polite(message)` 和 `set(message)` 有什么区别？（答：按文档说明前者就是"Sets the message with `politeness = \"polite\"`"，后者第二参可传、默认也是 `polite`，语义相同。）
3. 播报消息的默认内容从哪来？（答：从页面标题来——它挂的是 Unhead 的 `dom:rendered` 钩子。）

## 下一篇

198 - ClientOnly 里的组件不跑服务端：首屏 HTML 与水合期的两条推论

## 本文依据

- `api/composables/use-route-announcer`：页首 "This composable observes the page title changes and updates the announcer message accordingly."；important "This composable is available in Nuxt v3.12+."；Description 两句（"Used by `<NuxtRouteAnnouncer>` and controllable."、"It hooks into Unhead's `dom:rendered` hook to read the page's title and set it as the announcer message."）；Parameters 的 `politeness` 说明；Properties 两项的类型与描述；Methods 三项的签名与描述；示例的六成员解构；末尾 callout "For announcing dynamic in-page content changes (form validation, toasts, loading states), use `useAnnouncer` instead."
- `api/components/nuxt-route-announcer`：Props 两条（`atomic` 默认 `false`、`politeness` 默认 `polite`）；三个 callout "This component is optional."、"To achieve full customization, you can implement your own one based on its source code."、"You can hook into the underlying announcer instance using the `useRouteAnnouncer` composable, which allows you to set a custom announcement message."
