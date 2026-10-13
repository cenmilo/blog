---
url: /posts/20a994440210cc6067c90be0cc8c2798/
title: 005 - 约定与配置的分界：哪些事文件决定，哪些事写在 nuxt.config
date: 2026-10-13T11:00:00+08:00
updated: 2026-10-13T11:00:00+08:00
author: cmdragon

summary:
  摘要：Nuxt 把决策权分成两处：文件位置（约定）与 nuxt.config.ts（配置）。官方对配置文件的定位是：它位于项目根目录，用来覆盖或扩展应用行为；默认配置已经覆盖大多数场景，而这个文件"仍然可以定制并覆盖它的默认行为"。nuxt.config.ts 里最小写法是导出一个 defineNuxtConfig 函数包裹的对象，而 defineNuxtConfig 这个 helper 全局可用、不需要 import。分界可以用一句话记：一个东西"是什么、叫什么、在哪被扫描"由位置决定；"怎么运行、跑在哪一侧、给谁用"由配置决定。另外还有 app.config.ts，它暴露的是构建期就能确定的公开变量，与 runtimeConfig 的关键差别是它不能被环境变量覆盖。

categories:
  - 前端开发

tags:
  - nuxt4
  - 配置
  - 约定
  - nuxt-config
  - app-config

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


001 讲我把"目录约定"单独列成一层，这一讲就把分界说清楚。它关系到你以后遇到"我明明写了却没生效"时，是该改文件位置，还是该改配置。

## 官方的两句话，正好是分界的两侧

**约定这一侧：**

> Nuxt 使用约定和有主见的目录结构来自动化重复性工作，让开发者专注于推进功能。配置文件仍然可以定制并覆盖它的默认行为。

**配置这一侧：**

> Nuxt 用合理的默认值来配置，让你高效。默认情况下，Nuxt 的配置已经能覆盖大多数使用场景。`nuxt.config.ts` 文件可以覆盖或扩展这个默认配置。
>
> 位于 Nuxt 项目根目录的 `nuxt.config.ts` 文件，可以用来覆盖或扩展应用的行为。

两段合起来的意思很实在：**默认值 + 目录约定承担 90% 的决策，配置文件负责剩下的例外。** 所以"要不要写配置"本身就该是个例外驱动的决策，而不是从第一天开始堆配置。

## 配置文件最小长这样

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  // My Nuxt config
})
```

官方特意点了一句你可能一直没注意的事：

> 一个最小的配置文件导出一个 `defineNuxtConfig` 函数，里面装着你的配置。`defineNuxtConfig` 这个 helper 是**全局可用的，不需要 import**。

这就是 002 讲"自动导入"在配置文件里的第一次现身——**配置文件本身也在自动导入的覆盖范围内**。

还有一条对新手很实用的提醒：

> 你不必使用 TypeScript 来构建 Nuxt 应用。不过，我们**强烈建议**为 `nuxt.config` 文件使用 `.ts` 扩展名。这样你可以在 IDE 里获得提示，避免编辑配置时的拼写错误和失误。

为什么这条值得单独说：官方给的理由就是**怕你拼错**——建议用 `.ts` 正是为了在 IDE 里拿到提示和"避免编辑配置时的失误"。至于键名拼错之后会发生什么，**文档没有记载**，别把任何一种结果当成可以依赖的行为；能确定的只有"有类型提示时，你在写的那一刻就能看见错"。

## 一条判据：什么东西归位置，什么东西归配置

把两节内容合起来，我给你一条能直接用的判据：

**归文件位置（约定）决定的：**

| 事项 | 位置 |
| --- | --- |
| 一条路由的 URL 与它的页面组件 | `app/pages/` 的文件结构 |
| 一个组件在模板里的名字 | `app/components/` 的路径 + 文件名 |
| 一个函数能否不写 import 就用 | 放在 `app/composables/` 或 `app/utils/` |
| 哪段代码在导航前跑 | `app/middleware/`（以及是否带 `.global`） |
| 一个接口的地址 | `server/api/` 的文件路径 |
| 布局名 | `app/layouts/` 的文件名 |
| 应用根组件、错误页 | `app/app.vue`、`app/error.vue` |

**归配置决定的：**

| 事项 | 配置 |
| --- | --- |
| 整体渲染方式（要不要 SSR） | `ssr: true/false` |
| 某个 URL 的缓存/重定向/头 | `routeRules` |
| 构建产物去哪、目录改名 | `dir`、`buildDir`、`serverDir` |
| 模块与插件的注册与选项 | `modules`、`plugins` |
| 运行期要读的密钥与公开地址 | `runtimeConfig` |
| 构建器（Vite / webpack / rspack） | `builder` |

一句总结：**"它叫什么、在哪、由谁生成"归位置；"它怎么运行、跑在哪一侧、暴露给谁"归配置。**

这条判据还有个实用推论：**当你想改的是"名字"或"是否被扫到"，先看目录；当你想改的是"行为"，才去写配置。** 反过来做就会出现"改了配置但完全没反应"（其实该挪文件）或"文件挪来挪去还是不对"（其实该改配置）。

## 配置还能按环境分开写

官方给的写法是把整块配置放进环境键里：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  $production: {
    routeRules: {
      '/**': { isr: true },
    },
  },
  $development: {
    //
  },
  $env: {
    staging: {
      //
    },
  },
})
```

> 要在运行 Nuxt CLI 命令时选择某个环境，只需把名字传给 `--envName` 标志，例如：`nuxt build --envName staging`。

这一条后面会反复用到：004 讲说"生产要一台在跑的服务器"是有成本的，而 `$production` 这种写法正是"开发时保持动态、生产时整站走 CDN 缓存"的标准解法——注意示例里那行 `'/**': { isr: true }` 就是官方给的意图演示。

（顺带一句：官方也说明这套机制来自 `c12` 的环境相关配置；如果你写 layer，还可以用 `$meta` 键提供元数据。）

## runtimeConfig 与 app.config.ts：最容易混的两个

配置侧有两个东西长得很像，官方一句话把它们分开了：

**`runtimeConfig`：**

> `runtimeConfig` API 把环境变量的值暴露给你应用的其余部分。默认情况下，这些键**只在服务端可用**。`runtimeConfig.public` 与 `runtimeConfig.app`（由 Nuxt 内部使用）里的键在客户端也可用。
>
> 这些值应当定义在 `nuxt.config` 里，并且**可以使用环境变量覆盖**。

官方给的对照例子很干净：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  runtimeConfig: {
    // 只在服务端可用的私有键
    apiSecret: '123',
    // public 里的键也会暴露给客户端
    public: {
      apiBase: '/api',
    },
  },
})
```

```ini [.env]
# 这将覆盖 apiSecret 的值
NUXT_API_SECRET=api_secret_token
```

而 `app.config.ts` 的差别正好相反：

> 位于源码目录（默认是 `app/`）的 `app.config.ts` 文件，用于暴露**可以在构建期确定**的公开变量。与 `runtimeConfig` 选项相反，这些**不能**使用环境变量覆盖。

所以这两个的选择判据只有一条：**这个值要不要在部署时换掉？**

- 要换（密钥、每个环境不同的地址）→ `runtimeConfig`，靠 `.env` 覆盖。
- 不换（主题色、站点名这类公开常量）→ `app.config.ts`。

把密钥写进 `app.config.ts` 是危险操作：它是构建期确定、面向客户端的公开变量，而官方从没说它能被环境变量覆盖。

## 读配置文档的正确方式

官方在配置那一节给了一句指向：

> 每一个选项都描述在 **Configuration Reference** 里。

这句话值得当成一条操作纪律：**不要凭记忆写配置项的名字。** 官方对"拼错"能给的唯一防线就是那条 IDE 类型提示，而参考页里每个键都写着类型和默认值——你要改的一切实值都应该从那里抄，而不是从别人的博客里抄。本套教程里每一个配置项我都会给出它的默认值来源。

## 自查

- [ ] 遇到"没生效"的问题时，你会先判断该改文件位置还是改配置吗？
- [ ] 知道 `defineNuxtConfig` 不需要 import，也知道配置键名拼错时文档并没有承诺任何具体报错——唯一有依据的防线是 `.ts` 的类型提示。
- [ ] 你的 `nuxt.config` 是 `.ts` 后缀吗（为了 IDE 提示）？
- [ ] 需要按环境差异时，你会用 `$production`/`$env` + `--envName`，而不是在代码里到处判 `process.env.NODE_ENV` 吗？
- [ ] 需要部署时更换的值，是否放在 `runtimeConfig`（可被环境变量覆盖）而不是 `app.config.ts`？
- [ ] 写配置项之前，是否去 Configuration Reference 核对过键名、类型与默认值？

## 下一篇

006 - 代码风格这一页在讲什么：官方推荐的检查工具与规则来源

## 本文依据

- `getting-started/configuration`（"用合理默认值配置让你高效"、默认已覆盖大多数场景、`nuxt.config.ts` 位于根目录用于覆盖或扩展应用行为、最小写法与 `defineNuxtConfig` 全局可用无需 import、强烈建议 `.ts` 扩展名以获 IDE 提示、`$production`/`$development`/`$env` 环境覆盖与 `--envName`、c12 环境相关配置说明、`$meta` 用于 layer、`runtimeConfig` 默认仅服务端可见且 public/app 暴露客户端且可被环境变量覆盖（含 apiSecret + `.env` 的 NUXT_API_SECRET 例子）、`app.config.ts` 暴露构建期可确定的公开变量且不能被环境变量覆盖、"每个选项都描述在 Configuration Reference 里"）
- `getting-started/introduction`（约定 + 有主见目录结构自动化重复工作、配置仍可定制并覆盖默认行为）
- `directory-structure`（`app/` 各目录与 `app.config.ts`/`app.vue`/`error.vue` 三个文件的职责、`nuxt.config.ts`/`.nuxtrc`/`.nuxtignore`）
