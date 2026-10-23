---
url: /posts/6bebdae972c3a0ece72e5a105a3a0f4e/
title: 099 - NuxtTime：服务端渲染安全的时间显示与那个保留 prop
date: 2026-11-14T09:00:00+08:00
updated: 2026-11-14T09:00:00+08:00
author: cmdragon

summary:
  摘要：`<NuxtTime>` 的官方定位是"以与语言环境友好的格式显示时间，并保证服务端与客户端一致"，具体说就是：它让你以**对语言环境友好的格式**显示日期和时间，并带**正确的 `<time>` HTML 语义**，**确保服务端与客户端之间一致渲染、没有水合不匹配**；这一组件在 **Nuxt v3.17+** 才可用。它的必填项只有一个：`datetime`（类型 `Date | number | string`、Required `true`），可以是 Date 对象、时间戳数字或 ISO 格式的日期字符串，官方三种写法分别是 `:datetime="Date.now()"`、`:datetime="new Date()"`、`datetime="2023-06-15T09:30:00.000Z"`。可选的 `locale` 类型 `string`、默认"使用浏览器或服务端的默认语言环境"，官方要求的是 **BCP 47 语言标签**（例如 `'en-US'`、`'fr-FR'`、`'ja-JP'`）。格式化不需要写函数——**组件接受 `Intl.DateTimeFormat` 选项里的任何属性**，直接摊平成 prop：`year="numeric" month="long" day="numeric" hour="2-digit" minute="2-digit"` 官方说这会输出类似 "April 22, 2025, 08:30 AM"；注意官方模板示例里多词选项一律写成 kebab-case（`time-zone-name="short"`）。另有 `relative`（类型 `boolean`、默认 `false`）启用**相对时间**格式化，走 `Intl.RelativeTimeFormat` API，此时也能传入该 API 的属性，但这里有一条官方警告：**因为 `style` 是保留 prop，所以改用 `relativeStyle`**；配合 `numeric="auto"` 时官方给的预期输出是 "3 days ago" 或 "last Friday" 这类表达，取决于 `numeric` 的设置。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxttime
  - 水合
  - 国际化
  - 时间格式

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


"上次更新时间：3 天前"这种文本，在水合阶段是最容易出问题的一类：服务端和浏览器算出的结果可能不一样。`<NuxtTime>` 就是官方给的标准解法，这一篇把它的所有 prop 与两个坑过一遍。

## 官方要解决的是什么

| 官方原话 | 拆出来的承诺 |
| --- | --- |
| "`<NuxtTime>` 组件以**对语言环境友好的格式**显示时间，并保证**服务端与客户端一致**。" | 本地化 + 两侧一致 |
| "它让你以对语言环境友好的格式显示日期与时间，并带有**正确的 `<time>` HTML 语义**。它**确保服务端与客户端之间一致渲染，没有水合不匹配**。" | 语义标签 + 无水合不匹配 |
| （important）"这个组件在 **Nuxt v3.17+** 中可用。" | 版本门槛 |

三条里最实用的是最后半句：**没有水合不匹配**。这决定了它存在的理由——不是省一段 `Intl` 调用，而是让"时间"这种随环境变化的内容两侧算出同一个结果。

## `datetime`：唯一必填项，三种形态

- 类型：`Date | number | string`
- Required：`true`

官方列出可以提供的三种值：一个 `Date` 对象、一个时间戳（数字）、一个 ISO 格式的日期字符串。对应的官方写法：

```vue [app/app.vue]
<template>
  <NuxtTime :datetime="Date.now()" />
  <NuxtTime :datetime="new Date()" />
  <NuxtTime datetime="2023-06-15T09:30:00.000Z" />
</template>
```

注意第三个**没有冒号**——写成字符串字面量就行，因为它本来就是字符串。前两个是表达式，必须 `:`。

## `locale`：BCP 47 标签

- 类型：`string`；Required：`false`
- 默认：**使用浏览器或服务端的默认语言环境**

官方要求的是 BCP 47 语言标签，举例 `'en-US'`、`'fr-FR'`、`'ja-JP'`：

```vue [app/app.vue]
<template>
  <NuxtTime
    :datetime="Date.now()"
    locale="fr-FR"
  />
</template>
```

这一格默认值的措辞值得警觉——它默认的是**两侧各自的默认语言环境**。服务端和你机器的浏览器不一定同源，所以官方在"带自定义语言环境"那个示例里给三个组件分别显式写了 `locale="en-US"`、`locale="fr-FR"`、`locale="ja-JP"` 来对比同一天的 `weekday="long"` 结果。要控制显示语言就显式写，别把语言交给环境默认值。

## 格式化：把 `Intl.DateTimeFormat` 的选项摊平成 prop

官方说明：**该组件接受 `Intl.DateTimeFormat` 选项中的任何属性。**

```vue [app/app.vue]
<template>
  <NuxtTime
    :datetime="Date.now()"
    year="numeric"
    month="long"
    day="numeric"
    hour="2-digit"
    minute="2-digit"
  />
</template>
```

官方对这一段给的预期输出是："这会输出类似 'April 22, 2025, 08:30 AM' 的内容。"

更完整的一版把星期和时区也加上：

```vue [app/app.vue]
<template>
  <NuxtTime
    :datetime="Date.now()"
    weekday="long"
    year="numeric"
    month="short"
    day="numeric"
    hour="numeric"
    minute="numeric"
    second="numeric"
    time-zone-name="short"
  />
</template>
```

**这里藏着模板写法的一条规则：多词的选项在模板里写成 kebab-case。** 官方示例写的是 `time-zone-name="short"`，对应的 JS 选项名是 `timeZoneName`。这不是 Nuxt 的特例而是 Vue 的模板属性规范，但在这一组件上特别容易忘，因为其他 prop（`locale`、`relative`）都是单词。

## `relative`：相对时间走另一个 API

- 类型：`boolean`；Required：`false`；默认：`false`

官方说明：**启用使用 `Intl.RelativeTimeFormat` API 的相对时间格式化。**

```vue [app/app.vue]
<template>
  <!-- Shows something like "5 minutes ago" -->
  <NuxtTime
    :datetime="Date.now() - 5 * 60 * 1000"
    relative
  />
</template>
```

官方"相对时间"那组示例把方向一次说清：往前推 30 秒得到 `30 seconds ago`、往前 45 分钟得到 `45 minutes ago`、**往后**两天（`Date.now() + 2 * 24 * 60 * 60 * 1000`）得到 `in 2 days`。也就是说 `datetime` 是当前时刻之前的值就是"……前"，是之后的值就是"……后"。

### 那条必须单独记的警告

`relative` 为 `true` 时组件同样接受 `Intl.RelativeTimeFormat` 的属性——**但其中一个名字被换掉了。官方 warning：因为 `style` 是一个保留 prop，这里改用 `relativeStyle`。**

```vue [app/app.vue]
<template>
  <NuxtTime
    :datetime="Date.now() - 3 * 24 * 60 * 60 * 1000"
    relative
    numeric="auto"
    relative-style="long"
  />
</template>
```

官方给的预期输出："这会输出类似 '3 days ago' 或 'last Friday' 的内容，取决于 `numeric` 的设置。"

**`numeric="auto"` 就是那个差别**：它允许给出更口语化的近似表达（"last Friday"），而不是纯数字式的 "3 days ago"。而这一格在模板里又写作 kebab-case：`relative-style`。

## 全部 prop 收在一处

| prop | 类型 | 必需 | 默认 | 说明 |
| --- | --- | --- | --- | --- |
| `datetime` | `Date \| number \| string` | **是** | —— | Date 对象 / 时间戳 / ISO 字符串 |
| `locale` | `string` | 否 | 浏览器或服务端的默认语言环境 | BCP 47 语言标签 |
| `relative` | `boolean` | 否 | `false` | 启用 `Intl.RelativeTimeFormat` |
| `Intl.DateTimeFormat` 的任意选项 | —— | 否 | —— | 直接摊平成 prop；模板里写 kebab-case |
| `Intl.RelativeTimeFormat` 的任意选项 | —— | 否 | —— | 同上，**但 `style` 要写成 `relativeStyle`** |

## 自查

1. `<NuxtTime>` 从哪个版本可用？官方用它要解决的两件事分别是什么？
2. `datetime` 可以接受哪三种形态？哪种写法不需要加 `:`？
3. 不写 `locale` 时默认值是什么？官方为什么在示例里显式写三种语言？
4. 想在模板里传 `timeZoneName` 该怎么写？`Intl.RelativeTimeFormat` 的 `style` 呢？
5. `relative` 打开后走哪个 API？`numeric="auto"` 会带来什么差别？

## 下一篇

100 讲 `<NuxtClientFallback>` 与 `useHydration`：前者是实验特性、必须先开 `experimental.clientFallback`，服务端渲染某块出错时把那一块交给客户端；后者把状态从服务端带到客户端，且 `get` 只在服务端跑、`set` 只在客户端跑。
