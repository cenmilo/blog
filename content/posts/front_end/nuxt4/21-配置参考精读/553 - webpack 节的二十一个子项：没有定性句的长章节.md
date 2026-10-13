---
url: /posts/ed265821f3931d4f73e1f161e56a6bc0/
title: 553 - webpack 节的二十一个子项：没有定性句的长章节
date: 2027-04-14T10:00:00+08:00
updated: 2027-04-14T10:00:00+08:00
author: cmdragon

summary:
  摘要：`webpack` 是配置参考里最长的一节（五百多行），但**它连一句定性都没有**——`## webpack` 标题下面直接就是第一个子项。这一节实际有 **21 个 `###` 级子项**（索引里写的"67 个"不对）：`aggressiveCodeRemoval`、`analyze`、`cssSourceMap`、`devMiddleware`、`experiments`、`extractCSS`、`filenames`、`friendlyErrors`、`hotMiddleware`、`loaders`、`optimization`、`optimizeCSS`、`plugins`、`postcss`、`profile`、`serverURLPolyfill`、`warningIgnoreFilters`（外加各层更深的项）。读法只能是**按"谁的选项"分三类**：Nuxt 自己定义的几项（有默认值与说明）、原样转交给 webpack 生态某个包/某份配置的几项（只有 See 链接）、以及只给类型不给说明的几项。

categories:
  - 前端开发

tags:
  - nuxt4
  - webpack
  - 构建
  - 配置
  - 加载器
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

换成 webpack 构建之后，你会发现这一节的信息量和 Vite 那节完全不是一个量级：二十来个子项，一半写着默认值，另一半只留一个链接。

先解决"怎么读"，再谈具体项。

## 前提：这一节要显式换构建器才有意义

`builder` 那节写得直白：**"By default, Vite is used, but you can switch to webpack, Rspack, or even provide a custom builder implementation."**，而且**换 webpack 要自己装 `@nuxt/webpack-builder`**。所以下面这一整节默认不参与你的构建——它是给"已经把 `builder` 设成 webpack"的项目准备的。

## 二十一个子项的清单

| `###` 子项 | 官方说明 | 类型 | 默认 |
| --- | --- | --- | --- |
| `aggressiveCodeRemoval` | Hard-replaces `typeof process`, `typeof window` and `typeof document` to tree-shake bundle. | `boolean` | `false` |
| `analyze` | 用 webpack 时 Nuxt 以 `webpack-bundle-analyzer` 可视化打包产物；设 `true` 开启，或传对象（webpack 侧选项 / vite 侧 rollup-plugin-visualizer 选项） | `object` | 三项 JSON |
| `cssSourceMap` | Enables CSS source map support (defaults to `true` in development). | `boolean` | `false` |
| `devMiddleware` | See webpack-dev-middleware for available options.（下含 `stats`） | 未列 | 未列 |
| `experiments` | Configure webpack experiments | 未列 | 未列 |
| `extractCSS` | Enables Common CSS Extraction. | `boolean` | `true` |
| `filenames` | Customize bundle filenames.（下含 6 项） | 未列 | 未列 |
| `friendlyErrors` | Set to `false` to disable the overlay provided by FriendlyErrorsWebpackPlugin. | `boolean` | `true` |
| `hotMiddleware` | See webpack-hot-middleware for available options. | 未列 | 未列 |
| `loaders` | Customize the options of Nuxt's integrated webpack loaders.（下含 13 项） | 未列 | 未列 |
| `optimization` | Configure webpack optimization.（下含 4 项） | 未列 | 未列 |
| `optimizeCSS` | OptimizeCSSAssets plugin options. Defaults to true when `extractCSS` is enabled. | `boolean` | `false` |
| `plugins` | Add webpack plugins. | `array` | 未列 |
| `postcss` | Customize PostCSS Loader. same options as postcss-loader options（下含 `postcssOptions.plugins`） | 未列 | 未列 |
| `profile` | Enable the profiler in webpackbar. It is normally enabled by CLI argument `--profile`. | `boolean` | `false` |
| `serverURLPolyfill` | The polyfill library to load to provide URL and URLSearchParams. | `string` | `"url"` |
| `warningIgnoreFilters` | Filters to hide build warnings. | `array` | 未列 |

（表里十七行是 `###` 级；再加各层更深的 `####`/`#####`/`######`，全节约二十一项以上——**索引原写的"67 个子项"没有依据**。）

## 三类读法

**第一类：Nuxt 自己定义、给了默认值的。** `aggressiveCodeRemoval`、`analyze`、`cssSourceMap`、`extractCSS`、`friendlyErrors`、`optimizeCSS`、`profile`、`serverURLPolyfill` 这八项是"能直接照默认值推理"的部分。

**第二类：只留一个 See，选项在别人的文档里。** `devMiddleware`（webpack-dev-middleware）、`experiments`（webpack 官方 experiments 页）、`hotMiddleware`（webpack-hot-middleware）、`optimization`（webpack 官方 optimization 页）、`postcss`（postcss-loader 选项）、`plugins`（示例里 `new webpack.DefinePlugin(...)`）。

这一类的共同点是：**Nuxt 不重新定义它们，字段形状跟随上游**。所以在这里查"能写什么"是查不到的，得点那个链接。

**第三类：只有类型或只有默认值。** `plugins`（`array` 无默认）、`warningIgnoreFilters`（`array` 无默认）、`loaders`（无类型无默认，但下面 13 项几乎每项都有默认）。

## 三处默认值容易读错

**`cssSourceMap` 的说明与默认冲突式并存。** 官方写 "Enables CSS source map support (**defaults to `true` in development**)."，但同一项的 `- **Default:**` 是 `false`。两句一起读只能理解成：**列表值是 `false`，dev 场景另有开关**——这与 `sourcemap` 那节"客户端仅 dev 开"的行为是同一类安排。

**`optimizeCSS` 也一样：说明句 "Defaults to true when `extractCSS` is enabled."，列表默认 `false`。** 而 `extractCSS` 自己默认是 `true`——**三者连起来的效果是：CSS 抽取默认开、抽取时 CSS 压缩跟着开，但配置里那一行仍写 `false`。** 这种"说明句给条件默认、列表给静态默认"的写法在本节出现两次，读任何一项都要把两句一起看。

**`analyze` 与顶层 `build.analyze` 是同一份默认值。** 两处 JSON 完全相同（`template: "treemap"`、`projectRoot: "/<rootDir>"`、`filename: "/<rootDir>/.nuxt/analyze/{name}.html"`），而 `build` 那节的示例写的是**顶层 `analyze:`**、`webpack` 这节的示例写的是 `webpack.analyze:`——**同一个功能给了两个位置，官方没解释关系**（`build` 那篇里我按"两种写法各验一次"处理）。

## 一条与文件名有关的风险

`filenames` 那节开头有句提醒，管着这一整节：**"Be careful when using non-hashed based filenames in production as most browsers will cache the asset and not detect the changes on first load."**

它给的示例正是"dev 用可读名、生产带 contenthash"的写法：

```ts
export default defineNuxtConfig({
  webpack: {
    filenames: {
      chunk: ({ isDev }) => (isDev ? '[name].js' : '[id].[contenthash].js'),
    },
  },
})
```

`filenames` 下六个子项（`app`/`chunk`/`css`/`font`/`img`/`video`）**类型都是 `function`、都没有默认值**，函数签名从示例看是收到 `{ isDev }` 的对象。**六个子项各自默认模板字符串是什么，本节没写。**

## 自查

- `## webpack` 没有定性句；它是 **webpack 构建器**的配置面，前提是把 `builder` 换成 webpack 并**显式安装 `@nuxt/webpack-builder`**。
- 全节约二十一项（索引"67 个"无据），按三类读：Nuxt 自定义带默认的八项、只给 See 的转交项、只有类型或只有默认值的项。
- 两处"说明句条件默认 vs 列表 `false`"要合着读：`cssSourceMap`（dev 为 true）、`optimizeCSS`（`extractCSS` 开时为 true，而 `extractCSS` 默认 `true`）。
- `analyze` 与 `build.analyze` 默认值完全相同、示例位置不同，官方未说明关系。
- `filenames` 六项都是 `function` 且无默认；官方警告**生产用不带 hash 的文件名会被浏览器缓存住**，示例给的写法是 dev 用 `[name].js`、生产用 `[id].[contenthash].js`。

## 下一篇

下一篇 554 单读 `webpack.loaders`：那十三个加载器各自的默认值（`css` 与 `cssModules` 的 `esModule: false`／`importLoaders: 0`、`cssModules.modules.localIdentName` 的 `[local]_[hash:base64:5]`、`file` 的 `limit: 1000`、`sass` 的 `indentedSyntax: true`、`scss`/`stylus`/`vueStyle` 的 `sourceMap: false`）。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## webpack` 全节结构与逐项记载：`aggressiveCodeRemoval`/`analyze`/`cssSourceMap`/`devMiddleware.stats`/`experiments`/`extractCSS`（含两段示例与"not recommended to extract everything into a single file"那段权衡）/`filenames` 与非 hash 名的 `::callout` 及 `chunk` 函数示例/`friendlyErrors`/`hotMiddleware`/`loaders`/`optimization`/`optimizeCSS`/`plugins` 的 DefinePlugin 示例/`postcss.postcssOptions.plugins` 默认 `{autoprefixer:{},cssnano:{}}`/`profile`/`serverURLPolyfill`/`warningIgnoreFilters`；`## builder` 的默认 Vite 与 webpack/rspack 需显式安装两句）
- https://nuxt.com/docs/4.x/api/commands/build 与 https://nuxt.com/docs/4.x/api/nuxt-config（`--profile` 与 `webpack.profile` 的对应、`build.analyze` 与 `webpack.analyze` 同一份默认 JSON）
