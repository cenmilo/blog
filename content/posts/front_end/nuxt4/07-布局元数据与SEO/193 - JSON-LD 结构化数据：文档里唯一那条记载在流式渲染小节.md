---
url: /posts/50ed29001fa0f22419746bcc24a8d786/
title: JSON-LD 结构化数据：文档里唯一那条记载在流式渲染小节
date: 2026-12-15T10:00:00+08:00
updated: 2026-12-15T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：我想给页面加 JSON-LD，第一反应是去 Nuxt 文档找"结构化数据"那一节——**这一节不存在**。我把整套 4.x 文档里 `JSON-LD`、`ld+json`、`structured data`、`schema.org` 四个关键词都搜过，全库只命中一处，而且不在 SEO 章节，在 `guide/going-further/experimental-features` 的流式渲染（streaming）警告框里，原话是："JSON-LD scripts, `noscript` tags and body-positioned tags are the exception: they are rendered as real markup before `</body>`, where every client can read them."。所以文档实际能给我的只有两块拼图：一是 `useHead` 的 `script` 字段怎么把对象变成标签——"Each element is mapped to a `<script>` tag, where object properties correspond to HTML attributes"，而内联内容靠 `innerHTML`，官方 Usage 示例就是 `script: [{ innerHTML: 'console.log(\'Hello world\')' }]`；二是 `useHeadSafe` 的白名单把这条路堵了一半——`script` 一组只允许 `['type', 'textContent', 'nonce', 'blocking']`，**`innerHTML` 和 `src` 都不在里面**。把这两块拼起来，结论很具体：同样的数据用 `useHead` 还是 `useHeadSafe` 写，能用的属性不一样；而开了流式渲染时，JSON-LD 落的位置是 `</body>` 之前，不在 `<head>` 里。

categories:
  - 前端开发

tags:
  - nuxt4
  - JSON-LD
  - useHead
  - 结构化数据
  - SEO

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先说一件我踩了挺久才反应过来的事：Nuxt 文档里没有"结构化数据"这一节。不是藏得深，是没有。所以我打算按文档实际写了什么来讲，把我能确认的和只能自己验的分开。

## 全库唯一那条记载长什么样

`JSON-LD` 这四个字母在 4.x 文档里出现的唯一位置，是流式渲染那一节的 warning 框。这个框在讲一件很麻烦的事：开了流式之后，`<head>` 会先刷出去，晚注册的标签追不上它。原文（同一段的前半）：

> **Head tags registered after the shell flushes become client-side patches.** The renderer waits for the first render pass before flushing the shell, so `useHead`/`useSeoMeta` called synchronously in `<script setup>` (before any `await`), in plugins, or in middleware render into the shell `<head>` as usual. Tags registered later (after an `await` in setup, or inside a nested `<Suspense>` boundary) cannot reach the already-sent `<head>`; they are delivered as inline scripts that patch the DOM on the client.

紧接着就是那句跟 JSON-LD 有关的话：

> Clients that run JavaScript apply the patch, and clients matching `botRegex` never stream in the first place, so they receive the full buffered `<head>`. Anything else (a crawler your `botRegex` does not list, a link unfurler, `curl`) will not see those tags at all. **JSON-LD scripts, `noscript` tags and body-positioned tags are the exception: they are rendered as real markup before `</body>`, where every client can read them.** Keep other SEO-critical tags (canonical links, social meta) synchronous, or opt the route out of streaming with `routeRules: { '/path': { streaming: false } }`.

这句话给我的实用信息不是"怎么写 JSON-LD"，而是**别假设它在 `<head>` 里**。我原来的心智模型是"结构化数据 = head 里的一段 script"，按文档这句，流式场景下它是文档末尾的真实标记。要确认自己项目里的落点，最省事的办法就是 `curl` 一份服务端 HTML，看那段 JSON 出现在 `<head>` 还是 `</body>` 前面。

## 那到底怎么写：文档给的只有 `script` 字段这一条路

`useHead` 的参数表里跟 JSON-LD 有关的就一行：

| 字段 | 类型 | 文档说明（原话） |
| --- | --- | --- |
| `script` | `Script[]` | "Array of script objects. Each element is mapped to a `<script>` tag, where object properties correspond to HTML attributes." |

注意后半句：**对象属性对应的是 HTML 属性**。也就是说 `type`、`src`、`async` 这些是属性，而脚本的正文不是属性。文档没有专门教"内联内容怎么给"，但 Usage 示例直接给了写法：

```vue [app/app.vue]
<script setup lang="ts">
useHead({
  title: 'My App',
  meta: [
    { name: 'description', content: 'My amazing site.' },
  ],
  bodyAttrs: {
    class: 'test',
  },
  script: [{ innerHTML: 'console.log(\'Hello world\')' }],
})
</script>
```

`innerHTML` 就是内联内容的位置。把它和"属性对应 HTML 属性"合起来读，`type: 'application/ld+json'` 加 `innerHTML: JSON.stringify(...)` 是这个字段组合的自然写法——**"自然写法"不等于文档写过这种写法**，文档里没有任何一处 JSON-LD 示例，我不假装它有。

作为对照，`useHead` 页自己还有一个"外部脚本"的示例，用的是 `src`：

```vue [app/pages/external.vue]
<script setup lang="ts">
useHead({
  script: [
    {
      src: 'https://cdn.example.com/script.js',
      async: true,
    },
  ],
})
</script>
```

## 换用 useHeadSafe 时这条路会窄一截

189 那篇抄过 `useHeadSafe` 的白名单，这里只把 `script` 和 `noscript` 两组拎出来，因为它们直接决定 JSON-LD 能不能这么写：

```ts
const WhitelistAttributes = {
  meta: ['name', 'property', 'charset', 'content', 'media'],
  noscript: ['textContent'],
  style: ['media', 'textContent', 'nonce', 'title', 'blocking'],
  script: ['type', 'textContent', 'nonce', 'blocking'],
}
```

对照上面的写法就能看出两件事：

1. `script` 组里**没有 `innerHTML`，也没有 `src`**。所以带不可信来源的内容改用 `useHeadSafe` 时，内联正文只能走 `textContent` 这一族；而 `src` 直接不在允许列表里（文档没解释为什么，我也只能照抄，并把它列为要实测的一项）。
2. `noscript` 组只有 `['textContent']` 一个键——但白名单里 `htmlAttrs`、`bodyAttrs`、`meta`、`link` 等组是各自独立的，`noscript` 这一组之窄是原文就长这样。

`useHeadSafe` 页的定位句也顺带说明它是给谁用的："This is the recommended way to manage head data when working with user input, as it prevents XSS attacks by sanitizing potentially dangerous attributes."，并且"**potentially dangerous attributes like `innerHTML` in scripts or `http-equiv` in meta tags are automatically stripped out**"。被剥掉是静默的——所以如果是从 CMS 拿来的结构化数据，我倾向于先问一句：这段 JSON 里有没有用户能改的字段。

## 我把能确认与不能确认的分开列

| 项目 | 文档状态 | 依据或处理方式 |
| --- | --- | --- |
| JSON-LD 有专门章节 | **没有**，全库只命中流式那一处 | 四个关键词全库检索 |
| `script` 元素怎么变成标签 | 写了 | 参数表那行 "object properties correspond to HTML attributes" |
| 内联内容用什么键 | 示例里是 `innerHTML` | Usage 示例逐字可查 |
| `useHeadSafe` 下 `innerHTML` 可用 | **不可用**（不在白名单） | 白名单逐字 |
| 流式下 JSON-LD 的落点 | `</body>` 之前的真实标记 | 流式 warning 原话 |
| JSON-LD 与富结果的具体 schema 选型 | 与 Nuxt 无关，文档不涉及 | 去搜索引擎的文档查 |

## 自查

1. 我说"文档里没有 JSON-LD 章节"，这个否定结论是怎么得到的？（答：四个关键词全库检索，只命中 experimental-features 一处。）
2. `useHead` 和 `useHeadSafe` 写同一份内联脚本，差别在哪个键上？（答：`innerHTML` 只在 `useHead` 侧可用，`useHeadSafe` 的 `script` 白名单里只有 `type`/`textContent`/`nonce`/`blocking`。）
3. 流式渲染开着的时候，我怎么知道我的 JSON 落在 head 还是 body 末尾？（答：`curl` 服务端 HTML 看位置；同一段还给了 `routeRules: { '/path': { streaming: false } }` 这条退出路径。）

## 下一篇

194 - definePageMeta 里的 title 不是具名字段：它和浏览器标签上的标题是两回事

## 本文依据

- `api/composables/use-head`：参数表 "Array of script objects. Each element is mapped to a `<script>` tag, where object properties correspond to HTML attributes."；Usage 示例 `script: [{ innerHTML: 'console.log(\'Hello world\')' }]`；外部脚本示例 `script: [{ src: 'https://cdn.example.com/script.js', async: true }]`；以及 "If the data comes from a user or other untrusted source, we recommend you check out `useHeadSafe`."
- `api/composables/use-head-safe`：`WhitelistAttributes` 中 `script: ['type', 'textContent', 'nonce', 'blocking']`、`noscript: ['textContent']`；warning 段 "potentially dangerous attributes like `innerHTML` in scripts or `http-equiv` in meta tags are automatically stripped out to prevent XSS attacks"。
- `guide/going-further/experimental-features`（streaming 一节）："Head tags registered after the shell flushes become client-side patches."、"`useHead`/`useSeoMeta` called synchronously in `<script setup>` (before any `await`), in plugins, or in middleware render into the shell `<head>` as usual."、"JSON-LD scripts, `noscript` tags and body-positioned tags are the exception: they are rendered as real markup before `</body>`, where every client can read them."、"Keep other SEO-critical tags (canonical links, social meta) synchronous, or opt the route out of streaming with `routeRules: { '/path': { streaming: false } }`."
- 检索结论（否定性事实）：`JSON-LD`、`ld+json`、`structured data`、`schema.org` 四个关键词在 4.x 语料中仅命中 `guide_going-further_experimental-features.md` 一处。
