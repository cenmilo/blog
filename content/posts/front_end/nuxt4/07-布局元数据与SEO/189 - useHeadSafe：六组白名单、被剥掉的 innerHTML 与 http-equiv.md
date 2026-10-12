---
url: /posts/8ad4fdb5ba9f608aa919453ce5142bb6/
title: useHeadSafe：六组白名单、被剥掉的 innerHTML 与 http-equiv
date: 2026-12-14T09:00:00+08:00
updated: 2026-12-14T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：官方给这一页的定位句是"**The recommended way to provide head data with user input.**"，机制句是："The `useHeadSafe` composable is **a wrapper around the `useHead` composable that restricts the input to only allow safe values**. This is the recommended way to manage head data when working with user input, as it **prevents XSS attacks by sanitizing potentially dangerous attributes**."。被剥掉什么，官方点名列在 warning 里："> When using `useHeadSafe`, potentially dangerous attributes like **`innerHTML` in scripts** or **`http-equiv` in meta tags** are automatically stripped out to prevent XSS attacks. **Use this composable whenever you're working with user-generated content.**"。**放行范围是一张写死的白名单**（"The following attributes are whitelisted for each head element type"）：`htmlAttrs` 只允许 `['class','style','lang','dir']`、`bodyAttrs` 只允许 `['class','style']`、`meta` 只允许 `['name','property','charset','content','media']`、`noscript` 只有 `['textContent']`、`style` 是 `['media','textContent','nonce','title','blocking']`、`script` 是 `['type','textContent','nonce','blocking']`、`link` 一组最长：`['color','crossorigin','fetchpriority','href','hreflang','imagesrcset','imagesizes','integrity','media','referrerpolicy','rel','sizes','type']`。类型是 `export function useHeadSafe (input: MaybeComputedRef<HeadSafe>): void`，参数说明那句最关键："> **You can pass all the same values as `useHead`, but only safe attributes will be rendered.**"——**不在名单里的属性是被静默丢掉，不是报错**。

categories:
  - 前端开发

tags:
  - nuxt4
  - useHeadSafe
  - XSS
  - 白名单
  - 安全

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


把 CMS 里编辑的 `meta.description` 直接塞进 `useHead` 这件事，我在换到 `useHeadSafe` 之后才发现少了什么：那条描述里作者写的 `<strong>` 标签没了。这不是 bug——`useHeadSafe` 的行为是**按白名单放行**，而白名单是写死的。这一篇把它与 `useHead` 的差别列清楚。

## 它是什么：useHead 的受限包装

官方两句定义：

> The recommended way to provide head data with user input.

> The `useHeadSafe` composable is a wrapper around the `useHead` composable that **restricts the input to only allow safe values**. This is the recommended way to manage head data when working with user input, as it prevents XSS attacks by sanitizing potentially dangerous attributes.

**"wrapper around `useHead`" 这句很重要**：它不是另一套 head 系统，入参形状、反应式支持都沿用 `useHead`（类型 `MaybeComputedRef<HeadSafe>`，`ref`/`computed`/函数照旧）。差别只在**过滤**。

## 被点名剥掉的两种东西

官方 warning 里给的是具体属性名，不是泛泛的"危险内容"：

> When using `useHeadSafe`, potentially dangerous attributes like **`innerHTML` in scripts** or **`http-equiv` in meta tags** are automatically stripped out to prevent XSS attacks. Use this composable whenever you're working with user-generated content.

为什么是这两个，白名单能解释：

- `script` 的白名单里没有 `innerHTML`，只有 `textContent` → **内联脚本内容这条路被彻底封掉**。
- `meta` 的白名单里没有 `http-equiv` → 不能通过 `<meta http-equiv="refresh">` 之类做跳转或指令。

## 六组白名单原文

官方原话："The following attributes are whitelisted for each head element type:"

```ts
const WhitelistAttributes = {
  htmlAttrs: ['class', 'style', 'lang', 'dir'],
  bodyAttrs: ['class', 'style'],
  meta: ['name', 'property', 'charset', 'content', 'media'],
  noscript: ['textContent'],
  style: ['media', 'textContent', 'nonce', 'title', 'blocking'],
  script: ['type', 'textContent', 'nonce', 'blocking'],
  link: ['color', 'crossorigin', 'fetchpriority', 'href', 'hreflang', 'imagesrcset', 'imagesizes', 'integrity', 'media', 'referrerpolicy', 'rel', 'sizes', 'type'],
}
```

排成表，顺便标出"少掉的那个字段"意味着什么：

| 元素 | 放行属性 | 少掉什么、后果 |
| --- | --- | --- |
| `htmlAttrs` | `class`、`style`、`lang`、`dir` | 没有 `data-*`；语言与书写方向可设 |
| `bodyAttrs` | `class`、`style` | **连 `lang`/`dir` 都没有**（那两个只在 `htmlAttrs`） |
| `meta` | `name`、`property`、`charset`、`content`、`media` | 没有 `http-equiv` |
| `noscript` | `textContent` | 只能纯文本，不能嵌套标记 |
| `style` | `media`、`textContent`、`nonce`、`title`、`blocking` | 内容走 `textContent`：CSS 文本会被当**文本**处理 |
| `script` | `type`、`textContent`、`nonce`、`blocking` | 没有 `src`！也**没有** `innerHTML` |
| `link` | 13 项（`href`、`rel`、`integrity`、`crossorigin`、`referrerpolicy`…） | 属性最多的一组，外链样式表基本够用 |

**`script` 那一行我要标一个待验证点**：白名单里没有 `src`，而 188 的 `useHead` 示例里 `script: [{ src: '...', async: true }]` 是官方写法。**这两处的差别要么是"外部脚本不该走 safe 版"，要么是白名单只约束内联内容**。文档没有直接回答，我不替它选。真要在 safe 版里放外部脚本，先测一次 `src` 会不会被丢——一次渲染检查就能定。

另外注意 `style`/`script` 用的是 **`textContent`**，不是 `innerHTML`：**这正是"内容仍给、但不作为标记解析"的形状**，也解释了我在 CMS 场景丢掉 `<strong>` 的原因。

## 过滤是静默的

参数说明那句只有一行，但它是使用成本最高的一行：

> `input`: A `MaybeComputedRef<HeadSafe>` object containing head data. **You can pass all the same values as `useHead`, but only safe attributes will be rendered.**

"**only safe attributes will be rendered**"——不报错、不警告、不返回被丢清单。所以：

| 症状 | 第一嫌疑 |
| --- | --- |
| head 里少了某属性 | 它不在白名单里 |
| 内联脚本没执行 | `script` 用了 `innerHTML`（应改 `textContent`，但那不会执行） |
| `meta http-equiv` 不见了 | 被剥掉（官方点名列在 warning） |
| body 上 `lang` 没生效 | `bodyAttrs` 白名单不含 `lang`，该写到 `htmlAttrs` |

**验证动作固定一条**：渲染后直接看 `<head>`/`<html>`/`<body>` 的实际属性，而不是看传进去的对象。

返回值这一页没有歧义（与 187 那个矛盾不同）：

> This composable does not return any value.

## 官方示例：用户简介进 description

```vue [app/pages/user-profile.vue]
<script setup lang="ts">
// User-generated content that might contain malicious code
const userBio = ref('<script>alert("xss")<' + '/script>')

useHeadSafe({
  title: `User Profile`,
  meta: [
    {
      name: 'description',
      content: userBio.value, // Safely sanitized
    },
  ],
})
</script>
```

**注意这个例子的形状**：被过滤的是**属性值**（`content` 里塞了脚本字符串），不是属性名。`content` 本身在白名单里，所以标签会渲染，但里面的脚本不会成为可执行内容——这也说明"白名单"与"值被当作文本处理"是两层防护。

那行字符串拼接 `'<script>alert("xss")<' + '/script>'` 是为了在 `.vue` 文件里安全写出一个 `<script>` 字面量，官方这么写正是为了让示例能直接跑。

## 与 useSeoMeta 的分工：两处"XSS 安全"说法

`useSeoMeta` 那一页也自称安全，两处并排看才不误解：

| composable | 官方安全句 | 适用 |
| --- | --- | --- |
| `useHeadSafe` | "prevents XSS attacks by **sanitizing potentially dangerous attributes**" | 需要 `script`/`style`/`link` 等富结构、且值不可信 |
| `useSeoMeta` | "This is the recommended way to add meta tags to your site **as it is XSS safe** and has full TypeScript support."（190） | 只写 SEO 元信息；字段名固定，压根没有自由属性通道 |

**我的读法**：`useSeoMeta` 的安全来自"没有给你写任意属性的入口"，`useHeadSafe` 的安全来自"给了入口但按名单筛"。所以只要需求能落进 100+ 个扁平字段里，用 `useSeoMeta` 更省事；一旦要放 `<script>` 或 `<link>`，回到 `useHead` + 值可信吗？不可信就 `useHeadSafe`。

## 什么时候别用 safe 版

| 情况 | 建议 |
| --- | --- |
| 值来自 CMS/用户/URL 参数 | `useHeadSafe`（官方 "Use this composable whenever you're working with user-generated content."） |
| 需要外部脚本 `src` + `async` | 用 `useHead`（188 的官方示例形状），safe 版的 `src` 待验证 |
| 需要内联脚本执行 | 别用 safe 版（`script` 只给 `textContent`）；这类需求官方指的工具是 `<NuxtScript>`/Scripts 模块那一族（186 之外的话题） |
| 需要在 `body` 上放 `lang` | 写进 `htmlAttrs`，白名单决定这不是风格问题 |
| 全站固定、不来自输入 | `app.head`（186） |

## 自查

1. 官方给这一页的定位句与机制句分别是什么？它包装的是哪个函数？
2. warning 点名剥掉的是哪两种属性？为什么正好是这两个（用白名单回答）。
3. 默写六组白名单里 `htmlAttrs`、`bodyAttrs`、`meta`、`script` 四组。
4. `style` 与 `script` 用哪个字段承载内容？这与 `innerHTML` 的差别是什么？
5. 被过滤的属性会报错吗？官方那句原文怎么说？你的验证动作是什么？
6. `script` 的白名单里没有 `src` 而 188 的示例用了 `src`，你怎么处理这个不一致？
7. `useSeoMeta` 与 `useHeadSafe` 各自宣称的安全来自哪里？什么情况下选前者？

## 下一篇

190 讲 `useSeoMeta`：官方说它有 100+ 个全类型化字段、是"推荐的加 meta 方式"，以及它要求响应式值写成 `() => value` 这种 getter 形态——与 `useHead` 的 `computed` 写法不同。

## 本文依据

- `https://nuxt.com/docs/4.x/api/composables/use-head-safe`（本篇主要依据）
  - 原文（定位与机制）："The recommended way to provide head data with user input."；"The `useHeadSafe` composable is a wrapper around the `useHead` composable that restricts the input to only allow safe values. This is the recommended way to manage head data when working with user input, as it prevents XSS attacks by sanitizing potentially dangerous attributes."
  - 原文（warning）："When using `useHeadSafe`, potentially dangerous attributes like `innerHTML` in scripts or `http-equiv` in meta tags are automatically stripped out to prevent XSS attacks. Use this composable whenever you're working with user-generated content."
  - 原文（Type 与白名单）：`export function useHeadSafe (input: MaybeComputedRef<HeadSafe>): void`；"### Allowed Attributes" — "The following attributes are whitelisted for each head element type:" + `const WhitelistAttributes = { htmlAttrs: ['class','style','lang','dir'], bodyAttrs: ['class','style'], meta: ['name','property','charset','content','media'], noscript: ['textContent'], style: ['media','textContent','nonce','title','blocking'], script: ['type','textContent','nonce','blocking'], link: ['color','crossorigin','fetchpriority','href','hreflang','imagesrcset','imagesizes','integrity','media','referrerpolicy','rel','sizes','type'] }`；"See [@unhead/vue](...) for more detailed types."
  - 原文（Parameters 与 Return Values）："`input`: A `MaybeComputedRef<HeadSafe>` object containing head data. You can pass all the same values as `useHead`, but only safe attributes will be rendered."；"This composable does not return any value."
  - 原文（示例与 read-more）：`app/pages/user-profile.vue` 的 `const userBio = ref('<script>alert("xss")<' + '/script>')` 与 `useHeadSafe({ title: 'User Profile', meta: [{ name: 'description', content: userBio.value // Safely sanitized }] })`；read-more 指向 unhead 文档站
- `https://nuxt.com/docs/4.x/api/composables/use-head`
  - 对照用（见 187、188）：`script` 的 `innerHTML` 示例、`script: [{ src, async: true }]` 形态、`useHeadSafe` 那条 warning 的指向
- `https://nuxt.com/docs/4.x/api/composables/use-seo-meta`
  - 对照用（见 190）："This is the recommended way to add meta tags to your site as it is XSS safe and has full TypeScript support."
