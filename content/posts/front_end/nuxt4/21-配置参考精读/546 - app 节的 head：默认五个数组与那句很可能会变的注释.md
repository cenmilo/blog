---
url: /posts/d7a3af622ec7dec3c0510457394c587b/
title: 546 - app 节的 head：默认五个数组与那句很可能会变的注释
date: 2027-04-12T09:00:00+08:00
updated: 2027-04-12T09:00:00+08:00
author: cmdragon

summary:
  摘要：`app.head` 的定位是**"Set default configuration for `<head>` on every page."**（为每个页面设 `<head>` 的默认配置），类型 `object`。官方默认值里 `meta` 已经有两条——**viewport 与 charset**，另外 `link`、`style`、`script`、`noscript` 四个数组默认都是空的。SEO 那一节补了两件关键事：**`app.head` 是"静态地"定制整个应用的 head**，并且用 `::important` 明写 **"This method does not allow you to provide reactive data. We recommend using `useHead()` in `app.vue`."**；同一页还把这三处（`useHead`、`app.head`、组件）用的类型统称为 **non-reactive types**。官方示例里 `style` 那一栏上方还留着一句注释：**"please note that this is an area that is likely to change"**。

categories:
  - 前端开发

tags:
  - nuxt4
  - head
  - seo
  - useHead
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`viewport` 那一条 meta 你的项目里没写过，但源码里就是有。

它是 `app.head` 的默认值给的——这一节同时是"整站 head 的默认位置"和一条明确的能力边界。

## 默认值：五个数组，两条已填

`### head` 的说明一句：**"Set default configuration for `<head>` on every page."**

- **类型**：`object`
- **默认**（官方 JSON 原样）：

```json
{
  "meta": [
    {
      "name": "viewport",
      "content": "width=device-width, initial-scale=1"
    },
    {
      "charset": "utf-8"
    }
  ],
  "link": [],
  "style": [],
  "script": [],
  "noscript": []
}
```

读这份默认值有三件事：

**`meta` 里两条已经填好。** viewport 那条的 content 就是 `width=device-width, initial-scale=1`，charset 是 `utf-8`。**这意味着不写 `app.head` 时你的页面也已经带这两条**，重复再写一遍只会多一份。

**四个空数组是"可写清单"。** `link`、`style`、`script`、`noscript` 默认是空的，但结构已经给定——这一节能放哪几类东西，就按这五个键来。

**没有 `title`。** 默认 JSON 里没有标题相关的键；站点标题与页面标题走的是另一套（SEO 那一节的 `titleTemplate` 与各页的 `title`），不在这份默认值里。

## 官方示例与那句"很可能会变"

示例给的是五个键各填一项：

```ts
export default defineNuxtConfig({
  app: {
    head: {
      meta: [
      // <meta name="viewport" content="width=device-width, initial-scale=1">
        { name: 'viewport', content: 'width=device-width, initial-scale=1' },
      ],
      script: [
      // <script src="https://myawesome-lib.js"></script>
        { src: 'https://awesome-lib.js' },
      ],
      link: [
      // <link rel="stylesheet" href="https://myawesome-lib.css">
        { rel: 'stylesheet', href: 'https://awesome-lib.css' },
      ],
      // please note that this is an area that is likely to change
      style: [
      // <style>:root { color: red }</style>
        { textContent: ':root { color: red }' },
      ],
      noscript: [
      // <noscript>JavaScript is required</noscript>
        { textContent: 'JavaScript is required' },
      ],
    },
  },
})
```

注释里那五种渲染结果把"对象字段 → HTML"的对应写清了：`src` 出外链脚本、`rel`+`href` 出样式表链接、**`textContent` 出内联内容**（`style` 与 `noscript` 两处都是这个键）。另要提醒一句：官方那段外链脚本示例只给了 `src`，**没有完整性校验属性**——真在生产引入第三方脚本，得自己加 SRI 与 `crossorigin`，或者把审过的文件自托管，否则那份文件被换掉就会在你的站点上执行。

而那行 **"please note that this is an area that is likely to change"** 就写在 `style` 数组上方——**官方自己标注这一块很可能变化**。把它当作风险提示记下来：`style` 这类内联写法的形状不稳，别在多处依赖它。

## 边界：这一节不能放响应式数据

SEO 那一节先给定位：**"Providing an `app.head` property in your `nuxt.config.ts` allows you to statically customize the head for your entire app."**——注意用词是 **statically**。

紧接的 `::important` 把限制写明：

> This method does not allow you to provide reactive data. We recommend using `useHead()` in `app.vue`.

同页 Types 一节还有第三处印证：**"Below are the non-reactive types used for `useHead`, `app.head` and components."**

三句合起来是一条判断：**`app.head` 与 `useHead` 用的是同一套"非响应式"类型，但只有 `useHead` 能在组件里跟着数据变。** 所以：

- 整站固定不变的 head（favicon、默认 meta、第三方脚本外链）→ `app.head` 够用。
- 随路由、语言、用户状态变的东西 → 必须在 `app.vue` 或页面里用 `useHead()`。官方在无障碍那节给的例子正是语言切换：`htmlAttrs.lang` 要跟着变就得走 `useHead`，因为 `app.head` 里放不了响应式值。

## 一处迁移痕迹

从 Nuxt 2 迁过来的配置要改名：bridge 那一节写的动作是 **"In your `nuxt.config`, rename `head` to `app.head`."**，并当场附了一条：**"Note that objects no longer have a `hid` key for deduplication."**

也就是说：**旧配置里那个用于去重的 `hid` 键在 4.x 已经不是 head 对象的成员了**——迁移时如果还留着 `hid`，它不再起去重作用。这一条在 `api/nuxt-config` 的 `head` 小节里没有写，只在迁移文档里。

## 自查

- `app.head` 是 `object`，默认值里 `meta` 已含 viewport 与 charset 两条，`link`/`style`/`script`/`noscript` 四个数组为空；**默认值不含 title**。
- 官方示例用 `src`/`rel`+`href`/`textContent` 三种形态对应外链脚本、样式表链接与内联内容，并在 `style` 上方标注"这块很可能会变"。
- **这一节不能放响应式数据**（`::important` 原句），官方推荐把需要响应的位置改到 `app.vue` 用 `useHead()`；两者共用同一套 non-reactive 类型。
- 固定的整站 head 放 `app.head`，随路由/语言/状态变化的放 `useHead`。
- Nuxt 2 的 `head` 要改名成 `app.head`，且 **`hid` 键已不存在**（不再用于去重）。

## 下一篇

下一篇 547 读 `app` 节里的四个开关：`pageTransition`、`layoutTransition`、`viewTransition`、`keepalive`——官方对四个都写了同一句"可被 `definePageMeta` 覆盖"，其中三个还要求"只能放 JSON 可序列化的值"。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`### head` 整节：定义句、Type `object`、默认 JSON 五键（`meta` 两条含 viewport/charset）、示例全文含 `src`/`rel`/`href`/`textContent` 与"likely to change"注释）
- https://nuxt.com/docs/4.x/getting-started/seo-meta（statically customize 的定位句、"does not allow you to provide reactive data. We recommend using `useHead()` in `app.vue`" 的 `::important`、non-reactive types 一句）
- https://nuxt.com/docs/4.x/bridge/meta（rename `head` 为 `app.head` 与 objects no longer have a `hid` key 两句）
- https://nuxt.com/docs/4.x/guide/best-practices/accessibility（语言切换要走 `useHead` 而非 `app.head` 的用法对照）
