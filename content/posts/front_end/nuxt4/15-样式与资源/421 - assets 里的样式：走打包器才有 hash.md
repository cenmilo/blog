---
url: /posts/e7d96848be3ed1a462caa9c2326fc512/
title: 421 - assets 里的样式：走打包器才有 hash
date: 2027-03-01T10:00:00+08:00
updated: 2027-03-01T10:00:00+08:00
author: cmdragon

summary:
  摘要：`app/assets/` 的定性是"contains by convention every asset that you want the build tool (Vite or webpack) to process"——**走不走打包器是它与 `public/` 的唯一分野**，代价与好处都由此来。三类内容官方点名：样式表（CSS、SASS 等）、字体、不打算从 `public/` 提供的图片。两个关键行为：① Nuxt **不会**把 `app/assets/` 里的文件当静态 URL 提供（不会存在 `/assets/my-file.png` 这种东西），要静态 URL 就用 `public/`；② 模板里 `src` 写成**静态字符串字面量**时构建器才会重写它——`~/assets/img/nuxt.png` 这样"额外变成一个 import、解析到 hash 后的产物文件"，而 `:src` 绑一个运行时拼出来的值构建器看不见、原样使用不重写。目录本身**没有自动扫描**，名字也能换。

categories:
  - 前端开发

tags:
  - nuxt4
  - assets目录
  - hash
  - 静态资源
  - baseURL
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

"我把 logo 放进 `assets/img/`，然后 HTML 里写 `/assets/img/logo.png` 打不开"——这条报错误会重复很多次，因为 `assets/` 的工作方式和直觉相反：它是给打包器吃的，不是给服务器直出的。这一篇讲它的三条边界与 hash 从哪来。

## 定性：给构建器处理的东西

`directory-structure/app/assets` 开头：

> The assets/ directory is used to add all the website's assets that the build tool will process.

里面通常放三类（官方列的）：样式表（CSS、SASS 等）、字体、**不打算从 `public/` 提供的图片**。而 assets 那页最后一句就是把选择权交回给你：

> If you want to serve assets from the server, we recommend taking a look at the `public/` directory.

getting-started/assets 那页把两者并排说得更直白：

> Nuxt uses two directories to handle assets like stylesheets, fonts or images.
> - The `public/` directory content is served at the server root as-is.
> - The `app/assets/` directory contains by convention every asset that you want the build tool (Vite or webpack) to process.

**as-is 直出 vs 交给 Vite/webpack 处理**——这一个差别决定了下面所有行为。

## 边界一：不存在静态的 `/assets/...` URL

官方用 `::note` 单独钉住这条：

> Nuxt won't serve files in the `app/assets/` directory at a static URL like `/assets/my-file.png`. If you need a static URL, use the `public/` directory.

所以浏览器里输 `/assets/img/logo.png` 打不开是**设计如此**：这些文件被打包器消化成了带 hash 的产物，原始路径不存在。要一个稳定的、按目录结构直出的 URL，就得放 `public/`（422）。

引用方式是 `~/assets/` 路径：

```vue
<template>
  <img
    src="~/assets/img/nuxt.png"
    alt="Discover Nuxt"
  >
</template>
```

## 边界二：没有自动扫描、目录名可换

> By convention, Nuxt uses the `app/assets/` directory to store these files but there is no auto-scan functionality for this directory, and you can use any other name for it.

两句要紧的：这个目录**没有自动扫描**（对比 413 里 `shared/utils`、`app/composables` 那种会扫的目录），以及**名字可以换**——它是约定，不是硬编码位置。也就是说 `assets/` 里的东西必须被显式 import/引用才会进产物，不会因为你放了文件就自动打包。

## 边界三：只有静态字面量会被重写

hash 从哪来，官方在 Static vs. Dynamic `src` 一节说得很细：

> When an `src` is a static string literal in your template, the build tool rewrites it into a runtime helper that resolves the final URL. A public path such as `/img/nuxt.png` is wrapped so that your `app.baseURL` is applied when the page renders, and a bundled path such as `~/assets/img/nuxt.png` additionally becomes an import that resolves to the hashed output file.

拆成三层：

1. **静态字符串字面量**的 `src` → 被重写成解析最终 URL 的运行时 helper。
2. `public/` 路径（`/img/nuxt.png`）→ 包一层，让 `app.baseURL` 在渲染时生效。
3. `~/assets/...` 这种要走打包的路径 → **额外变成一个 import，解析到 hash 后的产物文件**。

所以 hash 不是"Nuxt 给文件加了个后缀"，而是**它成了一个被打包器解析的模块**，产物名由构建器决定。

因为 `app.baseURL` 是运行时才应用的，官方补了一条：静态 public 路径即使 base URL 要到部署时才定（比如用 `NUXT_APP_BASE_URL` 设）也能工作，且不依赖文件是否被构建处理——但**只有构建器看得见的那些字面量路径才有这套解析**。

## 反面：运行时拼的 `:src` 完全不重写

官方紧接着给了对照：

> A bound `:src` whose value is assembled at runtime is opaque to the build tool, so none of that rewriting happens. The string is used exactly as written:

```vue
<template>
  <!-- This does not work: the path is built at runtime, so Vite never sees it as an import. -->
  <img :src="`~/assets/img/${name}.png`">
</template>
```

这条正好是上面那句"没有自动扫描"的另一面：不写进字面量、构建器就不知道要打包那个文件。运行时拼出来的 public 路径（`/img/${name}.png`）也**不会**被加 `app.baseURL` 前缀，官方的处置办法是自己用 `useRuntimeConfig().app.baseURL` 拼（例如配合 `joinURL`）。

## 要"只引 URL、不打包内容"怎么办

如果不需要内容被内联、只要 URL，Vite 侧官方给的是显式 `?url`：

```ts
const logos = {
  light: () => import('./assets/img/logo-light.png?url'),
  dark: () => import('./assets/img/logo-dark.png?url'),
}

const logoUrl = (await logos[props.theme]()).default
```

路径只有运行时才知道时用 `import.meta.glob`（`query: '?url'`, `import: 'default'`），懒加载或 `eager: true` 立即加载。官方对这两条各有一句提醒：

> Every matching asset is still included in the build output. Lazy imports load each match on demand, while an eager glob loads all matches up front and can increase the initial JavaScript size or inline small assets.

以及一条 `::warning`：**在服务端渲染的标记里用 URL 之前，先 await 那个懒导入**——Vite 的 `new URL(..., import.meta.url)` 写法在 SSR 下不工作。

## 自查

- `assets/` 的定义就是"给构建工具处理"，与 `public/`（as-is 直出）相对；这一差别决定下面所有行为。
- `app/assets/` 里的文件**不会**以 `/assets/xxx` 静态 URL 提供（`::note`），要静态 URL 用 `public/`。
- 这个目录**没有自动扫描**、名字可按约定改；不被显式引用就不进产物。
- hash 来自"静态字面量 `src` 被重写成 import、解析到 hash 后的产物"；public 路径则是包一层让 `app.baseURL` 运行时生效。
- `:src` 绑运行时拼的值 → 构建器看不见、完全不重写（`~/assets/...` 直接失效，运行时拼的 public 路径也不带 baseURL 前缀，要自己用 `useRuntimeConfig().app.baseURL` 拼）。
- 只要 URL 用 `?url` 或 `import.meta.glob`；glob 的每个匹配都仍会进产物、eager 会撑大初始 JS；SSR 里用懒导入的 URL 前必须先 await（`new URL(..., import.meta.url)` 不支持 SSR）。

## 下一篇

下一篇 422 讲 `public/`：文件按根 URL 直出、保留原文件名，以及静态字面量会补 `app.baseURL`、运行时拼的不会这个前缀差别。
