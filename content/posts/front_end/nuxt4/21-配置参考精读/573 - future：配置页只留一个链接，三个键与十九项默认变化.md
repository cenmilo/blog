---
url: /posts/335063b21441f0e5961513ac357cd08c/
title: future：配置页只留一个链接，三个键与十九项默认变化
date: 2027-04-21T09:00:00+08:00
updated: 2027-04-21T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：配置参考里的 `## future` **一句正文都没有**，只有一个 read-more，而且它和 `## features` 指向**同一个锚点** `guide/going-further/features#features`。这个命名空间的真实内容在特性页的 `## future` 小节，一共**三个键**：`compatibilityVersion`、`multiApp`、`typescriptBundlerResolution`，**三个键都没有类型行、没有默认值行**。其中 `compatibilityVersion: 5` 的影响面最大——升级页替它列了一张**十九项**的清单，说的是"在 Nuxt 4 上把这一键设成 5，这十九处默认会跟着变"。

categories:
  - 前端开发

tags:
  - nuxt4
  - future命名空间
  - 兼容性版本
  - 提前采用
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

听说可以提前试下一个大版本的行为，你会去配置参考里搜 `future`。搜到的是这样一节：标题、一行空白、一个链接。**这个命名空间能配什么、默认值是什么，配置页一个字都没写。**

## 配置页这一节的全部

`## future` 整节只有一条 read-more：

> Learn more about opting-in to new features that will become default in a future (possibly major) version of the framework.

链接目标是 `https://nuxt.com/docs/4.x/guide/going-further/features#features`。**这一节既没有定性句，也没有类型行**——对比隔壁 `## runtimeConfig`（五句正文 ＋ `Type` ＋ 默认值 JSON ＋ 示例），这一节属于最薄的一档：**只有跳转，没有记载**。

还有一处会误导人：`## features` 那一节的 read-more **指向同一个锚点**。也就是说从配置页点过来，`features` 和 `future` 都落在特性页的 `## features` 小节上，你要自己往下滚才能看到 `## future`。这两节的分工只在各自指向的那句话里体现（"opt-in features" 与 "will become default in a future version"）。

## 三个键的记载只有正文句子

内容都在特性页的 `## future` 小节，它的开场句是给整个命名空间定的性：

> There is also a `future` namespace for early opting-in to new features that will become default in a future (possibly major) version of the framework.

然后是三个 `###` 小节。**三个都没有 `**Type**` 行、没有 `**Default**` 行**，取值只能从正文和示例里读。

### compatibilityVersion

官方写法：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  future: {
    compatibilityVersion: 5,
  },
})
```

特性页对它的说明是两句："This enables early access to Nuxt features or flags."，以及 "Setting `compatibilityVersion` to `5` changes defaults throughout your Nuxt configuration to opt in to Nuxt v5 behaviour"，并点名它包含启用 Vite Environment API。升级页补了使用前提：**从 Nuxt 4.2 起**可以这么试 Nuxt 5 的许多破坏性变更。

### multiApp

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  future: {
    multiApp: true,
  },
})
```

说明只有一句："This enables early access to the experimental multi-app support."，并给了跟踪这次支持的 issue 链接（nuxt/nuxt **#21635**）。**多应用支持到底意味着什么，这一节没展开**——它只在别处以 `appId` 的措辞（"For multi-app projects, the unique id of the Nuxt application"）出现过。

### typescriptBundlerResolution

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  future: {
    typescriptBundlerResolution: false,
  },
})
```

这一条的说明最完整：它打开 TypeScript 的 **Bundler** 模块解析模式，官方写明这是 Nuxt、Vite 这类框架的推荐设置，好处是"用带 `exports` 的现代库时类型支持更好"；并且**明确说了反方向**："You can set it to false to use the legacy 'Node' mode, which is the default for TypeScript."（示例正是给的 `false`。）

## 十九项：compatibilityVersion 到底改了什么

这一节真正需要逐条看的是升级页 "Testing Nuxt 5" 那张清单。官方给的前提句是 "Defaults throughout your Nuxt configuration then change to opt in to Nuxt 5 behaviour. On Nuxt 4 this covers:"——**下面十九项是这份清单的全部**：

| # | 官方条目 | 变化 |
| --- | --- | --- |
| 1 | Vite Environment API | `experimental.viteEnvironmentApi` 开启，Vite 插件的注册与配置方式随之改变 |
| 2 | Case-sensitive routing | 页面路由严格按大小写匹配，与 Nitro 一致，`/About` 不再匹配 `pages/about.vue` |
| 3 | Normalized page names | 页面组件名与路由名一致（`pages/foo/index.vue` 的组件名从 `index` 变成 `foo`），为的是 `<KeepAlive>` 行为一致 |
| 4 | Serializable page meta | `experimental.extractSerializablePageMeta` 在建时提取可静态分析的 `definePageMeta` 值 |
| 5 | Client-side payload extraction | `experimental.payloadExtraction` 变成 `'client'`，把 payload 内联进预渲染页的 HTML |
| 6 | `clearNuxtState` resets to defaults | 清状态后回到 `useState` 的初始值，而不是设成 `undefined` |
| 7 | Non-async `callHook` | `callHook` 可能返回 `void`，不再总是返回 `Promise` |
| 8 | Early return from `navigateTo` | `experimental.navigateToEarlyReturn` 让中间件重定向后不再继续往下跑 |
| 9 | Server-only head composables | `useServerHead`、`useServerHeadSafe`、`useServerSeoMeta` 不再自动导入；`unhead.legacy` 被忽略、Unhead 的 legacy 插件不再注册、Capo 排序常开 |
| 10 | PostCSS defaults off | 除非自己在 `postcss.plugins` 里开，`autoprefixer` 与 `cssnano` 不再被配置 |
| 11 | Keyed composables require `source` | `optimization.keyedComposables` 条目若没有字符串 `source`，就不再通过自动导入名匹配 |
| 12 | Comment node placeholders | client-only 组件用注释节点而不是 `<div>` 作为 SSR 占位 |
| 13 | Typed pages | `experimental.typedPages` 开启，路由带类型检查 |
| 14 | Error response format | `experimental.inlineErrorRendering` 开启：页面错误返回错误页、服务端路由错误返回 JSON，**与请求头无关** |
| 15 | Typed `$fetch` | `experimental.routeTypedFetch` 用生成的路由类型为 `$fetch` 与 `useFetch` 定返回值 |
| 16 | Nitro auto-imports off | 服务端代码显式 import Nitro 的 helper，不再依赖自动导入兼容层 |
| 17 | Vue Options API disabled | Options API 从客户端包里编译掉以减小体积 |
| 18 | Stricter side-effect imports | 生成的 `tsconfig.json` 打开 `noUncheckedSideEffectImports` |
| 19 | TypeScript `baseUrl` ignored | 生成的 TypeScript 配置不再用 `compilerOptions.baseUrl` 解析 Nuxt 别名 |

清单后面紧跟一条 `::note`，范围划得很清楚：**这十九项说的是"在 Nuxt 4 上把这个 flag 打开会改什么"**，Nuxt 5 本身还带着**任何 flag 都无法在这里打开**的变更，例如升级到 Nitro v3。

## 这一键与其他两类的分工，文档没有对照表

`future`（按行为名提前采用）、`compatibilityDate`（按日期锁定预设与模块的默认行为，见 `api/nuxt-config` 那一节和报错页 B5001）、`experimental`（逐个开关）三者都在"控制默认行为"，但**官方没有一处把三者对照着排过序**。能确证的只有交叉点上那几句：

- 第 1、4、5、8、13、15 项在特性页里以 `experimental.<名字>` 的形式存在，即这一键会去开那些实验开关。
- 个别实验开关的页面写着 "This flag is enabled when `future.compatibilityVersion` is set to `5` **or higher**, but you can also enable it explicitly"，另一处写着 "…but you can disable this feature"——**所以单项能不能改回来，文档是逐个开关分别说的，没有一张总表**。
- 取值只给过 `5`。**能不能写 `4`、能不能写更大的数、写了非数字会怎样，没有记载**（"or higher" 暗示存在更高档位，但没给例子）。

## 自查

- 配置页 `## future`：**无正文、无 Type、无 Default**，只有一个 read-more，且与 `## features` 同锚点。
- 真实内容在特性页 `## future`，**三个键**：`compatibilityVersion`、`multiApp`、`typescriptBundlerResolution`，三者都没有类型行与默认值行。
- `compatibilityVersion: 5` 需 **Nuxt 4.2+**，作用是把配置面的默认成批改成 Nuxt 5 行为；**清单十九项**，且官方声明这只是一部分（Nitro v3 之类的变更 flag 开不出来）。
- `multiApp` 只有一句 early access + tracker issue #21635；`typescriptBundlerResolution` 是唯一写清反方向的（`false` = TypeScript 默认的 legacy Node 模式）。
- 单项能否覆盖回去：文档按开关分别写，无总表；`compatibilityVersion` 的合法取值除 `5` 外无记载。

## 下一篇

下一篇 574 看隔壁那个同样只有链接的 `## features` 节：它的三个开关（`devLogs`、`inlineStyles`、`noScripts`）**在特性页里也一律没有类型行与默认值行**，而其中两个的适用范围是带条件的——`inlineStyles` 只在用 Vite 时可用的说法写在正文里，`noScripts` 的 `true` 会被**归一化成 `'production'`**。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## future` 与 `## features` 两条 read-more 指向同一锚点；`## appId` 里 "For multi-app projects" 那句）
- https://nuxt.com/docs/4.x/guide/going-further/features（`## future` 开场句、`compatibilityVersion` 两句说明与示例、指向 getting-started/upgrade#testing-nuxt-5 的 read-more、`multiApp` 说明与 issue #21635、`typescriptBundlerResolution` 全段与 `false` 示例）
- https://nuxt.com/docs/4.x/getting-started/upgrade（"From Nuxt 4.2+" 前提句、十九项清单逐条原文、"On Nuxt 4 this covers" 定性与末尾 `::note`）
- https://nuxt.com/docs/4.x/guide/going-further/experimental-features（个别开关 "enabled when `future.compatibilityVersion` is set to `5` or higher" 与可显式开启/关闭的措辞）
