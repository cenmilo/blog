---
url: /posts/c58f5b67fea197e1f9fde7c908878fc9/
title: 192 - OG 与 Twitter 卡片字段映射：扁平字段名与 meta 属性怎么对上
date: 2026-12-15T09:00:00+08:00
updated: 2026-12-15T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇处理一件事：**扁平字段名与最终 `<meta>` 标签之间的对应关系，官方文档里到底写了多少**。写明的只有四处，我把它们并排：`useSeoMeta` 示例的六个键是 `title`、`ogTitle`、`description`、`ogDescription`、`ogImage`、`twitterCard: 'summary_large_image'`；而 `useHead` 的 Basic Meta Tags 示例直接给了渲染形状——`meta: [{ name: 'description', content: 'Learn more about our company' }, { property: 'og:title', content: 'About Us' }, { property: 'og:description', content: '...' }]`。**两例并读得到三条对应**：`description → name="description"`、`ogTitle → property="og:title"`、`ogDescription → property="og:description"`，也就是 **OG 一族用 `property`、描述类用 `name`**——这正是 `useSeoMeta` 页把"**using `name` instead of `property`**"列为首要常见错误的原因（原文："This helps you avoid common mistakes, such as using `name` instead of `property`, as well as typos - with over 100+ meta tags fully typed."）。两个属性名都合法这一点也有独立依据：`useHeadSafe` 的白名单里 `meta` 一组是 `['name', 'property', 'charset', 'content', 'media']`。`title` 落点不同——186/188 的官方表述是 `title` "Sets the page title"，即 `<title>` 元素而非 meta。**其余映射（`twitterCard`、`ogImage`、`robots`）文档没有给渲染结果**，我只把命名规律列成待验证项并给出一次核对方法（渲染后读 `<head>`），不把它写成结论。

categories:
  - 前端开发

tags:
  - nuxt4
  - Open Graph
  - Twitter 卡片
  - meta
  - SEO

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


`ogTitle` 最终会变成 `<meta property="og:title">` 还是 `<meta name="og:title">`？这个问题官方文档没有一句直接回答，但把两节的示例并起来就能定死前一半答案——而后一半（Twitter 一族）我推不出来，只能验。这一篇把"文档写了的"和"要你确认的"分开列。

## 官方写明了的四处

| 出处 | 内容 |
| --- | --- |
| `useSeoMeta` Usage | `title`、`ogTitle`、`description`、`ogDescription`、`ogImage`、`twitterCard: 'summary_large_image'` |
| `useHead` Basic Meta Tags | `meta: [{ name: 'description', ... }, { property: 'og:title', ... }, { property: 'og:description', ... }]` |
| `useSeoMeta` 页定位段 | "This helps you avoid common mistakes, such as **using `name` instead of `property`**..." |
| `useHeadSafe` 白名单 | `meta: ['name', 'property', 'charset', 'content', 'media']` |

**第三、四处合起来说明这不是风格问题**：`name` 与 `property` 都是合法属性（白名单里都有），所以写错**不会报错**，只会让抓取方认不出来。官方把它列为"首要常见错误"，正是因为它是静默的。

## 两例并读能定死的对应关系

| 扁平字段 | 最终标签 | 依据强度 |
| --- | --- | --- |
| `description` | `<meta name="description">` | 直接写明（`useHead` 示例里就是 `name: 'description'`） |
| `ogTitle` | `<meta property="og:title">` | **两侧示例对齐**：`useSeoMeta` 用 `ogTitle`，`useHead` 用 `property: 'og:title'` |
| `ogDescription` | `<meta property="og:description">` | 同上 |
| `title` | `<title>`（不是 meta） | `useHead` 参数表：`title` — "Sets the page title."（188） |

**规律是"OG 一族用 `property`"**，这一点有官方示例支撑，也正是那句"避免用 `name` 写 `property`"针对的场景。

## 文档没写、只能验的那一半

| 扁平字段 | 我按命名规律预期 | 文档状态 |
| --- | --- | --- |
| `ogImage` | `<meta property="og:image">` | 未给渲染结果 |
| `twitterCard` | 键名带 `twitter:` 前缀，`name` 还是 `property` 不确定 | 未给渲染结果 |
| `robots` | `<meta name="robots">` | 未给渲染结果（190 的示例只写了 `robots: 'index, follow'`） |
| `title` 与 OG 标题是否联动 | 不联动（需手写两遍） | 示例里 `ogTitle` 是**手写**的，看不出是否有联动 |

**注意第二行**：Twitter 卡片的键名与前缀归属，两节示例都没出现渲染结果——不能把"肯定是 `name`"写成结论。

### 一次核对覆盖全表

```vue [app/pages/seo-check.vue]
<script setup lang="ts">
useSeoMeta({
  title: 'SEO Check',
  ogTitle: 'OG Title',
  ogImage: 'https://example.com/og.png',
  twitterCard: 'summary_large_image',
  description: 'desc',
  robots: 'index, follow',
})
</script>
```

**看渲染结果，不看入参**：`nuxt build && nuxt preview` 之后直接请求这一页，读响应 HTML 里的 `<head>` 段落，逐个记下每个字段渲染成了什么属性名与 `content`。**这一遍记下来就是你的映射表**，比任何推理都可靠，也只需一次；它同时能确认上面四个"未写"的格子。

## 为什么这些值最好手写全

官方示例本身给了线索：`useSeoMeta` 那段把 `title` 与 `ogTitle`、`description` 与 `ogDescription` **各写了两遍**（值相同）：

```vue
useSeoMeta({
  title: 'My Amazing Site',
  ogTitle: 'My Amazing Site',
  description: 'This is my amazing site, let me tell you all about it.',
  ogDescription: 'This is my amazing site, let me tell you all about it.',
  ogImage: 'https://example.com/image.png',
  twitterCard: 'summary_large_image',
})
```

**如果是自动联动的，示例没必要写两遍**。所以我的处理是：页面标题与 OG 标题、描述与 OG 描述**分别给值**（同一份数据源即可），不指望联动。这一点也属于"示例支撑、正文没写规则"的那类，用上一节的核对法一次确认。

## 写 OG 时会踩的三条

| 坑 | 文档依据 |
| --- | --- |
| 用 `name` 写 OG 键 | `useSeoMeta` 页把这句列为要帮忙避免的首要错误（且静默） |
| 用 `useHead` 手搓时数组元素漏 `content` | `useHead` 示例里每个元素都是 `{ name/property, content }` 成对出现（187、188） |
| 用户/编辑器提供的值直接进 OG 字段 | `useHead` 页 warning："If the data comes from a user or other untrusted source, we recommend you check out `useHeadSafe`"；`useSeoMeta` 自称 XSS safe（190） |

**第三条与 OG 图片关系最大**：`ogImage` 的值通常来自 CMS 字段，安全性取决于你选的 composable，而不是你自己做的转义。图片 URL 还需要是绝对地址（201 那条线处理这件事）。

## 落到项目里的两条规则

1. **优先 `useSeoMeta`**：字段名固定、有类型、不用自己决定 `name`/`property`（190 的推荐句）。
2. **要写非 meta 元素时回到 `useHead`**：`<link rel="canonical">`、`<base>` 这类（188 的字段表）；此时 `name`/`property` 由你自己写，**OG 一族照上表用 `property`**，其余按平台要求，拿不准就按"一次核对"那节测。

## 自查

1. 官方文档里与"字段映射"直接相关的四处分别是哪些？各自能定死什么？
2. 为什么把 OG 写成 `name` 不会报错？白名单那一行怎么佐证 `name`/`property` 都合法？
3. `title` 落在哪个元素？依据是哪句？
4. 三个"未写清"的字段是哪些？你的核对动作是什么、为什么值得做一次？
5. 官方示例为什么把 `title` 与 `ogTitle` 写成两遍？这对你的写法意味着什么？
6. 什么情况下必须回到 `useHead` 而不是 `useSeoMeta`？

## 下一篇

193 讲 JSON-LD 结构化数据：官方给的 `useHead` 写法（`script` 数组那族字段怎么组合），以及它与 189 白名单的冲突点在哪。

## 本文依据

- `https://nuxt.com/docs/4.x/api/composables/use-seo-meta`
  - 原文（六字段示例与定位）："The `useSeoMeta` composable lets you define your site's SEO meta tags as a flat object with full TypeScript support."；"This helps you avoid common mistakes, such as using `name` instead of `property`, as well as typos - with over 100+ meta tags fully typed."；important "This is the recommended way to add meta tags to your site as it is XSS safe and has full TypeScript support."；Usage 示例六个键（`title`、`ogTitle`、`description`、`ogDescription`、`ogImage: 'https://example.com/image.png'`、`twitterCard: 'summary_large_image'`）；"There are over 100 parameters. See the full list of parameters in the source code."
- `https://nuxt.com/docs/4.x/api/composables/use-head`
  - 原文（Basic Meta Tags 示例，给出渲染属性形态）：`useHead({ title: 'About Us', meta: [{ name: 'description', content: 'Learn more about our company' }, { property: 'og:title', content: 'About Us' }, { property: 'og:description', content: 'Learn more about our company' }] })`（`app/pages/about.vue`）；`title` — "Sets the page title."；`meta` — "Array of meta objects. Each element is mapped to a `<meta>` tag, where object properties correspond to HTML attributes."
- `https://nuxt.com/docs/4.x/api/composables/use-head-safe`
  - 原文（白名单）："`meta`: ['name', 'property', 'charset', 'content', 'media']"（`WhitelistAttributes` 对象内）；warning "potentially dangerous attributes like `innerHTML` in scripts or `http-equiv` in meta tags are automatically stripped out"；示例 `content: userBio.value // Safely sanitized`
- `https://nuxt.com/docs/4.x/getting-started/seo-meta`
  - 原文（同一六字段示例与 useHead 手写 meta 的对照）：`useSeoMeta({ title, ogTitle, description, ogDescription, ogImage, twitterCard })` 段；"This helps you avoid typos and common mistakes, such as using `name` instead of `property`."
- 相关篇章（见 187、188、189、190）：`useHead` 的字段与示例、`useHeadSafe` 的属性过滤、`useSeoMeta` 的反应式与服务端条件写法
