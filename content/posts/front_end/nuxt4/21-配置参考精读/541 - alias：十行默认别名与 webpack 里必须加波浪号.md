---
url: /posts/c561b0955f97c02941a367efc6f72787/
title: 541 - alias：十行默认别名与 webpack 里必须加波浪号
date: 2027-04-10T10:00:00+08:00
updated: 2027-04-10T10:00:00+08:00
author: cmdragon

summary:
  摘要：`alias` 的定位是**"定义额外别名以访问自定义目录"**（You can improve your DX by defining additional aliases to access custom directories within your JavaScript and CSS），类型 `object`。官方给的默认表有十行：`~` 与 `@` 指向 `/<rootDir>/app`，`~~` 与 `@@` 指向 `/<rootDir>`，`#shared` 指 `shared`、`#server` 指 `server`、`assets` 指 `app/assets`、`public` 指 `public`、`#build` 指 `.nuxt`、`#internal/nuxt/paths` 指 `.nuxt/paths.mjs`。两条 callout 各管一件事：**在 webpack 上下文里（图片源、CSS——但不含 JavaScript）访问别名必须加 `~` 前缀**；这些别名会被自动写进生成的 TypeScript 配置（`.nuxt/tsconfig.app.json` 等），要继续扩展就写在这里或 `typescript.tsConfig`。官方示例的自定义别名一律用 `fileURLToPath(new URL(..., import.meta.url))` 生成绝对路径。

categories:
  - 前端开发

tags:
  - nuxt4
  - alias
  - 路径别名
  - webpack
  - typescript
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

同样是 `@import`，在 `<style>` 里写 `style/variables.scss` 报错，加上 `~` 就好了。

这不是笔误问题，是这一节里那条硬性规则。

## 定位与那十行默认值

`## alias` 第一句：**"You can improve your DX by defining additional aliases to access custom directories within your JavaScript and CSS."**

- **类型**：`object`
- **默认**：一份十行 JSON

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

这张表里有四组规律，都能在行与行之间读出来：

| 组 | 键 | 指向 | 读出什么 |
| --- | --- | --- | --- |
| 双拼写 | `~` 与 `@`、`~~` 与 `@@` | 分别是 `app` 与根 | **同一目标两个别名**，官方给了两套习惯写法 |
| 带 `#` 的三条 | `#shared`、`#server`、`#build` | `shared`、`server`、`.nuxt` | **`#` 前缀用于非源码目录与生成目录** |
| 内部一条 | `#internal/nuxt/paths` | `.nuxt/paths.mjs` | 框架自己用的路径模块，命名带 `internal` |
| 两个裸名 | `assets`、`public` | `app/assets`、`public` | **无前缀的普通别名**，与 `dir.assets`/`dir.public` 同指向 |

**`<rootDir>` 是占位符**（取解析出来的项目根），别当字面字符串读。

还有一点要说清：**这张表里没有 `#components`、`#app`、`#imports`。** 但这三个别名在别处被反复使用——组件与自动导入的生成别名由那套机制自己维护，不在这份默认表里列。读到"表里没有"不等于"不存在"。

## 第一条 callout：webpack 上下文必须加 `~`

紧跟默认值的 callout，原文措辞很硬：

> **Note**: Within a webpack context (image sources, CSS - but not JavaScript) you *must* access your alias by prefixing it with `~`.

三个限定：**只在 webpack 上下文生效**；括号里点名范围是"图片源、CSS"；**并明确排除 JavaScript**。用词是 *must*。

官方那段 HTML 示例把三种写法都摆在一起，正好构成对照：

```html
<template>
  <img src="~images/main-bg.jpg">
</template>

<script>
import data from 'data/test.json'
</script>

<style>
// Uncomment the below
//@import '~style/variables.scss';
//@import '~style/utils.scss';
//@import '~style/base.scss';
body {
  background-image: url('~images/main-bg.jpg');
}
</style>
```

读法：**模板里的 `<img src>` 与 CSS 里的 `url()`、`@import` 都带 `~`，而 `<script>` 里的 `import data from 'data/test.json'` 不带**——这与 callout 那句"but not JavaScript"逐字对应。

这里有个常见误解要挡住：`~` 在这条规则里的角色是 **webpack 的引用前缀**，与默认表第一行那个指向 `<rootDir>/app` 的 `~` 别名**不是同一个用法**。前者是语法要求，后者是路径映射——所以"`~style/variables.scss`"里的 `~` 是前缀、`style` 才是别名。

## 第二条 callout：别名会进生成的 tsconfig

> **Note**: These aliases will be automatically added to the generated TypeScript configurations (`.nuxt/tsconfig.app.json`, `.nuxt/tsconfig.server.json`, etc.) so you can get full type support and path auto-complete. In case you need to extend options provided by the generated configurations further, make sure to add them here or within the `typescript.tsConfig` property in `nuxt.config`.

两件事：**你写的别名会被自动同步到生成的 TypeScript 配置里**，于是类型支持与路径补全跟着别名走；**要在这个基础上继续扩展，位置只有两处**——`alias` 本身，或 `typescript.tsConfig`。

注意生成的配置不止一份（`.nuxt/tsconfig.app.json`、`.nuxt/tsconfig.server.json`，官方用 "etc." 收尾），而 `typescript.appTsConfig` 那节还补了一条优先级：**放在 `appTsConfig` 里的选项优先于 `tsConfig`**。三处合起来决定编辑器看到什么。

## 自定义别名的官方写法

官方示例是三个自定义别名，每个都用同一种方式取绝对路径：

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

三个细节值得照抄：**从 `node:url` 导入 `fileURLToPath`**；用 `new URL(相对路径, import.meta.url)` 再转成路径，**基准是 `nuxt.config.ts` 自己所在目录**；键写成带引号的字符串。

这套写法与前面 `alias` 默认表里"指向 `<rootDir>/app`"的绝对形式并不冲突——**默认值是解析后的结果，示例给的是你自己怎么造一个结果**。文档没解释为什么用 `fileURLToPath` 而不是直接写相对字符串；能确证的只有官方示例用的就是这个形状。

## 自查

- `alias` 是 `object`，默认十行：`~`/`@` → `app`，`~~`/`@@` → 根，`#shared`/`#server`/`#build` → 对应目录，`assets`/`public` → `app/assets`/`public`，`#internal/nuxt/paths` → `.nuxt/paths.mjs`。
- **webpack 上下文（图片源、CSS，不含 JavaScript）里访问别名必须加 `~` 前缀**；这个 `~` 是引用语法，与指向 `app` 的 `~` 别名是两回事。
- 别名会自动写进生成的 `.nuxt/tsconfig.app.json`、`.nuxt/tsconfig.server.json` 等配置；要扩展只能写在 `alias` 或 `typescript.tsConfig`，而 `appTsConfig` 的选项优先于 `tsConfig`。
- 自定义别名官方一律用 `fileURLToPath(new URL('./...', import.meta.url))`，基准是配置文件所在目录。
- 默认表里没有 `#components`、`#app`、`#imports`，但它们在别处示例中出现——表未列不等于不存在。

## 下一篇

下一篇 542 读 `extensions`：Nuxt 解析器要试的扩展名默认是六个（`.js`、`.jsx`、`.mjs`、`.ts`、`.tsx`、`.vue`），顺序就是这份数组的顺序，而文档没有讨论同名多扩展时的选择规则。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## alias` 整节：定义句、Type `object`、十行默认 JSON、webpack 上下文必须加 `~` 的 callout、别名自动加入生成的 TypeScript 配置与可在 `typescript.tsConfig` 扩展的 callout、`fileURLToPath` 三别名示例与那段 HTML 用法示例；`### typescript.appTsConfig` 的"take precedence over `tsConfig`"一句；`## dir` 中 `assets`/`public` 的默认值）
