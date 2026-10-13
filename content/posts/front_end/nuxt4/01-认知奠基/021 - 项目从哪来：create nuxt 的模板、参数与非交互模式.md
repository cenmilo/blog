---
url: /posts/b7c9121203e087ad43337e065a163552/
title: 021 - 项目从哪来：create nuxt 的模板、参数与非交互模式
date: 2026-10-19T09:00:00+08:00
updated: 2026-10-19T09:00:00+08:00
author: cmdragon

summary:
  摘要：初始化新项目用的是 create nuxt（官方也说明 nuxt init 已从 @nuxt/cli 移除，请改用 npm create nuxt@latest 或你包管理器的等价写法），它基于 unjs/giget 来拉模板：会问你要从哪个模板开始、装哪些模块、用哪个包管理器，然后安装依赖并打印下一步该跑的命令。选项里最该记住的四组是 -t/--template 模板、-f/--force 覆盖已存在目录、-M/--modules 用逗号分隔且不带空格地指定模块、--packageManager 选择包管理器；另有 --install/--no-install（默认 true，脚手架完成后安装依赖）、--gitInit/--no-gitInit、--offline/--preferOffline、--nightly 指定 nuxt-nightly 的 dist-tag。非交互模式下必须把该问的都用参数给：目录、--template、--packageManager 与 --gitInit，缺什么就被报出来并列出可用模板、命令以状态码 2 退出。还有一个 npm 专属细节：npm create 会把初始化器之后形似 flag 的内容读成 npm 自己的配置并报 Unknown cli config，所以要加一个 -- 分隔，而 pnpm 等会把整条命令行原样转发。

categories:
  - 前端开发

tags:
  - nuxt4
  - 脚手架
  - cli
  - 模板
  - ci

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


010 讲说过"先跑起来"。这一讲就把第一步做实：**项目到底是怎么被创建出来的、每条命令在哪些包管理器下要怎么写、以及 CI 里怎么让它不问你问题。**

## 先认一个词的迁移史

官方在安装那一节给的是：

```bash
npm create nuxt@latest <project-name>
```

而在命令页里有一条 note 值得注意，因为它会让旧教程失效：

> `nuxt init` 已从 `@nuxt/cli` 移除。请使用 `npm create nuxt@latest`，或你的包管理器的等价写法。

看到写 `nuxt init` 的文章，是旧文档。

## 它到底做了什么

> `create-nuxt` 命令使用 unjs/giget 来初始化一个全新的 Nuxt 项目。它会询问**从哪个模板开始**、**是否安装模块**，以及**使用哪个包管理器**，然后安装依赖并打印接下来要运行的命令。

也就是说这条命令内部做了四件事：**拉模板 → 装模块 → 装依赖 → 告诉你下一步**。理解这个链条之后，"我不想被问"和"我想在 CI 里跑"就都变成了"把答案用参数给全"。

## 五种包管理器的官方写法

安装那一节给的对照（每种管理器的等价命令）：

```bash
npm create nuxt@latest <project-name>
yarn create nuxt <project-name>
pnpm create nuxt@latest <project-name>
bun create nuxt@latest <project-name>
deno -A npm:create-nuxt@latest <project-name>
```

注意两处差异不是笔误：

- `yarn` 那条**没写 `@latest`**。
- `deno` 那条要加 `-A`（权限标志），并且写的是 `npm:create-nuxt@latest`。

## 完整参数表

命令页给的完整形态：

```bash
npm create nuxt@latest [DIR] -- [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [-t, --template=<template-name>] [-f, --force] [--offline] [--preferOffline] [--install] [--gitInit] [--shell] [--packageManager=<npm|pnpm|yarn|bun|deno|aube|nub|upm>] [-M, --modules=<module-names>] [--nightly=<dist-tag>]
```

| 选项 | 默认 | 官方说明 |
| --- | --- | --- |
| `--cwd=<directory>` | `.` | 指定在哪个目录里创建项目 |
| `--logLevel=<silent\|info\|verbose>` | | 指定构建时日志级别 |
| `-t, --template=<template-name>` | | 模板名 |
| `-f, --force` | | **覆盖已存在的目录** |
| `--offline` | | 强制离线模式 |
| `--preferOffline` | | 偏好离线模式 |
| `--install` | `true` | **在脚手架完成之后安装依赖** |
| `--no-install` | | 跳过安装依赖 |
| `--gitInit` | | 初始化 git 仓库 |
| `--no-gitInit` | | 跳过 git 仓库初始化 |
| `--shell` | | 在项目目录里安装完成后启动 shell |
| `--packageManager=<...>` | | 包管理器选择（可选值含 npm、pnpm、yarn、bun、deno、aube、nub、upm） |
| `-M, --modules=<module-names>` | | 要安装的 Nuxt 模块（**逗号分隔且不带空格**） |
| `--no-modules` | | 跳过模块安装提示 |
| `--nightly=<dist-tag>` | | 使用 Nuxt nightly 发布频道（一个 `nuxt-nightly` 的 dist tag，默认为 `latest`） |

三行我建议标红记住：

1. **`--install` 默认是 `true`**。想"只要文件不要装"要显式 `--no-install`。
2. **`-M` 的值不能带空格**——`-M image,eslint` 可以，`-M image, eslint` 不行。
3. **`--nightly` 与 016 讲那条警告连着**：官方说当前 `latest` nightly 正在跟踪 v4 分支、"此刻特别可能出现破坏性变更"。用 `--nightly` 建项目就是主动进入这个状态。

## 非交互模式：CI 里必须给全

> 在没有终端可以提示的情况下，它本来要问的那些答案必须以参数形式给出：目录、`--template`、`--packageManager` 与 `--gitInit`。**任何缺失的项都会被报告出来，并附上可用的模板列表，命令以状态码 `2` 退出。**

```bash
npm create nuxt@latest my-app -- --template minimal --packageManager pnpm --no-gitInit
```

**这条是 CI/脚本里最容易写错的一行**，因为它同时有 `--`（下面解释）和四个必答项。少给一个，你会看到一个非零退出码而不是"它自己挑了默认值"。

还有个模板注册表的环境变量，做企业内私有模板时会用到：

> `NUXI_INIT_REGISTRY`：设置为自定义模板注册表。默认注册表来自 nuxt/starter/templates。

## `--` 为什么必须有：npm 会吞你的参数

官方这段解释了一个非常具体的现象：

> 使用 `npm create` 时需要那个 `--`，因为 npm 会把初始化器之后任何形似 flag 的内容读成 npm 自己的配置，并报 `Unknown cli config`，而不是把它们转发过去。其它包管理器会把整条命令行原样转发，所以直接把标志传给他们：`pnpm create nuxt@latest my-app --template minimal`。

也就是说：

- `npm create nuxt@latest my-app --template minimal` ← **少了 `--`，npm 会自己吃掉 `--template` 并警告。**
- `pnpm create nuxt@latest my-app --template minimal` ← 直接传即可。

这是那种"报错信息完全不指向真因"的经典坑，值得单独记。

## 建完之后：官方给的下一步是什么

安装那一节在建项目之后的两步很直白：

```bash
code <project-name>
```

> 或者从你的终端里切换到你的新项目目录：

```bash
cd <project-name>
```

然后启动开发服务器（同样是每种管理器各一条）：

```bash
npm run dev -- -o
yarn dev --open
pnpm dev -o
bun run dev -o
deno run dev -o
```

> 做得好！一个浏览器窗口应该会为 <http://localhost:3000> 自动打开。

两处细节：

- `npm run dev -- -o` 里那个 `--` 和上面是**同一类语法**：把 flag 交给脚本而不是 npm。
- `bun` 那条官方还留了一句："要使用 Bun 运行时进行开发：`bun --bun run dev -o`"。

如果你用 Docker 或想跨机器保持一致的开发环境，官方给的建议是：

> 如果你在使用 Docker，或想要一个跨机器一致的开发环境，考虑使用一个 dev container。

（这属于 032 讲的范围，这里只记"存在这条建议"。）

## 自查

- [ ] 你贴给同事的创建命令，是按他的包管理器写的吗（尤其 deno 的 `-A` 与 yarn 没 `@latest`）？
- [ ] `npm create ... --template x` 前面有没有那个 `--`？
- [ ] CI 里用非交互模式时，目录/`--template`/`--packageManager`/`--gitInit` 四项都给全了吗？
- [ ] 需要"只生成不装依赖"时，用的是 `--no-install` 吗（默认会装）？
- [ ] `-M` 传的模块列表里有没有混进空格？
- [ ] 用过 `--nightly` 的话，是否清楚它此刻跟踪的是哪条分支？

## 下一篇

022 - Vapor Mode：让个别组件跳过虚拟 DOM 渲染

## 本文依据

- `api/commands/init`（"init 命令初始化一个全新的 Nuxt 项目"、基于 unjs/giget、会问模板/模块/包管理器然后安装依赖并打印下一步、`nuxt init` 已从 @nuxt/cli 移除的 note、完整命令行与参数表全部选项及默认值（含 --install 默认 true、-f force 覆盖已存在目录、-M 逗号分隔不带空格、--nightly 使用 nuxt-nightly dist tag 默认 latest、--packageManager 可选值列表、--offline/--preferOffline/--shell/--gitInit/--cwd/--logLevel、--no-install/--no-gitInit/--no-modules）、非交互模式必须给目录/--template/--packageManager/--gitInit 且缺失被报告并列出可用模板后以状态码 2 退出、示例命令、npm create 需要 -- 否则被读成 npm 自身配置并报 Unknown cli config 而其它包管理器原样转发、NUXI_INIT_REGISTRY 自定义模板注册表与默认来自 nuxt/starter/templates）
- `getting-started/installation`（五种包管理器的 create 命令写法、code/cd 两行、五种启动开发服务器命令与 npm 的 -- -o、bun --bun 提示、"浏览器应为 localhost:3000 自动打开"、Docker/dev container 建议、在线沙箱提示）
- `guide/going-further/nightly-release-channel`（latest nightly 当前跟踪 v4 分支因此特别可能出现破坏性变更）
