---
url: /posts/036f228b82516592ac69416fe688b9d6/
title: nuxt task：定义并调用后台任务
date: 2027-01-27T09:00:00+08:00
updated: 2027-01-27T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：Nitro 有"后台任务（tasks）"这个概念——可被调用的服务端作业，比如填种子数据。Nuxt 给的入口是 nuxt task 命令，两个子命令：nuxt task list 列出项目暴露的任务、nuxt task run <NAME> 跑一个并可用 --payload 传参。这一篇讲清楚它怎么和"正在跑的 nuxt dev 服务器"对话（也能 --url 指到别的服务器），最关键的一条前提是：Nitro 只有在你开了 nitro.experimental.tasks: true 之后才去扫 tasks 目录、才谈得上"有任务可列可跑"。还要点明：任务本身怎么定义属于 Nitro 的事，Nuxt 文档这页只讲 CLI 和那个开关。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt-task
  - Nitro任务
  - CLI
  - payload
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

有些服务端活儿不是给浏览器调的接口，而是"手动触发一次作业"——灌种子数据、批量重建索引。Nitro 把这叫 tasks（后台任务），Nuxt 给的入口是 `nuxt task` 命令。

## 命令是干嘛的

官方定位：

> List and run Nitro tasks on your dev server.

以及它的工作方式：

> The `task` command talks to the Nitro tasks your project exposes, using the `nuxt dev` server running for it. Pass `--url` to target another server.

关键一句：**它是"对话"一个正在运行的服务器来列出/触发任务的**——默认对着你为这个项目跑着的 `nuxt dev`；要打到别的服务器，传 `--url`。

## 先开那个开关，否则没任务

这是这一篇最该记住的前提。官方 note：

> Nitro only scans your `tasks` directory when tasks are enabled, so a server with none exposed may just need this:

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  nitro: {
    experimental: {
      tasks: true,
    },
  },
})
```

翻译给学生：`nuxt task list` 列出来是空的、或"没有任务"，很多时候不是你没写任务，而是**任务功能默认关着**——只有 `nitro.experimental.tasks: true` 打开后，Nitro 才去扫 `tasks` 目录、才把里面的任务暴露出来。上一条（320）讲的"某些 Nitro 能力要显式开"，这里就是实例。

## 子命令一：nuxt task list

列出项目暴露的任务：

```bash [Terminal]
npx nuxt task list [ROOTDIR] [--cwd=<directory>] [--url=<url>] [--json]
```

官方参数与选项：

| 项 | 说明 |
| --- | --- |
| `ROOTDIR`（位置参数） | Nuxt 项目根目录，默认 `.` |
| `--cwd=<directory>` | 指定项目根目录 |
| `--url=<url>` | 要对话的服务器地址，默认那个正在跑的 dev server |
| `--json` | 以 JSON 打印输出 |

## 子命令二：nuxt task run

跑一个任务：

```bash [Terminal]
npx nuxt task run <NAME> [ROOTDIR] [--cwd=<directory>] [--url=<url>] [--payload=<json>]
```

| 项 | 说明 |
| --- | --- |
| `NAME=<name>` | 要运行的任务名（必填） |
| `ROOTDIR` | 项目根目录，默认 `.` |
| `--cwd` / `--url` | 同上 |
| `--payload=<json>` | 任务入参，可给一个 JSON 对象，或用 `--payload.key=value` 逐键拼 |

官方说明结果与传参：

> The task's result is printed as JSON. A payload can be given as a single JSON object or built up from individual keys:

```bash [Terminal]
npx nuxt task run db:seed --payload '{"count":10}'
npx nuxt task run db:seed --payload.count=10
```

两种传 payload 的写法等价（一个整 JSON、一个逐键）。任务名可以是带命名空间的（示例 `db:seed`）。结果以 JSON 打印。

## 任务本身怎么定义：那是 Nitro 的事

要说清楚边界：**这页 Nuxt 文档只讲 CLI 和那个开关，没教你怎么写一个 task 文件**。任务定义属于 Nitro——官方给了 read-more 指向 Nitro 的 tasks 文档（"Read more about Nitro tasks"）。同理，4.x 的 `server/` 目录页也**没有** `server/tasks` 这一节（301 提过），别按旧教程凭空建一个再指望它自动生效——先确认你开了 `nitro.experimental.tasks`。

所以顺序是：开 `nitro.experimental.tasks: true` → 按 Nitro 文档在项目里定义任务 → `nuxt task list` 能看到、`nuxt task run <NAME>` 触发。

## 自查

- `nuxt task` 是"对话一个正在跑的服务器"来列/跑任务，默认那个 `nuxt dev`，别的用 `--url`。
- 列出来是空的？先确认开了 `nitro.experimental.tasks: true`——任务默认关、不开不扫。
- 传参用 `--payload`：一个 JSON 对象或 `--payload.key=value` 逐键；结果以 JSON 打印。
- 任务定义本身查 Nitro tasks 文档，这页 Nuxt 文档只覆盖 CLI 与开关；别照旧教程乱建 `server/tasks` 而不开关。

## 下一篇

下一篇 322 讲 `nuxt curl`：直接对项目正在跑的 dev server 发一条 HTTP 请求、打印响应，做接口冒烟不用先记端口号。

## 本文依据

- nuxt task（`api/commands/task`）：定位句 "List and run Nitro tasks on your dev server."；"talks to the Nitro tasks your project exposes, using the `nuxt dev` server running for it. Pass --url to target another server."；`nuxt task list` 命令签名 + Arguments（ROOTDIR 默认 .）+ Options（--cwd/--url 默认运行中的 dev server/--json）；`nuxt task run` 签名 + Arguments（NAME 必填、ROOTDIR）+ Options（--cwd/--url/--payload）；"The task's result is printed as JSON. A payload can be given as a single JSON object or built up from individual keys" + `db:seed --payload '{"count":10}'` 与 `--payload.count=10` 两示例；note "Nitro only scans your `tasks` directory when tasks are enabled" + `nitro.experimental.tasks: true` 配置；read-more 指向 Nitro tasks 文档。
- server 目录（`directory-structure/server`）：无 server/tasks 一节（301 记录）；nitro 配置节（320）。
