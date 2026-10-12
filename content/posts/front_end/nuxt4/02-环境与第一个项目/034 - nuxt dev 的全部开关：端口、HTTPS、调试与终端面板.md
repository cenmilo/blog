---
url: /posts/c19761b30593a1493ea09aff3a8f5064/
title: nuxt dev 的全部开关：端口、HTTPS、调试与终端面板
date: 2026-10-23T10:00:00+08:00
updated: 2026-10-23T10:00:00+08:00
author: cmdragon

summary:
  摘要：nuxt dev 的命令行一共给了三十多个开关，可以分成六组：定位项目（ROOTDIR、--cwd、--dotenv、--envName、-e/--extends、--logLevel）、端口与主机（-p/--port、-h/--host、--takeover、--strictPort、-o/--open、--open.url、--clipboard、--qr、--tunnel、--public、--publicURL）、终端行为（--tui 默认为 true、--no-tui、--clear 默认 false、-f/--fork 默认随运行时而定、--no-fork）、HTTPS（--https 及 cert/key/pfx/passphrase/validityDays/domains 六个子参数）、调试与性能（--inspect、--inspect-brk、--profile）。官方还列了六个环境变量：NUXT_PORT/NITRO_PORT/PORT、NUXT_HOST/NITRO_HOST/HOST、NUXT_TUI、NUXT_TERM_THEME、NUXT_IGNORE_LOCK、NUXT_IGNORE_UPDATE_CHECK。三个容易踩的细节：--inspect 开的是 Nitro 工作线程的 127.0.0.1:9229，而 CLI 进程自己的 inspector 在下一个端口 9230；--https 装了 mkcert 才生成本机可信证书否则自签，而 Node 不读系统信任库，需要设 NODE_EXTRA_CA_CERTS；--profile 写出的 nuxt-dev.cpuprofile 任何版本都有，但构建耗时、perf-report.json 与 perf-trace.json 需要 Nuxt v4.4 或更新。

categories:
  - 前端开发

tags:
  - nuxt4
  - cli
  - 开发服务器
  - 调试
  - https

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


033 那张验收清单里，我们只用了 `nuxt dev` 的一小部分能力。这一篇把它的命令行**完整摊开一次**——不是为了背，是为了以后遇到"我要在手机上打开我本地那个页面""这个证书为什么浏览器不认""我要调试服务端代码"时，你知道官方其实给了开关，不必去改配置或写脚本。

## 用法行

```bash [Terminal]
npx nuxt dev [ROOTDIR] [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--dotenv=<path>...] [--envName=<environment>] [-e, --extends=<layer-name>...] [--inspect] [--inspect-brk] [--tui] [--clear] [-f, --fork] [-p, --port=<port>] [--takeover] [--strictPort] [-h, --host=<host>] [-o, --open] [--open.url=<url|path>] [--clipboard] [--qr] [--tunnel] [--public] [--publicURL=<url>] [--https] [--https.cert=<path>] [--https.key=<path>] [--https.pfx=<path>] [--https.passphrase=<passphrase>] [--https.validityDays=<days>] [--https.domains=<domain>...] [--profile=<verbose>]
```

唯一的位置参数是 `ROOTDIR`，默认 `.`。

官方补了一句总的定性：

> 本命令把 `process.env.NODE_ENV` 设为 `development`。

**这就是 dev 与 build/preview 的第一条分界**（035、039 会看到那两个命令把它设成 `production`）。

## 第一组：定位项目

| 开关 | 官方说明 |
| --- | --- |
| `--cwd=<directory>` | 指定 Nuxt 项目的根目录 |
| `--logLevel=<silent\|info\|verbose>` | 指定构建期日志级别 |
| `--dotenv=<path>...` | 要加载的 `.env` 路径，相对根目录。**可以重复传，靠后的文件优先级更高** |
| `--envName=<environment>` | 解析配置覆盖时使用的环境（**构建时默认 `production`，跑开发服务器时默认 `development`**） |
| `-e, --extends=<layer-name>...` | 从某个 Nuxt 图层继承 |

这一组里最容易被忽略的是 `--dotenv`：`024` 讲过跑起来的默认流程只读 `.env`，而这里可以显式指定别的文件、还能叠好几层。**"靠后的文件优先"这条规则，在 dev/build/generate/preview/prepare/typecheck 六个命令的说明里都是同一句原话**，所以你学一次就全都适用。

## 第二组：端口与对外可见性

| 开关 | 默认值 | 官方说明 |
| --- | --- | --- |
| `-p, --port=<port>` | | 监听的端口（默认 `NUXT_PORT \|\| NITRO_PORT \|\| PORT \|\| nuxtOptions.devServer.port`） |
| `-h, --host=<host>` | | 监听的主机（默认 `NUXT_HOST \|\| NITRO_HOST \|\| HOST \|\| nuxtOptions.devServer?.host`） |
| `--takeover` | | 停掉本项目已运行的 dev 服务器并在它的位置启动 |
| `--strictPort` | `false` | 请求的端口不可用时**退出**，而不是换一个 |
| `-o, --open` | `false` | 在浏览器打开 URL |
| `--open.url=<url\|path>` | | 要打开的路径或 URL，**替代**开发服务器根路径 |
| `--clipboard` | `false` | 把 URL 复制到剪贴板 |
| `--qr` | | 为公开 URL 打印二维码（**有公开 URL 时默认启用**） |
| `--tunnel` | | 通过 Cloudflare quick tunnel 暴露服务器 |
| `--public` | | 监听所有网络接口并允许任意 host 连接 |
| `--publicURL=<url>` | | 要显示的公开 URL（用于二维码与剪贴板） |

这一组回答的是三个具体问题：

**"我要用手机看本地那个页面"** → `--public`（监听所有网卡）。官方在说明里连着写了"并允许任意 host 连接"，这是个**安全相关的动作**，不是纯便利开关。

**"我要发给同事看"** → `--tunnel`。官方给的机制是 Cloudflare quick tunnel；这条会把你的本地服务器暴露到公网，**看完就停**。

**"我不想每次都手输地址"** → `-o`（开浏览器）、`--clipboard`（复制 URL）、`--qr`（扫码）。`--open.url` 适合你固定要看某个子路由的场景（例如每次都从 `/admin` 起）。

## 第三组：终端行为

| 开关 | 默认值 | 官方说明 |
| --- | --- | --- |
| `--tui` | `true` | 交互式终端 UI（固定状态面板、折叠日志、单键快捷键） |
| `--no-tui` | | 关闭交互式终端 UI，改为流式输出日志 |
| `--clear` | `false` | 重启时清屏 |
| `-f, --fork` | 取决于运行时 | 由 fork 出的子进程提供应用服务（**在运行时支持的平台上默认开启**） |
| `--no-fork` | | 关闭 fork 模式 |

关于 `--tui` 的关闭条件，官方正文比选项表说得更完整（033 抄过）：**输出不是终端、在 CI、附着了调试器、终端太小**都会自动退回纯日志流。手动控制的开关是 `--no-tui` 或 `NUXT_TUI=plain`，也可以 `NUXT_TUI=1` 强制打开——**但官方加了一句：输出被管道或重定向时强制打开也没用。**

`--fork` 这条只给事实：默认"运行时支持就开"。官方在这里没解释关掉之后有什么差别，我不替它补。

## 第四组：HTTPS

官方的一段完整说明：

> `--https` 通过 TLS 提供服务，装了 `mkcert` 时生成一张**本机可信**的证书，否则生成**自签**证书。`--https.cert` 与 `--https.key` 使用你已有的证书，`--https.pfx` 配 `--https.passphrase` 使用 PKCS#12 密钥库。

子参数表：

| 开关 | 说明 |
| --- | --- |
| `--https.cert=<path>` | TLS 证书路径 |
| `--https.key=<path>` | TLS 私钥路径 |
| `--https.pfx=<path>` | PKCS#12（.p12/.pfx）密钥库路径 |
| `--https.passphrase=<passphrase>` | TLS 私钥或密钥库的口令 |
| `--https.validityDays=<days>` | 生成的自签证书的有效期天数 |
| `--https.domains=<domain>...` | 生成证书用的域名，**可重复传，也可用逗号分隔的列表** |

然后是那条会坑人的 note，**逐字读**：

> Node 不读取你的系统信任库，所以**从 Node 发出的、访问使用生成证书的服务器的请求，不会信任它**。请把 `NODE_EXTRA_CA_CERTS` 设为签发它的证书机构。

这条的实操含义：**浏览器绿锁了，不代表你的服务端代码内部再请求自己也能通。** 浏览器读系统信任库，Node 不读——这不是 Nuxt 的怪癖，是 Node 的行为，官方在这里专门提醒，说明真的会撞。

## 第五组：调试与性能

**`--inspect`：开在服务端代码那一侧。**

> `--inspect` 为你的服务端代码打开 Node.js inspector：**服务端路由、中间件与服务端渲染，它们运行在一个 Nitro 工作线程里**。它接受可选的 `[host:]port`，默认是 `127.0.0.1:9229`，所以 Chrome DevTools（`chrome://inspect`）与大多数编辑器无需配置就能找到它。**服务器 reload 时你的调试器会自动重连。**

**`--inspect` 还有第二个端口，官方单独开了一节讲：**

> `nuxt.config`、模块与构建钩子运行在 **CLI 进程**里，它有自己的 inspector 在**下一个端口（默认 `9230`）**。在 `chrome://inspect` 里把 `localhost:9230` 加为一个目标（**Configure...**），或让你的编辑器指向它。

**这两段合起来是一条很硬的诊断信息：断点没命中，先看你打在哪一侧。** 页面渲染/接口的代码在 9229（工作线程），配置与模块加载在 9230（CLI 进程）。

**`--inspect-brk`：要调试"Nuxt 加载期间"的代码时用。**

> 要调试在 Nuxt 加载过程中就运行的代码，使用 `--inspect-brk`。CLI 进程随后会**等待调试器附着到下一个端口**才加载 Nuxt：

```bash
npx nuxt dev --inspect-brk
```

**`--profile`：写到哪个文件。**

> `--profile` 在进程退出时把 V8 CPU profile 写到项目里的 `nuxt-dev.cpuprofile`。`--profile=verbose` 还会向控制台打印完整报告。
>
> （note）构建耗时、`perf-report.json` 与 `perf-trace.json` 来自 Nuxt 自己的构建剖析，**需要 Nuxt v4.4 或更新版本**。在更早的版本上，CPU profile 仍然会写出来。

这条要记住的是一句"部分有效"：**旧版本上 `--profile` 不是白跑**（cpuprofile 照样有），只是少了那份耗时分解报告。

## 第六组：官方列的环境变量

这一页末尾给了六行表，我把它当"开关的另一种写法"收在这里：

| 变量 | 用途 |
| --- | --- |
| `NUXT_PORT`、`NITRO_PORT`、`PORT` | 要监听的端口，按此优先级 |
| `NUXT_HOST`、`NITRO_HOST`、`HOST` | 要监听的主机，按此优先级 |
| `NUXT_TUI` | `1` 强制打开交互式 UI，`plain` 退出 |
| `NUXT_TERM_THEME` | `light` 或 `dark`，用于终端背景色无法检测时 |
| `NUXT_IGNORE_LOCK` | `1` 时忽略本项目已运行的 dev 服务器 |
| `NUXT_IGNORE_UPDATE_CHECK` | `1` 时停止 CLI 检查更新的 Nuxt 版本 |

后两条值得单独说一句：`NUXT_TERM_THEME` 解决的是**面板配色在特殊终端下看不清**；`NUXT_IGNORE_UPDATE_CHECK` 解决的是**离线或 CI 环境里不该发检查请求**。这两项都不是"性能优化"，是让终端可读、让构建可复现。

## 自查

- [ ] 每次 dev 起来后自动检查更新，你想关掉，用哪个环境变量？
- [ ] 断点打在服务端路由里没命中，`--inspect` 的默认地址和端口是什么？该连哪个？
- [ ] 你要调试 `nuxt.config` 与模块加载，端口是哪个？要用哪个开关在加载前就停下来？
- [ ] 浏览器信任你的本地证书，但服务端内部请求报证书错误，官方给的处理是什么？
- [ ] `--dotenv` 传了三个文件，最终谁生效？`--envName` 在 dev 时的默认值是什么？

## 下一篇

035 讲两个"看起来没干什么活"的命令：`nuxt prepare` 与 `nuxt typecheck`——前者生成 `.nuxt` 里的类型，后者才是真的检查它们。顺便纠正一个直觉：这两个命令都把 NODE_ENV 设成了 `production`。

## 本文依据

- `api/commands/dev`（用法行全部选项与 ROOTDIR 默认 .；本命令设 process.env.NODE_ENV 为 development；选项表逐项：--cwd、--logLevel、--dotenv（可重复、靠后优先）、--envName（构建默认 production、dev 默认 development）、-e/--extends、--inspect（Enable the Node.js inspector for server code (--inspect=[host:]port), and for the CLI process on the next port）、--inspect-brk、--tui 默认 true、--no-tui、--clear 默认 false、-f/--fork 默认取决于运行时（on by default wherever the runtime supports it）、--no-fork、-p/--port 默认 NUXT_PORT||NITRO_PORT||PORT||nuxtOptions.devServer.port、--takeover/--no-takeover、--strictPort 默认 false、-h/--host 默认 NUXT_HOST||NITRO_HOST||HOST||nuxtOptions.devServer?.host、-o/--open 默认 false、--open.url、--clipboard 默认 false、--qr（enabled by default when one is available）、--tunnel（Cloudflare quick tunnel）、--public（listen on all network interfaces and allow any host to connect）、--publicURL、--https 与 https.cert/key/pfx/passphrase/validityDays/domains、--profile；HTTPS 一节：mkcert 生成本机可信否则自签、cert+key 用已有证书、pfx+passphrase 用 PKCS#12、note"Node does not read your system trust store… Set NODE_EXTRA_CA_CERTS to the certificate authority that issued it"；Debugging 一节：--inspect 覆盖 server routes/middleware/SSR 且它们运行在 Nitro worker thread、接受可选 [host:]port、默认 127.0.0.1:9229、Chrome DevTools 与多数编辑器无需配置、reload 后调试器自动重连；Debugging the CLI：nuxt.config/modules/build hooks 跑在 CLI 进程、其 inspector 在下一个端口默认 9230、在 chrome://inspect 的 Configure 里加 localhost:9230；--inspect-brk 让 CLI 在加载 Nuxt 前等待调试器附着于下一个端口并给出示例 npx nuxt dev --inspect-brk；--profile 退出时写 nuxt-dev.cpuprofile、=verbose 打印完整控制台报告、note 构建耗时与 perf-report.json/perf-trace.json 来自 Nuxt 自身构建剖析需 v4.4+，更旧版本仍写 CPU profile；Environment variables 表六行 NUXT_PORT/NITRO_PORT/PORT、NUXT_HOST/NITRO_HOST/HOST、NUXT_TUI、NUXT_TERM_THEME、NUXT_IGNORE_LOCK、NUXT_IGNORE_UPDATE_CHECK；禁用 UI 一节：--no-tui 或 NUXT_TUI=plain、NUXT_TUI=1 强制开启但对管道或重定向无效）
