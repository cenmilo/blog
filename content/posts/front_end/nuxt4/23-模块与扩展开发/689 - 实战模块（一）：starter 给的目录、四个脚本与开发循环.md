---
url: /posts/af9e69fd5a0e36679ede0827256f8e3b/
title: 实战模块（一）：starter 给的目录、四个脚本与开发循环
date: 2027-05-29T11:00:00+08:00
updated: 2027-05-29T11:00:00+08:00
author: cmdragon

summary:
  摘要：从命令到第一个能跑的模块，官方推荐的路只有一条：用模块 starter 模板建项目，进去之后装依赖、跑 `npm run dev:prepare`，然后在 `playground` 里用 `npm run dev` 开发。这篇讲这条路上的四个目录各自管什么、四个脚本各自做了什么，以及为什么文档说"大多数时候你根本不用自己构建模块"。项目本地还有一个更省事的家：`modules/` 目录，放进去就自动注册，不用装包。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块开发
  - 模块starter
  - 开发流程
  - 本地模块
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 先决定一件事：这个模块要不要发出去

如果你的模块只服务当前项目，它不必成为一个 npm 包。项目根下建 `modules/` 目录，Nuxt 就按两种模式自动扫描注册：`modules/*/index.ts` 和 `modules/*.ts`。这种本地模块**不用再单独加进 `nuxt.config.ts`**，并且写它的时候 `nuxt/kit` 是 `@nuxt/kit` 的一个辅助子路径，意思是你不必为此把 `@nuxt/kit` 加进项目依赖。

要发给别人用，才需要 starter 建出来的独立仓库。这一步判断错的成本挺高：本地模块改起来只动一个目录，独立仓库要处理构建、发布、版本。

## starter 建出来的是什么

官方推荐用 starter 模板起步，四条命令按包管理器分：

```bash [npm]
npm create nuxt -- -t module my-module
```

```bash [yarn]
yarn create nuxt -t module my-module
```

```bash [pnpm]
pnpm create nuxt -t module my-module
```

```bash [bun]
bun create nuxt --template=module my-module
```

注意 bun 那条的写法不同：它用 `--template=module`，另外三条用 `-t module`，这是文档原样。

跑完得到一个 `my-module` 项目，里面是开发与发布这个模块所需的全部样板。文档给的接下来四步：

1. 用你惯用的 IDE 打开 `my-module`
2. 用你惯用的包管理器安装依赖
3. 用 `npm run dev:prepare` 准备本地文件
4. 继续读这份文档了解模块

第 3 步很多人会跳过去，它的作用是把本地开发需要的产物先准备好——跳过之后的表现文档没有说明。

## 两个目录：`src` 是代码，`playground` 是测试它的真项目

文档把这两个目录的分工写得很直白：你的模块源码在 `src` 目录里，但开发模块时你需要一个 Nuxt 应用来试它，这就是 `playground` 目录的用途——它是一个**已经配好、会带着你的模块一起跑**的 Nuxt 应用，你可以随意改它。

它跟任何 Nuxt 应用一样可以交互：

- `npm run dev` 起它的开发服务器，**你在 `src` 里改模块时它会自己重载**
- `npm run dev:build` 构建它

还有一条 tip 扩展了用法：其他任何 `nuxt` 命令都可以针对 playground 来跑，形状是 `nuxt <COMMAND> playground`，并且鼓励你在 `package.json` 里再声明一些 `dev:*` 脚本引用它们。

## 四个脚本各自做什么

讲 starter 的那页按四类任务列了脚本，逐个落地：

**测试类。**starter 自带一套基础测试：ESLint 驱动的检查，跑 `npm run lint`；Vitest 驱动的运行器，跑 `npm run test` 或 `npm run test:watch`。文档留了一句：这套默认测试策略可以按你的需要加强。

**构建类。**模块有它自己的构建器，由 [`@nuxt/module-builder`](https://github.com/nuxt/module-builder#readme) 提供。文档对它的描述是三点：**不需要你这边任何配置、支持 TypeScript、并且确保你的资源被正确打包以便分发给其他 Nuxt 应用**。构建命令是 `npm run prepack`。

这里有个反直觉的提示值得单独抄下来：

> 虽然在某些情况下构建模块有用，但大多数时候你并不需要自己动手构建：开发时 `playground` 已经替你处理了，发布时 release 脚本也已经覆盖了。

**发布类。**发布前有一个前提被放在了重要提示里：先确认你有 npmjs.com 账户，并且本地已经用 `npm login` 登录过。

## `npm run release` 一次跑完的六步

你可以靠手动升版本号加 `npm publish` 来发布，但 starter 带了一个 release 脚本，目的是确保你发布出去的是**一个能工作的版本**。用法是先把改动全部提交（文档建议遵循 Conventional Commits，因为这样还能享受自动版本递增和 changelog 更新），然后 `npm run release`。

它依次做的六步，文档分成"先跑测试"和"再发布"两组：

- 跑 lint（`npm run lint`）
- 跑单元、集成与端到端测试（`npm run test`）
- 构建模块（`npm run prepack`）
- 按你的 Conventional Commits 递增版本号并生成 changelog
- 把模块发布到 npm——**为了确保发布产物里用的是更新后的版本号，模块会被再构建一次**
- 把代表新版本的 git 标签推到你的 git 远程

最后那条 tip 说这些步骤也可以改：默认 `release` 脚本可以按你的需要微调。

## 第一次动手的最小顺序

- 判断这个模块是本地还是要发布；本地的先看能不能放 `modules/`。
- 发布的就用四条命令里对应你包管理器的那条建骨架，装依赖，跑 `dev:prepare`。
- 把 `playground` 当项目跑起来（`npm run dev`），确认它带着你的模块热重载。
- 需要额外命令时按 `nuxt <COMMAND> playground` 的形状加脚本，而不是去 playground 目录里手动 cd。
- 发布前跑 `npm run release`，别自己 `npm publish`；前提是 `npm login` 已经做过。

下一篇往这个骨架里放第一个运行时能力：组件、组合式函数和插件分别从哪几个函数注册、路径为什么要走 resolver。

## 本文依据

- https://nuxt.com/docs/4.x/guide/modules/getting-started
- https://nuxt.com/docs/4.x/directory-structure/modules
