---
url: /posts/53bdbabb2db0c11506c88402de83fb29/
title: nuxt prepare 与 nuxt typecheck：让类型先就绪
date: 2026-10-23T11:00:00+08:00
updated: 2026-10-23T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：这两个命令管的是两件不同的事，很容易混。nuxt prepare 创建 .nuxt 目录并生成类型，官方明确它"在 CI 环境里或作为 package.json 的 postinstall 命令时很有用"，它只有五个选项（--cwd、--dotenv、--logLevel、--envName、-e/--extends）且把 process.env.NODE_ENV 设为 production。nuxt typecheck 才真正检查类型，它跑 vue-tsc 或 Golar，两者都没装时会给出你所用包管理器的安装命令；--checker 可指定用哪个检查器，项目里存在 golar.config.* 文件会让 Golar 成为默认（缺文件时会自动写一份）；只列 references 的 solution-style 根 tsconfig 会自动以 build mode 类型检查，--build 强制开启、--no-build 关闭，它同样把 NODE_ENV 设为 production。而官方在 TypeScript 那一页开头就说：默认情况下 nuxt dev 与 nuxt build 出于性能原因不做类型检查，要开启要么装 vue-tsc 与 typescript 后跑 nuxt typecheck，要么设 typescript.typeCheck: true。

categories:
  - 前端开发

tags:
  - nuxt4
  - typescript
  - prepare
  - typecheck
  - 命令行

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


031 讲过编辑器里的类型来自 `.nuxt` 里那四份生成的 tsconfig。这一篇讲两件配套的事：**怎么把类型生成出来**（`nuxt prepare`），以及**怎么真的检查它们**（`nuxt typecheck`）。

先给一句最重要的判断：**这两个命令都不能靠"我跑过 dev"来互相替代。**

## 默认情况：dev 和 build 都不检查类型

官方在 TypeScript 页开头就写了这句，值得贴在工位上：

> 默认情况下，Nuxt 在运行 `nuxt dev` 或 `nuxt build` 时**不检查类型**，原因是性能。

也就是说，你的项目能起、能构建、甚至能上线，**跟类型对不对没有关系**。要检查只有两条路：手动跑 `nuxt typecheck`，或者打开配置项 `typescript.typeCheck`。

## `nuxt prepare`：只生成，不检查

官方定义：

> `prepare` 命令在你的应用里创建一个 [`.nuxt`](https://nuxt.com/docs/4.x/directory-structure/nuxt) 目录并生成类型。**这在 CI 环境里，或者作为 `package.json` 的 `postinstall` 命令时很有用。**

用法行（选项很少）：

```bash [Terminal]
npx nuxt prepare [ROOTDIR] [--cwd=<directory>] [--dotenv=<path>...] [--logLevel=<silent|info|verbose>] [--envName=<environment>] [-e, --extends=<layer-name>...]
```

官方在末尾留了一条 note，**和 dev 相反**：

> 本命令把 `process.env.NODE_ENV` 设为 `production`。

三个要记住的点：

**一、它的产物是"类型就绪"，不是"能跑"。** `.nuxt` 里生成的是给编辑器和 `typecheck` 用的东西；052 会讲这个目录本身在 `nuxt dev` 时会被整体重建。

**二、`postinstall` 那条不是可选装饰。** 024 看过脚手架的 `package.json`：

```json [package.json]
{
  "scripts": {
    "postinstall": "nuxt prepare"
  }
}
```

**这就是为什么正常 clone + install 之后，编辑器就已经认识 `useFetch` 了**——安装依赖时它自动跑了一次。反过来说：如果你绕过包管理器手工挪 `node_modules`、或者在 CI 里跳过 install，就得显式跑一次 `nuxt prepare`。

**三、它按 `production` 解析配置。** 如果你用 `typescript.tsConfig` 之外的东西影响生成（比如按 `envName` 分支的配置，或者 `.env` 里的值会影响生成的类型），那 `nuxt prepare` 看到的和你 `nuxt dev` 看到的**不是同一套环境**。这一条是 036 里排查"CI 通过本地不通过"的入口之一。

## `nuxt typecheck`：真的检查

官方定义里有个少见的东西——它给了**两个**检查器：

> `typecheck` 命令运行 [`vue-tsc`](https://github.com/vuejs/language-tools/tree/master/packages/tsc) 或 [Golar](https://golar.dev/languages/vue/) 来检查你整个应用的类型。**如果两者都没安装，会提供你所用包管理器对应的安装命令。**

用法行：

```bash [Terminal]
npx nuxt typecheck [ROOTDIR] [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--dotenv=<path>...] [-e, --extends=<layer-name>...] [--checker=<vue-tsc|golar>] [-b, --build]
```

### 前置：装两个 devDependency

官方给的安装命令是按包管理器分组的（这是 `typescript` 页的原文，五种都有）：

```bash [npm]
npm install --save-dev vue-tsc typescript
```

```bash [yarn]
yarn add --dev vue-tsc typescript
```

```bash [pnpm]
pnpm add -D vue-tsc typescript
```

```bash [bun]
bun add -D vue-tsc typescript
```

```bash [deno]
deno add -D npm:vue-tsc npm:typescript
```

注意**要装两个**：`vue-tsc`（能读 `.vue` 文件的 tsc）和 `typescript` 本身。官方在 `typecheck` 页说的是"两者都没装会给你安装命令"，所以只装一个也可能报你意料之外的东西。

装完之后：

```bash [Terminal]
npx nuxt typecheck
```

### 用哪个检查器：`--checker`

> `--checker golar` 运行 [Golar](https://golar.dev/languages/vue/) 而不是 `vue-tsc`，而**你项目里存在一个 `golar.config.*` 文件会让它成为默认。如果缺这个文件，会替你写一份。**

这一条有个副作用值得知道：**第一次用 Golar 时，你的项目里会多出一个配置文件**（官方明说"a config file is written for you if one is missing"）。看到它不要当病毒，那是这条命令留下的。

### build mode：那份"只有 references"的 tsconfig

官方说明：

> 一个 solution-style 的 `tsconfig.json`（**只列出 `references` 的那一份**）会自动以 build mode 进行类型检查。`--build` 强制开启，`--no-build` 关掉它。

对照 031 讲的 project references，根目录那份 `tsconfig.json` 官方给的标准形态正好就是这个样子：

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

官方在这个文件下面还挂着一条 warning：

> 我们不建议直接修改这个文件的内容，因为这样做可能覆盖 Nuxt 或其它模块依赖的重要设置。**请改为通过 `nuxt.config.ts` 扩展它。**

所以：**这份文件"长这样"是正常状态，不是"空的、坏了"。** 它的作用就是把四个上下文挂进来，而 `typecheck` 靠认出它来决定用不用 build mode。

### 它也是 production

`typecheck` 页末尾同样有一条：

> 本命令把 `process.env.NODE_ENV` 设为 `production`。要覆盖它，请在 [`.env`](https://nuxt.com/docs/4.x/directory-structure/env) 文件里定义 `NODE_ENV`，或作为命令行参数传入。

对比一下三个命令的默认值，这就是这一节的全部要点：

| 命令 | NODE_ENV 默认 |
| --- | --- |
| `nuxt dev` | `development` |
| `nuxt prepare` | `production` |
| `nuxt typecheck` | `production`（可用 `.env` 或命令行覆盖） |

顺带注意：`typecheck` 的选项里**有 `--dotenv`，但没有 `--envName`**（`prepare` 两个都有）。也就是说这两个命令读 `.env` 的能力一致，但"用哪个环境解析配置覆盖"这一项，`typecheck` 没给开关。

## 想在 build 或 dev 期间就检查：配置项

官方的第二条路：

> 要在构建或开发期间启用类型检查，你也可以使用 `nuxt.config` 文件里的 [`typescript.typeCheck`](https://nuxt.com/docs/4.x/api/nuxt-config#typecheck) 选项：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  typescript: {
    typeCheck: true,
  },
})
```

这是个**取舍**，不是"更好的默认"：开头那句"默认不检查"给的理由是性能。把它打开，等于让每次 dev 与 build 都付这笔钱。我的读法是：**CI 里用 `nuxt typecheck` 单独一步，比在本地常开 `typeCheck: true` 更符合官方给的分工**（这是我的建议；官方只给了两个都存在的选项）。

## 自查

- [ ] 你说得出 `prepare` 与 `typecheck` 各自做什么、不做什么吗？
- [ ] `nuxt dev` 通过了，能推出类型也是对的吗？为什么？
- [ ] 装 `vue-tsc` 还要不要装 `typescript`？两者都没装时命令会怎么帮你？
- [ ] 根目录那份只有 `references` 的 tsconfig 是不是坏了？它让 `typecheck` 做了什么决定？
- [ ] `prepare` 和 `typecheck` 把 NODE_ENV 设成什么？这跟 `dev` 有什么不同后果？

## 下一篇

036 处理 `typecheck` 跑不动的四类原因：检查器没装、类型没生成、根 tsconfig 被手改过、以及你把代码放错了类型上下文。

## 本文依据

- `api/commands/prepare`（prepare 创建 .nuxt 目录并生成类型、在 CI 环境或作为 package.json 的 postinstall 命令时很有用；用法与选项 ROOTDIR/--cwd/--dotenv（可重复靠后优先）/--logLevel/--envName/-e extends；note 本命令设 process.env.NODE_ENV 为 production）
- `api/commands/typecheck`（typecheck 运行 vue-tsc 或 Golar 检查整个应用类型、两者都没安装时提供你所用包管理器的安装命令；用法与选项 ROOTDIR/--cwd/--logLevel/--dotenv/-e extends/--checker=<vue-tsc|golar>/-b --build；Type checkers 一节 --checker golar 跑 Golar 而项目里存在 golar.config.* 文件使其成为默认、缺文件会替你写一份；Build mode 一节 solution-style tsconfig.json（只列 references）自动以 build mode 检查、--build 强制、--no-build 关闭；note 本命令设 NODE_ENV 为 production，可在 .env 或命令行参数覆盖；read-more 指向 typescript#type-checking）
- `guide/concepts/typescript`（Type-checking 一节："By default, Nuxt doesn't check types when you run nuxt dev or nuxt build, for performance reasons"；启用需把 vue-tsc 与 typescript 装为开发依赖并给出 npm install --save-dev / yarn add --dev / pnpm add -D / bun add -D / deno add -D npm:vue-tsc npm:typescript 五种命令；随后 npx nuxt typecheck；也可用 typescript.typeCheck: true 在构建或开发期启用；Auto-generated Types 一节类型存放于 .nuxt 并在 dev 或 build 时生成、可运行 nuxt prepare 手动生成、warning 不建议直接修改 tsconfig.json 而应通过 nuxt.config.ts 扩展）
- `directory-structure/tsconfig`（Nuxt 自动生成 .nuxt/tsconfig.app.json、tsconfig.server.json、tsconfig.node.json、tsconfig.shared.json 四份，含推荐基础配置、自动导入与 API 路由类型引用与路径别名；项目根应包含 files: [] 加四条 references 的 tsconfig.json；warning 不建议直接修改该文件内容以免覆盖 Nuxt 或模块依赖的重要设置，应通过 nuxt.config.ts 扩展）
- `directory-structure/package`（最小 package.json 的 scripts 含 postinstall: "nuxt prepare"）
