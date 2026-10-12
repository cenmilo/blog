---
url: /posts/9cec4048c47f7f5b85c170f041e005ae/
title: ESM 与 CommonJS 互操作：库引进来就报语法错误的真实原因
date: 2026-10-17T09:00:00+08:00
updated: 2026-10-17T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：Nuxt 使用原生 ES 模块。历史上库靠 package.json 的 module 字段暴露 ESM 构建（例如 main 指向 .cjs.js、module 指向 .esm.js），webpack 与 Rollup 认它，但官方明确提醒：module 字段是打包器的约定，Node.js 本身不认，Node 只用 exports 和 main 两个字段做模块解析。于是当某个 .esm.js 被拿到 Node 的 ESM 环境里 import，就会报 "To load an ES module, set type: module in the package.json or use the .mjs extension" 加 SyntaxError: Unexpected token 'export'；命名导入时报的是 Named export not found 并附带 CommonJS 只能通过默认导出导入的提示。官方给的自救手段有 build.transpile（把库加进去，可能还得加上它依赖的包）、alias 到手动的 CJS 版本、以及对默认导出的手工 interop（import { default as pkg }、动态 import 后取 m.default || m），更稳的做法是用 mlly 的 interopDefault 以保留命名导出。库作者侧的推荐是：把 ESM 文件改名成 .mjs 结尾（最简单）、或整体 ESM-only 并设 type: module。

categories:
  - 前端开发

tags:
  - nuxt4
  - esm
  - commonjs
  - 排错
  - 依赖

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


这一讲的读者画像很具体：你 `npm install` 了一个包，代码里 `import` 它，然后**构建报出一个你完全看不懂的 `SyntaxError: Unexpected token 'export'`**。先说结论：**这不是你写错了，是那个包的发布方式与 Node 的解析规则没对上。**

## 前提：Nuxt 用的是原生 ES 模块

官方这一页的标题下就一句：

> Nuxt 使用原生 ES 模块。

"原生"这个词是关键。历史上我们写的 ESM 语法（`import`/`export`）都会被打包器转成别的格式，所以**没人关心 Node 自己怎么看这个文件**。现在它要看了——因为服务端那一侧是 Node 在跑。

## 两种格式与那句最要命的提醒

官方把两种模块写法直接摆出来对照：

```js
const a = require('./a')

module.exports.a = a
```

```js
import a from './a'

export { a }
```

然后讲清了历史上库是怎么"同时提供两份"的：

> 一个示例库可能同时暴露 CJS 与 ESM 两个版本，让我们自己挑要哪个：

```json
{
  "name": "sample-library",
  "main": "dist/sample-library.cjs.js",
  "module": "dist/sample-library.esm.js"
}
```

> 所以在 Nuxt 2 里，打包器（webpack）会在服务端构建时拉取 CJS 文件（`main`），并在客户端构建时使用 ESM 文件（`module`）。

**接下来这一句是整篇的核心**，请逐字读：

> `module` 字段是被 webpack 与 Rollup 这类打包器使用的约定，但 **Node.js 本身并不识别它**。Node.js 在做模块解析时**只使用 `exports` 与 `main` 字段**。

也就是说：那个能用的 `.esm.js`，在 Node 眼里**根本没有被登记过**。它只会被 `main` 指到的 `.cjs.js` 找到——而那个文件里全是 `require`。

## 三种有效后缀，Node 只认这个

> Node 支持下面几类导入：
> 1. 以 `.mjs` 结尾的文件——它们被期望使用 ESM 语法
> 2. 以 `.cjs` 结尾的文件——它们被期望使用 CJS 语法
> 3. 以 `.js` 结尾的文件——**除非它们的 `package.json` 里有 `"type": "module"`**，否则被期望使用 CJS 语法

第 3 条是绝大多数事故的来源：`.js` 的默认身份是 **CJS**。

顺带这也解释了 012/014 讲那句"我们的产物是 `.output/server/index.mjs`"——官方自己的说法是：

> 这就是我们为 Nuxt Nitro 所做的事：我们输出一个 `.output/server/index.mjs` 文件。这告诉 Node.js 把这个文件当作原生 ES 模块来对待。

## 你会看到的两条报错，逐字对照

**第一条：整个文件被当成 CJS 来解析。**

> 不过，如果你试图在一个 Node.js ESM 环境里导入一个带 `.esm.js` 文件的包，它不会工作，你会得到类似这样的错误：

```bash [Terminal]
(node:22145) Warning: To load an ES module, set "type": "module" in the package.json or use the .mjs extension.
/path/to/index.js:1

export default {}
^^^^^^

SyntaxError: Unexpected token 'export'
```

**第二条：命名导入拿不到。**

```bash [Terminal]
file:///path/to/index.mjs:5
import { named } from 'sample-library'
         ^^^^^
SyntaxError: Named export 'named' not found. The requested module 'sample-library' is a CommonJS module, which may not support all module.exports as named exports.

CommonJS modules can always be imported via the default export, for example using:

import pkg from 'sample-library';
const { named } = pkg;
```

这两条报错的**归责**官方说得很直白：

> 如果你遇到这些错误，问题几乎肯定出在上游库身上。他们 Needs 让他们的库支持被 Node 导入。

（原话是"the issue is almost certainly with the upstream library"。）所以你多半改不了库——能改的是自己项目的解析方式。

## 三个自救手段（按官方顺序）

**一、`build.transpile`：**

> 在此期间，你可以通过把 `sample-library` 加入 `build.transpile` 来告诉 Nuxt 不要尝试导入这些库：

```ts
export default defineNuxtConfig({
  build: {
    transpile: ['sample-library'],
  },
})
```

> 你可能发现你**还**需要把这些库所导入的其它包也加上。

那句"还要加依赖的依赖"是实战里最常卡住的一步——只加最外层往往不够。

**二、`alias` 指向 CJS 版本：**

> 某些情况下，你可能还需要手动把该库别名到 CJS 版本，例如：

```ts
export default defineNuxtConfig({
  alias: {
    'sample-library': 'sample-library/dist/sample-library.cjs.js',
  },
})
```

**三、默认导出的手工 interop。** 这是另一个变种：CJS 包经 `module.exports` 提供对象时，interop 有时会失败，你拿到的多了一层 `default`：

```js
import pkg from 'cjs-pkg'

console.log(pkg) // { default: { test: 123 } }
```

官方还指出一个必然如此的场景：**动态导入一定会拿到这层 `default`**：

```js
import('cjs-pkg').then(console.log) // [Module: null prototype] { default: { test: '123' } }
```

给的解法有两种：

```js
// 静态导入
import { default as pkg } from 'cjs-pkg'

// 动态导入
import('cjs-pkg').then(m => m.default || m).then(console.log)
```

更稳的是官方推荐的库：

> 为了处理更复杂的情况与更高的安全性，我们推荐 [mlly]，它能保留命名导出。

```js
import { interopDefault } from 'mlly'

// 假设形状是 { default: { foo: 'bar' }, baz: 'qux' }
import myModule from 'my-module'

console.log(interopDefault(myModule)) // { foo: 'bar', baz: 'qux' }
```

## 如果你是库作者：两条路

官方给的顺序就是推荐顺序：

> 好消息是修复 ESM 兼容问题相对简单。主要有两个选项：
> 1. **你可以把 ESM 文件改名成以 `.mjs` 结尾。** 这是**推荐的、最简单的**做法……为了最大程度的明确性，也建议把 CJS 文件改名成 `.cjs` 结尾。
> 2. **你可以选择让整个库 ESM-only。** 这意味着在你的 `package.json` 里设置 `"type": "module"` 并确保构建出的库使用 ESM 语法。**但你可能在依赖上遇到问题**——并且这种方式意味着你的库**只能**被用在 ESM 环境里。

作者侧还有一条最佳实践值得抄：

> - 优先使用命名导出而不是默认导出。这有助于减少 CJS 冲突。
> - 尽可能避免依赖 Node.js 内置模块与 CommonJS 或仅限 Node.js 的依赖，以让你的库能在浏览器与 Edge Worker 里可用而无需 Nitro 的 polyfill。
> - 使用新的 `exports` 字段配合条件导出。

```json
{
  "exports": {
    ".": {
      "import": "./dist/mymodule.mjs"
    }
  }
}
```

## 自查

- [ ] 报错里有 `Unexpected token 'export'` 时，你会不会先怀疑那个包的 `main` 指向的是 CJS 文件？
- [ ] 你知道 Node 只看 `exports` 和 `main`，不看 `module` 吗？
- [ ] `.js` 文件在 Node 眼里默认是什么格式？（除非包里有 `"type": "module"`）
- [ ] 用 `build.transpile` 时，有没有把该库依赖的包也加进去？
- [ ] 动态 `import()` 一个 CJS 包时，有没有处理 `m.default || m` 这一层？
- [ ] 自己发包时，是否优先选"ESM 产物改名 `.mjs`"这条，而不是"整个库 ESM-only"？

## 下一篇

016 - Nuxt 的版本规则：发布节奏、维护期限与 nightly 频道

## 本文依据

- `guide/concepts/esm`（"Nuxt 使用原生 ES 模块"；CJS 与 ESM 语法对照；`main`/`module` 双份发布的历史与 Nuxt 2 里 webpack 分别取用；`module` 字段是打包器约定、Node.js 只认 exports 与 main 的 note；原生 ESM 的两种启用方式（type: module 或 .mjs，推荐后者）与 Nitro 输出 .output/server/index.mjs 的说明；三类有效后缀与 .js 默认为 CJS 的判定；两条完整报错文本（To load an ES module…/Named export not found…）与"问题几乎肯定在上游库"；troubleshooting 三步：build.transpile（含需要一并加其依赖的提醒）、alias 到 CJS 版本、默认导出 interop（含动态 import 必然拿到 default 一层、import { default as pkg }、m.default || m、mlly 的 interopDefault 及示例）；库作者两条选项（.mjs 改名为推荐最简单、整体 ESM-only 及其限制与依赖风险）；migration 的 require→import 前后对照、__filename/__dirname/require.resolve 的替代写法；最佳实践三条（优先命名导出、避免 Node 内置与 CJS 依赖以能在浏览器与 Edge Worker 使用、使用 exports 条件导出及其 JSON 示例）
- `guide/concepts/server-engine`（独立于 node_modules 的服务端 dist、产物可跑在浏览器 service worker 等环境）
