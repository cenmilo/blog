---
url: /posts/6dd5fefb26a72aff5c47ea89485ee907/
title: 031 - 编辑器与语言服务：VS Code、JetBrains、Neovim 各要配什么
date: 2026-10-22T10:00:00+08:00
updated: 2026-10-22T10:00:00+08:00
author: cmdragon

summary:
  摘要：官方对编辑器的态度是"没有 IDE 要求"，但给了三条推荐：VS Code 加官方 Vue 扩展（marketplace 上的 Vue.volar，以前就叫 Volar）、WebStorm 与其它 JetBrains IDE（开箱提供很好的 Nuxt 支持）、其它编辑器如 Neovim 需要自己按 Vue Language Tools 指南配置 Vue Language Server，且官方专门警告：配置 Vue TypeScript 插件时 location 要指向 @vue/language-server 包的目录而不是它的二进制。另一半容易被忽略的事实是：编辑器里的类型不是插件自己变出来的，而来自 .nuxt 目录里自动生成的类型文件，它们只在跑 dev、build 或 nuxt prepare 时生成；官方还明确警告不要直接改 tsconfig.json，因为这会覆盖重要设置，应当通过 nuxt.config.ts 扩展。

categories:
  - 前端开发

tags:
  - nuxt4
  - typescript
  - 编辑器
  - 语言服务
  - tsconfig

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

023 里我把编辑器列成了三项前置之一，这一篇展开。但我想先讲一件比装哪个插件更重要的事，编辑器里的类型是从哪来的。很多人插件装对了、类型还是不亮，原因根本不在插件。

先说结论，官方对编辑器的原话是，文本编辑器，没有 IDE 要求，但我们推荐 Visual Studio Code 加官方 Vue 扩展，之前称为 Volar，或者 WebStorm，它连同其它 JetBrains IDE 都开箱提供了很好的 Nuxt 支持。如果你用别的编辑器，例如 Neovim，可以按照 Vue Language Tools 设置指南配置 Vue Language Server 支持。

拆成三类。VS Code，装官方 Vue 扩展，`Vue.volar`。WebStorm 或 JetBrains 系，什么都不用装，开箱支持 Nuxt。Neovim 或其它，自己配 Vue Language Server，照 Vue Language Tools 指南。

没有 IDE 要求这句是真的没有。项目在终端里就能跑，024 讲过，编辑器只影响你写的时候有没有正确提示。

一个搜索上的小坑，官方特意写了 previously known as Volar。你在 VS Code 市场里搜 `Volar` 和搜 Vue 都会命中同一个扩展，包名标识是 `Vue.volar`。别以为那是两个东西、装了两个。

类型不是插件变出来的，是 `.nuxt` 里生成的。官方在 TypeScript 这一页写得很明确，Nuxt 项目依赖自动生成的类型才能正常工作。这些类型存放在 `.nuxt` 目录里，在你运行开发服务器或构建应用时生成。你也可以通过运行 `nuxt prepare` 手动生成这些文件。

这一句解释了那个经典现象，新建项目、编辑器打开、`useFetch` 显示找不到名称。因为 `.nuxt` 里那批类型文件还不存在，它是 `dev`、`build`、`prepare` 的产物，不是编辑器插件的产物。

024 里我们看过脚手架的 `package.json`，它的 `postinstall` 是 `"nuxt prepare"`。所以正常流程下，`create nuxt` 会自动装依赖，这些类型在你 clone 或 install 完就已经生成过一次了。如果提示还是不亮，手动补一次即可。

```bash [Terminal]
npx nuxt prepare
```

`nuxt prepare` 本身在 035 展开，这里只当手动生成一次类型的命令用。

生成的不只是一份 tsconfig。官方说明了它为什么现在生成多份，Nuxt 使用 TypeScript project references 来提升类型检查性能并提供更好的 IDE 支持，这个特性让 TypeScript 可以把你的代码库拆成更小、更好管理的部分。当你运行 `nuxt dev`、`nuxt build` 或 `nuxt prepare` 时，Nuxt 会为应用的不同部分生成多个 `tsconfig.json` 文件。

四份文件与各自负责的范围，官方列表是这样的。`.nuxt/tsconfig.app.json` 管 `app/` 目录里的应用代码。`.nuxt/tsconfig.node.json` 管你的 `nuxt.config.ts` 以及其它上下文之外的文件。`.nuxt/tsconfig.server.json` 管服务端代码，适用时。`.nuxt/tsconfig.shared.json` 管在 app 与 server 之间共享的代码，比如类型、以及与环境无关的工具函数。

官方对这四份的定位是，每一份都被配置为引用其合适的依赖，为它们各自的上下文提供最优的类型检查。

这就是同一个 `useRoute` 在 `app/pages` 里有提示、在 `server/api` 里没提示的技术根源之一，它们落在不同的 tsconfig 上下文里。自动导入的边界在 017 讲过，065、066 会接着讲 `server/` 与 `shared/`。

另外官方还提到兼容性那一份，出于向后兼容，Nuxt 仍然会生成 `.nuxt/tsconfig.json`，不过我们建议使用 TypeScript project references。

这些生成的 tsconfig 里包含什么，官方也列了，生成的 `tsconfig.json` 文件里包含推荐的基础 TypeScript 配置、对自动导入与 API 路由类型的引用，以及像 `#imports`、`~/file`、`#build/file` 这样的路径别名。

一条 warning，别直接改 tsconfig.json。这是这一篇最要记住的操作禁忌，官方用的是 warning 级别，Nuxt 依赖这份配置，而且 Nuxt 模块也可能扩展它，因此不建议直接修改你的 `tsconfig.json` 文件，因为这样做可能覆盖掉重要的设置。请改为通过 `nuxt.config.ts` 来扩展它。

为什么这条对编辑器用户特别容易撞上，因为 tsconfig 就躺在项目根目录，看得见、能改，改完编辑器立刻有反应，然后下一次 `nuxt dev` 或 `prepare` 重新生成，你的修改就没了，或者更糟，你把框架依赖的设置覆盖掉了。要加 TypeScript 设置，走 `nuxt.config` 那条路，扩展写法在 `.nuxt` 与 `tsconfig` 那两篇展开。

Neovim 那条具体的坑。官方在 optimal setup 的补充说明里给了 Neovim 唯一一条，Neovim，配置 Vue TypeScript 插件时，确保 `location` 指向 `@vue/language-server` 包的目录，而不是它的二进制文件。

这条的坏消息是，指错也可能不报错，只是编辑器里没有类型提示，你会以为是 Nuxt 的问题。官方还补了一句设置指南里有一份可用的配置，别自己拼路径，照抄那份。

最后一类容易混淆的，编辑器亮红不等于构建会失败。官方的类型检查口径是这样的，跑 `nuxt typecheck` 才做检查。

```bash [Terminal]
npx nuxt typecheck
```

或者在构建与开发期间开启。

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  typescript: {
    typeCheck: true,
  },
})
```

这意味着，编辑器里没划红线不代表类型对，编辑器里划了红线也不代表 `nuxt build` 会失败。007 讲类型是谁给的时说过默认不检查，这一节就是它的操作含义。`vue-tsc` 与 `typescript` 怎么装，npm、pnpm、yarn、bun、deno 五种写法都有，在 035、036 讲。

## 自查

- [ ] 你用的编辑器属于官方三类里的哪一类？需要额外装什么？
- [ ] 编辑器里类型提示不亮，第一件事是检查插件还是检查 `.nuxt` 里有没有生成物？
- [ ] 四份生成的 tsconfig 分别覆盖哪些代码？你的 `server/api/` 文件落在哪一份？
- [ ] 为什么官方不建议直接改 `tsconfig.json`？正确入口是哪个文件？
- [ ] Neovim 的 `location` 应该指向包目录还是二进制？指错的症状是什么？

## 下一篇

032 处理另一类环境不一致：官方给的 Dev Container 配置只要两个文件，把 Node 版本、包管理器、端口转发和 `node_modules` 都钉死在容器里。
