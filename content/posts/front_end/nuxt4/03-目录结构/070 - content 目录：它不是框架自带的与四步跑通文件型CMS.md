---
url: /posts/fe5d1b847a5ddd0b16e573dcfe8c9622/
title: content 目录：它不是框架自带的，四步跑通一个文件型 CMS
date: 2026-11-04T10:00:00+08:00
updated: 2026-11-04T10:00:00+08:00
author: cmdragon

summary:
  摘要：content/ 目录用来为你的应用创建一个基于文件的 CMS，但要点是它由 Nuxt Content 模块驱动而不是框架自带——官方在这一页写的是 Nuxt Content 读取你项目里的 content/ 目录并解析 .md、.yml、.csv 与 .json 文件来创建基于文件的 CMS，并给了四项能力：用内置组件渲染内容、用类 MongoDB 的 API 查询内容、用 MDC 语法在 Markdown 文件里使用你的 Vue 组件、自动生成导航。启用方式官方给的是"一条命令同时安装 @nuxt/content 模块并把它加进 nuxt.config.ts"：npx nuxt module add content。渲染内容的做法官方写得很具体：添加一个 catch-all 路由，在里面用 useAsyncData(route.path, () => queryCollection('content').path(route.path).first()) 按当前路径取到那条内容，再用 <ContentRenderer v-if="page" :value="page" /> 渲染出来——这意味着"为什么我的 markdown 放进 content/ 却不显示"通常不是目录问题，而是你还没有这条 catch-all 页面。

categories:
  - 前端开发

tags:
  - nuxt4
  - 内容
  - content模块
  - markdown
  - 目录结构

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


050 那张目录地图上，`content/` 是唯一一个被官方标注"**由模块启用**"的位置。这一篇就把这句话拆开：**装什么、放什么、怎么让它显示出来。**

## 它由谁驱动：模块，不是框架

> `content/` 目录用于为你的应用创建一个基于文件的 CMS。
>
> **Nuxt Content 读取你项目里的 `content/` 目录，并解析 `.md`、`.yml`、`.csv` 与 `.json` 文件，为你的应用创建一个基于文件的 CMS。**

**主语是 Nuxt Content 这个模块。** 050 那句"由 Nuxt Content 模块启用"在这里得到落实：**你项目里没有 `content/` 目录是完全正常的**（对照 `app/`、`server/` 那些无条件生效的位置）。

四种被解析的扩展名，就这一句列全了：**`.md`、`.yml`、`.csv`、`.json`**。

官方给的四项能力（原样四条）：

> - 用内置组件渲染你的内容。
> - 用一个**类 MongoDB 的 API** 查询你的内容。
> - **用 MDC 语法在 Markdown 文件里使用你的 Vue 组件。**
> - 自动生成你的导航。

**第三条是 Nuxt 生态相对其它静态站生成器的关键差别**：Markdown 里可以直接用你自己的组件（062 那套组件命名规则在这里同样适用）。

## 启用：官方给的是"一条命令做两件事"

> 用一条命令在你的项目里安装 `@nuxt/content` 模块，**并把它加进你的 `nuxt.config.ts`**：

```bash [Terminal]
npx nuxt module add content
```

**这条命令就是 045 那条 `nuxt module add` 的实例**（官方在 045 抄过它的三个动作：装成依赖、写进 `package.json`、登记进 `nuxt.config` 的 `modules`）。

**所以如果你只写了 `mkdir content/` 而没有跑这一步，什么都不会发生**——这就是 070 开头那句"由模块启用"的实际后果。

## 放内容：一个文件就够

> 把你的 markdown 文件放进 `content/` 目录：

```md [content/index.md]
# Hello Content
```

> **这个模块会自动加载并解析它们。**

**到这一步为止，你都还没有任何可见的页面。** 这是这一节最该说清的：模块"加载并解析"≠"给你一条路由"。

## 渲染：官方给的是一条 catch-all 页面

> 要渲染内容页面，添加一个[catch-all 路由]，在其中使用 `<ContentRenderer>` 组件：

```vue [app/pages/[...slug].vue]
<script lang="ts" setup>
const route = useRoute()
const { data: page } = await useAsyncData(route.path, () => {
  return queryCollection('content').path(route.path).first()
})
</script>

<template>
  <div>
    <header><!-- ... --></header>

    <ContentRenderer
      v-if="page"
      :value="page"
    />

    <footer><!-- ... --></footer>
  </div>
</template>
```

**把这段拆成你已经认识的零件，它其实一点不神秘：**

| 这一行 | 你在哪一篇见过 |
| --- | --- |
| `app/pages/[...slug].vue` | 058：catch-all 匹配该路径下所有路由，参数是数组 |
| `useRoute()` + `route.path` | 057、058：读当前路径 |
| `await useAsyncData(key, handler)` | 028、210 之后：`useAsyncData` 的第一个参数是缓存键，这里用 `route.path` |
| `v-if="page"` | 028：取数结果为空时先判空 |
| `<ContentRenderer :value="page" />` | 模块自己的组件（`content.nuxt.com` 的文档） |
| `queryCollection('content')` | 模块提供的那套"类 MongoDB 查询 API" |

**`useAsyncData(route.path, ...)` 这个写法值得多看一眼**：键就是路径，所以**换页面路径 = 换缓存键**（210-249 那段会展开键与复用的规则）。

**这段代码也回答了一个高频疑问："我的 markdown 放进 `content/` 了，为什么浏览器打开 `/about` 是 404？"** 因为**这条 catch-all 页面是你必须自己写的**——官方在这页给的就是它，而不是"模块自动生成了路由"。

## 更深的内容能力：官方指向站外

> （tip）前往 content.nuxt.com 以了解更多 Content 模块的特性，例如**如何构建查询**，以及**如何用 MDC 语法在你的 Markdown 文件里使用 Vue 组件**。

**官方在这一页只到"能查、能用组件"这一层，具体 API 归 `content.nuxt.com`。** 所以这一系列里关于 `content/` 就讲到这里——**别指望在 Nuxt 主文档里找到 `queryCollection` 的全部方法列表。**

## 自查

- [ ] `content/` 是被框架扫描的目录吗？谁在读它？
- [ ] 它解析哪四种文件扩展名？
- [ ] 只把 markdown 放进去，浏览器能访问到页面吗？还需要什么？
- [ ] 官方给的取数写法里，缓存键是什么？为什么用它合适？
- [ ] "Markdown 里用我自己的组件"这件事，官方给的名字是什么？

## 下一篇

071 讲 `node_modules/`：官方在这一页只写了两句——它由包管理器创建、应加进 `.gitignore`。真正值得读的是它引出的两条：`modulesDir` 这个配置项与"自动导入不会为 `node_modules` 里的文件启用"这条性能取舍。

## 本文依据

- `directory-structure/content`（标题：使用 content/ 目录为你的应用创建基于文件的 CMS；Nuxt Content 读取项目里的 content/ 目录并解析 .md、.yml、.csv 与 .json 文件创建基于文件的 CMS；四项能力：用内置组件渲染内容、用类 MongoDB 的 API 查询内容、用 MDC 语法在 Markdown 文件里使用你的 Vue 组件、自动生成导航；Enable Nuxt Content：用一条命令在项目安装 @nuxt/content 模块并把它加进 nuxt.config.ts，命令为 npx nuxt module add content；Create Content：把 markdown 文件放进 content/ 目录（示例 content/index.md 内容 # Hello Content）并说明模块会自动加载并解析它们；Render Content：要渲染内容页面需添加一个 catch-all 路由并使用 <ContentRenderer> 组件，示例 app/pages/[...slug].vue 中 const route = useRoute() 与 const { data: page } = await useAsyncData(route.path, () => queryCollection('content').path(route.path).first())、模板里 header + <ContentRenderer v-if="page" :value="page" /> + footer；Documentation tip 指向 content.nuxt.com 并提到如何构建查询与用 MDC 语法在 Markdown 中使用 Vue 组件）
- `directory-structure`（对照用：content/ 一节原文为"由 Nuxt Content 模块启用，用于以 Markdown 文件为应用搭建基于文件的 CMS"）
- `api/commands/module`（对照用：nuxt module add 的三个动作——装依赖、写 package.json、登记进 nuxt.config 的 modules，见 045）
- `directory-structure/app/pages`（对照用：Catch-all Route 一节，见 058）
- `getting-started/data-fetching`（对照用：useAsyncData 的第一个参数是用于缓存的键，见 028）
