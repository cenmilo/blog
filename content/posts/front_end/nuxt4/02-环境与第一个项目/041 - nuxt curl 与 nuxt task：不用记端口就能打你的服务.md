---
url: /posts/87a4f11f085b03c8d674b231c65e16a8/
title: nuxt curl 与 nuxt task：不用记端口就能打你的服务
date: 2026-10-25T11:00:00+08:00
updated: 2026-10-25T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：nuxt curl 向"你项目正在运行的 nuxt dev 服务器"发一个 HTTP 请求并打印响应，路径按那个服务器解析，所以你不必知道它最终落在哪个端口——它是通过 dev 写在构建目录里的 nuxt.lock 找到服务器的，因此只在 nuxt dev 为该项目运行时可用；要打别的服务器就传绝对 URL。它的开关沿用了 curl 的习惯：-X/--method（默认 GET，给了 body 就是 POST）、-H/--header（Name: Value，可重复）、-d/--data（用 @- 读 stdin、@<file> 读文件）、-i/--include 带状态行与响应头、-I/--head 只发 HEAD 并只看响应头、-v/--verbose 把请求与响应头打到 stderr、--pretty 重排并高亮（终端默认开、管道时逐字节透传，可用 --pretty/--no-pretty 强制）。还有一条容易忽略的行为：服务器返回 HTTP 错误状态时它以 22 退出，官方标注这是匹配 curl --fail。nuxt task 则打 Nitro 任务，分 task list 与 task run <NAME>，--payload 可以传一个 JSON 对象也可以写成 --payload.key=value；官方特别提醒 Nitro 只在任务启用时才扫描你的 tasks 目录，所以一个"没有任务暴露"的服务器往往只需要设 nitro.experimental.tasks: true。

categories:
  - 前端开发

tags:
  - nuxt4
  - curl
  - task
  - 调试
  - 命令行

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


033、034 反复遇到一个词：`nuxt.lock`。这一篇讲两个**靠它工作**的命令。它们的价值都很具体：**你不用再记端口，也不用再打开浏览器点一遍。**

## `nuxt curl`：打的是你项目那个 dev 服务器

官方定义：

> `curl` 命令向你项目中运行的 `nuxt dev` 服务器发送一个 HTTP 请求并打印响应。**路径按那个服务器解析，所以你不需要知道它最终落在了哪个端口。**

它怎么找到服务器，官方写在下面这段里：

> dev 服务器是通过它写入你[构建目录](https://nuxt.com/docs/4.x/directory-structure/nuxt)的 `nuxt.lock` 文件被找到的，**所以这条命令只在 `nuxt dev` 为该项目运行时才可用**。传一个绝对 URL 可以和任何其它服务器通信。

这段话给出两个使用边界：

- **"它最终落在哪个端口"这件事彻底不用管。** 034 我们看过端口有四级优先级、还可能被自动换掉；`nuxt curl /api/hello` 不受这些影响。
- **dev 没跑着，命令就没目标。** 报"找不到服务器"时，先确认 dev 在跑，而不是怀疑路径写错。

## 参数与选项

```bash [Terminal]
npx nuxt curl <URL> [ROOTDIR] [--cwd=<directory>] [-X, --method=<method>] [-H, --header=<header>...] [-d, --data=<data>] [-i, --include] [-I, --head] [-v, --verbose] [--pretty]
```

| 项 | 官方说明 |
| --- | --- |
| `URL=<url\|path>` | 绝对 URL，**或一个按正在运行的 dev 服务器解析的路径** |
| `ROOTDIR` | 项目根目录（默认 `.`） |
| `--cwd=<directory>` | 指定项目根目录 |
| `-X, --method=<method>` | HTTP 方法（**默认 GET；提供了 body 时是 POST**） |
| `-H, --header=<header>...` | 形如 `Name: Value` 的请求头，**可重复** |
| `-d, --data=<data>` | 请求体。**用 `@-` 从 stdin 读取，用 `@<file>` 读取文件** |
| `-i, --include` | 在输出里**包含状态行与响应头** |
| `-I, --head` | 发送 `HEAD` 请求，**只显示响应头** |
| `-v, --verbose` | 把请求与响应头打到 **stderr** |
| `--pretty` | 重排缩进并语法高亮输出（**默认：终端里开、管道时关**）。用 `--pretty`/`--no-pretty` 强制 |

官方给的四条例子，覆盖了日常四种用法：

```bash [Terminal]
# Request a path on the running dev server
npx nuxt curl /api/hello

# Include the status line and response headers
npx nuxt curl /api/hello -i

# Send JSON
npx nuxt curl /api/todos -X POST -H "Content-Type: application/json" -d '{"title":"Buy milk"}'

# Read the body from a file, or from stdin with `@-`
npx nuxt curl /api/todos -d @todo.json
```

## 两条会直接影响脚本的行为

这两条不是"锦上添花"，而是**你在 CI 或管道里必须知道的**。

**一：错误状态会以退出码 22 结束。**

> 当服务器回答一个 HTTP 错误状态时，命令以 `22` 退出，**匹配 `curl --fail`**。

含义：`nuxt curl` **不会**因为 404 或 500 而"正常返回并把错误页打印出来"，它会以非零码结束。写检查脚本时这是好事（能判断），但如果你照搬"curl 一下看看返回什么"的习惯，会以为命令本身崩了。

**二：管道里是逐字节透传。**

> 当输出是终端时，JSON、HTML、XML 与其它文本响应会被重排缩进并语法高亮；**当它被管道传递时，逐字节原样输出**。用 `--no-pretty` 退出这一行为，或在管道里用 `--pretty` 强行打开。

这条正好是"人类看的输出"与"机器吃的输出"的分界，**官方把它做成默认自适应**：终端里好看，管道里保真。你要在管道里拿格式化版本，就得显式 `--pretty`。

## `-i`、`-I`、`-v` 三个"看头"的开关差别

这三个长得很像，用错就是白跑一次：

| 我想要 | 用哪个 | 官方措辞 |
| --- | --- | --- |
| 响应体**加上**状态行与响应头 | `-i, --include` | 在输出中包含状态行与响应头 |
| **只要**响应头，不要 body | `-I, --head` | 发送 HEAD 请求并只显示响应头 |
| 请求头也想看，且不污染 stdout | `-v, --verbose` | 把请求与响应头打到 stderr |

注意第二条的代价：`-I` **发的是 `HEAD` 请求**。如果你的接口对 HEAD 的处理和 GET 不一样，它就不是"只看头的 GET"（这一句是我按 `-I` 的定义给的使用提醒，官方没有单独比较两者）。

## `nuxt task`：打的是 Nitro 任务

官方定义：

> `task` 命令与你项目暴露的 [Nitro tasks](https://nitro.build/guide/tasks) 通信，使用为该项目运行的 `nuxt dev` 服务器。**传 `--url` 可以指向另一个服务器。**

两个子命令：

```bash [Terminal]
npx nuxt task list [ROOTDIR] [--cwd=<directory>] [--url=<url>] [--json]
```

```bash [Terminal]
npx nuxt task run <NAME> [ROOTDIR] [--cwd=<directory>] [--url=<url>] [--payload=<json>]
```

| 项 | 官方说明 |
| --- | --- |
| `NAME=<name>` | 要运行的任务的名字 |
| `--url=<url>` | 要通信的 Nuxt 服务器 URL（**默认：正在运行的 dev 服务器**） |
| `--json` | 以 JSON 打印输出（在 `list` 上） |
| `--payload=<json>` | 任务载荷，**可以是一个 JSON 对象，也可以写成 `--payload.key=value` 这样的键值对** |

关于输出的官方一句：

> 任务的结果以 JSON 打印。载荷可以给出单个 JSON 对象，或从零散键拼起来：

```bash [Terminal]
npx nuxt task run db:seed --payload '{"count":10}'
npx nuxt task run db:seed --payload.count=10
```

## 但 `task list` 很可能是空的——原因在这里

官方用一个 note 直接给了这条排查结论：

> **Nitro 只在任务被启用时才扫描你的 `tasks` 目录**，所以一个没有暴露任何任务的服务器，可能只需要这个：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  nitro: {
    experimental: {
      tasks: true,
    },
  },
})
```

**这是"我明明写了任务但列不出来"的第一顺位怀疑对象**：不是文件名错了、不是路径错了，而是**这个特性默认没开**（官方把它放在 `nitro.experimental` 下面）。

顺带说清这条命令的适用边界：`task list` 空、`task run` 找不到名字，都是**服务端那一侧**的事，跟 041 前半段的 `nuxt curl` 无关——`curl` 打的是任意 HTTP 路径，不关心它背后是不是任务。

## `nuxt.lock` 把这几件事串起来了

到这里可以回收前几篇那条线了：

| 命令 | 和 `nuxt.lock` 的关系 |
| --- | --- |
| `nuxt dev` | 写它（033/034：第二个 dev 因此报告已运行的实例而不抢端口） |
| `nuxt curl` | **读它**来定位服务器（所以只在 dev 运行时有目标） |
| `nuxt task` | 同样默认打正在运行的 dev 服务器；要指别的用 `--url` |
| `nuxt cleanup` | 删的是构建目录本身（040），顺带把锁删掉 |

**"curl 说没有服务器"、"task 连不上"这类报错，先看 `.nuxt` 在不在、dev 在不在跑**，比检查 URL 更省力（这一条是我按上面这张表的因果关系给的建议）。

## 自查

- [ ] `nuxt curl /api/hello` 为什么不需要知道端口？它靠什么找到服务器？
- [ ] 接口返回 500 时 `nuxt curl` 的退出码是多少？这条对写脚本意味着什么？
- [ ] 你把输出接进 `jq`，官方默认会给它格式化过的还是原样的字节？
- [ ] `-I` 和 `-i` 的差别不只是"少一个 body"，还差在哪？
- [ ] `nuxt task list` 空白一片，官方给的第一个配置项怀疑是什么？

## 下一篇

042 把"我这边一切正常"变成一句可核对的话：`nuxt info` 收集哪些信息、`--json` 怎么用，以及报障时官方要求的四步与提问礼仪。

## 本文依据

- `api/commands/curl`（curl 向项目中运行的 nuxt dev 服务器发送 HTTP 请求并打印响应、路径按该服务器解析所以不必知道它最终落在哪个端口；参数 URL=<url|path>（绝对 URL 或按运行中 dev 服务器解析的路径）与 ROOTDIR 默认 .；选项 --cwd、-X/--method（默认 GET，提供 body 时 POST）、-H/--header Name: Value 可重复、-d/--data（@- 读 stdin、@<file> 读文件）、-i/--include 包含状态行与响应头、-I/--head 发送 HEAD 并只显示响应头、-v/--verbose 把请求与响应头打到 stderr、--pretty 重排与语法高亮（终端默认开、管道默认关，可用 --pretty/--no-pretty 强制）；通过 dev 写入构建目录的 nuxt.lock 找到 dev 服务器故仅在 nuxt dev 为该项目运行时可用，传绝对 URL 可与任何其它服务器通信；四条示例（/api/hello、/api/hello -i、POST JSON 到 /api/todos、-d @todo.json）；note 服务器回答 HTTP 错误状态时以 22 退出，匹配 curl --fail）
- `api/commands/task`（task 与项目暴露的 Nitro tasks 通信并使用为该项目运行的 nuxt dev 服务器，传 --url 指向其它服务器；nuxt task list 用法与 --cwd/--url（默认正在运行的 dev 服务器）/--json；nuxt task run <NAME> 用法与 NAME、--cwd、--url、--payload（单个 JSON 对象或 --payload.key=value 键值对）；任务结果以 JSON 打印并给出 db:seed 两种写法示例；note Nitro 只在 tasks 启用时才扫描你的 tasks 目录，故未暴露任务的服务器可能只需 nitro.experimental.tasks: true）
- `api/commands/dev`（dev 把自身记录进构建目录里的 nuxt.lock；同一 lock 文件也是 nuxt curl 与 nuxt task 找到要通信的服务器的方式）
- `directory-structure/nuxt`（.nuxt 是构建目录、跑 nuxt dev 时整个目录会被重建）
