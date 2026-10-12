---
url: /posts/a6fbbae63cc68575728d477c29690857/
title: webpack 的 filenames 与 optimization：文件名与分包
date: 2027-04-15T09:00:00+08:00
updated: 2027-04-15T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：`webpack.filenames` 只有一句 **"Customize bundle filenames."** 加一句读 manifest 的引导，下面六个子项 `app`/`chunk`/`css`/`font`/`img`/`video` **类型全是 `function`、一个默认值都没给**；而这条提醒管着整节：**"Be careful when using non-hashed based filenames in production as most browsers will cache the asset and not detect the changes on first load."** `webpack.optimization` 是"配置 webpack optimization"，给了四项默认：**`minimize: true`（说明里注明开发环境默认就是关的）**、**`runtimeChunk: "single"`**、**`splitChunks.automaticNameDelimiter: "/"`**、**`splitChunks.chunks: "all"`**，另有 `minimizer`（"可以设成一组自定义插件数组"，无类型）与 `cacheGroups`（无类型无默认）。官方还把"把所有 CSS 抽成单文件"当作 **workaround** 给出，并当场写了不推荐的理由。

categories:
  - 前端开发

tags:
  - nuxt4
  - webpack
  - 分包
  - 文件名
  - 缓存
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

发新版本后，有用户点进旧页面白屏——因为他浏览器里那份 chunk 名字还在，内容却已经不在了。

这件事一半归文件名，一半归分包；webpack 这两节就是管它们的。

## filenames：六个函数，零默认值

`### filenames` 的记载只有两句：**"Customize bundle filenames."** 与 **"To understand a bit more about the use of manifests, take a look at webpack documentation."**

六个子项的表：

| 子项 | 类型 | 默认 |
| --- | --- | --- |
| `app` | `function` | 未列 |
| `chunk` | `function` | 未列 |
| `css` | `function` | 未列 |
| `font` | `function` | 未列 |
| `img` | `function` | 未列 |
| `video` | `function` | 未列 |

**六个全是 `function`、全都没有默认值。** 也就是说：**这套名字的默认形状本节没有给**，你只能从示例反推写法。官方唯一那条示例是"开发环境用可读名、生产带 contenthash"：

```ts
export default defineNuxtConfig({
  webpack: {
    filenames: {
      chunk: ({ isDev }) => (isDev ? '[name].js' : '[id].[contenthash].js'),
    },
  },
})
```

从这行能确证三件事：**函数收到一个对象、里面有 `isDev`**；返回值是 webpack 的模板字符串形态（`[name]`、`[id]`、`[contenthash]`）；**"dev 可读、生产带 hash"是官方认可的分工**。

紧跟的 `::callout` 就是这条分工的理由：

> Be careful when using non-hashed based filenames in production as most browsers will cache the asset and not detect the changes on first load.

**去掉 hash 的代价不是命名自由，是浏览器认不出变化。** 这与 517 讲的 `buildId`/manifest 是同一类问题的两个侧面：一个管"客户端怎么发现新构建"，一个管"产物文件名本身可不可辨"。

另两点文档未答：**`[name]` 这类模板能用的完整占位符清单**（本节只出现这三个）、**其余五个函数各自的默认模板**（连示例都没有）。

## extractCSS：把"单文件"标成 workaround

`filenames.css` 与分包常常一起想，官方在 `extractCSS` 那里正面处理了这件事。先给默认与机制：**`extractCSS` 默认 `true`**，说明是 "Enables Common CSS Extraction."，机制句 **"Using mini-css-extract-plugin under the hood, your CSS will be extracted into separate files, usually one per component. This allows caching your CSS and JavaScript separately."**

也可以传对象：

```ts
export default defineNuxtConfig({
  webpack: {
    extractCSS: true,
    // or
    extractCSS: {
      ignoreOrder: true,
    },
  },
})
```

然后是那段少见的"官方先给做法、再劝你别用"：

> If you want to extract all your CSS to a single file, there is a workaround for this. However, note that it is not recommended to extract everything into a single file. Extracting into multiple CSS files is better for caching and preload isolation. It can also improve page performance by downloading and resolving only those resources that are needed.

它给的 workaround 是往 `optimization.splitChunks.cacheGroups` 里塞一个 styles 组：

```ts
export default defineNuxtConfig({
  webpack: {
    extractCSS: true,
    optimization: {
      splitChunks: {
        cacheGroups: {
          styles: {
            name: 'styles',
            test: /\.(css|vue)$/,
            chunks: 'all',
            enforce: true,
          },
        },
      },
    },
  },
})
```

**读这段的正确姿势是：它示范的是"怎么改 cacheGroups"，不是"该这么做"。** 官方给的三点不推荐理由是缓存粒度、preload 隔离、按需下载。

## optimization：四项有默认

`### optimization` 的定位句是 "Configure webpack optimization."。四个子项：

| 子项 | 官方说明 | 类型 | 默认 |
| --- | --- | --- | --- |
| `minimize` | Set minimize to `false` to disable all minimizers. **(It is disabled in development by default).** | `boolean` | `true` |
| `minimizer` | You can set minimizer to a customized array of plugins. | 未列 | 未列 |
| `runtimeChunk` | （无说明句） | `string` | `"single"` |
| `splitChunks` | （无说明句） | 未列 | 未列 |

`splitChunks` 下三项：

| 子项 | 类型 | 默认 |
| --- | --- | --- |
| `automaticNameDelimiter` | `string` | `"/"` |
| `cacheGroups` | 未列 | 未列 |
| `chunks` | `string` | `"all"` |

两处要注意：

**`minimize` 的"默认"要看场景。** 列表值 `true`，说明句括号里补了"开发环境默认就是关的"。**读这一项不能只看列表那一行。**

**`runtimeChunk: "single"` 与 `splitChunks.chunks: "all"` 都没有说明句**，`automaticNameDelimiter` 默认 `"/"` 也一样（它决定自动生成的 chunk 名里那段分隔符）。**这三项合起来构成默认分包形状，但 Nuxt 侧没有解释它们的后果**——要理解语义得点 webpack 官方 optimization 那份文档（这就是这一节给的唯一去处）。

## 自查

- `filenames` 六个子项全是 `function`、零默认值；官方示例确证函数收 `{ isDev }`、返回 webpack 模板字符串；**非 hash 文件名在生产会让浏览器认不出变化**。
- `[name]`/`[id]`/`[contenthash]` 之外的可用占位符，本节没列。
- `extractCSS` 默认 `true`、可传对象（示例给了 `ignoreOrder: true`）；**"全部 CSS 合成一个文件"被官方明写为 workaround 且不推荐**，理由是缓存粒度、preload 隔离与按需下载。
- `optimization.minimize` 列表默认 `true` 但开发环境本身是关的；`runtimeChunk` 默认 `"single"`、`splitChunks.chunks` 默认 `"all"`、`automaticNameDelimiter` 默认 `"/"`，三项都无说明句。
- `minimizer` 与 `cacheGroups` 无类型无默认，形状要看 webpack 文档。

## 下一篇

下一篇 556 读 `esbuild` 节：它的定位是"配置 Nuxt 内部共用、并转交给 Vite 或 webpack 等其他构建器的 esbuild 选项"，下面四项（`target`/`jsxFactory`/`jsxFragment`/`tsconfigRaw`）与 `vite.esbuild`、`webpack.loaders.esbuild` 三处默认值完全相同。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`### filenames` 的两句记载、非 hash 文件名的 `::callout`、`chunk` 的 `({ isDev })` 示例与六个 `function` 子项；`### extractCSS` 的默认 `true`、mini-css-extract-plugin 机制句、`ignoreOrder` 示例与"single file 是 workaround／不推荐"整段及 cacheGroups 示例；`### optimization` 四项与 `splitChunks` 三项的说明句、类型与默认值）
- https://nuxt.com/docs/4.x/api/nuxt-config（对照项：`### vite.build`、`### vite.esbuild`、`#### loaders.esbuild` 的同名默认值）
