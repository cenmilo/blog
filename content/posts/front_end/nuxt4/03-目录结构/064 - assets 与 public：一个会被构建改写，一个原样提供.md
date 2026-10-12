---
url: /posts/41281857c43ef6e43b2d5a57d2641170/
title: assets 与 public：一个会被构建改写，一个原样提供
date: 2026-11-02T10:00:00+08:00
updated: 2026-11-02T10:00:00+08:00
author: cmdragon

summary:
  摘要：Nuxt 用两个目录处理样式表、字体或图片这类资源，官方给的分工是：public/ 的内容按原样在服务器根路径提供，app/assets/ 则按约定存放你希望构建工具（Vite 或 webpack）去处理的文件。public 页对结果的描述是"其中的文件在根路径提供且不被构建过程修改，适合必须保住名字的文件（例如 robots.txt）或大概率不会变的（例如 favicon.ico）"；assets 那一页则说明构建工具会"转换原始文件，主要出于性能或缓存目的（例如样式表压缩或浏览器缓存失效）"，并且明确"没有自动扫描功能，你也可以给它用别的名字"，引用方式是 ~/assets/ 路径，且 Nuxt 不会把 app/assets 里的文件以 /assets/my-file.png 这样的静态 URL 提供。最有价值的一节是关于静态与动态 src 的差别：模板里写成字符串字面量的 src 会被构建工具改写成一个运行时 helper——public 路径会被包一层以便渲染时应用 app.baseURL，bundled 路径则额外变成一个解析到带哈希输出文件的 import；而用 :src 在运行时拼出来的值对构建工具是不透明的，字符串原样使用，所以 :src="`~/assets/img/${name}.png`" 不工作，运行时拼出的 /img/${name}.png 也不会被加上 app.baseURL 前缀。

categories:
  - 前端开发

tags:
  - nuxt4
  - 静态资源
  - public
  - assets
  - 构建

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


"图片放哪个目录"这件事，多数人是靠试出来的。官方其实给了**两页判据 + 一节机制**，读完就不必试了。而机制那一节里有一个坑，几乎每个写动态图片列表的人都会撞。

## 官方给的分工：一句话版

> Nuxt 使用两个目录来处理样式表、字体或图片这类资源。
>
> - **`public/` 目录的内容按原样（as-is）在服务器根路径提供。**
> - **`app/assets/` 目录按约定包含你希望构建工具（Vite 或 webpack）处理的每一个资源。**

**一句是"原样提供"，一句是"交给构建工具处理"。** 这就是这一篇的全部主线。

## `public/`：保住名字的那一批

> 包含在 `public/` 目录里的文件**在根路径提供，且不被构建过程修改。** 这适合**必须保住自己名字**的文件（例如 `robots.txt`）*或者*大概率不会变的文件（例如 `favicon.ico`）。

```bash [Directory structure]
-| public/
---| favicon.ico
---| og-image.png
---| robots.txt
```

官方给的用法就是直接按 URL 引用：

```vue [app/app.vue]
<script setup lang="ts">
useSeoMeta({
  ogImage: '/og-image.png',
})
</script>
```

以及 Assets 页那个更完整的例子——**注意路径里带着 `/img/` 这一层，它是 `public/img/` 的映射结果**：

```vue [app/app.vue]
<template>
  <img
    src="/img/nuxt.png"
    alt="Discover Nuxt"
  >
</template>
```

> 例如，引用 `public/img/` 目录里的一张图片文件，它在静态 URL `/img/nuxt.png` 可用。

> （tip）这在 Nuxt 2 里被称为 `static/` 目录。

**这一句对读旧资料有用：`static/` 就是今天的 `public/`。**（617-629 讲迁移时会再遇到这类改名。）

## `app/assets/`：交给构建工具的那一批

官方在 Assets 页给的处理链条说得很清楚：

> Nuxt 使用 Vite（默认）或 webpack 来构建并打包你的应用。这些构建工具的主要功能是处理 JavaScript 文件，但它们可以通过 Vite 的 plugins 或 webpack 的 loaders 扩展，以处理其它种类的资源，例如样式表、字体或 SVG。**这一步会转换原始文件，主要出于性能或缓存目的（例如样式表压缩或浏览器缓存失效）。**

**"转换"与"缓存失效"这两个词解释了为什么这类文件的名字会变**（哈希进文件名了）。

然后是三条容易被误读的事实：

> 按约定，Nuxt 使用 `app/assets/` 目录来存放这些文件，**但这个目录没有自动扫描功能，你也可以给它用别的名字。**
>
> 在你应用的代码里，你可以使用 **`~/assets/` 路径**来引用位于 `app/assets/` 的文件。

```vue [app/app.vue]
<template>
  <img
    src="~/assets/img/nuxt.png"
    alt="Discover Nuxt"
  >
</template>
```

> （note）**Nuxt 不会把 `app/assets/` 目录里的文件以 `/assets/my-file.png` 这样的静态 URL 提供。如果你需要一个静态 URL，请使用 `public/` 目录。**

**这三条合起来是一组很有用的判据：**

- **"没有自动扫描"**——这个目录**不像 `components/`、`middleware/` 那样按文件生成什么**（050 那张地图上它是唯一一个由"构建工具处理"而非"框架扫描注册"定义的目录）。
- **"可以换别的名字"**——051 里 `dir.assets` 的默认值是 `"app/assets"`，且它被别名为 `~assets`。**约定不是强制。**
- **"不会以 `/assets/xxx` 提供"**——**这是"我把图片放进 `app/assets` 为什么 404"的官方答案。** 需要能被直接 URL 访问 → `public/`。

## 选哪个：一张判据表

| 你的需求 | 放哪 | 依据 |
| --- | --- | --- |
| 文件名必须稳定（`robots.txt`、`favicon.ico`） | `public/` | "必须保住自己名字的文件" |
| 需要能被一个固定 URL 访问到 | `public/` | "不会以静态 URL 提供 assets 里的文件" |
| 想让构建工具处理（压缩、缓存失效、哈希） | `app/assets/` | "转换原始文件，主要出于性能或缓存目的" |
| 样式表、字体这类要进打包图的 | `app/assets/` | 该页开头那句"样式表、字体或图片"，且构建工具通过插件/loaders 处理 |
| 只有运行时才知道路径 | 见下面两节 | 官方"Public Assets"与"Bundled Assets with Vite" |

## 核心机制：静态字面量会被改写，绑定的不会

这一节是官方专门写的"Static vs. Dynamic `src`"，也是最值钱的一段：

> 当模板里的 `src` 是一个**静态字符串字面量**时，构建工具会把它**改写成一个运行时 helper**，该 helper 解析出最终 URL。像 `/img/nuxt.png` 这样的 public 路径会被**包上一层，以便在页面渲染时应用你的 `app.baseURL`**；而像 `~/assets/img/nuxt.png` 这样的 bundled 路径**还会额外变成一个 import**，解析到带哈希的输出文件。

```vue
<template>
  <!-- Static paths are rewritten: app.baseURL is applied at runtime, and the bundled file is hashed. -->
  <img src="/img/nuxt.png">
  <img src="~/assets/img/nuxt.png">
</template>
```

> 因为 `app.baseURL` 是在**运行时**应用的，所以即使 base URL 只在部署时才知道（例如通过 `NUXT_APP_BASE_URL` 设置），静态的 public 路径依然能工作，而且**无论这个文件是否真的被构建处理，它都能工作。这种解析只发生在构建工具看得见的那些字面路径上。**

**这三句话里有三个可执行结论：**

1. **两种路径都会被改写**，但改写的结果不同：public 路径加上 `baseURL`，assets 路径**除了加 `baseURL` 还变成哈希后的 import**。
2. **子路径部署时静态 public 路径仍然正确**——因为 `baseURL` 是运行时才套上去的（016/029 讲过的 `NUXT_APP_BASE_URL` 场景）。
3. **"只对构建工具看得见的路径生效"**——这句是下面那个坑的来源。

坑在这里：

> **一个值在运行时才组装出来的绑定 `:src`，对构建工具是不透明的，所以这些改写都不会发生。字符串会按你写的样子原样使用：**

```vue
<template>
  <!-- This does not work: the path is built at runtime, so Vite never sees it as an import. -->
  <img :src="`~/assets/img/${name}.png`">
</template>
```

> 因此像 `/img/${name}.png` 这样在运行时拼出的 public 路径**不会**被加上 `app.baseURL` 前缀。**如果你的应用部署在站点根之下，请自己用 `useRuntimeConfig().app.baseURL` 给它加前缀（例如通过 `joinURL`）。**

**这两句要连起来记：`:src` 拼出来的路径没人改写——`~/assets/...` 那种写法根本不工作；`/img/...` 虽然能命中文件，但在子路径部署时会漏掉 `baseURL`。**

## 路径只有运行时才知道时：官方给的三条解法

**解法一：不需要处理就别处理——用 `public/` + URL 引用**

```vue [app/app.vue]
<script setup lang="ts">
const props = defineProps<{
  name: string
}>()

const imageUrl = computed(() => `/img/${props.name}.png`)
</script>

<template>
  <img
    :src="imageUrl"
    :alt="props.name"
  >
</template>
```

> `public/` 里的文件保留它们原本的文件名。

**解法二（Vite）：可能的文件是有限个 → 显式列出 import**

```vue [app/app.vue]
<script setup lang="ts">
const props = defineProps<{
  theme: 'light' | 'dark'
}>()

const logos = {
  light: () => import('./assets/img/logo-light.png?url'),
  dark: () => import('./assets/img/logo-dark.png?url'),
}

const logoUrl = (await logos[props.theme]()).default
</script>

<template>
  <img
    :src="logoUrl"
    alt="Nuxt"
  >
</template>
```

> 每一个 import 都有一个字面路径，所以 **Vite 在构建期能找到这两个文件，同时在运行时只加载被选中的那个模块。**

**解法三（Vite）：一个目录里很多同扩展名的文件 → 变量 dynamic import 或 glob**

```ts
async function getImageUrl (name: string) {
  const image = await import(`./assets/img/${name}.png?url`)
  return image.default
}
```

> **在这个例子里只有文件名可以是动态的。把目录与扩展名留在 import 里，Vite 才能在构建期找到那些可能的文件。**

复杂模式或需要一张可用文件的映射表时，用 `import.meta.glob`：

```ts
const images = import.meta.glob<string>('./assets/img/*.{png,jpg,svg}', {
  query: '?url',
  import: 'default',
})

async function getImageUrl (name: string) {
  const load = images[`./assets/img/${name}.png`]

  if (!load) {
    throw new Error(`Unknown image: ${name}`)
  }

  return await load()
}
```

> glob 导入默认是**懒加载**的。如果必须同步拿到这些 URL，添加 `eager: true`：

```ts
const images = import.meta.glob<string>('./assets/img/*.{png,jpg,svg}', {
  query: '?url',
  import: 'default',
  eager: true,
})
```

> **每一个匹配的资源仍然会被包含进构建输出。懒加载按需加载每个匹配项，而 eager 的 glob 会一次性加载所有匹配项，可能增大你的初始 JavaScript 体积，或把小资源内联进来。**

最后那条警告值得单独记住：

> **在服务端渲染的标记里使用某个懒导入的 URL 之前，请先 await 它。Vite 的 `new URL(..., import.meta.url)` 模式与 SSR 不兼容。**

## 自查

- [ ] `robots.txt` 与 `favicon.ico` 该放哪？官方给的理由分别是什么？
- [ ] 把图片放进 `app/assets/img/`，能用 `/img/x.png` 访问吗？正确 URL 是什么？
- [ ] `src="/img/a.png"` 与 `:src="\`/img/${name}.png\`"` 谁会被加上 `app.baseURL`？
- [ ] `:src="`~/assets/img/${name}.png`"` 为什么不工作？
- [ ] 六个主题图标该怎么写？六十个图标又该怎么写、`eager` 的代价是什么？
- [ ] SSR 标记里用懒导入的 URL，官方要求先做什么？哪个 Vite 模式与 SSR 不兼容？

## 下一篇

065 打开 `server/` 目录：`api`、`routes`、`middleware`、`plugins`、`utils`、`types` 各自什么时候被扫描、服务端中间件为什么不该返回值，以及 4.x 文档里 `server/` 这一侧到底有什么。

## 本文依据

- `getting-started/assets`（Nuxt 用两个目录处理样式表/字体/图片：public/ 内容按原样在服务器根路径提供、app/assets/ 按约定包含你希望构建工具 Vite 或 webpack 处理的每个资源；Public Directory 用作静态资源的公共服务器、可用根 URL / 从应用代码或浏览器取文件、示例 public/img/ 的文件在静态 URL /img/nuxt.png 可用并给出 app.vue 的 img src="/img/nuxt.png"；Assets Directory：Nuxt 用 Vite（默认）或 webpack 构建打包，其主功能是处理 JS 但可通过 plugins/loaders 扩展以处理样式表/字体/SVG 等资源，这一步转换原始文件主要为性能或缓存目的（样式表压缩或浏览器缓存失效）；按约定用 app/assets/ 存放这些文件但该目录没有自动扫描功能且可换别的名字、代码里用 ~/assets/ 路径引用；示例 img src="~/assets/img/nuxt.png"；note Nuxt 不会把 app/assets/ 里的文件以 /assets/my-file.png 这样的静态 URL 提供，需要静态 URL 请用 public/；Static vs Dynamic src：静态字符串字面量的 src 会被构建工具改写为解析最终 URL 的运行时 helper，public 路径被包一层以便页面渲染时应用 app.baseURL、bundled 路径额外变成解析到带哈希输出文件的 import，因 app.baseURL 在运行时应用故即使 base URL 只在部署时知道（如 NUXT_APP_BASE_URL）静态 public 路径也工作且无论文件是否被构建处理，这种解析只发生在构建工具看得见的路径上；绑定 :src 运行时组装的值对构建工具不透明故不改写、字符串原样使用，示例 :src="`~/assets/img/${name}.png`" 注释 This does not work；运行时拼出的 /img/${name}.png 不会被加 app.baseURL 前缀，若应用部署在站点根之下请用 useRuntimeConfig().app.baseURL 自行加前缀（例如 joinURL）；Public Assets 解法：不需处理或哈希的文件放 public/ 并按 URL 引用，示例 computed(() => `/img/${props.name}.png`) 并说明 public/ 文件保留原本文件名；Bundled Assets with Vite：可能文件已知时显式列出 import（logos 对象里 () => import('./assets/img/logo-light.png?url')，await logos[props.theme]()).default，说明每个 import 有字面路径故 Vite 构建期能找到两个文件而运行时只加载选中模块；同目录同扩展名很多文件时用变量 dynamic import（await import(`./assets/img/${name}.png?url`)）且只有文件名可动态、目录与扩展名留在 import 里；更宽模式或显式映射表用 import.meta.glob（query '?url'、import 'default'、默认 lazy、需同步 URL 时 eager: true），每个匹配资源仍进构建输出，eager 一次加载全部匹配可能增大初始 JS 体积或内联小资源；warning 在服务端渲染标记使用懒导入 URL 前必须 await，Vite 的 new URL(..., import.meta.url) 模式与 SSR 不兼容）
- `directory-structure/public`（public/ 目录用于服务你网站的静态资源；其中的文件在根路径提供且不被构建过程修改，适合必须保住名字的文件（robots.txt）或大概率不会变的（favicon.ico）；目录结构示例 favicon.ico/og-image.png/robots.txt；示例 useSeoMeta({ogImage: '/og-image.png'})；tip 这在 Nuxt 2 中称为 static/ 目录）
- `directory-structure/app/assets`（app/assets 是构建工具处理的打包资源目录，见 050 与 051 的 dir.assets 默认值 app/assets 与 ~assets 别名）
- `api/nuxt-config`（dir.assets 默认 "app/assets" 且"在你的构建里被别名为 ~assets"；app.baseURL 与 NUXT_APP_BASE_URL 的子路径部署机制见 029）
