---
url: /posts/f1c46176425b3e4b2eabd08b575615a3/
title: 357 - alias 与 extensions：模块解析的两把钥匙
date: 2027-02-08T09:00:00+08:00
updated: 2027-02-08T09:00:00+08:00
author: cmdragon

summary:
  摘要：import 一个路径时，"这个前缀指哪"由 alias 决定，"不写扩展名时补哪些"由 extensions 决定。nuxt-config 的 alias 默认有十项（~ 和 @ 指 app、~~ 和 @@ 指 rootDir、#shared/#server/#build、assets/public、#internal/nuxt/paths），你能往里加自定义别名；一条 webpack 上下文的规矩是——图片源、CSS（不含 JS）里访问别名必须加 `~` 前缀。extensions 默认是 .js/.jsx/.mjs/.ts/.tsx/.vue 六项（不含 .json），是 Nuxt 解析器不写后缀时补的候选。这一篇把两把钥匙配着讲，并接回它们进 tsconfig、自动补全那层。

categories:
  - 前端开发

tags:
  - nuxt4
  - alias
  - extensions
  - 模块解析
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


`import ... from '~/components/Foo'` 和 `from 'data/test.json'` 这两种写法能成立，靠的是两个配置：`alias`（路径前缀映射）和 `extensions`（补全扩展名）。这一篇一起讲。

## alias：路径前缀指哪

nuxt-config 的 `## alias` 默认是一张十项表：

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

`~`/`@`→ app 目录，`~~`/`@@`→ rootDir，还有 `#shared`/`#server`/`#build` 这些井号别名（051 讲过完整表）。官方定位它是"你可以加自定义别名，方便在 JS 和 CSS 里访问自定义目录"：

> You can improve your DX by defining additional aliases to access custom directories within your JavaScript and CSS.

例子——把 `images`/`style`/`data` 映射到具体目录：

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

## alias 的 webpack 上下文规矩（一条硬 note）

官方 callout 划了一条界：

> **Note**: Within a webpack context (image sources, CSS - but not JavaScript) you *must* access your alias by prefixing it with `~`.

也就是：**在 webpack 处理的上下文里（图片源、CSS，但不含 JavaScript）访问别名必须加 `~`**。示例里 `<img src="~images/main-bg.jpg">`、CSS 里 `@import '~style/...'`、`url('~images/...')` 都带 `~`，而 JS 里 `import data from 'data/test.json'` 不带。这条是"为什么图片/CSS 里的别名要加波浪号、JS 里不用"的依据。

## alias 与类型

另一条 callout：这些别名会被自动加进生成的 tsconfig（`.nuxt/tsconfig.app.json` 等），所以有类型支持和路径补全。要再扩配置用 `typescript.tsConfig`（076/155）。

## extensions：不写后缀时补哪些

`## extensions` 是解析器不写扩展名时的候选：

> The extensions that should be resolved by the Nuxt resolver.
> - Default: [".js", ".jsx", ".mjs", ".ts", ".tsx", ".vue"]

六项，**没有 `.json`**。这解释了为什么 `import data from 'data/test.json'`（357 上面的例子）要**带 `.json` 后缀**——`.json` 不在默认解析扩展名里，省略它就找不到。而 `.js`/`.ts`/`.vue` 可以省略。（081 还讲过：`vite.resolve.extensions` 是另一份、顺序不同且含 .json；同名不同扩展谁赢未定义。）

## 两把钥匙配着用

| 场景 | 谁起作用 |
| --- | --- |
| `~/`、`#shared` 等前缀 → 真实目录 | alias |
| `import X from './Foo'` 省略后缀 → 试哪个文件 | extensions |
| 图片/CSS 里用别名 | alias，且必须加 `~`（webpack 上下文） |
| import 一个 `.json` | 得写全后缀（.json 不在 extensions 默认里） |

## 自查

- 加自定义别名在 `alias`；记住图片/CSS 里访问别名必须加 `~`，JS 里不用。
- 别名会自动进生成的 tsconfig，有类型与补全；要再扩走 `typescript.tsConfig`。
- `extensions` 默认六项不含 `.json` → import json 要带后缀；`.js/.ts/.vue` 可省。
- 别把 Nuxt resolver 的 extensions 和 vite.resolve.extensions 当同一个（后者含 .json、顺序不同）。

## 下一篇

下一篇 358 讲 typesDir：生成的类型与 tsconfig 放哪、它为什么默认跟 buildDir 走却被刻意分开。
