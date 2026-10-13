---
url: /posts/9b45718ed193c1ef88209c0b1520a903/
title: 483 - NuxtRouteAnnouncer 那个隐藏元素播报换页
date: 2027-03-22T09:00:00+08:00
updated: 2027-03-22T09:00:00+08:00
author: cmdragon

summary:
  摘要：`<NuxtRouteAnnouncer>` 是官方给"客户端换页没播报"这个缺口的答案：它加一个**隐藏元素**，里面放页面标题，用来把路由变化播报给辅助技术（官方定性原话），要求 **Nuxt v3.12+**，放在 `app.vue` 或 `app/layouts/` 里。它的两个属性是 `atomic`（**默认 `false`**，决定只播变化还是整块内容重读）与 `politeness`（`off`/`polite`/`assertive`，默认 `polite`）。默认插槽能拿到 `message`，所以播报文案可以自己排。官方还写明这个组件**是可选的**、要完全自定义可以照它的源码自己实现一个。它读的是 Unhead 渲出的 `<title>`——标题没区分度就等于没装它。

categories:
  - 前端开发

tags:
  - nuxt4
  - 可访问性
  - 屏幕阅读器
  - NuxtRouteAnnouncer
  - live region
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

用键盘或读屏软件点一次站内链接：地址栏变了、内容换了，但什么声音都没有。整页加载时屏幕阅读器会自己播报，客户端导航它不知道。

官方补这个缺口的组件是 `<NuxtRouteAnnouncer>`。这一篇讲它是什么、放在哪、两个属性各自改变什么。

## 定性：一个放标题的隐藏元素

组件页的标题句就是它的定义：

> The <NuxtRouteAnnouncer> component adds a hidden element with the page title to announce route changes to assistive technologies.

拆开三件事：**渲染一个隐藏元素**、**内容是页面标题**、**用途是把路由变化播报给辅助技术**。

版本要求是 `::important` 级的一条：

> This composable is available in Nuxt v3.12+.

（组件页那句用的是 "This component"。）

## 放哪里

说明句是 "to enhance accessibility by informing assistive technologies about page title changes"——**目的是让辅助技术知道页面标题变了**，从而把导航变化播报给依赖屏幕阅读器的用户。示例是把它放在最外层模板的**第一行**：

```vue [app/app.vue]
<template>
  <NuxtRouteAnnouncer />
  <NuxtLayout>
    <NuxtPage />
  </NuxtLayout>
</template>
```

无障碍那一节给的形态更短（`<NuxtRouteAnnouncer />` 直接跟在 `<NuxtPage />` 之前），两处示例的差别只是有没有包布局——**共同点是它放在渲染页面的那一层，不是放在某个页面组件里**。

## 两个属性

组件页只列了这两个，措辞几乎对称：

| 属性 | 官方说明 | 默认 |
| --- | --- | --- |
| `atomic` | Controls if screen readers only announce changes or the entire content. Set to true for full content readouts on updates, false for changes only. | **`false`** |
| `politeness` | Sets the urgency for screen reader announcements: `off` (disable the announcement), `polite` (waits for silence), or `assertive` (interrupts immediately). | `polite` |

`atomic` 的两端差别要读准：**`false` 是只播"变了什么"**，**`true` 是更新时把整块内容重读一遍**。换页场景下整块内容就是标题，所以默认 `false` 意味着"只播新标题"。

`politeness` 三档里 `off` 是**关掉播报**——这个组件没有"关掉"的开关属性，紧急度设成 `off` 就是它。

## 默认插槽：自己决定播报文案

官方给的插槽示例把 `message` 取出来重排：

```vue
<template>
  <NuxtRouteAnnouncer>
    <template #default="{ message }">
      <p>{{ message }} was loaded.</p>
    </template>
  </NuxtRouteAnnouncer>
</template>
```

说明句是"You can pass custom HTML or components through the route announcer's default slot."——**插槽里放什么都行**，播报内容就是这个插槽渲染出来的文本。上面这句 `{{ message }} was loaded.` 会把"标题 + was loaded"整句播出去。

注意这与 `atomic` 是两件事：插槽决定**说什么**，`atomic` 决定**说多少**。

## 官方说它是可选的

三条 `callout` 里的第一条：

> This component is optional. To achieve full customization, you can implement your own one based on its source code.

也就是说**不装它也能做**——它只是一个把标题写进 live region 的现成实现，官方甚至鼓励照着它的源码自己写（源码链接指向 `packages/nuxt/src/app/components/nuxt-route-announcer.ts`）。

第二条 callout 指的是另一个入口：底层实例可以用 `useRouteAnnouncer` 挂钩、**设置自定义播报消息**（484 那篇讲这个方法集）。第三条做了分工划分：**页面内的内容变化（表单校验、toast、加载态）该用 `<NuxtAnnouncer>` 配 `useAnnouncer`**，不是这个组件（485）。

## 它的质量上限在标题上

这一条来自无障碍那一节，是装完之后最容易踩空的点：

> The announcer reads the title that Unhead rendered, so it is only as useful as your titles are. If two routes share the same `<title>`, users hear nothing on the way between them.

**它读的是 Unhead 渲染出的 `<title>`**，所以两条路由标题相同时，切换过程中**什么都听不到**——装了组件也没用。这正好接上 481 那条"每条路由一个可区分标题是单件最有价值的事"，以及 488 那篇要讲的 `titleTemplate` 写法。

## 自查

- 它是隐藏元素＋页面标题＋播报路由变化，**v3.12+** 才有。
- 位置在 `app.vue` 或 `app/layouts/`，示例里放在渲染页面那一层的首行。
- `atomic` 默认 `false`（只播变化；`true` 是整块重读），`politeness` 默认 `polite`，设 `off` 即关闭播报。
- 默认插槽能拿 `message` 自定义文案，与 `atomic` 各管一层。
- 组件是可选的，可照源码自写；页面内变化要用 `<NuxtAnnouncer>`＋`useAnnouncer`。
- 装完先确认每条路由的 `<title>` 可区分，否则听不到任何东西。

## 下一篇

下一篇 484 讲它的底层实例：`useRouteAnnouncer()` 返回 `message`、`politeness` 两个 ref 与 `set`/`polite`/`assertive` 三个方法，参数只有一个 `politeness`（默认 `polite`）；它靠挂进 Unhead 的 `dom:rendered` 钩子读标题并写进播报消息——所以手动 set 的内容与自动读到的标题走的是同一条通道。

## 本文依据

- `<NuxtRouteAnnouncer>` 组件页：标题句（adds a hidden element with the page title to announce route changes）、`::important` v3.12+、Usage 段（放 `app.vue` 或 `app/layouts/`、informing assistive technologies about page title changes、`app/app.vue` 示例）、Slots 段与 `#default="{ message }"` 示例、Props 两条原文（`atomic` 默认 `false`、`politeness` 三档默认 `polite`）、三条 callout（组件可选并可照源码自写、可用 `useRouteAnnouncer` 挂钩、页面内变化改用 `<NuxtAnnouncer>`＋`useAnnouncer`）、源文件路径（`api/components/nuxt-route-announcer`）
- 无障碍页 Route Announcements 一节：定位句（渲染隐藏 live region、每次导航后写入新页面标题）、`<NuxtRouteAnnouncer />` 在 `<NuxtPage />` 之前的示例、"reads the title that Unhead rendered ... If two routes share the same `<title>`, users hear nothing"（`guide/best-practices/accessibility`）
