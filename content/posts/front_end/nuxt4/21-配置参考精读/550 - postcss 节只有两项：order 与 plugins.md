---
url: /posts/651f3b78b0bbaceeef74a5c57891788a/
title: postcss 节只有两项：order 与 plugins
date: 2027-04-13T10:00:00+08:00
updated: 2027-04-13T10:00:00+08:00
author: cmdragon

summary:
  摘要：`postcss` 这一节**连一句定性都没有**——`## postcss` 下面直接就是两个子项：`order`（"A strategy for ordering PostCSS plugins."，类型 `function`，**没有默认值**）与 `plugins`（"Options for configuring PostCSS plugins."，只有一个指向 PostCSS 文档的 See，**连 Type 都没写**）；`plugins` 下面点名两项：**`autoprefixer`**（"Plugin to parse CSS and add vendor prefixes to CSS rules."）与 **`cssnano`**（类型 `object`，指向它的配置文件选项文档）。真正有用的信息在样式那一页：**Nuxt 内置 postcss**，默认已经预配好四个插件——**postcss-import（改进 `@import`）、postcss-url（转换 `url()`）、autoprefixer（自动加厂商前缀）、cssnano（压缩与 purge）**，而且 `plugins` 是以**包名为键**来配的。

categories:
  - 前端开发

tags:
  - nuxt4
  - postcss
  - css
  - 插件
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

想给 PostCSS 加一个嵌套语法插件，翻到配置参考的 `postcss` 那节——只有两行，什么都没解释。

这一节的真实内容，一大半写在样式那一页。

## 这一节的完整记载

`## postcss` 标题下**没有任何说明句**，直接进子项：

| 层级 | 键 | 官方说明 | 类型 | 默认 |
| --- | --- | --- | --- | --- |
| `###` | `order` | A strategy for ordering PostCSS plugins. | `function` | **未列** |
| `###` | `plugins` | Options for configuring PostCSS plugins. | **未列** | **未列** |
| `####` | `plugins.autoprefixer` | Plugin to parse CSS and add vendor prefixes to CSS rules.（See 指向 autoprefixer 仓库） | **未列** | **未列** |
| `####` | `plugins.cssnano` | （无说明句）| `object` | **未列**（See 指向 cssnano 的配置文件选项文档） |

三处缺口很扎眼：`order` 有类型但**没给默认函数长什么样**；`plugins` **连 Type 都没有**（它显然是对象，但文档只留了一句"配置插件的选项"和一个 PostCSS 文档链接）；`cssnano` 那一行**只有类型没有说明句**。

**这四行是本节的全部。** 要写插件配置，靠这一节不够——得看样式页。

## 样式页给的三条实际信息

第一句是定性：**"Nuxt comes with postcss built-in. You can configure it in your `nuxt.config` file."**（内置，不用另装 PostCSS 本体。）

第二条给写法，注意**键是包名**：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  postcss: {
    plugins: {
      'postcss-nested': {},
      'postcss-custom-media': {},
    },
  },
})
```

**两个插件各占一个键，值是给它自己的选项**（示例里两个都是空对象，也就是用插件自己的默认）。这与配置参考里 `plugins` 下面只点名 `autoprefixer` 与 `cssnano` 并不冲突——那两项只是**被单独列了说明的默认插件**，不是"只能写这两个"。

第三条是默认清单，这句最该背下来：

> By default, Nuxt comes with the following plugins already pre-configured:

| 插件 | 官方描述 |
| --- | --- |
| postcss-import | Improves the `@import` rule |
| postcss-url | Transforms `url()` statements |
| autoprefixer | Automatically adds vendor prefixes |
| cssnano | Minification and purge |

**四个都已预配**，所以 `@import`、`url()`、厂商前缀、压缩这四件事不需要你写任何配置。

第四条是 SFC 里的语法高亮：**"For proper syntax highlighting in SFC, you can use the postcss lang attribute."**

```vue
<style lang="postcss">
  /* Write postcss here */
</style>
```

## order 这一项容易读反

`order` 的说明只有 "A strategy for ordering PostCSS plugins."——**它排的是插件之间的顺序**，不是样式表的顺序。

这一点值得单独强调，因为上一节（`css`）讲的正是全局样式数组，而**数组项之间的层叠先后官方从来没写规则**。两件事撞在一个词上，很容易以为 `postcss.order` 就是"控制 `css: [...]` 里谁覆盖谁"的那个开关。**不是。** 它作用在插件执行顺序这一层。

另外这一项**类型是 `function` 却没有默认值**，官方也没给一个可用的示例签名。所以真要写它，签名参数是什么、返回值该怎么给，**只能从 PostCSS 侧或实测得出——文档在这点上没有可用信息**。

## 还有一条与打包器无关的提醒

`nitro.config`／`vite.config`／`webpack.config`／`postcss.config` 这四份独立配置文件都不再被读取，官方给的替代动作是**用 `nuxt.config` 里对应的键**——`postcss.config` 对应 `postcss`。

所以从旧项目拷来的 `postcss.config.js` 会静默失效（撞上有 `NUXT_B5004` 那条兼容性提示，虽然它的触发描述只点名 vite/webpack 两份）。**样式相关的配置只写在 `nuxt.config` 的 `postcss` 节里**，这是配置页与错误码页共同指向的结论。

## 自查

- `## postcss` 没有定性句，只有 `order`（`function`，无默认）与 `plugins`（无 Type、无默认，下面点名 `autoprefixer` 与 `cssnano`）两项；信息量集中在样式那一页。
- Nuxt **内置 postcss**；`plugins` 以**包名为键**、值是该插件自己的选项。
- 默认已预配四个插件：**postcss-import**（改进 `@import`）、**postcss-url**（转换 `url()`）、**autoprefixer**（加厂商前缀）、**cssnano**（压缩与 purge）——这四件事不用你配。
- `order` 排的是**插件顺序**，不是 `css` 数组里样式表的先后；它的函数签名文档没给。
- `postcss.config.js` 这类独立文件不被读取，配置只写在 `nuxt.config` 的 `postcss` 节；SFC 里要 PostCSS 语法高亮用 `lang="postcss"`。

## 下一篇

下一篇 551 进 `vite` 节：官方第一句是"直接传给 Vite 的配置"，第二句给出环境分层的规则——**顶层 `vite` 选项客户端与服务端共用，`$client` 与 `$server` 才会被并入各自的那次构建**，末尾还有一句"并非所有 Vite 选项都在 Nuxt 里受支持"。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## postcss` 整节：`### order` 的说明句与 `function` 类型、`### plugins` 的说明句与 See 链接、`#### autoprefixer` 的说明句与 See、`#### cssnano` 的 `object` 类型与 See，四行均未列默认）
- https://nuxt.com/docs/4.x/getting-started/styling（"Nuxt comes with postcss built-in"、以包名为键的 `plugins` 示例、四个默认插件及其官方描述、`lang="postcss"` 的语法高亮句与示例）
- https://nuxt.com/docs/4.x/getting-started/configuration 与 https://nuxt.com/docs/4.x/errors/b5004（`postcss.config` 不再被读、改用 `postcss` 键）
