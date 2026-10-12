---
url: /posts/89a6e85688a343138c92d90c6565caf0/
title: vue：五个子项、transformAssetUrls 的五个默认数组与一句警告
date: 2027-04-22T10:00:00+08:00
updated: 2027-04-22T10:00:00+08:00
author: cmdragon

summary:
  摘要：`## vue` 的定性句只有三个词 "Vue.js config"，下面是**五个子项**：`compilerOptions`（无类型无默认，只有 See）、`config`（**只有可序列化的选项能写进 `nuxt.config`，其余应在插件里运行时设**）、`propsDestructure`（默认 `true`）、`runtimeCompiler`（默认 `false`，带一条把它比作 `eval` 的安全警告）、`transformAssetUrls`。最后一项最反常：**它自己一句说明都没有、也没有外链**，却挂着五个子项，各自给一份属性名默认数组（`image`/`use` 是 `["xlink:href","href"]`，`img`/`source` 是 `["src"]`，`video` 是 `["src","poster"]`）。

categories:
  - 前端开发

tags:
  - nuxt4
  - Vue编译器
  - 运行时编译
  - 资源URL
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模板里写 `<img src="./logo.png">`，构建后这个相对路径变成了带 hash 的资源 URL；反过来，有些团队要在运行时把后台下发的字符串当模板渲染。这两件事分别归这一节的 `transformAssetUrls` 和 `runtimeCompiler` 管——但你在配置参考里能找到的是：前者**一句解释都没有**，后者带着一句警告。

## 定性句与五个子项

`## vue` 的定性句是三个字：**"Vue.js config"**，没有更多。五个子项的记载完整度差别很大：

| 子项 | Type | Default | 说明句 |
| --- | --- | --- | --- |
| `compilerOptions` | 无 | 无 | "Options for the Vue compiler that will be passed at build time." ＋ Vue 官方文档链接 |
| `config` | 无 | 无 | 全局配置 Vue 应用；**只有可序列化的选项能写进 `nuxt.config`**，其余应在 Nuxt 插件里运行时设 ＋ Vue app config 链接 |
| `propsDestructure` | `boolean` | `true` | "Enable reactive destructure for `defineProps`" |
| `runtimeCompiler` | `boolean` | `false` | "Include Vue compiler in runtime bundle." ＋ 一句展开 ＋ `::warning` |
| `transformAssetUrls` | 无 | 无 | **没有说明句，也没有外链**，只有下面五个带默认值的子项 |

两处值得单独记住。

**`config` 那条的限制是可执行的判据**：原句 "Only serializable options may be set in your `nuxt.config`. All other options should be set at runtime in a Nuxt plugin."——Vue 应用配置里那些函数型、实例型的选项（例如自定义错误处理、组件解析相关的钩子）**不能写在配置文件里**，要写就写到插件里。

**`compilerOptions` 是"建时传入"**："will be passed at build time" 这句定了它的时机，具体能传哪些项由 Vue 文档那侧决定，Nuxt 这页不列。

## propsDestructure：默认已经开着，示例却在教怎么开

`propsDestructure` 的类型行写 `boolean`，默认值 `true`——**响应式解构（reactive destructure，即 `const { foo } = defineProps()` 之后 `foo` 仍是响应式的，不必每次写 `props.foo`）在 Nuxt 里默认就是开的。**

而入门配置那一页讲它时的说法是另一套：

> You may need to enable experimental features in Vue, such as `propsDestructure`. Nuxt provides an easy way to do that in `nuxt.config.ts`, no matter which builder you are using:

配的代码块正是：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  vue: {
    propsDestructure: true,
  },
})
```两句并排读，有两点要注意：

- 那句 **"no matter which builder you are using"** 是这一路的承诺：不管你用哪个构建器，Vue 的这类实验特性都从 `vue` 这一节开，而不是去 Vite 或 webpack 的配置节里找对应开关。
- 示例写 `true`、默认值也是 `true`，所以**这段的作用其实是"怎么关"的反面**：真要关掉得写 `false`，而文档在这里没给关掉之后的影响面。它把 `propsDestructure` 归到 Vue 的实验特性里，同时配置参考把它写成默认开启的布尔项——**两处口径不同，文档没有对齐说明。**

同一页紧跟的一段还交代了一个历史：**`reactivityTransform` 自 Nuxt 3.9 与 Vue 3.4 起从 Vue 移到了 Vue Macros**，那边有 Nuxt 集成。也就是别再找 `vue.reactivityTransform` 这个键。

## runtimeCompiler：一句把风险说透的警告

默认 `false`。展开句是：

> Enabling this allows components to compile templates at runtime (for example, string `template` options or templates supplied via data).

即打开后组件可以在运行时编译模板（字符串 `template` 选项、或从数据里拿到的模板）。配套的 `::warning` 是这一节唯一的安全警告，措辞很硬：

> Runtime template compilation executes the compiled template as JavaScript. Never compile templates built from user-provided or otherwise untrusted input; doing so is equivalent to `eval` and can lead to cross-site scripting or remote code execution.

翻成动作就是：**模板字符串来自用户输入的场景，不能靠打开这一项来实现**（XSS 与远程代码执行）。Vue 安全指南的链接在同一句里给出来了。

服务端组件那一页还补了一条与它相关的行为：island 的 props 来自请求（URL 查询或 body），要当不可信输入处理，Nuxt 会拒掉最容易漏的那两类——**未声明的顶层 `as`**，以及**`vue.runtimeCompiler` 打开时 props 里任何位置的 `template`**。也就是说这一项开了之后，官方在服务端组件入口上确实按"它可能被滥用"来做了拦截；除此之外的动态组件解析（`<component :is>`、`h()`、`resolveDynamicComponent()`、多态 `as`/`asChild`）仍要你自己校验传进去的值。

## transformAssetUrls：五个默认数组，零句解释

这一项下面挂了五个子项，**每个只有 `**Type**: array` 和一份默认 JSON**，五个合起来是一张"标签名 → 属性名数组"的表：

| 子项 | 默认数组 |
| --- | --- |
| `image` | `["xlink:href", "href"]` |
| `img` | `["src"]` |
| `source` | `["src"]` |
| `use` | `["xlink:href", "href"]` |
| `video` | `["src", "poster"]` |

**`### transformAssetUrls` 标题与第一个子项之间一个字都没有**：没有它做什么的说明，没有取值语义，也没有指向 Vue 文档的 See（这一节里给了外链的只有 `compilerOptions`、`config`，以及 `runtimeCompiler` 警告里那条安全指南；`propsDestructure` 同样没有 See）。整套 4.x 文档里 `transformAssetUrls` 这个名字**只出现在这一处**，所以想弄清"这些属性名列表被拿去干什么"，本节的记载不够用，只能按名字到 Vue 编译器文档里查同名选项。

能确定的只有这张表本身：默认被列出的属性是 `img`/`source` 的 `src`、`video` 的 `src` 与 `poster`、`image` 与 `use` 的 `xlink:href` 和 `href`——**SVG 与位图标签走的是两个不同的属性集合**。要动它（加标签、加属性）时，因为文档没写合并规则，**覆盖还是追加需要实测**。

## 自查

- `## vue` 定性句只有 "Vue.js config"，五个子项的类型/默认行分布很不齐：只有 `propsDestructure` 与 `runtimeCompiler` 两条是完整的 `boolean` ＋ 默认值。
- `config` 那条限制照做：**可序列化的才写进配置文件**，函数型的进插件。
- `propsDestructure` 默认 `true`；入门页那段"可能要打开它"与它口径不同，且那页承诺这条路径与构建器无关。
- `runtimeCompiler` 默认 `false`；打开后模板按 JavaScript 执行，**等价 `eval`**，不要编译不可信输入；服务端组件入口会拒掉带 `template` 的 island props。
- `transformAssetUrls`：**零说明、零外链**，只有五个默认数组；合并行为未记载。
- `vue.reactivityTransform` 这个键已经不存在（自 Nuxt 3.9／Vue 3.4 移入 Vue Macros）。

## 下一篇

下一篇 578 看 `## unhead`：**只有两个子项**（`legacy`、`renderSSRHeadOptions`），但 `legacy` 那条把"开了它会带来哪三处变化"列成了行内清单——`Capo.js` 排序关闭、`DeprecationsPlugin` 支持 `hid`/`vmid`/`children`/`body`、`PromisesPlugin` 支持 promise 输入；而 `future.compatibilityVersion: 5` 会让 `unhead.legacy` 被忽略。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## vue` 定性句；五个 `###` 子项原文与 Type/Default 行；`config` 的 serializable 句；`runtimeCompiler` 的展开句与 `::warning` 全文；`transformAssetUrls` 无说明句的事实与五个 `####` 默认数组）
- https://nuxt.com/docs/4.x/getting-started/configuration（"Enabling Experimental Vue Features" 整段与 `propsDestructure: true` 示例、"no matter which builder you are using"、`reactivityTransform` 移入 Vue Macros 那段）
- https://nuxt.com/docs/4.x/guide/concepts/server-components（island props 属不可信输入、拒掉未声明顶层 `as` 与 `vue.runtimeCompiler` 开启时的 `template`、动态组件解析需自行校验）
