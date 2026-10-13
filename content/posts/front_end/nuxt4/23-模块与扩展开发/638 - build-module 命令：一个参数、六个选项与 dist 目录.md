---
url: /posts/246cec03dccb69b0b343f811a3980e1b/
title: 638 - build-module 命令：一个参数、六个选项与 dist 目录
date: 2027-05-12T11:00:00+08:00
updated: 2027-05-12T11:00:00+08:00
author: cmdragon

summary:
  摘要：`dist` 不是你手写的，是 **`nuxt build-module` 调用 `@nuxt/module-builder` 在你的 `rootDir` 里生成的**，里面装的是模块的完整构建。这条命令的入参很少：**一个参数 `ROOTDIR`（默认 `.`）＋六个选项**，其中 `--build`、`--stub`、`--sourcemap`、`--prepare` **四个默认都是 `false`**。三个要注意的点：**`--cwd` 的优先级高于 `ROOTDIR`**；**`--stub` 是"开发时用桩替代真正构建"、`--prepare` 是"为本地开发做准备"**，两者都面向开发但文档没写它们的关系；`--logLevel` 的取值是 `silent`、`info`、`verbose` 三种。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块构建
  - module-builder
  - 命令行
  - 发布流程
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块项目里 `dist` 目录经常是空的或者干脆不存在，直到你把它发到 npm，用户装下来的包才发现入口文件找不到。这一节讲的就是生成它的那条命令。

## 命令全貌

页标题给的定位是一句话：**发布前用来构建你的 Nuxt 模块的 Nuxt 命令**。完整形态：

```bash [Terminal]
npx nuxt build-module [ROOTDIR] [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--build] [--stub] [--sourcemap] [--prepare]
```

它做什么，原话是：**`build-module` 命令运行 `@nuxt/module-builder`，在你的 `rootDir` 内生成 `dist` 目录，里面包含 nuxt-module 的完整构建**。

三个信息点连在一起读：工具是 **module-builder**（不是 Vite 也不是 webpack），产物位置是 **`rootDir` 下的 `dist`**，产物内容是**完整构建**——也就是说 `src` 里的东西要经过它才变成能被 `package.json` 指向的可分发形态。

## 一个参数

| 参数 | 说明 |
| --- | --- |
| `ROOTDIR="."` | 指定工作目录（默认 `.`） |

只有这一个，而且**默认值就写在参数名里**。

## 六个选项

| 选项 | 默认 | 说明 |
| --- | --- | --- |
| `--cwd=<directory>` | | 指定工作目录，**优先级高于 `ROOTDIR`**（默认 `.`） |
| `--logLevel=<silent\|info\|verbose>` | | 指定构建期日志级别 |
| `--build` | `false` | 为分发构建模块 |
| `--stub` | `false` | 开发时用桩替代真正的 `dist` 构建 |
| `--sourcemap` | `false` | 生成 sourcemap |
| `--prepare` | `false` | 为本地开发准备模块 |

四个布尔默认全是 `false`，意味着**裸跑这条命令既不是分发构建、也不是桩、也没有 sourcemap、也不是准备**。页面没有解释裸跑时到底做什么，**这一点属于文档未说明**。

## `--cwd` 会盖掉参数

两个入参管同一件事——工作目录——但文档明确写了优先级：`--cwd` **"takes precedence over ROOTDIR"**。

所以 `nuxt build-module packages/my-module --cwd=.` 里生效的是 `.`，而不是那个位置参数。**混用时以 `--cwd` 为准**。

## 三个面向不同时刻的开关

| 你想做的事 | 该开的选项 | 文档措辞 |
| --- | --- | --- |
| 出一个能发到 npm 的产物 | `--build` | Build module for distribution |
| 开发时不想每次真构建 | `--stub` | Stub dist instead of actually building it for development |
| 让本地开发环境先准备一遍 | `--prepare` | Prepare module for local development |

`--stub` 那句是三个里信息量最大的：**它仍然产出 `dist`，只是里面是桩而非真实构建**，目的是让开发时的改动不必每次重新构建。**`--stub` 与 `--prepare` 的分工文档没有解释**，两个短语描述的是相近的场景却给了两个开关，要确认只能各自的仓库里看 module-builder 的实现（页面在末尾给了它的仓库链接与 CLI 源码链接）。

`--sourcemap` 单列，默认关闭——**分发构建不带 sourcemap 是需要显式要求的**。

日志这一项和全站命令一致，取值是 `silent`、`info`、`verbose` 三种。

## 与 starter 脚本的对应关系

starter 那一节给的是脚本名，不是命令：`npm run dev:prepare` 准备本地开发文件、`npm run prepack` 构建模块、`npm run dev` 起 playground、`npm run dev:build` 构建 playground，另外 `release` 脚本里会再构建。**这些脚本与上面六个选项之间的映射，两节都没有写**——它们由 module-builder 侧的约定决定。

要确认自己项目里的实际映射，最直接的地方是 `package.json` 的 `scripts` 字段：**把每个脚本展开看它最终调用的命令与选项**，而不是猜。这一节能确定的只有两件事：`dist` 由 `build-module` 生成，**playground 的构建不等于模块的构建**。

## 什么时候必须跑它

结构页那条 `::tip` 已经把日常情况排除了：**多数时候你不需要自己构建模块**，开发时 playground 会处理，发布时 release 脚本也会处理。所以剩下的场景其实很清楚：

- CI 里想在发布之前单独验一次产物是否完整
- 本地要排查"发出去的包为什么缺文件"
- 想给分发产物加上 sourcemap

而 release 脚本的六个动作里，**构建会发生两次**（测试阶段一次、发布时为了带上新版本号再一次），这也是为什么单跑一次 `build-module` 通过不代表发布产物一定正确。

## 自查三行

1. 跑一次带 `--build` 的命令，确认 `<rootDir>/dist` 出现且里面有模块入口。
2. 打开 `package.json`，看 `prepack` 展开后是哪条命令、带哪些选项。
3. 检查 `main`／`exports` 指向的路径是否落在 `dist` 里——**这是"用户装完找不到入口"最常见的原因**。

下一篇讲模块测试：单测那一节其实只有一个 RFC 链接。

## 下一篇

下一篇 639：模块测试：单测那一节只有一个 RFC、E2E 五步与 npm pack 手工验

## 本文依据

- https://nuxt.com/docs/4.x/api/commands/build-module
- https://nuxt.com/docs/4.x/guide/modules/getting-started
