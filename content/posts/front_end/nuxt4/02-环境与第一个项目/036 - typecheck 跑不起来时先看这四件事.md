---
url: /posts/6941f7b0cb71176d786840f32d1e4b6e/
title: typecheck 跑不起来时先看这四件事
date: 2026-10-24T09:00:00+08:00
updated: 2026-10-24T09:00:00+08:00
author: cmdragon

summary:
  摘要：nuxt typecheck 失败通常不是"类型真的写错了"，而是四件前置事项没到位。第一件是检查器本身：它跑 vue-tsc 或 Golar，两者都没装时会给出对应包管理器的安装命令，而官方要求的开发依赖是 vue-tsc 与 typescript 两个。第二件是类型是否生成：那四份 tsconfig 只来自 nuxt dev、nuxt build 或 nuxt prepare，正常流程靠 package.json 里的 postinstall "nuxt prepare" 兜住。第三件是根目录 tsconfig.json 还是不是"只有 references 的那一份"：它决定 typecheck 是否自动用 build mode（--build 强制、--no-build 关闭），官方警告不要直接改它，要定制应走 typescript.tsConfig 与 appTsConfig/sharedTsConfig/nodeTsConfig/serverTsConfig，其中 lib、jsx、jsxImportSource 只会落到 tsconfig.app.json，而 types、paths、noEmit 由 Nuxt 按上下文管理、写在 tsConfig 里不会生效。第四件是代码所在上下文：app、node、server、shared 四份各自引用合适的依赖，把服务端专用代码放在 app 上下文里检查，报的是"找不到"而不是逻辑错误。

categories:
  - 前端开发

tags:
  - nuxt4
  - typecheck
  - 排错
  - typescript
  - tsconfig

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


`nuxt typecheck` 第一次跑，报错列表通常长得让人不想看第二眼。但**大部分那次失败不是类型写错了，而是四件前置条件没到位**。这一篇按这四件事给你一条排查顺序——每一步的依据都是文档里的原话，不是我猜的常见原因。

## 第一查：检查器到底装没装

035 讲过这条命令的本质：

> `typecheck` 命令运行 `vue-tsc` 或 Golar 来检查你整个应用的类型。**如果两者都没安装，会提供你所用包管理器对应的安装命令。**

排查动作有两个方向：

**方向一：官方给了安装命令 → 照它给的装。** 装的是**两个**开发依赖：

```bash [pnpm]
pnpm add -D vue-tsc typescript
```

（npm/yarn/bun/deno 的写法在 035 里五种都列了。）

**方向二：想改用另一个检查器。** `--checker=<vue-tsc|golar>` 可以显式指定。这里有个**很容易误判的现象**要提前知道：

> `--checker golar` 运行 Golar 而不是 `vue-tsc`，而你项目里存在一个 `golar.config.*` 文件会让它成为默认。**如果缺这个文件，会替你写一份。**

也就是说：**"我的项目里什么时候多了个 golar.config？"** 答案很可能是你（或同事）跑过一次 Golar 那条路。**排查"我什么都没改为什么检查行为变了"时，先看看这个文件在不在。**

## 第二查：`.nuxt` 里有没有那四份 tsconfig

这一条是**最常见的 CI 失败原因**。官方说得很死：

> Nuxt 项目依赖自动生成的类型才能正常工作。这些类型存放在 `.nuxt` 目录里，**在你运行开发服务器或构建应用时生成**。你也可以通过运行 `nuxt prepare` 手动生成这些文件。

四份文件的名字（031 抄过一次）：

- `.nuxt/tsconfig.app.json` — `app/` 目录里的应用代码
- `.nuxt/tsconfig.node.json` — `nuxt.config.ts` 与其它上下文之外的文件
- `.nuxt/tsconfig.server.json` — 服务端代码（适用时）
- `.nuxt/tsconfig.shared.json` — app 与 server 之间共享的代码

**如果这四份不存在，`typecheck` 检查的就不是你以为的东西。** 而它们只在 `dev`、`build`、`prepare` 三个命令里产生。

排查动作（按成本从低到高）：

1. 本地先 `npx nuxt prepare` 一次，再 `npx nuxt typecheck`。**这两步的顺序不能反**——040 会看到 `nuxt cleanup` 把 `.nuxt` 整个清掉，清完不 prepare 就跑检查，是这个坑的人造版本。
2. CI 里检查是否真的执行了安装（正常情况由 `postinstall: "nuxt prepare"` 自动兜住）；被跳过就得显式加一步 `nuxt prepare`。

## 第三查：根目录那份 `tsconfig.json` 被改过没有

先看它**应该**长什么样（官方给的形态）：

```json [tsconfig.json]
{
  "files": [],
  "references": [
    {
      "path": "./.nuxt/tsconfig.app.json"
    },
    {
      "path": "./.nuxt/tsconfig.server.json"
    },
    {
      "path": "./.nuxt/tsconfig.shared.json"
    },
    {
      "path": "./.nuxt/tsconfig.node.json"
    }
  ]
}
```

这份文件同时决定了两件事，**都被 035 讲过**：

**一、它是"solution-style"，会被自动用 build mode 检查。**

> 一个 solution-style 的 `tsconfig.json`（只列出 `references` 的那一份）会自动以 build mode 进行类型检查。`--build` 强制开启，`--no-build` 关掉它。

**所以如果你往这份文件里塞了 `compilerOptions` 或 `include`，它就不再是"只列 references"的那一份了**——检查模式可能随之变化。这类改动的症状通常是"报错数量和以前完全不同"，而不是某一行类型错。

**二、官方明确不让改它。**

> 我们不建议直接修改这个文件的内容，因为这样做可能覆盖 Nuxt 或其它模块依赖的重要设置。请改为通过 `nuxt.config.ts` 扩展它。

正确的定制入口（官方给的完整一份）：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  typescript: {
    // 给每一份生成的 tsconfig 共享的 compilerOptions
    tsConfig: {
      compilerOptions: {
        // ...
      },
    },
    // 定制 tsconfig.app.json
    appTsConfig: {
      // ...
    },
    // 定制 tsconfig.shared.json
    sharedTsConfig: {
      // ...
    },
    // 定制 tsconfig.node.json
    nodeTsConfig: {
      // ...
    },
    // 定制 tsconfig.server.json
    serverTsConfig: {
      // ...
    },
  },
})
```

**然后是这一篇最容易救你半天的一段。** 官方加了一条 note，说共享那份 `tsConfig` 有例外：

> `typescript.tsConfig` 里设置的多数 `compilerOptions` 会与每个上下文共享，但**有少数不是**。像 `lib`、`jsx`、`jsxImportSource` 这些 DOM 与 Vue 专属的选项，只对你的应用代码有意义，因此**它们只应用到 `tsconfig.app.json`**。Nuxt 还**按上下文管理 `types`、`paths` 与 `noEmit`**（`node`、`shared`、`server` 三份刻意不产出任何内容、也不扫描环境类型），所以**在 `typescript.tsConfig` 里设置它们不会改变它们**。需要覆盖时请用对应的按上下文选项（`appTsConfig`、`nodeTsConfig`、`sharedTsConfig` 或 `serverTsConfig`）。

把这段翻成排查动作，是三条很具体的判断：

- 你写在 `typescript.tsConfig` 里的 `lib` **不会**进到 server 上下文——想让服务端认某个全局，得去 `serverTsConfig`。
- 你写在 `typescript.tsConfig` 里的 `types`、`paths`、`noEmit` **是无效改动**（官方写着"不会改变它们"）。**"我明明配了怎么没反应"就是这条。**
- 另一条同类：`typescript.serverTsConfig` 扩展的是 `tsconfig.server.json`；`nitro.typescript.tsConfig` 与它保持同步但**已废弃**，官方要求改设 `typescript.serverTsConfig`。**如果你在项目里看到 `nitro.typescript.tsConfig`，那就是旧写法。**

## 第四查：报错的代码在哪个上下文里

如果前三项都正常，报的是"找不到名字""属性不存在"这类，那**很可能不是类型写错，是文件放错位置**。官方给的四份配置的分工是：

> 每一份都被配置为引用其合适的依赖，**为它们各自的上下文提供最优的类型检查**。

这就是 031 那条现象的机制版本：**同一个名字在 `app/pages/` 里能解析、在 `server/api/` 里不能**（或者反过来），因为这两段代码根本不在同一份 tsconfig 下被检查。对照 031 那张表先定位文件属于哪一份，再看这个位置能不能用那个 API——**这一步查的是"位置"，不是"写法"。**

（自动导入的边界、`shared/` 与 `server/` 各自的可见范围属于 065、066、151 那几篇；这里只用作排查方向。）

## 收尾一个跑法差异：`typecheck` 是 production

前三查都过了但**"本地不报、CI 报"或反过来"**，看这条：

> 本命令把 `process.env.NODE_ENV` 设为 `production`。要覆盖它，请在 `.env` 文件里定义 `NODE_ENV`，或作为命令行参数传入。

而 `nuxt dev` 设的是 `development`（034）。**只要你的 `nuxt.config` 或生成的类型会因为环境不同而不同，这两个环境检查的就不是同一套代码路径。** 另外记着 035 那条差异：`typecheck` 有 `--dotenv`、**没有** `--envName`，所以要控制它读哪些 `.env` 只能靠 `--dotenv` 或环境变量本身。

## 自查

- [ ] `typecheck` 需要哪两个开发依赖？两者都没装时命令会做什么？
- [ ] 项目里出现 `golar.config.*` 文件说明什么？
- [ ] `.nuxt` 里那四份 tsconfig 由哪三个命令生成？正确的两步跑法顺序是什么？
- [ ] 你写在 `typescript.tsConfig` 里的 `lib`、`types`、`noEmit` 各自会发生什么？
- [ ] "本地不报 CI 报"这一类，`NODE_ENV` 能解释吗？

## 下一篇

037 进入构建命令：`nuxt build` 产出什么、`--target` 与 `--preset` 谁是正式名字、`--prerender` 会强制把 target 改成什么，以及不传 target 时官方按哪两个环境变量决定。

## 本文依据

- `api/commands/typecheck`（typecheck 运行 vue-tsc 或 Golar 检查整个应用类型、两者都没安装时提供你所用包管理器的安装命令；--checker=<vue-tsc|golar>；Type checkers 一节 --checker golar 与 golar.config.* 文件使其成为默认、缺文件会替你写一份；Build mode 一节 solution-style tsconfig.json（只列 references）自动以 build mode 检查、--build 强制、--no-build 关闭；note 设 NODE_ENV 为 production，可在 .env 或命令行参数覆盖；选项含 --dotenv 而无 --envName）
- `guide/concepts/typescript`（默认 nuxt dev 与 nuxt build 出于性能不检查类型；需装 vue-tsc 与 typescript 开发依赖并给出五种包管理器命令；npx nuxt typecheck；typescript.typeCheck: true；Auto-generated Types：类型存于 .nuxt 并在 dev 或构建时生成、可用 nuxt prepare 手动生成、生成的 tsconfig 含推荐基础配置与自动导入/API 路由类型引用与路径别名、warning 不建议直接改 tsconfig.json 应通过 nuxt.config.ts 扩展；Project References：四份文件各自覆盖 app/node/server/shared 且每份引用合适依赖为该上下文提供最优类型检查）
- `directory-structure/tsconfig`（根 tsconfig.json 应为 files: [] 加四条 references；warning 不建议直接修改内容以免覆盖 Nuxt 或其它模块依赖的重要设置、应通过 nuxt.config.ts 扩展；Extending 一节 typescript.tsConfig 为所有上下文共享 compilerOptions 且可用 appTsConfig/sharedTsConfig/nodeTsConfig/serverTsConfig 分别覆盖；note lib/jsx/jsxImportSource 只应用到 tsconfig.app.json、Nuxt 按上下文管理 types/paths/noEmit（node/shared/server 三份刻意不产出且不扫描环境类型）因此写在 typescript.tsConfig 不会改变它们需用对应按上下文选项；note typescript.serverTsConfig 扩展 tsconfig.server.json、nitro.typescript.tsConfig 保持同步但已废弃应改设 typescript.serverTsConfig）
- `api/commands/prepare`（创建 .nuxt 并生成类型、CI 或 postinstall 场景有用、设 NODE_ENV 为 production）
- `api/commands/dev`（本命令把 process.env.NODE_ENV 设为 development）
- `directory-structure/package`（最小 package.json 的 postinstall 为 nuxt prepare）
