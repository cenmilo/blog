---
url: /posts/0aefeb0d009e6f44fbc1ad434185dfa0/
title: 048 - nuxi 与 nuxt 是什么关系：CLI 的包名、回退与全局选项
date: 2026-10-28T09:00:00+08:00
updated: 2026-10-28T09:00:00+08:00
author: cmdragon

summary:
  摘要：官方把关系一次讲清：@nuxt/cli 就是 nuxt 这个命令，它是 nuxt 的依赖，所以任何 Nuxt 项目里本来就有它；同一个二进制还额外装成 nuxi、nuxi-ng 与 nuxt-cli 三个名字以兼容旧配置；脚手架新项目是另一个包 create-nuxt，好让它能在"还没有项目可装"的情况下运行；而 nuxi 是第三个包——一个没有依赖的独立 CLI，用于在项目之外运行命令或全局安装。回退规则是这一节最实用的部分：npx nuxi 在项目里会把命令交给项目所装的 @nuxt/cli，好让实际运行的代码与它旁边那个 Nuxt 版本匹配；只有在没有项目 CLI、项目那份早于 v3.26、或项目那份不认识这条命令时，它才回退到自己捆绑的实现。CLI 要求 Node v22.21+、v24.11+ 或 v26+，并且不支持 Nuxt 2 与 @nuxt/bridge。此外 --cwd 可以写在命令名之前或之后且会覆盖 ROOTDIR 位置参数；拼错的命令与开关会被拿去和已知项匹配并在交互终端里给你可运行的纠正；CLI 不提供的命令会被当作项目里的 nuxt-<command> 二进制去查找（nuxt build-module 正是这么来的）；每天后台检查一次更新并把提示打在命令结束之后。

categories:
  - 前端开发

tags:
  - nuxt4
  - cli
  - nuxi
  - 版本
  - 命令行

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

从 024 到现在，你敲过的每条命令前面都是 `npx nuxt` 或 `npm run`。但老资料里常见 `npx nuxi dev`。`nuxi` 是什么、它和 `nuxt` 是同一个东西吗。这一节官方把四个名字的关系一次讲完，还有一条能解释一半我这边命令行为不一样的回退规则。

四个名字分别是什么。官方的第一句就把主从关系定了，`@nuxt/cli` 就是 `nuxt` 这个命令，它运行开发服务器、把你的应用构建为生产版、脚手架文件、管理模块，并回答有关你项目的问题。它是 `nuxt` 的依赖，所以一个 Nuxt 项目里已经有它了。

这句项目里已经有它了很重要，你不需要单独安装 CLI，对照 024 的 `dependencies: { "nuxt": ... }`。

然后是那三个别名，以及两个分家的包。这个二进制被安装为 `nuxt`，同时也安装为 `nuxi`、`nuxi-ng` 与 `nuxt-cli`，以兼容更早的配置。脚手架新项目是一个独立的包 `create-nuxt`，这样它就能在还没有项目可安装的情况下运行。`nuxi` 是第三个包，一个没有依赖的独立 CLI，用于在项目之外运行命令，或者全局安装。

整理一下，四个名字的归属就清楚了。`@nuxt/cli` 是 CLI 本体，提供 `nuxt` 命令，是 `nuxt` 的依赖。`nuxi`、`nuxi-ng`、`nuxt-cli` 是同一个二进制的兼容别名，来自 `@nuxt/cli`。`nuxi`，作为独立包，是第三个包，无依赖的独立 CLI，用于项目之外或全局安装。`create-nuxt` 是另一个独立包，只做脚手架，所以它能在没有项目时运行，021 讲过 `npm create nuxt@latest`。

注意 `nuxi` 这个名字身兼两职，既是 `@nuxt/cli` 的别名二进制，又是一个独立包。官方这一节是分开两句写的，读到 `nuxi` 时要看上下文说的是哪个。

运行环境要求，比 Installation 页更细。官方在这里给的版本地板线，和 023 那条前置要求粒度不同。CLI 运行于 Node v22.21+、v24.11+ 或 v26+，并在这些版本上原生导入 TypeScript 配置。Nuxt 2 与 `@nuxt/bridge` 不受支持。

对照 023 抄过的那句 Node.js 22.x 或更新，你会发现这一节精确到了补丁号，`22.21`、`24.11`。这不是矛盾，那一节讲的是框架的安装前置，这一节讲的是 CLI 自身的运行条件。两处都原样列在上面，取更严的那条更保险。

后半句也值得记，`nuxi` 与 `@nuxt/cli` 这一套明确不支持 Nuxt 2 与 bridge，Nuxt 2 的迁移路线在 617 到 629 讲，那里用的是另一套机制。

回退规则，这是本篇最实用的一节。官方先说在项目里怎么运行，在项目内部，你的包管理器运行的就是项目所依赖的那个版本。

```bash [Terminal]
pnpm nuxt dev
npm exec nuxt dev
yarn nuxt dev
bun nuxt dev
```

对照 024，`npm run dev` 走的是 `scripts.dev`，这里这几条是 `npm exec` 与 `pnpm` 那种直接执行二进制的写法。两种都能跑，来源不同。

然后是 `nuxi` 独立包的行为，一段话里塞了三条判断，我逐句抄。在项目之外，或者全局安装时，`npx nuxi` 运行同样的命令。当项目里装有 `@nuxt/cli` 时，它会把每一条命令交给它，因此实际运行的代码与它旁边那个 Nuxt 版本相匹配，而不是你机器上装的任意一份。当没有项目 CLI、或项目那份早于 v3.26、或项目那份不认识这条命令时，它回退到自己捆绑的命令，所以一个钉在旧版本的项目依然能用到更新的命令。

把这段翻成四条可核对的规则。

项目里有 `@nuxt/cli`，且版本不早于 v3.26，且它认识这条命令，由项目里那份执行，这是常态，也是最该关心的情况。

项目里没有 `@nuxt/cli`，`nuxi` 自己捆绑的实现执行。

项目里那份早于 v3.26，同上，回退。

项目里那份不认识这条命令，新命令，同上，回退。

为什么这值得单独学一节，因为它解释了一类非常难查的现象，同一条命令在 A 项目和 B 项目行为不同。按官方这段，原因可以是其中一个用的是项目里那份 CLI，另一个回退到了全局 `nuxi` 的实现。排查顺序应该是先跑 042 的 `nuxt info` 看版本，再确认你在哪一层执行命令。这条排查顺序是我按上面那张表推的用法。

全局选项，`--cwd` 能写在命令名前面。官方原话，`--cwd` 可以写在命令名之前，也可以写在之后，这在一个 monorepo 里很有用。

```bash [Terminal]
npx nuxt --cwd apps/web dev
```

任何带 `ROOTDIR` 位置参数的命令，都会把显式的 `--cwd` 视为对它的覆盖。任何命令上的 `--help` 都会打印它的参数与选项，`--version` 打印 CLI 版本。

这一段回答了两个 033 到 047 里反复出现的问题。

`ROOTDIR` 与 `--cwd` 同时给出时谁赢，官方的答案很硬，`--cwd` 覆盖 `ROOTDIR`。

每个命令的选项怎么当场查，`--help`。这一条比翻文档快，034 到 046 那些表都来自 `--help` 的那份数据源。

拼错的命令与开关，它会给你纠正。官方原话，无法识别的命令会与存在的命令做匹配，无法识别的开关会与该命令声明的开关做匹配。在交互式终端里，它会提供纠正后的写法让你运行，否则它作为警告报告，这样一个拼错的开关就不再看起来像一个没起作用的开关。

最后那半句是重点，过去的行为是参数没生效但也不报错，官方在这里明确改了，现在拼错会被指出来。所以如果你在旧版本上加了个参数但好像没用，先怀疑拼写。

CLI 没有的命令，去项目里找 `nuxt-<command>`。官方原话，CLI 不提供的命令，会被当作你项目里的 `nuxt-<command>` 二进制来查找，所以一个依赖可以给自己加一条命令。`nuxt build-module` 就是这么工作的，它运行的是 `@nuxt/module-builder` 安装的那个 `nuxt-build-module` 二进制。

这条机制解释了为什么文档里有些命令看起来是 CLI 的、其实不是。049 讲 `build-module` 时就能直接接上这一句。

补全脚本，`nuxt complete <shell>`。官方原话，`nuxt complete <shell>` 打印一段补全脚本，由 `@bomb.sh/tab` 提供。它补全命令、开关以及一些开关的取值，`nuxt dev` 的端口与主机、`nuxt build --preset` 的 Nitro presets，以及日志级别。

官方给了三种 shell 的写法。

```bash [zsh]
npx nuxt complete zsh > "${fpath[1]}/_nuxt"
```

```bash [bash]
npx nuxt complete bash > /etc/bash_completion.d/nuxt
```

```bash [fish]
npx nuxt complete fish > ~/.config/fish/completions/nuxt.fish
```

外加三条补充，也支持 `powershell`，`create-nuxt complete <shell>` 有自己的脚本，它补全的是 starter 模板名，021 讲过模板选择，以及通过包管理器也能用补全，`pnpm nuxt` 加 Tab，前提是装了 tab 的包管理器补全。

注意它连 `--preset` 这个旧名字的取值都补，回到 037，`--preset` 仍作为 `--target` 的别名被接受。这是那条别名规则的一个侧面印证。

更新提示的完整规则。这一段很多人想关掉，官方的规则写得比能关细得多。每天一次，CLI 在后台检查是否存在更新的 Nuxt 版本，如果有，就在命令结束之后打印一句简短的提示让你运行 `nuxt upgrade`。`latest` 这个 dist-tag 从你在 `.npmrc` 里配置的 registry 读取，并缓存进你的用户级 `.nuxtrc`。提示不会为预发布版本显示，同一 minor 内的补丁版本也不会，除非你落后至少五个补丁。检查在 CI、StackBlitz、以及输出不是终端时跳过。设置 `NUXT_IGNORE_UPDATE_CHECK=1` 或 `NO_UPDATE_NOTIFIER=1` 跳过单次运行，或者在你的用户级 `.nuxtrc` 里加 `updateCheck.enabled=false` 永久退出。

值得逐条注意的点有四条。

提示出现在命令结束之后，它不会打断你正在跑的东西。

五个补丁这个阈值是官方写的具体数字，不是很久没更新。

034 里那条环境变量表已经出现过 `NUXT_IGNORE_UPDATE_CHECK`，这里给的是完整规则，还多了第二个变量 `NO_UPDATE_NOTIFIER=1` 和 `.nuxtrc` 那条永久选项。

`.npmrc` 加用户级 `.nuxtrc` 是 047 那条 `devtools enable` 写在用户级 `.nuxtrc` 的同一个文件，你的私有 registry 环境会同时影响更新检查这一项。

网络与代理，失败原因会说人话。官方原话，会走网络的命令，安装模块、搜索模块、下载模板、检查更新，会报告请求为什么失败，而不是原始错误，并建议一条可能成功的重试。在一个会重新签署 TLS 流量的代理后面，这意味着要设 `NODE_EXTRA_CA_CERTS`，指向你组织的根证书。Node 只有在设置了 `NODE_USE_ENV_PROXY` 时才读取代理环境变量，而 CLI 会把这一设置传给它派生的进程，所以由 `nuxt add` 发起的包管理器安装，看到的是和你一样的代理配置。

这一段有两条硬信息。

`NODE_EXTRA_CA_CERTS` 又一次出现。034 讲的是本地 HTTPS 证书，这里讲的是公司代理重签证书，同一个变量名解决两类 Node 不信任问题。

`NODE_USE_ENV_PROXY` 是必须显式设置的，官方写着 Node 只有在设置了它时才读代理环境变量。而且 CLI 会把它传给派生进程，所以 045 的 `nuxt add` 触发的安装不会掉配置。

预发布与 PR 构建。官方原话，`main` 上的每一个提交都会发布到 nightly 频道。

```bash [Terminal]
npm install -D @nuxt/cli-nightly
npx @nuxt/cli-nightly dev
```

`create-nuxt-nightly` 用它来脚手架，而 `npm create nuxt@latest -- --nightly` 会针对 Nuxt 的 nightly 频道脚手架一个项目。单个 pull request 的构建由 pkg.pr.new 发布，并从那个 pull request 本身链接过去，这是在修复发布之前确认它是否有效最快的办法。

这一节和 016、044 是同一件事的三个入口，016 讲 Nuxt 包的 nightly 怎么切、044 讲 `nuxt upgrade --channel=nightly`，这里讲的是 CLI 自己的 nightly，`@nuxt/cli-nightly`。016 那篇里 CLI 也可以用 nightly 的那条，官方在这里给的就是包名与安装命令。

而 `pkg.pr.new` 那句是很多人不知道的门路，想知道某个尚未发布的修复能不能解决你的问题，去那个 PR 下面取它挂的包。

## 自查

- [ ] `nuxt`、`nuxi`、`nuxi-ng`、`nuxt-cli` 四个名字来自哪个包？`create-nuxt` 为什么必须分开？
- [ ] `npx nuxi dev` 在项目里通常由谁实际执行？哪三种情况会回退？
- [ ] `ROOTDIR` 与 `--cwd` 同时给了，谁生效？
- [ ] CLI 不认识的命令名，它会去哪里找？`nuxt build-module` 属于这一类吗？
- [ ] 落后几个补丁才会弹更新提示？想永久关掉，官方给的第三种办法是什么？
- [ ] 你在企业代理后面，`nuxt add` 装包失败，官方点了哪两个环境变量？

## 下一篇

049 收尾这一段：`nuxt build-module`——048 说它是「项目里的 `nuxt-build-module` 二进制」，这一篇讲它的四个开关各自做什么。
