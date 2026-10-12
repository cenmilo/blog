---
url: /posts/7a1dd32294963c39e8b258d33ef80974/
title: 扩展体系总图：从 nuxt.config 到钩子到 Kit 的四层
date: 2027-06-01T11:00:00+08:00
updated: 2027-06-01T11:00:00+08:00
author: cmdragon

summary:
  摘要：Nuxt 的扩展面可以分成四层，每层能做的事不重叠：**`nuxt.config`** 是用户写的声明；**图层**复用文件与配置，同名按三行优先级覆盖；**模块**是在 `nuxt dev`/`nuxt build` 启动时按顺序运行的异步函数，能覆盖模板、配 loader、加 CSS 库；**Kit 与钩子**是模块的两只手——59 个函数负责"我要注册什么"，81 个钩子负责"什么时候轮到我"。还有一个已经翻篇的入口：`buildModules`，模块现在只存在于构建期。

categories:
  - 前端开发

tags:
  - nuxt4
  - 扩展体系
  - 模块开发
  - 生命周期钩子
  - nuxt-kit
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 为什么要有四层，官方给的理由是一句权衡

概念页把那件事说得很直白：做生产级应用时框架核心往往不够用，而 Nuxt 虽然能用配置项和插件扩展，**但要在多个项目里维护这些定制，既啰嗦又重复耗时**；反过来，把每个项目的需求都塞进核心，"会让 Nuxt 变得非常复杂、难以使用"。模块系统就是从这个矛盾里出来的。

## 第一层：`nuxt.config`，声明式的上限

这一层的能力边界是"能写成配置的东西"。它包含 60 个顶层配置节、192 个子项，从 `app.head` 到 `vite`、`nitro`、`experimental`。今天独立配置文件已经不再生效：`nitro.config`、`vite.config`、`postcss.config`、`webpack.config` 都必须写进 `nuxt.config.ts` 的对应节里。

## 第二层：图层，复用文件与配置

图层的目录结构与普通应用几乎一样，十个位置会被自动扫描继承（`app/components`、`app/composables`、`app/layouts`、`app/middleware`、`app/pages`、`app/plugins`、`app/utils`、`app/app.config.ts`、`server/*`、`nuxt.config.ts`）。它的独有能力是覆盖：项目文件压过 `~~/layers` 里自动扫描的图层（按字母序，Z 压 A），后者又压过 `extends` 里的图层。

来源可以是本地路径、npm 包，或者 git 仓库（`github:`、`gitlab:`、`bitbucket:`）。v4.3 之后还能做减法：把某个模块的配置键设成 `false`，就关掉它——**这尤其用于关掉从图层继承来的模块**。

## 第三层：模块，构建期按顺序跑的异步函数

概念页对模块的定义包含三个可读的要点：

> Nuxt 模块是**异步函数**，在开发模式启动或生产构建时**按顺序运行**。它们可以覆盖模板、配置 webpack loader、添加 CSS 库，以及做很多其他有用的事。

> 最重要的是，Nuxt 模块**可以作为 npm 包分发**，这让它们能在项目之间复用、与社区共享。

写法就是往 `modules` 数组里加，四种形态官方在同一处给全了：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  modules: [
    // Using package name (recommended usage)
    '@nuxtjs/example',

    // Load a local module
    './modules/example',

    // Add module with inline-options
    ['./modules/example', { token: '123' }],

    // Inline module definition
    async (inlineOptions, nuxt) => { },
  ],
})
```

注意最后两种：**元组是"模块加内联选项"，裸函数本身就可以是一个模块定义**。前者适合参数多，后者适合一次性的接线。

这一层有一条已经翻篇的历史，文档用 warning 标着：**Nuxt 模块现在只存在于构建期，Nuxt 2 里的 `buildModules` 已因此弃用**。所以看到旧项目里 `buildModules` 数组，直接并入 `modules`。

## 第四层：模块的两只手——Kit 与钩子

Kit 页开头的定位把这两只手写在同一句里：**Nuxt Kit 提供可组合的工具，让"与 Nuxt 钩子交互"、"与 Nuxt 接口交互"、"开发 Nuxt 模块"这三件事变得简单**。

两边分工是这样的：

| 入口 | 数量 | 性质 | 你要说的话 |
| --- | --- | --- | --- |
| Kit 函数 | 59 个，分布在 20 页 | 命令式调用 | "加一个组件"、"注册一条路由"、"生成一个文件" |
| 钩子 | 81 个，三张表（应用运行时 20、构建期 52、Nitro 运行时 9） | 事件订阅 | "在页面解析完之后、类型写出之前，做这件事" |

模块里最常见的组合就是这两者的搭配：`setup` 里用 Kit 注册资产，用 `nuxt.hook(...)` 在正确的时机再做后续接线。自定义钩子的触发放进 `modules:done`，收尾的 watcher 放进 `close`——都在构建期那张表上。

## Kit 的三条使用边界，写在同一页

概念页没讲、但决定"能不能用 Kit"的是这三条：

**一、只在模块里可用。**原文的 note 是：Nuxt Kit 的工具**只对模块可用，不 meant 被导入到运行时**（组件、Vue 组合式函数、页面、插件，或 server routes）。想在运行时读配置，走 `runtimeConfig`，而不是 import Kit。

**二、要显式安装，且版本不能低于 nuxt。**官方建议即使 Nuxt 已经装了也**显式把 `@nuxt/kit` 加进依赖**；配套那条 note 把关键依赖点出来了：`@nuxt/kit` 和 `@nuxt/schema` 是 Nuxt 的关键依赖，单独安装时要保证这两者的版本**大于等于你的 `nuxt` 版本**，否则会出现意外行为。

**三、它是 esm-only 包。**所以**不能 `require('@nuxt/kit')`**，在 CommonJS 环境里的替代办法是动态导入：

```ts [test.cjs]
// This does NOT work!
// const kit = require('@nuxt/kit')
async function main () {
  const kit = await import('@nuxt/kit')
}
main()
```

顺带一个例外：本地模块（放在项目 `modules/` 目录里的那种）可以用 `nuxt/kit` 这个辅助子路径，不必把 `@nuxt/kit` 加进项目依赖——"要显式安装"针对的是发出去的包。

## 四层怎么选，按能力倒着问

- 需要的东西能写成配置吗？能就停在第一层。
- 要复用的是文件、组件、页面或配置预设吗？是就用图层，需要覆盖链就更明显。
- 要执行代码（判断、读文件、按条件注册）吗？那是模块。
- 模块里要做的事有现成 Kit 函数吗？有就调；需要在特定时机做就订阅钩子；Kit 和钩子都不覆盖的，才考虑改打包配置或自定义构建器。

最后这一层是官方明确劝退的：写自定义 builder 属于高级话题，只想影响打包结果的话，**写一个注册打包器插件的模块通常才是合适的工具**。

## 本文依据

- https://nuxt.com/docs/4.x/guide/concepts/modules
- https://nuxt.com/docs/4.x/guide/going-further/kit
- https://nuxt.com/docs/4.x/api/advanced/hooks
- https://nuxt.com/docs/4.x/guide/going-further/layers
