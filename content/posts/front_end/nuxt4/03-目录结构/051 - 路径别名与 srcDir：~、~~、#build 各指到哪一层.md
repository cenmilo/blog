---
url: /posts/95318d75cddb231d8505d6ab129fa2ae/
title: 051 - 路径别名与 srcDir：~、~~、#build 各指到哪一层
date: 2026-10-29T09:00:00+08:00
updated: 2026-10-29T09:00:00+08:00
author: cmdragon

summary:
  摘要：Nuxt 默认给你十个路径别名，官方在 nuxt.config 的 alias 一节把默认值整个列了出来：~ 与 @ 指向 <rootDir>/app，~~ 与 @@ 指向 <rootDir>，#shared 指向 <rootDir>/shared，#server 指向 <rootDir>/server，assets 指向 <rootDir>/app/assets，public 指向 <rootDir>/public，#build 指向 <rootDir>/.nuxt，#internal/nuxt/paths 指向 <rootDir>/.nuxt/paths.mjs。这一节有两条官方明确写出的限制：在 webpack 上下文里（图片源、CSS，但不含 JavaScript）你必须用 ~ 前缀来访问你的别名；这些别名会自动加进生成的 TypeScript 配置（.nuxt/tsconfig.app.json、tsconfig.server.json 等）从而拿到类型与路径补全，要继续扩展就写在 alias 或 typescript.tsConfig 里。srcDir 的默认值在 Nuxt 4 是 "app"、在 compatibilityMode 为 3 的 Nuxt 3 是 "."，相对路径以 rootDir 为基准；而 serverDir 的默认值是 <rootDir>/server、相对路径同样相对 rootDir（不是相对 srcDir）。dir 那一节一共九个子项（app、assets、layouts、middleware、modules、pages、plugins、public、shared）且官方劝了一句：非必要保持默认。

categories:
  - 前端开发

tags:
  - nuxt4
  - 别名
  - srcdir
  - 路径
  - 配置

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

050 那张地图回答放哪儿。这一篇回答另外两个问题，代码里怎么引用它，别名，以及这些位置能不能改名，`srcDir`、`serverDir`、`dir`。

十个默认别名，先把它背下来一次。官方在 `alias` 一节给的默认值，是一份完整的 JSON。

```json
{
  "~": "/<rootDir>/app",
  "@": "/<rootDir>/app",
  "~~": "/<rootDir>",
  "@@": "/<rootDir>",
  "#shared": "/<rootDir>/shared",
  "#server": "/<rootDir>/server",
  "assets": "/<rootDir>/app/assets",
  "public": "/<rootDir>/public",
  "#build": "/<rootDir>/.nuxt",
  "#internal/nuxt/paths": "/<rootDir>/.nuxt/paths.mjs"
}
```

这张表里三组信息值得单独读。

一，`~` 与 `@` 指向 `app`，`~~` 与 `@@` 指向根。这就是 Nuxt 4 那个 `app/` 目录，050 讲过，在路径上的体现，`~/pages/index.vue` 里的 `~` 是 `app/`，不是项目根。少打一个波浪号就指错一层。

二，`#shared` 与 `#server` 各指向 050 里那两个目录。050 说 `shared/` 是两侧共用，而它就是靠 `#shared` 这个别名被引用的。

三，`#build` 指向 `.nuxt`，`#internal/nuxt/paths` 指向 `.nuxt/paths.mjs`。这两行是 052 的入口，生成目录不是只能去浏览器里看，它有别名可以直接写进 import。

两条官方写明的限制。

限制一，webpack 上下文里必须带 `~`。官方原话，注意，在一个 webpack 上下文里，图片源、CSS，但不含 JavaScript，你必须通过加 `~` 前缀来访问你的别名。

这一条的适用范围要说准，它讲的是 webpack builder 下的图片源与 CSS，JS 不在内。默认构建器是 Vite，088 讲 `builder`，所以这条不是天天遇到，但一旦切到 webpack 构建器而资源路径解析不了，这是第一个要看的地方。015 提过 `dir.assets` 的默认别名写作 `~assets`，正是这条 `~` 前缀写法。

限制二，其实是好消息，别名会自动进类型配置。官方原话，注意，这些别名会被自动加入生成的 TypeScript 配置，`.nuxt/tsconfig.app.json`、`.nuxt/tsconfig.server.json` 等，这样你就能拿到完整的类型支持与路径自动补全。如果你还需要进一步扩展这些生成配置提供的选项，请确保加在这里，或加在 `nuxt.config` 的 `typescript.tsConfig` 属性里。

这半句解决了一个常见犹豫，加个别名要不要顺手改 `tsconfig.json`。官方的答案是不用改，它自己会进生成配置，要扩展则写 `alias` 或 `typescript.tsConfig`。而 036 里那条别手改 tsconfig 的规则，在这里正好是它的正面说法。

自定义别名的官方写法。官方给的例子有个细节值得注意，它用 `fileURLToPath`。

```ts
import { fileURLToPath } from 'node:url'

export default defineNuxtConfig({
  alias: {
    'images': fileURLToPath(new URL('./assets/images', import.meta.url)),
    'style': fileURLToPath(new URL('./assets/style', import.meta.url)),
    'data': fileURLToPath(new URL('./assets/other/data', import.meta.url)),
  },
})
```

用法是模板里写 `~images/main-bg.jpg`、脚本里 `import data from 'data/test.json'`。

```html
<template>
  <img src="~images/main-bg.jpg">
</template>

<script>
import data from 'data/test.json'
</script>
```

为什么用 `new URL(..., import.meta.url)` 而不是直接写相对字符串。015 讲 ESM 时抄过官方那句 `require`、`__filename`、`__dirname` 在 ESM 模块里不可用，应当用 `import()` 与 `import.meta.filename` 替代，这个写法就是那条规则在配置文件里的落地，`import.meta.url` 配 `fileURLToPath` 换成绝对路径。

`srcDir`，默认是 `app`，而这是 4 与 3 的分界之一。官方定义是，定义你 Nuxt 应用的源目录。如果指定的是相对路径，它相对 `rootDir`。类型是 `string`，默认是 `"app"`，Nuxt 4，以及 `"."`，Nuxt 3 且 `compatibilityMode: 3`。

这行默认值就是文档里所有 `app/app.vue`、`app/pages/` 那种写法的来源。025、032、046 都提过这个前缀，它不是排版习惯而是 `srcDir` 的默认值，046 那条文件相对 `srcDir` 写出，`srcDir` 默认为 `app/`，用的正是同一个配置项。

官方给的例子也直接展示了期望的目录形态。

```ts
export default defineNuxtConfig({
  srcDir: 'app/',
})
```

官方说这要求以下目录结构。

```bash
-| app/
---| assets/
---| components/
---| composables/
---| layouts/
---| middleware/
---| pages/
---| plugins/
---| utils/
---| app.config.ts
```

这张清单值得和 050 对照读，它是 `srcDir` 那一层应用侧的目录，而 050 的定义里还多一个 `error.vue`。两处清单不完全一致，我照原文各引各的，不合并成一份标准答案。

`rootDir` 与 `serverDir`，两个基准不一样。官方对 `rootDir` 的说明带着一条可以不改写就用的性质，定义你应用的根目录。这个属性可以被覆盖，例如运行 `nuxt ./my-app/` 会把 `rootDir` 设为从当前工作目录算起的 `./my-app/` 的绝对路径。通常不需要配置这个选项。默认是 `/<rootDir>`。

`serverDir` 则是，定义你 Nuxt 应用的服务端目录，Nitro 的路由、中间件与插件就放在那里。如果指定的是相对路径，它相对于你的 `rootDir`。默认是 `/<rootDir>/server`。

这里有一个很容易弄错的点，我用一句话标出来，`srcDir` 与 `serverDir` 的相对路径基准都是 `rootDir`。也就是说服务端那一层不在 `app/` 里面，050 那张地图上 `server/` 与 `app/` 是并列的，这里从配置项的基准上再确认一次。

084 会讲到 `watch` 的基准是 `srcDir`，跟这两个不同。这类同名配置不同基准的坑，只能逐个查，不能靠类推。

`dir`，九个子项，和官方那句劝。官方定义这一节时的措辞很有态度，自定义 Nuxt 使用的默认目录结构。除非必要，最好保持默认。

它一共九个子项，默认值如下。`app` 默认 `"app"`，无说明。`assets` 默认 `"app/assets"`，资源目录，在你的构建里被别名为 `~assets`。`layouts` 默认 `"app/layouts"`，布局目录，其中每个文件会被自动注册为 Nuxt 布局。`middleware` 默认 `"app/middleware"`，中间件目录，其中每个文件会被自动注册为 Nuxt 中间件。`modules` 默认 `"modules"`，模块目录，其中每个文件会被自动注册为 Nuxt 模块。`pages` 默认 `"app/pages"`，被处理以自动生成你应用页面路由的目录。`plugins` 默认 `"app/plugins"`，插件目录，其中每个文件会被自动注册为 Nuxt 插件。`public` 默认 `"public"`，包含你的静态文件的目录，可以通过 Nuxt 服务器直接访问，并在应用生成时被复制进你的 `dist` 文件夹。`shared` 默认 `"shared"`，共享目录，该目录在应用与服务端之间共享。

三处读出来的信息。

一，这些默认值全都带着 `app/` 前缀。`dir.pages` 的默认是 `"app/pages"` 而不是 `"pages"`，因为它基准是项目根，而 `srcDir` 恰好在 `app`。005 讲的约定在配置里就是这么显式存在的。

二，四个子项的定义里都写着自动注册，layouts、middleware、modules、plugins。这句话和 050 那些目录定义是同一件事的两种说法，也是 062、059、060、061 那几篇的主题。

三，`dir.public` 的说明里出现了一个新词，`dist`。官方写在应用生成时被复制进你的 `dist` 文件夹，这与 037 讲的 `.output`、040 的 cleanup 清单里那条 `dist` 是同一族词。三份文档各自用词，我在 040 与 049 里都标过官方没有把它们互相定义清楚。你实际要记的是，`public/` 的内容会被原样复制进产物。

另外注意这一节里没有 `components`、`composables`、`utils`，也没有 `server`。`dir` 管的就是上面这九项，组件等的目录规则在 062 与 `imports` 和 `components` 配置里，服务端目录另有 `serverDir`。

`buildDir`，把 `.nuxt` 改成不带点的那个。050 末尾我说 `.nuxt` 不在源码结构那页，它住在这里。定义你已构建的 Nuxt 文件被放置的目录。许多工具假定 `.nuxt` 是一个隐藏目录，因为它以 `.` 开头。如果那会造成问题，你可以用这个选项来避免它。默认是 `/<rootDir>/.nuxt`。

官方给的例子是把它改成不带点的名字。

```ts
export default defineNuxtConfig({
  buildDir: 'nuxt-build',
})
```

这条与 040 那句 cleanup 清的是你的构建目录，也就是 `.nuxt` 除非 `buildDir` 另有指定，是同一条规则的两端。改名之后，`.nuxt` 这个约定就不再成立，凡是硬写 `.nuxt` 的地方都得跟着改。而 051 的别名表里 `#build` 指向 `<rootDir>/.nuxt`，改了 `buildDir`，`#build` 的含义也跟着变。这句是我按两处默认值连起来的推论，官方没在这里明写。

## 自查

- [ ] `~/utils/foo` 与 `~~/shared/foo` 分别落在哪个目录？
- [ ] 什么上下文里必须用 `~` 前缀访问别名？官方排除的是什么？
- [ ] 新加的别名要不要手动写进 `tsconfig.json`？官方给的入口是哪两处？
- [ ] `srcDir` 与 `serverDir` 的相对路径基准分别是什么？`dir.pages` 的默认值为什么带着 `app/`？
- [ ] 把 `buildDir` 改成 `nuxt-build` 之后，哪些地方要跟着改？

## 下一篇

052 打开那个生成目录：`.nuxt` 是谁的产物、「虚拟文件系统」意味着什么、为什么官方用 warning 级别写着「不要动里面的文件」，以及 DevTools 的 Virtual Files 标签为什么值得你点进去看一眼。
