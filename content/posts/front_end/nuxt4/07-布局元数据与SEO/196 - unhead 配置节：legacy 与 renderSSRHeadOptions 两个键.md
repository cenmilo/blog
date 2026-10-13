---
url: /posts/8997f22e52e5cbbbd5fdc21c7456216e/
title: 196 - unhead 配置节：legacy 与 renderSSRHeadOptions 两个键
date: 2026-12-16T10:00:00+08:00
updated: 2026-12-16T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`nuxt.config.ts` 里有一个顶层 `unhead` 节，定位句是 "An object that allows us to configure the `unhead` nuxt module."，而它下面**只列了两个键**。第一个是 `unhead.legacy`（`boolean`，**默认 `false`**），三条变更逐字照抄："This applies the following changes: - Disables Capo.js sorting - Adds the `DeprecationsPlugin`: supports `hid`, `vmid`, `children`, `body` - Adds the `PromisesPlugin`: supports promises as input"，并指向 unhead 的 migration 文档；第二个是 `unhead.renderSSRHeadOptions`（"An object that will be passed to `renderSSRHead` to customize the output."），默认值官方写成 JSON 一块：`{ "omitLineBreaks": false }`。这一篇我要解决两件自己被绕晕过的事：一是**开了 legacy 会影响标签顺序**（因为它关掉的是 Capo.js 排序，不只是"兼容旧键"），二是 `@unhead/vue` 这个名字在 `typescript.hoist` 的默认数组里也出现过一次，那是**类型深别名**，跟 head 行为无关，两处不能混。至于 `renderSSRHead` 自己还有哪些可传的选项，那一节没有列，得去 unhead 那边查。

categories:
  - 前端开发

tags:
  - nuxt4
  - unhead
  - nuxt.config
  - head
  - legacy

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


如果你把老项目往 4.x 挪，想找"head 相关的配置总入口"，结果发现 `nuxt-config` 那一节的 `unhead` 节短得出奇：两个键，各带一个示例。短是短，但里面有一句信息量很大。

## 节的位置与全貌

`api/nuxt-config` 里顺序是这样的：`typescript` 一节讲完四个键之后就是 `## unhead`，再往下是 `## vite`。整节内容完整照抄：

```ts
export default defineNuxtConfig({
  unhead: {
    legacy: true,
  },
})
```

```ts
export default defineNuxtConfig({
  unhead: {
    renderSSRHeadOptions: {
      omitLineBreaks: true,
    },
  },
})
```

两个键的类型和默认值：

| 键 | 类型 | 默认值 | 文档说明 |
| --- | --- | --- | --- |
| `unhead.legacy` | `boolean` | `false` | 开启 unhead 的 legacy 兼容模式（三条变更见下） |
| `unhead.renderSSRHeadOptions` | `object` | `{ "omitLineBreaks": false }` | "An object that will be passed to `renderSSRHead` to customize the output." |

## legacy 不只管旧键，它还管顺序

`legacy` 的说明句是这一节里最需要读仔细的一句：

> Enable the legacy compatibility mode for `unhead` module. This applies the following changes: - Disables Capo.js sorting - Adds the `DeprecationsPlugin`: supports `hid`, `vmid`, `children`, `body` - Adds the `PromisesPlugin`: supports promises as input

**第一条是 "Disables Capo.js sorting"，不是"支持旧写法"**。也就是说这个开关的作用域比我预想的大：它同时改变 head 标签的排序行为。所以我从这句话能得到的、也应该得到的推论是——把它打开之后，`<head>` 里标签的先后顺序可能变，而不是"只是多认几个键"。文档没有再写 Capo.js 排序按什么规则排，所以顺序会变成什么样只能实测：在同一个页面上分别用 `legacy: false` 和 `legacy: true` 各构建一次，`curl` 出 HTML 比对 `<head>` 里标签的顺序。

另外两条各自给的是一种输入形态的支持：

- `DeprecationsPlugin` 认 `hid`、`vmid`、`children`、`body` 这四个键。反过来说，**默认模式（`legacy: false`）下这四个键不在支持列表里**——这就是 195 末尾我说"老写法在 4.x 默认配置下是无效键"的依据，这里补上出处。
- `PromisesPlugin` 认"promises as input"，即把 Promise 直接当输入值。文档只写了这一句，没有例子，也没有说哪些字段接受 Promise，我不猜。

最后它挂了一个外部链接：**See:** [`unhead` migration documentation](https://unhead.unjs.io/docs/typescript/head/guides/get-started/migration)——想搞清 legacy 到底兼容了什么，那一节没有答案，得去那份迁移文档。

## renderSSRHeadOptions 只管输出形状

这一条我读到的信息非常有限，但边界很清楚：它是**传给 `renderSSRHead` 的参数**，作用对象是"输出"。文档给的默认值就是一块 JSON：

```json
{
  "omitLineBreaks": false
}
```

示例里把它设成 `true`。从键名看是"省略换行"，但文档没有写它省略之后 HTML 长什么样、影响不影响 SEO 抓取（换行本来就不影响解析），所以我把它记成一条**输出美化/压缩**性质的配置：想让服务端 HTML 里的 head 少几行换行，就开它；除此之外我不给它加戏。

`renderSSRHead` 的其他可选项那一节没列。这里有一句容易做错的地方：不要因为它是 unhead 的方法就顺手把 unhead 文档里的选项全抄进来——Nuxt 侧只保证这个对象会被透传，具体哪些键被接受，取决于你装的 unhead 版本。做法是 `nuxt preview` 之后看构建有没有告警、再看 HTML 有没有变化。

## 别和 typescript.hoist 里的同名项搞混

`@unhead/vue` 这个包名在 `nuxt-config` 里还出现在另一个地方，就在 `unhead` 节上面不远的 `typescript.hoist`：

> Modules to generate deep aliases for within `compilerOptions.paths`. This does not yet support subpaths. It may be necessary when using Nuxt within a pnpm monorepo with `shamefully-hoist=false`.

它的默认数组（节选）：

```json
[
  "nitropack/types",
  "defu",
  "h3",
  "ofetch",
  "@unhead/vue",
  "vue",
  "vue-router",
  "@nuxt/schema"
]
```

这一处是**类型解析**用的深别名，改它不会影响 head 的任何行为；反过来，`unhead` 那两个键也不解决 `pnpm` monorepo 里类型找不到的问题。我在同一页里来回看过三四次才确定它们是两个不相干的东西，所以专门写在这里。

## 自查

1. `unhead.legacy` 打开后的三条变更分别是什么？（答：关掉 Capo.js 排序；加 `DeprecationsPlugin` 支持 `hid`/`vmid`/`children`/`body`；加 `PromisesPlugin` 支持 Promise 作为输入。）
2. 默认模式下 `hid` 这个键还能用吗，依据是什么？（答：不能用——`legacy` 默认 `false`，而 `hid` 是 legacy 模式才"Adds"的能力。）
3. 想让服务端 HTML 里的 head 不换行，配什么？（答：`unhead.renderSSRHeadOptions.omitLineBreaks: true`。）

## 下一篇

197 - useRouteAnnouncer 返回什么：两个 Ref 与三个方法

## 本文依据

- `api/nuxt-config`（unhead 一节）：定位句 "An object that allows us to configure the `unhead` nuxt module."；`legacy` 的类型 `boolean`、默认 `false`、三条变更原文与 **See:** `unhead` migration documentation 链接；`renderSSRHeadOptions` 的说明句 "An object that will be passed to `renderSSRHead` to customize the output."、类型 `object`、默认 `{ "omitLineBreaks": false }` 与两个 `defineNuxtConfig` 示例。
- `api/nuxt-config`（typescript.hoist）：说明句 "Modules to generate deep aliases for within `compilerOptions.paths`. This does not yet support subpaths. It may be necessary when using Nuxt within a pnpm monorepo with `shamefully-hoist=false`." 与含 `@unhead/vue` 的默认数组。
- `getting-started/seo-meta`：`Nuxt head tag management is powered by [Unhead].` 一句，用来确认 head 的实际管理者是谁。
