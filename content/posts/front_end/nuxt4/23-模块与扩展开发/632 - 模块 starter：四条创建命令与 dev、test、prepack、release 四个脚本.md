---
url: /posts/d0ed79b125d92e7a7702e1efe27de08f/
title: 632 - 模块 starter：四条创建命令与 dev、test、prepack、release 四个脚本
date: 2027-05-10T11:00:00+08:00
updated: 2027-05-10T11:00:00+08:00
author: cmdragon

summary:
  摘要：写模块不用从空目录开始。`npm create nuxt -- -t module my-module` 会生成一个已经配好结构的项目，源码在 `src`，另外**带一个现成的 Nuxt 应用 `playground`**，专门用来验证模块生效没有。官方给了四条创建命令，注意它们的参数写法并不一致：**npm 那条要加 `--`，bun 那条用的是 `--template=module`**。装完依赖后必须先跑 `npm run dev:prepare`。日常只有六个脚本：`dev`、`dev:build`、`lint`、`test`、`prepack`、`release`。其中 `prepack` 由 `@nuxt/module-builder` 提供、**多数时候不需要手动跑**，而 `release` 会替你做完 lint、test、build、改版本号、发 npm、推 tag 六件事。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块开发
  - playground
  - module-builder
  - 发布流程
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块写完最麻烦的一件事是"怎么看它生效了没有"。官方 starter 的解法是**在你写模块的项目里塞一个完整的 Nuxt 应用**，让它直接引用你正在改的那份源码。这一篇讲清楚生成命令、目录分工，以及那几个脚本各自什么时候用。

## 五个词先说清

- **`src` 目录**：模块源码所在。
- **`playground` 目录**：一个真的 Nuxt 应用，已经配置成加载你的模块。
- **prepare**：让 Nuxt 把类型和内部文件先生成一遍。
- **prepack**：把源码打包成可发布的产物。
- **release script**：一键跑完测试再发布的脚本。

## 创建：四条命令，两种参数写法

官方推荐的起点是 starter 模板，四条命令对应四种包管理器。

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

**npm 那条多出来的 `--` 是分隔符**，它后面的 `-t module` 才是传给模板工具的参数，少了这个 `--`，参数会被 npm 自己吃掉。**bun 那条把短参换成了 `--template=module` 的长写法**，另三条都写 `-t module`。这是官方四条命令里唯一不一致的地方，抄错的表现通常是它开始问你要哪个模板。

跑完会得到一个 `my-module` 项目，官方说法是"里面有开发和发布模块所需的全部样板"。

## 四步后续动作

页面上明确列了编号四步：

1. 用你惯用的 IDE 打开 `my-module`
2. 用你惯用的包管理器安装依赖
3. **用 `npm run dev:prepare` 为开发准备好本地文件**
4. 接着看这一节了解模块的写法

第 3 步不是可选的。跳过它的表现是模块项目里那些该生成的类型与构建产物还不存在，`dev` 起不来或者 IDE 里一片红的导入路径。**它排在"安装依赖"之后、"启动开发服务器"之前**，这个位置是官方给的顺序。

## playground 是拿来验证的

这一节的关键句是：模块源码在 `src` 里，**但开发模块时你往往需要一个 Nuxt 应用来对着测**，`playground` 就是干这个的——一个你可以随便折腾、且已经配置成会用你模块的应用。

和它交互跟和普通 Nuxt 应用一样：

- `npm run dev` 启动它的开发服务器，**你在 `src` 目录里改动时它应该会自己重载**
- `npm run dev:build` 构建它

页面末尾的 `::note` 补了一句：**其余所有 `nuxt` 命令都能对着 `playground` 用**，形式是 `nuxt <COMMAND> playground`；并且鼓励你在 `package.json` 里再声明几个 `dev:*` 脚本引用它们。

这意味着你想跑 `nuxt build`、`nuxt generate` 之类的动作，不需要把模块装到别的项目里去验，直接指 playground。

## 测试是两件现成的事

starter 自带一套基础测试：

| 工具 | 命令 |
| --- | --- |
| ESLint 驱动的 linter | `npm run lint` |
| Vitest 驱动的测试运行器 | `npm run test` 或 `npm run test:watch` |

页面在这一节末尾有个 `::tip`，原话是"这套默认测试策略你可以按需增补"。**它没有约定测试文件放哪、也没有默认覆盖率的门槛**，这两件事都得自己加。

## 构建：一个不需要配置的 builder

模块自带构建器，由 `@nuxt/module-builder` 提供。官方给了三个特点：**你这边不需要任何配置、支持 TypeScript、会把你的资源正确打包以便分发给别的 Nuxt 应用**。

命令是：

```bash
npm run prepack
```

紧接着一条 `::tip` 把这条命令的必要性降到最低：

> While building your module can be useful in some cases, most of the time you won't need to build it on your own: the `playground` takes care of it while developing, and the release script also has you covered when publishing.

**开发时 playground 会处理构建，发布时 release 脚本也会处理**。所以要记住的反而是一个反面结论：`prepack` 不是你每次改完代码都要跑的那一步。

## 发布：release 脚本替你做的六件事

发布前有一条 `::important`：先确认你有 npmjs.com 账号，**并且本地已经用 `npm login` 登录过**。

官方的态度是：你当然可以直接改版本号然后 `npm publish`，但 starter 提供了一个 release 脚本，确保你发出去的是一个能用的版本。**用法是先提交所有改动，再跑 `npm run release`**。提交时推荐遵循 Conventional Commits，因为**版本号提升和 changelog 生成是从提交信息推出来的**。

`npm run release` 的执行顺序页面写得很明确，分两个阶段：

```
先跑测试套件：
  1. 运行 linter（npm run lint）
  2. 运行单元、集成与 e2e 测试（npm run test）
  3. 构建模块（npm run prepack）
测试通过后开始发布：
  4. 按 Conventional Commits 提升模块版本号并生成 changelog
  5. 把模块发布到 npm（为此会再构建一次，确保发布产物里带上新的版本号）
  6. 把一个代表新版本的 git tag 推到你的 git remote origin
```

两个细节值得单独注意。

**第 5 步会再构建一次**，官方解释是"确保发布产物里带上了更新后的版本号"。**这就是为什么手改版本号再 `npm publish` 容易出问题**：产物可能还是旧版本号打的那一份。

**第 6 步会往远端推 tag**。这一步是对外可见的动作，脚本把它放在发布之后——也就是说**如果前 5 步都过了，tag 推出去时模块已经在 npm 上了**。想改这个行为，页面末尾的 `::tip` 说的是：这个 `release` 脚本本身就在你的 `package.json` 里，可以随需微调。

## 六个脚本各自的时机

| 脚本 | 什么时候跑 | 会不会碰远端 |
| --- | --- | --- |
| `dev:prepare` | 装完依赖后第一次，只做一次为主 | 否 |
| `dev` | 写代码时常驻 | 否 |
| `dev:build` | 想验证 playground 能构建 | 否 |
| `lint` | 提交前，或被 `release` 自动跑 | 否 |
| `test` / `test:watch` | 改完逻辑，或被 `release` 自动跑 | 否 |
| `prepack` | 一般不用手动跑，`release` 里跑两次 | 否 |
| `release` | 决定发新版本时 | **会**：发 npm ＋ 推 git tag |

上手路径就是这四条命令里挑一条生成项目、跑一次 `dev:prepare`，然后把 `dev` 挂着写 `src`。

下一篇讲模块本体的三段结构：meta、defaults 和 setup。

## 下一篇

下一篇 633：模块定义文件长什么样：两种写法、五个键与 wrapper 的八条保证

## 本文依据

- https://nuxt.com/docs/4.x/guide/modules/getting-started
