---
url: /posts/7d5a244f7e4a1ca8a225042b53922053/
title: 203 - 多语言第一步：htmlAttrs 的 lang 与 dir 写在四个地方会怎样
date: 2026-12-18T11:00:00+08:00
updated: 2026-12-18T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：做多语言时我一开始把力气全花在文案切换上，直到浏览器读屏把英文调子套在中文页面上，才回去看 `<html>` 上那个 `lang`。文档里能写它的地方有四处，我把四处的原话贴齐：一，`app.head.htmlAttrs`（SEO 页的配置示例里就一行 `lang: 'en'`，配合注释 "It's good practice to set tags here that won't change such as your site title default, language and favicon."）；二，`useHead({ htmlAttrs })`（参数表 "Sets attributes of the `<html>` tag. Each object property is mapped to the corresponding attribute."，示例里 `lang: 'en'` 与 `class: computed(...)` 并排）；三，模块侧的 Kit 工具 `setGlobalHead`（参数清单里有 "htmlAttrs: Attributes to add to the `<html>` tag"，合并规则写的是 "merged with any existing head configuration using deep merging, with your provided values taking precedence."）；四，`useHeadSafe` 的白名单里 `htmlAttrs` 一组恰好是 `['class', 'style', 'lang', 'dir']`——**这是文档里唯一一处把 `dir` 明确列出来的地方**，做 RTL 时 `dir` 也在 `useHead` 那一族的映射规则内（属性名直接映射成 HTML 属性）。至于"四处同时写谁赢"，文档没有一句优先级表，我按未说明处理并给实测方法。

categories:
  - 前端开发

tags:
  - nuxt4
  - 多语言
  - htmlAttrs
  - useHead
  - 无障碍

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


`lang` 是 `<html>` 标签上的一个属性，值是语言代码，比如 `zh-CN`、`en`。屏幕阅读器靠它决定用什么发音引擎，翻译工具也读它。它跟你在页面上显示哪种文字是两件事——**这是最容易搞混的地方：文案切了，属性没切，读屏那边还是旧语言**。

## 四处入口，逐个看原话

| 写在哪 | 文档原话 | 能不能随路由变 |
| --- | --- | --- |
| `app.head.htmlAttrs`（nuxt.config） | 配置示例只给了 `htmlAttrs: { lang: 'en' }` 两行，同段建议 "It's good practice to set tags here that won't change such as your site title default, language and favicon." | **不能**——"This method does not allow you to provide reactive data." |
| `useHead({ htmlAttrs })` | "Sets attributes of the `<html>` tag. Each object property is mapped to the corresponding attribute." | 能，示例里就是 computed |
| Kit `setGlobalHead({ htmlAttrs })` | 参数清单 "- `htmlAttrs`: Attributes to add to the `<html>` tag" | 构建期，全站生效 |
| `useHeadSafe({ htmlAttrs })` | 白名单 `htmlAttrs: ['class', 'style', 'lang', 'dir']` | 能，但只允许这四个键 |

SEO 页那段配置是多数项目该放默认值的位置：

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  app: {
    head: {
      title: 'Nuxt', // default fallback title
      htmlAttrs: {
        lang: 'en',
      },
      link: [
        { rel: 'icon', type: 'image/x-icon', href: '/favicon.ico' },
      ],
    },
  },
})
```

注意它把 `language` 归进"不会变的那类"，所以官方给的定位是**站点默认语言**。而真正的多语言站点每个路由语言不同，那一行 `lang: 'en'` 就变成兜底值而不是最终值。

## 每个路由换语言：官方示例长这样

`useHead` 页的 "Body and HTML Attributes" 示例，正好是"静态 lang + 反应式 class"的组合：

```vue [app/pages/themed.vue]
<script setup lang="ts">
const isDark = ref(true)

useHead({
  htmlAttrs: {
    lang: 'en',
    class: computed(() => isDark.value ? 'dark' : 'light'),
  },
  bodyAttrs: {
    class: 'themed-page',
  },
})
</script>
```

从这张示例能确定两件事：`htmlAttrs` 的值可以是 computed（同页 note 也写了 "The properties of `useHead` can be dynamic, accepting `ref`, `computed` and `reactive` properties."），以及 `htmlAttrs` 与 `bodyAttrs` 是**两个独立的键**，分别打在 `<html>` 和 `<body>` 上。

把 `lang: 'en'` 换成 `lang: computed(() => locale.value)` 是我按映射规则做的推广——**文档示例里 `lang` 本身是字面量，没有把它写成 computed**，所以这一步属于合理套用而不是文档原话，落地时要自己确认渲染结果。

## dir 这件事只有白名单那一处提到

`dir`（文字方向，`ltr` 或 `rtl`）在整套文档里出现的清单只有两处相关：`useHeadSafe` 的 `htmlAttrs: ['class', 'style', 'lang', 'dir']`，以及迁移页那句列出可自定义项 "You can customize `title`, `titleTemplate`, `base`, `script`, `noscript`, `style`, `meta`, `link`, `htmlAttrs` and `bodyAttrs`."。没有小节讲 `dir` 怎么用、也没有说它是否会被某个模块覆盖。

所以我只能说：**属性名映射成 HTML 属性**（参数表那句）+ `useHeadSafe` 明确允许 `dir`。这已经够用，因为阿拉伯语站点要做的事就是让 `dir` 跟着语言走，机制和 `lang` 完全同一条。

## 模块也会写 head，这一层得知道

Kit 的 `setGlobalHead` 是唯一写明的"模块侧设置全局 head"的工具，它的合并规则很硬：

> The provided head configuration will be merged with any existing head configuration using deep merging, with your provided values taking precedence.

它的类型面也顺带回答了"`charset` 和 `viewport` 为什么能直接写在 head 配置里"：

```ts
interface AppHeadMetaObject extends SerializableHead {
  charset?: string
  viewport?: string
}
```

也就是说 `setGlobalHead` 的参数清单（12 项：`charset`、`viewport`、`meta`、`link`、`style`、`script`、`noscript`、`title`、`titleTemplate`、`bodyAttrs`、`htmlAttrs`）**比 `useHead` 参数表多了 `charset`/`viewport` 两个快捷键**——这两个键 186 讲过是 Nuxt 默认标签的带 key 快捷写法。

这里必须点一句优先级问题："your provided values taking precedence" 讲的是 `setGlobalHead` 与既有配置合并时它赢。它**没有说** `setGlobalHead` 和你写在 `app.vue` 里的 `useHead` 谁赢。所以我不会画一张四层优先级金字塔——文档没有这句话，画了就是编。

## 想确认自己项目里的实际结果

`lang` 只打在 `<html>` 上，一个属性就够，验证特别便宜：

1. 服务端：`nuxt build` + `nuxt preview`，`curl` 首页，看返回第一行 `<html lang="...">`。这是不跑 JS 的一方看到的值。
2. 客户端：切到一个不同语言的路由，在开发者工具里读 `document.documentElement.lang`。**这一步专门用来抓"URL 变了但属性没变"**——如果第 2 步值不动，说明你的 `useHead` 拿到的语言值不是反应式的。
3. 装了会写 head 的模块时，加测一条：临时注释掉自己的 `useHead`，只看服务端 HTML 里的 `lang` 是谁给的。这能确认 `setGlobalHead` 的"precedence"在你这个项目里落到哪一侧。

## 自查

1. 官方把 `language` 归进哪一类配置？依据句是什么？（答：`app.head` 那一类"不会变的标签"，依据 "It's good practice to set tags here that won't change such as your site title default, language and favicon."）
2. `useHeadSafe` 允许 `htmlAttrs` 的哪四个键？（答：`class`、`style`、`lang`、`dir`。）
3. 怎么确认语言切换后 `<html>` 上的属性真的跟着变了？（答：服务端 `curl` 读一次，客户端读 `document.documentElement.lang` 一次，两者都要看。）

## 下一篇

204 - canonical 链接：文档只提过一次，可用的工具是 useRequestURL
