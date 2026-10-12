---
url: /posts/09f831157b039e86736b2b35d53980d4/
title: Nuxt 的 TS 支持从哪来：.nuxt 里的 tsconfig 与生成类型
date: 2027-02-20T11:00:00+08:00
updated: 2027-02-20T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：新建项目里 `app.vue` 没 import `ref` 也有类型、`useFetch` 不用引就能用、`#imports` 能跳转——这些不是编辑器聪明，是 Nuxt 在 `.nuxt` 目录里**生成**了类型与 tsconfig。官方原话："Nuxt projects rely on auto-generated types to work properly."，生成时机是跑 dev server、构建、或手动 `nuxt prepare`。Nuxt 用 TypeScript **project references** 把项目切成四份配置：`.nuxt/tsconfig.app.json`（app 目录）、`tsconfig.node.json`（nuxt.config 与其它上下文外文件）、`tsconfig.server.json`（服务端）、`tsconfig.shared.json`（app 与 server 共享）；根 `tsconfig.json` 只有 `files: []` + 四条 `references`。一条重要警告：别直接改生成的 tsconfig，要扩展走 `nuxt.config`。

categories:
  - 前端开发

tags:
  - nuxt4
  - TypeScript
  - tsconfig
  - project references
  - nuxt prepare
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

刚接触 Nuxt + TypeScript 的人常有两个困惑：一是"我没 import `ref`、没 import `useFetch`，编辑器怎么知道它们存在"；二是"项目根那个 `tsconfig.json` 里 `"files": []`、只写了四条 references，是不是配错了"。两个答案都在 `.nuxt` 目录里。

## 类型是生成的，不是手写的

官方 `guide/concepts/typescript` 开头就定性：

> Nuxt projects rely on auto-generated types to work properly. These types are stored in the `.nuxt` directory and are generated when you run the dev server or build your application. You can also generate these files manually by running `nuxt prepare`.

三件事：类型存在 `.nuxt`；生成时机是 **dev server / build**；想手动生成，跑 **`nuxt prepare`**。

这就是第一个困惑的答案：自动导入的 `ref`、`useFetch`、`~/` 别名这些，都是生成物里声明好的。刚 clone 项目、还没跑过任何命令时 `.nuxt` 是空的，编辑器就会一片红——`nuxt prepare` 一下就有了（这也是 `package.json` 里 `postinstall: nuxt prepare` 的意义，见 075）。

## 生成的那四份 tsconfig

Nuxt 用 TypeScript **project references** 把项目切成几块分别检查（官方："Nuxt uses TypeScript project references to improve type-checking performance and provide better IDE support."）。跑 `nuxt dev`/`nuxt build`/`nuxt prepare` 时会生成多份 `tsconfig.json`：

| 文件 | 管什么 |
| --- | --- |
| `.nuxt/tsconfig.app.json` | `app/` 目录里的应用代码 |
| `.nuxt/tsconfig.node.json` | `nuxt.config.ts` 与其它不属于上述上下文外的文件 |
| `.nuxt/tsconfig.server.json` | 服务端代码（适用时） |
| `.nuxt/tsconfig.shared.json` | app 与 server 共享的代码（类型、非环境相关工具） |

每份都指向各自的依赖，"provide optimal type-checking for their specific context"。

还有一份遗留文件：官方 `::note` 说为向后兼容 Nuxt 仍生成 `.nuxt/tsconfig.json`，但推荐改用新的 project references 那几份，且**这个遗留文件未来会被移除**。

## 根 tsconfig.json 为什么只有四条引用

项目根那一份长这样（tsconfig 那页给的必需形态）：

```json [tsconfig.json]
{
  "files": [],
  "references": [
    { "path": "./.nuxt/tsconfig.app.json" },
    { "path": "./.nuxt/tsconfig.server.json" },
    { "path": "./.nuxt/tsconfig.shared.json" },
    { "path": "./.nuxt/tsconfig.node.json" }
  ]
}
```

`"files": []` 加四条 `references` **是设计不是坏了**：根文件本身不直接检查任何文件，只把四个上下文委托给 `.nuxt` 里生成的那四份。这样每个上下文各自有正确的 `lib`、types、paths，而不是全项目挤在一份配置里。

## 别直接改生成的配置

官方用 `::warning` 拦住最常见的错法：

> Nuxt relies on this configuration, and Nuxt modules can extend it as well. For this reason, it is not recommended to modify your `tsconfig.json` file directly, as doing so could overwrite important settings. Instead, extend it via `nuxt.config.ts`.

生成的 tsconfig 是产物，手改会被下次生成覆盖、还会破坏 Nuxt 与模块依赖的设置。要加编译器选项，走 `nuxt.config` 的 `typescript` 节（`tsConfig` 加共享项、`appTsConfig`/`serverTsConfig`/`nodeTsConfig`/`sharedTsConfig` 按上下文加）——397 专讲这一节。

## 为什么要切成四个上下文

官方列了 project references 四条好处：

- **Faster builds**：TS 能跳过没变的项目不重建
- **Better IDE performance**：IntelliSense 与报错更快
- **Isolated compilation**：一处报错不阻止其它部分编译
- **Clearer dependency management**：每个项目显式声明依赖

这直接推出一条实操规则，也是本系列反复强调的那点：**类型增强要放对目录**。官方 "Augmenting Types with Project References"：

> Since the project is divided into **multiple type contexts**, it's important to **augment types within the correct context** ... TypeScript will not recognize augmentations placed outside these directories unless they are explicitly included in the appropriate context.

- 要增强 `app` 上下文的类型 → 增强文件放 `app/`
- `server` 上下文 → 放 `server/`
- app 与 server 共享的 → 放 `shared/`

放错位置，类型就不被认。

## 严格检查从哪来

`typescript.strict`（官方 `## typescript` 里默认 `true`）与 strict checks 相关。typescript 那页说：

> [Strict checks] are enabled by default in Nuxt when the `typescript.typeCheck` option is enabled to give you greater type safety.

正在从 JS 往 TS 迁的项目想暂时关掉，就 `typescript: { strict: false }`。

## 自查

- 类型是 `.nuxt` 里的生成物；生成时机 dev/build，手动 `nuxt prepare`；刚 clone 报红先 prepare。
- 四份 tsconfig 对应四个上下文（app / node / server / shared）；`.nuxt/tsconfig.json` 是遗留、将来移除。
- 根 tsconfig `"files": []` + 四条 references 是设计，不是配错。
- 别手改生成的 tsconfig（`::warning`），扩展走 `nuxt.config` 的 `typescript` 节。
- 类型增强必须放对目录（app/server/shared），否则不被识别。

## 下一篇

下一篇 396 分清 Nuxt 里"类型自动给了"和"你要自己写"的两边：哪些由生成物覆盖、哪些需要显式声明（以及运行时配置的自动推型）。

## 本文依据

- "Nuxt projects rely on auto-generated types ... stored in the .nuxt directory ... generated when you run the dev server or build ... manually by running `nuxt prepare`"、生成的 tsconfig 含推荐基础配置+自动导入/API 路由类型/路径别名引用、不要手改 tsconfig 的 `::warning`、project references 四份文件与用途、遗留 `.nuxt/tsconfig.json` 将被移除的 `::note`、四条好处、"augment types within the correct context"（app/server/shared）、strict checks 随 `typescript.typeCheck` 默认启用（`guide/concepts/typescript`）；根 tsconfig `"files": []` 与四条 references 形态（`directory-structure/tsconfig`）。
