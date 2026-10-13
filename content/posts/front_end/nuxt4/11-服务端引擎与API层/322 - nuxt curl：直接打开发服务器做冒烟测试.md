---
url: /posts/cac7c48b0efb2f5fcbd829c4c685ba28/
title: 322 - nuxt curl：直接打开发服务器做冒烟测试
date: 2027-01-27T10:00:00+08:00
updated: 2027-01-27T10:00:00+08:00
author: cmdragon

summary:
  摘要：写完一个 server/api 接口，想马上验一下它返回什么，通常得先知道 dev 服务器跑在哪个端口。nuxt curl 免掉这步：它把路径直接解析到"为这个项目正在跑的 nuxt dev 服务器"，发一条 HTTP 请求并把响应打出来。这一篇讲它的定位（靠 build 目录里的 nuxt.lock 找到那个 dev server，所以只在 dev 跑着时可用，传绝对 URL 可打别的服务器）、选项（-X 方法、-H 头、-d 体、-i 带响应头、-I 只 HEAD、-v、--pretty），和那条"遇到 HTTP 错误退出码 22"的行为。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt-curl
  - CLI
  - 冒烟测试
  - dev-server
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇 `nuxt task` 也是"对话正在跑的 dev 服务器"。这一篇的 `nuxt curl` 更直接：**给那个服务器发一条 HTTP 请求，看响应**——不用你先去查它到底占用哪个端口。

## 它解决什么

官方定位：

> The curl command sends an HTTP request to your running Nuxt dev server.

关键那句让"免记端口"成立：

> The `curl` command sends an HTTP request to the `nuxt dev` server running for your project and prints the response. A path is resolved against that server, so you do not need to know which port it ended up on.

也就是说你只写路径（`/api/hello`），它自己补上"你这个项目那个 dev server"的地址。

## 它靠什么找到那个服务器

官方说清了寻址和前提：

> The dev server is found through the `nuxt.lock` file it writes into your [build directory], so the command only works while `nuxt dev` is running for that project. Pass an absolute URL to talk to any other server.

三点：

- 靠 dev 服务器写在 build 目录（`.nuxt`）里的 **`nuxt.lock` 文件**定位。
- 因此 **只在 `nuxt dev` 为这个项目跑着时可用**。
- 传的是**绝对 URL** 就能打任意别的服务器（不必是本地 dev）。

## 命令形状与参数

```bash [Terminal]
npx nuxt curl <URL> [ROOTDIR] [--cwd=<directory>] [-X, --method=<method>] [-H, --header=<header>...] [-d, --data=<data>] [-i, --include] [-I, --head] [-v, --verbose] [--pretty]
```

参数（Arguments）：

| 项 | 说明 |
| --- | --- |
| `URL=<url\|path>` | 绝对 URL，或相对正在跑的 dev server 解析的路径 |
| `ROOTDIR` | 项目根目录，默认 `.` |

选项（Options）：

| 选项 | 说明 |
| --- | --- |
| `--cwd=<directory>` | 指定项目根目录 |
| `-X, --method=<method>` | HTTP 方法（默认 GET；给了 body 时默认 POST） |
| `-H, --header=<header>...` | 请求头，`Name: Value` 形式，可重复 |
| `-d, --data=<data>` | 请求体；`@-` 读 stdin、`@<file>` 读文件 |
| `-i, --include` | 输出里带上响应状态行和头 |
| `-I, --head` | 发 `HEAD` 请求、只显示响应头 |
| `-v, --verbose` | 把请求和响应头打到 stderr |
| `--pretty` | 重排缩进 + 语法高亮（终端默认开、管道时默认关） |

## 官方示例

```bash [Terminal]
# 请求正在跑的 dev server 上的一个路径
npx nuxt curl /api/hello

# 带上状态行和响应头
npx nuxt curl /api/hello -i

# 发 JSON
npx nuxt curl /api/todos -X POST -H "Content-Type: application/json" -d '{"title":"Buy milk"}'

# 从文件或 stdin 读 body（@-）
npx nuxt curl /api/todos -d @todo.json
```

这四条基本覆盖日常冒烟：GET 看返回、`-i` 看头、POST 带 body、从文件喂 body。

## 输出的处理规则

官方对"什么时候重排、什么时候原样"讲得很细：

> JSON, HTML, XML and other textual responses are reindented and syntax-highlighted when the output is a terminal, and passed through byte for byte when it is piped. Use `--no-pretty` to opt out, or `--pretty` to force it on when piping.

记一句：**给人看（终端）自动重排高亮；管道给别的命令时逐字节透传**（不改动）。要强制可 `--pretty`/`--no-pretty`。这也是"为什么我 `npx nuxt curl /api/x | jq` 拿到的是原样 JSON"——被管道时它不重排。

## 一条和脚本/CI 相关的退出码

官方 note：

> The command exits with `22` when the server answers with an HTTP error status, matching `curl --fail`.

服务端返回 HTTP 错误状态时，`nuxt curl` 以 **22** 退出（对齐 `curl --fail`）。这条在写校验脚本时有用——可以靠退出码判断这次冒烟是不是撞上了 4xx/5xx。

## 自查

- 冒烟一个刚写的接口：`npx nuxt curl /api/hello`，不用管端口，路径自动补到本项目的 dev server。
- 前提是这个项目的 `nuxt dev` 正在跑（靠 `.nuxt/nuxt.lock` 找）；要打别的服务器就传绝对 URL。
- 看头用 `-i`，只发 HEAD 用 `-I`，发 body 用 `-d`（`@file`/`@-`）、`-X` 指定方法、`-H` 加头。
- 管道给别的命令时输出逐字节透传；脚本里可用退出码 22 判 HTTP 错误。

## 下一篇

300-339 服务端这一层的 CLI 与请求相关主题到 322 先告一段落。下一篇 323 继续 300-339 那几篇，编号与日期连续推进。

## 本文依据

- nuxt curl（`api/commands/curl`）：定位句 "The curl command sends an HTTP request to your running Nuxt dev server."；"sends an HTTP request to the nuxt dev server running for your project and prints the response. A path is resolved against that server, so you do not need to know which port it ended up on."；"found through the nuxt.lock file it writes into your build directory, so the command only works while nuxt dev is running ... Pass an absolute URL to talk to any other server."；命令签名 + Arguments（URL、ROOTDIR 默认 .）+ Options 全表（--cwd/-X/-H/-d 含 @- 与 @file/-i/-I/-v/--pretty）；四示例；pretty 输出规则（终端重排高亮、管道逐字节透传、--no-pretty/--pretty）；note "exits with 22 when the server answers with an HTTP error status, matching curl --fail."
