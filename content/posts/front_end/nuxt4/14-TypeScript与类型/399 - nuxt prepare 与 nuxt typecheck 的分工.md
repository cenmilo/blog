---
url: /posts/9a6ddcd2b8842b276a9e52183c14a3d1/
title: 399 - nuxt prepare 与 nuxt typecheck 的分工
date: 2027-02-22T09:00:00+08:00
updated: 2027-02-22T09:00:00+08:00
author: cmdragon

summary:
  摘要：两个命令名字都像"跟类型有关"，做的却是链上前后两步。`nuxt prepare` **只生成**——官方定义是"creates a .nuxt directory in your application and generates types"，推荐用在 CI 与 `package.json` 的 `postinstall`；它不检查任何类型。`nuxt typecheck` **才真跑检查**——"runs vue-tsc or Golar to check types throughout your app"，两个都没装时会直接把对应包管理器的安装命令递给你。两个命令都会把 `process.env.NODE_ENV` 设成 `production`（typecheck 额外说明可在 `.env` 或命令行参数覆盖）。typecheck 还有 `--checker=<vue-tsc|golar>` 选检查器、`-b/--build` 与 `--no-build` 控制是否走 TypeScript project references 的构建模式（solution 风格的 tsconfig 会自动按 build 模式检查）。

categories:
  - 前端开发

tags:
  - nuxt4
  - prepare
  - typecheck
  - vue-tsc
  - CI
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

CI 里想校验类型，很多人直接跑 `nuxt prepare`，结果一个类型错误都没报出来——因为 prepare 压根不做检查。这两个命令是一前一后两步，职责不同，选项也不同。

## 一句话分工

- **`nuxt prepare`**：官方定义 "The prepare command creates a `.nuxt` directory in your application and generates types."——建 `.nuxt`、生成类型，**到此为止**。
- **`nuxt typecheck`**：官方定义 "The typecheck command runs vue-tsc or Golar to check types throughout your app."——调起真正的类型检查器。

先 prepare（或由 dev/build 顺带生成）才有类型可查；typecheck 是在有类型之后去核对它们。

## nuxt prepare

官方给了它的用途定位：

> This can be useful in a CI environment or as a `postinstall` command in your `package.json`.

所以标准做法就是把 `nuxt prepare` 挂到 `postinstall`（075 提过），clone 完 `npm install` 就自动把类型生成了；CI 在跑 lint/typecheck 之前也要先有这一步。

命令形态：

```bash
npx nuxt prepare [ROOTDIR] [--cwd=<directory>] [--dotenv=<path>...] [--logLevel=<silent|info|verbose>] [--envName=<environment>] [-e, --extends=<layer-name>...]
```

| 项 | 说明（依文档） |
| --- | --- |
| `ROOTDIR` | 项目根目录，默认 `.` |
| `--cwd=<directory>` | 指定 Nuxt 项目根目录 |
| `--dotenv=<path>...` | 加载哪个 `.env`，相对根目录；可重复，靠后的优先 |
| `--logLevel=<silent\|info\|verbose>` | 构建期日志级别 |
| `--envName=<environment>` | 解析配置覆盖用哪个环境（构建时默认 `production`、dev server 默认 `development`） |
| `-e, --extends=<layer-name>...` | 从某个 Nuxt layer 扩展 |

官方 `::note`：**这个命令会把 `process.env.NODE_ENV` 设成 `production`。**

## nuxt typecheck

命令形态：

```bash
npx nuxt typecheck [ROOTDIR] [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--dotenv=<path>...] [-e, --extends=<layer-name>...] [--checker=<vue-tsc|golar>] [-b, --build]
```

| 项 | 说明（依文档） |
| --- | --- |
| `ROOTDIR` | 项目根目录，默认 `.` |
| `--cwd=<directory>` | 指定 Nuxt 项目根目录 |
| `--logLevel=<silent\|info\|verbose>` | 构建期日志级别 |
| `--dotenv=<path>...` | 加载哪个 `.env`；可重复，靠后的优先 |
| `-e, --extends=<layer-name>...` | 从某个 Nuxt layer 扩展 |
| `--checker=<vue-tsc\|golar>` | 用哪个类型检查器 |
| `-b, --build` | 以构建模式做类型检查，使用 TypeScript project references（默认自动检测） |
| `--no-build` | 不用 TypeScript project references 做类型检查 |

官方补了一句很贴心的行为：**两个检查器都没装时，它会把对应包管理器的安装命令提供给你**（"If neither is installed you are offered the install command for your package manager."）。

## 两个检查器：vue-tsc 与 Golar

`--checker golar` 用 [Golar](https://golar.dev/languages/vue/) 代替 `vue-tsc`；官方还给了默认规则：

> `--checker golar` runs Golar instead of `vue-tsc`, and a `golar.config.*` file in your project makes it the default. A config file is written for you if one is missing.

也就是项目里存在 `golar.config.*` 时，Golar 就是默认检查器；缺这个文件的话 Nuxt 会给你写一个。

注意和 397 那条呼应：`typescript.typeCheck` 与类型检查依赖 `typescript` + `vue-tsc`（官方 "Requires to install `typescript` and `vue-tsc` as dev dependencies."），装它们的做法在 typescript 那页（`npm install --save-dev vue-tsc typescript` 等五种包管理器写法）。

## build 模式与 project references

395 讲过 Nuxt 用 TypeScript project references 把项目切成四份。typecheck 的 `-b/--build` 就是顺着这套走构建模式，官方给了自动检测规则：

> A solution-style `tsconfig.json` (one that only lists `references`) is type-checked in build mode automatically. `--build` forces it, and `--no-build` turns it off.

因为根 `tsconfig.json` 正是 `"files": []` + 四条 `references` 的 solution 风格（395），所以默认就会按 build 模式检查——这也解释了 395 说的 project references 好处之一"Faster builds：TypeScript 可以跳过没变的项目"。要绕开用 `--no-build`。

typecheck 也设 `NODE_ENV=production`，但它额外给了覆盖方式：

> This command sets `process.env.NODE_ENV` to `production`. To override, define `NODE_ENV` in a [`.env`] file or as a command-line argument.

prepare 的那条 `::note` 没有"如何覆盖"这半句，两页在这点上写得不对称。

## 自查

- prepare 只生成 `.nuxt` 与类型（CI 与 `postinstall` 用），不检查任何东西；会把 `NODE_ENV` 设成 `production`。
- typecheck 才跑 `vue-tsc` 或 Golar；两者都没装会递上安装命令；同样设 `NODE_ENV=production`，且明确可用 `.env` 或命令行覆盖。
- 选检查器用 `--checker=golar`，项目里有 `golar.config.*` 时 Golar 成为默认（缺文件会帮你生成）。
- 根 tsconfig 是 solution 风格（只有 references），所以默认自动按 build 模式检查；`--build` 强制、`--no-build` 关掉。
- 类型检查依赖装了 `typescript` + `vue-tsc`。

## 下一篇

下一篇 400 进入路由的类型化：`typedPages` 实验开关打开后，`navigateTo`、`<NuxtLink>`、`router.push()` 的入参开始按路由表受类型约束。

## 本文依据

- prepare 定义、"useful in a CI environment or as a `postinstall` command"、ROOTDIR 与五个选项表、`::note` 设 `NODE_ENV=production`（`api/commands/prepare`）；typecheck 定义与 vue-tsc/Golar、"If neither is installed you are offered the install command"、ROOTDIR 与七个选项表（含 `--checker`、`-b/--build`、`--no-build`）、Type checkers 与 build mode 两段、`::note`（设 `NODE_ENV=production` 可在 `.env` 或命令行覆盖）（`api/commands/typecheck`）；依赖 `typescript`+`vue-tsc` 与安装写法、project references 与四份 tsconfig（`guide/concepts/typescript`、`api/nuxt-config`，见 395/397）。
