---
url: /posts/5c6aa340242590d30986636e2000b4db/
title: 049 - nuxt build-module：模块的构建、打包与类型声明
date: 2026-10-28T10:00:00+08:00
updated: 2026-10-28T10:00:00+08:00
author: cmdragon

summary:
  摘要：nuxt build-module 与前四十多篇那些命令有一个本质区别——它不是 @nuxt/cli 自带的命令。官方的 CLI 总览页写明：CLI 不提供的命令会按项目里的 nuxt-<command> 二进制查找，而 nuxt build-module 就是这么工作的，它运行的是 @nuxt/module-builder 安装的那个 nuxt-build-module 二进制。命令页自己的定义是"运行 @nuxt/module-builder，在你的 rootDir 里生成一个 dist 目录，其中装着你的 nuxt-module 的完整构建"，用途标注为"发布你的 Nuxt 模块之前构建它"。它只有四个功能开关，默认值全是 false：--build 表示为分发构建模块、--stub 表示为开发而把 dist 做成存根而不是真的构建、--sourcemap 表示生成 sourcemap、--prepare 表示为本地开发准备模块；另外官方明确 --cwd 比 ROOTDIR 优先。而模块起步那一节给的日常动作其实是包里的脚本：npm run dev 启动 playground 的开发服务器、npm run dev:build 构建它、npm run dev:prepare 为本地开发准备文件。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块开发
  - module-builder
  - 构建
  - 命令行

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

CLI 这一段，034 到 048，讲的都是构建你的应用。这一篇是它唯一的例外，`nuxt build-module` 构建的是模块本身，而且按官方说法，它其实不是 CLI 提供的命令。

它不是 CLI 自带的命令。048 抄过 CLI 总览页那条机制，这里正好用上，CLI 不提供的命令，会被当作你项目里的 `nuxt-<command>` 二进制来查找，所以一个依赖可以给自己加一条命令。`nuxt build-module` 就是这么工作的，它运行的是 `@nuxt/module-builder` 安装的那个 `nuxt-build-module` 二进制。

这条机制有三个实际后果。

它要求项目里装了 `@nuxt/module-builder`。你不可能在一个普通应用项目里顺手跑一次 `nuxt build-module`。

`nuxt build --target` 那套 preset 逻辑与它无关，037 讲过，因为跑的是别人的二进制。

想知道它的行为细节，权威来源是 module-builder 那份仓库。官方这一节也正是这么处理的，页末只留了一条 read more about `@nuxt/module-builder` 的链接，把源码位置指向 module-builder 仓库而不是 `nuxt/cli`。

命令页的定义，它生成什么、在哪。官方那句定义有两个具体信息，`build-module` 命令运行 `@nuxt/module-builder`，在你的 `rootDir` 里生成一个 `dist` 目录，其中装着你的 nuxt-module 的完整构建。

标题里的用途写得更直白，在你发布 Nuxt 模块之前构建它。

在 rootDir 里生成 dist 这一句，顺便解掉了 040 的一个悬案。那一篇列 cleanup 的五处清单时，第三条 `dist` 只能标注官方在这里没说明它是谁的产物。现在两份文档可以对上了，至少有一个官方记载的 `dist` 来源，就是模块构建产物，build-module 页的这句。

也就是说，在模块项目里跑 `nuxt cleanup`，会把你的模块构建产物一起删掉。这个因果是把两份清单对上才看得出来的，cleanup 页本身仍然没解释 `dist`。

四个功能开关，默认全是 `false`。

```bash [Terminal]
npx nuxt build-module [ROOTDIR] [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--build] [--stub] [--sourcemap] [--prepare]
```

`--build` 默认 `false`，为分发而构建模块。`--stub` 默认 `false`，为开发把 dist 做成存根，而不是真的构建。`--sourcemap` 默认 `false`，生成 sourcemap。`--prepare` 默认 `false`，为本地开发准备模块。

四个默认值全是 `false` 这一点，加上 `--build` 与 `--stub` 的措辞对照，能读出官方给的三种意图。

发布前出一份真产物，用 `--build`，依据是 for distribution。

本地开发时别真构建、用存根，用 `--stub`，依据是 Stub dist instead of actually building it for development。

第一次把本地开发环境准备好，用 `--prepare`，依据是 Prepare module for local development。

想在产物里带上 sourcemap，用 `--sourcemap`，依据是 Generate sourcemaps。

注意这一节没有 `--prerender`、也没有 `--target`，对照 037 的 build，因为模块不是一个可运行的站点，那套 preset 与预渲染概念在这儿不适用。这是选项表差集比对出来的判断。

同样要记一句，这一节没有本命令把 NODE_ENV 设为什么那条 note，034 到 043 里 dev、build、preview、prepare、typecheck、analyze 都有各自的说明，而这一条命令官方没写。

`--cwd` 在这里被明确写在前面。这一节的参数表有一处和其它命令不同的措辞。`ROOTDIR="."` 指定工作目录，默认 `.`。`--cwd=<directory>` 指定工作目录，它会优先于 ROOTDIR。

对照 048 那条全局规则，任何带 `ROOTDIR` 位置参数的命令，都会把显式的 `--cwd` 视为对它的覆盖，这里是同一件事在单个命令页里的明确重申。

另外这两项的措辞是工作目录而不是其它命令里的项目根目录，合理，因为被构建的对象是一个模块，不是一个 Nuxt 应用。

模块项目里日常用的其实是脚本。模块起步那一节给的官方起步流程，跟 `build-module` 这条命令的关系值得单独说清。

官方的模块起步命令，npm 写法是这样。

```bash [npm]
npm create nuxt -- -t module my-module
```

官方原话，这会创建一个 `my-module` 项目，里面带着开发与发布你的模块所需的全部样板。

然后是它给的下一步清单，在你的 IDE 里打开 `my-module`，用你喜欢的包管理器安装依赖，用 `npm run dev:prepare` 为本地开发准备文件，跟着这份文档了解更多关于 Nuxt 模块的知识。

以及 playground 那一侧的两条，用 `npm run dev` 启动它的开发服务器，当你在 `src` 目录里改动模块时它应当自己重载，用 `npm run dev:build` 构建它。

官方还补了一条很实用的 note，其它所有 `nuxt` 命令都可以针对 `playground` 目录使用，例如 `nuxt <COMMAND> playground`。你可以随意在 `package.json` 里声明更多引用它们的 `dev:*` 脚本以便使用。

这一整段把本篇和前几篇接上了，`nuxt <COMMAND> playground` 用的正是 048 那条 ROOTDIR 位置参数能力，把 playground 目录当根传给命令。

回收 032 那条对不上。032 讲 dev container 时我留了一个疑问，官方那份 `postStartCommand` 写的是 `pnpm install && pnpm dev:prepare`，而 024 那份最小 `package.json` 里没有 `dev:prepare`。

现在能给出更准的说法了，`dev:prepare` 是模块起步流程里出现的脚本名，上面那份下一步清单第 3 条，应用项目，024 那份最小 `package.json`，里没有它。所以有两个结论。

照抄那份 dev container 配置到应用项目时，`dev:prepare` 这一步会失败，要改成你项目里真实存在的脚本，应用项目里对应的是 `postinstall` 用的 `nuxt prepare`，035 讲过。

那份配置演示的环境更像 pnpm 加带 `dev:*` 脚本的项目，官方在模块页也说了随意声明更多 `dev:*` 脚本。

官方两处文档没有互相引用，这是把两份文档对起来才得出的结论，不是哪里写错了。

## 自查

- [ ] `nuxt build-module` 是靠哪个包安装的二进制运行的？它属于 CLI 自带命令吗？
- [ ] 它在哪个目录里生成什么？这件事对 `nuxt cleanup` 意味着什么？
- [ ] `--build`、`--stub`、`--prepare` 分别对应「发布/开发存根/本地准备」哪一种？
- [ ] 这一节里 `--cwd` 与 `ROOTDIR` 的优先级，官方是怎么写的？
- [ ] 在模块项目里，官方让你为本地开发准备文件用的命令是什么？它和 `nuxt prepare` 是同一个东西吗？

## 下一篇

从 050 起进入目录结构这一段：先看整棵树——官方在 directory-structure 那一节给每个目录都留了一句话定义，而「谁规定这个名字、谁规定这个位置」就是后面几十篇的总纲。
