---
url: /posts/7963c93deb844605a73243584c326c98/
title: 030 - 五种包管理器写出来的命令到底差在哪：create、dev、锁文件与 npx
date: 2026-10-22T09:00:00+08:00
updated: 2026-10-22T09:00:00+08:00
author: cmdragon

summary:
  摘要：官方 Installation 页为同一件事给了五种写法，差异不是排版而是真实存在的三类：一是调用 script 的语法不同（npm 与 bun 要写 run、yarn 与 pnpm 直接写脚本名、deno 用 deno run）；二是 npm 会把跟在后面的形似参数的东西当成自己的配置吞掉，所以官方在 npm 的写法里加了 `--`（npm run dev -- -o、npm create nuxt@latest 也要 --template），其它四家原样转发；三是 deno 的 create 写法根本不同，是 deno -A npm:create-nuxt@latest。另外，锁文件名按包管理器分别是 package-lock.json、yarn.lock、pnpm-lock.yaml、bun.lock 或 bun.lockb，官方在切 nightly 频道时明确要求"删除锁文件并重装"——这也意味着换包管理器时锁文件要一并处理。

categories:
  - 前端开发

tags:
  - nuxt4
  - 包管理器
  - 锁文件
  - npm
  - pnpm

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


024 那篇我把五种写法照抄了一遍，评论区（如果你会写评论的话）最容易冒出来的问题就是：**这些差异是文档随意写的，还是真的有区别？** 这一篇把它讲完。结论提前给：有三类真实差异，外加一个锁文件问题。

## 第一类差异：怎么调用一条 script

同一个 `npm run dev -- -o`，官方在五种管理器下给的写法是这样：

| 管理器 | 建项目 | 起开发服务 |
| --- | --- | --- |
| npm | `npm create nuxt@latest <project-name>` | `npm run dev -- -o` |
| yarn | `yarn create nuxt <project-name>` | `yarn dev --open` |
| pnpm | `pnpm create nuxt@latest <project-name>` | `pnpm dev -o` |
| bun | `bun create nuxt@latest <project-name>` | `bun run dev -o` |
| deno | `deno -A npm:create-nuxt@latest <project-name>` | `deno run dev -o` |

调用 script 这一步就能看出三种习惯：

- **npm、bun、deno 要写 `run`**：`npm run dev`、`bun run dev`、`deno run dev`。
- **yarn、pnpm 直接写脚本名**：`yarn dev`、`pnpm dev`。
- 长参数名也不一样：npm/pnpm/bun/deno 用短的一面（`-o`），yarn 那行写的是 `--open`。

别忘了 024 讲过的事实：这些 script 本身跟包管理器无关，五种写法最终执行的都是同一份 `package.json` 里的 `dev: "nuxt dev"`。**差异全在"怎么把这条 script 叫起来"这一层。**

## 第二类差异：npm 必须多写一个 `--`

这是最容易让人以为是笔误的一处。官方在 `create nuxt` 那一页专门用 note 说明了：

> `--` 对 `npm create` 是必需的，因为它会把 initializer 之后任何形似参数的东西读成 npm 自己的配置，并警告 `Unknown cli config` 而不是把它传下去。其它包管理器会把整条命令行原样转发，所以直接把参数传给它们：`pnpm create nuxt@latest my-app --template minimal`。

这段解释同时管住了两件事：

- 建项目时：`npm create nuxt@latest my-app -- --template minimal --packageManager pnpm --no-gitInit`（021 里那条非交互命令，npm 写法必须带 `--`）。
- 起服务时：`npm run dev -- -o`。

所以**只有 npm 那一列有 `--`**，不是文档风格不统一，是 npm 的行为确实不同。你在 npm 上少写这个 `--`，症状是框架完全没收到参数、终端反而抱怨一句 `Unknown cli config`。

## 第三类差异：deno 那条是另一个体系

deno 的 create 写法长得跟其它四个都不一样：

```bash [deno]
deno -A npm:create-nuxt@latest <project-name>
```

`npm:create-nuxt@latest` 这个前缀是 Deno 引用 npm 包的方式，而且它直接指包名（`create-nuxt`），不是 `create nuxt` 这种"子命令"形式。**`-A` 这个参数官方在 Installation 和 init 页都没解释**——它是 Deno 自己的权限开关，含义要查 Deno 文档（此处不给结论，照抄即可）。

bun 还有一条更隐蔽的官方注释：

```bash [bun]
bun run dev -o

# To use the Bun runtime during development
# bun --bun run dev -o
```

> （官方注释）要在开发期间使用 Bun 运行时

**换句话说：`bun run dev` 起的服务不一定用 Bun 的运行时跑。** 你想真的让 Bun 跑，要按注释那样加 `--bun`。这条我放在这里是因为它只靠抄命令是看不出来的——注释行是被官方写进代码块里的。

## `npx` 什么时候出现

`npx` 不用于跑 script，它用于**直接调命令**。官方在 preview 页给的就是这个形态：

```bash [Terminal]
npx nuxt preview [ROOTDIR] ...
```

同一页的 `.env` 那条 note 里，Node 20+ 的启动例子也是直接调 `node`，不经任何包管理器。对照 `package.json` 的 `scripts.preview: "nuxt preview"`，可以得出一个实用规则：

- 在**项目目录里**、`node_modules` 已装好：`npm run preview` 或 `npx nuxt preview` 都行。
- 项目**还没建**、依赖还没装：只能用 `npm create nuxt@latest` 这种 initializer 形态，或用 `npx` 显式拉包。

（后半句是我按"依赖装在 node_modules 里"这个事实给的操作建议，官方没有单独列这条规则。）

## 锁文件：换管理器或换频道时都要动它

官方在 nightly 频道那一页，两次都写了同一句要求：

> 删除锁文件（`package-lock.json`、`yarn.lock`、`pnpm-lock.yaml`、`bun.lock` 或 `bun.lockb`）并重装依赖。

这份括号里就是**包管理器与锁文件名的对照表**，我把它整理出来：

| 包管理器 | 锁文件 |
| --- | --- |
| npm | `package-lock.json` |
| yarn | `yarn.lock` |
| pnpm | `pnpm-lock.yaml` |
| bun | `bun.lock` 或 `bun.lockb` |
| deno | （官方这份列表里没有它） |

两个实用结论：

1. **一个项目应该只有一种锁文件。** 你今天 `npm install`、明天 `pnpm install`，锁文件会打架——症状是"我这装出来跟 CI 装出来版本不一样"。
2. **需要重装依赖树时（切版本频道、依赖解析诡异），官方给的动作就是"删锁文件 + 重装"。** 别只删 `node_modules` 留锁文件，那重装出来的还是锁文件里钉着的那套版本。

## 官方是怎么知道你打算用哪个管理器的

`create nuxt` 的行为说明里有这句：

> 它会问你要从哪个模板开始、是否安装模块、以及**使用哪个包管理器**，然后安装依赖并打印接下来该运行的命令。

非交互场景下就得显式传（021 讲过这四个必传项）：

```bash [Terminal]
npm create nuxt@latest my-app -- --template minimal --packageManager pnpm --no-gitInit
```

`--packageManager` 接受的取值，官方原样列的是：

> `--packageManager=<npm|pnpm|yarn|bun|deno|aube|nub|upm>`

**这份列表里除了常见五家，还带着 `aube`、`nub`、`upm` 三个我没在任何地方见过的名字。** 文档就是这么写的，我没有替它们补充含义——真要传这三个值，建议先自己验证一下它到底会生成什么。

顺带一个能佐证"锁文件名"的地方：Dev Container 的官方 Dockerfile 例子里，拷贝的正是 `package.json pnpm-lock.yaml pnpm-workspace.yaml` 这三个文件，并且用 `corepack enable` 打开 pnpm，再 `pnpm install --frozen-lockfile`。这说明**pnpm 在 Nuxt 官方示例里是被当成默认选择来演示的**（032 展开这份配置）。

## 自查

- [ ] `npm run dev -- -o` 里那个 `--` 换到 pnpm 为什么不需要？
- [ ] `bun run dev -o` 与 `bun --bun run dev -o` 的差别，官方是在哪一行说明的？
- [ ] 你的项目里现在有几份锁文件？分别是谁生成的？
- [ ] 官方在切 nightly 频道时要求的动作是什么？只删 `node_modules` 够吗？
- [ ] 项目还没建、依赖还没装时，为什么不能用 `npm run`？

## 下一篇

031 讲编辑器这一侧：VS Code 的官方 Vue 扩展、JetBrains 系"开箱支持"具体指什么、Neovim 那个 `location` 指向包目录的坑，以及配错之后你会看到什么假象。

## 本文依据

- `getting-started/installation`（五种 create 命令与五种 dev 命令的原样写法：npm create nuxt@latest / yarn create nuxt / pnpm create nuxt@latest / bun create nuxt@latest / deno -A npm:create-nuxt@latest；npm run dev -- -o / yarn dev --open / pnpm dev -o / bun run dev -o 与注释"To use the Bun runtime during development: bun --bun run dev -o" / deno run dev -o；A browser window should automatically open for http://localhost:3000）
- `api/commands/init`（note：-- 对 npm create 是必需的，npm 会把 initializer 之后形似参数的内容读作自身配置并警告 Unknown cli config 而不传下去，其它包管理器原样转发整条命令行故直接传参 pnpm create nuxt@latest my-app --template minimal；非交互示例 npm create nuxt@latest my-app -- --template minimal --packageManager pnpm --no-gitInit；--packageManager=<npm|pnpm|yarn|bun|deno|aube|nub|upm>；create-nuxt 会询问模板、是否安装模块与使用哪个包管理器，然后安装依赖并打印后续命令）
- `guide/going-further/nightly-release-channel`（"Remove lockfile (package-lock.json, yarn.lock, pnpm-lock.yaml, bun.lock or bun.lockb) and reinstall dependencies"，切上与切下两处均给出该列表）
- `api/commands/preview`（用法以 npx nuxt preview 开头；note 中 Node.js 20+ 启动例子 NODE_ENV=production node --env-file .env .output/server/index.mjs）
- `directory-structure/package`（最小 package.json 的 scripts：build/dev/generate/preview/postinstall 五项内容均为 nuxt 子命令）
- `guide/best-practices/devcontainers`（Dockerfile 例子 FROM node:lts、RUN npm i -g corepack && corepack enable、COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./、RUN pnpm install --frozen-lockfile；换包管理器时替换 corepack enable 并改 postStartCommand）
