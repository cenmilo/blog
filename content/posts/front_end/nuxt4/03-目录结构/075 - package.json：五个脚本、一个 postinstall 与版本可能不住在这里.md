---
url: /posts/5d0214150e507843061b351c4f07c313/
title: 075 - package.json：五个脚本、一个 postinstall，以及版本可能不住在这里
date: 2026-11-06T09:00:00+08:00
updated: 2026-11-06T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：官方给 package.json 的定位是"包含你应用的所有依赖与脚本"，并给出一份最小形态：name 为 nuxt-app、private 为 true、type 为 module，scripts 里恰好五项——build 是 "nuxt build"、dev 是 "nuxt dev"、generate 是 "nuxt generate"、preview 是 "nuxt preview"、postinstall 是 "nuxt prepare"，dependencies 里显式列出 nuxt、vue 与 vue-router 三项。这份文件解释了两件常见困惑：第一，npm run dev -- -o 里没有 nuxt 这个词，是因为它调用的是脚本、脚本内容才是 nuxt dev；第二，为什么刚 clone 完项目编辑器就已经认识自动导入——postinstall 会在装完依赖后自动执行 nuxt prepare，而 prepare 的动作正是创建 .nuxt 并生成类型。另外版本号不一定住在这里：官方在升级那一节明确，被钉在 pnpm catalog 里的依赖（形如 "nuxt": "catalog:"）是在 pnpm-workspace.yaml 里升级的，因为版本实际住在那儿；而切版本频道时官方要求的动作是删除锁文件并重装依赖，锁文件名按包管理器分别是 package-lock.json、yarn.lock、pnpm-lock.yaml、bun.lock 或 bun.lockb。

categories:
  - 前端开发

tags:
  - nuxt4
  - package-json
  - 脚本
  - 依赖
  - pnpm

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`package.json` 每个前端都认识。但这一节官方只给了一份最小文件，而它恰好解释了三件你八成遇到过的事，命令为什么找不到 nuxt、类型为什么装完就好了、版本号为什么我改了没生效。

官方给的最小形态。官方原话，`package.json` 文件包含你应用的所有依赖与脚本。你的 Nuxt 应用的最小 `package.json` 看起来应当是这样。

```json [package.json]
{
  "name": "nuxt-app",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "nuxt build",
    "dev": "nuxt dev",
    "generate": "nuxt generate",
    "preview": "nuxt preview",
    "postinstall": "nuxt prepare"
  },
  "dependencies": {
    "nuxt": "latest",
    "vue": "latest",
    "vue-router": "latest"
  }
}
```

这一节到此结束，官方没逐字段解释。所以下面分两栏写，能从这份文件直接读出的事实，和需要引用别的节才能确认的事实。

五个脚本，命令表其实藏在这里。

`dev` 是 `nuxt dev`，开发期起带 HMR 的本地服务，034 讲全部开关。`build` 是 `nuxt build`，产出 `.output`，037 讲。`generate` 是 `nuxt generate`，预渲染成静态站，038 讲。`preview` 是 `nuxt preview`，构建后本地验一遍，029 与 039 讲。

`postinstall` 是 `nuxt prepare`，这一行不用你调用，由包管理器在 `install` 结束时自动跑。

这张表解决了 024 那个疑问的根源，`npm run dev -- -o` 里没有 `nuxt` 这个词，是因为它调的是脚本，脚本内容才是 `nuxt dev`。对照 048 抄过的另一类写法，`pnpm nuxt dev`、`npm exec nuxt dev`，那是直接执行二进制，不走脚本表。

`postinstall` 这一行是这份文件里最被低估的一行。035 抄过 `prepare` 命令的用途说明，它创建一个 `.nuxt` 目录并生成类型，这在 CI 环境里，或者作为 `package.json` 的 `postinstall` 命令时很有用。

两个事实接起来，就解释了那个常见现象，clone 项目、`npm install`、打开编辑器，`useFetch` 已经有提示了。不是编辑器聪明，是 install 结束时自动跑了一次 `nuxt prepare`，031 讲的那些类型来自 `.nuxt` 就是它生成的。

反过来说，绕过包管理器搬 `node_modules`、或者 CI 里跳过 install，这份自动准备就没了，手动补一次 `npx nuxt prepare`。

三个 dependencies 字段，为什么 `vue` 和 `vue-router` 也在。官方这份最小清单里显式列了三家，`nuxt`、`vue`、`vue-router`。

这与 014 那张核心包分工直接对上，`nuxt` 是框架、`vue` 是视图层、`vue-router` 是路由。057 与 063 又给了两条相关的行为。

`app/pages/` 不存在时，Nuxt 就不会包含 `vue-router` 依赖，057 抄的原文，注意那是进不进包，不是删不删依赖项。

048 那句 `@nuxt/cli` 是 `nuxt` 的依赖，所以一个 Nuxt 项目里已经有它了，CLI 不在这三份里，它是被 `nuxt` 带进来的。

`type: module` 那一行不是装饰。015 抄过官方在 ESM 那一节写的规则，以 `.js` 结尾的文件，除非它的 `package.json` 里有 `type: module`，否则会被当作 CJS 处理。

所以脚手架从一开始就写好这一行，是在替你决定整个项目的模块语义，`.js` 直接按 ESM 解析。015 讲过 ESM 里没有 `__dirname`、`__filename`，这一行也决定了你写别名、写模块路径时用 `import.meta.url` 这一套。

`private: true` 这一行官方在这一节没解释含义，文档到这一句就断了。它通常是为了防止这个脚手架项目被误发布到 npm，不属于 Nuxt 专用机制。

版本可能不住在这里，两个例外。

例外一，pnpm catalog。044 抄过官方在升级那一节写的一段，一个被钉在 pnpm catalog 里的依赖，`"nuxt": "catalog:"`，会在 `pnpm-workspace.yaml` 里升级，版本实际住在那里，而不是在 `package.json` 里。

所以，如果你的 `package.json` 里 `nuxt` 的值是 `catalog:`，你去这里找版本号是找不到的。032 那份 dev container 的 `COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./` 三个文件并列，也印证了 workspace 项目里这三者是配套存在的。

例外二，真正决定装到哪个版本的是锁文件。030 抄过 nightly 那一节两处重复的同一句要求，删除锁文件，括号内是 `package-lock.json`、`yarn.lock`、`pnpm-lock.yaml`、`bun.lock` 或 `bun.lockb`，并重装依赖。

这份括号里的清单同时是包管理器与锁文件名对照表。而它给出的操作含义是，只删 `node_modules`、留下锁文件，你重装出来的还是锁文件钉着的那套版本。

那改版本该动哪儿，交给 `nuxt upgrade`。044 那句官方定义，`upgrade` 命令把 Nuxt 升级到最新版本。它会更新你 `package.json` 里的版本、用你的包管理器重新安装依赖，并移除构建缓存。

这三件动作里包含重装与清构建缓存，也就是说，手改 `package.json` 的数字，只完成了官方那条命令三步中的第一步。对照 040，`nuxt cleanup` 清的是构建目录与两处缓存，它并不管依赖。

判据给完，升 Nuxt 版本用 `nuxt upgrade`，可配 `--channel`。只加一个模块用 `nuxt add`，045 讲过它同时写 `package.json` 与 `nuxt.config`。装两个类型检查用的开发依赖，按 035 那份五种包管理器命令表。pnpm catalog 项目里改版本，改 `pnpm-workspace.yaml`。

## 自查

- [ ] `npm run dev -- -o` 最终执行的是哪一行内容？`--` 为什么必需？
- [ ] 五个脚本里哪个不需要你调用？它跑的是什么、为什么重要？
- [ ] 绕过 install 搬 `node_modules` 之后，第一件事该跑什么？
- [ ] `dependencies` 里为什么显式列了 `vue` 和 `vue-router`？CLI 为什么不在这里？
- [ ] 你的 `nuxt` 依赖写着 `catalog:`，版本该去哪个文件改？
- [ ] 只删 `node_modules` 保留锁文件，重装结果会不同吗？为什么？

## 下一篇

076 打开生成出来的那四份 `tsconfig`：官方给的根文件标准形态只有 `files: []` 加四条 `references`，而能不能往它里面写东西，官方用 warning 直接回答了。
