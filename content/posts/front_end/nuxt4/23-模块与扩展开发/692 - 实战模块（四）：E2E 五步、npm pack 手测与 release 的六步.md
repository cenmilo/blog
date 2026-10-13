---
url: /posts/1f311fcf423529bbc5396a872c9caf3c/
title: 692 - 实战模块（四）：E2E 五步、npm pack 手测与 release 的六步
date: 2027-05-30T11:00:00+08:00
updated: 2027-05-30T11:00:00+08:00
author: cmdragon

summary:
  摘要：模块发出去之前有三段路：仓库内用 fixture 跑端到端测试，仓库外用 `npm pack` 打一个 tarball 装进真实项目，最后交给 `npm run release` 走那六步。中间那个构建动作有专门的命令 `nuxt build-module`，六个开关里最容易选错的是 `--stub` 和 `--build`：前者是给开发用的打桩、不真正构建，后者才是为分发而构建。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块开发
  - 端到端测试
  - 发布流程
  - 构建命令
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 第一段：仓库内的端到端测试

测试这一节给的路线只有端到端是完整的：用 Nuxt Test Utils，五步走——在 `test/fixtures/*` 里建一个作为 fixture 的 Nuxt 应用、在测试文件里用这个 fixture 启动 Nuxt、用 `@nuxt/test-utils` 的工具与 fixture 交互（例如抓一个页面）、针对这个 fixture 做断言（例如某段 HTML 是否出现）、然后重复。

fixture 的配置文件和测试文件的完整写法在讲模块测试那两篇里逐行给过，这里只提三个会在发布前咬人的点：

- `describe` 的回调本身要 `async`，`setup` 在里面被 `await`；写成同步的表现是断言拿不到服务地址。
- fixture 的 `nuxt.config.ts` 里 `modules: [MyModule]` 引的是 `../../../src/module`，**测的是源码而不是构建产物**，所以这一步绿了并不等于 `dist` 里那个包能用。
- 断言的对象是渲染出来的 HTML 字符串，所以 SSR 与 CSR 的 fixture 要分开写（文档最后一行 `describe('csr', ...)` 就是这个意思）。

顺带把预期校准一下：单元测试那一节整节只有一条 tip，说他们仍在讨论和探索如何简化模块的单元与集成测试，并给了一个 RFC 链接。**官方文档目前不提供模块单测的推荐写法**。

## 第二段：仓库外用 tarball 装一次

这一步补的正是上面那个缺口。文档的说法是：playground 用于开发时的测试，而你要在**不属于你模块仓库的应用**里测它，就用 `npm pack`（或你包管理器的等价命令）从模块生成一个 tarball，然后在测试项目的 `package.json` 里把它加成一个依赖：

```json
"my-module": "file:/path/to/tarball.tgz"
```

装完之后就可以像在任何常规项目里那样引用 `my-module`。

这一段的产出物才是用户实际拿到的东西：打包后的 `dist`、`package.json` 的导出字段、资源是否被正确带上，都只在这里暴露。前一篇检查清单里那句"playground 能跑、发布后跑不了，通常是这里的差异造成的"，说的就是这一步没做。

## 中间那个动作：`nuxt build-module` 的六个开关

模块自己的构建由 `@nuxt/module-builder` 完成，命令页对它的定位一句话：**运行 `@nuxt/module-builder`，在你的 `rootDir` 下生成 `dist` 目录，里面是 nuxt-module 的完整构建**。

```bash [Terminal]
npx nuxt build-module [ROOTDIR] [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--build] [--stub] [--sourcemap] [--prepare]
```

参数只有一个：

| 参数 | 说明 |
| --- | --- |
| `ROOTDIR="."` | 指定工作目录（默认：`.`） |

选项六个，默认值都列出来：

| 选项 | 默认 | 说明 |
| --- | --- | --- |
| `--cwd=<directory>` | | 指定工作目录，**优先于 ROOTDIR**（默认：`.`） |
| `--logLevel=<silent\|info\|verbose>` | | 指定构建期日志级别 |
| `--build` | `false` | 为分发构建模块 |
| `--stub` | `false` | 为开发打桩 dist，而不是真正构建它 |
| `--sourcemap` | `false` | 生成 sourcemaps |
| `--prepare` | `false` | 为本地开发准备模块 |

三个开关之间的关系要读准：

- `--stub` 与 `--build` 是方向相反的两个动作。打桩出来的 `dist` 是给开发循环用的，**发布不能带着它**；`--build` 才产生分发产物。
- `--prepare` 是本地开发准备，starter 流程里那条 `npm run dev:prepare` 干的就是这件事。
- `--cwd` 和 `ROOTDIR` 都能指定工作目录，文档明确了优先级在 `--cwd`。

**文档没有说明这几件事**：`--stub` 与 `--build` 同时给出会怎样；`--sourcemap` 产出的映射放在哪里；`npm run prepack` 与这条命令的关系（starter 页只说构建用 `@nuxt/module-builder`、命令是 `prepack`，没有贴出脚本内容）。要确认得自己打开 starter 生成的 `package.json` 看那条脚本实际写了什么。

## 第三段：`npm run release` 把上面这些串起来

发布的前提在重要提示里：先有 npmjs.com 账户，并且本地已经 `npm login` 认证过。

你可以手动升版本再 `npm publish`，但 starter 的 release 脚本存在的理由是"确保你发布的是一个能工作的版本"。用它的顺序是：先提交所有改动（建议遵循 Conventional Commits，这样才能享受自动版本递增和 changelog 更新），然后 `npm run release`。

它做的六步，文档分两组给出：

- 先跑测试套件：执行 lint（`npm run lint`）、执行单元/集成/端到端测试（`npm run test`）、构建模块（`npm run prepack`）
- 测试通过才发布：按 Conventional Commits 递增版本并生成 changelog、把模块发布到 npm、把代表新版本的 git 标签推到 git remote

第六步里藏着一个细节值得单独注意：**为了确保发布产物用的是更新后的版本号，模块会被再构建一次**。也就是说 `prepack` 在这条链上至少跑了两遍，第一遍是为了测试，第二遍是为了带着新版本号发布。

## 三段路的顺序不能换

- 仓库内 E2E 绿：说明模块逻辑在你的 fixture 配置下成立。
- `npm pack` + 外部项目装上：说明构建产物是自洽的。
- `npm run release`：说明测试、版本、发布、标签是一件事，而不需要你自己记顺序。

如果只做了第一步就发，你发出去的是一个从没被真正构建过的东西。

到这里，模块这一段的正文就结束了。下一篇转向另一件事：实验特性机制——那 57 个开关的默认值、"提前启用"意味着什么，以及官方对风险的那句声明。

## 下一篇

下一篇 693：Kit 速查表：59 个函数分布在 20 页与四种查找入口

## 本文依据

- https://nuxt.com/docs/4.x/guide/modules/testing
- https://nuxt.com/docs/4.x/api/commands/build-module
- https://nuxt.com/docs/4.x/guide/modules/getting-started
