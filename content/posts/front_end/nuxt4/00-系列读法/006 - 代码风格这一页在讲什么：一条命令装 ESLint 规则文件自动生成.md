---
url: /posts/d6d8757973c8851f1bea4418b08ce792/
title: 代码风格这一页在讲什么：一条命令装 ESLint，规则文件自动生成
date: 2026-10-14T09:00:00+08:00
updated: 2026-10-14T09:00:00+08:00
author: cmdragon

summary:
  摘要：官方对代码风格这一页的定位只有一句："Nuxt 开箱支持 ESLint"。推荐做法是使用 @nuxt/eslint 模块，它会为你的项目设置 project-aware 的 ESLint 配置；安装命令是 npx nuxt module add eslint；启动 Nuxt 应用后，项目根目录下会生成一个 eslint.config.mjs 文件，你可以按需自定义。该模块是为 ESLint 的新 flat config 格式设计的，而这个格式自 ESLint v9 起是默认格式；如果你还在用遗留的 .eslintrc 配置，需要改用 @nuxt/eslint-config 手动配置，官方强烈建议迁移到 flat config 以保证面向未来。

categories:
  - 前端开发

tags:
  - nuxt4
  - eslint
  - 代码风格
  - 模块
  - 工程化

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


概念页里 `code-style` 是最短的一页，但它决定了你项目的第一道质量闸门。这一讲我不但讲它写了什么，还会讲它**没写什么**——那部分才是新手真正会踩的。

## 官方这一页的全部信息

标题下的定位是一句：

> Nuxt 开箱支持 ESLint。

推荐做法：

> 对于 Nuxt，推荐的做法是使用 `@nuxt/eslint` 模块来启用 ESLint 支持，它将为**你的项目**设置具备项目感知的 ESLint 配置。

快速安装：

```bash
npx nuxt module add eslint
```

以及生成物落在哪：

> 启动你的 Nuxt 应用，一个 `eslint.config.mjs` 文件将被生成在你的项目根目录下。你可以根据需要自定义它。

三条信息量都不小，逐条拆开。

## 第一条要点：它是"模块"，不是"依赖"

`npx nuxt module add eslint` 这条命令的名字里写的是 **module**。这正好接上 009 讲要说的事：Nuxt 的扩展方式是模块，而模块是构建期依次执行的异步函数。也就是说 ESLint 集成不是"你自己装个包再手写配置"，而是**让模块去生成并维护配置**。

注意这一步的顺序，官方写得很清楚：**是"启动 Nuxt 应用"之后 `eslint.config.mjs` 才出现在项目根目录**。所以：

- 装完模块没跑过一次 dev/build，你去看根目录是没有这个文件的——这不是失败，是还没生成。
- 生成之后它是**你的文件**（"你可以根据需要自定义它"）。要理解这一点，把 052 讲那条硬规则放在一起看就明白了：`.nuxt` 是生成物、不能手改；而根目录下的 `eslint.config.mjs` 是交给你的、可以改。

## 第二条要点："project-aware" 意味着规则跟着你的项目

官方用词是"设置具备项目感知的（project-aware）ESLint 配置"。这个词值得解释，因为它是很多人手写配置时丢掉的东西：Nuxt 项目里有 `.vue` 单文件组件、有自动导入、有 `#imports` 这类路径别名，还有 `app/` 与 `server/` 两套上下文。手写一份 ESLint 配置时，最容易出的问题恰恰是这三类：

- 认不出 `.vue` 里的 `<script setup>`；
- 把 `ref`、`useFetch`、`useState` 这类自动导入的名字报成"未定义变量"；
- 认不出 `~/`、`#imports` 这类别名解析。

模块生成的配置就是为处理这些而存在的。**这不是我从"project-aware"这个词推出来的功能清单**——官方在这一页只给了词和指向模块文档的链接（"你可以在 Nuxt ESLint 的文档中了解更多关于该模块与自定义的内容"）。真要定制规则，去那一页查，别在配置里瞎猜。

## 第三条要点：flat config 是默认，`.eslintrc` 是遗留

官方专门放了一段 callout，说的是配置格式：

> 该模块是为**新的 ESLint flat config 格式**设计的，这一格式**自 ESLint v9 起是默认格式**。如果你使用的是遗留的 `.eslintrc` 配置，你需要使用 `@nuxt/eslint-config` **手动配置**。我们强烈建议你迁移到 flat config，以面向未来。

于是有一个很现实的判断：

| 你的情况 | 该怎么做 |
| --- | --- |
| 新项目、ESLint v9+ | 直接 `npx nuxt module add eslint`，用生成的 `eslint.config.mjs` |
| 老项目带着 `.eslintrc.*` | 官方给的路是改用 `@nuxt/eslint-config` 手动配置；并且**强烈建议迁移到 flat config** |
| 网上抄来的配置写着 `extends: [...]` | 那大概率是遗留格式的规则集写法，别直接搬进 `eslint.config.mjs` |

最后那行是我给的操作提醒，不是官方内容——但它能省你一次"配置抄了没用"的时间。

## 这一页没写的三件事，你必须知道

这一页只讲 lint，不讲另外两件同样管代码质量的事。新手常把它们混在一起：

**一、类型检查不在这页。** 它属于 `typescript` 概念页（下一篇就讲），而且默认**不开**：Nuxt 不会在你跑 `nuxt dev` 或 `nuxt build` 时检查类型（性能原因）。所以"我装了 ESLint，类型就安全了"这个推理是不成立的——**lint 管的是风格与可疑写法，类型检查管的是类型**。

**二、格式化（Prettier 那一类）也不在这页。** 官方这一页只出现 ESLint 一个工具名。要不要再加格式化工具是你的项目决策，官方没有替你做。

**三、自动导入的 lint 支持才是这套组合的关键。** 005 讲说过"配置项名字拼错不会报错"，同理，**自动导入的名字如果不被 lint 认识，你会得到满屏假报错**。这正是用模块（而不是手写配置）的价值所在——但这一条属于我的判断，官方在这一页没写具体规则项。

## 装完之后建议做的三件事

1. 跑一次 dev，确认根目录出现了 `eslint.config.mjs`（官方给的就是"启动后生成"）。
2. 打开一个 `.vue` 文件，故意写一个拼错的自动导入名（比如 `useFetsh`），看它是否被报错——这是验证"项目感知配置真的生效"最快的办法。
3. 需要定制时，先读模块文档再改文件，因为官方明确说了"你可以根据需要自定义它"，也明确说了自定义细节在模块文档里。

## 自查

- [ ] 你知道 ESLint 集成在 Nuxt 里是以**模块**形式安装的吗？
- [ ] 装完有没有启动一次应用（配置文件是启动时生成的）？
- [ ] 你的项目是 flat config（`eslint.config.mjs`）还是遗留 `.eslintrc`？后者的话官方建议你怎么做？
- [ ] 有没有把"lint 通过"当成"类型检查通过"？
- [ ] 自动导入的名字在你的编辑器里是否被 lint 正常识别（没有满屏假报错）？

## 下一篇

007 - 类型是谁给的：先建立「Nuxt 会替你生成类型」的认知

## 本文依据

- `guide/concepts/code-style`（"Nuxt 开箱支持 ESLint"；推荐使用 `@nuxt/eslint` 模块为项目设置 project-aware 的 ESLint 配置；快速安装命令 `npx nuxt module add eslint`；"启动你的 Nuxt 应用，一个 `eslint.config.mjs` 文件将被生成在项目根目录下，你可以按需自定义它"；模块为新的 flat config 格式设计、该格式自 ESLint v9 起为默认、遗留 `.eslintrc` 需用 `@nuxt/eslint-config` 手动配置、强烈建议迁移 flat config、更多模块与自定义内容见 Nuxt ESLint 文档）
- `getting-started/configuration`（配置项拼错无提示、建议 `.ts` 以获 IDE 提示）
- `guide/concepts/typescript`（默认不做类型检查、需 `vue-tsc` + `nuxt typecheck` 或 `typescript.typeCheck`）
- `directory-structure/nuxt`（`.nuxt` 为生成物不可手改，与根目录可自定义文件的对照）
