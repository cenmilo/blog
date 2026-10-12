---
url: /posts/dc5a08b7cb2369d9b249474de80d6c8c/
title: webpack 的 loaders：十三个加载器的定制入口
date: 2027-04-14T11:00:00+08:00
updated: 2027-04-14T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`webpack.loaders` 的说明是一句：**"Customize the options of Nuxt's integrated webpack loaders."**——它定制的是 **Nuxt 已集成进来的那批加载器**的选项。下面有 13 个：`css`、`cssModules`、`esbuild`、`file`、`fontUrl`、`imgUrl`、`less`、`pugPlain`、`sass`、`scss`、`stylus`、`vue`、`vueStyle`。默认值里有几条会直接影响你怎么读产物：**`css` 与 `cssModules` 都是 `esModule: false`＋`importLoaders: 0`**；**`cssModules.modules.localIdentName` 默认 `"[local]_[hash:base64:5]"`**（这就是 CSS Module 类名的形状）；**`file`/`fontUrl`/`imgUrl` 三项形状完全相同，`esModule: false`＋`limit: 1000`**；**`sass` 的默认是 `sassOptions.indentedSyntax: true`，而 `scss`/`less`/`stylus`/`vueStyle` 都是 `sourceMap: false`**；`vue` 下面还挂着 `compilerOptions`、`propsDestructure: true` 与那张五标签的 `transformAssetUrls` 表。

categories:
  - 前端开发

tags:
  - nuxt4
  - webpack
  - 加载器
  - css-modules
  - 预处理器
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

CSS Modules 的类名在产物里长成 `_title_1a2b3`，不是你写的 `.title`。

那个形状来自一处默认值；而在 webpack 这一侧，这类"由加载器决定"的行为有十三处。

## 十三个加载器与它们的默认值

`### loaders` 的说明一句：**"Customize the options of Nuxt's integrated webpack loaders."** 下挂 13 个 `####` 项：

| 加载器 | 官方给的默认 | 更深的选项在哪 |
| --- | --- | --- |
| `css` | `esModule: false`、`importLoaders: 0`，另有 `url.filter`（`function`，无默认） | See 指向 css-loader |
| `cssModules` | 同上两项，另加 `modules.localIdentName: "[local]_[hash:base64:5]"` | See 指向 css-loader |
| `esbuild` | `target: "esnext"`、`jsxFactory: "h"`、`jsxFragment: "Fragment"`、`tsconfigRaw: {}` | See 指向 esbuild-loader |
| `file` | `{ "esModule": false }`，子项 `esModule: false`、`limit: 1000` | See 指向 file-loader 选项 |
| `fontUrl` | 与 `file` 完全同形：`esModule: false`、`limit: 1000` | 同上 |
| `imgUrl` | 同上 | 同上 |
| `less` | `{ "sourceMap": false }` | See 指向 less-loader |
| `pugPlain` | **无默认值、无类型** | See 指向 pug 的 options 参考 |
| `sass` | `{ "sassOptions": { "indentedSyntax": true } }`（子项 `sassOptions.indentedSyntax` 为 `boolean`、默认 `true`） | See 指向 sass-loader |
| `scss` | `{ "sourceMap": false }` | 同上 |
| `stylus` | `{ "sourceMap": false }` | See 指向 stylus-loader |
| `vue` | `compilerOptions`（`object`）、`propsDestructure: true`、`transformAssetUrls` 五标签表 | See 指向 vue-loader |
| `vueStyle` | `{ "sourceMap": false }` | （未给 See 链接） |

四组值得单独说。

## 第一组：css 与 cssModules 只差一项

两项的默认值几乎相同（`esModule: false`、`importLoaders: 0`、`url.filter`），**唯一多出来的是 `cssModules.modules.localIdentName`，默认 `"[local]_[hash:base64:5]"`**。

这一行解释了你看到的一切：`[local]` 是原始类名、`[hash:base64:5]` 是五位 base64 哈希——**CSS Modules 的类名形状是可配的，而默认形状就是"原名_短哈希"**。

`importLoaders: 0` 是另一条容易被忽略的：它指的是 `@import` 之后的资源还要再过几层加载器，默认 **0**（即不再过）。**这一项与 `css` 那节说的"按扩展名猜预处理器"是两回事**，前者是 loader 层数、后者是 Nuxt 选哪个预处理器。

`url.filter` 只有类型 `function`、**没有默认值也没有签名说明**——想改 URL 处理规则，本节给不了参数形状。

## 第二组：file / fontUrl / imgUrl 三项一模一样

三个加载器**默认 JSON 都是 `{ "esModule": false }`，子项都是 `esModule: false` 与 `limit: 1000`**，See 都指向 file-loader 的选项文档。

差别只在用途命名上：`file` 是通用文件、`fontUrl` 管字体、`imgUrl` 管图片。**三处都有 `limit: 1000`**——这就是那个"资源小于多少字节就内联"的阈值（file-loader 侧语义），**单位与内联形式本节没写**。

要改一处不影响另两处，就得分别设：

```ts
export default defineNuxtConfig({
  webpack: {
    loaders: {
      imgUrl: { limit: 2048 },
    },
  },
})
```

**上面这段是本系列按默认值形状给的示例，官方在这三个子项下都没给 Example。**

## 第三组：预处理器四类，默认值分两种形状

`less`、`scss`、`stylus`、`vueStyle` 四项默认都是 `{ "sourceMap": false }`；**只有 `sass` 不同**：

```json
{
  "sassOptions": {
    "indentedSyntax": true
  }
}
```

`sass` 与 `scss` 分家正是这个原因——`indentedSyntax: true` 表示缩进式语法（`.sass`），`scss` 走花括号语法。这与样式页那句"SCSS 与 SASS 的 `additionalData` 结尾符号不同、两份不能共用"是同一套区分的两处体现。

还有一条在别处：这些 loader 要能用，**预处理器本体得自己装**（样式页与 `css` 那节都写了 "still need to install the required loader"），**本节的默认值不等于依赖已就位**。

## 第四组：vue 这个 loader 里带着编译选项

`loaders.vue` 下有三项，其中两项在 Vite 侧也有同名对应：

| 项 | 类型 | 默认 |
| --- | --- | --- |
| `compilerOptions` | `object` | 未列 |
| `propsDestructure` | `boolean` | `true` |
| `transformAssetUrls` | `object` | 五标签表（video 的 `src`/`poster`、source/img 的 `src`、image/use 的 `xlink:href`+`href`） |

**`propsDestructure` 默认 `true` 与 `vite.vue.features.propsDestructure`、顶层 `vue.propsDestructure` 三处同值**，`transformAssetUrls` 那张五标签表也与 Vite 侧那份**逐字相同**。也就是说：**同一类模板编译选项在 webpack 与 Vite 两条链上各有一份入口**，换构建器时得两边都检查——**官方没有写"改一处两边同时生效"**。

`vueStyle` 是唯一没有 See 链接的一项，默认 `sourceMap: false`；**它对应哪个 loader、还有哪些可选项，本节没写**。

## 自查

- `webpack.loaders` 定制的是 **Nuxt 集成进来的**加载器，共 13 项；多数只给一个 See 链接，选项形状在上游文档。
- `cssModules.modules.localIdentName` 默认 `[local]_[hash:base64:5]` —— CSS Modules 类名的形状在这里改。
- `css`/`cssModules` 的 `importLoaders` 默认 `0`；`url.filter` 只有类型无默认无签名。
- `file`/`fontUrl`/`imgUrl` 三项默认完全相同（`esModule: false`、`limit: 1000`），互不影响，要分别设。
- `sass` 默认 `sassOptions.indentedSyntax: true`，`less`/`scss`/`stylus`/`vueStyle` 默认 `sourceMap: false`；预处理器与 loader 依赖仍需自装。
- `loaders.vue` 的 `propsDestructure`（`true`）与 `transformAssetUrls`（五标签表）在 Vite 侧各有一份同名同值入口，换构建器要两边检查；`vueStyle` 无 See 链接。

## 下一篇

下一篇 555 读 `webpack.filenames` 与 `webpack.optimization`：六个文件名函数都是 `function` 且无默认，而 `optimization` 的四项里 `minimize` 默认 `true`、`runtimeChunk` 默认 `"single"`、`splitChunks` 给了 `automaticNameDelimiter: "/"` 与 `chunks: "all"`。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`#### loaders` 全部 13 项与其下 `#####`/`######` 子项的说明句、See 链接与默认值：`css`/`cssModules` 的 `esModule: false`、`importLoaders: 0`、`url.filter`、`modules.localIdentName`；`esbuild` 四项；`file`/`fontUrl`/`imgUrl` 的 `{esModule:false}` 与 `limit: 1000`；`less`/`scss`/`stylus`/`vueStyle` 的 `{sourceMap:false}`；`sass` 的 `sassOptions.indentedSyntax: true`；`pugPlain` 只有 See；`vue` 的 `compilerOptions`/`propsDestructure: true`/`transformAssetUrls` 五标签表）
- https://nuxt.com/docs/4.x/getting-started/styling 与 https://nuxt.com/docs/4.x/directory-structure/app/assets（预处理器依赖要自装、SCSS 与 SASS 的 `additionalData` 结尾不同）
- https://nuxt.com/docs/4.x/api/nuxt-config（`### vite.vue` 与 `## vue` 中同名的 `propsDestructure` 与 `transformAssetUrls`，用于对照两条构建链）
