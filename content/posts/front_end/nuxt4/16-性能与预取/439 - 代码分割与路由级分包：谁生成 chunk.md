---
url: /posts/c0e282be35d9df4036fe954533c777ce/
title: 439 - 代码分割与路由级分包：谁生成 chunk
date: 2027-03-07T10:00:00+08:00
updated: 2027-03-07T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：chunk 不是你手动切的，是五处写法各自决定的：`app/pages/` 下每个页面用动态导入切开（官方原话"By using dynamic imports for each page"），组件名前加 `Lazy` 前缀，全局组件每个单独出一个 chunk，`addComponent` 的 `isAsync` 会无视 `Lazy` 前缀强制异步分包，服务端岛屿里带 `nuxt-client` 的那个组件只把自己的 chunk 发给浏览器。关闭分割只有一条官方写法（`codeSplitting: false`），而官方的态度是"通常只会增加首屏下载量"。最容易误判的一点是：切出 chunk 不等于不加载——懒组件不配条件渲染仍然会急切加载。

categories:
  - 前端开发

tags:
  - nuxt4
  - 代码分割
  - chunk
  - 懒加载
  - 性能
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

有人说"把组件拆成 chunk 就好"。你照着把一个页面的内容拆成七八个组件文件，构建出来主包一点没小。

问题在"谁切"这件事上：**Nuxt 里没有"分包"这个配置项让你逐个指定**，chunk 是由几种不同的写法各自产生的。搞清这几种写法，才知道该动哪一个。

## 第一刀是路由级的

路由那一节开篇就把这刀说明了：

> Every Vue file inside the [`app/pages/`](https://nuxt.com/docs/4.x/directory-structure/app/pages) directory creates a corresponding URL (or route) that displays the contents of the file. By using dynamic imports for each page, Nuxt leverages code-splitting to ship the minimum amount of JavaScript for the requested route.

关键词 **dynamic imports**：每个页面文件是一个动态导入，所以访问某条路由时只发这条路由需要的 JS。入门那一节把它列为框架自带能力之一（"Nuxt automatically splits your code into smaller chunks"），服务端引擎那一节也重复了一次（"Automatic code-splitting and async-loaded chunks"）。

这一刀是默认的，不用你写任何东西——**前提是你的内容真的分布在多个页面文件里**。全站的界面都堆在 `app.vue` 里，路由级分割就没有可切的东西。

## 后面四刀各有各的写法

| 写法 | 官方原文 | 出几个 chunk |
| --- | --- | --- |
| `Lazy` 前缀 | 组件名加 `Lazy` 前缀做动态导入，"delay loading the component code until the right moment" | 该组件单独一份 |
| 全局注册 | `components.global: true` "will create async chunks for all your components"；`global` 目录或 `.global.vue` 后缀 | **每个全局组件一份** |
| `isAsync` | "This flag indicates, component should be loaded async (with a separate chunk) regardless of using Lazy prefix or not." | 强制单独一份 |
| 岛屿里的 `nuxt-client` | "Only its chunk is shipped to the client; the rest of the island remains static." | 只有交互组件那份 |

`isAsync` 是 `addComponent` 的选项，默认 `false`。它的作用对象是"注册进来的组件"，官方写明它**不看 `Lazy` 前缀**——注册时就定成异步分包。`chunkName` 也是同一处选项，"Chunk name for the component. If not provided, it will be generated from the component name."，Kit 的解析示例里能看到给它传 `'headlessui/combobox'` 这类分组合名。

还有一句要单独记住的，来自服务端岛屿那一节：使用岛屿"can significantly increase the number of chunks generated at build time"——**chunk 数量本身会涨**。切得多不等于省得多。

## 切出 chunk 不等于少加载

组件目录那一节写了一句很直接的提醒，位置就在"延迟水合"小节的开头：

> Lazy components are great for controlling the chunk sizes in your app, but they don't always enhance runtime performance, as they still load eagerly unless conditionally rendered.

所以 `Lazy` 前缀要配条件渲染才有意义。官方给 `Lazy` 写的示例正是 `<LazyMountainsList v-if="show" />` 配一个 `v-if="!show"` 的按钮——**组件平时不在 DOM 里，需要时才挂载**，那份 chunk 才真的不会在首屏被取走。没有 `v-if` 的懒组件，切了文件也照样立刻下载。

同理，动态组件名的写法有一条硬约束，官方用 `::important` 标出：用 `resolveComponent` 处理动态组件时，只能传组件名，而且必须是字面量字符串，不能是变量也不能包含变量——**这个字符串是在编译步骤静态分析的**。你没法用变量名去"动态决定加载哪个 chunk"。

## 关掉了会怎样

官方明确写了分割默认开、推荐大多数应用保持开启，然后给了唯一一条关闭写法：

```ts
export default defineNuxtConfig({
  vite: {
    $client: {
      build: {
        rolldownOptions: {
          output: {
            codeSplitting: false,
          },
        },
      },
    },
  },
})
```

注意它写在 `vite.$client` 下面——只影响客户端包。后果官方也写了：**所有 JS 打进单个文件**。评价是原话：

> This ships all JavaScript in a single file. It is rarely beneficial — it usually increases the initial download, even on slow connections — so only disable code-splitting if you have measured that it helps your case.

一句"通常增加首屏下载量"，加上"只有你测过确实有帮助才关"。这一项不是性能优化开关。

## 文件名是另一件事

chunk 名和文件名分开。webpack 侧的 `build.filenames`（`webpack.filenames`）里 `chunk` 是个函数，官方示例把花哨的 chunk 名换成数字 id：

```ts
webpack: {
  filenames: {
    chunk: ({ isDev }) => (isDev ? '[name].js' : '[id].[contenthash].js'),
  },
}
```

同一节有一条 `::callout` 警告：**生产环境用不带 hash 的文件名要当心**，多数浏览器会缓存住这个资源、首次加载时察觉不到变化。这条和 072 那篇讲的构建重启无关，但它解释了为什么产物文件名默认带 hash。

切得越细，取 chunk 失败的概率越高。官方为此内置了处理：路由跳转时某个 chunk 下载失败会做一次硬重载（新部署会让旧的 hash chunk URL 失效），行为由 `experimental.emitRouteChunkError` 控制，可以设 `false` 关掉或 `manual` 自己接——这一项细节留给错误处理分区。

## 自查

- 默认那一刀是路由级的：每个页面文件一个动态导入；内容都堆在 `app.vue` 就没有可切的。
- 另外四刀各有出处：`Lazy` 前缀、全局组件（每个一份）、`addComponent` 的 `isAsync`、岛屿里带 `nuxt-client` 的组件。
- `Lazy` 不配 `v-if` 之类条件渲染就仍然急加载；`resolveComponent` 只能传字面量组件名。
- 关分割的唯一写法是 `vite.$client.build.rolldownOptions.output.codeSplitting: false`，官方评价是通常更差。
- chunk 名可给 `chunkName`；产物文件名里的 `chunk` 是函数，生产不带 hash 有缓存警告。

## 下一篇

下一篇 440 讲构建这一侧的缓存：`experimental.buildCache` 按配置与源文件的哈希缓存构建产物，但它默认关闭、只覆盖 `srcDir` 与 `serverDir` 的源码，而且**Nitro 每次都会重新构建**。哪九个文件一动就强制全量重建，下一篇一一对出来。

## 本文依据

- 每个页面文件对应一条路由、"By using dynamic imports for each page ... ship the minimum amount of JavaScript for the requested route"、代码分割默认开启与 `codeSplitting: false` 的完整配置示例及其评价原文（`getting-started/routing`）
- `Lazy` 前缀动态导入与 `<LazyMountainsList v-if="show" />` 示例、全局注册"though not recommended ... create async chunks for all your components"、`global` 目录与 `.global.vue` 及"each global component is rendered in a separate chunk"、`resolveComponent` 的 `::important` 静态分析禁令、"Lazy components ... still load eagerly unless conditionally rendered"（`directory-structure/app/components`）
- `addComponent` 选项表中的 `isAsync`、`chunkName` 原文与默认值、Kit 解析示例中的 `chunkName: 'headlessui/combobox'`（`api/kit/components`、`api/kit/resolving`）
- 岛屿的 `nuxt-client` 只发送自身 chunk、"Using islands can significantly increase the number of chunks generated at build time"（`guide/concepts/server-components`）
- Code splitting 列为自带能力（`getting-started/introduction`）；"Automatic code-splitting and async-loaded chunks"（`guide/concepts/server-engine`）
- `webpack.filenames` 的 `chunk` 函数与 `[id].[contenthash].js` 示例、非 hash 文件名的 `::callout` 警告（`api/nuxt-config`）
- chunk 下载失败的硬重载与 `experimental.emitRouteChunkError`（`getting-started/error-handling`、`guide/going-further/experimental-features`）
